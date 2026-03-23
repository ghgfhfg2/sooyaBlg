---
layout: post
title: "Node.js Rate Limiting 가이드: Token Bucket과 Redis로 API 남용 막기"
date: 2026-03-23 20:00:00 +0900
lang: ko
translation_key: nodejs-rate-limiting-token-bucket-redis-api-protection-guide
permalink: /development/blog/seo/2026/03/23/nodejs-rate-limiting-token-bucket-redis-api-protection-guide.html
alternates:
  ko: /development/blog/seo/2026/03/23/nodejs-rate-limiting-token-bucket-redis-api-protection-guide.html
  x_default: /development/blog/seo/2026/03/23/nodejs-rate-limiting-token-bucket-redis-api-protection-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, rate-limiting, token-bucket, redis, api, backend]
description: "Node.js API에서 rate limiting을 설계할 때 fixed window의 한계, token bucket의 장점, Redis 기반 분산 환경 구현 기준, 운영 지표와 우회 방지 포인트를 실무 중심으로 정리했습니다."
---

API를 공개하거나 모바일 앱·웹 프런트엔드 뒤에 붙여 운영하다 보면 결국 한 번은 **rate limiting** 문제를 만나게 됩니다.
평소에는 멀쩡하던 서버가 특정 IP의 과도한 호출, 로그인·인증 시도 남발, 크롤러의 짧은 폭주, 혹은 실수로 잘못 짠 클라이언트 재시도 때문에 갑자기 흔들리기 시작합니다.
이 글에서는 Node.js 서비스에서 **rate limiting** 을 왜 붙여야 하는지부터, `fixed window` 방식의 한계, **token bucket** 기반 설계, 그리고 **Redis를 이용한 분산 환경 구현 기준**까지 실무 관점에서 정리합니다.

## 왜 Node.js API에 rate limiting이 꼭 필요할까

### H3. 보안 문제만이 아니라 안정성 문제이기도 하다

rate limiting을 흔히 "봇 차단" 정도로만 생각하지만, 실제로는 **서비스 보호 장치**에 가깝습니다.
다음 같은 상황에서 특히 중요합니다.

- 로그인, 비밀번호 재설정, OTP 검증 같은 인증 엔드포인트
- 검색, 피드 조회, 추천 API처럼 호출 빈도가 높은 읽기 API
- 비용이 큰 AI 호출, 외부 결제 연동, 대용량 집계 API
- 파트너 API나 공개 API처럼 외부 클라이언트가 직접 붙는 구간

이런 구간은 악의적인 남용이 아니더라도 쉽게 무너질 수 있습니다.
예를 들어 모바일 앱이 버그로 1초마다 같은 요청을 반복하거나, 재시도 로직이 잘못되어 실패한 요청을 동시 다발로 다시 보내면 서버 입장에서는 공격과 비슷한 트래픽으로 보일 수 있습니다.

### H3. rate limiting이 없으면 장애가 더 빠르게 전파된다

문제는 단순히 요청 수가 늘어나는 데서 끝나지 않습니다.
상류 서비스에서 과도한 요청을 그대로 받으면 아래처럼 연쇄 반응이 생길 수 있습니다.

- Node.js 이벤트 루프가 과도한 요청 처리로 바빠진다
- DB 커넥션 풀이 빨리 고갈된다
- 외부 API 호출 비용이 급증한다
- 캐시 미스가 겹치면 backend 부하가 더 커진다
- 장애 순간에 클라이언트 재시도가 붙으며 상황이 악화된다

즉 rate limiting은 트래픽을 거절하는 장치이면서 동시에 **나머지 정상 사용자 경험을 지키는 장치**이기도 합니다.
이 점은 <a href="/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html">Node.js 재시도 전략 가이드</a>에서 다룬 재시도 증폭 문제와도 직접 연결됩니다.

## 어떤 기준으로 제한해야 할까

### H3. IP 하나만 보면 충분하지 않은 경우가 많다

가장 단순한 방식은 IP 기준 제한입니다.
간단하고 구현도 쉽지만, 실제 운영에서는 이것만으로 부족한 경우가 많습니다.

예를 들어 아래처럼 기준을 나눠야 할 수 있습니다.

- 비로그인 API: IP 기준
- 로그인 사용자 API: userId 기준
- 파트너 API: API key 기준
- 관리자 기능: 조직 또는 role 기준
- 민감한 액션: 엔드포인트별 별도 기준

NAT 환경이나 모바일 통신망에서는 여러 사용자가 같은 공인 IP를 공유할 수 있어서, IP 하나만으로 강하게 막으면 정상 사용자까지 같이 막을 위험이 있습니다.
반대로 API key나 userId만 보면, 비로그인 엔드포인트 공격에는 대응이 약할 수 있습니다.

### H3. 글로벌 한도와 엔드포인트별 한도를 분리하는 편이 안전하다

실무에서는 보통 한 가지만 두지 않습니다.
아래처럼 여러 레벨을 조합하는 경우가 많습니다.

- 전체 API 공통 한도: 초당 또는 분당 총량 보호
- 로그인/회원가입 한도: brute-force 방어
- 검색/비용 큰 API 한도: 비용·성능 보호
- webhook/callback 제외 또는 별도 정책: 공급자 재전송 허용 범위 고려

특히 로그인, 인증번호 전송, 파일 업로드, AI 추론 요청처럼 비용이나 민감도가 높은 API는 별도 정책이 필요합니다.
모든 엔드포인트에 똑같이 `100 req/min`을 적용하면 운영하기는 편해 보여도, 실제 보호 효과는 약할 수 있습니다.

## fixed window 방식은 왜 생각보다 거칠까

### H3. 구현은 쉽지만 경계 시점에 트래픽이 몰릴 수 있다

가장 많이 떠올리는 방식은 `1분 동안 100회` 같은 fixed window입니다.
예를 들어 현재 분(minute)을 키로 잡고 카운트를 올리는 식입니다.
이 방식은 구현이 단순하고 이해도 쉽습니다.

하지만 경계 시점 문제가 있습니다.
사용자가 12:00:59에 100번 요청하고, 12:01:00에 다시 100번 요청하면 사실상 짧은 순간에 200번이 들어갈 수 있습니다.
즉 평균적으로는 제한된 것처럼 보여도, **순간 스파이크를 충분히 완화하지 못할 수 있습니다.**

### H3. 완전히 틀린 방식은 아니지만 목적에 따라 부족할 수 있다

fixed window가 항상 나쁜 것은 아닙니다.
관리자 기능, 낮은 트래픽 내부 API, 간단한 abuse 방지에는 여전히 실용적입니다.
하지만 아래 요구가 있으면 더 부드러운 방식이 낫습니다.

- 순간 폭주를 완화하고 싶다
- 사용자 경험을 너무 거칠게 자르고 싶지 않다
- 분산 환경에서 보다 일관된 제어가 필요하다
- burst는 약간 허용하되 평균 속도는 제한하고 싶다

이럴 때 자주 검토하는 것이 **sliding window** 나 **token bucket** 입니다.

## token bucket은 무엇이 다른가

### H3. 짧은 burst는 허용하고 지속 남용은 막는 데 유리하다

token bucket은 일정 속도로 토큰이 채워지고, 요청이 올 때마다 토큰을 하나씩 소비하는 방식입니다.
버킷에 토큰이 남아 있으면 요청을 허용하고, 없으면 거절합니다.

예를 들어 아래처럼 설계할 수 있습니다.

- 버킷 크기: 20
- 토큰 보충 속도: 초당 5개

이 경우 사용자는 짧은 순간에 최대 20개까지 burst 요청을 보낼 수 있지만, 장기적으로는 초당 5개 수준 이상 계속 보내기 어렵습니다.
이 특성이 중요한 이유는 실서비스 트래픽이 늘 일정하지 않기 때문입니다.
정상 사용자도 화면 전환이나 초기 로딩 시 잠깐 여러 요청을 보낼 수 있는데, token bucket은 이런 **짧고 정상적인 스파이크**를 어느 정도 수용합니다.

### H3. 사용자 경험과 시스템 보호 사이 균형을 잡기 좋다

rate limiting은 너무 느슨하면 의미가 없고, 너무 빡빡하면 정상 사용자를 괴롭힙니다.
token bucket은 이 균형을 잡기 좋습니다.

- 순간 burst 허용
- 평균 속도 제한
- 구현 로직이 비교적 명확함
- 다양한 키(IP, userId, API key)에 재사용 가능

즉 "갑자기 몇 번 빠르게 쓰는 것"과 "계속 과하게 쓰는 것"을 어느 정도 구분해 다루기 쉽습니다.

## Node.js에서 Redis 기반으로 구현할 때 핵심은 무엇일까

### H3. 서버 메모리만 쓰면 인스턴스가 여러 대일 때 깨진다

Express 미들웨어 안에서 메모리 객체로 카운트를 들고 가는 예시는 많습니다.
로컬 개발이나 단일 인스턴스에서는 충분할 수 있지만, 운영에서 서버가 여러 대면 문제가 생깁니다.

- 인스턴스마다 카운트가 따로 쌓인다
- 로드밸런싱에 따라 제한 결과가 들쭉날쭉해진다
- 재시작 시 카운트가 모두 초기화된다
- autoscaling 환경에서는 정책 일관성이 무너진다

그래서 분산 환경에서는 보통 Redis 같은 **공유 저장소 기반 rate limiter**를 둡니다.

### H3. 읽기-수정-쓰기 분리를 줄이고 원자적으로 처리해야 한다

rate limiting은 트래픽이 많을수록 경쟁 조건(race condition)에 민감합니다.
예를 들어 아래처럼 분리하면 위험합니다.

1. 현재 토큰 수를 읽는다
2. 계산한다
3. 다시 저장한다

동시에 여러 요청이 들어오면 모두 같은 이전 값을 읽고 허용해버릴 수 있습니다.
그래서 Redis를 쓸 때는 `MULTI/EXEC`, `EVAL(Lua)`, 또는 이미 검증된 라이브러리를 활용해 **원자적 처리**를 보장하는 편이 안전합니다.
이 부분은 <a href="/development/blog/seo/2026/03/21/nodejs-etag-if-match-optimistic-locking-concurrency-guide.html">Node.js 동시성 제어 가이드</a>에서 다룬 경쟁 조건 사고방식과도 닿아 있습니다.

## Redis + token bucket 예시

### H3. 개념적으로는 마지막 갱신 시점과 현재 토큰 수를 함께 저장한다

아래 예시는 개념 이해를 위한 단순화 버전입니다.
실운영에서는 Lua 스크립트나 검증된 라이브러리 사용을 권장하지만, 핵심 아이디어는 비슷합니다.

```ts
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

type LimitResult = {
  allowed: boolean;
  remainingTokens: number;
  retryAfterSec: number;
};

export async function allowRequest({
  key,
  capacity,
  refillPerSec,
}: {
  key: string;
  capacity: number;
  refillPerSec: number;
}): Promise<LimitResult> {
  const now = Date.now();
  const redisKey = `rate-limit:${key}`;

  const raw = await redis.hgetall(redisKey);

  const lastRefillAt = raw.lastRefillAt ? Number(raw.lastRefillAt) : now;
  const currentTokens = raw.tokens ? Number(raw.tokens) : capacity;

  const elapsedSec = Math.max(0, (now - lastRefillAt) / 1000);
  const refilledTokens = Math.min(capacity, currentTokens + elapsedSec * refillPerSec);

  if (refilledTokens < 1) {
    const retryAfterSec = Math.ceil((1 - refilledTokens) / refillPerSec);

    await redis.expire(redisKey, Math.ceil(capacity / refillPerSec) * 2);

    return {
      allowed: false,
      remainingTokens: 0,
      retryAfterSec,
    };
  }

  const nextTokens = refilledTokens - 1;

  await redis.hset(redisKey, {
    tokens: nextTokens,
    lastRefillAt: now,
  });
  await redis.expire(redisKey, Math.ceil(capacity / refillPerSec) * 2);

  return {
    allowed: true,
    remainingTokens: Math.floor(nextTokens),
    retryAfterSec: 0,
  };
}
```

이 코드는 개념 설명용이라 원자성 측면에서는 보완이 필요합니다.
그래도 아래 포인트는 잘 보여줍니다.

- 키별로 버킷 상태를 저장한다
- 시간 경과만큼 토큰을 다시 채운다
- 토큰이 1개 이상이면 허용한다
- 없으면 `retryAfter`를 계산한다

### H3. Express 미들웨어에서는 응답 헤더까지 함께 설계하는 편이 좋다

클라이언트가 왜 막혔는지, 언제 다시 시도할 수 있는지 알 수 있어야 운영이 편합니다.
예를 들면 아래처럼 응답할 수 있습니다.

```ts
import type { Request, Response, NextFunction } from 'express';

export async function rateLimitMiddleware(req: Request, res: Response, next: NextFunction) {
  const clientIp = req.ip;
  const userId = req.user?.id;
  const key = userId ? `user:${userId}` : `ip:${clientIp}`;

  const result = await allowRequest({
    key,
    capacity: 20,
    refillPerSec: 5,
  });

  res.setHeader('X-RateLimit-Limit', '20');
  res.setHeader('X-RateLimit-Remaining', String(result.remainingTokens));

  if (!result.allowed) {
    res.setHeader('Retry-After', String(result.retryAfterSec));

    return res.status(429).json({
      message: 'Too Many Requests',
      retryAfterSec: result.retryAfterSec,
    });
  }

  return next();
}
```

운영 관점에서는 단순히 429를 내는 것보다, **재시도 기준을 클라이언트에 명확히 전달하는 것**이 중요합니다.
그래야 잘 만든 클라이언트가 불필요한 폭주를 줄일 수 있습니다.

## 어떤 값을 기본값으로 잡아야 할까

### H3. 엔드포인트 민감도와 비용부터 본다

정답 숫자는 없습니다.
중요한 것은 평균 호출량이 아니라 **남용 시 피해 비용**입니다.
예를 들어 아래처럼 접근할 수 있습니다.

- 로그인: 매우 낮게, 짧은 시간 강하게 제한
- 일반 조회 API: 사용자 경험을 해치지 않는 선에서 여유 있게 설정
- AI/비용 큰 API: 계정 단위로 강하게 제한
- 관리자/배치 API: 내부 호출 특성 반영

동일한 서비스라도 `/login` 과 `/feed` 와 `/ai/generate` 는 전혀 다른 정책이 필요합니다.

### H3. 숫자를 감으로 정하지 말고 관측값과 함께 조정한다

초기엔 대략적인 값으로 시작할 수밖에 없지만, 운영에서는 반드시 지표를 봐야 합니다.
예를 들면 다음을 함께 관찰합니다.

- 429 응답 비율
- 엔드포인트별 초당 요청 수
- 사용자·IP별 상위 호출량 분포
- 차단 직전 구간 사용자 이탈 여부
- 차단 후 재시도 패턴 변화

즉 rate limiting은 보안 설정이면서 동시에 **운영 튜닝 파라미터**입니다.
한 번 넣고 끝나는 기능이 아니라, 관측하고 조정해야 제대로 작동합니다.

## 우회와 오탐을 줄이려면 무엇을 봐야 할까

### H3. 프록시 환경에서는 실제 클라이언트 식별을 신중히 처리해야 한다

`req.ip` 하나 믿고 구현했다가, 프록시나 CDN 뒤에서 모든 요청이 같은 IP로 잡히는 실수를 자주 합니다.
Node.js/Express 환경에서는 `trust proxy` 설정과 전달 헤더 해석을 정확히 맞춰야 합니다.
그렇지 않으면 아래 문제가 생깁니다.

- 모든 사용자가 같은 IP로 집계된다
- 공격자가 조작한 헤더를 그대로 믿게 된다
- 정상 사용자가 한꺼번에 차단된다

즉 식별 키는 애플리케이션 로직이 아니라 **네트워크 토폴로지와 함께** 봐야 합니다.

### H3. 차단 응답이 너무 자세하면 공격자에게 힌트를 줄 수 있다

클라이언트 친화적이어야 한다고 해서, 언제 버킷이 가득 차는지나 내부 정책을 지나치게 자세히 노출할 필요는 없습니다.
보통은 아래 정도면 충분합니다.

- 429 상태 코드
- 간단한 에러 메시지
- `Retry-After`
- 필요 시 남은 횟수 헤더

보안 민감 엔드포인트라면 남은 횟수까지 굳이 자세히 알려주지 않는 편이 나을 수도 있습니다.
정답은 없지만, **사용성 정보와 공격 힌트 제공 사이 균형**을 잡아야 합니다.

## rate limiting만으로 끝내면 안 되는 이유

### H3. 재시도 정책, 캐시, 큐잉과 함께 봐야 실제 효과가 난다

rate limiting은 혼자서 모든 문제를 해결하지 않습니다.
예를 들어 클라이언트가 429를 받자마자 즉시 재시도하면 서버는 계속 압박을 받습니다.
그래서 아래 요소를 같이 맞추는 편이 좋습니다.

- 클라이언트의 backoff + jitter 적용
- 비싼 읽기 API의 캐시 전략
- 비동기 처리 가능한 작업의 큐잉
- 멱등성 보장이 필요한 쓰기 요청 보호

이 관점에서 보면 rate limiting은 단일 미들웨어가 아니라, **서비스 전체의 overload control 전략** 중 하나입니다.
캐시 스탬피드 방지는 <a href="/development/blog/seo/2026/03/20/nodejs-cache-stampede-prevention-redis-singleflight-guide.html">이 글</a>도 함께 보면 연결이 잘 됩니다.

## 실무 체크리스트

### H3. 배포 전 최소 점검 항목

rate limiting을 붙이기 전에는 아래를 점검해두는 편이 좋습니다.

- 어떤 기준으로 제한할지 결정했는가: IP, userId, API key, 조합 키
- 전역 정책과 엔드포인트별 정책을 분리했는가
- 단일 인스턴스가 아닌 분산 환경을 고려했는가
- Redis 연산이 원자적으로 처리되는가
- 429 응답과 `Retry-After` 정책이 일관적인가
- 프록시/CDN 환경에서 실제 클라이언트 식별이 올바른가
- 지표와 로그로 차단 현황을 추적할 수 있는가

### H3. 먼저 거칠게 시작하고, 운영 데이터로 다듬는 편이 현실적이다

처음부터 완벽한 수치를 맞추려 하기보다, 명백히 위험한 구간부터 제한을 두고 관측하면서 조정하는 편이 현실적입니다.
다만 인증·결제·비용 큰 API처럼 실패 비용이 높은 구간은 초기에 더 보수적으로 잡는 것이 낫습니다.

## 마무리

Node.js에서 rate limiting은 "봇 좀 막아보자" 수준의 부가 기능이 아니라, **정상 사용자 경험과 시스템 안정성을 지키는 핵심 보호 장치**에 가깝습니다.
특히 서버가 여러 대이고, 클라이언트 종류가 다양하며, 재시도·캐시·외부 API 비용이 얽힌 서비스일수록 단순 카운터보다는 **분산 환경에 맞는 정책과 저장소 설계**가 중요해집니다.

실무 기준으로는 아래처럼 기억해두면 도움이 됩니다.

- 단순한 fixed window는 쉽지만 순간 스파이크에 거칠 수 있다
- token bucket은 burst 허용과 평균 제한의 균형을 잡기 좋다
- 분산 환경에서는 Redis 기반 원자 처리 설계가 중요하다
- 429만 내지 말고 `Retry-After` 와 관측 지표까지 함께 설계해야 한다
- rate limiting은 재시도·캐시·큐잉과 함께 봐야 효과가 커진다

서비스가 커질수록 "얼마나 빠르게 처리하느냐" 못지않게 "언제 정중하게 거절하느냐"가 중요해집니다.
rate limiting은 그 거절을 시스템적으로 수행하는 가장 기본적인 도구입니다.

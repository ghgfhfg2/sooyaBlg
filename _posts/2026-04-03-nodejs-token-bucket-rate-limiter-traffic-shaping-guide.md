---
layout: post
title: "Node.js Token Bucket Rate Limiter 가이드: 트래픽 급증에도 API를 안정적으로 보호하는 방법"
date: 2026-04-03 08:00:00 +0900
lang: ko
translation_key: nodejs-token-bucket-rate-limiter-traffic-shaping-guide
permalink: /development/blog/seo/2026/04/03/nodejs-token-bucket-rate-limiter-traffic-shaping-guide.html
alternates:
  ko: /development/blog/seo/2026/04/03/nodejs-token-bucket-rate-limiter-traffic-shaping-guide.html
  x_default: /development/blog/seo/2026/04/03/nodejs-token-bucket-rate-limiter-traffic-shaping-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, rate-limiter, token-bucket, traffic-shaping, api, express, redis, resilience]
description: "Node.js 서비스에서 Token Bucket Rate Limiter를 설계해 순간 트래픽 급증을 흡수하고, API 남용과 과부하를 안정적으로 제어하는 실무 방법을 정리했습니다."
---

API를 운영하다 보면 평소에는 멀쩡하던 서버가 특정 순간에 갑자기 버거워지는 때가 있습니다.
봇이 한 엔드포인트를 과하게 두드리거나, 마케팅 캠페인 직후 요청이 몰리거나, 특정 클라이언트가 재시도 로직을 잘못 구현해 짧은 시간에 요청을 폭발시키는 경우가 대표적입니다.
문제는 이런 상황이 꼭 대규모 DDoS처럼 보이지는 않는다는 점입니다.
정상 사용자 요청과 비정상적 요청이 섞여 들어오면서, 중요한 API까지 같이 느려지고 실패하기 시작합니다.

이럴 때 필요한 것이 **rate limiter**이고, 그중에서도 실무에서 자주 쓰이는 방식이 **token bucket**입니다.
Token bucket은 모든 요청을 무조건 막는 방식이 아니라, **짧은 순간의 burst는 허용하되 지속적인 과속은 제한하는 전략**에 가깝습니다.
이 글에서는 Node.js 환경에서 token bucket rate limiter가 왜 필요한지, fixed window와는 무엇이 다른지, Redis를 곁들여 운영 가능한 형태로 설계하려면 무엇을 봐야 하는지 실무 관점에서 정리합니다.

## Node.js Token Bucket Rate Limiter가 필요한 이유

### H3. 평균 트래픽이 아니라 순간적인 급증이 시스템을 먼저 망가뜨린다

많은 팀이 rate limiting을 이야기할 때 초당 평균 요청 수부터 봅니다.
물론 중요한 지표입니다.
하지만 실제 장애는 평균보다 **짧은 시간에 얼마나 몰리는가**에서 더 자주 시작됩니다.

예를 들어 아래 같은 상황을 떠올려볼 수 있습니다.

- 로그인 API에 특정 클라이언트의 재시도 요청이 1초에 수십 건씩 몰림
- 인기 상품 오픈 직후 같은 상품 조회 요청이 폭증함
- 외부 파트너가 webhook 재전송을 공격적으로 수행함
- 크롤러가 검색 API를 짧은 간격으로 반복 호출함

이런 요청을 전부 즉시 거절하면 정상 사용자 경험까지 지나치게 나빠질 수 있습니다.
반대로 전부 받아주면 서버 자원과 downstream 의존성이 빠르게 소진됩니다.
Token bucket이 유용한 이유는 여기 있습니다.
**순간적인 스파이크는 어느 정도 받아주고, 지속적으로 과한 속도만 제어할 수 있기 때문**입니다.

### H3. rate limiter는 보안 기능이면서 동시에 가용성 보호 장치다

rate limiter를 흔히 남용 방지 기능으로만 보는 경우가 있습니다.
하지만 운영 관점에서는 그것보다 **가용성 보호 장치**라는 의미가 더 큽니다.
요청량이 갑자기 늘 때 제대로 된 limiter가 없으면 아래 문제가 함께 발생합니다.

- worker pool이 빠르게 포화됨
- DB connection이 치솟아 정상 요청도 대기함
- 외부 API quota를 소모해 연쇄 실패가 시작됨
- timeout과 retry가 다시 부하를 키우는 악순환이 생김

즉 rate limiting은 누군가를 벌주는 기능이 아니라, **전체 시스템을 살리기 위해 속도를 조절하는 운영 장치**입니다.
이 점에서 [Node.js Admission Control 가이드](/development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html)와도 자연스럽게 연결됩니다.
입구에서 무조건 다 받지 않는다는 점은 같지만, token bucket은 그중에서도 더 유연하게 burst를 다루는 방식입니다.

## Token Bucket은 무엇인가

### H3. 일정 속도로 토큰이 채워지고 요청은 토큰을 하나씩 소비하는 방식이다

Token bucket은 이해하면 꽤 직관적입니다.
버킷 안에 토큰이 일정 속도로 채워지고, 요청이 들어올 때마다 토큰을 하나 꺼내 씁니다.
토큰이 남아 있으면 요청을 허용하고, 토큰이 비어 있으면 제한합니다.

예를 들어 아래처럼 생각할 수 있습니다.

- 버킷 최대 크기: 20
- 토큰 보충 속도: 초당 5개
- 요청 1건당 토큰 1개 소비

이 경우 평소에는 초당 5건 수준으로 꾸준히 처리할 수 있습니다.
하지만 한순간에 15건이 몰려도 버킷에 토큰이 쌓여 있었다면 흡수할 수 있습니다.
반대로 오래도록 초당 30건이 들어오면 토큰이 바닥나면서 제한이 걸립니다.

핵심은 단순합니다.
**지속 가능한 평균 속도와 허용 가능한 순간 burst를 동시에 표현할 수 있다**는 점입니다.

### H3. fixed window보다 경계 효과가 덜하고, leaky bucket보다 burst 친화적이다

rate limiting에는 여러 방식이 있습니다.
Token bucket이 자주 선택되는 이유는 장단점 균형이 좋기 때문입니다.

간단히 비교하면 아래와 같습니다.

- fixed window: 구현은 쉽지만 윈도 경계에서 요청이 몰리면 실제보다 많이 허용될 수 있음
- sliding window: 더 정교하지만 계산과 저장 비용이 커질 수 있음
- leaky bucket: 출력 속도를 일정하게 만들기 좋지만 burst 허용이 상대적으로 제한적임
- token bucket: 적당한 burst 허용 + 지속 속도 제한을 함께 다루기 좋음

특히 사용자 경험이 중요한 API에서는 fixed window보다 token bucket이 더 자연스러운 경우가 많습니다.
딱 00초 기준으로 갑자기 허용량이 초기화되는 느낌보다, **조금씩 회복되는 한도를 제공하는 쪽이 현실 트래픽에 잘 맞기 때문**입니다.

## Node.js에서 Token Bucket을 어디에 적용하면 좋을까

### H3. 로그인, 인증, 검색, 공개 API 같은 남용 가능 엔드포인트가 대표적이다

모든 API에 똑같은 limiter를 거는 것은 좋은 전략이 아닙니다.
엔드포인트 성격에 따라 보호 수준이 달라야 합니다.
보통 아래 같은 곳에서 token bucket이 특히 유용합니다.

- 로그인, 비밀번호 재설정, OTP 검증 API
- 검색, 자동완성, 추천 조회 API
- 공개형 REST API나 파트너 API
- 비용이 큰 파일 변환, 리포트 생성, AI 추론 호출 API
- 외부 시스템으로 fan-out되는 webhook 트리거 API

이 중 일부는 보안 성격이 강하고, 일부는 비용 보호 성격이 강합니다.
하지만 공통점은 같습니다.
**짧은 burst는 있을 수 있지만, 지속적 과속은 서비스 전체 품질을 해친다**는 점입니다.

### H3. 사용자별, IP별, API key별로 키 공간을 나눠야 효과가 있다

rate limiter는 어디에 거는가만큼 **무엇을 기준으로 거는가**도 중요합니다.
한 서비스 전체에 하나의 전역 limiter만 두면 특정 사용자의 과속이 다른 사용자까지 같이 막을 수 있습니다.
그래서 실무에서는 보통 아래 단위를 조합합니다.

- IP 기준 limiter
- userId 기준 limiter
- API key 기준 limiter
- route 기준 limiter
- tenant 기준 limiter

예를 들어 로그인 API는 IP와 계정 기준을 함께 볼 수 있고, 파트너 API는 API key 기준이 더 적절할 수 있습니다.
멀티테넌트 SaaS라면 tenant별 버킷도 중요합니다.
핵심은 **누가 얼마나 어떤 자원을 소비하는가를 서비스 구조에 맞게 식별하는 것**입니다.

## Node.js Token Bucket Rate Limiter 구현 방식

### H3. 단일 인스턴스에서는 메모리 기반으로 시작할 수 있지만, 다중 인스턴스에서는 공유 저장소가 필요하다

개발 초기나 단일 프로세스 환경에서는 메모리 기반 limiter로 빠르게 실험할 수 있습니다.
다만 production에서 인스턴스가 여러 대라면 각 서버가 따로 카운트를 가지게 되어 제한이 쉽게 우회됩니다.
그래서 실제 운영에서는 보통 Redis 같은 공유 저장소를 둡니다.

개념을 단순화한 메모리 기반 예시는 아래처럼 볼 수 있습니다.

```js
const buckets = new Map();

function allowRequest(key, capacity, refillPerSec) {
  const now = Date.now();
  const current = buckets.get(key) ?? {
    tokens: capacity,
    lastRefillMs: now
  };

  const elapsedSec = (now - current.lastRefillMs) / 1000;
  const refilledTokens = elapsedSec * refillPerSec;

  current.tokens = Math.min(capacity, current.tokens + refilledTokens);
  current.lastRefillMs = now;

  if (current.tokens < 1) {
    buckets.set(key, current);
    return false;
  }

  current.tokens -= 1;
  buckets.set(key, current);
  return true;
}
```

이 코드는 원리를 설명하기에는 좋지만 production용으로는 부족합니다.
프로세스 재시작 시 상태가 날아가고, 멀티 인스턴스 환경에서는 정확하지 않으며, 동시성 경쟁도 고려되지 않습니다.
실무에서는 여기서 멈추지 않고 공유 저장소와 원자적 업데이트가 필요합니다.

### H3. Redis에서는 토큰 계산과 차감을 가능한 한 원자적으로 처리해야 한다

Node.js API 서버가 여러 대라면 Redis 기반 limiter가 가장 흔한 선택지입니다.
중요한 것은 토큰 보충과 소비를 분리된 여러 명령으로 처리하지 않는 것입니다.
그 사이에 경쟁 조건이 생기면 실제보다 বেশি 허용되거나, 반대로 과도하게 차단될 수 있습니다.

실무에서는 보통 아래 중 하나를 택합니다.

- Lua script로 refill + consume를 한 번에 처리
- 잘 검증된 rate limiting 라이브러리 사용
- Redis cell 같은 모듈 기반 접근 사용

개념적으로는 아래 흐름을 구현하게 됩니다.

1. 현재 시각과 마지막 보충 시각을 읽음
2. 경과 시간만큼 토큰을 계산해 버킷을 보충함
3. 보충 후 토큰이 1개 이상이면 차감 후 허용
4. 부족하면 거절하고 남은 대기 시간을 계산함

이렇게 해야 응답 헤더도 더 친절하게 줄 수 있습니다.
예를 들면 아래 같은 정보입니다.

- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`
- `Retry-After`

좋은 limiter는 단순히 막는 데서 끝나지 않습니다.
**언제 다시 시도할 수 있는지까지 예측 가능하게 알려주는 편이 클라이언트 품질에도 좋습니다**.

### H3. Express 미들웨어로 감싸되, 실패 응답도 일관되게 설계하는 편이 좋다

Node.js에서 Express를 쓴다면 limiter를 미들웨어 계층에 두는 것이 가장 단순합니다.
핵심은 엔드포인트마다 정책을 달리하면서도, 응답 형식은 일관되게 유지하는 것입니다.

```js
function tokenBucketMiddleware({ capacity, refillPerSec, keyResolver }) {
  return async (req, res, next) => {
    const key = keyResolver(req);
    const result = await consumeToken({ key, capacity, refillPerSec });

    res.setHeader('X-RateLimit-Limit', String(capacity));
    res.setHeader('X-RateLimit-Remaining', String(result.remaining));

    if (!result.allowed) {
      res.setHeader('Retry-After', String(result.retryAfterSec));
      return res.status(429).json({
        message: 'Too many requests',
        retryAfterSec: result.retryAfterSec
      });
    }

    next();
  };
}
```

여기서 중요한 것은 429를 너무 공격적으로 남발하지 않는 것입니다.
정말 보호가 필요한 경로에는 강하게 적용하되, 일반 읽기 API에는 burst 허용량을 좀 더 넉넉히 잡는 편이 사용자 경험에 유리합니다.
Limiter는 보안을 위한 철문이기도 하지만, 동시에 **정상 사용자를 괴롭히지 않는 완충 장치**여야 합니다.

## Backpressure, admission control과는 어떻게 연결될까

### H3. token bucket은 입구에서 속도를 제어하고, backpressure는 시스템 전체 압력을 낮춘다

token bucket은 요청이 들어오는 순간 허용 여부를 판단합니다.
하지만 그것만으로 시스템 전체 부하 제어가 끝나는 것은 아닙니다.
내부 큐, worker, downstream API, DB가 각각 다른 병목을 가질 수 있기 때문입니다.

이 지점은 [Node.js Backpressure 가이드](/development/blog/seo/2026/03/30/nodejs-backpressure-overload-control-guide.html)와 연결됩니다.
간단히 정리하면 역할은 이렇습니다.

- token bucket: 지금 이 요청을 받아도 되는가
- backpressure: 지금 생산 속도를 얼마나 늦춰야 하는가
- queue priority: 이미 받은 작업 중 무엇을 먼저 처리할 것인가

즉 token bucket은 첫 번째 방어선에 가깝습니다.
입구에서 너무 많은 요청을 그대로 받으면, 뒤쪽에서 아무리 잘 조절해도 회복이 늦어질 수 있습니다.

### H3. admission control보다 더 미세하게 트래픽을 다루는 도구로 볼 수 있다

admission control은 시스템 상태가 나쁠 때 요청을 더 넓게 제한하는 전략입니다.
반면 token bucket은 특정 사용자, 키, 라우트 단위로 훨씬 세밀하게 속도를 다룰 수 있습니다.

둘을 함께 쓰면 아래 같은 운영이 가능합니다.

- 평시: token bucket으로 사용자별 과속만 제한
- 과부하 조짐: 특정 고비용 엔드포인트의 버킷 크기 축소
- 심각한 장애: admission control로 신규 요청 자체를 더 보수적으로 수락

즉 token bucket은 평소 운영용 정밀 제어 장치이고, admission control은 **상황이 나빠졌을 때 더 넓게 브레이크를 밟는 전략**에 가깝습니다.
둘은 대체재보다 상호보완재로 보는 편이 맞습니다.

## 운영에서 꼭 봐야 할 지표

### H3. 차단 비율만 보지 말고, 누구를 얼마나 막고 있는지 봐야 한다

rate limiter를 도입하면 많은 팀이 먼저 429 응답 건수만 봅니다.
물론 기본 지표입니다.
하지만 그것만으로는 limiter가 잘 작동하는지 판단하기 어렵습니다.
실무에서는 아래 지표를 함께 보는 편이 좋습니다.

- route별 허용률과 차단률
- userId, API key, IP별 상위 차단 대상
- limiter 적용 후 응답 시간 변화
- downstream 오류율과의 상관관계
- 버킷 고갈 빈도와 시간대 패턴
- `Retry-After` 이후 재시도 성공률

이렇게 봐야 단순히 "많이 막고 있다"가 아니라, **정말 문제성 트래픽을 줄였는지, 정상 사용자까지 과하게 막고 있는지**를 판단할 수 있습니다.

### H3. rate limit 정책은 한 번 정하면 끝이 아니라 계속 조정해야 한다

초기 설정은 거의 항상 거칠 수밖에 없습니다.
특히 새 서비스나 트래픽 패턴이 자주 바뀌는 제품에서는 더 그렇습니다.
그래서 아래 같은 질문을 주기적으로 점검하는 편이 좋습니다.

- burst 크기를 너무 작게 잡아 정상 UX를 해치고 있지 않은가?
- refill 속도를 너무 크게 잡아 보호 효과가 약하지 않은가?
- 특정 파트너나 테넌트의 사용 패턴과 정책이 맞지 않는가?
- 429 이후 클라이언트가 올바르게 backoff하고 있는가?

좋은 limiter는 하드코딩된 숫자가 아니라, **트래픽과 사용자 행동을 보고 계속 다듬는 운영 정책**입니다.

## Node.js Token Bucket 적용 체크리스트

### H3. 아래 질문에 답할 수 있으면 production 적용 준비가 꽤 된 상태다

다음 질문에 명확히 답할 수 있으면 token bucket limiter를 운영에 올려도 혼란이 적습니다.

- 어떤 엔드포인트가 남용 방지보다 가용성 보호 측면에서 중요한가?
- limiter 기준 키는 IP, userId, API key, tenant 중 무엇이 적절한가?
- 허용할 burst와 지속 가능한 평균 속도는 각각 얼마인가?
- 멀티 인스턴스 환경에서 상태를 어떻게 공유하고 원자적으로 갱신할 것인가?
- 429 응답 시 클라이언트가 backoff하도록 문서화돼 있는가?
- 과부하 시 admission control이나 brownout과 어떻게 연결할 것인가?

token bucket의 목적은 요청을 기계적으로 거절하는 데 있지 않습니다.
**짧은 급증은 흡수하고, 지속적인 과속은 제어해 시스템 전체 품질을 지키는 것**이 핵심입니다.

## 자주 묻는 질문

### H3. token bucket이 있으면 DDoS 방어까지 충분한가요?

보통은 아닙니다.
application 레벨 rate limiting은 중요한 보호 장치지만, 대규모 네트워크 공격까지 혼자 막는 용도는 아닙니다.
CDN, WAF, edge rate limiting과 함께 계층적으로 구성하는 편이 안전합니다.

### H3. 로그인 API는 얼마나 빡빡하게 제한해야 하나요?

정답은 서비스 특성에 따라 다릅니다.
다만 일반적으로는 사용자 경험을 해치지 않을 정도의 작은 burst를 허용하되, 계정 추측이나 무차별 대입이 어려울 정도로 지속 속도는 낮게 잡는 편이 좋습니다.
IP 기준과 계정 기준을 함께 보는 것도 흔한 방법입니다.

### H3. 429를 많이 줄수록 시스템이 더 안전한가요?

반드시 그렇지는 않습니다.
정상 사용자까지 자주 막으면 오히려 재시도와 문의가 늘고 UX가 악화될 수 있습니다.
중요한 것은 많이 막는 것이 아니라, **막아야 할 트래픽을 정확히 제한하는 것**입니다.

## 마무리

Node.js 서비스에서 token bucket rate limiter는 단순한 요청 차단기가 아닙니다.
시스템이 감당할 수 있는 속도와 사용자가 기대하는 반응성을 절충하는 운영 장치에 가깝습니다.

트래픽은 늘 매끈하게 오지 않습니다.
짧은 burst는 정상 사용에서도 자연스럽게 발생하고, 잘못된 클라이언트나 봇은 그 틈을 훨씬 더 공격적으로 파고듭니다.
이때 token bucket을 잘 설계해 두면 정상적인 순간 급증은 흡수하고, 지속적인 과속만 부드럽게 제어할 수 있습니다.
특히 Node.js처럼 이벤트 루프와 downstream 의존성의 건강이 전체 체감 성능에 크게 영향을 주는 환경에서는, 입구에서 속도를 다루는 전략이 생각보다 훨씬 중요합니다.

결국 중요한 질문은 이것입니다.
**이 요청을 막을 것인가가 아니라, 어떤 속도까지는 서비스가 건강하게 받아낼 수 있는가**.
그 기준을 현실적으로 코드에 옮긴 것이 token bucket의 진짜 가치입니다.

---
layout: post
title: "Node.js Idempotency Key 가이드: 중복 요청과 중복 결제를 안전하게 막는 API 설계법"
date: 2026-03-27 20:00:00 +0900
lang: ko
translation_key: nodejs-idempotency-key-api-duplicate-request-guide
permalink: /development/blog/seo/2026/03/27/nodejs-idempotency-key-api-duplicate-request-guide.html
alternates:
  ko: /development/blog/seo/2026/03/27/nodejs-idempotency-key-api-duplicate-request-guide.html
  x_default: /development/blog/seo/2026/03/27/nodejs-idempotency-key-api-duplicate-request-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, idempotency, api, payment, backend, reliability]
description: "Node.js API에서 idempotency key를 적용해 중복 요청, 중복 결제, 재시도 충돌을 줄이는 실무 설계 방법을 예시와 함께 정리했습니다."
---

결제 API나 주문 생성 API를 운영하다 보면 "사용자가 버튼을 두 번 눌렀다", "네트워크가 끊겨서 클라이언트가 재시도했다", "프록시가 timeout 뒤 재전송했다" 같은 상황이 생각보다 자주 발생합니다.
이때 서버가 요청을 매번 새로운 작업으로 처리하면 **중복 결제, 중복 주문, 중복 데이터 생성** 같은 문제가 생깁니다.
이 문제를 줄이는 대표적인 방법이 **idempotency key**입니다.
같은 의도를 가진 요청에는 같은 키를 붙이고, 서버는 그 키를 기준으로 "이 요청은 이미 처리했는가"를 판단합니다.
이 글에서는 Node.js API에서 idempotency key가 왜 필요한지, 어떤 저장 전략을 써야 하는지, 결제·주문 같은 쓰기 API에 어떻게 적용하면 좋은지 실무 관점에서 정리합니다.

## Node.js idempotency key는 왜 필요한가

### H3. 재시도는 복원력을 높이지만 쓰기 API에서는 중복 작업 위험도 같이 키운다

네트워크가 불안정한 환경에서는 retry가 필요합니다.
문제는 읽기 요청과 달리 쓰기 요청은 같은 요청을 두 번 보내면 진짜로 결과가 두 번 생길 수 있다는 점입니다.
예를 들어 결제 승인 요청이 서버에서는 성공했는데, 응답이 클라이언트까지 도달하기 전에 끊겼다고 가정해보겠습니다.
클라이언트는 "실패한 것 같다"고 판단해 다시 요청을 보낼 수 있습니다.
서버가 이를 새로운 요청으로 받아들이면 결제가 두 번 승인될 수 있습니다.

즉 retry 자체가 나쁜 것은 아닙니다.
다만 쓰기 API에서는 retry를 안전하게 받기 위한 장치가 필요합니다.
그 역할을 하는 것이 idempotency key입니다.

### H3. 중복 클릭보다 더 무서운 것은 중간 시스템의 자동 재전송이다

실무에서는 사용자의 더블 클릭보다, 그 사이에 있는 시스템이 자동으로 재시도하는 경우가 더 까다롭습니다.
예를 들면 아래처럼 여러 층에서 중복 요청이 생길 수 있습니다.

- 모바일 앱이 응답 지연으로 재시도
- API gateway가 upstream timeout 후 재전송
- worker가 작업 상태를 모른 채 다시 enqueue
- 외부 결제 연동 과정에서 webhook을 중복 전달

이 상황에서 애플리케이션이 "같은 의도의 요청"을 식별하지 못하면 데이터 정합성이 무너집니다.
특히 돈, 재고, 쿠폰, 포인트처럼 한 번만 반영돼야 하는 도메인에서는 작은 중복이 바로 운영 사고로 이어집니다.

## idempotency key는 어떻게 동작하나

### H3. 같은 키면 같은 결과를 돌려주는 방식으로 보는 것이 이해가 쉽다

핵심 아이디어는 단순합니다.
클라이언트가 같은 작업 의도에 대해 동일한 idempotency key를 보냅니다.
서버는 이 키를 저장하고, 이미 처리된 키라면 새 작업을 다시 실행하지 않고 **기존 결과를 재사용**합니다.

예를 들어 아래 같은 흐름입니다.

1. 클라이언트가 `Idempotency-Key: pay_20260327_abc123` 헤더와 함께 결제 요청 전송
2. 서버가 해당 키가 처음인지 확인
3. 처음이면 실제 결제 처리 후 결과 저장
4. 같은 키로 다시 요청이 오면 결제를 다시 수행하지 않고 저장된 응답 반환

이 구조의 장점은 "중복을 감지했다"에서 끝나지 않는다는 점입니다.
클라이언트 입장에서는 같은 요청에 대해 같은 응답을 받으므로 재시도 로직을 훨씬 단순하게 만들 수 있습니다.

### H3. 키만 같다고 무조건 같은 요청으로 취급하면 안 된다

많이 놓치는 부분이 있습니다.
키가 같더라도 요청 본문이 다르면 위험합니다.
예를 들어 같은 idempotency key를 사용했는데 첫 번째 요청은 10,000원 결제이고 두 번째 요청은 100,000원 결제라면, 서버는 이를 동일한 요청으로 보면 안 됩니다.

그래서 보통 아래 둘 중 하나를 같이 저장합니다.

- 요청 payload 전체의 해시값
- 비교가 필요한 핵심 필드의 canonical hash

그리고 같은 키로 재요청이 들어왔을 때 payload hash가 다르면 409 Conflict 같은 명확한 오류를 반환합니다.
이 검증이 없으면 클라이언트 버그나 악의적 요청이 이상한 정합성 문제를 만들 수 있습니다.

## Node.js API에서 어디에 적용해야 할까

### H3. 결제, 주문 생성, 쿠폰 사용처럼 중복 실행 비용이 큰 쓰기 작업이 우선순위다

모든 POST 요청에 일괄 적용할 필요는 없습니다.
실제로는 중복 실행 시 피해가 큰 경로부터 붙이는 편이 효율적입니다.
대표적인 예시는 아래와 같습니다.

- 결제 승인, 환불 요청
- 주문 생성, 예약 확정
- 포인트 차감, 쿠폰 사용
- 계정 생성 후 외부 연동이 이어지는 작업
- 파일 업로드 후 과금이나 후처리가 붙는 작업

반대로 단순 로그 적재처럼 중복이 큰 문제가 아니거나, 애초에 서버가 natural key로 중복을 막는 구조라면 우선순위가 낮을 수 있습니다.
핵심은 **같은 작업이 두 번 실행되면 운영 비용이 커지는 쓰기 경로**를 먼저 찾는 것입니다.

### H3. 읽기 API보다 비동기 작업 시작 API에서 더 중요해지는 경우가 많다

동기 요청보다 비동기 작업 시작 API에서 idempotency key가 더 중요해지는 경우도 많습니다.
예를 들어 "보고서 생성 시작", "영상 인코딩 시작", "정산 배치 실행" 같은 요청은 즉시 결과가 나오지 않기 때문에 클라이언트가 재시도하기 쉽습니다.
이때 서버가 job enqueue를 매번 새로 해버리면 같은 작업이 여러 번 돌아갈 수 있습니다.

즉 idempotency key는 단순히 HTTP 응답 재사용만이 아니라 **중복 side effect 방지 장치**로 보는 편이 맞습니다.

## Node.js에서 idempotency key를 어떻게 구현할까

### H3. Redis 같은 빠른 저장소로 key 상태와 응답 메타를 관리하면 운영이 편하다

실무에서는 Redis를 많이 사용합니다.
속도가 빠르고 TTL 관리가 쉬워서 idempotency key 저장소로 잘 맞기 때문입니다.
보통은 아래 정보를 함께 저장합니다.

- idempotency key
- 요청 hash
- 현재 상태(`processing`, `completed`, `failed`)
- 응답 body 또는 응답 참조값
- HTTP status code
- 생성 시각과 만료 시각

개념 예시는 아래처럼 잡을 수 있습니다.

```ts
interface IdempotencyRecord {
  requestHash: string;
  status: 'processing' | 'completed' | 'failed';
  responseStatus?: number;
  responseBody?: unknown;
  createdAt: number;
  expiresAt: number;
}
```

첫 요청이 들어오면 `processing`으로 먼저 잠그고, 실제 작업이 끝나면 `completed`와 응답 값을 저장합니다.
같은 키가 다시 들어오면 저장된 상태를 보고 재처리 여부를 결정합니다.

### H3. processing 상태를 먼저 기록해야 동시 요청 경합을 줄일 수 있다

가장 위험한 순간은 거의 동시에 같은 요청이 두 번 들어오는 경우입니다.
둘 다 "아직 키가 없네"라고 보고 동시에 결제를 시작하면 idempotency key가 있어도 의미가 없어집니다.
그래서 첫 단계에서 원자적으로 lock 비슷한 상태를 선점해야 합니다.

Redis를 쓴다면 `SET key value NX EX 300` 같은 패턴으로 선점할 수 있습니다.
대략적인 흐름은 아래와 같습니다.

```ts
import crypto from 'node:crypto';
import express from 'express';

function hashPayload(payload: unknown) {
  return crypto.createHash('sha256').update(JSON.stringify(payload)).digest('hex');
}

app.post('/payments', async (req, res) => {
  const idempotencyKey = req.header('Idempotency-Key');
  if (!idempotencyKey) {
    return res.status(400).json({ message: 'Idempotency-Key header is required' });
  }

  const requestHash = hashPayload({
    userId: req.user.id,
    amount: req.body.amount,
    orderId: req.body.orderId,
  });

  const redisKey = `idem:payments:${idempotencyKey}`;
  const current = await redis.get(redisKey);

  if (current) {
    const record = JSON.parse(current);

    if (record.requestHash !== requestHash) {
      return res.status(409).json({ message: 'Idempotency key reused with different payload' });
    }

    if (record.status === 'completed') {
      return res.status(record.responseStatus).json(record.responseBody);
    }

    if (record.status === 'processing') {
      return res.status(409).json({ message: 'Request is already being processed' });
    }
  }

  const locked = await redis.set(
    redisKey,
    JSON.stringify({
      requestHash,
      status: 'processing',
      createdAt: Date.now(),
      expiresAt: Date.now() + 1000 * 60 * 10,
    }),
    'NX',
    'EX',
    60 * 10,
  );

  if (locked !== 'OK') {
    return res.status(409).json({ message: 'Duplicate request in progress' });
  }

  try {
    const payment = await paymentService.approve({
      userId: req.user.id,
      amount: req.body.amount,
      orderId: req.body.orderId,
    });

    const responseBody = { paymentId: payment.id, status: 'approved' };

    await redis.set(
      redisKey,
      JSON.stringify({
        requestHash,
        status: 'completed',
        responseStatus: 201,
        responseBody,
        createdAt: Date.now(),
        expiresAt: Date.now() + 1000 * 60 * 10,
      }),
      'EX',
      60 * 10,
    );

    return res.status(201).json(responseBody);
  } catch (error) {
    await redis.del(redisKey);
    throw error;
  }
});
```

실서비스에서는 응답 body를 그대로 저장할지, DB의 결과 ID만 저장할지, 실패 응답도 재사용할지 등을 더 세밀하게 정해야 합니다.
하지만 큰 흐름은 "선점 → 처리 → 결과 저장 → 재요청 시 재사용"입니다.

## 저장 전략은 어떻게 고르는 것이 좋을까

### H3. 응답 전체 저장 방식은 단순하고, 리소스 참조 저장 방식은 더 가볍다

idempotency key 저장 방식은 크게 두 가지로 나눌 수 있습니다.

첫째는 **응답 전체 저장**입니다.
처리 결과를 그대로 저장해두고, 재요청이 오면 같은 응답을 바로 반환합니다.
구현이 단순하고 클라이언트 경험도 일관적이라는 장점이 있습니다.

둘째는 **결과 리소스 참조 저장**입니다.
예를 들어 `paymentId`, `orderId`만 저장해두고, 재요청 시 DB에서 해당 리소스를 다시 조회해 응답을 구성합니다.
저장 공간은 아끼기 좋지만 조회 로직이 한 단계 더 필요합니다.

선택 기준은 대체로 아래와 같습니다.

- 응답이 작고 그대로 재사용 가능하면 응답 전체 저장
- 응답이 크거나 민감하면 참조값만 저장
- 결과가 계속 변할 수 있으면 참조 저장 후 현재 상태 재조회

### H3. TTL은 비즈니스 리스크와 재시도 패턴을 같이 보고 정해야 한다

TTL을 너무 짧게 잡으면 느린 재시도를 방어하지 못하고, 너무 길게 잡으면 저장소가 불필요하게 커집니다.
그래서 보통 도메인 성격에 따라 다르게 가져갑니다.

예를 들어 아래처럼 볼 수 있습니다.

- 결제 승인: 수십 분~24시간
- 주문 생성: 수십 분~몇 시간
- 짧은 작업 enqueue: 몇 분~수십 분

중요한 것은 "사용자가 언제까지 같은 요청을 다시 보낼 수 있는가"와 "중복 실행 시 피해가 얼마나 큰가"를 함께 보는 것입니다.
운영 로그를 보면 실제 재시도 분포가 생각보다 길게 나오는 경우도 많습니다.

## idempotency key를 쓸 때 자주 하는 실수

### H3. DB unique key가 있으니 충분하다고 생각하는 경우

DB unique 제약은 매우 중요합니다.
하지만 그것만으로 충분하지 않은 경우가 많습니다.
unique key는 중복 저장을 막을 수는 있어도, 클라이언트에 어떤 응답을 재현할지까지 해결해주지는 않습니다.
또한 외부 결제 호출처럼 DB 바깥 side effect가 먼저 일어나는 경로에서는 unique key만으로 사고를 막기 어렵습니다.

가장 안전한 구조는 보통 아래 조합입니다.

- API 레이어의 idempotency key
- 도메인 레벨 natural key 또는 unique constraint
- 외부 연동 레벨의 별도 중복 방지 정책

즉 한 겹만 믿기보다, 중복이 생길 수 있는 층마다 방어막을 두는 편이 낫습니다.

### H3. 실패 요청을 무조건 삭제하면 오히려 중복 실행 위험이 생길 수 있다

위 예시 코드에서는 단순화를 위해 에러 발생 시 키를 삭제했습니다.
하지만 실무에서는 이 정책을 조심해서 써야 합니다.
예를 들어 서버는 외부 결제 승인까지 끝냈는데, 그 뒤 응답 저장 단계에서만 실패했다면 키를 지우는 순간 재시도 시 결제가 다시 나갈 수 있습니다.

그래서 실패를 아래처럼 나눠보는 편이 좋습니다.

- 작업 시작 전 실패: 삭제 가능
- 외부 side effect 발생 전 실패: 재시도 가능
- side effect 발생 여부가 불명확한 실패: 삭제보다 조사 가능한 상태로 보존
- 명확히 완료됐지만 응답 저장만 실패: 결과 복구 로직 필요

이 구간을 대충 처리하면 idempotency key를 도입하고도 가장 중요한 경계에서 무너질 수 있습니다.

## 관련 패턴과 함께 보면 더 좋은 이유

### H3. retry, circuit breaker, webhook 검증과 함께 봐야 운영 품질이 올라간다

idempotency key는 혼자만으로 완결되는 패턴이 아닙니다.
재시도는 계속 필요하고, 장애 전파를 막기 위한 회로 차단도 필요하며, 외부 시스템이 같은 이벤트를 두 번 보내는 문제도 따로 다뤄야 합니다.

아래 글과 함께 보면 설계가 더 입체적으로 잡힙니다.

- <a href="/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html">Node.js Exponential Backoff + Jitter 가이드</a>
- <a href="/development/blog/seo/2026/03/23/nodejs-webhook-signature-verification-security-guide.html">Node.js Webhook Signature Verification 가이드</a>
- <a href="/development/blog/seo/2026/03/27/nodejs-circuit-breaker-failure-isolation-guide.html">Node.js Circuit Breaker 패턴 가이드</a>

retry는 일시적 실패 복구를 돕고, idempotency key는 재시도로 인한 중복 부작용을 줄이며, circuit breaker는 구조적 장애 확산을 막습니다.
셋을 역할별로 구분해 설계하면 훨씬 안정적입니다.

## 실무 체크리스트

### H3. 도입 전에 이 항목부터 정리하면 시행착오를 줄일 수 있다

- 어떤 쓰기 API가 중복 실행 시 가장 위험한가
- idempotency key는 헤더로 받을지 body 필드로 받을지 정했는가
- 동일 요청 판별용 request hash 규칙이 있는가
- 처리 중(`processing`) 상태를 원자적으로 선점하는가
- 완료 응답을 저장할지, 결과 ID만 저장할지 정했는가
- TTL이 실제 재시도 패턴과 비즈니스 위험에 맞는가
- side effect 발생 여부가 불명확한 실패를 조사 가능한 상태로 남기는가
- DB unique key와 역할 분담이 명확한가

## FAQ

### H3. 모든 POST API에 idempotency key를 붙여야 할까

그럴 필요는 없습니다.
중복 실행 피해가 크고, 클라이언트 재시도가 자주 발생하며, side effect가 무거운 경로부터 적용하는 편이 현실적입니다.
대표적으로 결제, 주문 생성, 예약 확정, 포인트 차감 같은 API가 우선순위가 높습니다.

### H3. 클라이언트가 키를 만들지, 서버가 만들어야 할까

대개는 클라이언트가 같은 의도에 대해 같은 키를 재사용할 수 있어야 하므로 클라이언트 생성 방식이 자연스럽습니다.
다만 서버가 세션 기반으로 중복 방지 토큰을 발급하는 구조도 가능합니다.
중요한 것은 "같은 작업 의도에는 같은 식별자"가 유지되도록 만드는 것입니다.

## 마무리

Node.js idempotency key는 단순한 중복 방지 옵션이 아닙니다.
**재시도가 당연한 네트워크 환경에서 쓰기 API를 안전하게 만들기 위한 기본 장치**에 가깝습니다.

특히 결제, 주문, 포인트처럼 한 번만 처리돼야 하는 도메인에서는 retry를 막는 것이 아니라 **retry를 받아도 결과가 망가지지 않게 설계하는 것**이 핵심입니다.
Node.js API를 운영 중인데 아직 idempotency key 기준이 없다면, 가장 사고 비용이 큰 쓰기 경로부터 먼저 도입하는 편이 확실히 낫습니다.

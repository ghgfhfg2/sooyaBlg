---
layout: post
title: "Node.js 웹훅 보안 가이드: Signature 검증으로 위조 요청과 중복 처리 리스크 줄이기"
date: 2026-03-23 08:00:00 +0900
lang: ko
translation_key: nodejs-webhook-signature-verification-security-guide
permalink: /development/blog/seo/2026/03/23/nodejs-webhook-signature-verification-security-guide.html
alternates:
  ko: /development/blog/seo/2026/03/23/nodejs-webhook-signature-verification-security-guide.html
  x_default: /development/blog/seo/2026/03/23/nodejs-webhook-signature-verification-security-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, webhook, signature, hmac, security, backend]
description: "Node.js에서 웹훅을 받을 때 HMAC signature 검증, raw body 보존, timestamp 허용 범위, 중복 이벤트 방지까지 함께 설계해 위조 요청과 운영 리스크를 줄이는 방법을 정리했습니다."
---

외부 결제사나 SaaS에서 웹훅을 붙일 때 가장 흔한 실수는 "일단 이벤트가 오면 처리부터 하자"로 시작하는 것입니다.
하지만 웹훅 엔드포인트는 공개 URL인 경우가 많고, 여기서 **signature 검증**을 빼먹으면 위조 요청·중복 처리·운영 혼선을 한 번에 떠안게 됩니다.
이 글에서는 Node.js에서 **웹훅 signature 검증**을 어떻게 설계해야 하는지, 그리고 raw body 보존·timestamp 검증·중복 이벤트 방지까지 실무 기준으로 정리합니다.

## 왜 웹훅 보안은 "헤더 확인" 정도로 끝내면 안 될까

### H3. 공개 URL이라는 점 자체가 공격 표면이 된다

웹훅은 보통 외부 서비스가 우리 서버로 요청을 보내는 구조입니다.
즉 클라이언트 앱처럼 로그인 세션을 전제로 하지 않고, 인터넷에서 접근 가능한 엔드포인트로 열리는 경우가 많습니다.
이때 단순히 `User-Agent` 나 특정 헤더 존재 여부만 보는 방식은 거의 보안 장치라고 보기 어렵습니다.

문제가 되는 패턴은 대개 이렇습니다.

- 요청이 특정 공급자에서 왔다고 "추정"만 하고 처리한다
- JSON body 내용이 그럴듯하면 진짜 이벤트라고 믿는다
- 테스트 단계에서 편하려고 검증을 생략한 채 운영까지 간다
- 재전송 이벤트와 위조 이벤트를 구분하지 못한다

결국 웹훅 수신부는 "외부가 호출하는 내부 API"에 가깝기 때문에, **요청 출처를 재현 가능한 방식으로 검증하는 절차**가 필수입니다.

### H3. body parser 설정 하나로도 검증이 깨질 수 있다

웹훅 signature 검증은 흔히 HMAC 기반으로 동작합니다.
공급자가 보낸 **원본 요청 바디(raw body)** 와 공유 비밀값(secret)으로 서명을 계산하고, 서버가 같은 방식으로 재계산해 비교합니다.

문제는 Express나 Fastify에서 JSON 파싱이 먼저 일어나면 body 문자열이 바뀔 수 있다는 점입니다.
공백, 줄바꿈, 인코딩, 키 순서 같은 차이만 생겨도 계산 결과가 달라질 수 있습니다.
그래서 웹훅 검증은 단순히 암호화 지식보다 **미들웨어 순서와 raw body 보존**이 더 중요할 때가 많습니다.

## Node.js에서 웹훅 signature 검증을 설계하는 기본 원칙

### H3. raw body를 먼저 확보하고 그 값으로 HMAC을 계산한다

가장 먼저 지켜야 할 원칙은 "파싱된 JSON"이 아니라 **원본 바이트 기준**으로 검증해야 한다는 점입니다.
공급자 문서가 `sha256=` 접두사를 붙이는지, 어떤 헤더 이름을 쓰는지, timestamp를 같이 포함하는지 먼저 확인해야 합니다.

Express에서는 특정 라우트에 raw body를 따로 받는 구성이 흔합니다.
예를 들면 아래처럼 갈 수 있습니다.

```ts
import express from 'express';

const app = express();

app.post(
  '/webhooks/provider',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const rawBody = req.body as Buffer;
    const signature = req.header('x-provider-signature');

    // 1) rawBody 기준 HMAC 계산
    // 2) 헤더 signature와 상수 시간 비교
    // 3) 검증 통과 후에만 JSON.parse 수행

    res.sendStatus(200);
  }
);
```

핵심은 간단합니다.

- 검증 전까지는 raw body를 유지한다
- 검증 성공 후에만 JSON으로 파싱한다
- 공급자 문서의 서명 포맷을 정확히 따른다

이 순서가 어긋나면 코드가 멀쩡해 보여도 운영에서 정상 이벤트를 계속 실패 처리할 수 있습니다.

### H3. `===` 대신 상수 시간 비교를 사용한다

서명 문자열 비교를 일반 문자열 비교로 끝내는 경우가 아직 많습니다.
하지만 민감한 비교에는 `crypto.timingSafeEqual` 같은 **상수 시간 비교**를 쓰는 편이 안전합니다.

예시는 아래처럼 단순화할 수 있습니다.

```ts
import crypto from 'node:crypto';

function verifySignature({ rawBody, signature, secret }: {
  rawBody: Buffer;
  signature: string;
  secret: string;
}) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');

  const received = signature.replace(/^sha256=/, '');

  const expectedBuffer = Buffer.from(expected, 'hex');
  const receivedBuffer = Buffer.from(received, 'hex');

  if (expectedBuffer.length !== receivedBuffer.length) {
    return false;
  }

  return crypto.timingSafeEqual(expectedBuffer, receivedBuffer);
}
```

실무에서는 공급자별로 아래 차이가 있습니다.

- HMAC 알고리즘이 `sha1`, `sha256`, `sha512` 중 무엇인지
- prefix가 붙는지
- timestamp와 body를 함께 서명하는지
- 여러 서명 버전을 한 헤더에 담는지

그래서 "HMAC이면 다 똑같다"보다 **공급자 명세를 그대로 구현한다**는 태도가 중요합니다.

## 재생 공격과 중복 이벤트는 어떻게 막을까

### H3. timestamp 허용 범위를 두고 너무 오래된 요청은 거절한다

signature가 맞는다고 끝이 아닙니다.
공격자가 과거의 정상 요청을 그대로 다시 보내는 **replay attack** 가능성도 봐야 합니다.
이를 줄이기 위해 많은 공급자가 timestamp 헤더를 함께 보내고, 서버는 허용 오차 범위를 둡니다.

예를 들어 아래 같은 기준을 둘 수 있습니다.

- 요청 timestamp가 현재 시각 기준 ±5분 이내인지 확인
- 범위를 벗어나면 400 또는 401로 거절
- 서버 시간이 크게 어긋나지 않도록 NTP 동기화 유지

이 체크가 있으면 캡처된 요청이 나중에 재전송되는 상황을 어느 정도 줄일 수 있습니다.
물론 운영 환경의 시간 동기화가 깨져 있으면 정상 요청까지 놓칠 수 있으니, 인프라 기본기와 같이 봐야 합니다.

### H3. 이벤트 ID 저장으로 중복 처리도 별도로 막아야 한다

웹훅 공급자는 네트워크 문제나 5xx 응답 때문에 같은 이벤트를 재전송할 수 있습니다.
이건 공격이 아니라 정상 동작일 수 있습니다.
그래서 signature 검증과 별개로 **중복 이벤트 방지** 계층이 필요합니다.

보통은 아래 식으로 설계합니다.

1. 이벤트 payload에서 provider event id를 추출한다
2. `provider + eventId` 조합을 저장소에 기록한다
3. 이미 처리한 이벤트면 비즈니스 로직을 다시 실행하지 않는다
4. 처리 상태를 `received`, `processing`, `done`, `failed` 등으로 관리한다

즉 웹훅 수신부는 두 질문에 모두 답해야 합니다.

- 이 요청이 진짜 공급자 요청인가?
- 이 이벤트를 이미 처리한 적이 있는가?

둘 중 하나만 해결하면 운영 사고는 계속 납니다.

## Express 운영에서 자주 놓치는 구현 포인트

### H3. 전역 `express.json()` 과 웹훅 라우트를 분리한다

가장 흔한 실수 중 하나는 앱 전체에 `express.json()` 을 먼저 걸어 둔 뒤, 웹훅 라우트에서도 검증이 될 거라고 생각하는 것입니다.
하지만 전역 JSON 파서가 먼저 body를 소비해 버리면 raw body 검증이 어려워질 수 있습니다.

실무에서는 보통 아래 둘 중 하나를 택합니다.

- 웹훅 라우트를 전역 JSON 파서보다 먼저 선언한다
- 웹훅 전용 raw parser를 별도 서브앱/라우터에 둔다

이때 중요한 건 코드 스타일보다 **검증이 실제로 재현되느냐**입니다.
운영 전에 공급자 테스트 이벤트와 로컬 재현 케이스로 꼭 확인해야 합니다.

### H3. 실패 응답 정책과 로그 필드를 명확히 나눈다

검증 실패가 발생했을 때도 응답과 로그 기준이 애매하면 운영이 힘들어집니다.
아래처럼 분리해 두면 좋습니다.

- signature 불일치: 인증 실패로 기록
- timestamp 만료: replay 가능성 또는 시간 오차로 기록
- 필수 헤더 누락: 잘못된 요청 포맷으로 기록
- 이벤트 중복: 경고가 아닌 정보성 로그로 기록
- 비즈니스 처리 실패: 검증 성공 이후 내부 처리 오류로 기록

로그에는 최소한 아래 필드를 남기는 편이 좋습니다.

- provider 이름
- event id
- delivery id 또는 request id
- 검증 결과
- 중복 여부
- 최종 처리 상태

단, secret 원문이나 전체 서명 문자열, 민감 payload는 그대로 남기지 않는 편이 안전합니다.

## Node.js 예시 흐름

### H3. 검증 성공 후 파싱, 중복 체크, 처리 순서로 분리한다

아래는 개념 설명용 단순 예시입니다.

```ts
app.post('/webhooks/provider', express.raw({ type: 'application/json' }), async (req, res) => {
  const rawBody = req.body as Buffer;
  const signature = req.header('x-provider-signature');
  const timestamp = req.header('x-provider-timestamp');

  if (!signature || !timestamp) {
    return res.status(400).json({ message: 'missing signature headers' });
  }

  const isFresh = isWithinTolerance(timestamp, 5 * 60);
  if (!isFresh) {
    return res.status(401).json({ message: 'stale webhook request' });
  }

  const isValid = verifyProviderSignature({
    rawBody,
    signature,
    timestamp,
    secret: process.env.PROVIDER_WEBHOOK_SECRET ?? '',
  });

  if (!isValid) {
    return res.status(401).json({ message: 'invalid signature' });
  }

  const event = JSON.parse(rawBody.toString('utf8'));
  const alreadyProcessed = await webhookEventStore.exists(event.id);

  if (alreadyProcessed) {
    return res.status(200).json({ ok: true, duplicate: true });
  }

  await webhookEventStore.markReceived(event.id);
  await handleWebhookEvent(event);
  await webhookEventStore.markDone(event.id);

  return res.status(200).json({ ok: true });
});
```

이 예시의 핵심은 복잡한 프레임워크가 아니라 순서입니다.

1. 필수 헤더 확인
2. timestamp 허용 범위 확인
3. raw body 기반 signature 검증
4. JSON 파싱
5. 이벤트 중복 체크
6. 비즈니스 로직 실행

순서가 뒤섞이면 보안 문제와 운영 문제를 동시에 부릅니다.

## 배포 전 체크리스트

### H3. 보안·구현 점검

- raw body가 실제 운영 경로에서 보존되는가
- 공급자 문서와 동일한 알고리즘·헤더 포맷·prefix를 쓰는가
- `timingSafeEqual` 등 상수 시간 비교를 적용했는가
- timestamp 허용 범위와 서버 시간 동기화 기준이 있는가
- 이벤트 ID 기반 중복 처리 방지 계층이 있는가

### H3. 콘텐츠·운영 점검

- 코드 예시에 실제 secret, 토큰, 내부 도메인이 포함되지 않았는가
- 실패 로그에 민감 payload를 그대로 남기지 않도록 했는가
- 2xx/4xx/5xx 응답 정책이 문서화되어 있는가
- 내부링크와 메타 설명, 태그가 일관되게 설정됐는가

## 요약

Node.js 웹훅 보안에서 중요한 것은 "헤더가 있나"가 아니라 **raw body 기준 signature 검증이 재현 가능하게 구현됐는가**입니다.
여기에 timestamp 허용 범위, 상수 시간 비교, 이벤트 ID 기반 중복 방지까지 함께 설계해야 위조 요청과 재전송 이벤트를 구분할 수 있습니다.
웹훅 수신부는 작은 엔드포인트처럼 보여도, 실제로는 외부와 맞닿은 보안 경계입니다.
그래서 편의보다 **검증 순서와 운영 계약**을 먼저 고정하는 편이 훨씬 안전합니다.

## 내부 링크

- [Node.js 재시도 전략 가이드: Exponential Backoff와 Jitter로 장애 증폭 막기](/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html)
- [Node.js OpenAPI 계약 검증 가이드: Zod와 Swagger로 API 불일치 줄이기](/development/blog/seo/2026/03/21/nodejs-openapi-zod-contract-validation-api-consistency-guide.html)
- [Node.js Kafka 중복 메시지 처리 가이드: Idempotent Consumer로 중복 소비 안전하게 막기](/development/blog/seo/2026/03/17/nodejs-idempotent-consumer-kafka-duplicate-message-handling-guide.html)

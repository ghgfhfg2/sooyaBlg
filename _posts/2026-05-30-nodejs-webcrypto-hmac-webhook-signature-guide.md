---
layout: post
title: "Node.js Web Crypto HMAC 가이드: 웹훅 서명 검증을 안전하게 구현하는 법"
date: 2026-05-30 20:00:00 +0900
lang: ko
translation_key: nodejs-webcrypto-hmac-webhook-signature-guide
permalink: /development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html
alternates:
  ko: /development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html
  x_default: /development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, webcrypto, hmac, webhook, security, backend]
description: "Node.js Web Crypto API로 HMAC 기반 웹훅 서명을 검증하는 방법을 정리했습니다. 원본 바디 보존, 타임스탬프 포함, timingSafeEqual 비교, 재현 가능한 테스트까지 예제로 설명합니다."
---

웹훅은 외부 서비스가 우리 서버로 이벤트를 밀어 넣는 단순한 통합 방식입니다.
결제 완료, 구독 변경, 배포 완료, 알림 수신처럼 실시간성이 필요한 흐름에서 자주 씁니다.

문제는 웹훅 엔드포인트가 인터넷에 열려 있다는 점입니다.
요청이 정말 신뢰한 서비스에서 온 것인지 확인하지 않으면, 누군가가 비슷한 JSON을 만들어 민감한 상태 변경을 유도할 수 있습니다.

이때 가장 기본이 되는 방어선이 **HMAC 서명 검증**입니다.
이 글에서는 Node.js Web Crypto API로 웹훅 서명을 검증하는 구조, 원본 바디 보존, 타임스탬프 검증, 안전한 비교 방식, 테스트 체크리스트를 정리합니다.

## HMAC 웹훅 서명이 필요한 이유

### H3. 웹훅 요청은 인증된 API 호출과 다르게 들어온다

일반적인 API 요청은 사용자 세션, OAuth 토큰, 내부 서비스 토큰처럼 인증 수단이 비교적 명확합니다.
반면 웹훅은 외부 서비스가 서버로 직접 보내는 요청이므로, 매번 사용자 인증 문맥이 붙어 있지 않은 경우가 많습니다.

그래서 웹훅 수신 서버는 최소한 아래를 확인해야 합니다.

- 요청 바디가 중간에 바뀌지 않았는가?
- 공유된 secret을 아는 발신자가 만든 요청인가?
- 오래된 요청을 누군가 다시 보내는 replay 공격은 아닌가?
- 검증 실패를 로그에 남기되 secret이나 원본 민감정보를 노출하지 않는가?

HMAC은 secret과 메시지를 함께 사용해 서명을 만듭니다.
수신 서버는 같은 secret과 같은 메시지로 서명을 다시 계산한 뒤, 헤더로 받은 서명과 비교합니다.

기존에 HTTP 요청 실패 처리를 정리하고 있다면 [HTTP request timeout and fail-fast 가이드](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)처럼 네트워크 안정성 기준과 함께 보는 편이 좋습니다.

### H3. 서명 대상은 파싱된 JSON이 아니라 원본 바디여야 한다

웹훅 서명 검증에서 가장 흔한 실수는 이미 파싱한 객체를 다시 `JSON.stringify()`해서 서명하는 것입니다.
JSON은 의미가 같아도 공백, 키 순서, 줄바꿈이 달라지면 바이트가 달라질 수 있습니다.

서명은 바이트 단위로 계산되므로, 발신자가 서명한 원본 요청 바디를 그대로 써야 합니다.
Express나 Fastify 같은 프레임워크를 쓴다면 웹훅 라우트만 원본 바디를 보존하도록 설정해야 합니다.

```js
app.post('/webhooks/payment', express.raw({ type: 'application/json' }), async (req, res) => {
  const verified = await verifyWebhookSignature({
    rawBody: req.body,
    signatureHeader: req.header('x-webhook-signature'),
    timestampHeader: req.header('x-webhook-timestamp'),
    secret: process.env.WEBHOOK_SECRET
  });

  if (!verified) {
    return res.status(401).json({ error: 'invalid_signature' });
  }

  const event = JSON.parse(req.body.toString('utf8'));
  await handlePaymentEvent(event);

  return res.status(204).send();
});
```

핵심은 검증을 먼저 하고, 그 다음에 JSON을 파싱하는 순서입니다.
이 순서가 지켜지면 파서 설정이나 포맷팅 차이가 서명 검증을 깨뜨릴 가능성이 줄어듭니다.

## Node.js Web Crypto로 HMAC 계산하기

### H3. secret은 CryptoKey로 가져오고 메시지는 Uint8Array로 다룬다

Node.js에서는 `globalThis.crypto.subtle`을 통해 Web Crypto API를 사용할 수 있습니다.
HMAC 서명을 만들려면 secret 문자열을 바이트로 바꾼 뒤 `importKey()`로 `CryptoKey`를 만들고, `sign()`으로 메시지 서명을 계산합니다.

```js
const encoder = new TextEncoder();

async function importHmacKey(secret) {
  return crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );
}

async function createHmacHex(message, secret) {
  const key = await importHmacKey(secret);
  const signature = await crypto.subtle.sign('HMAC', key, message);

  return Buffer.from(signature).toString('hex');
}
```

`message`는 `Buffer`나 `Uint8Array`처럼 원본 바디 바이트를 그대로 넘기는 편이 좋습니다.
문자열로 바꿨다가 다시 인코딩하면, 문자 인코딩이나 줄바꿈 처리에서 예상치 못한 차이가 생길 수 있습니다.

단순한 해시 계산과 HMAC의 차이를 함께 설명해야 한다면 [Node.js crypto.hash 가이드](/development/blog/seo/2026/05/12/nodejs-crypto-hash-one-shot-digest-guide.html)를 내부 참고 글로 연결해 둘 수 있습니다.

### H3. 타임스탬프를 서명 메시지에 포함하면 replay 위험을 줄일 수 있다

서명만 맞다고 해서 항상 안전한 것은 아닙니다.
누군가가 예전에 정상적으로 들어온 요청을 저장했다가 다시 보내면, 같은 바디와 같은 서명이므로 검증을 통과할 수 있습니다.

이 위험을 줄이려면 발신자가 보낸 타임스탬프를 서명 대상에 포함하고, 수신 서버에서 허용 시간 창을 검증합니다.

```js
function buildSignedPayload(timestamp, rawBody) {
  return Buffer.concat([
    Buffer.from(String(timestamp), 'utf8'),
    Buffer.from('.', 'utf8'),
    Buffer.from(rawBody)
  ]);
}

function isFreshTimestamp(timestamp, now = Date.now()) {
  const receivedAt = Number(timestamp) * 1000;
  const toleranceMs = 5 * 60 * 1000;

  return Number.isFinite(receivedAt) && Math.abs(now - receivedAt) <= toleranceMs;
}
```

발신 서비스마다 헤더 이름과 서명 포맷은 다를 수 있습니다.
중요한 것은 “무엇을 서명했는지”를 문서로 고정하고, 수신 서버와 테스트가 같은 규칙을 쓰게 만드는 것입니다.

## 안전한 검증 함수 만들기

### H3. 비교는 timingSafeEqual로 처리한다

서명 문자열을 단순히 `===`로 비교하면 길이나 비교 위치에 따라 미세한 시간 차이가 생길 수 있습니다.
웹훅 엔드포인트에서 이 차이가 곧바로 실전 취약점이 된다고 단정할 필요는 없지만, 보안 코드에서는 안전한 비교 함수를 기본값으로 두는 편이 좋습니다.

Node.js의 `crypto.timingSafeEqual()`은 같은 길이의 버퍼를 일정한 방식으로 비교합니다.
따라서 길이 검사를 먼저 하고, 길이가 같을 때만 비교해야 합니다.

```js
import { timingSafeEqual } from 'node:crypto';

function safeEqualHex(expectedHex, actualHex) {
  const expected = Buffer.from(expectedHex, 'hex');
  const actual = Buffer.from(actualHex, 'hex');

  if (expected.length !== actual.length) {
    return false;
  }

  return timingSafeEqual(expected, actual);
}
```

서명 헤더가 `sha256=...`처럼 prefix를 포함한다면, 비교 전에 허용한 형식만 파싱해야 합니다.
알 수 없는 알고리즘이나 빈 문자열을 조용히 통과시키면 안 됩니다.

### H3. 검증 실패 이유는 내부 로그에만 제한적으로 남긴다

웹훅 검증 실패는 운영에서 반드시 관측되어야 합니다.
하지만 응답 본문에 자세한 실패 이유를 그대로 내보낼 필요는 없습니다.

외부 응답은 `invalid_signature`처럼 단순하게 유지하고, 내부 로그에는 아래 정도만 남기는 편이 안전합니다.

- `missing_signature`
- `stale_timestamp`
- `malformed_signature`
- `signature_mismatch`
- 요청 ID 또는 웹훅 이벤트 ID
- 발신 서비스 이름

secret, 전체 서명값, 원본 요청 바디는 로그에 남기지 않습니다.
로그 예제의 민감정보 처리 기준은 [로그 예제 살균 가이드](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)와 같은 원칙으로 맞추면 됩니다.

## 전체 예제

### H3. 검증 함수는 입출력을 작게 유지한다

아래 예제는 원본 바디, 서명 헤더, 타임스탬프 헤더, secret만 받아 boolean을 반환합니다.
실제 서비스에서는 실패 이유를 별도 enum으로 반환해 메트릭에 연결해도 좋습니다.

```js
import { timingSafeEqual } from 'node:crypto';

const encoder = new TextEncoder();

async function importHmacKey(secret) {
  return crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );
}

async function hmacHex(message, secret) {
  const key = await importHmacKey(secret);
  const signature = await crypto.subtle.sign('HMAC', key, message);

  return Buffer.from(signature).toString('hex');
}

function parseSignature(header) {
  if (typeof header !== 'string') {
    return null;
  }

  const match = header.match(/^sha256=([a-f0-9]{64})$/i);
  return match ? match[1].toLowerCase() : null;
}

function isFreshTimestamp(timestamp, now = Date.now()) {
  const receivedAt = Number(timestamp) * 1000;
  const toleranceMs = 5 * 60 * 1000;

  return Number.isFinite(receivedAt) && Math.abs(now - receivedAt) <= toleranceMs;
}

function buildSignedPayload(timestamp, rawBody) {
  return Buffer.concat([
    Buffer.from(String(timestamp), 'utf8'),
    Buffer.from('.', 'utf8'),
    Buffer.from(rawBody)
  ]);
}

function safeEqualHex(expectedHex, actualHex) {
  const expected = Buffer.from(expectedHex, 'hex');
  const actual = Buffer.from(actualHex, 'hex');

  return expected.length === actual.length && timingSafeEqual(expected, actual);
}

export async function verifyWebhookSignature({
  rawBody,
  signatureHeader,
  timestampHeader,
  secret
}) {
  if (!secret || !isFreshTimestamp(timestampHeader)) {
    return false;
  }

  const receivedSignature = parseSignature(signatureHeader);
  if (!receivedSignature) {
    return false;
  }

  const signedPayload = buildSignedPayload(timestampHeader, rawBody);
  const expectedSignature = await hmacHex(signedPayload, secret);

  return safeEqualHex(expectedSignature, receivedSignature);
}
```

이 함수는 프레임워크에 직접 묶여 있지 않습니다.
덕분에 Express, Fastify, 서버리스 함수 어디에서든 원본 바디만 넘길 수 있으면 같은 검증 로직을 재사용할 수 있습니다.

## 테스트 체크리스트

### H3. 성공 케이스보다 실패 케이스를 더 많이 검증한다

서명 검증 코드는 정상 요청 하나만 통과한다고 충분하지 않습니다.
오히려 실패해야 하는 요청이 정확히 실패하는지가 더 중요합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('rejects a changed body', async () => {
  const secret = 'test_secret';
  const timestamp = Math.floor(Date.now() / 1000);
  const originalBody = Buffer.from('{"event":"paid"}');
  const changedBody = Buffer.from('{"event":"refunded"}');

  const signature = await hmacHex(buildSignedPayload(timestamp, originalBody), secret);

  const verified = await verifyWebhookSignature({
    rawBody: changedBody,
    signatureHeader: `sha256=${signature}`,
    timestampHeader: String(timestamp),
    secret
  });

  assert.equal(verified, false);
});
```

추가로 아래 케이스를 확인하면 운영 사고를 줄일 수 있습니다.

1. 올바른 바디와 서명은 통과한다.
2. 바디가 1바이트라도 바뀌면 실패한다.
3. 타임스탬프가 오래되면 실패한다.
4. 서명 헤더가 없거나 형식이 틀리면 실패한다.
5. secret이 없으면 실패한다.
6. 실패 로그에 secret과 원본 바디가 남지 않는다.

내장 테스트 러너로 이런 검증을 묶는 방법은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 참고하면 흐름을 잡기 쉽습니다.

## 운영 도입 체크리스트

### H3. 웹훅 라우트는 일반 JSON 미들웨어와 분리한다

웹훅 검증을 안정적으로 운영하려면 라우팅 단계에서부터 일반 API와 경계를 나누는 편이 좋습니다.

1. 웹훅 엔드포인트는 원본 바디를 보존한다.
2. 서명 검증 전에는 이벤트를 처리하지 않는다.
3. 타임스탬프 허용 범위를 문서화한다.
4. secret은 환경 변수나 secret manager에서 주입한다.
5. secret rotation 절차를 준비한다.
6. 검증 실패율을 메트릭으로 본다.
7. 실패 응답은 단순하게 유지한다.

환경 변수 관리 기준은 [Node.js loadEnvFile 가이드](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)와 함께 정리하면 팀 내 규칙으로 남기기 좋습니다.

## 마무리

웹훅 서명 검증은 화려한 보안 기능이 아니라, 외부 이벤트를 믿기 전에 거치는 기본 확인 절차입니다.
Node.js Web Crypto API를 쓰면 별도 패키지 없이도 HMAC 서명을 계산할 수 있고, 원본 바디와 타임스탬프, 안전한 비교 규칙을 함께 묶어 재사용 가능한 검증 함수를 만들 수 있습니다.

가장 중요한 기준은 단순합니다.
**검증 전에는 처리하지 않고, 원본 바디로 서명하며, 실패 정보는 필요한 만큼만 남기는 것**입니다.
이 세 가지를 지키면 웹훅 엔드포인트를 더 예측 가능하고 안전하게 운영할 수 있습니다.

## 함께 보면 좋은 글

- [Node.js crypto.hash 가이드: 짧은 문자열과 버퍼를 빠르게 해시하는 법](/development/blog/seo/2026/05/12/nodejs-crypto-hash-one-shot-digest-guide.html)
- [로그 예제 살균 가이드: 신뢰받는 개발 글을 위한 민감정보 처리법](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)
- [Node.js test runner 가이드: 내장 테스트 러너로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)

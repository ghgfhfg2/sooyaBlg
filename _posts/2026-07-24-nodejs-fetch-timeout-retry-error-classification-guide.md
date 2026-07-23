---
layout: post
title: "Node.js fetch timeout 재시도 가이드: 실패 원인을 분류하고 안전하게 다시 요청하는 법"
date: 2026-07-24 08:00:00 +0900
lang: ko
translation_key: nodejs-fetch-timeout-retry-error-classification-guide
permalink: /development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html
alternates:
  ko: /development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html
  x_default: /development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fetch, timeout, retry, abortsignal, error-handling, resilience, backend]
description: "Node.js fetch() 호출에서 timeout, AbortSignal, HTTP 상태 코드, 네트워크 오류를 분리해 판단하고 재시도 가능한 실패만 안전하게 다시 요청하는 실무 패턴을 정리합니다."
---

Node.js에서 외부 API를 호출할 때 `fetch()`는 가장 익숙한 선택지입니다.
하지만 운영 코드에서 `await fetch(url)`만 두면 장애가 생겼을 때 원인을 분류하기 어렵습니다.
응답이 너무 늦은 것인지, 서버가 500을 돌려준 것인지, 요청 자체가 취소된 것인지가 섞이면 재시도 정책도 흔들립니다.

이 글에서는 Node.js `fetch()` 호출에 timeout, 오류 분류, 제한적인 retry를 붙이는 기준을 정리합니다.
핵심은 모든 실패를 다시 시도하는 것이 아니라, **다시 시도해도 안전하고 의미 있는 실패만 좁게 고르는 것**입니다.
[Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html), [Node.js timeout budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html), [Node.js retry budget 가이드](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html)와 함께 보면 외부 I/O 안정화 흐름을 단계적으로 잡을 수 있습니다.

## fetch 실패를 먼저 분류해야 하는 이유

### H3. HTTP 오류와 네트워크 오류는 다르게 다뤄야 한다

`fetch()`는 HTTP 404나 500 응답을 받았다고 해서 자동으로 예외를 던지지 않습니다.
서버가 응답을 보낸 경우에는 `Response` 객체가 돌아오고, `response.ok`를 보고 성공 여부를 판단해야 합니다.
반대로 DNS 실패, 연결 문제, 요청 취소처럼 응답을 받지 못한 경우에는 예외 흐름으로 들어갑니다.

```js
const response = await fetch('https://api.example.com/items');

if (!response.ok) {
  throw new Error(`upstream returned ${response.status}`);
}

const data = await response.json();
```

이 차이를 놓치면 500 응답을 성공 경로처럼 처리하거나, 반대로 404처럼 재시도해도 소용없는 응답을 계속 다시 요청할 수 있습니다.
운영 코드에서는 최소한 다음 네 가지를 구분하는 편이 좋습니다.

- 정상 응답: `response.ok === true`
- 비정상 HTTP 응답: 4xx, 5xx
- timeout 또는 사용자 취소: `AbortSignal` 기반 취소
- 네트워크 오류: 연결 실패, DNS 실패, 소켓 종료 등

분류가 선명하면 로그와 메트릭도 좋아집니다.
대시보드에서 `upstream_timeout`, `upstream_5xx`, `upstream_4xx`, `network_error`를 따로 볼 수 있으면 장애 원인을 더 빨리 좁힐 수 있습니다.

### H3. timeout은 호출자 경험을 보호하는 상한이다

외부 API는 언젠가 응답할 수 있습니다.
하지만 사용자의 요청은 영원히 기다릴 수 없습니다.
그래서 `fetch()` 호출에는 서비스가 감당할 수 있는 timeout을 붙여야 합니다.

```js
const response = await fetch('https://api.example.com/profile', {
  signal: AbortSignal.timeout(1500)
});
```

이 코드는 1.5초 안에 응답을 받지 못하면 요청을 취소합니다.
다만 timeout 숫자는 감으로 정하면 안 됩니다.
전체 요청 시간이 2초여야 한다면 외부 API 하나에 1.9초를 주는 것은 위험합니다.
렌더링, 데이터 가공, 다른 의존성 호출, 응답 전송 시간이 모두 남아 있어야 하기 때문입니다.

timeout은 "외부 API가 느릴 때 얼마나 기다릴까"가 아니라 "우리 서비스가 이 의존성에 얼마의 시간을 빌려줄 수 있는가"에 가깝습니다.
이 관점은 retry와 함께 볼 때 더 중요해집니다.
첫 요청에 1.5초를 쓰고 두 번 더 재시도하면 총 대기 시간이 쉽게 4초를 넘습니다.

## 안전한 fetch 래퍼 만들기

### H3. 오류 타입을 작게 정의한다

먼저 호출부가 다룰 수 있는 오류 타입을 좁게 정의합니다.
상속 구조를 크게 만들 필요는 없지만, timeout과 HTTP 응답 오류는 구분해 두는 편이 실무에서 편합니다.

```js
export class UpstreamHttpError extends Error {
  constructor(message, { status, body }) {
    super(message);
    this.name = 'UpstreamHttpError';
    this.status = status;
    this.body = body;
  }
}

export class UpstreamTimeoutError extends Error {
  constructor(message = 'upstream request timed out') {
    super(message);
    this.name = 'UpstreamTimeoutError';
  }
}

export class UpstreamNetworkError extends Error {
  constructor(message, { cause }) {
    super(message, { cause });
    this.name = 'UpstreamNetworkError';
  }
}
```

오류 이름을 명확히 두면 로그에서 집계하기 쉽고, 테스트에서도 기대 동작을 고정하기 좋습니다.
민감한 응답 본문을 그대로 에러에 담지 않도록 주의해야 합니다.
예제에서는 구조를 보여 주기 위해 `body`를 받지만, 운영 코드에서는 길이 제한과 마스킹 기준을 함께 둬야 합니다.

### H3. AbortSignal을 조합해 호출자 취소도 존중한다

하위 함수가 자체 timeout만 만들면 상위 요청이 이미 취소됐는데도 외부 API 호출이 계속될 수 있습니다.
따라서 상위에서 전달된 `signal`과 내부 timeout signal을 하나로 묶는 방식이 좋습니다.

```js
function composeSignal({ signal, timeoutMs }) {
  const timeoutSignal = AbortSignal.timeout(timeoutMs);

  if (!signal) {
    return timeoutSignal;
  }

  return AbortSignal.any([signal, timeoutSignal]);
}
```

이렇게 만든 signal을 `fetch()`에 넘기면 사용자 취소와 내부 timeout을 같은 취소 경로로 처리할 수 있습니다.
단, 로그에서는 둘을 가능한 한 구분해야 합니다.
사용자가 브라우저 탭을 닫아 취소한 요청과 외부 API timeout은 운영 대응이 다르기 때문입니다.

### H3. HTTP 상태 코드를 재시도 가능 여부로 나눈다

모든 상태 코드는 의미가 다릅니다.
일반적으로 400, 401, 403, 404 같은 요청 오류는 같은 요청을 반복해도 좋아지지 않습니다.
반대로 408, 429, 500, 502, 503, 504는 일시적일 수 있어 제한적인 retry 후보가 됩니다.

```js
function isRetryableStatus(status) {
  return status === 408 ||
    status === 429 ||
    status === 500 ||
    status === 502 ||
    status === 503 ||
    status === 504;
}
```

이 함수는 정책입니다.
결제, 주문, 메시지 발송처럼 부작용이 있는 API라면 상태 코드만 보고 재시도하면 안 됩니다.
서버가 idempotency key를 지원하는지, 같은 요청이 중복 처리되지 않는지 먼저 확인해야 합니다.

## timeout과 retry를 함께 구현하기

### H3. 한 번의 시도를 작게 감싼다

먼저 단일 `fetch()` 시도를 함수로 분리합니다.
이 함수는 응답을 받고, HTTP 오류를 분류하고, 네트워크 오류를 감쌉니다.

```js
async function fetchJsonOnce(url, {
  method = 'GET',
  headers,
  body,
  signal,
  timeoutMs = 1500
} = {}) {
  try {
    const response = await fetch(url, {
      method,
      headers,
      body,
      signal: composeSignal({ signal, timeoutMs })
    });

    if (!response.ok) {
      const text = await response.text();

      throw new UpstreamHttpError(`upstream returned ${response.status}`, {
        status: response.status,
        body: text.slice(0, 1000)
      });
    }

    return await response.json();
  } catch (error) {
    if (error instanceof UpstreamHttpError) {
      throw error;
    }

    if (error?.name === 'TimeoutError' || error?.name === 'AbortError') {
      throw new UpstreamTimeoutError();
    }

    throw new UpstreamNetworkError('upstream request failed before response', {
      cause: error
    });
  }
}
```

여기서 중요한 점은 HTTP 오류를 직접 만든 타입으로 다시 던진다는 것입니다.
그래야 retry 단계에서 `status`를 보고 재시도 여부를 판단할 수 있습니다.
또한 응답 본문은 길이를 제한해 로그나 에러 객체가 커지지 않게 합니다.

### H3. retry는 횟수보다 전체 예산으로 제한한다

재시도 횟수만 제한하면 전체 시간이 길어질 수 있습니다.
실무에서는 `maxAttempts`와 함께 전체 deadline 또는 budget을 두는 편이 안전합니다.

```js
function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function backoffMs(attempt) {
  const base = 100 * 2 ** attempt;
  const jitter = Math.floor(Math.random() * 80);

  return Math.min(base + jitter, 800);
}

function shouldRetry(error) {
  if (error instanceof UpstreamTimeoutError) {
    return true;
  }

  if (error instanceof UpstreamNetworkError) {
    return true;
  }

  if (error instanceof UpstreamHttpError) {
    return isRetryableStatus(error.status);
  }

  return false;
}
```

timeout, 네트워크 오류, 일부 5xx는 retry 후보가 될 수 있습니다.
하지만 이 함수는 "재시도해도 된다"는 기술적 후보만 말합니다.
실제 호출부에서는 메서드와 API 성격까지 함께 확인해야 합니다.

```js
export async function fetchJsonWithRetry(url, options = {}) {
  const {
    maxAttempts = 3,
    timeoutMs = 1500,
    retryable = true
  } = options;

  let lastError;

  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    try {
      return await fetchJsonOnce(url, {
        ...options,
        timeoutMs
      });
    } catch (error) {
      lastError = error;

      const isLastAttempt = attempt === maxAttempts - 1;

      if (!retryable || isLastAttempt || !shouldRetry(error)) {
        throw error;
      }

      await sleep(backoffMs(attempt));
    }
  }

  throw lastError;
}
```

이 래퍼는 기본 구조를 보여 주기 위한 예제입니다.
운영 환경에서는 전체 deadline을 계산해 남은 시간이 거의 없으면 다음 재시도를 건너뛰는 로직을 추가하는 것이 좋습니다.
또한 `POST` 요청은 기본적으로 `retryable: false`로 두고, idempotency key가 있는 안전한 요청만 별도로 허용하는 편이 낫습니다.

## 호출부에서 지켜야 할 기준

### H3. GET과 부작용 있는 요청을 분리한다

조회 API는 재시도 후보가 되기 쉽습니다.
반대로 결제 승인, 쿠폰 사용, 알림 발송, 재고 차감처럼 부작용이 있는 요청은 실패해도 실제로 처리됐을 가능성이 있습니다.
이런 요청을 같은 payload로 다시 보내면 중복 처리 문제가 생길 수 있습니다.

```js
const product = await fetchJsonWithRetry(
  'https://api.example.com/products/123',
  {
    method: 'GET',
    maxAttempts: 3,
    timeoutMs: 1000
  }
);

await fetchJsonWithRetry(
  'https://api.example.com/orders',
  {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      'idempotency-key': crypto.randomUUID()
    },
    body: JSON.stringify({ productId: product.id }),
    maxAttempts: 2,
    timeoutMs: 1200,
    retryable: true
  }
);
```

두 번째 예제처럼 부작용 있는 요청을 재시도하려면 서버가 idempotency key를 실제로 지원해야 합니다.
클라이언트가 헤더만 보낸다고 안전해지는 것은 아닙니다.
서버가 같은 key의 중복 요청을 같은 결과로 처리한다는 계약이 있어야 합니다.

### H3. 로그에는 attempt와 error kind를 남긴다

retry가 들어간 코드는 성공했더라도 내부적으로 한두 번 실패했을 수 있습니다.
이 정보를 남기지 않으면 외부 API가 흔들리기 시작해도 늦게 알아차립니다.

```js
function classifyError(error) {
  if (error instanceof UpstreamTimeoutError) return 'timeout';
  if (error instanceof UpstreamNetworkError) return 'network';
  if (error instanceof UpstreamHttpError) return `http_${error.status}`;
  return 'unknown';
}
```

실제 로그에는 다음 필드를 권장합니다.

- `upstream`: 호출한 외부 서비스 이름
- `route`: 현재 서비스의 라우트 또는 작업 이름
- `attempt`: 몇 번째 시도인지
- `maxAttempts`: 최대 시도 횟수
- `errorKind`: timeout, network, http_503 같은 분류
- `durationMs`: 이번 시도 또는 전체 호출 시간

이 정도만 있어도 "재시도 덕분에 성공 중인지", "재시도가 부하를 키우는지", "특정 API만 504를 돌려주는지"를 훨씬 빨리 볼 수 있습니다.

## 테스트로 고정할 동작

### H3. 재시도 가능한 상태만 다시 호출되는지 확인한다

retry 래퍼는 장애 때만 드러나는 코드라 테스트가 중요합니다.
특히 400 계열은 재시도하지 않고, 503이나 timeout은 제한된 횟수만 다시 시도하는지 확인해야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('does not retry a non-retryable http status', async () => {
  const error = new UpstreamHttpError('bad request', {
    status: 400,
    body: 'invalid input'
  });

  assert.equal(shouldRetry(error), false);
});

test('retries a temporary upstream status', async () => {
  const error = new UpstreamHttpError('service unavailable', {
    status: 503,
    body: 'try later'
  });

  assert.equal(shouldRetry(error), true);
});

test('retries timeout errors', async () => {
  assert.equal(shouldRetry(new UpstreamTimeoutError()), true);
});
```

테스트는 복잡한 mock 서버까지 가지 않아도 시작할 수 있습니다.
정책 함수부터 고정해 두면 나중에 호출 래퍼를 바꿔도 재시도 기준이 흔들리지 않습니다.

### H3. 취소된 요청이 불필요하게 재시도되지 않게 한다

사용자가 요청을 끊었거나 상위 작업이 취소된 경우에는 재시도를 멈추는 편이 자연스럽습니다.
내부 timeout과 외부 취소를 같은 `UpstreamTimeoutError`로만 뭉개면 이 구분을 놓칠 수 있습니다.

운영 코드에서는 취소 이유를 함께 남기는 구조가 좋습니다.

```js
export class UpstreamCanceledError extends Error {
  constructor(message = 'upstream request canceled') {
    super(message);
    this.name = 'UpstreamCanceledError';
  }
}
```

상위 signal이 이미 `aborted`라면 fetch를 시작하지 않고 바로 취소 오류를 던질 수 있습니다.
이렇게 하면 사용자가 떠난 요청 때문에 외부 API를 계속 두드리는 상황을 줄일 수 있습니다.

## 도입 전 체크리스트

### H3. 숫자보다 정책을 먼저 정한다

`timeoutMs: 1500`, `maxAttempts: 3` 같은 숫자는 마지막에 정해야 합니다.
먼저 다음 질문에 답해야 합니다.

- 이 호출은 조회인가, 부작용이 있는 변경인가?
- 같은 요청을 다시 보내도 중복 처리되지 않는가?
- 상위 요청의 전체 timeout budget은 얼마인가?
- timeout, 4xx, 5xx, network 오류를 로그에서 구분할 수 있는가?
- retry가 실패율을 낮추는지, 외부 API 부하를 키우는지 볼 수 있는가?

이 질문 없이 숫자만 넣으면 장애 상황에서 retry가 보호 장치가 아니라 증폭 장치가 될 수 있습니다.
작게 시작하고, 메트릭을 보며 조정하는 편이 안전합니다.

### H3. 기본값은 보수적으로 둔다

처음부터 모든 요청에 retry를 켜지 않는 것이 좋습니다.
기본값은 짧은 timeout, 낮은 시도 횟수, 부작용 있는 요청의 retry 비활성화로 두는 편이 운영 리스크가 작습니다.

```js
const defaultFetchPolicy = {
  timeoutMs: 1000,
  maxAttempts: 2,
  retryable: false
};
```

그리고 호출부에서 정말 필요한 조회 API에만 `retryable: true`를 켭니다.
이 방식은 조금 번거롭지만, 안전하지 않은 재시도가 조용히 퍼지는 것을 막아 줍니다.

## FAQ

### H3. fetch는 500 응답에서 자동으로 throw하나요?

아니요.
`fetch()`는 HTTP 응답을 받으면 상태 코드가 500이어도 `Response`를 반환합니다.
`response.ok` 또는 `response.status`를 직접 확인해야 합니다.

### H3. timeout이 있으면 retry는 필요 없나요?

역할이 다릅니다.
timeout은 한 번의 호출이 너무 오래 걸리지 않게 막고, retry는 일시적 실패에서 회복할 기회를 줍니다.
다만 둘을 함께 쓰면 전체 대기 시간이 늘어날 수 있으므로 timeout budget 안에서 제한해야 합니다.

### H3. POST 요청도 재시도해도 되나요?

기본적으로는 조심해야 합니다.
서버가 idempotency key를 지원하고, 같은 key의 중복 요청을 안전하게 처리한다는 계약이 있을 때만 제한적으로 재시도하는 편이 좋습니다.

## 관련 글

- [Node.js AbortSignal.any 가이드: timeout과 사용자 취소를 함께 처리하는 법](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js Timeout Budget 가이드: deadline 전파로 느린 요청을 제어하는 법](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)
- [Node.js Retry Budget 가이드: 재시도가 장애를 키우지 않게 제한하는 법](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html)

Node.js `fetch()` 안정화의 출발점은 거창한 회복 로직이 아닙니다.
실패를 분류하고, timeout을 명시하고, 재시도 가능한 상황을 좁히는 것만으로도 운영 장애의 많은 부분을 더 읽기 쉬운 문제로 바꿀 수 있습니다.

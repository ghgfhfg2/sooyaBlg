---
layout: post
title: "Node.js fetch timeout 가이드: AbortSignal로 느린 외부 API를 안전하게 끊는 법"
date: 2026-05-21 20:00:00 +0900
lang: ko
translation_key: nodejs-fetch-abortsignal-timeout-retry-guide
permalink: /development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html
alternates:
  ko: /development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html
  x_default: /development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fetch, abortsignal, timeout, retry, external-api, javascript]
description: "Node.js 내장 fetch에 AbortSignal.timeout()을 연결해 외부 API 지연을 제한하고, 재시도·전체 데드라인·에러 분류·운영 로그까지 안전하게 설계하는 방법을 실무 예제로 정리했습니다."
---

외부 API 호출은 빠를 때보다 느릴 때가 더 위험합니다.
응답이 늦어지는 동안 서버의 요청 핸들러, 커넥션, 워커, 큐 작업이 계속 붙잡히고, 작은 지연이 동시에 쌓이면 전체 서비스가 느려집니다.
Node.js에는 내장 `fetch`가 있지만, 호출부에서 시간 제한을 명확히 걸지 않으면 “언젠가는 끝나겠지”라는 위험한 가정이 코드에 남을 수 있습니다.

`AbortSignal.timeout()`은 Node.js `fetch`에 실무적인 타임아웃을 붙이는 가장 단순한 방법입니다.
다만 요청 1회의 타임아웃만 넣는 것으로는 부족합니다.
재시도 횟수, 전체 데드라인, 취소 원인 분류, 로그 마스킹, 백오프까지 함께 설계해야 장애 상황에서 예측 가능한 동작을 만들 수 있습니다.
이 글에서는 Node.js 내장 `fetch`와 `AbortSignal.timeout()`을 사용해 느린 외부 API를 안전하게 끊는 방법을 정리합니다.
취소 신호 조합 자체가 궁금하다면 [Node.js AbortSignal.any와 timeout 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)를 함께 참고하세요.

## Node.js fetch에 타임아웃이 필요한 이유

### 느린 응답은 실패보다 더 오래 자원을 붙잡는다

외부 API가 즉시 실패하면 호출부는 빠르게 다음 경로를 선택할 수 있습니다.
문제는 응답이 오지도, 실패가 나지도 않는 애매한 상태입니다.
이때 애플리케이션은 소켓과 메모리, 요청 컨텍스트를 계속 유지합니다.

```js
const response = await fetch('https://api.example.com/profile/123');
const profile = await response.json();

console.log(profile.name);
```

위 코드는 읽기 쉽지만 운영 정책이 숨어 있습니다.
“얼마나 기다릴 것인가”, “기다리다 실패하면 어떤 에러로 볼 것인가”, “상위 요청이 취소되면 같이 멈출 것인가”가 정해져 있지 않습니다.
서비스 코드에서는 이 정책을 호출부마다 흩뿌리지 말고 작은 래퍼로 모으는 편이 안전합니다.

### 서버 타임아웃과 클라이언트 타임아웃은 다르다

많은 팀이 서버의 `requestTimeout`이나 프록시 타임아웃을 설정했으니 충분하다고 생각합니다.
하지만 그것은 들어오는 요청을 얼마나 오래 받을지에 대한 정책입니다.
외부로 나가는 `fetch` 호출이 얼마나 오래 기다릴지는 별도로 정해야 합니다.

내부 요청 하나가 외부 API 세 곳을 순서대로 호출한다면, 각 호출이 서버 전체 타임아웃을 거의 다 써 버릴 수 있습니다.
그래서 외부 호출에는 “한 번의 시도 제한”과 “전체 작업 제한”을 나누어 두는 것이 좋습니다.
HTTP 서버 쪽 제한을 같이 점검하려면 [Node.js requestTimeout, timeout, headersTimeout 차이](/development/blog/seo/2026/04/16/nodejs-requesttimeout-timeout-headerstimeout-difference-guide.html)를 참고하면 맥락을 맞추기 쉽습니다.

## AbortSignal.timeout 기본 사용법

### fetch에 signal을 전달한다

`AbortSignal.timeout(ms)`는 지정한 시간이 지나면 자동으로 abort되는 신호를 만듭니다.
Node.js 내장 `fetch`는 `signal` 옵션을 받으므로, 별도 타이머 정리 코드를 직접 작성하지 않아도 됩니다.

```js
const response = await fetch('https://api.example.com/products', {
  signal: AbortSignal.timeout(1500),
});

if (!response.ok) {
  throw new Error(`upstream returned ${response.status}`);
}

const products = await response.json();
console.log(products.length);
```

이 코드는 1.5초 안에 응답이 오지 않으면 요청을 중단합니다.
운영에서는 숫자를 감으로 정하기보다, 사용자 요청의 전체 제한 시간과 외부 API의 평소 지연 분포를 보고 결정해야 합니다.
예를 들어 사용자-facing API라면 500~1500ms, 배치 작업이라면 더 긴 시간을 줄 수 있습니다.

### 타임아웃 에러를 별도 타입으로 감싼다

취소 원인을 호출부에서 매번 문자열로 비교하면 코드가 약해집니다.
작은 커스텀 에러를 만들고, 내부 원인은 `cause`에 보존하는 편이 디버깅에 좋습니다.

```js
class UpstreamTimeoutError extends Error {
  constructor(message, options = {}) {
    super(message, options);
    this.name = 'UpstreamTimeoutError';
    this.retryable = true;
  }
}

async function fetchJsonWithTimeout(url, { timeoutMs = 1500 } = {}) {
  try {
    const response = await fetch(url, {
      signal: AbortSignal.timeout(timeoutMs),
      headers: { accept: 'application/json' },
    });

    if (!response.ok) {
      throw new Error(`upstream returned ${response.status}`);
    }

    return await response.json();
  } catch (error) {
    if (error.name === 'TimeoutError' || error.name === 'AbortError') {
      throw new UpstreamTimeoutError(`upstream timeout after ${timeoutMs}ms`, {
        cause: error,
      });
    }

    throw error;
  }
}
```

Node.js 버전과 취소 경로에 따라 에러 이름이 다르게 보일 수 있으므로, 서비스 내부에서는 직접 노출하지 말고 도메인 에러로 변환하는 방식을 추천합니다.
에러를 감쌀 때 `cause`를 남기면 원인 추적이 쉬워집니다.
관련 패턴은 [Node.js Error cause 가이드: 감싼 에러를 디버깅하기 쉽게 남기는 법](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)에서도 다뤘습니다.

## 재시도와 전체 데드라인을 분리하기

### 재시도마다 같은 타임아웃을 주면 전체 시간이 커진다

외부 API 호출은 일시적인 네트워크 흔들림 때문에 한 번 더 시도하면 성공할 수 있습니다.
하지만 각 시도에 2초 타임아웃을 주고 3번 재시도하면, 최악의 경우 대기 시간이 6초 이상으로 늘어납니다.
사용자 요청 안에서라면 이미 너무 깁니다.

```js
function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function remainingMs(deadlineAt) {
  return Math.max(0, deadlineAt - Date.now());
}
```

전체 데드라인을 먼저 정하고, 각 시도는 남은 시간 안에서만 실행되게 만드는 편이 안전합니다.
이렇게 하면 재시도를 넣어도 상위 요청의 시간 예산을 넘지 않습니다.

### 전체 예산 안에서만 재시도한다

다음 예제는 전체 2500ms 안에서 최대 3번까지 외부 API를 호출합니다.
각 시도는 1000ms를 넘지 않으며, 남은 시간이 부족하면 더 짧은 타임아웃을 사용합니다.

```js
class UpstreamHttpError extends Error {
  constructor(status, message, options = {}) {
    super(message, options);
    this.name = 'UpstreamHttpError';
    this.status = status;
    this.retryable = status === 429 || status >= 500;
  }
}

async function fetchJsonOnce(url, { timeoutMs }) {
  const response = await fetch(url, {
    signal: AbortSignal.timeout(timeoutMs),
    headers: { accept: 'application/json' },
  });

  if (!response.ok) {
    throw new UpstreamHttpError(response.status, `upstream returned ${response.status}`);
  }

  return response.json();
}

async function fetchJsonWithRetry(url, options = {}) {
  const maxAttempts = options.maxAttempts ?? 3;
  const perAttemptTimeoutMs = options.perAttemptTimeoutMs ?? 1000;
  const totalTimeoutMs = options.totalTimeoutMs ?? 2500;
  const deadlineAt = Date.now() + totalTimeoutMs;

  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    const left = remainingMs(deadlineAt);

    if (left <= 0) {
      throw new UpstreamTimeoutError(`upstream deadline exceeded after ${totalTimeoutMs}ms`, {
        cause: lastError,
      });
    }

    try {
      return await fetchJsonOnce(url, {
        timeoutMs: Math.min(perAttemptTimeoutMs, left),
      });
    } catch (error) {
      lastError = error;

      const retryable = error.retryable === true || error.name === 'TimeoutError' || error.name === 'AbortError';
      const canRetry = retryable && attempt < maxAttempts && remainingMs(deadlineAt) > 100;

      if (!canRetry) {
        throw error;
      }

      await sleep(Math.min(100 * attempt, remainingMs(deadlineAt)));
    }
  }

  throw lastError;
}
```

핵심은 “시도별 제한”과 “전체 제한”을 동시에 둔다는 점입니다.
재시도는 성공률을 높이는 도구이지, 무한정 기다리는 장치가 아닙니다.
타임아웃 예산을 서비스 전체로 전파하는 설계는 [Node.js timeout budget과 deadline propagation 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와도 잘 맞습니다.

## 상위 취소 신호와 함께 조합하기

### 사용자가 연결을 끊으면 외부 API도 멈춰야 한다

웹 서버에서는 클라이언트가 연결을 끊었는데도 백엔드가 외부 API를 계속 호출하는 일이 생길 수 있습니다.
이런 작업은 결과를 돌려줄 곳이 없으므로 빨리 멈추는 편이 낫습니다.
상위에서 받은 `AbortSignal`과 타임아웃 신호를 함께 조합하면 이 문제를 줄일 수 있습니다.

```js
function timeoutSignal(ms) {
  return AbortSignal.timeout(ms);
}

function combineSignals(signals) {
  const validSignals = signals.filter(Boolean);

  if (validSignals.length === 1) {
    return validSignals[0];
  }

  return AbortSignal.any(validSignals);
}

async function fetchJson(url, { signal, timeoutMs = 1500 } = {}) {
  const combinedSignal = combineSignals([signal, timeoutSignal(timeoutMs)]);

  const response = await fetch(url, {
    signal: combinedSignal,
    headers: { accept: 'application/json' },
  });

  if (!response.ok) {
    throw new UpstreamHttpError(response.status, `upstream returned ${response.status}`);
  }

  return response.json();
}
```

`AbortSignal.any()`를 쓰면 “사용자가 취소함”과 “시간 초과”를 하나의 `signal`로 묶을 수 있습니다.
호출부는 한 가지 옵션만 넘기면 되고, 래퍼 내부에서 정책을 일관되게 적용합니다.
상위 취소 지점을 명시적으로 넣는 습관은 파일 읽기, 큐 작업, 스트림 처리에도 그대로 적용할 수 있습니다.

### 응답 바디 읽기도 시간 예산 안에 포함한다

`fetch`는 헤더를 받는 것과 바디를 읽는 것이 분리되어 있습니다.
큰 JSON을 내려받거나 스트리밍 응답을 처리한다면 `response.json()` 단계에서도 시간이 걸릴 수 있습니다.
같은 `signal`을 `fetch`에 전달하면 바디 읽기 중에도 취소가 전파되지만, 호출부에서는 전체 시간을 기준으로 로그를 남기는 것이 좋습니다.

```js
async function measuredFetchJson(url, options = {}) {
  const startedAt = performance.now();

  try {
    const data = await fetchJson(url, options);
    const durationMs = Math.round(performance.now() - startedAt);

    console.log('upstream fetch succeeded', { durationMs });
    return data;
  } catch (error) {
    const durationMs = Math.round(performance.now() - startedAt);

    console.warn('upstream fetch failed', {
      durationMs,
      errorName: error.name,
      retryable: error.retryable === true,
    });

    throw error;
  }
}
```

운영 로그에는 전체 URL 대신 서비스 이름, 엔드포인트 키, 상태 코드, 시간, 재시도 횟수처럼 안전한 필드를 남기세요.
쿼리 문자열에는 사용자 ID, 검색어, 토큰 같은 민감정보가 섞일 수 있습니다.
로그 예시를 안전하게 다루는 기준은 [로그 예시 비식별화 가이드](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)를 참고할 수 있습니다.

## 운영 기준 잡기

### 어떤 요청을 재시도할지 구분한다

모든 실패를 재시도하면 장애를 키울 수 있습니다.
일반적으로 네트워크 오류, 타임아웃, 429, 5xx는 재시도 후보가 될 수 있습니다.
반대로 400, 401, 403, 404처럼 요청 자체가 잘못되었거나 권한이 없는 경우는 재시도해도 성공 가능성이 낮습니다.

```js
function isRetryableFetchError(error) {
  if (error.retryable === true) {
    return true;
  }

  if (error.name === 'TimeoutError' || error.name === 'AbortError') {
    return true;
  }

  return false;
}
```

재시도는 반드시 작은 백오프와 함께 사용하세요.
장애 중인 외부 API에 동시에 재시도를 몰아넣으면 상대 서비스를 더 압박하고, 내 서비스의 큐도 더 빠르게 쌓입니다.
외부 호출 보호 관점에서는 [Node.js circuit breaker 가이드](/development/blog/seo/2026/04/04/nodejs-circuit-breaker-failure-isolation-guide.html)와 함께 설계하는 것이 좋습니다.

### 관측 지표를 먼저 정한다

타임아웃 정책은 한 번 정하고 끝나는 값이 아닙니다.
서비스 트래픽, 외부 API 품질, 배포 환경이 바뀌면 조정해야 합니다.
최소한 다음 지표는 남기는 것을 추천합니다.

- 외부 API별 요청 수와 실패 수
- 타임아웃 수와 5xx 수
- p50, p95, p99 응답 시간
- 재시도 횟수와 최종 성공률
- 전체 데드라인 초과 수

이 지표가 있어야 “타임아웃을 늘릴지”, “재시도를 줄일지”, “서킷 브레이커를 열지”를 판단할 수 있습니다.
단순히 타임아웃을 길게 늘리는 것은 사용자의 대기 시간을 장애에 기부하는 선택이 될 수 있습니다.

## 실무 체크리스트

### 기본값은 짧고 명시적으로 둔다

외부 API 호출 래퍼에는 기본 타임아웃을 넣되, 호출부에서 의도를 드러낼 수 있게 옵션을 열어 두세요.
사용자 요청 경로와 백오피스 배치 경로는 같은 값을 쓰지 않는 편이 좋습니다.

```js
const profile = await measuredFetchJson('https://api.example.com/profile/123', {
  timeoutMs: 800,
});

const report = await measuredFetchJson('https://api.example.com/monthly-report', {
  timeoutMs: 5000,
});

console.log({ profileId: profile.id, reportReady: Boolean(report) });
```

짧은 타임아웃은 빠른 실패를 만들고, 빠른 실패는 폴백과 사용자 안내를 가능하게 합니다.
단, 정상 응답도 자주 끊을 정도로 짧으면 오히려 오류율을 높입니다.
실제 지연 분포를 보고 조정해야 합니다.

### 민감정보를 URL과 로그에 남기지 않는다

예제에서는 이해를 위해 URL을 문자열로 직접 썼지만, 실제 코드에서는 토큰이나 개인정보를 URL에 넣지 않는 편이 안전합니다.
인증 정보는 헤더로 보내고, 로그에는 헤더 값을 남기지 마세요.
오류 메시지에도 원본 URL 전체를 그대로 넣지 않는 것이 좋습니다.

```js
function safeUpstreamLogFields({ service, operation, status, durationMs }) {
  return {
    service,
    operation,
    status,
    durationMs,
  };
}

console.log('upstream request completed', safeUpstreamLogFields({
  service: 'billing-api',
  operation: 'get-invoice-summary',
  status: 200,
  durationMs: 342,
}));
```

블로그 글의 예제도 실제 토큰, 실제 사용자 식별자, 내부 호스트명을 그대로 담으면 안 됩니다.
공개 문서에서는 `api.example.com`처럼 문서용 도메인을 쓰고, 운영 로그 예시는 반드시 비식별화하세요.

## FAQ

### AbortSignal.timeout만 쓰면 충분한가요?

작은 스크립트나 단일 요청에는 충분할 수 있습니다.
서비스 코드에서는 재시도, 전체 데드라인, 상위 취소 신호, 에러 분류, 로그 정책까지 함께 두는 것을 추천합니다.
타임아웃은 시작점이지 전체 장애 대응 전략은 아닙니다.

### 타임아웃 값은 몇 ms가 적당한가요?

정답은 서비스마다 다릅니다.
사용자 요청 경로라면 전체 응답 목표에서 남은 시간 예산을 계산하고, 외부 API의 p95·p99 지연을 참고하세요.
배치 작업은 더 길게 줄 수 있지만, 무한 대기는 피해야 합니다.

### POST 요청도 재시도해도 되나요?

멱등성이 보장되지 않는 POST는 조심해야 합니다.
중복 결제, 중복 주문, 중복 알림처럼 부작용이 있는 요청은 idempotency key나 중복 방지 장치를 먼저 설계해야 합니다.
관련 내용은 [Node.js idempotency key 가이드](/development/blog/seo/2026/04/10/nodejs-idempotency-key-duplicate-request-prevention-guide.html)를 함께 보면 좋습니다.

## 정리

Node.js 내장 `fetch`를 운영 코드에서 안전하게 쓰려면 `AbortSignal.timeout()`으로 명시적인 시간 제한을 걸어야 합니다.
하지만 더 중요한 것은 시도별 타임아웃, 전체 데드라인, 재시도 가능 여부, 상위 취소 신호, 안전한 로그를 한 묶음으로 설계하는 것입니다.
느린 외부 API를 무한히 기다리지 않고 빠르게 실패시키면, 서비스는 폴백·재시도·사용자 안내 같은 다음 선택지를 가질 수 있습니다.
작은 fetch 래퍼 하나가 장애 전파를 줄이는 첫 번째 보호막이 됩니다.

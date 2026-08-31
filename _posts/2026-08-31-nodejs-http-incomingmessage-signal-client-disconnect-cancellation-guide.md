---
layout: post
title: "Node.js req.signal 가이드: 클라이언트 연결 종료 시 하위 작업을 안전하게 취소하는 법"
date: 2026-08-31 20:00:00 +0900
lang: ko
translation_key: nodejs-http-incomingmessage-signal-client-disconnect-cancellation-guide
permalink: /development/blog/seo/2026/08/31/nodejs-http-incomingmessage-signal-client-disconnect-cancellation-guide.html
alternates:
  ko: /development/blog/seo/2026/08/31/nodejs-http-incomingmessage-signal-client-disconnect-cancellation-guide.html
  x_default: /development/blog/seo/2026/08/31/nodejs-http-incomingmessage-signal-client-disconnect-cancellation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, http, incomingmessage, req-signal, abortsignal, fetch, server, cancellation]
description: "Node.js http.IncomingMessage의 req.signal로 클라이언트 연결 종료를 감지하고 fetch, DB 조회, 스트림 처리 같은 하위 작업을 취소하는 방법을 정리합니다. AbortSignal.any(), timeout, 응답 처리, 민감정보 로그 점검까지 실무 예제로 설명합니다."
---

HTTP 서버에서 오래 걸리는 요청을 처리할 때 놓치기 쉬운 문제가 있습니다.
브라우저 탭이 닫히거나 모바일 네트워크가 끊겼는데도 서버는 여전히 외부 API 호출, DB 조회, 파일 스트림 처리를 계속할 수 있다는 점입니다.
응답을 받을 클라이언트가 이미 사라졌다면 그 작업은 대부분 낭비가 됩니다.

Node.js `node:http`의 `IncomingMessage`에는 이런 상황을 다루기 위한 `message.signal`이 있습니다.
공식 문서 기준으로 이 값은 `AbortSignal`이며, 메시지가 완료되기 전에 destroy되거나 underlying socket이 닫히면 abort됩니다.
또한 Node.js 26.7.0부터는 메시지가 정상 완료된 뒤에는 signal이 abort되지 않는 것으로 변경되어, 정상 완료와 비정상 연결 종료를 더 명확히 구분할 수 있습니다.

이 글에서는 서버 요청 객체에서 흔히 `req.signal`로 접근하는 `IncomingMessage.signal`을 어떻게 하위 `fetch`, DB helper, 스트림 처리에 전달할지 정리합니다.
요청 자체의 timeout 전략은 [Node.js requestTimeout, timeout, headersTimeout 차이 가이드](/development/blog/seo/2026/04/16/nodejs-requesttimeout-timeout-headerstimeout-difference-guide.html), 여러 취소 신호를 합치는 패턴은 [Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html), 리스너 정리는 [Node.js addAbortListener 가이드](/development/blog/seo/2026/07/25/nodejs-events-addabortlistener-cleanup-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js HTTP 공식 문서](https://nodejs.org/api/http.html), [IncomingMessage message.signal 문서](https://nodejs.org/api/http.html#messagesignal), [Node.js AbortSignal 공식 문서](https://nodejs.org/api/globals.html#class-abortsignal)

## req.signal이 필요한 이유

### 클라이언트 연결 종료와 서버 작업 수명을 연결한다

HTTP 요청 handler는 클라이언트와 연결되어 있지만, handler 안에서 실행하는 비동기 작업은 자동으로 멈추지 않습니다.
예를 들어 외부 API를 조회한 뒤 결과를 응답하는 서버를 생각해볼 수 있습니다.

```js
import http from 'node:http';

http.createServer(async (req, res) => {
  const upstream = await fetch('https://example.com/api/report');
  const data = await upstream.json();

  res.setHeader('content-type', 'application/json');
  res.end(JSON.stringify(data));
}).listen(3000);
```

이 코드는 단순하지만, 클라이언트가 응답 전에 연결을 끊어도 `fetch()`는 계속 진행될 수 있습니다.
요청량이 많고 upstream 응답이 느린 서비스라면 이런 작업이 쌓여 서버와 외부 API 양쪽에 불필요한 부하를 만듭니다.

`req.signal`을 넘기면 클라이언트 연결 종료를 하위 작업의 취소 신호로 전달할 수 있습니다.

```js
import http from 'node:http';

http.createServer(async (req, res) => {
  try {
    const upstream = await fetch('https://example.com/api/report', {
      signal: req.signal
    });
    const data = await upstream.json();

    res.setHeader('content-type', 'application/json');
    res.end(JSON.stringify(data));
  } catch (err) {
    if (err.name === 'AbortError') return;

    res.statusCode = 500;
    res.end('Internal Server Error');
  }
}).listen(3000);
```

핵심은 "응답을 보낼 수 없는 요청"과 "계속 실행해야 하는 작업"을 분리하지 않는 것입니다.
사용자가 떠난 요청의 하위 작업은 가능한 빨리 정리되어야 합니다.

### close 이벤트만으로는 계약이 흐려지기 쉽다

오래된 코드에서는 `req.on('close')`나 `res.on('close')`를 직접 듣고 flag를 바꾸는 방식이 많았습니다.
이 방식도 동작할 수 있지만, 하위 함수마다 이벤트 이름과 정리 타이밍을 다시 알아야 합니다.

`AbortSignal`은 더 넓은 Node.js API와 잘 맞습니다.
`fetch`, timer, stream pipeline, 직접 만든 async helper가 모두 같은 형태의 `signal` 옵션을 받을 수 있습니다.
그래서 HTTP 계층에서 만든 취소 판단을 서비스 계층까지 일관되게 전달하기 쉽습니다.

## fetch와 DB 작업에 전달하는 패턴

### handler에서는 signal을 첫 번째 계약으로 둔다

요청 handler가 직접 모든 작업을 처리하면 처음에는 편합니다.
하지만 실무 코드에서는 인증, 입력 검증, DB 조회, 외부 API 호출이 곧 분리됩니다.
이때 helper 함수의 옵션에 `signal`을 포함해두면 나중에 취소 처리를 덧붙이기 쉽습니다.

```js
import http from 'node:http';

async function loadDashboard({ userId, signal }) {
  const profile = await fetch(`https://example.com/users/${userId}`, {
    signal
  }).then((res) => res.json());

  const metrics = await queryMetrics({ userId, signal });

  return { profile, metrics };
}

http.createServer(async (req, res) => {
  try {
    const url = new URL(req.url, 'http://localhost');
    const userId = url.searchParams.get('userId');

    if (!userId) {
      res.statusCode = 400;
      res.end('Missing userId');
      return;
    }

    const data = await loadDashboard({ userId, signal: req.signal });

    res.setHeader('content-type', 'application/json');
    res.end(JSON.stringify(data));
  } catch (err) {
    if (req.signal.aborted) return;

    res.statusCode = 500;
    res.end('Internal Server Error');
  }
}).listen(3000);
```

`queryMetrics()`가 사용하는 DB 클라이언트가 `AbortSignal`을 직접 지원하지 않을 수도 있습니다.
그래도 함수 계약에 `signal`을 넣어두면, 지원되는 드라이버로 바꾸거나 query timeout wrapper를 추가할 때 호출부를 크게 바꾸지 않아도 됩니다.

### timeout과 클라이언트 취소를 함께 묶는다

클라이언트 연결 종료만으로는 충분하지 않을 때가 많습니다.
사용자가 기다리고 있더라도 upstream이 너무 느리면 서버 쪽 timeout이 필요합니다.
이때 `AbortSignal.any()`로 `req.signal`과 `AbortSignal.timeout()`을 하나로 합칠 수 있습니다.

```js
function requestWorkSignal(req, timeoutMs) {
  return AbortSignal.any([
    req.signal,
    AbortSignal.timeout(timeoutMs)
  ]);
}

http.createServer(async (req, res) => {
  const signal = requestWorkSignal(req, 2500);

  try {
    const result = await fetch('https://example.com/search', { signal });
    const body = await result.text();

    res.statusCode = result.ok ? 200 : 502;
    res.end(body);
  } catch (err) {
    if (signal.aborted && req.signal.aborted) return;

    if (signal.aborted) {
      res.statusCode = 504;
      res.end('Gateway Timeout');
      return;
    }

    res.statusCode = 500;
    res.end('Internal Server Error');
  }
}).listen(3000);
```

여기서 클라이언트가 끊은 경우에는 굳이 504 응답을 보내려 하지 않습니다.
반대로 서버 timeout이면 클라이언트가 아직 연결되어 있을 수 있으므로 명시적으로 504를 반환합니다.
취소 원인을 구분하는 로그가 필요하다면 `req.signal.aborted`와 timeout signal의 상태를 따로 남기는 편이 좋습니다.

## 응답 처리에서 주의할 점

### abort 후에는 응답 쓰기를 시도하지 않는다

`req.signal`이 abort된 뒤에는 응답을 보낼 대상이 사라졌을 가능성이 큽니다.
이 상태에서 `res.end()`를 무조건 호출하면 의미 없는 쓰기 시도나 잡음 로그가 생길 수 있습니다.

```js
async function sendJson(res, req, value) {
  if (req.signal.aborted || res.destroyed) return;

  res.setHeader('content-type', 'application/json');
  res.end(JSON.stringify(value));
}
```

응답 helper를 따로 둔다면 `req.signal`이나 `res.destroyed`를 확인하는 기준을 한 곳에 모을 수 있습니다.
모든 handler에서 같은 guard를 반복하기보다, 응답을 쓰는 마지막 경계에 작게 넣는 편이 실수 가능성이 낮습니다.

### 정상 완료와 조기 종료를 다른 지표로 본다

Node.js 26.7.0 변경처럼 정상 완료 후 signal이 abort되지 않는 동작은 운영 지표를 해석할 때 중요합니다.
정상 응답이 끝난 요청까지 "취소"로 집계하면 abort rate가 과장됩니다.

```js
http.createServer(async (req, res) => {
  const startedAt = Date.now();

  res.on('finish', () => {
    recordHttpMetric({
      route: req.url,
      statusCode: res.statusCode,
      durationMs: Date.now() - startedAt,
      aborted: req.signal.aborted
    });
  });

  try {
    const data = await loadData({ signal: req.signal });
    await sendJson(res, req, data);
  } catch (err) {
    if (req.signal.aborted) return;

    res.statusCode = 500;
    res.end('Internal Server Error');
  }
}).listen(3000);
```

실제 서비스에서는 `finish`와 `close`를 함께 관찰해도 됩니다.
다만 지표 이름은 명확해야 합니다.
`client_disconnect`, `server_timeout`, `upstream_error`처럼 원인을 나누면 장애 분석 때 훨씬 빨리 좁혀갈 수 있습니다.

## 스트림과 긴 작업에 적용하기

### stream pipeline에는 signal을 그대로 전달한다

파일 다운로드, 압축, 프록시 응답처럼 스트림을 다루는 handler에서는 취소 처리가 더 중요합니다.
클라이언트가 중간에 끊겼는데도 파일 읽기나 압축이 계속되면 CPU와 I/O가 낭비됩니다.

```js
import http from 'node:http';
import { createReadStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';

http.createServer(async (req, res) => {
  try {
    res.setHeader('content-encoding', 'gzip');
    res.setHeader('content-type', 'text/plain');

    await pipeline(
      createReadStream('large-report.txt'),
      createGzip(),
      res,
      { signal: req.signal }
    );
  } catch (err) {
    if (req.signal.aborted) return;

    if (!res.headersSent) {
      res.statusCode = 500;
      res.end('Internal Server Error');
    }
  }
}).listen(3000);
```

`pipeline()`은 스트림 정리 책임을 한곳에 모으기 좋습니다.
직접 `pipe()`를 여러 번 연결하는 코드보다 오류와 취소를 다루는 기준이 선명합니다.

### CPU 작업은 체크포인트를 직접 넣는다

모든 작업이 자동으로 `AbortSignal`을 이해하는 것은 아닙니다.
큰 JSON 변환, 이미지 메타데이터 처리, 대량 배열 계산처럼 CPU를 오래 쓰는 코드는 중간 체크포인트를 직접 넣어야 합니다.

```js
import { scheduler } from 'node:timers/promises';

async function buildRows(items, { signal }) {
  const rows = [];

  for (let index = 0; index < items.length; index += 1) {
    signal.throwIfAborted();

    rows.push(transformItem(items[index]));

    if (index % 500 === 0) {
      await scheduler.yield();
    }
  }

  return rows;
}
```

`signal.throwIfAborted()`는 취소 확인 지점을 코드에 드러냅니다.
`scheduler.yield()`와 함께 쓰면 긴 반복문이 이벤트 루프를 독점하는 문제도 줄일 수 있습니다.

## 로그와 보안 점검

### 취소 로그에는 원인과 요청 ID만 남긴다

취소는 오류와 다릅니다.
사용자가 탭을 닫은 경우까지 error level로 남기면 실제 장애 신호가 묻힐 수 있습니다.
반대로 아무 기록도 없으면 upstream 비용이 왜 줄거나 늘었는지 설명하기 어렵습니다.

```js
function logCanceledRequest(req, reason) {
  console.info('request canceled', {
    reason,
    method: req.method,
    path: new URL(req.url, 'http://localhost').pathname,
    requestId: req.headers['x-request-id'] ?? null
  });
}
```

로그에는 access token, cookie, authorization header, 원본 query string 전체를 넣지 않는 것이 안전합니다.
검색어나 사용자 식별자가 URL에 들어갈 수 있다면 pathname과 request id 정도만 남기고, 필요한 값은 별도 마스킹 규칙을 거쳐야 합니다.

### 지원 버전을 먼저 확인한다

`IncomingMessage.signal`은 비교적 새 API입니다.
런타임이 Node.js 26.1.0 이상인지, 정상 완료 후 abort 여부까지 기대한다면 Node.js 26.7.0 이상인지 확인해야 합니다.
서비스가 Node.js 22 LTS나 24 LTS에 머물러 있다면 `req.on('close')` 기반 fallback을 유지해야 할 수 있습니다.

```js
function getRequestSignal(req) {
  if (req.signal) return req.signal;

  const controller = new AbortController();

  req.on('close', () => {
    if (!req.complete) {
      controller.abort(new Error('client disconnected'));
    }
  });

  return controller.signal;
}
```

fallback을 둘 때도 외부 함수에는 동일하게 `signal`을 넘기면 됩니다.
버전 차이는 HTTP adapter 안에 가두고, 비즈니스 로직은 `AbortSignal` 계약만 알게 만드는 구조가 오래 유지됩니다.

## 적용 체크리스트

- 요청 handler에서 오래 걸리는 `fetch`, DB helper, stream pipeline에 `req.signal`을 전달한다.
- 서버 timeout이 필요하면 `AbortSignal.any([req.signal, AbortSignal.timeout(ms)])`로 원인을 구분한다.
- `req.signal.aborted` 상태에서는 응답 쓰기를 시도하지 않는다.
- 정상 완료와 클라이언트 조기 종료를 같은 abort 지표로 섞지 않는다.
- 취소 로그에는 request id, method, pathname 정도만 남기고 민감한 header와 query 값은 제외한다.
- Node.js 26.1.0 미만 런타임이 섞여 있다면 `close` 이벤트 기반 fallback을 준비한다.

## FAQ

### req.signal은 res.close와 같은 의미인가요?

같은 문제를 다루지만 표현 방식이 다릅니다.
`close` 이벤트는 이벤트 기반 알림이고, `req.signal`은 하위 비동기 작업에 그대로 넘길 수 있는 취소 계약입니다.
새 코드에서는 가능하면 `AbortSignal` 형태로 통일하는 편이 유지보수에 유리합니다.

### 클라이언트가 연결을 끊으면 항상 에러 로그를 남겨야 하나요?

아닙니다.
모바일 네트워크 전환, 브라우저 이동, 사용자의 탭 닫기는 흔한 일입니다.
대부분은 info나 debug 수준의 취소 지표로 충분하며, upstream 장애나 서버 timeout과 구분해서 보는 것이 좋습니다.

### Express나 Fastify에서도 같은 방식으로 쓸 수 있나요?

프레임워크가 Node.js `IncomingMessage`를 감싸더라도 내부 요청 객체가 `req`로 노출되는 경우가 많습니다.
다만 프레임워크 버전과 adapter에 따라 접근 방식이 다를 수 있으므로, 먼저 실제 런타임에서 `req.signal` 존재 여부를 확인하고 fallback을 두는 편이 안전합니다.

## 함께 읽기

- [Node.js AbortSignal.any 가이드: 여러 취소 신호를 한 번에 묶어 안전하게 다루는 법](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js addAbortListener 가이드: AbortSignal 리스너를 안전하게 등록하고 정리하는 법](/development/blog/seo/2026/07/25/nodejs-events-addabortlistener-cleanup-guide.html)
- [Node.js stream pipeline AbortSignal 가이드: 스트림 취소와 정리를 안전하게 처리하는 법](/development/blog/seo/2026/07/24/nodejs-stream-pipeline-abortsignal-cleanup-guide.html)

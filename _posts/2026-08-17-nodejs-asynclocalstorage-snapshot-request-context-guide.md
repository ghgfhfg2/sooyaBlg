---
layout: post
title: "Node.js AsyncLocalStorage.snapshot 가이드: 요청 컨텍스트를 안전하게 보존하는 방법"
date: 2026-08-17 08:00:00 +0900
lang: ko
translation_key: nodejs-asynclocalstorage-snapshot-request-context-guide
permalink: /development/blog/seo/2026/08/17/nodejs-asynclocalstorage-snapshot-request-context-guide.html
alternates:
  ko: /development/blog/seo/2026/08/17/nodejs-asynclocalstorage-snapshot-request-context-guide.html
  x_default: /development/blog/seo/2026/08/17/nodejs-asynclocalstorage-snapshot-request-context-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, asynclocalstorage, async-context, logging, observability, backend, javascript]
description: "Node.js AsyncLocalStorage.snapshot()으로 요청 ID, 사용자 범위, 로깅 컨텍스트를 안전하게 보존하는 방법을 정리합니다. run(), getStore(), bind()와의 차이, 이벤트 핸들러와 큐 작업에서의 적용 패턴, 테스트 체크리스트까지 실무 예제로 설명합니다."
---

Node.js 서버에서 요청 ID, 테넌트 ID, 추적 ID를 로그마다 직접 인자로 넘기다 보면 함수 시그니처가 금방 지저분해집니다.
그래서 많은 백엔드 코드가 `AsyncLocalStorage`로 요청 컨텍스트를 관리합니다.
문제는 컨텍스트가 항상 기대한 위치에서 실행되지 않는다는 점입니다.
이벤트 핸들러를 나중에 호출하거나, 작업 큐에 함수를 저장했다가 실행하거나, 콜백을 외부 라이브러리에 넘기면 현재 요청의 정보가 사라진 것처럼 보일 수 있습니다.

`AsyncLocalStorage.snapshot()`은 현재 비동기 컨텍스트를 캡처한 뒤, 나중에 특정 함수를 그 컨텍스트 안에서 실행하게 해 주는 API입니다.
Node.js 공식 문서 기준으로 이 API는 `v22.15.0`과 `v23.11.0`에서 안정화된 것으로 표시되어 있습니다.
단순한 컨텍스트 보존에는 `AsyncResource`를 직접 다루는 것보다 읽기 쉽고, 요청 단위 로깅이나 관측성 코드에도 적용하기 좋습니다.

이 글에서는 `AsyncLocalStorage.snapshot()`을 언제 쓰고, 어디에 두면 위험한지, 테스트에서 어떤 시나리오를 확인해야 하는지 정리합니다.
관측성 설계는 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/08/07/nodejs-diagnostics-channel-observability-guide.html), 런타임 오류 분류는 [Node.js util.types.isNativeError 가이드](/development/blog/seo/2026/08/11/nodejs-util-types-isnativeerror-error-classification-guide.html), 이벤트 API 선택 기준은 [Node.js EventTarget vs EventEmitter 가이드](/development/blog/seo/2026/08/14/nodejs-eventtarget-eventemitter-selection-guide.html)도 함께 보면 좋습니다.

## AsyncLocalStorage.snapshot이 해결하는 문제

### 요청 컨텍스트는 비동기 흐름을 따라가야 한다

요청 단위 로그를 남기는 서버를 생각해 봅니다.
각 요청마다 `requestId`를 만들고, 요청 안에서 호출되는 함수는 `asyncLocalStorage.getStore()`로 현재 요청 정보를 가져올 수 있습니다.

```js
import http from 'node:http';
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';

const requestContext = new AsyncLocalStorage();

function log(message, fields = {}) {
  const store = requestContext.getStore();

  console.log(JSON.stringify({
    message,
    requestId: store?.requestId ?? null,
    ...fields
  }));
}

const server = http.createServer((req, res) => {
  requestContext.run({ requestId: randomUUID() }, async () => {
    log('request started', { url: req.url });

    res.end('ok');
  });
});

server.listen(3000);
```

이 정도 코드는 `snapshot()` 없이도 잘 동작합니다.
`run()`이 요청 처리 흐름을 감싸고 있고, 그 안에서 이어지는 비동기 작업은 같은 저장소를 볼 수 있기 때문입니다.
하지만 요청 도중 함수를 만들어 외부에 넘기거나, 나중에 실행할 작업으로 저장하면 이야기가 달라집니다.

### 나중에 실행할 함수에는 캡처 시점이 필요하다

예를 들어 요청 처리 중 만든 작업을 메모리 큐에 넣었다가 별도 타이밍에 실행한다고 가정해 보겠습니다.
큐 실행 시점에는 원래 요청 컨텍스트가 끝났을 수 있습니다.
이때 `snapshot()`으로 "이 함수는 지금의 요청 컨텍스트에서 실행되어야 한다"는 정보를 같이 저장할 수 있습니다.

```js
import { AsyncLocalStorage } from 'node:async_hooks';

const requestContext = new AsyncLocalStorage();
const pendingJobs = [];

export function enqueueAuditJob(payload) {
  const runInCapturedContext = AsyncLocalStorage.snapshot();

  pendingJobs.push(() => {
    return runInCapturedContext(() => {
      const store = requestContext.getStore();

      auditLog({
        requestId: store?.requestId,
        action: payload.action
      });
    });
  });
}

export async function flushJobs() {
  while (pendingJobs.length > 0) {
    const job = pendingJobs.shift();
    await job();
  }
}

function auditLog(record) {
  console.log(record);
}
```

핵심은 `snapshot()`을 큐 실행 시점이 아니라 작업을 등록하는 시점에 호출한다는 점입니다.
실행 시점의 컨텍스트가 아니라, 등록 시점의 컨텍스트를 보존해야 하기 때문입니다.

## run, bind, snapshot의 역할 구분

### run은 새 컨텍스트를 시작할 때 쓴다

`asyncLocalStorage.run(store, callback)`은 새 저장소를 만들고 콜백을 그 컨텍스트 안에서 실행합니다.
HTTP 요청, 메시지 소비, 배치 작업 단위처럼 "이제부터 이 흐름은 하나의 작업이다"라고 선언하는 진입점에 어울립니다.

```js
requestContext.run({ requestId, userId }, async () => {
  await handleRequest();
});
```

대부분의 애플리케이션에서는 `run()`이 기본입니다.
컨텍스트를 새로 시작하는 위치가 명확하고, 코드 리뷰에서도 요청 범위가 어디서 시작되는지 쉽게 보입니다.

### bind는 함수 하나를 현재 컨텍스트에 묶을 때 쓴다

`AsyncLocalStorage.bind(fn)`은 특정 함수를 현재 실행 컨텍스트에 묶은 새 함수를 반환합니다.
콜백 하나를 외부 API에 넘길 때는 `bind()`가 더 간단할 수 있습니다.

```js
const onDone = AsyncLocalStorage.bind((result) => {
  log('external callback finished', { result });
});

externalClient.once('done', onDone);
```

한 함수만 묶으면 되는 상황이라면 `bind()`가 읽기 쉽습니다.
반대로 여러 함수를 같은 캡처 컨텍스트에서 실행해야 하거나, 객체 안에 실행 래퍼를 저장해야 한다면 `snapshot()`이 더 유연합니다.

### snapshot은 실행 래퍼를 재사용할 때 쓴다

`snapshot()`은 현재 컨텍스트를 캡처한 실행 함수를 돌려줍니다.
그 실행 함수에 원하는 콜백을 넘기면, 콜백이 캡처된 컨텍스트 안에서 실행됩니다.

```js
class RequestScopedReporter {
  #runInScope = AsyncLocalStorage.snapshot();

  report(eventName, detail) {
    return this.#runInScope(() => {
      log('report event', {
        eventName,
        detail
      });
    });
  }
}

export function createReporter() {
  return new RequestScopedReporter();
}
```

이 패턴은 요청 처리 중 만든 객체가 나중에 이벤트를 보고해야 할 때 유용합니다.
다만 객체 수명이 요청보다 훨씬 길다면 저장소에 개인정보나 큰 객체를 넣지 않는 것이 중요합니다.
컨텍스트가 오래 붙잡히면 메모리와 개인정보 노출 리스크가 같이 커질 수 있습니다.

## 실무 적용 패턴

### 저장소에는 작고 필요한 값만 넣는다

`AsyncLocalStorage` 저장소에는 요청 객체 전체나 사용자 프로필 전체를 넣기보다, 로그와 추적에 꼭 필요한 식별자만 넣는 편이 안전합니다.
권장 형태는 작고 직렬화 가능한 객체입니다.

```js
requestContext.run({
  requestId,
  traceId,
  route: req.url
}, async () => {
  await handleRequest(req, res);
});
```

토큰, 세션 쿠키, 원본 Authorization 헤더, 주민등록번호 같은 민감정보는 저장소에 넣지 않습니다.
로그 함수가 실수로 저장소 전체를 출력할 수 있고, 오류 리포팅 도구가 컨텍스트를 자동 수집할 수도 있기 때문입니다.

### 이벤트 핸들러 등록 시점을 명확히 한다

이벤트 기반 코드는 컨텍스트가 헷갈리기 쉽습니다.
핸들러가 언제 등록되고 언제 실행되는지 분리되어 있기 때문입니다.
요청 안에서 등록한 핸들러가 요청 컨텍스트를 유지해야 한다면 등록 시점에 캡처합니다.

```js
function subscribeOnce(emitter, eventName, handler) {
  const runInCapturedContext = AsyncLocalStorage.snapshot();

  emitter.once(eventName, (...args) => {
    runInCapturedContext(() => handler(...args));
  });
}
```

이 코드는 이벤트가 어느 비동기 흐름에서 발생하든, 핸들러 본문은 구독 시점의 컨텍스트에서 실행합니다.
반대로 이벤트 발생 시점의 컨텍스트를 따라야 하는 코드라면 이런 래핑이 오히려 잘못된 결과를 만들 수 있습니다.
따라서 "등록 시점"과 "발생 시점" 중 어느 쪽 컨텍스트가 맞는지 먼저 정해야 합니다.

### 백그라운드 작업에는 요청 컨텍스트를 그대로 넘기지 않는다

`snapshot()`은 편리하지만 모든 백그라운드 작업에 요청 컨텍스트를 붙이는 도구는 아닙니다.
이메일 발송, 정산, 리포트 생성처럼 요청보다 오래 사는 작업은 독립적인 작업 ID를 새로 만들고 필요한 값만 명시적으로 넘기는 편이 좋습니다.

```js
export function enqueueBackgroundJob(input) {
  const store = requestContext.getStore();

  jobQueue.push({
    jobId: crypto.randomUUID(),
    requestedBy: store?.userId ?? null,
    traceId: store?.traceId ?? null,
    input
  });
}
```

작업 실행기는 이 값을 바탕으로 새 `run()` 범위를 시작할 수 있습니다.
이렇게 하면 요청 컨텍스트를 통째로 오래 보관하지 않으면서도 추적에 필요한 연결고리는 유지할 수 있습니다.

## 테스트 체크리스트

### 컨텍스트 보존과 단절을 모두 테스트한다

`snapshot()`을 적용한 코드는 성공 케이스만 보면 안 됩니다.
컨텍스트가 있어야 하는 곳에서는 유지되고, 없어야 하는 곳에서는 새 요청의 값으로 오염되지 않는지 확인해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { AsyncLocalStorage } from 'node:async_hooks';

const context = new AsyncLocalStorage();

test('snapshot keeps the context from registration time', async () => {
  const callbacks = [];

  context.run({ requestId: 'request-a' }, () => {
    const runInScope = AsyncLocalStorage.snapshot();
    callbacks.push(() => runInScope(() => context.getStore()));
  });

  const result = context.run({ requestId: 'request-b' }, () => callbacks[0]());

  assert.deepEqual(result, { requestId: 'request-a' });
});
```

이 테스트는 콜백 실행 시점의 컨텍스트가 `request-b`여도, 캡처된 컨텍스트는 `request-a`라는 점을 검증합니다.
반대로 이벤트 발생 시점의 컨텍스트를 따라야 하는 코드라면 이런 테스트가 실패해야 정상입니다.

### 로그 출력에는 민감정보가 섞이지 않아야 한다

컨텍스트 기반 로깅을 만들면 로그 함수가 편해지는 만큼 출력 범위도 넓어질 수 있습니다.
테스트에서는 최소한 다음 항목을 확인하는 것이 좋습니다.

- 로그에 `requestId`나 `traceId`가 포함되는가
- 요청 간 값이 섞이지 않는가
- 토큰, 쿠키, 원본 인증 헤더가 출력되지 않는가
- 컨텍스트가 없는 실행에서도 로그 함수가 예외를 던지지 않는가

이 체크리스트는 보안 점검이면서 운영 안정성 점검입니다.
컨텍스트가 없을 때도 `null`이나 `undefined`를 안전하게 처리해야 배치 작업, CLI, 테스트 코드에서 같은 로거를 재사용할 수 있습니다.

## 운영 전 확인할 기준

### snapshot은 컨텍스트의 수명을 늘릴 수 있다

`snapshot()`으로 만든 실행 래퍼는 캡처된 컨텍스트를 참조합니다.
그래서 래퍼를 오래 보관하면 저장소에 들어 있던 값도 예상보다 오래 살아 있을 수 있습니다.
저장소를 작게 유지하고, 요청 단위 객체를 전역 캐시나 장기 큐에 넣지 않는 습관이 필요합니다.

운영 코드에서는 다음 기준을 두면 안전합니다.

- 요청 진입점에서는 `run()`을 사용한다.
- 단일 콜백 보존에는 `bind()`를 먼저 검토한다.
- 여러 실행을 같은 캡처 컨텍스트에서 처리해야 할 때 `snapshot()`을 사용한다.
- 저장소에는 식별자와 라우팅 정보처럼 작은 값만 둔다.
- 장기 백그라운드 작업은 새 작업 컨텍스트를 만든다.

### 관측성 코드는 장애 원인이 되지 않아야 한다

컨텍스트 로깅은 문제를 찾기 위한 장치이지, 서비스 본문보다 중요한 기능은 아닙니다.
컨텍스트가 없다는 이유로 로그 함수가 실패하거나, 이벤트 핸들러가 예외를 삼키거나, 큐 작업이 중단되면 관측성 코드가 장애를 키우게 됩니다.

```js
function safeLog(message, fields = {}) {
  try {
    const store = requestContext.getStore();
    console.log(JSON.stringify({
      message,
      requestId: store?.requestId ?? null,
      ...fields
    }));
  } catch (error) {
    console.error('log failed', {
      message: error instanceof Error ? error.message : String(error)
    });
  }
}
```

실무에서는 로깅 실패를 완전히 숨기기보다, 민감정보 없이 짧게 남기고 본래 요청 처리는 계속되게 만드는 편이 좋습니다.
`AsyncLocalStorage.snapshot()`도 같은 원칙 안에서 사용해야 합니다.
컨텍스트를 더 정확히 보존하되, 컨텍스트가 없거나 예상과 다를 때도 애플리케이션이 무너지지 않아야 합니다.

## FAQ

### AsyncLocalStorage.snapshot은 모든 콜백에 써야 하나요?

아닙니다.
요청 처리 흐름 안에서 자연스럽게 이어지는 `await`, `Promise`, 타이머 작업은 보통 `run()`만으로 충분합니다.
`snapshot()`은 함수를 저장했다가 나중에 실행하거나, 객체가 생성 시점의 컨텍스트를 기억해야 하는 경우에 우선 검토합니다.

### snapshot과 bind 중 무엇을 먼저 선택해야 하나요?

콜백 하나만 현재 컨텍스트에 묶으면 된다면 `bind()`가 단순합니다.
같은 캡처 컨텍스트에서 여러 콜백을 실행하거나, 클래스 내부에 실행 래퍼를 보관해야 한다면 `snapshot()`이 더 적합합니다.

### 요청 컨텍스트에 사용자 정보를 넣어도 되나요?

사용자 전체 객체를 넣는 것은 피하는 편이 좋습니다.
로그와 추적에 필요한 `userId`, `tenantId`, `requestId` 같은 최소 식별자만 저장하고, 토큰이나 원본 인증 정보는 저장소와 로그에서 제외해야 합니다.

## 참고 자료

- [Node.js 공식 문서: Asynchronous context tracking](https://nodejs.org/api/async_context.html)

---
layout: post
title: "Node.js AsyncLocalStorage bind 가이드: 콜백에서도 요청 컨텍스트 유지하는 법"
date: 2026-05-09 20:00:00 +0900
lang: ko
translation_key: nodejs-asynclocalstorage-bind-callback-context-guide
permalink: /development/blog/seo/2026/05/09/nodejs-asynclocalstorage-bind-callback-context-guide.html
alternates:
  ko: /development/blog/seo/2026/05/09/nodejs-asynclocalstorage-bind-callback-context-guide.html
  x_default: /development/blog/seo/2026/05/09/nodejs-asynclocalstorage-bind-callback-context-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, asynclocalstorage, context, logging, backend]
description: "Node.js AsyncLocalStorage.bind로 콜백, 이벤트 핸들러, 지연 실행 코드에서도 요청 컨텍스트를 안정적으로 유지하는 방법을 정리했습니다. 로그 추적, 작업 큐, 실무 주의사항까지 예제로 설명합니다."
---

요청 단위 로그를 남기다 보면 처음에는 간단해 보입니다.
미들웨어에서 `requestId`를 만들고, 이후 함수들이 그 값을 읽어서 로그에 붙이면 됩니다.
하지만 콜백, 이벤트 핸들러, 지연 실행 함수가 섞이기 시작하면 어느 순간 컨텍스트가 끊긴 것처럼 보이는 로그가 생깁니다.

Node.js의 `AsyncLocalStorage`는 비동기 흐름 안에서 요청 컨텍스트를 유지하는 표준 도구입니다.
그중 `AsyncLocalStorage.bind()`는 **현재 비동기 컨텍스트를 함수에 묶어 두었다가 나중에 실행할 때도 같은 저장소를 읽게 해 주는 도구**입니다.
콜백을 다른 곳에 넘기거나 이벤트 리스너로 등록할 때 특히 유용합니다.

## AsyncLocalStorage.bind가 필요한 이유

### H3. 요청 컨텍스트는 함수 인자만으로 관리하기 어렵다

작은 코드에서는 `requestId`를 함수 인자로 계속 넘겨도 큰 문제가 없습니다.
하지만 실제 서버 코드에서는 호출 단계가 빠르게 늘어납니다.

- HTTP 요청 처리 미들웨어
- 서비스 함수와 저장소 함수
- 이벤트 리스너
- 재시도 콜백
- 백그라운드 작업 예약
- 관측 로그와 메트릭 기록

이 모든 함수에 `requestId`, `userId`, `traceId` 같은 값을 계속 인자로 전달하면 비즈니스 로직보다 컨텍스트 전달 코드가 더 눈에 띄게 됩니다.
그래서 요청 컨텍스트 로깅에는 [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)처럼 `AsyncLocalStorage`를 사용하는 패턴이 자주 쓰입니다.

### H3. 문제는 나중에 실행되는 콜백이다

`AsyncLocalStorage.run()`으로 요청 컨텍스트를 시작하면 일반적인 `await` 흐름에서는 저장소가 잘 이어집니다.
문제는 함수를 현재 위치에서 바로 실행하지 않고, 다른 객체나 모듈에 넘긴 뒤 나중에 호출하는 경우입니다.

예를 들어 이벤트 emitter에 등록한 리스너, 작업 큐의 콜백, 타이머로 미뤄 둔 함수는 작성 시점과 실행 시점이 다를 수 있습니다.
이때 실행 시점의 비동기 컨텍스트가 기대와 다르면 로그에서 `requestId`가 비거나 다른 요청의 값처럼 보일 수 있습니다.

`AsyncLocalStorage.bind()`는 이 틈을 줄이기 위한 도구입니다.
현재 컨텍스트를 함수에 캡처해 두고, 나중에 호출될 때 그 컨텍스트 안에서 실행되도록 감싸 줍니다.

## AsyncLocalStorage.bind 기본 사용법

### H3. 현재 컨텍스트를 함수에 묶기

기본 구조는 단순합니다.
`AsyncLocalStorage.run()` 안에서 콜백을 만들고, `AsyncLocalStorage.bind()`로 감싼 함수를 외부에 전달합니다.

```js
import { AsyncLocalStorage } from 'node:async_hooks';

const requestContext = new AsyncLocalStorage();

function log(message) {
  const store = requestContext.getStore();
  const requestId = store?.requestId ?? 'unknown';
  console.log(`[${requestId}] ${message}`);
}

function registerLater(callback) {
  setTimeout(callback, 10);
}

requestContext.run({ requestId: 'req_123' }, () => {
  const callback = AsyncLocalStorage.bind(() => {
    log('delayed callback executed');
  });

  registerLater(callback);
});
```

위 예제에서 콜백은 `setTimeout`을 통해 나중에 실행됩니다.
그래도 `bind()` 시점의 컨텍스트가 함수에 묶였기 때문에 `log()`는 `req_123`을 읽을 수 있습니다.

### H3. bind는 함수를 바로 실행하지 않는다

`bind()`는 함수를 실행하는 API가 아닙니다.
현재 컨텍스트를 보존하는 새 함수를 반환합니다.

```js
const boundHandler = AsyncLocalStorage.bind(handler);

// 나중에 호출
boundHandler();
```

이 차이를 이해해야 합니다.
컨텍스트를 캡처하는 시점은 `bind()`를 호출하는 순간이고, 실제 로직이 실행되는 시점은 반환된 함수를 호출하는 순간입니다.
따라서 `bind()`는 반드시 보존하고 싶은 컨텍스트가 활성화된 상태에서 호출해야 합니다.

## 이벤트 핸들러에서 컨텍스트 유지하기

### H3. EventEmitter 리스너에 requestId 붙이기

이벤트 기반 코드에서는 리스너를 등록한 뒤 나중에 이벤트가 발생합니다.
요청 처리 중에 이벤트 리스너를 만들고, 그 리스너가 같은 요청 컨텍스트를 사용해야 한다면 `bind()`를 적용할 수 있습니다.

```js
import { EventEmitter } from 'node:events';
import { AsyncLocalStorage } from 'node:async_hooks';

const bus = new EventEmitter();
const requestContext = new AsyncLocalStorage();

function logOrderEvent(event) {
  const store = requestContext.getStore();
  console.log({
    requestId: store?.requestId,
    event
  });
}

function handleRequest(requestId) {
  requestContext.run({ requestId }, () => {
    const listener = AsyncLocalStorage.bind((event) => {
      logOrderEvent(event);
    });

    bus.once('order:completed', listener);
  });
}

handleRequest('req_order_1');

bus.emit('order:completed', { orderId: 'order_100' });
```

요점은 리스너를 등록하는 시점입니다.
요청 컨텍스트 안에서 리스너를 `bind()`하면 이벤트가 나중에 발생해도 해당 리스너는 캡처된 저장소를 기준으로 동작합니다.

### H3. 모든 리스너에 무조건 bind를 붙일 필요는 없다

`bind()`는 필요한 곳에만 쓰는 편이 좋습니다.
요청과 무관한 전역 이벤트, 프로세스 생명주기 이벤트, 공통 모니터링 이벤트까지 모두 요청 컨텍스트에 묶으면 오히려 로그 해석이 어려워질 수 있습니다.

기준은 간단합니다.

- 이 콜백이 특정 요청에서 만들어졌는가?
- 나중에 실행되어도 그 요청의 컨텍스트가 필요한다?
- 함수 인자로 넘기는 방식보다 컨텍스트 저장소를 읽는 편이 더 명확한가?

세 질문에 답이 모두 예라면 `bind()`를 고려할 만합니다.
그렇지 않다면 명시적인 인자 전달이나 독립 로그가 더 안전할 수 있습니다.

## 작업 큐와 지연 실행 코드에서 쓰는 패턴

### H3. 큐에 넣는 함수의 컨텍스트를 캡처한다

간단한 메모리 큐나 내부 작업 스케줄러를 만들 때도 같은 원칙을 적용할 수 있습니다.
작업을 큐에 넣는 시점의 요청 컨텍스트를 캡처하면, 나중에 worker가 실행할 때도 원래 요청 정보를 로그에 남길 수 있습니다.

```js
import { AsyncLocalStorage } from 'node:async_hooks';

const requestContext = new AsyncLocalStorage();
const jobs = [];

function enqueue(job) {
  jobs.push(AsyncLocalStorage.bind(job));
}

async function runJobs() {
  while (jobs.length > 0) {
    const job = jobs.shift();
    await job();
  }
}

requestContext.run({ requestId: 'req_import_42' }, () => {
  enqueue(async () => {
    const store = requestContext.getStore();
    console.log(`process import for ${store.requestId}`);
  });
});

await runJobs();
```

이 패턴은 요청 처리 중에 부수 작업을 예약하고, 작업 실행 로그를 원래 요청과 연결하고 싶을 때 유용합니다.
다만 작업이 오래 지연되거나 프로세스 밖의 큐로 넘어간다면 별도 주의가 필요합니다.

### H3. 프로세스 밖 큐에는 컨텍스트 값을 명시적으로 저장한다

`AsyncLocalStorage.bind()`는 같은 Node.js 프로세스 안에서 함수 실행 컨텍스트를 보존하는 도구입니다.
메시지가 Redis, Kafka, SQS, 데이터베이스 큐처럼 프로세스 밖으로 나가면 함수 자체가 전달되는 것이 아니므로 `bind()`로 해결할 수 없습니다.

이 경우에는 필요한 값을 작업 payload에 명시적으로 넣어야 합니다.

```js
function createJobPayload(input) {
  const store = requestContext.getStore();

  return {
    ...input,
    trace: {
      requestId: store?.requestId,
      userId: store?.userId
    }
  };
}
```

외부 큐에서 꺼낸 작업은 payload의 `trace` 값을 기준으로 새 `AsyncLocalStorage.run()`을 시작하는 방식이 더 적합합니다.
중복 메시지나 재처리까지 고려한다면 [Node.js idempotent consumer 가이드: Kafka 중복 메시지 처리법](/development/blog/seo/2026/03/17/nodejs-idempotent-consumer-kafka-duplicate-message-handling-guide.html)처럼 멱등성 기준도 함께 설계해야 합니다.

## snapshot과 bind의 차이

### H3. bind는 함수 하나를 감싸는 방식이다

`AsyncLocalStorage.bind()`는 특정 함수 하나를 현재 컨텍스트에 묶어 반환합니다.
그래서 이벤트 리스너, 타이머 콜백, 큐에 넣는 작업처럼 “이 함수가 나중에 실행될 것”이라는 구조가 명확할 때 읽기 쉽습니다.

```js
const bound = AsyncLocalStorage.bind(() => {
  writeLog('called later');
});
```

함수 하나를 보존하면 충분한 경우에는 `bind()`가 가장 직관적입니다.

### H3. snapshot은 실행 래퍼를 만들어 재사용한다

반면 `AsyncLocalStorage.snapshot()`은 현재 컨텍스트를 캡처한 실행 함수를 반환합니다.
그 실행 함수에 여러 콜백을 넘겨 같은 컨텍스트에서 실행할 수 있습니다.

여러 함수를 같은 컨텍스트로 실행해야 하거나 클래스 인스턴스 안에 컨텍스트 실행기를 보관해야 한다면 [Node.js AsyncLocalStorage snapshot 가이드: 비동기 컨텍스트를 안전하게 넘기는 법](/development/blog/seo/2026/05/01/nodejs-asynclocalstorage-snapshot-context-handoff-guide.html)의 패턴이 더 자연스러울 수 있습니다.

정리하면 다음처럼 선택하면 됩니다.

- 함수 하나를 나중에 실행한다: `AsyncLocalStorage.bind()`
- 같은 컨텍스트로 여러 함수를 실행한다: `AsyncLocalStorage.snapshot()`
- 프로세스 밖으로 값을 넘긴다: payload에 trace 값을 명시한다

## 실무 로그 설계 예시

### H3. logger가 getStore를 읽도록 만든다

`bind()`의 효과를 제대로 보려면 로그 함수가 매번 현재 저장소를 읽어야 합니다.
요청 시작 시점의 값을 전역 변수에 넣는 방식은 동시 요청에서 안전하지 않습니다.

```js
const requestContext = new AsyncLocalStorage();

const logger = {
  info(message, extra = {}) {
    const store = requestContext.getStore();

    console.log(JSON.stringify({
      level: 'info',
      message,
      requestId: store?.requestId,
      userId: store?.userId,
      ...extra
    }));
  }
};
```

이렇게 해 두면 일반 `await` 흐름에서도, `bind()`로 감싼 콜백에서도 같은 logger를 사용할 수 있습니다.
로그에 걸린 시간을 함께 남기고 싶다면 [Node.js performance.now 가이드: 실행 시간을 정확하게 측정하는 법](/development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html)의 duration 측정 패턴을 결합하면 됩니다.

### H3. 에러 처리 콜백에도 컨텍스트를 남긴다

나중에 실행되는 콜백은 성공 로그보다 실패 로그가 더 중요할 때가 많습니다.
에러 핸들러를 등록할 때도 원래 요청 정보를 보존하면 장애 분석이 쉬워집니다.

```js
requestContext.run({ requestId: 'req_payment_7' }, () => {
  const onFailure = AsyncLocalStorage.bind((error) => {
    logger.info('payment callback failed', {
      errorName: error.name,
      errorMessage: error.message
    });
  });

  registerPaymentCallback({ onFailure });
});
```

운영 코드에서는 에러 객체 전체를 그대로 로그로 남기기보다 이름, 메시지, 안전한 코드, 추적 ID처럼 필요한 값만 구조화하는 편이 좋습니다.
감싼 에러의 원인을 유지해야 한다면 [Node.js Error cause 가이드: 감싼 에러의 원인을 안전하게 남기는 법](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)을 함께 적용할 수 있습니다.

## 주의할 점

### H3. bind 시점이 틀리면 원하는 컨텍스트가 아니다

`bind()`는 호출 시점의 컨텍스트를 캡처합니다.
따라서 요청 컨텍스트 밖에서 미리 만들어 둔 함수를 `bind()`하면 요청별 값이 들어가지 않습니다.

```js
// 좋지 않은 예: 요청 컨텍스트 밖에서 bind됨
const handler = AsyncLocalStorage.bind(() => {
  logger.info('called');
});

function handleRequest(requestId) {
  requestContext.run({ requestId }, () => {
    registerLater(handler);
  });
}
```

요청마다 다른 컨텍스트가 필요하다면 요청 처리 함수 안에서 `bind()`해야 합니다.
이 규칙만 지켜도 대부분의 혼란을 줄일 수 있습니다.

### H3. 컨텍스트에 큰 객체를 넣지 않는다

`AsyncLocalStorage` 저장소에는 추적에 필요한 작은 값만 넣는 것이 좋습니다.
요청 본문 전체, 사용자 객체 전체, 데이터베이스 연결, 큰 배열 같은 값을 넣으면 메모리 사용량과 보안 리스크가 커질 수 있습니다.

추천하는 저장소 값은 다음 정도입니다.

- `requestId`
- `traceId`
- `userId` 또는 내부 식별자
- `route`
- 안전하게 마스킹된 tenant 정보

개인정보, 인증 토큰, API 키는 컨텍스트 저장소와 로그 양쪽 모두에 남기지 않는 편이 안전합니다.
공통 운영 기준에서도 민감정보 마스킹은 필수 점검 항목입니다.

### H3. 테스트로 컨텍스트 유지 여부를 확인한다

컨텍스트 코드는 깨져도 기능 자체는 동작하는 것처럼 보일 수 있습니다.
그래서 로그와 추적이 중요한 코드라면 간단한 테스트를 두는 편이 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { AsyncLocalStorage } from 'node:async_hooks';

const storage = new AsyncLocalStorage();

test('bound callback keeps request context', async () => {
  let callback;

  storage.run({ requestId: 'req_test' }, () => {
    callback = AsyncLocalStorage.bind(() => storage.getStore());
  });

  assert.deepEqual(callback(), { requestId: 'req_test' });
});
```

내장 테스트 도구로 검증 기준을 만들고 싶다면 [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)을 참고하면 시작 비용을 줄일 수 있습니다.

## 적용 체크리스트

### H3. 콜백 등록 지점을 먼저 찾는다

`bind()`를 적용하기 전에는 컨텍스트가 끊기는 지점을 먼저 찾아야 합니다.
다음 패턴을 검색하면 후보를 빠르게 찾을 수 있습니다.

- `setTimeout`, `setImmediate`
- `eventEmitter.on`, `eventEmitter.once`
- `queue.push`, `enqueue`, `schedule`
- `callback`, `onSuccess`, `onFailure`
- 외부 SDK에 넘기는 handler 함수

모든 후보를 한 번에 바꾸기보다 요청 추적이 실제로 필요한 지점부터 적용하는 편이 안전합니다.

### H3. 운영 로그에서 requestId 누락률을 확인한다

수정 후에는 로그 품질을 확인해야 합니다.
단순히 코드가 실행되는지만 보는 것이 아니라, 요청 처리 중 발생한 비동기 로그에 `requestId`가 안정적으로 붙는지 확인합니다.

간단한 확인 항목은 다음과 같습니다.

- 정상 요청 로그에 같은 `requestId`가 이어지는가?
- 지연 실행 콜백 로그에도 `requestId`가 있는가?
- 에러 로그에서 추적 ID가 빠지지 않는가?
- 요청과 무관한 전역 로그에 잘못된 요청 ID가 붙지 않는가?

관측 포인트를 더 체계적으로 만들고 싶다면 [Node.js diagnostics_channel 가이드: 관측 코드를 느슨하게 연결하는 법](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)처럼 로그와 계측 코드를 분리하는 방식도 고려할 수 있습니다.

## 마무리

`AsyncLocalStorage.bind()`는 화려한 API는 아니지만, 콜백 중심 코드에서 요청 컨텍스트를 지키는 데 매우 실용적입니다.
핵심은 **컨텍스트가 살아 있는 시점에 함수를 묶고, 나중에 실행될 콜백에서 같은 저장소를 읽게 만드는 것**입니다.

일반적인 `await` 흐름은 `AsyncLocalStorage.run()`만으로 충분한 경우가 많습니다.
하지만 이벤트 리스너, 내부 큐, 지연 실행 함수처럼 실행 시점이 멀어지는 코드가 있다면 `bind()`를 적용할 지점을 점검해 보세요.
작은 변경만으로도 로그 추적성과 장애 분석 속도가 크게 좋아질 수 있습니다.

## FAQ

### H3. AsyncLocalStorage.bind와 Function.prototype.bind는 같은 기능인가요?

아닙니다.
`Function.prototype.bind`는 `this`와 일부 인자를 고정하는 JavaScript 함수 기능입니다.
`AsyncLocalStorage.bind()`는 현재 비동기 컨텍스트를 함수 실행에 연결하는 Node.js API입니다.
이름은 비슷하지만 해결하는 문제가 다릅니다.

### H3. AsyncLocalStorage.bind를 모든 콜백에 적용해도 되나요?

권장하지 않습니다.
요청 컨텍스트가 필요한 콜백에만 적용하는 편이 좋습니다.
전역 이벤트나 요청과 무관한 백그라운드 작업에 무리하게 붙이면 로그가 오히려 헷갈릴 수 있습니다.

### H3. 외부 메시지 큐에서도 bind로 requestId를 유지할 수 있나요?

같은 Node.js 프로세스 안에서 함수가 나중에 실행되는 경우에만 적합합니다.
Redis, Kafka, SQS, 데이터베이스 큐처럼 프로세스 밖으로 넘어가는 작업은 `requestId`나 `traceId`를 payload에 명시적으로 저장하고, worker에서 새 컨텍스트를 시작해야 합니다.

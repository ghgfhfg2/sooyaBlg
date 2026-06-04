---
layout: post
title: "Node.js EventEmitterAsyncResource 가이드: 이벤트에서도 요청 컨텍스트를 유지하는 법"
date: 2026-06-05 08:00:00 +0900
lang: ko
translation_key: nodejs-eventemitterasyncresource-context-propagation-guide
permalink: /development/blog/seo/2026/06/05/nodejs-eventemitterasyncresource-context-propagation-guide.html
alternates:
  ko: /development/blog/seo/2026/06/05/nodejs-eventemitterasyncresource-context-propagation-guide.html
  x_default: /development/blog/seo/2026/06/05/nodejs-eventemitterasyncresource-context-propagation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, events, asyncresource, asynclocalstorage, observability, backend]
description: "Node.js EventEmitterAsyncResource로 EventEmitter 이벤트에서도 AsyncLocalStorage 요청 컨텍스트를 유지하는 방법을 정리했습니다. AsyncResource와의 차이, emitDestroy 정리, 관측 이벤트 설계와 테스트 포인트를 예제로 설명합니다."
---

Node.js에서 `EventEmitter`는 여전히 가장 익숙한 이벤트 인터페이스입니다.
작은 내부 모듈, 스트림 래퍼, 작업 큐, 플러그인 훅, 관측 이벤트까지 많은 코드가 `emit()`과 `on()`을 중심으로 움직입니다.
하지만 요청 단위 로그나 추적 ID를 `AsyncLocalStorage`로 관리하는 서비스에서는 이벤트 경계에서 컨텍스트가 끊기는 문제가 생길 수 있습니다.

특히 이벤트를 나중에 실행되는 콜백, 큐, 타이머, 외부 라이브러리 브리지와 연결하면 "이 리스너가 어떤 요청에서 출발했는가"가 모호해집니다.
로그에는 이벤트 이름만 남고 요청 ID가 빠지거나, 잘못된 컨텍스트가 섞이면 장애 분석이 어려워집니다.

Node.js의 `node:events` 모듈에는 `EventEmitterAsyncResource`가 있습니다.
이 클래스는 `EventEmitter` 동작과 `AsyncResource` 컨텍스트 경계를 함께 다루게 해 줍니다.
이 글에서는 `EventEmitterAsyncResource`가 필요한 상황, 기본 사용법, `AsyncLocalStorage`와 함께 쓰는 패턴, 수명 정리와 테스트 체크리스트를 정리합니다.
커스텀 비동기 경계 자체가 먼저 낯설다면 [Node.js AsyncResource 가이드](/development/blog/seo/2026/04/26/nodejs-asyncresource-request-context-background-job-guide.html)를 먼저 읽으면 흐름을 잡기 쉽습니다.

## EventEmitterAsyncResource가 필요한 이유

### H3. 이벤트도 비동기 경계가 될 수 있다

`EventEmitter`는 동기적으로 리스너를 호출하는 경우가 많습니다.
그래서 간단한 코드에서는 컨텍스트 문제가 잘 드러나지 않습니다.
하지만 이벤트를 발행하는 쪽과 리스너를 등록하는 쪽이 분리되고, 그 사이에 큐나 타이머가 들어가면 이야기가 달라집니다.

```js
import { EventEmitter } from 'node:events';

const bus = new EventEmitter();

export function publishLater(event) {
  setImmediate(() => {
    bus.emit('job.finished', event);
  });
}
```

이 코드는 작동은 하지만, 관측 컨텍스트 관점에서는 경계가 흐릿합니다.
`setImmediate()`로 넘어간 뒤에도 "이 이벤트는 어떤 비동기 작업의 일부인가"를 런타임에 명확히 알려 주지 않습니다.
`AsyncLocalStorage`를 쓰는 서비스에서는 이 차이가 로그 상관관계, 트레이스 연결, 테스트 안정성에 영향을 줄 수 있습니다.

`EventEmitterAsyncResource`는 이벤트 발행을 특정 async resource 범위 안에서 실행하게 해 줍니다.
즉 이벤트 리스너가 호출될 때도 그 이벤트 emitter가 대표하는 비동기 작업의 컨텍스트를 따라가도록 설계할 수 있습니다.

### H3. EventEmitter와 AsyncResource를 따로 붙이는 코드를 줄인다

직접 `AsyncResource`를 만들고 `runInAsyncScope()`로 리스너 호출을 감싸는 방법도 있습니다.
하지만 이벤트 emitter를 직접 구현하다 보면 아래 책임이 흩어지기 쉽습니다.

- 이벤트 등록과 발행
- 비동기 리소스 이름 지정
- 컨텍스트 스코프 적용
- 수명 종료 시 `emitDestroy()` 호출
- 에러 이벤트와 관측 이벤트 처리

`EventEmitterAsyncResource`는 이런 책임을 하나의 클래스 안에서 다루게 해 줍니다.
이벤트 기반 추상화를 이미 만들고 있다면, `EventEmitter`를 상속하는 대신 이 클래스를 기준으로 시작하는 편이 코드 의도를 더 분명하게 남길 수 있습니다.

## 기본 사용법

### H3. node:events에서 가져와 이벤트 클래스를 만든다

가장 단순한 형태는 `EventEmitterAsyncResource`를 상속한 클래스를 만드는 것입니다.
생성자에는 async resource 이름을 넘겨 리소스의 역할을 구분합니다.
이 이름은 진단과 디버깅에서 사람이 읽을 수 있어야 합니다.

```js
import { EventEmitterAsyncResource } from 'node:events';

export class JobEvents extends EventEmitterAsyncResource {
  constructor(jobId) {
    super({ name: 'JobEvents' });
    this.jobId = jobId;
  }

  finish(result) {
    this.emit('finished', {
      jobId: this.jobId,
      result
    });
  }

  fail(error) {
    this.emit('error', error);
  }
}
```

사용하는 쪽에서는 일반 `EventEmitter`처럼 `on()`과 `emit()` 흐름을 그대로 쓸 수 있습니다.
차이는 내부적으로 이벤트 리스너 호출이 async resource 스코프와 연결된다는 점입니다.

```js
const events = new JobEvents('job-42');

events.on('finished', (event) => {
  console.log('job finished:', event.jobId);
});

events.finish({ ok: true });
```

이벤트 이름과 payload는 작게 유지하는 편이 좋습니다.
특히 운영 로그나 관측 이벤트로 이어질 수 있는 payload에는 토큰, 쿠키, 원문 개인정보 같은 민감정보를 넣지 않아야 합니다.

### H3. AsyncLocalStorage 컨텍스트 안에서 생성 위치를 의식한다

`EventEmitterAsyncResource`를 사용할 때 중요한 지점은 인스턴스를 만드는 위치입니다.
어떤 요청 컨텍스트 안에서 emitter를 만들었는지에 따라 리스너 실행 시 이어질 컨텍스트가 달라질 수 있습니다.

```js
import { AsyncLocalStorage } from 'node:async_hooks';
import { EventEmitterAsyncResource } from 'node:events';

const requestContext = new AsyncLocalStorage();

class TaskEvents extends EventEmitterAsyncResource {
  constructor() {
    super({ name: 'TaskEvents' });
  }
}

export function startTask(requestId) {
  return requestContext.run({ requestId }, () => {
    const events = new TaskEvents();

    setImmediate(() => {
      events.emit('progress', { step: 'indexed' });
    });

    return events;
  });
}
```

리스너에서는 저장된 컨텍스트를 읽어 요청 ID를 로그에 붙일 수 있습니다.

```js
const events = startTask('req-123');

events.on('progress', (event) => {
  const context = requestContext.getStore();

  console.log({
    requestId: context?.requestId,
    step: event.step
  });
});
```

핵심은 emitter가 어떤 비동기 작업을 대표하는지 정하는 것입니다.
전역 singleton 이벤트 버스처럼 모든 요청이 공유하는 객체라면 요청별 컨텍스트를 대표하기 어렵습니다.
반대로 요청, 작업, 배치, 스트림처럼 수명이 있는 단위라면 `EventEmitterAsyncResource`가 잘 맞습니다.
요청 컨텍스트 설계 자체는 [Node.js AsyncLocalStorage 가이드](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)와 함께 보면 좋습니다.

## 실무 적용 패턴

### H3. 작업 단위 이벤트 emitter로 만든다

가장 현실적인 적용처는 작업 단위 emitter입니다.
예를 들어 파일 처리, 데이터 동기화, 백그라운드 잡, 외부 API 수집처럼 하나의 작업이 여러 상태 이벤트를 내보내는 경우입니다.

```js
import { EventEmitterAsyncResource } from 'node:events';

export class ImportJob extends EventEmitterAsyncResource {
  #closed = false;

  constructor({ jobId }) {
    super({ name: 'ImportJob' });
    this.jobId = jobId;
  }

  progress(count) {
    if (this.#closed) {
      return;
    }

    this.emit('progress', {
      jobId: this.jobId,
      count
    });
  }

  complete(summary) {
    if (this.#closed) {
      return;
    }

    this.emit('complete', {
      jobId: this.jobId,
      summary
    });
    this.close();
  }

  close() {
    if (this.#closed) {
      return;
    }

    this.#closed = true;
    this.emitDestroy();
  }
}
```

이 예제에서 `close()`는 작업의 수명 종료를 명시합니다.
`emitDestroy()`는 async resource가 끝났다는 신호를 런타임 진단 도구에 전달하는 역할을 합니다.
작업이 성공하든 실패하든 종료 경로에서 한 번만 호출되도록 작은 가드를 두는 편이 안전합니다.

### H3. 에러 이벤트는 일반 EventEmitter 규칙을 따른다

`EventEmitterAsyncResource`도 `EventEmitter` 계열이므로 `'error'` 이벤트 규칙을 그대로 생각해야 합니다.
에러 이벤트를 발행할 수 있다면 리스너를 등록하거나, 상위 흐름에서 Promise reject로 변환하는 기준을 둬야 합니다.

```js
export function runImport(input, context) {
  const job = new ImportJob({ jobId: context.jobId });

  job.on('error', (error) => {
    console.error('import failed', {
      jobId: context.jobId,
      message: error.message
    });
  });

  queueMicrotask(async () => {
    try {
      await processInput(input, job);
      job.complete({ status: 'ok' });
    } catch (error) {
      job.emit('error', error);
      job.close();
    }
  });

  return job;
}
```

에러 객체 전체를 로그에 그대로 넣기보다 필요한 필드만 골라 남기는 습관이 좋습니다.
외부 API 응답, SQL, 사용자 입력, 인증 헤더가 에러 메시지나 cause 안에 섞일 수 있기 때문입니다.
에러 전파 기준은 [Node.js error cause 가이드](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)와 함께 정리하면 운영 로그 품질이 좋아집니다.

### H3. 관측 이벤트와 기능 이벤트를 분리한다

이벤트 emitter가 생기면 모든 것을 이벤트로 흘리고 싶어질 수 있습니다.
하지만 기능 흐름과 관측 흐름은 구분하는 편이 유지보수에 유리합니다.

예를 들어 `progress`, `complete`, `error`는 작업 제어와 사용자 피드백에 필요한 기능 이벤트입니다.
반면 메트릭, 로그, 트레이싱을 위한 이벤트는 `diagnostics_channel` 같은 별도 통로로 발행하는 편이 좋습니다.

```js
import diagnosticsChannel from 'node:diagnostics_channel';

const importEvents = diagnosticsChannel.channel('app.import.job');

function publishImportMetric(job, state) {
  if (!importEvents.hasSubscribers) {
    return;
  }

  importEvents.publish({
    jobId: job.jobId,
    state,
    timestamp: Date.now()
  });
}
```

이렇게 나누면 애플리케이션 코드는 작업 상태 이벤트에 집중하고, 로그·메트릭 수집은 구독자 쪽에서 독립적으로 붙일 수 있습니다.
관측 채널 설계는 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/05/18/nodejs-diagnostics-channel-observability-guide.html)를 함께 참고하면 좋습니다.

## 테스트와 운영 체크리스트

### H3. 컨텍스트가 유지되는지 테스트한다

`EventEmitterAsyncResource`를 도입했다면 단순히 이벤트가 발생하는지만 보지 말고, 리스너 안에서 `AsyncLocalStorage` 값이 유지되는지도 테스트해야 합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { AsyncLocalStorage } from 'node:async_hooks';

const store = new AsyncLocalStorage();

test('task event keeps request context', async () => {
  const requestId = await store.run({ requestId: 'req-test' }, async () => {
    const events = new TaskEvents();

    return await new Promise((resolve) => {
      events.on('progress', () => {
        resolve(store.getStore()?.requestId);
      });

      setImmediate(() => {
        events.emit('progress');
        events.emitDestroy();
      });
    });
  });

  assert.equal(requestId, 'req-test');
});
```

이런 테스트는 이벤트 발행 방식이 나중에 타이머, 큐, 외부 콜백으로 바뀌어도 컨텍스트 계약을 지켜 줍니다.
테스트 러너 구조는 [Node.js test runner subtest 가이드](/development/blog/seo/2026/05/24/nodejs-test-runner-subtest-structure-guide.html)처럼 작은 단위로 나누면 유지보수가 쉽습니다.

### H3. 종료 경로를 한 번만 실행한다

수명이 있는 async resource는 종료 경로가 중요합니다.
성공, 실패, 취소, 타임아웃이 동시에 가까운 시점에 일어날 수 있기 때문입니다.
`close()` 같은 메서드를 만들고 내부에 idempotent 가드를 두면 `emitDestroy()`가 여러 번 호출되는 문제를 줄일 수 있습니다.

```js
class SafeJobEvents extends EventEmitterAsyncResource {
  #destroyed = false;

  constructor() {
    super({ name: 'SafeJobEvents' });
  }

  close() {
    if (this.#destroyed) {
      return;
    }

    this.#destroyed = true;
    this.removeAllListeners();
    this.emitDestroy();
  }
}
```

`removeAllListeners()`는 모든 상황에서 무조건 필요한 것은 아닙니다.
다만 작업 객체의 수명이 끝났고 더는 이벤트를 받을 이유가 없다면 참조를 빨리 끊는 데 도움이 됩니다.
공유 emitter에는 함부로 쓰면 다른 모듈의 리스너까지 제거할 수 있으니, 작업 전용 emitter에만 적용하는 편이 안전합니다.

## 언제 쓰지 않는 편이 나은가

### H3. 전역 이벤트 버스에는 잘 맞지 않는다

전역 singleton 이벤트 버스는 여러 요청과 작업이 함께 쓰는 경우가 많습니다.
그런 객체를 하나의 async resource처럼 다루면 오히려 어떤 컨텍스트를 대표하는지 애매해집니다.

전역 버스가 필요하다면 payload에 명시적인 `requestId`, `jobId`, `traceId`를 넣고, 민감정보를 제한하는 계약을 만드는 편이 더 단순할 수 있습니다.
요청별 또는 작업별로 emitter 인스턴스를 만들 수 있는 구조일 때 `EventEmitterAsyncResource`의 장점이 더 잘 드러납니다.

### H3. 단순 동기 이벤트에는 과한 선택일 수 있다

같은 함수 호출 안에서 바로 `emit()`하고 끝나는 작은 이벤트라면 일반 `EventEmitter`로 충분할 수 있습니다.
컨텍스트 문제가 없는 코드에 async resource를 추가하면 이해해야 할 수명 관리만 늘어납니다.

도입 기준은 아래처럼 잡으면 실무적으로 무리가 적습니다.

- 이벤트가 요청, 작업, 배치, 스트림 같은 수명 단위를 대표한다.
- 리스너에서 `AsyncLocalStorage` 기반 요청 ID나 trace 정보를 읽어야 한다.
- 이벤트 발행이 타이머, 큐, 콜백 브리지, 외부 라이브러리 경계를 지난다.
- 관측 도구에서 async resource 수명과 이벤트 흐름을 더 정확히 보고 싶다.

이 중 하나도 해당하지 않는다면 일반 `EventEmitter`가 더 단순합니다.

## 마무리

`EventEmitterAsyncResource`는 모든 이벤트 코드를 바꿔야 하는 API가 아닙니다.
하지만 이벤트가 하나의 비동기 작업을 대표하고, 리스너에서 요청 컨텍스트나 추적 정보를 안정적으로 유지해야 한다면 꽤 유용한 선택지입니다.

핵심은 세 가지입니다.
첫째, emitter 인스턴스를 요청이나 작업처럼 수명이 분명한 단위로 만든다.
둘째, 생성 위치와 `AsyncLocalStorage` 컨텍스트의 관계를 테스트로 확인한다.
셋째, 작업이 끝나면 `emitDestroy()`를 한 번만 호출하도록 종료 경로를 명시한다.

이 기준을 지키면 이벤트 기반 코드에서도 로그 상관관계와 관측 품질을 더 안정적으로 유지할 수 있습니다.

## 함께 보면 좋은 글

- [Node.js AsyncResource 가이드: 백그라운드 작업에서도 요청 컨텍스트를 이어가는 법](/development/blog/seo/2026/04/26/nodejs-asyncresource-request-context-background-job-guide.html)
- [Node.js AsyncLocalStorage 가이드: 요청 컨텍스트 로깅을 안전하게 설계하는 법](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)
- [Node.js diagnostics_channel 가이드: 라이브러리 관측 이벤트를 안전하게 발행하는 법](/development/blog/seo/2026/05/18/nodejs-diagnostics-channel-observability-guide.html)

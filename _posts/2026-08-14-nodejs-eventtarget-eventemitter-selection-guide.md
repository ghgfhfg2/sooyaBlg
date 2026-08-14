---
layout: post
title: "Node.js EventTarget 가이드: EventEmitter와 addEventListener 선택 기준"
date: 2026-08-14 20:00:00 +0900
lang: ko
translation_key: nodejs-eventtarget-eventemitter-selection-guide
permalink: /development/blog/seo/2026/08/14/nodejs-eventtarget-eventemitter-selection-guide.html
alternates:
  ko: /development/blog/seo/2026/08/14/nodejs-eventtarget-eventemitter-selection-guide.html
  x_default: /development/blog/seo/2026/08/14/nodejs-eventtarget-eventemitter-selection-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, eventtarget, eventemitter, events, addeventlistener, abortsignal, backend, javascript]
description: "Node.js EventTarget과 EventEmitter의 차이를 실무 기준으로 정리합니다. addEventListener, dispatchEvent, once 옵션, AbortSignal 정리, 에러 이벤트 처리, 브라우저 호환 API 설계까지 예제로 설명합니다."
---

Node.js에서 이벤트를 다룬다고 하면 가장 먼저 떠오르는 도구는 `EventEmitter`입니다.
스트림, 서버, 소켓, 내부 작업 큐처럼 Node.js 생태계에서 오래 쓰인 객체들은 보통 `on()`, `once()`, `emit()` 패턴을 따릅니다.
하지만 현대 Node.js 코드에는 `EventTarget`, `Event`, `addEventListener()`, `dispatchEvent()` 같은 웹 표준 계열 API도 함께 등장합니다.
대표적으로 `AbortSignal`은 `EventTarget` 기반 객체입니다.

둘은 비슷해 보이지만 같은 도구가 아닙니다.
`EventEmitter`는 Node.js 내부 서비스와 라이브러리 이벤트에 익숙하고, `EventTarget`은 브라우저와 공유되는 이벤트 모델을 따릅니다.
선택 기준 없이 섞어 쓰면 이벤트 이름, listener 정리, 에러 처리, 타입 설계가 흐트러질 수 있습니다.

이 글에서는 Node.js에서 `EventTarget`을 언제 쓰면 좋은지, 기존 `EventEmitter`와 어떤 점이 다른지, `AbortSignal`을 이용해 listener를 안전하게 정리하는 패턴을 정리합니다.
취소 listener 정리는 [Node.js events.addAbortListener 가이드](/development/blog/seo/2026/07/25/nodejs-events-addabortlistener-cleanup-guide.html), async iterator로 이벤트를 소비하는 방식은 [Node.js events.on/once 가이드](/development/blog/seo/2026/05/15/nodejs-events-on-once-async-iterator-guide.html)와 함께 보면 좋습니다.
스트림 완료 이벤트까지 다뤄야 한다면 [Node.js stream.finished 가이드](/development/blog/seo/2026/08/04/nodejs-stream-finished-completion-error-handling-guide.html)도 이어서 참고할 수 있습니다.

## EventTarget이 필요한 순간

### H3. 브라우저와 공유되는 API 표면을 만든다

`EventTarget`은 웹 표준 이벤트 모델과 같은 형태를 제공합니다.
서버 코드에서 브라우저와 비슷한 API를 노출하거나, 런타임을 넘나드는 라이브러리를 만들 때 유리합니다.
예를 들어 작업 상태를 알리는 작은 객체를 아래처럼 만들 수 있습니다.

```js
class JobProgress extends EventTarget {
  update(percent) {
    this.dispatchEvent(new CustomEvent('progress', {
      detail: { percent }
    }));
  }

  complete(result) {
    this.dispatchEvent(new CustomEvent('complete', {
      detail: { result }
    }));
  }
}

const progress = new JobProgress();

progress.addEventListener('progress', event => {
  console.log('progress', event.detail.percent);
});

progress.update(40);
```

이 구조는 `addEventListener()`와 `dispatchEvent()`를 사용하므로 브라우저 이벤트 모델에 익숙한 코드와 잘 맞습니다.
특히 프런트엔드와 백엔드에서 같은 도메인 객체를 테스트하거나, web worker와 Node.js worker 사이의 사고방식을 맞추고 싶을 때 읽기 쉽습니다.

다만 Node.js 내부에서만 쓰는 서비스 객체라면 `EventEmitter`가 더 자연스러운 경우도 많습니다.
기존 스트림, 서버, 프로세스 이벤트와 함께 다뤄야 한다면 팀이 이미 알고 있는 `EventEmitter` 패턴을 유지하는 편이 운영 비용이 낮습니다.

### H3. AbortSignal과 같은 모델로 listener 수명을 관리한다

`EventTarget`의 장점 중 하나는 `addEventListener()` 옵션입니다.
`once`, `signal` 같은 옵션을 사용하면 listener 수명을 호출부에서 명확히 표현할 수 있습니다.

```js
const controller = new AbortController();
const target = new EventTarget();

target.addEventListener('ready', event => {
  console.log('ready', event.type);
}, {
  once: true,
  signal: controller.signal
});

target.dispatchEvent(new Event('ready'));
target.dispatchEvent(new Event('ready')); // once 때문에 실행되지 않음

controller.abort();
```

`once: true`는 이벤트를 한 번만 처리하겠다는 뜻입니다.
`signal`은 상위 작업이 취소될 때 listener도 같이 정리하겠다는 뜻입니다.
이 두 옵션을 함께 쓰면 "언제 제거해야 하지?"라는 고민이 줄어듭니다.

서버에서는 listener 누수가 장애로 이어질 수 있습니다.
요청마다 listener를 붙이는데 제거하지 않으면 메모리 사용량이 천천히 늘고, 같은 이벤트가 여러 번 처리되는 문제가 생깁니다.
`EventTarget`을 쓴다면 가능하면 `once` 또는 `signal` 중 하나를 기본으로 고려하는 편이 좋습니다.

## EventEmitter와의 차이

### H3. emit과 dispatchEvent는 반환 의미가 다르다

`EventEmitter`는 `emit(name, ...args)`로 이벤트를 보냅니다.
이벤트 이름과 인자 목록을 자유롭게 정할 수 있습니다.

```js
import { EventEmitter } from 'node:events';

const emitter = new EventEmitter();

emitter.on('progress', (percent, label) => {
  console.log(percent, label);
});

emitter.emit('progress', 50, 'image-resize');
```

반면 `EventTarget`은 `dispatchEvent(event)`로 `Event` 객체를 보냅니다.
추가 데이터는 보통 `CustomEvent`의 `detail`에 담습니다.

```js
const target = new EventTarget();

target.addEventListener('progress', event => {
  console.log(event.detail.percent, event.detail.label);
});

target.dispatchEvent(new CustomEvent('progress', {
  detail: {
    percent: 50,
    label: 'image-resize'
  }
}));
```

`EventEmitter`는 Node.js 스타일의 다중 인자를 자연스럽게 지원합니다.
`EventTarget`은 이벤트 객체 하나를 중심으로 데이터를 전달합니다.
따라서 새 API를 설계할 때는 이벤트 payload를 객체로 고정할지, 기존 Node.js 관례처럼 인자를 나눌지 먼저 결정해야 합니다.

검색 가능한 문서와 장기 유지보수를 생각하면 payload 객체를 선호할 때가 많습니다.
나중에 필드를 추가해도 호출부 인자 순서를 깨뜨리지 않기 때문입니다.

### H3. error 이벤트 처리 방식이 다르다

`EventEmitter`에서는 `'error'` 이벤트가 특별합니다.
listener가 없는 상태에서 `'error'`를 emit하면 프로세스가 예외를 던질 수 있습니다.
그래서 Node.js 라이브러리에서 `EventEmitter`를 쓰면 에러 이벤트 정책을 반드시 문서화해야 합니다.

```js
import { EventEmitter } from 'node:events';

const emitter = new EventEmitter();

emitter.on('error', error => {
  console.error('job failed', error.message);
});

emitter.emit('error', new Error('resize failed'));
```

`EventTarget`의 `'error'`는 Node.js `EventEmitter`의 특수한 `'error'`와 같은 의미로 동작한다고 기대하면 안 됩니다.
그저 이름이 `error`인 이벤트로 설계하는 편이 안전합니다.
오류 객체를 전달하려면 명시적인 detail 구조를 두는 것이 좋습니다.

```js
class JobTarget extends EventTarget {
  fail(error) {
    this.dispatchEvent(new CustomEvent('joberror', {
      detail: {
        message: error.message,
        code: error.code ?? 'JOB_FAILED'
      }
    }));
  }
}
```

여기서는 이벤트 이름을 `'error'`가 아니라 `'joberror'`로 정했습니다.
Node.js 스타일 오류 이벤트와 혼동하지 않기 위해서입니다.
팀 컨벤션에 따라 `'error'`를 쓸 수도 있지만, 그 경우에도 `EventEmitter`의 에러 처리와 다르다는 점을 문서에 남기는 편이 좋습니다.

## 선택 기준

### H3. Node.js 내부 객체에는 EventEmitter가 여전히 편하다

아래 조건에 많이 해당하면 `EventEmitter`가 자연스럽습니다.

- Node.js 스트림, 서버, 소켓, child process와 함께 이벤트를 연결한다.
- 기존 코드가 `on()`, `once()`, `off()`, `emit()` 패턴으로 작성돼 있다.
- 이벤트 listener 개수, `listenerCount()`, `removeAllListeners()` 같은 Node.js 도구가 필요하다.
- Node.js 라이브러리 사용자가 기대하는 관례를 따르고 싶다.
- `'error'` 이벤트를 Node.js 방식으로 다루고 싶다.

예를 들어 파일 처리 queue를 Node.js 서버 안에서만 사용할 계획이라면 `EventEmitter`가 단순합니다.

```js
import { EventEmitter } from 'node:events';

class UploadQueue extends EventEmitter {
  push(file) {
    this.emit('queued', file.name);
  }

  finish(file, result) {
    this.emit('done', file.name, result);
  }
}
```

이 코드는 Node.js 개발자에게 익숙합니다.
특별히 브라우저 호환 API가 필요하지 않다면 굳이 `EventTarget`으로 바꿀 이유가 없습니다.

### H3. 공개 API와 취소 친화 listener에는 EventTarget이 잘 맞는다

아래 조건에 많이 해당하면 `EventTarget`을 검토할 만합니다.

- 브라우저와 Node.js에서 비슷한 API를 제공하고 싶다.
- `AbortSignal` 기반 listener 정리가 중요하다.
- 이벤트 payload를 하나의 객체로 고정하고 싶다.
- `addEventListener()`에 익숙한 사용자에게 API를 제공한다.
- DOM 또는 웹 표준 이벤트 모델과 개념을 맞추고 싶다.

예를 들어 SDK가 서버와 브라우저를 모두 지원한다면 `EventTarget` 기반 API가 더 일관적일 수 있습니다.

```js
export class SyncClient extends EventTarget {
  notifyState(state) {
    this.dispatchEvent(new CustomEvent('statechange', {
      detail: { state }
    }));
  }
}

const client = new SyncClient();
const controller = new AbortController();

client.addEventListener('statechange', event => {
  console.log(event.detail.state);
}, {
  signal: controller.signal
});
```

사용자는 상위 화면, 요청, 테스트 수명주기가 끝날 때 `controller.abort()`만 호출하면 listener를 정리할 수 있습니다.
이런 API는 이벤트를 붙이는 쪽의 실수를 줄이는 데 도움이 됩니다.

## 실무 설계 패턴

### H3. 이벤트 이름과 payload를 문서화한다

이벤트 API는 한 번 공개되면 바꾸기 어렵습니다.
따라서 이벤트 이름과 payload 모양을 코드 가까이에 정의하는 편이 좋습니다.

```js
export const JOB_EVENTS = {
  progress: 'progress',
  complete: 'complete',
  failed: 'failed'
};

export function dispatchJobEvent(target, type, detail) {
  target.dispatchEvent(new CustomEvent(type, { detail }));
}
```

이렇게 상수를 두면 문자열 오타를 줄일 수 있습니다.
TypeScript를 쓰는 프로젝트라면 이벤트별 `detail` 타입까지 묶어 두면 더 좋습니다.
JavaScript만 쓰더라도 테스트에서 이벤트 payload를 검증하면 API 변경을 빨리 알아챌 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('progress event includes percent', () => {
  const target = new EventTarget();
  let received;

  target.addEventListener('progress', event => {
    received = event.detail;
  }, { once: true });

  dispatchJobEvent(target, JOB_EVENTS.progress, { percent: 80 });

  assert.deepEqual(received, { percent: 80 });
});
```

테스트는 이벤트 이름과 payload 계약을 보호합니다.
특히 공개 SDK나 내부 공통 모듈처럼 여러 팀이 쓰는 코드라면 이런 작은 테스트가 회귀를 잘 잡아 줍니다.

### H3. listener 정리는 호출자 책임으로 드러낸다

이벤트 API의 가장 흔한 문제는 listener를 붙인 뒤 제거하지 않는 것입니다.
`EventTarget`에서는 `AbortController`를 호출자가 들고 있게 하면 수명 관리가 명확해집니다.

```js
export function subscribeProgress(target, onProgress, { signal } = {}) {
  target.addEventListener('progress', event => {
    onProgress(event.detail);
  }, { signal });
}

const controller = new AbortController();

subscribeProgress(job, detail => {
  console.log(detail.percent);
}, {
  signal: controller.signal
});

controller.abort();
```

이 함수는 unsubscribe 함수를 따로 반환하지 않습니다.
대신 호출자가 이미 가진 `AbortSignal`에 listener 수명을 연결합니다.
요청 단위, 화면 단위, 테스트 단위로 abort 흐름이 잡혀 있다면 이 방식이 깔끔합니다.

`EventEmitter`를 쓰는 코드라면 반대로 제거 함수를 반환하는 패턴이 읽기 쉽습니다.

```js
export function onProgress(emitter, listener) {
  emitter.on('progress', listener);

  return () => {
    emitter.off('progress', listener);
  };
}
```

중요한 점은 어떤 방식이든 listener 정리 경로가 API 표면에 보여야 한다는 것입니다.
문서에만 "나중에 제거하세요"라고 쓰면 실제 호출부에서 빠뜨리기 쉽습니다.

## 피해야 할 실수

### H3. 두 이벤트 모델을 한 객체에 무리하게 섞는다

하나의 객체가 `on()`, `emit()`, `addEventListener()`, `dispatchEvent()`를 모두 제공하면 처음에는 편해 보입니다.
하지만 시간이 지나면 어떤 listener가 어떤 규칙으로 실행되는지 헷갈립니다.
에러 이벤트, once 처리, listener 제거 방식도 모델마다 다릅니다.

기존 `EventEmitter` 객체를 웹 표준 API처럼 노출해야 한다면 얇은 adapter를 별도로 두는 편이 낫습니다.

```js
export function eventEmitterToTarget(emitter, eventNames) {
  const target = new EventTarget();

  for (const name of eventNames) {
    emitter.on(name, (...args) => {
      target.dispatchEvent(new CustomEvent(name, {
        detail: { args }
      }));
    });
  }

  return target;
}
```

이 adapter도 장기적으로는 cleanup 문제가 생길 수 있으므로, 실제 운영 코드에서는 연결 해제 함수나 `AbortSignal`을 함께 설계해야 합니다.
핵심은 원본 객체의 이벤트 모델을 흐리지 않고, 경계에서만 변환하는 것입니다.

### H3. 이벤트를 제어 흐름의 전부로 만들지 않는다

이벤트는 상태 변화를 알리는 데 좋지만, 모든 제어 흐름을 이벤트로만 만들면 추적이 어려워집니다.
특히 성공, 실패, 취소가 모두 이벤트로만 흘러가면 호출자는 언제 작업이 끝났는지 Promise로 기다리기 어렵습니다.

작업 완료 자체는 Promise로 표현하고, 중간 상태만 이벤트로 알리는 구조가 더 읽기 쉬울 때가 많습니다.

```js
export async function runJob(job, target) {
  target.dispatchEvent(new CustomEvent('progress', {
    detail: { percent: 10 }
  }));

  const result = await job.execute();

  target.dispatchEvent(new CustomEvent('progress', {
    detail: { percent: 100 }
  }));

  return result;
}
```

이 구조에서는 `await runJob()`으로 완료를 기다릴 수 있고, progress UI는 이벤트를 구독할 수 있습니다.
이벤트와 Promise의 책임이 분리되어 디버깅하기도 쉽습니다.

## 마무리

Node.js에서 `EventTarget`은 `EventEmitter`를 대체하는 도구라기보다, 웹 표준 이벤트 모델이 필요한 곳에 쓰는 별도의 선택지입니다.
Node.js 내부 객체와 생태계 관례가 중요하면 `EventEmitter`가 여전히 편합니다.
브라우저와 공유되는 API, `AbortSignal` 기반 listener 정리, 객체형 payload 계약이 중요하면 `EventTarget`이 더 잘 맞습니다.

새 이벤트 API를 만들 때는 먼저 사용자가 어떤 모델을 기대할지 정하세요.
그다음 이벤트 이름, payload, 에러 처리, listener 정리 방식을 코드와 테스트에 남기면 장기적으로 다루기 쉬운 이벤트 인터페이스가 됩니다.

## 관련 글

- [Node.js events.addAbortListener 가이드: AbortSignal listener를 안전하게 정리하는 법](/development/blog/seo/2026/07/25/nodejs-events-addabortlistener-cleanup-guide.html)
- [Node.js events.on/once 가이드: 이벤트를 Promise와 async iterator로 다루는 법](/development/blog/seo/2026/05/15/nodejs-events-on-once-async-iterator-guide.html)
- [Node.js stream.finished 가이드: 스트림 완료와 에러를 안전하게 감지하는 법](/development/blog/seo/2026/08/04/nodejs-stream-finished-completion-error-handling-guide.html)

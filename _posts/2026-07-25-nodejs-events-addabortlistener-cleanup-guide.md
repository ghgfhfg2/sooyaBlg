---
layout: post
title: "Node.js addAbortListener 가이드: AbortSignal 리스너를 안전하게 등록하고 정리하는 법"
date: 2026-07-25 20:00:00 +0900
lang: ko
translation_key: nodejs-events-addabortlistener-cleanup-guide
permalink: /development/blog/seo/2026/07/25/nodejs-events-addabortlistener-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/07/25/nodejs-events-addabortlistener-cleanup-guide.html
  x_default: /development/blog/seo/2026/07/25/nodejs-events-addabortlistener-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, abortsignal, addabortlistener, events, cleanup, timeout, backend, reliability]
description: "Node.js events.addAbortListener()로 AbortSignal 리스너를 안전하게 등록하고, 작업 완료 후 dispose로 정리하는 패턴을 정리합니다. timeout, 요청 취소, 리스너 누수 방지 기준을 실무 예제로 설명합니다."
---

Node.js에서 `AbortSignal`은 fetch, stream, child process, timer처럼 오래 걸릴 수 있는 작업을 중단하는 공통 언어가 됐습니다.
문제는 signal을 받는 함수를 직접 만들 때입니다.
`signal.addEventListener('abort', ...)`를 바로 붙이면 짧아 보이지만, 작업이 정상 완료됐을 때 리스너를 제거하지 않거나 이미 취소된 signal을 놓치는 코드가 쉽게 생깁니다.

이 글에서는 Node.js `node:events`의 `addAbortListener()`를 사용해 AbortSignal 리스너를 안전하게 등록하고 정리하는 방법을 정리합니다.
핵심은 취소 가능 작업을 만들 때 **abort 처리와 cleanup 책임을 같은 범위에 두는 것**입니다.
[Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html), [Node.js scheduler.wait 가이드](/development/blog/seo/2026/07/21/nodejs-timers-promises-scheduler-wait-cancellable-delay-guide.html), [Node.js stream pipeline AbortSignal 가이드](/development/blog/seo/2026/07/24/nodejs-stream-pipeline-abortsignal-cleanup-guide.html)와 함께 보면 요청 취소 흐름을 더 일관되게 설계할 수 있습니다.

## addAbortListener가 필요한 이유

### H3. abort 이벤트 리스너는 작업보다 오래 남을 수 있다

AbortSignal을 받는 유틸을 만들 때 흔히 다음처럼 작성합니다.

```js
export function waitForJob(job, { signal } = {}) {
  return new Promise((resolve, reject) => {
    signal?.addEventListener('abort', () => {
      reject(signal.reason);
    });

    job.once('done', resolve);
    job.once('error', reject);
  });
}
```

이 코드는 기본 아이디어는 맞지만 운영 코드로는 부족합니다.
작업이 정상 완료돼도 abort 리스너가 signal에 남아 있을 수 있고, 같은 signal을 여러 작업에 넘기는 구조에서는 불필요한 리스너가 쌓입니다.
또한 이미 취소된 signal이 들어왔을 때 작업을 시작하지 않아야 하는데, 이 흐름도 따로 챙겨야 합니다.

Node.js의 `addAbortListener(signal, listener)`는 AbortSignal 전용 리스너를 등록하고, 나중에 해제할 수 있는 disposable 객체를 돌려줍니다.
따라서 작업이 끝났을 때 `dispose`를 호출하는 형태로 cleanup을 명시할 수 있습니다.

### H3. once 옵션만으로는 cleanup 기준이 충분하지 않다

`addEventListener('abort', listener, { once: true })`를 쓰면 abort가 발생했을 때 리스너가 한 번만 실행됩니다.
하지만 작업이 abort보다 먼저 끝나는 경우에는 리스너가 실행되지 않습니다.
즉, 성공 경로에서 직접 정리하지 않으면 리스너 수명은 signal 수명에 묶입니다.

작업 단위 유틸에서는 "abort가 오면 중단한다"와 "작업이 끝나면 abort 리스너를 제거한다"가 둘 다 필요합니다.
`addAbortListener()`는 이 두 번째 책임을 코드에 드러내기 좋습니다.

## 기본 패턴

### H3. 이미 취소된 signal은 먼저 거절한다

중단 가능한 함수를 만들 때 첫 단계는 시작 전 상태 확인입니다.
이미 취소된 signal을 받았다면 타이머, 파일 작업, 외부 요청을 시작하지 않는 편이 안전합니다.

```js
import { addAbortListener } from 'node:events';

export function throwIfAborted(signal) {
  if (signal?.aborted) {
    throw signal.reason ?? new DOMException('The operation was aborted', 'AbortError');
  }
}
```

이 작은 헬퍼는 작업 시작 전에 호출합니다.
이렇게 하면 "이미 취소됐는데도 내부 리소스를 만든 뒤 곧바로 닫는" 흐름을 줄일 수 있습니다.

### H3. disposable을 finally에서 정리한다

다음은 이벤트 기반 작업을 Promise로 감싸는 기본형입니다.
`addAbortListener()`가 돌려준 disposable을 `finally`에서 정리하는 점이 핵심입니다.

```js
import { addAbortListener } from 'node:events';

export function waitForReady(emitter, { signal } = {}) {
  if (signal?.aborted) {
    return Promise.reject(signal.reason);
  }

  return new Promise((resolve, reject) => {
    let abortDisposable;

    const cleanup = () => {
      emitter.off('ready', onReady);
      emitter.off('error', onError);
      abortDisposable?.[Symbol.dispose]?.();
    };

    const onReady = (value) => {
      cleanup();
      resolve(value);
    };

    const onError = (error) => {
      cleanup();
      reject(error);
    };

    emitter.once('ready', onReady);
    emitter.once('error', onError);

    if (signal) {
      abortDisposable = addAbortListener(signal, () => {
        cleanup();
        reject(signal.reason);
      });
    }
  });
}
```

이 예시는 abort, 성공, 실패 중 어떤 경로가 먼저 오더라도 같은 cleanup 함수를 호출합니다.
실무에서는 cleanup이 여러 번 호출될 수 있으므로, 필요하다면 `done` 플래그를 두어 중복 resolve나 중복 reject를 막습니다.

## timeout과 호출자 취소 조합

### H3. AbortSignal.any로 상위 signal과 timeout을 합친다

대부분의 작업은 호출자 취소와 내부 timeout을 동시에 가져야 합니다.
API 요청이 끊기면 멈춰야 하고, 요청이 살아 있어도 내부 작업이 너무 오래 걸리면 끊어야 하기 때문입니다.

```js
function composeSignal({ signal, timeoutMs }) {
  const timeoutSignal = AbortSignal.timeout(timeoutMs);

  if (!signal) {
    return timeoutSignal;
  }

  return AbortSignal.any([signal, timeoutSignal]);
}

export async function waitForReadyWithTimeout(emitter, {
  signal,
  timeoutMs = 5_000
} = {}) {
  const runSignal = composeSignal({ signal, timeoutMs });
  return waitForReady(emitter, { signal: runSignal });
}
```

이 구조에서는 하위 함수가 어떤 signal이 호출자 취소인지, 어떤 signal이 timeout인지 몰라도 됩니다.
하위 함수는 signal이 abort되면 작업을 정리하고 종료하는 책임만 가집니다.
상위 계층은 timeout 값과 전체 deadline을 관리합니다.

### H3. timeout 원인은 로그에서 별도로 남긴다

AbortSignal을 합치면 하위 함수의 코드는 단순해지지만, 운영 로그에서는 원인을 분리해야 합니다.
사용자 취소, 서버 shutdown, 내부 timeout은 대응 방식이 다릅니다.

```js
function classifyAbort({ signal, timeoutMs }) {
  if (!signal?.aborted) {
    return 'not_aborted';
  }

  if (signal.reason?.name === 'TimeoutError') {
    return `timeout_after_${timeoutMs}ms`;
  }

  if (signal.reason?.name === 'AbortError') {
    return 'caller_aborted';
  }

  return 'aborted';
}
```

로그에는 작업 이름, timeout, 요청 ID 정도를 남기면 충분합니다.
사용자 입력, 토큰, 내부 파일 경로를 그대로 남기면 민감정보가 섞일 수 있으므로 공개 로그와 사용자 응답에서는 필요한 범위만 요약해야 합니다.

## 이벤트 기반 API에 적용하기

### H3. 외부 SDK 콜백을 감쌀 때 취소 경계를 만든다

일부 라이브러리는 Promise API 대신 이벤트나 콜백으로 완료를 알려줍니다.
이런 SDK를 API 서버 안에서 사용할 때는 요청 취소와 timeout을 SDK 호출 경계에 연결해야 합니다.

```js
export function waitForUploadComplete(upload, { signal } = {}) {
  if (signal?.aborted) {
    return Promise.reject(signal.reason);
  }

  return new Promise((resolve, reject) => {
    let settled = false;
    let abortDisposable;

    const finish = (fn, value) => {
      if (settled) {
        return;
      }

      settled = true;
      upload.off('complete', onComplete);
      upload.off('failed', onFailed);
      abortDisposable?.[Symbol.dispose]?.();
      fn(value);
    };

    const onComplete = (result) => finish(resolve, result);
    const onFailed = (error) => finish(reject, error);

    upload.once('complete', onComplete);
    upload.once('failed', onFailed);

    if (signal) {
      abortDisposable = addAbortListener(signal, () => {
        upload.cancel?.();
        finish(reject, signal.reason);
      });
    }
  });
}
```

여기서는 abort가 오면 SDK가 제공하는 `cancel()`도 호출합니다.
단순히 Promise만 reject하면 내부 업로드가 계속 진행될 수 있기 때문입니다.
취소 가능한 외부 API를 감쌀 때는 "내 Promise를 끝낸다"와 "실제 작업을 멈춘다"를 분리해서 확인해야 합니다.

### H3. cleanup은 성공 경로에서도 테스트한다

취소 로직은 실패할 때만 보는 코드가 아닙니다.
정상 완료됐을 때 리스너가 제거되는지 테스트해야 누수를 줄일 수 있습니다.
간단한 단위 테스트에서는 fake emitter와 AbortController를 사용해 완료 후 abort를 호출해도 추가 reject가 발생하지 않는지 확인할 수 있습니다.

```js
import { EventEmitter } from 'node:events';
import test from 'node:test';
import assert from 'node:assert/strict';

test('waitForReady resolves once and ignores later abort', async () => {
  const emitter = new EventEmitter();
  const controller = new AbortController();

  const promise = waitForReady(emitter, { signal: controller.signal });

  emitter.emit('ready', { ok: true });
  controller.abort();

  await assert.doesNotReject(promise);
});
```

이 테스트는 리스너 제거의 모든 세부 구현을 검사하지 않습니다.
대신 사용자가 기대하는 동작, 즉 완료된 작업이 나중의 abort에 흔들리지 않는다는 계약을 확인합니다.

## 운영 체크리스트

### H3. AbortSignal을 받는 함수의 기준

중단 가능한 유틸을 만들 때는 다음 기준을 함께 적용하면 좋습니다.

- 시작 전에 `signal.aborted`를 확인한다.
- abort 리스너는 성공, 실패, 취소 경로에서 모두 정리한다.
- timeout과 호출자 취소를 상위 계층에서 조합한다.
- Promise reject만 하지 말고 실제 내부 작업도 멈춘다.
- 로그에는 취소 원인을 남기되 민감정보는 마스킹한다.

이 기준은 작은 helper 함수에도 적용할 가치가 있습니다.
한두 곳에서는 차이가 작아 보여도, API 핸들러와 작업 큐가 늘어나면 리스너 수명과 취소 정책이 운영 안정성에 직접 영향을 줍니다.

### H3. 직접 addEventListener를 써도 되는 경우

모든 코드가 `addAbortListener()`를 써야 하는 것은 아닙니다.
브라우저와 공유하는 유틸이거나, AbortSignal을 지원하지 않는 오래된 런타임까지 고려해야 한다면 `addEventListener()` 기반 fallback이 필요할 수 있습니다.
다만 Node.js 전용 서버 코드에서는 disposable을 돌려주는 API를 사용하면 cleanup 의도가 더 분명합니다.

중요한 것은 API 이름보다 수명 관리입니다.
작업이 끝났는데도 signal 리스너가 남아 있지 않은지, 이미 취소된 signal에서 작업을 시작하지 않는지, abort가 실제 리소스 정리로 이어지는지 확인해야 합니다.

## 마무리

`AbortSignal`은 단순한 timeout 옵션이 아니라 작업 수명을 표현하는 계약입니다.
`addAbortListener()`는 그 계약을 이벤트 기반 코드에 붙일 때 리스너 등록과 해제를 명확하게 만드는 도구입니다.

Node.js 서버에서 취소 가능한 helper를 직접 만든다면 `signal.aborted` 확인, disposable 정리, timeout 조합, 실제 작업 중단을 한 묶음으로 설계하세요.
이렇게 해 두면 요청 취소, 배치 timeout, graceful shutdown 같은 운영 흐름이 같은 방식으로 동작합니다.

## 관련 글

- [Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js scheduler.wait 가이드](/development/blog/seo/2026/07/21/nodejs-timers-promises-scheduler-wait-cancellable-delay-guide.html)
- [Node.js stream pipeline AbortSignal 가이드](/development/blog/seo/2026/07/24/nodejs-stream-pipeline-abortsignal-cleanup-guide.html)

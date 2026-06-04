---
layout: post
title: "Node.js events.addAbortListener 가이드: AbortSignal 리스너를 안전하게 정리하는 법"
date: 2026-06-04 20:00:00 +0900
lang: ko
translation_key: nodejs-events-addabortlistener-cleanup-guide
permalink: /development/blog/seo/2026/06/04/nodejs-events-addabortlistener-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/06/04/nodejs-events-addabortlistener-cleanup-guide.html
  x_default: /development/blog/seo/2026/06/04/nodejs-events-addabortlistener-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, events, abortsignal, cancellation, cleanup, backend]
description: "Node.js events.addAbortListener로 AbortSignal 취소 리스너를 안전하게 등록하고 정리하는 방법을 정리했습니다. addEventListener와의 차이, Symbol.dispose 사용법, 테스트·타임아웃·서버 종료 패턴까지 예제로 설명합니다."
---

Node.js에서 취소 가능한 작업을 만들 때 `AbortSignal`은 사실상 표준 인터페이스가 됐습니다.
`fetch`, `events.once()`, `events.on()`, 파일 읽기, 긴 반복 작업, 테스트 타임아웃까지 여러 API가 `signal`을 받습니다.
하지만 직접 `signal.addEventListener('abort', listener)`를 붙이는 코드에서는 리스너 정리를 빠뜨리기 쉽습니다.

작업은 이미 끝났는데 abort 리스너가 남아 있으면 불필요한 참조가 유지되고, 테스트에서는 다음 케이스에 영향을 줄 수 있습니다.
반대로 이미 abort된 signal을 뒤늦게 받았을 때 처리 흐름이 애매해지는 경우도 있습니다.

Node.js의 `node:events` 모듈에는 `addAbortListener()`가 있습니다.
이 API는 `AbortSignal`에 abort 리스너를 등록하고, 나중에 정리할 수 있는 disposable 객체를 돌려줍니다.
이 글에서는 `events.addAbortListener()`의 기본 사용법, `addEventListener()`와의 차이, 실무에서 취소 리스너를 안전하게 정리하는 패턴을 정리합니다.
취소 신호를 여러 조건으로 조합하는 흐름이 먼저 필요하다면 [Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)를 함께 참고하세요.

## addAbortListener가 필요한 이유

### H3. abort 리스너도 자원이다

취소 가능한 함수를 만들 때 가장 흔한 코드는 아래처럼 생겼습니다.

```js
export function waitForWork(signal) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => resolve('done'), 1000);

    signal.addEventListener('abort', () => {
      clearTimeout(timer);
      reject(signal.reason);
    });
  });
}
```

처음 보기에는 괜찮아 보이지만, 작업이 정상 완료됐을 때 abort 리스너가 제거되지 않습니다.
짧은 스크립트에서는 티가 나지 않을 수 있지만, 서버처럼 같은 함수를 반복 호출하는 환경에서는 리스너와 클로저가 오래 남을 수 있습니다.
테스트에서도 완료된 작업의 리스너가 뒤늦게 실행되며 디버깅을 어렵게 만들 수 있습니다.

`addAbortListener()`는 이 정리 단계를 더 명시적으로 표현하게 해 줍니다.
등록 결과로 받은 disposable을 작업 종료 경로에서 dispose하면 됩니다.

### H3. 이미 취소된 signal도 같은 경로로 다룬다

취소 가능한 함수는 signal이 이미 abort된 상태로 호출될 수 있습니다.
예를 들어 상위 요청이 취소된 뒤 하위 작업을 시작하려 하거나, 테스트에서 `AbortSignal.timeout(0)` 같은 값을 넘기는 경우입니다.

이때 함수 초반에 `signal.throwIfAborted()`를 호출하면 불필요한 작업을 시작하지 않을 수 있습니다.
그다음 실제 작업 중 취소를 받아야 한다면 `addAbortListener()`로 리스너를 붙입니다.

```js
import { addAbortListener } from 'node:events';

export async function runCancellableTask({ signal }) {
  signal?.throwIfAborted();

  const disposable = signal
    ? addAbortListener(signal, () => {
        console.log('task was aborted');
      })
    : null;

  try {
    await doWork(signal);
  } finally {
    disposable?.[Symbol.dispose]();
  }
}
```

핵심은 시작 전 체크와 실행 중 리스너를 구분하는 것입니다.
시작 전에는 `throwIfAborted()`로 빠르게 실패시키고, 실행 중에는 `addAbortListener()`로 정리 가능한 리스너를 붙입니다.
취소 체크포인트 자체는 [Node.js AbortSignal.throwIfAborted 가이드](/development/blog/seo/2026/05/06/nodejs-abortsignal-throwifaborted-cancellation-checkpoint-guide.html)와 같은 원칙입니다.

## 기본 사용법

### H3. 반환된 disposable을 finally에서 정리한다

`addAbortListener(signal, listener)`는 abort가 발생했을 때 실행할 listener를 등록합니다.
그리고 리스너를 제거할 수 있는 disposable 객체를 반환합니다.
작업이 성공하든 실패하든 정리되어야 하므로 보통 `finally`에서 dispose합니다.
Promise 안에서는 타이머가 resolve되거나 abort가 reject하더라도 한 번만 정리되게 만드는 것이 핵심입니다.

```js
import { addAbortListener } from 'node:events';

export function delay(ms, { signal } = {}) {
  signal?.throwIfAborted();

  return new Promise((resolve, reject) => {
    let settled = false;
    let disposable;

    const finish = (callback, value) => {
      if (settled) {
        return;
      }

      settled = true;
      clearTimeout(timer);
      disposable?.[Symbol.dispose]();
      callback(value);
    };

    const timer = setTimeout(() => {
      finish(resolve);
    }, ms);

    if (signal) {
      disposable = addAbortListener(signal, () => {
        finish(reject, signal.reason);
      });
    }
  });
}
```

이 패턴은 단순하지만 실무에서 중요합니다.
성공, 실패, 취소가 서로 경쟁할 수 있는 코드에서는 정리 함수가 여러 번 호출될 수 있기 때문입니다.
`settled` 같은 작은 가드가 있으면 타이머와 리스너 정리가 예측 가능해집니다.

### H3. using을 쓸 수 있는 환경에서는 범위를 더 좁힌다

Node.js의 명시적 리소스 관리 문법을 사용하는 환경이라면 disposable 객체를 `using`과 함께 다룰 수 있습니다.
다만 프로젝트의 Node.js 버전과 빌드 도구가 해당 문법을 지원하는지 먼저 확인해야 합니다.

```js
import { addAbortListener } from 'node:events';

export async function runWithAbortLog(signal) {
  signal.throwIfAborted();

  using abortListener = addAbortListener(signal, () => {
    console.log('operation aborted');
  });

  await doWork(signal);
}
```

`using`은 블록을 벗어날 때 dispose를 호출합니다.
지원 환경이 확실하다면 `try/finally`보다 정리 범위가 잘 보입니다.
반대로 여러 Node.js 버전을 지원하는 라이브러리라면 아직은 `try/finally`를 쓰는 편이 더 호환성이 좋을 수 있습니다.
리소스 정리 문법 자체는 [Node.js using disposable 가이드](/development/blog/seo/2026/05/02/nodejs-using-disposable-and-asyncdisposable-guide.html)를 함께 보면 좋습니다.

## addEventListener와 비교하기

### H3. addEventListener도 쓸 수 있지만 정리 책임이 흩어진다

`AbortSignal`은 EventTarget이므로 아래처럼 직접 리스너를 붙일 수도 있습니다.

```js
function attachAbort(signal, onAbort) {
  signal.addEventListener('abort', onAbort, { once: true });

  return () => {
    signal.removeEventListener('abort', onAbort);
  };
}
```

이 방식도 틀린 것은 아닙니다.
다만 매번 cleanup 함수를 직접 만들어야 하고, `{ once: true }`를 빠뜨리거나 `removeEventListener()`에 같은 함수 참조를 넘기지 못하는 실수가 생길 수 있습니다.

`addAbortListener()`는 "취소 리스너 등록"이라는 의도를 더 직접적으로 드러냅니다.
반환값도 disposable이라서 다른 리소스 정리 패턴과 맞추기 쉽습니다.

### H3. 취소 리스너 안에서는 오래 걸리는 작업을 하지 않는다

abort 리스너는 정리와 알림을 위한 경로로 보는 편이 안전합니다.
리스너 안에서 긴 비동기 작업을 시작하거나, 외부 API 호출을 하거나, 복잡한 상태 변경을 많이 넣으면 취소 흐름 자체가 무거워집니다.

좋은 abort 리스너는 대체로 아래 일만 합니다.

- 타이머 정리
- pending Promise reject
- 스트림 destroy 또는 reader cancel
- 내부 플래그 변경
- 짧은 진단 로그 기록

실제 후처리는 호출자나 상위 에러 처리 경로에서 담당하게 두는 편이 좋습니다.
취소는 기능 로직이 아니라 흐름 제어 신호라는 점을 유지해야 코드가 단순해집니다.

## 실무 적용 패턴

### H3. 외부 작업 래퍼에 취소 정리를 넣는다

직접 만든 비동기 함수가 `signal`을 받는다면, 작업 시작 전과 종료 후를 한 곳에서 관리하는 래퍼를 만들 수 있습니다.

```js
import { addAbortListener } from 'node:events';

export async function withAbortCleanup(signal, work) {
  signal?.throwIfAborted();

  let aborted = false;
  const disposable = signal
    ? addAbortListener(signal, () => {
        aborted = true;
      })
    : null;

  try {
    const result = await work(() => aborted);

    if (aborted) {
      throw signal.reason;
    }

    return result;
  } finally {
    disposable?.[Symbol.dispose]();
  }
}
```

이 예제는 하위 라이브러리가 `AbortSignal`을 직접 지원하지 않을 때 쓸 수 있는 완충 장치입니다.
다만 `aborted` 플래그만으로는 이미 실행 중인 I/O를 강제로 중단할 수 없습니다.
가능하면 `fetch`, 스트림, 데이터베이스 클라이언트처럼 하위 API가 signal을 직접 받도록 연결하는 것이 더 좋습니다.
느린 외부 API를 끊는 기준은 [Node.js fetch timeout 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)와 연결됩니다.

### H3. 테스트에서는 리스너 누수를 실패로 본다

테스트 코드에서 `AbortSignal`을 쓰면 타임아웃을 간단히 만들 수 있습니다.
하지만 테스트가 끝난 뒤 리스너가 남으면 다음 테스트에 영향을 줄 수 있습니다.
그래서 테스트 유틸리티에서도 `addAbortListener()`와 `finally` 정리를 습관화하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { addAbortListener } from 'node:events';
import test from 'node:test';

test('task stops when aborted', async () => {
  const controller = new AbortController();
  let stopped = false;

  const disposable = addAbortListener(controller.signal, () => {
    stopped = true;
  });

  try {
    controller.abort(new Error('test timeout'));

    assert.equal(stopped, true);
  } finally {
    disposable[Symbol.dispose]();
  }
});
```

작은 테스트에서는 과해 보일 수 있습니다.
하지만 공용 테스트 헬퍼나 반복 실행되는 이벤트 테스트에서는 정리 습관이 실패 원인을 크게 줄여 줍니다.
이벤트를 Promise와 async iterator로 다루는 테스트는 [Node.js events.once/on 가이드](/development/blog/seo/2026/05/15/nodejs-events-on-once-async-iterator-guide.html)와 함께 보면 구조를 잡기 쉽습니다.

## 발행 전 체크리스트

### H3. AbortSignal 리스너를 붙이기 전에 확인할 항목

- 함수 시작 시 이미 abort된 signal을 확인하는가?
- 작업 정상 완료 시 abort 리스너를 제거하는가?
- 실패와 취소 경로에서도 같은 정리 함수가 호출되는가?
- abort 리스너 안에 오래 걸리는 작업을 넣지 않았는가?
- 하위 API가 `signal`을 직접 지원한다면 그대로 전달하는가?
- 테스트에서 타임아웃 signal이 다음 테스트로 새지 않는가?
- 로그에 사용자 토큰, 세션 ID, 원본 요청 바디 같은 민감정보가 섞이지 않았는가?

## FAQ

### H3. addAbortListener는 addEventListener보다 항상 좋은가요?

항상은 아닙니다.
일반 EventTarget 이벤트를 다룰 때는 `addEventListener()`가 자연스럽습니다.
하지만 `AbortSignal`의 abort 이벤트만 다루고, 등록과 정리를 명확히 표현하고 싶다면 `addAbortListener()`가 더 읽기 좋습니다.

### H3. signal.throwIfAborted만 쓰면 충분한가요?

짧은 동기 작업이라면 충분할 수 있습니다.
하지만 작업이 비동기로 오래 지속되거나, 중간에 타이머·스트림·외부 요청을 정리해야 한다면 실행 중 abort를 받을 리스너가 필요합니다.
이때 `throwIfAborted()`는 시작 전 체크, `addAbortListener()`는 실행 중 정리로 나눠 생각하면 됩니다.

### H3. abort 리스너에서 reject하면 에러 이름은 어떻게 처리해야 하나요?

가능하면 `signal.reason`을 그대로 전달하세요.
상위에서 `AbortError`, `TimeoutError`, 사용자 취소 사유를 구분할 수 있기 때문입니다.
직접 새 에러를 만들 때는 `{ cause: signal.reason }`처럼 원인을 보존하는 편이 디버깅에 좋습니다.

## 정리

`events.addAbortListener()`는 거창한 API가 아니라, `AbortSignal` 리스너를 등록하고 정리하는 책임을 분명하게 만드는 작은 도구입니다.
취소 가능한 코드는 성공, 실패, timeout, 사용자 취소가 서로 경쟁하는 경우가 많습니다.
그래서 리스너 정리를 우연에 맡기지 않고, disposable을 `finally`에서 처리하는 습관이 중요합니다.

실무에서는 함수 시작점에 `signal.throwIfAborted()`를 두고, 실행 중 정리가 필요한 구간에는 `addAbortListener()`를 붙이세요.
그리고 하위 API가 signal을 직접 지원한다면 래퍼만 만들지 말고 실제 호출에도 signal을 전달해야 취소가 끝까지 전파됩니다.

## 함께 읽기

- [Node.js AbortSignal.any 가이드: timeout과 사용자 취소를 함께 처리하는 법](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js AbortSignal.throwIfAborted 가이드: 취소 체크포인트를 안전하게 넣는 법](/development/blog/seo/2026/05/06/nodejs-abortsignal-throwifaborted-cancellation-checkpoint-guide.html)
- [Node.js events.once/on 가이드: EventEmitter를 Promise와 async iterator로 다루는 법](/development/blog/seo/2026/05/15/nodejs-events-on-once-async-iterator-guide.html)

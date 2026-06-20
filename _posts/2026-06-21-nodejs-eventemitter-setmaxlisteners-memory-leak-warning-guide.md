---
layout: post
title: "Node.js EventEmitter setMaxListeners 가이드: 메모리 누수 경고를 숨기지 않고 해결하기"
date: 2026-06-21 08:00:00 +0900
lang: ko
translation_key: nodejs-eventemitter-setmaxlisteners-memory-leak-warning-guide
permalink: /development/blog/seo/2026/06/21/nodejs-eventemitter-setmaxlisteners-memory-leak-warning-guide.html
alternates:
  ko: /development/blog/seo/2026/06/21/nodejs-eventemitter-setmaxlisteners-memory-leak-warning-guide.html
  x_default: /development/blog/seo/2026/06/21/nodejs-eventemitter-setmaxlisteners-memory-leak-warning-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, eventemitter, setmaxlisteners, maxlistenersexceededwarning, memory-leak, observability, backend]
description: "Node.js EventEmitter의 MaxListenersExceededWarning을 단순히 setMaxListeners로 숨기지 않고 원인을 추적하는 방법을 정리합니다. 리스너 정리, once 사용, AbortSignal 기반 해제, 운영 로그 연결까지 실무 예제로 설명합니다."
---

Node.js 서비스를 운영하다 보면 `MaxListenersExceededWarning` 경고를 만날 때가 있습니다.
메시지는 보통 "possible EventEmitter memory leak detected"처럼 보이기 때문에 당황하기 쉽습니다.
하지만 이 경고는 당장 메모리가 새고 있다는 확정 판정이라기보다, 같은 이벤트에 리스너가 비정상적으로 많이 붙었다는 신호에 가깝습니다.

문제는 이 경고를 `emitter.setMaxListeners(0)`으로 조용히 덮는 습관입니다.
제한을 없애면 로그는 사라지지만, 요청마다 리스너가 누적되는 구조나 cleanup이 빠진 코드는 그대로 남습니다.
이 글에서는 `EventEmitter`의 리스너 제한을 언제 조정하고, 언제 원인을 고쳐야 하는지 실무 기준으로 정리합니다.

운영 경고를 구조화해서 다루는 방법은 [Node.js process.emitWarning 가이드](/development/blog/seo/2026/06/20/nodejs-process-emitwarning-operational-warning-guide.html)를 함께 보면 좋습니다.
비동기 이벤트를 테스트하는 패턴은 [Node.js events.on/once async iterator 가이드](/development/blog/seo/2026/05/15/nodejs-events-on-once-async-iterator-guide.html)와 연결됩니다.
리스너 해제에 AbortSignal을 쓰고 싶다면 [Node.js events.addAbortListener cleanup 가이드](/development/blog/seo/2026/06/04/nodejs-events-addabortlistener-cleanup-guide.html)를 참고하세요.

## MaxListenersExceededWarning이 의미하는 것

### H3. 기본 제한은 버그를 빨리 보이게 하는 안전장치다

Node.js의 `EventEmitter`는 기본적으로 같은 이벤트에 많은 리스너가 붙으면 경고를 발생시킵니다.
이 제한은 성능 최적화라기보다 실수 탐지에 가깝습니다.
요청, 작업, 구독, 테스트가 반복될 때 리스너가 해제되지 않으면 같은 이벤트에 핸들러가 계속 쌓이기 때문입니다.

```js
import { EventEmitter } from 'node:events';

const bus = new EventEmitter();

for (let i = 0; i < 12; i += 1) {
  bus.on('ready', () => {
    console.log('ready');
  });
}
```

위 코드는 단순 예시지만, 실제 서비스에서는 WebSocket 연결, job worker, HTTP 요청별 callback, 테스트 fixture에서 비슷한 일이 생깁니다.
리스너가 계속 늘어나면 이벤트 하나가 발생할 때 실행되는 함수도 늘고, 참조가 남아 객체가 정리되지 않을 수 있습니다.

### H3. 경고를 끄기 전에 발생 위치를 먼저 찾아야 한다

`setMaxListeners()`는 필요한 API입니다.
하지만 경고를 보자마자 제한을 키우면 가장 중요한 단서가 사라집니다.
먼저 어느 코드가 리스너를 반복해서 붙이는지 확인해야 합니다.

```js
process.on('warning', (warning) => {
  if (warning.name !== 'MaxListenersExceededWarning') {
    return;
  }

  logger.warn({
    warningName: warning.name,
    warningMessage: warning.message,
    stack: warning.stack
  }, 'event emitter listener limit exceeded');
});
```

운영에서는 경고 스택을 로그로 남기고, 로컬이나 staging에서는 `node --trace-warnings`로 호출 위치를 더 자세히 확인하는 편이 좋습니다.
스택에 찍힌 위치가 진짜 원인인지, 공통 helper를 거쳐서 보이는 간접 위치인지도 함께 봐야 합니다.

## 리스너가 누적되는 흔한 패턴

### H3. 요청마다 전역 emitter에 on을 붙인다

가장 흔한 실수는 요청 처리 함수 안에서 전역 emitter에 `on()`을 등록하고 해제하지 않는 것입니다.
요청이 끝나도 리스너가 남으면 다음 이벤트에서 과거 요청의 context까지 같이 실행될 수 있습니다.

```js
const bus = new EventEmitter();

export async function handleRequest(req, res) {
  bus.on('cache:refresh', () => {
    res.setHeader('x-cache-refresh', '1');
  });

  res.end('ok');
}
```

이 구조는 응답 객체를 리스너가 붙잡을 수 있어 위험합니다.
요청 단위로 필요한 이벤트라면 `once()`를 쓰거나, 요청 종료 시점에 반드시 `off()`를 호출해야 합니다.

```js
export async function handleRequest(req, res) {
  const onRefresh = () => {
    if (!res.headersSent) {
      res.setHeader('x-cache-refresh', '1');
    }
  };

  bus.once('cache:refresh', onRefresh);
  res.on('close', () => {
    bus.off('cache:refresh', onRefresh);
  });

  res.end('ok');
}
```

`once()`만으로 충분한 경우도 있지만, 이벤트가 발생하지 않고 요청이 먼저 끝나는 흐름까지 생각해야 합니다.
그래서 요청 수명과 리스너 수명을 같은 기준으로 묶는 것이 중요합니다.

### H3. 재연결 로직이 이전 리스너를 정리하지 않는다

외부 메시지 브로커나 WebSocket 클라이언트에서 재연결을 구현할 때도 리스너가 누적되기 쉽습니다.
새 connection을 만들 때마다 같은 handler를 등록하면서 이전 connection의 정리를 놓치는 방식입니다.

```js
let client;

async function reconnect() {
  client = await createClient();

  client.on('message', handleMessage);
  client.on('error', handleError);
  client.on('close', reconnect);
}
```

재연결을 설계할 때는 "새로 붙이는 코드"와 "기존 것을 해제하는 코드"가 한 쌍으로 보여야 합니다.
객체가 교체되는 구조라면 이전 객체에서 리스너를 제거하거나, connection 하나의 lifecycle을 관리하는 wrapper를 둡니다.

```js
function attachClientListeners(client) {
  client.on('message', handleMessage);
  client.on('error', handleError);
  client.once('close', reconnect);

  return () => {
    client.off('message', handleMessage);
    client.off('error', handleError);
    client.off('close', reconnect);
  };
}

let cleanupClient = () => {};

async function reconnect() {
  cleanupClient();

  const nextClient = await createClient();
  cleanupClient = attachClientListeners(nextClient);
}
```

이런 정리 함수를 반환하는 패턴은 테스트에서도 다루기 쉽습니다.
테스트가 끝난 뒤 cleanup을 호출하면 다음 테스트로 리스너가 새어 나가는 문제를 줄일 수 있습니다.

## setMaxListeners를 써도 되는 경우

### H3. 정상적으로 많은 구독자가 필요한 emitter는 제한을 명시한다

모든 경고가 버그는 아닙니다.
하나의 이벤트 버스에 여러 플러그인, 모듈, tenant별 handler가 붙는 구조라면 기본 제한보다 많은 리스너가 정상일 수 있습니다.
이 경우에는 제한을 없애기보다 예상 가능한 상한을 명시하는 편이 낫습니다.

```js
const pluginBus = new EventEmitter();

pluginBus.setMaxListeners(50);
```

숫자는 "경고를 피하기 위한 큰 값"이 아니라 설계상 가능한 최대 구독자 수를 기준으로 정합니다.
예상 상한이 30개인데 50으로 둔다면 어느 정도 여유가 있고, 50을 넘는 순간 다시 조사할 수 있습니다.
반대로 `0`이나 `Infinity`는 정말로 제한이 필요 없는 경우가 아니라면 피하는 것이 좋습니다.

### H3. 임시 증가 후 원래 값으로 되돌린다

짧은 작업 동안만 리스너가 늘어나는 구조라면 제한을 임시로 올렸다가 되돌릴 수 있습니다.
다만 이 패턴은 cleanup이 확실한 범위에서만 사용해야 합니다.

```js
async function waitForManySignals(emitter, signals) {
  const previousLimit = emitter.getMaxListeners();
  emitter.setMaxListeners(Math.max(previousLimit, signals.length + 1));

  try {
    return await Promise.all(
      signals.map((signal) => waitForSignal(emitter, signal))
    );
  } finally {
    emitter.setMaxListeners(previousLimit);
  }
}
```

`finally`에서 원래 값을 되돌리는 것이 핵심입니다.
작업 도중 오류가 나도 제한이 계속 높게 남지 않아야 다음 경고가 정상적으로 보입니다.

## AbortSignal로 리스너 수명을 묶기

### H3. 요청 취소와 cleanup을 같은 신호로 연결한다

Node.js의 여러 API는 `AbortSignal`을 cancellation 신호로 사용합니다.
이 패턴을 EventEmitter 리스너 정리에도 적용하면 수명 관리가 더 명확해집니다.

```js
function onWithSignal(emitter, eventName, listener, signal) {
  if (signal.aborted) {
    return;
  }

  emitter.on(eventName, listener);

  signal.addEventListener('abort', () => {
    emitter.off(eventName, listener);
  }, { once: true });
}

export function subscribeJobProgress(jobId, signal) {
  onWithSignal(jobBus, `job:${jobId}:progress`, handleProgress, signal);
}
```

이 방식은 HTTP 요청, background job, UI bridge처럼 "언제 끝나는지"가 이미 정해진 작업에 잘 맞습니다.
리스너 해제 위치가 여러 곳에 흩어지지 않고 취소 신호 하나로 모입니다.

### H3. 테스트로 리스너 개수를 확인한다

리스너 누적 문제는 테스트로 잡을 수 있습니다.
특히 helper 함수가 리스너를 붙이는 역할을 한다면, 작업 후 listener count가 원래대로 돌아오는지 확인합니다.

```js
import assert from 'node:assert/strict';
import { EventEmitter } from 'node:events';
import { test } from 'node:test';

test('removes listener when signal is aborted', () => {
  const emitter = new EventEmitter();
  const controller = new AbortController();

  onWithSignal(emitter, 'done', () => {}, controller.signal);

  assert.equal(emitter.listenerCount('done'), 1);

  controller.abort();

  assert.equal(emitter.listenerCount('done'), 0);
});
```

이 테스트는 구현 세부사항처럼 보일 수 있지만, 운영 장애로 이어지는 리스너 누적을 막는 안전장치가 됩니다.
이벤트 기반 코드가 많은 프로젝트라면 이런 작은 테스트가 디버깅 시간을 크게 줄입니다.

## 운영 점검 체크리스트

### H3. 경고 발생 시 바로 볼 항목

`MaxListenersExceededWarning`이 보이면 먼저 아래 순서로 확인합니다.

- 경고 스택에서 반복 등록 위치를 찾는다.
- `on()`을 `once()`로 바꿀 수 있는지 확인한다.
- 요청, connection, job 종료 시점에 `off()`가 호출되는지 본다.
- 재연결 또는 retry 로직에서 이전 리스너를 정리하는지 확인한다.
- 정상적인 다중 구독 구조라면 `setMaxListeners()`에 설계상 상한을 넣는다.

이 순서의 목적은 경고를 없애는 것이 아니라 원인을 분류하는 것입니다.
버그라면 cleanup을 고치고, 설계상 정상이라면 제한을 명시합니다.

### H3. 로그와 알림 기준

운영 로그에서는 경고 이름, 메시지, 스택, 프로세스 정보 정도를 남기면 충분합니다.
사용자 데이터, 토큰, 요청 본문, 내부 비밀 URL은 경고 로그에 넣지 않습니다.

```js
process.on('warning', (warning) => {
  logger.warn({
    warningName: warning.name,
    warningMessage: warning.message,
    pid: process.pid,
    stack: warning.stack
  }, 'node warning');
});
```

알림은 모든 경고가 아니라 새로 발생한 경고, 배포 직후 증가한 경고, 특정 emitter에서 반복되는 경고에 집중하는 편이 좋습니다.
일시적인 개발 환경 경고와 운영 트래픽에서 발생하는 경고를 구분하면 대응 우선순위가 명확해집니다.

## FAQ

### H3. setMaxListeners(0)은 항상 나쁜가요?

항상 나쁜 것은 아닙니다.
하지만 대부분의 애플리케이션 코드에서는 제한을 완전히 끄기보다 설계상 필요한 상한을 숫자로 두는 편이 좋습니다.
`0`은 버그 탐지 장치를 끄는 선택이므로, 왜 제한이 없어도 되는지 설명할 수 있을 때만 사용합니다.

### H3. MaxListenersExceededWarning이 뜨면 메모리 누수가 확정인가요?

확정은 아닙니다.
다만 리스너가 예상보다 많이 쌓였다는 강한 신호입니다.
경고가 반복된다면 `--trace-warnings`, `process.on('warning')`, `listenerCount()` 테스트로 등록 위치와 cleanup 누락을 확인해야 합니다.

### H3. 운영에서는 경고를 에러로 처리해야 하나요?

대부분은 에러로 즉시 종료하기보다 로그와 알림으로 추적하는 편이 낫습니다.
다만 배포 직후 특정 경고가 급증하거나, 요청별 리스너 누적으로 메모리 사용량이 계속 증가한다면 배포 중단이나 rollback 조건에 포함할 수 있습니다.


---
layout: post
title: "Node.js events.once/on 가이드: EventEmitter를 Promise와 async iterator로 다루는 법"
date: 2026-05-15 20:00:00 +0900
lang: ko
translation_key: nodejs-events-on-once-async-iterator-guide
permalink: /development/blog/seo/2026/05/15/nodejs-events-on-once-async-iterator-guide.html
alternates:
  ko: /development/blog/seo/2026/05/15/nodejs-events-on-once-async-iterator-guide.html
  x_default: /development/blog/seo/2026/05/15/nodejs-events-on-once-async-iterator-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, eventemitter, events, async-iterator, promise, backend]
description: "Node.js events.once와 events.on으로 EventEmitter 이벤트를 Promise와 async iterator처럼 다루는 방법을 정리했습니다. 단발성 이벤트 대기, 스트리밍 이벤트 처리, AbortSignal 취소, 에러 처리 기준까지 실무 예제로 설명합니다."
---

Node.js에서 `EventEmitter`는 오래된 API이지만 여전히 핵심 런타임 패턴입니다.
서버 연결, 스트림, 작업 큐, WebSocket 클라이언트, 커스텀 도메인 이벤트까지 많은 코드가 `emitter.on('event', listener)` 형태로 동작합니다.
문제는 이벤트 리스너가 많아질수록 흐름 제어가 흩어지고, 에러 처리와 정리 코드가 빠지기 쉽다는 점입니다.

Node.js의 `node:events` 모듈은 이런 코드를 더 현대적인 비동기 흐름으로 다룰 수 있게 `events.once()`와 `events.on()`을 제공합니다.
`once()`는 특정 이벤트를 Promise처럼 한 번 기다릴 때 유용하고, `on()`은 이벤트를 async iterator로 순회할 때 좋습니다.
이 글에서는 두 API의 차이, 실무 사용 예시, `AbortSignal`로 취소하는 방법, 에러 처리 기준을 정리합니다.
이벤트 기반 시스템의 실패 격리까지 함께 고민한다면 [Node.js EventEmitter captureRejections 가이드: async listener 에러를 안전하게 처리하기](/development/blog/seo/2026/04/25/nodejs-eventemitter-capturerejections-unhandled-async-errors-guide.html)도 같이 참고하면 좋습니다.

## events.once가 필요한 상황

### H3. 단발성 이벤트를 Promise처럼 기다린다

전통적인 이벤트 코드는 콜백 안에 다음 로직을 넣는 방식으로 작성됩니다.
짧은 코드에서는 괜찮지만, 연결 완료 후 인증하고, 메시지를 보내고, 응답을 기다리는 흐름이 이어지면 중첩이 빠르게 늘어납니다.

```js
import { EventEmitter, once } from 'node:events';

const worker = new EventEmitter();

setTimeout(() => {
  worker.emit('ready', { startedAt: new Date() });
}, 100);

const [metadata] = await once(worker, 'ready');
console.log('worker is ready:', metadata.startedAt.toISOString());
```

`once(emitter, eventName)`은 이벤트가 발생할 때까지 기다렸다가 이벤트 인자를 배열로 반환합니다.
이벤트 인자가 하나면 `[value]`, 여러 개면 `[first, second]`처럼 구조 분해해서 받으면 됩니다.
이 방식은 테스트, 초기화 절차, 배포 후 smoke check처럼 “한 번만 확인하면 되는 이벤트”에 잘 맞습니다.

```js
import { once } from 'node:events';
import net from 'node:net';

const server = net.createServer((socket) => {
  socket.end('ok');
});

server.listen(0, '127.0.0.1');
await once(server, 'listening');

const address = server.address();
console.log(`listening on ${address.address}:${address.port}`);

server.close();
```

이 코드는 `listening` 이벤트를 Promise처럼 기다리기 때문에 테스트 러너나 배포 검증 스크립트 안에서 순차적으로 읽힙니다.
운영 진단 스크립트를 가볍게 유지하는 방식은 [Node.js WebSocket 내장 클라이언트 가이드: ws 없이 실시간 연결 테스트하기](/development/blog/seo/2026/05/14/nodejs-built-in-websocket-client-guide.html)와도 잘 어울립니다.

### H3. error 이벤트는 별도로 생각해야 한다

`EventEmitter`에서 `error` 이벤트는 특별합니다.
리스너가 없는데 `error`가 발생하면 프로세스가 예외로 종료될 수 있습니다.
`once()`는 일반적으로 기다리는 동안 `error` 이벤트가 발생하면 Promise를 reject합니다.
따라서 단발성 대기 코드에서는 `try/catch`를 기본으로 두는 편이 안전합니다.

```js
import { once } from 'node:events';

async function waitUntilReady(client) {
  try {
    const [info] = await once(client, 'ready');
    return info;
  } catch (error) {
    throw new Error('client failed before ready', { cause: error });
  }
}
```

에러를 감싸서 던지면 호출 지점에서 “어떤 단계에서 실패했는지”가 더 분명해집니다.
원인 보존이 필요한 예외 설계는 [Node.js Error cause 가이드: 감싼 에러의 원인을 잃지 않고 디버깅하기](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)를 참고하세요.

## events.on으로 이벤트 스트림 순회하기

### H3. 반복되는 이벤트를 for await...of로 처리한다

`events.on()`은 특정 이벤트가 여러 번 발생하는 상황을 async iterator로 바꿔 줍니다.
리스너 콜백을 등록해 두는 대신 `for await...of` 루프에서 이벤트를 하나씩 처리할 수 있습니다.

```js
import { EventEmitter, on } from 'node:events';

const queue = new EventEmitter();

setInterval(() => {
  queue.emit('job', { id: crypto.randomUUID(), type: 'thumbnail' });
}, 500);

for await (const [job] of on(queue, 'job')) {
  console.log('received job:', job.id, job.type);
}
```

이 패턴의 장점은 비동기 처리를 자연스럽게 기다릴 수 있다는 점입니다.
이벤트가 들어올 때마다 DB 저장, 외부 API 호출, 파일 처리 같은 작업을 순차적으로 수행해야 한다면 콜백보다 흐름을 읽기 쉽습니다.
다만 모든 이벤트를 무한히 처리하는 루프가 되기 쉬우므로 종료 조건과 취소 신호를 반드시 설계해야 합니다.

### H3. AbortSignal로 루프를 멈춘다

운영 코드에서 이벤트 루프를 영원히 돌리는 것은 위험합니다.
배포 종료, 테스트 타임아웃, 사용자 요청 취소 같은 상황에서 리스너를 정리해야 메모리 누수와 중복 처리 문제를 줄일 수 있습니다.
`events.on()`은 옵션으로 `signal`을 받을 수 있어 `AbortController`와 함께 쓰기 좋습니다.

```js
import { EventEmitter, on } from 'node:events';

const emitter = new EventEmitter();
const controller = new AbortController();

setTimeout(() => controller.abort(), 5_000);

try {
  for await (const [message] of on(emitter, 'message', {
    signal: controller.signal,
  })) {
    console.log('message:', message);
  }
} catch (error) {
  if (error.name !== 'AbortError') {
    throw error;
  }
}
```

취소는 실패가 아니라 정상적인 종료 경로일 때가 많습니다.
그래서 `AbortError`를 별도로 처리하고, 그 외 에러만 다시 던지는 식으로 구분하는 편이 좋습니다.
취소 체크포인트를 함수 안에 명시하는 방법은 [Node.js AbortSignal.throwIfAborted 가이드: 취소 가능한 작업의 체크포인트 만들기](/development/blog/seo/2026/05/06/nodejs-abortsignal-throwifaborted-cancellation-checkpoint-guide.html)에서도 같은 원칙으로 다뤘습니다.

## once와 on을 고르는 기준

### H3. 한 번만 필요한 이벤트는 once가 낫다

다음 상황에서는 `once()`가 더 단순합니다.

- 서버가 `listening` 상태가 될 때까지 기다리기
- 커넥션이 `open` 되는 순간까지 기다리기
- 테스트에서 특정 이벤트가 정확히 한 번 발생하는지 검증하기
- 초기화 완료 이벤트를 기다린 뒤 다음 단계로 넘어가기

```js
import { once } from 'node:events';

export async function connectAndWait(client) {
  client.connect();
  const [session] = await once(client, 'connected');
  return session;
}
```

한 번만 필요한 이벤트를 `on()`으로 처리하면 루프 종료 조건을 따로 만들어야 합니다.
오히려 코드가 길어지고, 종료를 깜박하면 리스너가 계속 남을 수 있습니다.

### H3. 계속 들어오는 이벤트는 on이 낫다

반대로 다음 상황에서는 `on()`이 자연스럽습니다.

- 작업 큐 이벤트를 순차적으로 소비하기
- WebSocket 메시지를 일정 시간 동안 관찰하기
- 파일 변경 이벤트를 테스트에서 수집하기
- 커스텀 이벤트를 배치로 모아 처리하기

```js
import { on } from 'node:events';

export async function collectEvents(emitter, eventName, limit, signal) {
  const items = [];

  for await (const args of on(emitter, eventName, { signal })) {
    items.push(args);

    if (items.length >= limit) {
      break;
    }
  }

  return items;
}
```

`break`로 루프를 빠져나오면 async iterator가 정리되면서 내부 리스너도 제거됩니다.
그래도 실무에서는 `signal`을 함께 넘겨 테스트 타임아웃이나 상위 요청 취소와 연결하는 편이 더 안전합니다.

## 실무 예제: 이벤트 기반 작업 처리기 만들기

### H3. 이벤트를 순차 처리해 경쟁 상태를 줄인다

이벤트 리스너 콜백은 동시에 여러 번 실행될 수 있습니다.
작업 순서가 중요하거나 공유 자원을 수정한다면, `for await...of` 루프에서 하나씩 처리하는 구조가 더 예측 가능합니다.

```js
import { EventEmitter, on } from 'node:events';

export class JobBus extends EventEmitter {
  publish(job) {
    this.emit('job', job);
  }
}

export async function runWorker(bus, { signal }) {
  for await (const [job] of on(bus, 'job', { signal })) {
    await handleJob(job, signal);
  }
}

async function handleJob(job, signal) {
  signal?.throwIfAborted();

  console.log('start job:', job.id);
  await new Promise((resolve) => setTimeout(resolve, 100));
  console.log('done job:', job.id);
}
```

이 코드는 처리량을 최대로 높이는 구조는 아닙니다.
대신 작업 순서와 상태 변경의 예측 가능성을 우선합니다.
동시성이 필요하다면 `handleJob()`을 바로 `await`하지 않고 제한된 Promise pool로 넘기는 방식이 낫습니다.
동시성 제한 설계는 [Node.js Promise Pool 가이드: 외부 API 과부하를 막는 동시성 제한 패턴](/development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)과 연결해서 보면 좋습니다.

### H3. 테스트에서는 타임아웃을 명시한다

이벤트 테스트에서 가장 흔한 실패는 “이벤트가 안 왔는데 테스트가 오래 멈춰 있는 상황”입니다.
`AbortSignal.timeout()`이나 `AbortController`를 사용해 최대 대기 시간을 명시하면 실패 원인이 더 빨리 드러납니다.

```js
import assert from 'node:assert/strict';
import { EventEmitter, once } from 'node:events';
import test from 'node:test';

class UserCreatedBus extends EventEmitter {}

test('user.created event is emitted', async () => {
  const bus = new UserCreatedBus();
  const signal = AbortSignal.timeout(1_000);

  queueMicrotask(() => {
    bus.emit('user.created', { userId: 'user_123' });
  });

  const [event] = await once(bus, 'user.created', { signal });

  assert.equal(event.userId, 'user_123');
});
```

예시의 `user_123`은 설명용 가짜 식별자입니다.
실제 로그나 테스트 fixture에는 이메일, 전화번호, 토큰 같은 민감정보를 그대로 넣지 않는 편이 안전합니다.
테스트 러너 자체의 기본기는 [Node.js test runner 가이드: 내장 테스트 러너로 가볍게 테스트 시작하기](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 참고하세요.

## 주의할 점

### H3. 이벤트 생산 속도와 소비 속도를 맞춰야 한다

`events.on()`은 읽기 쉬운 구조를 제공하지만 자동으로 백프레셔 문제를 해결해 주지는 않습니다.
이벤트가 초당 수천 개 들어오는데 처리 함수가 느리면 메모리 사용량이 늘고 지연이 쌓일 수 있습니다.
이럴 때는 다음 기준을 함께 검토해야 합니다.

- 이벤트 생산자 쪽에서 속도 제한을 걸 수 있는가?
- 소비자 쪽에서 동시성 제한을 둘 것인가?
- 오래된 이벤트를 버려도 되는가, 반드시 보존해야 하는가?
- 실패 이벤트를 재시도 큐나 dead letter queue로 보낼 것인가?

스트림 기반 입력처럼 흐름 제어가 핵심인 경우에는 [Node.js Stream Backpressure 가이드: 메모리 스파이크 없이 대용량 데이터 처리하기](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)를 함께 보는 것이 좋습니다.

### H3. 리스너 정리는 코드 리뷰 체크리스트에 넣는다

이벤트 코드는 정상 동작할 때보다 종료될 때 더 많은 버그가 생깁니다.
테스트가 끝났는데 리스너가 남아 있거나, 재연결할 때 이전 리스너가 제거되지 않으면 같은 이벤트가 중복 처리됩니다.
`once()`와 `on()`을 쓸 때는 다음을 확인하세요.

- `once()`에는 필요한 경우 `signal`이나 상위 타임아웃이 있는가?
- `on()` 루프에는 `break`, `return`, `signal` 중 하나 이상의 종료 경로가 있는가?
- `AbortError`와 실제 장애 에러를 구분하는가?
- 이벤트 인자에 민감정보가 포함되어 로그로 노출되지 않는가?
- 고빈도 이벤트라면 처리 속도와 메모리 증가를 관찰하는가?

## 마무리

`events.once()`와 `events.on()`은 오래된 `EventEmitter` 코드를 완전히 새 API로 바꾸는 도구가 아닙니다.
대신 이벤트 기반 코드를 Promise와 async iterator 흐름 안으로 가져와 읽기 쉽게 만들고, 테스트와 운영 진단에서 종료 조건을 명확히 만드는 도구입니다.

한 번만 기다리는 초기화나 검증에는 `once()`를 사용하고, 반복되는 이벤트 소비에는 `on()`을 사용하세요.
그리고 두 경우 모두 에러 처리, 취소 신호, 리스너 정리를 함께 설계해야 합니다.
이 세 가지를 지키면 EventEmitter 기반 코드도 최신 Node.js 비동기 코드와 훨씬 자연스럽게 연결할 수 있습니다.

## FAQ

### H3. events.once는 emitter.once와 같은가요?

목적은 비슷하지만 사용 방식이 다릅니다.
`emitter.once()`는 콜백 리스너를 한 번 등록하고, `events.once()`는 이벤트를 Promise처럼 기다립니다.
`async/await` 흐름에서는 `events.once()`가 더 읽기 쉽습니다.

### H3. events.on은 모든 이벤트 처리에 써도 되나요?

아닙니다.
고빈도 이벤트나 백프레셔가 중요한 스트림 처리에서는 별도 흐름 제어가 필요합니다.
`events.on()`은 코드 구조를 단순하게 만들지만, 생산 속도와 소비 속도 문제를 자동으로 해결하지는 않습니다.

### H3. AbortSignal을 꼭 써야 하나요?

무한히 기다릴 수 있는 코드라면 쓰는 편이 안전합니다.
특히 테스트, 진단 스크립트, 서버 종료 처리에서는 `AbortSignal`을 연결해 리스너가 남지 않도록 만드는 것이 좋습니다.

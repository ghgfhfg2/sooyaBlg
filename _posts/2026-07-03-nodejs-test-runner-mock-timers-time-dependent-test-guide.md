---
layout: post
title: "Node.js test runner mock timers 가이드: 시간 의존 테스트를 빠르고 안정적으로 만드는 법"
date: 2026-07-03 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-mock-timers-time-dependent-test-guide
permalink: /development/blog/seo/2026/07/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html
alternates:
  ko: /development/blog/seo/2026/07/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html
  x_default: /development/blog/seo/2026/07/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, mock-timers, testing, javascript, ci, time-dependent-test]
description: "Node.js test runner의 mock timers로 setTimeout, setInterval, Date.now(), node:timers/promises 기반 테스트를 빠르고 안정적으로 작성하는 방법을 정리합니다. enable(), tick(), setTime(), CI 운영 기준과 주의점을 예제로 설명합니다."
---

시간이 들어간 코드는 테스트하기 까다롭습니다.
만료 시간, 재시도 지연, 예약 작업, debounce, polling처럼 실제 시간을 기다리면 테스트가 느려지고 CI가 불안정해집니다.
반대로 시간을 전부 주입 가능한 의존성으로 바꾸면 제품 코드가 테스트를 위해 과하게 복잡해질 수 있습니다.

Node.js 내장 test runner는 이런 상황을 위해 mock timers를 제공합니다.
[Node.js test runner 공식 문서](https://nodejs.org/api/test.html)에 따르면 테스트 컨텍스트의 `context.mock.timers.enable()`로 특정 timer API를 mock 처리하고, `tick()`으로 가짜 시간을 앞으로 이동시킬 수 있습니다.
`Date`도 함께 mock하면 `Date.now()`와 새로 생성되는 `Date` 객체의 시간까지 테스트 안에서 제어할 수 있습니다.

Node.js 내장 테스트 러너의 기본 구조는 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 참고하세요.
비동기 실패 검증은 [Node.js assert rejects 가이드](/development/blog/seo/2026/06/30/nodejs-assert-rejects-async-error-test-guide.html)와 함께 보면 좋습니다.
느린 테스트를 CI에서 분리하는 기준은 [Node.js test runner tags 가이드](/development/blog/seo/2026/07/02/nodejs-test-runner-tags-filter-guide.html)에서 이어서 정리했습니다.

## mock timers가 필요한 상황

### H3. 실제 시간을 기다리는 테스트를 제거한다

가장 흔한 문제는 테스트가 제품 코드의 지연 시간만큼 그대로 기다리는 경우입니다.
예를 들어 3초 뒤에 실행되는 timeout을 검증하려고 테스트도 3초를 기다리면, 테스트 하나는 괜찮아 보여도 전체 스위트에서는 금방 비용이 커집니다.
mock timers를 쓰면 실제 시간은 거의 흐르지 않지만, 테스트 안의 가짜 시간은 원하는 만큼 이동시킬 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function scheduleRetry(send) {
  setTimeout(() => {
    send('retry');
  }, 3000);
}

test('sends retry after the delay', (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });
  const send = context.mock.fn();

  scheduleRetry(send);

  assert.strictEqual(send.mock.callCount(), 0);

  context.mock.timers.tick(2999);
  assert.strictEqual(send.mock.callCount(), 0);

  context.mock.timers.tick(1);
  assert.strictEqual(send.mock.callCount(), 1);
  assert.strictEqual(send.mock.calls[0].arguments[0], 'retry');
});
```

이 테스트는 3초를 실제로 기다리지 않습니다.
`tick(2999)`에서는 아직 콜백이 실행되지 않아야 하고, 그다음 `tick(1)`에서 정확히 한 번 실행되어야 합니다.
시간 경계값을 명확히 검증할 수 있다는 점도 단순 대기보다 낫습니다.

### H3. 시간 의존 로직과 비동기 로직을 분리해서 본다

mock timers는 시간 흐름을 제어하는 도구입니다.
네트워크 요청, 파일 시스템, 데이터베이스 같은 외부 의존성을 자동으로 mock 처리하지는 않습니다.
따라서 시간 지연 자체가 관심사인지, 아니면 지연 후 실행되는 작업의 결과가 관심사인지 먼저 나누는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function debounce(fn, delay) {
  let timer;

  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

test('debounces repeated calls', (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });
  const save = context.mock.fn();
  const debouncedSave = debounce(save, 500);

  debouncedSave('a');
  debouncedSave('ab');
  debouncedSave('abc');

  context.mock.timers.tick(499);
  assert.strictEqual(save.mock.callCount(), 0);

  context.mock.timers.tick(1);
  assert.strictEqual(save.mock.callCount(), 1);
  assert.deepStrictEqual(save.mock.calls[0].arguments, ['abc']);
});
```

이 예제의 핵심은 debounce가 마지막 입력만 실행하는지입니다.
실제 저장소나 API 호출은 테스트 대상이 아닙니다.
시간 제어와 외부 의존성 검증을 한 테스트에 모두 넣으면 실패 원인이 흐려지므로, 가능한 한 역할을 작게 나누세요.

## 기본 사용법

### H3. enable()에서 mock할 API를 명시한다

`context.mock.timers.enable()`은 mock할 API 목록을 받습니다.
`setTimeout`, `setInterval`, `setImmediate`, `Date`처럼 필요한 API만 켜는 방식이 읽기 쉽습니다.
테스트가 어떤 시간 기능에 의존하는지 코드에서 바로 드러나기 때문입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function expiresAt(ttlMs) {
  return Date.now() + ttlMs;
}

test('calculates expiration from the current time', (context) => {
  context.mock.timers.enable({
    apis: ['Date'],
    now: 1_700_000_000_000
  });

  assert.strictEqual(expiresAt(60_000), 1_700_000_060_000);
});
```

`Date`만 필요한 테스트라면 timer callback까지 mock할 필요가 없습니다.
반대로 `setTimeout`만 검증하는 테스트에서 현재 시각이 중요하지 않다면 `Date`를 켜지 않아도 됩니다.
mock 범위가 작을수록 테스트 의도가 선명합니다.

### H3. tick()으로 예약된 timer를 실행한다

`tick()`은 가짜 시간을 지정한 밀리초만큼 앞으로 이동합니다.
그 과정에서 실행 시점에 도달한 timer callback이 동기적으로 처리됩니다.
공식 문서의 예제처럼 여러 번 나누어 호출하면 중간 상태도 검증할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function createPoller(read) {
  const id = setInterval(() => {
    read();
  }, 1000);

  return () => clearInterval(id);
}

test('polls until it is stopped', (context) => {
  context.mock.timers.enable({ apis: ['setInterval'] });
  const read = context.mock.fn();

  const stop = createPoller(read);

  context.mock.timers.tick(1000);
  context.mock.timers.tick(1000);
  assert.strictEqual(read.mock.callCount(), 2);

  stop();
  context.mock.timers.tick(3000);
  assert.strictEqual(read.mock.callCount(), 2);
});
```

이 테스트는 interval이 반복 실행되는지와 `clearInterval()` 이후 멈추는지를 함께 확인합니다.
실제 5초를 기다리지 않기 때문에 로컬 개발 루프와 CI 모두 빠르게 유지됩니다.
시간 기반 동작은 대기보다 시간 이동으로 검증하는 편이 안정적입니다.

## Date와 setTime 다루기

### H3. Date를 mock하면 현재 시각도 테스트 데이터가 된다

만료 시간, 일일 리셋, 날짜별 집계처럼 현재 시각이 결과에 영향을 준다면 `Date`를 mock해야 합니다.
`now` 값을 고정하면 테스트가 실행되는 날짜와 관계없이 같은 결과를 얻습니다.
이 방식은 CI 서버의 시간대나 실행 시각 때문에 테스트가 흔들리는 문제도 줄입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function buildDailyCacheKey(userId) {
  const day = new Date().toISOString().slice(0, 10);
  return `daily:${day}:${userId}`;
}

test('builds a stable daily cache key', (context) => {
  context.mock.timers.enable({
    apis: ['Date'],
    now: Date.parse('2026-07-03T00:00:00.000Z')
  });

  assert.strictEqual(buildDailyCacheKey('user-1'), 'daily:2026-07-03:user-1');
});
```

테스트가 날짜 경계에 민감하다면 입력 시간을 더 노골적으로 고정하세요.
`new Date()`를 그대로 두고 기대값만 오늘 날짜로 맞추는 테스트는 다음 날 바로 실패할 수 있습니다.
시간은 숨은 전역 상태이므로, 테스트에서는 명시적인 데이터처럼 다루는 편이 좋습니다.

### H3. setTime()은 시계를 옮기지만 과거 timer를 자동 실행하지 않는다

공식 문서에 따르면 `setTime()`은 mock된 `Date`의 시간을 직접 옮기지만, 그 시점 이전에 예약된 timer를 자동 실행하지 않습니다.
이미 실행 시점을 지난 timer를 처리하려면 `tick()`을 호출해야 합니다.
이 차이를 모르면 테스트가 "시간을 옮겼는데 callback이 왜 안 불리지"라는 식으로 헷갈릴 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('setTime changes the clock but tick runs the timer', (context) => {
  context.mock.timers.enable({ apis: ['setTimeout', 'Date'] });
  const expire = context.mock.fn();

  setTimeout(expire, 1000);

  context.mock.timers.setTime(800);
  assert.strictEqual(Date.now(), 800);
  assert.strictEqual(expire.mock.callCount(), 0);

  context.mock.timers.tick(200);
  assert.strictEqual(expire.mock.callCount(), 1);
});
```

`setTime()`은 현재 시각을 특정 값으로 맞추고 싶을 때 유용합니다.
`tick()`은 시간 경과와 timer 실행을 함께 검증하고 싶을 때 적합합니다.
두 메서드의 역할을 분리해서 쓰면 테스트의 의도가 더 잘 드러납니다.

## node:timers/promises와 함께 쓰기

### H3. promise 기반 timeout도 같은 시간 축에서 검증한다

Node.js에서는 `node:timers/promises`의 `setTimeout()`을 쓰는 코드도 많습니다.
공식 문서에 따르면 mock timers는 전역 timer뿐 아니라 Node.js timer 모듈과 promises 기반 timer에도 적용됩니다.
promise가 resolve되는 시점은 `tick()` 이후 `await`로 확인하면 됩니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { setTimeout as delay } from 'node:timers/promises';

async function waitAndReturn(value) {
  await delay(2000);
  return value;
}

test('resolves a timer promise without waiting', async (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });

  const resultPromise = waitAndReturn('done');

  context.mock.timers.tick(1999);
  await Promise.resolve();

  context.mock.timers.tick(1);
  await assert.doesNotReject(resultPromise);
  assert.strictEqual(await resultPromise, 'done');
});
```

promise 기반 timer는 callback timer보다 microtask 흐름을 더 신경 써야 합니다.
`tick()`으로 시간이 이동한 뒤에는 promise를 `await`해서 실제 resolve 결과를 확인하세요.
비동기 함수의 실패까지 검증해야 한다면 `assert.rejects()`나 `assert.doesNotReject()`를 함께 쓰는 편이 명확합니다.

### H3. abort, retry, timeout 정책은 작은 단위로 나눈다

실무 코드에서는 timer promise가 `AbortSignal`, retry loop, 외부 요청과 함께 섞이는 경우가 많습니다.
이때 한 테스트에서 모든 정책을 검증하려고 하면 시간 이동, signal 상태, mock 호출 횟수, 예외 메시지가 한꺼번에 얽힙니다.
테스트를 정책별로 나누면 실패 원인이 훨씬 빨리 보입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { setTimeout as delay } from 'node:timers/promises';

async function retryOnce(task) {
  try {
    return await task();
  } catch {
    await delay(1000);
    return task();
  }
}

test('waits before retrying once', async (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });
  const task = context.mock.fn(async () => {
    if (task.mock.callCount() === 1) {
      throw new Error('temporary');
    }

    return 'ok';
  });

  const resultPromise = retryOnce(task);

  await Promise.resolve();
  assert.strictEqual(task.mock.callCount(), 1);

  context.mock.timers.tick(1000);

  assert.strictEqual(await resultPromise, 'ok');
  assert.strictEqual(task.mock.callCount(), 2);
});
```

이 테스트는 retry가 한 번 더 호출되는지와 retry 전에 1초를 기다리는지만 확인합니다.
네트워크 오류 종류나 응답 파싱은 다른 테스트로 분리할 수 있습니다.
mock timers를 쓴다고 해서 테스트 범위를 크게 잡아야 하는 것은 아닙니다.

## CI에서 운영하는 기준

### H3. 느린 테스트를 빠른 테스트로 바꾸는 데 우선 사용한다

mock timers의 가장 큰 장점은 느린 테스트를 빠른 테스트로 바꾸는 것입니다.
CI에서 30초, 60초씩 기다리는 테스트가 있다면 먼저 timer mock으로 줄일 수 있는지 확인하세요.
실제 시간 대기는 end-to-end 수준에서 꼭 필요한 경우에만 남기는 편이 좋습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "node --test",
    "test:slow": "node --test --experimental-test-tag-filter=slow"
  }
}
```

시간을 기다리는 테스트가 꼭 필요하다면 `slow` 같은 tag로 별도 실행 경로를 두는 방법도 있습니다.
하지만 단위 테스트에서 검증하려는 것이 timer 동작 자체라면 mock timers가 더 적합합니다.
기본 테스트 명령은 빠르게 유지하고, 실제 시간에 의존하는 검증은 최소화하세요.

### H3. 테스트가 끝난 뒤 남는 timer를 만들지 않는다

테스트가 끝났는데 timer나 비동기 작업이 계속 남아 있으면 다음 테스트나 runner 진단에 영향을 줄 수 있습니다.
mock timers를 쓰더라도 interval을 만들었다면 종료 조건을 검증하거나 `clearInterval()`을 호출해야 합니다.
제품 코드가 정리 함수를 반환하도록 설계하면 테스트도 운영 코드도 다루기 쉬워집니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function startHeartbeat(send) {
  const id = setInterval(send, 1000);

  return {
    stop() {
      clearInterval(id);
    }
  };
}

test('stops heartbeat cleanly', (context) => {
  context.mock.timers.enable({ apis: ['setInterval'] });
  const send = context.mock.fn();

  const heartbeat = startHeartbeat(send);

  context.mock.timers.tick(2000);
  assert.strictEqual(send.mock.callCount(), 2);

  heartbeat.stop();
  context.mock.timers.tick(2000);
  assert.strictEqual(send.mock.callCount(), 2);
});
```

timer 기반 코드는 시작보다 정리가 더 중요할 때가 많습니다.
특히 interval, polling, background refresh는 종료 경로가 없으면 테스트뿐 아니라 실제 앱에서도 리소스를 붙잡습니다.
mock timers 테스트에 stop 경로를 포함하면 이런 문제를 일찍 발견할 수 있습니다.

## 자주 하는 실수

### H3. Date만 고정하고 timeout은 실제로 기다린다

`Date.now()`만 고정하면 현재 시각은 안정되지만, `setTimeout()`은 여전히 실제 시간으로 동작할 수 있습니다.
제품 코드가 현재 시각과 timeout을 함께 사용한다면 둘 다 mock 대상에 넣어야 합니다.
반대로 timeout만 mock하고 `Date`를 그대로 두면 경계일, 만료일, 로그 시각 검증이 흔들릴 수 있습니다.

```js
context.mock.timers.enable({
  apis: ['setTimeout', 'Date'],
  now: Date.parse('2026-07-03T00:00:00.000Z')
});
```

테스트 대상이 어떤 시간 API를 쓰는지 먼저 확인하세요.
필요한 API를 빠뜨리면 테스트가 느리거나 불안정해지고, 불필요한 API를 많이 켜면 테스트 의도가 흐려집니다.
mock 범위는 제품 코드의 시간 의존성과 맞춰야 합니다.

### H3. 시간 이동으로 모든 비동기 작업이 끝났다고 가정한다

`tick()`은 timer 시간을 이동시키지만, 그 timer callback 안에서 시작된 promise 체인까지 모두 자동으로 검증해 주지는 않습니다.
비동기 결과가 중요하다면 promise를 반환하거나 `await`해야 합니다.
테스트 함수가 너무 일찍 끝나면 나중에 발생한 예외가 별도 진단으로 보고될 수 있습니다.

```js
test('awaits async work after advancing time', async (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });
  const done = context.mock.fn();

  const promise = new Promise((resolve) => {
    setTimeout(async () => {
      await Promise.resolve();
      done();
      resolve();
    }, 100);
  });

  context.mock.timers.tick(100);
  await promise;

  assert.strictEqual(done.mock.callCount(), 1);
});
```

시간이 지났다는 사실과 비동기 작업이 끝났다는 사실은 다릅니다.
테스트가 관찰하려는 결과가 promise 이후에 만들어진다면 반드시 기다려야 합니다.
이 규칙을 지키면 CI에서만 가끔 실패하는 타이밍 문제를 줄일 수 있습니다.

## 실무 체크리스트

### H3. 도입 전에 확인할 것

mock timers를 도입하기 전에 프로젝트의 Node.js 버전을 확인하세요.
Node.js test runner API는 버전에 따라 안정도와 세부 동작이 달라질 수 있으므로, CI와 로컬 개발 환경의 Node.js 버전을 맞추는 것이 중요합니다.
특히 test runner의 일부 기능은 문서에서 experimental 또는 early development로 표시될 수 있습니다.

- 실제 시간을 기다리는 단위 테스트가 있는가?
- 현재 시각 때문에 날짜별로 결과가 달라지는 테스트가 있는가?
- `setTimeout`, `setInterval`, `Date`, `node:timers/promises` 중 무엇을 mock해야 하는가?
- `tick()` 이후 확인해야 할 promise를 빠뜨리지 않았는가?
- interval이나 background 작업을 정리하는 경로가 있는가?

시간 의존 테스트는 느린 것보다 불명확한 것이 더 큰 문제입니다.
mock timers를 쓰면 테스트가 빨라지는 동시에 "언제 실행되어야 하는가"를 코드로 명확히 표현할 수 있습니다.
다만 외부 의존성까지 한 번에 해결하려고 하지 말고, 시간 제어라는 한 가지 책임에 집중해서 쓰는 편이 가장 안정적입니다.

---
layout: post
title: "Node.js MockTimers 가이드: Date, tick, setTime으로 시간 의존 테스트 고정하기"
date: 2026-09-02 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-mock-timers-date-settime-guide
permalink: /development/blog/seo/2026/09/02/nodejs-test-runner-mock-timers-date-settime-guide.html
alternates:
  ko: /development/blog/seo/2026/09/02/nodejs-test-runner-mock-timers-date-settime-guide.html
  x_default: /development/blog/seo/2026/09/02/nodejs-test-runner-mock-timers-date-settime-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, node-test, mocktimers, date, settimeout, timer-test, test-runner, ci]
description: "Node.js test runner의 MockTimers로 Date.now(), setTimeout, setInterval 테스트를 고정하는 방법을 정리합니다. tick(), setTime(), runAll(), timers/promises 사용 시 주의점을 실무 예제로 설명합니다."
---

시간에 의존하는 테스트는 작게 보이지만 CI를 불안정하게 만드는 대표적인 원인입니다.
`setTimeout`을 실제로 기다리면 테스트 시간이 늘고, `Date.now()`를 그대로 쓰면 실행 시점에 따라 결과가 달라집니다.
재시도 로직, 토큰 만료, 예약 발송, 캐시 TTL처럼 시간 조건이 들어간 코드는 특히 이런 문제가 자주 드러납니다.

Node.js 내장 테스트 러너의 MockTimers를 사용하면 실제 시간을 기다리지 않고 타이머와 날짜를 테스트 안에서 제어할 수 있습니다.
공식 문서 기준으로 `context.mock.timers.enable()`, `tick()`, `setTime()`, `runAll()`을 조합해 `Date`, `setTimeout`, `setInterval`, `node:timers/promises` 기반 코드를 검증할 수 있습니다.

이 글에서는 Node.js MockTimers를 실무 테스트에 적용하는 기준을 정리합니다.
테스트 컨텍스트의 자동 복구 흐름은 [Node.js test runner global setup/teardown 가이드](/development/blog/seo/2026/08/27/nodejs-test-runner-global-setup-teardown-guide.html), 비동기 작업 정리는 [Node.js test runner async activity cleanup 가이드](/development/blog/seo/2026/08/29/nodejs-test-runner-async-activity-cleanup-guide.html), 테스트 명령 실행은 [Node.js --run 가이드](/development/blog/seo/2026/09/01/nodejs-run-package-script-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js Test runner 공식 문서](https://nodejs.org/api/test.html), [MockTimers 클래스 공식 문서](https://nodejs.org/api/test.html#class-mocktimers)

## MockTimers가 필요한 순간

### 실제 시간을 기다리는 테스트는 느리고 흔들린다

아래 함수는 지정된 시간이 지난 뒤 작업을 실행합니다.
프로덕션 코드로는 단순하지만 테스트에서 그대로 5초를 기다리면 테스트 하나가 빌드 시간을 잡아먹습니다.

```js
export function delayWork(callback, ms) {
  return setTimeout(callback, ms);
}
```

실제 타이머에 기대는 테스트는 이런 모양이 되기 쉽습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { delayWork } from './delay-work.js';

test('runs after delay', async () => {
  let called = false;

  delayWork(() => {
    called = true;
  }, 5000);

  await new Promise((resolve) => setTimeout(resolve, 5100));
  assert.equal(called, true);
});
```

이 테스트는 느립니다.
더 큰 문제는 실행 환경이 느려졌을 때 실패할 수도 있다는 점입니다.
타이머 테스트의 목적은 "5초를 실제로 기다리는 것"이 아니라 "5초가 지났다고 가정했을 때 콜백이 실행되는지"를 확인하는 것입니다.

MockTimers를 쓰면 시간을 테스트 안에서 앞으로 보낼 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { delayWork } from './delay-work.js';

test('runs after delay', (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });

  let called = false;

  delayWork(() => {
    called = true;
  }, 5000);

  assert.equal(called, false);

  context.mock.timers.tick(5000);

  assert.equal(called, true);
});
```

`tick(5000)`은 mock clock을 5초 앞으로 이동시키고, 그 시간 안에 실행되어야 하는 타이머를 처리합니다.
이제 테스트는 실제 5초를 기다리지 않습니다.

### 테스트 컨텍스트의 mock을 우선 사용한다

MockTimers는 전역 `mock` 객체에서도 사용할 수 있지만, 일반적인 단위 테스트에서는 `TestContext`의 `context.mock`을 우선 쓰는 편이 좋습니다.
테스트가 끝난 뒤 해당 컨텍스트의 mock이 자동으로 복구되기 때문입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('uses isolated timers', (context) => {
  const callback = context.mock.fn();

  context.mock.timers.enable({ apis: ['setTimeout'] });
  setTimeout(callback, 1000);

  context.mock.timers.tick(1000);

  assert.equal(callback.mock.callCount(), 1);
});
```

타이머 mock은 프로세스 전역 API에 영향을 줍니다.
따라서 테스트 파일이 커질수록 "어떤 테스트가 mock을 켰고 언제 되돌렸는지"가 중요해집니다.
컨텍스트 기반 mock을 기본값으로 두면 테스트 사이의 오염 가능성을 줄일 수 있습니다.

## Date와 timer를 함께 고정하기

### Date.now는 now 옵션으로 시작점을 정한다

만료 시간, TTL, 날짜별 집계 코드는 `Date.now()`를 직접 읽는 경우가 많습니다.
이때 테스트가 실행된 실제 시각에 기대면 결과를 고정하기 어렵습니다.

```js
export function isExpired(expiresAtMs) {
  return Date.now() >= expiresAtMs;
}
```

MockTimers에서 `Date`를 활성화하면 `Date.now()`를 제어할 수 있습니다.
초기 시간이 필요하다면 `enable()`에 `now`를 넣습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { isExpired } from './token.js';

test('checks expiration against fixed date', (context) => {
  context.mock.timers.enable({
    apis: ['Date'],
    now: new Date('2026-09-02T00:00:00.000Z')
  });

  assert.equal(isExpired(Date.parse('2026-09-02T00:10:00.000Z')), false);

  context.mock.timers.tick(10 * 60 * 1000);

  assert.equal(isExpired(Date.parse('2026-09-02T00:10:00.000Z')), true);
});
```

`Date`와 타이머를 함께 mock하면 하나의 내부 시계처럼 움직입니다.
즉 `tick()`으로 시간을 앞으로 보내면 `Date.now()`도 함께 증가합니다.
시간 의존 로직을 검증할 때 이 특성을 이용하면 실제 대기 없이 경계 조건을 확인할 수 있습니다.

### setTime은 날짜를 이동하지만 지난 타이머를 즉시 실행하지 않는다

`setTime()`은 mock clock의 현재 시간을 특정 값으로 바꿀 때 사용합니다.
다만 중요한 차이가 있습니다.
`setTime()`으로 타이머 예약 시간을 지나가도, 그 타이머가 즉시 실행되지는 않습니다.
지난 타이머를 실행하려면 이후 `tick(0)`처럼 타이머 큐를 처리하는 동작을 호출해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('setTime moves date without flushing timers', (context) => {
  const callback = context.mock.fn();

  context.mock.timers.enable({
    apis: ['Date', 'setTimeout'],
    now: 0
  });

  setTimeout(callback, 1000);

  context.mock.timers.setTime(1500);

  assert.equal(Date.now(), 1500);
  assert.equal(callback.mock.callCount(), 0);

  context.mock.timers.tick(0);

  assert.equal(callback.mock.callCount(), 1);
});
```

이 차이는 테스트 의도를 선명하게 만드는 데 도움이 됩니다.
현재 날짜만 바꾸고 싶다면 `setTime()`을 사용하고, 예약된 타이머 실행까지 검증하고 싶다면 `tick()` 또는 `runAll()`을 사용합니다.

## setTimeout과 setInterval 테스트 패턴

### clearTimeout까지 같이 검증한다

타이머를 mock할 때 `setTimeout`을 활성화하면 관련 clear API도 함께 다뤄집니다.
취소 로직이 있는 코드는 콜백 실행 여부뿐 아니라 취소 뒤 실행되지 않는지도 확인해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function scheduleRetry(work) {
  const timeout = setTimeout(work, 3000);

  return () => {
    clearTimeout(timeout);
  };
}

test('does not run canceled retry', (context) => {
  const retry = context.mock.fn();

  context.mock.timers.enable({ apis: ['setTimeout'] });

  const cancel = scheduleRetry(retry);
  cancel();

  context.mock.timers.tick(3000);

  assert.equal(retry.mock.callCount(), 0);
});
```

이 테스트는 재시도 취소, 요청 중단, 컴포넌트 unmount cleanup처럼 "예약했지만 실행되면 안 되는 작업"에 잘 맞습니다.
실제 시간을 기다리지 않기 때문에 실패 원인을 타이밍 지연이 아니라 로직 자체로 좁힐 수 있습니다.

### runAll은 큐에 있는 타이머를 모두 비운다

`runAll()`은 현재 예약된 타이머를 모두 실행할 때 유용합니다.
여러 단계로 이어지는 타이머 체인을 한 번에 검증하고 싶을 때 사용할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function runTwoStepJob(log) {
  setTimeout(() => {
    log.push('prepare');

    setTimeout(() => {
      log.push('send');
    }, 2000);
  }, 1000);
}

test('runs all queued timer jobs', (context) => {
  const log = [];

  context.mock.timers.enable({ apis: ['setTimeout', 'Date'] });

  runTwoStepJob(log);
  context.mock.timers.runAll();

  assert.deepEqual(log, ['prepare', 'send']);
});
```

단, `runAll()`은 큐 전체를 비우는 강한 동작입니다.
타이머 사이의 중간 상태를 검증해야 한다면 `tick(1000)`, `tick(2000)`처럼 단계별로 이동하는 편이 더 읽기 좋습니다.

## timers/promises 사용 시 주의점

### 모듈 전체를 import하면 mock 대상이 된다

Node.js 문서에서는 MockTimers가 global timer뿐 아니라 `node:timers`, `node:timers/promises` 모듈의 타이머도 다룰 수 있다고 설명합니다.
다만 destructuring으로 가져온 함수는 현재 이 API에서 지원되지 않는다고 안내합니다.
테스트 대상 코드에서는 모듈 전체를 import하는 방식이 더 안전합니다.

```js
import timersPromises from 'node:timers/promises';

export async function waitAndReturn(value, ms) {
  await timersPromises.setTimeout(ms);
  return value;
}
```

테스트에서는 같은 mock clock으로 promise 기반 타이머를 진행시킬 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { waitAndReturn } from './wait-and-return.js';

test('resolves promise timer with mock clock', async (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });

  const result = waitAndReturn('done', 2000);

  context.mock.timers.tick(2000);

  assert.equal(await result, 'done');
});
```

이 패턴은 retry delay, polling interval, debounce 이후 비동기 작업처럼 promise로 대기 시간을 표현한 코드에 좋습니다.
테스트 대상 코드가 `import { setTimeout } from 'node:timers/promises'`처럼 destructuring을 사용하고 있다면 mock 적용 여부를 먼저 확인해야 합니다.

### setInterval async iterator는 종료 조건을 분명히 둔다

`node:timers/promises`의 `setInterval`은 async iterator로 사용할 수 있습니다.
MockTimers로도 제어할 수 있지만, 테스트에서는 반복 종료 조건을 반드시 분명히 두어야 합니다.

```js
import timersPromises from 'node:timers/promises';

export async function collectHeartbeats(count, intervalMs) {
  const values = [];

  for await (const value of timersPromises.setInterval(intervalMs, 'beat')) {
    values.push(value);

    if (values.length === count) {
      break;
    }
  }

  return values;
}
```

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { collectHeartbeats } from './heartbeat.js';

test('collects interval ticks without waiting', async (context) => {
  context.mock.timers.enable({ apis: ['setInterval'] });

  const result = collectHeartbeats(3, 1000);

  context.mock.timers.tick(1000);
  context.mock.timers.tick(1000);
  context.mock.timers.tick(1000);

  assert.deepEqual(await result, ['beat', 'beat', 'beat']);
});
```

반복이 끝나지 않는 구조를 테스트하면 `runAll()` 같은 호출이 끝나지 않는 상황을 만들 수 있습니다.
interval 테스트에서는 "몇 번 반복한 뒤 종료한다"는 조건을 코드나 테스트 안에 명시해야 합니다.

## 도입 전 체크리스트

### mock할 API를 좁게 켠다

`enable()`의 `apis`에는 필요한 API만 넣는 편이 좋습니다.
예를 들어 날짜만 검증한다면 `['Date']`, 단순 timeout만 검증한다면 `['setTimeout']`으로 충분합니다.

```js
context.mock.timers.enable({ apis: ['Date'] });
context.mock.timers.enable({ apis: ['setTimeout'] });
context.mock.timers.enable({ apis: ['Date', 'setTimeout'] });
```

mock 범위가 넓을수록 테스트가 읽어야 하는 전역 변화도 늘어납니다.
작게 켜면 이 테스트가 어떤 시간 API를 제어하는지 바로 보입니다.

### 병렬 테스트와 공유 상태를 조심한다

타이머 mock은 전역 시간 API를 바꾸기 때문에 병렬 실행되는 테스트와 충돌할 수 있습니다.
테스트 파일 안에서 MockTimers를 많이 사용한다면 아래 기준을 점검하는 편이 좋습니다.

- 시간 mock을 켠 테스트가 같은 파일의 다른 테스트와 전역 상태를 공유하지 않는가?
- 테스트 대상 모듈이 import 시점에 timer 함수를 캡처하지 않는가?
- destructuring import 때문에 mock이 적용되지 않는 경로가 없는가?
- interval 테스트에 명확한 종료 조건이 있는가?
- 날짜 이동만 필요한 곳과 타이머 실행까지 필요한 곳을 구분했는가?

특히 flaky test를 줄이려는 목적이라면 MockTimers 자체보다 테스트 대상 코드의 의존성 주입 구조가 더 중요할 때도 있습니다.
예를 들어 현재 시각을 읽는 함수를 파라미터로 받게 만들면 굳이 전역 `Date`를 mock하지 않아도 되는 테스트가 생깁니다.

```js
export function shouldRefreshToken(token, now = Date.now()) {
  return token.expiresAt - now < 60_000;
}
```

이런 함수는 순수하게 값만 넣어 검증할 수 있습니다.
MockTimers는 전역 시간 API를 사용하는 코드, 외부 인터페이스상 타이머를 피하기 어려운 코드에 집중해서 쓰는 편이 좋습니다.

## FAQ

### MockTimers는 Jest fake timers와 같은 건가요?

목적은 비슷하지만 API와 적용 범위는 다릅니다.
Node.js MockTimers는 `node:test`의 mock 기능이며, Node.js 런타임의 timer API와 함께 동작하도록 설계되어 있습니다.
기존 Jest 테스트를 그대로 옮기기보다 `enable()`, `tick()`, `setTime()`, `runAll()` 기준으로 테스트 의도를 다시 표현하는 편이 좋습니다.

### setTime과 tick은 어떻게 구분하나요?

`setTime()`은 mock clock의 현재 시각을 특정 값으로 이동합니다.
그 자체로 지난 타이머를 즉시 실행하지는 않습니다.
`tick()`은 시간을 앞으로 진행하면서 실행 대상이 된 타이머를 처리합니다.
날짜 상태만 바꿀 때는 `setTime()`, 타이머 콜백 실행까지 검증할 때는 `tick()`을 우선 고려하면 됩니다.

### node:timers/promises도 테스트할 수 있나요?

가능합니다.
MockTimers는 promise 기반 timeout과 interval도 다룰 수 있습니다.
다만 destructuring import는 지원되지 않는다고 문서에 안내되어 있으므로, 테스트 대상 코드에서는 `import timersPromises from 'node:timers/promises'`처럼 모듈 객체를 통해 호출하는 방식을 우선 검토하는 편이 좋습니다.

## 마무리

Node.js MockTimers는 시간 의존 테스트를 빠르고 예측 가능하게 만드는 도구입니다.
`Date.now()`를 고정하고, `setTimeout`을 실제 대기 없이 실행하며, promise 기반 타이머까지 같은 테스트 흐름 안에서 검증할 수 있습니다.

도입할 때는 mock 범위를 작게 유지하고 `setTime()`과 `tick()`의 역할을 구분해야 합니다.
날짜만 바꾸는 테스트, 타이머 큐를 진행시키는 테스트, 모든 예약 작업을 비우는 테스트를 명확히 나누면 CI에서 흔들리는 시간 테스트를 줄일 수 있습니다.

---
layout: post
title: "Node.js test runner MockTimers 가이드: 시간 의존 테스트를 빠르고 안정적으로 만드는 법"
date: 2026-08-03 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-mock-timers-time-dependent-test-guide
permalink: /development/blog/seo/2026/08/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html
alternates:
  ko: /development/blog/seo/2026/08/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html
  x_default: /development/blog/seo/2026/08/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, mocktimers, timers, date, testing, ci, javascript]
description: "Node.js test runner의 MockTimers로 setTimeout, setInterval, Date.now 같은 시간 의존 코드를 실제 대기 없이 테스트하는 방법을 정리합니다. tick, runAll, setTime, 격리 기준과 CI 주의점까지 예제로 설명합니다."
---

시간에 의존하는 코드는 테스트가 느려지거나 흔들리기 쉽습니다.
재시도 로직은 몇 초를 기다려야 하고, 캐시 만료는 분 단위로 시간이 지나야 하며, 예약 작업은 특정 시각이 되어야 동작합니다.
이런 코드를 그대로 테스트하면 테스트 시간이 길어지고, CI 부하나 실행 순서에 따라 결과가 달라질 수 있습니다.

Node.js 내장 test runner는 `context.mock.timers`를 통해 타이머와 날짜를 가짜 시간으로 제어할 수 있습니다.
`setTimeout()`을 실제로 기다리지 않고 `tick()`으로 시간을 앞으로 밀거나, `Date.now()`의 기준 시각을 고정해 날짜 의존 테스트를 반복 가능하게 만들 수 있습니다.
중요한 점은 MockTimers가 단순한 속도 개선 도구가 아니라, 시간이라는 전역 상태를 테스트 안으로 끌어와 통제하는 장치라는 것입니다.

이 글에서는 Node.js test runner의 MockTimers를 실무 테스트에 적용하는 기준을 정리합니다.
테스트 구조를 먼저 정리해야 한다면 [Node.js test runner subtest 가이드](/development/blog/seo/2026/05/24/nodejs-test-runner-subtest-structure-guide.html)를 함께 보면 좋습니다.
실행 순서 의존성을 점검하려면 [Node.js test runner randomize seed 가이드](/development/blog/seo/2026/07/27/nodejs-test-runner-randomize-seed-order-dependent-guide.html)가 도움이 됩니다.
스냅샷에 시간 값이 섞이는 문제는 [Node.js test runner snapshot testing 가이드](/development/blog/seo/2026/07/17/nodejs-test-runner-snapshot-testing-guide.html)와 연결해서 관리할 수 있습니다.

## MockTimers가 필요한 상황

### H3. 실제 대기 시간을 테스트에서 제거한다

가장 단순한 예는 지연 실행입니다.
아래 함수는 3초 뒤 콜백을 실행합니다.

```js
export function scheduleRefresh(refresh) {
  return setTimeout(refresh, 3000);
}
```

이 함수를 실제 시간으로 테스트하면 테스트 하나가 최소 3초를 소비합니다.
테스트가 몇 개만 늘어도 CI 시간이 불필요하게 길어집니다.
MockTimers를 쓰면 타이머를 등록한 뒤 가짜 시간을 3초만큼 진행시켜 즉시 검증할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { scheduleRefresh } from '../src/schedule-refresh.js';

test('scheduleRefresh runs after 3 seconds', (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });

  const refresh = context.mock.fn();
  scheduleRefresh(refresh);

  assert.equal(refresh.mock.callCount(), 0);

  context.mock.timers.tick(3000);

  assert.equal(refresh.mock.callCount(), 1);
});
```

테스트는 실제로 3초를 기다리지 않습니다.
`tick(3000)`이 mock timer의 시간을 앞으로 보내고, 그 시점까지 실행되어야 할 타이머를 처리합니다.
대기 시간 자체가 요구사항이라면 "얼마나 기다렸는가"를 실제 벽시계로 재는 대신 "어떤 시간이 지나면 실행되는가"를 검증하는 편이 더 안정적입니다.

### H3. 날짜 기준을 고정한다

시간 의존 테스트에서 `setTimeout()`만큼 자주 문제를 만드는 것이 `Date.now()`입니다.
오늘 날짜, 만료 시각, 로그 타임스탬프, 토큰 유효 기간 같은 값이 현재 시각에 묶이면 같은 테스트도 실행한 날짜에 따라 다른 결과를 만들 수 있습니다.

```js
export function isExpired(expiresAt) {
  return Date.now() >= expiresAt;
}
```

MockTimers는 `Date`도 함께 mock할 수 있습니다.
기준 시각을 고정하면 만료 여부를 명확하게 테스트할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { isExpired } from '../src/session-expiry.js';

test('isExpired compares against fixed current time', (context) => {
  context.mock.timers.enable({
    apis: ['Date'],
    now: Date.parse('2026-08-03T11:00:00.000Z')
  });

  assert.equal(
    isExpired(Date.parse('2026-08-03T11:05:00.000Z')),
    false
  );

  assert.equal(
    isExpired(Date.parse('2026-08-03T10:59:59.000Z')),
    true
  );
});
```

이 테스트는 실행하는 지역, 시간, CI 큐 대기 시간에 영향을 받지 않습니다.
테스트 이름에도 "현재 시각"이 고정된다는 의도를 드러내면, 나중에 코드를 읽는 사람이 왜 MockTimers가 필요한지 바로 이해할 수 있습니다.

## tick, runAll, setTime의 차이

### H3. tick은 시간을 조금씩 앞으로 보낸다

`tick(milliseconds)`는 mock timer의 시간을 지정한 밀리초만큼 진행합니다.
여러 타이머가 서로 다른 지연 시간을 갖는 코드를 테스트할 때 유용합니다.

```js
export function scheduleRetry(task) {
  setTimeout(() => task('first'), 1000);
  setTimeout(() => task('second'), 3000);
}
```

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { scheduleRetry } from '../src/retry.js';

test('scheduleRetry runs retries in order', (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });

  const task = context.mock.fn();
  scheduleRetry(task);

  context.mock.timers.tick(1000);
  assert.equal(task.mock.callCount(), 1);
  assert.equal(task.mock.calls[0].arguments[0], 'first');

  context.mock.timers.tick(2000);
  assert.equal(task.mock.callCount(), 2);
  assert.equal(task.mock.calls[1].arguments[0], 'second');
});
```

첫 번째 `tick(1000)`은 1초 지점까지의 타이머만 실행합니다.
두 번째 `tick(2000)`은 총 3초 지점까지 시간을 진행하므로 두 번째 타이머가 실행됩니다.
이 방식은 지연 단계 자체가 요구사항인 재시도, 백오프, 알림 예약 테스트에 잘 맞습니다.

### H3. runAll은 남은 타이머를 모두 처리한다

`runAll()`은 현재 큐에 있는 타이머를 모두 실행합니다.
정확한 중간 시점보다 "예약된 일이 전부 끝났는가"가 중요할 때 적합합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

test('runAll flushes pending timers', (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });

  const events = [];

  setTimeout(() => events.push('slow'), 5000);
  setTimeout(() => events.push('fast'), 1000);

  context.mock.timers.runAll();

  assert.deepEqual(events, ['fast', 'slow']);
});
```

`runAll()`은 편하지만, 모든 시간 차이를 감춰 버릴 수 있습니다.
중간 상태가 중요한 테스트라면 `tick()`으로 단계별 검증을 남기는 편이 낫습니다.
반대로 정리 작업, debounce flush, 예약된 콜백 전체 실행처럼 최종 상태만 보면 되는 테스트라면 `runAll()`이 더 읽기 쉽습니다.

### H3. setTime은 현재 시각을 옮긴다

`setTime(milliseconds)`는 mock된 현재 시각을 특정 값으로 바꿉니다.
다만 이미 지난 시각에 예약된 타이머를 자동으로 실행하는 동작으로 이해하면 안 됩니다.
타이머 실행까지 검증하려면 `tick()` 또는 `runAll()`을 함께 사용해야 합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

test('setTime changes Date without flushing timers by itself', (context) => {
  context.mock.timers.enable({ apis: ['Date', 'setTimeout'] });

  const notify = context.mock.fn();
  setTimeout(notify, 1000);

  context.mock.timers.setTime(1200);

  assert.equal(Date.now(), 1200);
  assert.equal(notify.mock.callCount(), 0);

  context.mock.timers.tick(0);

  assert.equal(notify.mock.callCount(), 1);
});
```

이 차이를 모르면 테스트가 우연히 통과하거나, Node.js 버전과 테스트 구조에 따라 의미가 흐려질 수 있습니다.
`setTime()`은 날짜 기준을 바꾸는 도구, `tick()`은 타이머 진행을 발생시키는 도구라고 나누어 생각하면 안전합니다.

## 실무 예제: 캐시 만료 테스트

### H3. 만료 시간과 재계산을 분리해서 검증한다

캐시는 시간 의존 테스트의 대표 사례입니다.
TTL이 지나기 전에는 기존 값을 돌려주고, TTL이 지나면 새 값을 계산해야 합니다.

```js
export function createTtlCache({ ttlMs, load }) {
  let cached;
  let expiresAt = 0;

  return async function getValue() {
    const now = Date.now();

    if (cached !== undefined && now < expiresAt) {
      return cached;
    }

    cached = await load();
    expiresAt = now + ttlMs;
    return cached;
  };
}
```

MockTimers를 사용하면 테스트에서 TTL 경계를 정확하게 지나갈 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { createTtlCache } from '../src/ttl-cache.js';

test('ttl cache reloads after expiration', async (context) => {
  context.mock.timers.enable({
    apis: ['Date'],
    now: Date.parse('2026-08-03T00:00:00.000Z')
  });

  let version = 0;
  const load = context.mock.fn(async () => {
    version += 1;
    return `value-${version}`;
  });

  const getValue = createTtlCache({ ttlMs: 1000, load });

  assert.equal(await getValue(), 'value-1');
  assert.equal(await getValue(), 'value-1');
  assert.equal(load.mock.callCount(), 1);

  context.mock.timers.tick(999);
  assert.equal(await getValue(), 'value-1');

  context.mock.timers.tick(1);
  assert.equal(await getValue(), 'value-2');
  assert.equal(load.mock.callCount(), 2);
});
```

이 테스트는 "만료 직전"과 "만료 시점"을 나눠 확인합니다.
경계값 테스트는 캐시, 세션, 토큰, rate limit처럼 시간 비교가 들어가는 코드에서 특히 중요합니다.
실제 시간을 기다리는 방식으로는 이런 경계를 촘촘하게 검증하기 어렵습니다.

### H3. 현재 시각 주입과 MockTimers를 함께 고려한다

모든 코드가 반드시 `Date.now()`를 직접 호출해야 하는 것은 아닙니다.
도메인 로직은 `now()` 함수를 주입받게 만들고, 통합 수준 테스트에서만 MockTimers를 쓰는 설계도 좋습니다.

```js
export function createSessionPolicy({ now = Date.now, ttlMs }) {
  return {
    expiresAt() {
      return now() + ttlMs;
    },
    isValid(expiresAt) {
      return now() < expiresAt;
    }
  };
}
```

순수 로직은 주입으로 테스트하고, 실제 런타임 API와 연결되는 부분은 MockTimers로 테스트하면 역할이 분명해집니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { createSessionPolicy } from '../src/session-policy.js';

test('session policy can be tested with injected clock', () => {
  const policy = createSessionPolicy({
    now: () => 1000,
    ttlMs: 500
  });

  assert.equal(policy.expiresAt(), 1500);
  assert.equal(policy.isValid(1499), true);
  assert.equal(policy.isValid(1000), false);
});
```

MockTimers는 강력하지만 전역 타이머와 날짜 API를 바꿉니다.
따라서 작은 도메인 함수까지 모두 mock timer에 기대기보다, 시간 주입으로 충분한 곳과 실제 timer API 검증이 필요한 곳을 나누는 편이 유지보수에 좋습니다.

## 격리와 정리 기준

### H3. 테스트 컨텍스트의 mock을 우선 사용한다

Node.js test runner에서는 테스트 함수가 받는 `context`의 `mock`을 사용하는 방식이 가장 명확합니다.
테스트마다 독립된 컨텍스트를 사용하면 다른 테스트에 영향을 줄 가능성을 줄일 수 있습니다.

```js
test('uses per-test mock context', (context) => {
  context.mock.timers.enable({ apis: ['Date'], now: 100 });
  assert.equal(Date.now(), 100);
});
```

전역 mock tracker를 공유하거나, 여러 테스트가 같은 타이머 상태를 전제로 삼으면 순서 의존성이 생깁니다.
테스트가 많아질수록 이런 의존성은 CI에서만 드러나는 실패로 이어질 수 있습니다.
시간 mock은 가능한 한 테스트 하나 안에서 켜고, 그 테스트 안에서 필요한 검증을 끝내는 편이 좋습니다.

### H3. 병렬 테스트에서 전역 시간 변경을 조심한다

`Date`, `setTimeout`, `setInterval`은 전역 API입니다.
MockTimers는 테스트 컨텍스트를 통해 사용하더라도 코드가 관찰하는 대상은 전역 시간입니다.
같은 파일이나 같은 프로세스에서 병렬로 시간 의존 테스트가 실행되면 서로 영향을 줄 수 있습니다.

시간을 mock하는 테스트는 다음 기준을 두면 안정적입니다.

- 같은 파일 안에서 시간 mock 테스트를 과하게 병렬화하지 않는다.
- 테스트 하나가 끝난 뒤 남은 타이머를 전제로 다른 테스트가 동작하지 않게 한다.
- `Date.now()` 결과를 스냅샷이나 로그에 그대로 남길 때는 고정 시각을 사용한다.
- 랜덤 실행 순서에서도 통과하는지 주기적으로 확인한다.
- 테스트 실패 메시지에는 실제 토큰, 사용자 식별자, 내부 전체 경로를 남기지 않는다.

특히 `--test-concurrency`를 높이거나 랜덤 실행 순서를 도입한 프로젝트에서는 시간 mock 테스트를 별도 파일로 묶는 것도 실용적인 선택입니다.
핵심은 "시간이 전역 상태"라는 사실을 테스트 설계에 반영하는 것입니다.

## 타이머 API별 주의점

### H3. setInterval은 종료 조건을 테스트에 둔다

반복 작업은 테스트에서 무한히 진행되지 않도록 종료 조건을 명확히 해야 합니다.
예를 들어 주기적으로 상태를 수집하는 함수가 있다면, 테스트에서는 몇 번 실행됐는지까지만 확인하고 interval을 정리하는 구조가 필요합니다.

```js
export function startPolling(poll, intervalMs = 1000) {
  const timer = setInterval(poll, intervalMs);
  return () => clearInterval(timer);
}
```

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { startPolling } from '../src/polling.js';

test('startPolling runs on interval and can be stopped', (context) => {
  context.mock.timers.enable({
    apis: ['setInterval', 'clearInterval']
  });

  const poll = context.mock.fn();
  const stop = startPolling(poll, 1000);

  context.mock.timers.tick(3000);
  assert.equal(poll.mock.callCount(), 3);

  stop();
  context.mock.timers.tick(3000);
  assert.equal(poll.mock.callCount(), 3);
});
```

`clearInterval`까지 mock 대상에 포함하면 정리 동작도 같은 가짜 시간 안에서 검증할 수 있습니다.
반복 타이머는 "몇 번 실행되는가"뿐 아니라 "언제 멈추는가"까지 함께 테스트해야 운영 장애를 줄일 수 있습니다.

### H3. timers/promises는 비동기 흐름을 기다린다

`node:timers/promises`를 사용하는 코드는 타이머가 Promise로 표현됩니다.
가짜 시간을 진행한 뒤에는 해당 Promise를 `await`해서 비동기 흐름이 끝났는지 확인해야 합니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

export async function delayedValue(value) {
  await delay(1000);
  return value;
}
```

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { delayedValue } from '../src/delayed-value.js';

test('delayedValue resolves after mocked delay', async (context) => {
  context.mock.timers.enable({ apis: ['setTimeout'] });

  const result = delayedValue('ready');

  context.mock.timers.tick(1000);

  assert.equal(await result, 'ready');
});
```

`tick()`을 호출했다고 해서 테스트 함수가 자동으로 Promise 결과를 검증하는 것은 아닙니다.
비동기 함수가 반환한 Promise를 반드시 기다려야 누락된 rejection이나 후속 로직 실패를 잡을 수 있습니다.

## CI에서의 운영 체크리스트

### H3. 빠른 테스트보다 반복 가능한 테스트를 목표로 한다

MockTimers를 도입하면 테스트가 빨라집니다.
하지만 더 중요한 목표는 반복 가능성입니다.
시간을 고정하고, 타이머 진행을 명시하고, 전역 상태를 테스트 안에 가두면 CI에서만 발생하는 실패를 줄일 수 있습니다.

도입 전후로 다음 항목을 점검하세요.

- 실제 `setTimeout` 대기를 `tick()` 또는 `runAll()`로 바꿨는가?
- 날짜 비교 테스트에서 `Date.now()` 기준을 고정했는가?
- `setTime()`과 `tick()`의 역할을 혼동하지 않았는가?
- interval 테스트에는 종료 조건과 정리 검증이 있는가?
- Promise 기반 timer 테스트에서 결과 Promise를 `await`하는가?
- 테스트 파일 간 전역 시간 mock이 섞이지 않도록 구성했는가?
- 스냅샷, 로그, 실패 메시지에 민감정보가 들어가지 않는가?
- 랜덤 실행 순서나 병렬 실행에서도 테스트 의미가 유지되는가?

체크리스트의 핵심은 MockTimers를 "기다림 제거"가 아니라 "시간 제어"로 보는 것입니다.
시간을 제어하면 테스트가 빨라지는 것은 자연스러운 결과이고, 진짜 가치는 실패 원인이 더 선명해지는 데 있습니다.

## FAQ

### H3. MockTimers를 쓰면 모든 시간 테스트에서 now 주입이 필요 없나요?

아닙니다.
도메인 로직은 `now` 함수를 주입받게 만들면 더 단순하게 테스트할 수 있습니다.
MockTimers는 실제 `Date`, `setTimeout`, `setInterval`, `node:timers/promises` 같은 런타임 API와 연결된 동작을 검증할 때 특히 유용합니다.
순수 함수 수준에서는 주입을, 런타임 경계에서는 MockTimers를 쓰는 식으로 나누는 편이 좋습니다.

### H3. tick과 runAll 중 무엇을 기본으로 써야 하나요?

중간 상태가 요구사항이면 `tick()`이 좋습니다.
1초 뒤에는 첫 번째 재시도만 실행되고, 3초 뒤에는 두 번째 재시도까지 실행되는 식의 단계 검증을 남길 수 있기 때문입니다.
중간 상태가 중요하지 않고 대기 중인 타이머를 모두 비우는 것이 목적이라면 `runAll()`이 더 읽기 쉽습니다.

### H3. Date만 mock해도 setTimeout이 같이 제어되나요?

아닙니다.
`enable({ apis: [...] })`에 어떤 API를 mock할지 명시해야 합니다.
`Date.now()`만 고정하고 싶다면 `Date`를, 타이머 실행까지 제어하고 싶다면 `setTimeout`, `setInterval`, `clearInterval` 같은 API를 필요한 만큼 포함하세요.
날짜와 타이머가 함께 움직여야 하는 테스트라면 `apis: ['Date', 'setTimeout']`처럼 같이 지정하는 편이 명확합니다.

## 마무리

Node.js test runner의 MockTimers는 시간 의존 코드를 빠르고 반복 가능하게 테스트하는 도구입니다.
`tick()`으로 시간을 단계적으로 진행하고, `runAll()`로 남은 타이머를 비우며, `setTime()`으로 현재 시각을 고정하면 대기 없는 테스트를 만들 수 있습니다.

실무에서는 모든 테스트를 mock timer로 덮기보다 시간 주입, 작은 단위 테스트, 런타임 경계 테스트를 함께 설계하는 것이 좋습니다.
그 기준만 지키면 캐시 만료, 재시도, polling, 예약 작업, 날짜 계산처럼 흔들리기 쉬운 코드를 CI에서도 안정적으로 검증할 수 있습니다.

## 참고 자료

- [Node.js Test runner documentation](https://nodejs.org/api/test.html)
- [Node.js Timers documentation](https://nodejs.org/api/timers.html)

---
layout: post
title: "Node.js scheduler.wait 가이드: 취소 가능한 대기 유틸 안전하게 만들기"
date: 2026-07-21 20:00:00 +0900
lang: ko
translation_key: nodejs-timers-promises-scheduler-wait-cancellable-delay-guide
permalink: /development/blog/seo/2026/07/21/nodejs-timers-promises-scheduler-wait-cancellable-delay-guide.html
alternates:
  ko: /development/blog/seo/2026/07/21/nodejs-timers-promises-scheduler-wait-cancellable-delay-guide.html
  x_default: /development/blog/seo/2026/07/21/nodejs-timers-promises-scheduler-wait-cancellable-delay-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, timers-promises, scheduler-wait, abortsignal, timeout, retry, polling, backend]
description: "Node.js timers/promises의 scheduler.wait()와 AbortSignal로 취소 가능한 대기 유틸을 만드는 방법을 정리합니다. retry, polling, graceful shutdown, ref 옵션, 테스트 기준까지 실무 예제로 설명합니다."
---

Node.js 코드에서 "잠깐 기다렸다가 다시 시도한다"는 로직은 흔합니다.
재시도 사이에 backoff를 넣고, polling 주기를 맞추고, shutdown 중에는 더 이상 기다리지 않고 빠져나오고, 테스트에서는 실제 시간을 오래 쓰지 않게 만들어야 합니다.
처음에는 `new Promise(resolve => setTimeout(resolve, ms))` 한 줄이면 충분해 보이지만, 운영 코드가 되면 취소, 프로세스 종료, 에러 처리 기준이 곧바로 따라옵니다.

`node:timers/promises`의 `scheduler.wait()`는 Promise 기반으로 일정 시간 대기할 수 있는 API입니다.
Node.js 공식 문서 기준으로 `scheduler.wait(delay, options)`는 `setTimeout(delay, undefined, options)`와 같은 동작을 하는 Scheduling APIs 계열의 Experimental API이며, `AbortSignal`과 `ref` 옵션을 받을 수 있습니다.
이 글에서는 `scheduler.wait()`를 직접 쓰는 방법보다, 서비스 코드에서 재사용 가능한 취소 가능 대기 유틸로 감싸는 기준을 정리합니다.
[Node.js timers/promises scheduler.yield 가이드](/development/blog/seo/2026/05/20/nodejs-timers-promises-scheduler-yield-event-loop-guide.html), [Node.js AbortSignal timeout 취소 패턴 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html), [Node.js retry budget 가이드](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html)와 함께 보면 비동기 대기와 재시도 흐름을 더 안정적으로 설계할 수 있습니다.

## scheduler.wait가 필요한 상황

### H3. 임시 sleep 함수가 운영 코드로 새어 나올 때

개발 중에는 아래처럼 간단한 sleep 함수를 자주 만듭니다.

```js
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
```

작은 스크립트에서는 괜찮습니다.
하지만 이 함수가 API 서버, worker, queue consumer, 배치 작업으로 들어가면 부족한 점이 드러납니다.
대기 중인 작업을 shutdown 신호로 취소할 수 없고, 테스트에서 시간을 제어하기 어렵고, 대기 timer가 프로세스 종료를 붙잡는지 여부도 호출자가 선택할 수 없습니다.

운영 코드의 대기는 단순한 멈춤이 아니라 제어 가능한 상태여야 합니다.
특히 retry, polling, rate limit, graceful shutdown처럼 실패와 종료가 엮이는 곳에서는 `AbortSignal`을 받을 수 있는 대기 함수가 기본값이 되는 편이 안전합니다.

### H3. Promise 기반 타이머는 async 흐름과 잘 맞는다

`scheduler.wait()`는 콜백을 넘기는 대신 Promise를 반환합니다.
그래서 `await`과 `try/catch` 안에서 읽기 쉽고, 실패 경로를 한곳에서 정리하기 좋습니다.

```js
import { scheduler } from 'node:timers/promises';

await scheduler.wait(500);
```

이 한 줄은 500ms 뒤에 resolve됩니다.
다만 실무에서는 여기에 `signal`, `ref`, 에러 이름 처리, 입력값 검증을 붙여야 합니다.
API 자체를 모든 곳에 직접 흩뿌리기보다 얇은 유틸을 두면 정책을 바꾸기 쉽습니다.

## 취소 가능한 delay 유틸 만들기

### H3. AbortSignal을 옵션으로 받는다

먼저 기본 delay 함수를 만듭니다.
핵심은 호출자가 이미 가지고 있는 `AbortSignal`을 그대로 전달하는 것입니다.
HTTP 요청, queue job, shutdown controller가 같은 signal을 공유하면 대기 중인 작업도 같은 취소 흐름을 따를 수 있습니다.

```js
import { scheduler } from 'node:timers/promises';

export async function delay(ms, options = {}) {
  const duration = normalizeDelay(ms);

  await scheduler.wait(duration, {
    signal: options.signal,
    ref: options.ref ?? true
  });
}

function normalizeDelay(ms) {
  if (!Number.isFinite(ms) || ms < 0) {
    throw new TypeError('delay must be a non-negative finite number');
  }

  return Math.trunc(ms);
}
```

`normalizeDelay()`를 두는 이유는 실수로 `NaN`, 음수, 문자열이 들어오는 상황을 빨리 잡기 위해서입니다.
대기 시간은 종종 설정값이나 외부 응답에서 만들어지므로, 경계값을 조용히 넘기면 장애 상황에서 원인을 찾기 어려워집니다.

### H3. AbortError는 정상적인 취소 경로로 다룬다

`AbortSignal`로 대기를 취소하면 Promise는 `AbortError`로 reject될 수 있습니다.
이 에러를 무조건 장애로 로그에 남기면 shutdown, timeout, 사용자 요청 취소가 모두 오류처럼 보입니다.
호출부에서는 취소와 실패를 구분해야 합니다.

```js
export function isAbortError(error) {
  return error?.name === 'AbortError' || error?.code === 'ABORT_ERR';
}
```

사용 예시는 아래처럼 단순합니다.

```js
import { delay, isAbortError } from './delay.js';

export async function waitBeforeRetry(ms, signal) {
  try {
    await delay(ms, { signal });
    return 'continued';
  } catch (error) {
    if (isAbortError(error)) {
      return 'cancelled';
    }

    throw error;
  }
}
```

취소는 "실패"가 아니라 "더 진행하지 말라"는 제어 신호에 가깝습니다.
관측 로그에서도 `level: info`, `outcome: cancelled`처럼 따로 분류하면 실제 장애와 섞이지 않습니다.

## retry와 polling에 적용하기

### H3. retry 대기는 실패 사이의 완충 장치다

재시도 로직에서 대기 시간은 단순히 느리게 만드는 장치가 아닙니다.
하류 서비스가 회복할 시간을 주고, 같은 요청이 동시에 몰리는 상황을 줄이고, caller가 포기해야 할 때 빠르게 멈추게 만드는 장치입니다.

```js
import { delay } from './delay.js';

export async function retry(operation, options = {}) {
  const {
    attempts = 3,
    baseDelayMs = 100,
    signal
  } = options;

  let lastError;

  for (let attempt = 1; attempt <= attempts; attempt += 1) {
    try {
      return await operation({ attempt, signal });
    } catch (error) {
      lastError = error;

      if (attempt === attempts) {
        break;
      }

      const waitMs = baseDelayMs * 2 ** (attempt - 1);
      await delay(waitMs, { signal, ref: false });
    }
  }

  throw lastError;
}
```

여기서 `ref: false`를 선택한 이유는 재시도 대기 timer만 남았을 때 프로세스를 억지로 붙잡지 않기 위해서입니다.
CLI나 worker shutdown 흐름에서는 이 차이가 중요할 수 있습니다.
반대로 서버 요청 처리 중 반드시 대기가 끝나야 하는 작업이라면 기본값인 `ref: true`를 유지하는 편이 더 자연스럽습니다.

### H3. polling은 종료 조건과 취소 조건을 함께 둔다

polling 루프는 `while (true)`로 시작하기 쉽지만, 종료 조건이 불명확하면 장애가 됩니다.
성공 조건, 최대 시도 횟수, 외부 취소 신호를 함께 받아야 합니다.

```js
import { delay } from './delay.js';

export async function pollUntil(readState, options = {}) {
  const {
    intervalMs = 1000,
    maxAttempts = 30,
    signal
  } = options;

  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    const state = await readState({ attempt, signal });

    if (state.done) {
      return state.value;
    }

    await delay(intervalMs, { signal, ref: false });
  }

  throw new Error(`polling timed out after ${maxAttempts} attempts`);
}
```

이 패턴은 배포 상태 확인, 외부 API 처리 완료 확인, background job 상태 조회에 잘 맞습니다.
다만 polling 대상이 실패 상태를 반환한다면 `done: false`로 계속 기다리지 말고 즉시 예외를 던져야 합니다.
기다림은 일시적인 미완료 상태에만 써야 하고, 이미 실패가 확정된 상태를 가리는 데 쓰면 안 됩니다.

## graceful shutdown과 연결하기

### H3. shutdown signal을 모든 대기에 전달한다

서비스 종료 중에 새 작업을 받지 않는 것만으로는 부족합니다.
이미 retry 대기나 polling 대기 중인 작업도 빠져나와야 합니다.
공통 `AbortController`를 만들고 종료 이벤트에서 abort하면 대기 중인 함수들이 같은 규칙으로 멈춥니다.

```js
const shutdownController = new AbortController();

process.once('SIGTERM', () => {
  shutdownController.abort(new Error('process is shutting down'));
});

await pollUntil(readDeploymentStatus, {
  intervalMs: 2000,
  maxAttempts: 60,
  signal: shutdownController.signal
});
```

이 구조에서는 polling이 다음 interval을 기다리는 중이어도 `SIGTERM`을 받으면 즉시 reject됩니다.
대기 함수가 signal을 무시하면 shutdown 시간이 길어지고, orchestrator가 강제 종료를 걸 가능성이 커집니다.

### H3. 요청 timeout과 shutdown 취소를 합친다

실무에서는 하나의 취소 이유만 있지 않습니다.
요청별 timeout도 있고, 프로세스 shutdown도 있고, 상위 작업 취소도 있습니다.
여러 signal을 합칠 때는 `AbortSignal.any()`를 사용할 수 있습니다.

```js
export function combineSignals(...signals) {
  const activeSignals = signals.filter(Boolean);

  if (activeSignals.length === 0) {
    return undefined;
  }

  if (activeSignals.length === 1) {
    return activeSignals[0];
  }

  return AbortSignal.any(activeSignals);
}
```

그리고 호출부에서 timeout과 shutdown을 함께 묶습니다.

```js
const signal = combineSignals(
  AbortSignal.timeout(10_000),
  shutdownController.signal
);

await retry(sendWebhook, {
  attempts: 4,
  baseDelayMs: 250,
  signal
});
```

이렇게 하면 10초가 지나도 멈추고, 프로세스 종료 신호가 와도 멈춥니다.
대기와 실제 작업이 같은 signal을 공유해야 전체 흐름이 한 번에 취소됩니다.

## 테스트 기준 세우기

### H3. 실제 시간을 오래 기다리는 테스트는 피한다

대기 유틸을 테스트한다고 1초, 5초씩 실제로 기다리면 테스트 suite가 느려집니다.
대부분은 짧은 delay와 `AbortController`로 동작을 확인하고, retry나 polling 로직은 delay 함수를 주입해 테스트할 수 있게 만드는 편이 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('pollUntil retries until state is done', async () => {
  const calls = [];

  const value = await pollUntil(
    async ({ attempt }) => {
      calls.push(attempt);
      return attempt === 3
        ? { done: true, value: 'ready' }
        : { done: false };
    },
    {
      intervalMs: 1,
      maxAttempts: 5
    }
  );

  assert.equal(value, 'ready');
  assert.deepEqual(calls, [1, 2, 3]);
});
```

더 정교한 테스트가 필요하면 delay 구현을 옵션으로 주입합니다.
그러면 retry 횟수와 backoff 계산을 실제 timer 없이 검증할 수 있습니다.

```js
export async function retryWithDelay(operation, options = {}) {
  const wait = options.delay ?? delay;
  const waits = [];

  for (let attempt = 1; attempt <= options.attempts; attempt += 1) {
    try {
      return await operation({ attempt });
    } catch (error) {
      if (attempt === options.attempts) throw error;

      const waitMs = options.baseDelayMs * 2 ** (attempt - 1);
      waits.push(waitMs);
      await wait(waitMs, { signal: options.signal, ref: false });
    }
  }
}
```

테스트에서는 `delay: async () => {}`를 넘겨 즉시 진행하게 만들 수 있습니다.
시간 자체를 검증하는 테스트와 retry 정책을 검증하는 테스트를 분리하면 suite가 빠르고 안정적입니다.

### H3. 취소 테스트는 AbortError를 기대한다

취소 테스트는 "reject되는지"만 보면 부족합니다.
취소 에러가 다른 운영 오류와 섞이지 않도록 이름이나 code를 확인해야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { delay, isAbortError } from './delay.js';

test('delay rejects when signal is aborted', async () => {
  const controller = new AbortController();
  const waiting = delay(10_000, { signal: controller.signal });

  controller.abort();

  await assert.rejects(waiting, (error) => {
    assert.equal(isAbortError(error), true);
    return true;
  });
});
```

이 테스트는 실제로 10초를 기다리지 않습니다.
취소 신호가 즉시 전달되는지 확인하므로 빠르게 끝납니다.

## 운영에서 주의할 점

### H3. scheduler.wait는 아직 Experimental임을 표시한다

현재 Node.js 문서에서 `scheduler.wait()`는 Experimental로 표시됩니다.
프로젝트가 장기 지원 버전 호환성을 강하게 요구하거나 API 안정성에 보수적이라면 `timers/promises.setTimeout()`을 직접 쓰는 선택도 좋습니다.

```js
import { setTimeout as wait } from 'node:timers/promises';

await wait(500, undefined, { signal, ref: false });
```

둘 중 무엇을 쓰든 중요한 것은 호출부가 `AbortSignal`과 `ref` 정책을 명확히 다루는 것입니다.
API 이름보다 취소 가능한 흐름을 유지하는 설계가 더 큰 차이를 만듭니다.

### H3. 대기는 부하 제어를 대신하지 않는다

대기 시간을 늘린다고 과부하 문제가 자동으로 해결되지는 않습니다.
retry가 너무 많으면 하류 시스템에 더 큰 압박을 줄 수 있고, polling 간격이 너무 짧으면 상태 조회 API가 병목이 됩니다.
대기는 retry budget, concurrency limit, circuit breaker, queue length 제한과 함께 설계해야 합니다.

체크리스트는 간단합니다.

- 모든 대기 함수가 `signal`을 받는가?
- shutdown 중에도 대기 중인 retry와 polling이 빠져나오는가?
- `ref: false`가 필요한 CLI, worker, background task를 구분했는가?
- retry 횟수와 polling 최대 시간이 상한으로 묶여 있는가?
- 취소 로그와 실제 실패 로그가 분리되어 있는가?
- 테스트가 실제 시간을 오래 쓰지 않는가?

## 마무리

`scheduler.wait()`는 작은 API지만, 비동기 흐름의 품질을 드러내는 지점입니다.
대기 함수가 취소를 받지 못하면 shutdown이 느려지고, retry가 과해지고, 테스트가 오래 걸립니다.
반대로 `AbortSignal`, `ref`, 최대 시도 횟수, 에러 분류를 함께 정리하면 기다림도 운영 가능한 코드가 됩니다.

팀에서 이미 `setTimeout`을 감싼 sleep 함수가 여러 개 있다면 먼저 하나의 `delay()` 유틸로 모으세요.
그 다음 retry, polling, graceful shutdown에 같은 signal을 통과시키면 시간 의존 코드의 예측 가능성이 크게 좋아집니다.

## FAQ

### H3. scheduler.wait와 timers/promises setTimeout 중 무엇을 써야 하나요?

현재 문서 기준으로 `scheduler.wait()`는 Experimental API이고, 동작은 `timers/promises.setTimeout(delay, undefined, options)`와 같습니다.
안정성을 더 중시한다면 `setTimeout` Promise API를 쓰고, Scheduling APIs 흐름에 맞춘 표현을 선호한다면 `scheduler.wait()`를 검토할 수 있습니다.

### H3. ref: false는 항상 켜야 하나요?

아닙니다.
프로세스가 대기 timer 때문에 계속 살아 있지 않아도 되는 CLI, worker shutdown, background retry에는 유용합니다.
반대로 서버 요청 처리처럼 대기가 작업의 일부라면 기본값인 `ref: true`가 더 자연스럽습니다.

### H3. AbortError를 로그에 남기면 안 되나요?

남길 수는 있지만 장애 로그와 분리하는 편이 좋습니다.
사용자 취소, timeout, shutdown은 정상적인 제어 흐름일 수 있으므로 `cancelled`나 `aborted` 같은 outcome으로 분류하면 운영 알림의 잡음이 줄어듭니다.

## 관련 글

- [Node.js timers/promises scheduler.yield 가이드: 이벤트 루프 양보로 긴 작업 쪼개기](/development/blog/seo/2026/05/20/nodejs-timers-promises-scheduler-yield-event-loop-guide.html)
- [Node.js AbortSignal.any 가이드: timeout과 수동 취소를 함께 다루기](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js retry budget 가이드: 재시도로 과부하를 키우지 않는 법](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html)
- [Node.js 공식 문서: Timers Promises API](https://nodejs.org/api/timers.html#timers-promises-api)

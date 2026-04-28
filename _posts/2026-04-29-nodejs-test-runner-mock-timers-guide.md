---
layout: post
title: "Node.js test runner mock timers 가이드: setTimeout·Date 테스트를 안정적으로 제어하는 법"
date: 2026-04-29 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-mock-timers-guide
permalink: /development/blog/seo/2026/04/29/nodejs-test-runner-mock-timers-guide.html
alternates:
  ko: /development/blog/seo/2026/04/29/nodejs-test-runner-mock-timers-guide.html
  x_default: /development/blog/seo/2026/04/29/nodejs-test-runner-mock-timers-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, node-test, mock-timers, testing, settimeout, date, javascript, backend]
description: "Node.js test runner의 mock timers로 setTimeout, setInterval, Date 의존 코드를 빠르고 안정적으로 테스트하는 방법과 실무 주의사항을 예제와 함께 정리했습니다."
---

시간 의존 로직을 테스트할 때 가장 흔한 문제는 테스트가 느리고, 가끔씩 flaky해진다는 점입니다.
`setTimeout()`을 실제로 기다리게 두면 실행 시간이 늘어나고, `Date.now()`에 의존하는 코드는 실행 타이밍에 따라 결과가 흔들리기 쉽습니다.

이럴 때 유용한 기본 선택지가 **Node.js test runner의 mock timers**입니다.
결론부터 말하면 mock timers를 쓰면 **실제 시간을 기다리지 않고도 타이머와 현재 시각을 통제할 수 있어서**, 재시도·만료·debounce·polling 같은 로직을 더 빠르고 안정적으로 검증할 수 있습니다.

## Node.js test runner mock timers가 필요한 이유

### H3. 실제 시간을 기다리는 테스트는 느리고 불안정하다

예를 들어 아래 같은 코드는 기능 자체는 단순하지만 테스트 비용이 큽니다.

```js
export async function waitAndReturn(value, ms = 1000) {
  await new Promise((resolve) => setTimeout(resolve, ms));
  return value;
}
```

이 함수를 있는 그대로 테스트하면 문제가 바로 보입니다.

- 1초 대기를 실제로 기다려야 함
- 테스트 수가 늘수록 전체 실행 시간이 커짐
- CI 환경 부하에 따라 타이밍이 흔들릴 수 있음
- 경계값 테스트가 귀찮고 느려짐

즉 시간 기반 코드를 실시간으로 검증하는 방식은, 기능보다 대기 비용이 더 커지는 경우가 많습니다.

### H3. Date.now와 setTimeout은 함께 제어해야 의미 있는 검증이 된다

실무 코드에서는 타이머만 있는 경우보다 `Date.now()`나 만료 시각 계산이 함께 들어가는 경우가 많습니다.
예를 들어 토큰 만료, cache TTL, 재시도 backoff, polling 제한 시간은 “얼마나 기다렸는가”와 “지금 시각이 얼마인가”를 같이 봐야 합니다.

그래서 mock timers의 핵심 가치는 단순히 `setTimeout()`을 빨리 돌리는 것이 아니라,
**시간 흐름을 테스트 코드가 주도적으로 제어한다**는 데 있습니다.

## mock timers 기본 사용법

### H3. node:test에서 timers API를 활성화해 시간 제어를 시작한다

Node.js test runner에서는 테스트 문맥의 `mock.timers`를 이용해 타이머를 가짜 시간으로 바꿀 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function createDelayedFlag() {
  let ready = false;

  setTimeout(() => {
    ready = true;
  }, 1000);

  return () => ready;
}

test('1초 후 ready가 true가 된다', (t) => {
  t.mock.timers.enable({ apis: ['setTimeout'] });

  const isReady = createDelayedFlag();

  assert.equal(isReady(), false);

  t.mock.timers.tick(1000);

  assert.equal(isReady(), true);
});
```

이 패턴의 장점은 분명합니다.

- 테스트가 실제 1초를 기다리지 않음
- 실행 시간이 매우 짧아짐
- 경계값을 100ms, 500ms, 1000ms 식으로 쉽게 나눠 검증 가능

작지만 반복이 많은 시간 기반 유틸일수록 체감 차이가 큽니다.

### H3. tick으로 시간을 전진시키면 중간 상태도 검증하기 쉽다

mock timers를 쓰면 “완료 여부”만 아니라 중간 상태도 쉽게 볼 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function createRetryState() {
  let attempts = 0;

  setTimeout(() => {
    attempts += 1;
  }, 500);

  setTimeout(() => {
    attempts += 1;
  }, 1000);

  return () => attempts;
}

test('시간 흐름에 따라 attempts가 증가한다', (t) => {
  t.mock.timers.enable({ apis: ['setTimeout'] });

  const getAttempts = createRetryState();

  assert.equal(getAttempts(), 0);

  t.mock.timers.tick(500);
  assert.equal(getAttempts(), 1);

  t.mock.timers.tick(500);
  assert.equal(getAttempts(), 2);
});
```

이 방식은 [Node.js timers/promises setTimeout 가이드](/development/blog/seo/2026/04/28/nodejs-timers-promises-settimeout-delay-retry-guide.html)에서 다룬 retry·delay 로직을 테스트할 때 특히 잘 맞습니다.
운영 코드에서 대기 시간을 늘리지 않고도 재시도 흐름을 촘촘하게 확인할 수 있기 때문입니다.

## Date 의존 코드 테스트는 어떻게 할까

### H3. 현재 시각을 고정하면 만료 로직 테스트가 훨씬 단순해진다

시간 비교 로직은 현재 시각을 고정하는 것만으로도 테스트가 훨씬 읽기 쉬워집니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function isExpired(expiresAt) {
  return Date.now() >= expiresAt;
}

test('만료 시각 이전에는 false를 반환한다', (t) => {
  const now = new Date('2026-04-29T08:00:00+09:00');

  t.mock.timers.enable({
    apis: ['Date'],
    now,
  });

  const expiresAt = now.getTime() + 60_000;

  assert.equal(isExpired(expiresAt), false);
});
```

이 접근의 장점은 “테스트를 실행한 진짜 현재 시각”과 분리된다는 점입니다.
그래서 아침에 돌리든 밤에 돌리든 결과가 바뀌지 않습니다.

### H3. Date와 setTimeout을 함께 mock하면 TTL 검증이 자연스러워진다

실무에서는 보통 `Date`와 타이머를 같이 다루게 됩니다.
예를 들어 캐시 TTL이나 세션 만료는 아래처럼 구성되는 경우가 많습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function createCache(ttlMs) {
  const createdAt = Date.now();

  return {
    isExpired() {
      return Date.now() - createdAt >= ttlMs;
    },
  };
}

test('TTL이 지나면 cache가 만료된다', (t) => {
  const now = new Date('2026-04-29T08:00:00+09:00');

  t.mock.timers.enable({
    apis: ['Date', 'setTimeout'],
    now,
  });

  const cache = createCache(5000);

  assert.equal(cache.isExpired(), false);

  t.mock.timers.tick(5000);

  assert.equal(cache.isExpired(), true);
});
```

이 패턴은 단순하지만 매우 실용적입니다.
“5초를 기다린 뒤 만료되는가?”를 실제 5초 대기 없이 검증할 수 있기 때문입니다.

## mock timers가 특히 잘 맞는 실무 사례

### H3. debounce, throttle, polling, retry 테스트에 효과가 크다

시간 제어가 필요한 로직은 생각보다 많습니다.
대표적으로 아래 같은 코드에서 효과가 좋습니다.

- debounce 입력 처리
- polling 주기 제어
- retry backoff 스케줄링
- 세션 만료/토큰 갱신
- 일정 시간 후 상태 전이되는 background job

특히 retry와 polling은 시간이 길수록 테스트가 느려지는데,
mock timers를 쓰면 초·분 단위 흐름도 거의 즉시 검증할 수 있습니다.
이런 설계는 [Node.js Promise.any fallback 가이드](/development/blog/seo/2026/04/24/nodejs-promise-any-fallback-first-success-guide.html)처럼 비동기 분기 로직을 다룰 때도 함께 도움이 됩니다.

### H3. 이벤트 루프 순서 자체를 검증할 때는 다른 큐와 구분해야 한다

주의할 점도 있습니다.
mock timers는 `setTimeout`, `setInterval`, `Date` 같은 시간 API 제어에는 강하지만,
모든 비동기 순서를 한 번에 설명해 주는 도구는 아닙니다.

예를 들어 microtask와 next tick 순서는 별도 개념입니다.
`queueMicrotask()`, `Promise.then()`, `process.nextTick()`의 우선순위를 이해하지 못한 채 타이머만 mock하면 테스트 의도를 잘못 해석할 수 있습니다.
이 차이는 [Node.js queueMicrotask vs process.nextTick 차이 가이드](/development/blog/seo/2026/04/27/nodejs-queuemicrotask-process-nexttick-difference-guide.html)와 함께 보면 더 명확합니다.

## mock timers를 쓸 때 흔한 실수

### H3. 시간 전진만 하고 정리나 범위를 신경 쓰지 않는 경우

테스트에서 시간을 빠르게 움직일 수 있다고 해서 아무 데나 켜두면 오히려 헷갈릴 수 있습니다.
특히 아래 실수는 자주 보입니다.

- 테스트 전체가 아니라 필요한 케이스에만 enable하지 않음
- `Date`는 실제 시간인데 타이머만 mock해 계산이 어긋남
- 타이머가 여러 개 쌓이는 구조를 중간 검증 없이 한 번에만 tick함
- 테스트 의도보다 구현 세부 타이밍에 너무 강하게 결합함

즉 mock timers는 편한 도구이지만, **테스트를 더 결정적으로 만들기 위한 범위 제어**가 같이 필요합니다.

### H3. 실시간 동작 자체를 검증해야 하는 테스트까지 모두 대체하면 안 된다

모든 시간 테스트를 가짜 시간으로만 대체하는 것도 좋은 기본값은 아닙니다.
예를 들어 실제 지연, 실제 스케줄러 상호작용, 실제 I/O와 섞인 타이밍은 통합 테스트나 e2e 관점에서 한 번쯤 별도로 볼 필요가 있습니다.

권장 방식은 보통 아래와 같습니다.

1. 단위 테스트에서는 mock timers로 빠르고 결정적인 검증을 한다.
2. 통합 테스트에서는 실제 런타임 상호작용을 최소한으로 확인한다.
3. 운영 코드에서는 timeout, retry, polling 설정값을 너무 테스트 친화적으로만 설계하지 않는다.

이렇게 나누면 속도와 현실성을 둘 다 챙기기 좋습니다.

## 실무 체크리스트

### H3. Node.js test runner mock timers를 도입할 때 확인할 것

도입 전에 아래 항목을 점검하면 시행착오를 줄이기 좋습니다.

- 테스트 대상이 `setTimeout`, `setInterval`, `Date` 중 무엇에 의존하는가
- `Date`와 타이머를 함께 mock해야 의미가 맞는가
- 중간 상태 검증이 필요한가, 최종 상태만 보면 되는가
- microtask와 timer를 혼동하고 있지 않은가
- 단위 테스트와 통합 테스트 경계를 나눴는가

이 다섯 가지만 정리해도 flaky한 시간 테스트를 꽤 줄일 수 있습니다.

## 마무리

Node.js test runner의 mock timers는 단순히 테스트를 빠르게 만드는 기능이 아닙니다.
시간을 직접 통제해 **재현 가능하고 설명 가능한 테스트**를 만드는 도구에 가깝습니다.

특히 `setTimeout`, `setInterval`, `Date.now()`에 기대는 코드가 늘어날수록,
실제 시간을 기다리는 테스트보다 mock timers 중심 설계가 훨씬 유지보수에 유리합니다.
시간 기반 로직이 많다면, 먼저 기다리는 테스트부터 걷어내는 것이 가장 빠른 품질 개선일 수 있습니다.

## 관련 글

- [Node.js timers/promises setTimeout 가이드: sleep, backoff, 재시도를 깔끔하게 구현하는 법](/development/blog/seo/2026/04/28/nodejs-timers-promises-settimeout-delay-retry-guide.html)
- [Node.js queueMicrotask vs process.nextTick 차이 가이드: 언제 무엇을 써야 할까](/development/blog/seo/2026/04/27/nodejs-queuemicrotask-process-nexttick-difference-guide.html)
- [Node.js Promise.any 가이드: 첫 성공 결과로 fallback 체인을 단순하게 만드는 법](/development/blog/seo/2026/04/24/nodejs-promise-any-fallback-first-success-guide.html)

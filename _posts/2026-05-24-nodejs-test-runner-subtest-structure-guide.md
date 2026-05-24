---
layout: post
title: "Node.js test runner subtest 가이드: t.test로 테스트 구조를 읽기 쉽게 나누는 법"
date: 2026-05-24 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-subtest-structure-guide
permalink: /development/blog/seo/2026/05/24/nodejs-test-runner-subtest-structure-guide.html
alternates:
  ko: /development/blog/seo/2026/05/24/nodejs-test-runner-subtest-structure-guide.html
  x_default: /development/blog/seo/2026/05/24/nodejs-test-runner-subtest-structure-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, subtest, testing, javascript, ci]
description: "Node.js test runner의 t.test subtest로 테스트를 기능별·조건별로 구조화하는 방법을 정리했습니다. await 누락, 훅 범위, 병렬 실행, CI 리포트 가독성까지 실무 기준으로 설명합니다."
---

테스트 파일이 커지면 실패 원인보다 파일 구조를 먼저 해석해야 하는 순간이 옵니다.
같은 함수의 정상 케이스, 입력 검증, 예외 처리, 경계값 테스트가 한 줄로 이어져 있으면 CI 로그도 길어지고 수정 범위도 흐려집니다.
Node.js 내장 test runner의 `t.test` subtest를 쓰면 하나의 큰 테스트를 기능별·조건별 작은 단위로 나눌 수 있습니다.

기본 실행 방법은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)에서 다뤘고, 준비와 정리는 [Node.js test runner hooks 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html)에서 정리했습니다.
이 글에서는 `t.test`로 테스트 구조를 읽기 좋게 만들고, `await` 누락이나 훅 범위 같은 실무 실수를 줄이는 기준을 정리합니다.

## t.test subtest가 필요한 이유

### 실패 로그가 테스트 설계를 보여준다

테스트 이름은 문서입니다.
하지만 테스트가 너무 넓으면 실패 로그가 “어떤 조건에서 깨졌는지”를 충분히 설명하지 못합니다.
`test()` 하나 안에 여러 `assert`가 길게 들어가면 첫 번째 실패 이후의 조건은 확인하기 어렵고, CI 리포트에서도 실패 지점이 뭉뚱그려 보입니다.

`subtest`는 큰 주제를 유지하면서 세부 조건을 별도 테스트로 보여줍니다.
예를 들어 사용자 입력 검증을 테스트한다면 `빈 문자열`, `공백 문자열`, `너무 긴 문자열`을 각각 하위 테스트로 나눌 수 있습니다.
그러면 실패 로그가 곧 수정해야 할 조건 목록이 됩니다.

### 관련 테스트를 가까이 두되 책임은 나눈다

파일을 지나치게 많이 쪼개면 관련 맥락을 찾기 어렵습니다.
반대로 하나의 테스트에 모든 조건을 넣으면 실패 분석이 어려워집니다.
`t.test`는 이 중간 지점을 제공합니다.

관련 테스트는 같은 상위 테스트 아래에 두고, 실제 검증은 작은 하위 테스트로 분리합니다.
이 방식은 특히 파서, 검증 함수, 가격 계산, 권한 판정처럼 입력 조건이 많은 코드에서 유용합니다.

## 기본 사용법

### test 콜백에서 t를 받는다

Node.js test runner에서 `test()` 콜백은 테스트 컨텍스트 객체를 받을 수 있습니다.
보통 이 객체를 `t`라고 이름 붙이고, `t.test()`로 하위 테스트를 만듭니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function normalizeRole(role) {
  const value = role.trim().toLowerCase();

  if (!value) {
    throw new TypeError('role is required');
  }

  if (!['admin', 'editor', 'viewer'].includes(value)) {
    throw new RangeError('unsupported role');
  }

  return value;
}

test('normalizeRole', async (t) => {
  await t.test('trims and lowercases a valid role', () => {
    assert.equal(normalizeRole(' Admin '), 'admin');
  });

  await t.test('rejects an empty role', () => {
    assert.throws(() => normalizeRole('   '), TypeError);
  });

  await t.test('rejects an unsupported role', () => {
    assert.throws(() => normalizeRole('owner'), RangeError);
  });
});
```

상위 이름은 테스트 대상, 하위 이름은 조건이나 기대 동작을 설명하게 두면 읽기 쉽습니다.
`normalizeRole > rejects an unsupported role`처럼 계층이 자연스럽게 만들어지기 때문에 CI 로그에서도 원인을 좁히기 좋습니다.

### 하위 테스트도 await 한다

`t.test()`는 테스트 결과를 나타내는 Promise를 반환합니다.
상위 테스트가 비동기 함수라면 하위 테스트를 `await`하는 습관을 들이는 편이 안전합니다.
특히 하위 테스트 안에서 비동기 검증을 하거나 순서가 중요한 공유 자원을 다룬다면 `await` 누락은 흔한 불안정성의 원인이 됩니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

async function readProfile(id) {
  if (!id) {
    throw new TypeError('id is required');
  }

  return { id, name: 'Soo' };
}

test('readProfile', async (t) => {
  await t.test('returns a profile when id exists', async () => {
    const profile = await readProfile('user-1');
    assert.deepEqual(profile, { id: 'user-1', name: 'Soo' });
  });

  await t.test('rejects when id is empty', async () => {
    await assert.rejects(() => readProfile(''), TypeError);
  });
});
```

비동기 실패 검증은 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)에서 다룬 것처럼 `await assert.rejects(...)` 형태로 끝까지 기다려야 합니다.
상위 `await t.test(...)`와 내부 `await assert.rejects(...)`를 함께 지키면 테스트 완료 시점을 명확하게 만들 수 있습니다.

## 테스트 구조 설계 기준

### 상위 테스트는 기능 단위로 잡는다

상위 테스트는 “무엇을 테스트하는가”를 나타내는 이름이 좋습니다.
함수명, 모듈명, API 엔드포인트, 유스케이스처럼 개발자가 검색하기 쉬운 단위를 쓰면 됩니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function calculateShipping(order) {
  if (order.total >= 50000) return 0;
  if (order.region === 'remote') return 6000;
  return 3000;
}

test('calculateShipping', async (t) => {
  await t.test('returns free shipping for high value orders', () => {
    assert.equal(calculateShipping({ total: 70000, region: 'city' }), 0);
  });

  await t.test('adds remote area shipping fee', () => {
    assert.equal(calculateShipping({ total: 20000, region: 'remote' }), 6000);
  });

  await t.test('uses the default shipping fee', () => {
    assert.equal(calculateShipping({ total: 20000, region: 'city' }), 3000);
  });
});
```

이 구조는 나중에 조건이 늘어나도 안정적입니다.
무료 배송 정책이 바뀌면 상위 테스트 전체를 읽되, 실제 수정할 하위 조건을 빠르게 찾을 수 있습니다.

### 하위 테스트는 하나의 이유로 실패하게 만든다

좋은 하위 테스트는 실패 이유가 하나입니다.
한 하위 테스트에서 정상화, 저장, 이벤트 발행, 로그 기록을 모두 검증하면 실패했을 때 어느 책임이 깨졌는지 다시 분석해야 합니다.

아래처럼 결과와 부수효과를 나눠 두면 실패 지점이 선명해집니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function createUser(input, events) {
  const user = { id: 'user-1', email: input.email.toLowerCase() };
  events.push({ type: 'user.created', userId: user.id });
  return user;
}

test('createUser', async (t) => {
  await t.test('normalizes the email address', () => {
    const events = [];
    const user = createUser({ email: 'ADMIN@example.com' }, events);

    assert.equal(user.email, 'admin@example.com');
  });

  await t.test('emits a user.created event', () => {
    const events = [];
    const user = createUser({ email: 'admin@example.com' }, events);

    assert.deepEqual(events, [{ type: 'user.created', userId: user.id }]);
  });
});
```

의존성 호출 횟수와 인자를 검증해야 한다면 [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html)처럼 mock을 별도 하위 테스트로 분리하는 편이 좋습니다.

## 훅과 subtest 함께 쓰기

### beforeEach는 같은 레벨의 테스트에 적용된다

`beforeEach`와 `afterEach`는 선언된 레벨의 테스트에 적용됩니다.
상위 테스트 안에서 `t.beforeEach()`를 쓰면 그 상위 테스트 아래의 하위 테스트마다 준비 코드가 실행됩니다.
테스트 데이터가 하위 테스트마다 새로 필요할 때 유용합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

test('cart', async (t) => {
  let cart;

  t.beforeEach(() => {
    cart = [];
  });

  await t.test('starts empty', () => {
    assert.deepEqual(cart, []);
  });

  await t.test('adds an item', () => {
    cart.push({ sku: 'book', quantity: 1 });
    assert.equal(cart.length, 1);
  });
});
```

공유 상태를 직접 재사용하지 않고 하위 테스트마다 새로 만들면 순서 의존성이 줄어듭니다.
훅에서 네트워크, 데이터베이스, 임시 파일처럼 정리가 필요한 자원을 만들었다면 `afterEach`에서 닫거나 삭제해야 합니다.

### 상위 테스트마다 준비 범위를 좁힌다

모든 테스트 파일 최상단에 큰 `beforeEach`를 두면 사용하지 않는 준비 코드가 반복될 수 있습니다.
준비 비용이 큰 경우에는 관련 상위 테스트 안으로 훅을 이동시키는 편이 낫습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function makeStore() {
  return new Map();
}

test('session store', async (t) => {
  let store;

  t.beforeEach(() => {
    store = makeStore();
  });

  await t.test('saves a session', () => {
    store.set('sid-1', { userId: 'user-1' });
    assert.deepEqual(store.get('sid-1'), { userId: 'user-1' });
  });

  await t.test('deletes a session', () => {
    store.set('sid-1', { userId: 'user-1' });
    store.delete('sid-1');
    assert.equal(store.has('sid-1'), false);
  });
});
```

이렇게 하면 테스트 파일 안에 여러 모듈 테스트가 있어도 각 그룹의 준비 코드가 서로 섞이지 않습니다.

## 병렬 실행을 고려한 subtest 작성

### 공유 상태가 있으면 순서를 가정하지 않는다

Node.js test runner는 옵션에 따라 테스트를 병렬로 실행할 수 있습니다.
하위 테스트를 작성할 때 전역 배열, 전역 Map, 실제 파일 경로를 공유하면 실행 순서나 병렬성에 따라 실패가 흔들릴 수 있습니다.

가장 안전한 기준은 하위 테스트마다 데이터를 새로 만드는 것입니다.
상태가 필요하면 팩토리 함수를 두고, 파일이 필요하면 테스트별 임시 경로를 만듭니다.
임시 디렉터리 자동 정리는 [Node.js mkdtempDisposable 가이드](/development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html)를 참고할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function createCounter() {
  let value = 0;

  return {
    increment() {
      value += 1;
      return value;
    },
    value() {
      return value;
    },
  };
}

test('counter', async (t) => {
  await t.test('increments from zero', () => {
    const counter = createCounter();
    assert.equal(counter.increment(), 1);
  });

  await t.test('keeps its own state', () => {
    const counter = createCounter();
    counter.increment();
    counter.increment();
    assert.equal(counter.value(), 2);
  });
});
```

각 하위 테스트가 독립적인 인스턴스를 만들기 때문에 실행 순서가 바뀌어도 결과가 같습니다.

### 동시성은 성능보다 안정성을 먼저 본다

테스트 시간이 길어지면 병렬 실행을 고려하게 됩니다.
하지만 병렬화는 공유 자원 문제가 없을 때만 이득입니다.
파일 시스템, 포트, 환경 변수, 현재 작업 디렉터리, 날짜·시간 mock처럼 전역 영향이 있는 요소는 먼저 격리해야 합니다.

긴 작업 때문에 이벤트 루프 양보가 필요하다면 [Node.js scheduler.yield 가이드](/development/blog/seo/2026/05/20/nodejs-timers-promises-scheduler-yield-event-loop-guide.html)의 기준처럼 긴 루프를 작게 끊는 것도 도움이 됩니다.
테스트 병렬화는 느린 테스트를 숨기는 장치가 아니라, 독립적인 테스트를 더 빠르게 실행하는 장치로 보는 편이 안전합니다.

## CI 리포트에서 읽기 좋은 이름 짓기

### 조건과 기대 결과를 함께 쓴다

하위 테스트 이름은 짧아도 됩니다.
다만 “works”, “success”, “case 1”처럼 의미가 약한 이름은 CI에서 도움이 되지 않습니다.
조건과 기대 결과를 함께 적으면 실패 메시지가 바로 액션 아이템이 됩니다.

좋은 예시는 다음과 같습니다.

- `returns free shipping for high value orders`
- `rejects an empty role`
- `emits a user.created event`
- `keeps its own state`

이름이 너무 길어지면 테스트가 여러 책임을 갖고 있다는 신호일 수 있습니다.
그럴 때는 하위 테스트를 더 나누는 편이 좋습니다.

### reporter 출력도 함께 고려한다

CI에서 테스트 결과를 JUnit이나 spec reporter로 남긴다면 subtest 계층은 리포트 가독성에 직접 영향을 줍니다.
상위 테스트가 기능명이고 하위 테스트가 조건명이라면 실패 항목만 봐도 어떤 코드 영역을 봐야 하는지 알 수 있습니다.

리포트 설정은 [Node.js test runner reporter 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-reporter-ci-junit-guide.html)를 참고하면 됩니다.
테스트 구조와 reporter 설정을 함께 맞추면 로컬 실행, PR 체크, 배포 전 검증 로그가 같은 언어로 읽힙니다.

## 흔한 실수와 점검 기준

### await 없는 t.test

가장 흔한 실수는 상위 테스트 안에서 `t.test()`를 호출하고 기다리지 않는 것입니다.
단순한 동기 테스트에서는 당장 문제가 보이지 않을 수 있지만, 비동기 하위 테스트가 섞이면 상위 테스트 완료 시점이 불명확해집니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

async function loadValue() {
  return 42;
}

test('loadValue', async (t) => {
  await t.test('returns the stored value', async () => {
    assert.equal(await loadValue(), 42);
  });
});
```

팀 규칙으로 “상위 테스트 안의 `t.test`는 기본적으로 `await`한다”를 두면 리뷰에서 실수를 잡기 쉽습니다.

### 너무 깊은 계층

Subtest는 구조화 도구지만, 계층이 너무 깊어지면 오히려 읽기 어려워집니다.
대부분의 실무 테스트는 상위 테스트 하나와 하위 테스트 여러 개면 충분합니다.
세 단계 이상으로 내려가야 한다면 파일을 나누거나 상위 테스트 이름을 다시 잡는 것이 낫습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function parseFlag(value) {
  return value === 'true';
}

test('parseFlag', async (t) => {
  await t.test('returns true for the true string', () => {
    assert.equal(parseFlag('true'), true);
  });

  await t.test('returns false for other values', () => {
    assert.equal(parseFlag('false'), false);
    assert.equal(parseFlag(''), false);
  });
});
```

위처럼 단순한 함수는 깊은 describe 스타일 계층보다 납작한 하위 테스트가 더 잘 읽힙니다.

### 한 하위 테스트에 assert를 너무 많이 넣기

하위 테스트 하나에 `assert`가 여러 개 들어갈 수는 있습니다.
하지만 각 `assert`가 서로 다른 책임을 검증한다면 분리하는 편이 좋습니다.
반대로 같은 결과 객체의 여러 필드를 함께 확인하는 정도라면 `deepEqual` 하나로 묶어도 괜찮습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function formatUser(user) {
  return {
    label: `${user.name} <${user.email}>`,
    searchable: user.email.toLowerCase(),
  };
}

test('formatUser', async (t) => {
  await t.test('returns display and search fields', () => {
    assert.deepEqual(formatUser({ name: 'Soo', email: 'SOO@example.com' }), {
      label: 'Soo <SOO@example.com>',
      searchable: 'soo@example.com',
    });
  });
});
```

필드들이 하나의 출력 계약을 이룬다면 묶고, 서로 다른 부수효과라면 나누는 기준을 추천합니다.

## 실무 체크리스트

### 새 테스트 파일을 만들 때

새 테스트 파일을 만들 때는 아래 순서로 구조를 잡으면 됩니다.

1. 상위 테스트 이름을 함수명이나 유스케이스명으로 정한다.
2. 정상 케이스, 입력 검증, 예외, 경계값을 하위 테스트로 나눈다.
3. 하위 테스트마다 독립적인 데이터를 만든다.
4. 비동기 하위 테스트는 `await t.test(...)`와 내부 `await`을 함께 확인한다.
5. CI reporter에서 읽히는 이름인지 마지막에 한 번 더 본다.

이 기준은 작지만 오래 갑니다.
처음부터 완벽한 테스트 아키텍처를 만들기보다, 실패 로그가 곧 설계 문서처럼 읽히게 만드는 것이 중요합니다.

### 기존 테스트를 정리할 때

기존 테스트가 길어졌다면 모든 것을 한 번에 갈아엎기보다 실패가 잦거나 리뷰가 어려운 파일부터 나누는 편이 안전합니다.
긴 테스트를 상위 `test()`로 감싸고, 내부의 조건 묶음을 `await t.test()`로 한 단계씩 빼내면 됩니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function canAccess(user, document) {
  return user.role === 'admin' || document.ownerId === user.id;
}

test('canAccess', async (t) => {
  await t.test('allows admins', () => {
    assert.equal(canAccess({ id: 'u1', role: 'admin' }, { ownerId: 'u2' }), true);
  });

  await t.test('allows document owners', () => {
    assert.equal(canAccess({ id: 'u1', role: 'viewer' }, { ownerId: 'u1' }), true);
  });

  await t.test('denies unrelated viewers', () => {
    assert.equal(canAccess({ id: 'u1', role: 'viewer' }, { ownerId: 'u2' }), false);
  });
});
```

권한, 결제, 알림처럼 조건 조합이 많은 코드는 이 방식으로 읽기 쉬워집니다.
하위 테스트 이름만 훑어도 정책 표처럼 이해할 수 있기 때문입니다.

## FAQ

### t.test와 test를 여러 개 쓰는 방식은 무엇이 다른가요?

둘 다 사용할 수 있습니다.
서로 독립적인 기능이라면 최상위 `test()`를 여러 개 두면 됩니다.
반대로 하나의 기능 아래 여러 조건을 묶고 싶다면 `t.test()`가 더 읽기 좋습니다.
검색 가능한 기능명은 상위 테스트에, 구체적인 조건은 하위 테스트에 두는 방식을 추천합니다.

### 모든 t.test에 await이 꼭 필요한가요?

항상 문법적으로 필수인 것은 아니지만, 실무 규칙으로는 `await`을 붙이는 편이 안전합니다.
비동기 검증이 나중에 추가되어도 구조가 흔들리지 않고, 상위 테스트의 완료 시점도 분명해집니다.
특히 `assert.rejects`, 파일 작업, 타이머, mock 검증이 들어간다면 반드시 기다리는 습관이 좋습니다.

### subtest를 쓰면 테스트가 느려지나요?

일반적으로 구조화 자체가 병목이 되지는 않습니다.
테스트 시간을 좌우하는 것은 네트워크, 파일 시스템, 데이터베이스, 큰 fixture, 불필요한 전역 준비 코드인 경우가 많습니다.
오히려 subtest로 범위를 나누면 느린 조건을 더 쉽게 찾을 수 있습니다.

### describe/it 스타일을 쓰는 라이브러리와 함께 써도 되나요?

프로젝트 표준을 하나로 정하는 것이 가장 좋습니다.
Node.js 내장 test runner만 쓰기로 했다면 `test`와 `t.test` 중심으로 통일하면 의존성이 줄고 CI 설정도 단순해집니다.
이미 다른 테스트 프레임워크가 깊게 들어간 프로젝트라면 새 코드부터 점진적으로 맞추는 전략이 안전합니다.

## 마무리

`t.test` subtest는 테스트를 더 많이 쓰게 만드는 기능이 아니라, 이미 필요한 테스트를 읽기 좋은 구조로 정리하는 기능입니다.
상위 테스트는 기능을 설명하고, 하위 테스트는 조건과 기대 결과를 설명하게 두면 실패 로그가 훨씬 실용적으로 바뀝니다.

처음 적용할 때는 규칙을 단순하게 잡으면 됩니다.
상위 테스트 안의 `t.test`는 `await`하고, 하위 테스트는 하나의 이유로 실패하게 만들고, 공유 상태는 매번 새로 만듭니다.
이 세 가지만 지켜도 Node.js test runner 기반 테스트는 CI와 코드 리뷰에서 훨씬 안정적으로 읽힙니다.

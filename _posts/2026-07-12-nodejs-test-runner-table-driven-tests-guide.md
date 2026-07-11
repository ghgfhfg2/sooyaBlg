---
layout: post
title: "Node.js test runner table-driven tests 가이드: 반복 케이스를 읽기 좋게 정리하는 법"
date: 2026-07-12 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-table-driven-tests-guide
permalink: /development/blog/seo/2026/07/12/nodejs-test-runner-table-driven-tests-guide.html
alternates:
  ko: /development/blog/seo/2026/07/12/nodejs-test-runner-table-driven-tests-guide.html
  x_default: /development/blog/seo/2026/07/12/nodejs-test-runner-table-driven-tests-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, table-driven-test, subtest, testing, javascript, unit-test, ci]
description: "Node.js test runner에서 table-driven tests로 반복 케이스를 구조화하는 방법을 정리합니다. t.test 하위 테스트, 케이스 이름, 실패 로그, 경계값 검증, CI 가독성 기준을 예제로 설명합니다."
---

비슷한 입력과 기대 결과를 여러 번 검증하다 보면 테스트 파일이 빠르게 길어집니다.
처음에는 `test()`를 여러 개 복사해도 괜찮지만, 케이스가 늘어나면 어떤 조건만 다른지 파악하기 어려워지고 수정할 때 누락이 생기기 쉽습니다.
이럴 때 table-driven tests, 즉 케이스 배열을 순회하는 테스트 구조가 도움이 됩니다.

Node.js 내장 test runner에는 Jest의 `test.each()` 같은 전용 API가 있는 것은 아닙니다.
대신 배열과 `t.test()`를 조합하면 충분히 읽기 좋은 table-driven test를 만들 수 있습니다.
핵심은 반복을 줄이되 실패 로그에서 어떤 케이스가 깨졌는지 바로 보이게 만드는 것입니다.

이 글에서는 Node.js test runner에서 table-driven tests를 구성하는 기준, 케이스 이름을 짓는 법, 비동기 검증과 경계값 테스트를 다루는 패턴을 정리합니다.
하위 테스트 구조는 [Node.js test runner test context 가이드](/development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html)를 함께 보면 좋습니다.
준비와 정리 흐름은 [Node.js test runner beforeEach/afterEach 가이드](/development/blog/seo/2026/07/10/nodejs-test-runner-beforeeach-aftereach-hooks-guide.html), 반복 fixture 관리는 [Node.js test runner fixture helper 가이드](/development/blog/seo/2026/07/11/nodejs-test-runner-fixture-helper-temp-resource-guide.html)와 이어집니다.

## table-driven tests가 필요한 순간

### H3. 같은 함수에 입력만 다른 테스트가 반복될 때

가장 흔한 신호는 거의 같은 테스트가 여러 개 생기는 상황입니다.
아래처럼 입력과 기대 결과만 다르다면 케이스 배열로 옮길 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function normalizeRole(value) {
  return value.trim().toLowerCase().replaceAll(' ', '-');
}

test('normalizes an admin role', () => {
  assert.equal(normalizeRole(' Admin '), 'admin');
});

test('normalizes a support manager role', () => {
  assert.equal(normalizeRole('Support Manager'), 'support-manager');
});

test('normalizes a billing owner role', () => {
  assert.equal(normalizeRole(' billing owner '), 'billing-owner');
});
```

이 테스트들은 모두 같은 규칙을 확인합니다.
역할 이름을 공백 제거, 소문자 변환, 하이픈 치환으로 정규화하는지 보는 것입니다.
복사한 테스트가 많아질수록 새 케이스를 추가하거나 규칙 설명을 고치기 번거로워집니다.

### H3. 케이스 목록 자체가 명세가 될 때

table-driven test의 장점은 케이스 배열이 작은 명세처럼 읽힌다는 점입니다.
테스트 본문을 보기 전에 어떤 입력을 어떤 결과로 기대하는지 한눈에 확인할 수 있습니다.

```js
const roleCases = [
  { name: 'trims surrounding whitespace', input: ' Admin ', expected: 'admin' },
  { name: 'replaces spaces with hyphens', input: 'Support Manager', expected: 'support-manager' },
  { name: 'normalizes mixed whitespace and case', input: ' billing owner ', expected: 'billing-owner' }
];
```

좋은 케이스 배열은 테스트의 의도를 숨기지 않습니다.
`input`, `expected`처럼 단순한 필드를 사용하고, 케이스 이름에는 검증하려는 차이를 짧게 적습니다.
필드가 너무 많아지면 table이 아니라 작은 fixture 시스템이 되기 쉬우므로 주의해야 합니다.

## t.test로 케이스별 하위 테스트 만들기

### H3. 케이스 이름을 하위 테스트 이름으로 사용한다

Node.js test runner에서는 부모 테스트 안에서 `t.test()`를 반복 호출하는 방식이 자연스럽습니다.
각 케이스가 하위 테스트가 되므로 실패 로그에서도 깨진 케이스 이름이 따로 보입니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function normalizeRole(value) {
  return value.trim().toLowerCase().replaceAll(' ', '-');
}

test('normalizeRole', async (t) => {
  const cases = [
    { name: 'trims surrounding whitespace', input: ' Admin ', expected: 'admin' },
    { name: 'replaces spaces with hyphens', input: 'Support Manager', expected: 'support-manager' },
    { name: 'normalizes mixed whitespace and case', input: ' billing owner ', expected: 'billing-owner' }
  ];

  for (const testCase of cases) {
    await t.test(testCase.name, () => {
      assert.equal(normalizeRole(testCase.input), testCase.expected);
    });
  }
});
```

여기서 중요한 부분은 `await t.test(...)`입니다.
하위 테스트가 비동기 작업을 포함하지 않더라도 부모 테스트가 자식 테스트의 완료를 기다리게 만드는 습관을 들이는 편이 좋습니다.
나중에 케이스 안에 비동기 검증이 들어가도 구조를 크게 바꾸지 않아도 됩니다.

### H3. 실패 메시지는 입력과 기대값을 보강한다

하위 테스트 이름만으로도 대부분 충분하지만, 값이 많은 테스트에서는 assertion 메시지를 더해도 좋습니다.
단, 민감정보나 실제 사용자 데이터가 들어가지 않도록 주의해야 합니다.

```js
test('calculateDiscountRate', async (t) => {
  const cases = [
    { name: 'guest users receive no discount', role: 'guest', total: 30_000, expected: 0 },
    { name: 'members receive base discount', role: 'member', total: 30_000, expected: 0.05 },
    { name: 'vip users receive higher discount', role: 'vip', total: 30_000, expected: 0.15 }
  ];

  for (const testCase of cases) {
    await t.test(testCase.name, () => {
      const actual = calculateDiscountRate({
        role: testCase.role,
        total: testCase.total
      });

      assert.equal(
        actual,
        testCase.expected,
        `role=${testCase.role}, total=${testCase.total}`
      );
    });
  }
});

function calculateDiscountRate({ role }) {
  if (role === 'vip') return 0.15;
  if (role === 'member') return 0.05;
  return 0;
}
```

메시지는 실패 원인을 좁히는 정보만 담는 편이 좋습니다.
토큰, 실제 이메일, 세션 ID, 고객 이름 같은 값은 테스트 데이터라도 로그에 남기지 않는 기준을 유지하세요.
CI 로그는 생각보다 오래 보관되고 여러 도구로 전달될 수 있습니다.

## 케이스 배열을 설계하는 기준

### H3. 한 table에는 한 규칙만 담는다

table-driven test가 커지는 가장 흔한 이유는 하나의 배열에 여러 규칙을 몰아넣기 때문입니다.
입력 검증, 권한 검증, 포맷 변환, 오류 메시지 검증을 모두 한 table에 넣으면 필드가 늘어나고 테스트 이름도 흐려집니다.

```js
const cases = [
  {
    name: 'valid member checkout',
    userRole: 'member',
    orderTotal: 40_000,
    couponCode: 'WELCOME',
    expectedStatus: 'accepted',
    expectedDiscount: 0.1,
    expectedError: null
  }
];
```

이런 형태는 처음에는 편해 보이지만 곧 읽기 어려워집니다.
검증 대상이 너무 많기 때문입니다.
할인 계산 table과 주문 유효성 table을 분리하면 각 테스트가 더 작고 분명해집니다.

```js
const discountCases = [
  { name: 'welcome coupon applies ten percent', couponCode: 'WELCOME', expected: 0.1 },
  { name: 'missing coupon applies no discount', couponCode: null, expected: 0 }
];

const validationCases = [
  { name: 'member order is accepted', userRole: 'member', expectedStatus: 'accepted' },
  { name: 'blocked user order is rejected', userRole: 'blocked', expectedStatus: 'rejected' }
];
```

케이스 배열은 반복을 줄이는 도구이지만, 테스트 목적을 섞는 도구는 아닙니다.
한 table을 설명하는 문장을 만들기 어렵다면 둘 이상으로 나눌 때입니다.

### H3. 케이스 이름은 입력보다 의도를 말한다

`case 1`, `case 2` 같은 이름은 실패했을 때 거의 도움이 되지 않습니다.
반대로 모든 입력 값을 이름에 다 넣으면 리포트가 길어지고 핵심이 흐려집니다.

좋은 이름은 입력의 의미와 기대하는 규칙을 짧게 말합니다.

```js
const cases = [
  { name: 'returns free shipping at threshold', total: 50_000, expected: 0 },
  { name: 'charges shipping below threshold', total: 49_999, expected: 3_000 },
  { name: 'keeps free shipping above threshold', total: 80_000, expected: 0 }
];
```

이름만 읽어도 경계값을 검증한다는 사실이 드러납니다.
실패 로그에서는 부모 테스트 이름과 합쳐져 `calculateShipping returns free shipping at threshold`처럼 읽힙니다.
CI에서 테스트 결과를 검색할 때도 이런 이름이 훨씬 유용합니다.

## 경계값과 오류 케이스 다루기

### H3. 정상 케이스와 오류 케이스를 분리한다

정상 결과와 오류 throw를 같은 루프로 처리하려면 조건문이 필요해집니다.
조건문이 들어간 table-driven test는 테스트 코드 자체가 복잡해질 수 있습니다.
가능하면 성공 케이스와 실패 케이스를 분리하세요.

```js
function parsePort(value) {
  const port = Number(value);

  if (!Number.isInteger(port) || port < 1 || port > 65_535) {
    throw new RangeError('port must be an integer between 1 and 65535');
  }

  return port;
}

test('parsePort accepts valid ports', async (t) => {
  const cases = [
    { name: 'accepts minimum port', input: '1', expected: 1 },
    { name: 'accepts common dev port', input: '3000', expected: 3000 },
    { name: 'accepts maximum port', input: '65535', expected: 65_535 }
  ];

  for (const testCase of cases) {
    await t.test(testCase.name, () => {
      assert.equal(parsePort(testCase.input), testCase.expected);
    });
  }
});

test('parsePort rejects invalid ports', async (t) => {
  const cases = [
    { name: 'rejects zero', input: '0' },
    { name: 'rejects values above maximum', input: '65536' },
    { name: 'rejects non numeric text', input: 'local' }
  ];

  for (const testCase of cases) {
    await t.test(testCase.name, () => {
      assert.throws(() => parsePort(testCase.input), RangeError);
    });
  }
});
```

이 구조는 조금 길어 보일 수 있지만 실패 리포트는 더 명확합니다.
성공 케이스가 깨졌는지, 오류 케이스가 깨졌는지 부모 테스트 이름에서 바로 알 수 있습니다.
조건문을 줄이면 테스트 자체의 버그 가능성도 낮아집니다.

### H3. 경계값은 table의 첫 번째 후보가 된다

table-driven test는 경계값을 빠뜨리지 않도록 도와줍니다.
최솟값, 최댓값, 바로 아래, 바로 위, 빈 문자열, null에 가까운 값처럼 정책이 바뀌기 쉬운 지점을 케이스로 드러낼 수 있습니다.

```js
test('calculateShipping handles threshold boundaries', async (t) => {
  const cases = [
    { name: 'charges below threshold', total: 49_999, expected: 3_000 },
    { name: 'is free at threshold', total: 50_000, expected: 0 },
    { name: 'is free above threshold', total: 50_001, expected: 0 }
  ];

  for (const testCase of cases) {
    await t.test(testCase.name, () => {
      assert.equal(calculateShipping(testCase.total), testCase.expected);
    });
  }
});

function calculateShipping(total) {
  return total >= 50_000 ? 0 : 3_000;
}
```

경계값 테스트는 제품 정책을 코드로 고정하는 역할을 합니다.
나중에 무료 배송 기준이 바뀌면 이 table을 먼저 수정하게 되고, 변경 범위도 자연스럽게 드러납니다.

## 비동기 table-driven tests 작성하기

### H3. 각 케이스 안에서 await를 명확히 사용한다

비동기 함수도 같은 패턴으로 테스트할 수 있습니다.
차이는 하위 테스트 콜백을 `async`로 만들고, 테스트 대상 호출을 반드시 `await`하는 것입니다.

```js
async function resolvePlan(user) {
  if (user.role === 'admin') return 'enterprise';
  if (user.paid) return 'pro';
  return 'free';
}

test('resolvePlan', async (t) => {
  const cases = [
    { name: 'admin users use enterprise plan', user: { role: 'admin', paid: false }, expected: 'enterprise' },
    { name: 'paid members use pro plan', user: { role: 'member', paid: true }, expected: 'pro' },
    { name: 'free members use free plan', user: { role: 'member', paid: false }, expected: 'free' }
  ];

  for (const testCase of cases) {
    await t.test(testCase.name, async () => {
      const actual = await resolvePlan(testCase.user);

      assert.equal(actual, testCase.expected);
    });
  }
});
```

`forEach()`와 async 콜백을 조합하는 방식은 피하는 편이 좋습니다.
`Array.prototype.forEach()`는 콜백의 promise를 기다리지 않기 때문에 테스트 완료 시점이 예상과 달라질 수 있습니다.
순차 실행이 필요하면 `for...of`와 `await t.test()`가 가장 읽기 쉽습니다.

### H3. 병렬 실행보다 진단 가능성을 먼저 본다

케이스가 많다고 해서 무조건 병렬로 돌릴 필요는 없습니다.
단위 함수처럼 빠른 테스트라면 순차 실행 비용이 작고, 실패 로그도 안정적으로 읽힙니다.
외부 리소스나 fake server를 공유하는 케이스라면 순차 실행이 더 안전할 때도 많습니다.

성능 문제가 실제로 확인된 뒤에 병렬화를 검토하세요.
그때도 공유 상태, 임시 파일, mock, 환경 변수 변경이 없는지 먼저 확인해야 합니다.
테스트 병렬화 기준은 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html)와 함께 점검할 수 있습니다.

## 실무 체크리스트

### H3. table-driven test로 바꾸기 전 확인할 것

- 입력과 기대 결과만 다른 테스트가 반복되는가?
- 케이스 배열이 하나의 규칙으로 설명되는가?
- 각 케이스에 실패 로그에서 읽을 수 있는 이름이 있는가?
- 성공 케이스와 오류 케이스가 섞여 있지 않은가?
- 경계값과 대표값이 함께 들어 있는가?
- 케이스 데이터에 실제 개인정보나 토큰이 섞여 있지 않은가?
- 비동기 케이스에서 `await t.test()`와 `await` 호출을 빠뜨리지 않았는가?

table-driven test는 테스트를 짧게 만드는 기법이 아니라 반복되는 검증 의도를 일정한 모양으로 정리하는 기법입니다.
반복은 줄어들어야 하지만, 실패 원인도 함께 잘 보여야 합니다.

### H3. 피해야 할 신호

케이스 객체가 너무 커지면 table-driven test의 장점이 줄어듭니다.
필드가 열 개를 넘어가거나, 케이스마다 쓰는 필드가 다르거나, 테스트 본문에 조건문이 여러 개 생기면 분리 신호로 봐야 합니다.

```js
// 피하는 편이 좋은 방향
for (const testCase of cases) {
  if (testCase.shouldThrow) {
    assert.throws(() => run(testCase.input));
  } else if (testCase.shouldWarn) {
    assert.deepEqual(run(testCase.input), testCase.expected);
  } else {
    assert.equal(run(testCase.input), testCase.expected);
  }
}
```

이런 구조는 테스트 본문이 작은 실행 엔진처럼 변합니다.
성공, 실패, 경고, 권한 검증을 각각 나누면 테스트 이름과 assertion이 단순해집니다.
테스트를 읽는 사람이 table 해석 규칙을 외우지 않아도 되게 만드는 것이 중요합니다.

## 마무리

Node.js test runner에서 table-driven tests는 배열과 `t.test()`만으로 충분히 구현할 수 있습니다.
반복 케이스를 한곳에 모으고, 각 케이스를 하위 테스트로 실행하면 코드 중복을 줄이면서도 CI 실패 로그의 가독성을 유지할 수 있습니다.

좋은 table-driven test는 케이스 목록이 곧 명세처럼 읽힙니다.
한 table에는 한 규칙만 담고, 케이스 이름은 입력 값이 아니라 검증 의도를 말하게 하세요.
성공 케이스와 오류 케이스를 분리하고, 경계값을 명시하면 테스트는 더 짧아질 뿐 아니라 정책 변경에도 강해집니다.

작게 시작한다면 같은 함수에 대해 반복되는 세 개의 테스트를 하나의 부모 테스트와 세 개의 `t.test()` 하위 테스트로 바꿔 보세요.
그 과정에서 케이스 이름, 경계값, 실패 메시지를 함께 정리하면 테스트 파일이 훨씬 읽기 쉬워집니다.

## 관련 글

- [Node.js test runner test context 가이드](/development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html)
- [Node.js test runner fixture helper 가이드](/development/blog/seo/2026/07/11/nodejs-test-runner-fixture-helper-temp-resource-guide.html)
- [Node.js test runner concurrency 가이드](/development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html)

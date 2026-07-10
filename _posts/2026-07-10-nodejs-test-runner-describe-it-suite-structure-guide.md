---
layout: post
title: "Node.js test runner describe/it 가이드: 테스트 스위트를 읽기 쉽게 구조화하는 법"
date: 2026-07-10 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-describe-it-suite-structure-guide
permalink: /development/blog/seo/2026/07/10/nodejs-test-runner-describe-it-suite-structure-guide.html
alternates:
  ko: /development/blog/seo/2026/07/10/nodejs-test-runner-describe-it-suite-structure-guide.html
  x_default: /development/blog/seo/2026/07/10/nodejs-test-runner-describe-it-suite-structure-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, describe, it, suite, testing, javascript, unit-test, ci]
description: "Node.js test runner의 describe와 it으로 테스트 스위트를 읽기 쉽게 구조화하는 방법을 정리합니다. test와의 차이, 중첩 기준, hook 배치, CI 리포트에서 실패 원인을 빠르게 찾는 이름 작성법을 예제로 설명합니다."
---

테스트 파일이 작을 때는 `test()`를 위에서 아래로 나열해도 충분합니다.
하지만 기능 단위가 늘어나고 성공 케이스, 실패 케이스, 경계값 검증이 섞이면 테스트 이름만으로 전체 의도를 파악하기 어려워집니다.
Node.js 내장 test runner의 `describe()`와 `it()`은 테스트를 자연어에 가까운 계층으로 묶어, 실패 리포트와 코드 구조를 함께 읽기 좋게 만들어 줍니다.

`describe()`는 관련 테스트를 하나의 스위트로 묶고, `it()`은 그 안의 개별 동작을 표현합니다.
기능별로 입력 조건과 기대 결과를 나누면 CI에서 실패한 테스트의 위치를 빠르게 찾을 수 있고, hook도 필요한 범위에만 배치할 수 있습니다.

이 글에서는 Node.js test runner에서 `describe()`와 `it()`을 언제 쓰면 좋은지, 중첩을 어디까지 허용할지, hook과 fixture를 어떻게 배치할지 실무 기준으로 정리합니다.
테스트별 준비와 정리는 [Node.js test runner beforeEach/afterEach 가이드](/development/blog/seo/2026/07/10/nodejs-test-runner-beforeeach-aftereach-hooks-guide.html)를 함께 참고하세요.
테스트 이름 필터링은 [Node.js test runner name pattern 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html), 병렬 실행 전략은 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html)와 이어집니다.

## describe와 it이 필요한 순간

### H3. 기능 단위로 테스트를 읽고 싶을 때

`test()`만 사용해도 Node.js test runner는 충분히 동작합니다.
작은 유틸리티 함수나 독립적인 검증 몇 개만 있는 파일이라면 단순한 `test()` 목록이 오히려 명확합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function normalizeEmail(value) {
  return value.trim().toLowerCase();
}

test('normalizes email address', () => {
  assert.equal(normalizeEmail(' USER@example.COM '), 'user@example.com');
});
```

테스트가 늘어나면 같은 함수의 여러 조건을 하나의 묶음으로 보여 주고 싶어집니다.
이때 `describe()`를 사용하면 스위트 이름이 문맥이 되고, `it()`은 그 문맥 안에서 기대 동작을 설명합니다.

```js
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';

function normalizeEmail(value) {
  return value.trim().toLowerCase();
}

describe('normalizeEmail', () => {
  it('trims surrounding whitespace', () => {
    assert.equal(normalizeEmail(' user@example.com '), 'user@example.com');
  });

  it('lowercases the domain and local part', () => {
    assert.equal(normalizeEmail('USER@EXAMPLE.COM'), 'user@example.com');
  });
});
```

CI 출력에서는 스위트와 테스트 이름이 함께 드러납니다.
실패한 지점이 `normalizeEmail`의 공백 처리인지, 대소문자 처리인지 바로 구분할 수 있습니다.

### H3. 테스트 이름을 짧고 구체적으로 유지한다

`describe()`가 상위 문맥을 제공하므로 `it()` 이름에 같은 단어를 반복할 필요가 줄어듭니다.
예를 들어 모든 테스트 이름이 `normalizeEmail should...`로 시작한다면 스위트 이름으로 올리는 편이 낫습니다.

```js
describe('createInvoice', () => {
  it('calculates subtotal from line items', () => {
    // ...
  });

  it('applies discount before tax', () => {
    // ...
  });

  it('rejects an empty item list', () => {
    // ...
  });
});
```

좋은 테스트 이름은 실패 리포트를 읽는 사람이 코드를 열기 전에 문제 범위를 좁힐 수 있게 합니다.
너무 긴 문장은 검색과 리포트 가독성을 해치므로, 입력 조건과 기대 결과를 중심으로 짧게 씁니다.

## test와 describe/it의 선택 기준

### H3. 단일 검증 파일은 test만으로도 충분하다

모든 테스트 파일에 `describe()`를 의무적으로 넣을 필요는 없습니다.
테스트가 세 개 이하이고 같은 수준의 독립 검증이라면 `test()`만 사용하는 편이 더 단순합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('formats a Korean date label', () => {
  const date = new Date('2026-07-10T11:00:00.000Z');

  assert.equal(
    new Intl.DateTimeFormat('ko-KR', { timeZone: 'Asia/Seoul' }).format(date),
    '2026. 7. 10.'
  );
});
```

파일 자체가 이미 하나의 주제를 나타낸다면 별도 스위트가 중복이 될 수 있습니다.
반대로 한 파일 안에 여러 함수나 여러 사용자 흐름이 들어오면 `describe()`로 경계를 분명히 하는 편이 좋습니다.

### H3. 같은 fixture를 공유하면 describe로 범위를 좁힌다

스위트마다 필요한 fixture가 다르면 `describe()` 안에 hook을 배치해 범위를 제한할 수 있습니다.
이렇게 하면 한 기능의 준비 코드가 다른 기능 테스트에 영향을 주지 않습니다.

```js
import { beforeEach, describe, it } from 'node:test';
import assert from 'node:assert/strict';

function createCart() {
  return {
    items: [],
    add(item) {
      this.items.push(item);
    },
    total() {
      return this.items.reduce((sum, item) => sum + item.price, 0);
    }
  };
}

describe('cart totals', () => {
  let cart;

  beforeEach(() => {
    cart = createCart();
  });

  it('starts from zero', () => {
    assert.equal(cart.total(), 0);
  });

  it('adds item prices', () => {
    cart.add({ name: 'keyboard', price: 80_000 });
    cart.add({ name: 'mouse', price: 30_000 });

    assert.equal(cart.total(), 110_000);
  });
});
```

`cart` 변수는 이 스위트 안에서만 의미가 있습니다.
파일 아래쪽에 다른 스위트가 추가되더라도 fixture 범위가 섞이지 않습니다.
테스트 수가 늘어날수록 이런 작은 경계가 유지보수 비용을 줄여 줍니다.

## 중첩 구조를 설계하는 법

### H3. 두 단계 이상 중첩할 때는 이유를 남긴다

`describe()`는 중첩할 수 있습니다.
하지만 너무 깊은 중첩은 테스트 본문보다 구조를 읽는 시간이 길어지게 만듭니다.
대부분의 단위 테스트는 기능 이름과 조건 그룹, 두 단계 정도면 충분합니다.

```js
describe('parseWebhookPayload', () => {
  describe('valid payload', () => {
    it('returns event type and resource id', () => {
      // ...
    });
  });

  describe('invalid payload', () => {
    it('rejects missing event type', () => {
      // ...
    });

    it('rejects malformed JSON', () => {
      // ...
    });
  });
});
```

이 구조는 성공 입력과 실패 입력을 빠르게 구분하게 해 줍니다.
반대로 `service`, `method`, `case`, `variant`, `assertion`처럼 네다섯 단계로 내려가면 테스트 이름이 길어지고 hook 범위도 추적하기 어려워집니다.

### H3. 조건별 그룹은 리포트 검색을 기준으로 나눈다

중첩 기준은 개발자가 실패 리포트에서 검색할 키워드와 가까워야 합니다.
예를 들어 결제 로직이라면 `card payment`, `bank transfer`, `refund` 같은 사용자 흐름이 좋은 그룹이 될 수 있습니다.
구현 내부의 private 함수 이름이나 임시 변수 이름을 스위트 이름으로 쓰면 리팩터링 때 테스트 이름까지 자주 바뀝니다.

```js
describe('checkout', () => {
  describe('card payment', () => {
    it('creates an order when authorization succeeds', () => {
      // ...
    });

    it('keeps the cart when authorization fails', () => {
      // ...
    });
  });
});
```

테스트 이름은 문서이기도 합니다.
사용자가 관찰할 수 있는 동작과 도메인 언어를 중심으로 쓰면, 구현이 바뀌어도 테스트의 의미가 오래 유지됩니다.

## hook 배치와 실행 순서

### H3. hook은 가장 좁은 describe 안에 둔다

`beforeEach()`나 `afterEach()`가 특정 스위트에만 필요하다면 그 스위트 안에 둡니다.
파일 상단의 전역 hook에 모든 준비를 몰아넣으면, 어떤 테스트가 어떤 상태를 필요로 하는지 흐려집니다.

```js
import { afterEach, beforeEach, describe, it } from 'node:test';
import assert from 'node:assert/strict';

describe('temporary cache', () => {
  let cache;

  beforeEach(() => {
    cache = new Map();
  });

  afterEach(() => {
    cache.clear();
  });

  it('stores a value by key', () => {
    cache.set('profile:user_1', { id: 'user_1' });

    assert.deepEqual(cache.get('profile:user_1'), { id: 'user_1' });
  });
});
```

hook이 있는 위치는 테스트의 수명을 보여 줍니다.
이 스위트를 벗어난 테스트에는 `cache`가 필요 없다는 사실도 코드에서 바로 드러납니다.

### H3. 공유 상태보다 helper 함수를 우선한다

`describe()` 안에서 변수를 공유하는 방식은 편하지만, 지나치게 많아지면 테스트 본문이 어떤 입력으로 실행되는지 숨겨집니다.
공유 상태가 세 개 이상 늘어나면 fixture helper를 검토하세요.

```js
function createUserFixture(overrides = {}) {
  return {
    id: 'user_1',
    name: 'Kim',
    role: 'member',
    ...overrides
  };
}

describe('authorizeUser', () => {
  it('allows admin users', () => {
    const user = createUserFixture({ role: 'admin' });

    assert.equal(authorizeUser(user, 'settings:write'), true);
  });
});
```

helper 함수는 테스트 입력을 본문 가까이에 남깁니다.
공통 기본값은 재사용하되, 중요한 차이는 각 테스트 안에서 보이게 만드는 것이 핵심입니다.

## CI와 유지보수 체크리스트

### H3. 이름 필터링이 쉬운 구조인지 확인한다

Node.js test runner는 테스트 이름 패턴으로 실행 대상을 좁힐 수 있습니다.
스위트 이름과 테스트 이름이 도메인 키워드를 포함하면 로컬 디버깅이 쉬워집니다.

```bash
node --test --test-name-pattern "checkout"
node --test --test-name-pattern "invalid payload"
```

이 명령을 자주 사용한다면 스위트 이름이 검색어 역할을 합니다.
`works`, `handles case`, `should be valid`처럼 모호한 이름은 실패 리포트에서도 필터링에서도 도움이 적습니다.

### H3. 스위트 구조 점검 목록

발행 전 코드 리뷰처럼 테스트 구조도 주기적으로 점검하면 좋습니다.

- 한 파일 안에 서로 다른 기능이 섞여 있으면 `describe()`로 나누었는가?
- `it()` 이름만 보고 입력 조건과 기대 결과를 대략 알 수 있는가?
- hook은 필요한 가장 좁은 스위트 안에 있는가?
- 중첩이 세 단계 이상이라면 파일 분리나 helper 함수가 더 나은가?
- CI 실패 리포트에서 검색할 수 있는 도메인 키워드가 포함되어 있는가?

이 기준은 테스트를 예쁘게 정리하기 위한 규칙이 아닙니다.
실패가 났을 때 원인을 빨리 좁히고, 다음 사람이 테스트의 의도를 잃지 않게 하기 위한 운영 기준입니다.

## 마무리

`describe()`와 `it()`은 단순한 스타일 선택이 아닙니다.
테스트가 늘어날수록 실패 리포트, hook 범위, fixture 수명, 로컬 필터링 경험을 함께 결정합니다.

작은 파일은 `test()`만으로 시작하고, 기능이나 조건 그룹이 보이기 시작하면 `describe()`로 문맥을 올리세요.
`it()` 이름은 짧고 구체적으로 쓰고, hook은 가장 좁은 스위트 안에 둡니다.
이 세 가지 원칙만 지켜도 Node.js test runner 기반 테스트는 훨씬 읽기 쉽고 고치기 쉬운 구조가 됩니다.

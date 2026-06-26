---
layout: post
title: "Node.js assert.deepStrictEqual 가이드: 객체와 배열을 정확하게 비교하는 테스트 패턴"
date: 2026-06-26 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-deepstrictequal-object-comparison-test-guide
permalink: /development/blog/seo/2026/06/26/nodejs-assert-deepstrictequal-object-comparison-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/26/nodejs-assert-deepstrictequal-object-comparison-test-guide.html
  x_default: /development/blog/seo/2026/06/26/nodejs-assert-deepstrictequal-object-comparison-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, deepStrictEqual, test, assertion, object, backend]
description: "Node.js assert.deepStrictEqual()로 객체, 배열, 중첩 데이터의 값을 정확하게 비교하는 방법을 정리합니다. equal과의 차이, 부분 비교가 필요한 상황, 테스트가 쉽게 깨지지 않게 설계하는 기준을 예제로 설명합니다."
---

Node.js 테스트에서 객체나 배열을 비교할 때 `assert.equal()`만으로는 충분하지 않은 경우가 많습니다.
`equal()`은 두 값이 같은 참조인지 확인하는 데 가깝기 때문에, 내용이 같은 두 객체라도 서로 다른 객체라면 실패합니다.
응답 DTO, 설정 객체, 정규화 결과, 이벤트 페이로드처럼 구조화된 데이터를 검증하려면 값의 내부 구조까지 비교하는 assertion이 필요합니다.

이때 Node.js `node:assert/strict`의 `assert.deepStrictEqual()`을 사용하면 객체와 배열의 중첩 값을 엄격하게 비교할 수 있습니다.
[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 deep strict equality 비교가 원시값, 객체 속성, 배열 원소, 타입 차이를 함께 고려한다고 설명합니다.
테스트가 "대략 비슷한 값"이 아니라 "이 구조와 값이 정확히 나와야 한다"는 계약을 표현해야 한다면 가장 먼저 떠올릴 API입니다.

문자열 패턴 검증은 [Node.js assert.match 가이드](/development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html)를 참고하세요.
응답 객체의 일부 필드만 확인해야 한다면 [Node.js assert.partialDeepStrictEqual 가이드](/development/blog/seo/2026/06/23/nodejs-assert-partialdeepstrictequal-api-response-test-guide.html)가 더 적합합니다.
명시적으로 실패 경로를 표시해야 하는 테스트는 [Node.js assert.fail 가이드](/development/blog/seo/2026/06/26/nodejs-assert-fail-explicit-test-failure-guide.html)와 연결됩니다.

## deepStrictEqual이 필요한 이유

### H3. 객체 내용이 같은지 비교한다

두 객체가 같은 속성과 값을 가져도 자바스크립트에서는 서로 다른 참조일 수 있습니다.
그래서 객체 결과를 `assert.equal()`로 비교하면 기대와 다르게 실패합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function normalizeUser(input) {
  return {
    id: input.id,
    email: input.email.trim().toLowerCase(),
    roles: input.roles.slice().sort()
  };
}

test('normalizeUser returns normalized user object', () => {
  const user = normalizeUser({
    id: 'user_1',
    email: '  SOO@example.com ',
    roles: ['admin', 'member']
  });

  assert.deepStrictEqual(user, {
    id: 'user_1',
    email: 'soo@example.com',
    roles: ['admin', 'member']
  });
});
```

이 테스트는 반환 객체의 모양과 값을 한 번에 검증합니다.
필드별로 `assert.equal(user.id, ...)`를 여러 줄 쓰는 것보다 읽기 쉽고, 실패했을 때 실제 객체와 기대 객체의 차이를 확인하기도 좋습니다.
객체 전체가 테스트 계약이라면 `deepStrictEqual()`이 가장 자연스럽습니다.

### H3. 배열과 중첩 객체도 함께 확인한다

실무 데이터는 단순 객체 하나로 끝나지 않는 경우가 많습니다.
배열 안에 객체가 들어가거나, 객체 안에 다시 설정 객체가 들어갑니다.
`deepStrictEqual()`은 이런 중첩 구조까지 내려가며 값을 비교합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function buildInvoiceSummary(invoice) {
  return {
    invoiceId: invoice.id,
    totals: {
      subtotal: invoice.items.reduce((sum, item) => sum + item.price, 0),
      currency: 'KRW'
    },
    itemNames: invoice.items.map((item) => item.name)
  };
}

test('buildInvoiceSummary creates nested summary', () => {
  const summary = buildInvoiceSummary({
    id: 'inv_100',
    items: [
      { name: 'hosting', price: 12000 },
      { name: 'domain', price: 18000 }
    ]
  });

  assert.deepStrictEqual(summary, {
    invoiceId: 'inv_100',
    totals: {
      subtotal: 30000,
      currency: 'KRW'
    },
    itemNames: ['hosting', 'domain']
  });
});
```

중첩 구조를 검증할 때는 기대 객체를 테스트 안에 명확하게 배치하는 것이 좋습니다.
테스트를 읽는 사람이 반환 구조를 바로 파악할 수 있고, 변경이 발생했을 때 어떤 필드가 계약인지 알 수 있습니다.
반대로 기대 객체를 너무 멀리 떨어진 fixture 파일에 숨기면 작은 테스트의 가독성이 떨어질 수 있습니다.

## equal과 deepStrictEqual의 차이

### H3. equal은 참조와 원시값 비교에 적합하다

`assert.equal()`은 숫자, 문자열, 불리언처럼 단순한 원시값을 검증할 때 좋습니다.
객체에는 같은 참조인지 확인해야 하는 특수한 상황이 아니라면 잘 맞지 않습니다.

```js
import assert from 'node:assert/strict';

const expected = { status: 'ok' };
const actual = { status: 'ok' };

assert.notEqual(actual, expected);
assert.deepStrictEqual(actual, expected);
```

위 예시에서 두 객체의 내용은 같지만 참조는 다릅니다.
따라서 `notEqual()`은 통과하고 `deepStrictEqual()`도 통과합니다.
테스트 의도가 값의 내용인지, 같은 인스턴스인지에 따라 API를 구분해야 합니다.

### H3. 타입 차이를 느슨하게 넘기지 않는다

`node:assert/strict`의 `deepStrictEqual()`은 타입 차이를 엄격하게 봅니다.
문자열 `"1"`과 숫자 `1`은 다르고, 배열과 배열처럼 생긴 객체도 다르게 취급해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function parseQuery(input) {
  return {
    page: Number(input.page || 1),
    pageSize: Number(input.pageSize || 20)
  };
}

test('parseQuery returns numeric pagination values', () => {
  assert.deepStrictEqual(parseQuery({ page: '2', pageSize: '50' }), {
    page: 2,
    pageSize: 50
  });
});
```

API나 쿼리 파라미터를 다루면 문자열과 숫자가 섞이기 쉽습니다.
이 테스트는 반환 타입까지 계약으로 고정합니다.
타입 변환이 중요한 코드에서는 느슨한 비교보다 엄격한 비교가 버그를 더 빨리 드러냅니다.

## 깨지기 쉬운 테스트를 피하는 기준

### H3. 전체 객체가 계약일 때만 전체 비교를 사용한다

`deepStrictEqual()`은 강력하지만, 모든 상황에서 최선은 아닙니다.
객체 전체가 외부 계약이 아니라 내부 구현 세부사항이라면 전체 비교가 너무 쉽게 깨질 수 있습니다.
예를 들어 API 응답에 디버그용 필드나 서버 생성 시간이 추가되는 순간 테스트가 불필요하게 실패할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function createSession(userId) {
  return {
    userId,
    status: 'active',
    expiresInSeconds: 3600,
    issuedAt: new Date('2026-06-26T11:00:00.000Z')
  };
}

test('createSession returns stable contract fields', () => {
  const session = createSession('user_1');

  assert.deepStrictEqual(
    {
      userId: session.userId,
      status: session.status,
      expiresInSeconds: session.expiresInSeconds
    },
    {
      userId: 'user_1',
      status: 'active',
      expiresInSeconds: 3600
    }
  );
});
```

여기서는 `issuedAt` 전체 값을 직접 비교하지 않습니다.
시간 값은 테스트마다 흔들릴 가능성이 크기 때문입니다.
검증하고 싶은 계약 필드만 따로 꺼내 비교하면 테스트가 덜 취약해집니다.

### H3. 일부 필드만 중요하면 partialDeepStrictEqual을 고려한다

객체의 일부 속성만 검증하면 충분한 경우에는 `assert.partialDeepStrictEqual()`이 더 좋은 선택일 수 있습니다.
특히 API 응답처럼 필드가 점진적으로 늘어나는 구조에서는 부분 비교가 유지보수에 유리합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function buildUserResponse(user) {
  return {
    id: user.id,
    profile: {
      displayName: user.name,
      locale: 'ko-KR'
    },
    links: {
      self: `/users/${user.id}`
    }
  };
}

test('buildUserResponse includes required profile fields', () => {
  const response = buildUserResponse({
    id: 'user_1',
    name: 'Soo'
  });

  assert.partialDeepStrictEqual(response, {
    id: 'user_1',
    profile: {
      displayName: 'Soo'
    }
  });
});
```

부분 비교는 "이 필드는 반드시 있어야 한다"는 계약을 표현합니다.
반대로 전체 응답 스키마가 바뀌면 테스트가 실패해야 하는 상황이라면 `deepStrictEqual()`이 더 적합합니다.
핵심은 테스트가 지켜야 할 계약의 크기를 먼저 정하는 것입니다.

## 실무 패턴

### H3. 변환 함수의 입출력 테스트에 잘 맞는다

데이터 변환 함수는 입력과 출력이 명확합니다.
부작용이 적고 반환값이 안정적이라면 `deepStrictEqual()`로 전체 결과를 비교하기 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function groupOrdersByStatus(orders) {
  return orders.reduce((groups, order) => {
    groups[order.status] ||= [];
    groups[order.status].push(order.id);
    return groups;
  }, {});
}

test('groupOrdersByStatus groups order ids by status', () => {
  const result = groupOrdersByStatus([
    { id: 'order_1', status: 'paid' },
    { id: 'order_2', status: 'pending' },
    { id: 'order_3', status: 'paid' }
  ]);

  assert.deepStrictEqual(result, {
    paid: ['order_1', 'order_3'],
    pending: ['order_2']
  });
});
```

이런 테스트는 실패 원인을 찾기 쉽습니다.
입력과 기대 출력이 같은 테스트 안에 있으므로, 정렬 기준이나 그룹핑 기준이 바뀌었는지 바로 확인할 수 있습니다.
변환 함수가 커질수록 작은 입력 샘플을 여러 개 두고 경계 조건을 나누어 테스트하는 편이 좋습니다.

### H3. 순서가 의미 없는 배열은 먼저 정렬한다

배열 비교에서 순서는 중요한 계약입니다.
같은 원소를 가져도 순서가 다르면 `deepStrictEqual()`은 실패합니다.
정렬 순서가 요구사항이 아니라면 테스트 전에 안정적인 기준으로 정렬해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function collectEnabledFeatureNames(features) {
  return features
    .filter((feature) => feature.enabled)
    .map((feature) => feature.name);
}

test('collectEnabledFeatureNames returns enabled feature names', () => {
  const names = collectEnabledFeatureNames([
    { name: 'billing', enabled: true },
    { name: 'search', enabled: true },
    { name: 'export', enabled: false }
  ]).sort();

  assert.deepStrictEqual(names, ['billing', 'search']);
});
```

정렬이 요구사항이라면 함수가 반환한 순서를 그대로 검증해야 합니다.
하지만 원소 집합만 중요하다면 테스트 안에서 정렬해 불필요한 실패를 줄일 수 있습니다.
테스트가 무엇을 보장하는지 주석 없이도 드러나도록 입력과 기대값을 단순하게 유지하세요.

## 체크리스트

- 객체와 배열의 내용 전체가 계약이면 `assert.deepStrictEqual()`을 사용한다.
- 단순 원시값은 `assert.equal()`로 충분한지 먼저 본다.
- 일부 필드만 중요하면 `assert.partialDeepStrictEqual()`을 고려한다.
- 날짜, 난수, 생성 ID처럼 흔들리는 값은 전체 객체 비교에서 분리한다.
- 배열 순서가 요구사항인지, 단순 집합 검증인지 테스트 전에 결정한다.
- 기대 객체는 가능한 한 테스트 가까이에 두어 변경 의도를 읽기 쉽게 만든다.

## FAQ

### H3. deepStrictEqual은 API 응답 테스트에 항상 좋은가요?

항상 그렇지는 않습니다.
응답 전체가 공개 계약이고 필드 추가도 명시적으로 관리한다면 좋습니다.
하지만 응답에 부가 메타데이터가 자주 추가된다면 필요한 필드만 부분 비교하는 편이 유지보수에 유리합니다.

### H3. 객체 속성 순서가 다르면 실패하나요?

일반 객체의 속성 순서를 테스트 계약으로 삼는 경우는 드뭅니다.
중요한 것은 키와 값의 구조입니다.
반면 배열 원소의 순서는 값의 일부이므로 순서가 다르면 실패합니다.

### H3. snapshot 테스트와 어떤 차이가 있나요?

`deepStrictEqual()`은 기대 값을 테스트 코드 안에서 명시적으로 작성합니다.
그래서 작은 객체나 핵심 계약을 검증할 때 의도가 선명합니다.
snapshot은 출력이 크거나 반복될 때 편할 수 있지만, 변경 검토를 소홀히 하면 실제 계약보다 넓은 범위를 무심코 고정할 수 있습니다.

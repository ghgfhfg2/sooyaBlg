---
layout: post
title: "Node.js assert.strictEqual 가이드: 원시값과 참조를 정확하게 검증하는 테스트 패턴"
date: 2026-06-29 08:00:00 +0900
lang: ko
translation_key: nodejs-assert-strictequal-primitive-reference-test-guide
permalink: /development/blog/seo/2026/06/29/nodejs-assert-strictequal-primitive-reference-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/29/nodejs-assert-strictequal-primitive-reference-test-guide.html
  x_default: /development/blog/seo/2026/06/29/nodejs-assert-strictequal-primitive-reference-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, strictEqual, test, assertion, javascript, backend]
description: "Node.js assert.strictEqual()로 원시값, 타입 변환 결과, 객체 참조를 정확하게 검증하는 방법을 정리합니다. equal과의 차이, Object.is 기준, deepStrictEqual과 함께 쓰는 패턴을 예제로 설명합니다."
---

Node.js 테스트에서 가장 자주 쓰는 assertion 중 하나는 `assert.strictEqual()`입니다.
숫자, 문자열, 불리언, `null`, `undefined`처럼 단일 값이 정확히 맞아야 하는 경우에 테스트 의도를 가장 짧게 표현할 수 있습니다.
또한 객체를 비교할 때는 내용이 아니라 같은 참조인지 확인하므로, 캐시나 싱글턴처럼 동일 인스턴스 재사용이 계약인 코드에도 사용할 수 있습니다.

Node.js `node:assert/strict`의 `assert.strictEqual(actual, expected)`은 두 값이 엄격하게 같은지 확인합니다.
[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 strict assertion mode에서 엄격한 비교와 더 나은 실패 메시지를 제공한다고 설명합니다.
테스트가 "비슷한 값"이 아니라 "정확히 이 값"을 요구한다면 `strictEqual()`을 기본 선택지로 두는 것이 좋습니다.

객체와 배열의 내부 값까지 비교해야 한다면 [Node.js assert.deepStrictEqual 가이드](/development/blog/seo/2026/06/26/nodejs-assert-deepstrictequal-object-comparison-test-guide.html)를 참고하세요.
두 값이 같아지면 안 되는 상황은 [Node.js assert.notStrictEqual 가이드](/development/blog/seo/2026/06/27/nodejs-assert-notstrictequal-primitive-difference-test-guide.html)와 연결됩니다.
문자열 패턴을 검증해야 한다면 [Node.js assert.match 가이드](/development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html)가 더 적합합니다.

## strictEqual이 필요한 이유

### H3. 원시값의 정확한 결과를 고정한다

`strictEqual()`은 함수가 반환해야 하는 값이 하나로 정해져 있을 때 가장 읽기 좋습니다.
예를 들어 상태 전환 결과, 계산 결과, 파싱된 옵션 값처럼 작은 단위의 계약을 확인할 때 유용합니다.
값이 틀리면 테스트 실패 메시지에서 실제 값과 기대값을 바로 비교할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function nextStatus(current) {
  if (current === 'draft') {
    return 'review';
  }

  if (current === 'review') {
    return 'published';
  }

  return current;
}

test('nextStatus moves draft to review', () => {
  assert.strictEqual(nextStatus('draft'), 'review');
});
```

이 테스트는 `draft`가 `review`로 바뀌어야 한다는 계약만 검증합니다.
객체 전체나 여러 조건을 한 번에 확인하지 않기 때문에 실패 원인이 선명합니다.
작은 순수 함수는 이런 식으로 입력과 출력 하나를 좁게 고정하면 유지보수하기 쉽습니다.

### H3. 타입 변환 결과까지 계약으로 만든다

JavaScript에서는 `"10"`과 `10`처럼 값은 비슷해 보이지만 타입이 다른 데이터가 자주 섞입니다.
API 쿼리, 환경 변수, CLI 인자처럼 외부 입력은 대부분 문자열로 들어오기 때문입니다.
테스트에서 `strictEqual()`을 쓰면 반환값의 타입까지 의도적으로 고정할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function parseRetryCount(env) {
  return Number(env.RETRY_COUNT || 3);
}

test('parseRetryCount returns a number', () => {
  const retryCount = parseRetryCount({ RETRY_COUNT: '5' });

  assert.strictEqual(retryCount, 5);
  assert.strictEqual(typeof retryCount, 'number');
});
```

첫 번째 assertion은 값이 숫자 `5`인지 확인합니다.
두 번째 assertion은 결과 타입이 숫자라는 점을 문서화합니다.
입력 정규화 코드에서는 "값만 맞는지"보다 "타입까지 맞는지"가 실제 버그를 더 잘 잡습니다.

## equal과 strictEqual의 차이

### H3. 느슨한 비교를 테스트 기본값으로 두지 않는다

`assert.equal()`은 legacy assertion mode에서는 느슨한 동등성에 기대는 API입니다.
반면 `node:assert/strict`의 `strictEqual()`은 타입 차이를 넘기지 않습니다.
새 테스트를 작성할 때는 헷갈리는 변환을 숨기지 않도록 엄격한 비교를 기본값으로 삼는 편이 안전합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function normalizeEnabled(value) {
  return value === true || value === 'true';
}

test('normalizeEnabled converts string true to boolean true', () => {
  const enabled = normalizeEnabled('true');

  assert.strictEqual(enabled, true);
  assert.notStrictEqual(enabled, 'true');
});
```

이 테스트는 문자열 입력이 불리언으로 변환되어야 한다는 요구사항을 보여 줍니다.
`equal()`처럼 느슨한 비교를 쓰면 문자열과 불리언의 차이가 흐려질 수 있습니다.
변환 함수의 테스트에서는 결과 타입을 의식적으로 드러내는 assertion이 더 좋은 문서가 됩니다.

### H3. Object.is 기준의 특수값을 이해한다

`strictEqual()`은 `Object.is()` 기반의 엄격한 비교와 연결됩니다.
그래서 `NaN`은 `NaN`과 같은 값으로 취급되고, `0`과 `-0`은 구분됩니다.
대부분의 테스트에서는 문제가 되지 않지만 숫자 정규화 코드라면 이런 특수값을 알고 있어야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function parseRatio(input) {
  const value = Number(input);

  if (Number.isNaN(value)) {
    return NaN;
  }

  return value;
}

test('parseRatio keeps invalid numeric input as NaN', () => {
  const ratio = parseRatio('not-a-number');

  assert.strictEqual(ratio, NaN);
});
```

예전에는 `NaN` 비교를 별도 함수로 처리해야 한다고 기억하는 경우가 많았습니다.
하지만 Node.js의 엄격 assertion에서는 `strictEqual(ratio, NaN)`으로 의도를 직접 표현할 수 있습니다.
도메인에서 `-0`이 의미를 갖는다면 반대로 `0`과 `-0`을 구분해야 하는지도 테스트 이름에 드러내세요.

## 객체 참조를 검증하는 기준

### H3. 같은 인스턴스를 재사용해야 할 때 사용한다

객체에 `strictEqual()`을 사용하면 객체의 내부 값이 아니라 참조가 같은지 확인합니다.
이 특성은 캐시, 연결 풀, 설정 로더처럼 같은 인스턴스를 재사용해야 하는 코드에서 유용합니다.
값이 같은 새 객체를 만드는 것과 실제로 같은 객체를 돌려주는 것은 서로 다른 계약입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

const clients = new Map();

function getApiClient(region) {
  if (!clients.has(region)) {
    clients.set(region, {
      region,
      connected: true
    });
  }

  return clients.get(region);
}

test('getApiClient reuses the same client for a region', () => {
  const first = getApiClient('ap-northeast-2');
  const second = getApiClient('ap-northeast-2');

  assert.strictEqual(first, second);
});
```

이 테스트는 같은 지역에 대해 새 클라이언트를 계속 만들면 안 된다는 계약을 표현합니다.
`deepStrictEqual()`을 쓰면 두 객체의 내용이 같은지만 확인하므로 재사용 여부를 놓칠 수 있습니다.
참조 재사용이 성능이나 상태 공유에 중요하다면 `strictEqual()`이 정확한 도구입니다.

### H3. 객체 내용 비교에는 deepStrictEqual을 사용한다

반대로 함수가 새 객체를 만들어 반환해야 한다면 `strictEqual()`만으로는 부족합니다.
객체 내용이 같은지 확인해야 할 때는 `deepStrictEqual()`을 쓰고, 참조가 달라야 한다면 `notStrictEqual()`을 함께 둡니다.
테스트가 무엇을 계약으로 삼는지에 따라 assertion을 나눠야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function normalizeProfile(profile) {
  return {
    id: profile.id,
    email: profile.email.trim().toLowerCase()
  };
}

test('normalizeProfile returns a normalized copy', () => {
  const original = {
    id: 'user_1',
    email: '  SOO@example.com '
  };

  const normalized = normalizeProfile(original);

  assert.notStrictEqual(normalized, original);
  assert.deepStrictEqual(normalized, {
    id: 'user_1',
    email: 'soo@example.com'
  });
});
```

첫 번째 assertion은 새 객체를 반환한다는 참조 계약을 확인합니다.
두 번째 assertion은 새 객체의 내용이 정확한지 확인합니다.
객체 테스트에서는 "같은 객체인가"와 "같은 값인가"를 분리하면 테스트 실패 이유가 훨씬 명확해집니다.

## 실무에서 조심할 점

### H3. 기대값이 명확할 때 부정 assertion으로 우회하지 않는다

기대값이 분명한데 `notStrictEqual()`만 쓰면 테스트가 너무 넓어집니다.
예를 들어 상태가 `draft`가 아니기만 하면 되는 것이 아니라 `published`여야 한다면, 그 값을 직접 고정해야 합니다.
부정 assertion은 금지 조건을 표현할 때 쓰고, 핵심 결과는 긍정 assertion으로 확인하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function publish(status) {
  if (status !== 'review') {
    return status;
  }

  return 'published';
}

test('publish changes review to published', () => {
  const status = publish('review');

  assert.strictEqual(status, 'published');
});
```

이 테스트는 `status`가 `review`가 아니어야 한다고 말하지 않습니다.
정확히 `published`여야 한다고 말합니다.
테스트는 가능한 한 구현자가 지켜야 하는 최종 계약을 직접 표현해야 합니다.

### H3. 실패 메시지는 중요한 경계에서만 추가한다

`strictEqual()`은 기본 실패 메시지만으로도 실제 값과 기대값을 보여 줍니다.
그래서 모든 assertion에 설명 문구를 붙이면 오히려 테스트가 장황해질 수 있습니다.
다만 같은 값 비교가 여러 번 반복되는 경계 테스트에서는 메시지를 붙여 실패 위치를 빠르게 찾을 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function selectPlanLimit(plan) {
  if (plan === 'pro') {
    return 50;
  }

  return 3;
}

test('selectPlanLimit returns expected project limits', () => {
  assert.strictEqual(selectPlanLimit('free'), 3, 'free plan project limit');
  assert.strictEqual(selectPlanLimit('pro'), 50, 'pro plan project limit');
});
```

메시지는 assertion이 실패했을 때만 읽힙니다.
따라서 테스트 이름과 코드만으로 충분히 명확한 경우에는 생략해도 됩니다.
반대로 여러 케이스를 한 테스트에 묶었다면, 짧은 메시지가 디버깅 시간을 줄여 줍니다.

## strictEqual 체크리스트

### H3. 값 비교와 참조 비교를 구분한다

`strictEqual()`을 사용할 때는 먼저 비교 대상이 원시값인지 객체인지 확인하세요.
원시값이라면 정확한 값과 타입을 확인하는 용도입니다.
객체라면 같은 인스턴스인지 확인하는 용도이며, 내부 값 비교가 필요하면 `deepStrictEqual()`을 선택해야 합니다.

- 원시값 결과가 하나로 정해져 있으면 `strictEqual()`을 쓴다.
- 타입 변환 코드에서는 값과 `typeof`를 함께 확인할 수 있다.
- 객체 재사용이 계약이면 `strictEqual(actual, expectedReference)`을 쓴다.
- 객체 내용 비교가 계약이면 `deepStrictEqual()`을 쓴다.
- 같으면 안 되는 값은 `notStrictEqual()`로 분리한다.

이 기준만 지켜도 테스트 의도가 훨씬 선명해집니다.
특히 Node.js 내장 테스트 러너와 함께 작은 assertion을 자주 쓰면, 회귀가 발생했을 때 어떤 계약이 깨졌는지 빠르게 파악할 수 있습니다.
`strictEqual()`은 단순한 API지만, 정확한 값과 참조를 구분하는 습관을 만드는 데 가장 좋은 출발점입니다.

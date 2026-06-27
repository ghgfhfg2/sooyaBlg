---
layout: post
title: "Node.js assert.notStrictEqual 가이드: 원시값과 참조가 달라야 하는 테스트 패턴"
date: 2026-06-27 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-notstrictequal-primitive-difference-test-guide
permalink: /development/blog/seo/2026/06/27/nodejs-assert-notstrictequal-primitive-difference-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/27/nodejs-assert-notstrictequal-primitive-difference-test-guide.html
  x_default: /development/blog/seo/2026/06/27/nodejs-assert-notstrictequal-primitive-difference-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, notStrictEqual, test, assertion, javascript, backend]
description: "Node.js assert.notStrictEqual()로 두 값이 엄격히 같으면 안 되는 상황을 검증하는 방법을 정리합니다. notEqual과의 차이, 원시값 비교, 객체 참조 비교, 부정 assertion을 안전하게 쓰는 기준을 예제로 설명합니다."
---

테스트에서는 어떤 값이 기대값과 같아야 하는지만큼, 같아지면 안 되는 값도 확인해야 합니다.
새로 만든 식별자가 이전 식별자와 달라야 하거나, 캐시 키가 환경별로 분리되어야 하거나, 객체를 새 인스턴스로 복사해야 하는 상황이 그렇습니다.
이때 Node.js `node:assert/strict`의 `assert.notStrictEqual()`을 사용하면 두 값이 `Object.is()` 기준으로 같지 않아야 한다는 의도를 명확하게 표현할 수 있습니다.

[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 `assert.notStrictEqual(actual, expected)`이 두 값이 엄격히 같지 않은지 검증하는 API라고 설명합니다.
`node:assert/strict`를 사용하면 느슨한 동등성보다 타입과 값의 차이를 더 분명하게 다룰 수 있습니다.
특히 숫자, 문자열, 불리언, 심볼, 객체 참조처럼 "같으면 안 되는 값"을 좁게 검증할 때 유용합니다.

객체와 배열의 중첩 값이 달라야 한다면 [Node.js assert.notDeepStrictEqual 가이드](/development/blog/seo/2026/06/27/nodejs-assert-notdeepstrictequal-negative-test-guide.html)를 참고하세요.
객체 전체가 정확히 같아야 하는 테스트는 [Node.js assert.deepStrictEqual 가이드](/development/blog/seo/2026/06/26/nodejs-assert-deepstrictequal-object-comparison-test-guide.html)와 연결됩니다.
테스트에서 명시적인 실패 이유를 남겨야 한다면 [Node.js assert.fail 가이드](/development/blog/seo/2026/06/26/nodejs-assert-fail-explicit-test-failure-guide.html)를 함께 보면 좋습니다.

## assert.notStrictEqual이 필요한 이유

### H3. 원시값이 같아지면 안 되는 회귀를 잡는다

`assert.notStrictEqual()`은 값 하나의 차이가 중요한 테스트에 잘 맞습니다.
예를 들어 이전 상태와 새 상태가 같은 문자열로 남으면 안 되거나, 생성된 토큰 prefix가 환경별로 달라야 하는 경우입니다.
이때 객체 전체를 비교하기보다 핵심 필드 하나를 좁게 검증하는 편이 테스트 의도가 더 선명합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function buildCacheKey(environment, userId) {
  return `${environment}:user:${userId}`;
}

test('cache keys are separated by environment', () => {
  const productionKey = buildCacheKey('prod', 'user_1');
  const stagingKey = buildCacheKey('staging', 'user_1');

  assert.notStrictEqual(productionKey, stagingKey);
});
```

이 테스트는 운영 환경과 스테이징 환경의 캐시 키가 같아지면 안 된다는 계약을 표현합니다.
만약 구현이 환경명을 빠뜨려 `user:user_1`만 반환한다면 두 값이 같아지고 테스트가 실패합니다.
차이를 만들어야 하는 필드가 하나라면 `notStrictEqual()`이 가장 간단한 신호가 됩니다.

### H3. 타입 차이를 느슨하게 넘기지 않는다

JavaScript에서는 문자열 `"1"`과 숫자 `1`이 느슨한 비교에서 같게 취급될 수 있습니다.
하지만 테스트에서는 이런 차이를 의도적으로 드러내야 할 때가 많습니다.
`node:assert/strict`의 `notStrictEqual()`은 타입 차이를 포함해 두 값이 같은 값인지 판단합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function readPageNumber(query) {
  return Number(query.page || 1);
}

test('page number is converted to a number', () => {
  const page = readPageNumber({ page: '2' });

  assert.notStrictEqual(page, '2');
  assert.strictEqual(page, 2);
});
```

첫 번째 assertion은 반환값이 입력 문자열 그대로 남지 않았는지 확인합니다.
두 번째 assertion은 최종 값이 숫자 `2`인지 고정합니다.
부정 assertion만 쓰면 "문자열은 아니어야 한다"까지만 설명되므로, 기대값이 분명한 경우에는 긍정 assertion을 함께 두는 것이 좋습니다.

## notEqual과 notStrictEqual의 차이

### H3. 느슨한 비교 대신 엄격한 비교를 기본으로 삼는다

`assert.notEqual()`은 느슨한 동등성 기준을 사용합니다.
반면 `assert.notStrictEqual()`은 엄격한 기준으로 값을 비교합니다.
테스트에서는 대부분 타입 차이까지 계약에 포함하는 편이 안전하므로, 새 코드를 작성할 때는 `notStrictEqual()`을 기본 선택지로 두는 것이 좋습니다.

```js
import assert from 'node:assert/strict';

assert.notStrictEqual(0, false);
assert.notStrictEqual('1', 1);
assert.notStrictEqual(null, undefined);
```

위 값들은 느슨한 비교에서는 헷갈릴 수 있지만, 엄격한 비교에서는 서로 다른 값입니다.
API 입력 정규화, 쿼리 파라미터 변환, 설정값 파싱처럼 타입이 중요한 코드에서는 이런 차이가 버그를 줄이는 단서가 됩니다.
테스트가 타입 변환을 문서화해야 한다면 느슨한 부정 비교보다 엄격한 부정 비교를 선택하세요.

### H3. NaN과 -0 같은 특수값도 의식한다

`assert.notStrictEqual()`은 `Object.is()` 기반 비교와 연결되므로 특수한 숫자 값도 신경 써야 합니다.
예를 들어 `NaN`은 `NaN`과 같은 값으로 취급되고, `0`과 `-0`은 구분됩니다.
이런 특수값이 도메인에 들어온다면 테스트 이름과 assertion을 더 구체적으로 작성하는 것이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function normalizeDelta(value) {
  if (Object.is(value, -0)) {
    return 0;
  }

  return value;
}

test('normalizeDelta does not keep negative zero', () => {
  const delta = normalizeDelta(-0);

  assert.notStrictEqual(delta, -0);
  assert.strictEqual(delta, 0);
});
```

이 테스트는 `-0`을 일반적인 `0`으로 정규화해야 한다는 요구사항을 보여 줍니다.
특수값은 읽는 사람이 놓치기 쉬우므로, 부정 assertion 뒤에 최종 기대값을 함께 두면 훨씬 이해하기 쉽습니다.
실무에서는 이런 테스트에 짧은 이름을 붙여 숫자 처리 의도를 분명하게 남기는 편이 좋습니다.

## 객체 참조를 검증하는 기준

### H3. 새 객체를 반환해야 할 때 참조 차이를 확인한다

`notStrictEqual()`은 객체의 내용이 아니라 참조를 비교합니다.
따라서 복사 함수, 정규화 함수, immutable update 함수가 원본 객체를 그대로 반환하지 않아야 하는지 확인할 때 사용할 수 있습니다.
단, 객체 내용까지 검증해야 한다면 `deepStrictEqual()`을 함께 사용해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function updateUserName(user, name) {
  return {
    ...user,
    name
  };
}

test('updateUserName returns a new user object', () => {
  const original = {
    id: 'user_1',
    name: 'Soo'
  };

  const updated = updateUserName(original, 'Soo Kim');

  assert.notStrictEqual(updated, original);
  assert.deepStrictEqual(updated, {
    id: 'user_1',
    name: 'Soo Kim'
  });
});
```

첫 번째 assertion은 새 객체를 반환해야 한다는 참조 계약을 검증합니다.
두 번째 assertion은 새 객체의 실제 값이 기대한 구조인지 확인합니다.
참조만 다르고 내용이 틀린 객체도 있을 수 있으므로, immutable 패턴을 테스트할 때는 두 검증을 같이 두는 편이 안전합니다.

### H3. 같은 인스턴스를 재사용해야 하는 경우에는 쓰지 않는다

반대로 싱글턴, 캐시된 설정 객체, 연결 풀처럼 같은 인스턴스를 재사용해야 하는 코드도 있습니다.
이런 경우에는 `notStrictEqual()`이 아니라 `strictEqual()`이 맞습니다.
테스트 API는 "무엇이 달라야 하는가"뿐 아니라 "무엇이 같아야 하는가"를 기준으로 선택해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

const configCache = new Map();

function getConfig(name) {
  if (!configCache.has(name)) {
    configCache.set(name, { name, loaded: true });
  }

  return configCache.get(name);
}

test('getConfig reuses cached config object', () => {
  const first = getConfig('api');
  const second = getConfig('api');

  assert.strictEqual(first, second);
});
```

이 테스트에서는 참조가 같아야 합니다.
같은 인스턴스를 재사용하는 것이 성능이나 상태 공유의 일부 계약이기 때문입니다.
참조 비교 테스트를 작성할 때는 복사가 필요한 코드인지, 재사용이 필요한 코드인지 먼저 구분해야 합니다.

## 실무에서 조심할 점

### H3. 부정 assertion만으로 요구사항을 끝내지 않는다

`notStrictEqual()`은 두 값이 다르기만 하면 통과합니다.
그래서 "이 값이 아니면 된다"는 테스트가 너무 넓어지면 실제 요구사항을 놓칠 수 있습니다.
최종 기대값이 있다면 `strictEqual()`이나 `deepStrictEqual()`로 한 번 더 좁혀 주세요.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function nextStatus(current) {
  if (current === 'draft') {
    return 'review';
  }

  return 'published';
}

test('draft moves to review status', () => {
  const status = nextStatus('draft');

  assert.notStrictEqual(status, 'draft');
  assert.strictEqual(status, 'review');
});
```

첫 번째 assertion은 상태가 그대로 남지 않아야 한다는 회귀 방지 장치입니다.
두 번째 assertion은 정확히 어떤 상태로 바뀌어야 하는지 문서화합니다.
부정 검증은 보조 장치로 쓰고, 핵심 요구사항은 가능한 한 긍정 검증으로 고정하는 것이 좋습니다.

### H3. 랜덤과 시간 테스트는 입력을 고정한다

토큰, 시간, nonce처럼 매번 달라지는 값을 테스트할 때는 실제 랜덤성에 기대지 않는 편이 좋습니다.
생성기를 주입하거나 입력 값을 고정하면 테스트가 재현 가능해집니다.
`notStrictEqual()`은 서로 다른 입력이 서로 다른 값을 만드는지 확인하는 데 집중할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function buildRequestId(prefix, sequence) {
  return `${prefix}-${sequence}`;
}

test('request ids differ by sequence', () => {
  const first = buildRequestId('req', 1);
  const second = buildRequestId('req', 2);

  assert.notStrictEqual(first, second);
  assert.strictEqual(first, 'req-1');
  assert.strictEqual(second, 'req-2');
});
```

이 테스트는 난수나 현재 시각을 직접 사용하지 않습니다.
대신 sequence를 고정된 입력으로 넣어 결과 차이를 재현합니다.
테스트가 항상 같은 방식으로 실패하고 통과해야 디버깅 비용을 줄일 수 있습니다.

## 마무리 체크리스트

`assert.notStrictEqual()`은 원시값이나 객체 참조가 같으면 안 되는 상황을 좁게 검증할 때 유용합니다.
환경별 캐시 키, 타입 변환 결과, 새 객체 반환, 상태 전이처럼 값 하나의 차이가 중요한 테스트에서 읽기 쉬운 문서를 만들어 줍니다.
다만 부정 assertion은 "다르면 통과"하는 도구이므로, 최종 기대값이 있다면 `assert.strictEqual()`이나 `assert.deepStrictEqual()`을 함께 쓰는 편이 안전합니다.

발행 전에는 다음 기준으로 점검하면 좋습니다.

- 두 값이 달라야 하는 이유가 테스트 이름과 본문에 드러나는가?
- 원시값 비교인지, 객체 참조 비교인지 구분했는가?
- 기대값이 분명한 경우 긍정 assertion으로 한 번 더 고정했는가?
- 랜덤, 시간, 토큰 예시는 재현 가능한 더미 입력으로 구성했는가?
- 내부링크가 관련된 Node.js assert 글로 자연스럽게 연결되는가?

---
layout: post
title: "Node.js assert.doesNotThrow 가이드: 예외가 없어야 하는 코드를 명확하게 테스트하는 법"
date: 2026-06-24 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-doesnotthrow-no-throw-test-guide
permalink: /development/blog/seo/2026/06/24/nodejs-assert-doesnotthrow-no-throw-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/24/nodejs-assert-doesnotthrow-no-throw-test-guide.html
  x_default: /development/blog/seo/2026/06/24/nodejs-assert-doesnotthrow-no-throw-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, doesNotThrow, test, exception, backend]
description: "Node.js assert.doesNotThrow()로 예외가 발생하지 않아야 하는 경로를 명확하게 검증하는 방법을 정리합니다. assert.throws와의 차이, 사용을 줄여야 하는 상황, 테스트 의도를 선명하게 남기는 패턴을 예제로 설명합니다."
---

테스트에서는 "실패해야 하는 입력"만큼 "정상 입력이 예외 없이 처리되는가"도 중요합니다.
특히 설정 파서, 검증 함수, 변환 함수, CLI 옵션 처리처럼 잘못된 입력에서는 예외를 던지고 올바른 입력에서는 조용히 통과해야 하는 코드가 많습니다.
이때 Node.js `node:assert/strict`의 `assert.doesNotThrow()`를 사용하면 정상 경로가 예외를 던지지 않는다는 의도를 테스트 이름과 코드에 분명하게 남길 수 있습니다.

[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 `assert.doesNotThrow(fn)`이 전달한 함수가 예외를 던지지 않는지 확인하는 assertion이라고 설명합니다.
다만 이 메서드는 모든 정상 테스트에 습관처럼 붙일 도구는 아닙니다.
테스트 함수 안에서 예외가 던져지면 어차피 테스트는 실패하기 때문입니다.
중요한 것은 언제 `doesNotThrow()`가 의도를 더 선명하게 만들고, 언제 불필요한 포장이 되는지 구분하는 것입니다.

동기 테스트의 기본 구조가 필요하다면 [Node.js built-in test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 참고하세요.
비동기 예외는 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)에서 다룹니다.
문자열 에러 메시지를 덜 깨지게 검증하려면 [Node.js assert.match 가이드](/development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html)도 함께 연결됩니다.

## assert.doesNotThrow가 필요한 상황

### H3. 정상 경로가 테스트의 핵심 의도일 때 쓴다

일반적인 테스트에서는 함수 호출이 예외를 던지면 테스트 러너가 실패로 처리합니다.
그래서 단순히 "이 함수가 잘 실행된다"는 수준이라면 `assert.doesNotThrow()` 없이 결과 값을 검증하는 편이 더 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function parsePort(value) {
  const port = Number(value);

  if (!Number.isInteger(port) || port < 1 || port > 65535) {
    throw new RangeError('port must be between 1 and 65535');
  }

  return port;
}

test('parsePort returns numeric port for valid input', () => {
  assert.equal(parsePort('3000'), 3000);
});
```

이 테스트는 `parsePort('3000')`이 예외를 던지면 실패합니다.
반환 값까지 검증하므로 `doesNotThrow()`를 추가할 필요가 없습니다.
정상 경로의 결과가 명확하다면 결과 assertion이 가장 좋은 문서 역할을 합니다.

### H3. 반환 값보다 예외 여부가 계약일 때 유용하다

반대로 반환 값이 중요하지 않고 "이 입력은 예외를 던지면 안 된다"는 계약 자체가 핵심일 때가 있습니다.
예를 들어 설정 객체 검증 함수가 성공 시 아무것도 반환하지 않는다면, 예외 여부가 테스트의 주된 관심사입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function validateConfig(config) {
  if (!config.serviceName) {
    throw new Error('serviceName is required');
  }

  if (!['debug', 'info', 'warn', 'error'].includes(config.logLevel)) {
    throw new Error('logLevel is invalid');
  }
}

test('validateConfig accepts minimal production config', () => {
  assert.doesNotThrow(() => {
    validateConfig({
      serviceName: 'billing-api',
      logLevel: 'info'
    });
  });
});
```

이 테스트의 핵심은 `validateConfig()`가 값을 반환하는지가 아니라, 운영에 필요한 최소 설정을 정상 입력으로 인정하는지입니다.
이런 경우에는 `assert.doesNotThrow()`가 테스트 의도를 자연스럽게 드러냅니다.

## assert.throws와 함께 설계하기

### H3. 성공 케이스와 실패 케이스를 짝으로 둔다

검증 함수 테스트는 성공 케이스만 있으면 느슨하고, 실패 케이스만 있으면 사용 가능한 입력 범위가 흐려질 수 있습니다.
`doesNotThrow()`와 `throws()`를 함께 두면 허용되는 입력과 거부되는 입력을 나란히 설명할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function validateRetryPolicy(policy) {
  if (!Number.isInteger(policy.retries) || policy.retries < 0) {
    throw new RangeError('retries must be a non-negative integer');
  }

  if (policy.timeoutMs < 100) {
    throw new RangeError('timeoutMs must be at least 100');
  }
}

test('validateRetryPolicy accepts safe retry policy', () => {
  assert.doesNotThrow(() => {
    validateRetryPolicy({
      retries: 3,
      timeoutMs: 1000
    });
  });
});

test('validateRetryPolicy rejects negative retries', () => {
  assert.throws(
    () => validateRetryPolicy({ retries: -1, timeoutMs: 1000 }),
    /retries/
  );
});
```

이렇게 두면 테스트 파일만 읽어도 경계가 보입니다.
`retries: 3`은 허용되고, `retries: -1`은 거부됩니다.
정상 입력과 비정상 입력의 차이가 코드 리뷰에서 더 쉽게 드러납니다.

### H3. 에러 메시지 전체를 고정하지 않는다

실패 케이스에서는 에러 메시지 전체를 정확히 비교하기보다 핵심 신호를 검증하는 편이 유지보수에 유리합니다.
위 예제처럼 `/retries/` 정도를 확인하면 문구를 다듬어도 테스트가 쉽게 깨지지 않습니다.

문자열 패턴 검증 기준은 [Node.js assert.match 가이드](/development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html)와도 연결됩니다.
에러 메시지가 공개 API 계약이라면 더 엄격하게 검증해야 하지만, 내부 검증 함수라면 필드명과 에러 종류 중심으로 확인하는 편이 실용적입니다.

## 사용을 줄여야 하는 패턴

### H3. 결과 assertion을 대체하지 않는다

`assert.doesNotThrow()`는 "예외가 없었다"만 확인합니다.
함수가 잘못된 값을 반환해도 예외만 없으면 통과합니다.
그래서 결과가 중요한 함수에서는 반드시 반환 값이나 상태 변화를 따로 검증해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function normalizeEmail(email) {
  return email.trim().toLowerCase();
}

test('normalizeEmail trims and lowercases email', () => {
  const result = normalizeEmail(' User@Example.COM ');

  assert.equal(result, 'user@example.com');
});
```

이 테스트에 `assert.doesNotThrow()`를 추가해도 얻는 정보는 거의 없습니다.
`assert.equal()`이 이미 예외 여부와 결과 값을 함께 검증합니다.
예외 여부만 보는 테스트는 코드가 실제로 해야 할 일을 놓치기 쉽습니다.

### H3. 비동기 함수에는 assert.doesNotReject를 쓴다

`assert.doesNotThrow()`는 동기 함수에서 발생하는 예외를 다룹니다.
Promise가 거부되는지 확인해야 한다면 `assert.doesNotReject()`를 사용해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function saveDraft(draft) {
  if (!draft.title) {
    throw new Error('title is required');
  }

  return { id: 'draft_1', ...draft };
}

test('saveDraft accepts titled draft', async () => {
  await assert.doesNotReject(async () => {
    await saveDraft({ title: 'Release notes' });
  });
});
```

다만 비동기 함수도 반환 값이 중요하다면 결과를 검증하는 편이 우선입니다.

```js
test('saveDraft returns draft id', async () => {
  const result = await saveDraft({ title: 'Release notes' });

  assert.equal(result.id, 'draft_1');
  assert.equal(result.title, 'Release notes');
});
```

이 테스트는 Promise rejection이 발생하면 자동으로 실패합니다.
동기 테스트와 마찬가지로, 결과 assertion이 충분하면 별도의 `doesNotReject()`가 필요하지 않습니다.

## 실무 적용 체크리스트

### H3. 의도가 예외 여부인지 먼저 확인한다

`assert.doesNotThrow()`를 쓰기 전에 테스트가 정말 예외 여부를 검증하려는지 확인해야 합니다.
아래 질문에 답하면 선택이 쉬워집니다.

- 성공 시 반환 값이 중요한가?
- 성공 시 상태 변화나 출력이 중요한가?
- 성공 계약이 "아무 예외도 던지지 않음"으로 충분한가?
- 실패 케이스에서 어떤 에러를 던져야 하는지도 함께 검증했는가?

반환 값이나 상태 변화가 중요하다면 그 값을 직접 검증하세요.
예외 없음이 계약이라면 `assert.doesNotThrow()`를 사용해도 좋습니다.
검증 함수처럼 성공 시 조용히 통과하는 API라면 특히 잘 맞습니다.

### H3. 실패 메시지는 테스트 이름으로 보완한다

`assert.doesNotThrow()`는 실패했을 때 실제로 던져진 에러를 보여 줍니다.
그래도 테스트 이름이 모호하면 무엇이 정상 입력인지 알기 어렵습니다.
테스트 이름에는 "무엇을 허용해야 하는가"를 직접 적는 편이 좋습니다.

```js
test('validateConfig accepts readonly replica config', () => {
  assert.doesNotThrow(() => {
    validateConfig({
      serviceName: 'reporting-worker',
      logLevel: 'warn',
      databaseRole: 'readonly'
    });
  });
});
```

이 이름은 테스트가 단순히 "에러가 안 난다"를 확인하는 것이 아니라, 읽기 전용 replica 설정을 정상 설정으로 인정해야 한다는 정책을 설명합니다.
테스트가 문서처럼 읽히려면 assertion보다 테스트 이름이 더 중요할 때가 많습니다.

## FAQ

### H3. assert.doesNotThrow를 모든 정상 테스트에 넣어야 하나요?

아니요.
테스트 안에서 예외가 발생하면 테스트는 기본적으로 실패합니다.
반환 값, 상태 변화, 호출 횟수처럼 더 구체적인 결과를 검증할 수 있다면 그 assertion을 우선하세요.
`assert.doesNotThrow()`는 예외가 없어야 한다는 사실 자체가 테스트의 핵심일 때 쓰는 편이 좋습니다.

### H3. 비동기 함수에도 assert.doesNotThrow를 쓸 수 있나요?

Promise rejection 검증에는 적합하지 않습니다.
비동기 함수에는 `await assert.doesNotReject(...)`를 사용하거나, 더 나은 경우에는 `await`로 결과를 받은 뒤 반환 값을 직접 검증하세요.
비동기 실패 경로는 `assert.rejects()`로 별도 테스트를 두는 것이 명확합니다.

### H3. assert.doesNotThrow와 assert.ok 중 무엇이 더 좋나요?

두 메서드는 목적이 다릅니다.
`assert.ok()`는 값이 truthy인지 검증하고, `assert.doesNotThrow()`는 함수 호출 중 예외가 발생하지 않는지 검증합니다.
값의 의미를 확인해야 한다면 `assert.ok()`보다 `assert.equal()`, `assert.deepStrictEqual()`, `assert.match()`처럼 더 구체적인 assertion을 먼저 고려하세요.

## 마무리

`assert.doesNotThrow()`는 정상 경로를 테스트하는 만능 도구가 아니라, "이 입력은 예외 없이 받아들여져야 한다"는 계약을 드러내는 도구입니다.
결과 값을 검증할 수 있는 함수라면 결과 assertion을 우선하고, 성공 시 반환 값이 없거나 중요하지 않은 검증 함수에서는 `doesNotThrow()`로 의도를 명확하게 남기면 됩니다.

테스트의 목표는 단순히 통과하는 코드가 아니라, 코드가 지켜야 할 계약을 읽기 쉬운 형태로 남기는 것입니다.
정상 경로와 실패 경로를 함께 설계하면 테스트는 더 안정적이고, 리팩터링 때 깨져야 할 때만 깨지는 신호가 됩니다.

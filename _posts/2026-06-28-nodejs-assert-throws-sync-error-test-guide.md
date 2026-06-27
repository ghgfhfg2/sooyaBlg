---
layout: post
title: "Node.js assert.throws 가이드: 동기 함수 예외를 정확하게 검증하는 테스트 패턴"
date: 2026-06-28 08:00:00 +0900
lang: ko
translation_key: nodejs-assert-throws-sync-error-test-guide
permalink: /development/blog/seo/2026/06/28/nodejs-assert-throws-sync-error-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/28/nodejs-assert-throws-sync-error-test-guide.html
  x_default: /development/blog/seo/2026/06/28/nodejs-assert-throws-sync-error-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, throws, test, assertion, error, backend]
description: "Node.js assert.throws()로 동기 함수가 의도한 예외를 던지는지 검증하는 방법을 정리합니다. 에러 타입, 메시지 패턴, 커스텀 에러 속성, doesNotThrow와의 구분을 예제로 설명합니다."
---

테스트는 성공 경로만 확인해서는 충분하지 않습니다.
잘못된 입력을 거부해야 하는 파서, 필수 설정이 빠졌을 때 즉시 실패해야 하는 초기화 코드, 지원하지 않는 상태 전이를 막아야 하는 도메인 함수는 실패 경로도 계약입니다.
이런 동기 함수의 예외 계약을 표현할 때 Node.js `node:assert/strict`의 `assert.throws()`를 사용할 수 있습니다.

[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 `assert.throws(fn[, error][, message])`가 함수 실행 중 예외가 발생하는지 검증한다고 설명합니다.
핵심은 "예외가 나기만 하면 된다"에서 끝내지 않는 것입니다.
실무 테스트에서는 에러 타입, 메시지 패턴, 커스텀 속성 중 무엇이 외부 계약인지 골라 좁게 검증해야 합니다.

예외가 발생하지 않아야 하는 경로는 [Node.js assert.doesNotThrow 가이드](/development/blog/seo/2026/06/24/nodejs-assert-doesnotthrow-no-throw-test-guide.html)를 참고하세요.
비동기 함수의 실패 검증은 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)가 더 적합합니다.
명시적인 실패 분기를 표시해야 한다면 [Node.js assert.fail 가이드](/development/blog/seo/2026/06/26/nodejs-assert-fail-explicit-test-failure-guide.html)와 함께 보면 좋습니다.

## assert.throws가 필요한 이유

### H3. 잘못된 입력을 즉시 거부하는지 확인한다

검증 함수나 파서는 잘못된 입력을 조용히 보정하면 안 되는 경우가 있습니다.
특히 설정, 권한, 금액, 식별자처럼 값이 틀리면 뒤쪽 시스템까지 영향을 주는 입력은 초기에 실패시키는 편이 안전합니다.
`assert.throws()`는 이런 실패 경로를 테스트 이름과 함께 문서화합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function parsePort(value) {
  const port = Number(value);

  if (!Number.isInteger(port) || port < 1 || port > 65535) {
    throw new RangeError('port must be an integer between 1 and 65535');
  }

  return port;
}

test('parsePort rejects an out-of-range port', () => {
  assert.throws(
    () => parsePort('70000'),
    RangeError
  );
});
```

이 테스트는 범위를 벗어난 포트가 통과하면 안 된다는 규칙을 보여 줍니다.
반환값을 비교하는 테스트와 달리, 실패해야 하는 호출을 콜백 안에 넣어야 합니다.
`assert.throws(parsePort('70000'), RangeError)`처럼 함수 호출 결과를 넘기면 assertion이 실행되기 전에 예외가 먼저 터지므로 올바른 패턴이 아닙니다.

### H3. 에러 타입으로 실패 종류를 구분한다

예외가 발생한다는 사실만으로는 테스트가 너무 넓을 수 있습니다.
타입 에러인지, 범위 에러인지, 도메인 전용 에러인지 구분해야 호출자가 후속 처리를 안정적으로 할 수 있습니다.
이때 두 번째 인자로 에러 생성자를 넘기면 의도한 타입의 예외인지 확인할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

class MissingConfigError extends Error {
  constructor(name) {
    super(`missing required config: ${name}`);
    this.name = 'MissingConfigError';
    this.code = 'ERR_MISSING_CONFIG';
  }
}

function readRequiredConfig(env, name) {
  if (!env[name]) {
    throw new MissingConfigError(name);
  }

  return env[name];
}

test('readRequiredConfig throws MissingConfigError for absent values', () => {
  assert.throws(
    () => readRequiredConfig({}, 'DATABASE_URL'),
    MissingConfigError
  );
});
```

커스텀 에러 타입은 호출자가 `instanceof`나 에러 코드로 분기할 수 있게 해 줍니다.
테스트에서 타입을 고정하면 나중에 구현이 일반 `Error`로 바뀌는 회귀를 잡을 수 있습니다.
운영 코드가 에러 종류에 따라 복구 전략을 달리한다면 타입 검증은 중요한 계약입니다.

## 메시지와 속성을 검증하는 방법

### H3. 메시지는 정규식으로 핵심만 확인한다

에러 메시지는 사용자나 개발자가 문제를 이해하는 데 도움을 줍니다.
하지만 문장 전체를 지나치게 고정하면 조사나 문구가 조금 바뀌어도 테스트가 깨질 수 있습니다.
외부 계약이 메시지의 핵심 키워드라면 정규식으로 필요한 부분만 검증하는 것이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function parseJsonObject(input) {
  const parsed = JSON.parse(input);

  if (parsed === null || Array.isArray(parsed) || typeof parsed !== 'object') {
    throw new TypeError('JSON payload must be an object');
  }

  return parsed;
}

test('parseJsonObject rejects non-object JSON payloads', () => {
  assert.throws(
    () => parseJsonObject('[1, 2, 3]'),
    /must be an object/
  );
});
```

이 테스트는 에러 메시지에 객체여야 한다는 설명이 포함되는지 확인합니다.
정규식은 문장 전체보다 덜 취약하지만, 너무 느슨하면 엉뚱한 메시지도 통과할 수 있습니다.
도메인에서 중요한 단어를 기준으로 짧고 구체적인 패턴을 잡는 편이 좋습니다.

### H3. 에러 객체의 속성까지 확인한다

API나 라이브러리 코드에서는 에러 메시지보다 `code`, `statusCode`, `details` 같은 구조화된 속성이 더 안정적인 계약일 수 있습니다.
`assert.throws()`의 두 번째 인자로 검증 함수를 넘기면 던져진 에러 객체를 직접 검사할 수 있습니다.
검증 함수는 조건이 맞으면 `true`를 반환해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

class ValidationError extends Error {
  constructor(field, reason) {
    super(`${field}: ${reason}`);
    this.name = 'ValidationError';
    this.code = 'ERR_VALIDATION';
    this.field = field;
  }
}

function createUser(input) {
  if (!input.email.includes('@')) {
    throw new ValidationError('email', 'invalid format');
  }

  return {
    id: 'user_1',
    email: input.email
  };
}

test('createUser exposes validation error metadata', () => {
  assert.throws(
    () => createUser({ email: 'invalid-email' }),
    (error) => {
      assert.equal(error.code, 'ERR_VALIDATION');
      assert.equal(error.field, 'email');
      assert.match(error.message, /invalid format/);
      return true;
    }
  );
});
```

이 패턴은 에러 객체의 여러 속성을 한 번에 확인할 때 유용합니다.
검증 함수 안에서도 `assert.equal()`이나 `assert.match()`를 사용하면 실패 이유가 더 분명해집니다.
단, 검증 함수가 너무 길어지면 테스트가 읽기 어려워지므로 핵심 계약만 남기는 것이 좋습니다.

## doesNotThrow와 rejects와의 차이

### H3. 성공 경로에는 doesNotThrow를 과하게 쓰지 않는다

`assert.doesNotThrow()`는 함수가 예외를 던지지 않아야 한다는 의도를 표현합니다.
하지만 대부분의 성공 경로 테스트는 반환값을 검증하는 것만으로 충분합니다.
반환값 assertion이 실행되려면 함수가 이미 예외 없이 끝나야 하기 때문입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function normalizeEmail(email) {
  if (!email.includes('@')) {
    throw new TypeError('email must include @');
  }

  return email.trim().toLowerCase();
}

test('normalizeEmail lowercases a valid email', () => {
  assert.equal(normalizeEmail(' SOO@example.com '), 'soo@example.com');
});
```

이 테스트에는 `doesNotThrow()`가 없어도 됩니다.
함수가 예외를 던지면 `assert.equal()`까지 도달하지 못하고 테스트가 실패합니다.
성공 경로에서는 반환값이나 상태 변화를 직접 검증하고, 실패 경로에서 `assert.throws()`를 사용하는 식으로 역할을 나누면 테스트가 간결해집니다.

### H3. Promise 실패에는 rejects를 사용한다

`assert.throws()`는 동기 함수가 바로 던지는 예외를 검증합니다.
`async` 함수나 Promise를 반환하는 함수의 실패는 동기 예외가 아니라 rejected Promise입니다.
이 경우에는 `assert.rejects()`를 사용해야 테스트가 의도대로 동작합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function loadUser(id) {
  if (!id) {
    throw new TypeError('id is required');
  }

  return {
    id
  };
}

test('loadUser rejects when id is missing', async () => {
  await assert.rejects(
    () => loadUser(''),
    TypeError
  );
});
```

`loadUser()`는 내부에서 `throw`를 사용하지만 `async` 함수이므로 결과는 rejected Promise입니다.
이런 함수에 `assert.throws()`를 쓰면 Promise 실패를 제대로 기다리지 못합니다.
테스트 대상이 동기 함수인지 비동기 함수인지 먼저 구분하는 것이 예외 테스트의 출발점입니다.

## 실무 체크리스트

### H3. 예외 계약을 너무 넓게 잡지 않는다

`assert.throws(() => fn())`처럼 에러 조건을 생략하면 어떤 예외든 통과합니다.
이 방식은 "무언가 실패한다"는 수준만 확인하므로 회귀를 놓치기 쉽습니다.
가능하면 에러 타입, 메시지 패턴, 에러 코드 중 하나 이상을 함께 검증하세요.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function transitionOrder(status, nextStatus) {
  if (status === 'paid' && nextStatus === 'draft') {
    throw new Error('paid order cannot return to draft');
  }

  return nextStatus;
}

test('transitionOrder rejects invalid backward transition', () => {
  assert.throws(
    () => transitionOrder('paid', 'draft'),
    /cannot return to draft/
  );
});
```

이 테스트는 실패한다는 사실보다 "결제 완료 주문이 초안으로 돌아갈 수 없다"는 도메인 규칙을 보여 줍니다.
예외 테스트는 구현의 부작용이 아니라 비즈니스 규칙을 설명할 때 가장 가치가 큽니다.
테스트 이름, 입력값, 에러 검증 조건이 같은 규칙을 가리키도록 정리하세요.

### H3. 부작용이 있는 함수는 실행 전후 상태를 함께 본다

예외가 발생했을 때 상태가 일부만 바뀌면 더 큰 문제가 됩니다.
검증 실패 후 배열에 항목이 추가되거나, 트랜잭션 시작 후 정리가 누락되는 식의 버그가 생길 수 있습니다.
이런 함수는 `assert.throws()`와 상태 검증을 함께 두는 것이 안전합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function addUniqueItem(items, item) {
  if (items.includes(item)) {
    throw new Error('item already exists');
  }

  items.push(item);
}

test('addUniqueItem does not mutate the list for duplicates', () => {
  const items = ['nodejs'];

  assert.throws(
    () => addUniqueItem(items, 'nodejs'),
    /already exists/
  );

  assert.deepStrictEqual(items, ['nodejs']);
});
```

이 테스트는 예외 발생과 상태 보존을 동시에 검증합니다.
실패 경로가 중요한 코드일수록 "예외를 던졌는가" 다음에 "남긴 상태가 안전한가"를 확인해야 합니다.
특히 큐, 캐시, 결제, 권한 변경처럼 부작용이 있는 로직에서는 이 후속 검증이 회귀를 줄입니다.

## 정리

`assert.throws()`는 동기 함수의 실패 경로를 테스트 계약으로 만드는 API입니다.
에러가 발생한다는 사실만 확인하기보다 타입, 메시지 패턴, 구조화된 속성 중 중요한 기준을 함께 검증하면 테스트가 더 실무적으로 변합니다.
성공 경로는 반환값을 직접 확인하고, 비동기 실패는 `assert.rejects()`로 분리하면 예외 테스트의 의도가 선명해집니다.

테스트가 실패 경로를 잘 설명하면 코드를 바꾸는 사람도 어떤 입력을 거부해야 하는지 빠르게 이해할 수 있습니다.
`assert.throws()`를 사용할 때는 "어떤 예외든 괜찮다"가 아니라 "이 규칙 때문에 이 실패가 나야 한다"는 문장으로 테스트를 설계하세요.

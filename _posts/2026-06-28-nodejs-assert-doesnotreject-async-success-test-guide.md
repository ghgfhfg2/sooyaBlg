---
layout: post
title: "Node.js assert.doesNotReject 가이드: 비동기 함수가 실패하지 않아야 하는 경로를 테스트하는 법"
date: 2026-06-28 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-doesnotreject-async-success-test-guide
permalink: /development/blog/seo/2026/06/28/nodejs-assert-doesnotreject-async-success-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/28/nodejs-assert-doesnotreject-async-success-test-guide.html
  x_default: /development/blog/seo/2026/06/28/nodejs-assert-doesnotreject-async-success-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, doesNotReject, test, promise, async, backend]
description: "Node.js assert.doesNotReject()로 Promise 기반 코드가 reject되지 않아야 하는 경로를 검증하는 방법을 정리합니다. await 패턴, rejects와의 차이, 남용을 피하는 기준을 예제로 설명합니다."
---

비동기 테스트에서 실패 경로는 `assert.rejects()`로 명확하게 검증할 수 있습니다.
반대로 "이 작업은 reject되면 안 된다"는 사실 자체가 테스트의 핵심일 때는 Node.js `node:assert/strict`의 `assert.doesNotReject()`를 사용할 수 있습니다.
다만 이 assertion은 모든 성공 경로에 습관적으로 붙이는 도구가 아닙니다.

[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 `assert.doesNotReject(asyncFn[, error][, message])`가 전달된 Promise 또는 async 함수가 완료될 때까지 기다린다고 설명합니다.
동시에 rejection을 다시 rejection으로 바꾸는 것에는 실익이 적을 수 있으므로, 필요한 코드 경로에 명확한 메시지를 남기는 편이 낫다는 주의도 함께 둡니다.
따라서 실무에서는 "값을 검증하는 테스트"와 "reject되지 않아야 한다는 계약을 드러내는 테스트"를 구분해야 합니다.

비동기 실패 경로는 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)를 참고하세요.
동기 함수의 예외 없음 검증은 [Node.js assert.doesNotThrow 가이드](/development/blog/seo/2026/06/24/nodejs-assert-doesnotthrow-no-throw-test-guide.html)와 연결됩니다.
예외가 반드시 발생해야 하는 동기 경로는 [Node.js assert.throws 가이드](/development/blog/seo/2026/06/28/nodejs-assert-throws-sync-error-test-guide.html)에서 다룹니다.

## assert.doesNotReject가 하는 일

### H3. Promise가 fulfilled 되는지 기다린다

`assert.doesNotReject()`는 Promise 또는 Promise를 반환하는 함수를 받아 rejection이 발생하지 않는지 확인합니다.
테스트 함수 안에서는 반드시 `await`를 붙여야 합니다.
`await`가 빠지면 assertion이 끝나기 전에 테스트가 완료되어 비동기 실패를 놓칠 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function warmupCache(cache, key, value) {
  await cache.set(key, value);
  return cache.get(key);
}

test('warmupCache completes for writable cache', async () => {
  const cache = new Map();
  cache.set = async function set(key, value) {
    Map.prototype.set.call(this, key, value);
  };
  cache.get = async function get(key) {
    return Map.prototype.get.call(this, key);
  };

  await assert.doesNotReject(async () => {
    await warmupCache(cache, 'feature:home', 'enabled');
  });
});
```

이 테스트는 캐시 준비 작업이 정상적인 저장소에서는 reject되지 않아야 한다는 의도를 보여 줍니다.
반환값이 중요한 계약이라면 `doesNotReject()`만으로 끝내지 말고 반환값도 별도로 검증해야 합니다.
예외 없음은 성공의 일부일 뿐, 결과가 맞다는 뜻은 아니기 때문입니다.

### H3. Promise 자체와 async 함수를 모두 받을 수 있다

`assert.doesNotReject()`에는 이미 만들어진 Promise를 넘길 수도 있고, Promise를 반환하는 함수를 넘길 수도 있습니다.
대부분의 테스트에서는 함수를 넘기는 쪽이 더 읽기 좋습니다.
테스트 대상 호출과 assertion 의도가 한 곳에 묶이기 때문입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function publishAuditEvent(event) {
  if (!event.type) {
    throw new TypeError('event.type is required');
  }

  return {
    accepted: true,
    id: `audit:${event.type}`
  };
}

test('publishAuditEvent accepts a valid audit event', async () => {
  await assert.doesNotReject(() => publishAuditEvent({
    type: 'user.login',
    actorId: 'user_1'
  }));
});
```

여기서 `() => publishAuditEvent(...)`처럼 함수를 넘기면 assertion이 호출 시점을 관리합니다.
`await assert.doesNotReject(publishAuditEvent(...))`도 가능하지만, 복잡한 입력을 다룰수록 함수 형태가 실패 위치를 읽기 쉽습니다.
어느 쪽이든 테스트 함수가 `async`이고 assertion 앞에 `await`가 있어야 합니다.

## 언제 쓰고 언제 줄일까

### H3. 결과 검증이 있으면 보통 중복이다

대부분의 성공 경로 테스트는 값을 직접 검증하는 것만으로 충분합니다.
Promise가 reject되면 결과 assertion까지 도달하지 못하고 테스트가 실패합니다.
그래서 단순한 성공 테스트에 `doesNotReject()`를 덧붙이면 같은 정보를 두 번 말하는 코드가 되기 쉽습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function loadProfile(id) {
  return {
    id,
    status: 'active'
  };
}

test('loadProfile returns an active profile', async () => {
  const profile = await loadProfile('user_1');

  assert.deepStrictEqual(profile, {
    id: 'user_1',
    status: 'active'
  });
});
```

이 테스트에는 `assert.doesNotReject()`가 없어도 됩니다.
`loadProfile()`이 reject되면 `profile`을 받을 수 없고 테스트는 실패합니다.
반환값이나 상태 변화가 핵심이라면 그 결과를 직접 검증하는 편이 더 선명합니다.

### H3. 성공 자체가 계약일 때만 명시한다

`doesNotReject()`가 어울리는 경우는 성공 결과보다 "이 입력은 실패로 취급하지 않는다"는 정책을 드러내고 싶을 때입니다.
예를 들어 중복 이벤트를 무시해야 하는 소비자, 이미 적용된 마이그레이션을 다시 실행해도 안전해야 하는 작업, 선택 기능이 꺼져 있을 때 조용히 통과해야 하는 코드가 있습니다.
이런 테스트에서는 rejection이 없어야 한다는 사실이 중요한 계약입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function applyMigration(state, name) {
  if (state.applied.has(name)) {
    return {
      skipped: true
    };
  }

  state.applied.add(name);
  return {
    skipped: false
  };
}

test('applyMigration does not reject for already applied migration', async () => {
  const state = {
    applied: new Set(['2026_06_add_index'])
  };

  await assert.doesNotReject(() => applyMigration(state, '2026_06_add_index'));
});
```

이 테스트의 핵심은 "이미 적용됨" 상태가 장애가 아니라는 점입니다.
반환 객체까지 계약이라면 `skipped: true`도 함께 검증할 수 있습니다.
하지만 장애로 처리하지 않는 정책을 문서화하려는 목적이라면 `doesNotReject()`가 읽기 쉬운 신호가 됩니다.

## rejects와 함께 설계하기

### H3. 실패해야 하는 입력은 rejects로 고정한다

정상 입력이 reject되지 않아야 한다면, 비정상 입력은 어떻게 실패해야 하는지도 함께 테스트하는 것이 좋습니다.
성공 경로만 있으면 검증 로직이 사라져도 테스트가 통과할 수 있습니다.
반대 경로를 `assert.rejects()`로 고정하면 정책 경계가 더 분명해집니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function reserveInventory({ sku, quantity }) {
  if (!sku) {
    throw new TypeError('sku is required');
  }

  if (quantity < 1) {
    throw new RangeError('quantity must be greater than zero');
  }

  return {
    sku,
    quantity,
    reserved: true
  };
}

test('reserveInventory accepts a positive quantity', async () => {
  await assert.doesNotReject(() => reserveInventory({
    sku: 'book-1',
    quantity: 2
  }));
});

test('reserveInventory rejects an invalid quantity', async () => {
  await assert.rejects(
    () => reserveInventory({
      sku: 'book-1',
      quantity: 0
    }),
    RangeError
  );
});
```

이렇게 성공과 실패를 나누면 입력 검증의 경계가 드러납니다.
첫 번째 테스트는 유효한 수량이 장애가 아님을 보여 주고, 두 번째 테스트는 잘못된 수량이 반드시 실패해야 한다는 계약을 확인합니다.
비동기 API는 성공과 실패 모두 `await` 기반으로 작성해야 테스트 러너가 정확히 기다릴 수 있습니다.

### H3. 에러 타입 필터는 신중하게 사용한다

`doesNotReject()`의 두 번째 인자로 에러 타입이나 정규식, 검증 함수를 넘길 수 있습니다.
이 인자는 특정 에러와 매칭되는 rejection만 assertion 실패로 다루는 방식입니다.
하지만 대부분의 성공 경로 테스트에서는 어떤 rejection이든 실패로 보는 편이 더 자연스럽습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

class RetryableNetworkError extends Error {
  constructor(message) {
    super(message);
    this.name = 'RetryableNetworkError';
  }
}

async function syncOptionalWidget(client) {
  try {
    await client.sync();
  } catch (error) {
    if (error instanceof RetryableNetworkError) {
      return {
        deferred: true
      };
    }

    throw error;
  }
}

test('syncOptionalWidget handles retryable network failures internally', async () => {
  await assert.doesNotReject(() => syncOptionalWidget({
    async sync() {
      throw new RetryableNetworkError('temporary timeout');
    }
  }));
});
```

이 테스트는 재시도 가능한 네트워크 실패를 함수 내부에서 흡수해야 한다는 정책을 확인합니다.
여기서는 특정 에러 필터를 `doesNotReject()`에 넘기지 않았습니다.
외부로 어떤 rejection이든 새어 나오면 테스트가 실패해야 하기 때문입니다.

## 실무 체크리스트

### H3. await 누락을 가장 먼저 의심한다

`doesNotReject()`는 Promise를 반환합니다.
테스트가 가끔 통과하거나 비동기 실패가 늦게 출력된다면 assertion 앞에 `await`가 빠졌는지 먼저 확인하세요.
Node.js test runner에서는 테스트 콜백을 `async`로 만들고 내부 Promise를 모두 기다리는 습관이 중요합니다.

### H3. 성공 결과가 중요하면 값을 검증한다

reject되지 않았다는 사실은 "작업이 성공했다"의 최소 조건입니다.
생성된 객체, 저장된 상태, 호출된 의존성, 반환된 플래그가 중요하다면 별도 assertion을 추가해야 합니다.
특히 API 응답 테스트에서는 `doesNotReject()`보다 응답 구조와 상태 코드를 검증하는 쪽이 더 실용적입니다.

### H3. 실패 메시지는 테스트 이름에 남긴다

`doesNotReject()` 자체는 실패한 rejection을 다시 보여 주기 때문에 별도 메시지가 항상 필요하지는 않습니다.
대신 테스트 이름에 "무엇이 reject되면 안 되는지"를 구체적으로 적으면 유지보수자가 의도를 빠르게 이해할 수 있습니다.
예를 들어 `does not reject for duplicate webhook delivery`처럼 정책을 문장으로 남기는 방식이 좋습니다.

## FAQ

### H3. assert.doesNotReject는 모든 async 성공 테스트에 써야 하나요?

아닙니다.
반환값을 직접 검증하는 테스트라면 Promise가 reject될 때 자연스럽게 실패합니다.
`doesNotReject()`는 reject되지 않아야 한다는 사실 자체가 핵심 계약일 때만 쓰는 편이 좋습니다.

### H3. assert.doesNotReject와 assert.rejects는 무엇이 다른가요?

`assert.rejects()`는 Promise가 실패해야 하는 경로를 검증합니다.
`assert.doesNotReject()`는 Promise가 실패하지 않아야 하는 경로를 검증합니다.
입력 검증, 권한 오류, 네트워크 실패처럼 실패가 계약인 경우에는 `rejects`를 우선 사용하세요.

### H3. async 함수가 아니라 일반 값을 반환하면 어떻게 되나요?

`doesNotReject()`는 Promise 또는 Promise를 반환하는 함수를 기대합니다.
함수가 Promise를 반환하지 않으면 올바른 테스트 대상이 아닙니다.
동기 함수의 예외 없음 검증은 `assert.doesNotThrow()`를 사용하거나, 더 좋게는 반환값을 직접 검증하세요.

## 마무리

`assert.doesNotReject()`는 비동기 성공 경로를 표현할 수 있지만, 남용하면 테스트가 장황해집니다.
결과가 중요한 테스트에서는 반환값을 직접 확인하고, 실패하지 않아야 한다는 정책 자체가 중요한 경로에서만 명시적으로 사용하세요.
성공 경로의 `doesNotReject()`와 실패 경로의 `rejects()`를 함께 설계하면 Promise 기반 API의 계약이 훨씬 읽기 쉬워집니다.

## 관련 글

- [Node.js assert.rejects 가이드: 비동기 에러 테스트를 정확하게 작성하는 법](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)
- [Node.js assert.doesNotThrow 가이드: 예외가 없어야 하는 코드를 명확하게 테스트하는 법](/development/blog/seo/2026/06/24/nodejs-assert-doesnotthrow-no-throw-test-guide.html)
- [Node.js assert.throws 가이드: 동기 함수 예외를 정확하게 검증하는 테스트 패턴](/development/blog/seo/2026/06/28/nodejs-assert-throws-sync-error-test-guide.html)

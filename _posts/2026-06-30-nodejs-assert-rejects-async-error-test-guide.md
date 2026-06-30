---
layout: post
title: "Node.js assert.rejects 가이드: 비동기 실패를 정확하게 검증하는 테스트 패턴"
date: 2026-06-30 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-rejects-async-error-test-guide
permalink: /development/blog/seo/2026/06/30/nodejs-assert-rejects-async-error-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/30/nodejs-assert-rejects-async-error-test-guide.html
  x_default: /development/blog/seo/2026/06/30/nodejs-assert-rejects-async-error-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, rejects, async-test, promise, testing, javascript]
description: "Node.js assert.rejects()로 Promise 실패와 비동기 예외를 정확하게 검증하는 방법을 정리합니다. await 사용법, 에러 타입과 메시지 매칭, throws와의 차이, 테스트 안정화 패턴을 예제로 설명합니다."
---

비동기 코드는 성공 경로보다 실패 경로를 놓치기 쉽습니다.
Promise가 reject되어야 하는 함수가 resolve되거나, 잘못된 에러 타입을 던져도 테스트가 느슨하면 결함이 그대로 남습니다.
Node.js의 `assert.rejects()`는 비동기 함수가 의도한 방식으로 실패하는지 검증할 때 가장 기본이 되는 assertion입니다.

[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 `assert.rejects()`가 async function이나 Promise가 reject되는지 확인하고, 에러 조건을 함께 검증할 수 있다고 설명합니다.
핵심은 테스트 안에서 `await assert.rejects(...)` 형태로 기다리는 것입니다.
기다리지 않은 assertion은 테스트가 끝난 뒤 실패하거나, 아예 실패를 감지하지 못하는 불안정한 테스트가 될 수 있습니다.

비동기 성공 경로는 [Node.js assert.doesNotReject 가이드](/development/blog/seo/2026/06/25/nodejs-assert-doesnotreject-async-success-test-guide.html)를 참고하세요.
동기 함수의 예외 검증은 [Node.js assert.throws 가이드](/development/blog/seo/2026/06/28/nodejs-assert-throws-sync-error-test-guide.html)와 연결됩니다.
에러 메시지 안의 일부 문자열을 검증해야 한다면 [Node.js assert.match 가이드](/development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html)도 함께 보면 좋습니다.

## assert.rejects가 필요한 상황

### H3. Promise가 반드시 실패해야 하는 계약을 고정한다

로그인, 결제, 파일 처리, API 호출처럼 외부 조건에 따라 실패할 수 있는 함수는 실패 자체가 중요한 계약입니다.
예를 들어 필수 입력이 없으면 성공값을 반환하지 않고 명확한 에러로 거절되어야 합니다.
`assert.rejects()`를 사용하면 Promise가 reject되지 않았을 때 테스트가 실패합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function createUser(input) {
  if (!input.email) {
    throw new TypeError('email is required');
  }

  return {
    id: 'user-1',
    email: input.email
  };
}

test('createUser rejects when email is missing', async () => {
  await assert.rejects(
    () => createUser({}),
    TypeError
  );
});
```

이 테스트는 이메일이 없을 때 `TypeError`로 실패해야 한다는 사실을 고정합니다.
함수가 실수로 기본 이메일을 넣어 성공하거나, 다른 종류의 에러를 던지면 테스트가 깨집니다.
실패 경로를 테스트 이름에 직접 드러내면 나중에 코드를 읽는 사람도 의도를 빠르게 이해할 수 있습니다.

### H3. async 함수는 함수 형태로 넘기는 편이 읽기 쉽다

`assert.rejects()`에는 Promise 자체를 넘길 수도 있고, Promise를 반환하는 함수를 넘길 수도 있습니다.
실무에서는 함수 형태가 더 읽기 쉽고, 테스트가 실행되는 시점을 assertion 안으로 묶을 수 있습니다.
특히 여러 입력을 준비한 뒤 실패 호출만 분리해 보여 줄 때 유용합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function loadReport({ role }) {
  if (role !== 'admin') {
    throw new Error('admin role is required');
  }

  return { rows: [] };
}

test('loadReport rejects non-admin users', async () => {
  const viewer = { role: 'viewer' };

  await assert.rejects(
    async () => loadReport(viewer),
    /admin role is required/
  );
});
```

여기서는 비동기 호출을 `async () => loadReport(viewer)`로 감쌌습니다.
테스트 본문에서 준비 단계와 검증 단계를 분리할 수 있어 실패 원인이 더 선명합니다.
에러 메시지는 정규식으로 필요한 부분만 검증해 문구 전체에 과하게 묶이지 않도록 했습니다.

## 에러 타입과 메시지를 함께 검증하기

### H3. 타입만 보면 실패 이유가 넓게 잡힐 수 있다

에러 타입만 검증하면 같은 타입의 다른 실패를 통과시킬 수 있습니다.
예를 들어 `TypeError`는 입력 검증, 속성 접근, 잘못된 함수 호출 등 여러 이유로 발생할 수 있습니다.
중요한 도메인 실패라면 타입과 메시지 또는 커스텀 속성을 함께 확인하는 편이 안전합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

class PermissionError extends Error {
  constructor(action) {
    super(`permission denied: ${action}`);
    this.name = 'PermissionError';
    this.code = 'E_PERMISSION';
  }
}

async function deletePost(user, postId) {
  if (!user.canDelete) {
    throw new PermissionError(`delete post ${postId}`);
  }

  return true;
}

test('deletePost rejects without delete permission', async () => {
  await assert.rejects(
    () => deletePost({ canDelete: false }, 'post-42'),
    {
      name: 'PermissionError',
      code: 'E_PERMISSION',
      message: /delete post post-42/
    }
  );
});
```

객체 조건을 사용하면 에러의 `name`, `code`, `message`를 한 번에 검증할 수 있습니다.
이 패턴은 도메인 에러를 명시적으로 다루는 서비스 코드에 잘 맞습니다.
문자열 전체가 자주 바뀐다면 `message`는 정규식으로 핵심 단어만 확인하는 편이 유지보수에 유리합니다.

### H3. 에러 메시지 전체보다 의미 있는 조각을 검증한다

테스트가 에러 메시지 전체에 묶이면 작은 문구 수정에도 자주 깨집니다.
반대로 아무 메시지도 검증하지 않으면 엉뚱한 실패를 놓칠 수 있습니다.
균형점은 사용자나 호출자에게 중요한 키워드, 필드명, 에러 코드만 확인하는 것입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function parsePayload(payload) {
  if (!payload.startsWith('{')) {
    throw new SyntaxError('invalid JSON payload: expected object');
  }

  return JSON.parse(payload);
}

test('parsePayload rejects invalid JSON payload shape', async () => {
  await assert.rejects(
    () => parsePayload('not-json'),
    /invalid JSON payload/
  );
});
```

이 테스트는 메시지의 핵심 의미만 검증합니다.
나중에 `expected object` 같은 세부 문구가 바뀌어도 테스트 의도는 유지됩니다.
반면 `invalid JSON payload`라는 중요한 실패 이유가 사라지면 테스트가 알려 줍니다.

## throws와 rejects를 구분하는 기준

### H3. 동기 예외는 throws, 비동기 실패는 rejects를 쓴다

`assert.throws()`는 함수가 호출되는 즉시 던지는 동기 예외를 검증합니다.
Promise 안에서 나중에 reject되는 실패에는 `assert.rejects()`를 써야 합니다.
두 assertion을 섞어 쓰면 테스트가 통과하지 않거나, 비동기 실패를 제대로 기다리지 못할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function readConfigSync(source) {
  if (!source) {
    throw new Error('source is required');
  }

  return { source };
}

async function readConfigAsync(source) {
  if (!source) {
    throw new Error('source is required');
  }

  return { source };
}

test('sync config validation uses throws', () => {
  assert.throws(
    () => readConfigSync(''),
    /source is required/
  );
});

test('async config validation uses rejects', async () => {
  await assert.rejects(
    () => readConfigAsync(''),
    /source is required/
  );
});
```

두 함수는 같은 메시지로 실패하지만 테스트 방식은 다릅니다.
동기 함수는 `assert.throws()`로 즉시 검증하고, async 함수는 `await assert.rejects()`로 Promise rejection을 기다립니다.
함수 구현이 async인지 아닌지는 테스트 assertion 선택에 직접 영향을 줍니다.

### H3. await 누락은 테스트 신뢰도를 떨어뜨린다

`assert.rejects()`는 Promise를 반환합니다.
따라서 테스트에서 `await`하거나 반환하지 않으면 test runner가 assertion 완료를 기다리지 못할 수 있습니다.
이 실수는 테스트가 거짓으로 통과하는 원인이 되므로 코드 리뷰에서 꼭 확인해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function reserveSeat(seatId) {
  if (!seatId) {
    throw new Error('seatId is required');
  }

  return { seatId };
}

test('reserveSeat rejects missing seatId', async () => {
  await assert.rejects(
    () => reserveSeat(''),
    /seatId is required/
  );
});
```

좋은 규칙은 `rejects`가 보이면 앞에 `await`도 함께 보여야 한다고 생각하는 것입니다.
테스트 콜백도 `async`로 선언해 assertion을 자연스럽게 기다리게 만드세요.
Promise를 직접 반환하는 스타일도 가능하지만, 여러 assertion이 섞이면 `await`가 더 명확합니다.

## 안정적인 비동기 실패 테스트 체크리스트

### H3. 실패 조건을 하나씩 좁게 테스트한다

비동기 실패 테스트는 한 번에 너무 많은 조건을 검증하면 원인이 흐려집니다.
입력 누락, 권한 부족, 외부 서비스 실패, timeout은 서로 다른 테스트로 나누는 편이 좋습니다.
각 테스트가 하나의 실패 계약만 고정하면 리팩터링 중 깨진 지점을 빠르게 찾을 수 있습니다.

- `await assert.rejects(...)` 형태로 assertion 완료를 기다리는가?
- 에러 타입, 코드, 메시지 중 중요한 조건을 검증하는가?
- 동기 함수에는 `throws`, async 함수에는 `rejects`를 사용했는가?
- 외부 네트워크 대신 mock이나 주입 가능한 함수로 실패를 재현하는가?
- 에러 메시지 전체보다 중요한 키워드나 코드에 초점을 맞췄는가?

## FAQ

### H3. assert.rejects에 Promise를 바로 넘겨도 되나요?

가능합니다.
다만 함수 형태로 넘기면 호출 시점이 assertion 안에 들어가고, 준비 코드와 실행 코드를 분리하기 쉬워집니다.
팀 규칙으로 하나를 정해 일관되게 쓰는 것이 좋습니다.

### H3. 에러 메시지를 꼭 검증해야 하나요?

항상 필요한 것은 아닙니다.
하지만 같은 타입의 에러가 여러 경로에서 발생할 수 있다면 메시지 일부나 에러 코드를 함께 검증하는 편이 안전합니다.
사용자에게 노출되는 검증 메시지라면 더 명확하게 테스트하는 것이 좋습니다.

### H3. rejects 테스트가 가끔 통과하고 가끔 실패하면 어디를 봐야 하나요?

먼저 `await` 누락 여부를 확인하세요.
그다음 공유 mock, 전역 상태, 타이머, 외부 서비스 응답처럼 테스트 순서나 실행 타이밍에 영향을 받는 요소를 분리해야 합니다.
비동기 실패 테스트는 재현 가능한 입력과 격리된 의존성에서 가장 안정적으로 동작합니다.

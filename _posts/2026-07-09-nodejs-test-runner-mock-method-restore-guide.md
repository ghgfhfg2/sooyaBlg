---
layout: post
title: "Node.js test runner mock.method 가이드: 객체 메서드를 안전하게 대체하고 복원하는 법"
date: 2026-07-09 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-mock-method-restore-guide
permalink: /development/blog/seo/2026/07/09/nodejs-test-runner-mock-method-restore-guide.html
alternates:
  ko: /development/blog/seo/2026/07/09/nodejs-test-runner-mock-method-restore-guide.html
  x_default: /development/blog/seo/2026/07/09/nodejs-test-runner-mock-method-restore-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, mock, mock-method, testing, javascript, unit-test, ci]
description: "Node.js test runner의 mock.method로 기존 객체 메서드를 대체하고 호출을 검증하며 안전하게 복원하는 방법을 정리합니다. mock.fn과의 차이, restore/reset 기준, Date.now와 console 테스트 예제를 함께 설명합니다."
---

테스트할 코드가 의존성을 함수 인자로 받는다면 `mock.fn()`만으로 충분합니다.
하지만 이미 만들어진 객체의 메서드, 전역 객체의 일부 함수, 모듈에서 가져온 client 인스턴스처럼 "기존 메서드"를 잠깐 바꿔야 할 때가 있습니다.
Node.js 내장 test runner의 `mock.method()`는 이런 상황에서 메서드 대체, 호출 기록, 복원을 한 번에 다룰 수 있는 도구입니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#mockmethodobject-methodname-implementation-options)는 `mock.method(object, methodName[, implementation][, options])` 형태로 기존 객체의 메서드를 mock으로 바꿀 수 있다고 설명합니다.
같은 MockTracker에는 `mock.reset()`과 `mock.restoreAll()`도 있어 테스트가 끝난 뒤 mock 상태를 정리할 수 있습니다.
이 글에서는 `t.mock.method()`를 기준으로 객체 메서드를 안전하게 대체하고, 호출 인자를 검증하며, 전역 상태 오염을 피하는 기준을 정리합니다.

함수 호출 검증의 기본은 [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/07/09/nodejs-test-runner-mock-fn-call-verification-guide.html)를 먼저 보면 좋습니다.
시간 의존 테스트를 더 넓게 고정하는 방법은 [Node.js test runner mock timers 가이드](/development/blog/seo/2026/07/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html)에서 다뤘습니다.
테스트 격리와 병렬 실행 기준은 [Node.js test runner isolation 가이드](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html)와 함께 확인하세요.

## mock.method가 필요한 상황

### H3. 이미 존재하는 객체의 메서드를 바꾼다

`mock.fn()`은 새 함수를 만들어 테스트 대상에 넘기는 방식입니다.
반면 `mock.method()`는 이미 존재하는 객체의 특정 메서드를 mock 함수로 교체합니다.
테스트 대상 코드가 객체 인스턴스를 직접 사용하거나, 메서드 호출 자체를 감시해야 한다면 `mock.method()`가 더 자연스럽습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function createAuditService(logger) {
  return {
    recordLogin(user) {
      logger.info('user.login', {
        userId: user.id
      });
    }
  };
}

test('writes login audit log', (t) => {
  const logger = {
    info(message, context) {
      console.log(message, context);
    }
  };

  const info = t.mock.method(logger, 'info');
  const service = createAuditService(logger);

  service.recordLogin({ id: 'user_123' });

  assert.equal(info.mock.callCount(), 1);
  assert.deepEqual(info.mock.calls[0].arguments, [
    'user.login',
    { userId: 'user_123' }
  ]);
});
```

이 테스트는 실제 `logger.info()` 구현을 실행하지 않습니다.
대신 audit log가 어떤 메시지와 컨텍스트로 호출됐는지 확인합니다.
로깅, metric, event publish처럼 결과값보다 외부 경계 호출이 중요한 코드는 `mock.method()`로 관찰 지점을 만들기 좋습니다.

### H3. mock.fn과 mock.method의 선택 기준을 나눈다

둘 중 무엇을 쓸지는 테스트 대상 코드가 의존성을 받는 방식에 따라 결정하면 됩니다.
새 함수를 주입할 수 있다면 `mock.fn()`이 단순하고, 기존 객체의 메서드를 바꿔야 한다면 `mock.method()`가 적합합니다.
가능하다면 의존성 주입을 먼저 고려하고, 어쩔 수 없이 기존 객체를 다뤄야 할 때 메서드 mock을 선택하세요.

```text
mock.fn이 잘 맞는 경우:
- sendEmail 함수를 인자로 넘길 수 있다
- publish 함수를 테스트 안에서 직접 만들 수 있다
- 순수 함수에 가까운 작은 단위를 검증한다

mock.method가 잘 맞는 경우:
- logger.info 같은 기존 객체 메서드를 감시한다
- Date.now 같은 전역 메서드를 짧게 대체한다
- client.request 같은 인스턴스 메서드의 호출 인자를 확인한다
```

테스트가 간단해지는 방향은 대체로 설계도 단순해지는 방향입니다.
메서드 mock이 계속 늘어난다면 코드가 외부 객체에 강하게 붙어 있다는 신호일 수 있습니다.
그럴 때는 테스트 기법보다 먼저 의존성을 더 작은 인터페이스로 나눌 수 있는지 검토하는 편이 좋습니다.

## 호출 검증과 반환값 제어

### H3. 메서드 호출 횟수와 인자를 확인한다

`mock.method()`가 반환하는 mock 함수는 `mock.fn()`과 비슷하게 호출 기록을 제공합니다.
`callCount()`로 횟수를 확인하고, `mock.calls`에서 인자를 꺼내 검증할 수 있습니다.
이 방식은 기존 객체를 유지하면서 특정 메서드만 테스트 대상으로 끌어오는 데 유용합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

async function syncUser(user, apiClient) {
  const response = await apiClient.request('/users/sync', {
    method: 'POST',
    body: JSON.stringify({
      id: user.id,
      email: user.email
    })
  });

  return response.ok;
}

test('sends user sync request', async (t) => {
  const apiClient = {
    async request() {
      throw new Error('real request must not run');
    }
  };

  const request = t.mock.method(apiClient, 'request', async () => {
    return { ok: true };
  });

  const ok = await syncUser({
    id: 'user_123',
    email: 'dev@example.com'
  }, apiClient);

  assert.equal(ok, true);
  assert.equal(request.mock.callCount(), 1);
  assert.deepEqual(request.mock.calls[0].arguments, [
    '/users/sync',
    {
      method: 'POST',
      body: JSON.stringify({
        id: 'user_123',
        email: 'dev@example.com'
      })
    }
  ]);
});
```

이 예시는 실제 네트워크 요청을 막고, request 메서드의 반환값을 제어합니다.
동시에 endpoint와 payload가 기대한 형태인지 확인합니다.
외부 API 호출 테스트에서는 네트워크 성공 여부보다 "어떤 요청을 만들었는가"가 단위 테스트의 핵심인 경우가 많습니다.

### H3. 원래 구현을 유지하면서 호출만 감시할 수도 있다

때로는 메서드 동작은 그대로 두고 호출 기록만 보고 싶을 수 있습니다.
이 경우 대체 구현을 넣지 않고 메서드를 mock 처리하면 원래 메서드가 호출되면서 기록도 남길 수 있습니다.
단, 원래 구현이 파일, 네트워크, 전역 상태를 건드린다면 테스트에서 그대로 실행해도 안전한지 먼저 확인해야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

const formatter = {
  normalize(input) {
    return input.trim().toLowerCase();
  }
};

function buildSlug(title) {
  return formatter.normalize(title).replaceAll(' ', '-');
}

test('normalizes title before building slug', (t) => {
  const normalize = t.mock.method(formatter, 'normalize');

  const slug = buildSlug('  Node Test Runner  ');

  assert.equal(slug, 'node-test-runner');
  assert.equal(normalize.mock.callCount(), 1);
  assert.deepEqual(normalize.mock.calls[0].arguments, ['  Node Test Runner  ']);
});
```

이런 spy 성격의 테스트는 조심해서 써야 합니다.
`normalize()` 호출 자체가 중요한 계약이 아니라면, 최종 slug 값만 검증하는 테스트가 더 오래 유지됩니다.
호출 검증은 외부 부작용이나 운영상 중요한 경계에 가까울수록 가치가 큽니다.

## 전역 메서드 mock 주의사항

### H3. Date.now는 짧은 범위에서만 대체한다

시간 의존 코드는 테스트를 흔들리게 만드는 대표적인 원인입니다.
`Date.now()`를 사용하는 코드라면 `mock.method()`로 해당 메서드만 짧게 대체할 수 있습니다.
다만 전역 객체를 바꾸는 테스트는 다른 테스트와 섞일 때 위험해질 수 있으므로 `t.mock` 범위 안에서 최소한으로 사용해야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function createSession(userId) {
  return {
    userId,
    issuedAt: Date.now()
  };
}

test('creates session with fixed timestamp', (t) => {
  t.mock.method(Date, 'now', () => 1_783_512_000_000);

  assert.deepEqual(createSession('user_123'), {
    userId: 'user_123',
    issuedAt: 1_783_512_000_000
  });
});
```

전역 시간 mock은 테스트를 읽기 쉽게 만들지만, 범위가 넓어지면 원인 모를 실패를 만들 수 있습니다.
가능하면 `now` 함수를 인자로 받는 구조가 더 안전합니다.
전역 메서드를 mock해야 한다면 한 테스트 안에서만 쓰고, 병렬 실행 중 공유 상태가 새지 않도록 주의하세요.

### H3. console mock은 출력 억제와 검증을 분리한다

테스트 중 불필요한 로그가 많이 나오면 실패 원인을 찾기 어려워집니다.
`console.warn`이나 `console.error`를 mock으로 바꾸면 출력은 억제하면서 경고가 발생했는지도 확인할 수 있습니다.
다만 로그 메시지 전체 문구를 지나치게 엄격하게 고정하면 문구 수정만으로 테스트가 깨질 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function parseConfig(config) {
  if (!config.timeout) {
    console.warn('timeout is not configured');
  }

  return {
    timeout: config.timeout ?? 1000
  };
}

test('warns when timeout is missing', (t) => {
  const warn = t.mock.method(console, 'warn', () => {});

  const result = parseConfig({});

  assert.deepEqual(result, { timeout: 1000 });
  assert.equal(warn.mock.callCount(), 1);
  assert.match(warn.mock.calls[0].arguments[0], /timeout/);
});
```

여기서는 경고 문구 전체가 아니라 핵심 키워드만 확인합니다.
테스트가 보호해야 하는 것은 "timeout 누락 시 경고한다"는 동작이지, 모든 문장 부호가 그대로 유지되는지는 아닙니다.
운영 로그도 사용자에게 노출되는 public API처럼 다뤄야 할 때만 더 엄격하게 검증하세요.

## 복원과 정리 전략

### H3. t.mock 범위를 우선 사용한다

top-level `mock` export도 사용할 수 있지만, 일반적인 단위 테스트에서는 `t.mock`을 우선 쓰는 편이 안전합니다.
테스트 컨텍스트에 속한 mock은 해당 테스트의 생명주기와 함께 관리하기 쉬워서, 여러 테스트가 같은 파일에서 실행될 때 상태가 덜 섞입니다.
특히 전역 메서드를 대체하는 경우에는 테스트별 범위를 좁히는 것이 중요합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

const featureFlags = {
  isEnabled(name) {
    return name === 'stable-feature';
  }
};

function shouldUseNewCheckout() {
  return featureFlags.isEnabled('new-checkout');
}

test('uses mocked feature flag in this test only', (t) => {
  t.mock.method(featureFlags, 'isEnabled', () => true);

  assert.equal(shouldUseNewCheckout(), true);
});

test('uses original feature flag in the next test', () => {
  assert.equal(shouldUseNewCheckout(), false);
});
```

이 예시는 각 테스트가 어떤 상태를 기대하는지 분명하게 보여 줍니다.
첫 번째 테스트에서는 feature flag 메서드를 바꾸지만, 다음 테스트는 원래 구현을 기준으로 검증합니다.
테스트 간 상태가 섞이지 않는 구조는 CI에서 재현하기 어려운 실패를 줄여 줍니다.

### H3. reset과 restoreAll의 차이를 의식한다

MockTracker의 `reset()`은 mock을 기본 동작으로 되돌리고 내부 mock 상태를 초기화하는 데 쓰입니다.
`restoreAll()`은 이 tracker로 만든 mock의 원래 동작을 복원하지만, mock instance 자체는 그대로 남는다는 차이가 있습니다.
테스트 코드에서는 대부분 `t.mock`의 생명주기에 맡기는 편이 단순하지만, 여러 mock을 한 테스트 안에서 단계별로 정리해야 한다면 차이를 알고 쓰는 것이 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

const store = {
  read(key) {
    return `real:${key}`;
  }
};

test('restores method before second phase', (t) => {
  const read = t.mock.method(store, 'read', (key) => `mock:${key}`);

  assert.equal(store.read('a'), 'mock:a');
  assert.equal(read.mock.callCount(), 1);

  t.mock.restoreAll();

  assert.equal(store.read('a'), 'real:a');
});
```

한 테스트 안에서 mock 상태와 원래 상태를 모두 검증해야 한다면 명시적 복원이 필요할 수 있습니다.
하지만 대부분의 테스트에서는 mock을 오래 들고 가지 않는 것이 더 좋습니다.
복원 코드가 많이 필요하다면 테스트를 두 개로 나누거나, 전역 객체 대신 주입 가능한 작은 객체를 사용하는 방향도 검토하세요.

## 실무 적용 체크리스트

### H3. mock.method를 쓰기 전에 확인할 것

`mock.method()`는 기존 객체를 바꾼다는 점에서 강력하지만, 그만큼 테스트 오염 가능성도 있습니다.
메서드를 대체해야 하는 이유가 분명한지, 테스트 범위가 충분히 좁은지 먼저 확인하세요.
외부 경계 호출을 검증하는 목적이라면 좋은 선택이지만, 단순 내부 helper 호출 순서를 고정하려는 목적이라면 값 중심 테스트가 더 나을 수 있습니다.

```text
checklist:
- 새 함수 주입 대신 기존 메서드를 바꿔야 하는 이유가 있는가?
- t.mock.method를 사용해 테스트별 범위를 좁혔는가?
- 전역 객체(Date, console 등)를 mock할 때 병렬 실행 영향을 고려했는가?
- 호출 인자에 개인정보, 토큰, API 키 같은 민감정보가 노출되지 않는가?
- 문구 전체보다 계약에 가까운 핵심 값만 검증하고 있는가?
- mock 복원이 필요한 경우 reset과 restoreAll의 차이를 이해하고 있는가?
```

좋은 mock 테스트는 구현을 얼리는 테스트가 아니라 외부 경계의 계약을 보호하는 테스트입니다.
`mock.method()`는 기존 객체 메서드를 다룰 수 있게 해 주지만, 모든 의존성을 메서드 mock으로 해결하라는 뜻은 아닙니다.
의존성 주입으로 단순하게 만들 수 있는 코드는 그렇게 두고, 기존 객체를 다뤄야 하는 좁은 구간에서만 메서드 mock을 쓰면 테스트는 더 안정적으로 유지됩니다.

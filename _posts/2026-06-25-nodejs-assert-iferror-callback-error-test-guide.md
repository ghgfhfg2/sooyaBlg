---
layout: post
title: "Node.js assert.ifError 가이드: 콜백 에러를 간결하게 검증하는 테스트 패턴"
date: 2026-06-25 08:00:00 +0900
lang: ko
translation_key: nodejs-assert-iferror-callback-error-test-guide
permalink: /development/blog/seo/2026/06/25/nodejs-assert-iferror-callback-error-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/25/nodejs-assert-iferror-callback-error-test-guide.html
  x_default: /development/blog/seo/2026/06/25/nodejs-assert-iferror-callback-error-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, ifError, callback, test, error, backend]
description: "Node.js assert.ifError()로 error-first callback의 에러 인자를 간결하게 검증하는 방법을 정리합니다. 콜백 테스트 구조, Promise 코드와의 차이, assert.fail과 함께 쓰는 패턴을 예제로 설명합니다."
---

Node.js 프로젝트를 오래 운영하다 보면 Promise와 `async/await`가 기본이 된 코드 안에서도 콜백 기반 API를 만날 때가 있습니다.
파일 시스템, 오래된 라이브러리, 이벤트 브리지, 스트림 주변 코드처럼 `callback(err, result)` 형태가 아직 남아 있는 영역이 많기 때문입니다.
이런 코드를 테스트할 때는 성공 콜백에서 `err`가 비어 있는지 먼저 확인해야 합니다.

이때 Node.js `node:assert/strict`의 `assert.ifError()`를 쓰면 error-first callback의 에러 인자를 짧고 명확하게 검증할 수 있습니다.
[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 `assert.ifError(value)`가 전달된 값이 `undefined`나 `null`이 아닐 때 assertion 실패를 일으키는 API라고 설명합니다.
즉 콜백의 첫 번째 인자인 `err`가 실제 에러 값을 담고 있으면 테스트를 즉시 실패시킬 수 있습니다.

동기 예외 검증은 [Node.js assert.doesNotThrow 가이드](/development/blog/seo/2026/06/24/nodejs-assert-doesnotthrow-no-throw-test-guide.html)를 참고하세요.
비동기 Promise 실패 검증은 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)와 연결됩니다.
내장 테스트 러너의 기본 구조가 필요하다면 [Node.js built-in test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 보면 좋습니다.

## assert.ifError가 필요한 이유

### H3. error-first callback에서는 err 확인이 첫 관문이다

Node.js의 전통적인 콜백 패턴은 첫 번째 인자로 에러를 전달합니다.
성공하면 `err`는 보통 `null` 또는 `undefined`이고, 실패하면 `Error` 객체가 들어옵니다.
테스트에서는 이 값을 확인하지 않고 결과만 검증하면 실패 원인이 흐려질 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function loadUser(id, callback) {
  if (!id) {
    callback(new Error('id is required'));
    return;
  }

  callback(null, {
    id,
    name: 'Soo'
  });
}

test('loadUser returns user for valid id', (t, done) => {
  loadUser('user_1', (err, user) => {
    assert.ifError(err);
    assert.equal(user.id, 'user_1');
    assert.equal(user.name, 'Soo');
    done();
  });
});
```

`assert.ifError(err)`는 성공 콜백에서 에러가 없어야 한다는 의도를 직접 보여줍니다.
이 줄이 없으면 `user`가 `undefined`인 상태에서 뒤쪽 assertion이 실패할 수 있고, 실제 원인이 콜백 에러였다는 사실이 늦게 드러납니다.
에러 인자를 먼저 확인하면 실패 메시지와 스택이 더 읽기 쉬워집니다.

### H3. null과 undefined는 통과로 본다

`assert.ifError()`는 `null`과 `undefined`를 에러 없음으로 취급합니다.
그래서 콜백 구현이 `callback(null, value)`를 쓰든 `callback(undefined, value)`를 쓰든 성공 경로를 같은 방식으로 테스트할 수 있습니다.

```js
import assert from 'node:assert/strict';

assert.ifError(null);
assert.ifError(undefined);
```

반대로 `false`, `0`, 빈 문자열처럼 falsy 값이라도 `null`이나 `undefined`가 아니면 실패합니다.
이 점이 단순한 `assert.ok(!err)`보다 명확합니다.
콜백의 에러 인자는 "거짓 같은 값이면 괜찮다"가 아니라 "에러 자리가 비어 있어야 한다"는 계약에 가깝기 때문입니다.

## 콜백 테스트를 안전하게 작성하기

### H3. done을 반드시 한 번만 호출한다

콜백 테스트에서 흔한 실수는 성공 경로와 실패 경로가 섞이면서 `done()`이 여러 번 호출되는 구조입니다.
테스트 대상 함수가 콜백을 한 번만 부른다는 보장이 없거나, assertion 실패를 직접 `done(err)`로 넘겨야 하는 경우에는 `try/catch`로 감싸는 편이 안전합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function readProfile(callback) {
  setTimeout(() => {
    callback(null, {
      id: 'profile_1',
      locale: 'ko-KR'
    });
  }, 10);
}

test('readProfile returns locale', (t, done) => {
  readProfile((err, profile) => {
    try {
      assert.ifError(err);
      assert.equal(profile.locale, 'ko-KR');
      done();
    } catch (error) {
      done(error);
    }
  });
});
```

`assert.ifError()`가 실패하면 `catch`에서 `done(error)`로 전달됩니다.
이렇게 하면 콜백 안에서 발생한 assertion 실패가 테스트 러너에 제대로 보고됩니다.
작은 테스트에서는 단순 구조로 충분하지만, 콜백 안에서 여러 검증을 한다면 이 패턴이 더 안정적입니다.

### H3. 실패 경로는 assert.ifError로 검증하지 않는다

`assert.ifError()`는 성공 경로에서 "에러가 없어야 한다"를 확인하는 도구입니다.
실패해야 하는 입력을 테스트할 때는 에러가 실제로 존재하는지, 그리고 어떤 종류의 에러인지 확인해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function parseConfig(input, callback) {
  if (!input.serviceName) {
    callback(new Error('serviceName is required'));
    return;
  }

  callback(null, {
    serviceName: input.serviceName
  });
}

test('parseConfig rejects missing serviceName', (t, done) => {
  parseConfig({}, (err, config) => {
    try {
      assert.ok(err);
      assert.match(err.message, /serviceName/);
      assert.equal(config, undefined);
      done();
    } catch (error) {
      done(error);
    }
  });
});
```

실패 경로에서는 `assert.ok(err)`와 `assert.match(err.message, /serviceName/)`처럼 기대한 에러 신호를 직접 검증합니다.
에러 메시지를 너무 길게 고정하면 문구 수정 때 테스트가 자주 깨질 수 있으므로, 핵심 필드명이나 에러 코드를 확인하는 편이 실무적으로 좋습니다.
문자열 패턴 검증은 [Node.js assert.match 가이드](/development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html)와도 이어집니다.

## Promise 코드와의 차이

### H3. Promise에는 assert.rejects를 우선 사용한다

Promise 기반 API라면 `assert.ifError()`보다 `assert.rejects()` 또는 결과 assertion을 사용하는 편이 자연스럽습니다.
`assert.ifError()`는 콜백의 에러 인자를 검사하는 데 맞는 도구이고, Promise rejection은 Promise assertion으로 다루는 편이 의도가 더 분명합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function loadUserAsync(id) {
  if (!id) {
    throw new Error('id is required');
  }

  return {
    id,
    name: 'Soo'
  };
}

test('loadUserAsync rejects missing id', async () => {
  await assert.rejects(
    () => loadUserAsync(''),
    /id is required/
  );
});

test('loadUserAsync returns user for valid id', async () => {
  const user = await loadUserAsync('user_1');

  assert.equal(user.id, 'user_1');
  assert.equal(user.name, 'Soo');
});
```

성공 경로에서는 `await` 중 rejection이 발생하면 테스트가 실패합니다.
따라서 반환 값이 중요한 함수라면 `doesNotReject()`보다 결과를 직접 검증하는 편이 정보가 많습니다.
콜백 기반 API를 Promise로 감싼 뒤 테스트한다면, 콜백 에러는 `reject()`로 연결하고 테스트에서는 Promise 스타일 assertion을 사용하세요.

### H3. util.promisify로 테스트 표면을 단순화할 수 있다

콜백 API를 계속 직접 테스트해야 하는 경우도 있지만, 새 코드에서는 Promise 표면을 만들어 테스트하는 편이 더 단순할 때가 많습니다.
Node.js의 `util.promisify()`를 쓰면 error-first callback API를 Promise 함수처럼 다룰 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { promisify } from 'node:util';

function fetchSettings(callback) {
  callback(null, {
    featureFlag: true,
    timeoutMs: 1000
  });
}

const fetchSettingsAsync = promisify(fetchSettings);

test('fetchSettings returns feature flag', async () => {
  const settings = await fetchSettingsAsync();

  assert.equal(settings.featureFlag, true);
  assert.equal(settings.timeoutMs, 1000);
});
```

이 방식은 콜백 테스트의 `done`, `try/catch`, 중복 호출 문제를 줄여 줍니다.
다만 콜백 자체가 공개 API라면 최소한의 콜백 테스트도 남겨 두는 것이 좋습니다.
라이브러리 사용자에게 제공하는 계약이 콜백이라면, 그 표면이 실제로 올바르게 동작하는지 확인해야 합니다.

## assert.fail과 함께 쓰는 패턴

### H3. 호출되면 안 되는 콜백을 명시한다

때로는 특정 상황에서 성공 콜백이 호출되면 안 되는 코드를 테스트해야 합니다.
이때 `assert.fail()`을 함께 쓰면 "여기까지 오면 실패"라는 의도를 분명히 남길 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function connect(options, callback) {
  if (!options.url) {
    callback(new Error('url is required'));
    return;
  }

  callback(null, { connected: true });
}

test('connect fails without url', (t, done) => {
  connect({}, (err, connection) => {
    try {
      assert.ok(err);
      assert.match(err.message, /url/);

      if (connection) {
        assert.fail('connection should not be returned on failure');
      }

      done();
    } catch (error) {
      done(error);
    }
  });
});
```

실패 경로에서 결과 객체가 함께 전달되면 호출자가 잘못된 상태를 사용할 수 있습니다.
이런 계약을 테스트에 적어 두면 나중에 구현이 바뀌어도 의도치 않은 혼합 상태를 빨리 잡을 수 있습니다.
성공 경로에서는 `assert.ifError(err)`를 쓰고, 실패 경로에서는 에러와 결과의 부재를 각각 확인하는 식으로 나누면 읽기 쉽습니다.

### H3. 콜백이 호출되지 않는 문제도 따로 잡는다

콜백 테스트에서는 에러 인자뿐 아니라 콜백이 실제로 호출됐는지도 중요합니다.
`node:test`의 `done`을 사용하는 테스트는 `done()`이 호출되지 않으면 타임아웃이나 미완료 상태로 실패할 수 있지만, 원인을 더 선명하게 만들고 싶다면 테스트 대상의 흐름을 작게 유지하세요.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function resolveCache(key, callback) {
  if (key === 'hit') {
    callback(null, 'cached-value');
  }
}

test('resolveCache returns cached value', (t, done) => {
  resolveCache('hit', (err, value) => {
    try {
      assert.ifError(err);
      assert.equal(value, 'cached-value');
      done();
    } catch (error) {
      done(error);
    }
  });
});
```

테스트가 콜백 호출 여부와 결과 값을 함께 확인하므로 실패 지점이 좁아집니다.
복잡한 비동기 흐름에서는 콜백 API를 Promise로 감싸거나, 이벤트 기반 코드라면 이벤트 발생을 기다리는 헬퍼를 만드는 편이 테스트 유지보수에 더 낫습니다.

## 실무 적용 체크리스트

### H3. 성공 경로와 실패 경로를 분리한다

`assert.ifError()`는 성공 경로의 첫 줄에 두는 것이 가장 읽기 쉽습니다.
실패 경로에서는 `assert.ok(err)`, `assert.match(err.message, /keyword/)`, 에러 코드 검증처럼 구체적인 assertion을 사용하세요.
한 테스트 안에서 성공과 실패를 모두 검증하려고 하면 콜백 흐름이 복잡해집니다.

### H3. 결과 assertion을 생략하지 않는다

`assert.ifError(err)`는 에러가 없다는 사실만 알려 줍니다.
콜백이 잘못된 값을 반환해도 `err`가 비어 있으면 통과합니다.
따라서 성공 경로에서는 결과 객체, 상태 변화, 호출 횟수처럼 실제 계약을 함께 검증해야 합니다.

### H3. 새 코드는 Promise 표면을 고려한다

콜백 API가 프로젝트의 공개 계약이라면 `assert.ifError()`는 여전히 유용합니다.
하지만 내부 구현이나 신규 코드라면 Promise 표면을 제공하고 `async/await` 테스트로 단순화하는 편이 유지보수에 좋습니다.
테스트가 단순할수록 실패 원인을 더 빨리 찾을 수 있습니다.

## FAQ

### H3. assert.ifError는 assert.ok(!err)와 같은가요?

의도는 비슷하지만 완전히 같지는 않습니다.
`assert.ok(!err)`는 falsy 값을 모두 에러 없음처럼 다루지만, `assert.ifError()`는 `null`과 `undefined`만 통과시킵니다.
콜백의 에러 인자를 검증할 때는 `assert.ifError(err)`가 더 정확한 표현입니다.

### H3. async 함수에서도 assert.ifError를 써도 되나요?

일반적인 Promise 기반 `async` 함수에는 쓰지 않는 편이 좋습니다.
Promise rejection은 `assert.rejects()`로 검증하고, 성공 경로는 `await`로 결과를 받은 뒤 값을 직접 확인하세요.
`assert.ifError()`는 error-first callback의 첫 번째 인자를 확인할 때 가장 잘 맞습니다.

### H3. 콜백 테스트에서 try/catch는 항상 필요한가요?

항상 필요하지는 않습니다.
하지만 콜백 안에서 여러 assertion을 실행하고 `done`으로 완료를 알려야 한다면 `try/catch`로 assertion 실패를 `done(error)`에 전달하는 패턴이 안정적입니다.
테스트가 Promise를 반환하도록 바꿀 수 있다면 그쪽이 더 단순합니다.

## 마무리

`assert.ifError()`는 오래된 API처럼 보이지만, 콜백 기반 Node.js 코드를 테스트할 때는 여전히 선명한 도구입니다.
성공 콜백의 첫 줄에서 `err`를 확인하면 실패 원인을 빠르게 좁힐 수 있고, 결과 assertion과 함께 두면 테스트 계약도 더 분명해집니다.

핵심은 역할을 나누는 것입니다.
콜백 성공 경로에서는 `assert.ifError(err)`로 에러 부재를 확인하고, 반환 값은 별도 assertion으로 검증하세요.
실패 경로에서는 `assert.ok(err)`와 메시지 또는 코드 검증을 사용하세요.
Promise 코드라면 `assert.rejects()`와 결과 assertion을 우선 선택하면 됩니다.

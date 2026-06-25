---
layout: post
title: "Node.js assert.fail 가이드: 테스트에서 명시적으로 실패를 표시하는 패턴"
date: 2026-06-26 08:00:00 +0900
lang: ko
translation_key: nodejs-assert-fail-explicit-test-failure-guide
permalink: /development/blog/seo/2026/06/26/nodejs-assert-fail-explicit-test-failure-guide.html
alternates:
  ko: /development/blog/seo/2026/06/26/nodejs-assert-fail-explicit-test-failure-guide.html
  x_default: /development/blog/seo/2026/06/26/nodejs-assert-fail-explicit-test-failure-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, fail, test, assertion, error, backend]
description: "Node.js assert.fail()로 테스트에서 도달하면 안 되는 분기와 누락된 실패 경로를 명시적으로 표시하는 방법을 정리합니다. assert.throws, assert.rejects, 콜백 테스트와 함께 쓰는 실무 패턴을 예제로 설명합니다."
---

테스트 코드를 작성하다 보면 "이 줄까지 오면 실패"라고 직접 표시해야 하는 순간이 있습니다.
예상한 예외가 발생하지 않았거나, 실패 콜백 대신 성공 콜백이 호출됐거나, 아직 구현되지 않은 케이스를 임시로 남겨야 하는 상황입니다.
이때 Node.js `node:assert/strict`의 `assert.fail()`을 사용하면 테스트 실패 의도를 코드에 명확하게 남길 수 있습니다.

[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 `assert.fail()`이 assertion 실패를 일으키는 API라고 설명합니다.
메시지를 전달하면 실패 이유를 테스트 출력에 남길 수 있으므로, 단순히 에러를 던지는 것보다 테스트 의도를 읽기 쉽습니다.
특히 `node:test`와 함께 쓰면 실패 지점과 실패 이유가 한눈에 드러납니다.

동기 코드가 예외 없이 통과해야 하는 정상 경로 검증은 [Node.js assert.doesNotThrow 가이드](/development/blog/seo/2026/06/24/nodejs-assert-doesnotthrow-no-throw-test-guide.html)를 참고하세요.
비동기 실패 검증은 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)와 연결됩니다.
콜백 기반 성공 경로의 에러 인자 검증은 [Node.js assert.ifError 가이드](/development/blog/seo/2026/06/25/nodejs-assert-iferror-callback-error-test-guide.html)를 함께 보면 좋습니다.

## assert.fail이 필요한 이유

### H3. 도달하면 안 되는 분기를 테스트에 표시한다

`assert.fail()`의 가장 단순한 역할은 도달하면 안 되는 분기를 실패로 만드는 것입니다.
조건문이나 콜백 흐름 안에서 "여기에 오면 구현이 잘못된 것"이라는 신호를 직접 남길 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function findUser(id) {
  if (id === 'known') {
    return {
      id,
      name: 'Soo'
    };
  }

  return null;
}

test('findUser returns a known user', () => {
  const user = findUser('known');

  if (!user) {
    assert.fail('known user should exist');
  }

  assert.equal(user.id, 'known');
  assert.equal(user.name, 'Soo');
});
```

이 테스트는 `user`가 없을 때 뒤쪽 assertion에서 애매하게 실패하지 않습니다.
`known user should exist`라는 메시지로 실패 원인을 바로 보여 줍니다.
작은 차이처럼 보이지만, 테스트가 많아질수록 실패 메시지의 정확도는 디버깅 시간을 크게 줄여 줍니다.

### H3. throw new Error보다 테스트 의도가 분명하다

물론 `throw new Error('message')`로도 테스트를 실패시킬 수 있습니다.
하지만 테스트 코드에서는 `assert.fail()`이 더 직접적인 문서가 됩니다.
이 줄은 애플리케이션 예외가 아니라 assertion 실패를 의도적으로 만든다는 뜻을 담고 있기 때문입니다.

```js
import assert from 'node:assert/strict';

function assertActiveUser(user) {
  if (user.status !== 'active') {
    assert.fail(`expected active user, got ${user.status}`);
  }
}
```

테스트 헬퍼 안에서 이런 패턴을 쓰면 실패 메시지도 도메인에 맞게 다듬을 수 있습니다.
다만 운영 코드의 입력 검증을 `assert.fail()`에 의존하는 것은 피하는 편이 좋습니다.
사용자 입력, API 요청, 폼 데이터처럼 정상적으로 실패할 수 있는 값은 명시적인 validation과 오류 응답으로 다뤄야 합니다.

## 예외 테스트에서 쓰는 패턴

### H3. try/catch보다 assert.throws를 먼저 고려한다

동기 함수가 예외를 던져야 한다면 보통 `assert.throws()`가 가장 간결합니다.
예외 발생 여부와 메시지 패턴을 한 번에 검증할 수 있기 때문입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function parsePort(value) {
  const port = Number(value);

  if (!Number.isInteger(port)) {
    throw new TypeError('port must be an integer');
  }

  return port;
}

test('parsePort rejects non-integer value', () => {
  assert.throws(
    () => parsePort('abc'),
    /integer/
  );
});
```

이 경우에는 `assert.fail()`이 필요하지 않습니다.
`assert.throws()`가 "예외가 반드시 발생해야 한다"는 계약을 이미 표현합니다.
테스트 API가 더 정확한 표현을 제공한다면 그 API를 우선 선택하는 것이 좋습니다.

### H3. 수동 catch가 필요할 때 누락된 throw를 잡는다

예외 객체의 여러 속성을 직접 확인해야 해서 `try/catch`를 쓰는 경우도 있습니다.
이때는 예외가 발생하지 않은 경로에 `assert.fail()`을 두면 테스트가 잘못 통과하는 일을 막을 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

class ConfigError extends Error {
  constructor(message, code) {
    super(message);
    this.name = 'ConfigError';
    this.code = code;
  }
}

function loadConfig(input) {
  if (!input.serviceName) {
    throw new ConfigError('serviceName is required', 'CONFIG_SERVICE_NAME');
  }

  return input;
}

test('loadConfig throws ConfigError for missing serviceName', () => {
  let error;

  try {
    loadConfig({});
  } catch (caught) {
    error = caught;
  }

  if (!error) {
    assert.fail('loadConfig should throw for missing serviceName');
  }

  assert.equal(error.name, 'ConfigError');
  assert.equal(error.code, 'CONFIG_SERVICE_NAME');
  assert.match(error.message, /serviceName/);
});
```

여기서 `assert.fail()`이 없으면 `loadConfig({})`가 예외를 던지지 않아도 뒤쪽 검증이 실행되지 않거나, 테스트 의도가 흐려질 수 있습니다.
실패해야 하는 입력을 테스트할 때 가장 위험한 실수입니다.
수동 `try/catch`를 쓴다면 잡힌 에러를 변수에 보관한 뒤, 에러가 없을 때 `assert.fail()`을 호출하는 구조가 안전합니다.

## 비동기 테스트에서 쓰는 패턴

### H3. Promise 실패는 assert.rejects를 우선 사용한다

Promise 기반 함수가 실패해야 한다면 `assert.rejects()`가 기본 선택입니다.
비동기 rejection을 기다리고, 기대한 메시지나 에러 타입까지 함께 검증할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function publishJob(job) {
  if (!job.id) {
    throw new Error('job id is required');
  }

  return {
    status: 'queued'
  };
}

test('publishJob rejects missing id', async () => {
  await assert.rejects(
    () => publishJob({}),
    /job id/
  );
});
```

이 테스트에는 `assert.fail()`이 필요하지 않습니다.
`publishJob({})`가 reject되지 않으면 `assert.rejects()`가 알아서 실패를 보고합니다.
Promise 실패를 검증할 때는 수동 `try/catch`보다 `assert.rejects()`가 더 짧고 안전합니다.

### H3. 비동기 catch에서 조건이 맞지 않으면 fail을 쓴다

그래도 복잡한 에러 객체를 직접 확인해야 하거나, 여러 비동기 단계를 함께 다뤄야 한다면 수동 `try/catch`가 필요할 수 있습니다.
이때도 성공 경로 끝에는 `assert.fail()`을 둬야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function syncWorkspace(options) {
  if (!options.token) {
    const error = new Error('token is required');
    error.code = 'AUTH_TOKEN_REQUIRED';
    throw error;
  }

  return {
    synced: true
  };
}

test('syncWorkspace reports missing token code', async () => {
  let error;

  try {
    await syncWorkspace({});
  } catch (caught) {
    error = caught;
  }

  if (!error) {
    assert.fail('syncWorkspace should reject without token');
  }

  assert.equal(error.code, 'AUTH_TOKEN_REQUIRED');
  assert.match(error.message, /token/);
});
```

이 구조는 `assert.rejects()`보다 장황합니다.
하지만 에러 객체의 여러 필드를 검증해야 하고, 프로젝트의 에러 코드 정책까지 함께 확인해야 한다면 읽을 만한 선택입니다.
단순한 메시지 검증이라면 `assert.rejects()`로 돌아가는 편이 좋습니다.

## 콜백 테스트와 assert.fail

### H3. 성공 콜백이 호출되면 안 되는 케이스를 표시한다

error-first callback API에서는 실패 경로 테스트가 조금 더 조심스럽습니다.
에러가 있어야 하고, 성공 결과는 없어야 합니다.
이때 결과 값이 같이 들어오면 `assert.fail()`로 계약 위반을 명시할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function createSession(input, callback) {
  if (!input.userId) {
    callback(new Error('userId is required'));
    return;
  }

  callback(null, {
    id: 'session_1',
    userId: input.userId
  });
}

test('createSession fails without userId', (t, done) => {
  createSession({}, (err, session) => {
    try {
      assert.ok(err);
      assert.match(err.message, /userId/);

      if (session) {
        assert.fail('session should not be returned when userId is missing');
      }

      done();
    } catch (error) {
      done(error);
    }
  });
});
```

실패 경로에서 에러와 결과가 동시에 전달되면 호출자는 모호한 상태를 받게 됩니다.
테스트에서 이 계약을 잡아 두면 콜백 구현이 바뀔 때 위험한 회귀를 빠르게 발견할 수 있습니다.
성공 경로에서는 `assert.ifError(err)`로 에러 부재를 확인하고, 실패 경로에서는 `assert.ok(err)`와 `assert.fail()`을 조합하면 역할이 분명해집니다.

### H3. 호출되면 안 되는 이벤트에도 사용할 수 있다

이벤트 기반 코드에서는 특정 이벤트가 발생하면 안 되는 테스트가 필요할 수 있습니다.
예를 들어 정상 입력에서는 `error` 이벤트가 없어야 합니다.
이런 경우 이벤트 핸들러 안에서 `assert.fail()`을 호출해 잘못된 이벤트 발생을 테스트 실패로 만들 수 있습니다.

```js
import assert from 'node:assert/strict';
import { EventEmitter } from 'node:events';
import { test } from 'node:test';

function runWorker(input) {
  const worker = new EventEmitter();

  queueMicrotask(() => {
    if (!input.taskId) {
      worker.emit('error', new Error('taskId is required'));
      return;
    }

    worker.emit('done', { taskId: input.taskId });
  });

  return worker;
}

test('runWorker emits done for valid task', async () => {
  const worker = runWorker({ taskId: 'task_1' });

  const result = await new Promise((resolve, reject) => {
    worker.once('error', (error) => {
      reject(error);
    });

    worker.once('done', resolve);
  });

  assert.equal(result.taskId, 'task_1');
});
```

이 예제에서는 `reject(error)`가 더 자연스럽습니다.
Promise로 감싼 이벤트 테스트에서는 에러 이벤트를 rejection으로 연결하면 테스트 러너가 실패를 잘 보고합니다.
반대로 동기적인 감시 코드나 테스트 헬퍼 안에서 "이 이벤트가 발생하면 실패"라는 메시지를 직접 남기고 싶다면 `assert.fail('unexpected error event')`를 사용할 수 있습니다.

## 남용을 피해야 하는 경우

### H3. 더 구체적인 assertion을 대체하지 않는다

`assert.fail()`은 범용 실패 도구입니다.
하지만 값 비교에는 `assert.equal()`, 객체 비교에는 `assert.deepEqual()` 또는 `assert.deepStrictEqual()`, 예외 검증에는 `assert.throws()`와 `assert.rejects()`가 더 적합합니다.

```js
import assert from 'node:assert/strict';

const actual = {
  status: 'ready'
};

assert.deepEqual(actual, {
  status: 'ready'
});
```

위 테스트를 직접 조건문과 `assert.fail()`로 바꾸면 실패 출력의 품질이 떨어집니다.
전용 assertion은 actual, expected, diff 정보를 더 잘 보여 줍니다.
`assert.fail()`은 전용 assertion으로 표현하기 어려운 흐름 제어 지점에만 쓰는 편이 좋습니다.

### H3. TODO 테스트를 방치하는 용도로 쓰지 않는다

`assert.fail('TODO')`는 아직 작성하지 않은 테스트를 표시하는 데 사용할 수 있습니다.
하지만 main 브랜치에 오래 남겨 두면 항상 실패하는 테스트가 되어 CI를 막거나, 반대로 테스트 실행에서 제외되며 잊히기 쉽습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('export report handles empty rows', { todo: true }, () => {
  assert.fail('define expected empty report behavior');
});
```

Node.js test runner의 `todo` 옵션처럼 의도를 드러내는 기능을 함께 쓰면 더 안전합니다.
다만 실제 배포 전에는 TODO 테스트를 구현하거나 이슈로 분리해야 합니다.
실패를 남기는 것은 시작점이지, 테스트 설계를 대신하지 않습니다.

## 실무 적용 체크리스트

### H3. 메시지는 구체적으로 쓴다

`assert.fail('failed')`처럼 넓은 메시지는 도움이 적습니다.
어떤 함수, 어떤 입력, 어떤 기대가 깨졌는지 적어야 실패 로그만 보고도 다음 행동을 정할 수 있습니다.
예를 들어 `loadConfig should throw for missing serviceName`처럼 조건과 기대를 함께 쓰는 편이 좋습니다.

### H3. 수동 try/catch에는 실패 경로를 반드시 둔다

예외가 발생해야 하는 테스트에서 수동 `try/catch`를 쓴다면 `try` 블록의 마지막에 `assert.fail()`을 둡니다.
그 줄이 없으면 예외가 누락돼도 테스트가 통과할 수 있습니다.
가능하면 `assert.throws()`와 `assert.rejects()`로 더 단순하게 표현할 수 있는지 먼저 확인하세요.

### H3. 검증 API의 역할을 나눈다

값 검증은 값 assertion, 예외 검증은 예외 assertion, 흐름 제어의 불가능 지점은 `assert.fail()`로 나누면 테스트가 읽기 쉬워집니다.
모든 실패를 `assert.fail()`로 처리하면 실패 출력이 빈약해지고 유지보수가 어려워집니다.
반대로 특정 분기가 절대 실행되면 안 된다는 신호에는 `assert.fail()`만큼 직접적인 표현도 드뭅니다.

## FAQ

### H3. assert.fail과 throw new Error는 무엇이 다른가요?

둘 다 테스트를 실패시킬 수 있습니다.
하지만 `assert.fail()`은 assertion 실패를 의도적으로 만든다는 의미가 더 분명하고, 테스트 코드에서 실패 의도를 읽기 쉽습니다.
애플리케이션 로직의 일반적인 오류 처리에는 도메인 에러나 validation을 사용하고, 테스트 흐름의 실패 표시에는 `assert.fail()`을 쓰는 식으로 구분하세요.

### H3. assert.fail을 운영 코드에서도 써도 되나요?

가능은 하지만 신중해야 합니다.
불변 조건이나 내부 개발 오류를 잡는 용도라면 쓸 수 있지만, 사용자 입력처럼 예상 가능한 실패를 처리하는 용도로는 적합하지 않습니다.
운영 코드에서는 명시적인 에러 타입, 상태 코드, validation 결과를 반환하는 방식이 더 낫습니다.

### H3. assert.fail 메시지는 꼭 필요하나요?

필수는 아니지만 실무에서는 넣는 편이 좋습니다.
메시지가 없으면 실패는 보이지만 왜 실패했는지 파악하기 어렵습니다.
특히 CI 로그만 보고 원인을 찾아야 하는 상황에서는 구체적인 메시지가 큰 차이를 만듭니다.

## 마무리

`assert.fail()`은 테스트에서 실패를 직접 선언하는 도구입니다.
도달하면 안 되는 분기, 예외가 반드시 발생해야 하는 수동 `try/catch`, 콜백 실패 경로의 모호한 결과처럼 전용 assertion만으로 표현하기 어려운 지점에 잘 맞습니다.

핵심은 남용하지 않는 것입니다.
값 비교와 예외 검증은 더 구체적인 assertion API에 맡기고, `assert.fail()`은 "여기까지 오면 테스트가 깨져야 한다"는 흐름의 신호로 사용하세요.
메시지를 구체적으로 남기면 실패 로그가 곧 디버깅 안내서가 됩니다.

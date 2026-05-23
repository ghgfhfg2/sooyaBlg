---
layout: post
title: "Node.js assert.rejects 가이드: 비동기 예외 테스트를 안정적으로 작성하는 법"
date: 2026-05-24 08:00:00 +0900
lang: ko
translation_key: nodejs-assert-rejects-async-error-testing-guide
permalink: /development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html
alternates:
  ko: /development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html
  x_default: /development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, rejects, async, error-handling, testing, javascript]
description: "Node.js assert.rejects로 Promise 실패와 async 함수 예외를 정확하게 테스트하는 방법을 정리했습니다. rejects와 throws의 차이, Error 객체 검증, AbortError 테스트, 흔한 실수와 CI에서 안정적인 패턴까지 다룹니다."
---

비동기 코드는 성공 경로보다 실패 경로를 놓치기 쉽습니다.
`await`을 빠뜨리거나, `assert.throws`로 Promise 실패를 검사하거나, 에러 메시지만 느슨하게 비교하면 테스트는 통과하지만 실제 버그를 막지 못할 수 있습니다.
특히 네트워크 요청, 파일 처리, 작업 취소, 권한 검사처럼 실패가 정상 흐름의 일부인 코드에서는 비동기 예외 테스트가 품질의 핵심입니다.

Node.js의 `node:assert/strict` 모듈은 `assert.rejects`로 Promise rejection을 검증할 수 있습니다.
기본 테스트 실행은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)에서 다뤘고, 의존성 분리는 [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html)에서 정리했습니다.
이 글에서는 `assert.rejects`를 실무 테스트에 안정적으로 적용하는 기준을 정리합니다.

## assert.rejects가 필요한 이유

### async 함수의 실패는 동기 예외와 다르다

`async` 함수 안에서 `throw`가 발생하면 호출 지점에서 바로 예외가 던져지는 것이 아니라 거부된 Promise가 반환됩니다.
그래서 동기 예외를 검사하는 `assert.throws`로는 기대한 방식대로 검증할 수 없습니다.

```js
import assert from 'node:assert/strict';

async function loadUser(id) {
  if (!id) {
    throw new TypeError('id is required');
  }

  return { id };
}

await assert.rejects(
  () => loadUser(''),
  TypeError,
);
```

`assert.rejects`는 함수가 반환한 Promise가 reject되는지 기다린 뒤, reject 이유가 기대한 조건과 맞는지 확인합니다.
테스트 본문에서 `await`을 함께 사용해야 테스트 러너가 검증 완료 시점을 정확히 알 수 있습니다.

### 실패 경로는 제품 동작의 일부다

비동기 실패는 “예외 상황”만 의미하지 않습니다.
사용자가 잘못된 값을 넣었을 때, 외부 API가 타임아웃됐을 때, 취소 신호가 들어왔을 때, 파일이 없을 때 모두 실패 경로가 됩니다.
이 경로를 테스트하지 않으면 장애가 났을 때 사용자에게 어떤 메시지가 보이는지, 재시도가 가능한지, 로그가 남는지 확인하기 어렵습니다.

[Node.js fetch AbortSignal timeout 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)처럼 취소와 타임아웃을 다루는 코드에서는 성공보다 실패 테스트가 더 중요할 때도 많습니다.
`assert.rejects`는 이런 실패 조건을 문서처럼 남기는 역할을 합니다.

## assert.rejects 기본 문법

### Promise를 직접 전달하기

가장 단순한 형태는 Promise를 직접 전달하는 방식입니다.
이미 Promise를 만들어 놓은 경우에는 아래처럼 쓸 수 있습니다.

```js
import assert from 'node:assert/strict';

function parseJsonAsync(input) {
  return Promise.resolve().then(() => JSON.parse(input));
}

await assert.rejects(
  parseJsonAsync('{ invalid json'),
  SyntaxError,
);
```

이 방식은 짧고 읽기 쉽습니다.
다만 Promise가 `assert.rejects` 호출 전에 이미 실행되므로, 테스트 준비 과정과 검증을 분리하기 어려운 경우가 있습니다.
대부분의 실무 테스트에서는 다음처럼 함수를 전달하는 형태가 더 안전합니다.

### async 함수를 전달하기

`assert.rejects`의 첫 번째 인자로 함수를 넘기면 Node.js가 그 함수를 호출하고 반환된 Promise를 기다립니다.
이 방식은 테스트 대상 호출을 검증 블록 안에 넣을 수 있어 의도가 더 명확합니다.

```js
import assert from 'node:assert/strict';

async function createInvoice(input) {
  if (!input.userId) {
    throw new TypeError('userId is required');
  }

  return { id: 'invoice_1', userId: input.userId };
}

await assert.rejects(
  async () => createInvoice({ userId: '' }),
  TypeError,
);
```

함수를 전달할 때는 `async () => ...` 또는 `() => promiseReturningFunction()` 형태를 사용하면 됩니다.
중요한 점은 첫 번째 인자가 “reject될 Promise를 반환”해야 한다는 것입니다.

## 에러 타입과 메시지 검증하기

### 생성자 함수로 타입을 검증하기

두 번째 인자로 `TypeError`, `RangeError`, 사용자 정의 에러 클래스 같은 생성자를 전달하면 reject 이유가 해당 에러의 인스턴스인지 확인합니다.

```js
import assert from 'node:assert/strict';

class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}

async function updateProfile(input) {
  if (!input.email) {
    throw new ValidationError('email is required', 'email');
  }

  return { ok: true };
}

await assert.rejects(
  () => updateProfile({ email: '' }),
  ValidationError,
);
```

타입 검증은 메시지 문자열보다 안정적입니다.
문구가 조금 바뀌어도 에러의 의미가 유지된다면 테스트를 매번 고칠 필요가 없습니다.

### 정규식으로 메시지를 검증하기

사용자에게 노출되는 메시지나 운영 로그에서 중요한 문구라면 정규식으로 메시지를 검증할 수 있습니다.
너무 긴 문장 전체를 비교하기보다 핵심 키워드만 확인하는 편이 유지보수에 좋습니다.

```js
import assert from 'node:assert/strict';

async function reserveSeat(count) {
  if (count < 1) {
    throw new RangeError('seat count must be greater than zero');
  }

  return { reserved: count };
}

await assert.rejects(
  () => reserveSeat(0),
  /greater than zero/,
);
```

정규식 검증은 빠르게 쓸 수 있지만, 타입 검증 없이 메시지만 확인하면 다른 종류의 에러가 같은 문구를 포함할 때 테스트가 잘못 통과할 수 있습니다.
중요한 테스트에서는 객체 검증이나 커스텀 검증 함수를 함께 고려합니다.

### 객체 패턴으로 에러 속성을 검증하기

`assert.rejects`는 객체를 전달해 에러의 속성을 비교할 수 있습니다.
커스텀 에러의 `code`, `field`, `status`처럼 분기 로직에 쓰이는 값을 확인할 때 유용합니다.

```js
import assert from 'node:assert/strict';

class ApiError extends Error {
  constructor(message, code, status) {
    super(message);
    this.name = 'ApiError';
    this.code = code;
    this.status = status;
  }
}

async function fetchAccount() {
  throw new ApiError('account not found', 'ACCOUNT_NOT_FOUND', 404);
}

await assert.rejects(
  () => fetchAccount(),
  {
    name: 'ApiError',
    code: 'ACCOUNT_NOT_FOUND',
    status: 404,
  },
);
```

속성 기반 검증은 API 레이어, 도메인 예외, 작업 취소 코드처럼 후속 처리에 영향을 주는 값을 명확히 고정합니다.
단, 에러 객체에 민감정보를 담지 않는 설계가 먼저입니다.

## 커스텀 검증 함수 사용하기

### 복합 조건을 한곳에서 확인하기

검증 조건이 복잡하면 두 번째 인자로 함수를 전달할 수 있습니다.
이 함수는 reject 이유를 받아 `true`를 반환하면 통과하고, `false`를 반환하거나 예외를 던지면 실패합니다.

```js
import assert from 'node:assert/strict';

async function readConfig(fileName) {
  const error = new Error(`config file is not allowed: ${fileName}`);
  error.code = 'CONFIG_FILE_BLOCKED';
  error.safeToShow = true;
  throw error;
}

await assert.rejects(
  () => readConfig('../secret.json'),
  (error) => {
    assert.equal(error.code, 'CONFIG_FILE_BLOCKED');
    assert.equal(error.safeToShow, true);
    assert.match(error.message, /not allowed/);
    return true;
  },
);
```

검증 함수 안에서 `assert.equal`, `assert.match`를 쓰면 실패 원인이 더 구체적으로 출력됩니다.
마지막에 `return true`를 잊지 않는 것이 중요합니다.

### 검증 함수에서 너무 많은 구현을 알지 않기

커스텀 검증 함수는 강력하지만 테스트가 구현 세부사항에 과하게 묶일 수 있습니다.
스택 문자열, 내부 파일 경로, 에러 메시지 전체처럼 자주 바뀌는 값을 고정하면 리팩터링 때 테스트가 불필요하게 깨집니다.

검증 기준은 다음 순서로 잡는 것이 좋습니다.

- 사용자나 호출자가 실제로 의존하는 에러 타입
- 분기 처리에 쓰이는 `code`, `status`, `field`
- 문서화된 메시지의 핵심 키워드
- 재시도, 취소, 입력 오류를 구분하는 안전한 플래그

이 기준을 넘어서 내부 구현까지 확인해야 한다면 테스트 대상의 공개 계약이 아직 불명확한 신호일 수 있습니다.

## assert.throws와 assert.rejects 차이

### assert.throws는 동기 예외 전용이다

`assert.throws`는 함수 호출 중 즉시 던져지는 예외를 검사합니다.
Promise가 reject되는 비동기 실패에는 적합하지 않습니다.

```js
import assert from 'node:assert/strict';

function parsePort(value) {
  const port = Number(value);

  if (!Number.isInteger(port) || port <= 0) {
    throw new RangeError('port must be a positive integer');
  }

  return port;
}

assert.throws(
  () => parsePort('abc'),
  RangeError,
);
```

위 코드는 동기 함수라서 `assert.throws`가 맞습니다.
반대로 `async function`이나 Promise를 반환하는 함수라면 `assert.rejects`를 사용해야 합니다.

### async 함수에 assert.throws를 쓰면 테스트가 흔들린다

다음 코드는 의도와 다르게 동작할 수 있는 대표적인 실수입니다.

```js
import assert from 'node:assert/strict';

async function deleteUser(id) {
  if (!id) {
    throw new TypeError('id is required');
  }

  return { deleted: true };
}

await assert.rejects(
  () => deleteUser(''),
  TypeError,
);
```

`deleteUser('')`는 즉시 `TypeError`를 던지는 것이 아니라 reject된 Promise를 반환합니다.
따라서 `assert.throws`가 아니라 `assert.rejects`로 기다려야 합니다.
테스트 이름에도 “rejects”나 “비동기 실패” 같은 표현을 넣으면 리뷰 때 실수를 줄일 수 있습니다.

## 테스트 러너와 함께 쓰는 패턴

### test 함수 안에서는 반드시 await한다

Node.js test runner에서 `assert.rejects`를 사용할 때는 `await`을 빠뜨리지 않아야 합니다.
`await`이 없으면 검증 Promise가 끝나기 전에 테스트가 종료될 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

async function sendMessage(input) {
  if (!input.text) {
    throw new TypeError('text is required');
  }

  return { id: 'message_1' };
}

test('text가 없으면 메시지를 보낼 수 없다', async () => {
  await assert.rejects(
    () => sendMessage({ text: '' }),
    TypeError,
  );
});
```

테스트 콜백 자체도 `async`여야 `await`을 사용할 수 있습니다.
이 패턴은 CI에서 테스트 완료 시점을 안정적으로 맞추는 데 중요합니다.

### 여러 실패 조건은 테스트를 나눈다

하나의 테스트에서 여러 `assert.rejects`를 연달아 검사할 수는 있지만, 실패 원인을 빠르게 파악하려면 조건별로 테스트를 나누는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

async function createProject(input) {
  if (!input.name) {
    throw new TypeError('name is required');
  }

  if (input.name.length > 20) {
    throw new RangeError('name is too long');
  }

  return { id: 'project_1' };
}

test('name이 없으면 TypeError가 발생한다', async () => {
  await assert.rejects(
    () => createProject({ name: '' }),
    TypeError,
  );
});

test('name이 너무 길면 RangeError가 발생한다', async () => {
  await assert.rejects(
    () => createProject({ name: 'a'.repeat(21) }),
    RangeError,
  );
});
```

테스트가 작을수록 reporter 출력도 읽기 쉽습니다.
CI 출력 관리는 [Node.js test runner reporter 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-reporter-ci-junit-guide.html)를 함께 참고하면 좋습니다.

## AbortError와 타임아웃 테스트

### 취소 가능한 함수는 에러 이름을 검증한다

취소 흐름은 Node.js 서비스에서 점점 중요해지고 있습니다.
사용자 요청이 끊겼거나 타임아웃이 지나면 불필요한 작업을 멈춰야 리소스를 아낄 수 있습니다.
이때 테스트는 취소가 실제 rejection으로 이어지는지 확인해야 합니다.

```js
import assert from 'node:assert/strict';

async function waitForWork(signal) {
  if (signal.aborted) {
    throw new DOMException('The operation was aborted', 'AbortError');
  }

  await new Promise((resolve) => setTimeout(resolve, 10));
  return 'done';
}

const controller = new AbortController();
controller.abort();

await assert.rejects(
  () => waitForWork(controller.signal),
  {
    name: 'AbortError',
  },
);
```

`AbortError`는 환경과 API에 따라 클래스 비교보다 `name` 비교가 더 안정적인 경우가 있습니다.
특히 fetch, stream, timers promise처럼 취소 가능한 API를 섞어 쓸 때는 팀의 검증 기준을 정해 두는 편이 좋습니다.

### timeout 테스트는 시간을 짧고 명확하게 잡는다

시간 기반 테스트는 느리고 flaky해지기 쉽습니다.
실제 30초 타임아웃을 기다리는 테스트보다, 테스트용 옵션으로 10ms 같은 작은 값을 주입하는 구조가 낫습니다.

```js
import assert from 'node:assert/strict';

async function runWithTimeout(task, timeoutMs) {
  const timeout = new Promise((_, reject) => {
    setTimeout(() => {
      const error = new Error('operation timed out');
      error.code = 'TIMEOUT';
      reject(error);
    }, timeoutMs);
  });

  return Promise.race([task(), timeout]);
}

await assert.rejects(
  () => runWithTimeout(
    () => new Promise((resolve) => setTimeout(resolve, 50)),
    5,
  ),
  { code: 'TIMEOUT' },
);
```

실무에서는 타이머를 직접 오래 기다리기보다 테스트 가능한 timeout 값을 주입하고, 필요하면 [Node.js test runner mock timers 가이드](/development/blog/seo/2026/04/29/nodejs-test-runner-mock-timers-guide.html)처럼 시간을 제어하는 접근을 검토합니다.

## 흔한 실수와 예방 기준

### await을 빠뜨리지 않는다

가장 흔한 실수는 `await assert.rejects(...)`에서 `await`을 빼는 것입니다.
겉으로는 테스트가 통과하는 것처럼 보여도 실제 검증이 테스트 종료 후에 실패할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

async function failLater() {
  throw new Error('later failure');
}

test('비동기 실패를 기다린다', async () => {
  await assert.rejects(
    () => failLater(),
    /later failure/,
  );
});
```

리뷰에서는 `assert.rejects` 앞에 `await` 또는 `return`이 있는지 먼저 확인하면 많은 실수를 막을 수 있습니다.
팀 규칙으로 “test 콜백 안의 rejects는 반드시 await”을 정해도 좋습니다.

### 성공 Promise를 실패로 착각하지 않는다

`assert.rejects`는 Promise가 성공하면 실패합니다.
이 특성 덕분에 실패해야 하는 입력이 실제로 실패하는지 확인할 수 있습니다.
반대로 테스트 데이터가 잘못되어 성공 경로를 타면 즉시 알 수 있습니다.

```js
import assert from 'node:assert/strict';

async function requireAdmin(user) {
  if (user.role !== 'admin') {
    throw new Error('admin role is required');
  }

  return true;
}

await assert.rejects(
  () => requireAdmin({ role: 'member' }),
  /admin role/,
);
```

테스트 데이터는 가능한 한 작고 명확해야 합니다.
역할, 상태, 권한처럼 실패 조건을 만드는 필드만 드러나면 읽는 사람이 테스트 의도를 바로 이해할 수 있습니다.

### 에러 메시지 전체에 과하게 의존하지 않는다

에러 메시지는 사용자 경험이나 운영 로그에서 중요하지만, 테스트가 모든 문장부호까지 고정할 필요는 없습니다.
문구 전체 비교는 번역, UX 개선, 로깅 정책 변경 때 불필요한 실패를 만들 수 있습니다.

권장 순서는 다음과 같습니다.

1. 에러 클래스나 `name`으로 종류를 확인한다.
2. `code`, `status`, `field` 같은 안정적인 속성을 확인한다.
3. 꼭 필요한 경우 정규식으로 핵심 문구만 확인한다.

이 기준을 따르면 테스트가 의미 있는 계약은 지키면서도 리팩터링에 덜 취약해집니다.

## 실무 체크리스트

### 비동기 예외 테스트 작성 전 확인할 것

- 테스트 대상이 동기 함수인지 Promise 반환 함수인지 먼저 구분한다.
- Promise 실패는 `assert.rejects`, 동기 예외는 `assert.throws`를 사용한다.
- `assert.rejects` 앞에는 `await` 또는 `return`을 둔다.
- 에러 타입, 코드, 상태처럼 호출자가 의존하는 계약을 우선 검증한다.
- 시간 기반 테스트는 timeout 값을 작게 주입하거나 mock timer 사용을 검토한다.

### 코드 리뷰에서 볼 것

- async 함수에 `assert.throws`를 쓰지 않았는가?
- 실패 경로 테스트가 성공 경로와 같은 fixture를 공유해 상태가 섞이지 않는가?
- 검증 함수가 `return true`를 빠뜨리지 않았는가?
- 내부 구현 세부사항보다 공개 계약을 검증하고 있는가?
- 에러 객체나 테스트 로그에 토큰, 실제 이메일, 개인정보가 들어가지 않는가?

테스트 준비와 정리 기준은 [Node.js test runner hooks 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html)와 함께 보면 더 안정적인 구조를 만들 수 있습니다.
mock 호출 실패를 검사한다면 [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html)도 같이 연결하면 좋습니다.

## FAQ

### assert.rejects와 assert.doesNotReject는 언제 나눠 쓰나요?

`assert.rejects`는 실패해야 하는 입력이나 조건을 검증할 때 사용합니다.
`assert.doesNotReject`는 특정 비동기 작업이 실패하지 않아야 한다는 계약을 명시할 때 사용할 수 있지만, 보통은 성공 결과를 직접 `assert.equal`이나 `assert.deepEqual`로 확인하는 편이 더 정보가 많습니다.

### async 함수 안에서 throw하면 assert.throws로 잡을 수 있나요?

아니요.
`async` 함수의 `throw`는 reject된 Promise로 변환됩니다.
따라서 `assert.rejects(() => asyncFunction(), ErrorType)` 형태로 검증해야 합니다.

### 에러 메시지와 에러 코드를 모두 검사해야 하나요?

항상 둘 다 검사할 필요는 없습니다.
호출자가 분기 처리에 쓰는 값이 `code`라면 `code`를 우선 검증하고, 사용자에게 보여야 하는 문구가 중요할 때만 메시지의 핵심 키워드를 추가로 확인하는 편이 좋습니다.

### CI에서만 비동기 예외 테스트가 실패하는 이유는 무엇인가요?

대부분은 `await` 누락, 공유 fixture 오염, 시간 기반 테스트의 불안정성, 환경별 에러 메시지 차이 때문입니다.
테스트를 작게 나누고, 훅으로 상태를 정리하고, 에러 객체의 안정적인 속성을 검증하면 CI 전용 실패를 줄일 수 있습니다.

## 마무리

`assert.rejects`는 Node.js 비동기 코드의 실패 계약을 검증하는 기본 도구입니다.
핵심은 Promise rejection을 반드시 기다리고, 에러 메시지 전체보다 타입과 안정적인 속성을 중심으로 확인하는 것입니다.
실패 경로 테스트가 잘 정리되면 타임아웃, 취소, 입력 검증, 외부 의존성 오류를 더 자신 있게 다룰 수 있습니다.

비동기 테스트의 다음 단계로는 테스트별 상태 정리, mock 호출 검증, CI reporter 출력 정리를 함께 표준화하는 것이 좋습니다.
관련 글로 [Node.js test runner hooks 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html), [Node.js test runner reporter 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-reporter-ci-junit-guide.html), [Node.js assert partialDeepStrictEqual 가이드](/development/blog/seo/2026/05/18/nodejs-assert-partialdeepstrictequal-object-subset-test-guide.html)를 이어서 참고해 보세요.

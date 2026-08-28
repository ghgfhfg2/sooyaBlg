---
layout: post
title: "Node.js test runner assert.register 가이드: 커스텀 assertion을 팀 규칙으로 관리하기"
date: 2026-08-28 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-assert-register-custom-assertion-guide
permalink: /development/blog/seo/2026/08/28/nodejs-test-runner-assert-register-custom-assertion-guide.html
alternates:
  ko: /development/blog/seo/2026/08/28/nodejs-test-runner-assert-register-custom-assertion-guide.html
  x_default: /development/blog/seo/2026/08/28/nodejs-test-runner-assert-register-custom-assertion-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, assert-register, custom-assertion, testing, javascript, ci]
description: "Node.js test runner의 assert.register()로 커스텀 assertion을 등록하고 팀 테스트 규칙을 일관되게 관리하는 방법을 정리합니다. preload 구성, 오류 메시지, TypeScript 타입 보강, CI 운영 기준까지 실무 예제로 설명합니다."
---

테스트 코드를 오래 운영하다 보면 같은 검증 패턴이 여러 파일에 반복됩니다.
응답 객체가 표준 에러 형태인지 확인하거나, API 결과가 페이지네이션 계약을 지키는지 보거나, 이벤트 payload에 민감한 필드가 없는지 검사하는 코드가 대표적입니다.

이런 검증은 헬퍼 함수로도 분리할 수 있습니다.
하지만 Node.js 내장 test runner를 기준으로 팀의 공통 assertion처럼 쓰고 싶다면 `assert.register()`를 검토할 만합니다.
공식 문서 기준으로 `node:test`의 `assert` 객체는 `TestContext`에서 사용할 assertion을 구성하는 API를 제공하며, `assert.register(name, fn)`은 이름이 붙은 새 assertion 함수를 등록합니다.

이 글에서는 Node.js test runner에서 `assert.register()`를 사용해 커스텀 assertion을 만들고, preload 파일, 오류 메시지, 타입 보강, CI 운영 기준까지 실무 흐름으로 정리합니다.
스냅샷 직렬화와 리뷰 기준은 [Node.js test runner snapshot 가이드](/development/blog/seo/2026/07/17/nodejs-test-runner-snapshot-testing-guide.html), 커버리지 리포트 구성은 [Node.js test runner lcov 가이드](/development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html), 객체 부분 검증은 [assert.partialDeepStrictEqual 가이드](/development/blog/seo/2026/05/18/nodejs-assert-partialdeepstrictequal-object-subset-test-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js Test runner 공식 문서](https://nodejs.org/api/test.html)

## assert.register가 필요한 순간

### 반복되는 검증을 테스트 언어로 올리고 싶을 때

테스트 헬퍼 함수는 충분히 유용합니다.
다만 검증 이름이 프로젝트 도메인에 가까워질수록 `assert` 네임스페이스에 올리는 편이 테스트 의도를 더 직접적으로 보여 줄 때가 있습니다.

예를 들어 아래 코드는 매번 세부 필드를 직접 확인합니다.

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';

test('returns a validation error', async () => {
  const response = await createUser({ email: 'bad-value' });

  assert.equal(response.status, 400);
  assert.equal(response.body.error.code, 'VALIDATION_ERROR');
  assert.match(response.body.error.message, /email/i);
  assert.ok(Array.isArray(response.body.error.details));
});
```

한두 번이면 괜찮지만 여러 테스트 파일에서 반복되면 문제가 생깁니다.
어떤 파일은 `details`를 확인하고, 어떤 파일은 빠뜨리고, 어떤 파일은 에러 코드 문자열을 다르게 비교할 수 있습니다.

이럴 때 커스텀 assertion을 등록하면 테스트 코드를 아래처럼 정리할 수 있습니다.

```js
test('returns a validation error', async (t) => {
  const response = await createUser({ email: 'bad-value' });

  t.assert.apiValidationError(response, 'email');
});
```

검증 기준은 한 곳에 모이고, 테스트 본문에는 기대하는 동작만 남습니다.

### 팀 규칙을 CI에서 강제하고 싶을 때

커스텀 assertion은 단순 편의 기능보다 운영 규칙에 가깝게 쓰는 편이 좋습니다.

예를 들어 다음 기준이 팀에 있다면 공통 assertion으로 만들 만합니다.

- API 에러 응답은 항상 `{ error: { code, message, details } }` 형태를 지킨다.
- 공개 로그 payload에는 `password`, `token`, `authorization` 필드가 없어야 한다.
- 페이지네이션 응답은 `items`, `nextCursor`, `hasMore` 계약을 유지한다.
- 이벤트 메시지는 `type`, `version`, `occurredAt` 필드를 반드시 가진다.

이런 규칙은 실패했을 때 메시지도 중요합니다.
일반적인 `assert.deepEqual()` 실패보다 "어떤 계약이 깨졌는지"를 바로 알려 주는 assertion이 리뷰와 장애 분석 모두에 유리합니다.

## 기본 사용법

### preload 파일에서 assertion을 등록한다

`assert.register()`는 테스트 실행 전에 한 번 등록하는 방식이 가장 관리하기 쉽습니다.
프로젝트 루트에 `test/setup-assertions.js` 같은 파일을 두고, 테스트 실행 명령에서 `--import`로 로드합니다.

```js
// test/setup-assertions.js
import { assert } from 'node:test';
import assertStrict from 'node:assert/strict';

assert.register('apiValidationError', (response, fieldName) => {
  assertStrict.equal(response.status, 400);
  assertStrict.equal(response.body?.error?.code, 'VALIDATION_ERROR');
  assertStrict.match(response.body?.error?.message ?? '', new RegExp(fieldName, 'i'));
  assertStrict.ok(Array.isArray(response.body?.error?.details));
});
```

이후 `package.json`의 테스트 스크립트에서 setup 파일을 먼저 불러옵니다.

```json
{
  "scripts": {
    "test": "node --import ./test/setup-assertions.js --test"
  }
}
```

이렇게 구성하면 개별 테스트 파일마다 등록 코드를 반복하지 않아도 됩니다.
또 로컬과 CI가 같은 명령을 쓰면 assertion 등록 누락으로 인한 차이를 줄일 수 있습니다.

### TestContext의 assert에서 호출한다

등록한 assertion은 테스트 콜백의 `TestContext`에서 사용할 수 있습니다.

```js
import { test } from 'node:test';

test('rejects an invalid email', async (t) => {
  const response = await createUser({
    email: 'invalid-email',
    name: 'Soo',
  });

  t.assert.apiValidationError(response, 'email');
});
```

여기서 핵심은 커스텀 assertion 내부에서도 기존 `node:assert/strict`를 그대로 활용한다는 점입니다.
새 assertion은 완전히 새로운 테스트 프레임워크를 만드는 것이 아니라, 반복되는 검증 조합에 이름을 붙이는 방식으로 접근하는 편이 안전합니다.

## 실무 예제

### 페이지네이션 응답 검증하기

목록 API는 테스트마다 비슷한 형태를 확인합니다.
응답이 배열인지, 커서 필드가 문자열 또는 null인지, `hasMore`가 boolean인지 확인하는 규칙을 assertion으로 분리해 보겠습니다.

```js
// test/setup-assertions.js
import { assert } from 'node:test';
import assertStrict from 'node:assert/strict';

assert.register('paginatedResponse', (value) => {
  assertStrict.ok(value && typeof value === 'object', 'response must be an object');
  assertStrict.ok(Array.isArray(value.items), 'response.items must be an array');
  assertStrict.equal(typeof value.hasMore, 'boolean', 'response.hasMore must be a boolean');

  if (value.nextCursor !== null) {
    assertStrict.equal(typeof value.nextCursor, 'string', 'response.nextCursor must be a string or null');
    assertStrict.ok(value.nextCursor.length > 0, 'response.nextCursor must not be empty');
  }
});
```

테스트에서는 세부 구조 대신 계약 이름을 읽게 됩니다.

```js
import { test } from 'node:test';

test('lists posts with pagination contract', async (t) => {
  const response = await listPosts({ limit: 20 });

  t.assert.paginatedResponse(response.body);
});
```

이 방식의 장점은 계약이 바뀔 때 한 곳만 고치면 된다는 것입니다.
예를 들어 `totalCount`를 반드시 포함하도록 정책이 바뀌면 커스텀 assertion에 검증을 추가하고, 관련 테스트가 자연스럽게 새 기준을 따르게 만들 수 있습니다.

### 민감 필드가 없는지 점검하기

로그나 이벤트 payload를 테스트할 때는 민감정보가 섞이지 않았는지 확인하는 assertion도 유용합니다.
공개 개발 글이나 예제 코드에서도 실제 토큰, 이메일, 운영 식별자를 노출하지 않는 것이 기본 원칙입니다.

```js
// test/setup-assertions.js
import { assert } from 'node:test';
import assertStrict from 'node:assert/strict';

const sensitiveKeyPattern = /password|passwd|secret|token|authorization|cookie/i;

assert.register('withoutSensitiveKeys', (value) => {
  const found = [];

  function visit(node, path) {
    if (!node || typeof node !== 'object') {
      return;
    }

    for (const [key, child] of Object.entries(node)) {
      const nextPath = path ? `${path}.${key}` : key;

      if (sensitiveKeyPattern.test(key)) {
        found.push(nextPath);
      }

      visit(child, nextPath);
    }
  }

  visit(value, '');

  assertStrict.deepEqual(found, [], `sensitive keys found: ${found.join(', ')}`);
});
```

사용 예시는 간단합니다.

```js
import { test } from 'node:test';

test('publishes sanitized audit event', (t) => {
  const event = buildAuditEvent({
    userId: 'user_123',
    action: 'login.failed',
    reason: 'invalid_password',
  });

  t.assert.withoutSensitiveKeys(event);
});
```

이 assertion은 키 이름 기준의 1차 방어입니다.
실제 서비스에서는 값 자체에 토큰 형태가 들어가지 않는지, 로그 redaction 레이어가 제대로 적용되는지도 별도로 확인해야 합니다.

## 오류 메시지 설계

### 실패 원인을 한 줄로 좁힌다

커스텀 assertion은 실패 메시지가 좋아야 쓸 가치가 있습니다.
`assert.ok(false)`처럼 뭉뚱그린 실패를 던지면 테스트 작성자는 다시 assertion 내부를 열어 봐야 합니다.

아래처럼 각 검증에 의미 있는 메시지를 붙이면 CI 로그만 보고도 원인을 빨리 좁힐 수 있습니다.

```js
assert.register('eventEnvelope', (event) => {
  assertStrict.equal(typeof event?.type, 'string', 'event.type must be a string');
  assertStrict.match(event.type, /^[a-z]+(?:\.[a-z]+)+$/, 'event.type must use dotted lowercase format');
  assertStrict.equal(typeof event?.version, 'number', 'event.version must be a number');
  assertStrict.ok(Number.isInteger(event.version), 'event.version must be an integer');
  assertStrict.equal(typeof event?.occurredAt, 'string', 'event.occurredAt must be an ISO string');
});
```

좋은 메시지는 길 필요가 없습니다.
검증 대상, 기대 타입 또는 정책, 실제로 깨진 지점을 짧게 드러내면 충분합니다.

### 너무 많은 책임을 한 assertion에 넣지 않는다

커스텀 assertion 하나가 너무 많은 것을 검사하면 실패 원인이 흐려집니다.
예를 들어 "정상 응답 전체"를 한 번에 검증하는 assertion보다 아래처럼 계약을 나누는 편이 유지보수하기 쉽습니다.

- `t.assert.paginatedResponse(body)`
- `t.assert.withoutSensitiveKeys(event)`
- `t.assert.eventEnvelope(event)`
- `t.assert.apiValidationError(response, fieldName)`

작게 나눈 assertion은 테스트 본문에서도 조합하기 쉽습니다.
반대로 거대한 assertion은 테스트가 실제로 무엇을 보장하는지 숨길 수 있습니다.

## TypeScript 프로젝트에서의 타입 보강

### 테스트 전용 선언 파일을 둔다

TypeScript를 쓰는 프로젝트라면 런타임 등록과 타입 인식은 별개입니다.
`assert.register()`로 함수가 등록되어도 TypeScript가 `t.assert.apiValidationError()`를 자동으로 알지는 못합니다.

테스트 전용 타입 선언 파일을 두고 `TestContext`의 `assert` 타입을 보강합니다.
정확한 선언 방식은 프로젝트의 TypeScript 설정과 Node.js 타입 패키지 버전에 맞춰 조정해야 하지만, 방향은 아래와 같습니다.

```ts
// test/types/node-test-assertions.d.ts
import 'node:test';

declare module 'node:test' {
  interface TestContext {
    assert: TestContext['assert'] & {
      apiValidationError(response: unknown, fieldName: string): void;
      paginatedResponse(value: unknown): void;
      withoutSensitiveKeys(value: unknown): void;
    };
  }
}
```

그리고 `tsconfig.json` 또는 테스트 전용 `tsconfig.test.json`에서 이 선언 파일이 포함되도록 설정합니다.

```json
{
  "include": [
    "src/**/*.ts",
    "test/**/*.ts",
    "test/types/**/*.d.ts"
  ]
}
```

타입 보강은 런타임 동작을 바꾸지 않습니다.
따라서 테스트 실행 명령의 `--import ./test/setup-assertions.js` 구성과 타입 선언 파일을 함께 관리해야 합니다.

### 이름 충돌을 피한다

`assert.register()`는 이름으로 assertion을 등록합니다.
기존 assertion과 같은 이름을 쓰면 덮어쓸 수 있으므로, 팀 커스텀 assertion 이름은 도메인 의미가 드러나는 쪽이 좋습니다.

예를 들어 `ok`, `equal`, `match`처럼 일반적인 이름은 피합니다.
대신 `apiValidationError`, `paginatedResponse`, `eventEnvelope`처럼 프로젝트 계약을 표현하는 이름을 사용합니다.

공통 패키지로 여러 저장소에 배포한다면 접두사를 붙이는 방식도 고려할 수 있습니다.

```js
assert.register('serviceApiValidationError', validateApiError);
assert.register('serviceEventEnvelope', validateEventEnvelope);
```

## CI 운영 기준

### 등록 파일 누락을 빠르게 실패시킨다

커스텀 assertion은 setup 파일이 빠지면 테스트 런타임에서 `t.assert.someName is not a function` 같은 오류로 드러납니다.
그래도 원인을 더 빠르게 찾으려면 setup 파일 로딩 여부를 별도 테스트로 확인할 수 있습니다.

```js
import { test } from 'node:test';

test('custom assertions are registered', (t) => {
  t.assert.equal(typeof t.assert.paginatedResponse, 'function');
  t.assert.equal(typeof t.assert.withoutSensitiveKeys, 'function');
});
```

이 테스트는 작지만 유용합니다.
CI 명령이 바뀌거나, 테스트 스크립트를 별도로 실행하는 패키지에서 setup 파일이 빠졌을 때 바로 알려 줍니다.

### 공통 assertion도 테스트한다

커스텀 assertion은 테스트를 돕는 코드지만, 그 자체도 코드입니다.
특히 여러 서비스가 공유하는 assertion이라면 정상 케이스와 실패 케이스를 작게라도 테스트하는 편이 좋습니다.

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';

test('withoutSensitiveKeys passes sanitized objects', (t) => {
  t.assert.withoutSensitiveKeys({
    userId: 'user_123',
    action: 'post.created',
  });
});

test('withoutSensitiveKeys fails sensitive keys', (t) => {
  assert.throws(() => {
    t.assert.withoutSensitiveKeys({
      userId: 'user_123',
      token: 'masked-example',
    });
  }, /sensitive keys found: token/);
});
```

실패 케이스 테스트에서는 실제 토큰이나 운영 데이터를 쓰지 않습니다.
예제 값은 반드시 가짜 값으로 두고, 공개 로그에도 안전한 문자열만 사용합니다.

## 도입 체크리스트

### H3. assertion 후보를 고른다

처음부터 많은 assertion을 만들 필요는 없습니다.
아래 기준을 만족하는 검증부터 분리하면 효과가 큽니다.

- 세 개 이상의 테스트 파일에서 반복된다.
- 실패했을 때 도메인 계약을 설명해야 한다.
- 팀 코드 리뷰에서 자주 놓치는 규칙이다.
- 민감정보, 응답 스키마, 이벤트 계약처럼 일관성이 중요한 영역이다.

반대로 한 테스트에서만 쓰이는 검증은 일반 `assert` 호출로 두는 편이 낫습니다.
커스텀 assertion은 테스트를 읽기 쉽게 만들어야지, 단순 비교를 숨기기 위한 장식이 되어서는 안 됩니다.

### H3. 등록 위치와 실행 명령을 고정한다

도입할 때는 다음 항목을 같이 정리합니다.

- `test/setup-assertions.js`처럼 등록 위치를 하나로 정한다.
- `package.json`의 `test` 스크립트에 `--import`를 포함한다.
- CI도 같은 test 스크립트를 호출하게 한다.
- TypeScript 프로젝트라면 테스트 전용 선언 파일을 포함한다.
- 커스텀 assertion 자체의 정상/실패 테스트를 둔다.

이 기준이 있어야 새 테스트 파일을 추가해도 assertion 사용 방식이 흔들리지 않습니다.

### H3. 공개 예제의 민감정보를 점검한다

테스트 글이나 문서에 예제를 옮길 때는 실제 서비스 값이 들어가지 않았는지 확인해야 합니다.
특히 인증 헤더, 쿠키, API 키, 내부 도메인, 실제 사용자 식별자는 공개 글에 넣지 않습니다.

예제에는 `user_123`, `masked-example`, `https://api.example.com`처럼 의도가 드러나는 가짜 값을 사용합니다.
오류 메시지도 운영 로그 원문을 그대로 붙이기보다 필요한 구조만 재현하는 편이 안전합니다.

## FAQ

### assert.register는 기존 assert 헬퍼를 대체하나요?

대체라기보다 보완에 가깝습니다.
`equal`, `deepEqual`, `match`, `throws` 같은 기본 assertion은 계속 사용하고, 반복되는 팀 고유 검증만 `assert.register()`로 올리는 방식이 적절합니다.

### 모든 테스트 파일에서 setup 파일을 import해야 하나요?

보통은 테스트 파일마다 직접 import하지 않고, 실행 명령에서 `node --import ./test/setup-assertions.js --test`처럼 preload로 로드하는 편이 관리하기 쉽습니다.
이렇게 하면 로컬과 CI가 같은 등록 상태로 테스트를 실행할 수 있습니다.

### 커스텀 assertion이 너무 많아지면 어떻게 관리하나요?

도메인별 파일로 나누되 등록 진입점은 하나로 유지하는 방식이 좋습니다.
예를 들어 `test/assertions/api.js`, `test/assertions/events.js`를 만들고 `test/setup-assertions.js`에서 한 번에 등록하면 실행 명령은 단순하게 유지됩니다.

## 마무리

`assert.register()`는 Node.js test runner에서 반복되는 검증을 팀의 테스트 언어로 끌어올리는 도구입니다.
작고 자주 쓰이는 계약에 이름을 붙이면 테스트 본문은 더 읽기 쉬워지고, 실패 메시지는 더 실무적으로 바뀝니다.

도입할 때는 preload 파일, CI 명령, TypeScript 타입 보강, 민감정보 없는 예제까지 함께 정리하세요.
그렇게 해 두면 커스텀 assertion은 편의 함수가 아니라 테스트 품질을 일정하게 유지하는 운영 기준이 됩니다.


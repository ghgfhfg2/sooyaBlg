---
layout: post
title: "Node.js assert.doesNotReject 가이드: 비동기 코드가 실패하지 않는지 테스트하는 법"
date: 2026-06-25 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-doesnotreject-async-success-test-guide
permalink: /development/blog/seo/2026/06/25/nodejs-assert-doesnotreject-async-success-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/25/nodejs-assert-doesnotreject-async-success-test-guide.html
  x_default: /development/blog/seo/2026/06/25/nodejs-assert-doesnotreject-async-success-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, doesNotReject, async, promise, test, backend]
description: "Node.js assert.doesNotReject()로 Promise 기반 비동기 코드가 rejection 없이 완료되는지 검증하는 방법을 정리합니다. assert.rejects와의 차이, 결과 assertion을 함께 쓰는 기준, 실무 테스트 패턴을 예제로 설명합니다."
---

비동기 테스트에서는 실패해야 하는 입력을 `assert.rejects()`로 확인하는 일만큼, 정상 입력이 rejection 없이 완료되는지도 중요합니다.
예를 들어 저장, 발행, 동기화, 외부 API 호출 래퍼처럼 성공 시 반환 값보다 "비동기 작업이 실패하지 않았는가"가 핵심인 코드가 있습니다.
이때 Node.js `node:assert/strict`의 `assert.doesNotReject()`를 사용하면 Promise 기반 코드가 거부되지 않아야 한다는 의도를 테스트에 명확하게 남길 수 있습니다.

[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 `assert.doesNotReject(asyncFn)`가 전달한 비동기 함수 또는 Promise가 rejection을 만들지 않는지 검증하는 API라고 설명합니다.
동기 예외를 다루는 `assert.doesNotThrow()`와 달리, 이 메서드는 Promise rejection을 대상으로 합니다.
따라서 `async/await` 코드, Promise를 반환하는 SDK 래퍼, 비동기 초기화 함수의 정상 경로를 검증할 때 더 자연스럽습니다.

동기 예외 검증은 [Node.js assert.doesNotThrow 가이드](/development/blog/seo/2026/06/24/nodejs-assert-doesnotthrow-no-throw-test-guide.html)를 참고하세요.
실패해야 하는 비동기 경로는 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)와 연결됩니다.
테스트 구조 자체가 필요하다면 [Node.js built-in test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 보면 좋습니다.

## assert.doesNotReject가 필요한 상황

### H3. 정상 비동기 경로가 핵심 계약일 때 쓴다

대부분의 `async` 테스트에서는 함수가 rejection을 만들면 테스트 러너가 자동으로 실패를 보고합니다.
그래서 반환 값이 중요한 함수라면 `doesNotReject()`를 추가하기보다 결과를 직접 검증하는 편이 더 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function createDraft(input) {
  if (!input.title) {
    throw new Error('title is required');
  }

  return {
    id: 'draft_1',
    title: input.title.trim()
  };
}

test('createDraft returns a draft id', async () => {
  const draft = await createDraft({ title: 'Release note' });

  assert.equal(draft.id, 'draft_1');
  assert.equal(draft.title, 'Release note');
});
```

이 테스트는 `createDraft()`가 reject되면 그 즉시 실패합니다.
반환 값까지 검증하므로 `assert.doesNotReject()`가 없어도 정상 경로의 의미가 충분히 드러납니다.
결과가 중요한 테스트에서는 결과 assertion이 가장 좋은 문서입니다.

반대로 반환 값이 없거나, 반환 값보다 rejection이 없다는 계약이 더 중요한 함수라면 `doesNotReject()`가 유용합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function publishHealthCheck(event) {
  if (!event.serviceName) {
    throw new Error('serviceName is required');
  }

  await Promise.resolve();
}

test('publishHealthCheck accepts valid service event', async () => {
  await assert.doesNotReject(async () => {
    await publishHealthCheck({
      serviceName: 'billing-api',
      status: 'ok'
    });
  });
});
```

이 테스트의 관심사는 반환 값이 아니라 유효한 이벤트가 정상적으로 처리된다는 점입니다.
`assert.doesNotReject()`를 쓰면 "이 입력은 실패하면 안 된다"는 의도가 코드에 바로 보입니다.

### H3. Promise를 직접 넘길 수도 있다

`assert.doesNotReject()`에는 비동기 함수뿐 아니라 Promise 자체도 전달할 수 있습니다.
다만 테스트 안에서 지연 실행이 필요하거나 인자를 명확히 보여주고 싶다면 `async () => { ... }` 형태가 읽기 쉽습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function warmupCache() {
  return Promise.resolve('ready');
}

test('warmupCache completes without rejection', async () => {
  await assert.doesNotReject(warmupCache());
});
```

이 방식은 짧지만, Promise가 만들어지는 시점이 assertion 호출보다 앞설 수 있습니다.
실무 테스트에서는 아래처럼 함수 형태로 감싸면 호출 시점과 테스트 의도가 더 분명합니다.

```js
test('warmupCache completes without rejection', async () => {
  await assert.doesNotReject(async () => {
    await warmupCache();
  });
});
```

테스트 대상 함수에 인자가 많거나 준비 단계가 필요하다면 함수 형태가 유지보수에 유리합니다.
리뷰어가 "어떤 입력이 reject되면 안 되는지"를 한눈에 볼 수 있기 때문입니다.

## assert.rejects와 함께 설계하기

### H3. 성공 케이스와 실패 케이스를 짝으로 둔다

비동기 검증 함수는 정상 입력과 비정상 입력을 함께 테스트해야 경계가 선명해집니다.
`doesNotReject()`는 허용되는 입력을 보여주고, `rejects()`는 거부되는 입력을 보여줍니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function validateWebhookPayload(payload) {
  if (!payload.eventId) {
    throw new Error('eventId is required');
  }

  if (!payload.type) {
    throw new Error('type is required');
  }
}

test('validateWebhookPayload accepts minimal payload', async () => {
  await assert.doesNotReject(async () => {
    await validateWebhookPayload({
      eventId: 'evt_123',
      type: 'invoice.created'
    });
  });
});

test('validateWebhookPayload rejects missing eventId', async () => {
  await assert.rejects(
    () => validateWebhookPayload({ type: 'invoice.created' }),
    /eventId/
  );
});
```

이렇게 두면 테스트 파일만 읽어도 API의 입력 계약이 보입니다.
`eventId`와 `type`이 있으면 통과하고, `eventId`가 없으면 실패합니다.
문자열 전체를 고정하지 않고 `/eventId/`처럼 핵심 신호만 확인하면 문구 수정에도 덜 취약합니다.

### H3. 에러를 숨기기 위해 사용하지 않는다

`assert.doesNotReject()`는 실패를 무시하는 도구가 아닙니다.
비동기 함수가 reject되면 테스트를 실패시켜야 합니다.
따라서 아래처럼 catch에서 에러를 삼키는 방식은 피해야 합니다.

```js
test('bad pattern: do not swallow rejection', async () => {
  try {
    await validateWebhookPayload({});
  } catch (error) {
    // 실패가 사라져 테스트가 잘못 통과할 수 있다.
  }
});
```

이 코드는 실패해야 할 상황을 숨깁니다.
정상 입력이 reject되면 테스트가 깨져야 하고, 실패 입력이 reject되어야 한다면 `assert.rejects()`로 검증해야 합니다.
테스트에서 에러를 잡는다면 `assert.fail()`로 명시적으로 실패시키거나, 더 간단하게 assertion API에 맡기는 편이 좋습니다.

## 사용을 줄여야 하는 패턴

### H3. 결과 검증을 대체하지 않는다

`assert.doesNotReject()`는 Promise가 reject되지 않았다는 사실만 확인합니다.
함수가 잘못된 값을 반환해도 rejection이 없으면 통과합니다.
그래서 반환 값이나 상태 변화가 중요한 테스트에서는 반드시 별도 assertion을 둬야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function normalizeUser(input) {
  return {
    id: input.id,
    email: input.email.trim().toLowerCase()
  };
}

test('normalizeUser normalizes email', async () => {
  const user = await normalizeUser({
    id: 'user_1',
    email: ' Soo@Example.COM '
  });

  assert.equal(user.email, 'soo@example.com');
});
```

이 테스트에 `doesNotReject()`를 추가해도 얻는 정보는 거의 없습니다.
`await normalizeUser()`가 reject되면 테스트는 이미 실패하고, `assert.equal()`이 실제 결과까지 확인합니다.
비동기 함수가 값을 반환한다면 "실패하지 않는다"보다 "무엇을 반환한다"를 우선 검증하세요.

### H3. 동기 함수에는 assert.doesNotThrow를 쓴다

동기 함수가 예외를 던지지 않는지 확인하려면 `assert.doesNotReject()`가 아니라 `assert.doesNotThrow()`를 사용해야 합니다.
둘은 대상이 다릅니다.

```js
import assert from 'node:assert/strict';

function parseRetryCount(value) {
  const count = Number(value);

  if (!Number.isInteger(count) || count < 0) {
    throw new RangeError('retry count must be a non-negative integer');
  }

  return count;
}

assert.doesNotThrow(() => {
  parseRetryCount('3');
});
```

동기 코드를 Promise assertion으로 감싸면 테스트 의도가 흐려집니다.
반대로 Promise 기반 코드를 `doesNotThrow()`로 감싸면 비동기 rejection을 제대로 다루지 못할 수 있습니다.
동기 예외는 `doesNotThrow()`, 비동기 rejection은 `doesNotReject()`로 구분하는 습관이 중요합니다.

## 실무 적용 체크리스트

### H3. 정상 경로의 관심사를 먼저 고른다

`assert.doesNotReject()`를 쓰기 전에 테스트의 핵심 관심사가 무엇인지 먼저 정하세요.
아래 질문에 답하면 선택이 쉬워집니다.

- 반환 값이 중요한가?
- 상태 변화나 저장 결과를 확인해야 하는가?
- 성공 계약이 "rejection 없이 완료됨"으로 충분한가?
- 같은 입력 계약의 실패 케이스를 `assert.rejects()`로 함께 검증했는가?
- 테스트가 에러를 삼키지 않고 실패를 정확히 드러내는가?

반환 값이 중요하면 결과를 직접 검증하고, rejection이 없어야 한다는 계약 자체가 중요하면 `assert.doesNotReject()`를 사용하세요.
비동기 초기화, 이벤트 발행, 상태 동기화처럼 성공 시 조용히 완료되는 API에서는 특히 잘 맞습니다.

### H3. 테스트 이름에 기대 조건을 구체적으로 적는다

`doesNotReject()`는 코드만으로도 의도를 드러내지만, 테스트 이름까지 구체적이면 더 좋습니다.
`works`나 `does not fail`처럼 넓은 이름보다 어떤 입력이 허용되는지 적어야 나중에 테스트가 문서 역할을 합니다.

```js
async function syncSettings(settings) {
  if (!settings.serviceName) {
    throw new Error('serviceName is required');
  }
}

test('syncSettings accepts empty optional metadata', async () => {
  await assert.doesNotReject(async () => {
    await syncSettings({
      serviceName: 'billing-api',
      metadata: {}
    });
  });
});
```

이름에 `empty optional metadata`가 들어가면 이 테스트가 보호하는 계약이 분명해집니다.
나중에 누군가 빈 메타데이터를 거부하도록 구현을 바꾸면 테스트 실패가 곧 회귀 신호가 됩니다.
좋은 비동기 정상 경로 테스트는 단순히 "안 터진다"가 아니라 "이 조건에서는 실패하지 않아야 한다"를 남깁니다.

## 마무리

`assert.doesNotReject()`는 Promise 기반 코드의 정상 경로를 명확히 표현하는 도구입니다.
다만 모든 `async` 테스트에 습관적으로 붙일 필요는 없습니다.
반환 값이 중요하면 결과를 검증하고, 비동기 작업이 rejection 없이 끝나는 것 자체가 계약이라면 `doesNotReject()`를 사용하세요.

성공 케이스는 `doesNotReject()`, 실패 케이스는 `assert.rejects()`로 짝을 맞추면 비동기 API의 경계가 훨씬 선명해집니다.
테스트가 에러를 숨기지 않고, 어떤 입력이 허용되는지 드러내도록 작성하는 것이 핵심입니다.

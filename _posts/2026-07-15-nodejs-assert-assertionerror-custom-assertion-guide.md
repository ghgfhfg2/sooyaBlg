---
layout: post
title: "Node.js assert.AssertionError 가이드: 커스텀 assertion 실패를 읽기 좋게 만드는 법"
date: 2026-07-15 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-assertionerror-custom-assertion-guide
permalink: /development/blog/seo/2026/07/15/nodejs-assert-assertionerror-custom-assertion-guide.html
alternates:
  ko: /development/blog/seo/2026/07/15/nodejs-assert-assertionerror-custom-assertion-guide.html
  x_default: /development/blog/seo/2026/07/15/nodejs-assert-assertionerror-custom-assertion-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, assertionerror, testing, javascript, custom-assertion, ci, error-message]
description: "Node.js assert.AssertionError로 커스텀 assertion 실패를 표준 AssertionError처럼 만드는 방법을 정리합니다. actual, expected, operator, message 설계와 CI 로그에서 읽기 좋은 실패 메시지 기준을 예제로 설명합니다."
---

테스트가 실패했을 때 가장 먼저 보는 것은 실패 메시지입니다.
값은 틀렸는데 메시지가 모호하면 원인을 찾기 위해 테스트 코드와 제품 코드를 계속 왕복해야 합니다.
반대로 실패 메시지에 `actual`, `expected`, `operator`가 잘 담겨 있으면 CI 로그만 보고도 무엇이 어긋났는지 빠르게 좁힐 수 있습니다.

Node.js의 `assert.AssertionError`는 커스텀 assertion helper를 만들 때 실패를 표준 assertion 실패처럼 표현하게 해 주는 클래스입니다.
[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 `new assert.AssertionError(options)`가 `actual`, `expected`, `operator`, `message`, `stackStartFn` 같은 정보를 받을 수 있다고 설명합니다.
이 글에서는 커스텀 assertion을 언제 만들면 좋은지, `AssertionError`에 어떤 값을 넣어야 하는지, CI 로그에 민감정보를 남기지 않으면서도 충분히 읽기 좋은 실패를 만드는 기준을 정리합니다.

기본 동등성 검증은 [Node.js assert.strictEqual 가이드](/development/blog/seo/2026/06/29/nodejs-assert-strictequal-primitive-reference-test-guide.html), 객체 비교는 [Node.js assert.deepStrictEqual 가이드](/development/blog/seo/2026/06/26/nodejs-assert-deepstrictequal-object-comparison-test-guide.html), 의도적인 실패 표시는 [Node.js assert.fail 가이드](/development/blog/seo/2026/06/26/nodejs-assert-fail-explicit-test-failure-guide.html)와 함께 보면 좋습니다.

## AssertionError가 필요한 상황

### H3. 반복되는 검증을 helper로 묶을 때

테스트 코드에서 같은 조건을 여러 번 검증한다면 helper를 만들고 싶어집니다.
예를 들어 API 응답이 항상 `{ ok, code, data }` 모양을 갖는 프로젝트라면 매 테스트마다 `strictEqual`과 `deepStrictEqual`을 나열하는 것보다 도메인에 맞는 helper가 읽기 쉽습니다.

```js
import assert from 'node:assert/strict';

export function assertSuccessResponse(response, expectedCode) {
  assert.equal(response.ok, true);
  assert.equal(response.code, expectedCode);
}
```

이 helper도 동작은 합니다.
하지만 실패했을 때 메시지는 `true == false` 또는 `'USER_CREATED' == 'ORDER_CREATED'`처럼 낮은 수준의 비교만 보여 줍니다.
테스트를 읽지 않으면 "어떤 응답 계약이 깨졌는지"가 바로 보이지 않습니다.

이때 `AssertionError`를 직접 던지면 helper의 의도를 실패 메시지에 담을 수 있습니다.

```js
import assert from 'node:assert/strict';

export function assertSuccessResponse(response, expectedCode) {
  if (response.ok !== true || response.code !== expectedCode) {
    throw new assert.AssertionError({
      actual: {
        ok: response.ok,
        code: response.code
      },
      expected: {
        ok: true,
        code: expectedCode
      },
      operator: 'assertSuccessResponse',
      message: 'response should be a successful API result',
      stackStartFn: assertSuccessResponse
    });
  }
}
```

이제 실패는 일반 `Error`가 아니라 assertion 실패로 분류됩니다.
테스트 runner와 CI 로그도 `actual`, `expected`, `operator` 정보를 더 잘 보여 줄 수 있습니다.

### H3. 일반 Error보다 테스트 실패 의도를 명확히 한다

커스텀 helper에서 `throw new Error('invalid response')`를 던져도 테스트는 실패합니다.
하지만 일반 Error는 "테스트 대상 코드가 예외를 던진 것인지", "검증 helper가 실패를 표시한 것인지"를 구분하기 어렵게 만듭니다.

```js
export function assertPositiveAmount(amount) {
  if (amount <= 0) {
    throw new Error('amount should be positive');
  }
}
```

이 메시지는 짧지만, 실제 값과 기대 조건이 분리되어 있지 않습니다.
나중에 실패 로그만 봤을 때 `amount`가 `0`인지, `-1`인지, 문자열 `"100"`인지 알기 어렵습니다.

```js
import assert from 'node:assert/strict';

export function assertPositiveAmount(amount) {
  if (typeof amount !== 'number' || amount <= 0) {
    throw new assert.AssertionError({
      actual: amount,
      expected: 'number greater than 0',
      operator: 'assertPositiveAmount',
      message: 'amount should be a positive number',
      stackStartFn: assertPositiveAmount
    });
  }
}
```

좋은 assertion 실패는 사람이 읽는 문장과 도구가 읽을 수 있는 구조를 함께 갖습니다.
`message`는 문제를 설명하고, `actual`과 `expected`는 비교 대상을 보여 주며, `operator`는 어떤 검증 규칙이 깨졌는지 알려 줍니다.

## options 필드 설계하기

### H3. actual에는 관찰한 값을 넣는다

`actual`은 테스트가 실제로 본 값입니다.
여기에는 디버깅에 필요한 최소 정보만 넣는 편이 좋습니다.
응답 객체 전체를 그대로 넣으면 로그가 너무 길어지고, 토큰이나 이메일 같은 민감정보가 CI에 남을 수 있습니다.

```js
throw new assert.AssertionError({
  actual: {
    status: response.status,
    contentType: response.headers.get('content-type')
  },
  expected: {
    status: 201,
    contentType: 'application/json'
  },
  operator: 'assertCreatedJsonResponse',
  message: 'response should be a created JSON response',
  stackStartFn: assertCreatedJsonResponse
});
```

이 예시는 응답 본문이나 헤더 전체를 남기지 않습니다.
검증에 필요한 status와 content-type만 보여 줍니다.
실패 메시지는 충분히 유용하지만, 세션 쿠키나 Authorization 헤더가 노출될 위험은 줄어듭니다.

### H3. expected에는 규칙을 구체적으로 표현한다

`expected`는 반드시 실제 값과 같은 타입일 필요는 없습니다.
단순 비교라면 기대 값을 그대로 넣으면 되고, 범위나 조건이라면 사람이 이해할 수 있는 설명을 넣을 수 있습니다.

```js
export function assertLatencyBudget(durationMs, budgetMs) {
  if (durationMs > budgetMs) {
    throw new assert.AssertionError({
      actual: `${durationMs}ms`,
      expected: `<= ${budgetMs}ms`,
      operator: 'assertLatencyBudget',
      message: 'operation should stay within the latency budget',
      stackStartFn: assertLatencyBudget
    });
  }
}
```

여기서 기대 값은 단일 숫자가 아니라 조건입니다.
`expected: '<= 200ms'`처럼 규칙을 명확히 쓰면 실패 로그를 보는 사람이 테스트 helper 구현을 열어 보지 않아도 됩니다.

다만 너무 자유로운 문장만 남기면 자동 diff의 장점이 줄어듭니다.
객체 구조를 비교하는 helper라면 `actual`과 `expected`를 비슷한 shape로 맞추는 것이 좋습니다.

### H3. operator에는 helper 이름이나 비교 규칙을 넣는다

`operator`는 어떤 assertion이 실패했는지 보여 주는 짧은 이름입니다.
Node.js 기본 assertion에서는 `strictEqual`, `deepStrictEqual` 같은 값이 들어갑니다.
커스텀 helper에서는 함수 이름이나 팀에서 합의한 규칙 이름을 넣으면 좋습니다.

```js
throw new assert.AssertionError({
  actual: normalized,
  expected: expected,
  operator: 'assertNormalizedUser',
  message: 'normalized user should match the expected public shape',
  stackStartFn: assertNormalizedUser
});
```

`operator`를 매번 긴 문장으로 쓰기보다 검색 가능한 짧은 이름으로 유지하면 CI 로그에서 같은 실패 유형을 모아 보기 쉽습니다.
예를 들어 `assertPublicProfile`, `assertWebhookPayload`, `assertLatencyBudget`처럼 도메인 helper 이름을 그대로 쓰는 방식이 실용적입니다.

## stackStartFn으로 stack trace 줄이기

### H3. helper 내부 프레임을 숨긴다

커스텀 assertion helper가 실패하면 stack trace에는 helper 내부 줄이 먼저 보입니다.
하지만 테스트를 고치는 사람에게 더 중요한 위치는 helper를 호출한 테스트 코드입니다.
`stackStartFn`에 helper 함수를 넘기면 그 함수 위쪽부터 stack trace가 시작되도록 만들 수 있습니다.

```js
import assert from 'node:assert/strict';

export function assertHasPublicId(user) {
  if (!/^usr_[a-z0-9]+$/.test(user.publicId)) {
    throw new assert.AssertionError({
      actual: user.publicId,
      expected: 'usr_ prefix followed by lowercase letters or digits',
      operator: 'assertHasPublicId',
      message: 'user publicId should be safe to expose',
      stackStartFn: assertHasPublicId
    });
  }
}
```

이 설정은 실패 원인을 숨기는 것이 아니라 노이즈를 줄이는 역할입니다.
helper 내부 구현보다 "어떤 테스트 케이스가 잘못된 user를 만들었는가"가 더 빨리 보입니다.

### H3. 너무 많은 정보를 메시지에 넣지 않는다

`message`는 짧고 안정적인 문장으로 두는 편이 좋습니다.
동적인 데이터는 `actual`과 `expected`에 넣고, 메시지는 실패의 의미를 설명합니다.

```js
throw new assert.AssertionError({
  actual: sanitizedActual,
  expected: sanitizedExpected,
  operator: 'assertWebhookPayload',
  message: 'webhook payload should match the public contract',
  stackStartFn: assertWebhookPayload
});
```

이렇게 나누면 같은 유형의 실패를 검색하기 쉽고, 값 비교는 별도 필드에서 확인할 수 있습니다.
특히 CI 로그나 테스트 리포터가 assertion 정보를 구조화해서 보여 주는 환경에서는 이 구분이 더 중요합니다.

## CI 로그와 보안 기준

### H3. 민감정보는 helper에서 먼저 줄인다

테스트 실패 로그는 예상보다 오래 남습니다.
CI 시스템, 알림, PR 코멘트, 외부 품질 도구까지 복사될 수 있습니다.
그래서 `AssertionError`에 넣는 값은 helper 단계에서 먼저 정리해야 합니다.

```js
function publicUserShape(user) {
  return {
    id: user.id,
    role: user.role,
    hasEmail: Boolean(user.email),
    hasAccessToken: Boolean(user.accessToken)
  };
}

export function assertPublicUser(user, expected) {
  const actualShape = publicUserShape(user);

  if (!isSamePublicShape(actualShape, expected)) {
    throw new assert.AssertionError({
      actual: actualShape,
      expected,
      operator: 'assertPublicUser',
      message: 'user should match the public response contract',
      stackStartFn: assertPublicUser
    });
  }
}
```

이 패턴은 값 자체가 아니라 존재 여부나 공개 가능한 shape만 남깁니다.
실제 이메일, 토큰, 주소, 내부 URL, 결제 식별자 같은 값은 실패 로그에 들어가지 않게 막아야 합니다.

### H3. helper 테스트도 따로 둔다

커스텀 assertion helper는 테스트 인프라입니다.
잘못 만들면 제품 코드가 틀려도 통과하거나, 맞는 코드가 실패할 수 있습니다.
따라서 helper 자체도 작은 테스트를 두는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { assertPositiveAmount } from './assertions.js';

test('assertPositiveAmount throws AssertionError for invalid value', () => {
  assert.throws(
    () => assertPositiveAmount(0),
    {
      name: 'AssertionError',
      operator: 'assertPositiveAmount',
      expected: 'number greater than 0'
    }
  );
});
```

이 테스트는 helper가 일반 Error가 아니라 AssertionError를 던지는지 확인합니다.
또한 `operator`와 `expected`가 의도한 형태로 남는지 검증합니다.
helper가 많아질수록 이런 테스트가 실패 메시지 품질을 지켜 줍니다.

## AssertionError 사용 체크리스트

### H3. 커스텀 assertion은 실패 로그까지 설계한다

`AssertionError`는 단순히 에러 클래스를 바꾸는 도구가 아닙니다.
테스트 실패를 사람이 이해하기 좋은 구조로 정리하는 도구입니다.
도메인 helper를 만들 때는 성공 조건뿐 아니라 실패했을 때 어떤 정보를 보여 줄지도 함께 설계해야 합니다.

- 반복되는 검증을 helper로 묶을 만큼 의미가 있는가?
- `actual`에는 관찰한 최소 정보만 들어가는가?
- `expected`에는 기대 값이나 규칙이 명확히 표현되는가?
- `operator`는 검색 가능한 짧은 이름인가?
- `message`는 실패 의미를 설명하고, 동적 값은 필드로 분리했는가?
- 이메일, 토큰, 쿠키, 내부 URL 같은 민감정보가 로그에 남지 않는가?
- helper 자체를 검증하는 테스트가 있는가?

## FAQ

### H3. assert.fail()과 AssertionError는 어떻게 다른가요?

`assert.fail()`도 assertion 실패를 만들 수 있습니다.
단순히 "여기까지 오면 안 된다"를 표시할 때는 `assert.fail()`이 충분합니다.
반면 커스텀 helper에서 `actual`, `expected`, `operator`, `stackStartFn`을 세밀하게 설계하고 싶다면 `AssertionError`를 직접 만드는 편이 더 명확합니다.

### H3. actual에 원본 객체 전체를 넣어도 되나요?

작은 값이라면 괜찮을 수 있지만, API 응답이나 사용자 객체 전체를 넣는 것은 위험합니다.
로그가 지나치게 길어지고 민감정보가 노출될 수 있습니다.
검증에 필요한 공개 가능한 필드만 골라 넣는 습관이 안전합니다.

### H3. 커스텀 assertion helper는 많이 만들수록 좋나요?

아닙니다.
프로젝트 도메인에서 반복되는 계약을 더 읽기 쉽게 만들 때만 helper가 가치 있습니다.
한두 번 쓰는 검증이라면 기본 `assert.strictEqual()`이나 `assert.deepStrictEqual()`을 직접 쓰는 편이 더 단순합니다.

## 정리

`assert.AssertionError`는 커스텀 assertion 실패를 Node.js 표준 assertion처럼 표현하게 해 줍니다.
`actual`, `expected`, `operator`, `message`, `stackStartFn`을 의도적으로 설계하면 CI 로그만 보고도 실패 원인을 더 빨리 파악할 수 있습니다.

핵심은 실패 메시지를 많이 쓰는 것이 아니라 필요한 정보를 구조화하는 것입니다.
반복되는 도메인 검증에는 `AssertionError`를 쓰되, 민감정보를 줄이고 helper 자체도 테스트하면 장기적으로 읽기 쉬운 테스트 스위트를 만들 수 있습니다.

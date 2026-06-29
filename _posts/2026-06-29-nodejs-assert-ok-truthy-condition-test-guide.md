---
layout: post
title: "Node.js assert.ok 가이드: 조건 기반 테스트를 명확하게 쓰는 패턴"
date: 2026-06-29 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-ok-truthy-condition-test-guide
permalink: /development/blog/seo/2026/06/29/nodejs-assert-ok-truthy-condition-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/29/nodejs-assert-ok-truthy-condition-test-guide.html
  x_default: /development/blog/seo/2026/06/29/nodejs-assert-ok-truthy-condition-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, ok, test, assertion, javascript, backend]
description: "Node.js assert.ok()로 조건 기반 테스트를 작성하는 방법을 정리합니다. truthy 검증, 커스텀 메시지, strictEqual과의 구분, 실무에서 피해야 할 넓은 assertion 패턴을 예제로 설명합니다."
---

Node.js 테스트를 작성하다 보면 값 하나를 정확히 비교하기보다 "이 조건을 만족하는가"를 확인해야 할 때가 있습니다.
예를 들어 배열에 결과가 하나 이상 있는지, 날짜가 특정 범위 안에 있는지, 생성된 객체에 필수 속성이 있는지처럼 조건 자체가 테스트 의도인 경우입니다.
이때 `node:assert/strict`의 `assert.ok()`를 사용하면 불리언 조건을 짧게 검증할 수 있습니다.

[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 `assert.ok(value[, message])`가 값이 truthy인지 검사한다고 설명합니다.
핵심은 `ok()`를 아무 곳에나 쓰는 것이 아니라, 정확한 값 비교보다 조건 표현이 더 자연스러운 곳에만 쓰는 것입니다.
너무 넓은 조건은 실패 원인을 흐리게 만들 수 있으므로 `strictEqual()`, `deepStrictEqual()`, `match()`와 역할을 분리해야 합니다.

정확한 원시값을 고정해야 한다면 [Node.js assert.strictEqual 가이드](/development/blog/seo/2026/06/29/nodejs-assert-strictequal-primitive-reference-test-guide.html)를 먼저 참고하세요.
객체 내용 비교에는 [Node.js assert.deepStrictEqual 가이드](/development/blog/seo/2026/06/26/nodejs-assert-deepstrictequal-object-comparison-test-guide.html)가 더 적합합니다.
문자열 패턴 검증은 [Node.js assert.match 가이드](/development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html)와 함께 보면 좋습니다.

## assert.ok가 필요한 이유

### H3. 조건 자체가 테스트 의도일 때 사용한다

`assert.ok()`는 전달한 값이 truthy이면 통과하고 falsy이면 실패합니다.
그래서 테스트하려는 대상이 특정 값 하나가 아니라 조건이라면 코드가 짧고 직접적입니다.
검색 결과가 비어 있지 않아야 한다거나, 생성된 ID가 문자열이어야 한다는 식의 조건을 표현하기 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function searchArticles(articles, keyword) {
  return articles.filter((article) => article.title.includes(keyword));
}

test('searchArticles returns at least one matching article', () => {
  const results = searchArticles(
    [
      { title: 'Node.js assert.ok guide' },
      { title: 'Writing resilient tests' }
    ],
    'Node.js'
  );

  assert.ok(results.length > 0);
});
```

이 테스트의 핵심은 정확한 배열 길이가 아니라 검색 결과가 존재한다는 조건입니다.
만약 결과가 반드시 1개여야 한다면 `assert.strictEqual(results.length, 1)`이 더 좋은 선택입니다.
테스트 의도가 "조건"인지 "정확한 값"인지 먼저 구분하면 assertion 선택이 쉬워집니다.

### H3. truthy와 falsy 기준을 명확히 이해한다

`assert.ok()`는 JavaScript의 truthy/falsy 규칙을 따릅니다.
`false`, `0`, `''`, `null`, `undefined`, `NaN`은 실패하고, 빈 배열이나 빈 객체는 truthy라서 통과합니다.
이 차이를 모르면 빈 배열을 "결과 없음"으로 검증하려다 테스트가 잘못 통과할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function buildPayload(input) {
  return {
    id: input.id,
    tags: input.tags || []
  };
}

test('buildPayload creates a payload object', () => {
  const payload = buildPayload({ id: 'post_1' });

  assert.ok(payload);
  assert.ok(Array.isArray(payload.tags));
});
```

여기서 `assert.ok(payload)`는 객체가 생성됐는지 확인합니다.
하지만 `assert.ok(payload.tags)`는 빈 배열도 통과하므로 태그가 하나 이상 있어야 한다는 검증에는 맞지 않습니다.
그런 요구사항이라면 `assert.ok(payload.tags.length > 0)`처럼 조건을 명시해야 합니다.

## 좋은 메시지를 붙이는 방법

### H3. 실패했을 때 바로 이해되는 문장을 남긴다

조건식이 길어질수록 실패 메시지가 중요해집니다.
`assert.ok(condition, message)`의 두 번째 인자에 짧은 설명을 넣으면 테스트 실패 로그를 읽는 사람이 의도를 바로 파악할 수 있습니다.
메시지는 구현 설명보다 깨진 계약을 말하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function isValidSlug(slug) {
  return /^[a-z0-9]+(?:-[a-z0-9]+)*$/.test(slug);
}

test('generated slug uses lowercase words separated by hyphens', () => {
  const slug = 'nodejs-assert-ok-guide';

  assert.ok(
    isValidSlug(slug),
    'slug must contain lowercase letters, numbers, and hyphen separators'
  );
});
```

이 메시지는 정규식이 어떻게 생겼는지보다 slug가 지켜야 할 규칙을 설명합니다.
테스트가 실패했을 때 개발자는 입력값을 보고 어떤 계약이 깨졌는지 바로 알 수 있습니다.
복잡한 조건에는 메시지를 붙이고, 단순한 비교에는 더 구체적인 assertion을 쓰는 식으로 균형을 잡는 것이 좋습니다.

### H3. 여러 조건은 나눠서 검증한다

하나의 `assert.ok()` 안에 여러 조건을 `&&`로 묶으면 어느 부분이 실패했는지 알기 어렵습니다.
검증해야 할 계약이 여러 개라면 assertion을 나누거나, 더 구체적인 assertion API를 섞어 쓰는 편이 좋습니다.
테스트는 짧아야 하지만 실패 원인을 숨길 만큼 짧아져서는 안 됩니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function createSession(userId) {
  return {
    userId,
    token: `session-${userId}`,
    expiresAt: Date.now() + 60_000
  };
}

test('createSession returns an active session', () => {
  const session = createSession('user_1');

  assert.strictEqual(session.userId, 'user_1');
  assert.ok(session.token.startsWith('session-'), 'token must use the session prefix');
  assert.ok(session.expiresAt > Date.now(), 'session must expire in the future');
});
```

`userId`는 정확한 값 비교가 자연스럽기 때문에 `strictEqual()`을 사용했습니다.
토큰 접두사와 만료 시간은 조건 검증이므로 `ok()`가 잘 맞습니다.
이처럼 한 테스트 안에서도 assertion을 목적별로 섞으면 실패 메시지가 더 선명해집니다.

## strictEqual과 ok를 구분하는 기준

### H3. 불리언 반환값은 strictEqual이 더 명확할 수 있다

함수가 실제로 `true` 또는 `false`를 반환해야 한다면 `assert.ok()`보다 `assert.strictEqual(value, true)`가 더 명확한 경우가 많습니다.
`ok()`는 truthy 값을 모두 통과시키므로 문자열 `'yes'`, 숫자 `1`, 객체 같은 값도 통과합니다.
반환 타입까지 계약이라면 엄격한 비교를 사용해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function hasAdminRole(user) {
  return user.roles.includes('admin');
}

test('hasAdminRole returns boolean true for admins', () => {
  const result = hasAdminRole({ roles: ['member', 'admin'] });

  assert.strictEqual(result, true);
});
```

이 테스트는 결과가 truthy이면 되는 것이 아니라 불리언 `true`여야 한다는 계약을 담고 있습니다.
나중에 구현이 `'true'` 같은 문자열을 반환하면 `ok()`는 통과하지만 `strictEqual()`은 실패합니다.
검증 함수, 권한 함수, 플래그 계산 함수처럼 반환 타입이 중요한 곳에서는 엄격한 비교가 더 안전합니다.

### H3. 범위와 포함 여부는 ok가 읽기 쉽다

반대로 값이 특정 범위 안에 있거나 컬렉션에 포함되는지만 보면 되는 경우에는 `ok()`가 자연스럽습니다.
정확한 값 하나를 고정하면 테스트가 불필요하게 깨질 수 있는 영역입니다.
시간, 정렬 순서, 샘플 데이터 개수처럼 구현에 따라 조금 달라질 수 있는 결과는 조건으로 계약을 좁히는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function summarizeDurations(durations) {
  const total = durations.reduce((sum, value) => sum + value, 0);

  return {
    total,
    average: total / durations.length
  };
}

test('summarizeDurations returns an average in the expected range', () => {
  const summary = summarizeDurations([120, 180, 240]);

  assert.ok(summary.average >= 120);
  assert.ok(summary.average <= 240);
});
```

이 테스트는 평균이 입력 범위 안에 있어야 한다는 성질을 확인합니다.
물론 평균값이 정확히 `180`이어야 한다는 요구사항이라면 `strictEqual()`을 쓰는 편이 더 좋습니다.
조건 테스트는 구현을 너무 고정하지 않으면서 중요한 성질을 지키고 싶을 때 빛납니다.

## 실무에서 피해야 할 패턴

### H3. 값 비교를 ok로 숨기지 않는다

`assert.ok(result === expected)`처럼 쓰는 패턴은 동작은 하지만 실패 메시지가 덜 친절합니다.
Node.js assertion은 `strictEqual()`을 사용할 때 실제 값과 기대값의 차이를 더 잘 보여 줍니다.
값 비교라면 비교 전용 assertion을 쓰고, `ok()`는 조건 표현이 필요한 곳에 남겨두는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function normalizeEmail(email) {
  return email.trim().toLowerCase();
}

test('normalizeEmail trims spaces and lowercases text', () => {
  const normalized = normalizeEmail('  SOO@example.com ');

  assert.strictEqual(normalized, 'soo@example.com');
});
```

이 테스트를 `assert.ok(normalized === 'soo@example.com')`로 써도 통과는 합니다.
하지만 실패했을 때 어떤 값이 실제로 나왔는지 확인하기가 불편해집니다.
테스트 코드는 통과할 때보다 실패할 때 더 많이 읽히므로 실패 로그의 품질을 우선해야 합니다.

### H3. 넓은 존재 검증으로 계약을 대신하지 않는다

`assert.ok(response)` 같은 검증은 너무 넓을 수 있습니다.
응답 객체가 존재한다는 사실만 확인하면 상태 코드, 본문 형태, 필수 필드 누락 같은 회귀를 놓칩니다.
존재 확인은 시작점으로만 두고, 중요한 계약은 별도의 assertion으로 좁혀야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function buildResponse(data) {
  return {
    statusCode: 200,
    body: {
      ok: true,
      data
    }
  };
}

test('buildResponse returns a successful response shape', () => {
  const response = buildResponse({ id: 'article_1' });

  assert.ok(response);
  assert.strictEqual(response.statusCode, 200);
  assert.deepStrictEqual(response.body, {
    ok: true,
    data: { id: 'article_1' }
  });
});
```

첫 번째 assertion은 응답 객체의 존재만 확인합니다.
실제 계약은 상태 코드와 본문 구조에 있으므로 뒤의 assertion이 더 중요합니다.
`ok()` 하나로 테스트를 끝내고 싶어질 때는 "이 테스트가 깨져야 하는 변경은 무엇인가"를 다시 생각해 보세요.

## FAQ

### H3. assert.ok와 assert는 같은가요?

Node.js의 `assert(value)`는 기본적으로 `assert.ok(value)`와 같은 의미로 사용할 수 있습니다.
하지만 테스트 코드에서는 `assert.ok()`처럼 의도를 명시하는 편이 검색과 리뷰에 더 편합니다.
팀에서 하나의 스타일을 정해 일관되게 쓰는 것이 좋습니다.

### H3. 빈 배열을 검사할 때 assert.ok(results)를 써도 되나요?

배열이 만들어졌는지만 확인한다면 가능합니다.
하지만 결과가 하나 이상 있어야 한다면 빈 배열도 truthy라는 점 때문에 `assert.ok(results.length > 0)`처럼 써야 합니다.
배열의 정확한 내용까지 확인해야 한다면 `deepStrictEqual()`을 사용하세요.

### H3. ok에 너무 많은 조건이 들어가면 어떻게 나누나요?

정확한 값은 `strictEqual()`, 객체 구조는 `deepStrictEqual()`, 문자열 패턴은 `match()`, 범위나 포함 여부는 `ok()`로 분리합니다.
조건마다 assertion을 나누면 실패 위치가 선명해지고 테스트 이름도 더 구체적으로 다듬기 쉬워집니다.

## 마무리

`assert.ok()`는 조건 기반 테스트를 짧게 표현하는 도구입니다.
하지만 truthy 검증은 생각보다 넓기 때문에 정확한 값, 타입, 객체 구조를 확인해야 하는 곳에서는 더 구체적인 assertion을 선택해야 합니다.
좋은 기준은 단순합니다.
테스트 의도가 "이 조건을 만족하는가"라면 `ok()`를 쓰고, "정확히 이 값인가"라면 `strictEqual()`이나 다른 전용 assertion을 쓰세요.

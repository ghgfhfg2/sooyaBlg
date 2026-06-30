---
layout: post
title: "Node.js test runner skip todo only 가이드: 테스트 실행 범위를 안전하게 좁히는 법"
date: 2026-07-01 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-skip-todo-only-focused-tests-guide
permalink: /development/blog/seo/2026/07/01/nodejs-test-runner-skip-todo-only-focused-tests-guide.html
alternates:
  ko: /development/blog/seo/2026/07/01/nodejs-test-runner-skip-todo-only-focused-tests-guide.html
  x_default: /development/blog/seo/2026/07/01/nodejs-test-runner-skip-todo-only-focused-tests-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, skip, todo, only, testing, javascript]
description: "Node.js test runner의 skip, todo, only 옵션으로 테스트 실행 범위를 좁히고 미완성 테스트를 관리하는 방법을 정리합니다. t.skip(), t.todo(), --test-only, runOnly()의 차이와 CI에서 주의할 기준을 예제로 설명합니다."
---

테스트를 작성하다 보면 모든 테스트를 항상 실행하기 어려운 순간이 있습니다.
외부 서비스 준비가 끝나지 않았거나, 아직 버그가 남아 있거나, 특정 실패 케이스만 빠르게 좁혀 보고 싶을 수 있습니다.
Node.js 내장 test runner는 이런 상황을 위해 `skip`, `todo`, `only`를 제공합니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html)에 따르면 `skip`은 테스트를 실행하지 않도록 표시하고, `todo`는 미완성 또는 알려진 문제를 기록하되 실패로 처리하지 않는 용도입니다.
`only`는 `--test-only` 플래그와 함께 선택한 테스트만 실행할 때 사용합니다.
세 옵션은 편리하지만, 잘못 쓰면 실제 실패를 숨기거나 CI에서 테스트 범위를 의도치 않게 줄일 수 있습니다.

테스트 러너 기본 사용법은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 참고하세요.
병렬 실행과 격리 기준은 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html)와 이어집니다.
실패 테스트를 다시 실행하는 흐름은 [Node.js test runner 실패 테스트 재실행 가이드](/development/blog/seo/2026/05/26/nodejs-test-rerun-failures-flaky-test-debugging-guide.html)도 함께 보면 좋습니다.

## skip, todo, only를 구분해야 하는 이유

### H3. skip은 지금 실행하면 안 되는 테스트를 제외한다

`skip`은 테스트를 아예 실행하지 않는 선택입니다.
플랫폼 차이, 외부 의존성, 아직 준비되지 않은 테스트 환경처럼 실행 자체가 의미 없거나 위험할 때 사용합니다.
문자열을 넣으면 결과에 건너뛴 이유가 함께 남아 나중에 정리하기 쉽습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('exports report as a PDF file', { skip: 'PDF renderer is not installed in CI yet' }, async () => {
  const result = await exportReport({ format: 'pdf' });

  assert.strictEqual(result.contentType, 'application/pdf');
});
```

이 테스트는 실행되지 않습니다.
중요한 점은 `skip`이 실패를 고치는 도구가 아니라는 것입니다.
일시적으로 제외했다면 이슈 번호, 조건, 제거 기준을 남겨야 오래된 skip이 테스트 스위트에 묻히지 않습니다.

### H3. todo는 미완성 테스트를 기록하되 실패로 막지 않는다

`todo`는 테스트를 실행하지만, 실패해도 전체 실행의 실패로 처리하지 않는 표시입니다.
아직 구현되지 않은 기능이나 재현은 되었지만 수정 전인 버그를 테스트 코드로 먼저 남길 때 유용합니다.
테스트 이름만 남기는 것보다 실제 기대 동작을 코드로 표현할 수 있다는 장점이 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function normalizeUsername(value) {
  return value.trim();
}

test('normalizes unicode whitespace', { todo: 'support full unicode whitespace normalization' }, () => {
  assert.strictEqual(normalizeUsername('\u00a0sooya\u00a0'), 'sooya');
});
```

이 예제는 앞으로 맞춰야 할 동작을 테스트로 남깁니다.
`todo`가 많아지면 실제 품질 신호가 흐려질 수 있으므로, 릴리스 전에 todo 개수를 점검하는 습관이 필요합니다.
특히 오래된 todo는 구현 계획이 없는 메모가 되기 쉬워 주기적으로 제거하거나 실제 실패 테스트로 전환해야 합니다.

### H3. only는 디버깅 중 선택 실행에만 사용한다

`only`는 특정 테스트만 실행하고 싶을 때 사용합니다.
Node.js test runner에서는 보통 `node --test --test-only`처럼 플래그를 함께 사용해야 선택 실행이 적용됩니다.
로컬 디버깅에서는 매우 편하지만, 커밋에 남으면 CI가 전체 테스트를 돌지 않는 위험한 상태가 될 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('calculates discount for returning users', { only: true }, () => {
  const price = applyDiscount({ price: 10000, returningUser: true });

  assert.strictEqual(price, 9000);
});
```

```bash
node --test --test-only
```

이 조합은 `only: true`가 붙은 테스트를 빠르게 확인할 때 적합합니다.
다만 저장소에는 `only`가 남지 않도록 커밋 전 검색하거나 lint 규칙을 두는 편이 안전합니다.
`only`는 문제를 좁히는 도구이지, 테스트 선택 정책을 영구적으로 표현하는 도구가 아닙니다.

## 테스트 컨텍스트 메서드로 동적으로 제어하기

### H3. t.skip()은 조건부 제외에 쓴다

테스트 선언 시점에는 알 수 없고 실행 중에만 알 수 있는 조건이 있다면 테스트 컨텍스트의 `skip()`을 사용할 수 있습니다.
공식 문서에서도 `skip()`을 호출한 뒤 추가 로직이 있다면 함께 반환해야 한다고 설명합니다.
그렇지 않으면 skip 표시를 남긴 뒤에도 아래 코드가 계속 실행될 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('uses platform specific cache directory', (t) => {
  if (process.platform === 'win32') {
    t.skip('POSIX cache path assertion does not apply to Windows');
    return;
  }

  const cachePath = getCachePath('sooya-blog');

  assert.match(cachePath, /\/\.cache\/sooya-blog$/);
});
```

조건부 skip은 플랫폼별 테스트나 optional dependency 테스트에 잘 맞습니다.
하지만 단순히 불안정한 테스트를 숨기기 위해 사용하면 원인 분석이 늦어집니다.
flaky test라면 skip보다 재현 조건, 격리 문제, 시간 의존성을 먼저 확인하는 편이 좋습니다.

### H3. t.todo()는 실행 중 발견한 미완성 상태를 표시한다

`todo()`도 테스트 컨텍스트에서 호출할 수 있습니다.
테스트 본문 안에서 특정 기능 플래그나 런타임 조건에 따라 아직 완료되지 않은 상태를 표시할 때 사용할 수 있습니다.
다만 호출 뒤의 코드가 계속 실행된다는 점은 `skip()`과 마찬가지로 주의해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('supports compact JSON output', (t) => {
  if (!process.env.ENABLE_COMPACT_JSON) {
    t.todo('compact JSON output is behind a feature flag');
  }

  const output = renderJson({ ok: true }, { compact: true });

  assert.strictEqual(output, '{"ok":true}');
});
```

이 패턴은 기능 플래그 전환기에 쓸 수 있습니다.
기능이 정식으로 켜진 뒤에는 `todo()`를 제거하고 일반 테스트로 바꾸는 것이 좋습니다.
테스트가 실패해도 통과로 보이는 기간이 길어질수록 실제 회귀를 놓칠 가능성이 커집니다.

### H3. runOnly()는 하위 테스트 범위를 좁힌다

하위 테스트 안에서 선택 실행 범위를 제어해야 한다면 `t.runOnly(true)`를 사용할 수 있습니다.
이 메서드는 같은 테스트 컨텍스트의 하위 테스트 중 `only`가 붙은 항목만 실행하도록 바꿉니다.
중첩 테스트를 많이 쓰는 파일에서 특정 케이스만 좁혀 볼 때 유용합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('cart pricing rules', { only: true }, async (t) => {
  t.runOnly(true);

  await t.test('regular user discount', () => {
    assert.strictEqual(calculatePrice({ total: 10000, tier: 'regular' }), 10000);
  });

  await t.test('member discount', { only: true }, () => {
    assert.strictEqual(calculatePrice({ total: 10000, tier: 'member' }), 9000);
  });
});
```

중첩 테스트에서는 부모 테스트도 선택 실행 대상이 되어야 합니다.
부모가 실행되지 않으면 자식 테스트까지 도달할 수 없기 때문입니다.
디버깅이 끝나면 부모와 자식의 `only`를 모두 제거했는지 확인하세요.

## CI에서 안전하게 운영하는 기준

### H3. only가 남아 있으면 배포 전에 막는다

`only`는 로컬 디버깅에만 허용하는 것이 좋습니다.
커밋에 남아 있으면 전체 테스트 스위트가 아니라 일부 테스트만 통과한 상태로 배포될 수 있습니다.
간단한 검색 스크립트만 있어도 실수를 줄일 수 있습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "npm run check:no-only && node --test",
    "check:no-only": "node scripts/check-no-only.js"
  }
}
```

검색 스크립트는 프로젝트 스타일에 맞춰 구현하면 됩니다.
정규식만으로 충분한 작은 프로젝트도 있고, AST 파서로 `test(..., { only: true })`를 찾는 편이 더 안전한 프로젝트도 있습니다.
핵심은 CI에서 `only`가 남은 상태를 정상으로 보지 않는 것입니다.

### H3. skip과 todo는 이유와 만료 기준을 남긴다

`skip`과 `todo`는 모두 유용하지만, 이유 없이 남으면 테스트 신뢰도를 낮춥니다.
각 항목에는 왜 제외했는지, 언제 제거할지, 어떤 이슈와 연결되는지 남기는 편이 좋습니다.
특히 회귀 테스트를 `todo`로 오래 두면 실제 버그가 다시 들어와도 CI가 알려주지 못합니다.

```js
test('retries failed webhook delivery', {
  todo: 'tracked in ISSUE-142; turn into a normal test after retry queue lands'
}, async () => {
  const result = await deliverWebhook({ retry: true });

  assert.strictEqual(result.attempts, 3);
});
```

문자열 이유는 나중에 검색하고 정리하는 단서가 됩니다.
월별로 `skip`과 `todo` 목록을 확인하거나, 릴리스 전에 남은 항목을 검토하는 운영 루프를 두면 좋습니다.
테스트 코드는 제품 코드만큼 오래 남기 때문에, 임시 예외에도 정리 기준이 필요합니다.

## skip, todo, only 선택 체크리스트

### H3. 의도에 맞는 옵션을 고른다

세 옵션은 비슷해 보이지만 목적이 다릅니다.
실행하면 안 되는 테스트는 `skip`, 미완성 기대 동작은 `todo`, 로컬 디버깅 선택 실행은 `only`로 나눠 쓰면 테스트 결과를 해석하기 쉬워집니다.
무엇보다 `only`는 저장소에 남기지 않는다는 원칙을 분명히 해야 합니다.

- 환경이 맞지 않아 실행하지 않을 테스트인가? `skip`
- 아직 구현 전이지만 기대 동작을 기록해야 하는가? `todo`
- 지금 특정 테스트만 빠르게 디버깅하는 중인가? `only`
- skip이나 todo에 이유와 제거 기준이 적혀 있는가?
- CI에서 only 잔존을 막는 검사가 있는가?

## FAQ

### H3. skip과 todo 중 어느 쪽을 써야 하나요?

테스트를 실행하지 않아야 한다면 `skip`을 쓰고, 실행은 하되 아직 실패를 빌드 실패로 막지 않으려면 `todo`를 씁니다.
예를 들어 플랫폼이 달라 실행할 수 없는 테스트는 `skip`, 구현 예정 기능의 기대 동작은 `todo`가 자연스럽습니다.

### H3. only를 커밋해도 괜찮은 경우가 있나요?

일반적으로는 없습니다.
`only`는 로컬 디버깅용으로 보고, CI나 main 브랜치에는 남지 않게 막는 편이 안전합니다.
특정 테스트 묶음만 실행하는 영구 정책이 필요하다면 파일 구조, npm script, 테스트 태그 전략을 별도로 설계하는 것이 좋습니다.

### H3. t.skip()을 호출하면 바로 함수가 끝나나요?

아닙니다.
`t.skip()`은 skip 표시를 남기지만 테스트 함수의 나머지 코드를 자동으로 중단하지 않습니다.
추가 로직이 있다면 `return`을 함께 써서 의도치 않은 실행을 막아야 합니다.

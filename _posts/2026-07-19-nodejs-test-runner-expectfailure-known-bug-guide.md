---
layout: post
title: "Node.js test runner expectFailure 가이드: 알려진 버그를 안전하게 실패 기대 테스트로 관리하기"
date: 2026-07-19 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-expectfailure-known-bug-guide
permalink: /development/blog/seo/2026/07/19/nodejs-test-runner-expectfailure-known-bug-guide.html
alternates:
  ko: /development/blog/seo/2026/07/19/nodejs-test-runner-expectfailure-known-bug-guide.html
  x_default: /development/blog/seo/2026/07/19/nodejs-test-runner-expectfailure-known-bug-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, expectfailure, known-bug, regression-test, testing, javascript, ci]
description: "Node.js test runner의 expectFailure 옵션으로 알려진 버그와 미구현 동작을 안전하게 관리하는 방법을 정리합니다. todo, skip과의 차이, 에러 매칭, CI 운영 기준, 민감정보 점검까지 예제로 설명합니다."
---

알려진 버그를 테스트로 남겨야 할 때 가장 어려운 점은 "실패를 기록하되 CI 전체를 막지는 않는" 균형입니다.
그냥 실패 테스트를 커밋하면 배포 파이프라인이 막히고, `skip`으로 꺼 버리면 테스트가 아예 실행되지 않아 버그가 고쳐졌는지도 알기 어렵습니다.
`todo`는 미완성 상태를 표시하는 데 유용하지만, 특정 에러가 반드시 발생해야 한다는 계약을 표현하기에는 부족할 때가 있습니다.

Node.js test runner는 이런 경우를 위해 `expectFailure` 옵션을 제공합니다.
[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#expecting-tests-to-fail)에 따르면 `expectFailure`가 지정된 테스트는 예외를 던져야 통과하고, 예외를 던지지 않으면 실패합니다.
즉 "현재는 깨지는 것이 맞다"를 테스트 결과에 명시할 수 있습니다.
이 글에서는 `expectFailure`를 알려진 버그, 회귀 방지, 미구현 동작 관리에 적용하는 방법과 `skip`, `todo`와의 차이, CI에서 남겨야 할 기준을 정리합니다.
[Node.js test runner skip todo only 가이드](/development/blog/seo/2026/07/01/nodejs-test-runner-skip-todo-only-focused-tests-guide.html), [Node.js test runner snapshot testing 가이드](/development/blog/seo/2026/07/17/nodejs-test-runner-snapshot-testing-guide.html), [Node.js test runner reporter 가이드](/development/blog/seo/2026/07/18/nodejs-test-runner-reporters-spec-tap-dot-junit-guide.html)와 함께 보면 테스트 상태를 더 읽기 좋게 운영할 수 있습니다.

## expectFailure가 필요한 상황

### H3. 알려진 버그를 실행 가능한 문서로 남긴다

버그를 이슈 트래커에만 적어 두면 시간이 지나면서 재현 조건이 흐려집니다.
반대로 테스트로 남기면 입력, 기대 동작, 현재 실패 이유가 코드 안에 같이 보존됩니다.
`expectFailure`는 이 테스트가 아직 제품 품질을 보장하는 통과 테스트가 아니라, 고쳐야 할 실패를 추적하는 테스트라는 뜻을 결과에 남깁니다.

예를 들어 할인 금액 반올림 버그가 아직 고쳐지지 않았다면 아래처럼 작성할 수 있습니다.

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';

function calculateDiscountCents(priceCents, rate) {
  return Math.floor(priceCents * rate);
}

test('rounds discount to the nearest cent', {
  expectFailure: 'known rounding bug; remove after ISSUE-412 is fixed'
}, () => {
  assert.equal(calculateDiscountCents(1999, 0.15), 300);
});
```

현재 구현은 `299`를 반환하므로 assertion이 예외를 던지고, `expectFailure`가 붙은 테스트는 통과로 보고됩니다.
나중에 버그가 고쳐져 assertion이 더 이상 예외를 던지지 않으면 테스트가 실패합니다.
이 실패는 좋은 신호입니다.
이제 `expectFailure`를 제거하고 일반 회귀 테스트로 전환해야 한다는 뜻이기 때문입니다.

### H3. 실패가 사라졌을 때도 신호를 만든다

`skip`된 테스트는 실행되지 않습니다.
따라서 버그가 우연히 고쳐져도 CI가 알려 주지 못합니다.
`todo` 테스트는 실행되지만 실패가 빌드를 막지 않는다는 의미가 강합니다.

`expectFailure`는 한 단계 더 구체적입니다.
현재는 실패해야 하고, 성공해 버리면 상태가 바뀐 것입니다.
이 특징 때문에 알려진 버그 관리에 잘 맞습니다.

- 버그를 재현하는 입력이 있다.
- 현재 실패 원인이 명확하다.
- 고쳐지면 일반 테스트로 바꿔야 한다.
- 실패 메시지나 에러 코드까지 검증할 수 있다.
- CI 리포트에서 "예상 실패"와 실제 실패를 구분하고 싶다.

단순 메모가 아니라 실행 가능한 상태 전환 장치를 만들고 싶다면 `expectFailure`가 더 적합합니다.

## expectFailure 기본 사용법

### H3. 옵션 객체에 expectFailure를 넣는다

가장 단순한 사용법은 테스트 옵션에 `expectFailure: true`를 넣는 것입니다.

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';

function normalizeUsername(value) {
  return value.trim().toLowerCase();
}

test('normalizes full-width spaces in usernames', {
  expectFailure: true
}, () => {
  assert.equal(normalizeUsername('　SOOYA　'), 'sooya');
});
```

이 테스트는 현재 full-width space를 처리하지 못한다는 사실을 남깁니다.
하지만 `true`만 쓰면 왜 예상 실패인지 리포트에서 바로 알기 어렵습니다.
실무에서는 문자열 이유를 같이 남기는 편이 좋습니다.

```js
test('normalizes full-width spaces in usernames', {
  expectFailure: 'unicode whitespace normalization is tracked in ISSUE-518'
}, () => {
  assert.equal(normalizeUsername('　SOOYA　'), 'sooya');
});
```

이유에는 담당자 이름이나 실제 사용자 정보보다 추적 가능한 이슈 번호, 기능 플래그, 제거 조건을 넣는 것이 안전합니다.
테스트 리포트는 CI artifact, PR 댓글, 알림 메시지로 복사될 수 있으므로 개인정보나 내부 토큰처럼 보이는 값을 넣지 마세요.

### H3. it.expectFailure()로 의도를 더 짧게 표현한다

`it()` 스타일을 쓰는 코드베이스에서는 `it.expectFailure()` 형태를 사용할 수 있습니다.

```js
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';

describe('coupon validation', () => {
  it.expectFailure('rejects expired coupon across timezone boundary', () => {
    const result = validateCoupon({
      expiresAt: '2026-07-19T00:00:00+09:00',
      now: '2026-07-18T16:00:00Z'
    });

    assert.equal(result.valid, false);
  });
});
```

이 방식은 테스트 이름만 봐도 실패가 예상된다는 점이 드러납니다.
다만 팀에서 `test()` 스타일을 주로 쓴다면 옵션 객체 방식이 더 일관적일 수 있습니다.
핵심은 문법보다 상태입니다.
예상 실패인 이유와 제거 조건이 함께 보여야 운영 가능한 테스트가 됩니다.

## 에러를 구체적으로 매칭하기

### H3. RegExp로 실패 메시지를 좁힌다

예상 실패 테스트가 아무 에러나 받아들이면 실제로는 다른 문제가 생겼는데도 통과할 수 있습니다.
공식 문서 기준으로 `expectFailure`에는 `RegExp`, 함수, 객체, `Error` 같은 matcher를 넣을 수 있습니다.
가장 간단한 방식은 메시지를 정규식으로 좁히는 것입니다.

```js
test('rejects duplicate idempotency key', {
  expectFailure: /ERR_DUPLICATE_IDEMPOTENCY_KEY/
}, () => {
  submitPayment({
    idempotencyKey: 'fixture-key',
    amountCents: 1200
  });
});
```

이 테스트는 특정 에러 메시지나 코드가 나와야 예상 실패로 인정됩니다.
다른 예외가 발생하면 테스트가 실패하므로, 예상하지 못한 회귀를 더 빨리 볼 수 있습니다.

정규식은 너무 넓게 쓰지 않는 편이 좋습니다.
`/error/`, `/failed/`처럼 흔한 단어만 매칭하면 의도와 다른 실패도 통과할 수 있습니다.
가능하면 제품 코드의 안전한 에러 코드, public validation message, fixture 이름처럼 노출 가능한 값을 기준으로 매칭하세요.

### H3. label과 match를 함께 남긴다

실패 이유와 구체적인 매칭 조건을 동시에 남기고 싶다면 `{ label, match }` 형태가 유용합니다.

```js
test('preserves retry budget after partial outage', {
  expectFailure: {
    label: 'known retry budget drift; remove after ISSUE-621',
    match: { code: 'ERR_RETRY_BUDGET_DRIFT' }
  }
}, () => {
  const budget = createRetryBudget({ windowMs: 60_000, maxRetries: 3 });

  budget.recordFailure({ reason: 'timeout' });
  budget.recordFailure({ reason: 'timeout' });

  assert.equal(budget.remaining(), 1);
});
```

이 방식은 리포트에서 사람이 읽을 이유와 기계가 확인할 실패 조건을 분리합니다.
`label`에는 제거 기준을 담고, `match`에는 예상 에러의 구조를 담으면 좋습니다.

단, 에러 객체 전체를 snapshot처럼 비교하지는 마세요.
스택 트레이스, 절대 경로, 런타임 버전, 내부 파일명이 섞이면 실행 환경에 따라 흔들릴 수 있습니다.
예상 실패 매칭은 안정적인 public field 중심으로 좁히는 것이 좋습니다.

## skip, todo, expectFailure 선택 기준

### H3. 실행하면 안 되면 skip을 쓴다

`skip`은 테스트를 실행하지 않습니다.
외부 서비스가 없거나, 특정 플랫폼에서 실행할 수 없거나, 위험한 부작용이 있는 테스트라면 `skip`이 맞습니다.

```js
test('runs browser visual comparison', {
  skip: process.env.RUN_VISUAL_TESTS !== '1'
    ? 'set RUN_VISUAL_TESTS=1 for local visual checks'
    : false
}, async () => {
  await runVisualComparison();
});
```

이 경우에는 실행 자체가 조건부입니다.
현재 실패하는 버그를 추적하려는 목적이라면 `skip`보다 `expectFailure`를 먼저 검토하세요.

### H3. 미완성 기대 동작이면 todo를 쓴다

`todo`는 pending 구현이나 테스트 계획을 표시할 때 자연스럽습니다.
테스트 본문이 아직 충분히 구체적이지 않거나, 요구사항만 남겨 둔 단계라면 `todo`가 더 읽기 쉽습니다.

```js
test('exports billing report as parquet', {
  todo: 'add after analytics export format is finalized'
});
```

반대로 재현 가능한 입력과 현재 실패가 이미 있다면 `expectFailure`가 더 강한 신호를 줍니다.
`todo`는 "아직 할 일"이고, `expectFailure`는 "지금은 이 방식으로 실패해야 하는 알려진 상태"입니다.

### H3. 알려진 버그는 expectFailure로 좁힌다

선택 기준을 간단히 정리하면 다음과 같습니다.

- 실행하면 안 되는 테스트: `skip`
- 아직 구현이나 테스트 본문이 없는 항목: `todo`
- 실행해야 하며 현재 실패가 예상되는 버그: `expectFailure`
- 로컬에서 잠시 집중 실행할 테스트: `only`

공식 문서에서는 `skip`, `todo`, `expectFailure`가 함께 적용될 때 `skip`이나 `todo`가 우선한다고 설명합니다.
따라서 한 테스트에 여러 상태를 섞지 말고 하나의 의도만 남기는 편이 좋습니다.
리포트에서 상태가 겹치면 팀이 테스트 결과를 해석하기 어려워집니다.

## CI에서 expectFailure를 운영하는 법

### H3. 예상 실패 개수를 추적한다

`expectFailure`는 유용하지만 많아지면 위험합니다.
예상 실패가 계속 늘어난다는 것은 버그를 테스트로 기록하고만 있고 해결하지 못한다는 뜻일 수 있습니다.
CI에서는 reporter 결과나 테스트 로그에서 예상 실패 개수를 정기적으로 확인하세요.

처음에는 단순한 정책으로도 충분합니다.

- 새 `expectFailure`에는 이슈 번호나 제거 조건이 있어야 한다.
- 릴리스 전에는 예상 실패 목록을 확인한다.
- 같은 영역에 예상 실패가 누적되면 별도 개선 작업으로 묶는다.
- 버그가 고쳐져 테스트가 실패하면 `expectFailure`를 제거한다.
- 예상 실패를 `todo`나 `skip`으로 조용히 낮추지 않는다.

`expectFailure`의 목적은 실패를 숨기는 것이 아니라, 실패 상태를 더 정확하게 드러내는 것입니다.

### H3. 버전 조건을 명확히 둔다

`expectFailure`는 비교적 새 API입니다.
Node.js 공식 문서 기준으로 v25.5.0, v24.14.0에 추가된 기능이므로, 프로젝트의 Node.js 버전이 이 범위보다 낮다면 사용할 수 없습니다.

팀에서 여러 Node.js 버전을 지원한다면 `package.json`의 `engines`, CI matrix, 개발 환경 문서에 기준을 맞춰야 합니다.

```json
{
  "engines": {
    "node": ">=24.14.0"
  },
  "scripts": {
    "test": "node --test",
    "test:ci": "node --test --test-reporter=spec --test-reporter=junit --test-reporter-destination=stdout --test-reporter-destination=reports/node-test.xml"
  }
}
```

지원 버전을 올릴 수 없다면 같은 목적을 `todo`와 별도 이슈 추적으로 관리하고, 버전 업그레이드 후 `expectFailure`로 전환하는 흐름이 안전합니다.

### H3. 고쳐진 순간에는 일반 테스트로 바꾼다

`expectFailure`가 붙은 테스트가 예외를 던지지 않게 되면 테스트는 실패합니다.
이때는 제품 코드가 좋아졌을 가능성이 큽니다.
테스트를 다시 `todo`로 돌리거나 expectation을 흐리기보다, 아래 순서로 처리하세요.

1. 실제 버그가 의도대로 해결됐는지 확인한다.
2. `expectFailure` 옵션과 이유 문구를 제거한다.
3. 테스트 이름을 현재 기대 동작에 맞게 다듬는다.
4. 관련 이슈나 릴리스 노트에 회귀 테스트 추가를 남긴다.
5. CI에서 전체 테스트와 리포트를 확인한다.

이 과정을 거치면 예상 실패 테스트가 품질 부채로 쌓이지 않고, 고쳐진 버그를 막는 회귀 테스트로 자연스럽게 전환됩니다.

## 민감정보와 유해 표현 점검

### H3. 실패 이유에는 공개 가능한 정보만 남긴다

`expectFailure`의 label이나 message는 테스트 출력에 노출됩니다.
실제 고객 이메일, 주문 번호, 내부 호스트명, API token, 세션 cookie, 장애 대응 채널 이름 같은 값은 넣지 마세요.

좋은 이유 문구는 짧고 추적 가능합니다.

```js
expectFailure: 'known timezone boundary bug; remove after ISSUE-734'
```

피해야 할 문구는 너무 구체적이거나 민감한 값을 담습니다.

```js
expectFailure: 'fails for <redacted-user> in <redacted-tenant>'
```

테스트 fixture도 같은 기준을 적용합니다.
이메일은 `user@example.test`, 토큰은 `fixture-token`, 계정 ID는 `account_123`처럼 가짜 값으로 충분합니다.

### H3. 실패를 고정하되 차별적 표현은 남기지 않는다

테스트 이름은 오래 남습니다.
버그를 설명할 때 특정 사람, 국가, 장애, 성별, 집단을 조롱하거나 비하하는 표현을 넣으면 안 됩니다.
입력 데이터가 로케일, 언어, 시간대, 접근성 조건과 관련되더라도 표현은 기술적으로 유지하세요.

예를 들어 `rejects invalid locale fallback`은 괜찮지만, 특정 사용자 집단을 낮춰 부르는 이름은 테스트 이름에 들어가면 안 됩니다.
테스트 리포트는 코드 리뷰와 CI 화면에 반복 노출되므로 제품 문서만큼 조심해서 작성하는 편이 좋습니다.

## expectFailure 도입 체크리스트

### H3. 커밋 전에 확인할 항목

- 프로젝트 Node.js 버전이 `expectFailure`를 지원하는가?
- 예상 실패가 실제 알려진 버그나 미구현 동작을 재현하는가?
- 실패 이유에 이슈 번호나 제거 조건이 있는가?
- matcher가 너무 넓지 않은가?
- 민감정보, 개인정보, 내부 토큰, 실제 고객 데이터가 출력되지 않는가?
- `skip`, `todo`, `expectFailure`가 한 테스트에 섞이지 않았는가?
- 버그가 고쳐질 때 일반 테스트로 전환하는 기준이 보이는가?

이 기준을 지키면 `expectFailure`는 실패를 덮는 예외가 아니라, 고쳐야 할 상태를 선명하게 표시하는 도구가 됩니다.

### H3. expectFailure와 todo 중 무엇을 써야 하나요?

재현 가능한 실패가 있고, 현재 실패해야 하는 것이 테스트 의도라면 `expectFailure`를 쓰는 편이 좋습니다.
아직 구현 계획이나 테스트 항목만 남기는 단계라면 `todo`가 더 적합합니다.

### H3. expectFailure를 오래 남겨도 되나요?

가능하면 오래 남기지 않는 것이 좋습니다.
예상 실패가 오래 유지되면 알려진 버그가 품질 기준 안에 굳어질 수 있습니다.
이슈 번호, 제거 조건, 정기 점검 기준을 함께 두세요.

### H3. expectFailure가 통과 테스트로 바뀌면 어떻게 해야 하나요?

`expectFailure`가 붙은 테스트가 더 이상 예외를 던지지 않으면 테스트는 실패해야 합니다.
그때는 제품 코드가 고쳐졌는지 확인하고, `expectFailure` 옵션을 제거해 일반 회귀 테스트로 전환하세요.

## 마무리

Node.js test runner의 `expectFailure`는 알려진 버그를 테스트로 남기는 데 좋은 도구입니다.
`skip`처럼 실행을 끄지 않고, `todo`보다 더 구체적으로 "지금은 실패해야 하는 상태"를 표현할 수 있습니다.

다만 예상 실패는 어디까지나 임시 상태입니다.
이유와 제거 조건을 남기고, 에러 매칭을 좁히고, 민감정보가 리포트에 섞이지 않게 관리해야 합니다.
잘 운영하면 `expectFailure`는 테스트 실패를 숨기는 장치가 아니라, 버그가 고쳐지는 순간을 놓치지 않는 회귀 테스트 전환 장치가 됩니다.

## 관련 글

- [Node.js test runner skip todo only 가이드: 테스트 실행 범위를 안전하게 좁히는 법](/development/blog/seo/2026/07/01/nodejs-test-runner-skip-todo-only-focused-tests-guide.html)
- [Node.js test runner snapshot testing 가이드: UI 없이 값의 변화 추적하기](/development/blog/seo/2026/07/17/nodejs-test-runner-snapshot-testing-guide.html)
- [Node.js test runner reporter 가이드: spec, tap, dot, junit 출력 선택 기준](/development/blog/seo/2026/07/18/nodejs-test-runner-reporters-spec-tap-dot-junit-guide.html)

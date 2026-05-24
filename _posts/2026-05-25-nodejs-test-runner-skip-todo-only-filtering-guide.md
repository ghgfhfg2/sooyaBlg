---
layout: post
title: "Node.js test runner skip·todo·only 가이드: 테스트 실행 범위를 안전하게 제어하는 법"
date: 2026-05-25 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-skip-todo-only-filtering-guide
permalink: /development/blog/seo/2026/05/25/nodejs-test-runner-skip-todo-only-filtering-guide.html
alternates:
  ko: /development/blog/seo/2026/05/25/nodejs-test-runner-skip-todo-only-filtering-guide.html
  x_default: /development/blog/seo/2026/05/25/nodejs-test-runner-skip-todo-only-filtering-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, skip, todo, only, testing, javascript, ci]
description: "Node.js test runner에서 skip, todo, only, name pattern으로 테스트 실행 범위를 제어하는 방법을 정리했습니다. 로컬 디버깅과 CI 안전장치, 임시 테스트 관리 기준까지 실무 관점으로 설명합니다."
---

테스트가 많아질수록 “전체 테스트를 항상 돌릴 것인가”와 “지금 고치는 부분만 빠르게 확인할 것인가” 사이에서 균형이 필요합니다.
로컬에서는 실패한 테스트 하나만 빠르게 반복하고 싶고, CI에서는 실수로 임시 설정이 들어가 전체 검증이 줄어드는 일을 막아야 합니다.
Node.js 내장 test runner는 `skip`, `todo`, `only`, 이름 패턴 필터를 통해 이 실행 범위를 제어할 수 있습니다.

기본 실행법은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)에서 다뤘고, 테스트 구조화는 [Node.js test runner subtest 가이드](/development/blog/seo/2026/05/24/nodejs-test-runner-subtest-structure-guide.html)에서 정리했습니다.
이 글에서는 테스트를 임시로 제외하거나, 앞으로 작성할 테스트를 표시하거나, 특정 테스트만 실행할 때 지켜야 할 실무 기준을 정리합니다.

## 실행 범위 제어가 필요한 이유

### 빠른 피드백과 검증 누락은 다르다

개발 중에는 하나의 실패를 고치기 위해 같은 테스트를 여러 번 실행합니다.
이때 매번 전체 테스트를 돌리면 피드백이 느려지고 집중도도 떨어집니다.
`only`나 이름 패턴 필터는 이런 로컬 반복 작업에 유용합니다.

하지만 CI에 `only`가 남아 있거나, 실제로 고쳐야 할 테스트를 `skip`으로 숨기면 품질 문제가 됩니다.
실행 범위 제어는 속도를 높이기 위한 도구이지, 실패를 감추기 위한 도구가 아닙니다.
따라서 로컬 편의와 CI 안전장치를 함께 설계해야 합니다.

### 테스트 상태를 코드에 명확히 남긴다

테스트를 주석 처리하면 왜 빠졌는지, 언제 복구해야 하는지 알기 어렵습니다.
반면 `skip`과 `todo`는 테스트 리포트에 상태가 남습니다.
“현재 환경에서 실행하지 않음”, “아직 구현 예정”, “일시적으로 격리” 같은 의미를 코드와 리포트에 동시에 남길 수 있습니다.

CI 리포트를 읽기 좋게 만들고 싶다면 [Node.js test runner reporter와 JUnit 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-reporter-ci-junit-guide.html)처럼 리포터 설정과 함께 상태 표시 방식을 맞추는 것이 좋습니다.

## skip으로 조건부 제외하기

### 특정 테스트를 명시적으로 건너뛰기

`skip`은 지금 실행하면 안 되는 테스트를 명시적으로 제외할 때 씁니다.
테스트 옵션에 `skip`을 넣으면 테스트 본문은 실행되지 않고, 리포트에는 건너뛴 테스트로 표시됩니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function calculateDiscount(total) {
  if (total >= 100000) return 0.15;
  return 0;
}

test('applies high-value discount', { skip: 'policy is being reviewed' }, () => {
  assert.equal(calculateDiscount(120000), 0.15);
});
```

이 방식은 테스트를 삭제하거나 주석 처리하는 것보다 낫습니다.
다만 이유 없이 `skip: true`만 남기면 나중에 복구하기 어렵습니다.
가능하면 문자열로 사유를 남기고, 이슈 번호나 제거 기준을 함께 적는 편이 좋습니다.

### 환경에 따라 조건부로 건너뛰기

운영체제, 런타임 옵션, 외부 도구 설치 여부에 따라 일부 테스트를 실행하지 않아야 할 수 있습니다.
이때는 조건식을 `skip`에 넣을 수 있습니다.

```js
import assert from 'node:assert/strict';
import { platform } from 'node:process';
import test from 'node:test';

function normalizePathForWindows(input) {
  return input.replaceAll('/', '\\');
}

test('normalizes windows path separators', {
  skip: platform !== 'win32' ? 'windows-only behavior' : false,
}, () => {
  assert.equal(normalizePathForWindows('logs/app.txt'), 'logs\\app.txt');
});
```

조건부 `skip`은 크로스 플랫폼 테스트에서 유용합니다.
중요한 점은 “실행하지 않는 이유”가 코드만 봐도 이해되어야 한다는 것입니다.
단순히 실패하는 환경을 피하려고 조건을 늘리면 테스트 신뢰도가 떨어집니다.

## todo로 예정된 테스트 표시하기

### 아직 구현하지 않은 요구사항을 남기기

`todo`는 테스트를 아직 작성하지 않았거나, 기능 구현이 끝나지 않았음을 표시할 때 사용합니다.
테스트 계획을 코드에 남기되 현재 실패로 처리하지 않고 싶을 때 적합합니다.

```js
import test from 'node:test';

test('exports billing report as CSV', { todo: 'CSV export will be added after billing API is stable' });
```

이렇게 작성하면 테스트 목록에 예정 항목이 표시됩니다.
제품 요구사항이나 리팩터링 계획을 테스트 파일 가까이에 남길 수 있어, 나중에 빠진 검증을 찾기 쉽습니다.

### todo와 skip을 구분한다

`todo`와 `skip`은 비슷해 보이지만 의미가 다릅니다.
`todo`는 “아직 작성하거나 완성해야 할 테스트”에 가깝고, `skip`은 “테스트는 있지만 지금은 실행하지 않음”에 가깝습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

test('validates coupon expiration date', { todo: 'expiration rule is not implemented yet' }, () => {
  assert.fail('write this test when the rule is implemented');
});

test('sends receipt email', { skip: 'email sandbox is temporarily unavailable' }, () => {
  assert.fail('this would call the email sandbox');
});
```

구현 예정이면 `todo`, 환경 문제나 일시 격리라면 `skip`을 선택하는 식으로 팀 기준을 정해 두면 리포트 해석이 쉬워집니다.
오래된 `skip`은 기술 부채가 되기 쉽기 때문에 주기적으로 정리해야 합니다.

## only로 로컬 디버깅 범위 좁히기

### 특정 테스트만 실행하기

`only`는 로컬에서 특정 테스트만 집중적으로 실행할 때 유용합니다.
테스트 옵션에 `only: true`를 넣고 test runner를 실행하면 해당 테스트만 대상으로 삼을 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function parseAmount(input) {
  const value = Number(input);
  if (!Number.isFinite(value)) {
    throw new TypeError('amount must be numeric');
  }
  return value;
}

test('parseAmount rejects non numeric input', { only: true }, () => {
  assert.throws(() => parseAmount('abc'), TypeError);
});
```

`only`는 빠른 반복을 위해 좋은 도구입니다.
그러나 커밋에 남으면 CI가 전체 테스트를 돌리지 못할 수 있으므로 반드시 제거해야 합니다.
로컬 훅이나 CI 스크립트에서 `only: true` 문자열을 검사하는 안전장치를 두는 것도 좋은 방법입니다.

### subtest에서 only 사용하기

상위 테스트 아래의 특정 하위 테스트만 좁혀서 확인할 수도 있습니다.
이 방식은 큰 테스트 그룹 안에서 실패한 조건 하나를 고칠 때 편리합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function getShippingFee(region) {
  if (region === 'remote') return 6000;
  return 3000;
}

test('getShippingFee', async (t) => {
  await t.test('uses default fee', () => {
    assert.equal(getShippingFee('city'), 3000);
  });

  await t.test('uses remote area fee', { only: true }, () => {
    assert.equal(getShippingFee('remote'), 6000);
  });
});
```

하위 테스트를 좁혀 실행할 때도 상위 테스트의 준비 코드와 훅 범위를 확인해야 합니다.
훅 동작이 헷갈린다면 [Node.js test runner hooks 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html)를 기준으로 같은 레벨의 테스트에 어떤 준비 코드가 적용되는지 먼저 점검하는 편이 안전합니다.

## 이름 패턴으로 테스트 선택하기

### 코드 수정 없이 특정 테스트 실행하기

커밋에 `only`가 남는 위험을 줄이고 싶다면 이름 패턴 필터를 우선 고려할 수 있습니다.
Node.js test runner는 테스트 이름을 기준으로 실행 대상을 좁히는 옵션을 제공합니다.
예를 들어 특정 이름이 들어간 테스트만 실행하도록 스크립트를 구성할 수 있습니다.

```bash
node --test --test-name-pattern="parseAmount"
```

이 방식은 테스트 파일을 수정하지 않아도 되기 때문에 로컬 디버깅에 안전합니다.
테스트 이름을 일관되게 작성해 두면 함수명, 도메인 용어, 오류 코드로 필요한 테스트를 빠르게 찾을 수 있습니다.

### 이름 규칙을 검색 가능하게 만든다

이름 패턴 필터를 잘 쓰려면 테스트 이름 자체가 검색 가능해야 합니다.
“works”, “success”, “case 1”처럼 모호한 이름보다 테스트 대상과 조건을 함께 쓰는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function maskEmail(email) {
  const [name, domain] = email.split('@');
  return `${name[0]}***@${domain}`;
}

test('maskEmail keeps first character and domain', () => {
  assert.equal(maskEmail('admin@example.com'), 'a***@example.com');
});

test('maskEmail handles short local part', () => {
  assert.equal(maskEmail('a@example.com'), 'a***@example.com');
});
```

이름이 명확하면 패턴 실행뿐 아니라 CI 실패 로그도 좋아집니다.
테스트 이름을 문서처럼 쓰는 습관은 `skip`, `todo`, `only`보다 먼저 챙겨야 할 기본기입니다.

## CI에서 안전하게 운영하기

### only가 남아 있는지 검사한다

`only`는 로컬 전용으로 쓰는 것이 안전합니다.
CI에서는 전체 테스트가 실행되어야 하므로 `only: true`가 커밋에 남아 있는지 검사하는 단계를 둘 수 있습니다.
간단한 프로젝트라면 문자열 검색만으로도 실수를 많이 줄일 수 있습니다.

```bash
if grep -R "only: true" test src --include="*.js"; then
  echo "Remove test runner only flags before committing."
  exit 1
fi
```

프로젝트 구조에 따라 `test`, `src`, `_tests__` 같은 디렉터리를 조정하면 됩니다.
문자열 검색은 완벽한 정적 분석은 아니지만, 실수 방지용 안전망으로는 충분히 실용적입니다.

### skip과 todo는 리포트에서 추적한다

`skip`과 `todo`는 무조건 금지할 필요는 없습니다.
대신 개수가 갑자기 늘지 않는지, 오래된 항목이 방치되지 않는지 확인해야 합니다.
JUnit 리포트를 남기는 CI라면 추세를 추적하기 쉽습니다.

테스트 커버리지까지 같이 확인하려면 [Node.js test runner coverage 가이드](/development/blog/seo/2026/05/10/nodejs-test-runner-coverage-v8-guide.html)를 참고해 실행 범위와 커버리지 변화를 함께 보는 것이 좋습니다.
`skip`이 늘었는데 커버리지가 떨어졌다면 일시 조치가 장기 부채로 바뀌고 있을 가능성이 큽니다.

## 실무 기준: 언제 무엇을 쓸까

### skip을 쓰는 경우

`skip`은 테스트 본문이 있지만 현재 조건에서 실행할 수 없을 때 사용합니다.
예를 들어 특정 OS 전용 기능, 외부 도구가 필요한 통합 테스트, 일시적으로 불안정한 외부 샌드박스 테스트가 여기에 해당합니다.
단, “계속 실패해서”라는 이유만으로 `skip`을 남기는 것은 피해야 합니다.

```js
import test from 'node:test';

test('runs image optimizer integration test', {
  skip: process.env.RUN_IMAGE_TESTS !== '1' ? 'set RUN_IMAGE_TESTS=1 to run locally' : false,
}, () => {
  // integration test body
});
```

환경 변수를 이용하면 기본 CI에서는 빠르게 실행하고, 별도 잡이나 로컬에서는 무거운 테스트를 켤 수 있습니다.
이때도 실제 검증이 영원히 꺼져 있지 않도록 별도 일정이나 워크플로를 두는 것이 좋습니다.

### todo를 쓰는 경우

`todo`는 앞으로 작성할 테스트를 잊지 않기 위한 표시입니다.
새 요구사항을 먼저 테스트 목록에 남겨 두거나, 리팩터링 중 아직 검증하지 못한 경계를 표시할 때 쓸 수 있습니다.

```js
import test from 'node:test';

test('prevents duplicate refund requests', {
  todo: 'add after refund idempotency key is introduced',
});
```

좋은 `todo`는 다음 행동이 보입니다.
“나중에 작성”보다는 “idempotency key 도입 후 작성”처럼 조건을 남기는 편이 관리하기 쉽습니다.

### only를 쓰는 경우

`only`는 로컬에서만 짧게 사용합니다.
커밋 전에는 제거해야 하고, 가능하면 이름 패턴 필터로 대체하는 습관을 들이는 편이 안전합니다.

```bash
node --test --test-name-pattern="refund"
```

테스트 파일을 수정하지 않고도 필요한 범위만 실행할 수 있다면 `only`보다 이 방식이 낫습니다.
그래도 하위 테스트 하나를 빠르게 고칠 때는 `only`가 편리하므로, 제거를 보장하는 검사를 함께 두면 됩니다.

## 비동기 테스트와 함께 쓸 때 주의할 점

### skip된 테스트는 비동기 검증도 실행하지 않는다

`skip`된 테스트 본문은 실행되지 않습니다.
따라서 내부의 `await assert.rejects(...)` 같은 검증도 수행되지 않습니다.
실패 경로 테스트를 임시로 건너뛰면 실제 예외 동작이 깨져도 알 수 없습니다.

비동기 예외 검증 패턴은 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)를 기준으로 작성하되, skip을 넣었다면 반드시 복구 기준을 남겨야 합니다.
특히 네트워크 타임아웃, 취소, 권한 실패처럼 운영 장애와 연결되는 테스트는 오래 건너뛰지 않는 것이 좋습니다.

### mock과 only를 함께 쓸 때 격리를 확인한다

`only`로 특정 테스트만 실행하면 평소 전체 실행에서 드러나던 순서 의존 문제가 보이지 않을 수 있습니다.
mock 상태를 테스트마다 초기화하지 않으면 단독 실행과 전체 실행 결과가 달라질 수 있습니다.

의존성 호출을 검증하는 테스트라면 [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html)처럼 테스트마다 mock을 새로 만들고, 하위 테스트 사이에 상태를 공유하지 않는 구조가 안전합니다.
`only`는 디버깅 도구일 뿐, 격리 설계를 대신하지 못합니다.

## 발행 전 체크리스트처럼 쓰는 규칙

### 커밋 전 확인 항목

테스트 실행 범위 제어를 사용하는 프로젝트라면 커밋 전 아래 항목을 확인하면 좋습니다.

- `only: true`가 남아 있지 않은가?
- `skip`에는 구체적인 사유와 제거 기준이 있는가?
- `todo`는 다음 행동이 드러나는 문장인가?
- 이름 패턴으로 실행할 수 있을 만큼 테스트 이름이 명확한가?
- CI에서는 전체 테스트와 필요한 리포터가 실행되는가?

이 기준을 문서로만 두기보다 npm script, CI step, PR 체크리스트에 녹여 두면 실수가 줄어듭니다.
테스트 도구는 개발자를 감시하기 위한 장치가 아니라, 반복 실수를 자동으로 막아 주는 안전망이어야 합니다.

### 추천 운영 방식

로컬에서는 `--test-name-pattern`을 기본 디버깅 도구로 사용하고, 정말 필요할 때만 `only`를 짧게 씁니다.
`skip`은 환경 조건이나 일시 격리에만 쓰고, 오래된 항목은 정리합니다.
`todo`는 요구사항 추적용으로 쓰되 구현 시점이 지나면 실제 테스트로 바꿉니다.

이렇게 역할을 나누면 테스트 리포트가 단순한 성공·실패 목록이 아니라 프로젝트의 현재 품질 상태를 보여주는 문서가 됩니다.
테스트를 더 잘 나누고 싶다면 [Node.js test runner subtest 가이드](/development/blog/seo/2026/05/24/nodejs-test-runner-subtest-structure-guide.html)와 함께 적용해 보세요.

## FAQ

### skip과 todo 중 무엇을 먼저 써야 하나요?

테스트 본문이 있고 현재 조건에서만 실행하지 않을 이유가 있다면 `skip`이 맞습니다.
아직 구현 예정이거나 테스트 계획만 남기는 단계라면 `todo`가 더 적합합니다.
둘을 구분하면 리포트에서 “일시 제외”와 “작성 예정”을 분리해서 볼 수 있습니다.

### only를 CI에서 써도 되나요?

일반적인 CI에서는 권장하지 않습니다.
CI는 전체 검증을 수행해야 하므로 `only`가 남아 있으면 중요한 테스트가 빠질 수 있습니다.
CI에서 일부 테스트만 실행해야 한다면 파일 경로, 태그 역할의 이름 규칙, `--test-name-pattern`, 별도 npm script처럼 명시적인 실행 전략을 쓰는 편이 안전합니다.

### 이름 패턴 필터가 only보다 나은가요?

커밋에 임시 변경이 남지 않는다는 점에서는 이름 패턴 필터가 더 안전합니다.
다만 하위 테스트 하나를 짧게 고치는 상황에서는 `only`가 편할 수 있습니다.
팀 기준은 “로컬에서만 사용하고 커밋 전 자동 검사로 제거한다” 정도가 현실적입니다.

### skip이 많아지면 어떻게 관리해야 하나요?

먼저 사유 없는 `skip`을 없애고, 오래된 항목을 정리해야 합니다.
가능하면 `skip` 사유에 제거 조건을 쓰고, CI 리포트에서 skip 개수를 추적하세요.
커버리지 하락과 함께 skip이 늘어난다면 실패를 숨기고 있는 신호일 수 있습니다.

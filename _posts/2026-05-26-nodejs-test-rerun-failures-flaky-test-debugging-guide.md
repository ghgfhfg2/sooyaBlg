---
layout: post
title: "Node.js test runner 실패 테스트 재실행 가이드: --test-rerun-failures로 디버깅 시간을 줄이는 법"
date: 2026-05-26 08:00:00 +0900
lang: ko
translation_key: nodejs-test-rerun-failures-flaky-test-debugging-guide
permalink: /development/blog/seo/2026/05/26/nodejs-test-rerun-failures-flaky-test-debugging-guide.html
alternates:
  ko: /development/blog/seo/2026/05/26/nodejs-test-rerun-failures-flaky-test-debugging-guide.html
  x_default: /development/blog/seo/2026/05/26/nodejs-test-rerun-failures-flaky-test-debugging-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, test-rerun-failures, flaky-test, testing, ci, javascript]
description: "Node.js 내장 test runner의 --test-rerun-failures 옵션으로 실패한 테스트만 다시 실행하는 방법을 정리했습니다. 상태 파일 관리, flaky test 디버깅, CI 적용 기준, 주의할 점까지 실무 관점으로 설명합니다."
---

테스트가 한두 개일 때는 전체 테스트를 다시 실행해도 부담이 작습니다.
하지만 테스트 파일이 늘고 통합 테스트, 파일 시스템 테스트, 네트워크 mock 테스트가 섞이면 실패 하나를 확인하기 위해 전체 스위트를 반복 실행하는 시간이 꽤 커집니다.
Node.js 내장 test runner의 `--test-rerun-failures` 옵션은 이전 실행 상태를 파일로 저장하고, 이미 통과한 테스트를 제외한 실패 테스트만 다시 실행할 수 있게 해 줍니다.

기본 실행법은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)에서, 저장할 때마다 빠르게 피드백을 받는 방법은 [Node.js test runner watch 모드 가이드](/development/blog/seo/2026/05/25/nodejs-test-runner-watch-mode-fast-feedback-guide.html)에서 다뤘습니다.
이 글에서는 실패 테스트 재실행을 로컬 디버깅과 CI 재시도 흐름에 안전하게 적용하는 방법을 정리합니다.

## --test-rerun-failures가 해결하는 문제

### 전체 테스트 반복 실행 비용을 줄인다

`--test-rerun-failures`는 지정한 상태 파일에 테스트 실행 결과를 저장합니다.
처음 실행에서는 전체 테스트를 돌리고, 다음 실행에서는 상태 파일을 기준으로 아직 통과하지 못한 테스트만 다시 실행합니다.

```bash
node --test --test-rerun-failures .test-rerun-state.json
```

이 옵션은 “실패한 테스트를 고칠 때마다 전체 테스트를 계속 다시 돌리는 상황”에서 특히 유용합니다.
예를 들어 300개 테스트 중 2개가 실패했다면, 수정 후에는 그 2개를 중심으로 빠르게 확인할 수 있습니다.

### flaky test를 숨기는 도구가 아니다

중요한 점은 이 옵션이 실패를 감추기 위한 장치가 아니라는 것입니다.
실패 테스트를 다시 실행해 통과 여부를 빠르게 확인하게 해 주지만, 불안정한 테스트를 “가끔 통과하니 괜찮다”로 처리하면 안 됩니다.

flaky test는 보통 다음 원인에서 나옵니다.

- 시간 의존 로직이 고정되어 있지 않음
- 테스트 간 전역 상태가 공유됨
- mock이나 fixture가 정리되지 않음
- 파일 시스템, 타이머, 네트워크 의존성이 불안정함
- 테스트 실행 순서에 암묵적으로 의존함

실패 재실행은 원인을 좁히는 도구로 쓰고, 최종적으로는 테스트가 한 번의 전체 실행에서도 안정적으로 통과하도록 고쳐야 합니다.

## 기본 사용법

### 상태 파일 경로를 지정한다

가장 단순한 형태는 상태 파일 경로만 넘기는 방식입니다.

```bash
node --test --test-rerun-failures .test-rerun-state.json
```

상태 파일이 없다면 test runner가 새로 만듭니다.
상태 파일에는 어떤 테스트가 어느 실행에서 통과했는지에 대한 정보가 저장됩니다.
테스트 이름뿐 아니라 파일 위치 정보도 기준에 포함되므로, 테스트 파일을 크게 옮기거나 순서를 바꾸면 이전 상태와 맞지 않을 수 있습니다.

### npm scripts로 고정한다

팀에서 반복해서 쓸 명령은 `package.json` scripts로 고정하는 편이 좋습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:failed": "node --test --test-rerun-failures .test-rerun-state.json"
  }
}
```

`npm test`는 항상 전체 검증용으로 남겨 두고, `npm run test:failed`는 디버깅용으로 분리합니다.
이렇게 하면 재실행 옵션이 CI의 기본 검증 범위를 줄이는 실수도 막을 수 있습니다.

### 상태 파일은 보통 커밋하지 않는다

`.test-rerun-state.json`은 실행 환경에 따라 달라지는 임시 상태입니다.
대부분의 프로젝트에서는 Git에 올리지 않는 것이 안전합니다.

```gitignore
.test-rerun-state.json
```

상태 파일은 로컬 디버깅 힌트에 가깝습니다.
다른 개발자나 CI 환경에서 같은 상태를 공유할 필요가 없다면 저장소에 포함하지 않는 편이 관리가 쉽습니다.

## 실전 예제로 흐름 이해하기

### 실패하는 테스트를 만든다

다음 예시는 할인율 계산 함수와 테스트입니다.
의도적으로 경계값 처리가 틀린 상태라고 가정해 보겠습니다.

```js
export function applyDiscount(price, rate) {
  if (rate < 0 || rate > 1) {
    throw new RangeError('rate must be between 0 and 1');
  }

  return Math.round(price * (1 - rate));
}
```

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { applyDiscount } from '../src/apply-discount.js';

test('applies ten percent discount', () => {
  assert.equal(applyDiscount(10000, 0.1), 9000);
});

test('allows zero percent discount', () => {
  assert.equal(applyDiscount(10000, 0), 10000);
});

test('rejects rate greater than one', () => {
  assert.throws(() => applyDiscount(10000, 1.1), RangeError);
});
```

이처럼 작은 단위 테스트는 실패 위치를 빠르게 찾기 좋습니다.
비동기 실패를 검증해야 한다면 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)를 함께 참고하면 좋습니다.

### 실패만 다시 실행하며 수정한다

처음에는 전체 테스트가 실행되고 상태 파일이 만들어집니다.

```bash
npm run test:failed
```

실패 원인을 수정한 뒤 같은 명령을 다시 실행하면, 이미 통과한 테스트를 제외하고 아직 통과하지 못한 테스트를 중심으로 확인할 수 있습니다.
디버깅 중에는 이 흐름이 빠릅니다.

다만 수정이 끝난 뒤에는 반드시 전체 테스트를 다시 실행해야 합니다.

```bash
npm test
```

실패 테스트만 통과했다고 해서 전체 회귀 검증이 끝난 것은 아닙니다.
수정한 코드가 다른 영역의 테스트를 깨뜨렸을 수 있기 때문입니다.

## watch 모드와 함께 쓸 때의 기준

### 빠른 피드백 목적이 다르다

`--watch`와 `--test-rerun-failures`는 둘 다 피드백 속도를 높이지만 역할이 다릅니다.

- `--watch`: 파일 저장 시 자동 재실행
- `--test-rerun-failures`: 이전 실행에서 통과하지 못한 테스트 중심 재실행

개발 중 한 파일을 계속 고치는 상황이라면 watch 모드가 자연스럽습니다.
여러 테스트가 실패한 뒤 하나씩 고쳐 나가는 상황이라면 실패 재실행 옵션이 더 맞습니다.

```bash
node --test --watch
```

```bash
node --test --test-rerun-failures .test-rerun-state.json
```

테스트 이름으로 범위를 좁히는 방법은 [Node.js test runner skip·todo·only 가이드](/development/blog/seo/2026/05/25/nodejs-test-runner-skip-todo-only-filtering-guide.html)에서 다룬 `--test-name-pattern`과 함께 비교해 두면 팀 규칙을 만들기 쉽습니다.

### 마지막 관문은 항상 전체 실행이다

빠른 도구는 개발 속도를 높여 주지만, 배포 전 신뢰도를 보장하는 최종 관문은 전체 실행입니다.
따라서 scripts 이름도 의도를 드러내는 편이 좋습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:watch": "node --test --watch",
    "test:failed": "node --test --test-rerun-failures .test-rerun-state.json"
  }
}
```

커밋 전 체크리스트는 단순합니다.

1. `npm run test:failed`로 실패 원인을 빠르게 고친다.
2. `.test-rerun-state.json` 같은 임시 파일이 커밋 대상에 없는지 확인한다.
3. `npm test`로 전체 테스트를 한 번 실행한다.
4. CI에서도 필터 없는 전체 테스트가 실행되는지 확인한다.

## CI에서 사용할 때의 주의점

### 기본 CI 검증을 줄이지 않는다

CI에서 `--test-rerun-failures`를 쓰고 싶다면 기본 검증을 대체하지 않도록 조심해야 합니다.
잘못 구성하면 이전 상태 파일 때문에 일부 테스트가 실행되지 않아 회귀를 놓칠 수 있습니다.

권장 흐름은 다음과 같습니다.

```yaml
- name: Run full test suite
  run: npm test
```

실패 재실행은 기본 CI 단계보다 로컬 개발 또는 별도 디버깅 job에 더 잘 어울립니다.
CI에서 자동 재시도를 넣는 경우에도 “최종 성공으로 처리할 기준”을 명확히 해야 합니다.

### flaky test 관찰용으로 분리한다

간헐적으로 실패하는 테스트를 추적해야 한다면 별도 job에서 상태 파일을 만들어 관찰할 수 있습니다.
하지만 본 배포 판단은 전체 테스트 결과를 기준으로 두는 편이 안전합니다.

```yaml
- name: Debug failed tests only
  if: failure()
  run: node --test --test-rerun-failures .test-rerun-state.json
```

이런 단계는 로그를 더 모으기 위한 보조 수단입니다.
재실행에서 통과했다고 해서 첫 실패가 사라지는 것은 아닙니다.
실패 로그, 실행 시간, 테스트 이름, 관련 커밋을 함께 남겨 원인을 추적해야 합니다.

## 상태 파일을 안전하게 관리하기

### 테스트 위치 변경에 민감하다는 점을 이해한다

상태 파일은 테스트 파일 경로와 위치 정보를 기준으로 테스트를 식별합니다.
테스트를 함수 안에서 동적으로 만들거나, 반복문으로 여러 테스트를 생성하거나, 파일 순서를 크게 바꾸면 이전 상태와 매칭이 어긋날 수 있습니다.

따라서 실패 재실행을 자주 쓰는 테스트는 가능한 한 정적인 구조가 좋습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

const cases = [
  { name: 'empty string', input: '', expected: false },
  { name: 'valid email', input: 'user@example.com', expected: true },
];

function isEmail(value) {
  return /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(value);
}

for (const item of cases) {
  test(`validates email: ${item.name}`, () => {
    assert.equal(isEmail(item.input), item.expected);
  });
}
```

반복문으로 테스트를 만들 수는 있지만, 케이스 순서와 이름이 자주 바뀌면 상태 파일의 효용이 줄어듭니다.
장기적으로는 테스트 이름을 명확히 유지하고, 실패 분석이 쉬운 구조를 우선하는 편이 좋습니다.

### 필요할 때 상태 파일을 초기화한다

테스트 구조를 크게 바꿨거나 상태가 이상하다고 느껴지면 상태 파일을 지우고 다시 시작하면 됩니다.

```bash
rm -f .test-rerun-state.json
node --test --test-rerun-failures .test-rerun-state.json
```

상태 파일은 소스 코드가 아니라 캐시입니다.
이상한 결과가 나오면 오래 붙잡고 분석하기보다 삭제 후 전체 실행으로 기준점을 다시 잡는 편이 빠릅니다.

## flaky test를 고치는 실무 패턴

### 시간 의존 테스트는 기준 시간을 주입한다

현재 시간에 직접 의존하는 테스트는 실행 시점에 따라 결과가 달라질 수 있습니다.
함수 내부에서 `Date.now()`를 바로 호출하기보다 기준 시간을 인자로 받게 만들면 테스트가 안정됩니다.

```js
export function isExpired(expiresAt, now = Date.now()) {
  return expiresAt <= now;
}
```

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { isExpired } from '../src/session.js';

test('detects expired session with fixed time', () => {
  const now = Date.parse('2026-05-26T00:00:00Z');
  const expiresAt = Date.parse('2026-05-25T23:59:59Z');

  assert.equal(isExpired(expiresAt, now), true);
});
```

이런 방식은 실패 재실행 여부와 관계없이 테스트 품질을 높입니다.
타이머 기반 비동기 흐름을 다룬다면 [Node.js timers/promises scheduler.yield 가이드](/development/blog/seo/2026/05/20/nodejs-timers-promises-scheduler-yield-event-loop-guide.html)도 함께 참고할 수 있습니다.

### mock과 임시 파일은 테스트마다 정리한다

이전 테스트의 mock 상태가 다음 테스트에 남으면 실패 재실행에서 더 헷갈리는 결과가 나옵니다.
테스트가 다시 실행될 때마다 같은 초기 상태에서 시작하도록 정리 코드를 넣어야 합니다.

```js
import assert from 'node:assert/strict';
import { afterEach, mock, test } from 'node:test';

const metrics = {
  count(name) {
    return `count:${name}`;
  },
};

afterEach(() => {
  mock.restoreAll();
});

test('records signup metric', () => {
  const countMock = mock.method(metrics, 'count', () => 'ok');

  assert.equal(metrics.count('signup'), 'ok');
  assert.equal(countMock.mock.callCount(), 1);
});
```

mock 사용 패턴은 [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html)와 이어서 보면 좋습니다.
파일 시스템 테스트라면 [Node.js mkdtempDisposable 가이드](/development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html)처럼 임시 리소스 생명주기를 분리하는 습관이 중요합니다.

## 팀 규칙으로 정리하기

### 어떤 상황에서 쓰는지 문서화한다

실패 재실행 옵션은 팀에 도입할 때 사용 기준을 짧게라도 문서화하는 것이 좋습니다.

```md
## Test commands

- `npm test`: full test suite. Required before commit and in CI.
- `npm run test:watch`: local save-and-rerun feedback.
- `npm run test:failed`: rerun tests that have not passed in the local state file.

Do not commit `.test-rerun-state.json`.
A passing `test:failed` run does not replace `npm test`.
```

이 정도 규칙만 있어도 “빠른 명령”과 “최종 검증”을 혼동할 가능성이 줄어듭니다.

### 실패 로그를 남기는 습관을 만든다

flaky test를 고치려면 재실행 결과보다 첫 실패 로그가 더 중요할 때가 많습니다.
실패한 테스트 이름, 에러 메시지, 실행 환경, 관련 변경 사항을 이슈나 PR 코멘트에 남기면 나중에 원인을 찾기 쉽습니다.

좋은 기록은 다음 정보를 포함합니다.

- 실패 테스트 이름
- 첫 실패 에러 메시지
- 재실행 결과
- 로컬/CI 환경 차이
- 최근 바뀐 코드 영역
- 임시 조치와 최종 수정 방향

이런 기록은 테스트 안정성뿐 아니라 기술 블로그나 운영 문서의 신뢰도도 높여 줍니다.
재현 가능한 예시와 민감정보 마스킹 기준은 [로그 예시 비식별화 가이드](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)와도 연결됩니다.

## 자주 묻는 질문

### --test-rerun-failures만 통과하면 커밋해도 되나요?

아니요.
`--test-rerun-failures`는 디버깅 속도를 높이는 보조 명령입니다.
커밋 전에는 필터 없는 `npm test` 또는 프로젝트의 전체 검증 명령을 실행해야 합니다.

### 상태 파일을 저장소에 올려도 되나요?

대부분은 올리지 않는 것이 좋습니다.
상태 파일은 실행 환경과 테스트 위치에 따라 달라지는 임시 데이터입니다.
`.gitignore`에 추가하고 로컬 캐시처럼 관리하는 편이 안전합니다.

### watch 모드와 실패 재실행 중 무엇을 먼저 써야 하나요?

한 파일을 계속 고치는 중이면 watch 모드가 편합니다.
여러 실패를 하나씩 고치는 중이면 `--test-rerun-failures`가 유용합니다.
어느 쪽이든 마지막에는 전체 테스트를 실행해야 합니다.

### flaky test가 재실행에서 통과하면 괜찮은 건가요?

괜찮지 않습니다.
재실행 통과는 “간헐적 실패일 가능성”을 보여 줄 뿐입니다.
시간, 전역 상태, mock 정리, 파일 시스템, 테스트 순서 의존성을 확인해 한 번의 전체 실행에서도 안정적으로 통과하도록 고쳐야 합니다.

## 마무리

`--test-rerun-failures`는 Node.js 내장 test runner를 더 실무적으로 쓰게 해 주는 옵션입니다.
실패한 테스트를 고치는 동안 전체 스위트를 반복 실행하는 비용을 줄이고, flaky test를 추적할 때 필요한 피드백을 빠르게 얻을 수 있습니다.

다만 이 옵션은 최종 검증을 대체하지 않습니다.
상태 파일은 커밋하지 않고, 로컬 디버깅용 scripts로 분리하며, 커밋 전과 CI에서는 항상 전체 테스트를 실행하는 원칙을 지키는 것이 좋습니다.
이 기준만 세워 두면 빠른 피드백과 테스트 신뢰도를 함께 가져갈 수 있습니다.

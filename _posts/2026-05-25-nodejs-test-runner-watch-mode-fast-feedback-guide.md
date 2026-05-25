---
layout: post
title: "Node.js test runner watch 모드 가이드: 저장할 때마다 빠르게 테스트 피드백 받는 법"
date: 2026-05-25 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-watch-mode-fast-feedback-guide
permalink: /development/blog/seo/2026/05/25/nodejs-test-runner-watch-mode-fast-feedback-guide.html
alternates:
  ko: /development/blog/seo/2026/05/25/nodejs-test-runner-watch-mode-fast-feedback-guide.html
  x_default: /development/blog/seo/2026/05/25/nodejs-test-runner-watch-mode-fast-feedback-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, watch-mode, testing, javascript, ci, developer-experience]
description: "Node.js 내장 test runner의 watch 모드를 로컬 개발에 적용하는 방법을 정리했습니다. npm scripts, 테스트 파일 구조, 이름 필터, mock·fixture 정리, CI 분리 기준까지 실무 관점으로 설명합니다."
---

테스트는 작성하는 것만큼 자주 실행하는 방식도 중요합니다.
코드를 조금 바꿀 때마다 수동으로 명령어를 다시 입력하면 흐름이 끊기고, 반대로 테스트를 늦게 돌리면 작은 문제가 커진 뒤에야 발견됩니다.
Node.js 내장 test runner의 `--watch` 모드는 저장할 때마다 관련 테스트를 다시 실행해 로컬 개발 피드백을 짧게 만들어 줍니다.

기본 실행법은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)에서 다뤘고, 특정 테스트만 고르는 기준은 [Node.js test runner skip·todo·only 가이드](/development/blog/seo/2026/05/25/nodejs-test-runner-skip-todo-only-filtering-guide.html)에서 정리했습니다.
이 글에서는 `--watch`를 실무 프로젝트에 넣을 때 필요한 npm scripts, 테스트 파일 구조, mock 정리, CI 분리 기준을 함께 살펴봅니다.

## watch 모드가 필요한 순간

### 저장 후 즉시 실패를 확인한다

`--watch`는 테스트 파일이나 의존 파일이 바뀌었을 때 테스트를 다시 실행합니다.
작은 리팩터링, 유효성 검사 함수 수정, 에러 처리 로직 변경처럼 반복 확인이 많은 작업에서 특히 유용합니다.

```bash
node --test --watch
```

이 명령은 현재 프로젝트에서 test runner가 찾을 수 있는 테스트를 감시합니다.
코드를 저장하면 터미널에 최신 결과가 다시 출력되므로, 브라우저 새로고침처럼 테스트도 자연스럽게 반복할 수 있습니다.

### 전체 테스트와 빠른 테스트를 구분한다

watch 모드는 로컬 피드백을 빠르게 만드는 도구입니다.
CI에서 전체 검증을 대신하는 도구가 아닙니다.
따라서 프로젝트에는 보통 두 가지 실행 경로가 필요합니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:watch": "node --test --watch"
  }
}
```

`npm test`는 CI와 배포 전 확인에 쓰고, `npm run test:watch`는 개발 중 반복 작업에 씁니다.
이렇게 역할을 분리하면 로컬 편의 때문에 CI 검증 범위가 줄어드는 일을 막을 수 있습니다.

## 테스트 파일 구조를 watch 친화적으로 만들기

### 소스와 테스트의 의존 관계를 단순하게 둔다

watch 모드의 효과는 테스트 파일이 적절히 나뉘어 있을 때 커집니다.
하나의 큰 테스트 파일에 모든 케이스가 몰려 있으면 작은 변경에도 많은 테스트가 반복 실행되어 피드백이 느려집니다.
기능 단위로 테스트 파일을 나누면 실패 위치도 더 빨리 찾을 수 있습니다.

```text
src/
  price.js
  coupon.js
test/
  price.test.js
  coupon.test.js
```

예를 들어 가격 계산 로직과 쿠폰 검증 로직을 분리하면, 실패 메시지를 읽을 때 어느 영역의 문제인지 바로 좁힐 수 있습니다.
테스트 구조를 더 읽기 좋게 나누고 싶다면 [Node.js test runner subtest 가이드](/development/blog/seo/2026/05/24/nodejs-test-runner-subtest-structure-guide.html)를 함께 참고하면 좋습니다.

### 테스트 대상 함수는 작게 노출한다

watch 모드에서는 작은 단위의 테스트가 자주 실행됩니다.
테스트하려는 함수가 거대한 모듈 초기화나 외부 연결을 항상 동반하면 저장할 때마다 불필요한 시간이 늘어납니다.
가능하면 순수 계산, 포맷 변환, 입력 검증처럼 빠르게 검증할 수 있는 단위를 분리해 두는 편이 좋습니다.

```js
export function normalizeEmail(input) {
  return input.trim().toLowerCase();
}
```

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { normalizeEmail } from '../src/email.js';

test('normalizes email address', () => {
  assert.equal(normalizeEmail('  User@Example.COM '), 'user@example.com');
});
```

이런 테스트는 빠르고 안정적입니다.
watch 모드에서 자주 실행되어도 부담이 작고, 실패했을 때 원인을 좁히기 쉽습니다.

## 이름 필터와 함께 쓰기

### 작업 중인 영역만 반복 실행한다

프로젝트가 커지면 watch 모드에서도 전체 테스트 재실행이 부담이 될 수 있습니다.
이때 `--test-name-pattern`을 함께 쓰면 현재 고치는 테스트 이름만 반복 실행할 수 있습니다.

```bash
node --test --watch --test-name-pattern="coupon"
```

테스트 이름은 검색 가능한 문장처럼 작성하는 것이 좋습니다.
`works`나 `case 1`보다 `applies coupon discount`처럼 기능과 기대 결과가 드러나는 이름이 watch 모드에서 훨씬 유용합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function applyCoupon(total, coupon) {
  if (coupon === 'WELCOME10') return total * 0.9;
  return total;
}

test('coupon applies welcome discount', () => {
  assert.equal(applyCoupon(10000, 'WELCOME10'), 9000);
});

test('coupon keeps total when code is unknown', () => {
  assert.equal(applyCoupon(10000, 'UNKNOWN'), 10000);
});
```

이름 필터는 로컬 집중을 위한 도구입니다.
커밋 전에는 반드시 `npm test`처럼 필터 없는 전체 테스트도 한 번 실행해야 합니다.

### only는 임시로만 사용한다

`test.only`는 빠르게 하나의 테스트만 확인할 때 편하지만, 커밋에 남으면 검증 누락으로 이어질 수 있습니다.
watch 모드에서는 `--test-name-pattern`이나 파일 경로 지정으로도 대부분의 집중 실행을 해결할 수 있습니다.

```bash
node --test --watch test/coupon.test.js
```

특정 파일만 감시하는 방식은 의도가 명확합니다.
코드 안에 임시 상태가 남지 않으므로 리뷰와 CI에서 더 안전합니다.
`only`, `skip`, `todo`의 차이는 [Node.js test runner skip·todo·only 가이드](/development/blog/seo/2026/05/25/nodejs-test-runner-skip-todo-only-filtering-guide.html)를 기준으로 정리해 두면 팀 규칙을 만들기 쉽습니다.

## mock과 fixture를 안전하게 정리하기

### 테스트 간 상태를 남기지 않는다

watch 모드는 같은 프로세스나 반복 실행 흐름에서 테스트를 자주 돌리게 만듭니다.
이때 전역 상태, mock 호출 기록, 임시 파일이 제대로 정리되지 않으면 처음에는 통과하다가 두 번째 실행에서 실패하는 테스트가 생길 수 있습니다.

```js
import assert from 'node:assert/strict';
import { afterEach, mock, test } from 'node:test';

const notifier = {
  send(message) {
    return `sent:${message}`;
  },
};

afterEach(() => {
  mock.restoreAll();
});

test('sends welcome notification', () => {
  const sendMock = mock.method(notifier, 'send', () => 'queued');

  assert.equal(notifier.send('welcome'), 'queued');
  assert.equal(sendMock.mock.callCount(), 1);
});
```

`afterEach`에서 mock을 복구하면 다음 테스트가 이전 테스트의 가짜 구현에 영향을 받지 않습니다.
mock 함수 사용법은 [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html)와 함께 보면 더 자연스럽게 이어집니다.

### 임시 파일은 테스트마다 새로 만든다

파일 시스템을 다루는 테스트는 watch 모드에서 특히 조심해야 합니다.
이전 실행에서 만든 파일이 남아 있으면 다음 실행의 전제 조건이 달라질 수 있습니다.
가능하면 테스트마다 임시 디렉터리를 만들고 끝나면 정리합니다.

```js
import assert from 'node:assert/strict';
import { mkdir, readFile, rm, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';
import test from 'node:test';

async function saveJson(filePath, value) {
  await writeFile(filePath, JSON.stringify(value), 'utf8');
}

test('writes json file', async (t) => {
  const dir = join(tmpdir(), `watch-mode-${Date.now()}`);
  await mkdir(dir, { recursive: true });

  t.after(async () => {
    await rm(dir, { recursive: true, force: true });
  });

  const filePath = join(dir, 'result.json');
  await saveJson(filePath, { ok: true });

  assert.equal(await readFile(filePath, 'utf8'), '{"ok":true}');
});
```

임시 리소스를 테스트 안에서 만들고 테스트 안에서 지우면 반복 실행의 신뢰도가 올라갑니다.
파일 복사나 임시 디렉터리 정리는 [Node.js fs.cp 재귀 복사 가이드](/development/blog/seo/2026/05/17/nodejs-fs-cp-recursive-copy-guide.html)와 [Node.js mkdtempDisposable 가이드](/development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html)를 참고할 수 있습니다.

## watch 모드와 비동기 테스트

### 완료 신호를 명확히 한다

watch 모드에서 비동기 테스트가 불안정하면 저장할 때마다 결과가 달라져 개발 흐름을 방해합니다.
Promise를 반환하거나 `async` 함수를 사용해 테스트 완료 시점을 명확히 해야 합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

async function fetchProfile(userId) {
  return { id: userId, name: 'Soo' };
}

test('loads user profile', async () => {
  const profile = await fetchProfile('user-1');

  assert.deepEqual(profile, { id: 'user-1', name: 'Soo' });
});
```

타이머, 이벤트, 네트워크 호출처럼 완료 조건이 늦게 오는 테스트는 타임아웃과 취소 전략도 함께 설계해야 합니다.
취소 패턴은 [Node.js fetch AbortSignal timeout 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)가 실무 예시로 연결됩니다.

### 실패해야 하는 비동기 경로도 검증한다

watch 모드는 성공 경로만 빠르게 보는 용도로 쓰기 쉽습니다.
하지만 입력 검증, 권한 오류, 외부 API 실패처럼 실패해야 정상인 경로도 함께 반복 확인해야 합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

async function requirePositiveNumber(value) {
  if (value <= 0) {
    throw new RangeError('value must be positive');
  }
  return value;
}

test('rejects non-positive number', async () => {
  await assert.rejects(
    () => requirePositiveNumber(0),
    { name: 'RangeError', message: 'value must be positive' }
  );
});
```

비동기 예외를 확인할 때는 `try/catch`로 대충 감싸기보다 `assert.rejects`를 쓰는 편이 의도가 선명합니다.
자세한 패턴은 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)에서 다뤘습니다.

## CI와 로컬 명령을 분리하는 기준

### CI에서는 watch를 쓰지 않는다

CI는 한 번 실행하고 종료되어야 합니다.
따라서 `--watch`는 로컬 전용 script에만 둡니다.
CI 설정에서는 항상 종료 가능한 명령을 사용해야 합니다.

```yaml
name: test

on:
  pull_request:
  push:
    branches: [main]

jobs:
  node-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci
      - run: npm test
```

이 구조에서는 개발자가 로컬에서 `npm run test:watch`를 쓰더라도 CI는 항상 `npm test`로 전체 검증을 수행합니다.
리포트가 필요하다면 [Node.js test runner reporter와 JUnit 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-reporter-ci-junit-guide.html)를 붙여 테스트 결과를 CI 화면에서 읽기 좋게 만들 수 있습니다.

### 커밋 전 체크 명령을 따로 둔다

watch 모드는 개발 중 반복 실행에 최적화되어 있습니다.
커밋 전에는 필터 없는 전체 테스트와 정적 검사를 묶은 명령을 따로 두는 편이 안전합니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:watch": "node --test --watch",
    "check": "npm test"
  }
}
```

프로젝트에 린트, 타입 체크, 빌드가 있다면 `check`에 함께 넣으면 됩니다.
핵심은 watch 모드 결과만 보고 커밋하지 않는 것입니다.

## 실무 적용 체크리스트

### watch 모드 도입 전 확인할 것

- `test`와 `test:watch` script를 분리했는가?
- 테스트 파일이 기능 단위로 나뉘어 있는가?
- 테스트 이름이 필터링하기 좋게 구체적인가?
- mock, 타이머, 임시 파일이 테스트마다 정리되는가?
- CI에서는 종료 가능한 `npm test`만 실행하는가?

### 팀 규칙으로 남기면 좋은 것

- `test.only`는 커밋 금지
- `skip`은 사유 문자열이나 이슈 링크와 함께 사용
- watch 모드는 로컬 전용
- 커밋 전에는 전체 테스트 실행
- 느린 통합 테스트는 별도 script로 분리

이 규칙을 문서화해 두면 watch 모드는 “검증을 줄이는 도구”가 아니라 “문제를 더 빨리 발견하는 도구”가 됩니다.

## FAQ

### Node.js test runner watch 모드는 언제 쓰면 좋나요?

함수 수정, 리팩터링, 테스트 작성처럼 저장과 확인을 반복하는 로컬 개발에 적합합니다.
CI나 배포 검증처럼 한 번 실행하고 종료되어야 하는 환경에서는 쓰지 않습니다.

### watch 모드에서도 전체 테스트를 매번 돌려야 하나요?

작은 프로젝트라면 전체 테스트를 watch로 돌려도 괜찮습니다.
프로젝트가 커지면 파일 경로 지정이나 `--test-name-pattern`으로 현재 작업 범위를 좁히고, 커밋 전에는 필터 없는 전체 테스트를 실행하는 방식이 좋습니다.

### test.only와 watch 모드 중 무엇이 더 안전한가요?

대부분은 watch 모드에 파일 경로나 이름 필터를 붙이는 방식이 더 안전합니다.
`test.only`는 코드에 임시 상태를 남기므로 커밋 전 제거되지 않으면 CI 검증 범위가 줄어들 수 있습니다.

### watch 모드에서 테스트가 두 번째 실행부터 실패하면 무엇을 봐야 하나요?

전역 상태, mock 복구, 임시 파일 정리, 타이머 정리 여부를 먼저 확인하세요.
반복 실행에서만 실패하는 테스트는 테스트 간 격리가 깨졌다는 신호일 가능성이 큽니다.

## 마무리

`node --test --watch`는 Node.js 내장 test runner를 로컬 개발 루프에 자연스럽게 붙이는 가장 간단한 방법입니다.
다만 watch 모드 자체보다 중요한 것은 테스트가 반복 실행에 견딜 만큼 작고, 독립적이고, 정리 가능한 구조인지입니다.

로컬에서는 watch 모드로 빠르게 피드백을 받고, 커밋 전과 CI에서는 필터 없는 전체 테스트로 검증 범위를 회복하세요.
이 균형을 지키면 테스트는 개발 속도를 늦추는 절차가 아니라 저장할 때마다 품질을 확인해 주는 안전망이 됩니다.

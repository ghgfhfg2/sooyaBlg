---
layout: post
title: "Node.js test runner isolation 가이드: 테스트 파일을 안전하게 격리 실행하는 방법"
date: 2026-07-06 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-isolation-child-process-guide
permalink: /development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html
alternates:
  ko: /development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html
  x_default: /development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, isolation, child-process, testing, javascript, ci]
description: "Node.js test runner의 process-level test isolation 동작을 정리합니다. 테스트 파일별 child process 실행, --test-concurrency와의 관계, isolation을 끄는 경우의 위험, 전역 상태와 CI 안정화 기준을 예제로 설명합니다."
---

테스트가 많아질수록 한 파일의 전역 상태가 다른 파일에 영향을 주지 않는 구조가 중요해집니다.
테스트가 로컬에서는 통과하지만 CI에서만 실패하거나, 실행 순서가 바뀌면 결과가 달라진다면 테스트 격리 수준부터 확인해야 합니다.
Node.js 내장 test runner는 기본적으로 테스트 파일을 프로세스 단위로 분리해 이런 문제를 줄입니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#test-runner-execution-model)는 process-level test isolation이 켜져 있을 때 각 테스트 파일이 별도 child process에서 실행된다고 설명합니다.
또 동시에 실행되는 child process 수는 `--test-concurrency` 플래그로 제어됩니다.
반대로 isolation을 끄면 여러 테스트 파일이 같은 test runner process로 import되기 때문에 전역 상태가 파일 사이에서 섞일 수 있습니다.

테스트 병렬 실행 수를 조절하는 방법은 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html)와 이어집니다.
CI 작업을 여러 shard로 나누는 전략은 [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html)를 참고하세요.
테스트 전체의 공통 준비와 정리는 [Node.js test runner global setup 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-global-setup-teardown-guide.html)에서 함께 다뤘습니다.

## test isolation이 필요한 이유

### H3. 테스트 파일 사이의 전역 상태 누수를 막는다

Node.js 테스트 파일은 일반 JavaScript 파일입니다.
파일 최상단에서 환경 변수를 바꾸거나, 전역 객체를 수정하거나, singleton cache를 초기화하면 같은 프로세스 안의 다른 코드에 영향을 줄 수 있습니다.
테스트 파일별 child process 격리는 이런 영향 범위를 파일 하나로 줄이는 안전장치입니다.

```js
// test/a.test.js
import test from 'node:test';
import assert from 'node:assert/strict';

globalThis.featureFlag = 'new-checkout';

test('uses new checkout flag', () => {
  assert.equal(globalThis.featureFlag, 'new-checkout');
});
```

```js
// test/b.test.js
import test from 'node:test';
import assert from 'node:assert/strict';

test('starts without checkout flag', () => {
  assert.equal(globalThis.featureFlag, undefined);
});
```

기본 isolation이 켜져 있다면 두 파일은 서로 다른 child process에서 실행되므로 `globalThis.featureFlag`가 공유되지 않습니다.
하지만 isolation을 끄면 두 파일이 같은 process에 로드될 수 있고, 실행 순서에 따라 두 번째 테스트가 실패할 수 있습니다.
전역 상태를 쓰는 테스트가 있다면 격리를 끄기 전에 반드시 영향 범위를 확인해야 합니다.

### H3. 테스트 실패를 파일 단위로 해석하기 쉬워진다

process-level isolation에서는 테스트 파일 하나가 독립 실행 단위가 됩니다.
공식 문서가 설명하듯 child process가 exit code 0으로 끝나면 해당 테스트 파일은 통과로 간주되고, 그렇지 않으면 실패로 처리됩니다.
이 모델은 CI 로그에서 어떤 파일이 실패했는지 추적하기 쉽습니다.

```bash
node --test test/users.test.js test/orders.test.js
```

이 명령은 지정된 테스트 파일을 test runner 실행 모델에 따라 처리합니다.
각 파일이 독립 child process에서 실행되면 한 파일의 비정상 종료가 다른 파일의 메모리 상태를 직접 오염시키기 어렵습니다.
물론 외부 데이터베이스, 파일 시스템, 네트워크 포트처럼 프로세스 밖 리소스는 여전히 충돌할 수 있으므로 별도 정리 기준이 필요합니다.

## --test-concurrency와 isolation의 관계

### H3. 파일 병렬성은 child process 수로 제어한다

`--test-concurrency`는 process-level isolation이 켜진 상태에서 동시에 실행할 child process 수를 조절합니다.
테스트 파일이 많을수록 값을 높이면 전체 실행 시간이 줄어들 수 있지만, CPU와 외부 리소스 경쟁도 함께 늘어납니다.
성능만 보고 값을 크게 잡기보다 프로젝트가 사용하는 리소스를 기준으로 정해야 합니다.

```bash
node --test --test-concurrency=4
```

이 명령은 동시에 실행되는 테스트 child process 수를 제한합니다.
예를 들어 테스트가 데이터베이스, 브라우저, 파일 잠금, mock server port를 공유한다면 concurrency가 높을수록 충돌 가능성이 커집니다.
CI에서는 runner 사양이 로컬보다 작을 수 있으므로 로컬에서 빠른 값이 CI에서도 안정적이라고 가정하지 않는 편이 좋습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "node --test --test-concurrency=2 --test-reporter=spec"
  }
}
```

로컬 기본값은 단순하게 두고 CI에서는 안정적인 concurrency 값을 명시하는 방식이 관리하기 쉽습니다.
실패가 간헐적으로 발생한다면 먼저 concurrency를 낮춰서 리소스 경합인지 확인하세요.
문제가 사라진다면 테스트 자체의 격리 또는 fixture 설계를 손봐야 합니다.

### H3. 파일 안의 subtest concurrency와 구분한다

파일 단위 병렬성과 파일 안의 subtest 병렬성은 같은 개념이 아닙니다.
공식 문서는 테스트 파일 하나가 `node:test`로 테스트를 정의하면, 그 파일 안의 테스트는 단일 application thread 안에서 실행된다고 설명합니다.
즉 `--test-concurrency`는 주로 여러 테스트 파일을 몇 개의 child process로 동시에 실행할지에 관한 설정입니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('user service', async (t) => {
  await t.test('creates user', async () => {
    assert.equal(await createUser('A'), 'A');
  });

  await t.test('updates user', async () => {
    assert.equal(await updateUser('A'), 'A');
  });
});

async function createUser(name) {
  return name;
}

async function updateUser(name) {
  return name;
}
```

이 파일의 subtest 실행 방식은 파일 바깥의 child process 개수와 별개로 봐야 합니다.
파일 내부에서 공유하는 fixture가 있다면 `beforeEach`, `afterEach`, 임시 객체 생성 기준을 명확히 세우세요.
파일 사이 격리가 되어 있어도 파일 안에서 같은 객체를 여러 테스트가 함께 쓰면 순서 의존 버그가 생길 수 있습니다.

## isolation을 끄기 전에 확인할 것

### H3. 같은 process에서 파일들이 서로 영향을 줄 수 있다

Node.js 공식 문서는 process-level test isolation을 비활성화하면 모든 테스트 파일이 같은 test runner process로 import된다고 설명합니다.
이 경우 한 파일이 수정한 global state를 다른 파일이 볼 수 있습니다.
빠른 실행이나 특수한 로더 구성을 위해 isolation을 끄고 싶을 수 있지만, 기본값을 바꾸는 결정은 테스트 신뢰도와 맞바꾸는 일입니다.

```bash
node --test --test-isolation=none
```

이 명령은 테스트 파일을 같은 process에서 실행하도록 만들 수 있습니다.
작은 유틸리티 라이브러리처럼 모든 테스트가 순수 함수 중심이고 전역 상태가 거의 없다면 문제가 드러나지 않을 수 있습니다.
하지만 애플리케이션 테스트에서는 환경 변수, module cache, mock 상태, timer, 이벤트 리스너가 누적되기 쉽습니다.

### H3. isolation을 끄는 대신 병목을 좁힌다

테스트가 느리다는 이유만으로 isolation을 끄면 원인을 숨긴 채 불안정성을 키울 수 있습니다.
먼저 느린 테스트 파일을 찾아 분리하고, fixture 준비 시간을 줄이고, 불필요한 외부 연결을 mock으로 바꾸는 편이 안전합니다.
실행 범위를 좁혀 확인하려면 name pattern이나 파일 경로를 함께 쓰는 방식도 있습니다.

```bash
node --test test/users.test.js --test-name-pattern="creates user"
```

이 명령은 특정 파일과 테스트 이름으로 실행 범위를 줄입니다.
전체 실행 속도를 바꾸기 전에 실패 재현과 병목 분석을 먼저 하는 흐름입니다.
테스트 이름으로 범위를 좁히는 방법은 [Node.js test runner name pattern 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html)와 연결됩니다.

## 전역 상태를 안전하게 다루는 패턴

### H3. process.env 변경은 테스트 뒤 되돌린다

process-level isolation이 있더라도 한 파일 안에서는 여러 테스트가 같은 `process.env`를 봅니다.
테스트 하나가 환경 변수를 바꾸고 되돌리지 않으면 같은 파일의 다음 테스트가 영향을 받을 수 있습니다.
환경 변수는 테스트 시작 전에 복사하고 끝난 뒤 복원하는 패턴을 추천합니다.

```js
import test, { beforeEach, afterEach } from 'node:test';
import assert from 'node:assert/strict';

let originalEnv;

beforeEach(() => {
  originalEnv = { ...process.env };
});

afterEach(() => {
  process.env = { ...originalEnv };
});

test('uses staging api url', () => {
  process.env.API_BASE_URL = 'https://staging.example.test';

  assert.equal(getApiBaseUrl(), 'https://staging.example.test');
});

function getApiBaseUrl() {
  return process.env.API_BASE_URL ?? 'http://localhost:3000';
}
```

이 예시는 파일 내부 테스트 간 환경 변수 누수를 줄입니다.
실제 프로젝트에서는 `process.env` 전체를 교체하는 방식이 라이브러리와 충돌할 수 있으므로 필요한 key만 저장하고 복원하는 방식도 고려하세요.
핵심은 테스트가 바꾼 값을 테스트가 책임지고 원상복구한다는 점입니다.

### H3. mock과 timer는 테스트 단위로 정리한다

mock 함수, fake timer, 이벤트 리스너는 테스트가 끝난 뒤 남아 있으면 다음 테스트의 실패 원인이 됩니다.
파일 사이 isolation은 파일 밖으로 누수가 번지는 것을 막지만, 파일 안의 누수까지 자동으로 해결해 주지는 않습니다.
`afterEach`에서 reset과 restore를 명시해 두면 실패를 해석하기 쉽습니다.

```js
import test, { afterEach } from 'node:test';
import assert from 'node:assert/strict';

const calls = [];

afterEach(() => {
  calls.length = 0;
});

test('records one call', () => {
  calls.push('created');

  assert.deepEqual(calls, ['created']);
});

test('starts from empty calls', () => {
  assert.deepEqual(calls, []);
});
```

이 코드는 단순하지만 원칙은 실제 mock 상태에도 같습니다.
테스트가 공유 배열, singleton client, in-memory cache를 수정한다면 정리 위치를 테스트 근처에 두세요.
global setup은 전체 실행 전후의 큰 준비를 맡기고, 테스트별 상태 정리는 hook이 맡는 식으로 역할을 나누면 좋습니다.

## CI 운영 기준

### H3. isolation 기본값을 문서화한다

팀에서 Node.js test runner를 쓴다면 `package.json` scripts에 isolation 정책이 드러나야 합니다.
기본 isolation을 유지한다면 별도 옵션을 쓰지 않아도 되지만, CI 문서에는 "테스트 파일은 process-level isolation을 전제로 작성한다"는 기준을 남겨 두는 것이 좋습니다.
그래야 누군가 속도를 이유로 isolation을 끄려 할 때 영향 범위를 검토할 수 있습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "node --test --test-concurrency=2 --test-reporter=spec",
    "test:debug": "node --test --test-concurrency=1"
  }
}
```

`test:debug`처럼 concurrency를 낮춘 명령을 따로 두면 순서 의존 실패를 확인하기 쉽습니다.
랜덤 순서, shard, rerun failures 같은 옵션을 함께 쓰는 프로젝트라면 isolation 정책을 더 분명히 적어야 합니다.
테스트 실행 모델은 개별 assertion보다 더 큰 품질 계약입니다.

### H3. 외부 리소스는 process 밖에서도 격리한다

child process 격리는 JavaScript process 내부 상태를 분리하는 데 강합니다.
하지만 같은 데이터베이스 schema, 같은 Redis key, 같은 임시 파일 경로, 같은 포트를 여러 child process가 쓰면 여전히 충돌합니다.
CI 안정성을 높이려면 외부 리소스 이름에도 테스트 실행 id나 worker id를 반영해야 합니다.

```js
import { mkdtemp } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

export async function createTestWorkspace() {
  return mkdtemp(join(tmpdir(), 'app-test-'));
}
```

테스트마다 고유 임시 디렉터리를 만들면 파일 충돌을 줄일 수 있습니다.
데이터베이스도 가능하다면 테스트 파일별 schema, transaction rollback, 고유 prefix를 사용하세요.
프로세스 격리는 기본 안전망이고, 외부 리소스 격리는 프로젝트가 직접 설계해야 하는 부분입니다.

## 발행 전 체크리스트

### H3. 테스트 격리 점검 항목

- 테스트 파일이 전역 상태, 환경 변수, module cache에 의존하지 않는가?
- `--test-concurrency` 값을 올렸을 때 외부 리소스 충돌이 발생하지 않는가?
- isolation을 끄는 명령이 있다면 이유와 복구 조건이 문서화되어 있는가?
- CI에서 간헐 실패가 발생할 때 concurrency를 낮춰 재현해 볼 수 있는가?
- global setup과 per-test hook의 역할이 섞이지 않았는가?

이 체크리스트는 테스트가 많아진 뒤 한 번에 적용하기보다 새 테스트를 추가할 때마다 확인하는 편이 좋습니다.
특히 전역 상태와 외부 리소스는 작은 프로젝트에서도 빠르게 부채가 됩니다.
처음에는 느슨해 보여도 CI 병렬성이 커지는 순간 문제가 드러납니다.

## FAQ

### H3. Node.js test runner는 기본적으로 테스트 파일을 격리하나요?

공식 문서 기준으로 process-level test isolation이 켜져 있으면 각 matching test file은 별도 child process에서 실행됩니다.
일반적인 사용에서는 이 기본 모델을 유지하는 편이 안전합니다.
특별한 이유 없이 isolation을 끄면 파일 사이 전역 상태 누수가 생길 수 있습니다.

### H3. --test-concurrency를 1로 두면 isolation이 필요 없나요?

아닙니다.
concurrency를 1로 두면 동시에 실행되는 child process 수는 줄지만, 파일별 process 격리라는 모델 자체와는 다른 문제입니다.
순차 실행에서도 같은 process에 여러 파일을 import하면 전역 상태가 섞일 수 있으므로 isolation 정책은 따로 봐야 합니다.

### H3. isolation을 끄면 테스트가 항상 빨라지나요?

항상 그렇지는 않습니다.
프로세스 생성 비용은 줄 수 있지만 전역 상태 누수, module cache 충돌, 정리 실패 때문에 디버깅 비용이 커질 수 있습니다.
성능 문제는 먼저 느린 파일과 외부 리소스 병목을 찾고, 마지막에 isolation 변경을 검토하는 순서가 좋습니다.

## 마무리

Node.js test runner의 isolation은 테스트 파일을 독립적인 실행 단위로 유지하기 위한 기본 안전장치입니다.
`--test-concurrency`는 이 안전장치 위에서 동시에 몇 개의 child process를 돌릴지 정하는 운영 옵션으로 이해하면 쉽습니다.
속도를 높이고 싶다면 isolation을 끄기보다 fixture, 외부 리소스, 테스트 범위, shard 전략을 먼저 정리하세요.

테스트가 믿을 수 있으려면 결과가 실행 순서와 주변 상태에 덜 흔들려야 합니다.
파일 사이에는 process-level isolation을 유지하고, 파일 안에서는 hook과 cleanup으로 상태를 되돌리는 구조를 만들면 CI 실패를 훨씬 읽기 쉬워집니다.
작은 프로젝트라도 지금 `npm test` 명령과 CI script에 isolation 기준이 드러나는지 확인해 보세요.

---
layout: post
title: "Node.js test runner global setup 가이드: 테스트 전역 준비와 정리를 안정적으로 분리하는 법"
date: 2026-08-27 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-global-setup-teardown-guide
permalink: /development/blog/seo/2026/08/27/nodejs-test-runner-global-setup-teardown-guide.html
alternates:
  ko: /development/blog/seo/2026/08/27/nodejs-test-runner-global-setup-teardown-guide.html
  x_default: /development/blog/seo/2026/08/27/nodejs-test-runner-global-setup-teardown-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, global-setup, teardown, testing, ci, fixture, javascript]
description: "Node.js test runner의 --test-global-setup으로 테스트 전역 준비와 정리를 분리하는 방법을 정리합니다. globalSetup, globalTeardown, fixture 서버, 임시 리소스, CI 실패 대응 기준을 실무 예제로 설명합니다."
---

테스트가 늘어나면 `beforeEach`만으로는 준비하기 애매한 리소스가 생깁니다.
예를 들어 테스트 전체에서 한 번만 띄우면 되는 mock 서버, 임시 데이터베이스 파일, 공통 fixture 디렉터리, 테스트용 환경 변수 검증 같은 작업입니다.
이런 준비 코드를 모든 테스트 파일에 복사하면 실행 시간도 늘고, 실패했을 때 어디에서 문제가 시작됐는지 추적하기 어려워집니다.

Node.js 내장 test runner에는 이런 전역 준비를 분리하기 위한 `--test-global-setup` 옵션이 있습니다.
Node.js v26.7.0 공식 문서 기준으로 global setup/teardown은 v24.0.0에 추가되었고, 안정도는 Early development로 표시되어 있습니다.
즉 최신 런타임에서는 실험해볼 만하지만, 운영 CI에 넣을 때는 Node.js 버전 고정과 실패 시나리오 검증을 함께 해야 합니다.

이 글에서는 Node.js test runner의 `globalSetup`, `globalTeardown`을 어디에 쓰면 좋은지, 테스트 파일의 `beforeEach`와 어떻게 역할을 나눌지, CI에서 흔들리지 않게 관리하는 방법을 정리합니다.
기본 테스트 작성은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html), 파일 단위 훅은 [Node.js test runner hooks 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html), 실행 순서 흔들림 점검은 [Node.js test runner randomize seed 가이드](/development/blog/seo/2026/07/27/nodejs-test-runner-randomize-seed-order-dependent-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js test runner 공식 문서](https://nodejs.org/api/test.html)

## global setup이 필요한 순간

### 테스트 전체에서 한 번만 필요한 준비가 있을 때

테스트 준비 코드는 크게 두 종류로 나눌 수 있습니다.
첫 번째는 각 테스트마다 새로 만들어야 하는 상태입니다.
사용자 객체, mock 호출 기록, 임시 파일 이름처럼 테스트 격리를 위해 매번 초기화해야 하는 값이 여기에 들어갑니다.

두 번째는 테스트 실행 전체에서 한 번만 준비해도 되는 상태입니다.
예를 들어 아래 작업은 모든 테스트 파일마다 반복할 필요가 없습니다.

- 테스트용 fixture 루트 디렉터리 생성
- 공통 mock HTTP 서버 시작
- 테스트 데이터베이스 스키마 준비
- 필수 환경 변수의 존재 여부 검증
- CI에서 사용할 임시 포트나 경로 계산

이런 작업을 각 테스트 파일의 `before`에 흩어두면 테스트가 느려지고, 준비 실패 메시지도 중복됩니다.
`globalSetup`은 이 공통 준비를 테스트 실행 시작 전에 한 번 수행하는 위치로 적합합니다.

### beforeEach와 역할을 섞지 않는 것이 중요하다

전역 준비가 생겼다고 해서 모든 테스트 초기화를 그쪽으로 옮기면 안 됩니다.
`globalSetup`은 테스트 실행 전체의 기반을 만드는 곳이고, `beforeEach`는 개별 테스트의 격리를 보장하는 곳입니다.

예를 들어 mock 서버 자체는 전역으로 한 번 띄울 수 있습니다.
하지만 요청 기록, 테스트별 응답 fixture, 인증 토큰 역할 값은 각 테스트에서 초기화하는 편이 안전합니다.

```js
// test/setup.mjs
export async function globalSetup() {
  process.env.TEST_API_BASE_URL = 'http://127.0.0.1:4010';
}

export async function globalTeardown() {
  delete process.env.TEST_API_BASE_URL;
}
```

```js
// user.test.mjs
import { beforeEach, test } from 'node:test';
import assert from 'node:assert/strict';

let requests;

beforeEach(() => {
  requests = [];
});

test('uses configured test API base URL', () => {
  requests.push({ url: process.env.TEST_API_BASE_URL });

  assert.equal(requests[0].url, 'http://127.0.0.1:4010');
});
```

이 구조에서는 전역 설정과 테스트별 상태가 분리됩니다.
리뷰할 때도 "전체 실행에 필요한 기반"과 "개별 테스트 격리"가 눈에 띄게 나뉩니다.

## --test-global-setup 기본 구조

### setup 모듈에서 globalSetup과 globalTeardown을 export한다

공식 문서에 따르면 `--test-global-setup`으로 지정한 모듈은 테스트 실행 전에 평가됩니다.
이 모듈은 `globalSetup`과 `globalTeardown` 함수를 export할 수 있습니다.
`globalSetup`은 모든 테스트가 시작되기 전에 한 번 실행되고, `globalTeardown`은 테스트가 끝난 뒤 한 번 실행됩니다.

ESM 프로젝트라면 아래처럼 작성할 수 있습니다.

```js
// test/global-setup.mjs
import { mkdir, rm } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

const fixtureRoot = join(tmpdir(), 'my-service-test-fixtures');

export async function globalSetup() {
  await rm(fixtureRoot, { recursive: true, force: true });
  await mkdir(fixtureRoot, { recursive: true });

  process.env.TEST_FIXTURE_ROOT = fixtureRoot;
}

export async function globalTeardown() {
  await rm(fixtureRoot, { recursive: true, force: true });
}
```

실행은 CLI에서 옵션을 붙이면 됩니다.

```bash
node --test --test-global-setup ./test/global-setup.mjs
```

`package.json` 스크립트로 고정하면 로컬과 CI의 실행 방식이 맞아집니다.

```json
{
  "scripts": {
    "test": "node --test --test-global-setup ./test/global-setup.mjs"
  }
}
```

테스트 명령이 여러 개라면 global setup을 쓰는 명령과 쓰지 않는 명령을 분리하는 것도 좋습니다.
작은 유닛 테스트까지 무거운 fixture 서버를 띄우면 전체 피드백이 느려질 수 있기 때문입니다.

## mock 서버를 전역 fixture로 띄우기

### 서버 시작과 종료를 한 모듈에서 관리한다

API 클라이언트 테스트에서는 실제 외부 API 대신 로컬 mock 서버를 쓰는 경우가 많습니다.
이 서버를 테스트 파일마다 띄우면 포트 충돌이나 종료 누락이 생기기 쉽습니다.
전역 setup에 서버 생애주기를 모아두면 실행 흐름이 단순해집니다.

```js
// test/global-setup.mjs
import http from 'node:http';

let server;

export async function globalSetup() {
  server = http.createServer((req, res) => {
    if (req.url === '/health') {
      res.writeHead(200, { 'content-type': 'application/json' });
      res.end(JSON.stringify({ ok: true }));
      return;
    }

    res.writeHead(404, { 'content-type': 'application/json' });
    res.end(JSON.stringify({ error: 'not_found' }));
  });

  await new Promise((resolve, reject) => {
    server.once('error', reject);
    server.listen(0, '127.0.0.1', resolve);
  });

  const address = server.address();

  if (!address || typeof address === 'string') {
    throw new Error('Failed to resolve mock server address');
  }

  process.env.TEST_API_BASE_URL = `http://127.0.0.1:${address.port}`;
}

export async function globalTeardown() {
  if (!server) {
    return;
  }

  await new Promise((resolve, reject) => {
    server.close((error) => {
      if (error) reject(error);
      else resolve();
    });
  });
}
```

여기서는 고정 포트 대신 `listen(0)`을 사용합니다.
CI는 여러 작업이 같은 머신에서 동시에 돌 수 있으므로, 포트를 하드코딩하면 간헐적인 충돌이 생기기 쉽습니다.
동적으로 할당된 포트를 `process.env.TEST_API_BASE_URL`에 넣어 테스트 파일이 같은 기준 URL을 보게 만들면 됩니다.

### 테스트 파일은 전역 리소스를 소비만 한다

테스트 파일에서는 서버를 직접 시작하거나 종료하지 않습니다.
이미 준비된 base URL을 읽고, 테스트별 검증에 집중합니다.

```js
// health-client.test.mjs
import test from 'node:test';
import assert from 'node:assert/strict';

test('health endpoint returns ok', async () => {
  const response = await fetch(`${process.env.TEST_API_BASE_URL}/health`);
  const body = await response.json();

  assert.equal(response.status, 200);
  assert.deepEqual(body, { ok: true });
});
```

이 방식은 테스트가 어떤 전역 리소스에 의존하는지 명확히 드러냅니다.
다만 테스트가 `process.env` 값에 강하게 묶이므로, 값이 없을 때 빠르게 실패하는 검증도 넣는 편이 좋습니다.

```js
function getRequiredEnv(name) {
  const value = process.env[name];

  if (!value) {
    throw new Error(`Missing required test env: ${name}`);
  }

  return value;
}
```

에러 메시지는 구체적으로 남기되 실제 토큰이나 비밀 값은 출력하지 않는 것이 좋습니다.
테스트 로그는 CI 아티팩트로 오래 남을 수 있기 때문입니다.

## 실패 시나리오를 먼저 정한다

### globalSetup이 실패하면 테스트는 실행하지 않는 편이 맞다

공식 문서 기준으로 `globalSetup` 함수가 에러를 던지면 테스트는 실행되지 않고 프로세스는 non-zero exit code로 종료됩니다.
또한 이 경우 `globalTeardown`은 호출되지 않습니다.

이 동작은 자연스럽습니다.
테스트의 기반 리소스가 준비되지 않았다면 개별 테스트를 실행해도 대부분 실패하거나, 더 나쁘게는 잘못된 환경을 바라보며 통과할 수 있습니다.

그래서 setup 단계에서는 실패를 숨기지 말아야 합니다.
예를 들어 mock 서버 시작에 실패했는데 기본 URL을 외부 개발 서버로 대체하는 코드는 피해야 합니다.

```js
export async function globalSetup() {
  const baseUrl = await startMockServer();

  if (!baseUrl.startsWith('http://127.0.0.1:')) {
    throw new Error('Test API base URL must be local');
  }

  process.env.TEST_API_BASE_URL = baseUrl;
}
```

테스트가 실제 외부 API를 호출하지 않는다는 경계를 setup에서 확인하면 사고를 줄일 수 있습니다.
네트워크 호출이 필요한 통합 테스트라도, 테스트용 계정과 테스트용 엔드포인트를 명확히 분리해야 합니다.

### teardown은 여러 번 호출되어도 안전하게 만든다

정리 코드는 실패한 테스트 뒤에도 실행될 수 있습니다.
따라서 teardown은 가능한 한 멱등적으로 작성하는 편이 좋습니다.
이미 닫힌 서버, 이미 삭제된 디렉터리, 연결되지 않은 클라이언트를 정리하려고 해도 큰 문제가 없어야 합니다.

```js
export async function globalTeardown() {
  await Promise.allSettled([
    closeMockServer(),
    removeFixtureDirectory(),
    disconnectDatabase()
  ]);
}
```

다만 `Promise.allSettled`를 무조건 쓰면 정리 실패를 조용히 삼킬 수 있습니다.
실무에서는 각 결과를 확인해 로그를 남기고, 치명적인 정리 실패는 exit code에 반영할지 결정해야 합니다.
예를 들어 임시 파일 삭제 실패는 경고로 충분할 수 있지만, 테스트 데이터베이스 연결 종료 실패는 CI 자원 누수를 만들 수 있습니다.

## 전역 상태를 과하게 늘리지 않기

### 공유 상태는 읽기 전용에 가깝게 유지한다

global setup은 편리하지만, 공유 상태를 많이 만들수록 테스트 순서 의존성이 생깁니다.
특히 전역 데이터베이스에 테스트가 직접 쓰고 지우는 구조는 병렬 실행에서 흔들리기 쉽습니다.

권장 기준은 단순합니다.
전역 setup은 무거운 기반을 만들고, 개별 테스트는 자신이 건드릴 상태를 따로 준비합니다.

```js
import { mkdir } from 'node:fs/promises';
import { join } from 'node:path';
import test from 'node:test';

test('writes isolated output', async (t) => {
  const root = process.env.TEST_FIXTURE_ROOT;
  const outputDir = join(root, t.name.replaceAll(/\W+/g, '-'));

  await mkdir(outputDir, { recursive: true });

  // 이 테스트만 사용하는 파일을 outputDir 아래에 생성한다.
});
```

위 예시처럼 전역 루트 아래에 테스트별 디렉터리를 나누면 전체 fixture 생애주기와 개별 테스트 격리를 함께 가져갈 수 있습니다.
병렬 테스트에서는 이름 충돌 가능성도 고려해 파일명에 worker id나 난수를 섞는 편이 더 안전합니다.

### NODE_TEST_WORKER_ID를 로그 해석에 활용한다

Node.js test runner는 프로세스 격리 모드에서 테스트 파일별 worker를 사용할 수 있습니다.
현재 테스트 파일이 어느 worker에서 실행되는지 `NODE_TEST_WORKER_ID` 환경 변수로 확인할 수 있습니다.

이 값을 테스트 데이터 경로나 로그 필드에 포함하면 병렬 실행 중 충돌을 줄이고, CI 실패 로그도 읽기 쉬워집니다.

```js
export function getWorkerFixtureDir(testName) {
  const workerId = process.env.NODE_TEST_WORKER_ID ?? '1';
  const safeName = testName.toLowerCase().replaceAll(/\W+/g, '-');

  return `${process.env.TEST_FIXTURE_ROOT}/worker-${workerId}/${safeName}`;
}
```

핵심은 전역 setup이 "공유해서 써도 되는 것"과 "테스트별로 반드시 분리해야 하는 것"을 구분하게 만드는 일입니다.
구분이 흐리면 테스트가 통과하더라도 신뢰하기 어려워집니다.

## CI에 넣기 전 체크할 것

### Node.js 버전을 먼저 고정한다

`--test-global-setup`은 비교적 새로운 test runner 기능입니다.
따라서 CI 이미지의 Node.js 버전이 로컬과 다르면 옵션 자체가 동작하지 않을 수 있습니다.
`package.json`의 `engines`, `.nvmrc`, GitHub Actions의 `setup-node` 버전을 맞춰 두면 재현성이 좋아집니다.

```yaml
name: test

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: npm
      - run: npm ci
      - run: npm test
```

버전을 고정했다면 테스트 명령도 하나로 모아야 합니다.
로컬에서는 `node --test`만 실행하고 CI에서는 `--test-global-setup`을 붙이면, 두 환경의 결과가 달라질 수 있습니다.

### setup 로그에는 비밀 값을 남기지 않는다

전역 setup은 환경 변수를 확인하는 위치가 되기 쉽습니다.
이때 누락 여부를 기록하는 것은 좋지만, 실제 값은 출력하지 않아야 합니다.

```js
const requiredEnvNames = ['TEST_API_BASE_URL', 'TEST_DATABASE_URL'];

export function assertRequiredTestEnv() {
  const missing = requiredEnvNames.filter((name) => !process.env[name]);

  if (missing.length > 0) {
    throw new Error(`Missing required test env: ${missing.join(', ')}`);
  }
}
```

위 코드는 어떤 키가 빠졌는지만 보여줍니다.
값 자체는 로그에 남기지 않으므로 CI 출력 공유나 아티팩트 보관에서도 부담이 적습니다.
민감정보 마스킹 기준은 테스트 코드에서도 운영 코드와 같은 수준으로 가져가는 편이 좋습니다.

## 실무 체크리스트

### 도입 전 체크

- Node.js CI 버전이 `--test-global-setup`을 지원하는가
- 전역으로 준비할 리소스와 테스트별로 초기화할 상태를 구분했는가
- mock 서버, 임시 디렉터리, 테스트 DB가 외부 운영 리소스와 분리되어 있는가
- setup 실패 시 테스트를 중단하는 것이 맞는지 팀 기준을 정했는가

### 도입 후 체크

- 로컬과 CI의 테스트 명령이 같은 옵션을 사용하는가
- teardown이 이미 정리된 리소스에도 안전하게 동작하는가
- 병렬 실행에서 fixture 경로나 포트 충돌이 없는가
- 로그에 토큰, 쿠키, 실제 개인정보, 내부 접속 정보가 남지 않는가
- 전역 상태 때문에 테스트 순서 의존성이 생기지 않았는가

## 마무리

Node.js test runner의 global setup/teardown은 테스트 준비 코드를 한곳에 모으는 기능입니다.
잘 쓰면 mock 서버나 공통 fixture처럼 비용이 큰 준비 작업을 줄이고, CI 실패 원인을 더 빠르게 좁힐 수 있습니다.

다만 전역 setup은 테스트 격리를 대신하지 않습니다.
공유 리소스는 작게 유지하고, 개별 테스트의 변경 가능한 상태는 `beforeEach`, 테스트별 임시 경로, 명시적인 cleanup으로 분리해야 합니다.
이 기준만 지켜도 내장 test runner 기반 테스트 스위트가 훨씬 예측 가능해집니다.

## 관련 글

- [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)
- [Node.js test runner hooks 가이드: beforeEach와 afterEach로 테스트 정리 누락 줄이기](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html)
- [Node.js test runner randomize seed 가이드: 순서 의존 테스트를 찾아내는 법](/development/blog/seo/2026/07/27/nodejs-test-runner-randomize-seed-order-dependent-guide.html)

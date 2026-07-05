---
layout: post
title: "Node.js test runner global setup 가이드: 테스트 전후 공통 준비를 한곳에서 관리하는 법"
date: 2026-07-05 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-global-setup-teardown-guide
permalink: /development/blog/seo/2026/07/05/nodejs-test-runner-global-setup-teardown-guide.html
alternates:
  ko: /development/blog/seo/2026/07/05/nodejs-test-runner-global-setup-teardown-guide.html
  x_default: /development/blog/seo/2026/07/05/nodejs-test-runner-global-setup-teardown-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, global-setup, teardown, testing, javascript, ci]
description: "Node.js test runner의 --test-global-setup으로 테스트 실행 전후 공통 준비와 정리를 관리하는 방법을 정리합니다. globalSetup, globalTeardown, fixture 수명, CI 적용 기준, per-test hook과의 차이를 예제로 설명합니다."
---

테스트가 늘어나면 모든 파일마다 같은 준비 코드를 반복하게 됩니다.
임시 디렉터리를 만들고, 테스트용 서버를 띄우고, 데이터베이스 상태를 맞추고, 끝난 뒤 리소스를 정리하는 코드가 여러 곳에 흩어지면 실패 원인을 찾기 어려워집니다.
이럴 때 테스트 실행 전체를 기준으로 한 번만 필요한 작업은 global setup과 global teardown으로 분리할 수 있습니다.

Node.js 내장 test runner는 이 용도로 `--test-global-setup` 옵션을 제공합니다.
[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#global-setup-and-teardown)에 따르면 global setup module은 모든 테스트가 실행되기 전에 평가되며, `globalSetup`과 `globalTeardown` 함수를 export할 수 있습니다.
또 이 기능은 Node.js v24.0.0에 추가되었고 early development 상태로 문서화되어 있으므로, 팀 표준으로 도입할 때는 Node.js 버전을 명확히 고정하는 편이 안전합니다.

테스트 실행 범위를 줄이는 방법은 [Node.js test runner name pattern 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html)와 이어집니다.
CI 병렬 실행 기준은 [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html)를 참고하세요.
테스트가 사용할 환경을 반복 가능하게 만드는 관점에서는 [Node.js test runner coverage lcov 가이드](/development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html)와 함께 보면 좋습니다.

## global setup이 필요한 순간

### H3. 테스트 전체에서 한 번만 준비할 리소스가 있다

모든 테스트 파일이 같은 준비 작업을 반복하면 실행 시간이 길어지고 실패 지점도 흐려집니다.
예를 들어 테스트용 HTTP 서버, 임시 작업 디렉터리, 공통 fixture 파일, 외부 서비스 mock 서버는 전체 테스트 실행에서 한 번만 준비해도 충분한 경우가 많습니다.
이런 준비를 각 테스트 파일의 `before()` hook에 넣으면 파일 수만큼 반복 실행됩니다.

```bash
node --test --test-global-setup=./test/global-setup.mjs
```

이 명령은 테스트 실행 전에 `./test/global-setup.mjs`를 로드합니다.
해당 모듈이 `globalSetup`을 export하면 테스트 실행 전에 한 번 호출되고, `globalTeardown`을 export하면 테스트가 끝난 뒤 한 번 호출됩니다.
테스트 파일 내부의 hook보다 실행 범위가 더 넓다는 점이 핵심입니다.

### H3. CI와 로컬의 준비 절차를 맞춘다

CI에서만 별도 shell script로 fixture를 만들고 로컬에서는 테스트 파일이 알아서 준비하게 두면 환경 차이가 생깁니다.
global setup module을 사용하면 로컬과 CI가 같은 Node.js 코드 경로로 준비 절차를 실행할 수 있습니다.
이렇게 하면 "CI에서는 되는데 로컬에서는 안 되는" 설정 차이를 줄일 수 있습니다.

```json
{
  "scripts": {
    "test": "node --test --test-global-setup=./test/global-setup.mjs",
    "test:ci": "node --test --test-global-setup=./test/global-setup.mjs --test-reporter=spec"
  }
}
```

같은 setup module을 공유하되 reporter, coverage, shard 같은 CI 전용 옵션만 별도로 더하는 방식이 관리하기 쉽습니다.
준비 코드는 테스트 명령 안에 보이게 두고, 환경 변수나 임시 경로는 문서화된 기본값을 갖게 하세요.
팀원이 저장소를 처음 받았을 때 `npm test` 하나로 같은 준비 절차를 재현할 수 있어야 합니다.

## 기본 작성 방법

### H3. ESM 모듈에서 globalSetup과 globalTeardown을 export한다

global setup module은 테스트 실행 전후의 수명 주기를 담당합니다.
공식 문서는 `globalSetup`이 모든 테스트 시작 전에 실행되고, `globalTeardown`이 모든 테스트 완료 뒤 실행된다고 설명합니다.
가장 단순한 형태는 아래와 같습니다.

```js
// test/global-setup.mjs
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

let fixtureDir;

export async function globalSetup() {
  fixtureDir = await mkdtemp(join(tmpdir(), 'node-test-fixture-'));
  await writeFile(join(fixtureDir, 'users.json'), '[]\n');
  process.env.TEST_FIXTURE_DIR = fixtureDir;
}

export async function globalTeardown() {
  if (fixtureDir) {
    await rm(fixtureDir, { recursive: true, force: true });
  }
}
```

이 예시는 테스트 실행 전에 임시 디렉터리를 만들고, 테스트가 끝난 뒤 삭제합니다.
테스트 파일은 `process.env.TEST_FIXTURE_DIR` 값을 읽어 fixture 위치를 사용할 수 있습니다.
임시 파일 경로처럼 테스트 실행마다 달라지는 값은 코드에 하드코딩하지 말고 setup 단계에서 주입하는 편이 안전합니다.

```js
// test/users.test.mjs
import test from 'node:test';
import assert from 'node:assert/strict';
import { readFile } from 'node:fs/promises';
import { join } from 'node:path';

test('reads prepared user fixture', async () => {
  const fixtureDir = process.env.TEST_FIXTURE_DIR;
  const body = await readFile(join(fixtureDir, 'users.json'), 'utf8');

  assert.equal(body.trim(), '[]');
});
```

이 구조에서는 테스트 파일이 fixture 생성 방식까지 알 필요가 없습니다.
테스트는 필요한 입력이 어디에 있는지만 확인하고, 준비와 정리는 global setup module이 맡습니다.
관심사를 나누면 fixture 변경이 생겨도 테스트 본문 수정 범위가 줄어듭니다.

### H3. CommonJS 프로젝트에서는 module.exports를 사용한다

CommonJS 기반 프로젝트라면 같은 개념을 `module.exports`로 표현할 수 있습니다.
프로젝트의 module type과 파일 확장자 규칙을 맞추는 것이 중요합니다.
`type: "commonjs"` 또는 `.cjs` 환경에서는 아래처럼 작성합니다.

```js
// test/global-setup.cjs
const { mkdtemp, rm } = require('node:fs/promises');
const { join } = require('node:path');
const { tmpdir } = require('node:os');

let directory;

async function globalSetup() {
  directory = await mkdtemp(join(tmpdir(), 'app-test-'));
  process.env.TEST_WORK_DIR = directory;
}

async function globalTeardown() {
  if (directory) {
    await rm(directory, { recursive: true, force: true });
  }
}

module.exports = { globalSetup, globalTeardown };
```

테스트 runner가 setup module을 로드할 수 있도록 경로와 모듈 형식을 명확히 맞추세요.
ESM과 CommonJS가 섞인 저장소에서는 setup module만 별도 확장자로 고정하는 편이 혼란을 줄입니다.
특히 CI 명령은 shell별 경로 해석 차이를 줄이기 위해 상대 경로를 명시적으로 쓰는 것이 좋습니다.

## 실패와 정리 기준

### H3. globalSetup이 실패하면 테스트는 실행되지 않는다

공식 문서는 `globalSetup` 함수가 error를 throw하면 테스트가 실행되지 않고 프로세스가 non-zero exit code로 종료된다고 설명합니다.
또 이 경우 `globalTeardown`은 호출되지 않습니다.
따라서 setup 단계에서 만든 리소스는 실패 가능성을 고려해 가능한 한 작은 단위로 만들고, 실패 전까지 만든 것만 정리할 수 있게 작성해야 합니다.

```js
// test/global-setup.mjs
import { mkdir, rm } from 'node:fs/promises';

const createdPaths = [];

export async function globalSetup() {
  await mkdir('.tmp/test-cache', { recursive: true });
  createdPaths.push('.tmp/test-cache');

  await prepareDatabase();
}

export async function globalTeardown() {
  await Promise.all(
    createdPaths.map((path) => rm(path, { recursive: true, force: true }))
  );
}

async function prepareDatabase() {
  throw new Error('database is not reachable');
}
```

이 예시는 일부 리소스를 만든 뒤 데이터베이스 준비에서 실패합니다.
하지만 `globalSetup` 실패 시 `globalTeardown`이 호출되지 않을 수 있으므로, 실제 코드에서는 setup 내부에서 실패 정리를 직접 처리하는 구조가 더 안전합니다.
특히 임시 파일, 포트 점유, 컨테이너 실행처럼 남으면 다음 테스트를 방해하는 리소스는 조심해야 합니다.

```js
export async function globalSetup() {
  const cleanup = [];

  try {
    await mkdir('.tmp/test-cache', { recursive: true });
    cleanup.push(() => rm('.tmp/test-cache', { recursive: true, force: true }));

    await prepareDatabase();
  } catch (error) {
    await Promise.allSettled(cleanup.map((fn) => fn()));
    throw error;
  }
}
```

setup 실패 경로까지 정리하면 CI 재시도도 훨씬 안정적입니다.
테스트가 실패한 것이 아니라 준비가 실패한 상황임을 로그에서 구분할 수 있도록 에러 메시지도 구체적으로 남기세요.
단, 로그에 비밀번호, 토큰, 실제 사용자 데이터 같은 민감정보를 출력해서는 안 됩니다.

### H3. globalTeardown은 실패해도 원인을 숨기지 않는다

정리 단계에서 실패하면 테스트 결과 해석이 어려워질 수 있습니다.
예를 들어 테스트 자체는 모두 통과했지만 임시 서버 종료에 실패했다면 프로세스가 오래 남거나 다음 job이 영향을 받을 수 있습니다.
정리 실패는 조용히 무시하지 말고, 어떤 리소스 정리에 실패했는지 알 수 있게 남겨야 합니다.

```js
export async function globalTeardown() {
  const results = await Promise.allSettled([
    stopMockServer(),
    removeFixtureDirectory(),
    disconnectDatabase()
  ]);

  const rejected = results.filter((result) => result.status === 'rejected');

  if (rejected.length > 0) {
    throw new Error(`global teardown failed: ${rejected.length} task(s) failed`);
  }
}
```

여러 정리 작업을 순서대로 실행하다가 첫 실패에서 멈추면 뒤의 리소스가 남을 수 있습니다.
`Promise.allSettled()`로 가능한 정리를 끝까지 시도한 뒤 실패 개수만 요약해 throw하면 운영 로그가 더 읽기 쉽습니다.
상세 로그가 필요하다면 민감정보를 제거한 에러 이름과 코드 중심으로 남기세요.

## per-test hook과 나누는 기준

### H3. 전체 실행 준비는 global setup에 둔다

global setup은 테스트 run 전체에서 한 번이면 충분한 준비에 맞습니다.
테스트용 포트 예약, 공통 fixture 디렉터리 생성, mock server 시작, 테스트 환경 변수 기본값 설정 같은 작업이 여기에 해당합니다.
이 작업들은 테스트 파일마다 반복되면 비용이 크고, 순서가 꼬이면 충돌 가능성도 커집니다.

```js
// good fit for global setup
export async function globalSetup() {
  process.env.API_BASE_URL = await startMockApiServer();
}
```

반대로 각 테스트가 독립적으로 가져야 하는 상태까지 global setup에 넣으면 테스트 간 결합이 생깁니다.
특정 테스트가 데이터를 변경했을 때 다음 테스트가 영향을 받는다면 global fixture가 너무 많은 책임을 가진 것입니다.
공유 준비와 개별 상태를 분리해야 테스트 실패가 재현 가능합니다.

### H3. 테스트별 상태는 beforeEach나 fixture helper에 둔다

테스트마다 초기화해야 하는 데이터는 `beforeEach()`나 helper 함수로 유지하는 편이 낫습니다.
사용자 생성, 장바구니 초기화, 캐시 비우기처럼 테스트 결과에 직접 영향을 주는 상태는 개별 테스트 근처에 있어야 읽기 쉽습니다.
global setup은 토대를 만들고, 테스트별 hook은 실제 시나리오 상태를 만듭니다.

```js
import { beforeEach, test } from 'node:test';
import assert from 'node:assert/strict';

let store;

beforeEach(() => {
  store = new Map();
});

test('saves a user in isolated store', () => {
  store.set('user:1', { name: 'Ada' });
  assert.equal(store.get('user:1').name, 'Ada');
});
```

이 예시는 테스트마다 새로운 `Map`을 사용합니다.
global setup으로 `store`를 공유했다면 다른 테스트의 변경이 남을 수 있습니다.
테스트 독립성이 중요한 상태는 가능한 한 테스트 본문과 가까운 곳에서 만들고 버리세요.

## CI 적용 기준

### H3. Node.js 버전을 고정한다

`--test-global-setup`은 비교적 새로운 test runner 기능입니다.
공식 문서가 early development 상태로 표시하므로, CI와 로컬 개발 환경의 Node.js 버전을 맞추는 것이 중요합니다.
버전이 달라지면 옵션 지원 여부나 동작 세부 사항이 달라질 수 있습니다.

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
          node-version: 26
      - run: npm ci
      - run: npm run test:ci
```

Node.js 버전은 `package.json`의 `engines`, `.nvmrc`, CI 설정 중 한 곳에만 숨겨 두지 말고 함께 맞추는 편이 좋습니다.
테스트 실행 자체가 인프라 신호이기 때문에, 버전 차이는 가능한 빨리 드러나야 합니다.
릴리스 노트를 확인해야 하는 기능이라면 의존하는 CLI 옵션을 테스트 문서에도 남겨 두세요.

### H3. shard와 함께 쓸 때 공유 리소스 충돌을 피한다

CI에서 shard를 사용하면 여러 job이 같은 setup module을 동시에 실행할 수 있습니다.
이때 고정된 포트, 고정된 디렉터리, 같은 테스트 데이터베이스 이름을 사용하면 job끼리 충돌합니다.
global setup은 한 job 안에서는 한 번이지만, 전체 CI matrix 기준으로는 여러 번 실행될 수 있다는 점을 기억해야 합니다.

```js
export async function globalSetup() {
  const shard = process.env.TEST_SHARD_INDEX ?? 'local';
  process.env.TEST_WORK_DIR = `.tmp/test-${process.pid}-${shard}`;
}
```

실제 운영에서는 `mkdtemp()`처럼 충돌 가능성이 낮은 API를 우선 사용하세요.
데이터베이스나 외부 서비스도 job별 namespace를 분리하는 편이 안전합니다.
테스트 병렬화가 늘어날수록 global setup은 "공유"보다 "격리된 준비"를 만드는 역할에 가까워집니다.

## 운영 체크리스트

### H3. global setup 도입 전 확인할 것

global setup은 반복 코드를 줄이는 좋은 도구지만, 너무 많은 책임을 넣으면 테스트 전체가 하나의 큰 상태에 묶입니다.
도입 전에는 아래 기준을 확인하세요.

- 전체 테스트 실행에서 한 번만 준비해도 되는 리소스인가?
- setup 실패 시 남는 파일, 서버, 연결을 정리할 수 있는가?
- teardown 실패가 로그에서 분명하게 드러나는가?
- 테스트별 상태를 global setup에 과하게 넣지 않았는가?
- CI shard, 병렬 job, 로컬 재실행에서 리소스 이름이 충돌하지 않는가?
- Node.js 버전과 test runner 옵션을 문서와 CI에 함께 고정했는가?

이 체크리스트를 통과하면 global setup은 테스트 스위트를 더 단순하게 만듭니다.
반대로 테스트마다 달라야 하는 상태를 전역으로 올리면 실패 재현성이 떨어집니다.
준비 코드를 줄이는 것보다 테스트의 독립성을 지키는 것이 더 중요합니다.

### H3. 안전한 기본 구조

실무에서는 global setup module을 작게 유지하는 편이 좋습니다.
환경 기본값, 임시 작업 공간, mock server 같은 공통 토대만 만들고, 실제 테스트 데이터는 개별 테스트나 helper에서 준비하세요.
아래처럼 역할을 나누면 유지보수가 편합니다.

```text
test/
  global-setup.mjs       # 전체 실행 전후 공통 준비와 정리
  helpers/
    create-user.mjs      # 테스트별 fixture helper
    reset-store.mjs      # 테스트별 상태 초기화
  users.test.mjs
  payments.test.mjs
```

이 구조에서 global setup은 테스트 환경의 바닥을 만들고, helper는 테스트 시나리오의 입력을 만듭니다.
파일 이름만 봐도 어떤 코드가 전체 수명 주기를 다루고 어떤 코드가 개별 테스트 상태를 다루는지 알 수 있습니다.
테스트 실패가 났을 때도 확인해야 할 범위가 명확해집니다.

## 마무리

Node.js test runner의 `--test-global-setup`은 테스트 실행 전후 공통 준비를 한곳으로 모으는 실용적인 옵션입니다.
`globalSetup`과 `globalTeardown`을 사용하면 임시 디렉터리, mock server, 공통 환경 변수 같은 리소스를 반복 없이 관리할 수 있습니다.

다만 이 기능은 모든 테스트 상태를 전역으로 올리기 위한 장치가 아닙니다.
전체 실행에서 한 번이면 충분한 준비만 global setup에 두고, 테스트별 데이터와 검증 상태는 각 테스트 가까이에 유지하세요.
그 경계를 지키면 큰 테스트 스위트에서도 준비 절차는 단순해지고 실패 재현성은 유지됩니다.

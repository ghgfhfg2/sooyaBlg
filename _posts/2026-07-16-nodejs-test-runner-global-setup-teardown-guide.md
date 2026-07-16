---
layout: post
title: "Node.js test runner global setup 가이드: 테스트 전체 준비와 정리를 한곳에 모으는 법"
date: 2026-07-16 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-global-setup-teardown-guide
permalink: /development/blog/seo/2026/07/16/nodejs-test-runner-global-setup-teardown-guide.html
alternates:
  ko: /development/blog/seo/2026/07/16/nodejs-test-runner-global-setup-teardown-guide.html
  x_default: /development/blog/seo/2026/07/16/nodejs-test-runner-global-setup-teardown-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, global-setup, global-teardown, ci, testing, javascript, cleanup]
description: "Node.js test runner의 --test-global-setup 옵션과 globalSetup/globalTeardown 함수로 테스트 전체 준비와 정리를 관리하는 방법을 정리합니다. 임시 서버, 테스트 데이터, CI 환경 변수, 리소스 cleanup 기준을 예제로 설명합니다."
---

테스트 파일마다 서버를 열고, fixture를 만들고, 환경 변수를 맞추는 코드가 반복되면 테스트 흐름이 금방 흐려집니다.
처음에는 몇 줄짜리 helper로 충분해 보이지만, 테스트가 늘어나면 "어떤 준비가 전체 테스트에 필요한가"와 "어떤 준비가 개별 테스트에만 필요한가"가 뒤섞입니다.
이 경계가 흐리면 CI에서만 실패하는 테스트, 종료되지 않는 테스트, 서로 영향을 주는 테스트가 생기기 쉽습니다.

Node.js test runner는 전체 테스트 실행 전후에 한 번씩 동작하는 `globalSetup`과 `globalTeardown`을 지원합니다.
명령줄에서는 `--test-global-setup` 옵션으로 setup 모듈을 지정합니다.
공식 문서 기준으로 이 모듈은 테스트 시작 전 실행할 `globalSetup` 함수와 테스트 완료 후 실행할 `globalTeardown` 함수를 내보낼 수 있습니다.
테스트가 끝나지 않는 문제는 [Node.js test runner force exit 가이드](/development/blog/seo/2026/07/15/nodejs-test-runner-force-exit-hanging-process-guide.html), 개별 테스트 cleanup은 [Node.js test runner AbortSignal cleanup 가이드](/development/blog/seo/2026/07/08/nodejs-test-runner-signal-abort-cleanup-guide.html), 공통 hook 구조는 [Node.js test runner beforeEach/afterEach 가이드](/development/blog/seo/2026/07/10/nodejs-test-runner-beforeeach-aftereach-hooks-guide.html)와 함께 보면 좋습니다.

## global setup이 필요한 이유

### H3. 테스트 전체에 필요한 준비를 한 번만 실행한다

모든 테스트 파일이 같은 준비 작업을 반복할 필요는 없습니다.
예를 들어 테스트용 디렉터리 생성, 임시 환경 변수 설정, mock server 부팅, 공통 fixture 준비는 테스트 실행 전체에서 한 번만 해도 충분한 경우가 많습니다.

테스트 파일마다 같은 준비 코드를 넣으면 다음 문제가 생깁니다.

- 테스트 실행 시간이 불필요하게 길어진다.
- setup 로직이 파일마다 조금씩 달라진다.
- 정리 코드 누락으로 열린 handle이 남는다.
- CI와 로컬 실행 기준이 달라진다.

`globalSetup`은 이런 공통 준비를 한곳에 모으는 진입점입니다.
테스트 파일 안의 `before()`나 `beforeEach()`가 각 파일 또는 각 테스트의 준비라면, global setup은 전체 테스트 프로세스를 위한 준비에 가깝습니다.

```js
// test/setup.mjs
export async function globalSetup() {
  process.env.NODE_ENV = 'test';
  process.env.FEATURE_FLAGS = 'stable';
}

export async function globalTeardown() {
  delete process.env.FEATURE_FLAGS;
}
```

실행할 때는 setup 모듈을 명시합니다.

```bash
node --test --test-global-setup=./test/setup.mjs
```

이렇게 두면 테스트 파일은 "테스트가 어떤 환경에서 실행되는가"보다 실제 검증하려는 동작에 집중할 수 있습니다.

### H3. 전체 정리 기준을 명확히 남긴다

setup보다 더 중요한 것은 teardown입니다.
테스트 전체에서 만든 리소스가 남아 있으면 Node.js 프로세스가 종료되지 않거나, 다음 CI job에 영향을 줄 수 있습니다.

global teardown에 넣기 좋은 작업은 다음과 같습니다.

- 테스트용 HTTP 서버 종료
- 임시 디렉터리 삭제
- 테스트 DB connection 종료
- file watcher 정리
- background interval 중단

반대로 개별 테스트가 만든 리소스를 global teardown에서 한꺼번에 치우는 방식은 피하는 편이 좋습니다.
어느 테스트가 어떤 리소스를 만들었는지 추적하기 어렵고, 테스트 사이의 의존성이 숨어 버립니다.
개별 테스트가 만든 서버나 timer는 해당 테스트의 `t.after()`나 파일 단위 `after()`에서 정리하고, 전체 실행에 필요한 리소스만 global teardown에 둡니다.

## setup 모듈 구성하기

### H3. setup 파일은 작고 명시적으로 둔다

setup 모듈은 테스트 실행의 숨은 전제가 됩니다.
그래서 너무 많은 일을 넣으면 테스트가 읽기 어려워집니다.
처음에는 `test/setup.mjs`처럼 위치와 이름을 단순하게 두고, 실제 준비 작업은 역할별 함수로 나누는 편이 관리하기 좋습니다.

```js
// test/setup.mjs
import { mkdtemp, rm } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

let tempRoot;

export async function globalSetup() {
  tempRoot = await mkdtemp(join(tmpdir(), 'app-test-'));
  process.env.APP_TEST_TMP = tempRoot;
}

export async function globalTeardown() {
  if (tempRoot) {
    await rm(tempRoot, { recursive: true, force: true });
  }

  delete process.env.APP_TEST_TMP;
}
```

이 예시는 전체 테스트가 공유할 임시 루트 디렉터리를 하나 만들고, 테스트가 끝난 뒤 삭제합니다.
테스트 파일은 `process.env.APP_TEST_TMP`를 읽어 자기 테스트에 필요한 하위 디렉터리를 만들 수 있습니다.
중요한 점은 global setup이 모든 fixture 파일을 직접 만들지 않는다는 것입니다.
전체 실행에 필요한 최소한의 기반만 제공하고, 테스트별 데이터는 테스트 안에서 만드는 편이 격리성이 좋습니다.

### H3. package script에 실행 기준을 고정한다

setup 옵션은 팀원이 매번 기억해서 입력하기보다 package script에 고정하는 편이 안전합니다.
그래야 로컬과 CI가 같은 전제에서 테스트를 실행합니다.

```json
{
  "scripts": {
    "test": "node --test --test-global-setup=./test/setup.mjs",
    "test:ci": "node --test --test-global-setup=./test/setup.mjs --test-reporter=spec"
  }
}
```

CI에서 커버리지를 함께 본다면 coverage 옵션도 같은 script에 연결할 수 있습니다.

```json
{
  "scripts": {
    "test:coverage": "node --test --test-global-setup=./test/setup.mjs --experimental-test-coverage --test-coverage-include='src/**/*.js'"
  }
}
```

커버리지 기준을 함께 운영한다면 [Node.js test runner coverage include/exclude 가이드](/development/blog/seo/2026/07/16/nodejs-test-runner-coverage-include-exclude-guide.html)처럼 측정 대상도 명확히 고정해야 합니다.
setup이 테스트 환경을 고정하고, include/exclude가 측정 범위를 고정한다고 보면 됩니다.

## 서버와 외부 리소스 다루기

### H3. 테스트용 서버는 주소를 안전하게 전달한다

통합 테스트에서는 실제 HTTP 서버를 띄우고 요청을 보내는 패턴이 자주 나옵니다.
이때 고정 포트를 쓰면 로컬과 CI에서 충돌할 수 있습니다.
가능하면 포트 `0`으로 서버를 열어 운영체제가 빈 포트를 고르게 하고, 결정된 주소만 테스트에 전달합니다.

```js
// test/setup.mjs
import http from 'node:http';

let server;

export async function globalSetup() {
  server = http.createServer((req, res) => {
    if (req.url === '/health') {
      res.end('ok');
      return;
    }

    res.statusCode = 404;
    res.end('not found');
  });

  await new Promise((resolve) => server.listen(0, resolve));

  const { port } = server.address();
  process.env.TEST_BASE_URL = `http://127.0.0.1:${port}`;
}

export async function globalTeardown() {
  if (server) {
    await new Promise((resolve, reject) => {
      server.close((error) => {
        if (error) reject(error);
        else resolve();
      });
    });
  }

  delete process.env.TEST_BASE_URL;
}
```

테스트 파일은 환경 변수로 전달된 주소만 사용합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('health endpoint returns ok', async () => {
  const response = await fetch(`${process.env.TEST_BASE_URL}/health`);

  assert.equal(response.status, 200);
  assert.equal(await response.text(), 'ok');
});
```

이 방식은 테스트 파일마다 서버를 열지 않아도 되고, 서버 종료 책임도 global teardown에 명확히 남습니다.
다만 테스트가 서버 상태를 바꾸는 경우에는 주의가 필요합니다.
공유 서버 안의 메모리 상태나 DB 상태가 테스트 사이에 누적되면 순서 의존성이 생길 수 있습니다.

### H3. 공유 리소스와 테스트 격리를 분리한다

global setup은 비용이 큰 준비 작업을 줄이는 데 도움이 됩니다.
하지만 모든 것을 공유하면 테스트가 서로 영향을 주기 쉽습니다.
좋은 기준은 "비싼 인프라는 공유하고, 데이터는 테스트마다 격리한다"입니다.

예를 들어 테스트용 서버나 DB container는 전체 실행에서 한 번만 띄울 수 있습니다.
대신 각 테스트는 자기 namespace, schema, temp directory, unique id를 사용해야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { mkdir, rm, writeFile, readFile } from 'node:fs/promises';
import { join } from 'node:path';
import { randomUUID } from 'node:crypto';

test('writes a user fixture in an isolated directory', async (t) => {
  const root = process.env.APP_TEST_TMP;
  const caseDir = join(root, randomUUID());

  await mkdir(caseDir);

  t.after(async () => {
    await rm(caseDir, { recursive: true, force: true });
  });

  const filePath = join(caseDir, 'user.json');

  await writeFile(filePath, JSON.stringify({ name: 'test-user' }));

  const saved = JSON.parse(await readFile(filePath, 'utf8'));
  assert.equal(saved.name, 'test-user');
});
```

이 예시에서는 전체 temp root는 global setup이 만들고, 개별 case directory는 테스트가 만듭니다.
이 경계를 지키면 성능과 격리성을 함께 얻을 수 있습니다.

## CI에서 운영하는 기준

### H3. CI 로그에는 상태보다 원인을 남긴다

global setup이 실패하면 테스트 본문이 실행되기 전에 job이 멈춥니다.
그래서 실패 메시지는 짧고 원인을 추적할 수 있어야 합니다.
다만 민감정보를 그대로 출력하면 안 됩니다.

좋은 로그는 다음 정보를 포함합니다.

- 어떤 준비 단계에서 실패했는지
- 재시도 가능한 문제인지
- 필요한 환경 변수가 존재하는지 여부
- 외부 리소스 연결이 실패했는지 여부

나쁜 로그는 실제 토큰, DB 비밀번호, 개인 식별 정보를 그대로 출력하는 로그입니다.
테스트 로그는 CI artifact로 남거나 외부 서비스에 전송될 수 있으므로, 값 자체보다 "있다/없다"와 리소스 이름 수준으로 제한하는 편이 안전합니다.

```js
export async function globalSetup() {
  if (!process.env.TEST_DATABASE_URL) {
    throw new Error('TEST_DATABASE_URL is required for integration tests');
  }
}
```

환경 변수 값을 출력하지 않아도 원인은 충분히 전달됩니다.
이런 기준은 공통 블로그 가이드의 민감정보 보호 원칙과도 맞습니다.

### H3. force exit보다 teardown을 먼저 고친다

테스트가 끝나지 않을 때 `--test-force-exit`을 붙이면 CI 대기는 줄어듭니다.
하지만 global teardown이 리소스를 제대로 닫지 못한 상태라면 문제는 계속 남습니다.

우선순위는 다음과 같이 잡는 편이 좋습니다.

1. 어떤 리소스가 전체 실행에서 만들어지는지 목록화한다.
2. global teardown에서 해당 리소스를 모두 닫는다.
3. 개별 테스트 리소스는 `t.after()`나 `after()`로 옮긴다.
4. 그래도 CI가 멈추면 임시로 force exit를 붙이고 원인 추적 이슈를 남긴다.

이 순서를 지키면 force exit가 cleanup 누락을 가리는 도구가 아니라, 파이프라인 보호 장치로만 쓰입니다.

## 실무 체크리스트

### H3. global setup에 넣어도 되는 것

global setup은 전체 테스트 실행의 공통 전제를 만드는 곳입니다.
다음 항목은 넣어도 좋은 후보입니다.

- 전체 테스트가 공유하는 임시 루트 디렉터리
- 고정된 테스트 환경 변수
- 테스트용 mock server 또는 local server
- 통합 테스트용 공통 connection pool
- 한 번만 필요한 schema 준비

단, 이 항목들도 테스트 사이에 상태를 공유하지 않도록 설계해야 합니다.
공통 connection pool은 공유할 수 있지만, 데이터는 테스트별로 분리해야 합니다.
공통 서버는 공유할 수 있지만, 요청 결과가 이전 테스트에 의존하면 안 됩니다.

### H3. global setup에 넣지 않는 편이 좋은 것

모든 준비 코드를 global setup으로 올리면 테스트가 읽기 어려워집니다.
다음 항목은 개별 테스트나 파일 단위 hook에 두는 편이 더 낫습니다.

- 특정 테스트에서만 필요한 fixture
- 테스트마다 달라지는 mock 응답
- 테스트 케이스별 사용자 데이터
- 테스트 순서에 따라 바뀌는 상태
- 실패 원인을 특정 테스트와 연결해야 하는 리소스

global setup은 편의를 위한 만능 hook이 아닙니다.
테스트 전체에 필요한 최소한의 바닥을 만들고, 나머지는 테스트 가까이에 두는 것이 유지보수에 유리합니다.

## 마무리

`--test-global-setup`은 Node.js test runner에서 전체 테스트 환경을 정돈하는 유용한 옵션입니다.
잘 쓰면 반복 setup 코드를 줄이고, CI와 로컬의 실행 전제를 맞추고, 리소스 정리 위치를 명확히 만들 수 있습니다.

핵심은 경계입니다.
전체 실행에 필요한 준비는 `globalSetup`에 두고, 전체 실행에서 만든 리소스는 `globalTeardown`에서 닫습니다.
개별 테스트가 만든 데이터와 리소스는 테스트 가까이에서 정리합니다.
이 기준만 지켜도 테스트는 더 빠르고, 덜 흔들리고, 실패 원인을 찾기 쉬워집니다.

## FAQ

### H3. globalSetup은 모든 테스트 파일보다 먼저 실행되나요?

네.
`--test-global-setup`으로 지정한 모듈의 `globalSetup`은 테스트가 시작되기 전에 한 번 실행됩니다.
전체 테스트에 필요한 환경 변수나 공유 리소스를 준비할 때 사용할 수 있습니다.

### H3. globalTeardown에서 모든 테스트 데이터를 지워도 되나요?

전체 실행에서 만든 공통 리소스는 정리해도 됩니다.
하지만 개별 테스트가 만든 데이터까지 global teardown에서 한꺼번에 지우면 테스트 간 의존성이 숨어 버릴 수 있습니다.
테스트별 데이터는 가능한 한 해당 테스트의 cleanup에서 처리하세요.

### H3. beforeEach와 globalSetup은 어떻게 나눠야 하나요?

전체 테스트 실행에서 한 번이면 충분한 준비는 global setup에 둡니다.
각 테스트마다 새로 만들어야 하는 데이터, mock, 상태는 `beforeEach()`나 테스트 내부 setup에 둡니다.
성능보다 격리가 더 중요한 리소스라면 테스트 가까이에 두는 편이 안전합니다.

### H3. 참고한 공식 문서는 무엇인가요?

Node.js 공식 test runner 문서와 command-line API 문서에서 `globalSetup`, `globalTeardown`, `--test-global-setup` 동작을 확인했습니다.
프로젝트에 적용할 때는 사용하는 Node.js 버전의 공식 문서를 함께 확인하는 것이 좋습니다.

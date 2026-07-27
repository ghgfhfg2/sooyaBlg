---
layout: post
title: "Node.js test runner randomize 가이드: 순서 의존 테스트를 찾는 법"
date: 2026-07-27 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-randomize-seed-order-dependent-guide
permalink: /development/blog/seo/2026/07/27/nodejs-test-runner-randomize-seed-order-dependent-guide.html
alternates:
  ko: /development/blog/seo/2026/07/27/nodejs-test-runner-randomize-seed-order-dependent-guide.html
  x_default: /development/blog/seo/2026/07/27/nodejs-test-runner-randomize-seed-order-dependent-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, randomize, seed, flaky-test, ci, testing, javascript]
description: "Node.js test runner의 --test-randomize와 --test-random-seed로 테스트 실행 순서를 섞고, 순서 의존 flaky test를 재현 가능한 방식으로 찾는 방법을 정리합니다. CI 적용 기준, 시드 기록, 격리 전략까지 실무 예제로 설명합니다."
---

테스트가 로컬에서는 통과하는데 CI에서는 가끔 실패한다면 원인은 느린 네트워크나 타이밍만이 아닐 수 있습니다.
테스트가 실행 순서에 의존하고 있을 가능성도 큽니다.
앞 테스트가 만든 전역 상태, 임시 파일, 환경 변수, mock, 데이터베이스 row가 다음 테스트에 남아 있으면 순서가 바뀌는 순간 실패가 드러납니다.

Node.js 내장 test runner에는 이런 문제를 찾기 위한 `--test-randomize`와 `--test-random-seed` 옵션이 있습니다.
공식 문서 기준으로 이 기능은 테스트 파일과 파일 안의 queued test 실행 순서를 섞고, 출력된 seed를 다시 사용해 같은 순서를 재현할 수 있게 합니다.
이 글에서는 Node.js test runner randomize를 언제 켜면 좋은지, CI에서는 어떻게 운용해야 하는지, 순서 의존 테스트를 발견했을 때 어떤 기준으로 고쳐야 하는지 정리합니다.
실패한 테스트를 좁혀 실행하는 방법은 [Node.js test runner name pattern 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html), 재시도와 worker 정보를 함께 기록하는 방법은 [Node.js test runner workerID와 attempt 가이드](/development/blog/seo/2026/07/18/nodejs-test-runner-workerid-attempt-flaky-ci-guide.html)를 함께 보면 좋습니다.

## 테스트 순서 랜덤화가 필요한 이유

### H3. 순서 의존 테스트는 통과하는 척한다

순서 의존 테스트는 항상 실패하지 않습니다.
대부분의 시간에는 정해진 순서로 실행되기 때문에 정상처럼 보입니다.
문제는 파일 추가, 테스트 이름 변경, CI 병렬도 변경, Node.js 버전 변경처럼 실행 순서에 영향을 주는 작은 변화가 생겼을 때 갑자기 드러납니다.

대표적인 원인은 아래와 같습니다.

- 테스트가 `process.env`를 바꾼 뒤 복구하지 않는다.
- 전역 singleton, module cache, logger 설정을 공유한다.
- 임시 디렉터리나 SQLite 파일을 같은 경로에 쓴다.
- mock timer, mock function, stub을 테스트 뒤 reset하지 않는다.
- 테스트 데이터베이스의 row를 삭제하지 않고 다음 테스트가 이어서 사용한다.
- 날짜, locale, timezone 같은 런타임 설정을 전역으로 바꾼다.

이런 문제는 테스트를 많이 작성할수록 찾기 어려워집니다.
한 파일만 실행하면 통과하고 전체 suite에서만 실패하기 때문입니다.
`--test-randomize`는 의도적으로 실행 순서를 흔들어 숨은 결합을 더 빨리 드러내는 도구입니다.

### H3. 랜덤화는 불안정성을 만드는 기능이 아니라 드러내는 기능이다

테스트 순서를 섞으면 CI가 불안정해진다고 느낄 수 있습니다.
하지만 실제로는 이미 존재하던 불안정성이 보이기 시작한 것입니다.
랜덤화가 없으면 같은 문제가 나중에 배포 직전, Node.js 업그레이드 직후, 병렬 실행 설정 변경 후에 더 비싼 방식으로 나타납니다.

그래서 랜덤화는 처음부터 모든 브랜치의 필수 gate로 켜기보다 탐지 단계와 차단 단계를 나눠 도입하는 편이 좋습니다.
처음에는 nightly job이나 별도 CI workflow에서 실행하고, 실패 사례가 줄어들면 pull request 검증으로 옮기는 방식이 안전합니다.

## 기본 사용법

### H3. --test-randomize로 순서를 섞는다

Node.js test runner는 `node --test` 명령에 `--test-randomize`를 붙여 실행 순서를 랜덤화할 수 있습니다.

```bash
node --test --test-randomize
```

랜덤화가 켜지면 runner는 사용한 seed를 diagnostic message로 출력합니다.
이 seed는 실패를 다시 재현할 때 필요하므로 CI 로그에서 잘 보존해야 합니다.
테스트 실패 로그를 수집하는 시스템이 있다면 seed 값을 별도 필드로 파싱해 남기는 것도 좋습니다.

패키지 스크립트에는 아래처럼 별도 명령을 두는 편이 명확합니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:random": "node --test --test-randomize"
  }
}
```

일상적인 로컬 개발에서는 `npm test`로 빠르게 실행하고, 순서 의존성이 의심될 때 `npm run test:random`을 쓰는 식으로 시작할 수 있습니다.

### H3. --test-random-seed로 같은 실패를 재현한다

랜덤화로 실패를 찾았다면 다음 단계는 재현입니다.
`--test-random-seed=<number>`를 사용하면 같은 seed로 테스트 순서를 다시 만들 수 있습니다.

```bash
node --test --test-random-seed=12345
```

`--test-random-seed`를 지정하면 랜덤화도 함께 켜지는 방식으로 이해하면 됩니다.
실패 로그에 seed가 있다면 아래처럼 그대로 복사해 로컬에서 실행합니다.

```bash
npm run test:random -- --test-random-seed=12345
```

단, package manager가 인자를 넘기는 방식은 프로젝트마다 다를 수 있습니다.
팀에서 사용하는 명령을 문서화해 두면 실패 재현 시간이 줄어듭니다.

### H3. watch 모드와 함께 쓰지 않는다

Node.js 공식 문서 기준으로 `--test-randomize`와 `--test-random-seed`는 `--watch` 모드와 함께 지원되지 않습니다.
watch 모드는 빠른 피드백을 위한 개발 루프이고, randomize는 실행 순서 결합을 찾기 위한 검증 루프에 가깝습니다.
두 목적을 분리해서 스크립트를 두는 편이 혼란이 적습니다.

```json
{
  "scripts": {
    "test:watch": "node --test --watch",
    "test:random": "node --test --test-randomize"
  }
}
```

watch 중에 순서 문제가 의심되면 watch를 끄고 seed 기반 실행으로 넘어가는 흐름을 권장합니다.

## 랜덤화가 잘 듣는 테스트 구조

### H3. 독립 테스트는 순서가 바뀌어도 의미가 유지된다

순서 랜덤화의 전제는 각 테스트가 독립적으로 실행될 수 있어야 한다는 점입니다.
테스트 A가 먼저 실행되어야 테스트 B가 의미를 갖는 구조라면 randomize가 문제를 만든 것이 아니라 테스트 설계의 결합을 드러낸 것입니다.

좋은 테스트는 필요한 준비를 자기 안에서 끝내고, 사용한 자원을 자기 안에서 정리합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function createCounter() {
  let count = 0;

  return {
    increment() {
      count += 1;
      return count;
    }
  };
}

test('counter starts from one after increment', () => {
  const counter = createCounter();

  assert.equal(counter.increment(), 1);
});

test('counter increments independently', () => {
  const counter = createCounter();

  counter.increment();

  assert.equal(counter.increment(), 2);
});
```

각 테스트가 새 counter를 만들기 때문에 실행 순서가 바뀌어도 결과가 유지됩니다.
반대로 module scope에 공유 counter를 하나만 두면 어떤 테스트가 먼저 실행됐는지에 따라 결과가 달라질 수 있습니다.

### H3. 순차 await subtest는 랜덤화되지 않을 수 있다

Node.js 문서에서는 subtest를 하나씩 `await`하는 패턴은 각 subtest가 이전 subtest 이후에 시작되므로 선언 순서를 유지한다고 설명합니다.
즉 랜덤화를 켰다고 해서 모든 중첩 테스트가 자동으로 섞이는 것은 아닙니다.

```js
import test from 'node:test';

test('sequential subtests', async (t) => {
  for (const name of ['create', 'update', 'delete']) {
    await t.test(name, async () => {
      // 순차 await 패턴은 선언 순서대로 실행된다.
    });
  }
});
```

이런 구조가 항상 나쁜 것은 아닙니다.
다만 순서 의존성을 찾는 목적이라면 테스트를 독립된 top-level test로 나누거나, subtest 생성 방식이 랜덤화 대상에 들어가는지 확인해야 합니다.
순서가 실제 업무 흐름을 검증하는 시나리오라면 이름에 그 의도를 드러내고, 일반 단위 테스트와 분리하는 편이 좋습니다.

## CI에 적용하는 방법

### H3. 처음에는 별도 job으로 관찰한다

기존 suite에 순서 의존 테스트가 많다면 randomize를 바로 필수 gate로 넣는 순간 배포 흐름이 자주 막힐 수 있습니다.
처음에는 실패해도 main pipeline 전체를 막지 않는 별도 job으로 관찰하는 편이 좋습니다.

예를 들어 GitHub Actions에서는 아래처럼 별도 job을 둘 수 있습니다.

```yaml
name: test

on:
  pull_request:
  schedule:
    - cron: '0 18 * * *'

jobs:
  node-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 26
          cache: npm
      - run: npm ci
      - run: npm test

  node-test-randomized:
    runs-on: ubuntu-latest
    continue-on-error: true
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 26
          cache: npm
      - run: npm ci
      - run: npm run test:random
```

처음부터 `continue-on-error: true`로 두는 이유는 실패를 무시하기 위해서가 아닙니다.
실패 패턴을 모으고, 어떤 테스트가 전역 상태에 의존하는지 목록화한 뒤, 충분히 줄었을 때 필수 검증으로 승격하기 위해서입니다.

### H3. seed는 실패 리포트의 핵심 정보다

랜덤화 실패를 고칠 때 가장 중요한 정보는 seed입니다.
seed가 없으면 "가끔 실패한다"는 말만 남고, 실패 순서를 다시 만들기 어렵습니다.

CI 로그에는 최소한 아래 정보를 남기는 편이 좋습니다.

- Node.js 버전
- 실행 명령
- random seed
- 실패한 테스트 파일과 테스트 이름
- worker id와 retry attempt
- 테스트 격리 설정과 병렬도

이미 flaky test 대응 체계를 만들고 있다면 seed를 같은 단위로 기록하세요.
worker 정보와 재시도 횟수는 [Node.js test runner workerID와 attempt 가이드](/development/blog/seo/2026/07/18/nodejs-test-runner-workerid-attempt-flaky-ci-guide.html)처럼 diagnostic context에 포함하면 원인 분석이 쉬워집니다.

### H3. 고정 seed job도 함께 둘 수 있다

완전 랜덤 job만 있으면 매번 다른 순서를 검사할 수 있지만, 특정 실패를 안정적으로 추적하기는 어렵습니다.
반대로 고정 seed job은 탐색 범위는 좁지만 재현성이 좋습니다.

팀 규모가 커지면 아래처럼 두 가지를 나눌 수 있습니다.

- pull request: 최근에 실패했던 seed 또는 고정 seed로 빠르게 검증
- nightly: 매일 다른 random seed로 넓게 탐색
- 장애 분석: 실패 로그의 seed로 로컬 재현

이 구조는 실패를 "우연히 본 현상"에서 "다시 실행할 수 있는 작업 단위"로 바꿔 줍니다.

## 발견된 순서 의존성 고치기

### H3. 전역 상태를 테스트 안으로 끌어내린다

가장 흔한 해결책은 전역 상태를 줄이는 것입니다.
테스트 파일 상단에서 한 번만 만든 객체를 여러 테스트가 공유하고 있다면, factory 함수로 바꿔 각 테스트가 새 인스턴스를 갖게 만드는 편이 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function createAppForTest() {
  return {
    settings: new Map(),
    set(key, value) {
      this.settings.set(key, value);
    },
    get(key) {
      return this.settings.get(key);
    }
  };
}

test('stores a setting', () => {
  const app = createAppForTest();

  app.set('theme', 'dark');

  assert.equal(app.get('theme'), 'dark');
});
```

이 패턴은 데이터베이스, HTTP server, queue client, cache mock에도 똑같이 적용됩니다.
테스트가 필요한 자원을 만들고, 테스트가 끝나면 닫거나 삭제하는 구조가 순서 랜덤화에 강합니다.

### H3. afterEach에서 복구 책임을 명확히 한다

전역 변경이 불가피한 경우에는 `afterEach`에서 복구해야 합니다.
특히 환경 변수, mock tracker, timer mock은 남기 쉽습니다.

```js
import { afterEach, mock, test } from 'node:test';
import assert from 'node:assert/strict';

const originalEnv = { ...process.env };

afterEach(() => {
  mock.restoreAll();
  mock.timers.reset();

  process.env = { ...originalEnv };
});

test('uses feature flag from env', () => {
  process.env.FEATURE_CHECKOUT_V2 = 'true';

  assert.equal(process.env.FEATURE_CHECKOUT_V2, 'true');
});
```

환경 변수 전체를 바꾸는 방식은 프로젝트에 따라 부담이 있을 수 있습니다.
그럴 때는 테스트 helper를 만들어 변경한 key만 복구하세요.
중요한 기준은 "다음 테스트가 이전 테스트의 흔적을 볼 수 없어야 한다"는 점입니다.

### H3. 파일과 데이터베이스는 고유 경로를 쓴다

파일 시스템을 쓰는 테스트는 순서 의존성이 쉽게 생깁니다.
모든 테스트가 `tmp/test.db`, `tmp/output.json` 같은 고정 경로를 공유하면 어떤 테스트가 먼저 실행됐는지에 따라 결과가 달라집니다.

Node.js에서는 `mkdtemp()`로 테스트마다 고유 임시 디렉터리를 만드는 방식이 단순하고 효과적입니다.

```js
import { mkdtemp, rm, writeFile, readFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';
import test from 'node:test';
import assert from 'node:assert/strict';

async function withTempDir(fn) {
  const dir = await mkdtemp(join(tmpdir(), 'app-test-'));

  try {
    return await fn(dir);
  } finally {
    await rm(dir, { recursive: true, force: true });
  }
}

test('writes output independently', async () => {
  await withTempDir(async (dir) => {
    const file = join(dir, 'output.json');

    await writeFile(file, JSON.stringify({ ok: true }));

    assert.deepEqual(JSON.parse(await readFile(file, 'utf8')), { ok: true });
  });
});
```

대량 파일 순회나 정리 로직을 함께 테스트한다면 [Node.js fs.promises.opendir 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html)를 참고해 파일 handle 정리까지 확인하는 것이 좋습니다.

## 도입 체크리스트

Node.js test runner randomize를 도입할 때는 아래 항목을 확인하세요.

- `node --test --test-randomize`를 별도 스크립트로 추가했는가?
- CI 로그에 random seed가 보존되는가?
- 실패 seed를 `--test-random-seed`로 재실행하는 명령이 문서화됐는가?
- watch 모드와 randomize 검증을 분리했는가?
- 전역 mock, timer, 환경 변수를 `afterEach`에서 복구하는가?
- 임시 파일과 테스트 데이터베이스가 테스트별 고유 경로를 쓰는가?
- 순차 subtest가 랜덤화 탐지 범위를 좁히고 있지 않은가?
- 발견된 순서 의존 실패를 flaky로 묻지 않고 격리 문제로 추적하는가?

## 마무리

`--test-randomize`는 테스트 suite를 일부러 불안정하게 만드는 옵션이 아닙니다.
이미 숨어 있던 순서 의존성을 더 빨리 드러내고, `--test-random-seed`로 같은 실패를 다시 볼 수 있게 만드는 검증 도구입니다.

도입은 작게 시작하는 편이 좋습니다.
먼저 별도 CI job에서 관찰하고, 실패 seed를 남기고, 전역 상태와 임시 파일 공유를 줄이세요.
그 다음 randomize job을 필수 검증으로 올리면 테스트 suite는 실행 순서 변화, 병렬도 변경, 런타임 업그레이드에 더 강해집니다.

## 함께 보면 좋은 글

- [Node.js test runner name pattern 가이드: 테스트 이름으로 실행 범위를 좁히는 방법](/development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html)
- [Node.js test runner workerID와 attempt 가이드: flaky CI 원인 추적하기](/development/blog/seo/2026/07/18/nodejs-test-runner-workerid-attempt-flaky-ci-guide.html)
- [Node.js fs.promises.opendir 가이드: 대량 파일 디렉터리 순회를 안전하게 처리하는 법](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html)
- [Node.js test runner force exit 가이드: 종료되지 않는 테스트 프로세스 추적하기](/development/blog/seo/2026/07/15/nodejs-test-runner-force-exit-hanging-process-guide.html)

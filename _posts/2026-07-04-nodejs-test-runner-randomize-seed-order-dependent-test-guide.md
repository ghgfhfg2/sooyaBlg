---
layout: post
title: "Node.js test runner randomize 가이드: 테스트 실행 순서를 섞고 seed로 재현하는 방법"
date: 2026-07-04 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-randomize-seed-order-dependent-test-guide
permalink: /development/blog/seo/2026/07/04/nodejs-test-runner-randomize-seed-order-dependent-test-guide.html
alternates:
  ko: /development/blog/seo/2026/07/04/nodejs-test-runner-randomize-seed-order-dependent-test-guide.html
  x_default: /development/blog/seo/2026/07/04/nodejs-test-runner-randomize-seed-order-dependent-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, randomize, random-seed, testing, javascript, flaky-test]
description: "Node.js test runner의 --test-randomize와 --test-random-seed 옵션으로 순서 의존 테스트를 찾고, 실패한 실행 순서를 seed로 재현하는 방법을 정리합니다. CI 적용 기준, subtest 작성 패턴, flaky test 추적 방법을 예제로 설명합니다."
---

테스트가 항상 같은 순서로만 실행되면 숨어 있는 순서 의존성이 늦게 드러납니다.
앞선 테스트가 전역 상태를 바꾸거나 임시 파일을 남겼는데도, 우연히 다음 테스트가 그 상태를 전제로 통과할 수 있기 때문입니다.
이런 테스트는 로컬에서는 조용하다가 CI 병렬화, 파일 분리, Node.js 버전 변경 같은 순간에 갑자기 실패합니다.

Node.js 내장 test runner는 이런 문제를 찾기 위해 `--test-randomize`와 `--test-random-seed` 옵션을 제공합니다.
[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#randomizing-tests-execution-order)에 따르면 `--test-randomize`는 발견된 테스트 파일과 각 파일 안에서 대기 중인 테스트 실행 순서를 섞고, 실행 시 사용한 seed를 진단 메시지로 출력합니다.
같은 문서는 `--test-random-seed=<number>`를 지정하면 동일한 무작위 순서를 재현할 수 있으며, 이 옵션만으로도 randomize가 활성화된다고 설명합니다.

테스트 파일을 여러 CI 작업으로 나누는 흐름은 [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html)와 함께 보면 좋습니다.
테스트 내부 병렬성은 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html)에서 먼저 정리했습니다.
실패한 테스트를 다시 돌리는 운영 기준은 [Node.js test runner rerun failures 가이드](/development/blog/seo/2026/07/03/nodejs-test-runner-rerun-failures-ci-guide.html)로 이어집니다.

## randomize가 필요한 상황

### H3. 순서가 바뀌면 실패하는 테스트를 찾는다

순서 의존 테스트는 테스트 코드가 서로 독립적이지 않다는 신호입니다.
전역 변수, 공유 mock, 공용 임시 디렉터리, 재사용되는 데이터베이스 row가 대표적인 원인입니다.
항상 같은 순서로만 실행하면 이런 문제가 정상처럼 보일 수 있습니다.

```bash
node --test --test-randomize
```

이 명령은 테스트 파일과 테스트 큐의 순서를 섞어 실행합니다.
한 번의 실행으로 모든 문제를 찾을 수는 없지만, 정해진 순서에 기대던 테스트를 흔들어 볼 수 있습니다.
특히 테스트가 많고 오래된 프로젝트라면 주기적으로 randomize 실행을 추가하는 것만으로도 숨은 결합을 발견할 가능성이 높아집니다.

### H3. 실패 원인은 seed로 좁힌다

무작위 실행의 단점은 같은 실패를 다시 만들기 어렵다는 점입니다.
Node.js test runner는 이 문제를 줄이기 위해 실행에 사용한 seed를 출력합니다.
CI 로그에 seed가 남아 있으면 같은 순서를 다시 실행할 수 있습니다.

```bash
node --test --test-random-seed=12345
```

`--test-random-seed`를 지정하면 해당 seed 기반의 실행 순서가 재현됩니다.
따라서 CI에서 randomize 실패가 발생했을 때는 실패 로그, Node.js 버전, 실행 명령, seed 값을 함께 남겨야 합니다.
seed 없이 단순히 "가끔 실패한다"로만 기록하면 원인 분석 비용이 크게 늘어납니다.

## 기본 사용법

### H3. 로컬에서는 문제 재현용 명령을 분리한다

일상적인 로컬 테스트 명령을 매번 무작위로 만들 필요는 없습니다.
대신 순서 의존성을 점검하는 명령을 별도 script로 두면 팀원이 같은 방식으로 실행할 수 있습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:random": "node --test --test-randomize"
  }
}
```

```bash
npm run test:random
```

문제가 발견되면 CI 로그나 로컬 출력에 나온 seed를 복사해 재현 명령으로 바꿉니다.
이때 seed 값은 실패 이슈나 pull request 코멘트에 남겨 두는 편이 좋습니다.
같은 실패를 다른 사람이 재현할 수 있어야 순서 의존성을 안정적으로 제거할 수 있습니다.

### H3. CI에서는 정기 실행부터 시작한다

모든 pull request에서 randomize를 강제하면 초기에 flaky test가 많이 드러나 개발 흐름이 흔들릴 수 있습니다.
처음 도입할 때는 야간 job이나 main 브랜치 정기 job으로 시작하는 편이 현실적입니다.
문제가 줄어든 뒤 pull request 필수 체크로 올리는 순서가 좋습니다.

```yaml
name: randomized-test

on:
  schedule:
    - cron: "0 18 * * *"
  workflow_dispatch:

jobs:
  random-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 26
      - run: npm ci
      - run: node --test --test-randomize
```

정기 job에서 실패가 나오면 실패 seed를 이슈에 남기고, 해당 seed로 재현하는 pull request를 만듭니다.
이 방식은 팀이 randomize 도입 초기에 받는 부담을 낮춥니다.
테스트 격리가 어느 정도 정리된 뒤에는 pull request 체크에도 추가할 수 있습니다.

## subtest 작성 패턴 주의점

### H3. 순차 await 패턴은 섞이지 않을 수 있다

공식 문서는 subtest를 하나씩 `await`하는 패턴에서는 각 subtest가 이전 subtest 완료 뒤 시작되므로 선언 순서가 유지될 수 있다고 설명합니다.
즉 `--test-randomize`를 켰다고 해서 모든 nested test가 무조건 섞이는 것은 아닙니다.
작성 패턴에 따라 randomize가 관찰할 수 있는 범위가 달라집니다.

```js
import test from 'node:test';

test('math', async (t) => {
  for (const name of ['adds', 'subtracts', 'multiplies']) {
    await t.test(name, async () => {
      // sequential subtest
    });
  }
});
```

이 코드는 각 subtest를 순서대로 기다립니다.
데이터 기반 테스트를 이렇게 작성하면 읽기는 쉽지만, 순서 섞기 관점에서는 효과가 제한될 수 있습니다.
순서 의존성을 더 잘 드러내야 하는 테스트라면 suite 스타일 API나 sibling test 구조를 검토하세요.

### H3. suite 스타일은 sibling test를 함께 큐에 올린다

`describe()`와 `it()` 같은 suite 스타일 API는 sibling test를 함께 큐에 올리는 구조라 randomize와 더 잘 맞습니다.
같은 기능의 독립 케이스라면 순차 loop보다 sibling test로 표현하는 편이 순서 의존성 점검에 유리할 수 있습니다.

```js
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';

describe('math', () => {
  it('adds', () => {
    assert.equal(1 + 1, 2);
  });

  it('subtracts', () => {
    assert.equal(3 - 1, 2);
  });

  it('multiplies', () => {
    assert.equal(2 * 3, 6);
  });
});
```

중요한 기준은 테스트가 서로의 실행 결과에 기대지 않게 만드는 것입니다.
실행 순서가 바뀌어도 통과한다면 테스트는 더 독립적이고, 병렬 실행이나 shard 적용에도 강해집니다.
randomize는 테스트 설계를 대신해 주는 기능이 아니라 독립성 문제를 드러내는 도구입니다.

## 실패를 고치는 기준

### H3. 공유 상태를 테스트 안으로 끌어들인다

randomize 실패가 나왔을 때 가장 먼저 볼 것은 공유 상태입니다.
테스트 파일 상단의 mutable 변수, 한 번만 초기화되는 mock, 모든 테스트가 같이 쓰는 fixture가 실패 원인인 경우가 많습니다.
가능하면 테스트마다 필요한 상태를 새로 만들고, cleanup을 명시하세요.

```js
import { beforeEach, test } from 'node:test';
import assert from 'node:assert/strict';

let cart;

beforeEach(() => {
  cart = [];
});

test('adds item', () => {
  cart.push({ id: 'book', quantity: 1 });
  assert.equal(cart.length, 1);
});

test('starts empty', () => {
  assert.deepEqual(cart, []);
});
```

이 예시는 `beforeEach()`로 각 테스트의 시작 상태를 맞춥니다.
더 좋은 방법은 상태를 전역 변수에 두지 않고 factory 함수로 생성해 테스트 내부에서 받는 것입니다.
핵심은 이전 테스트가 무엇을 했든 다음 테스트의 출발점이 흔들리지 않게 만드는 것입니다.

### H3. 외부 자원 이름을 고정하지 않는다

파일 시스템, 포트, 데이터베이스, 캐시 key처럼 프로세스 밖에 남는 자원은 순서 의존성과 flaky test를 만들기 쉽습니다.
randomize 실패가 외부 자원 충돌에서 온다면 테스트별 고유 이름을 쓰고 종료 시 정리해야 합니다.

```js
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { test } from 'node:test';
import assert from 'node:assert/strict';

test('writes report file', async (t) => {
  const dir = await mkdtemp(join(tmpdir(), 'report-test-'));
  t.after(() => rm(dir, { recursive: true, force: true }));

  const file = join(dir, 'result.json');
  await writeFile(file, JSON.stringify({ ok: true }));

  assert.match(file, /result\.json$/);
});
```

임시 디렉터리를 테스트마다 만들면 실행 순서가 바뀌어도 파일 충돌 가능성이 줄어듭니다.
데이터베이스를 쓰는 테스트라면 transaction rollback, schema 분리, container 분리 같은 기준을 팀 상황에 맞게 선택하세요.
외부 자원을 공유해야 한다면 테스트가 동시에 실행되거나 순서가 바뀌어도 안전한지 먼저 확인해야 합니다.

## 운영 체크리스트

### H3. randomize 실패 로그에 남길 정보

randomize는 실패를 "운 좋게 다시 만나는" 방식으로 운영하면 안 됩니다.
실패한 실행을 재현할 수 있는 정보를 남겨야 실제 개선으로 이어집니다.
CI 로그나 이슈 템플릿에는 다음 항목을 포함하세요.

```text
Node.js version: 26.x
command: node --test --test-randomize
seed: 12345
failed test file: test/cart.test.js
failed test name: starts empty
```

이 정보가 있으면 담당자는 `node --test --test-random-seed=12345`로 같은 순서를 다시 확인할 수 있습니다.
만약 shard와 함께 사용한다면 shard index와 total도 같이 남겨야 합니다.
실행 환경을 충분히 기록해야 randomize 실패가 단순한 소음이 아니라 고칠 수 있는 결함이 됩니다.

### H3. watch 모드와 함께 쓰지 않는다

공식 문서는 `--test-randomize`와 `--test-random-seed`가 watch 모드와 함께 지원되지 않는다고 설명합니다.
개발 중 빠른 피드백은 watch 모드에 맡기고, 순서 의존성 탐지는 별도 명령이나 CI job으로 분리하는 것이 좋습니다.

```bash
# 빠른 개발 피드백
node --test --watch

# 순서 의존성 점검
node --test --test-randomize
```

역할을 분리하면 팀원이 명령의 의도를 이해하기 쉽습니다.
watch는 수정 중인 테스트를 빠르게 확인하는 도구이고, randomize는 테스트 스위트의 독립성을 검증하는 도구입니다.
두 명령을 같은 목적처럼 다루면 실패 원인과 기대 동작을 혼동하기 쉽습니다.

## FAQ

### H3. randomize를 켜면 테스트가 더 좋은 테스트가 되나요?

아니요.
`--test-randomize`는 순서 의존성을 드러내는 도구입니다.
테스트를 좋게 만드는 작업은 공유 상태 제거, fixture 격리, 외부 자원 정리, 재현 가능한 실패 로그를 통해 이루어집니다.

### H3. seed를 고정해서 항상 같은 값으로 실행해도 되나요?

문제 재현에는 좋지만, 항상 같은 seed만 쓰면 다양한 순서를 탐색하는 효과가 줄어듭니다.
정기 job에서는 randomize로 새 seed를 만들고, 실패 분석 단계에서 해당 seed를 고정하는 방식이 더 실용적입니다.

### H3. shard와 randomize를 같이 써도 되나요?

가능하지만 운영 기준을 명확히 해야 합니다.
실패 로그에 shard 값과 seed를 모두 남겨야 하며, 테스트 파일 분산과 파일 내부 순서 섞기가 함께 작동한다는 점을 팀이 이해해야 합니다.
처음에는 randomize 단독으로 안정성을 높인 뒤 shard와 조합하는 편이 분석하기 쉽습니다.

## 정리

`--test-randomize`는 테스트 실행 순서를 섞어 순서 의존성을 찾는 기능입니다.
`--test-random-seed`는 실패한 순서를 다시 실행할 수 있게 해 주는 재현 장치입니다.
둘을 제대로 쓰려면 seed를 로그에 남기고, 공유 상태와 외부 자원을 테스트마다 격리해야 합니다.

Node.js test runner를 CI에 본격적으로 붙이고 있다면 randomize를 정기 job으로 먼저 도입하세요.
실패가 나왔을 때 seed로 재현하고, 테스트가 왜 이전 실행 상태에 기대고 있었는지 제거하면 됩니다.
그 과정을 반복하면 shard, concurrency, coverage 같은 다음 단계의 CI 최적화도 훨씬 안정적으로 적용할 수 있습니다.

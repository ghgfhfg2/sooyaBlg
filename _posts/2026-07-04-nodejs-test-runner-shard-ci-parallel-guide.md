---
layout: post
title: "Node.js test runner shard 가이드: CI 테스트를 여러 작업으로 나누는 방법"
date: 2026-07-04 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-shard-ci-parallel-guide
permalink: /development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html
alternates:
  ko: /development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html
  x_default: /development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, shard, testing, javascript, ci, parallel-test]
description: "Node.js test runner의 --test-shard 옵션으로 테스트 파일을 여러 CI 작업에 나누어 실행하는 방법을 정리합니다. shard 인덱스 규칙, GitHub Actions matrix 구성, 병렬 실행과의 차이, 실패 추적 기준을 예제로 설명합니다."
---

테스트 스위트가 커지면 한 번의 CI 실행 시간이 배포 속도를 결정합니다.
개별 테스트를 아무리 최적화해도 전체 파일 수가 늘어나면 하나의 job에서 모든 테스트를 순서대로 처리하는 방식에는 한계가 있습니다.
이때 필요한 것은 테스트를 더 느슨하게 만드는 것이 아니라, 같은 기준의 테스트를 여러 작업에 안정적으로 나누는 전략입니다.

Node.js 내장 test runner는 이 흐름을 위해 `--test-shard` 옵션을 제공합니다.
[Node.js CLI 공식 문서](https://nodejs.org/api/cli.html#--test-shard)에 따르면 `--test-shard=<index>/<total>` 형식으로 전체 테스트 파일을 `total`개 조각으로 나누고, 그중 `index`번째 조각만 실행할 수 있습니다.
또 [Node.js test runner 공식 문서](https://nodejs.org/api/test.html)는 shard가 여러 머신이나 프로세스에 테스트 실행을 수평 분산하는 데 적합하며, watch 모드와는 함께 쓰지 않는다고 설명합니다.

테스트 파일 내부 병렬성은 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html)를 먼저 참고하세요.
CI에서 커버리지 결과까지 모아야 한다면 [Node.js test runner coverage lcov 가이드](/development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html)와 함께 보면 좋습니다.
실패한 테스트를 다시 확인하는 흐름은 [Node.js test runner rerun failures 가이드](/development/blog/seo/2026/07/03/nodejs-test-runner-rerun-failures-ci-guide.html)에서 이어서 정리했습니다.

## shard가 해결하는 문제

### H3. 하나의 긴 CI job을 여러 짧은 job으로 나눈다

긴 테스트 job은 피드백을 늦춥니다.
특히 풀 리퀘스트마다 20분 이상 테스트를 기다려야 한다면 개발자는 작은 수정도 늦게 확인하고, 실패 원인을 찾는 시간도 길어집니다.
`--test-shard`는 테스트 파일 목록을 여러 조각으로 나누어 각각 별도 job에서 실행하게 해 줍니다.

```bash
node --test --test-shard=1/4
node --test --test-shard=2/4
node --test --test-shard=3/4
node --test --test-shard=4/4
```

각 명령은 전체 테스트 중 자기 조각에 해당하는 파일만 실행합니다.
CI matrix와 연결하면 네 개 job이 동시에 돌고, 가장 느린 조각이 전체 테스트 단계의 시간이 됩니다.
테스트 전체량이 같더라도 대기 시간이 줄어드는 이유가 여기에 있습니다.

### H3. 테스트 파일 단위 분산이라는 점을 이해한다

shard는 테스트 케이스 하나하나를 균등하게 나누는 기능이 아닙니다.
공식 문서 표현처럼 테스트 파일을 나눈 뒤 해당 조각에 속한 파일을 실행합니다.
따라서 파일마다 실행 시간이 크게 다르면 네 조각으로 나누어도 실제 시간은 고르게 줄지 않을 수 있습니다.

```text
fast-a.test.js      2s
fast-b.test.js      3s
api-heavy.test.js  90s
db-heavy.test.js   75s
```

이런 구조에서는 파일 수 기준으로는 균등해 보여도 실행 시간 기준으로는 한쪽 shard가 훨씬 느릴 수 있습니다.
느린 테스트 파일은 기능 단위로 나누거나, 별도 통합 테스트 job으로 분리하는 편이 낫습니다.
shard는 분산 실행의 시작점이고, 테스트 파일 설계까지 자동으로 고쳐 주지는 않습니다.

## 기본 사용법

### H3. index와 total 규칙을 지킨다

`--test-shard` 값은 `<index>/<total>` 형식입니다.
`index`는 1부터 시작하는 양의 정수이고, `total`은 전체 조각 수입니다.
`1/3`, `2/3`, `3/3`처럼 같은 `total` 안에서 모든 index를 빠짐없이 실행해야 전체 테스트 스위트를 덮을 수 있습니다.

```bash
node --test --test-shard=1/3
node --test --test-shard=2/3
node --test --test-shard=3/3
```

`0/3`처럼 0부터 시작한다고 착각하면 안 됩니다.
또 `1/3`과 `2/4`처럼 서로 다른 total을 섞으면 전체 테스트 분배 기준이 달라집니다.
CI에서는 matrix 값으로 `index`와 `total`을 한곳에서 관리해 실수를 줄이세요.

### H3. package script는 인자를 받을 수 있게 둔다

팀에서 쓰는 명령은 `package.json`에 고정하되, shard 값은 CI가 주입하게 만드는 편이 유연합니다.
로컬에서는 전체 테스트를 돌리고, CI에서는 같은 기본 명령에 shard 옵션만 붙일 수 있습니다.
이렇게 하면 테스트 실행 옵션이 여러 곳에 흩어지지 않습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "node --test"
  }
}
```

```bash
npm run test:ci -- --test-shard=1/4
```

npm script 뒤의 `--`는 이후 인자를 실제 테스트 명령으로 넘기기 위해 사용합니다.
프로젝트에서 pnpm이나 yarn을 쓴다면 각 패키지 매니저의 인자 전달 규칙에 맞추면 됩니다.
핵심은 shard 값이 코드에 박혀 있지 않고 CI matrix에서 명확히 보인다는 점입니다.

## GitHub Actions matrix 예시

### H3. matrix로 shard index를 분산한다

GitHub Actions에서는 matrix를 사용해 같은 job 정의를 여러 번 실행할 수 있습니다.
`shard` 값을 `1`, `2`, `3`, `4`로 만들고, 명령에서는 `shard/4` 형식으로 넘기면 됩니다.
이 구조는 shard 수를 늘리거나 줄일 때도 수정 범위가 작습니다.

```yaml
name: test

on:
  pull_request:

jobs:
  node-test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 26
      - run: npm ci
      - run: npm run test:ci -- --test-shard=${{ matrix.shard }}/4
```

`fail-fast: false`를 두면 한 shard가 실패해도 나머지 shard 결과를 끝까지 볼 수 있습니다.
테스트 전체 상태를 파악해야 하는 상황에서는 이 설정이 유용합니다.
다만 비용을 줄이는 것이 더 중요하다면 팀 정책에 따라 fail-fast를 켜도 됩니다.

### H3. shard 수는 실행 시간과 비용을 함께 보고 정한다

shard 수를 늘리면 대기 시간은 줄어들 수 있지만, CI job 수와 초기화 비용은 늘어납니다.
`npm ci`, 빌드 캐시 복원, 컨테이너 시작 시간이 큰 프로젝트에서는 shard를 너무 많이 쪼개면 오히려 총 사용량이 커질 수 있습니다.
처음에는 2개나 4개처럼 작은 수로 시작하고 실제 시간을 측정하세요.

```text
before: 1 job 18m
after:  4 jobs, slowest shard 6m
cost:   install step repeated 4 times
```

이 결과라면 개발자 대기 시간은 크게 줄었지만 CI 사용량은 증가했을 수 있습니다.
캐시가 잘 동작하는지, 느린 테스트 파일이 특정 shard에 몰리는지 함께 확인해야 합니다.
shard 수는 고정된 정답이 아니라 테스트 스위트의 크기와 CI 비용 구조에 맞춰 조정하는 값입니다.

## concurrency와 shard의 차이

### H3. concurrency는 한 프로세스 안의 병렬성에 가깝다

`--test-concurrency`는 한 실행 안에서 테스트를 얼마나 병렬로 처리할지 조정하는 옵션입니다.
반면 `--test-shard`는 전체 테스트 파일을 여러 실행 단위로 나눕니다.
둘 다 테스트 시간을 줄일 수 있지만, 적용 위치와 실패 원인 분석 방식이 다릅니다.

```bash
node --test --test-concurrency=4
node --test --test-shard=2/4
```

첫 번째 명령은 하나의 test runner 실행 안에서 병렬성을 조정합니다.
두 번째 명령은 네 조각 중 두 번째 조각만 실행합니다.
큰 프로젝트에서는 shard로 job을 나누고, 각 job 안에서 적절한 concurrency를 쓰는 방식이 자연스럽습니다.

### H3. 공유 자원 충돌을 먼저 정리한다

테스트를 여러 shard로 나누면 동시에 실행되는 프로세스가 늘어납니다.
같은 포트, 같은 임시 파일 경로, 같은 테스트 데이터베이스를 공유하면 로컬에서는 통과하던 테스트가 CI에서만 실패할 수 있습니다.
분산 실행 전에는 테스트가 독립적으로 실행될 수 있는지 점검해야 합니다.

```js
import { mkdtemp } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { test } from 'node:test';

test('writes an isolated temp file', async (t) => {
  const dir = await mkdtemp(join(tmpdir(), 'app-test-'));
  t.after(async () => {
    // remove temp files created by this test
  });

  // use dir instead of a shared ./tmp path
});
```

테스트마다 독립된 임시 디렉터리와 포트를 쓰면 shard 수를 늘려도 실패 가능성이 줄어듭니다.
데이터베이스가 필요하다면 schema, database name, container를 shard별로 분리하는 방식을 검토하세요.
분산 실행은 테스트 격리 수준을 그대로 드러내기 때문에, 먼저 격리 기준을 정하는 것이 중요합니다.

## 실패 추적과 운영 기준

### H3. 실패한 shard를 바로 재현할 수 있어야 한다

CI에서 `3/4` shard가 실패했다면 로컬에서도 같은 조각을 실행할 수 있어야 합니다.
실패 로그에는 shard index와 total, Node.js 버전, 테스트 명령이 남아야 합니다.
그래야 실패를 재현하는 사람이 전체 테스트를 다시 돌리지 않고 문제 범위를 좁힐 수 있습니다.

```bash
node --test --test-shard=3/4
```

로컬에서 같은 shard가 통과한다면 CI 환경 차이를 의심해야 합니다.
시간대, CPU 수, 파일 시스템 속도, 외부 서비스 접근 여부가 흔한 원인입니다.
반대로 같은 shard가 로컬에서도 실패한다면 해당 조각의 테스트 파일부터 집중해서 보면 됩니다.

### H3. coverage와 artifact는 shard별로 분리한다

shard마다 별도 프로세스에서 테스트가 실행되므로 결과 파일 이름도 겹치지 않게 해야 합니다.
커버리지, JUnit, 로그 파일을 모두 같은 이름으로 쓰면 마지막 job의 결과만 남거나 업로드 단계에서 충돌할 수 있습니다.
matrix 값을 파일명에 넣으면 추적이 쉬워집니다.

```yaml
- run: >
    node --test
    --test-shard=${{ matrix.shard }}/4
    --experimental-test-coverage
    --test-reporter=spec
    --test-reporter=lcov
    --test-reporter-destination=stdout
    --test-reporter-destination=coverage/lcov-${{ matrix.shard }}.info
```

coverage 결과를 하나로 합치는 단계가 필요하다면 별도 job에서 artifact를 내려받아 병합하세요.
이때도 각 shard가 어떤 파일을 만들었는지 이름만 봐도 알 수 있어야 합니다.
테스트 분산은 실행만 나누는 것이 아니라 결과 수집 방식까지 함께 설계해야 완성됩니다.

## 적용 체크리스트

### H3. 도입 전 확인할 것

shard를 도입하기 전에는 테스트 스위트가 파일 단위로 독립적인지 확인하세요.
특정 테스트가 이전 파일의 전역 상태나 생성 파일에 의존한다면 shard 분산 후 순서가 바뀌면서 실패할 수 있습니다.
이런 실패는 shard의 문제가 아니라 테스트 격리가 부족하다는 신호입니다.

```text
checklist
- test files can run independently
- temp files and ports are unique per test or shard
- CI logs include shard index and total
- artifacts use shard-specific names
- full test command is easy to reproduce locally
```

처음 적용할 때는 전체 테스트 job을 바로 제거하지 말고, 짧은 기간 동안 shard job과 비교해 보는 것도 좋습니다.
두 결과가 안정적으로 일치하면 기존 job을 줄이거나 제거할 수 있습니다.
테스트 시간을 줄이는 변화일수록, 실패를 놓치지 않는 검증 루프가 먼저 필요합니다.

## 마무리

`--test-shard`는 Node.js 내장 test runner만으로 CI 테스트를 수평 분산할 수 있게 해 주는 실용적인 옵션입니다.
`<index>/<total>` 규칙을 지키고, CI matrix와 연결하며, artifact 이름을 shard별로 분리하면 큰 테스트 스위트의 대기 시간을 줄일 수 있습니다.

다만 shard는 테스트 격리와 결과 수집 문제를 대신 해결하지 않습니다.
공유 자원 충돌을 줄이고, 실패한 shard를 바로 재현할 수 있게 만들고, 느린 테스트 파일을 계속 관찰해야 합니다.
그 기준까지 갖추면 테스트 분산은 단순한 속도 개선을 넘어 CI 운영 품질을 높이는 도구가 됩니다.

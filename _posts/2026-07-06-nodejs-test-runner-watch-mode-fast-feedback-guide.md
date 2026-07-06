---
layout: post
title: "Node.js test runner watch mode 가이드: 파일 변경마다 테스트를 빠르게 다시 실행하는 방법"
date: 2026-07-06 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-watch-mode-fast-feedback-guide
permalink: /development/blog/seo/2026/07/06/nodejs-test-runner-watch-mode-fast-feedback-guide.html
alternates:
  ko: /development/blog/seo/2026/07/06/nodejs-test-runner-watch-mode-fast-feedback-guide.html
  x_default: /development/blog/seo/2026/07/06/nodejs-test-runner-watch-mode-fast-feedback-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, watch-mode, testing, javascript, developer-experience, ci]
description: "Node.js test runner의 watch mode를 사용해 파일 변경마다 관련 테스트를 다시 실행하는 방법을 정리합니다. node --test --watch 기본 사용법, 테스트 범위 좁히기, CI와 분리해야 하는 이유, flaky test를 피하는 패턴을 예제로 설명합니다."
---

테스트를 자주 실행할수록 버그를 빨리 발견할 수 있지만, 매번 명령을 다시 입력하는 흐름은 금방 부담이 됩니다.
작은 리팩터링이나 테스트 작성 중에는 저장할 때마다 필요한 테스트만 다시 돌고, 실패 지점이 바로 보이는 구조가 훨씬 효율적입니다.
Node.js 내장 test runner의 watch mode는 이런 빠른 피드백 루프를 만들 때 사용할 수 있는 기본 도구입니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#watch-mode)는 `--watch` 플래그를 넘기면 test runner가 테스트 파일과 의존 파일의 변경을 감지하고, 변경과 관련된 테스트를 다시 실행한다고 설명합니다.
또 watch mode는 프로세스를 종료하기 전까지 계속 실행됩니다.
즉 CI용 일회성 명령이 아니라 로컬 개발 중 반복 확인을 위한 실행 모드로 보는 편이 좋습니다.

테스트 파일 단위 격리 모델은 [Node.js test runner isolation 가이드](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html)에서 다뤘습니다.
테스트 이름으로 실행 범위를 좁히는 방법은 [Node.js test runner name pattern 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html)를 참고하세요.
태그 기반 분류가 필요하다면 [Node.js test runner tags filter 가이드](/development/blog/seo/2026/07/02/nodejs-test-runner-tags-filter-guide.html)와 함께 설계하면 좋습니다.

## watch mode가 필요한 순간

### H3. 구현과 테스트를 번갈아 고칠 때 피드백을 줄인다

기능 코드를 고치고 테스트 명령을 다시 입력하는 반복은 작은 비용처럼 보이지만, 하루 종일 쌓이면 집중을 끊습니다.
watch mode를 켜 두면 저장할 때마다 테스트가 다시 실행되므로 실패를 더 빨리 확인할 수 있습니다.
기본 사용법은 단순합니다.

```bash
node --test --watch
```

이 명령은 test runner를 watch mode로 실행합니다.
테스트 파일이나 테스트가 의존하는 파일이 바뀌면 관련 테스트가 다시 실행됩니다.
개발자는 터미널을 열어 둔 채 에디터에서 구현과 테스트를 계속 수정하면 됩니다.

watch mode는 특히 작은 단위 테스트를 작성할 때 효과가 큽니다.
테스트 하나를 먼저 실패하게 만들고, 구현을 고치고, 다시 통과하는 과정을 짧게 반복할 수 있기 때문입니다.
다만 통합 테스트처럼 외부 리소스 준비 시간이 긴 테스트까지 모두 watch 대상에 넣으면 오히려 피드백이 느려질 수 있습니다.

### H3. 전체 테스트보다 좁은 범위로 시작한다

프로젝트가 커질수록 전체 테스트를 watch mode로 계속 돌리는 방식은 무거워집니다.
처음에는 수정 중인 파일과 가까운 테스트 파일만 지정하는 편이 좋습니다.
이렇게 하면 변경 감지 후 다시 실행해야 할 범위가 줄어듭니다.

```bash
node --test --watch test/users.service.test.js
```

이 명령은 특정 테스트 파일을 watch mode로 실행합니다.
현재 작업 중인 모듈이 명확하다면 이 방식이 가장 빠른 로컬 루프가 됩니다.
나중에 변경 범위가 넓어졌을 때 전체 테스트를 한 번 실행해 회귀를 확인하면 됩니다.

테스트 이름까지 좁힐 수도 있습니다.
실패한 케이스 하나를 고치는 중이라면 파일 경로와 `--test-name-pattern`을 함께 쓰는 방식이 실용적입니다.

```bash
node --test --watch test/users.service.test.js --test-name-pattern="creates user profile"
```

이 명령은 지정한 파일 안에서 이름 패턴에 맞는 테스트를 중심으로 확인하는 흐름입니다.
정규식 패턴이 너무 넓으면 예상보다 많은 테스트가 실행될 수 있으므로, 로컬 재현 명령은 읽기 쉬운 문자열부터 시작하세요.
여러 사람이 공유하는 문서나 PR 코멘트에 남길 때도 복잡한 정규식보다 명확한 테스트 이름이 낫습니다.

## package.json에 안전하게 추가하기

### H3. test와 test:watch를 분리한다

watch mode는 계속 실행되는 명령입니다.
그래서 CI에서 쓰는 `test` 스크립트와 로컬 개발용 `test:watch` 스크립트를 분리해야 합니다.
CI가 끝나지 않는 문제를 피하고, 명령 의도를 분명히 하기 위해서입니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:watch": "node --test --watch"
  }
}
```

`npm test`는 한 번 실행하고 종료되는 명령으로 유지합니다.
`npm run test:watch`는 로컬에서 터미널을 열어 둔 채 사용하는 명령으로 둡니다.
이 구분만 지켜도 CI 설정 실수를 크게 줄일 수 있습니다.

프로젝트 규모가 커지면 더 좁은 watch 스크립트를 추가할 수 있습니다.
예를 들어 서비스 계층 테스트만 자주 수정한다면 별도 스크립트가 유용합니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:watch": "node --test --watch",
    "test:watch:service": "node --test --watch test/**/*.service.test.js"
  }
}
```

스크립트가 많아질수록 이름은 실행 범위를 드러내야 합니다.
`test:watch:service`처럼 대상이 분명하면 팀원이 명령을 잘못 고를 가능성이 줄어듭니다.
반대로 `test:fast`, `test:dev`처럼 기준이 모호한 이름은 시간이 지나면 관리하기 어려워집니다.

### H3. CI에서는 watch mode를 사용하지 않는다

CI는 결정적인 결과를 남기는 환경이어야 합니다.
watch mode는 파일 변경을 기다리며 계속 실행되므로 CI job에 넣으면 작업이 끝나지 않거나 timeout으로 실패할 수 있습니다.
CI에서는 항상 종료되는 test runner 명령을 사용하세요.

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
          node-version: 24
      - run: npm ci
      - run: npm test
```

이 예시는 CI에서 `npm test`만 실행합니다.
`npm test`가 `node --test`처럼 종료되는 명령이면 결과를 안정적으로 수집할 수 있습니다.
watch mode는 로컬 개발 경험을 높이는 도구로 두고, 배포 전 검증은 별도 CI 명령으로 분리하세요.

## flaky test를 줄이는 watch mode 운영법

### H3. 저장할 때마다 상태가 누적되지 않게 만든다

watch mode는 같은 개발 세션에서 테스트를 여러 번 실행하게 만듭니다.
테스트가 임시 파일, 포트, 데이터베이스 레코드, mock 상태를 제대로 정리하지 않으면 두 번째 실행부터 실패할 수 있습니다.
이런 실패는 실제 구현 버그와 구분하기 어려우므로 테스트 정리 기준을 먼저 점검해야 합니다.

```js
import test, { afterEach } from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

let tempDir;

afterEach(async () => {
  if (tempDir) {
    await rm(tempDir, { recursive: true, force: true });
    tempDir = undefined;
  }
});

test('writes user export file', async () => {
  tempDir = await mkdtemp(join(tmpdir(), 'user-export-'));
  const file = join(tempDir, 'users.json');

  await writeFile(file, JSON.stringify([{ id: 1, name: 'Ada' }]));

  assert.ok(file.endsWith('users.json'));
});
```

이 예시는 테스트가 만든 임시 디렉터리를 테스트 뒤에 삭제합니다.
watch mode 자체가 상태 정리를 대신해 주지는 않습니다.
테스트가 만든 리소스는 테스트가 직접 정리해야 반복 실행에서도 같은 결과를 기대할 수 있습니다.

### H3. 외부 리소스 의존 테스트는 별도로 분리한다

watch mode에 모든 테스트를 넣으면 빠른 피드백이라는 장점이 줄어듭니다.
데이터베이스, 브라우저, 외부 API mock server처럼 준비 비용이 큰 테스트는 별도 스크립트나 태그로 분리하는 편이 좋습니다.
로컬 watch 루프는 빠른 단위 테스트 중심으로 유지하세요.

```json
{
  "scripts": {
    "test": "node --test",
    "test:unit:watch": "node --test --watch test/unit/**/*.test.js",
    "test:integration": "node --test test/integration/**/*.test.js"
  }
}
```

이 구성에서는 단위 테스트만 watch mode로 반복 실행합니다.
통합 테스트는 명시적으로 필요할 때 실행합니다.
작업 중에는 빠른 피드백을 유지하고, 커밋 전에는 전체 `npm test` 또는 CI와 같은 범위의 명령으로 확인하는 흐름이 좋습니다.

## 실무 체크리스트

### H3. watch mode 도입 전 확인할 기준

watch mode는 설정 자체보다 운영 기준이 중요합니다.
다음 기준을 정해 두면 팀 안에서 명령 사용 방식이 일관됩니다.

- `npm test`는 CI에서도 쓸 수 있게 한 번 실행 후 종료한다.
- `npm run test:watch`는 로컬 개발 전용으로 둔다.
- 자주 고치는 영역은 좁은 watch 스크립트를 추가한다.
- 임시 파일, mock, timer, process.env 변경은 테스트 뒤 정리한다.
- 외부 리소스 의존 테스트는 빠른 watch 루프에서 분리한다.

이 체크리스트는 개발 속도와 테스트 신뢰도를 함께 지키기 위한 최소 기준입니다.
watch mode를 켰을 때 실패가 자주 흔들린다면 먼저 테스트의 상태 누수와 외부 리소스 충돌을 의심하세요.
실패가 재현되지 않는 상태로 watch 범위만 넓히면 원인 분석이 더 어려워질 수 있습니다.

## 마무리

Node.js test runner watch mode는 로컬 개발 중 테스트 피드백을 짧게 만드는 도구입니다.
`node --test --watch`만으로 시작할 수 있지만, 실제 프로젝트에서는 실행 범위와 스크립트 이름, CI 분리 기준까지 함께 정해야 안정적으로 쓸 수 있습니다.
빠른 단위 테스트는 watch mode로 반복하고, 커밋 전에는 종료되는 전체 테스트 명령으로 확인하는 흐름을 추천합니다.

핵심은 watch mode를 CI 대체재가 아니라 개발 중 피드백 루프로 쓰는 것입니다.
테스트가 만든 상태를 스스로 정리하고, 무거운 테스트를 분리하면 Node.js 기본 도구만으로도 충분히 가벼운 테스트 개발 환경을 만들 수 있습니다.

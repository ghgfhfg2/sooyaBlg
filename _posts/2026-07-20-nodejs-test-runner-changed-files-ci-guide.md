---
layout: post
title: "Node.js test runner 변경 파일 실행 가이드: PR에서 필요한 테스트만 빠르게 돌리기"
date: 2026-07-20 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-changed-files-ci-guide
permalink: /development/blog/seo/2026/07/20/nodejs-test-runner-changed-files-ci-guide.html
alternates:
  ko: /development/blog/seo/2026/07/20/nodejs-test-runner-changed-files-ci-guide.html
  x_default: /development/blog/seo/2026/07/20/nodejs-test-runner-changed-files-ci-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, changed-files, ci, testing, javascript, git-diff, fast-feedback]
description: "Node.js test runner로 PR에서 변경된 테스트 파일만 빠르게 실행하는 CI 래퍼 설계 방법을 정리합니다. git diff, 파일 매핑, fallback 전체 실행, reporter, 민감정보 없는 로그 기준을 예제로 설명합니다."
---

PR 검증 시간이 길어지면 개발자는 테스트 결과를 기다리는 동안 맥락을 잃기 쉽습니다.
모든 변경에 전체 테스트를 돌리는 방식은 단순하고 안전하지만, 저장소가 커질수록 피드백이 느려지고 작은 수정도 배포 흐름을 막습니다.
이럴 때는 변경된 파일을 기준으로 먼저 필요한 테스트만 빠르게 실행하고, main branch나 야간 작업에서 전체 회귀 테스트를 보완하는 전략이 현실적입니다.

Node.js 내장 test runner는 `node --test`만으로도 충분히 쓸 수 있지만, PR에서 변경 파일을 읽고 테스트 파일 목록을 조립하려면 작은 CI 래퍼를 두는 편이 좋습니다.
이 글에서는 `git diff`로 변경 파일을 찾고, 관련 테스트 파일을 선택하고, 대상이 비어 있을 때의 fallback을 정하는 방법을 정리합니다.
[Node.js test runner run() API 가이드](/development/blog/seo/2026/07/19/nodejs-test-runner-run-api-custom-runner-guide.html), [Node.js test runner reporter 가이드](/development/blog/seo/2026/07/18/nodejs-test-runner-reporters-spec-tap-dot-junit-guide.html), [Node.js test runner shard CI 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html)와 함께 보면 PR 테스트 파이프라인을 더 안정적으로 설계할 수 있습니다.

## 변경 파일 기반 테스트가 필요한 이유

### H3. 빠른 피드백과 전체 회귀 테스트는 역할이 다르다

PR마다 전체 테스트를 돌리면 누락 위험은 줄어듭니다.
하지만 전체 테스트가 20분, 40분씩 걸리기 시작하면 개발자는 작은 수정도 늦게 확인하게 됩니다.
리뷰어도 실패 원인을 바로 보지 못하고, 여러 커밋이 쌓인 뒤에야 테스트 실패를 마주칩니다.

변경 파일 기반 테스트는 이 문제를 줄이는 빠른 1차 필터입니다.
예를 들어 `src/invoice/format.js`가 바뀌었다면 `test/invoice/format.test.js`를 먼저 실행합니다.
문서만 바뀌었다면 테스트를 생략하거나 smoke test만 돌릴 수 있습니다.
반대로 shared helper나 package 설정이 바뀌면 전체 테스트로 fallback합니다.

중요한 점은 이 전략이 전체 테스트를 완전히 대체하지 않는다는 것입니다.
PR에서는 빠른 관련 테스트를 먼저 돌리고, merge 전 필수 job이나 main branch에서는 전체 회귀 테스트를 유지하는 구성이 안전합니다.

### H3. 테스트 선택 규칙은 코드 리뷰 대상이어야 한다

변경 파일 기반 실행에서 가장 위험한 부분은 "어떤 파일이 바뀌면 어떤 테스트를 돌릴지"입니다.
이 규칙이 CI YAML의 긴 shell 문자열로만 남아 있으면 리뷰하기 어렵고, 예외가 늘수록 유지보수가 힘들어집니다.

작은 JavaScript 래퍼로 분리하면 규칙을 함수로 표현할 수 있습니다.
소스 파일과 테스트 파일의 매핑, 전체 실행 fallback 조건, 문서 변경 처리, 로그 출력 기준을 한 파일에서 볼 수 있습니다.
테스트 실행 정책이 코드가 되면 변경 자체도 PR 리뷰에서 다룰 수 있습니다.

## git diff로 변경 파일 찾기

### H3. 기준 브랜치를 명시한다

CI에서 변경 파일을 찾을 때는 비교 기준을 명확히 해야 합니다.
PR이면 보통 base branch와 현재 HEAD를 비교합니다.
로컬에서는 `origin/main...HEAD`처럼 merge base 기준 비교를 쓰면 현재 브랜치에서 바뀐 파일을 잡기 쉽습니다.

```bash
git diff --name-only --diff-filter=ACMR origin/main...HEAD
```

`--name-only`는 파일 경로만 출력합니다.
`--diff-filter=ACMR`는 추가, 복사, 수정, 이름 변경 파일을 대상으로 삼습니다.
삭제된 파일은 실행할 테스트 파일로 넘길 수 없으므로 보통 제외합니다.

CI 서비스마다 base branch 환경 변수 이름은 다릅니다.
GitHub Actions라면 PR 이벤트에서 base ref를 얻을 수 있고, GitLab이나 Buildkite도 비슷한 값을 제공합니다.
다만 원본 환경 변수를 그대로 로그에 출력하지 않는 편이 좋습니다.
브랜치명이나 repository 경로가 외부 알림으로 복사될 수 있기 때문입니다.

### H3. 변경 파일 목록은 정규화해서 다룬다

shell 출력은 줄바꿈 문자열입니다.
테스트 래퍼 안에서는 이를 배열로 바꾸고, 빈 줄과 위험한 상대 경로를 걸러내는 편이 좋습니다.

```js
// scripts/changed-tests.mjs
import { execFileSync } from 'node:child_process';

function getChangedFiles(base = 'origin/main') {
  const output = execFileSync('git', [
    'diff',
    '--name-only',
    '--diff-filter=ACMR',
    `${base}...HEAD`
  ], { encoding: 'utf8' });

  return output
    .split('\n')
    .map((file) => file.trim())
    .filter(Boolean)
    .filter((file) => !file.startsWith('../') && !file.startsWith('/'));
}
```

여기서는 `execFileSync`를 사용했습니다.
명령과 인자를 분리해 넘기면 shell 문자열 조합보다 안전하고, base 값도 제한된 값으로 관리하기 쉽습니다.
실제 CI에서는 `base`를 환경 변수에서 읽더라도 허용 가능한 ref 형식인지 확인한 뒤 쓰는 편이 좋습니다.

## 관련 테스트 파일 고르기

### H3. 테스트 파일 변경은 그대로 실행한다

가장 단순한 규칙은 테스트 파일이 바뀌었으면 그 파일을 그대로 실행하는 것입니다.
Node.js test runner는 파일 경로를 인자로 받을 수 있으므로 별도 glob을 만들 필요가 없습니다.

```js
function isTestFile(file) {
  return (
    file.endsWith('.test.js') ||
    file.endsWith('.test.mjs') ||
    file.endsWith('.spec.js') ||
    file.endsWith('.spec.mjs')
  );
}

function selectDirectTestFiles(changedFiles) {
  return changedFiles.filter(isTestFile);
}
```

이 규칙은 예측하기 쉽습니다.
테스트 파일만 고친 PR은 빠르게 해당 테스트만 실행됩니다.
문제는 소스 파일만 바뀐 경우입니다.
이때는 프로젝트의 디렉터리 구조를 기준으로 관련 테스트를 찾는 규칙이 필요합니다.

### H3. 소스 파일과 테스트 파일의 경로 규칙을 정한다

프로젝트마다 테스트 위치는 다릅니다.
예를 들어 `src/users/service.js`의 테스트가 `test/users/service.test.js`에 있다면 소스 경로에서 테스트 경로를 계산할 수 있습니다.

```js
import { existsSync } from 'node:fs';

function testCandidateForSource(file) {
  if (!file.startsWith('src/') || !file.endsWith('.js')) return null;

  const withoutSrc = file.slice('src/'.length);
  const withoutExt = withoutSrc.replace(/\.js$/, '');

  return `test/${withoutExt}.test.js`;
}

function selectMappedTestFiles(changedFiles) {
  return changedFiles
    .map(testCandidateForSource)
    .filter(Boolean)
    .filter((file) => existsSync(file));
}
```

이 방식은 단순하지만 강력합니다.
저장소 구조가 일관돼 있다면 변경 파일 하나에서 테스트 파일 후보를 바로 찾을 수 있습니다.
단, shared utility처럼 여러 기능이 함께 쓰는 파일은 이 규칙만으로 충분하지 않을 수 있습니다.
그런 파일은 전체 실행 fallback 대상으로 분리하세요.

### H3. shared 파일 변경은 전체 테스트로 fallback한다

변경 파일 기반 테스트의 실수는 영향 범위를 과소평가할 때 생깁니다.
공통 설정, lockfile, test helper, shared library, build config가 바뀌었는데 가까운 테스트만 실행하면 중요한 회귀를 놓칠 수 있습니다.

```js
function requiresFullTest(file) {
  return (
    file === 'package.json' ||
    file === 'package-lock.json' ||
    file === 'pnpm-lock.yaml' ||
    file.startsWith('scripts/') ||
    file.startsWith('test/helpers/') ||
    file.startsWith('src/shared/')
  );
}
```

이 목록은 프로젝트마다 달라져야 합니다.
핵심은 애매한 파일을 무리하게 부분 실행으로 처리하지 않는 것입니다.
빠른 테스트는 좋은 목표지만, 테스트 선택 규칙이 신뢰를 잃으면 결국 모두가 전체 실행 버튼만 누르게 됩니다.

## Node.js test runner로 실행하기

### H3. 파일 목록을 node --test에 넘긴다

가장 작은 래퍼는 변경 파일을 읽고 테스트 파일 목록을 만든 뒤 `node --test`에 넘기는 구조입니다.

```js
// scripts/test-changed.mjs
import { spawnSync } from 'node:child_process';
import { existsSync } from 'node:fs';

const base = process.env.CI_BASE_REF || 'origin/main';
const changedFiles = getChangedFiles(base);

const full = changedFiles.some(requiresFullTest);
const selected = full
  ? []
  : unique([
      ...selectDirectTestFiles(changedFiles),
      ...selectMappedTestFiles(changedFiles)
    ]);

const args = full || selected.length === 0
  ? ['--test']
  : ['--test', ...selected];

const result = spawnSync(process.execPath, args, {
  stdio: 'inherit'
});

process.exitCode = result.status ?? 1;

function unique(values) {
  return [...new Set(values)].filter((file) => existsSync(file));
}
```

이 예시는 `selected.length === 0`일 때 전체 테스트를 실행합니다.
문서만 바뀐 PR에서 테스트를 완전히 생략하고 싶다면 이 부분을 프로젝트 정책에 맞게 바꾸면 됩니다.
다만 처음 도입할 때는 전체 fallback이 더 안전합니다.

위 코드에서 `getChangedFiles`, `requiresFullTest`, `selectDirectTestFiles`, `selectMappedTestFiles`는 앞에서 만든 함수입니다.
실무에서는 한 파일에 몰아넣기보다 `scripts/test-selection.mjs`처럼 분리해서 선택 규칙 자체를 테스트할 수 있게 두는 편이 좋습니다.

### H3. run() API를 쓰면 실행 정책을 더 세밀하게 다룰 수 있다

Node.js test runner의 `run()` API를 쓰면 파일 목록과 reporter 이벤트 처리를 JavaScript 안에서 더 직접적으로 다룰 수 있습니다.
변경 파일 기반 실행 결과를 JSON summary로 남기거나, 실패 수만 별도 출력하고 싶을 때 유용합니다.

```js
import { run } from 'node:test';

const stream = run({
  files: selected,
  isolation: 'process'
});

let failures = 0;

for await (const event of stream) {
  if (event.type === 'test:fail') {
    failures += 1;
  }
}

console.log(JSON.stringify({
  type: 'changed-test-summary',
  mode: selected.length > 0 ? 'changed' : 'full',
  testFileCount: selected.length,
  failures
}));

process.exitCode = failures > 0 ? 1 : 0;
```

여기서도 로그에는 원본 변경 파일 전체를 찍지 않았습니다.
CI 로그에는 필요한 최소 정보만 남기는 편이 안전합니다.
상세 파일 목록이 필요하다면 repository 내부 artifact로 제한하거나, 민감한 경로를 마스킹한 뒤 출력하세요.

## CI 스크립트 구성 예시

### H3. package script를 역할별로 나눈다

변경 파일 기반 테스트는 기본 테스트 명령을 대체하기보다 별도 script로 두는 편이 좋습니다.

```json
{
  "scripts": {
    "test": "node --test --test-reporter=spec",
    "test:changed": "node scripts/test-changed.mjs",
    "test:ci": "node --test --test-reporter=spec --test-reporter=junit --test-reporter-destination=stdout --test-reporter-destination=reports/node-test.xml"
  }
}
```

`test`는 로컬 기본 명령입니다.
`test:changed`는 PR의 빠른 피드백용입니다.
`test:ci`는 전체 회귀와 리포트 파일 생성을 담당합니다.
역할이 분리돼 있으면 실패했을 때 어떤 수준의 검증이 깨졌는지 바로 알 수 있습니다.

### H3. PR job과 전체 job을 함께 둔다

CI에서는 빠른 job과 안전한 job을 함께 설계합니다.

```yaml
name: test

on:
  pull_request:
  push:
    branches: [main]

jobs:
  changed:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci
      - run: npm run test:changed

  full:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci
      - run: npm run test:ci
```

이 구성은 PR에서 변경 테스트를 빠르게 보여 주고, 전체 job도 같이 유지합니다.
프로젝트 규모가 커져 전체 job 비용이 부담된다면 main branch나 nightly로 옮길 수 있지만, 중요한 배포 경로에서는 전체 회귀 테스트를 완전히 없애지 않는 편이 좋습니다.

## 운영 중 점검할 기준

### H3. false negative를 기록한다

변경 파일 기반 테스트에서 가장 중요한 지표는 놓친 회귀입니다.
부분 테스트는 통과했는데 전체 테스트에서 실패했다면, 그 변경 파일과 실패 테스트의 관계를 기록해야 합니다.
이 기록이 쌓이면 fallback 목록이나 경로 매핑 규칙을 개선할 수 있습니다.

예를 들어 `src/shared/date.js` 변경 뒤 여러 도메인 테스트가 실패했다면 해당 경로를 `requiresFullTest()`에 넣는 편이 좋습니다.
특정 디렉터리 변경이 자주 넓은 영향을 만든다면 도메인별 test suite를 따로 묶는 방법도 있습니다.

### H3. 로그에는 선택 결과만 남긴다

CI 로그는 외부 서비스, PR 코멘트, 알림 도구로 복사될 수 있습니다.
따라서 변경 파일 전체, 로컬 절대 경로, 환경 변수 원본, 내부 서비스 URL을 그대로 출력하지 않는 것이 좋습니다.

추천 로그는 아래 정도면 충분합니다.

```json
{
  "type": "changed-test-selection",
  "mode": "changed",
  "changedFileCount": 5,
  "testFileCount": 3,
  "fallback": false
}
```

실패 분석에 파일명이 꼭 필요하다면 repository 기준 상대 경로만 남기고, 사용자 홈 디렉터리나 임시 작업 디렉터리는 제거하세요.
[로그 예시 sanitization 가이드](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)에서 다룬 것처럼 예제 로그도 실제 토큰과 내부 식별자를 흉내 내지 않는 편이 안전합니다.

## 적용 체크리스트

### H3. 처음 도입할 때 확인할 것

- PR에서 base branch와 HEAD 비교가 의도대로 동작하는가?
- 테스트 파일 변경은 직접 실행되는가?
- 소스 파일에서 관련 테스트 파일을 찾는 규칙이 명확한가?
- shared/config/lockfile 변경은 전체 테스트로 fallback하는가?
- 변경 테스트가 비어 있을 때 생략할지 전체 실행할지 정했는가?
- CI 로그에 민감한 환경 변수나 절대 경로가 남지 않는가?
- 전체 회귀 테스트가 별도 job이나 일정으로 유지되는가?

### H3. 실패가 반복될 때 볼 것

부분 테스트가 자주 통과하고 전체 테스트에서만 실패한다면 선택 규칙이 너무 좁은 것입니다.
이때는 테스트 속도를 더 줄이기보다 영향 범위 모델을 먼저 고쳐야 합니다.
반대로 부분 테스트가 너무 자주 전체 fallback으로 바뀐다면 shared 경로가 과하게 넓거나 테스트 구조가 도메인별로 분리되지 않은 상태일 수 있습니다.

변경 파일 기반 테스트는 한 번에 완성되는 기능이 아닙니다.
초기에는 보수적으로 전체 fallback을 많이 두고, 실패 기록을 보면서 점진적으로 좁히는 방식이 안전합니다.

## FAQ

### H3. 변경된 테스트만 돌리면 전체 테스트는 없어도 되나요?

아니요.
변경 파일 기반 테스트는 빠른 피드백용 1차 검증입니다.
shared 코드, 설정, 의존성, 테스트 helper 변경은 영향 범위가 넓기 때문에 전체 회귀 테스트가 여전히 필요합니다.

### H3. 문서만 바뀐 PR은 테스트를 생략해도 되나요?

프로젝트 정책에 따라 가능합니다.
다만 처음에는 전체 fallback이나 짧은 smoke test를 유지하고, 문서 변경만 있는 PR이 충분히 안정적이라는 확신이 생긴 뒤 생략하는 편이 좋습니다.

### H3. test runner run() API와 CLI 중 무엇을 써야 하나요?

단순히 파일 목록만 넘기면 CLI가 충분합니다.
테스트 이벤트를 읽어 summary를 만들거나, reporter와 selection 정책을 코드로 더 세밀하게 묶고 싶다면 `run()` API가 더 적합합니다.

## 마무리

Node.js test runner로 변경 파일 기반 테스트를 만들면 PR 피드백 시간을 줄이면서도 기본 테스트 도구를 단순하게 유지할 수 있습니다.
핵심은 `git diff`로 변경 파일을 찾는 것보다, 어떤 변경을 부분 실행으로 처리하고 어떤 변경을 전체 테스트로 돌릴지 명확히 정하는 데 있습니다.

처음부터 공격적으로 테스트 범위를 줄이지 마세요.
테스트 파일 변경은 직접 실행하고, 소스 파일은 명확한 경로 규칙으로 매핑하고, shared/config 변경은 전체 실행으로 fallback하는 정도가 좋은 출발점입니다.
그 위에 reporter와 summary 로그를 얹으면 빠르면서도 설명 가능한 PR 테스트 파이프라인을 만들 수 있습니다.

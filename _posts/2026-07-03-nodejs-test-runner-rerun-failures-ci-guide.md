---
layout: post
title: "Node.js test runner rerun failures 가이드: 실패한 테스트만 다시 실행하는 CI 전략"
date: 2026-07-03 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-rerun-failures-ci-guide
permalink: /development/blog/seo/2026/07/03/nodejs-test-runner-rerun-failures-ci-guide.html
alternates:
  ko: /development/blog/seo/2026/07/03/nodejs-test-runner-rerun-failures-ci-guide.html
  x_default: /development/blog/seo/2026/07/03/nodejs-test-runner-rerun-failures-ci-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, rerun-failures, testing, javascript, ci, flaky-test]
description: "Node.js test runner의 --test-rerun-failures 옵션으로 실패한 테스트만 다시 실행하는 방법을 정리합니다. 상태 파일 관리, CI 캐시 기준, flaky 테스트 추적 원칙과 주의점을 예제로 설명합니다."
---

테스트 스위트가 커질수록 실패를 다시 확인하는 비용도 커집니다.
로컬에서는 작은 수정 하나를 검증하려고 전체 테스트를 반복 실행하게 되고, CI에서는 일시적인 환경 문제인지 실제 회귀인지 확인하느라 시간을 쓰게 됩니다.
이때 필요한 것은 실패를 숨기는 장치가 아니라, 실패한 테스트를 빠르게 다시 검증하는 운영 방식입니다.

Node.js 내장 test runner는 이런 흐름을 위해 `--test-rerun-failures` 옵션을 제공합니다.
[Node.js test runner 공식 문서](https://nodejs.org/api/test.html)에 따르면 이 옵션은 테스트 실행 상태를 JSON 파일로 저장하고, 다음 실행에서 이전에 통과한 테스트를 건너뛰어 실패한 테스트 중심으로 다시 실행할 수 있게 합니다.
다만 테스트 순서나 파일 위치가 바뀌면 이전 성공 기록을 잘못 해석할 수 있으므로, 결정적인 실행 순서와 상태 파일 관리 기준이 함께 필요합니다.

내장 테스트 러너의 기본 구조는 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 참고하세요.
실패 원인이 비동기 예외라면 [Node.js assert rejects 가이드](/development/blog/seo/2026/06/30/nodejs-assert-rejects-async-error-test-guide.html)와 함께 보면 좋습니다.
불안정한 테스트를 분리하는 정책은 [Node.js test runner tags 가이드](/development/blog/seo/2026/07/02/nodejs-test-runner-tags-filter-guide.html)에서 이어서 정리했습니다.

## rerun failures가 해결하는 문제

### H3. 전체 테스트 재실행 비용을 줄인다

테스트 실패 후 다시 실행할 때 항상 전체 스위트를 돌리면 피드백 시간이 길어집니다.
특히 통합 테스트, CLI 테스트, 파일 시스템을 많이 쓰는 테스트가 섞여 있으면 실패 하나를 확인하기 위해 이미 통과한 테스트까지 계속 반복합니다.
`--test-rerun-failures`는 이전 실행 상태를 기준으로 실패한 테스트에 집중하게 해 줍니다.

```bash
node --test --test-rerun-failures=.test-rerun.json
```

처음 실행에서는 상태 파일이 없으므로 test runner가 파일을 생성합니다.
다음 실행부터는 저장된 상태를 참고해 이전에 통과한 테스트를 건너뛰고, 실패했거나 아직 검증되지 않은 테스트를 실행합니다.
이 방식은 실패를 무시하는 것이 아니라, 다시 봐야 할 테스트를 좁히는 도구입니다.

### H3. flaky 테스트를 자동 승인하지 않는다

rerun failures는 flaky 테스트를 통과 처리하는 기능이 아닙니다.
일시적으로 실패한 테스트를 다시 실행할 수는 있지만, 왜 흔들렸는지 추적하지 않으면 같은 문제가 반복됩니다.
따라서 재실행은 원인 분석을 빠르게 시작하는 장치로 보고, 실패 기록은 CI 로그나 이슈에 남겨야 합니다.

```text
failure: payment retry test failed once
rerun: same test passed on second run
action: mark as flaky candidate, inspect timeout and external dependency
```

두 번째 실행에서 통과했다고 해서 문제를 닫아 버리면 테스트 스위트의 신뢰도가 떨어집니다.
재실행 결과는 "지금 병합해도 되는가"와 "어떤 테스트를 안정화해야 하는가"를 나눠 판단하는 데 사용하세요.
flaky 테스트는 숨기는 대상이 아니라 추적하고 줄여야 할 품질 부채입니다.

## 기본 사용법

### H3. 상태 파일 경로를 명시한다

`--test-rerun-failures`에는 상태를 저장할 파일 경로를 넘깁니다.
상태 파일은 JSON 형식이며, 테스트 실행 시도와 통과한 테스트 정보를 기록합니다.
로컬에서는 프로젝트 루트의 임시 파일로 두고, CI에서는 작업 디렉터리 안의 명확한 경로로 고정하는 편이 좋습니다.

```bash
node --test --test-rerun-failures=.cache/node-test-rerun.json
```

상태 파일 경로가 매번 달라지면 이전 실행 정보를 활용할 수 없습니다.
반대로 여러 브랜치나 서로 다른 테스트 명령이 같은 파일을 공유하면 잘못된 결과를 만들 수 있습니다.
명령의 목적마다 상태 파일을 분리하면 추적과 정리가 쉬워집니다.

### H3. package script로 재실행 명령을 고정한다

팀에서 반복해서 쓰는 명령은 `package.json` scripts에 넣는 것이 좋습니다.
로컬 기본 테스트와 재실행 테스트를 분리하면 개발자가 상황에 맞는 명령을 고르기 쉽습니다.
CI에서도 같은 script를 호출하면 로컬과 서버의 실행 방식이 크게 어긋나지 않습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:rerun": "node --test --test-rerun-failures=.cache/node-test-rerun.json"
  }
}
```

처음 실패를 확인할 때는 `npm test`로 전체 상태를 보는 흐름이 자연스럽습니다.
그다음 수정 후 빠르게 같은 실패를 확인하고 싶을 때 `npm run test:rerun`을 사용할 수 있습니다.
재실행 명령만 계속 돌리면 새로 깨진 테스트를 놓칠 수 있으므로, 병합 전에는 전체 테스트를 다시 실행해야 합니다.

### H3. 상태 파일은 보통 커밋하지 않는다

rerun 상태 파일은 테스트 기대값이 아니라 실행 기록입니다.
따라서 일반적인 프로젝트에서는 `.gitignore`에 넣고 커밋하지 않는 편이 안전합니다.
브랜치마다 테스트 파일과 순서가 달라질 수 있는데, 오래된 실행 기록이 저장소에 들어가면 오히려 혼란이 생깁니다.

```gitignore
.cache/node-test-rerun.json
.test-rerun.json
```

snapshot 파일처럼 사람이 리뷰해야 하는 기대값과 rerun 상태 파일은 역할이 다릅니다.
snapshot은 테스트의 기준값이지만, rerun 파일은 특정 실행의 이력입니다.
기준값 관리가 필요하다면 [Node.js test runner snapshot 가이드](/development/blog/seo/2026/07/02/nodejs-test-runner-snapshot-testing-guide.html)의 원칙을 따르고, rerun 파일은 임시 상태로 다루세요.

## CI에서 적용하는 방법

### H3. 같은 job 안에서 실패 재확인을 빠르게 한다

가장 단순한 CI 전략은 전체 테스트가 실패했을 때 같은 job 안에서 rerun 명령을 한 번 더 실행하는 것입니다.
이 방식은 일시적인 실패를 빠르게 재확인할 수 있지만, 두 번째 실행이 통과했다고 해서 실패 기록을 없애서는 안 됩니다.
로그에는 첫 번째 실패와 두 번째 결과가 모두 남아야 합니다.

```yaml
name: test

on:
  pull_request:

jobs:
  node-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 26
      - run: npm ci
      - run: mkdir -p .cache
      - name: Run tests
        run: npm test || npm run test:rerun
```

이 구성은 짧고 이해하기 쉽지만 한계도 있습니다.
첫 번째 실패가 실제 회귀였는데 두 번째 실행에서 우연히 통과하면 원인을 놓칠 수 있습니다.
중요한 테스트에서는 별도 로그 수집이나 flaky 후보 라벨링을 함께 운영하는 편이 좋습니다.

### H3. 상태 파일 캐시는 신중하게 다룬다

CI 캐시를 사용하면 job 사이에서도 rerun 상태를 유지할 수 있습니다.
하지만 테스트 파일, Node.js 버전, 브랜치가 달라진 상태에서 같은 캐시를 쓰면 이전 성공 기록이 현재 실행을 오염시킬 수 있습니다.
공식 문서가 경고하듯, 테스트 실행 순서나 파일 위치가 바뀌면 이전 기록을 잘못 해석할 수 있습니다.

```yaml
- uses: actions/cache@v4
  with:
    path: .cache/node-test-rerun.json
    key: node-test-rerun-${{ runner.os }}-${{ hashFiles('test/**/*.js', 'src/**/*.js', 'package-lock.json') }}
```

캐시 key에는 테스트와 제품 코드, lockfile처럼 실행 결과에 영향을 주는 입력을 포함하세요.
그래도 장기 캐시보다 같은 workflow 안의 재실행에만 쓰는 편이 단순하고 안전한 경우가 많습니다.
rerun 상태가 테스트 선택에 영향을 주는 만큼, 캐시 범위는 보수적으로 잡아야 합니다.

## 테스트 순서와 결정성

### H3. 랜덤 실행과 함께 쓰는 기준을 정한다

테스트 순서를 랜덤화하면 순서 의존 버그를 찾는 데 도움이 됩니다.
하지만 rerun 상태 파일은 이전 실행의 성공 정보를 바탕으로 동작하므로, 실행 순서가 매번 크게 달라지면 해석이 어려워질 수 있습니다.
두 기능을 함께 쓸 때는 seed와 상태 파일을 함께 고정하거나, 서로 다른 CI 단계로 분리하는 편이 낫습니다.

```bash
node --test --test-randomize --test-random-seed=20260703
```

순서 의존 버그를 찾는 목적이라면 randomize 전용 job을 두세요.
실패 재확인이 목적이라면 deterministic 실행을 유지하세요.
하나의 명령에 모든 목적을 섞으면 실패 원인을 읽기 어려워집니다.

### H3. 테스트 이름과 파일 이동은 상태 파일에 영향을 준다

rerun 상태는 테스트가 어디에서 어떤 이름으로 실행됐는지에 의존합니다.
테스트 파일을 옮기거나 이름을 바꾸면 기존 상태 파일이 더 이상 정확하지 않을 수 있습니다.
큰 리팩터링 뒤에는 상태 파일을 지우고 전체 테스트를 다시 실행하는 편이 안전합니다.

```bash
rm -f .cache/node-test-rerun.json
node --test
```

상태 파일 삭제는 테스트 기준을 지우는 것이 아닙니다.
오래된 실행 이력을 버리고 현재 코드 기준으로 다시 측정하는 과정입니다.
테스트 구조를 바꾼 PR에서는 rerun보다 전체 실행 결과를 우선 확인하세요.

## flaky 테스트 운영 원칙

### H3. 재실행 횟수보다 원인 분류가 중요하다

재실행 횟수를 늘리면 겉으로 보이는 실패율은 줄어들 수 있습니다.
하지만 네트워크 의존, 시간 의존, 순서 의존, 외부 서비스 의존 중 무엇이 문제인지 분류하지 않으면 근본적인 안정성은 좋아지지 않습니다.
rerun은 실패를 빨리 좁히는 도구이고, 안정화는 별도의 작업입니다.

```text
flaky cause candidates
- time: real timers, timezone, date boundary
- order: shared global state, uncleaned temp files
- external: network, database, rate limit
- concurrency: port collision, parallel write conflict
```

시간 의존 문제가 의심된다면 [Node.js test runner mock timers 가이드](/development/blog/seo/2026/07/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html)를 참고해 실제 대기 시간을 줄이세요.
병렬 실행 충돌이 의심된다면 concurrency 설정과 테스트 격리 기준을 다시 봐야 합니다.
재실행이 반복되는 테스트는 반드시 별도 이슈로 남기는 습관이 필요합니다.

### H3. 실패 기록을 리뷰 가능한 형태로 남긴다

CI가 재실행 후 통과하더라도 첫 번째 실패 로그는 리뷰어가 볼 수 있어야 합니다.
테스트가 실제로 흔들렸다는 사실이 사라지면, 팀은 테스트 스위트의 건강 상태를 판단할 수 없습니다.
아티팩트, job summary, 이슈 코멘트 중 하나로 실패 기록을 남기면 후속 정리가 쉬워집니다.

```bash
node --test --test-reporter=spec 2>&1 | tee test-first-run.log
```

로그를 남길 때는 민감정보가 출력되지 않도록 주의해야 합니다.
테스트 환경 변수, 인증 토큰, 실제 사용자 데이터가 실패 메시지에 섞이지 않게 fixture와 mock 데이터를 점검하세요.
개발 편의를 위한 로그가 보안 사고로 이어지지 않게 하는 것도 테스트 운영의 일부입니다.

## 실무 체크리스트

### H3. 도입 전에 확인할 것

- `--test-rerun-failures`를 지원하는 Node.js 버전을 로컬과 CI에서 함께 쓰는가?
- 상태 파일 경로가 명령 목적별로 분리되어 있는가?
- rerun 상태 파일을 저장소에 커밋하지 않도록 `.gitignore`를 설정했는가?
- 테스트 파일 이동, 이름 변경, 랜덤 실행이 상태 파일 해석에 미치는 영향을 팀이 알고 있는가?
- 재실행 후 통과한 flaky 후보를 추적할 방법이 있는가?
- 병합 전에는 전체 테스트를 다시 실행하는 기준을 유지하는가?

`--test-rerun-failures`는 큰 테스트 스위트에서 실패 확인 시간을 줄여 주는 실용적인 옵션입니다.
하지만 이 기능의 가치는 실패를 덮는 데 있지 않고, 실패를 더 빨리 다시 보고 원인을 분류하는 데 있습니다.
상태 파일을 임시 실행 기록으로 다루고, CI에서는 로그와 추적 기준을 함께 남기면 빠른 피드백과 테스트 신뢰도를 동시에 지킬 수 있습니다.

## 함께 보면 좋은 글

- [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)
- [Node.js test runner tags 가이드: 테스트 그룹을 태그로 나누고 CI에서 필터링하는 법](/development/blog/seo/2026/07/02/nodejs-test-runner-tags-filter-guide.html)
- [Node.js test runner mock timers 가이드: 시간 의존 테스트를 빠르고 안정적으로 만드는 법](/development/blog/seo/2026/07/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html)

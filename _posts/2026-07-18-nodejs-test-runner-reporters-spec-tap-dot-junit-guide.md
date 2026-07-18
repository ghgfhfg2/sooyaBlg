---
layout: post
title: "Node.js test runner reporter 가이드: spec, tap, dot, junit 출력 선택 기준"
date: 2026-07-18 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-reporters-spec-tap-dot-junit-guide
permalink: /development/blog/seo/2026/07/18/nodejs-test-runner-reporters-spec-tap-dot-junit-guide.html
alternates:
  ko: /development/blog/seo/2026/07/18/nodejs-test-runner-reporters-spec-tap-dot-junit-guide.html
  x_default: /development/blog/seo/2026/07/18/nodejs-test-runner-reporters-spec-tap-dot-junit-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, reporter, tap, junit, ci, testing, javascript]
description: "Node.js test runner의 spec, tap, dot, junit, lcov reporter를 상황별로 선택하는 기준을 정리합니다. 로컬 개발, CI 로그, 테스트 리포트 파일, 민감정보 점검까지 예제로 설명합니다."
---

테스트는 통과 여부만큼 출력 형식도 중요합니다.
로컬에서는 실패한 테스트 이름과 assertion diff를 빠르게 읽어야 하고, CI에서는 로그가 너무 길어지지 않아야 하며, 품질 대시보드나 PR 코멘트 도구는 JUnit XML 같은 기계가 읽기 좋은 파일을 요구할 수 있습니다.
같은 테스트라도 어디에서 실행하느냐에 따라 좋은 reporter가 달라집니다.

Node.js test runner는 `--test-reporter` 옵션과 `node:test/reporters` 모듈을 통해 `spec`, `tap`, `dot`, `junit`, `lcov` 같은 built-in reporter를 제공합니다.
이 글에서는 각 reporter의 목적, 로컬과 CI에서의 선택 기준, `--test-reporter-destination`으로 출력 파일을 나누는 방법, 그리고 테스트 로그에 민감정보가 섞이지 않게 점검하는 기준을 정리합니다.
[Node.js test runner custom reporter 가이드](/development/blog/seo/2026/07/12/nodejs-test-runner-custom-reporter-ci-report-guide.html), [Node.js test runner coverage lcov CI 가이드](/development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html), [Node.js test runner workerId와 attempt 가이드](/development/blog/seo/2026/07/18/nodejs-test-runner-workerid-attempt-flaky-ci-guide.html)와 함께 보면 테스트 결과 수집 흐름을 더 안정적으로 설계할 수 있습니다.

## Node.js test runner reporter가 필요한 이유

### H3. 사람용 출력과 시스템용 출력은 다르다

로컬 개발자는 보통 실패 원인을 바로 읽고 싶습니다.
어떤 테스트가 실패했는지, 기대값과 실제값이 어떻게 다른지, 어느 파일을 열어야 하는지가 중요합니다.
이 목적에는 사람이 읽기 좋은 `spec` reporter가 잘 맞습니다.

반면 CI 시스템은 사람이 보는 화면만 상대하지 않습니다.
테스트 결과를 PR 상태 검사, 테스트 히스토리, flaky test 탐지, 커버리지 리포트와 연결해야 합니다.
이때는 `junit`이나 `lcov`처럼 파일로 저장하고 도구가 읽을 수 있는 형식이 더 유용합니다.

따라서 reporter 선택은 취향 문제가 아니라 소비자 문제입니다.
출력을 읽는 대상이 사람인지, CI 도구인지, 커버리지 분석기인지 먼저 정하면 선택이 쉬워집니다.

### H3. 기본 출력에만 의존하면 운영 단서가 부족해진다

작은 프로젝트에서는 기본 reporter만으로도 충분합니다.
하지만 테스트 수가 늘고 병렬 CI, shard, 재시도, coverage threshold를 함께 쓰기 시작하면 출력 목적을 나눌 필요가 생깁니다.

- 로컬 실행은 실패 원인을 빠르게 보여준다.
- PR CI 로그는 너무 길지 않게 유지한다.
- 테스트 결과 파일은 CI artifact로 보관한다.
- 커버리지 결과는 별도 도구가 읽을 수 있게 저장한다.
- flaky test 분석에 필요한 최소 메타데이터를 남긴다.

Node.js test runner reporter 옵션은 이 흐름을 분리하는 출발점입니다.
한 화면에 모든 정보를 쏟아내기보다, 사람용과 시스템용 결과를 나누는 편이 오래 운영하기 좋습니다.

## built-in reporter 선택 기준

### H3. spec은 로컬 개발과 일반 CI 로그에 적합하다

`spec` reporter는 테스트 결과를 사람이 읽기 좋은 형태로 보여 줍니다.
Node.js 공식 문서 기준으로 기본 reporter이며, 테스트 이름과 성공/실패 흐름을 읽기 쉽게 표현합니다.
대부분의 로컬 실행과 일반적인 CI 로그에는 `spec`을 먼저 선택하면 됩니다.

```bash
node --test --test-reporter=spec
```

추천 상황은 다음과 같습니다.

- 로컬에서 실패 원인을 읽으며 개발할 때
- PR CI 로그에서 실패한 테스트 이름을 바로 확인해야 할 때
- 테스트 수가 아직 많지 않아 출력량이 부담되지 않을 때
- 별도 테스트 결과 업로더가 필요하지 않을 때

단점도 있습니다.
사람이 읽기 좋은 출력은 버전별 세부 표현이 달라질 수 있으므로 프로그램이 파싱할 대상으로 삼지 않는 편이 좋습니다.
테스트 결과를 자동 처리해야 한다면 reporter 출력 문자열을 정규식으로 읽기보다 JUnit XML이나 test stream 기반 custom reporter를 쓰세요.

### H3. dot은 큰 테스트 묶음에서 로그를 줄일 때 좋다

`dot` reporter는 통과한 테스트를 점 하나로, 실패한 테스트를 별도 표시로 보여 주는 압축 출력입니다.
테스트 수가 많고 CI 로그 길이를 줄이고 싶을 때 유용합니다.

```bash
node --test --test-reporter=dot
```

`dot`은 "전체가 대체로 잘 돌고 있는지"를 빠르게 보는 데 적합합니다.
다만 실패 원인을 자세히 읽는 목적에는 부족할 수 있습니다.
실패가 났을 때는 CI artifact에 `junit` 결과를 같이 남기거나, 실패 재현 단계에서 `spec`으로 다시 실행하는 흐름을 준비해 두는 편이 좋습니다.

예를 들어 PR마다 빠른 상태 확인에는 `dot`을 쓰고, nightly나 main branch에서는 `junit`과 coverage를 함께 저장하는 식으로 나눌 수 있습니다.

### H3. tap은 TAP 생태계와 연결할 때 선택한다

`tap` reporter는 TAP 형식으로 테스트 결과를 출력합니다.
기존 인프라가 TAP을 읽거나, TAP 기반 리포트 도구와 연결해야 한다면 선택할 수 있습니다.

```bash
node --test --test-reporter=tap
```

TAP은 오래된 테스트 출력 형식이라 다양한 도구와 연결하기 쉽습니다.
하지만 모든 팀이 TAP을 직접 읽는 것은 아니므로, 사람용 로그로는 `spec`이 더 편할 때가 많습니다.
이미 TAP 수집기가 있는 조직에서는 `tap`을 시스템용 출력으로 쓰고, 개발자 로컬 명령은 `spec`으로 유지하는 구성이 자연스럽습니다.

### H3. junit은 CI 테스트 리포트 파일에 적합하다

`junit` reporter는 JUnit XML 형식의 결과를 만듭니다.
GitHub Actions, GitLab CI, Jenkins, CircleCI, Buildkite 같은 CI 환경에서 테스트 결과 파일을 업로드하거나 대시보드로 보여 줄 때 자주 쓰입니다.

```bash
node --test --test-reporter=junit --test-reporter-destination=reports/node-test.xml
```

JUnit XML은 사람이 터미널에서 읽기 위한 형식은 아닙니다.
대신 실패한 테스트 이름, suite, duration, error message를 CI 도구가 구조화해서 보여 주기 좋습니다.
테스트 실패를 장기적으로 추적하려면 `junit` 결과를 artifact로 보관하는 편이 유리합니다.

주의할 점은 테스트 이름입니다.
`describe()`와 `it()` 이름이 너무 짧거나 중복되면 JUnit 리포트에서도 식별이 어렵습니다.
`returns 400 when email is invalid`처럼 조건과 기대 결과를 함께 담아야 실패 목록을 바로 읽을 수 있습니다.

### H3. lcov는 coverage 리포트와 함께 쓴다

`lcov` reporter는 테스트 커버리지 결과를 LCOV 형식으로 출력할 때 사용합니다.
공식 문서 기준으로 `--experimental-test-coverage`와 함께 쓰는 reporter입니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-reporter=lcov \
  --test-reporter-destination=coverage/lcov.info
```

LCOV는 사람이 직접 읽기보다 coverage 서비스나 HTML 리포트 생성기가 소비하는 형식입니다.
따라서 터미널에는 `spec` 또는 `dot`을 보여 주고, coverage 파일은 별도 destination으로 저장하는 구성이 더 실용적입니다.

[coverage threshold 가이드](/development/blog/seo/2026/07/08/nodejs-test-runner-coverage-threshold-ci-guide.html)처럼 최소 기준을 함께 걸면 단순 측정에서 벗어나 품질 게이트로 운영할 수 있습니다.

## reporter destination으로 출력 나누기

### H3. 터미널과 파일을 분리한다

테스트 실행에서 자주 필요한 패턴은 "터미널에는 사람이 읽기 쉬운 출력, 파일에는 CI가 읽을 결과"를 동시에 남기는 것입니다.
Node.js test runner는 reporter를 여러 번 지정하고 각각 destination을 둘 수 있습니다.

```bash
mkdir -p reports

node --test \
  --test-reporter=spec \
  --test-reporter=junit \
  --test-reporter-destination=stdout \
  --test-reporter-destination=reports/node-test.xml
```

이 구성에서는 첫 번째 reporter인 `spec`이 첫 번째 destination인 `stdout`으로 나가고, 두 번째 reporter인 `junit`이 두 번째 destination 파일로 저장됩니다.
옵션 순서가 짝을 이루므로 팀 스크립트에서는 줄바꿈을 맞춰 실수를 줄이는 것이 좋습니다.

### H3. npm scripts에 역할별 명령을 둔다

모든 상황을 하나의 `npm test`에 몰아넣으면 로컬 개발과 CI 요구가 충돌합니다.
역할별 script를 나눠 두면 reporter 선택도 명확해집니다.

```json
{
  "scripts": {
    "test": "node --test --test-reporter=spec",
    "test:ci": "mkdir -p reports && node --test --test-reporter=spec --test-reporter=junit --test-reporter-destination=stdout --test-reporter-destination=reports/node-test.xml",
    "test:coverage": "mkdir -p coverage && node --test --experimental-test-coverage --test-reporter=lcov --test-reporter-destination=coverage/lcov.info"
  }
}
```

이렇게 나누면 개발자는 기본 명령으로 빠르게 실패를 읽고, CI는 artifact에 필요한 파일을 안정적으로 수집할 수 있습니다.
테스트 환경이 커질수록 명령 이름이 문서 역할을 하므로 `test:ci`, `test:coverage`, `test:watch`처럼 목적이 드러나게 유지하세요.

## CI에서 reporter를 운영하는 흐름

### H3. PR에서는 읽기 쉬운 실패 로그를 우선한다

PR 검증에서는 리뷰어가 실패 원인을 빨리 확인해야 합니다.
너무 많은 JSON, XML, coverage summary가 터미널에 섞이면 오히려 디버깅이 느려집니다.

PR CI의 기본값은 다음처럼 단순하게 시작해도 충분합니다.

```bash
npm run test:ci
```

그리고 CI 설정에서 `reports/node-test.xml`을 artifact나 테스트 리포트로 업로드합니다.
터미널에는 `spec` 출력이 남고, CI UI에는 JUnit 결과가 남는 구조입니다.

### H3. main branch나 nightly에서는 분석 파일을 남긴다

모든 PR에서 coverage와 상세 리포트를 강하게 돌리면 시간이 늘어날 수 있습니다.
프로젝트 규모에 따라 main branch merge 후나 nightly workflow에서 coverage, JUnit, flaky test 분석 파일을 더 오래 보관하는 방법도 좋습니다.

```bash
npm run test:ci
npm run test:coverage
```

이때 중요한 것은 "실패한 뒤에야 필요한 파일이 없다는 것을 깨닫지 않는 것"입니다.
테스트 리포트, coverage, shard별 로그, 재시도 로그는 CI가 실패해도 업로드되도록 `always()` 조건을 거는 편이 안전합니다.

### H3. shard와 reporter 파일 이름을 함께 설계한다

테스트를 shard로 나누면 JUnit 파일도 shard별로 분리해야 합니다.
모든 shard가 같은 `reports/node-test.xml`에 쓰면 마지막 작업이 이전 결과를 덮어쓸 수 있습니다.

```bash
mkdir -p reports

node --test \
  --test-shard="${SHARD_INDEX}/${SHARD_TOTAL}" \
  --test-reporter=spec \
  --test-reporter=junit \
  --test-reporter-destination=stdout \
  --test-reporter-destination="reports/node-test-shard-${SHARD_INDEX}.xml"
```

파일명에는 shard 번호, Node.js 버전, OS 이름처럼 결과를 구분하는 값을 넣을 수 있습니다.
다만 민감한 runner 내부 경로나 토큰, 브랜치 보호 규칙처럼 외부에 노출할 필요가 없는 값은 넣지 마세요.

## reporter 출력에 민감정보가 섞이지 않게 하는 기준

### H3. 테스트 이름과 diagnostic 메시지도 로그다

reporter는 테스트 결과만 출력하는 것처럼 보이지만, 실제로는 테스트 이름, 에러 메시지, diagnostic, stdout, stderr를 함께 전달할 수 있습니다.
따라서 테스트 안에서 출력한 값이 그대로 CI 로그나 JUnit XML에 들어갈 수 있습니다.

피해야 할 예시는 다음과 같습니다.

- 실제 이메일, 전화번호, 주소처럼 보이는 fixture를 테스트 이름에 넣는다.
- API token, session cookie, authorization header를 에러 메시지에 포함한다.
- 요청/응답 body 전체를 `console.log()`로 남긴다.
- 임시 디렉터리 절대 경로를 그대로 diagnostic에 넣는다.
- 외부 서비스 계정명이나 내부 호스트명을 snapshot에 저장한다.

테스트 로그는 실패 순간에 가장 넓게 공유되기 쉽습니다.
PR 댓글, CI artifact, 알림 시스템, 로그 수집기로 복사될 수 있으므로 공개 가능한 최소 정보만 남기는 기준이 필요합니다.

### H3. 안전한 diagnostic helper를 만든다

테스트별로 아무 값이나 출력하게 두면 시간이 갈수록 로그 품질이 흔들립니다.
허용 필드를 제한한 helper를 만들면 reporter가 어떤 형식이든 안전한 메타데이터만 남길 수 있습니다.

```js
// test/helpers/diagnostic.mjs
const ALLOWED_KEYS = new Set([
  'caseId',
  'fixture',
  'reason',
  'workerId',
  'attempt'
]);

export function diagnostic(t, fields) {
  const safe = {};

  for (const [key, value] of Object.entries(fields)) {
    if (!ALLOWED_KEYS.has(key)) continue;
    safe[key] = String(value).slice(0, 120);
  }

  t.diagnostic(JSON.stringify({
    test: t.fullName,
    file: t.filePath?.replace(process.cwd(), '<repo>'),
    ...safe
  }));
}
```

이 helper는 완벽한 보안 장치가 아닙니다.
하지만 "무엇을 남길 수 있는가"를 코드로 제한한다는 점에서 로그 사고 가능성을 줄여 줍니다.
민감정보가 들어갈 수 있는 payload는 helper 입력으로 받지 않는 방식이 가장 단순합니다.

## 실무 선택표

### H3. 상황별 기본 조합

아래 조합을 출발점으로 삼으면 대부분의 프로젝트에서 무리 없이 운영할 수 있습니다.

| 상황 | 추천 reporter | destination | 목적 |
| --- | --- | --- | --- |
| 로컬 개발 | `spec` | `stdout` | 실패 원인을 빠르게 읽기 |
| 큰 테스트 묶음 | `dot` | `stdout` | CI 로그 줄이기 |
| PR CI | `spec` + `junit` | `stdout` + XML 파일 | 사람용 로그와 시스템용 리포트 분리 |
| TAP 도구 연동 | `tap` | 파일 또는 stdout | 기존 TAP 파이프라인 유지 |
| 커버리지 수집 | `lcov` | `coverage/lcov.info` | coverage 서비스와 연동 |

처음에는 `spec`과 `junit` 조합만으로 충분합니다.
그 다음 테스트 수가 늘면 `dot`, coverage가 중요해지면 `lcov`, 조직에 기존 TAP 도구가 있으면 `tap`을 추가하면 됩니다.

### H3. reporter 선택보다 중요한 운영 규칙

reporter 자체보다 중요한 것은 일관성입니다.
어떤 명령이 어떤 출력 파일을 만들고, CI가 그 파일을 어디에 업로드하며, 실패했을 때 개발자가 어디부터 보면 되는지 정해져 있어야 합니다.

운영 규칙은 짧게 유지하세요.

- 로컬 기본 명령은 `spec`으로 둔다.
- CI는 JUnit XML을 항상 artifact로 남긴다.
- coverage는 별도 명령과 별도 파일로 분리한다.
- shard를 쓰면 shard별 파일명을 분리한다.
- 테스트 로그에는 payload 원문을 남기지 않는다.

이 다섯 가지가 지켜지면 reporter를 바꾸더라도 테스트 결과를 읽는 흐름은 크게 흔들리지 않습니다.

## 발행 전 점검 체크리스트

### H3. Node.js test runner reporter 적용 전 확인할 것

- `npm test`는 로컬 개발자가 읽기 쉬운 출력인가?
- CI는 JUnit XML이나 필요한 테스트 리포트 파일을 저장하는가?
- coverage 출력과 일반 테스트 출력이 섞이지 않는가?
- shard, OS, Node.js 버전별 파일명이 서로 덮어쓰지 않는가?
- 테스트 이름이 JUnit 리포트에서도 식별 가능할 만큼 구체적인가?
- `console.log()`, `t.diagnostic()`, assertion message에 민감정보가 섞이지 않는가?

reporter는 테스트의 신뢰도를 직접 높여 주지는 않습니다.
하지만 실패한 테스트를 더 빨리 읽고, CI 결과를 더 오래 추적하고, 민감한 로그가 퍼지는 일을 줄여 줍니다.
작은 프로젝트라도 로컬용 `spec`, CI용 `junit`, coverage용 `lcov`를 분리해 두면 테스트 운영이 훨씬 단단해집니다.

---
layout: post
title: "Node.js test runner custom reporter 가이드: CI 로그와 리포트를 목적별로 나누는 법"
date: 2026-07-12 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-custom-reporter-ci-report-guide
permalink: /development/blog/seo/2026/07/12/nodejs-test-runner-custom-reporter-ci-report-guide.html
alternates:
  ko: /development/blog/seo/2026/07/12/nodejs-test-runner-custom-reporter-ci-report-guide.html
  x_default: /development/blog/seo/2026/07/12/nodejs-test-runner-custom-reporter-ci-report-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, custom-reporter, ci, testing, javascript, reporter, observability]
description: "Node.js test runner custom reporter로 CI 로그와 테스트 리포트를 목적별로 나누는 방법을 정리합니다. --test-reporter, --test-reporter-destination, Transform 기반 reporter, 로그 위생 기준을 예제로 설명합니다."
---

테스트가 실패했을 때 개발자가 먼저 보는 것은 코드가 아니라 로그입니다.
테스트 자체가 잘 작성되어 있어도 CI 출력이 너무 길거나, 필요한 정보가 파일로 남지 않거나, 민감한 값이 그대로 찍히면 문제를 찾는 시간이 늘어납니다.
Node.js 내장 test runner의 reporter 설정은 이 지점을 다루는 실용적인 도구입니다.

기본 reporter만으로도 작은 프로젝트는 충분합니다.
하지만 팀에서 CI를 운영하면 콘솔에는 사람이 읽을 요약을 남기고, 파일에는 도구가 읽을 구조화된 결과를 저장하고, 필요하면 프로젝트 규칙에 맞는 custom reporter를 붙이고 싶어집니다.
Node.js test runner는 `--test-reporter`와 `--test-reporter-destination` 옵션을 제공하며, 공식 문서에서도 custom reporter를 stream 기반 모듈로 연결할 수 있다고 설명합니다.

이 글에서는 Node.js test runner custom reporter를 언제 쓰면 좋은지, built-in reporter와 목적별 출력을 어떻게 조합하는지, 간단한 Transform 기반 reporter를 만들 때 어떤 기준을 지켜야 하는지 정리합니다.
테스트 실행 범위를 줄이는 전략은 [Node.js test runner name pattern 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html), CI 분산 실행은 [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html), 커버리지 리포트는 [Node.js test runner coverage lcov 가이드](/development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html)와 함께 보면 좋습니다.

## Node.js test runner reporter가 필요한 이유

### H3. 콘솔 로그와 저장 리포트의 목적이 다르다

CI 콘솔은 사람이 빠르게 읽는 공간입니다.
실패한 테스트 이름, 에러 메시지, 재현 명령처럼 바로 판단할 수 있는 정보가 중요합니다.
반대로 테스트 결과 파일은 도구가 읽는 공간입니다.
대시보드, 품질 게이트, 실패 추세 분석, PR 주석 생성 같은 자동화는 구조화된 결과를 선호합니다.

이 둘을 같은 출력으로 해결하려 하면 어느 쪽도 만족하기 어렵습니다.
콘솔에 모든 이벤트를 자세히 찍으면 사람이 읽기 힘들고, 파일에 사람이 읽기 좋은 문장을 저장하면 후속 자동화가 어렵습니다.
reporter를 분리하면 각 출력의 역할을 명확하게 정할 수 있습니다.

```bash
node --test \
  --test-reporter=spec \
  --test-reporter-destination=stdout \
  --test-reporter=tap \
  --test-reporter-destination=./reports/test-results.tap
```

위 명령은 콘솔에는 `spec` 형식으로 보기 좋은 결과를 출력하고, 파일에는 TAP 형식 결과를 저장합니다.
프로젝트에서 별도 리포트 파서를 쓰거나 장기 보관이 필요하다면 이런 식으로 목적을 나누는 편이 운영하기 쉽습니다.

### H3. custom reporter는 팀의 테스트 운영 규칙을 담는다

custom reporter는 단순히 예쁜 로그를 만들기 위한 기능이 아닙니다.
팀이 중요하게 보는 테스트 정보를 일정한 형식으로 남기는 장치입니다.

예를 들어 다음 요구가 있다면 custom reporter를 검토할 수 있습니다.

- 실패한 테스트 이름과 파일 경로만 별도 요약으로 남긴다.
- flaky 의심 테스트를 태그나 이름 규칙으로 모아 별도 로그에 기록한다.
- CI 시스템이 읽기 쉬운 JSON Lines 형식으로 테스트 이벤트를 저장한다.
- 내부 대시보드가 요구하는 필드명으로 결과를 변환한다.

다만 reporter가 테스트 성공과 실패의 의미를 바꾸면 안 됩니다.
reporter는 결과를 표현하고 전달하는 계층이지, 테스트 정책 자체를 우회하는 계층이 아닙니다.
실패를 숨기거나 민감정보를 자세히 남기는 reporter는 장기적으로 신뢰를 해칩니다.

## built-in reporter부터 조합하기

### H3. 먼저 기본 reporter로 충분한지 확인한다

custom reporter를 만들기 전에는 built-in reporter 조합으로 해결되는지 확인하는 것이 좋습니다.
대부분의 프로젝트는 사람이 읽을 출력과 도구가 읽을 파일을 나누는 것만으로도 충분합니다.

```json
{
  "scripts": {
    "test": "node --test --test-reporter=spec",
    "test:ci": "node --test --test-reporter=spec --test-reporter-destination=stdout --test-reporter=tap --test-reporter-destination=./reports/test-results.tap"
  }
}
```

로컬에서는 짧고 읽기 좋은 `spec` 출력만 사용합니다.
CI에서는 같은 `spec` 출력을 콘솔에 남기면서 TAP 파일도 저장합니다.
이렇게 나누면 로컬 개발 경험과 CI 자동화 요구가 충돌하지 않습니다.

```bash
mkdir -p reports
npm run test:ci
```

리포트 디렉터리를 미리 만드는 단계도 중요합니다.
테스트 자체가 아니라 결과 저장 경로 문제로 CI가 실패하면 원인을 찾는 데 불필요한 시간이 듭니다.

### H3. reporter destination은 명시적으로 관리한다

여러 reporter를 붙일 때는 reporter와 destination의 순서를 의식해야 합니다.
각 reporter가 어디로 출력되는지 명령만 보고 알 수 있어야 합니다.

```bash
node --test \
  --test-reporter=spec \
  --test-reporter-destination=stdout \
  --test-reporter=json \
  --test-reporter-destination=./reports/test-results.json
```

이런 명령은 CI 로그를 읽는 사람에게도 의도가 분명합니다.
`spec`은 표준 출력, `json`은 파일 저장입니다.
반대로 destination을 생략하거나 여러 줄 스크립트에 섞어두면 결과가 어디로 가는지 추적하기 어려워집니다.

운영 기준은 간단합니다.
콘솔 출력은 실패 원인을 빠르게 찾는 데 필요한 만큼만 남기고, 장기 분석이나 도구 연동은 파일로 분리합니다.
테스트 로그가 길어지는 문제를 reporter로 해결하려면 먼저 출력의 독자를 나누어야 합니다.

## Transform 기반 custom reporter 만들기

### H3. test event를 받아 필요한 정보만 변환한다

Node.js test runner의 custom reporter는 테스트 이벤트 스트림을 받아 다른 출력으로 바꾸는 방식으로 만들 수 있습니다.
간단한 예로 실패 이벤트만 JSON Lines 형태로 남기는 reporter를 만들 수 있습니다.

```js
// reporters/failure-summary.js
import { Transform } from 'node:stream';

export default new Transform({
  writableObjectMode: true,
  transform(event, encoding, callback) {
    if (event.type !== 'test:fail') {
      callback();
      return;
    }

    const payload = {
      type: event.type,
      name: event.data?.name,
      file: event.data?.file,
      message: sanitizeMessage(event.data?.details?.error?.message)
    };

    callback(null, `${JSON.stringify(payload)}\n`);
  }
});

function sanitizeMessage(message) {
  if (!message) return undefined;

  return String(message)
    .replaceAll(/Bearer\s+[A-Za-z0-9._-]+/g, 'Bearer [REDACTED]')
    .replaceAll(/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/g, '[EMAIL]');
}
```

이 reporter는 모든 이벤트를 출력하지 않습니다.
실패 이벤트만 골라 이름, 파일, 메시지를 남깁니다.
또 에러 메시지에 토큰이나 이메일이 섞여 들어올 수 있으므로 간단한 마스킹을 적용합니다.

실행 명령은 다음처럼 구성할 수 있습니다.

```bash
mkdir -p reports
node --test \
  --test-reporter=spec \
  --test-reporter-destination=stdout \
  --test-reporter=./reporters/failure-summary.js \
  --test-reporter-destination=./reports/failures.ndjson
```

콘솔에는 사람이 읽는 결과가 남고, `reports/failures.ndjson`에는 실패 요약만 저장됩니다.
나중에 이 파일을 PR 주석, 알림, 내부 대시보드 입력으로 연결할 수 있습니다.

### H3. reporter 안에서 무거운 일을 하지 않는다

custom reporter는 테스트 실행 흐름 가까이에 붙습니다.
그래서 reporter 안에서 네트워크 요청, 대용량 파일 처리, 복잡한 통계를 직접 수행하면 테스트 실행이 느려지거나 불안정해질 수 있습니다.

좋은 reporter는 이벤트를 가볍게 변환하고 저장합니다.
후처리가 필요하다면 테스트가 끝난 뒤 별도 스크립트가 리포트 파일을 읽어 처리하는 편이 안전합니다.

```json
{
  "scripts": {
    "test:ci": "mkdir -p reports && node --test --test-reporter=spec --test-reporter-destination=stdout --test-reporter=./reporters/failure-summary.js --test-reporter-destination=./reports/failures.ndjson",
    "test:report": "node scripts/publish-test-report.js ./reports/failures.ndjson"
  }
}
```

이 구조에서는 test runner가 테스트와 기본 리포트 생성을 담당합니다.
알림 발송이나 대시보드 업로드는 별도 단계가 맡습니다.
문제가 생겼을 때 실패 지점을 나누어 볼 수 있고, reporter 자체의 책임도 작게 유지됩니다.

## CI에서 custom reporter를 운영하는 기준

### H3. 리포트 파일은 산출물로 보관한다

CI에서 파일 reporter를 사용한다면 결과 파일을 artifact로 보관하는 흐름까지 함께 설계해야 합니다.
파일을 생성해도 job이 끝나며 사라지면 디버깅 가치가 줄어듭니다.

```yaml
name: test

on:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run test:ci
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: node-test-reports
          path: reports/
```

`if: always()`를 붙이면 테스트가 실패해도 리포트 파일을 업로드할 수 있습니다.
실패한 job에서 리포트가 가장 필요한데, 기본 조건 때문에 업로드가 건너뛰어지면 reporter를 붙인 의미가 줄어듭니다.

Node.js 버전도 명시해야 합니다.
test runner 기능은 Node.js 버전에 따라 지원 범위가 달라질 수 있으므로 로컬, CI, 문서의 버전을 맞추는 것이 좋습니다.

### H3. 로그 위생을 reporter 요구사항에 포함한다

테스트 리포트는 여러 사람이 접근하는 CI, artifact 저장소, 알림 시스템으로 퍼질 수 있습니다.
따라서 reporter 설계에는 로그 위생 기준이 반드시 들어가야 합니다.

다음 값은 그대로 남기지 않는 편이 안전합니다.

- API 키, bearer token, session id
- 실제 사용자 이메일, 전화번호, 주소
- 결제 식별자, 주문 상세, 내부 계정명
- 운영 DB 연결 문자열
- 외부 서비스 webhook URL

테스트 데이터라 하더라도 실제 값과 닮은 문자열은 마스킹하는 습관을 들이는 편이 좋습니다.
특히 custom reporter는 이벤트를 구조화해서 저장하므로, 검색과 복사가 쉬워집니다.
편리한 형식일수록 민감정보가 남지 않게 더 엄격하게 관리해야 합니다.

## custom reporter 적용 전 체크리스트

### H3. 유지보수 비용을 먼저 계산한다

custom reporter는 코드입니다.
한 번 만들면 Node.js 버전 변화, CI 환경 변화, 팀의 리포트 요구 변화에 맞춰 관리해야 합니다.
단순히 콘솔 모양이 마음에 들지 않는 정도라면 built-in reporter 조합이나 후처리 스크립트가 더 낫습니다.

적용 전에는 아래 질문을 확인하세요.

- built-in reporter와 destination 조합으로 해결할 수 없는가?
- reporter가 테스트 결과의 의미를 바꾸지 않는가?
- 실패해도 테스트 자체의 원인을 가리지 않는가?
- 민감정보 마스킹 기준이 있는가?
- 리포트 파일이 CI artifact로 보관되는가?
- Node.js 버전과 reporter 사용법이 문서화되어 있는가?

이 질문에 답하기 어렵다면 custom reporter보다 작은 리포트 후처리부터 시작하는 편이 낫습니다.
테스트 운영 도구는 복잡함을 줄여야지, 실패 원인을 하나 더 만들어서는 안 됩니다.

### H3. 팀 문서에는 실행 명령과 출력 위치를 같이 적는다

reporter 설정은 package script만 봐도 어느 정도 알 수 있지만, 운영 문서에는 더 구체적으로 남기는 것이 좋습니다.
특히 CI 실패를 처리하는 사람이 reporter 구현을 모를 수도 있습니다.

```md
## Test reports

- Console: `spec` reporter, GitHub Actions log
- Failure summary: `reports/failures.ndjson`
- Full machine-readable result: `reports/test-results.tap`
- Artifact name: `node-test-reports`
- Sensitive values: token/email/webhook URL must be masked before output
```

이 정도만 있어도 실패 리포트를 찾는 시간이 줄어듭니다.
새 팀원이 들어와도 어떤 파일을 봐야 하는지 바로 알 수 있습니다.

## 마무리

Node.js test runner custom reporter는 테스트 결과를 팀의 운영 방식에 맞게 전달하는 도구입니다.
처음부터 복잡한 reporter를 만들기보다 `--test-reporter`와 `--test-reporter-destination`으로 콘솔 출력과 파일 리포트를 분리하는 것부터 시작하세요.

그다음 built-in reporter로 부족한 지점이 명확해졌을 때 custom reporter를 작게 추가하면 됩니다.
좋은 reporter는 실패를 숨기지 않고, 필요한 정보를 빠르게 보여주며, 민감정보를 남기지 않습니다.
테스트 신뢰도는 assertion 코드뿐 아니라 실패를 읽고 복구하는 경험에서도 만들어집니다.

## 함께 읽기

- [Node.js test runner coverage 가이드: 내장 커버리지와 lcov를 CI에 연결하는 법](/development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html)
- [Node.js test runner shard 가이드: CI 테스트를 여러 작업으로 나누는 방법](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html)
- [Node.js test runner table-driven tests 가이드: 반복 케이스를 읽기 좋게 정리하는 법](/development/blog/seo/2026/07/12/nodejs-test-runner-table-driven-tests-guide.html)


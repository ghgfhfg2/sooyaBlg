---
layout: post
title: "Node.js test runner coverage 가이드: 내장 커버리지와 lcov를 CI에 연결하는 법"
date: 2026-07-01 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-coverage-lcov-ci-guide
permalink: /development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html
alternates:
  ko: /development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html
  x_default: /development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, coverage, lcov, ci, testing, javascript]
description: "Node.js test runner의 --experimental-test-coverage와 lcov reporter로 커버리지 리포트를 만들고 CI에 연결하는 방법을 정리합니다. include/exclude 기준, ignore 주석, 커버리지 운영 원칙을 예제로 설명합니다."
---

테스트가 통과한다는 사실만으로는 코드가 충분히 검증됐는지 알기 어렵습니다.
특히 리팩터링이 잦은 프로젝트에서는 어떤 파일과 분기가 테스트에 닿았는지 확인하는 기준이 필요합니다.
Node.js 내장 test runner는 별도 테스트 프레임워크 없이도 커버리지 통계를 수집하고, lcov 파일로 내보내는 기능을 제공합니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html)는 `--experimental-test-coverage` 플래그를 사용하면 테스트 완료 후 커버리지 통계를 보고한다고 설명합니다.
또한 `NODE_V8_COVERAGE`를 지정하면 V8 coverage 파일을 디렉터리에 남길 수 있고, `lcov` reporter를 사용하면 CI나 외부 품질 도구가 읽기 쉬운 `lcov.info` 파일을 만들 수 있습니다.
이 글에서는 내장 커버리지를 실무 CI에 붙일 때 필요한 실행 명령, 제외 기준, 운영 원칙을 정리합니다.

기본 테스트 실행법은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 참고하세요.
병렬 실행 기준은 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html)와 이어집니다.
테스트 범위를 좁히는 옵션은 [Node.js test runner skip todo only 가이드](/development/blog/seo/2026/07/01/nodejs-test-runner-skip-todo-only-focused-tests-guide.html)도 함께 보면 좋습니다.

## Node.js test runner coverage가 하는 일

### H3. --experimental-test-coverage로 통계를 수집한다

Node.js test runner에서 커버리지를 켜려면 `node --test`에 `--experimental-test-coverage`를 더합니다.
이 플래그는 테스트 실행 중 어떤 코드가 실행됐는지 수집하고, 테스트가 끝난 뒤 커버리지 요약을 출력합니다.
이름에 experimental이 붙어 있으므로 Node.js 버전별 출력 형태나 세부 동작은 공식 문서를 기준으로 확인하는 편이 좋습니다.

```bash
node --test --experimental-test-coverage
```

처음 적용할 때는 커버리지 숫자를 곧바로 품질 점수처럼 다루기보다, 테스트가 닿지 않는 핵심 파일을 찾는 탐색 도구로 사용하는 편이 안전합니다.
커버리지가 높아도 assertion이 약하면 결함을 놓칠 수 있고, 커버리지가 낮아도 위험도가 낮은 glue code일 수 있습니다.
숫자는 의사결정의 시작점이지 결론이 아닙니다.

### H3. lcov reporter는 CI 연동에 적합하다

터미널 요약만으로 충분한 로컬 실행과 달리 CI에서는 파일 형태의 리포트가 필요할 때가 많습니다.
Node.js test runner는 `lcov` reporter를 제공하며, `--test-reporter-destination`으로 출력 파일을 지정할 수 있습니다.
공식 문서 예시처럼 coverage 플래그와 lcov reporter를 함께 사용하면 `lcov.info`를 생성할 수 있습니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-reporter=lcov \
  --test-reporter-destination=coverage/lcov.info
```

`lcov` reporter는 커버리지 파일 생성에 집중합니다.
테스트 결과를 사람이 읽기 좋은 형태로도 남기고 싶다면 CI 로그용 reporter와 커버리지 reporter를 함께 구성하는 방식을 검토하세요.
프로젝트의 CI 도구가 여러 reporter를 어떻게 처리하는지 확인하고, 실패 시 원인을 바로 볼 수 있게 로그와 산출물을 분리하는 것이 좋습니다.

## include와 exclude 기준 잡기

### H3. 기본 제외 규칙을 이해한다

Node.js 커버리지 리포트는 기본적으로 Node.js core module과 `node_modules` 내부 파일을 포함하지 않습니다.
또한 매칭된 테스트 파일은 기본적으로 커버리지 리포트에서 제외됩니다.
대부분의 애플리케이션에서는 이 기본값이 자연스럽습니다.

```bash
node --test --experimental-test-coverage
```

처음부터 include/exclude를 많이 추가하면 리포트의 의미가 흐려질 수 있습니다.
먼저 기본값으로 실행해 보고, 실제 제품 코드가 빠졌는지 또는 생성 파일이 섞였는지 확인하세요.
그다음 프로젝트 구조에 맞춰 최소한의 규칙만 더하는 편이 유지보수에 좋습니다.

### H3. generated code와 entrypoint는 별도로 판단한다

커버리지에서 제외할 파일은 단순히 숫자를 올리기 위해 고르면 안 됩니다.
자동 생성 파일, framework bootstrap, 타입 정의를 위한 얇은 entrypoint처럼 테스트 가치가 낮은 코드는 제외 후보가 될 수 있습니다.
반대로 비즈니스 규칙, 장애 처리, 권한 검사처럼 위험도가 높은 코드는 커버리지 대상에 남겨야 합니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-exclude='dist/**' \
  --test-coverage-exclude='scripts/generated/**'
```

제외 규칙은 코드 리뷰에서 설명할 수 있어야 합니다.
팀원이 "왜 이 파일은 커버리지에서 빠졌나요?"라고 물었을 때, 생성물인지, 테스트 비용이 과한지, 다른 방식으로 검증되는지 답할 수 있어야 합니다.
설명할 수 없는 exclude는 품질 신호를 약하게 만듭니다.

## ignore 주석을 쓸 때의 원칙

### H3. 도달 불가능한 방어 코드는 짧게 제외한다

커버리지 도구는 실제로 실행된 줄을 기준으로 계산하기 때문에, 방어용 branch나 플랫폼별 branch가 숫자를 낮출 수 있습니다.
Node.js는 coverage를 끄고 켜는 주석과 다음 줄만 무시하는 주석을 지원합니다.
짧은 예외에는 범위가 작은 `ignore next`를 우선 고려하세요.

```js
export function parsePort(value) {
  const port = Number.parseInt(value, 10);

  /* node:coverage ignore next */
  if (!Number.isInteger(port)) {
    return 3000;
  }

  return port;
}
```

이런 주석은 테스트하기 어려운 코드를 숨기는 도구가 아닙니다.
정말로 테스트 가치가 낮거나 런타임 환경상 재현하기 어려운 branch에만 좁게 사용해야 합니다.
한 파일에 ignore가 많아진다면 코드 구조나 테스트 전략을 다시 봐야 합니다.

### H3. disable과 enable은 범위를 명확히 닫는다

여러 줄을 제외해야 한다면 `disable`과 `enable` 주석을 사용할 수 있습니다.
다만 범위를 닫지 않으면 의도보다 넓은 코드가 커버리지에서 사라질 수 있으므로, 아주 짧은 block에만 쓰는 편이 좋습니다.
리뷰에서 눈에 잘 띄도록 주석 주변에 이유를 남기는 것도 도움이 됩니다.

```js
export function createDebugReporter(enabled) {
  /* node:coverage disable */
  if (!enabled) {
    return {
      write() {}
    };
  }
  /* node:coverage enable */

  return {
    write(message) {
      process.stderr.write(`${message}\n`);
    }
  };
}
```

이 예제처럼 제외 범위가 명확하면 나중에 제거하기 쉽습니다.
긴 함수 전체를 제외하는 방식은 피하고, 정말 필요한 분기만 작게 감싸세요.
커버리지 예외도 제품 코드의 일부처럼 주기적으로 정리해야 합니다.

## CI에서 운영하는 방법

### H3. package script로 실행 명령을 고정한다

커버리지 명령은 사람이 매번 직접 입력하기보다 `package.json`에 고정하는 편이 좋습니다.
로컬용 테스트와 CI용 커버리지 명령을 분리하면 실행 목적이 명확해집니다.
CI 산출물 경로도 script에 포함하면 워크플로 파일이 단순해집니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:coverage": "node --test --experimental-test-coverage",
    "test:coverage:lcov": "node --test --experimental-test-coverage --test-reporter=lcov --test-reporter-destination=coverage/lcov.info"
  }
}
```

이렇게 두면 개발자는 빠른 확인에는 `npm test`를 사용하고, 병합 전 점검에는 `npm run test:coverage:lcov`를 사용할 수 있습니다.
CI에서는 커버리지 파일을 artifact로 저장하거나, 품질 도구에 업로드하는 단계를 뒤에 붙이면 됩니다.
중요한 것은 로컬과 CI가 같은 명령을 공유하도록 만드는 것입니다.

### H3. 커버리지 임계값은 점진적으로 올린다

커버리지 도입 첫날부터 높은 임계값을 걸면 팀이 숫자 맞추기에 몰릴 수 있습니다.
기존 프로젝트라면 현재 기준선을 먼저 측정하고, 핵심 모듈부터 조금씩 목표를 올리는 편이 현실적입니다.
신규 코드에는 더 엄격한 기준을 적용하되, 오래된 영역은 개선 계획과 함께 관리하세요.

```text
baseline: current project coverage
target: prevent new critical modules from dropping below agreed coverage
review: inspect uncovered branches in payment, auth, data mutation, retry logic
```

커버리지 운영의 목표는 테스트를 많이 쓰는 것이 아니라, 위험한 변경을 더 빨리 발견하는 것입니다.
따라서 전체 평균보다 중요한 파일의 미검증 branch를 보는 습관이 더 값집니다.
수치와 코드 리뷰를 함께 사용해야 커버리지가 실제 품질 신호가 됩니다.

## 적용 전 체크리스트

### H3. 커버리지를 품질 신호로 유지한다

Node.js 내장 coverage는 시작 비용이 낮고 CI에 붙이기 쉽습니다.
하지만 설정만 추가하고 리포트를 보지 않으면 숫자는 금방 배경 소음이 됩니다.
도입 전에는 리포트 생성 위치, 제외 규칙, ignore 주석 기준, 리뷰 루프를 함께 정하는 것이 좋습니다.

- `node --test --experimental-test-coverage`가 로컬에서 안정적으로 실행되는가?
- `coverage/lcov.info` 같은 산출물 경로가 CI artifact 규칙과 맞는가?
- `node_modules`, 생성 파일, 테스트 파일 제외 기준이 명확한가?
- `/* node:coverage ignore next */` 사용 이유를 코드 리뷰에서 설명할 수 있는가?
- 커버리지 숫자보다 핵심 branch 검증 여부를 함께 확인하는가?

커버리지는 테스트의 목적이 아니라 관찰 도구입니다.
좋은 테스트 이름, 명확한 assertion, 실패 원인을 좁히는 fixture 설계가 함께 있어야 리포트가 의미를 갖습니다.
Node.js test runner를 이미 사용하고 있다면, coverage는 별도 도구를 크게 늘리지 않고 테스트 운영 수준을 한 단계 올릴 수 있는 실용적인 다음 단계입니다.

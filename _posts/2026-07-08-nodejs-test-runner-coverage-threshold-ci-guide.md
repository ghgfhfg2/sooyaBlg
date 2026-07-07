---
layout: post
title: "Node.js test runner coverage threshold 가이드: CI에서 커버리지 기준을 실패로 만드는 법"
date: 2026-07-08 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-coverage-threshold-ci-guide
permalink: /development/blog/seo/2026/07/08/nodejs-test-runner-coverage-threshold-ci-guide.html
alternates:
  ko: /development/blog/seo/2026/07/08/nodejs-test-runner-coverage-threshold-ci-guide.html
  x_default: /development/blog/seo/2026/07/08/nodejs-test-runner-coverage-threshold-ci-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, coverage-threshold, coverage, ci, testing, javascript, quality-gate]
description: "Node.js test runner의 --test-coverage-lines, --test-coverage-branches, --test-coverage-functions 옵션으로 CI 커버리지 기준을 실패 조건으로 만드는 방법을 정리합니다. 기준선 설정, 점진적 상향, lcov 리포트와 함께 운영하는 패턴을 예제로 설명합니다."
---

커버리지 리포트를 만들었지만 아무도 보지 않는다면 품질 기준으로 작동하기 어렵습니다.
테스트 실행 뒤 숫자만 출력되고 CI는 계속 성공한다면, 커버리지는 참고 자료일 뿐 배포 판단 기준이 아닙니다.
Node.js 내장 test runner는 커버리지 수집에 더해 line, branch, function 커버리지 최소 기준을 CLI 옵션으로 지정할 수 있습니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html)는 `--experimental-test-coverage`로 커버리지를 수집하고, 커버리지 결과가 reporter의 `test:coverage` 이벤트로 전달된다고 설명합니다.
Node.js CLI는 `--test-coverage-lines`, `--test-coverage-branches`, `--test-coverage-functions` 옵션으로 최소 커버리지 기준을 지정할 수 있습니다.
기준을 넘지 못하면 테스트 실행 결과를 실패로 만들 수 있으므로, CI에서 "테스트는 통과했지만 커버리지가 기준 미달인 변경"을 막는 데 사용할 수 있습니다.

커버리지 리포트 생성과 lcov 저장은 [Node.js test runner coverage lcov 가이드](/development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html)에서 먼저 다뤘습니다.
CI 테스트 시간을 나누는 방법은 [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html)를 참고하세요.
실패한 테스트만 다시 확인하는 흐름은 [Node.js test runner rerun failures 가이드](/development/blog/seo/2026/07/03/nodejs-test-runner-rerun-failures-ci-guide.html)와 함께 설계하면 좋습니다.

## coverage threshold가 필요한 이유

### H3. 리포트를 CI 실패 조건으로 바꾼다

커버리지 리포트는 사람이 열어 봐야 의미가 생깁니다.
하지만 바쁜 리뷰 과정에서는 `lcov.info`나 터미널 요약을 매번 확인하지 못할 수 있습니다.
threshold를 걸면 커버리지 하락을 CI의 빨간 신호로 바꿀 수 있습니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-lines=80 \
  --test-coverage-branches=70 \
  --test-coverage-functions=80
```

이 명령은 테스트를 실행하고 커버리지 통계를 수집한 뒤, 전체 line, branch, function 비율이 기준에 못 미치면 실패로 처리합니다.
테스트 assertion은 모두 통과했더라도 커버리지 기준을 만족하지 못하면 병합 전에 확인할 수 있습니다.
품질 기준을 문서에만 두지 않고 실행 가능한 규칙으로 옮기는 방식입니다.

### H3. 새 코드의 무검증 증가를 막는다

프로젝트의 전체 커버리지를 하루 만에 크게 올리기는 어렵습니다.
그래도 threshold를 낮은 기준선부터 걸면 새 변경이 기존보다 더 나쁜 상태로 들어오는 것을 줄일 수 있습니다.
특히 리팩터링 중 테스트 파일을 실수로 제외하거나, 중요한 분기를 새로 추가하고 검증하지 않은 경우를 빨리 발견할 수 있습니다.

```text
current line coverage: 76%
temporary gate: 75%
next target: 78%
review focus: uncovered branch in payment, auth, retry, data mutation code
```

처음에는 현재 수치보다 아주 조금 낮거나 같은 기준으로 시작해도 충분합니다.
목표는 숫자 경쟁이 아니라 하락 방지입니다.
기준선이 안정되면 핵심 모듈 테스트를 보강하면서 조금씩 올리는 편이 팀에 덜 부담스럽습니다.

## 세 가지 threshold 옵션 이해하기

### H3. line threshold는 실행된 줄의 비율을 본다

`--test-coverage-lines`는 전체 줄 기준으로 실행된 비율을 검사합니다.
가장 이해하기 쉬운 지표라서 첫 도입 기준으로 적합합니다.
다만 줄이 실행됐다고 해서 그 줄의 결과가 충분히 검증됐다는 뜻은 아니므로 assertion 품질과 함께 봐야 합니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-lines=80
```

line 기준은 넓은 빈틈을 찾는 데 좋습니다.
예를 들어 새 service 파일이 테스트에서 전혀 import되지 않았다면 line coverage가 바로 떨어질 수 있습니다.
반대로 조건 분기 한쪽만 실행된 경우에는 line 기준만으로 위험을 충분히 드러내지 못할 수 있습니다.

### H3. branch threshold는 조건문 빈틈을 드러낸다

`--test-coverage-branches`는 if, switch, 조건 연산자처럼 분기된 경로가 얼마나 실행됐는지 봅니다.
실무에서는 line coverage보다 branch coverage가 더 중요한 신호가 될 때가 많습니다.
성공 경로만 테스트하고 실패 경로, 권한 없음, timeout, 재시도 종료 같은 분기를 놓치는 경우가 흔하기 때문입니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-lines=80 \
  --test-coverage-branches=70
```

branch 기준은 처음부터 너무 높게 잡으면 기존 코드에서 실패가 많이 날 수 있습니다.
대신 위험도가 높은 모듈의 미검증 분기를 먼저 줄이고, 전체 기준은 점진적으로 올리는 전략이 현실적입니다.
결제, 인증, 데이터 삭제, 장애 복구 코드는 전체 평균보다 더 엄격하게 리뷰하는 편이 좋습니다.

### H3. function threshold는 호출되지 않은 API를 찾는다

`--test-coverage-functions`는 함수 단위로 실행 여부를 봅니다.
파일은 import됐지만 특정 helper나 error handler가 전혀 호출되지 않는 경우를 찾는 데 유용합니다.
특히 utility 모듈이나 adapter 파일에서 "존재하지만 테스트가 닿지 않는 함수"를 발견하기 쉽습니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-functions=85
```

function 기준은 공개 API가 늘어나는 프로젝트에서 효과적입니다.
새 함수를 추가하고 테스트를 빼먹으면 바로 지표가 흔들릴 수 있습니다.
다만 작은 함수가 많은 코드베이스에서는 수치가 민감하게 움직일 수 있으므로, line과 branch 기준을 함께 보고 판단하는 것이 좋습니다.

## package script로 기준 고정하기

### H3. 로컬용과 CI용 명령을 분리한다

개발자가 매번 threshold 옵션을 기억하게 만들면 운영이 흔들립니다.
`package.json` script에 기준을 고정하면 로컬과 CI가 같은 명령을 공유할 수 있습니다.
빠른 로컬 테스트와 병합 전 품질 게이트를 분리하면 실행 목적도 명확해집니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:coverage": "node --test --experimental-test-coverage",
    "test:coverage:gate": "node --test --experimental-test-coverage --test-coverage-lines=80 --test-coverage-branches=70 --test-coverage-functions=80"
  }
}
```

개발 중에는 `npm test`로 빠르게 확인하고, PR 전에는 `npm run test:coverage:gate`를 실행할 수 있습니다.
CI에서는 gate script를 필수 job으로 두면 커버리지 기준 미달 변경이 자연스럽게 막힙니다.
명령이 한곳에 있으면 기준 변경도 리뷰하기 쉽습니다.

### H3. lcov 리포트와 gate를 함께 남긴다

threshold는 실패 여부를 알려 주지만, 어디가 비었는지 자세히 보려면 리포트가 필요합니다.
CI에서 기준 미달이 발생했을 때 `lcov.info`나 HTML 리포트를 artifact로 남기면 원인 파악이 빨라집니다.
Node.js test runner의 lcov reporter를 함께 쓰면 외부 커버리지 도구와 연결하기도 쉽습니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-lines=80 \
  --test-coverage-branches=70 \
  --test-coverage-functions=80 \
  --test-reporter=spec \
  --test-reporter=lcov \
  --test-reporter-destination=stdout \
  --test-reporter-destination=coverage/lcov.info
```

CI 로그에는 사람이 읽기 좋은 reporter를 남기고, coverage 디렉터리에는 lcov 파일을 저장하는 식으로 역할을 나눌 수 있습니다.
프로젝트의 Node.js 버전에 따라 reporter 다중 지정 방식과 출력 위치는 공식 문서와 실제 CI 로그로 확인하세요.
중요한 것은 실패 원인을 "기준 미달"에서 멈추지 않고 "어느 파일의 어떤 분기가 비었는지"까지 추적할 수 있게 만드는 것입니다.

## GitHub Actions에 적용하기

### H3. coverage gate job을 명시적으로 둔다

GitHub Actions에서는 테스트 job 안에 coverage gate를 넣거나, 별도 job으로 분리할 수 있습니다.
처음 도입할 때는 일반 테스트와 같은 job에 두는 편이 단순합니다.
테스트 시간이 길어지면 shard나 matrix 전략으로 분리한 뒤 리포트를 모으는 구조를 검토하면 됩니다.

```yaml
name: test

on:
  pull_request:
  push:
    branches: [main]

jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: npm
      - run: npm ci
      - run: npm run test:coverage:gate
```

이 구성은 `test:coverage:gate`가 실패하면 PR 상태를 실패로 만듭니다.
팀원이 어떤 기준으로 막혔는지 바로 볼 수 있도록 script 이름에 gate 의도를 드러내는 것이 좋습니다.
커버리지 기준은 숨은 정책이 아니라 합의된 병합 조건이어야 합니다.

### H3. artifact를 남겨 실패 분석 시간을 줄인다

threshold 실패가 났을 때 숫자만 보면 다음 행동이 애매합니다.
리포트를 artifact로 업로드하면 리뷰어가 실패한 job에서 바로 내려받아 확인할 수 있습니다.
외부 서비스에 업로드하지 않아도, CI 안에서 보관하는 것만으로 충분히 도움이 됩니다.

```yaml
      - run: npm run test:coverage:gate
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-report
          path: coverage/
```

`if: always()`를 두면 threshold 실패가 나도 coverage 산출물을 남길 수 있습니다.
다만 coverage 디렉터리에 민감한 로그나 환경 파일이 섞이지 않도록 출력 경로를 분리하세요.
테스트 리포트는 공유해도 되는 품질 산출물만 포함해야 합니다.

## 기준선을 정하는 실무 방법

### H3. 현재 수치를 먼저 측정한다

기준을 정하기 전에 현재 커버리지를 측정해야 합니다.
기존 프로젝트에서 갑자기 90% 기준을 걸면 대부분의 팀은 테스트 설계보다 숫자 맞추기에 집중하게 됩니다.
먼저 현재 수치를 기록하고, 실패하지 않는 최소 gate를 만든 뒤 개선 계획을 세우는 편이 낫습니다.

```bash
npm run test:coverage

# 예시 기록
# lines: 77.4%
# branches: 62.1%
# functions: 79.0%
```

이 수치가 나왔다면 첫 gate는 lines 77, branches 60, functions 78처럼 잡을 수 있습니다.
그다음 핵심 모듈의 테스트를 보강한 뒤 2주 또는 4주 단위로 기준을 올립니다.
커버리지 기준은 한번 정하고 끝나는 숫자가 아니라, 코드베이스와 함께 움직이는 운영 지표입니다.

### H3. 전체 평균보다 위험한 파일을 먼저 본다

전체 coverage가 높아도 중요한 파일이 비어 있으면 품질 위험은 큽니다.
반대로 전체 coverage가 낮아도 위험도가 낮은 entrypoint나 생성 코드가 대부분이라면 우선순위가 다를 수 있습니다.
threshold는 전체 하락을 막는 장치로 두고, 리뷰에서는 위험한 파일의 branch를 더 자세히 보세요.

```text
priority modules:
- auth/session validation
- payment state transition
- retry and timeout handling
- data migration and destructive update path
- webhook signature verification
```

이런 영역은 단순 line coverage보다 실패 경로와 예외 경로 테스트가 중요합니다.
특히 권한 검사와 상태 전이는 "성공한다"보다 "잘못된 요청을 거부한다"가 더 중요한 경우가 많습니다.
coverage threshold는 이런 리뷰 대화를 시작하게 만드는 신호로 써야 합니다.

## threshold를 도입할 때 주의할 점

### H3. 숫자를 올리기 위한 테스트를 경계한다

커버리지 기준이 생기면 숫자를 올리기 위한 얕은 테스트가 늘어날 수 있습니다.
함수를 호출만 하고 결과를 검증하지 않는 테스트는 coverage에는 도움이 되지만 결함 발견에는 약합니다.
따라서 threshold와 함께 assertion 품질, 테스트 이름, 실패 메시지도 리뷰해야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { calculateDiscount } from '../src/discount.js';

test('caps discount at configured maximum', () => {
  const discount = calculateDiscount({
    price: 10000,
    rate: 0.5,
    maxDiscount: 3000
  });

  assert.equal(discount, 3000);
});
```

좋은 테스트는 코드를 실행하는 데서 멈추지 않고, 비즈니스 규칙을 검증합니다.
coverage threshold는 이런 테스트가 빠지지 않게 돕는 보조 장치입니다.
숫자가 목표가 되면 중요한 assertion이 약해질 수 있으므로, 리뷰 기준을 함께 유지해야 합니다.

### H3. exclude와 ignore를 작은 범위로 제한한다

threshold를 통과하기 위해 exclude나 ignore 주석을 늘리면 기준의 신뢰도가 떨어집니다.
자동 생성 파일, 빌드 산출물, 환경별로 재현하기 어려운 방어 코드처럼 설명 가능한 경우에만 제외하세요.
제외 규칙은 코드 리뷰에서 이유를 남길 수 있어야 합니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-exclude='dist/**' \
  --test-coverage-exclude='scripts/generated/**' \
  --test-coverage-lines=80
```

exclude는 프로젝트 구조를 반영하는 설정이지, 테스트하기 귀찮은 파일을 숨기는 통로가 아닙니다.
ignore 주석도 한 줄 또는 짧은 block에만 쓰는 편이 좋습니다.
예외가 많아지면 기준을 낮추기보다 테스트 전략과 코드 구조를 먼저 점검해야 합니다.

## 적용 전 체크리스트

### H3. CI gate로 쓸 준비가 됐는지 확인한다

coverage threshold는 작은 옵션이지만 팀의 병합 흐름에 직접 영향을 줍니다.
그래서 도입 전에는 기준, 리포트, 예외 규칙, 실패 대응을 함께 정해야 합니다.
처음부터 완벽한 수치를 찾기보다, 하락을 막는 현실적인 기준으로 시작하는 것이 좋습니다.

- 현재 line, branch, function 커버리지를 측정했는가?
- `test:coverage:gate`처럼 CI에서 실행할 script가 고정됐는가?
- 기준 미달 시 확인할 lcov 또는 artifact 경로가 있는가?
- `--test-coverage-exclude` 규칙이 설명 가능한 파일에만 적용됐는가?
- 숫자보다 중요한 assertion과 위험 branch를 코드 리뷰에서 함께 보는가?

Node.js test runner의 coverage threshold는 커버리지 리포트를 행동 가능한 CI 기준으로 바꿔 줍니다.
처음에는 낮은 기준으로 하락을 막고, 핵심 모듈의 branch와 function 검증을 늘리면서 점진적으로 올리는 방식이 가장 안정적입니다.
테스트가 통과했는지와 함께 충분히 검증됐는지를 묻기 시작하면, 커버리지는 단순한 숫자가 아니라 배포 품질을 지키는 운영 신호가 됩니다.

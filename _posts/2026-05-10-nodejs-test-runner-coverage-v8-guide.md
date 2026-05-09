---
layout: post
title: "Node.js test runner coverage 가이드: 내장 커버리지로 테스트 누락 찾는 법"
date: 2026-05-10 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-coverage-v8-guide
permalink: /development/blog/seo/2026/05/10/nodejs-test-runner-coverage-v8-guide.html
alternates:
  ko: /development/blog/seo/2026/05/10/nodejs-test-runner-coverage-v8-guide.html
  x_default: /development/blog/seo/2026/05/10/nodejs-test-runner-coverage-v8-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, test-runner, coverage, testing, backend]
description: "Node.js 내장 test runner와 V8 커버리지로 테스트 누락을 찾는 방법을 정리했습니다. --experimental-test-coverage 실행법, 결과 해석, CI 기준, 실무 주의사항까지 예제로 설명합니다."
---

테스트를 추가했는데도 배포 후 같은 종류의 버그가 반복된다면, 문제는 테스트 개수보다 **어디가 검증되지 않았는지 보이지 않는 것**일 수 있습니다.
커버리지는 테스트 품질을 자동으로 보장하지는 않지만, 변경이 잦은 코드에서 사각지대를 찾는 데 매우 유용한 신호입니다.

Node.js 내장 `node:test` runner는 별도 테스트 프레임워크 없이 테스트를 실행할 수 있고, V8 기반 커버리지 출력도 함께 사용할 수 있습니다.
이 글에서는 `--experimental-test-coverage`로 테스트 커버리지를 확인하고, 실무에서 어떤 기준으로 해석하면 좋은지 정리합니다.
기본 테스트 작성법이 먼저 필요하다면 [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)을 함께 보면 좋습니다.

## Node.js test runner coverage가 필요한 이유

### H3. 테스트 실행 성공만으로는 누락 영역을 알기 어렵다

`node --test`가 초록색으로 끝났다는 사실은 현재 작성된 테스트가 통과했다는 뜻입니다.
하지만 중요한 분기, 예외 처리, 장애 복구 코드가 실제로 실행됐는지는 별도 확인이 필요합니다.

예를 들어 다음 코드는 정상 경로만 테스트하면 `throw` 분기가 계속 비어 있을 수 있습니다.

```js
export function parsePositiveInteger(value) {
  const number = Number(value);

  if (!Number.isInteger(number) || number <= 0) {
    throw new TypeError('value must be a positive integer');
  }

  return number;
}
```

이 함수의 정상 입력만 검증하면 테스트는 통과합니다.
하지만 잘못된 입력을 받았을 때 어떤 오류를 내는지, 호출자가 그 오류를 어떻게 처리해야 하는지는 검증되지 않습니다.
커버리지는 이런 누락을 빠르게 드러냅니다.

### H3. 커버리지는 목표가 아니라 탐색 도구다

커버리지 100%가 항상 좋은 설계를 의미하지는 않습니다.
의미 없는 getter, 단순 상수, 외부 시스템 연결 코드까지 숫자만 맞추려고 테스트하면 유지보수 비용이 커질 수 있습니다.

실무에서는 다음 질문에 답하는 도구로 쓰는 편이 좋습니다.

- 최근 수정한 코드가 테스트에서 실제로 실행됐는가?
- 장애가 나면 치명적인 분기와 예외 경로가 비어 있지는 않은가?
- 리팩터링 후 테스트가 검증하는 범위가 줄어들지 않았는가?
- CI에서 최소 기준을 두어 큰 누락을 조기에 막을 수 있는가?

즉 커버리지는 품질 점수판이 아니라, 테스트 리뷰를 시작하게 해 주는 지도에 가깝습니다.

## 기본 실행법: --experimental-test-coverage

### H3. node --test에 coverage 옵션을 붙인다

Node.js 내장 test runner를 쓰고 있다면 커버리지 실행은 간단합니다.

```bash
node --test --experimental-test-coverage
```

프로젝트에서 테스트 파일을 명시적으로 지정하고 싶다면 패턴을 함께 전달할 수 있습니다.

```bash
node --test --experimental-test-coverage "test/**/*.test.js"
```

`package.json`에는 다음처럼 스크립트를 분리해 두면 편합니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:coverage": "node --test --experimental-test-coverage"
  }
}
```

이렇게 두면 평소에는 빠른 테스트를 돌리고, PR 전이나 CI에서는 커버리지까지 확인하는 흐름을 만들 수 있습니다.

### H3. 실험적 옵션 여부를 문서에 남긴다

`--experimental-test-coverage`는 Node.js 버전에 따라 출력 형식이나 세부 동작이 달라질 수 있습니다.
따라서 팀 프로젝트라면 README나 개발 문서에 다음 내용을 함께 남기는 것이 안전합니다.

- 프로젝트가 기준으로 삼는 Node.js 버전
- 커버리지 실행 명령
- CI에서 실패로 볼 최소 기준
- 제외할 파일이나 디렉터리 기준

Node.js 버전 차이를 줄이는 방법은 [npm ci와 lockfile 가이드](/development/blog/seo/2026/03/06/npm-ci-lockfile-reproducible-build-guide.html)처럼 재현 가능한 설치와 런타임 고정을 함께 적용하는 것입니다.
테스트 도구는 코드만큼 실행 환경의 영향을 받기 때문입니다.

## 커버리지 결과 해석하기

### H3. line, branch, function을 구분해서 본다

커버리지 결과를 볼 때는 하나의 숫자만 보지 않는 편이 좋습니다.
주로 다음 항목을 나눠 봅니다.

- line coverage: 실행된 코드 라인의 비율
- branch coverage: `if`, `switch`, 삼항 연산자 같은 분기 실행 비율
- function coverage: 선언된 함수가 호출된 비율

라인 커버리지가 높아도 branch 커버리지가 낮으면 예외 경로나 실패 분기가 비어 있을 수 있습니다.
반대로 branch 커버리지를 올리려고 모든 방어 코드를 억지로 테스트하면 테스트가 구현 세부사항에 과하게 묶일 수 있습니다.

권장 순서는 다음과 같습니다.

1. 핵심 도메인 로직의 line/function 누락을 먼저 본다.
2. 장애 대응과 입력 검증 코드의 branch 누락을 본다.
3. 단순 glue code나 외부 어댑터는 숫자보다 통합 테스트 필요성을 판단한다.

### H3. 낮은 커버리지는 원인별로 다르게 처리한다

커버리지가 낮은 파일을 발견했을 때 무조건 테스트만 추가하면 금방 피곤해집니다.
먼저 낮은 이유를 분류해야 합니다.

| 원인 | 대응 |
| --- | --- |
| 핵심 로직인데 테스트가 없음 | 단위 테스트를 우선 추가 |
| 예외 분기만 비어 있음 | 실패 입력, timeout, 재시도 실패 케이스 추가 |
| 외부 API 호출 코드 | 인터페이스를 분리하고 mock 또는 contract 테스트 고려 |
| 초기화 스크립트 | 스모크 테스트나 별도 제외 기준 검토 |
| 사용되지 않는 죽은 코드 | 제거하거나 deprecated 표시 |

예외 처리와 오류 래핑이 복잡한 코드라면 [Node.js Error cause 가이드: 래핑된 오류 디버깅 방법](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)처럼 실패 원인을 보존하는 테스트도 함께 작성하는 것이 좋습니다.

## 예제로 보는 커버리지 개선 흐름

### H3. 정상 경로만 있는 테스트부터 시작한다

먼저 함수와 테스트를 준비합니다.

```js
// src/discount.js
export function calculateDiscountPrice(price, rate) {
  if (price < 0) {
    throw new RangeError('price must be greater than or equal to 0');
  }

  if (rate < 0 || rate > 1) {
    throw new RangeError('rate must be between 0 and 1');
  }

  return Math.round(price * (1 - rate));
}
```

```js
// test/discount.test.js
import test from 'node:test';
import assert from 'node:assert/strict';
import { calculateDiscountPrice } from '../src/discount.js';

test('calculates discount price', () => {
  assert.equal(calculateDiscountPrice(10000, 0.2), 8000);
});
```

이 테스트는 정상 경로를 검증하지만, `price`와 `rate` 검증 분기는 실행하지 않습니다.
커버리지 결과에서 branch가 낮게 나오면 테스트 누락을 의심할 수 있습니다.

### H3. 실패 경로를 명시적으로 추가한다

입력 검증은 서비스 안정성에 직접 영향을 주므로 실패 경로 테스트를 추가하는 편이 좋습니다.

```js
test('rejects negative price', () => {
  assert.throws(
    () => calculateDiscountPrice(-1, 0.2),
    /price must be greater than or equal to 0/
  );
});

test('rejects invalid rate', () => {
  assert.throws(
    () => calculateDiscountPrice(10000, 1.5),
    /rate must be between 0 and 1/
  );
});
```

이제 정상 경로와 주요 실패 경로가 모두 테스트에서 실행됩니다.
중요한 점은 숫자를 올리려고 테스트를 추가한 것이 아니라, 실제로 API 계약에 해당하는 동작을 검증했다는 점입니다.

## CI에서 커버리지 기준 적용하기

### H3. 처음부터 높은 기준을 걸지 않는다

기존 프로젝트에 커버리지를 처음 도입한다면 90% 같은 높은 기준부터 걸지 않는 것이 좋습니다.
오래된 코드가 많은 저장소에서는 테스트 작성보다 기준 우회와 예외 목록 관리에 에너지를 쓰게 될 수 있습니다.

현실적인 시작점은 다음과 같습니다.

- 현재 전체 커버리지를 한 번 측정한다.
- 핵심 패키지나 신규 코드에만 기준을 둔다.
- 전체 기준은 현재 수치보다 조금 낮게 시작한다.
- PR마다 커버리지가 크게 하락하지 않도록 감시한다.

특히 배포 안정성이 중요한 서비스라면 커버리지 숫자와 함께 [readiness/liveness probe 가이드](/development/blog/seo/2026/03/10/readiness-liveness-probe-deploy-reliability-guide.html)처럼 런타임 헬스체크도 같이 설계해야 합니다.
테스트는 배포 전 방어선이고, 헬스체크는 배포 후 방어선입니다.

### H3. npm script로 CI 명령을 고정한다

CI 워크플로에는 긴 명령을 직접 쓰기보다 `package.json` 스크립트를 호출하는 편이 유지보수에 좋습니다.

```yaml
name: test

on:
  pull_request:
  push:
    branches: [main]

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
      - run: npm run test:coverage
```

로컬과 CI가 같은 스크립트를 실행하면 실패 재현이 쉬워집니다.
테스트가 느려진다면 전체 커버리지는 main 브랜치에서, PR은 변경 범위 중심으로 나누는 전략도 고려할 수 있습니다.

## 커버리지 도입 시 주의할 점

### H3. 외부 시스템 호출은 직접 때리지 않는다

커버리지를 올리겠다고 실제 결제, 실제 메일 발송, 실제 운영 API를 테스트에서 호출하면 위험합니다.
테스트는 재현 가능해야 하며, 비용과 부작용이 없어야 합니다.

외부 시스템을 다루는 코드는 다음 방식으로 분리하는 것이 안전합니다.

- 순수 로직은 단위 테스트로 검증한다.
- 외부 클라이언트는 인터페이스 뒤에 둔다.
- 네트워크 응답은 fixture나 mock으로 고정한다.
- 실제 연동은 제한된 staging 환경에서 별도 실행한다.

시간 의존 코드라면 [Node.js test runner mock timers 가이드](/development/blog/seo/2026/04/29/nodejs-test-runner-mock-timers-guide.html)를 활용해 타이머와 날짜를 제어할 수 있습니다.
테스트가 현재 시간에 흔들리지 않아야 커버리지 결과도 신뢰할 수 있습니다.

### H3. 숫자보다 변경 위험을 먼저 본다

커버리지 보고서를 볼 때 가장 중요한 파일은 커버리지가 가장 낮은 파일이 아닐 수 있습니다.
실제로는 최근 자주 변경되고, 장애가 나면 영향이 크고, 테스트가 부족한 파일이 먼저입니다.

우선순위는 다음 기준으로 정합니다.

1. 결제, 인증, 권한, 데이터 삭제처럼 영향이 큰 코드
2. 장애 대응, 재시도, timeout, fallback처럼 평소 실행이 적은 코드
3. 최근 리팩터링했거나 자주 수정되는 코드
4. 여러 서비스가 공유하는 공통 유틸리티

이 순서로 보면 커버리지 개선이 단순 숫자 맞추기가 아니라 실제 안정성 개선으로 이어집니다.

## 실무 체크리스트

### H3. 커버리지 실행 전 확인할 것

- `node --test`만 실행했을 때 모든 테스트가 먼저 통과하는가?
- 프로젝트의 Node.js 버전이 로컬과 CI에서 일치하는가?
- 외부 네트워크, 실제 데이터 삭제, 실제 발송 같은 부작용이 제거됐는가?
- 테스트 fixture에 개인정보나 토큰이 들어 있지 않은가?
- 커버리지 기준을 README나 개발 문서에 남겼는가?

### H3. 커버리지 결과 리뷰 질문

- 새로 추가한 코드가 보고서에 포함됐는가?
- 중요한 실패 분기가 비어 있지 않은가?
- 테스트가 구현 세부사항이 아니라 공개 동작을 검증하는가?
- 숫자를 올리기 위해 의미 없는 테스트를 추가하지 않았는가?
- 제외한 파일은 합리적인 이유가 있는가?

## FAQ

### H3. Node.js test runner coverage만으로 충분한가요?

작은 라이브러리나 백엔드 유틸리티는 충분할 수 있습니다.
하지만 브라우저 UI, 복잡한 mocking, HTML 리포트, 변경 라인 기준 커버리지 같은 기능이 필요하다면 c8, Vitest, Jest 같은 도구를 함께 검토할 수 있습니다.
핵심은 도구보다 팀이 꾸준히 실행할 수 있는 흐름입니다.

### H3. 커버리지 몇 퍼센트를 목표로 해야 하나요?

정답은 없습니다.
신규 핵심 로직은 높은 기준을 둘 수 있지만, 오래된 레거시 프로젝트는 현재 수치를 기준으로 점진 개선하는 편이 안전합니다.
처음에는 전체 숫자보다 중요 파일의 누락 분기와 PR별 하락 방지를 우선하세요.

### H3. 커버리지에서 제외해도 되는 파일이 있나요?

빌드 산출물, 타입 선언, 단순 설정 파일, 진입점 스크립트처럼 테스트 가치가 낮은 파일은 제외할 수 있습니다.
다만 제외 기준을 문서화해야 합니다.
무심코 제외 목록이 늘어나면 커버리지의 신뢰도가 빠르게 떨어집니다.

## 마무리

Node.js 내장 test runner와 V8 커버리지를 함께 쓰면 별도 도구를 크게 늘리지 않고도 테스트 사각지대를 확인할 수 있습니다.
`node --test --experimental-test-coverage`로 현재 상태를 측정하고, 숫자를 목표로 삼기보다 중요한 분기와 위험한 변경을 먼저 살펴보세요.

좋은 커버리지 운영은 “몇 퍼센트인가”보다 “중요한 코드가 실제로 검증되고 있는가”에 가깝습니다.
그 관점만 유지하면 커버리지는 테스트를 귀찮게 만드는 지표가 아니라, 배포 전 놓친 부분을 알려주는 실용적인 안전망이 됩니다.

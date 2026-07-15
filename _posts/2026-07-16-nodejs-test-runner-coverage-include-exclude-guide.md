---
layout: post
title: "Node.js test runner coverage include/exclude 가이드: CI 커버리지 범위를 정확하게 정하는 법"
date: 2026-07-16 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-coverage-include-exclude-guide
permalink: /development/blog/seo/2026/07/16/nodejs-test-runner-coverage-include-exclude-guide.html
alternates:
  ko: /development/blog/seo/2026/07/16/nodejs-test-runner-coverage-include-exclude-guide.html
  x_default: /development/blog/seo/2026/07/16/nodejs-test-runner-coverage-include-exclude-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, coverage, include, exclude, ci, testing, javascript]
description: "Node.js test runner의 --test-coverage-include와 --test-coverage-exclude 옵션으로 커버리지 측정 범위를 정하는 방법을 정리합니다. 소스 코드 포함, 생성 파일 제외, CI threshold와 lcov 리포트 연결 기준을 예제로 설명합니다."
---

커버리지 숫자는 테스트 품질을 이해하는 보조 지표입니다.
하지만 측정 범위가 흐리면 숫자가 쉽게 왜곡됩니다.
테스트 파일, 빌드 산출물, generated code, 스크립트성 파일까지 함께 들어가면 실제 제품 코드가 얼마나 검증되는지 보기 어렵습니다.

Node.js 내장 test runner는 커버리지 측정 범위를 조정할 수 있도록 `--test-coverage-include`와 `--test-coverage-exclude` 옵션을 제공합니다.
이 옵션을 잘 쓰면 "무엇을 테스트 책임 범위로 볼 것인가"를 CI 명령 안에 명시할 수 있습니다.
[Node.js test runner coverage threshold 가이드](/development/blog/seo/2026/07/08/nodejs-test-runner-coverage-threshold-ci-guide.html), [Node.js test runner coverage lcov 가이드](/development/blog/seo/2026/07/01/nodejs-test-runner-coverage-lcov-ci-guide.html), [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html)와 함께 보면 커버리지 운영 흐름을 더 안정적으로 잡을 수 있습니다.

## coverage include/exclude가 필요한 이유

### H3. 커버리지 숫자는 측정 대상에 따라 달라진다

같은 테스트를 실행해도 어떤 파일을 커버리지 대상에 포함하는지에 따라 결과가 달라집니다.
예를 들어 `src/**`만 측정하면 제품 코드 중심의 숫자가 나오지만, `test/**`나 `dist/**`까지 함께 들어가면 의미가 흐려집니다.

```bash
node --test --experimental-test-coverage
```

기본 실행은 빠르게 시작하기 좋습니다.
하지만 프로젝트가 커질수록 커버리지 대상 파일을 명확히 제한해야 합니다.
그렇지 않으면 다음과 같은 문제가 생깁니다.

- 테스트 helper가 커버리지 대상에 들어가 숫자를 올린다.
- 빌드 산출물이 원본 코드와 중복 집계된다.
- generated code 때문에 threshold가 불필요하게 흔들린다.
- CLI 스크립트와 서버 코드가 같은 기준으로 평가된다.

커버리지는 "테스트를 많이 썼는가"보다 "중요한 실행 경로가 검증되는가"를 봐야 합니다.
그래서 include/exclude 규칙은 단순한 옵션이 아니라 테스트 책임 범위를 정의하는 문서에 가깝습니다.

### H3. CI에서 같은 기준을 반복 실행한다

로컬에서 한 번 확인하는 커버리지보다 CI에서 매번 같은 기준으로 측정하는 커버리지가 더 중요합니다.
팀원이 각자 다른 명령으로 커버리지를 보면 숫자를 비교하기 어렵습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:coverage": "node --test --experimental-test-coverage --test-coverage-include='src/**/*.js' --test-coverage-exclude='src/**/*.generated.js'"
  }
}
```

이렇게 script에 범위를 고정하면 리뷰와 CI가 같은 기준을 공유합니다.
나중에 threshold를 추가하더라도 "어떤 파일을 기준으로 80%인가"가 명확합니다.

## include 규칙 설계하기

### H3. 제품 코드의 진입점을 먼저 포함한다

`--test-coverage-include`는 커버리지 대상에 넣을 파일 패턴을 지정할 때 사용합니다.
가장 흔한 시작점은 애플리케이션 소스 디렉터리입니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-include='src/**/*.js'
```

이 명령은 `src` 아래 JavaScript 파일을 커버리지 측정 대상으로 삼습니다.
테스트 파일이나 빌드 결과물이 별도 디렉터리에 있다면 제품 코드만 보는 기준을 만들기 쉽습니다.

TypeScript를 빌드해서 실행하는 프로젝트라면 실제 Node.js가 실행하는 파일 위치를 기준으로 생각해야 합니다.
테스트가 `dist`를 실행한다면 `dist/**/*.js`를 측정할 수도 있고, source map과 별도 도구를 조합해 원본 기준 리포트를 만들 수도 있습니다.
중요한 것은 팀이 보는 커버리지 리포트가 어떤 파일 세계를 기준으로 하는지 명확히 쓰는 것입니다.

### H3. 패키지별 책임 범위를 나눈다

모노레포나 여러 패키지를 가진 프로젝트에서는 전체 저장소를 한 번에 측정하는 것보다 패키지별로 범위를 나누는 편이 읽기 쉽습니다.

```json
{
  "scripts": {
    "test:coverage:api": "node --test packages/api/test/**/*.test.js --experimental-test-coverage --test-coverage-include='packages/api/src/**/*.js'",
    "test:coverage:webhook": "node --test packages/webhook/test/**/*.test.js --experimental-test-coverage --test-coverage-include='packages/webhook/src/**/*.js'"
  }
}
```

이 방식은 실패 원인을 좁히기 좋습니다.
API 패키지의 threshold가 깨졌는지, webhook 패키지의 테스트가 부족한지 CI 로그에서 바로 보입니다.

다만 공통 패키지가 여러 곳에서 함께 쓰인다면 중복 측정이 생길 수 있습니다.
공통 코드까지 한 번에 보는 별도 job을 두거나, 패키지별 리포트와 전체 리포트를 구분해 운영하는 편이 안전합니다.

## exclude 규칙 설계하기

### H3. 생성 파일과 빌드 산출물을 제외한다

`--test-coverage-exclude`는 커버리지 대상에서 뺄 파일 패턴을 지정합니다.
가장 먼저 제외할 후보는 generated code와 build output입니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-include='src/**/*.js' \
  --test-coverage-exclude='src/**/*.generated.js' \
  --test-coverage-exclude='dist/**'
```

생성 파일은 사람이 직접 테스트를 설계하기 어렵습니다.
API schema에서 생성된 client, protobuf 출력물, ORM generated file 같은 코드는 원본 정의나 통합 테스트로 검증하는 편이 자연스럽습니다.
이런 파일을 커버리지 threshold에 그대로 넣으면 팀이 통제하기 어려운 숫자 때문에 CI가 흔들릴 수 있습니다.

빌드 산출물도 주의해야 합니다.
원본 `src`와 결과물 `dist`가 함께 들어가면 같은 코드가 다른 파일로 중복 집계될 수 있습니다.
측정 기준을 원본으로 둘지 산출물로 둘지 하나를 선택하세요.

### H3. 제외 이유를 코드 리뷰에서 설명 가능해야 한다

exclude 규칙은 커버리지를 낮추는 파일을 숨기는 도구가 아닙니다.
특정 파일을 제외했다면 리뷰어가 이유를 이해할 수 있어야 합니다.

좋은 제외 후보는 다음과 같습니다.

- 빌드 산출물: `dist/**`, `coverage/**`
- 자동 생성 파일: `**/*.generated.js`, `**/__generated__/**`
- 타입 선언이나 schema 출력물
- 외부 도구가 관리하는 migration snapshot
- 테스트 fixtures처럼 실행 대상이 아닌 데이터 파일

반대로 핵심 비즈니스 로직, 인증/권한 처리, 결제/정산 경계, 데이터 삭제 로직을 exclude로 빼는 것은 위험합니다.
이런 파일의 커버리지가 낮다면 제외하기보다 테스트를 작게 추가하거나 설계를 나누는 쪽이 낫습니다.

## threshold와 함께 쓰는 흐름

### H3. 범위를 고정한 뒤 threshold를 건다

커버리지 threshold를 먼저 걸고 include/exclude를 나중에 정하면 숫자가 계속 흔들립니다.
순서는 반대가 좋습니다.
먼저 측정 범위를 합의하고, 그다음 line, branch, function 기준을 정합니다.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-include='src/**/*.js' \
  --test-coverage-exclude='src/**/*.generated.js' \
  --test-coverage-lines=80 \
  --test-coverage-branches=70 \
  --test-coverage-functions=80
```

이 명령은 제품 코드 범위를 먼저 고정하고, 그 범위 안에서 threshold를 적용합니다.
커버리지 기준이 깨졌을 때도 "범위가 잘못됐나"와 "테스트가 부족한가"를 분리해서 볼 수 있습니다.

초기 도입이라면 threshold를 현재 수치보다 아주 조금 낮게 시작하는 편이 좋습니다.
그 다음 새 코드에서 기준을 올리고, 오래된 영역은 별도 개선 작업으로 천천히 끌어올리면 CI가 실질적인 품질 장치로 작동합니다.

### H3. lcov 리포트와 콘솔 출력을 함께 남긴다

사람이 CI 로그에서 빠르게 보는 값과 도구가 업로드하는 리포트는 목적이 다릅니다.
콘솔에는 요약을 남기고, 파일에는 lcov를 저장하면 리뷰와 분석을 함께 챙길 수 있습니다.

```bash
mkdir -p coverage

node --test \
  --experimental-test-coverage \
  --test-coverage-include='src/**/*.js' \
  --test-coverage-exclude='src/**/*.generated.js' \
  --test-reporter=spec \
  --test-reporter-destination=stdout \
  --test-reporter=lcov \
  --test-reporter-destination=coverage/lcov.info
```

이때도 include/exclude 규칙은 reporter와 별개입니다.
어떤 형식으로 출력하든 커버리지 대상 범위가 먼저 정해져야 합니다.
CI artifact나 외부 커버리지 서비스에 올리는 파일도 같은 기준에서 나온 결과인지 확인하세요.

## 실무에서 자주 생기는 실수

### H3. 테스트 파일을 커버리지 대상에 넣는다

테스트 파일 자체가 커버리지 대상에 들어가면 숫자가 좋아 보일 수 있습니다.
테스트는 당연히 실행되므로 많은 줄이 covered로 표시됩니다.
하지만 이 숫자는 제품 코드의 검증 정도를 보여 주지 않습니다.

```bash
# 피하는 편이 좋은 예
node --test \
  --experimental-test-coverage \
  --test-coverage-include='**/*.js'
```

작은 프로젝트에서는 편해 보이지만, 시간이 지나면 리포트가 섞입니다.
가능하면 제품 코드 디렉터리를 명시하고 테스트 디렉터리는 제외하세요.

```bash
node --test \
  --experimental-test-coverage \
  --test-coverage-include='src/**/*.js' \
  --test-coverage-exclude='test/**'
```

테스트 helper가 `src` 안에 섞여 있다면 디렉터리 구조를 정리하는 것도 좋은 신호입니다.
커버리지 규칙이 복잡해지는 이유가 실제 코드 구조의 애매함일 수 있기 때문입니다.

### H3. exclude를 임시 회피책으로 남겨 둔다

CI가 깨졌을 때 가장 쉬운 해결은 문제 파일을 exclude에 추가하는 것입니다.
하지만 이 방식이 반복되면 threshold는 의미를 잃습니다.

예외적으로 제외해야 한다면 제거 조건을 함께 남기는 편이 좋습니다.

```json
{
  "scripts": {
    "test:coverage": "node --test --experimental-test-coverage --test-coverage-include='src/**/*.js' --test-coverage-exclude='src/legacy/**'"
  }
}
```

위처럼 `legacy/**`를 제외한다면 issue나 TODO에 다음 정보를 남기세요.

- 왜 지금 테스트하기 어려운가?
- 어떤 리팩터링 이후 다시 포함할 것인가?
- 제외된 영역을 다른 테스트나 모니터링으로 보완하고 있는가?

exclude는 영구 면제가 아니라 임시 경계여야 합니다.
핵심 영역일수록 제외 목록에 오래 두지 않는 것이 좋습니다.

## 보안과 로그 위생

### H3. 커버리지 리포트에 민감 경로를 남기지 않는다

커버리지 리포트는 파일 경로와 소스 라인 정보를 포함할 수 있습니다.
대부분의 경우 문제없지만, 내부 사용자명, 임시 디렉터리, 비공개 프로젝트 경로가 외부 서비스로 전송되는 환경이라면 확인이 필요합니다.

CI artifact를 외부에 공개하거나 PR comment로 요약을 남긴다면 다음을 점검하세요.

- 절대 경로 대신 저장소 기준 상대 경로가 보이는가?
- 리포트에 환경 변수 값이나 테스트 fixture의 민감 데이터가 포함되지 않는가?
- 외부 커버리지 서비스 접근 권한이 저장소 공개 범위와 맞는가?

커버리지 include/exclude 규칙 자체에는 토큰이나 비밀 값을 넣지 마세요.
경로 패턴은 저장소 구조만 표현해야 합니다.

### H3. generated fixture에 실제 개인정보를 넣지 않는다

커버리지에서 제외된 파일이라고 해서 안전 점검에서 제외되는 것은 아닙니다.
`__fixtures__`, `__generated__`, `snapshots` 같은 디렉터리에 실제 사용자 이메일, 토큰, 실서비스 payload가 들어가면 리포트와 무관하게 저장소 리스크가 됩니다.

테스트 데이터는 가짜 값으로 만들고, 실제 로그를 복사했다면 익명화한 뒤 커밋해야 합니다.
커버리지 설정은 테스트 품질 기준이고, 민감정보 관리는 별도의 기본 안전 기준입니다.

## 적용 체크리스트

### H3. coverage 범위를 정할 때 확인할 것

`--test-coverage-include`와 `--test-coverage-exclude`를 적용하기 전에는 다음 항목을 확인하세요.

- 제품 코드 디렉터리가 include에 명확히 들어갔는가?
- 테스트 파일, build output, generated code가 불필요하게 섞이지 않는가?
- exclude 규칙마다 설명 가능한 이유가 있는가?
- threshold는 include/exclude 범위를 고정한 뒤 설정했는가?
- lcov, CI artifact, 외부 커버리지 서비스가 같은 기준의 결과를 쓰는가?
- 커버리지 리포트나 fixture에 민감정보가 남지 않는가?

커버리지 설정은 한 번 만들고 끝나는 파일이 아닙니다.
프로젝트 구조가 바뀌면 include/exclude 규칙도 같이 점검해야 합니다.

## 마무리

Node.js test runner의 `--test-coverage-include`와 `--test-coverage-exclude`는 커버리지 숫자를 더 믿을 수 있게 만드는 기본 옵션입니다.
제품 코드 중심으로 include를 잡고, generated code와 build output처럼 사람이 직접 테스트 책임을 지기 어려운 파일만 신중하게 exclude하면 CI 결과가 훨씬 선명해집니다.

좋은 커버리지 운영은 높은 숫자보다 설명 가능한 숫자에서 시작합니다.
측정 범위를 먼저 문서화하고, 그 위에 threshold와 lcov 리포트를 얹으면 테스트 품질을 꾸준히 개선할 수 있습니다.

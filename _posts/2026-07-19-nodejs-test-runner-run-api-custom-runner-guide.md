---
layout: post
title: "Node.js test runner run() API 가이드: 자체 테스트 실행기와 CI 래퍼 만들기"
date: 2026-07-19 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-run-api-custom-runner-guide
permalink: /development/blog/seo/2026/07/19/nodejs-test-runner-run-api-custom-runner-guide.html
alternates:
  ko: /development/blog/seo/2026/07/19/nodejs-test-runner-run-api-custom-runner-guide.html
  x_default: /development/blog/seo/2026/07/19/nodejs-test-runner-run-api-custom-runner-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, run-api, custom-runner, ci, testing, javascript, reporter]
description: "Node.js test runner의 run() API로 자체 테스트 실행기와 CI 래퍼를 만드는 방법을 정리합니다. files, globPatterns, setup, testNamePatterns, isolation 옵션과 로그 위생 기준을 예제로 설명합니다."
---

Node.js 테스트를 항상 `node --test` 명령으로만 실행할 필요는 없습니다.
대부분의 프로젝트에서는 CLI만으로 충분하지만, 테스트 파일 목록을 코드로 고르거나, CI 환경에 맞춰 reporter를 연결하거나, 실패 이벤트를 별도 시스템으로 보내야 할 때는 명령줄 옵션만으로 표현하기 불편해집니다.
이때 `node:test` 모듈의 `run()` API를 쓰면 테스트 실행 자체를 JavaScript 코드 안에서 조립할 수 있습니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html) 기준으로 `run()`은 테스트 파일, glob 패턴, 격리 방식, 이름 필터, 태그 필터, reporter 연결을 코드에서 다룰 수 있게 해 줍니다.
이 글에서는 `run()` API로 자체 테스트 실행기와 CI 래퍼를 만드는 방법, `node --test`와 역할을 나누는 기준, 민감정보가 섞이지 않는 로그 설계까지 정리합니다.
[Node.js test runner custom reporter 가이드](/development/blog/seo/2026/07/12/nodejs-test-runner-custom-reporter-ci-report-guide.html), [Node.js test runner reporters 가이드](/development/blog/seo/2026/07/18/nodejs-test-runner-reporters-spec-tap-dot-junit-guide.html), [Node.js test runner workerId와 attempt 가이드](/development/blog/seo/2026/07/18/nodejs-test-runner-workerid-attempt-flaky-ci-guide.html)와 함께 보면 테스트 실행과 리포팅 흐름을 더 안정적으로 설계할 수 있습니다.

## run() API가 필요한 상황

### H3. CLI 옵션이 길어지는 순간을 줄인다

처음에는 아래처럼 명령 하나로 충분합니다.

```bash
node --test
```

하지만 CI가 커지면 옵션이 길어집니다.
테스트 파일 패턴, 이름 필터, 태그 필터, 커버리지, reporter destination, 격리 방식, 재시도 정책이 한 줄에 섞입니다.
이 명령이 package script, CI yaml, 로컬 shell alias에 흩어지면 어떤 환경에서 어떤 테스트가 실행되는지 파악하기 어려워집니다.

`run()` API는 이런 실행 정책을 코드로 모으는 선택지입니다.
테스트 자체를 바꾸는 기능이 아니라, 테스트를 어떻게 찾고 실행하고 관측할지를 프로그래밍 방식으로 관리합니다.
특히 아래 상황에서 유용합니다.

- 변경된 패키지의 테스트만 실행하고 싶다.
- PR 라벨이나 환경 변수에 따라 테스트 범위를 다르게 잡고 싶다.
- 테스트 이벤트를 읽어 실패 요약을 직접 만들고 싶다.
- reporter를 여러 destination으로 나누고 싶다.
- 격리 방식과 실행 대상 파일을 명확한 코드 리뷰 대상으로 만들고 싶다.

반대로 단순한 프로젝트라면 `node --test`를 유지하는 편이 좋습니다.
실행기가 필요해서 쓰는 것이지, 모든 테스트 명령을 감싸기 위해 쓰는 기능은 아닙니다.

### H3. 테스트 정책을 실행 코드로 문서화한다

CI 스크립트는 빠르게 길어지고, 조건문이 많아지면 리뷰하기 어렵습니다.
예를 들어 `main` 브랜치에서는 전체 테스트를 돌리고, PR에서는 변경된 workspace만 돌리고, 야간 작업에서는 느린 통합 테스트까지 포함한다고 해 봅시다.
이 정책을 shell script로만 유지하면 테스트 파일 목록과 리포트 처리 로직이 문자열 조합에 의존하기 쉽습니다.

`run()`으로 만든 실행기는 이 정책을 JavaScript 객체와 함수로 표현합니다.
파일 선택, 필터, 이벤트 구독을 분리할 수 있고, 실행기 자체도 테스트할 수 있습니다.

## 가장 작은 자체 테스트 실행기

### H3. run()이 반환하는 TestsStream을 처리한다

`run()`은 테스트 실행 이벤트를 읽을 수 있는 stream을 반환합니다.
가장 단순한 실행기는 대상 파일을 넘기고, stream이 끝날 때까지 기다린 뒤 실패 여부에 따라 종료 코드를 정하는 방식입니다.

```js
// scripts/run-tests.mjs
import { run } from 'node:test';

const stream = run({
  files: [
    'test/unit/order-summary.test.mjs',
    'test/unit/price-format.test.mjs'
  ]
});

let failed = false;

for await (const event of stream) {
  if (event.type === 'test:fail') {
    failed = true;
  }
}

process.exitCode = failed ? 1 : 0;
```

이 예시는 reporter를 직접 만들지는 않습니다.
핵심은 테스트 실행 결과를 이벤트로 관찰할 수 있다는 점입니다.
CI에서는 이 이벤트를 바탕으로 실패 수, 실패 테스트 이름, duration, diagnostic 메시지를 별도 요약으로 만들 수 있습니다.

다만 처음부터 모든 이벤트를 직접 렌더링하려고 하면 부담이 큽니다.
실무에서는 내장 reporter를 함께 쓰고, 추가로 필요한 요약만 stream에서 뽑는 방식이 더 유지보수하기 쉽습니다.

### H3. files와 globPatterns는 함께 쓰지 않는다

`run()`에는 직접 파일 목록을 넘기는 `files`와 패턴으로 파일을 찾는 `globPatterns`가 있습니다.
두 옵션은 목적이 다릅니다.

```js
import { run } from 'node:test';

run({
  globPatterns: ['test/**/*.test.mjs']
});
```

`files`는 변경 파일 분석, workspace 매핑, 특정 실패 재현처럼 이미 파일 목록을 알고 있을 때 적합합니다.
`globPatterns`는 CLI의 기본 탐색에 가까운 방식으로, 테스트 파일 구조가 일관된 저장소에서 편합니다.

둘을 동시에 섞으면 실행 대상이 어디서 왔는지 흐려집니다.
CI 래퍼에서는 보통 다음처럼 기준을 나눕니다.

- PR 변경 기반 실행: `files`
- 전체 회귀 실행: `globPatterns`
- 로컬 기본 실행: `node --test`
- 디버깅 재현: 명시적인 `files`

실행 대상이 비어 있을 때도 명확히 처리해야 합니다.
변경된 테스트가 없으면 성공으로 끝낼지, 전체 smoke test를 돌릴지, CI 정책에 맞춰 코드로 고정해 두는 편이 좋습니다.

## CI 래퍼로 확장하기

### H3. 환경 변수는 테스트 선택에만 쓰고 로그에는 줄인다

CI 래퍼는 환경 변수를 자주 읽습니다.
브랜치명, PR 번호, 변경 범위, 테스트 모드 같은 값은 실행 정책에 필요합니다.
하지만 환경 변수를 그대로 diagnostic 로그에 찍으면 토큰, 내부 URL, 사용자명, 저장소 경로가 함께 노출될 수 있습니다.

아래처럼 입력 값은 실행 조건에만 쓰고, 로그에는 정규화된 모드만 남기는 편이 안전합니다.

```js
// scripts/ci-test-runner.mjs
import { run } from 'node:test';

const mode = process.env.CI_TEST_MODE === 'changed' ? 'changed' : 'full';

const options = mode === 'changed'
  ? { files: ['test/unit/order-summary.test.mjs'] }
  : { globPatterns: ['test/**/*.test.mjs'] };

const stream = run({
  ...options,
  isolation: 'process',
  setup(testStream) {
    testStream.on('test:diagnostic', (event) => {
      console.log(JSON.stringify({
        type: 'diagnostic',
        mode,
        message: String(event.data?.message ?? '').slice(0, 300)
      }));
    });
  }
});

let failures = 0;

for await (const event of stream) {
  if (event.type === 'test:fail') failures += 1;
}

console.log(JSON.stringify({ type: 'summary', mode, failures }));
process.exitCode = failures > 0 ? 1 : 0;
```

여기서 `mode`는 안전한 값으로 제한했습니다.
원본 브랜치명이나 PR 제목을 그대로 출력하지 않아도, 실패 분석에 필요한 실행 맥락은 충분히 남길 수 있습니다.

### H3. setup에서 이벤트 리스너를 먼저 붙인다

`run()` 옵션의 `setup` 함수는 테스트가 실행되기 전에 `TestsStream`에 리스너를 붙일 때 사용할 수 있습니다.
실패 이벤트, 진단 이벤트, coverage 이벤트를 별도 로그로 보내야 한다면 이 위치가 적합합니다.

```js
import { run } from 'node:test';

const stream = run({
  globPatterns: ['test/**/*.test.mjs'],
  setup(testStream) {
    testStream.on('test:fail', (event) => {
      const name = event.data?.name ?? '<unknown>';
      const file = event.data?.file ?? '<unknown>';

      console.error(JSON.stringify({
        type: 'test-fail',
        name,
        file: file.replace(process.cwd(), '<repo>')
      }));
    });
  }
});

for await (const _event of stream) {
  // Stream을 소비해야 실행 완료를 기다릴 수 있습니다.
}
```

절대 경로는 `<repo>`처럼 치환했습니다.
CI 로그는 외부 알림, 이슈 댓글, artifact로 복사될 수 있으므로 로컬 사용자명, 임시 디렉터리, 내부 빌드 경로를 그대로 남기지 않는 것이 좋습니다.

## 필터 옵션을 코드로 관리하기

### H3. testNamePatterns로 부분 실행을 안전하게 만든다

`testNamePatterns`는 실행할 테스트 이름을 정규식으로 제한합니다.
CLI의 `--test-name-pattern`과 같은 역할을 코드에서 다룰 수 있습니다.
예를 들어 특정 패키지의 smoke 테스트만 실행하는 래퍼는 아래처럼 만들 수 있습니다.

```js
import { run } from 'node:test';

const stream = run({
  globPatterns: ['packages/**/test/**/*.test.mjs'],
  testNamePatterns: [/smoke/i]
});

let failed = false;

for await (const event of stream) {
  if (event.type === 'test:fail') failed = true;
}

process.exitCode = failed ? 1 : 0;
```

정규식은 너무 넓게 잡지 않는 편이 좋습니다.
`/api/`처럼 흔한 단어는 예상보다 많은 테스트를 포함할 수 있습니다.
팀에서 부분 실행을 자주 쓴다면 테스트 이름에 `[smoke]`, `[integration]`, `[billing]`처럼 검색 가능한 접두어를 일관되게 붙이는 방식이 더 안정적입니다.

### H3. testTagFilters는 의도 중심 필터에 어울린다

Node.js test runner의 태그 필터를 쓰는 프로젝트라면 `run()`에서도 `testTagFilters`를 사용할 수 있습니다.
이름 패턴이 사람이 읽는 문장에 의존한다면, 태그는 실행 의도를 더 분명히 나타냅니다.

```js
import { run } from 'node:test';

const stream = run({
  globPatterns: ['test/**/*.test.mjs'],
  testTagFilters: ['critical']
});

for await (const _event of stream) {}
```

중요한 점은 태그가 테스트 품질의 우회로가 되지 않게 하는 것입니다.
`critical`만 자주 돌리고 나머지를 장기간 방치하면 전체 회귀 테스트의 의미가 약해집니다.
PR에서는 빠른 태그 필터를 쓰더라도, main 브랜치나 야간 작업에서는 전체 테스트를 돌리는 보완 루프가 필요합니다.

## 격리 방식과 child process 옵션

### H3. 기본은 process isolation을 유지한다

`run()`의 `isolation` 옵션은 테스트 파일을 어떻게 실행할지 정합니다.
기본값인 process 격리는 테스트 파일마다 별도 child process를 사용합니다.
전역 상태, module cache, process 환경 변경이 다른 파일로 번지는 것을 줄여 주므로 CI에서는 이 기본값을 유지하는 편이 안전합니다.

```js
import { run } from 'node:test';

run({
  globPatterns: ['test/**/*.test.mjs'],
  isolation: 'process'
});
```

`isolation: 'none'`은 모든 테스트 파일을 현재 프로세스에서 실행합니다.
시작 비용을 줄이고 디버깅을 쉽게 만들 수 있지만, 파일 간 전역 상태 공유가 생길 수 있습니다.
성능 때문에 격리를 끄기 전에 테스트가 전역 상태를 건드리지 않는지, mock restore가 빠지지 않았는지, 환경 변수를 되돌리는지 먼저 확인해야 합니다.

### H3. execArgv는 필요한 플래그만 넘긴다

프로세스 격리를 사용할 때는 child process에 넘길 Node.js 플래그를 `execArgv`로 지정할 수 있습니다.
예를 들어 TypeScript 로더, warning 정책, inspector 옵션을 테스트 실행에 맞게 넣을 수 있습니다.

```js
import { run } from 'node:test';

const stream = run({
  globPatterns: ['test/**/*.test.mjs'],
  isolation: 'process',
  execArgv: ['--no-warnings']
});

for await (const _event of stream) {}
```

여기에 너무 많은 플래그를 넣으면 로컬 실행과 CI 실행이 달라집니다.
테스트 성공을 위해 필요한 최소 플래그만 남기고, 제품 런타임과 무관한 디버깅 플래그는 별도 npm script로 분리하는 편이 좋습니다.

## 실패 요약 만들기

### H3. 실패 이벤트에서 필요한 필드만 모은다

자체 실행기를 만드는 가장 큰 이유 중 하나는 CI에서 읽기 좋은 실패 요약을 남기기 위해서입니다.
테스트 로그 전체를 뒤지는 대신, 실패한 테스트 이름과 파일, 짧은 메시지만 모아 마지막에 출력할 수 있습니다.

```js
import { run } from 'node:test';

const failures = [];

const stream = run({
  globPatterns: ['test/**/*.test.mjs']
});

for await (const event of stream) {
  if (event.type !== 'test:fail') continue;

  failures.push({
    name: event.data?.name ?? '<unknown>',
    file: String(event.data?.file ?? '<unknown>').replace(process.cwd(), '<repo>'),
    message: String(event.data?.details?.error?.message ?? '').slice(0, 200)
  });
}

if (failures.length > 0) {
  console.error(JSON.stringify({
    type: 'test-summary',
    failures
  }, null, 2));
}

process.exitCode = failures.length > 0 ? 1 : 0;
```

메시지는 길이를 제한했습니다.
에러 메시지에는 요청 body, fixture payload, 내부 URL이 섞일 수 있습니다.
CI 요약은 문제를 찾기 위한 색인 역할만 하게 하고, 자세한 내용은 제한된 artifact에서 확인하도록 나누는 편이 안전합니다.

### H3. flaky test 추적과 연결한다

`run()` 래퍼는 `workerId`, `attempt`, reporter 이벤트와 함께 쓰면 flaky test 추적에 도움이 됩니다.
실패가 몇 번째 시도에서 발생했는지, 어떤 worker에서 재현됐는지, 어떤 태그 필터로 실행됐는지를 구조화해서 남길 수 있기 때문입니다.

다만 실행기가 테스트 결과를 "좋게 보이게" 만들어서는 안 됩니다.
재시도에서 통과했다고 실패를 숨기거나, 특정 태그 테스트만 통과하면 전체 CI를 성공으로 처리하는 식의 정책은 장기적으로 품질 신뢰도를 떨어뜨립니다.
실행기는 관측과 선택을 돕는 도구이고, 실패 판정 기준은 명확해야 합니다.

## 운영 체크리스트

### H3. 자체 실행기에 넣을 기준

`run()` API를 도입한다면 실행기 자체도 운영 대상입니다.
다음 기준을 먼저 정해 두면 CI가 커져도 흔들림이 줄어듭니다.

- 기본 실행은 전체 테스트인지, 변경 기반 테스트인지 정한다.
- `files`와 `globPatterns` 중 하나만 쓰는 규칙을 둔다.
- `isolation: 'process'`를 기본값으로 두고 예외를 문서화한다.
- 환경 변수 원문을 로그에 남기지 않는다.
- 절대 경로, 토큰, 사용자 식별자는 요약에서 제거한다.
- 실패가 있으면 반드시 non-zero exit code를 반환한다.
- 부분 실행은 전체 회귀 실행으로 보완한다.

이 기준이 있으면 자체 실행기가 "편한 우회 스크립트"가 아니라 팀의 테스트 정책을 담는 작은 도구가 됩니다.

### H3. CLI와 run()을 함께 남긴다

자체 실행기를 만들더라도 `node --test` 진입점을 완전히 없애지 않는 편이 좋습니다.
로컬에서 빠르게 재현하거나, Node.js 문서의 예제를 그대로 확인하거나, 실행기 자체가 문제인지 테스트 코드가 문제인지 구분할 때 CLI가 기준점이 됩니다.

추천하는 구조는 다음과 같습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "node scripts/ci-test-runner.mjs",
    "test:smoke": "node scripts/ci-test-runner.mjs --smoke"
  }
}
```

`test`는 단순하고 예측 가능하게 유지하고, CI 정책은 `test:ci`에 모읍니다.
이렇게 나누면 새 팀원이 들어와도 기본 실행 경로를 쉽게 이해할 수 있고, CI 특화 로직은 한 파일에서 리뷰할 수 있습니다.

## 마무리

`run()` API는 Node.js test runner를 코드에서 제어하고 싶을 때 쓰는 실무적인 확장 지점입니다.
테스트 파일 선택, 이름과 태그 필터, stream 이벤트, setup 리스너, process isolation을 한곳에서 조합할 수 있어 CI 래퍼나 자체 테스트 실행기를 만들 때 유용합니다.

핵심은 단순합니다.
일반적인 로컬 실행은 `node --test`로 남기고, CI에서 필요한 선택 실행과 실패 요약만 `run()`으로 감싸세요.
그리고 실행 로그에는 실패 분석에 필요한 최소 메타데이터만 남겨 민감정보와 내부 경로가 흘러나가지 않게 관리하는 것이 좋습니다.

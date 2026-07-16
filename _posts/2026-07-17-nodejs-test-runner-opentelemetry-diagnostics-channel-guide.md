---
layout: post
title: "Node.js test runner OpenTelemetry 가이드: 테스트 실행 흐름을 추적으로 관찰하는 법"
date: 2026-07-17 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-opentelemetry-diagnostics-channel-guide
permalink: /development/blog/seo/2026/07/17/nodejs-test-runner-opentelemetry-diagnostics-channel-guide.html
alternates:
  ko: /development/blog/seo/2026/07/17/nodejs-test-runner-opentelemetry-diagnostics-channel-guide.html
  x_default: /development/blog/seo/2026/07/17/nodejs-test-runner-opentelemetry-diagnostics-channel-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, opentelemetry, diagnostics-channel, observability, testing, javascript, ci]
description: "Node.js test runner의 diagnostics_channel 기반 테스트 계측으로 OpenTelemetry 추적 흐름을 붙이는 방법을 정리합니다. node.test tracing channel, AsyncLocalStorage context, CI 테스트 관측성 기준을 예제로 설명합니다."
---

테스트가 실패했을 때 필요한 정보는 단순한 pass/fail 결과만이 아닙니다.
어떤 테스트가 오래 걸렸는지, 어떤 suite에서 에러가 집중되는지, CI 환경에서 특정 테스트만 느려지는지까지 함께 봐야 원인을 빠르게 좁힐 수 있습니다.
테스트 수가 많아질수록 로그를 눈으로 훑는 방식만으로는 한계가 생깁니다.

Node.js test runner는 `diagnostics_channel`을 통해 테스트 실행 이벤트를 관측 도구와 연결할 수 있는 계측 지점을 제공합니다.
공식 문서 기준으로 test runner는 `node.test` tracing channel에 테스트 시작, 종료, 에러 이벤트를 게시합니다.
이 글에서는 `diagnostics_channel`과 `AsyncLocalStorage`를 이용해 테스트 실행 흐름을 추적하는 방법, OpenTelemetry로 확장할 때의 설계 기준, CI에서 과도한 로그를 피하는 운영 방식을 정리합니다.
[Node.js test runner custom reporter 가이드](/development/blog/seo/2026/07/12/nodejs-test-runner-custom-reporter-ci-report-guide.html), [Node.js test runner coverage include/exclude 가이드](/development/blog/seo/2026/07/16/nodejs-test-runner-coverage-include-exclude-guide.html), [Node.js AsyncLocalStorage request context 가이드](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)와 함께 보면 테스트 관측성 흐름을 더 단단하게 만들 수 있습니다.

## test runner 계측이 필요한 이유

### H3. 테스트 로그만으로는 흐름을 보기 어렵다

CI에서 테스트가 실패하면 보통 가장 먼저 보는 것은 reporter 출력입니다.
어떤 테스트가 실패했는지, assertion message가 무엇인지 확인하는 데는 충분합니다.
하지만 다음 질문에는 일반 로그만으로 답하기 어렵습니다.

- 특정 suite가 매번 느린가?
- 실패한 테스트 직전에 어떤 테스트가 실행됐는가?
- timeout이 발생한 테스트의 실제 실행 시간은 어느 정도인가?
- CI worker별로 테스트 시간이 다르게 분포하는가?
- 테스트 실패와 외부 mock server 로그를 같은 trace로 묶을 수 있는가?

이런 정보는 테스트 결과 리포트와 관측성 데이터를 연결할 때 더 잘 보입니다.
`diagnostics_channel`은 테스트 코드 자체를 크게 바꾸지 않고 실행 이벤트를 구독할 수 있는 진입점을 제공합니다.

### H3. reporter와 tracing은 역할이 다르다

custom reporter는 사람이나 CI가 읽을 결과물을 만드는 데 적합합니다.
예를 들어 TAP, spec, lcov, junit 같은 출력은 실패 원인을 리뷰하고 기록하는 데 좋습니다.

반면 tracing은 실행 흐름을 시간축으로 보는 데 강합니다.
테스트 시작과 종료, 에러를 span처럼 다루면 느린 테스트와 실패 위치를 관측 시스템에서 검색할 수 있습니다.

두 방식은 경쟁 관계가 아닙니다.
CI에는 읽기 쉬운 reporter를 남기고, 장기 분석이나 flaky test 추적에는 tracing 이벤트를 붙이는 식으로 나눌 수 있습니다.

## diagnostics_channel로 test runner 이벤트 구독하기

### H3. node.test tracing channel을 구독한다

Node.js test runner는 `diagnostics_channel`의 tracing channel을 통해 테스트 실행 이벤트를 노출합니다.
핵심 채널은 `node.test`입니다.
이 채널에서 시작, 종료, 에러 이벤트를 받아 테스트 이름과 파일, 중첩 수준 같은 정보를 사용할 수 있습니다.

```js
// test/instrumentation.mjs
import dc from 'node:diagnostics_channel';

const testChannel = dc.tracingChannel('node.test');

testChannel.start.subscribe((event) => {
  console.log('[test:start]', {
    name: event.name,
    file: event.file,
    nesting: event.nesting,
    type: event.type
  });
});

testChannel.end.subscribe((event) => {
  console.log('[test:end]', {
    name: event.name,
    file: event.file,
    nesting: event.nesting,
    type: event.type
  });
});

testChannel.error.subscribe((event) => {
  console.error('[test:error]', {
    name: event.name,
    file: event.file,
    message: event.error?.message
  });
});
```

테스트 실행 전에 이 파일을 preload하면 이벤트를 받을 수 있습니다.

```bash
node --import ./test/instrumentation.mjs --test
```

처음에는 콘솔 출력으로 이벤트 형태를 확인하고, 실제 운영에서는 OpenTelemetry span 생성이나 JSON 로그 전송으로 바꾸는 편이 좋습니다.
CI 로그에 모든 이벤트를 그대로 남기면 오히려 실패 원인을 찾기 어려워질 수 있습니다.

### H3. 이벤트 데이터는 최소 필드부터 사용한다

테스트 이벤트에서 가장 먼저 쓸 만한 필드는 다음입니다.

- `name`: 테스트 또는 suite 이름
- `file`: 테스트 파일 경로
- `nesting`: suite와 subtest의 중첩 깊이
- `type`: `test` 또는 `suite`
- `error`: 에러 이벤트에 포함되는 Error 객체

이 필드만으로도 느린 테스트 목록, 파일별 실패 빈도, suite별 실행 시간을 만들 수 있습니다.
처음부터 모든 assertion 로그나 fixture 데이터를 붙이려고 하면 민감정보 노출 위험과 저장 비용이 커집니다.

테스트 관측성은 제품 로그보다 더 쉽게 방심할 수 있습니다.
fixture 안에 이메일, 토큰 형태의 문자열, 실제 사용자와 비슷한 데이터가 들어갈 수 있기 때문입니다.
추적 데이터에 payload를 넣기보다 테스트 이름, 파일, duration, 실패 코드처럼 구조화된 최소 정보만 남기는 편이 안전합니다.

## AsyncLocalStorage로 테스트 컨텍스트 전달하기

### H3. bindStore로 테스트별 컨텍스트를 만든다

`AsyncLocalStorage`를 함께 쓰면 현재 실행 중인 테스트 이름이나 시작 시간을 비동기 작업 안에서도 읽을 수 있습니다.
Node.js 공식 문서의 예시처럼 tracing channel의 `start.bindStore()`를 사용하면 테스트 실행 흐름에 store를 연결할 수 있습니다.

```js
// test/instrumentation.mjs
import dc from 'node:diagnostics_channel';
import { AsyncLocalStorage } from 'node:async_hooks';

export const testStorage = new AsyncLocalStorage();

const testChannel = dc.tracingChannel('node.test');

testChannel.start.bindStore(testStorage, (event) => ({
  testName: event.name,
  file: event.file,
  startedAt: performance.now()
}));

testChannel.end.subscribe((event) => {
  const store = testStorage.getStore();
  if (!store) return;

  const durationMs = Math.round(performance.now() - store.startedAt);

  console.log(JSON.stringify({
    event: 'test.end',
    name: event.name,
    file: event.file,
    durationMs
  }));
});
```

이 방식의 장점은 테스트 본문을 크게 바꾸지 않아도 컨텍스트가 이어진다는 점입니다.
HTTP mock client, DB helper, fixture loader 같은 내부 유틸에서 `testStorage.getStore()`를 읽으면 어느 테스트가 해당 작업을 만들었는지 남길 수 있습니다.

```js
// test/helpers/log-test-context.mjs
import { testStorage } from '../instrumentation.mjs';

export function logFixtureAccess(resource) {
  const store = testStorage.getStore();

  console.log(JSON.stringify({
    event: 'fixture.access',
    testName: store?.testName,
    file: store?.file,
    resource
  }));
}
```

단, 이 helper를 제품 코드에 섞지 않는 것이 중요합니다.
테스트 계측은 테스트 실행 환경의 관심사로 두고, 애플리케이션 런타임의 로깅 컨텍스트와는 경계를 나누세요.

### H3. subtest와 병렬 실행을 고려한다

test runner에서는 subtest와 병렬 실행을 함께 사용할 수 있습니다.
이때 컨텍스트가 섞이면 관측 데이터가 오히려 혼란스러워집니다.

좋은 기준은 다음입니다.

- top-level test와 subtest 이름을 모두 구분해서 기록한다.
- `file`과 `nesting`을 함께 남겨 같은 이름의 테스트를 구분한다.
- duration은 start/end 이벤트 기준으로 계산한다.
- 공유 전역 변수에 현재 테스트 이름을 저장하지 않는다.
- 병렬 테스트에서는 `AsyncLocalStorage` 같은 컨텍스트 전파 도구를 사용한다.

병렬 테스트 자체의 격리 기준은 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/07/13/nodejs-test-runner-concurrency-parallel-test-guide.html)를 함께 참고하면 좋습니다.
관측성도 결국 테스트 격리 위에 올라가기 때문에, 테스트가 전역 상태를 공유하면 tracing 데이터도 흔들립니다.

## OpenTelemetry로 확장하는 설계 기준

### H3. 테스트 하나를 span 하나로 본다

OpenTelemetry를 붙인다면 가장 단순한 모델은 테스트 하나를 span 하나로 보는 것입니다.
suite는 부모 span, 개별 test는 자식 span으로 구성할 수 있습니다.
처음부터 복잡한 계층을 만들기보다 파일, 테스트 이름, 상태, duration을 attribute로 남기는 방식이 다루기 쉽습니다.

```js
// pseudo-code: 실제 tracer 초기화 코드는 프로젝트 설정에 맞춘다.
testChannel.start.subscribe((event) => {
  const span = tracer.startSpan(event.name, {
    attributes: {
      'test.file': event.file ?? 'unknown',
      'test.nesting': event.nesting,
      'test.type': event.type
    }
  });

  spanMap.set(`${event.file}:${event.name}:${event.nesting}`, span);
});
```

실제 구현에서는 span을 찾기 위한 key 설계를 더 신중히 해야 합니다.
같은 파일 안에서 같은 이름의 테스트가 반복 생성될 수 있기 때문입니다.
가능하면 이벤트의 위치 정보, nesting, 실행 순서 카운터를 함께 써서 충돌을 줄입니다.

### H3. 실패 정보는 요약해서 기록한다

에러 이벤트가 오면 span status를 error로 바꾸고, 에러 메시지와 코드 정도만 남기는 편이 좋습니다.
stack trace 전체를 저장할지 여부는 팀의 보안 기준과 저장 비용을 보고 결정해야 합니다.

```js
testChannel.error.subscribe((event) => {
  const error = event.error;

  console.log(JSON.stringify({
    event: 'test.error',
    name: event.name,
    file: event.file,
    errorName: error?.name,
    errorCode: error?.code,
    errorMessage: error?.message
  }));
});
```

특히 assertion message에 실제 응답 본문이나 fixture 값이 그대로 들어가는 프로젝트라면 더 조심해야 합니다.
민감정보가 섞일 가능성이 있는 값은 trace attribute에 넣지 말고, 필요한 경우 마스킹된 요약만 남깁니다.

## CI에서 운영할 때의 체크리스트

### H3. 관측성은 기본 테스트 결과를 대체하지 않는다

테스트 tracing을 붙여도 CI의 기본 pass/fail 결과와 reporter 출력은 유지해야 합니다.
개발자가 PR에서 가장 먼저 보는 정보는 여전히 실패한 테스트 이름과 assertion message입니다.
OpenTelemetry는 그 다음 단계의 분석 도구로 보는 편이 좋습니다.

권장 흐름은 다음과 같습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "node --import ./test/instrumentation.mjs --test --test-reporter=spec",
    "test:coverage": "node --import ./test/instrumentation.mjs --test --experimental-test-coverage --test-coverage-include='src/**/*.js'"
  }
}
```

로컬 기본 명령은 가볍게 유지하고, CI나 진단용 명령에서만 계측을 켜면 개발 피드백 속도를 지킬 수 있습니다.
문제가 반복되는 기간에는 tracing을 켜고, 안정화된 뒤에는 sampling이나 조건부 활성화로 비용을 줄일 수 있습니다.

### H3. 민감정보와 비용을 함께 점검한다

테스트 관측성 데이터를 외부 APM이나 로그 저장소로 보낸다면 발행 전후로 다음 항목을 확인하세요.

- 테스트 이름에 실제 사용자 정보가 들어가지 않는가?
- assertion message에 토큰, 세션, 개인식별정보가 포함되지 않는가?
- fixture payload 전체를 attribute로 저장하지 않는가?
- CI job마다 너무 많은 span을 만들지 않는가?
- 실패한 테스트만 상세 기록하는 옵션이 필요한가?
- 저장 기간과 접근 권한이 팀 정책에 맞는가?

관측성은 문제 해결 속도를 높이기 위한 도구입니다.
테스트 실행마다 과도한 데이터를 남기면 비용이 늘고, 민감정보 관리 부담도 커집니다.
처음에는 duration, status, file, test name 같은 낮은 위험의 구조화 데이터부터 시작하는 편이 좋습니다.

## 마무리

Node.js test runner의 `diagnostics_channel` 계측은 테스트를 단순한 콘솔 출력이 아니라 관찰 가능한 실행 흐름으로 바꿔 줍니다.
`node.test` tracing channel에서 시작, 종료, 에러 이벤트를 구독하고, `AsyncLocalStorage`로 테스트별 컨텍스트를 전파하면 느린 테스트와 flaky test를 더 체계적으로 추적할 수 있습니다.

실무에서는 reporter, coverage, tracing의 역할을 분리하는 것이 핵심입니다.
reporter는 사람이 읽는 결과, coverage는 검증 범위, tracing은 시간축 분석을 담당하게 두면 CI 품질 신호가 더 선명해집니다.

## FAQ

### H3. 모든 프로젝트에 테스트 tracing이 필요한가요?

아닙니다.
테스트 수가 적고 CI 시간이 짧다면 기본 reporter만으로 충분합니다.
테스트가 많아졌거나, CI에서만 느려지는 테스트와 flaky test를 추적해야 할 때 도입하는 편이 좋습니다.

### H3. OpenTelemetry 없이 diagnostics_channel만 써도 되나요?

가능합니다.
처음에는 JSON 로그로 start, end, error 이벤트를 남기는 것만으로도 충분한 정보를 얻을 수 있습니다.
나중에 장기 추세 분석이 필요해지면 OpenTelemetry span이나 APM 연동으로 확장하면 됩니다.

### H3. 테스트 이름을 trace attribute로 남겨도 안전한가요?

대체로 안전하지만 팀의 테스트 이름 규칙에 달려 있습니다.
실제 이메일, 토큰, 고객사명 같은 값이 테스트 이름에 들어가지 않도록 하고, 동적 데이터는 마스킹하거나 케이스 ID로 대체하는 편이 좋습니다.

## 참고 자료

- [Node.js 공식 문서: Test runner](https://nodejs.org/api/test.html)
- [Node.js 공식 문서: Diagnostics Channel](https://nodejs.org/api/diagnostics_channel.html)

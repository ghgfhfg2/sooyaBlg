---
layout: post
title: "Node.js test runner context.log 가이드: CI 로그를 구조화해 실패 원인을 빠르게 찾는 법"
date: 2026-08-30 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-context-log-structured-ci-diagnostics-guide
permalink: /development/blog/seo/2026/08/30/nodejs-test-runner-context-log-structured-ci-diagnostics-guide.html
alternates:
  ko: /development/blog/seo/2026/08/30/nodejs-test-runner-context-log-structured-ci-diagnostics-guide.html
  x_default: /development/blog/seo/2026/08/30/nodejs-test-runner-context-log-structured-ci-diagnostics-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, context-log, diagnostics, ci, testing, reporter, observability]
description: "Node.js test runner의 context.log()로 테스트별 구조화 로그를 남기고 CI 실패 원인을 빠르게 추적하는 방법을 정리합니다. t.diagnostic(), console.log(), custom reporter와의 차이, 민감정보 마스킹 기준까지 실무 예제로 설명합니다."
---

테스트가 CI에서만 실패할 때 가장 답답한 부분은 실패 지점보다 **실패 직전의 상태**입니다.
로컬에서는 재현되지 않고, CI 로그에는 assertion diff만 남아 있으며, 필요한 값은 이미 사라져 있는 경우가 많습니다.
그래서 급하게 `console.log()`를 넣고 다시 푸시하지만, 병렬 테스트와 파일 격리가 섞이면 로그가 어느 테스트에서 나온 것인지 따라가기 어렵습니다.

Node.js 내장 테스트 러너를 쓰고 있다면 이 문제를 줄이기 위해 **`context.log()`**를 검토할 수 있습니다.
공식 Node.js 문서 기준으로 `context.log(message[, data])`는 테스트 컨텍스트에서 로그 메시지와 선택적인 구조화 데이터를 남기는 API이며, 테스트 러너 이벤트 스트림에서는 `test:stdout`이나 `test:diagnostic`과 별도로 `test:log` 이벤트로 다룰 수 있습니다.
문서에는 이 API가 Node.js 26.6.0에 추가된 것으로 표시되어 있으므로, 팀의 런타임 버전을 먼저 확인한 뒤 적용해야 합니다.

이 글에서는 `context.log()`를 언제 쓰면 좋은지, 기존 `t.diagnostic()`이나 `console.log()`와 어떻게 나눌지, CI custom reporter에서 어떤 형태로 활용하면 좋은지 정리합니다.

테스트 러너의 기본 구조가 필요하다면 [Node.js built-in test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 참고하세요.
리포터 출력 전략은 [Node.js test runner reporters 가이드](/development/blog/seo/2026/07/18/nodejs-test-runner-reporters-spec-tap-dot-junit-guide.html), flaky test 원인 추적은 [Node.js test runner workerId와 attempt 가이드](/development/blog/seo/2026/07/18/nodejs-test-runner-workerid-attempt-flaky-ci-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js test runner 공식 문서](https://nodejs.org/api/test.html), [Node.js test runner context.log 문서](https://nodejs.org/api/test.html#contextlogmessage-data), [Node.js test runner events 문서](https://nodejs.org/api/test.html#class-testsstream)

## context.log가 필요한 상황

### 테스트 실패 원인과 실행 상태를 같은 단위로 묶는다

테스트 코드에 `console.log()`를 넣으면 빠르게 값을 확인할 수 있습니다.
하지만 테스트 파일이 여러 프로세스에서 실행되고, subtest가 병렬로 섞이고, CI가 stdout을 따로 접으면 로그와 실패 테스트를 사람이 다시 맞춰야 합니다.

`context.log()`는 테스트 컨텍스트에 묶인 로그를 남깁니다.
메시지와 함께 구조화 데이터도 넘길 수 있으므로, 사람이 읽는 문장과 자동 수집 가능한 필드를 동시에 남기기 쉽습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function calculateDiscount({ tier, subtotal }) {
  if (tier === 'vip') return Math.round(subtotal * 0.15);
  if (tier === 'member') return Math.round(subtotal * 0.05);
  return 0;
}

test('vip discount is applied', (t) => {
  const input = { tier: 'vip', subtotal: 120_000 };
  const discount = calculateDiscount(input);

  t.log('discount calculation snapshot', {
    tier: input.tier,
    subtotal: input.subtotal,
    discount
  });

  assert.equal(discount, 18_000);
});
```

이 로그는 단순한 디버그 출력이라기보다 테스트 증거에 가깝습니다.
테스트가 실패했을 때 "어떤 입력과 계산 결과로 실패했는지"를 바로 볼 수 있어야 하고, 통과했을 때도 필요한 수준 이상으로 시끄럽지 않아야 합니다.

### console.log 대신 테스트 러너 이벤트로 보낸다

`console.log()`는 애플리케이션 코드와 테스트 코드가 같은 stdout을 공유합니다.
테스트 중 실행된 실제 코드가 stdout을 사용하는 경우, 테스트 보조 로그와 제품 로그가 뒤섞일 수 있습니다.

반면 `t.log()`는 테스트 러너가 해석할 수 있는 이벤트로 전달됩니다.
custom reporter를 만들면 로그의 테스트 이름, 파일, line, nesting, testId 같은 실행 문맥과 함께 처리할 수 있습니다.
CI에서 실패 테스트 주변 로그만 접거나, 특정 severity만 별도 파일로 저장하는 방식도 가능해집니다.

```js
// reporter.mjs
import { Transform } from 'node:stream';

export default new Transform({
  writableObjectMode: true,
  transform(event, encoding, callback) {
    if (event.type === 'test:log') {
      const payload = {
        test: event.data.name,
        file: event.data.file,
        message: event.data.message,
        data: event.data.data
      };

      callback(null, `${JSON.stringify(payload)}\n`);
      return;
    }

    callback();
  }
});
```

```bash
node --test --test-reporter ./reporter.mjs
```

이렇게 하면 로그를 문자열 검색에만 의존하지 않고, JSON 라인으로 수집해 CI artifact나 로그 플랫폼에 넘길 수 있습니다.
다만 reporter 출력 포맷은 팀의 운영 방식에 맞춰 작게 시작하는 편이 좋습니다.
처음부터 모든 이벤트를 저장하면 테스트 로그가 너무 커져서 정작 실패 원인을 찾기 어려워질 수 있습니다.

## diagnostic, log, assertion message를 나누는 기준

### t.diagnostic은 사람이 읽는 보조 설명에 둔다

`t.diagnostic()`은 테스트 결과에 진단 메시지를 남길 때 유용합니다.
예를 들어 fixture가 어떤 모드로 초기화됐는지, 테스트가 어떤 외부 호출을 fake로 대체했는지 설명할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('payment gateway decline path', (t) => {
  t.diagnostic('gateway client is replaced with an in-memory fake');

  const result = { status: 'declined', reason: 'insufficient_funds' };

  assert.equal(result.status, 'declined');
});
```

사람이 읽는 문장이라면 `diagnostic`이 충분합니다.
반대로 나중에 reporter에서 필드 단위로 뽑아야 하는 값이라면 `log`가 더 적합합니다.

### t.log는 구조화해야 가치가 커진다

`t.log()`를 단순 문자열 출력으로만 쓰면 `console.log()`와 큰 차이가 줄어듭니다.
실무에서는 메시지는 짧게 두고, 분석에 필요한 값은 `data` 객체로 분리하는 편이 좋습니다.

```js
test('retry budget is respected', (t) => {
  const attempts = [
    { status: 503, retryAfterMs: 100 },
    { status: 503, retryAfterMs: 200 },
    { status: 200, retryAfterMs: 0 }
  ];

  const final = attempts.at(-1);

  t.log('retry sequence completed', {
    attemptCount: attempts.length,
    statuses: attempts.map((attempt) => attempt.status),
    finalStatus: final.status
  });

  assert.equal(final.status, 200);
});
```

이 구조는 CI에서 다음 질문에 답하기 쉽습니다.

- 어떤 테스트에서 재시도가 발생했는가?
- 몇 번째 시도에서 성공했는가?
- 실패가 특정 HTTP 상태에 몰려 있는가?
- 특정 파일이나 suite에서만 로그가 늘어나는가?

로그는 테스트를 통과시키기 위한 장식이 아니라 실패 분석 비용을 줄이는 데이터입니다.
검색 가능한 필드 이름을 정하고, 팀이 자주 보는 값만 남겨야 오래 유지됩니다.

### assertion message는 실패 조건 자체를 설명한다

assertion message는 실패했을 때 가장 가까운 곳에 보이는 설명입니다.
그래서 실행 상태 전체를 담기보다 "무엇을 기대했는지"를 짧게 말하는 데 쓰는 편이 좋습니다.

```js
assert.equal(
  final.status,
  200,
  'retry sequence should eventually return a successful response'
);
```

실패 조건은 assertion message에, 실행 과정은 `t.log()`에, 테스트 설계 의도는 `t.diagnostic()`에 두면 역할이 겹치지 않습니다.
이 구분이 생기면 테스트 로그를 지우지 않고도 읽기 좋은 상태로 유지할 수 있습니다.

## 민감정보를 남기지 않는 로그 설계

### data 객체에 원본 요청을 그대로 넣지 않는다

테스트 로그도 로그입니다.
CI artifact, GitHub Actions output, 외부 로그 수집기, 팀 메신저 알림으로 복사될 수 있습니다.
그래서 `request`, `headers`, `env`, `config` 객체를 통째로 넣는 습관은 피해야 합니다.

```js
function sanitizeHeaders(headers) {
  const visible = {};

  for (const [name, value] of Object.entries(headers)) {
    if (/authorization|cookie|token|secret|key/i.test(name)) {
      visible[name] = '<redacted>';
    } else {
      visible[name] = value;
    }
  }

  return visible;
}

test('webhook request includes required headers', (t) => {
  const headers = {
    authorization: 'Bearer test-token',
    'content-type': 'application/json',
    'x-request-id': 'req_123'
  };

  t.log('webhook headers checked', {
    headers: sanitizeHeaders(headers)
  });

  assert.equal(headers['content-type'], 'application/json');
});
```

예제의 토큰은 가짜 값이지만, 실제 프로젝트에서는 테스트 fixture에도 운영 토큰과 비슷한 형식의 값이 들어갈 수 있습니다.
마스킹 함수는 테스트 헬퍼로 분리해 반복 적용하는 편이 안전합니다.

### 로컬 경로와 계정 식별자도 줄인다

테스트 실패 로그에는 파일 경로, 사용자 이름, 임시 디렉터리, 계정 ID가 쉽게 들어갑니다.
이 값들은 치명적인 비밀은 아니더라도 공개 저장소나 외부 이슈에 그대로 붙이면 부담이 됩니다.

```js
import { basename } from 'node:path';

function summarizeFixturePath(path) {
  return {
    basename: basename(path),
    location: '<fixture-dir>'
  };
}

test('fixture file is loaded', (t) => {
  const fixturePath = '/Users/example/project/test/fixtures/orders.json';

  t.log('fixture resolved', summarizeFixturePath(fixturePath));

  assert.equal(basename(fixturePath), 'orders.json');
});
```

로그에는 원인 분석에 필요한 최소 정보만 남깁니다.
전체 경로가 필요한 경우에도 기본 로그가 아니라 실패 재현 가이드나 내부 보안 채널에서 다루는 편이 좋습니다.

## CI에서 context.log를 운영하는 방식

### 실패 테스트 주변 로그만 오래 보관한다

모든 통과 테스트 로그를 오래 보관하면 비용과 노이즈가 커집니다.
대부분의 팀에서는 아래 정도의 정책으로 시작하는 것이 현실적입니다.

- 로컬 실행: `spec` reporter로 사람이 읽기 좋게 출력한다.
- PR CI: 실패 테스트의 `test:log` 이벤트를 JSON artifact로 저장한다.
- main 브랜치 야간 테스트: flaky 후보를 찾기 위해 로그 카운트와 실패 이벤트를 함께 집계한다.
- 외부 공유용 로그: 민감정보 마스킹을 한 번 더 확인한 뒤 첨부한다.

이 정책은 작은 팀에서도 적용하기 쉽습니다.
처음부터 관측 플랫폼을 크게 붙이지 않아도, `test:log` 이벤트를 JSON Lines 파일로 저장하는 것만으로 실패 분석 속도가 좋아질 수 있습니다.

### 로그 필드 이름을 테스트 도메인에 맞춘다

좋은 구조화 로그는 필드 이름이 일관적입니다.
테스트마다 `id`, `requestId`, `req_id`, `rid`가 섞이면 검색이 어려워집니다.

```js
const TEST_LOG_KEYS = {
  requestId: 'requestId',
  attemptCount: 'attemptCount',
  finalStatus: 'finalStatus',
  fixtureName: 'fixtureName'
};

test('request retry metadata is visible', (t) => {
  t.log('retry metadata', {
    [TEST_LOG_KEYS.requestId]: 'req_123',
    [TEST_LOG_KEYS.attemptCount]: 2,
    [TEST_LOG_KEYS.finalStatus]: 200
  });

  assert.ok(true);
});
```

상수까지 둘 필요가 없는 프로젝트도 많습니다.
다만 팀 단위로 자주 쓰는 필드 이름은 문서화해 두면 reporter, 대시보드, 검색 쿼리를 오래 유지하기 쉽습니다.

### 런타임 버전을 명시한다

`context.log()`는 비교적 새로운 API입니다.
문서 기준 추가 버전이 Node.js 26.6.0이므로, CI와 로컬 개발 환경이 그보다 낮으면 바로 사용할 수 없습니다.
이럴 때는 아래 순서로 적용합니다.

```json
{
  "engines": {
    "node": ">=26.6.0"
  },
  "scripts": {
    "test": "node --test",
    "test:ci": "node --test --test-reporter=spec"
  }
}
```

아직 런타임을 올릴 수 없다면 `t.diagnostic(JSON.stringify(...))` 방식으로 임시 운영할 수 있습니다.
다만 이 경우 reporter에서 구조화 데이터를 다루는 경험이 깔끔하지 않을 수 있으므로, 장기적으로는 런타임 업그레이드와 함께 `t.log()`로 옮기는 계획을 세우는 편이 좋습니다.

## 적용 체크리스트

### Node.js test runner context.log 도입 전 확인할 것

- CI와 로컬 Node.js 버전이 `context.log()`를 지원하는가?
- 로그 메시지와 구조화 데이터의 역할을 나눴는가?
- 토큰, 쿠키, API 키, 사용자 식별자, 로컬 경로를 마스킹하는가?
- custom reporter가 `test:log` 이벤트를 너무 많이 저장하지 않는가?
- assertion message, `t.diagnostic()`, `t.log()`의 사용 기준이 팀 안에서 합의됐는가?

`context.log()`의 장점은 테스트 로그를 더 많이 남기는 것이 아닙니다.
실패 원인을 찾는 데 필요한 값을 테스트 실행 문맥과 함께 남기는 것입니다.
작게 시작해서 실패가 잦은 테스트부터 구조화하면, CI 로그는 단순 출력물이 아니라 회귀를 설명하는 자료가 됩니다.


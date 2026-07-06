---
layout: post
title: "Node.js test runner timeout 가이드: 멈춘 테스트를 빠르게 실패시키는 법"
date: 2026-07-07 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-timeout-hanging-test-guide
permalink: /development/blog/seo/2026/07/07/nodejs-test-runner-timeout-hanging-test-guide.html
alternates:
  ko: /development/blog/seo/2026/07/07/nodejs-test-runner-timeout-hanging-test-guide.html
  x_default: /development/blog/seo/2026/07/07/nodejs-test-runner-timeout-hanging-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, timeout, hanging-test, testing, javascript, ci, reliability]
description: "Node.js test runner의 timeout 옵션으로 멈춘 테스트와 느린 hook을 빠르게 실패시키는 방법을 정리합니다. test timeout, hook timeout, waitFor timeout, AbortSignal 조합, CI 적용 기준을 예제로 설명합니다."
---

테스트가 실패하지 않고 계속 멈춰 있으면 CI 시간이 낭비되고 배포 판단도 늦어집니다.
특히 네트워크 mock, 파일 watcher, timer, 외부 프로세스, 데이터베이스 연결이 테스트 안에 섞이면 "언제 끝날지 모르는 테스트"가 생기기 쉽습니다.
Node.js 내장 test runner의 `timeout` 옵션은 이런 테스트를 일정 시간 뒤 실패로 바꿔 원인을 찾기 쉽게 만듭니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#testname-options-fn)는 `timeout` 옵션을 테스트가 지정한 밀리초보다 오래 걸릴 때 실패시키는 값으로 설명합니다.
다만 문서는 실행 중인 테스트가 application thread를 막고 있으면 예약된 취소가 제때 동작하지 않을 수 있다고도 설명합니다.
즉 test timeout은 "테스트를 실패로 표시하는 안전장치"이지, 모든 비동기 작업을 자동으로 정리하는 취소 시스템은 아닙니다.

테스트 파일 간 격리 기준은 [Node.js test runner isolation 가이드](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html)와 이어집니다.
CI 병렬 실행 시간을 줄이는 방법은 [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html)를 참고하세요.
외부 호출 자체의 취소 설계는 [Node.js fetch AbortSignal timeout 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)에서 함께 다뤘습니다.

## test timeout이 필요한 이유

### H3. 멈춘 테스트를 무한 대기에서 실패로 바꾼다

테스트는 통과하거나 실패해야 다음 판단을 할 수 있습니다.
하지만 Promise가 resolve되지 않거나 callback이 호출되지 않으면 테스트 프로세스가 오래 붙잡힐 수 있습니다.
이때 timeout을 걸어두면 문제를 "느린 테스트"가 아니라 "시간 제한을 넘긴 테스트"로 분명하게 볼 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('loads user profile', { timeout: 500 }, async () => {
  const profile = await loadProfile();

  assert.equal(profile.name, 'sooya');
});

async function loadProfile() {
  return new Promise(() => {
    // 버그: resolve 또는 reject가 호출되지 않는다.
  });
}
```

이 테스트는 500ms 안에 끝나지 않으면 실패합니다.
timeout이 없다면 테스트가 왜 멈췄는지 CI 로그에서 바로 보이지 않을 수 있습니다.
반대로 timeout을 너무 짧게 잡으면 정상적인 느린 환경에서도 실패하므로, 실제 실행 시간 분포를 보고 값을 정해야 합니다.

### H3. 느린 테스트를 조용히 방치하지 않는다

테스트가 가끔 10초, 30초씩 걸리기 시작하면 보통 시스템의 다른 문제가 숨어 있습니다.
fixture 생성이 비싸졌거나, 외부 네트워크에 기대고 있거나, retry가 과도하게 돌고 있을 수 있습니다.
timeout은 이런 변화를 빨리 드러내는 품질 기준으로도 사용할 수 있습니다.

```js
test('calculates invoice summary', { timeout: 1000 }, async () => {
  const summary = await calculateInvoiceSummary({
    userId: 'user_123',
    month: '2026-07'
  });

  assert.equal(summary.total, 30000);
});
```

이 예시에서 1초 제한은 성능 테스트가 아니라 단위 테스트의 정상 범위를 표현합니다.
계산 로직이 갑자기 데이터베이스나 원격 API에 의존하기 시작하면 timeout 실패로 드러날 가능성이 큽니다.
테스트의 목적이 빠른 피드백이라면, 느려지는 변화를 조기에 잡는 것이 중요합니다.

## 테스트와 hook에 timeout 적용하기

### H3. test 함수의 options에 timeout을 둔다

가장 기본적인 방법은 `test()` 두 번째 인자로 options 객체를 넘기는 것입니다.
`timeout` 값은 밀리초 단위이며, 테스트 함수가 그 시간 안에 완료되지 않으면 실패합니다.
비동기 테스트에서는 Promise가 settle되는 시간이 기준이 됩니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { setTimeout as delay } from 'node:timers/promises';

test('returns cached value quickly', { timeout: 200 }, async () => {
  const value = await readFromCache('feature:checkout');

  assert.equal(value, 'enabled');
});

async function readFromCache(key) {
  await delay(50);
  return key === 'feature:checkout' ? 'enabled' : 'disabled';
}
```

짧은 단위 테스트는 수백 밀리초 안에 끝나는 것이 자연스럽습니다.
반면 브라우저, 컨테이너, 실제 데이터베이스를 다루는 통합 테스트라면 더 긴 값을 잡을 수 있습니다.
중요한 것은 모든 테스트에 같은 숫자를 강제로 넣는 것이 아니라 테스트 종류별 기대 시간을 분리하는 것입니다.

### H3. hook timeout은 준비와 정리 실패를 드러낸다

`before`, `after`, `beforeEach`, `afterEach` 같은 hook도 timeout 옵션을 받을 수 있습니다.
테스트 본문은 빠른데 준비 단계에서 멈추는 경우가 많으므로 hook에도 별도 제한을 두는 편이 좋습니다.
특히 데이터베이스 초기화, 임시 서버 시작, fixture 다운로드 같은 작업은 명시적인 시간 상한이 필요합니다.

```js
import { before, after, test } from 'node:test';
import assert from 'node:assert/strict';

let server;

before(async () => {
  server = await startMockServer();
}, { timeout: 2000 });

after(async () => {
  await server?.close();
}, { timeout: 1000 });

test('calls mock server', { timeout: 500 }, async () => {
  const response = await fetch(server.url);

  assert.equal(response.status, 200);
});

async function startMockServer() {
  return {
    url: 'http://127.0.0.1:3000',
    close: async () => {}
  };
}
```

setup이 멈추면 테스트 본문까지 도달하지 못합니다.
정리 hook이 멈추면 다음 테스트 파일이나 CI job 종료가 지연될 수 있습니다.
테스트 본문과 hook의 timeout을 따로 두면 어느 단계가 병목인지 더 쉽게 확인할 수 있습니다.

## timeout과 취소를 분리해서 설계하기

### H3. timeout은 실패 표시이고 취소는 별도로 연결한다

test timeout이 발생한다고 해서 테스트 안에서 시작한 모든 작업이 자동으로 깔끔하게 취소되는 것은 아닙니다.
공식 문서도 timeout이 테스트를 취소하는 신뢰할 수 있는 메커니즘은 아니라고 설명합니다.
외부 요청, timer, stream, watcher는 가능하면 `AbortSignal`과 정리 로직을 함께 연결해야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('fetches upstream status', { timeout: 1000 }, async () => {
  const response = await fetch('https://example.com/status', {
    signal: AbortSignal.timeout(800)
  });

  assert.equal(response.status, 200);
});
```

이 구조에서는 테스트 전체 timeout이 1000ms이고, 실제 fetch는 800ms 안에 끝나야 합니다.
테스트 제한보다 외부 호출 제한을 조금 짧게 두면 실패 원인이 fetch timeout인지 test timeout인지 구분하기 쉽습니다.
테스트 runner의 timeout은 마지막 울타리로 두고, 개별 작업은 자기 취소 기준을 갖게 만드는 방식입니다.

### H3. CPU를 막는 코드는 timeout으로 즉시 끊기 어렵다

JavaScript 실행 스레드를 오래 점유하는 동기 루프는 timeout 처리를 지연시킬 수 있습니다.
timeout은 타이머 기반으로 동작하므로 event loop가 돌아야 실패 처리가 진행됩니다.
따라서 CPU를 오래 쓰는 코드에는 test timeout만 믿지 말고 입력 크기 제한이나 별도 worker 격리를 고려해야 합니다.

```js
test('rejects too large input before heavy work', { timeout: 300 }, () => {
  assert.throws(
    () => parsePayload('x'.repeat(2_000_000)),
    /payload too large/
  );
});

function parsePayload(payload) {
  if (payload.length > 100_000) {
    throw new Error('payload too large');
  }

  return JSON.parse(payload);
}
```

이 예시는 무거운 작업을 시작하기 전에 입력을 거부합니다.
테스트가 빠르게 실패하도록 만들려면 제품 코드에도 방어선이 있어야 합니다.
timeout은 느린 현상을 발견하는 도구이고, 근본 해결은 작업 자체가 멈추지 않게 설계하는 데 있습니다.

## waitFor timeout으로 비동기 상태를 기다리기

### H3. 조건이 언젠가 참이 되는 테스트에는 waitFor를 쓴다

최근 Node.js test runner에는 `context.waitFor()`가 있어 조건 함수가 성공할 때까지 polling할 수 있습니다.
공식 문서에 따르면 `waitFor`는 조건 함수가 성공하거나 polling timeout이 지날 때까지 반복합니다.
이 방식은 `setTimeout`으로 임의 시간을 기다리는 테스트보다 의도가 명확합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('eventually writes audit log', async (t) => {
  const logs = [];

  setTimeout(() => {
    logs.push({ type: 'LOGIN' });
  }, 80);

  await t.waitFor(() => {
    assert.equal(logs.at(-1)?.type, 'LOGIN');
  }, {
    interval: 20,
    timeout: 300
  });
});
```

이 테스트는 20ms 간격으로 조건을 확인하고 300ms 안에 성공하지 못하면 실패합니다.
무조건 300ms를 기다리는 방식보다 빠르게 끝날 수 있고, 실패 메시지도 조건 검증과 연결됩니다.
eventually consistent한 동작을 테스트할 때는 고정 sleep보다 waitFor가 더 읽기 쉽습니다.

### H3. waitFor timeout과 test timeout의 역할을 나눈다

`waitFor` timeout은 특정 조건을 기다리는 시간입니다.
test timeout은 테스트 전체 실행 시간의 상한입니다.
둘을 함께 쓸 때는 `waitFor` timeout을 test timeout보다 짧게 두는 것이 해석하기 쉽습니다.

```js
test('updates job status', { timeout: 1000 }, async (t) => {
  const job = createJob();

  job.start();

  await t.waitFor(() => {
    assert.equal(job.status, 'done');
  }, {
    interval: 50,
    timeout: 700
  });
});

function createJob() {
  return {
    status: 'pending',
    start() {
      setTimeout(() => {
        this.status = 'done';
      }, 100);
    }
  };
}
```

조건 대기에는 700ms를 주고, 나머지 시간은 테스트 준비와 정리에 남겨 둡니다.
이렇게 하면 조건이 늦은 것인지 테스트 전체가 멈춘 것인지 더 빨리 구분할 수 있습니다.
CI 로그를 읽는 사람에게도 실패 원인이 선명해집니다.

## CI에서 timeout 기준 정하기

### H3. 테스트 종류별 기본값을 나눈다

모든 테스트에 같은 timeout을 넣으면 너무 느슨하거나 너무 빡빡해집니다.
단위 테스트, 통합 테스트, e2e 성격의 테스트는 기대 시간이 다릅니다.
파일 이름, 디렉터리, test helper를 기준으로 기본값을 나누면 운영하기 쉽습니다.

```js
// test/helpers/timeouts.js
export const timeouts = {
  unit: 300,
  integration: 3000,
  external: 8000
};
```

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { timeouts } from './helpers/timeouts.js';

test('normalizes email', { timeout: timeouts.unit }, () => {
  assert.equal(normalizeEmail('USER@EXAMPLE.COM'), 'user@example.com');
});

function normalizeEmail(value) {
  return value.trim().toLowerCase();
}
```

값을 helper로 모으면 프로젝트 기준을 한곳에서 조정할 수 있습니다.
단위 테스트가 `external` timeout을 가져다 쓰기 시작하면 리뷰에서 바로 보입니다.
테스트 timeout은 숫자 자체보다 "이 테스트가 어떤 범주인가"를 드러내는 신호가 되어야 합니다.

### H3. 실패 로그에는 timeout 숫자와 작업 이름이 보여야 한다

timeout 실패가 발생했을 때 어떤 작업을 기다리던 중이었는지 알 수 있어야 합니다.
테스트 이름이 너무 추상적이면 "works" 같은 로그만 남아 원인을 찾기 어렵습니다.
테스트 이름에는 대상, 조건, 기대 결과를 넣는 편이 좋습니다.

```js
test('payment callback marks order as paid within polling window', {
  timeout: 1500
}, async (t) => {
  const order = { status: 'pending' };

  setTimeout(() => {
    order.status = 'paid';
  }, 100);

  await t.waitFor(() => {
    assert.equal(order.status, 'paid');
  }, {
    timeout: 1000
  });
});
```

이름만 봐도 payment callback, order status, polling window가 핵심임을 알 수 있습니다.
CI에서 같은 테스트가 반복적으로 timeout된다면 외부 의존성, polling 조건, fixture 상태를 먼저 보면 됩니다.
좋은 테스트 이름은 timeout 디버깅 시간을 줄이는 작은 문서입니다.

## 발행 전 실무 체크리스트

### H3. timeout을 넣기 전에 확인할 항목

timeout은 멈춘 테스트를 숨기는 도구가 아니라 드러내는 도구여야 합니다.
값만 늘려서 실패를 피하면 느린 원인이 계속 남습니다.
아래 항목을 기준으로 테스트를 점검하세요.

- 테스트가 실제 네트워크나 외부 서비스에 의존하지 않는가?
- Promise가 모든 경로에서 resolve 또는 reject되는가?
- timer, watcher, server, socket을 테스트 뒤 정리하는가?
- `waitFor`가 필요한 곳에 고정 sleep을 쓰고 있지 않은가?
- test timeout보다 fetch, DB, polling timeout이 더 짧게 잡혀 있는가?
- CI runner 성능을 고려한 값인지, 로컬 기준만 반영한 값인지 확인했는가?

이 체크리스트는 [Node.js test runner global setup 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-global-setup-teardown-guide.html)에서 다룬 fixture 수명 관리와도 연결됩니다.
테스트가 끝나도 리소스가 남아 있으면 다음 파일의 timeout으로 이어질 수 있습니다.
timeout 실패는 단일 테스트 문제가 아니라 테스트 환경 전체의 정리 문제일 수도 있습니다.

## FAQ

### H3. 모든 테스트에 timeout을 넣어야 하나요?

모든 테스트에 기계적으로 넣을 필요는 없습니다.
하지만 외부 I/O, timer, polling, mock server, worker, stream을 다루는 테스트에는 명시적인 timeout이 도움이 됩니다.
단위 테스트는 helper로 짧은 기본값을 두고, 통합 테스트는 더 긴 기준을 분리하는 방식이 실용적입니다.

### H3. timeout을 크게 잡으면 안정적인가요?

아닙니다.
timeout을 크게 잡으면 실패가 늦게 드러날 뿐입니다.
테스트가 느린 이유가 외부 의존성, retry 폭증, 정리 누락이라면 값 증가보다 원인 분리가 먼저입니다.

### H3. test timeout과 AbortSignal.timeout은 같은가요?

역할이 다릅니다.
test timeout은 테스트 runner가 테스트를 실패로 판정하는 기준입니다.
`AbortSignal.timeout()`은 fetch 같은 개별 작업에 취소 신호를 전달하는 도구입니다.
실무에서는 개별 작업 timeout을 더 짧게 두고, test timeout을 전체 상한으로 두는 구성이 읽기 쉽습니다.

## 마무리

Node.js test runner의 timeout 옵션은 멈춘 테스트를 빠르게 실패시키는 기본 안전장치입니다.
하지만 timeout만으로 리소스 정리와 작업 취소가 자동 해결되지는 않습니다.
테스트, hook, waitFor, 외부 호출의 시간 제한을 분리해두면 CI 실패를 훨씬 빠르게 해석할 수 있습니다.

새 테스트를 추가할 때는 "이 테스트가 정상이라면 얼마나 빨리 끝나야 하는가"를 함께 정하세요.
그 기준이 코드에 들어가면 느려진 테스트가 조용히 쌓이지 않습니다.
결과적으로 timeout은 테스트를 더 엄격하게 만들기보다, 팀이 신뢰할 수 있는 피드백 시간을 지키는 운영 기준이 됩니다.

## 관련 글

- [Node.js test runner isolation 가이드: 테스트 파일을 안전하게 격리 실행하는 방법](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html)
- [Node.js test runner global setup 가이드: 테스트 전후 공통 준비를 한곳에서 관리하는 법](/development/blog/seo/2026/07/05/nodejs-test-runner-global-setup-teardown-guide.html)
- [Node.js fetch AbortSignal timeout 가이드: 외부 호출 타임아웃과 재시도 다루기](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)

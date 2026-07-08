---
layout: post
title: "Node.js test runner signal 가이드: 중단된 테스트의 비동기 작업을 정리하는 법"
date: 2026-07-08 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-signal-abort-cleanup-guide
permalink: /development/blog/seo/2026/07/08/nodejs-test-runner-signal-abort-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/07/08/nodejs-test-runner-signal-abort-cleanup-guide.html
  x_default: /development/blog/seo/2026/07/08/nodejs-test-runner-signal-abort-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, signal, abortsignal, cancellation, cleanup, testing, javascript, ci]
description: "Node.js test runner의 TestContext.signal로 중단된 테스트의 fetch, timer, stream, watcher 같은 비동기 작업을 함께 취소하는 방법을 정리합니다. timeout, t.after 정리, AbortSignal 조합, CI 안정화 패턴을 예제로 설명합니다."
---

테스트 timeout은 멈춘 테스트를 실패로 표시해 주지만, 테스트 안에서 시작한 비동기 작업을 항상 깔끔하게 정리해 주지는 않습니다.
fetch 요청, timer, stream, watcher, child process 같은 작업이 남아 있으면 테스트는 실패했는데 프로세스가 늦게 끝나거나 다음 테스트에 영향을 줄 수 있습니다.
Node.js 내장 test runner의 `TestContext.signal`은 테스트가 중단될 때 내부 작업에도 취소 신호를 전달할 수 있는 안전장치입니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html)는 `context.signal`을 테스트가 abort될 때 하위 작업을 중단하는 데 사용할 수 있는 `AbortSignal`로 설명합니다.
또한 timeout은 테스트 실패를 표시하는 장치에 가깝기 때문에, 실제 비동기 작업에는 별도의 취소 신호와 정리 로직을 연결하는 편이 좋습니다.
이 글에서는 `t.signal`을 fetch, delay, stream 성격의 작업에 연결하고, `t.after`와 함께 리소스를 닫는 실무 패턴을 정리합니다.

멈춘 테스트를 빨리 실패시키는 기준은 [Node.js test runner timeout 가이드](/development/blog/seo/2026/07/07/nodejs-test-runner-timeout-hanging-test-guide.html)에서 먼저 다뤘습니다.
테스트 파일 격리와 child process 실행 모델은 [Node.js test runner isolation 가이드](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html)를 참고하세요.
외부 요청 자체의 timeout과 retry 설계는 [Node.js fetch AbortSignal timeout 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)와 함께 보면 좋습니다.

## TestContext.signal이 필요한 이유

### H3. 실패한 테스트의 남은 작업을 줄인다

테스트가 timeout으로 실패해도, 테스트 안에서 시작한 비동기 작업이 즉시 멈춘다는 보장은 없습니다.
특히 네트워크 요청이나 긴 timer는 별도 취소 신호를 받지 않으면 계속 실행될 수 있습니다.
`t.signal`을 작업에 전달하면 테스트가 중단될 때 작업도 같은 생명주기를 따르게 만들 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('loads upstream status', { timeout: 1000 }, async (t) => {
  const response = await fetch('https://example.com/status', {
    signal: t.signal
  });

  assert.equal(response.status, 200);
});
```

이 예시에서 테스트가 중단되면 fetch도 abort 신호를 받을 수 있습니다.
timeout은 테스트 결과를 실패로 바꾸고, `t.signal`은 테스트 내부 작업이 더 오래 살아남지 않도록 돕습니다.
두 기능은 경쟁 관계가 아니라 서로 다른 층의 안전장치입니다.

### H3. 테스트 종료 후 로그와 부작용을 줄인다

취소되지 않은 작업은 테스트가 끝난 뒤 로그를 남기거나, 열려 있는 handle 때문에 프로세스 종료를 늦출 수 있습니다.
CI에서는 이런 문제가 "가끔 늦는 테스트", "알 수 없는 open handle", "다음 테스트에서만 실패하는 상태 오염"으로 보입니다.
작업 시작 지점에서 signal을 받도록 설계하면 실패 후 정리 비용이 줄어듭니다.

```text
symptom:
- timeout failure after 1000ms
- request still retries in the background
- after hook runs, but process exits late

fix:
- pass t.signal to the async operation
- close explicit resources in t.after
- avoid shared global state between tests
```

테스트가 실패했을 때 중요한 것은 실패를 빨리 보는 것뿐 아니라 실패 뒤 남은 상태를 줄이는 것입니다.
`t.signal`은 작업 단위 취소를 테스트 생명주기와 연결하는 가장 직접적인 방법입니다.
리소스가 명시적으로 열리는 경우에는 signal만으로 끝내지 말고 정리 hook도 함께 둬야 합니다.

## fetch와 delay에 signal 연결하기

### H3. fetch는 t.signal을 그대로 받을 수 있다

Node.js의 fetch는 `AbortSignal`을 지원합니다.
테스트 함수의 첫 번째 인자인 context에서 `signal`을 꺼내 fetch 옵션에 넘기면 됩니다.
이렇게 하면 테스트가 취소될 때 외부 요청도 같은 취소 경로를 타게 됩니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

async function getHealth(url, signal) {
  const response = await fetch(`${url}/health`, { signal });

  return response.json();
}

test('returns healthy status', { timeout: 1500 }, async (t) => {
  const result = await getHealth('https://example.com', t.signal);

  assert.equal(result.status, 'ok');
});
```

테스트 대상 함수가 signal을 인자로 받을 수 있게 만들면 테스트뿐 아니라 실제 서비스 코드에서도 취소 처리가 쉬워집니다.
HTTP 요청, 데이터 조회, 큐 작업처럼 오래 걸릴 수 있는 함수는 가능하면 `signal` 옵션을 받도록 설계하세요.
테스트에서만 쓰는 특수한 장치가 아니라 운영 코드의 취소 가능성을 높이는 인터페이스가 됩니다.

### H3. timers/promises delay도 취소 가능하게 만든다

retry backoff나 polling 테스트에서는 `setTimeout` 계열 delay가 자주 등장합니다.
`node:timers/promises`의 `setTimeout`은 signal 옵션을 받을 수 있으므로, 테스트 취소 시 기다림도 중단할 수 있습니다.
긴 delay를 테스트에 직접 넣어야 한다면 반드시 취소 가능하게 만드는 편이 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { setTimeout as delay } from 'node:timers/promises';

async function waitForReady(check, signal) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    if (await check()) {
      return true;
    }

    await delay(100, undefined, { signal });
  }

  return false;
}

test('waits until service is ready', { timeout: 1000 }, async (t) => {
  let count = 0;

  const ready = await waitForReady(async () => {
    count += 1;
    return count === 3;
  }, t.signal);

  assert.equal(ready, true);
});
```

이 패턴은 polling 테스트의 종료 조건을 분명하게 만듭니다.
테스트 timeout이 먼저 발생하면 delay가 abort되고, 불필요하게 남은 반복을 계속하지 않습니다.
CI에서 느린 실패를 줄이는 데 특히 효과적입니다.

## timeout과 signal을 함께 설계하기

### H3. test timeout은 바깥 제한으로 둔다

테스트 timeout은 전체 테스트가 얼마 안에 끝나야 하는지 표현합니다.
반면 개별 작업 timeout은 외부 요청이나 polling 같은 세부 작업의 한계를 표현합니다.
실무에서는 작업 timeout을 더 짧게 두고, test timeout을 마지막 보호막으로 두는 편이 원인 분석에 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function combineSignals(...signals) {
  return AbortSignal.any(signals.filter(Boolean));
}

test('calls upstream within budget', { timeout: 1200 }, async (t) => {
  const signal = combineSignals(t.signal, AbortSignal.timeout(900));

  const response = await fetch('https://example.com/api', { signal });

  assert.equal(response.status, 200);
});
```

이 예시에서는 fetch가 900ms 안에 끝나야 하고, 테스트 전체는 1200ms 안에 끝나야 합니다.
fetch 제한을 넘으면 요청 timeout 원인을 볼 수 있고, 그 밖의 정리까지 포함해 전체 테스트가 너무 오래 걸리면 test timeout이 잡아 줍니다.
제한을 한 곳에만 몰아두기보다 작업별 시간 예산을 나누는 것이 디버깅에 유리합니다.

### H3. abort 에러를 무조건 숨기지 않는다

취소는 정상적인 제어 흐름일 수 있지만, 모든 abort 에러를 조용히 삼키면 실제 문제를 놓칠 수 있습니다.
테스트가 의도적으로 취소를 검증하는 경우와, 예상치 못한 timeout으로 취소된 경우를 구분해야 합니다.
특히 production 코드의 catch block에서 `AbortError`를 무조건 성공처럼 처리하지 않도록 주의하세요.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { setTimeout as delay } from 'node:timers/promises';

test('cancels a pending delay', async () => {
  const controller = new AbortController();
  const pending = delay(1000, 'done', { signal: controller.signal });

  controller.abort();

  await assert.rejects(pending, {
    name: 'AbortError'
  });
});
```

취소 자체를 테스트할 때는 reject 형태와 에러 이름을 명확히 검증하는 편이 좋습니다.
반대로 일반 테스트에서 abort가 발생했다면 timeout이나 조기 종료 원인을 로그와 함께 확인해야 합니다.
취소를 숨기는 것이 아니라 관찰 가능한 실패로 만드는 것이 목표입니다.

## t.after와 함께 리소스 정리하기

### H3. signal은 취소이고 after는 닫기다

`t.signal`은 비동기 작업에 취소 신호를 전달합니다.
하지만 서버, 파일 handle, watcher, 데이터베이스 연결처럼 명시적으로 닫아야 하는 리소스는 `t.after`에서 정리해야 합니다.
취소와 close는 역할이 다르므로 둘 다 필요한 경우가 많습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('uses a temporary watcher', async (t) => {
  const watcher = createFakeWatcher();

  t.after(() => {
    watcher.close();
  });

  const event = await watcher.next({ signal: t.signal });

  assert.equal(event.type, 'change');
});

function createFakeWatcher() {
  return {
    async next({ signal }) {
      signal.throwIfAborted();
      return { type: 'change' };
    },
    close() {}
  };
}
```

이 구조에서 `watcher.next()`는 취소 신호를 받고, watcher 자체는 테스트가 끝날 때 닫힙니다.
실제 파일 watcher나 서버 객체도 같은 생각으로 다루면 됩니다.
작업 취소와 리소스 정리를 분리하면 실패 상황에서도 남는 handle을 줄일 수 있습니다.

### H3. after 정리는 idempotent하게 만든다

정리 함수는 여러 경로에서 호출될 수 있다고 생각하고 작성하는 편이 안전합니다.
테스트가 성공해도 정리하고, 실패해도 정리하며, 중간에 abort가 발생해도 정리해야 합니다.
따라서 close 함수는 이미 닫힌 상태에서도 문제없이 동작하도록 만드는 것이 좋습니다.

```js
function createConnection() {
  let closed = false;

  return {
    async query(sql, { signal } = {}) {
      signal?.throwIfAborted();
      return [{ sql }];
    },
    async close() {
      if (closed) {
        return;
      }

      closed = true;
    }
  };
}
```

idempotent한 정리는 테스트 실패 후 디버깅을 단순하게 만듭니다.
정리 과정 자체가 다시 실패하면 원래 테스트 실패 원인이 가려질 수 있습니다.
테스트 리소스 close는 가능하면 작고, 예측 가능하고, 여러 번 호출되어도 안전해야 합니다.

## CI에서 적용할 기준

### H3. 오래 걸리는 테스트부터 signal을 넣는다

모든 테스트에 한 번에 취소 설계를 넣기는 어렵습니다.
먼저 timeout이 있거나, 외부 요청을 하거나, polling과 retry를 포함하거나, watcher와 서버를 여는 테스트부터 `t.signal`을 연결하세요.
실패했을 때 남은 작업이 큰 테스트일수록 효과가 큽니다.

```text
priority:
- fetch, undici, database client, queue consumer
- timers/promises delay, retry backoff, polling loop
- fs watcher, stream pipeline, local test server
- child process, worker thread, long-running fixture
```

이 목록에 해당하는 테스트는 timeout만으로 충분하지 않을 가능성이 높습니다.
테스트가 중단될 때 실제 작업도 멈출 수 있는지 확인해야 합니다.
CI 로그에서 open handle, 늦은 stderr, 종료 지연이 보인다면 signal 연결을 우선 점검하세요.

### H3. 테스트 helper의 표준 인자로 만든다

`t.signal`을 매번 직접 넘기는 방식도 가능하지만, 테스트 helper가 signal 옵션을 받게 만들면 누락이 줄어듭니다.
HTTP helper, fixture loader, polling utility, mock server client 같은 공통 함수는 `{ signal }` 옵션을 표준으로 두는 편이 좋습니다.
이렇게 하면 새 테스트를 작성할 때도 취소 가능한 구조가 기본값이 됩니다.

```js
async function createTestClient({ baseUrl, signal }) {
  return {
    async get(path) {
      return fetch(`${baseUrl}${path}`, { signal });
    }
  };
}

test('uses a cancellable test client', async (t) => {
  const client = await createTestClient({
    baseUrl: 'https://example.com',
    signal: t.signal
  });

  const response = await client.get('/health');

  assert.equal(response.status, 200);
});
```

helper가 signal을 받으면 테스트 작성자가 취소 처리 세부사항을 매번 기억하지 않아도 됩니다.
공통 helper의 인터페이스를 조금 바꾸는 것만으로 테스트 스위트 전체의 종료 안정성이 좋아질 수 있습니다.
특히 CI에서만 가끔 멈추는 테스트를 줄이는 데 도움이 됩니다.

## 적용 전 체크리스트

### H3. 테스트 중단 시나리오를 기준으로 점검한다

`TestContext.signal`은 테스트가 실패하거나 중단될 때 더 빛나는 기능입니다.
성공 경로만 보면 없어도 되는 코드처럼 보일 수 있지만, CI에서는 실패 뒤 남는 작업이 전체 파이프라인을 느리게 만듭니다.
취소와 정리를 테스트 설계의 일부로 보세요.

- timeout이 있는 테스트에서 내부 비동기 작업에도 `t.signal`을 전달하는가?
- fetch, delay, polling, retry helper가 `{ signal }` 옵션을 받는가?
- 서버, watcher, handle, connection은 `t.after`에서 닫히는가?
- abort 에러를 의도한 경우와 예상치 못한 경우로 구분하는가?
- 정리 함수가 여러 번 호출되어도 안전한가?

Node.js test runner의 `t.signal`은 테스트 실패를 더 짧고 깨끗하게 만드는 도구입니다.
timeout이 "언제 실패할지"를 정한다면, signal은 "실패할 때 무엇을 멈출지"를 정합니다.
긴 비동기 작업과 명시적 리소스 정리를 테스트 생명주기와 연결하면, CI에서 느린 종료와 불안정한 후속 실패를 줄일 수 있습니다.

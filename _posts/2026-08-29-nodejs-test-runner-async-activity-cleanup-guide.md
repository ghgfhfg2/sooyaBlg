---
layout: post
title: "Node.js test runner async activity 가이드: 테스트 종료 후 남는 비동기 작업 정리법"
date: 2026-08-29 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-async-activity-cleanup-guide
permalink: /development/blog/seo/2026/08/29/nodejs-test-runner-async-activity-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/08/29/nodejs-test-runner-async-activity-cleanup-guide.html
  x_default: /development/blog/seo/2026/08/29/nodejs-test-runner-async-activity-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, async-activity, cleanup, testing, ci, timer, javascript]
description: "Node.js test runner에서 테스트 함수가 끝난 뒤 남는 timer, fetch, watcher, promise 같은 extraneous asynchronous activity를 줄이는 방법을 정리합니다. t.signal, t.after, timeout, CI 진단 기준을 예제로 설명합니다."
---

테스트가 통과했다고 해서 테스트 안에서 시작한 모든 비동기 작업이 정리됐다는 뜻은 아닙니다.
`setInterval`, 백그라운드 promise, 파일 watcher, fetch 요청, mock 서버가 테스트 함수보다 오래 살아 있으면 다음 테스트를 흔들거나 CI 프로세스를 늦게 종료시킬 수 있습니다.

Node.js test runner 공식 문서는 테스트 함수가 끝난 뒤에도 남아 있는 비동기 작업을 extraneous asynchronous activity로 설명합니다.
테스트 러너는 이런 작업을 감지해 다루지만, 완료를 기다리기 위해 테스트 결과 보고를 늦추지는 않습니다.
즉 테스트가 끝났다면 그 테스트가 시작한 작업도 스스로 정리되도록 설계해야 합니다.

이 글에서는 Node.js test runner에서 테스트 종료 후 남는 비동기 작업을 줄이는 기준을 정리합니다.
기본 테스트 구조는 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html), 취소 신호 연결은 [Node.js TestContext.signal 가이드](/development/blog/seo/2026/07/08/nodejs-test-runner-signal-abort-cleanup-guide.html), 파일 단위 정리는 [Node.js test runner hooks 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js test runner 공식 문서](https://nodejs.org/api/test.html)

## async activity가 문제가 되는 순간

### 테스트 함수가 끝난 뒤 작업이 계속 돈다

가장 흔한 문제는 테스트가 비동기 작업을 시작만 하고 기다리거나 취소하지 않는 경우입니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('starts background polling', async () => {
  setInterval(() => {
    // polling metrics
  }, 1000);

  assert.equal(1 + 1, 2);
});
```

이 테스트는 assertion만 보면 통과합니다.
하지만 interval은 테스트가 끝난 뒤에도 계속 남습니다.
작은 예제에서는 단순한 실수처럼 보이지만, 실제 서비스 테스트에서는 watcher, queue consumer, retry loop, mock 서버가 같은 방식으로 남을 수 있습니다.

### 다음 테스트의 결과를 오염시킨다

남은 비동기 작업은 단순히 프로세스를 오래 붙잡는 것에서 끝나지 않습니다.
공유 파일을 다시 쓰거나, mock 서버 요청 기록을 바꾸거나, 로그를 늦게 남기면 다음 테스트의 실패 원인이 됩니다.

```js
import { writeFile } from 'node:fs/promises';
import test from 'node:test';
import assert from 'node:assert/strict';

test('writes report later', async () => {
  setTimeout(() => {
    void writeFile('./tmp/report.json', '{"status":"late"}');
  }, 50);

  assert.equal(true, true);
});
```

테스트 함수는 timer가 실행되기 전에 끝납니다.
뒤늦은 파일 쓰기는 다른 테스트가 같은 경로를 읽을 때 간헐적인 실패를 만들 수 있습니다.

## t.after로 리소스 생명주기를 묶는다

### 시작한 리소스는 같은 테스트에서 닫는다

테스트 안에서 서버, timer, watcher를 만들었다면 정리 책임도 같은 테스트에 두는 편이 읽기 쉽습니다.
Node.js test runner의 `t.after()`는 테스트가 끝날 때 실행할 정리 함수를 등록하는 데 적합합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('polls until condition is met', async (t) => {
  const interval = setInterval(() => {
    // collect test-only diagnostics
  }, 1000);

  t.after(() => {
    clearInterval(interval);
  });

  assert.equal(typeof interval, 'object');
});
```

핵심은 리소스를 만든 직후에 정리 함수를 등록하는 것입니다.
중간 assertion에서 실패하더라도 정리 경로가 남아 있어야 합니다.

### 서버 종료도 Promise로 기다린다

HTTP 서버를 여는 테스트에서는 `server.close()`를 호출하는 것만으로 충분하지 않을 수 있습니다.
테스트는 서버가 실제로 닫힐 때까지 기다려야 다음 테스트와 포트, 연결, 로그가 섞이지 않습니다.

```js
import http from 'node:http';
import test from 'node:test';
import assert from 'node:assert/strict';

function closeServer(server) {
  return new Promise((resolve, reject) => {
    server.close((error) => {
      if (error) reject(error);
      else resolve();
    });
  });
}

test('serves health response', async (t) => {
  const server = http.createServer((req, res) => {
    res.end('ok');
  });

  await new Promise((resolve, reject) => {
    server.once('error', reject);
    server.listen(0, '127.0.0.1', resolve);
  });

  t.after(() => closeServer(server));

  const address = server.address();

  assert.notEqual(address, null);
});
```

서버 테스트가 많다면 [Node.js server asyncDispose 가이드](/development/blog/seo/2026/08/18/nodejs-server-asyncdispose-graceful-shutdown-guide.html)의 종료 패턴을 함께 적용할 수 있습니다.

## t.signal로 취소 가능한 작업을 연결한다

### fetch와 timer에는 AbortSignal을 넘긴다

테스트가 timeout으로 중단되거나 실패했을 때, 내부 작업도 같은 생명주기를 따라야 합니다.
`t.signal`은 테스트가 abort될 때 하위 작업에 취소 신호를 전달하는 표준 통로입니다.

```js
import { setTimeout as delay } from 'node:timers/promises';
import test from 'node:test';
import assert from 'node:assert/strict';

test('waits for async condition', { timeout: 500 }, async (t) => {
  await delay(100, undefined, { signal: t.signal });

  assert.equal(true, true);
});
```

fetch, timer promise, stream pipeline, watcher처럼 signal을 받는 API라면 테스트 helper도 `{ signal }` 옵션을 받게 만드는 편이 좋습니다.

```js
export async function fetchJson(url, { signal } = {}) {
  const response = await fetch(url, { signal });

  return response.json();
}
```

```js
test('loads json fixture', async (t) => {
  const body = await fetchJson('https://example.com/data.json', {
    signal: t.signal
  });

  assert.equal(typeof body, 'object');
});
```

외부 요청 예제는 실제 테스트에서는 로컬 mock 서버로 바꾸는 편이 안정적입니다.
중요한 점은 테스트가 끝났을 때 요청도 같이 끝날 수 있는 구조를 만드는 것입니다.

### timeout과 사용자 취소를 함께 묶는다

테스트 timeout과 helper 내부 timeout이 모두 필요할 때는 `AbortSignal.any()`로 신호를 합칠 수 있습니다.

```js
export async function readWithTimeout(read, { signal, timeoutMs = 300 } = {}) {
  const timeoutSignal = AbortSignal.timeout(timeoutMs);
  const signals = [timeoutSignal];

  if (signal) {
    signals.push(signal);
  }

  return read({
    signal: AbortSignal.any(signals)
  });
}
```

이렇게 하면 테스트가 abort돼도 helper가 계속 기다리지 않고, helper 자체의 제한 시간도 유지됩니다.
취소 신호 조합 기준은 [Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)를 참고할 수 있습니다.

## CI에서 남은 작업을 찾는 기준

### 열린 handle이 무엇인지 먼저 줄인다

CI에서 테스트가 끝났는데 프로세스가 늦게 종료된다면 아래 항목을 먼저 확인합니다.

- `setInterval`을 만들고 `clearInterval`하지 않은 테스트
- `setTimeout` callback 안에서 뒤늦게 파일을 쓰는 코드
- 닫히지 않은 HTTP 서버나 socket
- 종료하지 않은 file watcher
- await하지 않은 promise
- 실패 경로에서 cleanup이 빠지는 helper

문제가 되는 테스트를 찾을 때는 한 번에 설정을 여러 개 바꾸지 않는 편이 좋습니다.
먼저 테스트 파일 범위를 좁히고, 그다음 timeout, concurrency, randomize 순서로 확인합니다.

```bash
node --test test/report.test.mjs
node --test --test-name-pattern="report"
node --test --test-concurrency=1
```

테스트 이름 필터링은 [Node.js test runner name pattern 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html)와 함께 보면 재현 범위를 줄이는 데 도움이 됩니다.

### cleanup helper를 테스트한다

정리 코드는 "있다"가 아니라 "실제로 호출된다"까지 확인해야 합니다.
특히 공통 helper가 서버나 watcher를 열어 준다면 cleanup 함수가 실패 경로에서도 실행되는지 테스트로 고정하는 편이 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('cleanup is called when test helper fails', async () => {
  let cleaned = false;

  async function runWithCleanup(work) {
    try {
      await work();
    } finally {
      cleaned = true;
    }
  }

  await assert.rejects(
    runWithCleanup(async () => {
      throw new Error('fixture failed');
    }),
    /fixture failed/
  );

  assert.equal(cleaned, true);
});
```

공통 fixture는 정상 경로보다 실패 경로에서 더 자주 문제를 만듭니다.
`finally`, `t.after`, `AbortSignal` 중 하나로 정리 책임이 코드에 드러나야 합니다.

## 도입 체크리스트

- 테스트 안에서 시작한 timer, server, watcher, queue consumer를 같은 테스트에서 정리하는가?
- 리소스를 만든 직후 `t.after()` 또는 `try/finally`로 cleanup을 등록하는가?
- fetch, delay, watcher 같은 취소 가능한 API에 `t.signal`을 전달하는가?
- helper 함수가 `{ signal }` 옵션을 받아 테스트 생명주기와 연결되는가?
- timeout 이후에도 실행될 수 있는 callback이 공유 파일이나 mock 상태를 바꾸지 않는가?
- CI에서 hanging 테스트를 좁힐 수 있는 name pattern, 단일 파일 실행, concurrency 조정 명령이 있는가?

## FAQ

### 테스트가 통과했는데 왜 비동기 작업 정리가 중요한가요?

테스트 결과는 통과해도 남은 작업이 다음 테스트의 파일, mock 상태, 로그, 서버 포트를 바꿀 수 있습니다.
간헐 실패를 줄이려면 테스트 함수가 끝날 때 내부 작업도 함께 끝나야 합니다.

### t.after와 afterEach 중 무엇을 써야 하나요?

특정 테스트에서 만든 리소스는 `t.after()`가 읽기 쉽습니다.
모든 테스트가 같은 방식으로 초기화하고 정리해야 하는 상태라면 `beforeEach`와 `afterEach`가 더 적합합니다.

### t.signal만 넘기면 cleanup이 끝나나요?

아닙니다.
`t.signal`은 취소 가능한 작업에 중단 신호를 전달합니다.
서버 종료, 임시 파일 삭제, interval 해제처럼 명시적인 정리가 필요한 리소스는 `t.after()`나 `finally`에서 별도로 처리해야 합니다.

## 마무리

Node.js test runner의 async activity 문제는 대부분 "시작한 곳과 정리하는 곳이 멀리 떨어진 코드"에서 생깁니다.
테스트 안에서 리소스를 만들었다면 바로 cleanup을 등록하고, 취소 가능한 비동기 작업에는 `t.signal`을 넘기는 습관이 필요합니다.

CI에서 간헐 실패가 늘었다면 assertion만 보지 말고 테스트가 끝난 뒤에도 살아 있는 작업을 확인하세요.
남은 timer, watcher, 요청, 서버를 줄이는 것만으로도 테스트 결과는 훨씬 예측 가능해집니다.

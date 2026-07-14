---
layout: post
title: "Node.js test runner force exit 가이드: 끝나지 않는 테스트 프로세스를 다루는 법"
date: 2026-07-15 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-force-exit-hanging-process-guide
permalink: /development/blog/seo/2026/07/15/nodejs-test-runner-force-exit-hanging-process-guide.html
alternates:
  ko: /development/blog/seo/2026/07/15/nodejs-test-runner-force-exit-hanging-process-guide.html
  x_default: /development/blog/seo/2026/07/15/nodejs-test-runner-force-exit-hanging-process-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, force-exit, hanging-test, ci, testing, javascript, cleanup]
description: "Node.js test runner의 --test-force-exit 옵션으로 종료되지 않는 테스트 프로세스를 임시로 끊는 방법을 정리합니다. 열린 핸들 진단, 타임아웃과의 차이, CI 적용 기준, 근본 원인 정리 패턴을 예제로 설명합니다."
---

테스트가 모두 통과했는데도 `node --test` 프로세스가 끝나지 않는 경우가 있습니다.
대부분은 테스트 본문이 실패해서가 아니라 서버, timer, socket, watcher, DB 연결 같은 리소스가 이벤트 루프를 계속 붙잡고 있기 때문입니다.
이 상태를 그대로 두면 로컬에서는 터미널이 멈춘 것처럼 보이고, CI에서는 job timeout까지 시간을 낭비합니다.

Node.js test runner의 `--test-force-exit` 옵션은 이런 상황에서 테스트 실행이 끝난 뒤 프로세스를 강제로 종료하도록 만드는 안전핀입니다.
다만 이 옵션은 원인을 해결하는 도구가 아니라 CI가 무한 대기하지 않게 막는 임시 장치에 가깝습니다.
테스트가 멈추는 원인을 좁히는 방법은 [Node.js test runner timeout 가이드](/development/blog/seo/2026/07/07/nodejs-test-runner-timeout-hanging-test-guide.html), 열린 리소스 진단은 [Node.js process.getActiveResourcesInfo 가이드](/development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html), 취소 가능한 정리는 [Node.js test runner AbortSignal cleanup 가이드](/development/blog/seo/2026/07/08/nodejs-test-runner-signal-abort-cleanup-guide.html)와 함께 보면 좋습니다.

## force exit가 필요한 상황

### H3. 테스트는 끝났지만 이벤트 루프가 남아 있을 때

`--test-force-exit`이 겨냥하는 문제는 assertion 실패가 아닙니다.
테스트 파일의 실행은 끝났지만 Node.js 프로세스 안에 아직 살아 있는 비동기 리소스가 있어 종료되지 않는 상황입니다.

아래 테스트는 HTTP 서버를 열고 닫지 않습니다.
assertion은 통과하지만 서버 handle이 남아 있기 때문에 프로세스가 계속 살아 있을 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import http from 'node:http';

test('server responds with ok', async () => {
  const server = http.createServer((req, res) => {
    res.end('ok');
  });

  await new Promise((resolve) => server.listen(0, resolve));

  const { port } = server.address();
  const response = await fetch(`http://127.0.0.1:${port}`);

  assert.equal(await response.text(), 'ok');
});
```

좋은 해결책은 `server.close()`를 호출하는 것입니다.
`--test-force-exit`을 붙이면 CI가 무한 대기하는 문제는 줄일 수 있지만, 열린 서버를 정리하지 않은 테스트 품질 문제는 그대로 남습니다.

```js
test('server responds with ok and closes handle', async (t) => {
  const server = http.createServer((req, res) => {
    res.end('ok');
  });

  t.after(() => {
    server.close();
  });

  await new Promise((resolve) => server.listen(0, resolve));

  const { port } = server.address();
  const response = await fetch(`http://127.0.0.1:${port}`);

  assert.equal(await response.text(), 'ok');
});
```

force exit는 이런 정리 누락을 숨길 수 있습니다.
그래서 기본 test script에 바로 넣기보다, 원인 조사와 CI 보호를 분리해서 다루는 편이 안전합니다.

### H3. CI job timeout을 막는 임시 보호막으로 쓴다

테스트 프로세스가 끝나지 않으면 CI는 보통 10분, 30분, 60분 같은 상위 timeout에 도달할 때까지 기다립니다.
이 시간은 실패 원인을 알려 주지 못하고 배포 파이프라인만 지연시킵니다.

이때 `--test-force-exit`은 테스트 결과가 이미 나온 뒤 프로세스를 종료하게 해 CI 대기를 줄이는 데 도움이 됩니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "node --test --test-force-exit"
  }
}
```

로컬 기본값은 그대로 두고 CI 전용 script에만 적용하면, 개발자는 열린 핸들을 직접 체감할 수 있습니다.
반면 CI는 전체 파이프라인이 무한 대기하지 않도록 보호됩니다.
이 구분은 force exit를 남용하지 않게 해 줍니다.

## timeout과 force exit의 차이

### H3. timeout은 느린 테스트 본문을 실패시킨다

test runner의 timeout은 특정 테스트가 너무 오래 걸릴 때 실패시키는 장치입니다.
예를 들어 Promise가 resolve되지 않거나, 테스트 본문 안에서 기다리는 작업이 멈추면 timeout이 직접적인 신호를 줍니다.

```js
import test from 'node:test';

test('waits for a delayed job', { timeout: 1000 }, async () => {
  await new Promise(() => {
    // never resolves
  });
});
```

이 경우 문제는 테스트 본문 안에 있습니다.
timeout이 실패 위치를 알려 주므로, 해당 테스트를 작게 나누거나 비동기 조건을 명확히 해야 합니다.

### H3. force exit는 실행 이후 남은 리소스를 끊는다

force exit는 테스트 본문이 모두 끝난 뒤에도 프로세스가 살아 있을 때 의미가 있습니다.
즉 "어떤 테스트가 오래 걸리는가"보다 "무엇이 프로세스를 붙잡고 있는가"에 가까운 문제입니다.

대표적인 원인은 다음과 같습니다.

- 닫지 않은 HTTP 서버나 TCP 서버
- 해제하지 않은 `setInterval()`
- 종료하지 않은 file watcher
- 반환하지 않은 DB, Redis, message queue 연결
- abort하지 않은 background polling 작업

따라서 timeout과 force exit는 경쟁 관계가 아닙니다.
timeout은 개별 테스트의 느림을 잡고, force exit는 전체 테스트 실행 뒤 남은 handle 때문에 CI가 멈추는 것을 막습니다.

## 열린 리소스를 진단하는 흐름

### H3. active resource 목록을 먼저 확인한다

force exit를 켰다면, 다음 작업은 남은 리소스의 종류를 확인하는 것입니다.
Node.js의 `process.getActiveResourcesInfo()`는 이벤트 루프를 붙잡는 활성 리소스 유형 목록을 반환합니다.
테스트가 끝나기 직전이나 after hook에서 이 값을 기록하면 어디부터 볼지 좁힐 수 있습니다.

```js
import test from 'node:test';

test.after(() => {
  const resources = process.getActiveResourcesInfo();

  if (resources.length > 0) {
    console.warn('active resources after tests:', resources);
  }
});
```

이 로그는 원인 해결을 위한 단서입니다.
다만 CI 로그에는 민감한 연결 문자열, 토큰, 사용자 데이터가 찍히지 않도록 리소스 유형 수준만 남기는 편이 좋습니다.
실제 URL이나 환경 변수 값을 그대로 출력하는 디버그 코드는 피해야 합니다.

### H3. 리소스별 정리 위치를 명확히 둔다

원인이 보이면 테스트마다 리소스 정리 위치를 고정합니다.
테스트 안에서 만든 리소스는 가능한 한 같은 테스트의 `t.after()`에서 닫습니다.
여러 테스트가 공유하는 리소스는 `before()`와 `after()` hook으로 수명을 한곳에 모읍니다.

```js
import { before, after, test } from 'node:test';
import assert from 'node:assert/strict';
import http from 'node:http';

let server;
let baseUrl;

before(async () => {
  server = http.createServer((req, res) => {
    res.end('ok');
  });

  await new Promise((resolve) => server.listen(0, resolve));
  baseUrl = `http://127.0.0.1:${server.address().port}`;
});

after(async () => {
  await new Promise((resolve, reject) => {
    server.close((error) => {
      if (error) reject(error);
      else resolve();
    });
  });
});

test('reads from shared server', async () => {
  const response = await fetch(baseUrl);

  assert.equal(response.status, 200);
});
```

정리 코드는 성공 경로뿐 아니라 실패 경로에서도 실행되어야 합니다.
그래서 본문 마지막에 `close()`를 직접 두는 방식보다 hook이나 `try/finally`가 더 안전합니다.

## CI에 적용하는 기준

### H3. 기본 명령과 CI 명령을 분리한다

모든 환경에서 force exit를 켜면 리소스 누수를 알아차리기 어려워집니다.
그래서 로컬 기본 명령은 열린 handle을 그대로 드러내고, CI 명령만 보호막을 두는 방식이 실무적으로 균형이 좋습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "node --test --test-force-exit --test-reporter=spec"
  }
}
```

CI에서도 force exit를 조용히 통과시키기보다 active resource 로그나 별도 진단 job을 붙이는 편이 좋습니다.
그렇게 해야 "일단 끝났으니 괜찮다"가 아니라 "파이프라인은 보호했고, 원인은 계속 추적한다"는 운영 기준이 남습니다.

### H3. force exit는 이슈와 함께 남긴다

`--test-force-exit`을 추가했다면 왜 필요한지 기록해야 합니다.
특정 테스트 그룹, 특정 외부 리소스, 특정 플랫폼 문제처럼 제거 조건을 함께 적어 두면 나중에 옵션을 걷어낼 수 있습니다.

```json
{
  "scripts": {
    "test:ci": "node --test --test-force-exit"
  }
}
```

위 script만 보면 의도를 알기 어렵습니다.
프로젝트 문서나 CI 설정 주석에는 다음처럼 제거 기준을 남기는 편이 좋습니다.

- 어떤 테스트가 열린 handle을 남기는지 추적 중인가?
- `process.getActiveResourcesInfo()` 로그에서 어떤 리소스가 반복되는가?
- 언제 force exit를 제거할 것인가?

force exit가 영구 설정이 되면 테스트 정리 품질이 낮아질 수 있습니다.
옵션을 추가하는 순간부터 제거 계획을 같이 두는 것이 중요합니다.

## 실무 체크리스트

### H3. force exit 적용 전후로 확인할 것

`--test-force-exit`은 테스트 운영을 편하게 만들지만, 잘못 쓰면 문제를 감춥니다.
적용 전후에는 다음 항목을 확인하세요.

- 개별 테스트 timeout으로 잡아야 할 느린 테스트가 아닌가?
- 서버, interval, watcher, DB 연결을 닫는 hook이 있는가?
- CI 로그에 활성 리소스 유형을 남길 수 있는가?
- force exit를 로컬 기본 script가 아니라 CI script에만 둘 것인가?
- 옵션 제거 기준이나 추적 이슈가 있는가?

테스트 프로세스가 멈추는 문제는 대개 작은 정리 누락에서 시작합니다.
force exit는 배포 흐름을 막지 않게 도와주는 장치일 뿐, 정리 누락을 고치는 마지막 단계는 아닙니다.

### H3. 민감정보 없는 진단 로그를 유지한다

종료 지연을 추적하다 보면 연결 정보, 환경 변수, 요청 payload를 출력하고 싶은 유혹이 생깁니다.
하지만 CI 로그는 장기간 보관되거나 외부 도구로 전송될 수 있습니다.
리소스 진단 로그에는 구체적인 값보다 유형과 개수만 남기는 편이 안전합니다.

```js
const counts = process.getActiveResourcesInfo().reduce((acc, type) => {
  acc[type] = (acc[type] ?? 0) + 1;
  return acc;
}, {});

console.warn('active resource counts after tests:', counts);
```

이 정도면 `Timeout`, `TCPServerWrap`, `FSReqCallback` 같은 단서를 얻으면서도 실제 접속 정보는 노출하지 않습니다.
문제가 좁혀진 뒤에는 해당 테스트 파일에서 재현하고, 필요한 값은 로컬 디버깅 환경에서만 확인하는 편이 좋습니다.

## 마무리

Node.js test runner의 `--test-force-exit`은 끝나지 않는 테스트 프로세스로 CI가 묶이는 상황을 줄여 주는 옵션입니다.
하지만 이 옵션은 열린 서버, interval, watcher, 외부 연결을 정리하지 않는 문제를 해결하지 않습니다.
따라서 로컬 기본 테스트에서는 문제를 드러내고, CI에서는 임시 보호막으로 쓰며, active resource 로그와 정리 hook으로 근본 원인을 줄여 가는 흐름이 좋습니다.

테스트가 모두 통과했는데도 프로세스가 끝나지 않는다면 먼저 남은 리소스를 확인하세요.
그다음 테스트 수명 주기에 맞춰 `t.after()`, `after()`, `try/finally`, AbortSignal cleanup을 배치하면 force exit 없이도 종료되는 테스트에 가까워집니다.

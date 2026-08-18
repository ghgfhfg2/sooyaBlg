---
layout: post
title: "Node.js server[Symbol.asyncDispose] 가이드: HTTP 서버 종료를 Promise로 다루는 방법"
date: 2026-08-18 20:00:00 +0900
lang: ko
translation_key: nodejs-server-asyncdispose-graceful-shutdown-guide
permalink: /development/blog/seo/2026/08/18/nodejs-server-asyncdispose-graceful-shutdown-guide.html
alternates:
  ko: /development/blog/seo/2026/08/18/nodejs-server-asyncdispose-graceful-shutdown-guide.html
  x_default: /development/blog/seo/2026/08/18/nodejs-server-asyncdispose-graceful-shutdown-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, http-server, asyncdispose, graceful-shutdown, promise, deployment, backend, javascript]
description: "Node.js server[Symbol.asyncDispose]()로 HTTP 서버 종료를 Promise 기반으로 다루는 방법을 정리합니다. server.close()와의 관계, await using 적용 기준, 테스트 서버 정리, 롤링 배포 종료 절차까지 실무 예제로 설명합니다."
---

Node.js에서 HTTP 서버를 종료하는 코드는 생각보다 자주 흐트러집니다.
`server.close()`는 오래된 표준 API라 대부분의 코드에서 익숙하지만, 콜백 기반이라 `async/await` 흐름에 끼워 넣을 때 래퍼를 직접 만들게 됩니다.
테스트에서는 서버를 띄운 뒤 닫는 코드를 빠뜨리기 쉽고, 배포 종료 처리에서는 신호 처리, readiness 차단, idle 연결 정리, 종료 예산까지 함께 고려해야 합니다.

`server[Symbol.asyncDispose]()`는 이런 종료 흐름을 Promise 기반으로 표현할 수 있게 해 주는 API입니다.
Node.js 공식 문서 기준으로 HTTP, HTTPS, HTTP/2, net 서버 계열에서 제공되며, HTTP/HTTPS 서버의 경우 `v24.2.0`에서 더 이상 실험적 기능이 아닌 것으로 표시되어 있습니다.
내부적으로는 `server.close()`를 호출하고, 서버가 닫히면 fulfillment되는 Promise를 반환합니다.

이 글에서는 `server[Symbol.asyncDispose]()`를 언제 쓰면 좋은지, `server.close()`와 어떻게 나눠 볼지, 테스트 서버와 운영 종료 코드에서 어떤 식으로 적용하면 안전한지 정리합니다.
연결 정리 단계는 [Node.js closeIdleConnections, closeAllConnections 가이드](/development/blog/seo/2026/04/16/nodejs-closeidleconnections-closeallconnections-graceful-drain-guide.html), 배포 종료 전체 흐름은 [Node.js graceful shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html), 명시적 리소스 관리 문법은 [Node.js using, asyncDisposable 가이드](/development/blog/seo/2026/05/02/nodejs-using-disposable-and-asyncdisposable-guide.html)와 함께 보면 좋습니다.

## server[Symbol.asyncDispose]가 해결하는 문제

### server.close는 콜백 기반이다

Node.js HTTP 서버를 종료하는 기본 API는 `server.close()`입니다.
새 연결을 더 받지 않고, 기존 연결이 정리되면 콜백을 호출합니다.
작은 예제에서는 충분히 단순합니다.

```js
import http from 'node:http';

const server = http.createServer((req, res) => {
  res.end('ok');
});

server.listen(3000);

process.once('SIGTERM', () => {
  server.close((error) => {
    if (error) {
      console.error('server close failed', error);
      process.exitCode = 1;
    }
  });
});
```

하지만 실제 애플리케이션에서는 종료 전에 해야 할 일이 더 많습니다.
readiness를 먼저 내리고, 큐 소비를 멈추고, 관측성 exporter를 flush하고, 종료 예산을 넘기면 더 강한 정리 절차를 실행해야 합니다.
이 흐름을 `async` 함수로 작성하면 콜백 API가 작은 마찰을 만듭니다.

```js
function closeServer(server) {
  return new Promise((resolve, reject) => {
    server.close((error) => {
      if (error) {
        reject(error);
        return;
      }

      resolve();
    });
  });
}
```

많은 코드베이스가 이런 래퍼를 직접 갖고 있습니다.
래퍼 자체가 나쁜 것은 아니지만, 서버 객체가 이미 Promise 기반 dispose 메서드를 제공한다면 같은 의도를 더 짧고 명확하게 표현할 수 있습니다.

### asyncDispose는 종료를 await 가능한 작업으로 만든다

`server[Symbol.asyncDispose]()`는 서버 종료를 Promise로 다룹니다.
따라서 종료 순서를 `await`로 자연스럽게 이어 쓸 수 있습니다.

```js
import http from 'node:http';

const server = http.createServer((req, res) => {
  res.end('ok');
});

server.listen(3000);

async function shutdown() {
  await server[Symbol.asyncDispose]();
}

process.once('SIGTERM', () => {
  shutdown().catch((error) => {
    console.error('shutdown failed', error);
    process.exitCode = 1;
  });
});
```

핵심은 이 API가 마법 같은 강제 종료 도구가 아니라는 점입니다.
공식 동작은 `server.close()`를 호출하고 닫힘을 기다리는 것입니다.
따라서 진행 중 요청을 어떻게 마무리할지, idle keep-alive 연결을 언제 닫을지, 종료 제한 시간을 어떻게 둘지는 여전히 애플리케이션이 결정해야 합니다.

## server.close와 asyncDispose의 역할 구분

### 콜백 생태계와 호환해야 하면 close가 자연스럽다

기존 코드가 콜백 기반 유틸리티로 구성되어 있거나, 지원해야 하는 Node.js 버전이 넓다면 `server.close()`를 유지하는 편이 현실적일 수 있습니다.
특히 라이브러리 코드라면 사용자의 런타임 버전을 함부로 좁히면 안 됩니다.

```js
export function closeHttpServer(server) {
  return new Promise((resolve, reject) => {
    server.close((error) => {
      if (error) {
        reject(error);
      } else {
        resolve();
      }
    });
  });
}
```

이 래퍼는 오래된 Node.js 버전에서도 의도가 명확합니다.
프로젝트의 최소 지원 버전이 `server[Symbol.asyncDispose]()`를 안정적으로 제공하지 않는다면, 이 방식이 더 예측 가능합니다.

### 애플리케이션 코드에서는 asyncDispose가 읽기 쉽다

반대로 런타임 버전을 통제할 수 있는 서비스 코드라면 `server[Symbol.asyncDispose]()`가 종료 흐름을 더 잘 드러냅니다.
서버 종료가 하나의 비동기 단계로 보이고, 다른 cleanup 작업과 같은 문법으로 묶입니다.

```js
async function stopApp({ server, jobConsumer, metrics }) {
  await jobConsumer.pause();
  await server[Symbol.asyncDispose]();
  await metrics.flush();
}
```

이 코드는 어떤 순서로 정리되는지 한눈에 보입니다.
`server.close()`를 Promise로 감싼 함수 이름을 프로젝트마다 다르게 만들 필요도 줄어듭니다.
팀에서 Node.js 버전 기준을 명확히 관리한다면 애플리케이션 진입점과 테스트 헬퍼에 먼저 적용해 볼 만합니다.

### 버전 차이는 기능 검사로 흡수할 수 있다

라이브러리나 공용 유틸리티에서는 기능 검사 fallback을 둘 수 있습니다.
서버 객체가 `Symbol.asyncDispose`를 지원하면 그것을 쓰고, 아니면 `server.close()`를 Promise로 감쌉니다.

```js
export function disposeServer(server) {
  if (typeof server?.[Symbol.asyncDispose] === 'function') {
    return server[Symbol.asyncDispose]();
  }

  return new Promise((resolve, reject) => {
    server.close((error) => {
      if (error) {
        reject(error);
      } else {
        resolve();
      }
    });
  });
}
```

이렇게 두면 호출부는 항상 `await disposeServer(server)`로 통일할 수 있습니다.
다만 운영 서버 코드에서는 fallback이 너무 많은 버전 차이를 숨기지 않게 해야 합니다.
배포 런타임의 Node.js 버전은 별도 문서나 CI에서 명시적으로 검증하는 편이 좋습니다.

## 테스트 서버 정리에 적용하기

### 테스트가 끝나면 서버를 반드시 닫는다

통합 테스트에서는 임시 HTTP 서버를 자주 띄웁니다.
테스트가 실패했을 때 서버를 닫지 못하면 프로세스가 끝나지 않거나, 다음 테스트에서 포트 충돌이 발생합니다.
`try/finally`와 `server[Symbol.asyncDispose]()`를 함께 쓰면 정리 경로를 짧게 유지할 수 있습니다.

```js
import assert from 'node:assert/strict';
import http from 'node:http';
import { test } from 'node:test';

function listen(server) {
  return new Promise((resolve) => {
    server.listen(0, () => resolve(server.address()));
  });
}

test('responds with health status', async () => {
  const server = http.createServer((req, res) => {
    res.end('ok');
  });

  const address = await listen(server);

  try {
    const response = await fetch(`http://127.0.0.1:${address.port}/health`);
    assert.equal(await response.text(), 'ok');
  } finally {
    await server[Symbol.asyncDispose]();
  }
});
```

`finally` 안에 `await`가 들어가면 테스트 실패 여부와 무관하게 서버 종료를 기다립니다.
테스트 러너 입장에서는 떠 있는 handle이 줄어들고, 개발자는 실패 원인을 포트 누수와 섞어 보지 않아도 됩니다.

### await using은 지원 범위를 확인한 뒤 쓴다

명시적 리소스 관리 문법을 사용할 수 있는 환경이라면 `await using`으로 서버 수명을 블록에 묶을 수도 있습니다.
이 문법은 코드가 매우 간결해지는 대신, 팀의 Node.js 버전과 트랜스파일 설정을 확인해야 합니다.

```js
import http from 'node:http';
import { test } from 'node:test';

test('serves temporary endpoint', async () => {
  await using server = http.createServer((req, res) => {
    res.end('temporary');
  });

  await new Promise((resolve) => {
    server.listen(0, resolve);
  });

  const { port } = server.address();
  const response = await fetch(`http://127.0.0.1:${port}/`);

  // 테스트 블록이 끝나면 server[Symbol.asyncDispose]()가 호출된다.
  if (!response.ok) {
    throw new Error(`unexpected status: ${response.status}`);
  }
});
```

모든 프로젝트에 이 문법을 바로 넣을 필요는 없습니다.
테스트 코드에서 먼저 적용해 보고, CI 런타임과 린터가 안정적으로 처리하는지 확인한 뒤 범위를 넓히는 편이 안전합니다.

## 운영 종료 흐름에 넣는 방법

### 신호 처리 함수는 한 번만 실행되게 만든다

운영 환경에서는 `SIGTERM`, `SIGINT`가 짧은 간격으로 여러 번 들어올 수 있습니다.
종료 함수가 중복 실행되면 서버 종료, 큐 정지, exporter flush가 서로 꼬일 수 있습니다.
먼저 idempotent한 shutdown 래퍼를 만듭니다.

```js
let shutdownPromise;

export function shutdownOnce(app) {
  if (!shutdownPromise) {
    shutdownPromise = shutdown(app);
  }

  return shutdownPromise;
}
```

이제 실제 종료 함수에서 `server[Symbol.asyncDispose]()`를 하나의 단계로 사용합니다.

```js
async function shutdown({ server, readiness, jobs, logger }) {
  readiness.markNotReady();
  logger.info('shutdown started');

  await jobs.pause();
  await server[Symbol.asyncDispose]();

  logger.info('shutdown completed');
}

process.once('SIGTERM', () => {
  shutdownOnce(app).catch((error) => {
    app.logger.error('shutdown failed', { message: error.message });
    process.exitCode = 1;
  });
});
```

여기서 `readiness.markNotReady()`는 새 트래픽이 더 들어오지 않게 하는 단계입니다.
쿠버네티스나 로드밸런서 환경에서는 서버를 닫기 전에 먼저 트래픽 대상에서 빠지는 시간이 필요합니다.
`asyncDispose`는 서버 객체를 닫는 단계이지, 트래픽 차단 전체를 대신하지 않습니다.

### 종료 예산을 별도로 둔다

`server[Symbol.asyncDispose]()`는 서버가 닫힐 때까지 기다립니다.
하지만 운영 종료에는 보통 제한 시간이 있습니다.
컨테이너가 `terminationGracePeriodSeconds` 안에 종료되지 않으면 강제 종료될 수 있고, 배포 시스템도 무한정 기다려 주지 않습니다.

```js
function timeoutAfter(ms) {
  return new Promise((_, reject) => {
    const timer = setTimeout(() => {
      reject(new Error(`shutdown timeout after ${ms}ms`));
    }, ms);

    timer.unref();
  });
}

async function shutdownWithBudget(app, budgetMs = 25_000) {
  await Promise.race([
    shutdown(app),
    timeoutAfter(budgetMs)
  ]);
}
```

종료 예산을 넘겼을 때 바로 `process.exit(1)`을 호출할지, `closeAllConnections()` 같은 강한 정리를 한 번 더 시도할지는 서비스 성격에 따라 달라집니다.
사용자 요청을 최대한 살려야 하는 API와, 빠른 교체가 중요한 내부 worker는 기준이 다를 수 있습니다.
중요한 것은 예산을 코드와 로그에 명시해 운영자가 종료 지연을 추적할 수 있게 만드는 것입니다.

### idle 연결 정리는 별도 단계로 남긴다

`server.close()` 계열은 새 요청 수락을 멈추고 기존 연결이 정리되기를 기다립니다.
keep-alive idle 연결이 오래 남는 서비스라면 종료가 예상보다 길어질 수 있습니다.
이때 `closeIdleConnections()`를 보조 단계로 검토할 수 있습니다.

```js
async function shutdownHttpServer(server) {
  const closing = server[Symbol.asyncDispose]();

  if (typeof server.closeIdleConnections === 'function') {
    server.closeIdleConnections();
  }

  await closing;
}
```

이 코드는 종료를 시작한 뒤 idle 연결 정리를 요청합니다.
진행 중 요청까지 끊을 수 있는 `closeAllConnections()`는 더 강한 수단이므로 기본 경로보다 종료 예산 초과 경로에 두는 편이 안전합니다.
사용자 요청을 끊는 동작은 장애 지표와 직접 연결되기 때문입니다.

## 로깅과 관측성 체크리스트

### 종료 시작과 완료를 모두 남긴다

서버 종료 코드는 평소에는 눈에 잘 띄지 않지만, 배포 장애가 생기면 가장 먼저 확인해야 하는 영역이 됩니다.
로그에는 최소한 종료 시작, 서버 close 완료, timeout 여부, 최종 exitCode를 남기는 편이 좋습니다.

```js
async function shutdown(app) {
  const startedAt = Date.now();

  app.logger.info('shutdown started', {
    reason: app.shutdownReason
  });

  await app.server[Symbol.asyncDispose]();

  app.logger.info('http server closed', {
    durationMs: Date.now() - startedAt
  });

  await app.metrics.flush();

  app.logger.info('shutdown finished', {
    durationMs: Date.now() - startedAt
  });
}
```

로그에 `process.env` 전체, 요청 헤더 전체, 인증 토큰, 쿠키 같은 값을 남기면 안 됩니다.
종료 진단에 필요한 값은 이유, 단계, 소요 시간, 남은 작업 수처럼 제한된 운영 정보입니다.

### 테스트에서는 떠 있는 handle을 실패로 본다

테스트 서버 정리의 목적은 테스트가 끝난 뒤 프로세스 상태가 깨끗하다는 점을 보장하는 것입니다.
CI에서 테스트가 가끔 멈춘다면 다음 항목을 먼저 확인합니다.

- 테스트가 실패해도 `finally`에서 서버를 닫는가
- `listen(0)`으로 포트 충돌을 피하는가
- fetch 응답 본문을 소비하거나 취소하는가
- 서버 종료 Promise를 `await`하는가
- 테스트 전역에 공유 서버를 만들었다면 suite teardown에서 닫는가

`server[Symbol.asyncDispose]()`는 마지막 두 항목을 단순하게 만듭니다.
하지만 응답 스트림을 방치하거나, 별도 interval을 만들고 정리하지 않는 문제까지 자동으로 해결하지는 않습니다.

## 적용 전 점검

### 최소 Node.js 버전을 먼저 확인한다

이 API를 운영 코드에 바로 넣기 전에 프로젝트가 실제로 실행되는 Node.js 버전을 확인해야 합니다.
로컬 `.nvmrc`, Dockerfile, CI 설정, 배포 런타임이 서로 다르면 로컬에서는 통과하고 운영에서 실패할 수 있습니다.

```bash
node -p "process.versions.node"
node -p "typeof require('node:http').createServer()[Symbol.asyncDispose]"
```

두 번째 명령이 `function`을 출력하면 현재 런타임에서 해당 메서드를 사용할 수 있습니다.
공용 패키지라면 `engines.node`와 테스트 매트릭스도 함께 맞춰야 합니다.

### close 래퍼를 한 번에 모두 바꾸지 않는다

이미 프로젝트에 `closeServer()`, `stopHttpServer()`, `shutdownServer()` 같은 래퍼가 있다면 전부 지우고 바꾸기보다 호출부의 역할을 먼저 나누는 편이 좋습니다.
테스트 헬퍼처럼 범위가 좁은 곳부터 바꾸면 회귀 위험이 작습니다.
운영 종료 코드는 readiness, job consumer, metrics, tracing exporter와 연결되어 있으므로 별도 검증이 필요합니다.

마이그레이션 순서는 다음처럼 잡을 수 있습니다.

1. 테스트 전용 임시 서버 정리에 적용한다.
2. 내부 애플리케이션 헬퍼에 `disposeServer()` fallback을 추가한다.
3. 운영 shutdown 함수에서 종료 단계 로그를 보강한다.
4. 배포 환경의 Node.js 버전과 종료 예산을 확인한다.
5. idle 연결 정리와 강제 종료 경로를 별도 테스트한다.

이 순서라면 문법 변경과 운영 동작 변경을 한 번에 섞지 않을 수 있습니다.

## FAQ

### server[Symbol.asyncDispose]는 server.close와 다른가요?

동작의 중심은 같습니다.
`server[Symbol.asyncDispose]()`는 `server.close()`를 호출하고, 서버가 닫히면 완료되는 Promise를 반환합니다.
차이는 콜백이 아니라 `await`로 종료 흐름을 작성할 수 있다는 점입니다.

### 이 API만 쓰면 graceful shutdown이 완성되나요?

아닙니다.
서버를 닫는 단계는 graceful shutdown의 일부입니다.
운영에서는 readiness 차단, 큐 소비 중지, 진행 중 작업 대기, idle 연결 정리, 종료 예산, 로그와 지표까지 함께 설계해야 합니다.

### await using을 바로 써도 되나요?

프로젝트의 Node.js 버전, 린터, 테스트 러너, 빌드 도구가 명시적 리소스 관리 문법을 안정적으로 지원한다면 사용할 수 있습니다.
그렇지 않다면 `try/finally` 안에서 `await server[Symbol.asyncDispose]()`를 호출하는 방식이 더 이식성 좋습니다.

### closeAllConnections도 같이 호출해야 하나요?

항상 같이 호출할 필요는 없습니다.
`closeAllConnections()`는 진행 중 연결까지 끊을 수 있는 강한 도구입니다.
기본 종료 경로에서는 `server[Symbol.asyncDispose]()`와 필요 시 `closeIdleConnections()`를 우선 검토하고, 종료 예산을 넘겼을 때의 예외 경로로 `closeAllConnections()`를 두는 편이 안전합니다.

## 마무리

`server[Symbol.asyncDispose]()`는 Node.js 서버 종료 코드를 더 현대적인 `async/await` 흐름으로 정리해 주는 작은 API입니다.
새로운 종료 전략을 대신 만들어 주지는 않지만, 테스트 서버 cleanup과 애플리케이션 shutdown 함수에서 콜백 래퍼를 줄이는 데 실용적입니다.

도입할 때는 최소 Node.js 버전, 종료 예산, readiness 차단 순서, keep-alive 연결 정리 기준을 함께 확인하세요.
서버 종료는 평소에는 조용하지만, 배포 장애 때는 서비스 신뢰도를 가르는 경로입니다.
작은 API라도 종료 로그와 테스트를 곁들이면 훨씬 다루기 쉬운 운영 코드가 됩니다.

## 관련 글

- [Node.js graceful shutdown 가이드: 진행 중 요청을 안전하게 비우는 법](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)
- [Node.js closeIdleConnections, closeAllConnections 가이드: 배포 종료 시 keep-alive 연결 정리하는 법](/development/blog/seo/2026/04/16/nodejs-closeidleconnections-closeallconnections-graceful-drain-guide.html)
- [Node.js using, asyncDisposable 가이드: 리소스 정리를 명시적으로 다루는 법](/development/blog/seo/2026/05/02/nodejs-using-disposable-and-asyncdisposable-guide.html)

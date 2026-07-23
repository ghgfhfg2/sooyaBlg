---
layout: post
title: "Node.js postMessageToThread 가이드: 부모-자식이 아닌 Worker끼리 안전하게 메시지 보내기"
date: 2026-07-23 20:00:00 +0900
lang: ko
translation_key: nodejs-worker-threads-postmessagetothread-cross-thread-guide
permalink: /development/blog/seo/2026/07/23/nodejs-worker-threads-postmessagetothread-cross-thread-guide.html
alternates:
  ko: /development/blog/seo/2026/07/23/nodejs-worker-threads-postmessagetothread-cross-thread-guide.html
  x_default: /development/blog/seo/2026/07/23/nodejs-worker-threads-postmessagetothread-cross-thread-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, worker-threads, postmessagetothread, concurrency, thread, messaging, backend]
description: "Node.js worker_threads의 postMessageToThread()로 부모-자식 관계가 아닌 Worker thread 사이에 메시지를 전달하는 방법을 정리합니다. threadId 관리, workerMessage 이벤트, timeout, 오류 처리, 종료 흐름까지 실무 예제로 설명합니다."
---

Node.js에서 `Worker`를 쓰다 보면 처음에는 main thread와 worker 사이의 단순한 요청-응답만 필요합니다.
이때는 `worker.postMessage()`와 `parentPort.postMessage()`만으로 충분합니다.
하지만 worker가 또 다른 worker를 만들거나, 작업자들 사이에 라우팅이 필요한 구조가 되면 부모-자식 관계만으로 메시지를 전달하기가 애매해집니다.

`node:worker_threads`의 `postMessageToThread()`는 thread ID를 기준으로 다른 thread에 값을 보내는 API입니다.
Node.js 공식 worker_threads 문서 기준으로 이 API는 대상 thread의 `workerMessage` 이벤트로 메시지를 전달하고, 직접적인 부모-자식 관계가 아닐 때 사용하는 용도에 맞습니다.
이 글에서는 `postMessageToThread()`를 언제 쓰고, `threadId`를 어떻게 관리하며, timeout과 오류 처리를 어떻게 붙여야 안전한지 정리합니다.
[Node.js worker_threads 성능 가이드](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html), [Worker environment data 가이드](/development/blog/seo/2026/05/19/nodejs-worker-threads-environment-data-guide.html), [BroadcastChannel worker 통신 가이드](/development/blog/seo/2026/05/30/nodejs-broadcastchannel-worker-communication-guide.html)와 함께 보면 Worker 기반 병렬 처리 설계를 단계적으로 정리할 수 있습니다.

## postMessageToThread가 필요한 상황

### H3. 부모-자식 채널만으로 메시지 경로가 길어질 때

기본 Worker 통신은 부모와 자식 사이에 잘 맞습니다.
main thread가 worker를 만들고, worker가 main thread로 결과를 돌려주는 구조라면 `worker.postMessage()`와 `parentPort.postMessage()`가 가장 단순합니다.

문제는 worker tree가 깊어질 때입니다.
예를 들어 main thread가 coordinator worker를 만들고, coordinator worker가 다시 parser worker와 renderer worker를 만든다고 가정해 보겠습니다.
renderer worker가 main thread에 상태를 알려야 하는데 직접 부모가 아니라면, 기존 방식에서는 중간 worker가 메시지를 계속 중계해야 합니다.

`postMessageToThread()`는 이런 중계 코드를 줄일 수 있습니다.
대상 `threadId`를 알고 있다면 현재 thread에서 다른 thread의 `workerMessage` 이벤트로 메시지를 보낼 수 있습니다.

```js
import process from 'node:process';
import {
  Worker,
  isMainThread,
  postMessageToThread,
  threadId,
  workerData
} from 'node:worker_threads';

if (isMainThread) {
  const worker = new Worker(new URL(import.meta.url), {
    workerData: { mainThreadId: threadId }
  });

  process.on('workerMessage', (message, sourceThreadId) => {
    console.log('message from worker', sourceThreadId, message);
  });

  worker.on('exit', (code) => {
    console.log('worker exited', code);
  });
} else {
  await postMessageToThread(workerData.mainThreadId, {
    type: 'worker.ready',
    from: threadId
  });
}
```

이 예제는 구조를 보여 주기 위한 최소 형태입니다.
실무에서는 thread ID를 아무 곳에나 흩뿌리지 않고, 등록과 해제를 관리하는 얇은 registry를 두는 편이 안전합니다.

### H3. 모든 Worker 통신을 대체하는 API는 아니다

`postMessageToThread()`를 Worker 통신의 기본값으로 두면 오히려 복잡해질 수 있습니다.
부모와 자식 사이의 명확한 요청-응답은 기존 `postMessage()`가 더 읽기 쉽습니다.
한 메시지를 여러 worker에게 알려야 한다면 `BroadcastChannel`이 더 자연스러운 경우도 있습니다.

선택 기준은 메시지의 방향입니다.

- 부모가 자식에게 작업을 배정한다: `worker.postMessage()`
- 자식이 부모에게 결과를 돌려준다: `parentPort.postMessage()`
- 여러 worker가 같은 이벤트를 구독한다: `BroadcastChannel`
- 직접 부모-자식 관계가 아닌 특정 thread 하나에 보낸다: `postMessageToThread()`

이 구분을 팀 규칙으로 남겨 두면 Worker 코드가 커져도 메시지 경로를 추적하기 쉽습니다.
특히 장애 상황에서는 "누가 누구에게 보냈는가"를 빠르게 찾는 것이 중요합니다.

## threadId를 안전하게 관리하기

### H3. threadId registry를 명시적으로 둔다

`postMessageToThread()`는 대상 thread ID를 알아야 합니다.
그래서 worker가 시작될 때 자신의 역할과 `threadId`를 coordinator에 등록하고, 종료될 때 제거하는 구조가 필요합니다.

```js
const workersByRole = new Map();

export function registerWorker(role, worker) {
  workersByRole.set(role, {
    threadId: worker.threadId,
    worker
  });

  worker.once('exit', () => {
    const current = workersByRole.get(role);

    if (current?.worker === worker) {
      workersByRole.delete(role);
    }
  });
}

export function getThreadIdByRole(role) {
  return workersByRole.get(role)?.threadId;
}
```

registry는 단순해야 합니다.
worker 객체 전체를 여기저기 넘기기보다 역할 이름과 thread ID를 좁게 관리하면 호출부가 덜 엉킵니다.
다만 thread ID는 프로세스 안에서만 의미가 있으므로 파일이나 데이터베이스에 장기 저장하지 않는 편이 좋습니다.

### H3. threadName은 운영자가 읽는 라벨로 쓴다

최근 Node.js worker 문서에는 `worker.threadName`도 소개되어 있습니다.
이 값은 사람이 읽는 식별자에 가깝고, 메시지 라우팅의 고유 키로 쓰기에는 `threadId`가 더 직접적입니다.
역할 이름, 로그 라벨, 진단 출력에는 thread name을 쓰고 실제 전송 대상은 thread ID로 고정하는 방식이 명확합니다.

```js
const parser = new Worker(new URL('./parser-worker.js', import.meta.url), {
  name: 'parser-worker'
});

registerWorker('parser', parser);

parser.on('online', () => {
  console.log({
    threadId: parser.threadId,
    threadName: parser.threadName
  });
});
```

이름은 디버깅을 돕지만, 중복될 수 있다는 전제로 다루는 편이 안전합니다.
라우팅 키와 표시용 라벨을 섞지 않으면 운영 로그와 코드가 더 단정해집니다.

## 메시지 수신과 응답 패턴

### H3. workerMessage 이벤트에서 source를 확인한다

`postMessageToThread()`로 보낸 값은 대상 thread의 `workerMessage` 이벤트에서 받을 수 있습니다.
이 이벤트는 메시지 값과 보낸 thread ID를 함께 다룰 수 있으므로, 응답을 돌려보낼 때 source를 기준으로 삼을 수 있습니다.

```js
import process from 'node:process';
import { postMessageToThread, threadId } from 'node:worker_threads';

process.on('workerMessage', async (message, sourceThreadId) => {
  if (message?.type !== 'render.request') {
    return;
  }

  try {
    const html = await renderDocument(message.payload);

    await postMessageToThread(sourceThreadId, {
      type: 'render.success',
      requestId: message.requestId,
      html,
      from: threadId
    });
  } catch (error) {
    await postMessageToThread(sourceThreadId, {
      type: 'render.failure',
      requestId: message.requestId,
      message: error instanceof Error ? error.message : 'Unknown render error',
      from: threadId
    });
  }
});
```

메시지에는 `type`과 `requestId`를 넣는 것이 좋습니다.
`type`은 수신자가 어떤 작업인지 구분하게 해 주고, `requestId`는 비동기 응답을 원래 요청과 연결하는 데 필요합니다.
큰 객체를 그대로 보내기보다 필요한 데이터만 작게 보내는 것도 중요합니다.

### H3. 메시지 shape를 검증한다

Worker끼리 주고받는 메시지도 신뢰 경계를 가져야 합니다.
같은 프로세스 안의 코드라 해도 버전이 어긋나거나 잘못된 호출부가 생기면 예상하지 못한 값이 들어올 수 있습니다.

```js
function isRenderRequest(value) {
  return Boolean(
    value &&
    value.type === 'render.request' &&
    typeof value.requestId === 'string' &&
    typeof value.payload === 'object'
  );
}

process.on('workerMessage', async (message, sourceThreadId) => {
  if (!isRenderRequest(message)) {
    await postMessageToThread(sourceThreadId, {
      type: 'message.rejected',
      reason: 'Invalid render request'
    }, undefined, 1_000);
    return;
  }

  await handleRenderRequest(message, sourceThreadId);
});
```

검증 실패를 조용히 무시할지, 거절 메시지를 보낼지는 서비스 성격에 따라 다릅니다.
운영 환경에서는 최소한 경고 로그나 카운터를 남겨 잘못된 메시지 흐름을 추적할 수 있게 하는 편이 좋습니다.

## timeout과 오류 처리

### H3. timeout 없이 기다리는 코드를 피한다

공식 문서 기준으로 `postMessageToThread()`는 메시지가 성공적으로 처리되면 fulfilled되는 Promise를 반환합니다.
대상 thread에 listener가 없거나, 처리 중 오류가 발생하거나, timeout이 지나면 오류가 날 수 있습니다.
그래서 운영 코드에서는 `timeout` 옵션을 명시적으로 두는 편이 좋습니다.

```js
import { postMessageToThread } from 'node:worker_threads';

export async function sendWorkerMessage(threadId, message) {
  try {
    await postMessageToThread(threadId, message, undefined, 2_000);

    return { ok: true };
  } catch (error) {
    return {
      ok: false,
      code: error?.code ?? 'UNKNOWN_WORKER_MESSAGE_ERROR',
      message: error instanceof Error ? error.message : 'Unknown worker messaging error'
    };
  }
}
```

timeout 값은 메시지 성격에 맞춰 정해야 합니다.
상태 알림이나 health check는 짧게, 큰 작업 결과 전달은 조금 더 길게 둘 수 있습니다.
다만 무한 대기는 장애를 늦게 드러내므로 기본값으로 두지 않는 편이 안전합니다.

### H3. 오류 코드를 운영 로그에 남긴다

`postMessageToThread()` 실패는 원인이 여러 가지일 수 있습니다.
대상 thread가 이미 종료됐을 수도 있고, 수신 listener가 없을 수도 있고, listener 안에서 예외가 났을 수도 있습니다.
에러 메시지만 남기면 나중에 집계하기 어렵기 때문에 `code`, 대상 thread ID, 메시지 type을 함께 남깁니다.

```js
async function notifyRenderer(rendererThreadId, payload) {
  const message = {
    type: 'render.invalidate',
    payload
  };

  const result = await sendWorkerMessage(rendererThreadId, message);

  if (!result.ok) {
    console.warn(JSON.stringify({
      level: 'warn',
      event: 'worker.message.failed',
      targetThreadId: rendererThreadId,
      messageType: message.type,
      errorCode: result.code
    }));
  }
}
```

로그에는 원본 payload를 그대로 넣지 않는 것이 좋습니다.
payload 안에 사용자 입력, 문서 내용, 내부 경로가 포함될 수 있기 때문입니다.
운영 로그에는 라우팅과 장애 분석에 필요한 최소 정보만 남기세요.

## 종료와 재시작 흐름

### H3. 종료 중인 worker에는 새 메시지를 보내지 않는다

Worker가 종료되는 중인데 registry에 thread ID가 남아 있으면 메시지 전송 실패가 늘어납니다.
`exit` 이벤트에서 registry를 지우는 것만으로도 도움이 되지만, 작업자 상태를 조금 더 명확히 관리하면 재시작 중 혼란을 줄일 수 있습니다.

```js
const workers = new Map();

export function markWorkerStopping(role) {
  const entry = workers.get(role);

  if (entry) {
    entry.state = 'stopping';
  }
}

export async function sendToRole(role, message) {
  const entry = workers.get(role);

  if (!entry || entry.state !== 'online') {
    return {
      ok: false,
      code: 'WORKER_NOT_AVAILABLE'
    };
  }

  return sendWorkerMessage(entry.threadId, message);
}
```

이렇게 하면 호출부는 thread ID의 존재 여부만 보는 대신, worker가 실제로 메시지를 받을 준비가 되었는지 확인할 수 있습니다.
재시작 전략이 있는 서비스라면 새 worker가 `online`이 된 뒤 registry를 교체하는 방식으로 메시지 손실을 줄일 수 있습니다.

### H3. 요청-응답은 correlation map으로 정리한다

`postMessageToThread()`의 Promise는 메시지가 대상에서 처리됐는지를 알려 주는 흐름에 가깝습니다.
비즈니스 응답을 별도 메시지로 돌려받는 구조라면 `requestId`를 키로 하는 correlation map을 둬야 합니다.

```js
const pendingRequests = new Map();

export function waitForWorkerResponse(requestId, timeoutMs) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      pendingRequests.delete(requestId);
      reject(new Error(`Worker response timed out: ${requestId}`));
    }, timeoutMs);

    pendingRequests.set(requestId, {
      resolve(value) {
        clearTimeout(timer);
        resolve(value);
      },
      reject(error) {
        clearTimeout(timer);
        reject(error);
      }
    });
  });
}

process.on('workerMessage', (message) => {
  if (message?.type !== 'render.success' && message?.type !== 'render.failure') {
    return;
  }

  const pending = pendingRequests.get(message.requestId);

  if (!pending) {
    return;
  }

  pendingRequests.delete(message.requestId);

  if (message.type === 'render.failure') {
    pending.reject(new Error(message.message));
    return;
  }

  pending.resolve(message);
});
```

요청-응답 map은 반드시 timeout과 정리를 포함해야 합니다.
응답이 오지 않는 요청을 계속 쌓아 두면 작은 메시징 기능이 메모리 누수의 시작점이 될 수 있습니다.

## 도입 전 체크리스트

### H3. 메시지 경로를 문서화한다

Worker 통신은 코드만 보면 방향이 잘 드러나지 않을 때가 많습니다.
특히 `postMessageToThread()`는 thread ID만 있으면 직접 메시지를 보낼 수 있으므로, 규칙 없이 쓰면 의존 관계가 숨어 버립니다.

도입 전에 아래 항목을 정리하세요.

- 어떤 역할의 worker가 어떤 역할의 worker에게 직접 메시지를 보낼 수 있는가?
- 부모-자식 `postMessage()`와 `postMessageToThread()`를 나누는 기준은 무엇인가?
- thread ID registry의 owner는 어디인가?
- timeout 기본값과 재시도 정책은 무엇인가?
- 잘못된 메시지 shape는 거절, 무시, 경고 중 무엇으로 처리하는가?

작은 프로젝트에서는 주석이나 README 한 단락으로도 충분합니다.
중요한 것은 메시지 경로가 암묵적인 호출 습관으로만 남지 않게 하는 것입니다.

### H3. 민감정보를 메시지와 로그에 싣지 않는다

Worker 메시지는 같은 프로세스 내부 통신이지만, 그렇다고 모든 데이터를 넣어도 되는 것은 아닙니다.
메시지가 디버그 로그, 오류 리포트, 테스트 snapshot에 남을 수 있기 때문입니다.
액세스 토큰, 세션 쿠키, 원본 개인정보, 내부 인증 헤더는 메시지 payload에 넣지 않는 편이 안전합니다.

전달해야 하는 값은 작업에 필요한 최소 데이터로 제한하세요.
로그에는 `requestId`, `messageType`, `sourceThreadId`, `targetThreadId`, `errorCode`처럼 라우팅과 진단에 필요한 값만 남기는 것이 좋습니다.
실제 사용자 데이터는 별도의 권한 경계가 있는 저장소에서 읽고, worker 메시지에는 참조 ID만 싣는 구조가 더 안전합니다.

## 관련 글

- [Node.js worker_threads 성능 가이드](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)
- [Worker environment data 가이드](/development/blog/seo/2026/05/19/nodejs-worker-threads-environment-data-guide.html)
- [BroadcastChannel worker 통신 가이드](/development/blog/seo/2026/05/30/nodejs-broadcastchannel-worker-communication-guide.html)

## 마무리

`postMessageToThread()`는 Worker 구조가 깊어졌을 때 특정 thread로 직접 메시지를 보내는 선택지를 제공합니다.
다만 이 편리함은 thread ID 관리, 메시지 shape 검증, timeout, 종료 상태 처리와 함께 써야 실무 코드에서 안정적으로 동작합니다.

부모-자식 통신은 기존 `postMessage()`로 단순하게 유지하고, 직접 관계가 아닌 특정 thread 간 메시지에만 `postMessageToThread()`를 쓰세요.
그 위에 registry와 관찰 가능한 로그를 얹으면 Worker 기반 병렬 처리에서도 메시지 경로를 잃지 않고 운영할 수 있습니다.

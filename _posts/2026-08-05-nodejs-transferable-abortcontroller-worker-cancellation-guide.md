---
layout: post
title: "Node.js transferable AbortController 가이드: Worker 작업 취소 신호를 안전하게 전달하는 법"
date: 2026-08-05 20:00:00 +0900
lang: ko
translation_key: nodejs-transferable-abortcontroller-worker-cancellation-guide
permalink: /development/blog/seo/2026/08/05/nodejs-transferable-abortcontroller-worker-cancellation-guide.html
alternates:
  ko: /development/blog/seo/2026/08/05/nodejs-transferable-abortcontroller-worker-cancellation-guide.html
  x_default: /development/blog/seo/2026/08/05/nodejs-transferable-abortcontroller-worker-cancellation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, abortcontroller, worker-threads, cancellation, backend, javascript, reliability, concurrency]
description: "Node.js transferable AbortController와 transferableAbortSignal로 Worker thread 작업에 취소 신호를 전달하는 방법을 정리합니다. timeout, 사용자 취소, 리소스 정리, 메시지 프로토콜 설계까지 실무 예제로 설명합니다."
---

Node.js에서 오래 걸리는 이미지 변환, 로그 분석, 압축, 리포트 생성 같은 작업을 Worker thread로 넘기면 메인 이벤트 루프를 보호할 수 있습니다.
하지만 작업을 시작하는 것만큼 중요한 문제가 하나 더 있습니다.
사용자가 요청을 취소했거나 timeout이 지났을 때, 이미 Worker로 넘어간 작업을 어떻게 멈출 것인지입니다.

메인 스레드 안에서는 `AbortController`와 `AbortSignal`을 비교적 쉽게 쓸 수 있습니다.
문제는 Worker thread 경계를 넘는 순간 일반 객체처럼 신호를 그대로 공유할 수 없다는 점입니다.
이 글에서는 Node.js의 `transferableAbortController()`와 `transferableAbortSignal()`을 사용해 Worker 작업 취소 신호를 안전하게 전달하는 실무 패턴을 정리합니다.

기본적인 취소 설계가 낯설다면 [Node.js AbortSignal.any, AbortSignal.timeout 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)를 먼저 보면 흐름을 잡기 쉽습니다.
Worker로 CPU 작업을 분리하는 기준은 [Node.js worker_threads 성능 가이드](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)와 연결됩니다.
파일 처리 취소와 정리는 [Node.js stream.pipeline AbortSignal 정리 가이드](/development/blog/seo/2026/07/24/nodejs-stream-pipeline-abortsignal-cleanup-guide.html)도 함께 참고할 만합니다.

## Worker 작업에도 취소 신호가 필요한 이유

### H3. timeout이 메인 스레드에서만 끝나면 실제 작업은 계속 돌 수 있다

API 요청에 timeout을 걸었다고 해서 Worker 안의 계산이 자동으로 멈추지는 않습니다.
메인 스레드가 "응답은 포기했다"고 결정해도, Worker는 여전히 CPU를 쓰거나 파일을 읽고 있을 수 있습니다.
트래픽이 많아지면 이런 잔여 작업이 쌓여서 큐 지연, 메모리 압박, 배포 종료 지연으로 이어질 수 있습니다.

```js
const timeoutSignal = AbortSignal.timeout(5000);

timeoutSignal.addEventListener('abort', () => {
  console.log('request timed out');
});
```

이 코드는 timeout을 감지할 뿐입니다.
이미 Worker로 넘긴 작업에 별도 취소 경로가 없다면, 실제 비용은 계속 발생합니다.
취소는 "응답을 그만 기다린다"와 "진행 중인 작업을 멈춘다"를 함께 설계해야 의미가 있습니다.

### H3. Worker 종료만으로 해결하면 작업 단위 복구가 거칠어진다

가장 단순한 방식은 timeout 때마다 Worker를 강제로 종료하는 것입니다.
하지만 이 방식은 현재 작업만 정리하는 것이 아니라 Worker의 모든 상태와 큐를 한 번에 버리는 선택이 됩니다.
짧은 일회성 Worker라면 괜찮을 수 있지만, Worker pool이나 장기 실행 Worker에서는 비용이 큽니다.

작업 단위로 취소할 수 있으면 아래 이점이 생깁니다.

- timeout이 난 작업만 중단한다.
- Worker 자체는 재사용할 수 있다.
- 취소 사유를 로그와 메트릭에 남길 수 있다.
- 부분 파일, 임시 디렉터리, 버퍼 같은 리소스를 정리할 시간을 줄 수 있다.

즉 Worker 종료는 마지막 수단이고, 기본값은 협력적 취소입니다.
`AbortSignal`은 이 협력적 취소를 표현하는 표준적인 방식입니다.

## transferable AbortController는 무엇을 해결하나

### H3. AbortSignal을 Worker로 보낼 수 있는 값으로 만든다

Worker thread는 `postMessage()`로 값을 주고받습니다.
일반 객체는 구조화 복제되지만, 모든 런타임 객체가 원하는 의미 그대로 복제되는 것은 아닙니다.
`AbortSignal`은 상태 변화가 중요하므로 단순 복사보다 "취소 상태가 전달되는 신호"로 다루는 편이 안전합니다.

Node.js는 `node:util`의 `transferableAbortController()`와 `transferableAbortSignal()`을 제공합니다.
이 도구를 사용하면 Worker로 전달 가능한 취소 신호를 만들 수 있습니다.

```js
import {
  transferableAbortController
} from 'node:util';

const controller = transferableAbortController();

worker.postMessage(
  {
    type: 'run-job',
    jobId,
    signal: controller.signal
  },
  [controller.signal]
);
```

메인 스레드는 `controller.abort()`를 호출하고, Worker는 전달받은 `signal.aborted`나 `abort` 이벤트를 확인합니다.
이렇게 하면 timeout, 사용자 취소, 서버 종료 같은 이벤트를 Worker 작업까지 같은 경로로 전파할 수 있습니다.

### H3. 이미 있는 AbortSignal을 전달해야 한다면 transferableAbortSignal을 쓴다

항상 새 controller를 만들 수 있는 것은 아닙니다.
요청 timeout, 사용자 취소, 서버 종료 신호를 `AbortSignal.any()`로 합친 뒤 Worker에 넘기고 싶을 수 있습니다.
이때는 `transferableAbortSignal()`을 사용해 기존 신호를 전달 가능한 형태로 만들 수 있습니다.

```js
import {
  transferableAbortSignal
} from 'node:util';

const requestSignal = AbortSignal.timeout(5000);
const shutdownSignal = getShutdownSignal();
const combined = AbortSignal.any([requestSignal, shutdownSignal]);
const signal = transferableAbortSignal(combined);

worker.postMessage(
  {
    type: 'run-job',
    jobId,
    signal
  },
  [signal]
);
```

실무에서는 이 방식이 자주 편합니다.
취소 원인은 메인 스레드에서 조합하고, Worker는 전달받은 신호만 일관되게 관찰하면 됩니다.
취소 정책과 작업 구현이 섞이지 않아 테스트도 쉬워집니다.

## Worker 쪽에서는 어떻게 취소를 처리할까

### H3. 작업 시작 전에 먼저 aborted 상태를 확인한다

메시지가 Worker에 도착하기 전에 이미 timeout이 발생했을 수 있습니다.
그래서 Worker는 이벤트 리스너를 붙이기 전에 `signal.aborted`를 먼저 확인해야 합니다.
이 작은 순서 차이가 레이스 컨디션을 줄입니다.

```js
import { parentPort } from 'node:worker_threads';

parentPort.on('message', async (message) => {
  if (message.type !== 'run-job') return;

  const { jobId, signal, input } = message;

  try {
    signal.throwIfAborted();

    const result = await runJob(input, { signal });

    parentPort.postMessage({
      type: 'job-complete',
      jobId,
      result
    });
  } catch (error) {
    parentPort.postMessage({
      type: 'job-failed',
      jobId,
      aborted: signal.aborted,
      message: signal.aborted ? 'job aborted' : error.message
    });
  }
});
```

`throwIfAborted()`를 초기에 호출하면 이미 취소된 작업을 비싼 처리로 넘기지 않을 수 있습니다.
이 패턴은 [Node.js AbortSignal.throwIfAborted 가이드](/development/blog/seo/2026/05/06/nodejs-abortsignal-throwifaborted-cancellation-checkpoint-guide.html)에서 다룬 체크포인트 방식과 같습니다.

### H3. 긴 루프에는 취소 체크포인트를 넣는다

`AbortSignal`은 코드를 자동으로 중단시키는 마법이 아닙니다.
작업 코드가 적절한 지점에서 신호를 확인해야 합니다.
특히 CPU 바운드 루프는 이벤트가 처리될 틈이 적을 수 있으므로, 일정 단위마다 취소 여부를 확인하는 구조가 필요합니다.

```js
async function runJob(items, { signal }) {
  const output = [];

  for (const item of items) {
    signal.throwIfAborted();

    output.push(expensiveTransform(item));

    if (output.length % 100 === 0) {
      await scheduler.yield();
    }
  }

  return output;
}
```

`scheduler.yield()`는 긴 작업 사이에 이벤트 루프가 다른 일을 처리할 기회를 줍니다.
이 방식은 [Node.js scheduler.yield 가이드](/development/blog/seo/2026/05/02/nodejs-scheduler-yield-event-loop-responsiveness-guide.html)와도 잘 맞습니다.
단, 체크포인트를 너무 촘촘하게 넣으면 처리량이 떨어질 수 있으므로 작업 단위와 지연 허용 범위에 맞춰 조정해야 합니다.

## 메시지 프로토콜은 어떻게 잡는 게 좋을까

### H3. jobId와 취소 사유를 항상 같이 남긴다

Worker 작업은 비동기로 완료되므로 어떤 응답이 어떤 요청의 결과인지 명확해야 합니다.
`jobId` 없이 성공과 실패 메시지만 오가면 timeout, 재시도, 취소가 섞였을 때 추적이 어려워집니다.

```js
function cancelJob(jobId, controller, reason) {
  controller.abort(reason);

  logger.info({
    jobId,
    reason: String(reason?.message ?? reason ?? 'aborted')
  }, 'worker job cancellation requested');
}
```

취소 사유에는 민감정보를 넣지 않는 편이 좋습니다.
사용자 입력 전체, 파일 전체 경로, 토큰이 포함된 URL 대신 `timeout`, `client_closed`, `shutdown`처럼 분류 가능한 값만 남기는 구조가 안전합니다.
로그 마스킹 원칙은 [Node.js 시스템 오류 코드 가이드](/development/blog/seo/2026/08/05/nodejs-system-error-code-handling-guide.html)의 방향과 같습니다.

### H3. 완료 후 늦게 도착한 결과는 버릴 수 있어야 한다

취소 신호를 보냈다고 해서 Worker가 즉시 멈춘다는 보장은 없습니다.
이미 마지막 단계까지 끝났거나, 취소 체크포인트가 늦게 있는 작업이라면 완료 메시지가 뒤늦게 도착할 수 있습니다.
메인 스레드는 취소된 job의 결과를 다시 응답이나 저장소에 반영하지 않도록 상태를 관리해야 합니다.

```js
const jobs = new Map();

worker.on('message', (message) => {
  if (message.type !== 'job-complete') return;

  const job = jobs.get(message.jobId);

  if (!job || job.signal.aborted) {
    return;
  }

  job.resolve(message.result);
  jobs.delete(message.jobId);
});
```

이 방어 로직은 idempotency와도 닮아 있습니다.
결과가 늦게 오거나 중복으로 오더라도 한 번만 반영하도록 설계해야 합니다.
요청 중복 처리 관점은 [Node.js Idempotency Key 가이드](/development/blog/seo/2026/03/27/nodejs-idempotency-key-api-duplicate-request-guide.html)와 함께 볼 수 있습니다.

## 운영에서 주의할 점

### H3. 취소가 곧 롤백은 아니다

취소 신호는 작업 중단 의도를 전달할 뿐, 이미 끝난 부작용을 자동으로 되돌리지 않습니다.
Worker가 임시 파일을 만들었거나 일부 데이터를 저장했거나 외부 API를 호출했다면, 별도 보상 로직이나 정리 절차가 필요합니다.

```js
async function runWithTempDir(input, { signal }) {
  const tempDir = await createTempDir();

  try {
    signal.throwIfAborted();
    return await convertFiles(input, tempDir, { signal });
  } finally {
    await cleanupTempDir(tempDir);
  }
}
```

`finally`에서 정리하는 구조는 단순하지만 효과적입니다.
임시 디렉터리 생성과 삭제가 많다면 [Node.js mkdtemp 임시 디렉터리 정리 가이드](/development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html)를 같이 점검하세요.

### H3. Worker pool에서는 취소와 재사용 기준을 분리한다

작업 하나가 취소됐다고 Worker를 반드시 버릴 필요는 없습니다.
반대로 취소 후에도 메모리 사용량이 비정상적으로 높거나, 네이티브 리소스 상태가 의심된다면 해당 Worker를 교체하는 편이 안전할 수 있습니다.

운영 기준은 아래처럼 나눌 수 있습니다.

- 정상적인 사용자 취소: 작업만 중단하고 Worker 재사용
- timeout 반복: 작업 크기, 체크포인트, pool 크기 점검
- 치명 오류: Worker 폐기 후 새 Worker 생성
- 메모리 압박: Worker 교체와 입력 크기 제한 검토

중요한 것은 `abort`와 `terminate`를 같은 의미로 쓰지 않는 것입니다.
`abort`는 협력적 취소이고, `terminate`는 실행 환경 자체를 버리는 결정입니다.

## 실무 체크리스트

### H3. 구현 전 체크

- Worker로 넘긴 작업이 timeout 뒤에도 계속 실행되는지 확인했는가
- 메인 스레드의 요청 취소, timeout, shutdown 신호를 하나의 `AbortSignal`로 묶었는가
- 전달 가능한 signal을 만들기 위해 `transferableAbortController()` 또는 `transferableAbortSignal()`을 사용했는가
- 메시지에 `jobId`를 포함해 결과와 취소를 추적할 수 있는가

### H3. 운영 전 체크

- Worker 코드가 시작 전과 긴 루프 중간에 `throwIfAborted()`를 호출하는가
- 취소된 job의 늦은 완료 메시지를 무시하는가
- 임시 파일, 버퍼, 외부 리소스를 `finally`에서 정리하는가
- 취소 사유 로그에 개인정보나 민감정보가 섞이지 않는가
- 반복 timeout이 발생할 때 Worker pool 크기와 작업 입력 크기를 조정할 기준이 있는가

## 자주 묻는 질문

### H3. Worker에 signal을 넘기면 작업이 자동으로 중단되나요?

아닙니다.
`AbortSignal`은 취소 상태를 전달합니다.
작업 코드가 `signal.throwIfAborted()`를 호출하거나 `abort` 이벤트를 구독해 직접 멈춰야 합니다.

### H3. timeout마다 worker.terminate()를 호출해도 되나요?

작은 일회성 Worker라면 가능할 수 있습니다.
하지만 Worker pool에서는 비용이 크고 다른 상태까지 잃을 수 있으므로, 먼저 작업 단위 취소를 설계한 뒤 치명 오류나 비정상 상태에서만 `terminate()`를 검토하는 편이 좋습니다.

### H3. transferableAbortController와 transferableAbortSignal 중 무엇을 쓰면 되나요?

새 취소 제어권을 만들고 Worker에 넘길 때는 `transferableAbortController()`가 단순합니다.
이미 합쳐진 요청 timeout이나 shutdown 신호를 전달하려면 `transferableAbortSignal()`이 더 자연스럽습니다.

## 마무리

Worker thread는 무거운 작업을 메인 이벤트 루프 밖으로 옮기는 좋은 도구입니다.
하지만 취소 전파를 설계하지 않으면 사용자가 떠난 뒤에도 비용이 계속 쌓일 수 있습니다.

`transferableAbortController()`와 `transferableAbortSignal()`을 사용하면 메인 스레드의 timeout, 사용자 취소, 종료 신호를 Worker 작업까지 일관되게 전달할 수 있습니다.
핵심은 Worker가 신호를 자주 확인하고, 늦게 도착한 결과를 무시하며, 취소 후 리소스를 정리하는 것입니다.

## 함께 읽기

- [Node.js AbortSignal.any, AbortSignal.timeout 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js worker_threads 성능 가이드](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)
- [Node.js stream.pipeline AbortSignal 정리 가이드](/development/blog/seo/2026/07/24/nodejs-stream-pipeline-abortsignal-cleanup-guide.html)

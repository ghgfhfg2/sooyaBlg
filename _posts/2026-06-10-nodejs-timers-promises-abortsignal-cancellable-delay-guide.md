---
layout: post
title: "Node.js timers/promises 가이드: AbortSignal로 취소 가능한 지연 만들기"
date: 2026-06-10 08:00:00 +0900
lang: ko
translation_key: nodejs-timers-promises-abortsignal-cancellable-delay-guide
permalink: /development/blog/seo/2026/06/10/nodejs-timers-promises-abortsignal-cancellable-delay-guide.html
alternates:
  ko: /development/blog/seo/2026/06/10/nodejs-timers-promises-abortsignal-cancellable-delay-guide.html
  x_default: /development/blog/seo/2026/06/10/nodejs-timers-promises-abortsignal-cancellable-delay-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, timers, promises, abortsignal, retry, gracefulshutdown, backend]
description: "Node.js timers/promises의 setTimeout()과 AbortSignal로 취소 가능한 지연을 만드는 방법을 정리합니다. 재시도 대기, 종료 신호 전파, 취소 에러 처리와 테스트 패턴을 예제로 설명합니다."
---

Node.js 애플리케이션에서는 재시도 전 대기, 요청 간격 조절, 주기 작업처럼 일정 시간 멈추는 코드가 자주 필요합니다.
단순한 `setTimeout()`도 시간을 기다리는 데는 충분하지만, 서버 종료나 요청 취소가 발생했을 때 남은 대기를 즉시 끝내려면 별도의 정리 코드가 필요합니다.

`node:timers/promises`의 `setTimeout()`은 Promise로 기다릴 수 있고 `AbortSignal`도 지원합니다.
덕분에 콜백을 직접 감싸지 않고도 취소 가능한 지연 함수를 만들고, 동일한 취소 신호를 네트워크 요청이나 파일 작업까지 전달할 수 있습니다.

이 글에서는 `timers/promises`의 기본 사용법, `AbortController`로 대기를 취소하는 방법, 재시도 로직과 graceful shutdown에 적용하는 패턴, 취소 에러 처리와 테스트 기준을 정리합니다.
이벤트 루프에 실행 기회를 양보하는 방법이 필요하다면 [Node.js scheduler.yield 가이드](/development/blog/seo/2026/05/20/nodejs-timers-promises-scheduler-yield-event-loop-guide.html)도 함께 참고하세요.

## 콜백 setTimeout과 timers/promises의 차이

### H3. Promise 기반 지연은 비동기 흐름을 단순하게 만든다

일반 `setTimeout()`은 콜백을 등록합니다.
`async` 함수 안에서 기다리려면 보통 Promise로 한 번 감싸야 합니다.

```js
function delay(ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
}

await delay(1_000);
```

작은 예제에서는 충분하지만, 취소 기능까지 직접 구현하면 타이머 핸들을 보관하고 `clearTimeout()`을 호출하며 Promise의 완료 상태도 관리해야 합니다.

Node.js 내장 `node:timers/promises`를 사용하면 같은 목적을 더 명확하게 표현할 수 있습니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

await delay(1_000);
```

이름을 `delay`로 바꾸어 가져오면 전역 `setTimeout()`과 구분하기 쉽습니다.
첫 번째 인자는 기다릴 밀리초, 두 번째 인자는 대기 후 반환할 값, 세 번째 인자는 옵션입니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

const result = await delay(500, { status: 'ready' });
console.log(result);
```

### H3. 지연과 이벤트 루프 양보는 목적이 다르다

지연 시간을 지정한 `setTimeout()`은 일정 시간이 지난 뒤 작업을 계속하는 도구입니다.
반면 `scheduler.yield()`는 특정 시간을 기다리기보다 다른 이벤트 루프 작업에 실행 기회를 주는 용도입니다.

재시도 간격이나 호출 속도 제한에는 `setTimeout()`이 맞고, 긴 반복 작업 중 응답성을 유지하려면 `scheduler.yield()`가 더 알맞을 수 있습니다.
두 동작을 같은 의미로 사용하면 불필요한 지연이나 과도한 CPU 사용이 생길 수 있습니다.

## AbortSignal로 지연 취소하기

### H3. AbortController의 signal을 옵션으로 전달한다

`timers/promises`의 `setTimeout()`은 옵션의 `signal`을 통해 취소 신호를 받습니다.
대기 중 `abort()`가 호출되면 Promise는 즉시 거부됩니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

const controller = new AbortController();

const pendingDelay = delay(30_000, undefined, {
  signal: controller.signal
});

setTimeout(() => {
  controller.abort();
}, 100);

try {
  await pendingDelay;
} catch (error) {
  if (error.name === 'AbortError') {
    console.log('대기가 취소되었습니다.');
  } else {
    throw error;
  }
}
```

30초 지연을 만들었지만 약 100밀리초 후 취소 신호를 보내므로 남은 시간을 기다리지 않습니다.
취소는 정상적인 제어 흐름일 수 있으므로, 다른 장애와 구분해서 처리해야 합니다.

### H3. 호출자가 취소 신호를 소유하게 한다

재사용 가능한 함수 안에서 매번 `AbortController`를 새로 만들면 상위 작업이 취소 시점을 제어하기 어렵습니다.
취소 가능한 함수는 `signal`을 인자로 받아 호출자가 생명주기를 관리하게 만드는 편이 좋습니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

export async function waitBeforeNextAttempt(ms, { signal } = {}) {
  await delay(ms, undefined, { signal });
}
```

호출자는 하나의 신호를 여러 단계에 전달할 수 있습니다.

```js
const controller = new AbortController();

await waitBeforeNextAttempt(1_000, {
  signal: controller.signal
});
```

이 구조는 지연 함수뿐 아니라 `fetch()`, 일부 파일 시스템 API, 이벤트 리스너 정리에도 같은 취소 정책을 적용하기 쉽습니다.
파일 읽기 취소 예제는 [Node.js fs.readFile AbortSignal 가이드](/development/blog/seo/2026/05/19/nodejs-fs-readfile-abortsignal-cancel-file-read-guide.html)에서 확인할 수 있습니다.

## 재시도 대기를 취소 가능하게 만들기

### H3. 실패 직후 반복하지 말고 간격을 둔다

외부 API나 데이터베이스 작업이 실패했을 때 즉시 반복하면 장애 중인 대상에 더 큰 부하를 줄 수 있습니다.
재시도 횟수를 제한하고, 시도 사이에 대기 시간을 두며, 상위 작업이 취소되면 즉시 중단해야 합니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

export async function retry(operation, {
  attempts = 3,
  delayMs = 500,
  signal
} = {}) {
  let lastError;

  for (let attempt = 1; attempt <= attempts; attempt += 1) {
    signal?.throwIfAborted();

    try {
      return await operation({ attempt, signal });
    } catch (error) {
      if (signal?.aborted) throw error;

      lastError = error;
      if (attempt === attempts) break;

      await delay(delayMs * attempt, undefined, { signal });
    }
  }

  throw lastError;
}
```

이 예제는 시도 횟수가 늘어날수록 대기 시간을 늘리는 단순한 선형 백오프를 사용합니다.
실제 서비스에서는 오류 종류가 재시도 가능한지 먼저 판별하고, 여러 인스턴스가 동시에 재시도하지 않도록 무작위 지연을 더하는 것도 고려해야 합니다.

### H3. 취소와 작업 실패를 같은 로그로 남기지 않는다

사용자 요청 취소나 서버 종료 때문에 재시도가 멈춘 경우를 일반 오류로 기록하면 경보와 장애 통계가 부정확해질 수 있습니다.
취소 여부는 별도로 분류하되, 원본 요청 본문이나 인증 토큰 같은 민감정보를 로그에 넣지 않아야 합니다.

```js
try {
  await retry(runJob, { signal: controller.signal });
} catch (error) {
  if (controller.signal.aborted) {
    logger.info('job cancelled', { jobType: 'daily-summary' });
  } else {
    logger.error('job failed', {
      jobType: 'daily-summary',
      errorName: error.name
    });
  }
}
```

외부 요청의 제한 시간과 재시도 정책을 함께 설계하려면 [Node.js fetch AbortSignal.timeout 재시도 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)를 참고하세요.

## graceful shutdown에 적용하기

### H3. 종료 신호를 모든 대기 작업에 전달한다

서버가 종료 신호를 받았는데 백그라운드 작업이 긴 재시도 대기 중이라면 프로세스 종료가 늦어질 수 있습니다.
공용 `AbortController`를 만들고 종료 시 취소하면 남아 있는 대기와 작업을 한 번에 중단할 수 있습니다.

```js
const shutdownController = new AbortController();

function beginShutdown(signalName) {
  console.log(`shutdown requested: ${signalName}`);
  shutdownController.abort(new Error('application shutdown'));
}

process.once('SIGTERM', () => beginShutdown('SIGTERM'));
process.once('SIGINT', () => beginShutdown('SIGINT'));

await runBackgroundLoop({
  signal: shutdownController.signal
});
```

`abort()`에 이유를 전달할 수 있지만, 외부 입력이나 민감정보를 이유 문자열에 포함하지 않는 편이 안전합니다.
또한 취소 신호만 보낸 뒤 바로 프로세스를 강제 종료하기보다, 서버 연결과 데이터 저장 작업이 정리될 시간을 제한적으로 제공해야 합니다.

### H3. 타이머만 취소한다고 종료가 완성되지는 않는다

취소 가능한 지연은 종료 시간을 줄이는 한 요소입니다.
열린 HTTP 서버, 소켓, 파일 감시자, 워커, 메시지 포트도 각각 정리해야 합니다.

종료 처리에서는 다음 순서를 명확하게 정하는 것이 좋습니다.

1. 새 요청과 새 작업 수신을 중단한다.
2. 공용 취소 신호를 전달해 대기와 재시도를 멈춘다.
3. 진행 중인 작업이 제한 시간 안에 정리되도록 기다린다.
4. 데이터베이스 연결, 서버, 워커 등 소유한 리소스를 닫는다.
5. 남은 활성 리소스를 확인하고 종료한다.

프로세스를 붙잡는 리소스 유형을 확인하는 방법은 [process.getActiveResourcesInfo 종료 지연 진단 가이드](/development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html)와 연결됩니다.

## 테스트할 때 확인할 항목

### H3. 정상 완료와 취소 경로를 각각 검증한다

취소 가능한 함수는 정상적으로 시간이 지난 경우와 대기 중 취소된 경우를 모두 테스트해야 합니다.
테스트 시간이 길어지지 않도록 지연 시간은 짧게 두고, 에러 이름만 확인하기보다 실제로 예상한 취소 신호에서 발생했는지도 확인합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { setTimeout as delay } from 'node:timers/promises';

test('delay resolves with the provided value', async () => {
  const value = await delay(5, 'done');
  assert.equal(value, 'done');
});

test('delay rejects when aborted', async () => {
  const controller = new AbortController();
  const pending = delay(10_000, undefined, {
    signal: controller.signal
  });

  controller.abort();

  await assert.rejects(pending, {
    name: 'AbortError'
  });
});
```

재시도 함수라면 최대 시도 횟수, 재시도하지 않아야 하는 오류, 대기 중 취소, 작업 실행 중 취소를 별도 사례로 검증하는 것이 좋습니다.

## 실무 체크리스트

- `node:timers/promises`의 `setTimeout()`을 `delay` 같은 이름으로 가져왔는가?
- 취소가 필요한 대기에는 호출자가 제공한 `AbortSignal`을 전달했는가?
- 재시도 횟수와 최대 대기 시간을 제한했는가?
- 취소를 일반 장애와 구분해 처리하는가?
- 취소 로그에 요청 본문, 토큰, 개인정보를 남기지 않는가?
- 종료 시 타이머뿐 아니라 서버, 소켓, 워커도 정리하는가?
- 정상 완료와 취소 경로를 모두 테스트했는가?

## FAQ

### H3. 전역 setTimeout 대신 timers/promises를 항상 써야 하나요?

아닙니다.
단순 콜백 예약에는 전역 `setTimeout()`이 자연스럽고, `async/await` 흐름에서 결과를 기다리거나 `AbortSignal`로 취소해야 할 때 `timers/promises`가 편리합니다.

### H3. abort하면 타이머 Promise는 정상 완료되나요?

아닙니다.
대기 중 신호가 취소되면 Promise는 일반적으로 `AbortError`로 거부됩니다.
따라서 호출부에서 취소를 예상된 흐름으로 처리할지, 상위로 전달할지 정책을 정해야 합니다.

### H3. AbortSignal.timeout과 timers/promises setTimeout은 같은 기능인가요?

목적이 다릅니다.
`AbortSignal.timeout(ms)`는 일정 시간이 지나면 취소되는 신호를 만들고, `timers/promises`의 `setTimeout(ms)`는 일정 시간을 기다리는 Promise를 만듭니다.
작업의 제한 시간을 정할 때는 전자를, 재시도 간격처럼 의도적으로 기다릴 때는 후자를 사용하는 편이 명확합니다.

## 마무리

`node:timers/promises`의 `setTimeout()`은 지연을 Promise 흐름으로 표현하고 `AbortSignal`로 취소할 수 있게 해 줍니다.
재시도와 백그라운드 작업에서 상위 취소 신호를 전달하면 서버 종료나 요청 취소 시 불필요한 대기를 즉시 멈출 수 있습니다.

다만 취소 가능한 타이머만으로 안정적인 종료나 재시도 정책이 완성되지는 않습니다.
재시도 가능한 오류 구분, 횟수와 시간 제한, 민감정보 없는 로그, 다른 활성 리소스 정리, 취소 경로 테스트까지 함께 설계해야 운영에서 예측 가능한 동작을 만들 수 있습니다.

---
layout: post
title: "Node.js timer unref 가이드: 프로세스 종료를 붙잡지 않는 타이머 설계"
date: 2026-06-21 20:00:00 +0900
lang: ko
translation_key: nodejs-timer-unref-hasref-exit-control-guide
permalink: /development/blog/seo/2026/06/21/nodejs-timer-unref-hasref-exit-control-guide.html
alternates:
  ko: /development/blog/seo/2026/06/21/nodejs-timer-unref-hasref-exit-control-guide.html
  x_default: /development/blog/seo/2026/06/21/nodejs-timer-unref-hasref-exit-control-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, timer, unref, hasref, settimeout, setinterval, gracefulshutdown, backend]
description: "Node.js timer의 unref(), ref(), hasRef(), refresh()를 사용해 진단용 타이머와 백그라운드 작업이 프로세스 종료를 불필요하게 붙잡지 않도록 설계하는 방법을 정리합니다."
---

Node.js 서비스가 요청 처리를 모두 끝냈는데도 프로세스가 종료되지 않는 경우가 있습니다.
원인은 열려 있는 서버 소켓일 수도 있지만, 의외로 단순한 `setTimeout()`이나 `setInterval()`이 이벤트 루프를 붙잡고 있는 경우도 많습니다.
특히 진단용 로그, 캐시 정리, health check, shutdown fallback처럼 보조 목적의 타이머는 서비스의 생명주기를 방해하지 않아야 합니다.

이때 사용할 수 있는 도구가 타이머 핸들의 `unref()`입니다.
[Node.js 공식 문서](https://nodejs.org/api/timers.html)에 따르면 `Timeout`과 `Immediate` 객체는 기본적으로 이벤트 루프를 활성 상태로 유지하며, `unref()`를 호출하면 그 객체 하나만으로는 프로세스가 계속 실행되지 않습니다.
이 글에서는 `unref()`, `ref()`, `hasRef()`, `refresh()`를 실무에서 어떻게 구분해 쓰는지 정리합니다.

종료 지연 원인을 먼저 찾고 싶다면 [Node.js process.getActiveResourcesInfo 가이드](/development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html)를 함께 보면 좋습니다.
취소 가능한 지연은 [Node.js timers/promises AbortSignal 가이드](/development/blog/seo/2026/06/10/nodejs-timers-promises-abortsignal-cancellable-delay-guide.html)와 연결됩니다.
전체 종료 흐름은 [Node.js graceful shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)를 참고하세요.

## timer.unref가 필요한 이유

### H3. 기본 타이머는 이벤트 루프를 유지한다

Node.js의 일반 타이머는 기본적으로 프로세스 종료를 막을 수 있습니다.
아래 코드는 실행할 일이 없어 보여도 60초 타이머가 남아 있으므로 그 전까지 프로세스가 살아 있을 수 있습니다.

```js
setTimeout(() => {
  console.log('delayed cleanup');
}, 60_000);
```

업무 처리에 꼭 필요한 타이머라면 이 동작이 맞습니다.
예를 들어 결제 상태 확인, 큐 재시도, 사용자에게 반환해야 하는 응답과 연결된 timeout은 남은 작업으로 취급해야 합니다.

반대로 진단 로그나 보조 정리 작업처럼 "프로세스가 살아 있으면 실행하고, 종료할 상황이면 생략해도 되는" 타이머는 다릅니다.
이런 타이머가 종료를 붙잡으면 배포, 테스트, 서버리스 실행 환경에서 불필요한 지연이 생깁니다.

### H3. unref는 타이머를 취소하는 API가 아니다

`unref()`는 타이머를 지우지 않습니다.
타이머가 실행될 시간이 오고 프로세스가 아직 살아 있다면 콜백은 실행됩니다.
차이는 그 타이머 하나 때문에 프로세스를 계속 붙잡지는 않는다는 점입니다.

```js
const timeout = setTimeout(() => {
  logger.info('background diagnostic snapshot');
}, 30_000);

timeout.unref();
```

이 패턴은 "best effort" 성격의 작업에 어울립니다.
진단 스냅샷이 남으면 좋지만, 다른 모든 작업이 끝났다면 그 로그 하나를 위해 프로세스를 30초 더 유지할 필요는 없습니다.

## unref를 적용하기 좋은 타이머

### H3. shutdown fallback 타이머

graceful shutdown을 구현할 때는 일정 시간 안에 정리가 끝나지 않으면 강제로 종료하는 fallback 타이머를 두는 경우가 많습니다.
이 타이머는 종료 과정을 보호하기 위한 안전장치지만, 정상적으로 정리가 끝난 뒤에는 프로세스를 붙잡으면 안 됩니다.

```js
export function installShutdownTimeout({ timeoutMs = 10_000 } = {}) {
  const forceExitTimer = setTimeout(() => {
    logger.error({ timeoutMs }, 'shutdown timeout exceeded');
    process.exit(1);
  }, timeoutMs);

  forceExitTimer.unref();

  return () => {
    clearTimeout(forceExitTimer);
  };
}
```

핵심은 두 가지입니다.
첫째, `unref()`로 fallback 타이머가 정상 종료를 방해하지 않게 합니다.
둘째, shutdown이 정상 완료되면 `clearTimeout()`으로 명시적으로 정리합니다.
`unref()`를 호출했다고 해서 cleanup이 필요 없어지는 것은 아닙니다.

### H3. 관측성 보조 타이머

운영 코드에는 주기적으로 상태를 요약하는 타이머가 많습니다.
예를 들어 이벤트 루프 지연, 메모리 사용량, 큐 길이, 활성 리소스 개수를 로그로 남길 수 있습니다.

```js
export function startRuntimeSnapshotLogger() {
  const interval = setInterval(() => {
    logger.info({
      resources: process.getActiveResourcesInfo()
    }, 'runtime snapshot');
  }, 60_000);

  interval.unref();

  return () => {
    clearInterval(interval);
  };
}
```

이 interval은 프로세스가 계속 실행되는 동안에는 유용합니다.
하지만 서버가 이미 닫히고 처리할 요청도 없다면, 관측성 로그를 남기기 위해 프로세스를 계속 유지하는 것은 보통 이득이 없습니다.
그래서 보조 관측 작업에는 `unref()`를 기본값으로 두는 편이 실무적으로 안전합니다.

### H3. CLI의 안내 메시지와 느린 작업 경고

CLI 도구에서는 작업이 오래 걸릴 때 안내 메시지를 지연 출력하는 타이머를 둘 수 있습니다.
작업이 빨리 끝나면 타이머를 지우고, 작업이 오래 걸리면 사용자에게 상태를 알려 주는 방식입니다.

```js
export async function runWithSlowNotice(task) {
  const noticeTimer = setTimeout(() => {
    console.error('Still working. This may take a little longer.');
  }, 5_000);

  noticeTimer.unref();

  try {
    return await task();
  } finally {
    clearTimeout(noticeTimer);
  }
}
```

이 안내 메시지는 있으면 좋지만 필수 결과는 아닙니다.
작업이 이미 끝났다면 메시지를 출력할 이유가 없고, CLI 종료를 늦출 이유도 없습니다.

## unref를 쓰면 안 되는 경우

### H3. 업무 정확성과 연결된 타이머

모든 타이머에 `unref()`를 붙이는 것은 위험합니다.
타이머 콜백이 실제 업무 결과를 만들거나 데이터 정합성을 보장한다면 프로세스가 그 작업을 기다려야 할 수 있습니다.

```js
const retryTimer = setTimeout(async () => {
  await retryPaymentConfirmation(orderId);
}, 30_000);
```

이런 타이머를 무심코 `unref()`하면 프로세스가 먼저 종료되어 재시도 자체가 실행되지 않을 수 있습니다.
중요한 재시도는 인메모리 타이머보다 큐, durable job, 외부 스케줄러로 옮기는 편이 더 안전합니다.
`unref()`는 중요한 작업을 가볍게 만드는 도구가 아니라, 중요하지 않은 타이머가 생명주기를 방해하지 않게 하는 도구입니다.

### H3. 테스트 실패를 숨기는 용도로 쓰면 안 된다

테스트가 끝나지 않는다고 모든 타이머에 `unref()`를 붙이면 원인을 놓치기 쉽습니다.
테스트가 멈춘다는 것은 대개 정리되지 않은 서버, interval, watcher, socket이 있다는 신호입니다.

```js
test('starts worker', async (t) => {
  const stopWorker = startWorker();

  t.after(() => {
    stopWorker();
  });
});
```

테스트에서는 먼저 생성한 리소스를 명확히 정리해야 합니다.
`unref()`는 진단용 fallback처럼 테스트 종료를 붙잡지 않아야 하는 보조 타이머에만 제한적으로 적용하는 편이 좋습니다.

## hasRef와 ref로 상태를 명확히 다루기

### H3. hasRef로 현재 동작을 확인한다

타이머가 이벤트 루프를 붙잡는 상태인지 확인하려면 `hasRef()`를 사용할 수 있습니다.
이 값은 테스트와 진단 로그에서 의도를 검증할 때 유용합니다.

```js
const timer = setTimeout(sendSlowOperationWarning, 10_000);

console.log(timer.hasRef()); // true

timer.unref();

console.log(timer.hasRef()); // false
```

운영 코드에서 `hasRef()`를 자주 분기 조건으로 쓰는 일은 많지 않습니다.
대신 "이 모듈의 진단 타이머는 unref 상태여야 한다"처럼 의도를 문서화하고 테스트하는 데 적합합니다.

```js
import assert from 'node:assert/strict';

function createSnapshotInterval() {
  const interval = setInterval(writeSnapshot, 60_000);
  interval.unref();
  return interval;
}

const interval = createSnapshotInterval();

try {
  assert.equal(interval.hasRef(), false);
} finally {
  clearInterval(interval);
}
```

실제 모듈 API에서 내부 interval을 노출할지는 신중히 정해야 합니다.
테스트 편의를 위해 내부 구현을 그대로 공개하기보다, 필요한 경우 진단용 getter나 주입 가능한 timer factory를 두는 편이 더 깔끔할 수 있습니다.

### H3. ref는 다시 기다려야 하는 작업에만 사용한다

`ref()`는 `unref()`한 타이머를 다시 이벤트 루프 유지 대상으로 되돌립니다.
흔한 사용처는 많지 않지만, 작업 상태가 바뀌면서 보조 타이머가 필수 타이머로 바뀌는 경우에는 명시적으로 사용할 수 있습니다.

```js
const timeout = setTimeout(onDeadlineExceeded, deadlineMs);
timeout.unref();

function markAsCritical() {
  timeout.ref();
}
```

다만 이런 코드는 읽는 사람이 의도를 놓치기 쉽습니다.
처음부터 중요한 deadline이라면 `unref()`를 하지 않는 편이 더 명확합니다.
`ref()`와 `unref()`를 여러 곳에서 번갈아 호출해야 한다면 타이머의 책임이 너무 넓어진 것은 아닌지 먼저 확인해야 합니다.

## refresh와 함께 쓰는 패턴

### H3. 활동이 있을 때 deadline을 연장한다

`refresh()`는 기존 타이머 객체를 유지한 채 시작 시간을 갱신합니다.
같은 timeout을 반복해서 새로 만들지 않고 idle timeout을 구현할 때 유용합니다.

```js
export function createIdleWarningTimer({ idleMs, onIdle }) {
  const timer = setTimeout(onIdle, idleMs);
  timer.unref();

  return {
    touch() {
      timer.refresh();
    },
    stop() {
      clearTimeout(timer);
    }
  };
}
```

이 예제에서 idle 경고는 보조 신호입니다.
프로세스가 계속 살아 있고 활동이 없다면 경고를 남기지만, 다른 작업이 모두 끝났다면 이 타이머가 종료를 막지 않습니다.

### H3. refresh는 clearTimeout을 대체하지 않는다

`refresh()`는 타이머를 다시 예약하는 API에 가깝습니다.
타이머가 더 이상 필요 없다면 여전히 `clearTimeout()`이나 `clearInterval()`로 정리해야 합니다.

```js
const idleTimer = createIdleWarningTimer({
  idleMs: 30_000,
  onIdle: () => logger.warn('worker is idle')
});

worker.on('job', () => {
  idleTimer.touch();
});

worker.on('close', () => {
  idleTimer.stop();
});
```

작업 수명과 타이머 수명을 같은 이벤트에 묶어 두면 종료 지연과 메모리 누수를 줄일 수 있습니다.
`unref()`는 마지막 방어선이고, 기본은 필요 없는 타이머를 확실히 정리하는 것입니다.

## timers/promises에서 ref 옵션 쓰기

### H3. Promise 기반 delay도 ref 정책을 가진다

`node:timers/promises`의 `setTimeout()`을 사용할 때도 `ref` 옵션을 지정할 수 있습니다.
기본값은 일반 타이머처럼 프로세스를 붙잡는 동작입니다.
보조 대기라면 `ref: false`를 명시해 의도를 드러낼 수 있습니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

await delay(30_000, undefined, {
  ref: false
});
```

다만 `await` 중인 함수가 중요한 작업이라면 `ref: false`가 적절하지 않을 수 있습니다.
프로세스가 종료되어도 괜찮은 대기인지, 아니면 반드시 완료되어야 하는 업무 흐름인지 먼저 구분해야 합니다.

### H3. AbortSignal과 ref false를 함께 사용한다

보조 delay라도 종료 신호를 받으면 즉시 취소되는 편이 좋습니다.
`AbortSignal`과 `ref: false`를 함께 사용하면 타이머가 생명주기를 덜 방해하면서도 명시적으로 취소할 수 있습니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

export async function waitBeforeOptionalSnapshot(signal) {
  try {
    await delay(10_000, undefined, {
      signal,
      ref: false
    });

    await writeRuntimeSnapshot();
  } catch (error) {
    if (error.name !== 'AbortError') {
      throw error;
    }
  }
}
```

이 패턴은 shutdown 중 선택적 진단 파일을 남기거나, 백그라운드 상태 점검을 늦춰 실행할 때 사용할 수 있습니다.
취소 에러를 삼킬지 다시 던질지는 작업의 중요도에 따라 정해야 합니다.

## 운영 체크리스트

### H3. 타이머를 만들 때 목적을 먼저 분류한다

타이머를 추가할 때는 구현보다 먼저 목적을 정리하는 편이 좋습니다.

- 업무 결과를 보장하는 타이머인가?
- 진단, 로그, 안내처럼 보조 목적의 타이머인가?
- 프로세스 종료 시 생략되어도 되는가?
- shutdown 또는 테스트 cleanup에서 명시적으로 정리되는가?
- `AbortSignal`이나 stop 함수로 수명을 제어할 수 있는가?

이 질문에 답하면 `unref()`를 붙일지, 큐나 스케줄러로 옮길지, cleanup 함수를 반환할지 결정하기 쉬워집니다.

### H3. 종료 지연은 활성 리소스와 함께 확인한다

종료가 느려졌다면 `unref()`를 추가하기 전에 어떤 리소스가 남았는지 확인해야 합니다.
`process.getActiveResourcesInfo()`로 `Timeout`이 남는지 보고, shutdown 단계별로 스냅샷을 남기면 조사 범위를 줄일 수 있습니다.

```js
function logActiveResources(stage) {
  logger.info({
    stage,
    resources: process.getActiveResourcesInfo()
  }, 'active resources');
}
```

타이머가 남아 있다는 사실만으로 바로 `unref()`가 정답은 아닙니다.
정리해야 할 타이머인지, 종료를 붙잡지 않아야 할 타이머인지 분류한 뒤 수정해야 합니다.

## 자주 묻는 질문

### H3. unref를 호출하면 콜백이 절대 실행되지 않나요?

아닙니다.
프로세스가 다른 작업 때문에 계속 살아 있다면 시간이 지난 뒤 콜백은 실행될 수 있습니다.
다만 그 타이머 하나만 남았을 때는 프로세스가 종료될 수 있습니다.

### H3. setInterval에도 unref를 써도 되나요?

쓸 수 있습니다.
진단 로그나 보조 상태 점검처럼 프로세스 종료를 붙잡으면 안 되는 interval에는 유용합니다.
하지만 interval이 중요한 업무 처리라면 `unref()`보다 명시적인 stop 함수와 shutdown 연결을 먼저 설계해야 합니다.

### H3. unref와 clearTimeout은 무엇이 다른가요?

`unref()`는 타이머가 이벤트 루프를 유지하지 않게 만들 뿐이고, 타이머 자체는 남아 있습니다.
`clearTimeout()`은 예약된 타이머를 취소합니다.
따라서 종료를 방해하지 않게 하려면 `unref()`, 더 이상 필요 없는 타이머를 없애려면 `clearTimeout()`을 사용합니다.

## 마무리

Node.js의 `timer.unref()`는 종료 지연을 덮어버리는 편법이 아니라, 보조 타이머의 생명주기 의도를 명확히 표현하는 API입니다.
중요한 업무 타이머는 프로세스가 기다리게 두고, 진단이나 안내처럼 생략 가능한 타이머는 `unref()` 또는 `ref: false`로 종료를 방해하지 않게 설계하는 것이 좋습니다.

정리하면 타이머를 만들 때마다 "이 작업이 프로세스를 계속 살려 둘 만큼 중요한가"를 먼저 물어야 합니다.
그 답이 아니라면 `unref()`, 명시적인 cleanup, `AbortSignal`을 조합해 종료가 예측 가능한 Node.js 서비스를 만들 수 있습니다.

## 함께 읽기

- [Node.js process.getActiveResourcesInfo 가이드: 프로세스 종료 지연 원인 찾기](/development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html)
- [Node.js timers/promises 가이드: AbortSignal로 취소 가능한 지연 만들기](/development/blog/seo/2026/06/10/nodejs-timers-promises-abortsignal-cancellable-delay-guide.html)
- [Node.js graceful shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)

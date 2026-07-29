---
layout: post
title: "Node.js process.finalization 가이드: 종료 시점 리소스 정리를 안전하게 다루는 법"
date: 2026-07-29 20:00:00 +0900
lang: ko
translation_key: nodejs-process-finalization-register-cleanup-guide
permalink: /development/blog/seo/2026/07/29/nodejs-process-finalization-register-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/07/29/nodejs-process-finalization-register-cleanup-guide.html
  x_default: /development/blog/seo/2026/07/29/nodejs-process-finalization-register-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, process, finalization, cleanup, resource-management, shutdown, reliability, javascript]
description: "Node.js process.finalization.register, registerBeforeExit, unregister로 종료 시점 리소스 정리를 보조하는 방법을 정리합니다. 콜백 보장 한계, 클로저 누수, 명시적 dispose 패턴, 테스트 기준까지 실무 예제로 설명합니다."
---

리소스 정리는 대부분 명시적으로 끝내는 것이 가장 안전합니다.
파일 핸들, 임시 디렉터리, 네트워크 연결, 네이티브 리소스처럼 수명이 있는 값은 `close()`, `dispose()`, `finally` 블록으로 닫는 흐름이 먼저입니다.
하지만 프로세스 종료 시점에 마지막 방어선을 두고 싶을 때가 있습니다.

Node.js에는 `process.finalization` API가 있습니다.
`process.finalization.register()`와 `registerBeforeExit()`는 추적 중인 객체가 아직 가비지 컬렉션되지 않은 상태로 프로세스 종료 단계에 들어갈 때 정리 콜백을 실행하도록 등록합니다.
공식 문서 기준 이 API는 Node.js v22.5.0에서 추가됐고, 현재 안정도는 Active Development로 표시됩니다.
따라서 핵심 정리 경로로 삼기보다는 명시적 정리를 보조하는 안전장치로 이해하는 편이 좋습니다.

이 글에서는 `process.finalization`을 언제 검토할지, 콜백을 어떻게 작성해야 누수를 줄일 수 있는지, `unregister()`를 어디에 넣어야 하는지 정리합니다.
프로세스 종료 정책 자체는 [Node.js unhandledRejection/uncaughtException 종료 가이드](/development/blog/seo/2026/04/15/nodejs-unhandledrejection-uncaughtexception-shutdown-guide.html)를 먼저 보고, 열린 핸들 진단은 [Node.js getActiveResourcesInfo 가이드](/development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html)와 함께 보면 좋습니다.

## process.finalization이 필요한 상황

### H3. 명시적 dispose의 마지막 보조 장치다

`process.finalization`은 "정리를 잊어도 무조건 해결해 주는 기능"이 아닙니다.
종료 이벤트와 가비지 컬렉션 상태에 기대는 API이므로 호출되지 않는 상황이 있을 수 있습니다.
예를 들어 강제 종료, 치명적 크래시, 운영체제 수준의 kill, 일부 비정상 종료에서는 콜백을 기대하면 안 됩니다.

그래서 실무 기준은 단순합니다.
정상 경로에서는 직접 닫고, `process.finalization`은 놓친 정리를 줄이는 보조 장치로 둡니다.
리소스가 데이터 손실, 결제 중복, 잠금 파일 방치처럼 중요한 영향을 만들 수 있다면 이 API만으로는 부족합니다.

적합한 대상은 아래에 가깝습니다.

- 작은 임시 캐시 객체의 동기 정리
- 네이티브 애드온 래퍼의 가벼운 release 호출
- 테스트나 CLI에서 남은 핸들 방지를 돕는 보조 cleanup
- 명시적 `dispose()` 호출 누락을 줄이기 위한 방어 로직
- 메트릭, 디버그 로그, 상태 플래그처럼 실패해도 치명적이지 않은 정리

반대로 데이터베이스 트랜잭션 확정, 결제 취소, 큐 ack, 파일 내구성 보장 같은 작업은 명시적 성공/실패 경로에서 처리해야 합니다.
중요 파일 쓰기는 [Node.js writeFile flush 가이드](/development/blog/seo/2026/07/29/nodejs-fspromises-writefile-flush-durable-write-guide.html)처럼 쓰기 자체의 보장 범위를 따로 설계하는 편이 맞습니다.

### H3. exit와 beforeExit의 차이를 알아야 한다

`finalization.register(ref, callback)`은 프로세스가 `exit` 이벤트를 낼 때, 등록된 `ref` 객체가 아직 살아 있으면 콜백을 호출하는 방식입니다.
`exit` 단계에서는 비동기 작업을 새로 기다릴 수 없습니다.
따라서 콜백은 동기적이고 짧아야 합니다.

`finalization.registerBeforeExit(ref, callback)`은 `beforeExit` 이벤트 쪽에 연결됩니다.
`beforeExit`은 이벤트 루프가 비었을 때 발생하고, 비동기 작업을 추가하면 프로세스가 조금 더 살아 있을 수 있습니다.
다만 이것도 모든 종료 상황에서 호출되는 것은 아닙니다.
명시적 `process.exit()`, uncaught exception, 시그널 종료 같은 흐름에서는 기대와 다르게 동작할 수 있습니다.

운영 코드에서는 보통 아래처럼 나눕니다.

- `register`: 동기 정리만 필요한 마지막 방어선
- `registerBeforeExit`: 자연 종료 직전 점검이나 가벼운 비동기 여지가 필요한 보조 흐름
- 직접 `dispose()`: 실제 정리의 기본 경로
- `try/finally`: 요청, 작업, 테스트 단위의 가장 신뢰 가능한 정리 경로

## 기본 사용법

### H3. 전역 함수 콜백으로 등록한다

공식 문서는 `finalization.register()`에 넘기는 함수가 불필요한 객체를 클로저로 붙잡지 않도록 주의하라고 안내합니다.
특히 화살표 함수는 바깥 컨텍스트를 잡기 쉬워서, 정리하려는 객체가 오히려 가비지 컬렉션되지 못하는 구조를 만들 수 있습니다.

아래처럼 정리 콜백은 루트 스코프의 일반 함수로 두는 편이 안전합니다.

```js
import { finalization } from 'node:process';

function finalizeTempHandle(handle, event) {
  handle.disposeSync({
    reason: `process_${event}`
  });
}

export function createTempHandle(path) {
  const handle = {
    path,
    closed: false,
    disposeSync({ reason } = {}) {
      if (this.closed) {
        return;
      }

      this.closed = true;
      // 실제 서비스에서는 여기에서 동기 close/release만 수행한다.
      console.info('temp_handle_disposed', { reason });
    }
  };

  finalization.register(handle, finalizeTempHandle);
  return handle;
}
```

이 예제의 핵심은 콜백이 `handle`을 외부에서 캡처하지 않는다는 점입니다.
콜백은 Node.js가 넘겨주는 `ref` 인자를 사용합니다.
이렇게 해야 등록 자체가 객체 수명을 불필요하게 늘리지 않습니다.

### H3. 직접 정리했다면 unregister를 호출한다

리소스를 정상적으로 닫았다면 `finalization.unregister(ref)`로 등록을 해제합니다.
그렇지 않으면 이미 닫은 리소스에 대해 종료 시점 콜백이 다시 호출될 수 있습니다.
정리 함수 자체를 idempotent하게 만드는 것도 중요하지만, 등록 해제까지 해 두면 의도가 더 명확합니다.

```js
import { finalization } from 'node:process';

function finalizeClient(client, event) {
  client.closeSync({ reason: `process_${event}` });
}

export function createClient(connection) {
  const client = {
    connection,
    closed: false,
    closeSync({ reason } = {}) {
      if (this.closed) {
        return;
      }

      this.closed = true;
      this.connection.close();
      console.info('client_closed', { reason });
    },
    close() {
      this.closeSync({ reason: 'explicit_close' });
      finalization.unregister(this);
    }
  };

  finalization.register(client, finalizeClient);
  return client;
}
```

중요한 점은 `close()`가 두 가지 일을 함께 한다는 것입니다.
먼저 리소스를 닫고, 그다음 finalization 등록을 제거합니다.
테스트에서는 `close()`를 호출한 뒤 종료 콜백이 중복 호출되지 않는지 확인하면 됩니다.

## 클로저 누수를 피하는 설계

### H3. this를 캡처하는 화살표 콜백을 피한다

가장 흔한 실수는 클래스 생성자 안에서 화살표 함수로 등록하는 방식입니다.
겉으로는 짧고 읽기 좋아 보이지만, 바깥 `this`와 주변 값을 붙잡을 수 있습니다.
정리 대상 객체가 콜백 때문에 계속 살아 있으면 finalization의 장점이 사라집니다.

피해야 할 예시는 아래와 같습니다.

```js
import { finalization } from 'node:process';

class LeakySession {
  constructor(socket) {
    this.socket = socket;
    finalization.register(this, () => this.dispose());
  }

  dispose() {
    this.socket.close();
  }
}
```

대신 정리 함수는 클래스 바깥에 두고, 콜백 인자로 전달되는 객체만 사용합니다.

```js
import { finalization } from 'node:process';

function finalizeSession(session, event) {
  session.disposeSync({ reason: `process_${event}` });
}

export class Session {
  constructor(socket) {
    this.socket = socket;
    this.closed = false;
    finalization.register(this, finalizeSession);
  }

  disposeSync({ reason } = {}) {
    if (this.closed) {
      return;
    }

    this.closed = true;
    this.socket.close();
    console.info('session_disposed', { reason });
  }

  close() {
    this.disposeSync({ reason: 'explicit_close' });
    finalization.unregister(this);
  }
}
```

이 구조는 반복되는 리소스 클래스에도 적용하기 쉽습니다.
콜백은 객체를 받기만 하고, 클래스 인스턴스는 명시적으로 닫을 수 있으며, 정상 정리 후에는 등록을 해제합니다.

### H3. 콜백에서 많은 일을 하지 않는다

finalization 콜백은 정리의 "마지막 기회"에 가깝습니다.
여기에서 네트워크 요청, 긴 파일 쓰기, 복잡한 로깅, 재시도 루프를 넣으면 종료 흐름이 불안정해집니다.
특히 `exit` 이벤트에서는 비동기 작업이 완료될 시간을 기대할 수 없습니다.

콜백에서는 아래 정도만 허용하는 기준을 추천합니다.

- 이미 메모리에 있는 핸들의 동기 close
- idempotent한 release 호출
- 짧은 동기 상태 변경
- 민감정보를 제외한 최소 로그
- 실패해도 앱 정합성을 깨지 않는 정리

에러도 콜백 안에서 삼키거나 최소한으로 기록해야 합니다.
종료 중 정리 실패가 다시 예외를 던져 종료 경로를 더 복잡하게 만들면, 원래 장애 원인을 가리기 쉽습니다.

```js
function finalizeResource(resource, event) {
  try {
    resource.disposeSync({ reason: `process_${event}` });
  } catch (error) {
    console.warn('resource_finalization_failed', {
      code: error?.code,
      name: error?.name
    });
  }
}
```

로그에는 토큰, 연결 문자열, 전체 파일 경로, 사용자 입력 원문을 넣지 않습니다.
종료 시점 로그는 사고 분석에 남을 가능성이 높기 때문에 평소보다 더 보수적으로 줄이는 편이 좋습니다.

## 명시적 정리와 함께 쓰기

### H3. try/finally가 기본 경로다

작업 단위가 명확하다면 `try/finally`가 가장 읽기 쉽고 안전합니다.
`process.finalization`은 이 흐름을 대체하지 않습니다.

```js
export async function runJob(input) {
  const client = createClient(input.connection);

  try {
    await client.send(input.payload);
    await client.flush();
  } finally {
    client.close();
  }
}
```

이 코드에서 정상 정리는 `finally`에서 수행됩니다.
`client.close()` 내부에서 `finalization.unregister(this)`를 호출하므로 종료 시점 중복 정리도 피할 수 있습니다.
테스트도 단순합니다.
성공, 실패, 중간 throw 상황에서 `close()`가 한 번 호출되는지만 보면 됩니다.

### H3. Symbol.dispose와도 역할을 나눌 수 있다

최근 Node.js와 JavaScript 생태계에서는 명시적 리소스 관리 패턴으로 `Symbol.dispose`와 `Symbol.asyncDispose`를 검토하는 경우가 늘고 있습니다.
이 패턴은 코드 블록의 수명과 리소스 수명을 맞추는 데 유용합니다.
관련 내용은 [Node.js using disposable 가이드](/development/blog/seo/2026/05/02/nodejs-using-disposable-and-asyncdisposable-guide.html)를 함께 참고하면 좋습니다.

역할을 나누면 아래와 같습니다.

- `Symbol.dispose`: 스코프를 벗어날 때 명시적으로 정리
- `try/finally`: 런타임과 문법 지원 범위가 넓은 기본 정리
- `process.finalization`: 프로세스 종료 시점의 보조 정리
- `AbortSignal`: 실행 중 취소와 정리 타이밍 전달

취소 가능한 작업은 [Node.js AbortSignal.throwIfAborted 가이드](/development/blog/seo/2026/05/06/nodejs-abortsignal-throwifaborted-cancellation-checkpoint-guide.html)처럼 중간 체크포인트를 두고, 마지막 cleanup만 finalization에 기대지 않도록 나눠야 합니다.

## 운영 점검 기준

### H3. 열린 핸들을 먼저 관찰한다

프로세스가 종료되지 않는 문제가 있다면 finalization을 추가하기 전에 열린 리소스를 확인해야 합니다.
`process.getActiveResourcesInfo()`는 이벤트 루프를 붙잡고 있는 리소스 종류를 파악하는 데 도움이 됩니다.

```js
import { getActiveResourcesInfo } from 'node:process';

export function logActiveResources(logger = console) {
  logger.info({
    event: 'active_resources_snapshot',
    resources: getActiveResourcesInfo().sort()
  });
}
```

이 값은 리소스의 이름이나 개별 객체를 완벽히 알려 주지는 않습니다.
하지만 `Timeout`, `TCPServerWrap`, `FSReqCallback` 같은 종류가 반복적으로 남는지 확인하면 어디를 더 봐야 하는지 감을 잡을 수 있습니다.
종료 지연 원인을 찾는 단계와 finalization 적용 단계는 분리하는 편이 안전합니다.

### H3. 테스트에서는 정상 정리 경로를 검증한다

finalization 콜백이 호출되는지에만 테스트를 걸면 환경에 따라 흔들릴 수 있습니다.
대신 애플리케이션 코드의 정상 정리 경로를 테스트하고, finalization 등록은 작은 단위로 확인합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('client.close disposes once and unregisters finalization', () => {
  let closeCount = 0;
  const client = createClient({
    close() {
      closeCount += 1;
    }
  });

  client.close();
  client.close();

  assert.equal(closeCount, 1);
});
```

종료 시그널까지 포함한 통합 테스트가 필요하다면 별도 child process를 띄워 stdout, stderr, exit code를 확인하는 방식이 더 안정적입니다.
테스트 러너의 종료와 hanging process 점검은 [Node.js test runner force-exit 가이드](/development/blog/seo/2026/07/15/nodejs-test-runner-force-exit-hanging-process-guide.html)를 연결해 보면 좋습니다.

## 적용 체크리스트

### H3. 아래 항목을 만족할 때만 적용한다

`process.finalization`은 유용하지만 신중하게 써야 하는 API입니다.
아래 기준을 통과하는 경우에만 코드에 넣는 것을 추천합니다.

- 리소스에 명시적 `close()` 또는 `dispose()` 경로가 있다.
- 정리 함수가 idempotent하게 작성되어 있다.
- 정상 정리 후 `finalization.unregister(ref)`를 호출한다.
- finalization 콜백은 루트 스코프의 일반 함수다.
- 콜백이 외부 객체나 `this`를 불필요하게 캡처하지 않는다.
- 콜백은 동기적이고 짧다.
- 콜백 실패가 데이터 정합성을 깨지 않는다.
- Node.js 지원 버전과 API 안정도 변화를 추적할 수 있다.

이 중 앞의 세 가지가 없다면 finalization을 붙이기 전에 리소스 수명 설계를 먼저 고치는 편이 좋습니다.

### H3. API 안정도 변화를 릴리스 노트에서 확인한다

`process.finalization`은 현재 Active Development로 분류됩니다.
Node.js 버전 업그레이드 과정에서 동작이나 권장 패턴이 조정될 수 있습니다.
서비스 런타임이 v22 이상이라고 해도 공개 라이브러리나 CLI에 바로 강하게 의존하기 전에는 지원 버전 범위와 fallback 전략을 확인해야 합니다.

실무에서는 아래처럼 관리할 수 있습니다.

- 앱 코드: Node.js 런타임 버전을 고정하고 작은 리소스부터 적용
- 라이브러리: optional helper로 제공하고 문서에 지원 버전 명시
- 테스트: 정리 함수의 idempotency와 unregister 흐름 중심으로 검증
- 운영: 종료 지연 진단과 finalization 적용 효과를 분리해서 기록

참고 자료:

- [Node.js Process API 공식 문서](https://nodejs.org/api/process.html)

## FAQ

### H3. process.finalization만 쓰면 close를 안 해도 되나요?

아니요.
`close()`나 `dispose()` 같은 명시적 정리가 기본입니다.
`process.finalization`은 종료 시점에 남은 객체를 보조적으로 정리하는 장치로 보는 편이 안전합니다.

### H3. 콜백에서 async 함수를 써도 되나요?

권장하지 않습니다.
특히 `exit` 단계에서는 비동기 작업 완료를 기대할 수 없습니다.
짧은 동기 정리만 넣고, 비동기 정리는 정상 실행 경로의 `finally`나 shutdown handler에서 처리하세요.

### H3. 화살표 함수를 절대 쓰면 안 되나요?

핵심은 정리 대상 객체와 주변 컨텍스트를 불필요하게 캡처하지 않는 것입니다.
하지만 화살표 함수는 바깥 스코프를 잡기 쉬우므로 finalization 콜백에는 루트 스코프의 일반 함수를 쓰는 기준이 더 안전합니다.

### H3. 언제 unregister를 호출해야 하나요?

리소스를 정상적으로 닫은 직후 호출합니다.
`close()` 또는 `dispose()` 메서드 안에 `finalization.unregister(this)`를 함께 넣으면 중복 정리를 피하고 호출 위치도 명확해집니다.

## 마무리

`process.finalization`은 Node.js 프로세스 종료 시점에 리소스 정리를 보조할 수 있는 API입니다.
하지만 보장된 cleanup 메커니즘으로 오해하면 위험합니다.
명시적 `dispose()`, `try/finally`, idempotent 정리 함수, `unregister()` 호출이 먼저이고, finalization은 그 위에 얹는 마지막 방어선입니다.

운영 코드에 적용할 때는 콜백을 짧고 동기적으로 유지하세요.
화살표 함수로 `this`를 캡처하지 말고, 정상 정리 경로를 테스트하세요.
그렇게 쓰면 `process.finalization.register()`는 종료 시점 리소스 누수를 줄이는 실용적인 보조 도구가 될 수 있습니다.

## 내부 링크

- [Node.js unhandledRejection/uncaughtException 종료 가이드](/development/blog/seo/2026/04/15/nodejs-unhandledrejection-uncaughtexception-shutdown-guide.html)
- [Node.js getActiveResourcesInfo 가이드](/development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html)
- [Node.js using disposable 가이드](/development/blog/seo/2026/05/02/nodejs-using-disposable-and-asyncdisposable-guide.html)

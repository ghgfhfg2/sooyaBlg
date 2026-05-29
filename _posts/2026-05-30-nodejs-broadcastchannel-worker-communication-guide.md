---
layout: post
title: "Node.js BroadcastChannel 가이드: 워커 간 상태 신호를 단순하게 전달하는 법"
date: 2026-05-30 08:00:00 +0900
lang: ko
translation_key: nodejs-broadcastchannel-worker-communication-guide
permalink: /development/blog/seo/2026/05/30/nodejs-broadcastchannel-worker-communication-guide.html
alternates:
  ko: /development/blog/seo/2026/05/30/nodejs-broadcastchannel-worker-communication-guide.html
  x_default: /development/blog/seo/2026/05/30/nodejs-broadcastchannel-worker-communication-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, broadcastchannel, worker-threads, concurrency, backend]
description: "Node.js BroadcastChannel로 여러 워커 스레드에 상태 변경 신호를 전달하는 방법을 정리했습니다. 기본 사용법, 메시지 설계, close와 unref 처리, 운영 주의점까지 예제로 설명합니다."
---

Node.js에서 워커 스레드를 쓰다 보면 “모든 워커에게 같은 신호를 알려야 하는” 순간이 생깁니다.
예를 들어 설정이 갱신됐거나, 캐시를 비워야 하거나, 배치 작업을 멈춰야 하는 경우입니다.

워커마다 `parentPort`로 개별 메시지를 보내도 되지만, 워커 수가 늘어나면 전달 코드가 금방 복잡해집니다.
이럴 때 `BroadcastChannel`은 단순한 1대다 신호 전달 도구로 쓸 수 있습니다.

핵심은 **데이터 공유가 아니라 상태 변경 이벤트를 넓게 알리는 통로로 쓰는 것**입니다.
이 글에서는 Node.js `BroadcastChannel` 기본 사용법, 워커 간 메시지 설계, 정리 누락을 줄이는 `close()`와 `unref()` 기준, 운영 체크리스트를 정리합니다.

## Node.js BroadcastChannel이 필요한 상황

### H3. 모든 워커에게 같은 신호를 보내야 할 때가 있다

워커 스레드는 CPU 중심 작업을 분리할 때 유용합니다.
이미지 처리, 리포트 생성, 대량 파싱처럼 메인 스레드를 오래 잡아먹는 작업을 분산할 수 있습니다.

하지만 여러 워커가 동시에 떠 있으면 공통 상태 신호가 필요해집니다.

- 설정 파일이 바뀌었으니 다음 작업부터 새 설정을 읽는다.
- 캐시 버전이 바뀌었으니 로컬 캐시를 비운다.
- 배포 전 종료 준비가 시작됐으니 새 작업을 받지 않는다.
- 특정 job group이 취소됐으니 관련 작업을 중단한다.
- 진단 모드가 켜졌으니 짧은 기간만 추가 로그를 남긴다.

이런 메시지는 특정 워커 하나만 받으면 안 됩니다.
반대로 큰 payload를 모든 워커에 계속 복사하는 구조도 좋지 않습니다.
`BroadcastChannel`은 “작고 명확한 신호”를 여러 수신자에게 보내는 데 잘 맞습니다.

### H3. parentPort와 역할을 분리하면 코드가 읽기 쉬워진다

`parentPort`는 메인 스레드와 특정 워커 사이의 직접 통신에 적합합니다.
작업 입력, 작업 결과, 에러 보고처럼 요청과 응답의 상대가 분명한 메시지는 `parentPort`로 두는 편이 자연스럽습니다.

반면 `BroadcastChannel`은 같은 이름의 채널을 연 모든 곳에 메시지를 보냅니다.
그래서 아래처럼 역할을 나누면 유지보수성이 좋아집니다.

- `parentPort`: 개별 작업 할당과 결과 응답
- `BroadcastChannel`: 전체 워커가 알아야 하는 상태 변경
- `workerData` 또는 `setEnvironmentData`: 워커 생성 시점의 초기 설정
- 공유 메모리: 정말 필요한 경우의 저수준 데이터 공유

워커 초기 설정을 함께 다루고 있다면 [Node.js worker_threads setEnvironmentData 가이드](/development/blog/seo/2026/05/19/nodejs-worker-threads-environment-data-guide.html)처럼 생성 시점 설정과 런타임 신호를 구분해 두는 편이 좋습니다.

## 기본 사용법

### H3. 같은 이름의 채널을 만들고 postMessage로 알린다

`BroadcastChannel`은 채널 이름을 기준으로 연결됩니다.
메인 스레드와 워커가 같은 이름의 채널을 열면, 한쪽에서 보낸 메시지를 다른 쪽에서 받을 수 있습니다.

```js
import { BroadcastChannel, Worker, isMainThread, threadId } from 'node:worker_threads';

const channel = new BroadcastChannel('app:control');

if (isMainThread) {
  for (let i = 0; i < 3; i += 1) {
    new Worker(new URL(import.meta.url));
  }

  setTimeout(() => {
    channel.postMessage({
      type: 'cache.invalidate',
      cacheName: 'feature-flags',
      version: Date.now()
    });
  }, 100);
} else {
  channel.onmessage = (event) => {
    console.log({
      threadId,
      message: event.data
    });
  };
}
```

이 예제에서 메인 스레드는 워커를 여러 개 만들고, 같은 채널로 캐시 무효화 신호를 보냅니다.
워커는 각자 같은 이름의 채널을 열고 메시지를 받습니다.

실무에서는 `console.log` 대신 로거나 메트릭 이벤트로 연결하는 편이 좋습니다.
관측 이벤트 분리를 함께 고민한다면 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/05/18/nodejs-diagnostics-channel-observability-guide.html)와도 잘 맞습니다.

### H3. 메시지는 작고 버전 가능한 객체로 만든다

`BroadcastChannel.postMessage()`에는 구조화 복제가 가능한 값을 보낼 수 있습니다.
그렇다고 큰 객체를 그대로 보내는 방식이 항상 좋은 것은 아닙니다.

워커 수가 많을수록 메시지 크기와 처리 비용이 커집니다.
따라서 메시지는 아래처럼 작고 명확한 이벤트 형태로 설계하는 편이 안전합니다.

```js
channel.postMessage({
  type: 'config.reload',
  version: 42,
  requestedAt: new Date().toISOString()
});
```

수신자는 `type`을 기준으로 처리하고, 실제 상세 데이터는 필요한 시점에 각자 읽게 만들 수 있습니다.

```js
channel.onmessage = async (event) => {
  const message = event.data;

  if (message?.type === 'config.reload') {
    await reloadConfig({ expectedVersion: message.version });
    return;
  }

  if (message?.type === 'cache.invalidate') {
    clearNamedCache(message.cacheName);
  }
};
```

이 구조는 나중에 메시지 종류가 늘어나도 추적하기 쉽습니다.
또한 로그에 남길 때도 전체 데이터가 아니라 이벤트 타입, 버전, 처리 결과만 기록할 수 있어 민감정보 노출 가능성이 줄어듭니다.

## 워커 제어 패턴

### H3. 종료 신호는 새 작업 차단과 현재 작업 정리로 나눈다

운영 종료나 배포 직전에는 워커가 새 작업을 받지 않도록 알려야 합니다.
이때 `shutdown` 메시지는 단순히 프로세스를 즉시 끝내라는 뜻이 아니라, 종료 절차를 시작하라는 신호로 두는 편이 안전합니다.

```js
let acceptingJobs = true;

channel.onmessage = (event) => {
  if (event.data?.type !== 'worker.shutdown') {
    return;
  }

  acceptingJobs = false;
};

export async function runJob(job) {
  if (!acceptingJobs) {
    return { skipped: true, reason: 'worker_is_shutting_down' };
  }

  return processJob(job);
}
```

실제 종료는 진행 중 작업의 성격에 따라 달라집니다.
짧은 작업은 마저 끝내고, 오래 걸리는 작업은 취소 가능한 지점을 만들어야 합니다.
HTTP 서버 종료와 비슷하게 다룬다면 [Node.js graceful shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)를 함께 참고할 수 있습니다.

### H3. 캐시 무효화는 값이 아니라 버전을 전달한다

캐시를 여러 워커가 들고 있을 때는 큰 캐시 데이터를 방송하기보다 “버전이 바뀌었다”는 사실만 보내는 편이 좋습니다.

```js
const localCache = new Map();
let currentVersion = 0;

channel.onmessage = (event) => {
  const message = event.data;

  if (message?.type !== 'cache.version.changed') {
    return;
  }

  if (message.version <= currentVersion) {
    return;
  }

  currentVersion = message.version;
  localCache.clear();
};
```

이렇게 하면 메시지를 여러 번 받아도 같은 버전은 무시할 수 있습니다.
네트워크 캐시 전략을 같이 정리하고 있다면 [Node.js stale-while-revalidate 캐시 전략 가이드](/development/blog/seo/2026/03/26/nodejs-stale-while-revalidate-cache-strategy-guide.html)처럼 캐시 갱신 기준을 별도로 문서화해 두는 편이 좋습니다.

## 정리와 생명주기 관리

### H3. 더 이상 쓰지 않는 채널은 close로 닫는다

`BroadcastChannel`을 열어 두면 메시지 수신을 위해 핸들이 유지됩니다.
테스트나 짧은 CLI에서 채널을 닫지 않으면 프로세스가 예상보다 오래 살아 있거나 테스트가 깔끔하게 끝나지 않을 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { BroadcastChannel } from 'node:worker_threads';

test('receives a broadcast message', async () => {
  const sender = new BroadcastChannel('test:control');
  const receiver = new BroadcastChannel('test:control');

  try {
    const received = new Promise((resolve) => {
      receiver.onmessage = (event) => resolve(event.data);
    });

    sender.postMessage({ type: 'ping' });

    assert.deepEqual(await received, { type: 'ping' });
  } finally {
    sender.close();
    receiver.close();
  }
});
```

테스트 격리와 정리 패턴은 [Node.js test runner hooks 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html)처럼 `afterEach`나 `finally`로 일관되게 처리하는 편이 좋습니다.

### H3. 백그라운드 신호 채널은 unref를 검토한다

`BroadcastChannel`에는 `unref()`가 있습니다.
채널만 남아 있을 때 프로세스가 종료될 수 있게 만드는 용도입니다.

```js
const channel = new BroadcastChannel('app:control');

channel.unref();
```

다만 `unref()`는 “정리하지 않아도 된다”는 뜻이 아닙니다.
운영 서버처럼 계속 떠 있는 프로세스에서는 명시적인 `close()` 기준이 더 중요합니다.
짧은 스크립트나 테스트 보조 채널처럼 프로세스 종료를 붙잡으면 곤란한 경우에 제한적으로 쓰는 편이 안전합니다.

## 운영에서 주의할 점

### H3. BroadcastChannel은 영속 메시지 큐가 아니다

`BroadcastChannel`은 현재 떠 있는 같은 프로세스 안의 채널 인스턴스에 메시지를 전달하는 데 적합합니다.
메시지를 저장해 두거나, 죽어 있던 워커가 나중에 다시 받아 가는 큐가 아닙니다.

따라서 아래 요구가 있다면 다른 도구를 써야 합니다.

- 메시지를 반드시 한 번 이상 처리해야 한다.
- 프로세스 재시작 뒤에도 이벤트를 복구해야 한다.
- 여러 서버 인스턴스 사이에 신호를 전파해야 한다.
- 실패한 메시지를 재시도하거나 DLQ로 보내야 한다.
- 처리 순서와 중복 제거가 비즈니스 요구사항이다.

이런 경우에는 Redis Pub/Sub, 메시지 큐, 이벤트 로그, 데이터베이스 기반 outbox 같은 구조가 더 적합합니다.
서비스 간 일관성이 필요하다면 [Node.js outbox pattern 가이드](/development/blog/seo/2026/03/28/nodejs-outbox-pattern-event-driven-consistency-guide.html)를 검토하는 편이 좋습니다.

### H3. 메시지에는 비밀값과 개인정보를 담지 않는다

브로드캐스트 메시지는 여러 수신자에게 전달됩니다.
그래서 특정 워커만 알아야 하는 값, 토큰, 사용자 개인정보, 원본 요청 payload를 넣으면 위험합니다.

안전한 기본값은 아래와 같습니다.

1. 메시지에는 이벤트 타입과 내부 버전, 짧은 식별자만 담는다.
2. 상세 데이터는 권한이 있는 저장소에서 각 워커가 다시 읽는다.
3. 로그에는 메시지 전체가 아니라 허용된 필드만 남긴다.
4. 채널 이름에도 고객명, 토큰, 내부 경로 같은 민감한 정보를 넣지 않는다.
5. 테스트 fixture에는 실제 운영 데이터를 넣지 않는다.

공개 개발 글에 예제를 옮길 때도 실제 로그나 운영 식별자는 사용하지 않는 것이 원칙입니다.

## 도입 체크리스트

### H3. 작은 신호부터 적용한다

`BroadcastChannel`을 도입할 때는 아래 순서가 현실적입니다.

1. 모든 워커가 알아야 하는 이벤트를 2~3개만 고른다.
2. 채널 이름 규칙을 `app:control`, `cache:events`처럼 짧고 안정적으로 정한다.
3. 메시지에 `type`과 `version`을 넣어 확장 여지를 둔다.
4. 큰 데이터는 보내지 않고 각 워커가 다시 읽게 만든다.
5. 테스트에서 `close()` 정리가 되는지 확인한다.
6. 운영 로그에는 이벤트 타입, 처리 결과, 지연 시간만 남긴다.

처음부터 모든 워커 통신을 `BroadcastChannel`로 옮기기보다, 캐시 무효화나 종료 준비처럼 실패 범위가 작고 의미가 분명한 신호부터 시작하는 편이 좋습니다.

## 마무리

Node.js `BroadcastChannel`은 워커 간 통신을 복잡한 메시지 라우터로 만들지 않고도, 공통 상태 신호를 단순하게 전달할 수 있는 도구입니다.
다만 이 도구의 강점은 데이터 공유가 아니라 이벤트 알림에 있습니다.

작은 메시지, 명확한 타입, 버전 기반 처리, `close()` 정리를 기본값으로 두면 워커 수가 늘어나도 통신 코드를 읽기 쉽게 유지할 수 있습니다.
반드시 보장되어야 하는 이벤트나 서버 간 전파가 필요한 신호는 별도 메시징 인프라로 넘기고, `BroadcastChannel`은 프로세스 내부의 가벼운 제어 채널로 쓰는 것이 안전합니다.

## 함께 보면 좋은 글

- [Node.js worker_threads setEnvironmentData 가이드: 워커 공통 설정을 안전하게 전달하는 법](/development/blog/seo/2026/05/19/nodejs-worker-threads-environment-data-guide.html)
- [Node.js diagnostics_channel 가이드: 라이브러리 관측 이벤트를 안전하게 발행하는 법](/development/blog/seo/2026/05/18/nodejs-diagnostics-channel-observability-guide.html)
- [Node.js test runner hooks 가이드: beforeEach와 afterEach로 테스트 정리 누락 줄이기](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html)

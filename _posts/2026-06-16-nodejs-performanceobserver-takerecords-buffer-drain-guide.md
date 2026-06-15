---
layout: post
title: "Node.js PerformanceObserver.takeRecords 가이드: 관측 버퍼를 안전하게 비우는 법"
date: 2026-06-16 08:00:00 +0900
lang: ko
translation_key: nodejs-performanceobserver-takerecords-buffer-drain-guide
permalink: /development/blog/seo/2026/06/16/nodejs-performanceobserver-takerecords-buffer-drain-guide.html
alternates:
  ko: /development/blog/seo/2026/06/16/nodejs-performanceobserver-takerecords-buffer-drain-guide.html
  x_default: /development/blog/seo/2026/06/16/nodejs-performanceobserver-takerecords-buffer-drain-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, performanceobserver, takerecords, perf-hooks, monitoring, observability]
description: "Node.js PerformanceObserver.takeRecords()로 observer 내부에 쌓인 성능 항목을 동기적으로 꺼내고, disconnect 전 플러시·테스트 격리·중복 처리 방지 패턴을 적용하는 방법을 설명합니다."
---

Node.js에서 `PerformanceObserver`를 사용하면 `mark`, `measure`, `function`, `gc` 같은 성능 항목이 생길 때 콜백으로 받아 처리할 수 있습니다.
대부분의 서비스 코드는 콜백 안에서 항목을 소비하지만, 짧게 실행되는 CLI, 테스트, 종료 직전 플러시처럼 "지금까지 observer에 들어온 항목을 바로 꺼내야 하는" 순간도 있습니다.

`performanceObserver.takeRecords()`는 observer 내부 큐에 저장된 항목을 배열로 반환하고 그 큐를 비웁니다.
콜백이 실행되기 전 남아 있는 항목을 마지막으로 소비하거나, 테스트에서 비동기 콜백 타이밍에 덜 흔들리는 검증을 만들 때 유용합니다.

이 글에서는 `takeRecords()`의 역할, 콜백 처리와의 차이, `disconnect()` 전 플러시 순서, 테스트에서의 사용법, 운영 코드에서 중복 처리와 민감정보 노출을 피하는 기준을 정리합니다.
관측 가능한 entry type을 먼저 확인하려면 [Node.js PerformanceObserver.supportedEntryTypes 가이드](/development/blog/seo/2026/06/13/nodejs-performanceobserver-supportedentrytypes-guide.html)를 함께 참고하세요.

## takeRecords가 필요한 이유

### H3. observer 콜백만으로는 마지막 항목을 놓칠 수 있다

`PerformanceObserver`는 관측 대상 항목이 생기면 등록한 콜백을 호출합니다.
하지만 프로세스가 바로 종료되거나 테스트가 observer를 곧바로 해제하면 콜백이 처리하기 전에 항목이 내부 큐에 남아 있을 수 있습니다.

```js
import { performance, PerformanceObserver } from 'node:perf_hooks';

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(entry.name, entry.duration);
  }
});

observer.observe({ entryTypes: ['measure'] });

performance.mark('build:start');
performance.mark('build:end');
performance.measure('build:total', 'build:start', 'build:end');
```

긴 서버 프로세스에서는 다음 이벤트 루프 턴에 콜백이 실행되어도 충분한 경우가 많습니다.
반면 짧은 스크립트나 테스트에서는 "콜백이 언젠가 실행될 것"에 기대면 결과가 환경에 따라 달라질 수 있습니다.

### H3. takeRecords는 현재 큐를 동기적으로 비운다

`takeRecords()`를 호출하면 observer가 아직 콜백으로 전달하지 않은 항목을 즉시 배열로 받을 수 있습니다.
반환과 동시에 observer 내부 큐는 비워지므로 같은 항목을 다시 `takeRecords()`로 받을 수 없습니다.

```js
const pendingEntries = observer.takeRecords();

for (const entry of pendingEntries) {
  console.log({
    name: entry.name,
    type: entry.entryType,
    durationMs: Number(entry.duration.toFixed(2))
  });
}
```

이 메서드는 전역 성능 타임라인을 지우지 않습니다.
observer의 내부 큐를 비우는 작업과 `performance.clearMarks()`, `performance.clearMeasures()`로 타임라인 항목을 정리하는 작업은 서로 다른 책임입니다.
타임라인 정리 기준은 [Node.js performance.clearMarks·clearMeasures 가이드](/development/blog/seo/2026/06/15/nodejs-performance-clearmarks-clearmeasures-timeline-cleanup-guide.html)에서 자세히 다룹니다.

## 기본 사용 패턴

### H3. 종료 직전에 남은 항목을 플러시한다

CLI나 배치 작업처럼 명확한 종료 지점이 있는 코드에서는 observer를 해제하기 전에 `takeRecords()`로 남은 항목을 소비하는 순서가 이해하기 쉽습니다.

```js
import { performance, PerformanceObserver } from 'node:perf_hooks';
import { setTimeout as delay } from 'node:timers/promises';

const timings = [];

const observer = new PerformanceObserver((list) => {
  timings.push(...list.getEntries());
});

observer.observe({ entryTypes: ['measure'] });

async function buildReport() {
  performance.mark('report:start');
  await delay(30);
  performance.mark('report:end');
  performance.measure('report:total', 'report:start', 'report:end');
}

try {
  await buildReport();
} finally {
  timings.push(...observer.takeRecords());
  observer.disconnect();

  performance.clearMeasures('report:total');
  performance.clearMarks('report:start');
  performance.clearMarks('report:end');
}

console.log(
  timings.map((entry) => ({
    name: entry.name,
    durationMs: Number(entry.duration.toFixed(2))
  }))
);
```

핵심은 `disconnect()`보다 `takeRecords()`를 먼저 호출하는 것입니다.
해제 전에 남은 큐를 비우면 마지막 측정값을 처리할 기회를 명확히 만들 수 있습니다.

### H3. 소비 함수는 콜백과 플러시에서 공유한다

콜백과 종료 플러시가 서로 다른 형식으로 로그를 만들면 같은 entry type인데도 필드명이 달라질 수 있습니다.
항목 소비 함수를 하나로 만들면 중복 처리와 포맷 불일치를 줄일 수 있습니다.

```js
function consumePerformanceEntries(entries, sink) {
  for (const entry of entries) {
    sink({
      event: entry.name,
      entryType: entry.entryType,
      durationMs: Number(entry.duration.toFixed(2)),
      startTimeMs: Number(entry.startTime.toFixed(2))
    });
  }
}

const observer = new PerformanceObserver((list) => {
  consumePerformanceEntries(list.getEntries(), console.log);
});

observer.observe({ entryTypes: ['measure'] });

// 종료 경계
consumePerformanceEntries(observer.takeRecords(), console.log);
observer.disconnect();
```

운영 로그에는 필요한 필드만 남기는 편이 안전합니다.
entry 이름에 사용자 입력, 전체 URL, 이메일, 토큰 같은 값이 들어가지 않도록 고정된 이름 규칙을 먼저 정해야 합니다.

## callback, takeRecords, disconnect의 차이

### H3. callback은 비동기 소비 경로다

observer 콜백은 성능 항목을 지속적으로 처리하는 기본 경로입니다.
서버처럼 오래 실행되는 프로세스에서는 콜백에서 항목을 구조화 로그나 메트릭 시스템으로 보내는 방식이 자연스럽습니다.

```js
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    metrics.observe(entry.name, entry.duration);
  }
});

observer.observe({ entryTypes: ['function'] });
```

함수 실행 시간을 관측하는 기본 흐름은 [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)를 참고하면 됩니다.
`takeRecords()`는 이 콜백 경로를 대체하기보다 종료와 테스트 같은 경계에서 보완하는 도구에 가깝습니다.

### H3. takeRecords는 큐 배출이고 disconnect는 구독 해제다

`takeRecords()`는 observer 내부에 이미 쌓인 항목을 꺼냅니다.
새 항목 관측을 멈추지는 않습니다.

`disconnect()`는 observer의 관측을 중단합니다.
해제 이후에는 새 성능 항목이 생겨도 해당 observer가 받지 않습니다.

```js
const remaining = observer.takeRecords();
consumePerformanceEntries(remaining, console.log);

observer.disconnect();
```

종료 루틴에서는 보통 `takeRecords()`로 남은 항목을 먼저 소비하고, 그 다음 `disconnect()`로 더 이상 관측하지 않게 만드는 순서를 사용합니다.
반대로 장기 실행 서버에서 일시적으로 큐만 비우고 계속 관측하려면 `disconnect()`를 호출하지 않아야 합니다.

## 테스트에서 takeRecords 사용하기

### H3. 비동기 콜백 대기보다 명시적 플러시가 단순할 때가 있다

테스트에서 observer 콜백이 호출될 때까지 타이머를 기다리면 테스트가 느려지고 흔들릴 수 있습니다.
측정 항목을 만든 직후 `takeRecords()`를 호출하면 현재 observer 큐를 직접 검증할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { performance, PerformanceObserver } from 'node:perf_hooks';

test('records report measure entries', () => {
  performance.clearMarks();
  performance.clearMeasures();

  const observer = new PerformanceObserver(() => {});
  observer.observe({ entryTypes: ['measure'] });

  performance.mark('report:start');
  performance.mark('report:end');
  performance.measure('report:total', 'report:start', 'report:end');

  const entries = observer.takeRecords();

  observer.disconnect();
  performance.clearMeasures('report:total');
  performance.clearMarks('report:start');
  performance.clearMarks('report:end');

  assert.equal(entries.length, 1);
  assert.equal(entries[0].name, 'report:total');
  assert.equal(entries[0].entryType, 'measure');
});
```

테스트가 전역 성능 타임라인을 공유한다는 점도 중요합니다.
테스트 시작과 종료에서 관련 항목을 정리해야 이전 테스트가 남긴 `mark`나 `measure` 때문에 검증이 오염되지 않습니다.

### H3. 테스트별 observer를 만들고 반드시 해제한다

하나의 observer를 여러 테스트가 공유하면 어떤 테스트가 만든 항목인지 구분하기 어려워집니다.
테스트마다 observer를 만들고 `finally` 또는 테스트 훅에서 `disconnect()`를 호출하면 범위를 좁힐 수 있습니다.

```js
test('flushes only entries observed in this test', () => {
  const observer = new PerformanceObserver(() => {});

  try {
    observer.observe({ entryTypes: ['measure'] });

    performance.mark('job:start');
    performance.mark('job:end');
    performance.measure('job:total', 'job:start', 'job:end');

    const entries = observer.takeRecords();

    assert.deepEqual(
      entries.map((entry) => entry.name),
      ['job:total']
    );
  } finally {
    observer.disconnect();
    performance.clearMeasures('job:total');
    performance.clearMarks('job:start');
    performance.clearMarks('job:end');
  }
});
```

여러 테스트가 병렬로 실행된다면 `job:total`처럼 고정된 이름이 충돌할 수 있습니다.
병렬 테스트에서는 테스트별 접두사를 만들거나, 해당 파일 안에서 성능 타임라인을 독점하는 구조인지 먼저 확인해야 합니다.

## 운영 코드에서 주의할 점

### H3. takeRecords 결과를 콜백 결과와 중복 저장하지 않는다

observer 콜백이 이미 받은 항목은 `takeRecords()`에 다시 남아 있지 않습니다.
하지만 애플리케이션 코드에서 콜백 결과 배열과 플러시 결과 배열을 합칠 때 같은 항목을 다른 경로로 두 번 저장하는 구조를 만들 수 있습니다.

```js
const pending = [];

const observer = new PerformanceObserver((list) => {
  pending.push(...list.getEntries());
});

function flushPerformanceEntries() {
  pending.push(...observer.takeRecords());

  const batch = pending.splice(0, pending.length);
  consumePerformanceEntries(batch, sendMetric);
}
```

한 곳의 임시 배열에 모았다가 배치 단위로 비우면 중복 전송 가능성을 줄일 수 있습니다.
메트릭 전송이 실패하는 경우에는 재시도 큐와 observer 큐를 혼동하지 않도록 별도 자료구조를 두는 편이 좋습니다.

### H3. 무제한 버퍼처럼 사용하지 않는다

`takeRecords()`가 있다고 해서 observer를 무제한 큐로 사용해도 된다는 뜻은 아닙니다.
소비가 느리거나 메트릭 시스템 장애가 길어지면 애플리케이션 메모리와 로그 비용이 함께 커질 수 있습니다.

```js
const MAX_BATCH_SIZE = 100;
const pending = [];

function enqueueEntries(entries) {
  for (const entry of entries) {
    pending.push(entry);

    if (pending.length >= MAX_BATCH_SIZE) {
      flushPerformanceEntries();
    }
  }
}
```

운영 환경에서는 배치 크기, 전송 주기, 실패 시 폐기 또는 샘플링 기준을 정해야 합니다.
성능 관측 코드는 장애를 설명하기 위한 보조 수단이지, 장애 상황에서 서비스 본문보다 더 많은 자원을 사용해서는 안 됩니다.

### H3. 민감한 detail과 이름을 그대로 내보내지 않는다

`performance.mark()`와 `performance.measure()`는 `detail` 값을 가질 수 있습니다.
또한 entry 이름도 개발자가 정합니다.
편의를 위해 요청 객체나 사용자 입력 전체를 넣으면 로그와 메트릭에 민감정보가 퍼질 수 있습니다.

```js
function sanitizeEntry(entry) {
  return {
    event: entry.name,
    entryType: entry.entryType,
    durationMs: Number(entry.duration.toFixed(2))
  };
}
```

사용자 ID, 이메일, 토큰, 전체 쿼리스트링, 외부 API 응답 전문은 성능 entry에 넣지 않는 편이 좋습니다.
필요하다면 제한된 enum 값이나 내부 추적 ID처럼 노출 범위가 통제된 값만 사용해야 합니다.

## 실무 적용 체크리스트

### H3. 종료 플러시 순서를 표준화한다

짧은 프로세스와 테스트에서는 다음 순서를 기본값으로 삼을 수 있습니다.

1. observer 콜백에서 평소처럼 항목을 수집한다.
2. 종료 직전에 `observer.takeRecords()`로 남은 항목을 합친다.
3. 합친 항목을 같은 소비 함수로 처리한다.
4. `observer.disconnect()`로 관측을 해제한다.
5. 필요한 `mark`, `measure`, resource timing 항목을 별도로 정리한다.

이 순서를 함수로 감싸면 각 스크립트가 제각각 플러시 로직을 갖지 않아도 됩니다.

```js
function closePerformanceObserver(observer, consume) {
  const remaining = observer.takeRecords();

  if (remaining.length > 0) {
    consume(remaining);
  }

  observer.disconnect();
}
```

공통 함수 안에서는 전역 타임라인을 무조건 모두 지우지 않는 편이 안전합니다.
어떤 이름을 정리할지는 측정 항목을 만든 코드가 책임지는 구조가 더 예측 가능합니다.

### H3. 서버에서는 주기 플러시와 종료 플러시를 나눈다

장기 실행 서버는 콜백에서 바로 전송하거나, 짧은 배치 주기로 전송하는 방식이 적합합니다.
다만 종료 신호를 받을 때는 마지막으로 `takeRecords()`를 호출해 남은 항목을 처리할 수 있습니다.

```js
process.once('SIGTERM', () => {
  try {
    flushPerformanceEntries();
    consumePerformanceEntries(observer.takeRecords(), sendMetric);
  } finally {
    observer.disconnect();
  }
});
```

실제 서비스에서는 비동기 전송의 타임아웃도 함께 두어야 합니다.
종료 과정에서 메트릭 전송 때문에 프로세스가 오래 붙잡히면 배포나 오토스케일링 흐름에 영향을 줄 수 있습니다.

## 마무리

`PerformanceObserver.takeRecords()`는 observer 내부에 남은 성능 항목을 동기적으로 꺼내고 큐를 비우는 메서드입니다.
일상적인 관측은 콜백에서 처리하되, 테스트와 종료 경계에서는 `takeRecords()`로 마지막 항목을 명시적으로 플러시하면 결과를 더 예측 가능하게 만들 수 있습니다.

다만 이 메서드는 전역 성능 타임라인을 정리하지 않습니다.
observer 큐 배출, observer 해제, `mark`와 `measure` 정리는 각각 다른 책임이므로 순서를 분리해 두는 편이 좋습니다.

다음 글을 함께 보면 흐름을 이어서 정리할 수 있습니다.

- [Node.js PerformanceObserver.supportedEntryTypes 가이드](/development/blog/seo/2026/06/13/nodejs-performanceobserver-supportedentrytypes-guide.html)
- [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)
- [Node.js performance.clearMarks·clearMeasures 가이드](/development/blog/seo/2026/06/15/nodejs-performance-clearmarks-clearmeasures-timeline-cleanup-guide.html)

---
layout: post
title: "Node.js createHistogram 가이드: 지연 시간 백분위를 가볍게 측정하는 법"
date: 2026-05-31 20:00:00 +0900
lang: ko
translation_key: nodejs-perf-hooks-createhistogram-latency-percentile-guide
permalink: /development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html
alternates:
  ko: /development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html
  x_default: /development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, perf_hooks, latency, percentile, observability, backend]
description: "Node.js perf_hooks.createHistogram으로 요청 지연 시간의 p50, p95, p99 백분위를 측정하는 방법을 정리했습니다. 기본 사용법, 단위 변환, 스냅샷 리포트, 운영 지표 설계 주의점을 예제로 설명합니다."
---

서비스 성능을 볼 때 평균 응답 시간만 확인하면 중요한 문제를 놓치기 쉽습니다.
대부분의 요청은 빠르지만 일부 요청만 갑자기 느려지는 상황에서는 평균이 크게 흔들리지 않을 수 있기 때문입니다.

그래서 운영 지표에서는 `p50`, `p95`, `p99` 같은 백분위를 함께 봅니다.
Node.js에서는 `node:perf_hooks`의 `createHistogram()`을 사용해 지연 시간 분포를 가볍게 모을 수 있습니다.

이 글에서는 `createHistogram()` 기본 사용법, 밀리초 단위로 측정값을 기록하는 방법, 주기적으로 스냅샷을 출력하는 패턴, 그리고 운영 지표로 사용할 때 주의할 점을 정리합니다.
짧은 구간의 실행 시간을 먼저 측정하고 싶다면 [Node.js performance.now 가이드](/development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html)를 함께 보면 좋습니다.

## createHistogram이 필요한 이유

### H3. 평균은 꼬리 지연을 잘 보여 주지 못한다

평균 응답 시간은 이해하기 쉽지만, 사용자 경험을 충분히 설명하지 못합니다.
예를 들어 100개의 요청 중 95개는 30ms 안에 끝나고 5개가 2초씩 걸린다면 평균은 어느 정도 올라가지만, 실제 문제는 느린 5개 요청에 있습니다.

백분위는 이런 꼬리 지연을 더 잘 드러냅니다.

- `p50`: 절반의 요청이 이 시간 안에 끝난다.
- `p95`: 95%의 요청이 이 시간 안에 끝난다.
- `p99`: 거의 최악에 가까운 느린 요청 구간을 본다.

운영에서는 평균보다 `p95`, `p99`가 알림 기준에 더 적합한 경우가 많습니다.
특히 외부 API, 데이터베이스, 파일 처리처럼 일부 호출이 튀는 구간에서는 백분위 지표가 문제를 더 빨리 보여 줍니다.

### H3. 히스토그램은 측정값의 분포를 모은다

`createHistogram()`은 숫자 측정값을 기록하고, 이후 최소값, 최대값, 평균, 표준편차, 백분위를 조회할 수 있게 해 줍니다.
매 요청의 값을 배열에 계속 쌓아 두지 않아도 되므로 단순한 메모리 리스트보다 운영 코드에 넣기 쉽습니다.

```js
import { createHistogram } from 'node:perf_hooks';

const latency = createHistogram();

latency.record(12);
latency.record(35);
latency.record(140);

console.log(latency.percentile(50));
console.log(latency.percentile(95));
```

`record()`에는 양의 정수 값을 넣어야 합니다.
따라서 `performance.now()`처럼 소수점 밀리초 값을 반환하는 API와 함께 쓸 때는 단위를 정하고 정수로 변환하는 규칙을 먼저 정하는 편이 좋습니다.

## 기본 측정 패턴

### H3. 요청 처리 시간을 밀리초 정수로 기록한다

가장 단순한 형태는 요청 시작 시각과 종료 시각의 차이를 구한 뒤 히스토그램에 기록하는 것입니다.

```js
import { createServer } from 'node:http';
import { createHistogram, performance } from 'node:perf_hooks';

const requestLatencyMs = createHistogram();

const server = createServer(async (req, res) => {
  const startedAt = performance.now();

  try {
    await handleRequest(req, res);
  } finally {
    const durationMs = performance.now() - startedAt;
    requestLatencyMs.record(Math.max(1, Math.round(durationMs)));
  }
});

async function handleRequest(req, res) {
  res.writeHead(200, { 'content-type': 'text/plain; charset=utf-8' });
  res.end('ok');
}

server.listen(3000);
```

여기서는 `Math.round()`로 밀리초 정수로 바꾸고, 너무 짧은 요청이 0으로 기록되지 않도록 `Math.max(1, ...)`을 적용했습니다.
팀에 따라 마이크로초나 나노초 단위로 저장할 수도 있지만, 단위가 바뀌면 대시보드와 로그 메시지에도 같은 기준을 명확히 남겨야 합니다.

### H3. 비동기 작업 단위에도 같은 구조를 적용한다

HTTP 요청뿐 아니라 외부 API 호출, 파일 처리, 큐 작업에도 같은 패턴을 적용할 수 있습니다.
중요한 점은 측정 대상의 경계를 좁게 잡는 것입니다.

```js
import { createHistogram, performance } from 'node:perf_hooks';

const externalApiLatencyMs = createHistogram();

async function measureExternalApi(callApi) {
  const startedAt = performance.now();

  try {
    return await callApi();
  } finally {
    const durationMs = performance.now() - startedAt;
    externalApiLatencyMs.record(Math.max(1, Math.round(durationMs)));
  }
}
```

이렇게 감싸 두면 호출 성공 여부와 관계없이 지연 시간이 기록됩니다.
실패한 요청의 지연 시간도 운영상 중요한 신호가 될 수 있기 때문입니다.

다만 실패율과 지연 시간은 서로 다른 지표입니다.
백분위 지연 시간만 보고 장애를 판단하기보다, 오류율과 함께 보는 편이 안전합니다.
지표를 더 넓게 설계하려면 [Node.js diagnostics_channel 관측성 가이드](/development/blog/seo/2026/05/18/nodejs-diagnostics-channel-observability-guide.html)와 연결해 이벤트 발행 구조를 잡아도 좋습니다.

## 리포트 출력과 초기화

### H3. 주기적으로 스냅샷을 남긴다

히스토그램은 특정 구간의 분포를 보는 데 유용합니다.
예를 들어 1분마다 현재 값을 로그로 남기고 다음 구간을 새로 측정할 수 있습니다.

```js
import { createHistogram } from 'node:perf_hooks';

const requestLatencyMs = createHistogram();

function snapshotLatency(histogram) {
  if (histogram.count === 0) {
    return null;
  }

  return {
    count: histogram.count,
    minMs: histogram.min,
    meanMs: Number(histogram.mean.toFixed(2)),
    p50Ms: histogram.percentile(50),
    p95Ms: histogram.percentile(95),
    p99Ms: histogram.percentile(99),
    maxMs: histogram.max
  };
}

setInterval(() => {
  const snapshot = snapshotLatency(requestLatencyMs);

  if (snapshot) {
    console.log(JSON.stringify({ metric: 'request_latency', ...snapshot }));
    requestLatencyMs.reset();
  }
}, 60_000).unref();
```

`reset()`을 호출하면 다음 구간을 새로 모을 수 있습니다.
이 방식은 자체 로그 기반 리포트에는 충분히 단순하지만, 장기 추세 분석이 필요하다면 Prometheus, OpenTelemetry, APM 같은 외부 지표 시스템으로 내보내는 구조를 고려해야 합니다.

### H3. 사람이 읽는 로그와 기계가 읽는 로그를 구분한다

운영 로그로 남길 때는 사람이 보기 좋은 문장보다 구조화된 JSON이 더 다루기 쉽습니다.
검색, 집계, 알림 조건을 만들 때 필드가 고정되어 있어야 실수가 줄어듭니다.

```js
function formatLatencyLog(name, histogram) {
  const snapshot = snapshotLatency(histogram);

  if (!snapshot) {
    return null;
  }

  return JSON.stringify({
    type: 'latency_histogram',
    name,
    unit: 'ms',
    ...snapshot
  });
}
```

색상이나 장식이 들어간 CLI 출력과 운영 로그를 함께 다루고 있다면 [Node.js stripVTControlCharacters 가이드](/development/blog/seo/2026/05/31/nodejs-util-strip-vt-control-characters-cli-output-guide.html)의 출력 분리 기준도 참고할 수 있습니다.

## 운영 적용 시 주의할 점

### H3. 지표 이름과 단위를 고정한다

백분위 지표는 이름과 단위가 흔들리면 비교하기 어렵습니다.
처음부터 아래 기준을 정해 두는 편이 좋습니다.

- 지표 이름은 `request_latency`, `external_api_latency`처럼 측정 대상을 드러낸다.
- 단위는 `ms`처럼 로그 필드나 대시보드 이름에 명시한다.
- `p95`, `p99`가 어느 시간 창의 값인지 함께 기록한다.
- 성공 요청만 볼지, 실패 요청도 포함할지 규칙을 정한다.
- 라벨을 너무 많이 늘려 고카디널리티 지표를 만들지 않는다.

특히 URL 전체, 사용자 ID, 주문 ID처럼 값이 계속 늘어나는 필드를 지표 라벨로 쓰면 관측성 비용과 성능 문제가 생길 수 있습니다.
라우트 패턴이나 작업 유형처럼 제한된 값만 라벨로 쓰는 편이 안전합니다.

### H3. 이벤트 루프 지연과 요청 지연을 혼동하지 않는다

요청 지연 시간은 사용자가 경험한 전체 처리 시간에 가깝습니다.
반면 이벤트 루프 지연은 Node.js 런타임이 작업을 제때 처리하지 못하고 밀린 정도를 보여 줍니다.

두 지표는 연결되어 있지만 같은 값은 아닙니다.
CPU 바운드 작업이 늘어나면 이벤트 루프 지연과 요청 지연이 함께 나빠질 수 있고, 외부 API가 느려지면 요청 지연만 나빠질 수도 있습니다.

런타임 자체의 밀림을 보고 싶다면 [Node.js 이벤트 루프 지연 모니터링 가이드](/development/blog/seo/2026/04/13/nodejs-event-loop-lag-monitoring-guide.html)를 함께 적용하세요.
요청 지연 히스토그램과 이벤트 루프 지연 지표를 나란히 보면 병목이 애플리케이션 내부인지 외부 의존성인지 판단하기 쉬워집니다.

## 실무 체크리스트

### H3. 백분위 지표를 넣기 전 확인할 항목

- 평균만 보고 있던 지표에 `p50`, `p95`, `p99`를 추가했는가?
- `record()`에 들어가는 값의 단위와 정수 변환 규칙이 명확한가?
- 주기적 스냅샷과 `reset()` 시점이 운영 목적에 맞는가?
- 성공과 실패 요청을 같은 히스토그램에 넣을지 분리할지 정했는가?
- 지표 라벨에 사용자별 고유값이나 민감정보가 들어가지 않는가?

작게 시작한다면 요청 전체 지연 시간 하나만 먼저 기록해도 충분합니다.
그 다음 외부 API, 데이터베이스, 큐 작업처럼 병목 후보가 분명한 구간으로 넓히면 지표가 과하게 늘어나는 일을 막을 수 있습니다.

### H3. 내부 링크로 이어서 볼 글

- [Node.js performance.now 가이드: 짧은 실행 시간을 안정적으로 측정하는 법](/development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html)
- [Node.js diagnostics_channel 가이드: 관측성 이벤트를 느슨하게 연결하는 법](/development/blog/seo/2026/05/18/nodejs-diagnostics-channel-observability-guide.html)
- [Node.js 이벤트 루프 지연 모니터링 가이드](/development/blog/seo/2026/04/13/nodejs-event-loop-lag-monitoring-guide.html)

## 마무리

`createHistogram()`은 복잡한 관측성 플랫폼을 대체하는 도구는 아닙니다.
하지만 Node.js 코드 안에서 지연 시간 분포를 빠르게 확인하고, 평균으로는 보이지 않는 꼬리 지연을 잡아내는 출발점으로는 충분히 쓸 만합니다.

운영에서 중요한 것은 숫자를 많이 모으는 것이 아니라, 같은 기준으로 꾸준히 비교할 수 있게 만드는 것입니다.
측정 단위, 시간 창, 성공과 실패 포함 여부를 명확히 정해 두면 `p95`, `p99` 지표가 성능 개선의 우선순위를 훨씬 선명하게 보여 줍니다.

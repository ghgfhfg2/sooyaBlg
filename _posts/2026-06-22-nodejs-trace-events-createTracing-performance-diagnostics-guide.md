---
layout: post
title: "Node.js trace_events 가이드: createTracing으로 성능 문제를 타임라인에서 추적하기"
date: 2026-06-22 08:00:00 +0900
lang: ko
translation_key: nodejs-trace-events-createTracing-performance-diagnostics-guide
permalink: /development/blog/seo/2026/06/22/nodejs-trace-events-createTracing-performance-diagnostics-guide.html
alternates:
  ko: /development/blog/seo/2026/06/22/nodejs-trace-events-createTracing-performance-diagnostics-guide.html
  x_default: /development/blog/seo/2026/06/22/nodejs-trace-events-createTracing-performance-diagnostics-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, trace_events, createTracing, performance, diagnostics, observability, backend]
description: "Node.js trace_events와 createTracing()으로 V8, async_hooks, node.perf 이벤트를 수집하고 성능 문제를 타임라인에서 분석하는 방법을 정리합니다. 카테고리 선택, 운영 적용, 파일 관리 주의점을 예제로 설명합니다."
---

Node.js 성능 문제를 로그와 지표만으로 설명하기 어려운 순간이 있습니다.
CPU가 높다는 사실은 보이지만 어떤 구간에서 일이 몰렸는지 모르거나, `performance.mark()`로 남긴 측정값과 V8, async_hooks, 파일 시스템 이벤트를 한 화면에서 맞춰 보고 싶을 때가 그렇습니다.
이럴 때 사용할 수 있는 진단 도구가 `node:trace_events`입니다.

[Node.js 공식 문서](https://nodejs.org/api/tracing.html)에 따르면 `node:trace_events` 모듈은 V8, Node.js core, userspace 코드에서 생성되는 tracing 정보를 한곳에 모으는 메커니즘을 제공합니다.
CLI의 `--trace-event-categories` 플래그로 켤 수도 있고, 애플리케이션 코드에서 `createTracing()`으로 특정 구간만 켤 수도 있습니다.
이 글에서는 `trace_events.createTracing()`, `getEnabledCategories()`, 카테고리 선택 기준, 운영 환경에서의 주의점을 실무 관점으로 정리합니다.

이벤트 루프 부하를 숫자로 먼저 보고 싶다면 [Node.js performance.eventLoopUtilization 가이드](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)를 함께 보면 좋습니다.
코드 구간에 직접 이름을 붙이는 방법은 [Node.js performance.mark/measure 가이드](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)와 연결됩니다.
GC pause를 별도 지표로 관찰하려면 [Node.js PerformanceObserver GC 가이드](/development/blog/seo/2026/06/12/nodejs-performanceobserver-gc-pause-monitoring-guide.html)를 참고하세요.

## trace_events가 필요한 상황

### H3. 로그와 메트릭 사이의 빈칸을 채운다

운영 지표는 "무슨 일이 나빠졌는지"를 잘 보여 줍니다.
예를 들어 p95 응답 시간이 올랐고, 이벤트 루프 지연도 커졌고, CPU 사용량이 증가했다는 사실은 대시보드에서 금방 볼 수 있습니다.
하지만 그 지표만으로는 "그 순간 어떤 작업들이 겹쳤는지"를 설명하기 어렵습니다.

trace events는 여러 이벤트를 시간축 위에 올려 봅니다.
V8, Node.js 내부 동작, 성능 마크, async_hooks 같은 정보를 수집하면 문제 구간을 더 좁힐 수 있습니다.
정상 요청과 느린 요청을 비교하거나, 배포 전후의 startup 흐름을 비교할 때 특히 유용합니다.

### H3. 항상 켜 두는 도구가 아니라 짧게 켜는 진단 도구다

trace events는 자세한 정보를 남기는 만큼 비용과 파일 크기를 생각해야 합니다.
모든 카테고리를 항상 켜 두는 방식은 보통 적절하지 않습니다.
대신 재현 환경, staging, 짧은 운영 진단 구간에서 필요한 카테고리만 켜고 끄는 방식이 좋습니다.

```bash
node --trace-event-categories node.perf,v8 server.js
```

위 명령은 Node.js 프로세스를 trace event 수집 상태로 실행합니다.
실행 후 생성된 trace log는 Chrome의 `chrome://tracing` 같은 도구에서 열어 타임라인으로 확인할 수 있습니다.
파일은 기본적으로 `node_trace.${rotation}.log` 패턴으로 생성되며, 필요하면 `--trace-event-file-pattern`으로 이름을 제어할 수 있습니다.

## createTracing 기본 사용법

### H3. 코드에서 필요한 구간만 tracing한다

`createTracing()`은 지정한 카테고리를 다루는 `Tracing` 객체를 반환합니다.
처음에는 비활성 상태이고, `enable()`을 호출해야 수집이 시작됩니다.
진단하고 싶은 작업이 끝나면 `disable()`을 호출해 수집을 멈춥니다.

```js
import { createTracing } from 'node:trace_events';

const tracing = createTracing({
  categories: ['node.perf', 'v8']
});

tracing.enable();

await runExpensiveStartupStep();

tracing.disable();
```

이 방식은 전체 프로세스 실행 시간을 모두 기록하고 싶지 않을 때 좋습니다.
예를 들어 서버 부팅, 큰 데이터 로딩, batch job의 특정 단계처럼 범위가 명확한 작업을 분석할 수 있습니다.

### H3. enabled와 categories로 상태를 확인한다

`Tracing` 객체에는 현재 객체가 활성화됐는지 알려 주는 `enabled` 속성과 이 객체가 다루는 카테고리를 보여 주는 `categories` 속성이 있습니다.
디버깅 로그에 남겨 두면 진단 코드가 실제로 켜졌는지 확인하기 쉽습니다.

```js
import { createTracing } from 'node:trace_events';

const tracing = createTracing({
  categories: ['node.perf']
});

console.log({
  categories: tracing.categories,
  enabledBefore: tracing.enabled
});

tracing.enable();

console.log({
  enabledAfter: tracing.enabled
});

tracing.disable();
```

운영 코드에서는 trace 활성화 여부를 환경 변수나 관리 명령으로 제한하는 편이 좋습니다.
무심코 모든 환경에서 켜지지 않도록 기본값은 비활성으로 두고, 필요할 때만 명시적으로 켭니다.

## 카테고리 선택 기준

### H3. node.perf는 mark와 measure를 타임라인으로 연결한다

`node.perf` 카테고리는 Performance API 측정값을 trace events에 포함할 때 사용합니다.
이미 `performance.mark()`와 `performance.measure()`를 코드에 넣어 두었다면, trace 타임라인에서 애플리케이션 기준의 작업 이름을 함께 볼 수 있습니다.

```js
import { performance } from 'node:perf_hooks';
import { createTracing } from 'node:trace_events';

const tracing = createTracing({ categories: ['node.perf'] });

tracing.enable();

performance.mark('report:build:start');
await buildReport();
performance.mark('report:build:end');
performance.measure(
  'report:build',
  'report:build:start',
  'report:build:end'
);

tracing.disable();
```

이 패턴은 내부 함수명을 모두 노출하지 않으면서도, 분석에 필요한 업무 구간을 타임라인에 표시할 수 있다는 장점이 있습니다.
작업 이름은 로그와 대시보드에서도 같은 이름을 쓰면 나중에 비교가 쉬워집니다.

### H3. v8은 GC와 컴파일 흐름을 볼 때 쓴다

`v8` 카테고리는 GC, 컴파일, 실행과 관련된 이벤트를 보는 데 도움이 됩니다.
특정 요청이 느린 원인이 애플리케이션 코드인지, GC pause인지, cold start 시점의 컴파일 비용인지 구분할 때 단서가 됩니다.

다만 `v8` 이벤트는 양이 많을 수 있습니다.
운영에서 장시간 켜 두기보다 문제를 재현하는 짧은 구간에 한정하는 편이 좋습니다.
GC 지표 자체를 지속적으로 모니터링하려면 trace events보다 `PerformanceObserver`로 `gc` 항목을 수집하는 방식이 더 단순할 수 있습니다.

### H3. node.async_hooks는 컨텍스트 흐름을 깊게 볼 때만 사용한다

`node.async_hooks`는 비동기 리소스 생성과 연결 관계를 자세히 보여 줍니다.
요청 컨텍스트가 어디서 끊기는지, background job이 어떤 trigger async id에서 시작됐는지 추적할 때 도움이 됩니다.

하지만 이 카테고리는 이벤트 수가 빠르게 늘어날 수 있습니다.
기본 관측성 지표처럼 항상 켜는 용도보다는, 재현 가능한 버그를 좁히는 디버깅 도구로 다루는 것이 좋습니다.
요청 컨텍스트 설계 자체는 [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)를 먼저 기준으로 잡는 편이 안전합니다.

## getEnabledCategories로 현재 상태 점검하기

### H3. 여러 tracing 객체와 CLI 플래그는 합쳐서 계산된다

`getEnabledCategories()`는 현재 활성화된 trace event 카테고리 목록을 comma-separated string으로 반환합니다.
중요한 점은 이 값이 하나의 `Tracing` 객체만 보지 않는다는 것입니다.
여러 객체가 켠 카테고리와 CLI 플래그로 켠 카테고리가 합쳐진 결과입니다.

```js
import {
  createTracing,
  getEnabledCategories
} from 'node:trace_events';

const perfTracing = createTracing({ categories: ['node.perf'] });
const v8Tracing = createTracing({ categories: ['v8'] });

perfTracing.enable();
v8Tracing.enable();

console.log(getEnabledCategories());

v8Tracing.disable();

console.log(getEnabledCategories());

perfTracing.disable();
```

이 함수는 진단 endpoint나 startup 로그에서 현재 trace 설정을 확인할 때 유용합니다.
다만 외부에 그대로 노출하면 내부 운영 설정이 드러날 수 있으므로, 공개 API 응답에 넣기보다는 내부 로그나 관리자용 화면에서만 사용하는 편이 낫습니다.

### H3. disable은 겹친 카테고리를 모두 끄지 않는다

같은 카테고리를 여러 `Tracing` 객체나 CLI 플래그가 켠 상태라면, 어떤 객체 하나를 `disable()`해도 해당 카테고리가 계속 활성일 수 있습니다.
예를 들어 `node.perf`를 CLI에서 켰고 코드에서도 켰다면, 코드의 tracing을 끄더라도 CLI 설정 때문에 남아 있을 수 있습니다.

이 특성은 안전합니다.
한 모듈이 자신이 켠 tracing을 끈다고 해서 다른 진단 작업까지 갑자기 중단하지 않기 때문입니다.
대신 "내 코드에서 disable했는데 왜 카테고리가 남아 있지?"라는 혼동을 줄이려면 `getEnabledCategories()`로 전체 상태를 확인해야 합니다.

## 운영 적용 패턴

### H3. 환경 변수로 짧은 진단 구간만 허용한다

서비스 코드에 tracing helper를 넣을 때는 기본값을 꺼 두고, 명확한 환경 변수나 설정이 있을 때만 켜는 편이 안전합니다.

```js
import { createTracing } from 'node:trace_events';

export async function runWithOptionalTrace(name, task) {
  if (process.env.NODE_TRACE_DIAGNOSTICS !== '1') {
    return task();
  }

  const tracing = createTracing({
    categories: ['node.perf', 'v8']
  });

  tracing.enable();

  try {
    return await task();
  } finally {
    tracing.disable();
    console.info({ name }, 'trace diagnostics completed');
  }
}
```

`finally`에서 `disable()`을 호출해야 작업이 실패해도 tracing 상태가 길게 남지 않습니다.
이 helper를 batch job, migration, startup step처럼 범위가 분명한 곳에 먼저 적용하면 파일 크기와 분석 범위를 모두 관리하기 쉽습니다.

### H3. 파일 이름과 보관 정책을 정한다

trace 파일은 분석에는 유용하지만 장기간 쌓이면 디스크를 빠르게 사용합니다.
운영 환경에서 사용할 가능성이 있다면 파일 패턴, 저장 위치, 수집 후 삭제 기준을 함께 정해야 합니다.

```bash
node \
  --trace-event-categories node.perf,v8 \
  --trace-event-file-pattern 'trace-${pid}-${rotation}.log' \
  server.js
```

파일을 artifact로 수집한다면 민감한 업무명이나 내부 경로가 들어가지 않는지도 확인해야 합니다.
trace events 자체가 개인정보를 직접 수집하기 위한 도구는 아니지만, 사용자 코드가 남긴 mark 이름이나 console 관련 이벤트가 운영 맥락을 드러낼 수 있습니다.
그래서 외부 공유 전에는 로그와 같은 기준으로 검토하는 편이 좋습니다.

### H3. signal 종료 시 파일 생성까지 확인한다

trace log는 프로세스 종료 시점에 파일로 정리됩니다.
`SIGINT`, `SIGTERM` 같은 signal로 종료되는 환경에서는 종료 핸들러가 너무 거칠게 프로세스를 끊지 않는지 확인해야 합니다.

```js
process.on('SIGTERM', () => {
  console.info('received SIGTERM');
  process.exit(143);
});
```

실제 서비스에서는 서버 close, inflight request drain, queue worker stop 같은 graceful shutdown 절차와 함께 다룹니다.
trace 진단을 켠 날에는 종료 후 파일이 정상적으로 생성됐는지까지 확인해야 분석을 이어갈 수 있습니다.

## trace_events를 남용하지 않는 기준

### H3. 장기 지표는 metrics로, 짧은 분석은 trace로 본다

trace events는 문제 구간을 자세히 보는 도구입니다.
반면 SLO, 알림, 대시보드처럼 장기적으로 봐야 하는 값은 metrics로 관리하는 편이 좋습니다.

예를 들어 이벤트 루프 활용률, GC pause p95, 요청 latency p99는 metrics로 지속 수집합니다.
그 값이 이상해졌을 때 짧게 trace events를 켜서 어떤 작업이 겹쳤는지 보는 흐름이 자연스럽습니다.

### H3. 카테고리는 질문에 맞춰 최소화한다

분석 질문이 "내 performance.measure 구간이 어디서 길어졌나?"라면 `node.perf`부터 시작합니다.
질문이 "GC가 요청 지연과 겹쳤나?"라면 `v8`을 추가합니다.
질문이 "비동기 컨텍스트가 어디서 끊겼나?"라면 `node.async_hooks`를 제한적으로 사용합니다.

처음부터 모든 카테고리를 켜면 파일은 커지고 분석은 어려워집니다.
trace events의 가치는 많은 데이터를 쌓는 데 있는 것이 아니라, 좋은 질문에 맞는 타임라인을 얻는 데 있습니다.

## FAQ

### H3. trace_events와 PerformanceObserver는 무엇이 다른가요?

`PerformanceObserver`는 코드 안에서 성능 항목을 관찰하고 로그나 지표로 보내기 좋습니다.
`trace_events`는 여러 종류의 런타임 이벤트를 trace 파일로 모아 타임라인에서 분석하기 좋습니다.
지속 모니터링은 `PerformanceObserver`, 짧은 원인 분석은 `trace_events`로 나누면 관리가 쉽습니다.

### H3. production에서 trace_events를 켜도 되나요?

가능은 하지만 기본값으로 항상 켜 두는 방식은 피하는 편이 좋습니다.
카테고리, 기간, 파일 보관, 민감정보 검토 기준을 정하고 짧게 켜야 합니다.
먼저 staging에서 같은 설정으로 파일 크기와 성능 영향을 확인하는 것이 안전합니다.

### H3. createTracing은 Worker threads에서도 쓸 수 있나요?

Node.js 문서 기준으로 `node:trace_events` 모듈의 기능은 Worker threads에서 사용할 수 없습니다.
Worker 기반 구조라면 메인 스레드에서 가능한 진단 범위를 먼저 확인하고, worker 내부의 장기 지표는 별도 로그와 metrics로 보완하는 편이 좋습니다.

## 마무리

`node:trace_events`는 평소에 항상 켜 두는 관측성 기능이라기보다, 복잡한 성능 문제를 시간축으로 풀어 보는 진단 도구에 가깝습니다.
`createTracing()`으로 필요한 구간만 켜고, `node.perf`, `v8`, `node.async_hooks` 같은 카테고리를 질문에 맞게 선택하면 로그와 metrics 사이의 빈칸을 줄일 수 있습니다.

처음 도입할 때는 `performance.mark()`와 `node.perf` 조합부터 시작해 보세요.
애플리케이션의 업무 구간이 trace 타임라인에 보이기 시작하면, GC나 async_hooks 이벤트를 추가했을 때도 훨씬 덜 헤매게 됩니다.

## 함께 보면 좋은 글

- [Node.js performance.eventLoopUtilization 가이드: 이벤트 루프 부하 측정하기](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)
- [Node.js performance.mark/measure 가이드: 코드 구간 측정하기](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)
- [Node.js PerformanceObserver GC 가이드: GC pause 모니터링하기](/development/blog/seo/2026/06/12/nodejs-performanceobserver-gc-pause-monitoring-guide.html)
- [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)

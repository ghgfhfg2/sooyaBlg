---
layout: post
title: "Node.js PerformanceObserver GC 가이드: 가비지 컬렉션 일시정지 측정하기"
date: 2026-06-12 08:00:00 +0900
lang: ko
translation_key: nodejs-performanceobserver-gc-pause-monitoring-guide
permalink: /development/blog/seo/2026/06/12/nodejs-performanceobserver-gc-pause-monitoring-guide.html
alternates:
  ko: /development/blog/seo/2026/06/12/nodejs-performanceobserver-gc-pause-monitoring-guide.html
  x_default: /development/blog/seo/2026/06/12/nodejs-performanceobserver-gc-pause-monitoring-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performanceobserver, garbagecollection, gc, monitoring, observability]
description: "Node.js PerformanceObserver로 GC 성능 항목을 수집하고 가비지 컬렉션 일시정지 시간, 종류, 백분위를 관찰하는 방법을 재현 가능한 예제와 운영 체크리스트로 설명합니다."
---

Node.js 서비스의 응답 시간이 간헐적으로 튈 때는 외부 API나 데이터베이스뿐 아니라 가비지 컬렉션(GC)도 확인해야 합니다.
객체 생성량이 갑자기 늘거나 힙 압력이 높아지면 GC가 더 자주 실행되고, 일부 작업은 JavaScript 실행을 잠시 멈추게 할 수 있습니다.

`node:perf_hooks`의 `PerformanceObserver`는 런타임이 생성한 `gc` 성능 항목을 관찰할 수 있습니다.
이를 이용하면 각 GC 작업의 실행 시간과 종류를 수집하고, 응답 지연이 발생한 시점의 메모리 상태와 비교할 수 있습니다.

이 글에서는 Node.js에서 GC 성능 항목을 관찰하는 기본 방법, 종류별 집계와 백분위 계산, 운영 모니터링에 적용할 때의 주의점을 정리합니다.
이벤트 루프가 전반적으로 얼마나 바쁜지 먼저 확인하려면 [Node.js performance.eventLoopUtilization 가이드](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)도 함께 참고하세요.

## GC 일시정지를 관찰해야 하는 이유

### H3. 평균 응답 시간은 짧은 지연 급증을 가릴 수 있다

대부분의 요청이 빠르더라도 GC가 실행된 짧은 구간에 일부 요청의 응답 시간이 크게 늘어날 수 있습니다.
평균값만 보면 이런 현상이 작게 보이므로 p95·p99 응답 시간과 GC 실행 시간을 같은 타임라인에서 비교하는 편이 좋습니다.

GC 관찰은 다음 상황에서 유용합니다.

- 배포 후 p99 응답 시간이 갑자기 증가했을 때
- 큰 JSON 변환이나 대량 객체 생성 이후 지연이 발생할 때
- 메모리 사용량이 반복적으로 오르내리며 처리량이 떨어질 때
- 힙 크기 또는 캐시 정책 변경 전후를 비교할 때

GC가 관찰됐다는 사실만으로 메모리 누수나 성능 장애를 확정할 수는 없습니다.
실행 빈도, 일시정지 시간, 힙 사용량, 요청 지연의 변화를 함께 봐야 합니다.

### H3. PerformanceObserver는 GC 실행 결과를 성능 항목으로 전달한다

`PerformanceObserver`에 `entryTypes: ['gc']`를 지정하면 GC가 완료된 뒤 성능 항목을 받을 수 있습니다.

```js
import { PerformanceObserver } from 'node:perf_hooks';

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log({
      name: entry.name,
      durationMs: Number(entry.duration.toFixed(2)),
      detail: entry.detail
    });
  }
});

observer.observe({ entryTypes: ['gc'] });
```

`entry.duration`은 밀리초 단위입니다.
`entry.detail`에는 GC 종류와 플래그를 구분하는 값이 포함되므로, 운영에서는 숫자를 그대로 저장하기보다 허용된 이름으로 매핑해 집계하는 편이 읽기 쉽습니다.

## 재현 가능한 GC 관찰 예제

### H3. 객체를 반복 생성해 관찰 흐름을 확인한다

GC 실행 시점은 런타임과 환경에 따라 달라지므로 아래 예제에서 반드시 같은 수의 항목이 발생한다고 기대하면 안 됩니다.
예제의 목적은 `PerformanceObserver` 등록과 결과 수집 흐름을 확인하는 것입니다.

```js
import { PerformanceObserver } from 'node:perf_hooks';
import { setTimeout as delay } from 'node:timers/promises';

const entries = [];

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntriesByType('gc')) {
    entries.push({
      durationMs: entry.duration,
      kind: entry.detail.kind,
      flags: entry.detail.flags
    });
  }
});

observer.observe({ entryTypes: ['gc'] });

for (let round = 0; round < 20; round += 1) {
  const temporary = Array.from(
    { length: 20_000 },
    (_, index) => ({ round, index, value: `item-${index}` })
  );

  temporary.length = 0;
  await delay(10);
}

await delay(100);
observer.disconnect();

console.log({
  gcCount: entries.length,
  totalDurationMs: entries.reduce(
    (total, entry) => total + entry.durationMs,
    0
  )
});
```

실제 서비스 요청 경로에 메모리 압력을 만들기 위한 코드를 추가하면 안 됩니다.
재현 실험은 로컬이나 격리된 테스트 환경에서 수행하고, 운영에서는 자연스럽게 발생한 성능 항목만 수집해야 합니다.

### H3. 강제 GC는 제한된 실험에서만 사용한다

Node.js를 `--expose-gc` 옵션으로 실행하면 `global.gc()`를 호출할 수 있습니다.
이는 계측 코드가 동작하는지 확인하는 제한된 테스트에는 도움이 되지만, 운영 트래픽에서 주기적으로 강제 GC를 실행하는 방식은 일시정지를 늘리고 런타임의 GC 판단을 방해할 수 있습니다.

```js
if (typeof global.gc !== 'function') {
  throw new Error('Run this test with node --expose-gc');
}

global.gc();
```

테스트 결과를 해석할 때도 강제 GC 시간과 실제 운영에서 자연스럽게 발생하는 GC 시간을 동일하게 취급하지 않아야 합니다.

## GC 종류별로 결과 집계하기

### H3. 숫자 종류를 고정된 라벨로 변환한다

GC 종류를 구분할 때는 `perf_hooks.constants`의 상수를 사용할 수 있습니다.
알 수 없는 값도 처리해 새 런타임이나 예상하지 못한 입력 때문에 집계 코드가 실패하지 않도록 합니다.

```js
import {
  constants,
  PerformanceObserver
} from 'node:perf_hooks';

function gcKindName(kind) {
  const names = [];

  if (kind & constants.NODE_PERFORMANCE_GC_MAJOR) names.push('major');
  if (kind & constants.NODE_PERFORMANCE_GC_MINOR) names.push('minor');
  if (kind & constants.NODE_PERFORMANCE_GC_INCREMENTAL) {
    names.push('incremental');
  }
  if (kind & constants.NODE_PERFORMANCE_GC_WEAKCB) names.push('weakcb');

  return names.length > 0 ? names.join('+') : 'unknown';
}

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntriesByType('gc')) {
    metrics.observe('node_gc_duration_ms', entry.duration, {
      kind: gcKindName(entry.detail.kind)
    });
  }
});

observer.observe({ entryTypes: ['gc'] });
```

고정된 소수의 GC 종류만 라벨로 사용하면 메트릭 카디널리티를 예측하기 쉽습니다.
프로세스 ID, 요청 ID, 사용자 ID처럼 값의 종류가 계속 늘어나는 필드는 메트릭 라벨에 넣지 않는 편이 안전합니다.

### H3. 호출별 로그보다 구간별 요약을 우선한다

GC가 자주 발생하는 프로세스에서 모든 항목을 로그로 남기면 로그 양과 저장 비용이 커질 수 있습니다.
관찰 구간마다 횟수, 총 실행 시간, 최대값, 백분위를 집계하면 적은 데이터로 추세를 확인할 수 있습니다.

```js
function summarize(durationsMs) {
  if (durationsMs.length === 0) {
    return { count: 0, totalMs: 0, maxMs: 0, p95Ms: 0 };
  }

  const sorted = [...durationsMs].sort((a, b) => a - b);
  const p95Index = Math.ceil(sorted.length * 0.95) - 1;

  return {
    count: sorted.length,
    totalMs: sorted.reduce((total, value) => total + value, 0),
    maxMs: sorted.at(-1),
    p95Ms: sorted[p95Index]
  };
}
```

표본이 적은 구간의 백분위는 쉽게 흔들립니다.
백분위를 기록할 때는 반드시 표본 수를 함께 남기고, 짧은 구간 하나보다 여러 구간의 추세를 비교해야 합니다.

## 운영 모니터링에 적용하기

### H3. GC 시간과 메모리 상태를 함께 기록한다

GC 실행 시간이 길어진 원인을 좁히려면 같은 구간의 힙 사용량과 외부 메모리도 확인해야 합니다.

```js
function memorySnapshot() {
  const usage = process.memoryUsage();

  return {
    heapUsedBytes: usage.heapUsed,
    heapTotalBytes: usage.heapTotal,
    externalBytes: usage.external,
    arrayBuffersBytes: usage.arrayBuffers
  };
}
```

메모리 사용량은 순간값이므로 한 번의 스냅샷만으로 누수를 판단하면 안 됩니다.
정상 트래픽의 기준선을 만들고, GC 이후에도 힙 사용량의 저점이 계속 높아지는지 장기간 확인해야 합니다.

메모리 압력을 빠르게 확인하는 방법은 [Node.js process.availableMemory 가이드](/development/blog/seo/2026/05/21/nodejs-process-availablememory-memory-pressure-guide.html)와 함께 적용할 수 있습니다.

### H3. 요청 지연과 이벤트 루프 지표를 같은 시간축에서 비교한다

GC 일시정지와 요청 지연이 같은 시점에 반복해서 발생하면 객체 생성량, 캐시 크기, 큰 입력 처리 경로를 우선 조사할 근거가 됩니다.
반대로 GC 시간은 안정적인데 요청 지연만 늘었다면 외부 의존성이나 큐 대기 같은 다른 원인을 살펴봐야 합니다.

운영 대시보드에서는 다음 지표를 함께 비교하는 편이 좋습니다.

- GC 종류별 실행 횟수와 총 실행 시간
- GC 실행 시간의 p95·p99·최대값
- 힙 사용량과 프로세스 RSS
- 요청 응답 시간과 오류율
- 이벤트 루프 활용률과 지연
- 배포, 설정, 트래픽 변화 시점

함수 단위의 실행 시간이 필요한 경우 [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)를 연결해 병목 후보를 더 좁힐 수 있습니다.

### H3. 관찰자 생명주기를 명확히 관리한다

상시 모니터링이라면 애플리케이션 시작 시 관찰자를 한 번 등록하고 종료 절차에서 해제합니다.
테스트나 임시 진단에서는 `finally` 블록에서 `disconnect()`를 호출해 다음 실행에 관찰자가 남지 않도록 합니다.

```js
const observer = new PerformanceObserver(handleGcEntries);
observer.observe({ entryTypes: ['gc'] });

try {
  await runScenario();
} finally {
  observer.disconnect();
}
```

관찰자 콜백 안에서 무거운 동기 처리나 대량 로그를 실행하면 계측 코드 자체가 이벤트 루프 지연을 만들 수 있습니다.
콜백에서는 필요한 값만 빠르게 집계하고, 전송과 저장은 별도 경로에서 처리하는 것이 좋습니다.

## 해석할 때 주의할 점

### H3. GC 발생 자체는 장애가 아니다

GC는 JavaScript 런타임의 정상적인 메모리 관리 과정입니다.
짧은 GC가 자주 관찰된다는 사실만으로 문제라고 판단하면 불필요한 최적화로 이어질 수 있습니다.

사용자 지연, 처리량 감소, 메모리 기준선 변화와 함께 나타나는지 확인해야 합니다.
특히 정상 시간대와 장애 시간대의 분포를 비교하면 조사 우선순위를 정하기 쉽습니다.

### H3. 관찰 결과만으로 메모리 누수를 확정할 수 없다

GC 시간이 늘거나 major GC가 발생해도 그 원인이 반드시 누수인 것은 아닙니다.
일시적인 대량 작업, 트래픽 증가, 캐시 워밍, 큰 응답 생성도 같은 현상을 만들 수 있습니다.

누수가 의심되면 충분한 기간의 힙 추세를 확인하고, 제한된 환경에서 힙 스냅샷이나 진단 보고서를 사용해 유지되는 객체를 조사해야 합니다.
[Node.js process.report 가이드](/development/blog/seo/2026/05/29/nodejs-process-report-production-diagnostics-guide.html)는 프로세스 상태를 진단 자료로 남기는 방법을 설명합니다.

### H3. 로그에 객체 내용이나 민감정보를 넣지 않는다

GC 분석에는 사용자 요청 본문, 토큰, 이메일, 객체 전체 내용이 필요하지 않습니다.
메트릭과 로그에는 시간, 종류, 횟수, 메모리 크기처럼 집계 가능한 최소 정보만 남깁니다.

```js
logger.info({
  event: 'gc-summary',
  intervalSeconds: 60,
  count: summary.count,
  p95Ms: Number(summary.p95Ms.toFixed(2)),
  maxMs: Number(summary.maxMs.toFixed(2))
});
```

## 발행 및 적용 전 체크리스트

### H3. 재현성과 운영 비용을 함께 확인한다

GC 모니터링을 적용하기 전에는 다음 항목을 확인할 수 있습니다.

- `gc` 성능 항목을 정상적으로 수집하는가?
- `entry.duration`을 밀리초 단위로 처리하는가?
- 알 수 없는 GC 종류도 안전하게 집계하는가?
- 표본 수와 백분위를 함께 기록하는가?
- 관찰자 콜백이 무거운 동기 작업을 하지 않는가?
- 로그와 메트릭에 개인정보나 객체 내용이 포함되지 않는가?
- 요청 지연, 메모리, 이벤트 루프 지표를 함께 비교하는가?

## 자주 묻는 질문

### H3. GC 횟수가 많으면 메모리 누수인가요?

아닙니다.
객체를 많이 생성하는 정상 작업이나 트래픽 증가도 GC 횟수를 늘릴 수 있습니다.
GC 이후 힙 사용량의 저점이 장기간 계속 높아지는지, 처리량과 지연이 함께 나빠지는지 확인해야 합니다.

### H3. PerformanceObserver 콜백은 언제 실행되나요?

관찰 대상 성능 항목이 생성된 뒤 콜백으로 전달됩니다.
콜백 자체도 애플리케이션과 같은 이벤트 루프에서 실행되므로, 내부 작업을 작고 빠르게 유지해야 합니다.

### H3. 운영에서 global.gc를 호출해도 되나요?

정기적인 운영 전략으로 권장하기 어렵습니다.
강제 GC는 예측하기 어려운 일시정지를 만들 수 있으므로 제한된 진단 실험에만 사용하고, 운영 문제는 객체 할당 패턴과 메모리 정책을 조사해 해결하는 편이 좋습니다.

## 마무리

Node.js의 `PerformanceObserver`를 사용하면 GC 실행 시간과 종류를 애플리케이션 내부에서 관찰할 수 있습니다.
종류별 횟수와 일시정지 분포를 집계하고 메모리·요청 지연·이벤트 루프 지표와 함께 비교하면 간헐적인 성능 저하의 원인을 더 구체적으로 좁힐 수 있습니다.

다만 GC는 정상적인 런타임 동작이며, 관찰 결과 하나만으로 장애나 메모리 누수를 단정해서는 안 됩니다.
계측 비용과 데이터 양을 제한하고, 민감정보 없이 장기 추세와 사용자 영향을 중심으로 해석하는 것이 중요합니다.

관련 글:

- [Node.js performance.eventLoopUtilization 가이드: 이벤트 루프 부하 측정하기](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)
- [Node.js performance.timerify 가이드: 함수 실행 시간과 백분위 측정하기](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)
- [Node.js process.availableMemory 가이드: 메모리 압력 확인하기](/development/blog/seo/2026/05/21/nodejs-process-availablememory-memory-pressure-guide.html)

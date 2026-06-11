---
layout: post
title: "Node.js performance.timerify 가이드: 함수 실행 시간과 백분위 측정하기"
date: 2026-06-11 20:00:00 +0900
lang: ko
translation_key: nodejs-performance-timerify-function-duration-guide
permalink: /development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html
alternates:
  ko: /development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html
  x_default: /development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, timerify, performanceobserver, histogram, monitoring]
description: "Node.js performance.timerify()로 동기·비동기 함수 실행 시간을 측정하고 PerformanceObserver와 createHistogram으로 호출별 기록과 백분위를 수집하는 방법을 예제로 설명합니다."
---

Node.js 애플리케이션이 느려졌다는 사실을 확인한 뒤에는 어떤 함수가 시간을 사용했는지 좁혀야 합니다.
모든 코드에 직접 시작·종료 시각을 기록하면 측정 코드가 반복되고, 예외나 비동기 처리에서 종료 기록을 빠뜨리기도 쉽습니다.

`performance.timerify()`는 기존 함수를 감싸 호출 시간을 측정할 수 있는 `node:perf_hooks` API입니다.
호출별 측정 결과는 `PerformanceObserver`로 관찰하고, `createHistogram()`을 연결하면 여러 호출의 p50·p95·p99 같은 백분위도 계산할 수 있습니다.

이 글에서는 `performance.timerify()`로 동기 함수와 비동기 함수를 측정하는 방법, 히스토그램으로 결과를 요약하는 방법, 운영 코드에 적용할 때의 주의점을 정리합니다.
서비스 전체의 이벤트 루프 부하를 먼저 확인하려면 [Node.js performance.eventLoopUtilization 가이드](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)도 함께 참고하세요.

## performance.timerify가 필요한 이유

### H3. 함수 경계에서 실행 시간을 일관되게 측정한다

함수 실행 시간을 직접 측정할 때는 보통 다음처럼 시작 시각과 종료 시각의 차이를 계산합니다.

```js
import { performance } from 'node:perf_hooks';

const startedAt = performance.now();
const result = await buildReport();
const durationMs = performance.now() - startedAt;

console.log({ durationMs });
```

간단한 진단에는 충분하지만 여러 함수에 반복 적용하면 측정 코드가 핵심 로직에 섞입니다.
예외가 발생하는 경로와 비동기 함수의 완료 시점까지 빠짐없이 처리하려면 코드도 더 복잡해집니다.

`performance.timerify()`는 원본 함수를 감싼 새 함수를 반환합니다.
감싼 함수를 호출하면 Node.js가 실행 시간을 측정하고 `function` 유형의 성능 항목을 생성합니다.

다음과 같은 상황에서 유용합니다.

- 특정 변환 함수가 큰 입력에서 느려지는지 확인할 때
- 캐시 적용 전후의 함수 실행 시간을 비교할 때
- 비동기 서비스 함수의 지연 분포를 임시로 관찰할 때
- 성능 회귀가 의심되는 함수의 기준선을 만들 때

다만 함수 호출 시간을 측정하는 도구이지, 느린 원인을 자동으로 분석하는 프로파일러는 아닙니다.

### H3. 원본 함수 대신 감싼 함수를 호출해야 한다

기본 사용법은 측정할 함수를 `performance.timerify()`에 전달하고 반환된 함수를 호출하는 것입니다.

```js
import {
  performance,
  PerformanceObserver
} from 'node:perf_hooks';

function normalizeRecords(records) {
  return records.map((record) => ({
    id: record.id,
    name: record.name.trim().toLowerCase()
  }));
}

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log({
      name: entry.name,
      durationMs: Number(entry.duration.toFixed(2))
    });
  }
});

observer.observe({ entryTypes: ['function'] });

const timedNormalizeRecords = performance.timerify(normalizeRecords);

timedNormalizeRecords([
  { id: 1, name: ' Alpha ' },
  { id: 2, name: ' Beta ' }
]);
```

원본 `normalizeRecords()`를 직접 호출하면 측정 항목이 생성되지 않습니다.
측정하려는 호출 경로가 실제로 감싼 함수를 사용하도록 연결해야 합니다.

## PerformanceObserver로 호출별 결과 수집하기

### H3. function 성능 항목만 구독한다

`PerformanceObserver`는 생성된 성능 항목을 비동기 콜백으로 전달합니다.
`entryTypes: ['function']`을 지정하면 `timerify()`가 만든 함수 측정 항목을 관찰할 수 있습니다.

```js
import {
  performance,
  PerformanceObserver
} from 'node:perf_hooks';

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntriesByType('function')) {
    metrics.observe('function_duration_ms', entry.duration, {
      functionName: entry.name
    });
  }
});

observer.observe({ entryTypes: ['function'] });

export const timedRenderSummary = performance.timerify(renderSummary);
```

함수 이름은 측정값을 구분하는 데 편리하지만, 이름의 종류가 무제한으로 늘어나면 메트릭 시스템의 카디널리티가 커질 수 있습니다.
운영에서는 허용한 함수 이름만 라벨로 사용하거나, 측정 대상별로 고정된 이름을 매핑하는 편이 안전합니다.

### H3. 관찰이 끝나면 observer를 해제한다

일시적인 진단이나 테스트에서 관찰을 시작했다면 완료 후 `disconnect()`를 호출해 구독을 해제합니다.

```js
const observer = new PerformanceObserver(handleEntries);
observer.observe({ entryTypes: ['function'] });

try {
  await runScenario();
} finally {
  observer.disconnect();
}
```

테스트마다 새 관찰자를 만들고 해제하지 않으면 같은 항목을 여러 번 처리하거나 테스트 간 상태가 섞일 수 있습니다.
상시 모니터링이라면 애플리케이션 시작 시 한 번 등록하고, 종료 절차에서 해제하는 구조가 관리하기 쉽습니다.

## 비동기 함수 실행 시간 측정하기

### H3. Promise가 완료될 때까지 측정한다

`performance.timerify()`로 비동기 함수를 감싸면 반환된 Promise가 완료될 때까지의 시간이 측정됩니다.
따라서 데이터베이스나 외부 API를 기다리는 서비스 함수의 전체 지연을 관찰할 수 있습니다.

```js
import {
  performance,
  PerformanceObserver
} from 'node:perf_hooks';
import { setTimeout as delay } from 'node:timers/promises';

async function fetchProfile(userId) {
  await delay(40);
  return { userId, status: 'active' };
}

const observer = new PerformanceObserver((list) => {
  const [entry] = list.getEntries();

  console.log({
    name: entry.name,
    durationMs: Number(entry.duration.toFixed(2))
  });
});

observer.observe({ entryTypes: ['function'] });

const timedFetchProfile = performance.timerify(fetchProfile);
await timedFetchProfile('user-example');

await new Promise((resolve) => setImmediate(resolve));
observer.disconnect();
```

이 측정값에는 함수 내부의 CPU 실행 시간뿐 아니라 I/O 대기 시간도 포함됩니다.
값이 크다고 해서 해당 함수가 CPU를 오래 사용했다고 단정하면 안 됩니다.

### H3. 계층별 지연을 함께 측정해야 원인을 좁힐 수 있다

서비스 함수가 데이터베이스 조회와 외부 API 호출을 모두 포함한다면 최상위 함수의 실행 시간만으로는 어느 의존성이 느린지 알 수 없습니다.
요청 전체, 주요 서비스 함수, 외부 의존성처럼 필요한 경계에서 각각 시간을 측정하면 원인을 더 빠르게 좁힐 수 있습니다.

측정 경계를 지나치게 많이 추가하면 오버헤드와 데이터 양이 늘어납니다.
먼저 사용자 지연에 영향이 큰 경로를 선택하고, 실제로 조사에 쓰이는 항목만 유지하는 것이 좋습니다.

## createHistogram으로 백분위 측정하기

### H3. 호출별 로그 대신 분포를 요약한다

호출마다 실행 시간을 로그로 남기면 호출량이 많은 서비스에서 저장 비용이 커집니다.
`createHistogram()`으로 만든 히스토그램을 `timerify()`에 연결하면 관찰 구간의 실행 시간 분포를 요약할 수 있습니다.

```js
import {
  createHistogram,
  performance
} from 'node:perf_hooks';

function calculateScore(values) {
  return values.reduce((total, value) => total + value, 0);
}

const histogram = createHistogram();
const timedCalculateScore = performance.timerify(calculateScore, {
  histogram
});

for (let index = 0; index < 1_000; index += 1) {
  timedCalculateScore([index, index + 1, index + 2]);
}

console.log({
  count: histogram.count,
  minMs: histogram.min / 1e6,
  p50Ms: histogram.percentile(50) / 1e6,
  p95Ms: histogram.percentile(95) / 1e6,
  maxMs: histogram.max / 1e6
});
```

히스토그램 값은 나노초 단위이므로 밀리초로 표시하려면 `1e6`으로 나눕니다.
지연 시간 백분위의 의미와 히스토그램 설정은 [Node.js perf_hooks.createHistogram 가이드](/development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html)에서 더 자세히 확인할 수 있습니다.

### H3. 측정 구간을 명확히 구분한다

배포 전후나 처리 방식 변경 전후를 비교하려면 같은 히스토그램에 모든 결과를 계속 누적하지 않는 편이 좋습니다.
관찰 구간을 정하고 결과를 내보낸 뒤 `reset()`해 다음 구간을 시작할 수 있습니다.

```js
function snapshotAndReset(histogram) {
  const snapshot = {
    count: histogram.count,
    p50Ms: histogram.percentile(50) / 1e6,
    p95Ms: histogram.percentile(95) / 1e6,
    p99Ms: histogram.percentile(99) / 1e6
  };

  histogram.reset();
  return snapshot;
}
```

호출 수가 매우 적은 구간의 백분위는 쉽게 흔들립니다.
백분위만 보지 말고 호출 수, 입력 크기, 오류율, 처리량도 함께 확인해야 합니다.

## 운영 코드에 적용하는 패턴

### H3. 측정 대상과 활성화 조건을 제한한다

모든 함수를 항상 감싸기보다 성능 회귀 위험이 크거나 사용자 지연에 직접 영향을 주는 함수부터 선택하는 것이 좋습니다.
환경 변수나 설정으로 진단 기능을 켜고 끌 수 있게 만들면 문제 조사 기간에만 상세 측정을 활성화할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

export function optionallyTimerify(fn, enabled) {
  return enabled ? performance.timerify(fn) : fn;
}

export const parseCatalog = optionallyTimerify(
  parseCatalogData,
  process.env.ENABLE_FUNCTION_TIMING === 'true'
);
```

설정값에는 인증 정보나 사용자 데이터를 넣지 않아야 합니다.
또한 진단 기능을 켠 상태와 끈 상태에서 처리량과 지연을 비교해 측정 오버헤드를 확인해야 합니다.

### H3. 입력값과 반환값을 성능 로그에 그대로 남기지 않는다

함수 실행 시간을 조사하는 데 요청 본문, 이메일, 토큰, 전체 쿼리 문자열이 필요한 경우는 드뭅니다.
로그와 메트릭에는 함수 식별자, 실행 시간, 결과 상태, 제한된 크기 구간처럼 필요한 최소 정보만 남깁니다.

```js
logger.info({
  event: 'function-timing-summary',
  functionName: 'build-search-index',
  count: snapshot.count,
  p95Ms: Number(snapshot.p95Ms.toFixed(2))
});
```

성능 측정이 개인정보나 민감정보 수집 경로가 되지 않도록 필드 목록을 고정하고 검토해야 합니다.

### H3. 타이밍 결과와 오류율을 함께 본다

빠르게 실패한 호출은 실행 시간이 짧게 기록될 수 있습니다.
따라서 평균 실행 시간이 줄었다는 사실만으로 성능이 개선됐다고 판단하면 오류 증가를 놓칠 수 있습니다.

운영에서는 다음 항목을 함께 확인하는 편이 좋습니다.

- 함수 호출 수와 성공·실패 수
- p50, p95, p99 실행 시간
- 입력 크기 구간과 처리량
- 요청 전체 응답 시간
- 이벤트 루프 활용률과 CPU 사용률
- 배포 또는 설정 변경 시점

## performance.timerify 사용 시 주의할 점

### H3. 측정값은 함수 내부 원인을 알려 주지 않는다

`timerify()`는 함수 경계의 총 실행 시간을 보여 줍니다.
어떤 코드 줄이 느린지, CPU 연산과 I/O 대기가 각각 얼마나 차지하는지는 알려 주지 않습니다.

함수 실행 시간이 길다는 사실을 확인한 뒤에는 CPU 프로파일, 데이터베이스 쿼리 시간, 외부 API 추적, 이벤트 루프 지표를 사용해 원인을 좁혀야 합니다.

### H3. 작은 함수의 대량 측정은 오버헤드가 될 수 있다

매우 짧고 자주 호출되는 함수는 측정 비용이 실제 함수 비용에 비해 크게 보일 수 있습니다.
성능 테스트에서는 측정 전후의 처리량을 비교하고, 운영에서는 표본 수집이나 제한된 진단 기간을 고려해야 합니다.

### H3. 원본 함수의 정체성과 호출 방식 변화를 확인한다

`performance.timerify()`는 새 함수를 반환하므로 원본 함수와 참조가 다릅니다.
함수 참조를 키로 사용하는 코드, 엄격한 동일성 비교, 메타데이터를 함수 객체에 직접 붙이는 코드가 있다면 영향이 없는지 확인해야 합니다.

객체 메서드를 감쌀 때는 해당 메서드가 `this`에 의존하는지도 테스트해야 합니다.
호출 방식이 바뀌면 성능 측정과 무관한 동작 오류가 생길 수 있습니다.

## 테스트와 검증 체크리스트

### H3. 절대 시간보다 동일 조건의 상대 변화를 비교한다

개발 머신과 CI는 하드웨어, 백그라운드 작업, 런타임 상태가 다르므로 짧은 함수의 실행 시간을 고정된 숫자로 단정하는 테스트는 불안정할 수 있습니다.
같은 환경과 입력에서 변경 전후를 반복 측정하고 충분한 호출 수의 분포를 비교하는 편이 좋습니다.

발행 또는 운영 적용 전에는 다음 항목을 확인할 수 있습니다.

- 실제 호출 경로가 감싼 함수를 사용하는가?
- 비동기 함수의 완료 시점까지 측정되는가?
- 관찰자를 필요한 시점에 해제하는가?
- 히스토그램의 나노초 값을 올바른 단위로 변환하는가?
- 측정 활성화 전후의 처리량과 지연을 비교했는가?
- 로그와 메트릭에 민감정보가 포함되지 않는가?

## 자주 묻는 질문

### H3. performance.timerify와 performance.now는 무엇이 다른가요?

`performance.now()`는 원하는 두 시점의 차이를 직접 계산할 때 사용합니다.
`performance.timerify()`는 함수를 감싸 호출 시간을 성능 항목이나 히스토그램으로 수집할 때 편리합니다.
한두 지점의 임시 측정에는 `performance.now()`가 단순하고, 같은 함수의 반복 호출을 일관되게 관찰하려면 `timerify()`가 유용합니다.

### H3. timerify 결과만으로 CPU 병목을 찾을 수 있나요?

아닙니다.
비동기 함수의 측정값에는 I/O 대기 시간도 포함될 수 있고, 긴 실행 시간만으로 어느 코드가 CPU를 사용했는지 알 수 없습니다.
CPU 병목이 의심되면 프로파일과 이벤트 루프 지표를 함께 확인해야 합니다.

### H3. PerformanceObserver와 histogram을 모두 사용해야 하나요?

반드시 그렇지는 않습니다.
개별 호출의 이름과 시간을 확인해야 하면 `PerformanceObserver`가 유용하고, 많은 호출의 분포를 낮은 데이터 양으로 요약하려면 히스토그램이 적합합니다.
조사 목적에 따라 하나만 사용하거나 제한된 기간에 함께 사용할 수 있습니다.

## 마무리

`performance.timerify()`는 Node.js 함수의 실행 시간을 기존 로직과 분리해 측정할 수 있는 실용적인 도구입니다.
`PerformanceObserver`로 호출별 결과를 확인하고 `createHistogram()`으로 백분위를 요약하면 성능 회귀와 지연 분포를 더 구체적으로 관찰할 수 있습니다.

다만 측정값은 느린 원인을 직접 설명하지 않으며, 과도한 계측은 오버헤드와 데이터 비용을 늘릴 수 있습니다.
사용자 지연에 중요한 함수부터 제한적으로 적용하고, 오류율·처리량·이벤트 루프·프로파일 결과와 함께 해석해야 합니다.

관련 글:

- [Node.js performance.eventLoopUtilization 가이드: 이벤트 루프 부하 측정하기](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)
- [Node.js perf_hooks.createHistogram 가이드: 지연 시간 백분위수 측정하기](/development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html)
- [Node.js process.getActiveResourcesInfo 가이드: 프로세스 종료 지연 원인 찾기](/development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html)

---
layout: post
title: "Node.js performance.mark·measure 가이드: 코드 구간 실행 시간 측정하기"
date: 2026-06-12 20:00:00 +0900
lang: ko
translation_key: nodejs-performance-mark-measure-user-timing-guide
permalink: /development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html
alternates:
  ko: /development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html
  x_default: /development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, mark, measure, usertiming, observability]
description: "Node.js performance.mark()와 performance.measure()로 코드 구간 실행 시간을 측정하고 PerformanceObserver로 수집하는 방법을 재현 가능한 예제와 운영 체크리스트로 설명합니다."
---

Node.js 애플리케이션에서 느린 작업을 찾으려면 요청 전체 시간뿐 아니라 내부 단계별 시간도 확인해야 합니다.
데이터 조회, 변환, 렌더링처럼 여러 단계가 이어지는 작업은 전체 실행 시간만으로 병목 지점을 구분하기 어렵습니다.

`node:perf_hooks`의 `performance.mark()`와 `performance.measure()`를 사용하면 코드의 시작점과 종료점에 이름을 붙이고 두 지점 사이의 실행 시간을 측정할 수 있습니다.
측정 결과는 성능 타임라인에서 조회하거나 `PerformanceObserver`로 수집할 수 있어 임시 진단부터 제한적인 운영 계측까지 활용할 수 있습니다.

이 글에서는 mark와 measure의 기본 사용법, 비동기 작업의 단계별 측정, 관찰자를 이용한 결과 수집, 운영 적용 시 주의점을 정리합니다.
함수 전체의 실행 시간을 자동으로 측정하려면 [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)도 함께 참고하세요.

## performance.mark와 measure가 필요한 이유

### H3. 원하는 코드 경계를 직접 정의할 수 있다

`performance.timerify()`는 함수 호출 전체를 측정하는 데 편리합니다.
하지만 하나의 함수 안에 여러 처리 단계가 있거나 여러 함수에 걸친 업무 흐름을 측정하려면 시작점과 종료점을 직접 정하는 방식이 더 적합할 수 있습니다.

mark와 measure는 다음 상황에서 유용합니다.

- 데이터 조회와 변환 중 어느 단계가 느린지 비교할 때
- 캐시 조회부터 응답 생성까지 특정 업무 흐름을 측정할 때
- 배포 전후 같은 처리 구간의 실행 시간을 비교할 때
- 테스트에서 성능 기준선을 기록하고 회귀를 조사할 때

측정 지점을 많이 추가하면 이름 관리와 데이터 처리 비용도 늘어납니다.
사용자 지연에 영향을 주거나 실제 조사에 활용할 구간부터 선택하는 것이 좋습니다.

### H3. mark는 시점이고 measure는 구간이다

`performance.mark()`는 성능 타임라인에 이름이 있는 시점을 기록합니다.
`performance.measure()`는 두 mark 사이의 시간을 계산해 `measure` 성능 항목을 만듭니다.

```js
import { performance } from 'node:perf_hooks';

performance.mark('report:start');

await buildReport();

performance.mark('report:end');
performance.measure('report:total', 'report:start', 'report:end');

const [entry] = performance.getEntriesByName('report:total', 'measure');

console.log({
  name: entry.name,
  durationMs: Number(entry.duration.toFixed(2))
});
```

`entry.duration`은 밀리초 단위입니다.
이름은 `report:start`, `report:end`, `report:total`처럼 기능과 역할을 구분할 수 있는 고정된 규칙으로 관리하면 조회와 집계가 쉬워집니다.

## 단계별 실행 시간 측정하기

### H3. 하나의 작업을 여러 구간으로 나눈다

보고서 생성 작업이 조회, 변환, 렌더링 단계로 구성되어 있다면 각 경계에 mark를 추가해 단계별 시간을 측정할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

export async function createReport(reportId) {
  performance.mark('report:start');

  const records = await loadRecords(reportId);
  performance.mark('report:loaded');

  const summary = transformRecords(records);
  performance.mark('report:transformed');

  const output = await renderReport(summary);
  performance.mark('report:end');

  performance.measure('report:load', 'report:start', 'report:loaded');
  performance.measure(
    'report:transform',
    'report:loaded',
    'report:transformed'
  );
  performance.measure(
    'report:render',
    'report:transformed',
    'report:end'
  );
  performance.measure('report:total', 'report:start', 'report:end');

  return output;
}
```

이 구조를 사용하면 전체 시간이 늘어난 시점에 어떤 단계의 변화가 컸는지 비교할 수 있습니다.
다만 같은 고정 이름을 사용하는 작업이 동시에 실행되면 서로 다른 요청의 mark가 섞일 수 있으므로 동시 실행 환경에서는 별도 측정 전략이 필요합니다.

### H3. 동시 요청에서는 고유한 측정 이름을 신중하게 사용한다

요청마다 고유한 이름을 생성하면 mark 충돌을 피할 수 있지만, 요청 ID나 사용자 ID를 그대로 메트릭 라벨이나 로그에 저장하면 카디널리티와 개인정보 위험이 커집니다.

짧은 진단 실험에서는 내부에서만 사용하는 임시 식별자로 측정 이름을 분리하고, 외부로 내보내는 결과는 고정된 구간 이름으로 집계할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';
import { randomUUID } from 'node:crypto';

export async function measureReportJob(run) {
  const runKey = randomUUID();
  const startMark = `report:${runKey}:start`;
  const endMark = `report:${runKey}:end`;

  performance.mark(startMark);

  try {
    return await run();
  } finally {
    performance.mark(endMark);
    performance.measure('report:total', startMark, endMark);
    performance.clearMarks(startMark);
    performance.clearMarks(endMark);
  }
}
```

이 예제의 UUID는 측정 항목을 분리하기 위한 임시 값이며 사용자 정보와 연결하지 않습니다.
운영 메트릭에는 `report:total`처럼 종류가 제한된 이름만 사용해야 합니다.

## PerformanceObserver로 측정 결과 수집하기

### H3. measure 항목만 구독한다

성능 타임라인을 매번 직접 조회하는 대신 `PerformanceObserver`로 새로 생성된 measure 항목을 받을 수 있습니다.

```js
import {
  performance,
  PerformanceObserver
} from 'node:perf_hooks';

const allowedMeasures = new Set([
  'report:load',
  'report:transform',
  'report:render',
  'report:total'
]);

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntriesByType('measure')) {
    if (!allowedMeasures.has(entry.name)) continue;

    metrics.observe('operation_duration_ms', entry.duration, {
      operation: entry.name
    });
  }
});

observer.observe({ entryTypes: ['measure'] });
```

허용된 측정 이름만 메트릭 라벨로 사용하면 예상하지 못한 이름 때문에 시계열 수가 계속 늘어나는 문제를 줄일 수 있습니다.
관찰자 콜백에서는 무거운 동기 작업이나 호출별 대량 로그를 피하고 필요한 값만 빠르게 집계해야 합니다.

### H3. 관찰자 생명주기를 관리한다

테스트나 일시적인 진단에서는 작업이 끝난 뒤 `disconnect()`로 관찰자를 해제합니다.
상시 계측이라면 애플리케이션 시작 시 한 번 등록하고 종료 절차에서 정리하는 구조가 적합합니다.

```js
const observer = new PerformanceObserver(handleMeasures);
observer.observe({ entryTypes: ['measure'] });

try {
  await runScenario();
} finally {
  observer.disconnect();
}
```

테스트마다 관찰자를 새로 만들면서 해제하지 않으면 같은 항목을 여러 번 처리하거나 테스트 간 결과가 섞일 수 있습니다.

## 성능 타임라인 정리하기

### H3. 사용한 mark와 measure를 명시적으로 삭제한다

mark와 measure 항목은 성능 타임라인에 저장됩니다.
장시간 실행되는 프로세스에서 계속 새 항목을 생성하면 불필요한 메모리 사용과 조회 비용이 늘 수 있으므로 처리 후 정리해야 합니다.

```js
function clearReportTimings() {
  performance.clearMarks('report:start');
  performance.clearMarks('report:loaded');
  performance.clearMarks('report:transformed');
  performance.clearMarks('report:end');

  performance.clearMeasures('report:load');
  performance.clearMeasures('report:transform');
  performance.clearMeasures('report:render');
  performance.clearMeasures('report:total');
}
```

특정 이름을 전달하면 해당 이름의 항목만 삭제할 수 있습니다.
다른 기능의 계측 결과까지 지우지 않도록 측정 기능이 소유한 이름만 정리하는 편이 안전합니다.

### H3. 예외가 발생해도 정리 코드가 실행되게 한다

측정 대상 작업이 실패하면 종료 mark와 정리 코드가 실행되지 않을 수 있습니다.
`try`와 `finally`를 사용하면 성공과 실패 모두에서 필요한 정리를 수행할 수 있습니다.

```js
performance.mark('sync:start');

try {
  await synchronizeData();
} finally {
  performance.mark('sync:end');
  performance.measure('sync:total', 'sync:start', 'sync:end');
  performance.clearMarks('sync:start');
  performance.clearMarks('sync:end');
}
```

실패한 작업과 성공한 작업을 별도로 비교해야 한다면 측정 이름이나 별도 고정 속성으로 결과를 구분할 수 있습니다.
오류 메시지 전체나 입력 데이터는 성능 메트릭에 포함하지 않는 것이 좋습니다.

## 운영 환경에 적용하는 방법

### H3. 전체 요청보다 병목 후보부터 측정한다

모든 코드 경계에 mark를 추가하면 계측 코드와 데이터 양이 빠르게 늘어납니다.
먼저 응답 지연이 크거나 최근 변경된 경로를 선택하고, 측정 결과가 실제 의사결정에 도움이 되는지 확인해야 합니다.

운영에서는 다음 정보를 함께 비교하면 원인을 더 구체적으로 좁힐 수 있습니다.

- 구간별 실행 시간의 p50, p95, p99
- 요청 전체 응답 시간과 오류율
- 이벤트 루프 활용률과 지연
- GC 일시정지 시간과 메모리 사용량
- 외부 API와 데이터베이스의 응답 시간
- 배포 및 설정 변경 시점

이벤트 루프 부하를 함께 확인하는 방법은 [Node.js performance.eventLoopUtilization 가이드](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)에서 확인할 수 있습니다.

### H3. detail에는 민감정보를 넣지 않는다

mark와 measure는 추가 정보를 담는 `detail` 옵션을 사용할 수 있지만 요청 본문, 인증 토큰, 이메일, 사용자 식별자 같은 민감정보를 넣으면 안 됩니다.
성능 분석에는 기능명, 처리 결과 종류, 제한된 상태값처럼 종류가 고정된 정보만 사용합니다.

```js
performance.mark('export:start', {
  detail: {
    format: 'csv',
    source: 'scheduled-job'
  }
});
```

`detail` 객체가 진단 도구나 로그로 전달될 가능성을 고려해 공개되어도 문제가 없는 최소 정보만 기록해야 합니다.

### H3. 단일 측정값보다 분포와 추세를 본다

한 번의 실행 시간이 길었다고 바로 성능 회귀로 판단하기는 어렵습니다.
입력 크기, 캐시 상태, 시스템 부하, 외부 의존성 상태에 따라 값이 달라질 수 있기 때문입니다.

구간별 실행 시간을 일정 기간 집계하고 호출 수와 백분위를 함께 확인하면 일반적인 성능과 드문 지연 급증을 구분하기 쉽습니다.
지연 분포를 히스토그램으로 요약하는 방법은 [Node.js perf_hooks.createHistogram 가이드](/development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html)를 참고하세요.

## 발행 및 적용 전 체크리스트

### H3. 측정 정확도와 운영 비용을 함께 확인한다

- 시작 mark와 종료 mark의 이름이 정확히 연결되는가?
- 동시 실행 작업의 mark가 서로 섞이지 않는가?
- 예외가 발생해도 필요한 measure와 정리가 수행되는가?
- 사용한 mark와 measure를 적절히 삭제하는가?
- 관찰자 콜백이 무거운 동기 작업을 하지 않는가?
- 메트릭 라벨의 종류가 제한되어 있는가?
- detail, 로그, 메트릭에 민감정보가 포함되지 않는가?
- 단일 값이 아니라 호출 수와 지연 분포를 함께 보는가?

## 자주 묻는 질문

### H3. performance.now로 직접 측정하는 것과 무엇이 다른가요?

`performance.now()`로 시작과 종료 시각의 차이를 직접 계산할 수도 있습니다.
mark와 measure는 이름이 있는 측정 지점을 성능 타임라인에 남기고 `PerformanceObserver`와 연결할 수 있어 여러 단계의 측정 결과를 일관된 방식으로 수집할 때 편리합니다.

### H3. mark와 measure를 많이 만들면 자동으로 정리되나요?

자동 정리에 의존하지 않는 편이 좋습니다.
장시간 실행되는 프로세스에서는 `clearMarks()`와 `clearMeasures()`를 사용해 더 이상 필요하지 않은 항목을 명시적으로 삭제해야 합니다.

### H3. measure 시간이 길면 CPU 병목인가요?

반드시 그렇지는 않습니다.
측정 구간에 비동기 I/O가 포함되면 데이터베이스, 네트워크, 큐 대기 시간도 duration에 포함됩니다.
CPU 프로파일, 외부 의존성 시간, 이벤트 루프와 GC 지표를 함께 확인해야 원인을 구분할 수 있습니다.

## 마무리

Node.js의 `performance.mark()`와 `performance.measure()`를 사용하면 애플리케이션의 의미 있는 코드 경계를 직접 정의하고 단계별 실행 시간을 측정할 수 있습니다.
`PerformanceObserver`와 연결하면 측정 결과를 일관되게 수집하고 전체 지연이 늘어난 원인을 세부 단계로 좁힐 수 있습니다.

운영에서는 측정 지점과 이름의 종류를 제한하고, 동시 실행 충돌과 타임라인 정리를 고려해야 합니다.
민감정보 없이 호출 수와 지연 분포를 함께 관찰하면 계측 비용을 관리하면서 성능 회귀를 조사하는 데 활용할 수 있습니다.

관련 글:

- [Node.js performance.timerify 가이드: 함수 실행 시간과 백분위 측정하기](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)
- [Node.js performance.eventLoopUtilization 가이드: 이벤트 루프 부하 측정하기](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)
- [Node.js perf_hooks.createHistogram 가이드: 지연 시간 백분위 측정하기](/development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html)

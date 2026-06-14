---
layout: post
title: "Node.js performance.getEntriesByType 가이드: 성능 타임라인 조회하고 정리하기"
date: 2026-06-14 20:00:00 +0900
lang: ko
translation_key: nodejs-performance-getentriesbytype-timeline-query-guide
permalink: /development/blog/seo/2026/06/14/nodejs-performance-getentriesbytype-timeline-query-guide.html
alternates:
  ko: /development/blog/seo/2026/06/14/nodejs-performance-getentriesbytype-timeline-query-guide.html
  x_default: /development/blog/seo/2026/06/14/nodejs-performance-getentriesbytype-timeline-query-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, perf-hooks, performanceentry, monitoring, observability]
description: "Node.js performance.getEntriesByType()으로 mark와 measure 성능 항목을 유형별 조회하고, 필터링·요약·테스트·타임라인 정리를 안전하게 수행하는 방법을 설명합니다."
---

Node.js 애플리케이션에서 `performance.mark()`와 `performance.measure()`로 구간을 측정하면 결과는 성능 타임라인에 `PerformanceEntry`로 남습니다.
측정값을 생성하는 것만으로는 병목을 찾기 어렵고, 원하는 유형의 항목을 조회해 비교하거나 요약하는 과정이 필요합니다.

`node:perf_hooks`의 `performance.getEntriesByType()`은 현재 성능 타임라인에서 지정한 유형의 항목을 배열로 반환합니다.
이를 활용하면 `measure` 항목만 모아 느린 구간을 찾고, 테스트에서 예상 측정값을 검증하며, 처리 후 타임라인을 명시적으로 정리할 수 있습니다.

이 글에서는 `performance.getEntriesByType()`의 기본 사용법, 조회 결과 필터링과 요약, `PerformanceObserver`와의 역할 차이, 장기 실행 프로세스에서의 정리 방법을 정리합니다.
측정 항목을 만드는 방법부터 확인하려면 [Node.js performance.mark·measure 가이드](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)를 함께 참고하세요.

## performance.getEntriesByType이 필요한 이유

### H3. 성능 항목을 유형별로 분리한다

성능 타임라인에는 `mark`, `measure` 등 서로 다른 의미를 가진 항목이 함께 존재할 수 있습니다.
`performance.getEntriesByType('measure')`를 호출하면 측정 구간 결과만 가져올 수 있어 후속 처리 코드가 단순해집니다.

```js
import { performance } from 'node:perf_hooks';

performance.mark('report:start');
await Promise.resolve();
performance.mark('report:end');
performance.measure('report:total', 'report:start', 'report:end');

const measures = performance.getEntriesByType('measure');

for (const entry of measures) {
  console.log({
    name: entry.name,
    durationMs: Number(entry.duration.toFixed(2))
  });
}
```

반환되는 값은 `PerformanceEntry` 객체의 배열입니다.
각 항목의 `name`, `entryType`, `startTime`, `duration`을 사용해 측정 결과를 비교할 수 있습니다.

### H3. 현재 타임라인의 스냅샷을 조회한다

`getEntriesByType()`은 호출 시점까지 타임라인에 기록된 항목을 조회하는 방식입니다.
앞으로 생성될 항목을 지속적으로 받는 구독 기능은 아니므로, 실시간 수집이 필요하면 `PerformanceObserver`를 사용해야 합니다.

조회 방식은 다음 상황에 적합합니다.

- 하나의 작업이 끝난 뒤 관련 측정값을 한 번에 요약할 때
- 테스트에서 생성된 mark와 measure를 검증할 때
- 진단 명령이 실행된 시점의 타임라인 상태를 확인할 때
- 수집한 항목을 처리한 뒤 명시적으로 정리할 때

## 기본 조회와 필터링

### H3. measure 항목을 이름으로 추가 필터링한다

같은 유형 안에서도 기능별 접두사를 정하면 필요한 항목만 골라낼 수 있습니다.
이름은 사용자 ID나 요청 URL처럼 종류가 계속 늘어나는 값 대신 제한된 고정 문자열을 사용하는 편이 안전합니다.

```js
import { performance } from 'node:perf_hooks';

function getReportMeasures() {
  return performance
    .getEntriesByType('measure')
    .filter((entry) => entry.name.startsWith('report:'));
}

for (const entry of getReportMeasures()) {
  console.log(`${entry.name}: ${entry.duration.toFixed(2)}ms`);
}
```

측정 이름의 종류를 제한하면 로그와 메트릭의 카디널리티가 불필요하게 커지는 문제도 줄일 수 있습니다.

### H3. 느린 측정 구간만 찾는다

조회 결과는 배열이므로 일반적인 배열 메서드로 임계값을 넘는 항목을 필터링하고 실행 시간이 긴 순서로 정렬할 수 있습니다.

```js
function findSlowMeasures(thresholdMs) {
  if (!Number.isFinite(thresholdMs) || thresholdMs < 0) {
    throw new TypeError('thresholdMs must be a non-negative finite number');
  }

  return performance
    .getEntriesByType('measure')
    .filter((entry) => entry.duration >= thresholdMs)
    .sort((a, b) => b.duration - a.duration)
    .map((entry) => ({
      name: entry.name,
      durationMs: Number(entry.duration.toFixed(2))
    }));
}

console.log(findSlowMeasures(100));
```

임계값은 환경과 기능의 응답 시간 목표를 기준으로 정해야 합니다.
개발 환경에서 임의로 고른 값만 사용하면 운영 환경의 실제 지연 특성을 놓칠 수 있습니다.

## 조회 결과를 안전하게 요약하기

### H3. 원본 객체 대신 필요한 필드만 직렬화한다

성능 항목을 외부 로그나 진단 응답으로 보낼 때는 필요한 필드만 명시적으로 선택하는 편이 좋습니다.
측정 이름에도 개인정보, 토큰, 전체 쿼리 문자열 같은 민감정보를 포함하지 않아야 합니다.

```js
function serializeMeasures() {
  return performance.getEntriesByType('measure').map((entry) => ({
    name: entry.name,
    startTimeMs: Number(entry.startTime.toFixed(2)),
    durationMs: Number(entry.duration.toFixed(2))
  }));
}
```

`startTime`은 성능 타임라인 기준의 상대 시간입니다.
로그 시스템에서 절대 시각이 필요하면 [Node.js performance.timeOrigin 가이드](/development/blog/seo/2026/06/14/nodejs-performance-timeorigin-absolute-timestamp-guide.html)의 변환 방식을 적용할 수 있습니다.

### H3. 평균만으로 판단하지 않는다

평균 실행 시간은 일부 매우 느린 호출을 가릴 수 있습니다.
호출 수, 최댓값, 백분위와 함께 확인해야 일반적인 처리 시간과 지연 급증을 구분하기 쉽습니다.

```js
function summarizeDurations(entries) {
  if (entries.length === 0) {
    return { count: 0, averageMs: 0, maxMs: 0 };
  }

  const durations = entries.map((entry) => entry.duration);
  const total = durations.reduce((sum, duration) => sum + duration, 0);

  return {
    count: durations.length,
    averageMs: Number((total / durations.length).toFixed(2)),
    maxMs: Number(Math.max(...durations).toFixed(2))
  };
}

console.log(summarizeDurations(performance.getEntriesByType('measure')));
```

대량의 측정값을 장기간 집계하려면 타임라인 배열 전체를 반복해서 조회하기보다 전용 히스토그램이나 메트릭 시스템을 사용하는 편이 효율적입니다.
지연 분포를 백분위로 요약하려면 [Node.js perf_hooks.createHistogram 가이드](/development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html)를 참고하세요.

## PerformanceObserver와 역할 구분하기

### H3. 일괄 조회와 지속 수집을 구분한다

`performance.getEntriesByType()`과 `PerformanceObserver`는 성능 항목을 다루지만 사용 시점이 다릅니다.

- `getEntriesByType()`: 현재 타임라인에 있는 항목을 필요할 때 일괄 조회한다.
- `PerformanceObserver`: 새 성능 항목이 생성될 때 콜백으로 지속 수집한다.

작업 종료 후 한 번 요약하면 충분하다면 조회 방식이 단순합니다.
반면 장시간 실행되는 서버에서 측정값을 계속 메트릭 시스템으로 보내야 한다면 관찰기 방식이 적합합니다.

### H3. 지원 항목을 런타임에서 확인한다

런타임과 Node.js 버전에 따라 사용할 수 있는 성능 항목 유형이 달라질 수 있습니다.
배포 환경에서 지원 여부를 확인하려면 [Node.js PerformanceObserver.supportedEntryTypes 가이드](/development/blog/seo/2026/06/13/nodejs-performanceobserver-supportedentrytypes-guide.html)의 검사 패턴을 활용할 수 있습니다.

지원되지 않거나 타임라인에 저장되지 않는 항목을 조회하면 원하는 데이터가 나오지 않을 수 있습니다.
필수 계측 데이터가 있다면 CI와 애플리케이션 시작 단계에서 요구사항을 명시적으로 검증해야 합니다.

## 타임라인 항목 정리하기

### H3. 조회는 항목을 삭제하지 않는다

`performance.getEntriesByType()`으로 항목을 조회해도 성능 타임라인에서 자동으로 제거되지는 않습니다.
장시간 실행되는 프로세스에서 mark와 measure를 계속 생성한다면 처리 후 `clearMarks()`와 `clearMeasures()`로 필요 없는 항목을 정리해야 합니다.

```js
import { performance } from 'node:perf_hooks';

function collectAndClearMeasures() {
  const snapshot = performance.getEntriesByType('measure').map((entry) => ({
    name: entry.name,
    durationMs: entry.duration
  }));

  performance.clearMeasures();
  performance.clearMarks();

  return snapshot;
}
```

전체 항목을 지우는 방식은 다른 기능이 같은 타임라인을 사용하지 않을 때만 안전합니다.
여러 모듈이 성능 항목을 공유한다면 이름 규칙과 소유 범위를 정하고, 이름을 지정해 필요한 항목만 정리해야 합니다.

### H3. 측정과 정리를 하나의 경계로 묶는다

작업 단위로 고유한 고정 접두사를 사용하고 `try`와 `finally`로 정리하면 예외가 발생해도 불필요한 항목이 남는 문제를 줄일 수 있습니다.

```js
async function measureReportBuild(buildReport) {
  const startMark = 'report:build:start';
  const endMark = 'report:build:end';
  const measureName = 'report:build';

  performance.mark(startMark);

  try {
    return await buildReport();
  } finally {
    performance.mark(endMark);
    performance.measure(measureName, startMark, endMark);

    const [entry] = performance
      .getEntriesByType('measure')
      .filter((item) => item.name === measureName);

    console.log({
      event: measureName,
      durationMs: Number(entry.duration.toFixed(2))
    });

    performance.clearMarks(startMark);
    performance.clearMarks(endMark);
    performance.clearMeasures(measureName);
  }
}
```

동시에 여러 작업이 같은 mark 이름을 사용하면 측정 경계가 섞일 수 있습니다.
동시 요청마다 타임라인 항목을 만드는 설계가 필요하다면 이름 충돌, 높은 카디널리티, 정리 책임을 함께 검토해야 합니다.

## 테스트에서 성능 항목 검증하기

### H3. 테스트 전후에 타임라인을 초기화한다

테스트가 같은 프로세스에서 실행되면 이전 테스트가 만든 항목이 다음 테스트 결과에 영향을 줄 수 있습니다.
각 테스트의 시작과 종료에서 관련 항목을 정리하면 검증 결과를 결정적으로 유지하기 쉽습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { performance } from 'node:perf_hooks';

test('creates one report measure', () => {
  performance.clearMarks();
  performance.clearMeasures();

  performance.mark('report:start');
  performance.mark('report:end');
  performance.measure('report:total', 'report:start', 'report:end');

  const measures = performance.getEntriesByType('measure');

  assert.equal(measures.length, 1);
  assert.equal(measures[0].name, 'report:total');
  assert.ok(measures[0].duration >= 0);

  performance.clearMarks();
  performance.clearMeasures();
});
```

실행 시간의 정확한 숫자를 단위 테스트에서 고정하면 시스템 부하에 따라 테스트가 불안정해질 수 있습니다.
항목 생성 여부와 값의 범위를 검증하고, 실제 성능 기준은 별도의 벤치마크와 운영 지표로 관리하는 편이 좋습니다.

## 자주 묻는 질문

### H3. getEntriesByType을 호출하면 항목이 타임라인에서 사라지나요?

아닙니다.
조회 후에도 항목은 남아 있으므로 더 이상 필요하지 않다면 `clearMarks()`와 `clearMeasures()`로 명시적으로 정리해야 합니다.

### H3. 특정 이름의 항목 하나만 찾으려면 어떻게 하나요?

유형별 배열을 조회한 뒤 이름으로 필터링할 수 있습니다.
이름과 유형을 알고 있다면 `performance.getEntriesByName(name, type)`을 사용하는 방법도 있습니다.

### H3. 서버 요청마다 getEntriesByType을 호출해도 되나요?

타임라인 항목이 많아질수록 전체 배열 조회와 필터링 비용도 커질 수 있습니다.
요청별 실시간 수집이 필요하다면 `PerformanceObserver`, 샘플링, 히스토그램, 전용 메트릭 시스템을 검토하고 항목 정리 정책을 마련해야 합니다.

## 적용 체크리스트

- [ ] 조회하려는 `PerformanceEntry` 유형과 목적을 명확히 정한다.
- [ ] 측정 이름에 개인정보와 민감정보를 포함하지 않는다.
- [ ] 이름 종류를 제한해 로그와 메트릭 카디널리티를 관리한다.
- [ ] 조회 결과에서 필요한 필드만 직렬화한다.
- [ ] 평균뿐 아니라 호출 수, 최댓값, 백분위를 함께 확인한다.
- [ ] 장기 실행 프로세스에서 처리한 mark와 measure를 정리한다.
- [ ] 테스트 전후에 관련 성능 항목을 초기화한다.

## 마무리

`performance.getEntriesByType()`은 Node.js 성능 타임라인에서 원하는 유형의 항목을 일괄 조회하는 간단한 도구입니다.
작업 종료 후 측정 결과를 요약하거나 테스트에서 성능 항목 생성을 검증할 때 특히 유용합니다.

조회는 수집의 끝이 아니라 처리 과정의 일부입니다.
필요한 필드만 안전하게 요약하고, 실시간 수집에는 `PerformanceObserver`를 사용하며, 처리한 타임라인 항목을 명시적으로 정리하면 성능 계측을 예측 가능하게 운영할 수 있습니다.

관련 글:

- [Node.js performance.mark·measure 가이드: 코드 구간 실행 시간 측정하기](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)
- [Node.js PerformanceObserver.supportedEntryTypes 가이드: 지원 성능 항목 확인하기](/development/blog/seo/2026/06/13/nodejs-performanceobserver-supportedentrytypes-guide.html)
- [Node.js performance.timeOrigin 가이드: 상대 시간을 절대 타임스탬프로 변환하기](/development/blog/seo/2026/06/14/nodejs-performance-timeorigin-absolute-timestamp-guide.html)

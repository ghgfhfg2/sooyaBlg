---
layout: post
title: "Node.js performance.getEntriesByName 가이드: 이름으로 성능 측정값 찾기"
date: 2026-06-16 20:00:00 +0900
lang: ko
translation_key: nodejs-performance-getentriesbyname-named-timing-query-guide
permalink: /development/blog/seo/2026/06/16/nodejs-performance-getentriesbyname-named-timing-query-guide.html
alternates:
  ko: /development/blog/seo/2026/06/16/nodejs-performance-getentriesbyname-named-timing-query-guide.html
  x_default: /development/blog/seo/2026/06/16/nodejs-performance-getentriesbyname-named-timing-query-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, getentriesbyname, perf-hooks, monitoring, observability]
description: "Node.js performance.getEntriesByName()으로 같은 이름의 mark·measure 성능 항목을 조회하고, entryType 필터·이름 규칙·타임라인 정리까지 실무 기준으로 적용하는 방법을 설명합니다."
---

Node.js 성능 측정 코드를 만들다 보면 "특정 작업 이름으로 기록된 값만 다시 꺼내고 싶다"는 요구가 자주 생깁니다.
전체 타임라인을 훑는 `performance.getEntries()`나 타입 단위로 조회하는 `performance.getEntriesByType()`도 유용하지만, 배치 작업, 빌드 단계, 테스트 검증처럼 관심 있는 이름이 정해져 있다면 `performance.getEntriesByName()`이 더 직접적입니다.

`performance.getEntriesByName(name[, type])`은 성능 타임라인에서 지정한 이름과 일치하는 항목을 배열로 반환합니다.
선택적으로 `entryType`을 함께 넘기면 같은 이름을 가진 `mark`, `measure`, `resource` 항목이 섞이는 문제를 줄일 수 있습니다.

이 글에서는 `getEntriesByName()`의 기본 동작, `mark`와 `measure`를 이름으로 조회하는 패턴, 테스트에서 흔들리지 않게 쓰는 방법, 운영 코드에서 이름 규칙과 민감정보 노출을 관리하는 기준을 정리합니다.
타입 단위 조회가 먼저 필요하다면 [Node.js performance.getEntriesByType 가이드](/development/blog/seo/2026/06/14/nodejs-performance-getentriesbytype-timeline-query-guide.html)를 함께 참고하세요.

## getEntriesByName이 필요한 순간

### H3. 전체 타임라인보다 특정 작업 이름이 중요할 때

`performance.mark()`와 `performance.measure()`는 전역 성능 타임라인에 항목을 남깁니다.
측정 항목이 많아지면 전체 목록에서 원하는 값을 직접 찾는 코드가 금방 산만해집니다.

```js
import { performance } from 'node:perf_hooks';

performance.mark('report:render:start');
performance.mark('report:render:end');
performance.measure(
  'report:render',
  'report:render:start',
  'report:render:end'
);

const entries = performance.getEntriesByName('report:render', 'measure');

console.log(entries.at(-1).duration);
```

이름을 기준으로 조회하면 "보고서 렌더링에 걸린 시간"처럼 의미 있는 단위만 뽑아낼 수 있습니다.
특히 같은 프로세스 안에서 여러 기능을 측정하는 CLI나 테스트에서는 조회 조건이 명확해집니다.

### H3. 이름만 넘기면 여러 entryType이 함께 나올 수 있다

첫 번째 인자만 넘기면 이름이 같은 모든 성능 항목이 반환됩니다.
`mark`와 `measure`에 같은 이름을 쓰거나, 라이브러리가 같은 이름의 항목을 만들면 결과가 예상보다 넓어질 수 있습니다.

```js
performance.mark('checkout');
performance.measure('checkout', {
  start: performance.now() - 25,
  end: performance.now()
});

const mixedEntries = performance.getEntriesByName('checkout');

console.log(mixedEntries.map((entry) => entry.entryType));
```

실무 코드에서는 두 번째 인자로 타입을 함께 지정하는 편이 안전합니다.
측정값이 필요한 상황이라면 `'measure'`, 시작점과 종료점 자체를 검증해야 한다면 `'mark'`처럼 의도를 드러내는 방식이 좋습니다.

## 기본 사용 패턴

### H3. 가장 최근 measure만 읽는다

같은 이름의 `measure`가 여러 번 기록될 수 있습니다.
예를 들어 워커 풀에서 같은 작업이 반복 실행되거나, 테스트 안에서 동일한 함수를 여러 번 호출하면 같은 이름의 항목이 누적됩니다.

```js
function getLatestMeasure(name) {
  const measures = performance.getEntriesByName(name, 'measure');
  return measures.at(-1) ?? null;
}

const latest = getLatestMeasure('report:render');

if (latest) {
  console.log({
    name: latest.name,
    durationMs: Number(latest.duration.toFixed(2))
  });
}
```

배열의 마지막 항목을 읽는 방식은 "가장 최근 결과"가 필요한 간단한 검증에 잘 맞습니다.
다만 장기 실행 서버에서는 같은 이름의 항목이 계속 쌓이지 않도록 정리 전략도 함께 둬야 합니다.

### H3. 여러 번 실행된 작업은 통계로 요약한다

반복 실행되는 작업은 한 번의 duration보다 분포가 더 중요할 때가 많습니다.
`getEntriesByName()`으로 같은 이름의 측정값을 모은 뒤 평균, 최댓값, 개수를 요약하면 로그 한 줄로 흐름을 파악할 수 있습니다.

```js
function summarizeMeasures(name) {
  const durations = performance
    .getEntriesByName(name, 'measure')
    .map((entry) => entry.duration);

  if (durations.length === 0) {
    return null;
  }

  const total = durations.reduce((sum, value) => sum + value, 0);

  return {
    name,
    count: durations.length,
    avgMs: Number((total / durations.length).toFixed(2)),
    maxMs: Number(Math.max(...durations).toFixed(2))
  };
}
```

이 패턴은 테스트보다 로컬 진단 스크립트나 배치 작업 리포트에 더 잘 맞습니다.
운영 메트릭은 가능하면 `PerformanceObserver`에서 실시간으로 소비하고, 타임라인 조회는 보조 진단용으로 제한하는 편이 좋습니다.

## mark와 measure 이름 규칙

### H3. 시작·종료 mark와 measure 이름을 분리한다

가독성을 위해 `job:start`, `job:end`, `job:total`처럼 역할이 드러나는 이름을 쓰면 조회 코드도 단순해집니다.
같은 문자열을 `mark`와 `measure`에 동시에 쓰면 타입 필터를 빠뜨렸을 때 실수하기 쉽습니다.

```js
performance.mark('invoice:pdf:start');
await renderInvoicePdf();
performance.mark('invoice:pdf:end');

performance.measure(
  'invoice:pdf:total',
  'invoice:pdf:start',
  'invoice:pdf:end'
);

const [measure] = performance.getEntriesByName(
  'invoice:pdf:total',
  'measure'
);
```

이름 규칙은 팀 안에서 고정해 두는 것이 좋습니다.
기능명, 단계명, 역할을 일정한 순서로 배치하면 검색과 집계가 쉬워집니다.

### H3. 사용자 입력을 이름에 직접 넣지 않는다

성능 entry 이름은 로그, 테스트 출력, 메트릭 라벨로 이어지기 쉽습니다.
사용자 이메일, 주문번호, 전체 URL, 검색어 같은 값을 그대로 넣으면 민감정보가 외부 시스템에 퍼질 수 있습니다.

```js
// 피해야 할 예: 사용자 입력이 이름에 섞인다.
performance.mark(`search:${userQuery}:start`);

// 권장: 고정된 이름과 제한된 detail만 사용한다.
performance.mark('search:request:start', {
  detail: { source: 'web' }
});
```

성능 측정 이름은 낮은 카디널리티를 유지해야 합니다.
값이 계속 달라지는 문자열은 집계를 어렵게 만들고, 비용과 개인정보 리스크를 동시에 키웁니다.

## 테스트에서 활용하기

### H3. 특정 이름의 측정값만 검증한다

테스트에서는 전체 성능 타임라인에 이전 테스트가 남긴 항목이 섞일 수 있습니다.
테스트가 만드는 이름을 고정하고 `entryType`까지 지정하면 검증 범위를 좁힐 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { performance } from 'node:perf_hooks';

test('render report records a measure', async () => {
  performance.clearMarks('report:render:start');
  performance.clearMarks('report:render:end');
  performance.clearMeasures('report:render');

  performance.mark('report:render:start');
  await Promise.resolve();
  performance.mark('report:render:end');
  performance.measure(
    'report:render',
    'report:render:start',
    'report:render:end'
  );

  const measures = performance.getEntriesByName('report:render', 'measure');

  assert.equal(measures.length, 1);
  assert.equal(measures[0].name, 'report:render');
  assert.ok(measures[0].duration >= 0);

  performance.clearMeasures('report:render');
  performance.clearMarks('report:render:start');
  performance.clearMarks('report:render:end');
});
```

테스트 전후에 관련 항목을 지우면 같은 파일의 다른 테스트와 충돌할 가능성이 줄어듭니다.
정리 API의 차이는 [Node.js performance.clearMarks·clearMeasures 가이드](/development/blog/seo/2026/06/15/nodejs-performance-clearmarks-clearmeasures-timeline-cleanup-guide.html)에서 이어서 볼 수 있습니다.

### H3. 병렬 테스트에서는 이름 충돌을 더 조심한다

Node.js 테스트 러너에서 파일이나 하위 테스트가 병렬로 실행되면 같은 이름의 성능 항목이 서로 섞일 수 있습니다.
성능 타임라인이 프로세스 단위라는 점을 감안해 테스트별 접두사를 붙이거나, 성능 측정 검증을 직렬 테스트로 분리해야 합니다.

```js
function createMeasureName(testId, action) {
  return `test:${testId}:${action}`;
}

const measureName = createMeasureName('render-report', 'total');

performance.measure(measureName, startMark, endMark);

const entries = performance.getEntriesByName(measureName, 'measure');
```

접두사를 붙이더라도 실제 사용자 데이터나 무작위로 긴 문자열을 넣는 것은 피해야 합니다.
테스트 이름처럼 관리 가능한 값만 사용하는 편이 좋습니다.

## getEntriesByName과 PerformanceObserver의 역할

### H3. 타임라인 조회는 사후 확인에 적합하다

`getEntriesByName()`은 이미 성능 타임라인에 남아 있는 항목을 조회합니다.
즉, "방금 만든 측정값을 확인한다"거나 "진단 스크립트가 마지막에 결과를 요약한다"는 흐름에 잘 맞습니다.

```js
const report = summarizeMeasures('invoice:pdf:total');

if (report) {
  console.log(report);
}
```

반면 장기 실행 서버에서 모든 측정값을 주기적으로 타임라인에서 뒤지는 방식은 비효율적일 수 있습니다.
새 항목이 생기는 즉시 처리해야 한다면 `PerformanceObserver`를 함께 고려해야 합니다.

### H3. observer는 실시간 소비에 적합하다

`PerformanceObserver`는 성능 항목이 만들어질 때 콜백으로 받아 처리합니다.
관측 가능한 항목을 실시간으로 로그나 메트릭 시스템에 보내야 한다면 observer가 더 자연스럽습니다.

```js
import { PerformanceObserver } from 'node:perf_hooks';

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.name === 'invoice:pdf:total') {
      console.log(entry.duration);
    }
  }
});

observer.observe({ entryTypes: ['measure'] });
```

종료 직전에 observer 내부 큐를 비우는 방법은 [Node.js PerformanceObserver.takeRecords 가이드](/development/blog/seo/2026/06/16/nodejs-performanceobserver-takerecords-buffer-drain-guide.html)를 참고하면 됩니다.
타임라인 조회와 observer는 경쟁 관계가 아니라, 조회 시점과 처리 방식이 다른 도구입니다.

## 실무 적용 체크리스트

### H3. 이름과 타입을 함께 조회한다

다음 기준을 기본값으로 두면 실수를 줄일 수 있습니다.

1. `performance.getEntriesByName(name, 'measure')`처럼 타입을 함께 넘긴다.
2. `mark` 이름과 `measure` 이름을 같은 문자열로 만들지 않는다.
3. 반복 측정은 마지막 항목만 읽을지, 전체를 요약할지 먼저 정한다.
4. 테스트 시작과 종료에서 관련 `mark`, `measure`를 정리한다.
5. 사용자 입력, 토큰, 이메일, 전체 URL을 entry 이름에 넣지 않는다.

이 기준은 코드 리뷰에서도 확인하기 쉽습니다.
성능 측정 코드는 장애 분석을 돕는 도구이므로, 측정 코드 자체가 불확실성과 리스크를 늘리지 않게 만드는 것이 중요합니다.

### H3. 타임라인 정리를 조회 코드와 함께 설계한다

`getEntriesByName()`은 조회만 하고 항목을 삭제하지 않습니다.
같은 이름의 측정이 반복되는 코드라면 조회 후 언제 `clearMeasures()`와 `clearMarks()`를 호출할지 정해야 합니다.

```js
try {
  const latest = getLatestMeasure('report:render');
  console.log(latest?.duration);
} finally {
  performance.clearMeasures('report:render');
  performance.clearMarks('report:render:start');
  performance.clearMarks('report:render:end');
}
```

단, 공용 서버 코드에서 무조건 전체 타임라인을 지우면 다른 모듈의 측정값까지 사라질 수 있습니다.
정리할 이름을 명시하고, 측정을 만든 코드가 정리 책임도 갖는 구조가 가장 예측 가능합니다.

## 마무리

`performance.getEntriesByName()`은 Node.js 성능 타임라인에서 특정 이름의 항목만 꺼내는 단순하지만 유용한 API입니다.
이름과 `entryType`을 함께 지정하면 `mark`와 `measure`가 섞이는 문제를 줄이고, 테스트나 진단 스크립트에서 필요한 값만 확인할 수 있습니다.

핵심은 이름 규칙과 정리 전략입니다.
측정 이름은 고정되고 낮은 카디널리티를 유지해야 하며, 민감정보를 포함하지 않아야 합니다.
반복 실행되는 코드에서는 조회 후 `clearMeasures()`와 `clearMarks()`로 타임라인을 관리해야 합니다.

다음 글을 함께 보면 Node.js 성능 타임라인 흐름을 더 체계적으로 정리할 수 있습니다.

- [Node.js performance.getEntriesByType 가이드](/development/blog/seo/2026/06/14/nodejs-performance-getentriesbytype-timeline-query-guide.html)
- [Node.js performance.clearMarks·clearMeasures 가이드](/development/blog/seo/2026/06/15/nodejs-performance-clearmarks-clearmeasures-timeline-cleanup-guide.html)
- [Node.js PerformanceObserver.takeRecords 가이드](/development/blog/seo/2026/06/16/nodejs-performanceobserver-takerecords-buffer-drain-guide.html)

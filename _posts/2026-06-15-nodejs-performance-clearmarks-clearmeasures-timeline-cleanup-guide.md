---
layout: post
title: "Node.js performance.clearMarks·clearMeasures 가이드: 성능 타임라인 누적 방지하기"
date: 2026-06-15 08:00:00 +0900
lang: ko
translation_key: nodejs-performance-clearmarks-clearmeasures-timeline-cleanup-guide
permalink: /development/blog/seo/2026/06/15/nodejs-performance-clearmarks-clearmeasures-timeline-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/06/15/nodejs-performance-clearmarks-clearmeasures-timeline-cleanup-guide.html
  x_default: /development/blog/seo/2026/06/15/nodejs-performance-clearmarks-clearmeasures-timeline-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, perf-hooks, clearmarks, clearmeasures, monitoring]
description: "Node.js performance.clearMarks()와 clearMeasures()로 성능 타임라인 누적을 방지하고, 이름별 정리·예외 처리·동시 실행·테스트 격리 패턴을 적용하는 방법을 설명합니다."
---

Node.js에서 `performance.mark()`와 `performance.measure()`로 성능 구간을 측정하면 생성된 항목은 성능 타임라인에 남습니다.
짧게 실행되는 스크립트에서는 큰 문제가 되지 않을 수 있지만, 서버와 워커처럼 오래 실행되는 프로세스가 측정 항목을 계속 만들면 조회 결과가 불필요하게 커지고 서로 다른 작업의 항목이 섞일 수 있습니다.

`node:perf_hooks`의 `performance.clearMarks()`와 `performance.clearMeasures()`는 각각 mark와 measure 항목을 타임라인에서 제거합니다.
측정값을 소비한 뒤 소유한 항목만 정리하면 타임라인 누적을 줄이고 테스트와 진단 결과를 예측 가능하게 유지할 수 있습니다.

이 글에서는 전체 정리와 이름별 정리의 차이, `try`와 `finally`를 활용한 정리 책임, 동시 실행 시 이름 충돌 방지, 테스트 격리 방법을 정리합니다.
측정 항목을 만드는 기본 방법은 [Node.js performance.mark·measure 가이드](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)를 먼저 참고하세요.

## clearMarks와 clearMeasures가 필요한 이유

### H3. 조회와 정리는 별도 작업이다

`performance.getEntriesByType()`이나 `performance.getEntriesByName()`으로 항목을 조회해도 해당 항목은 자동으로 삭제되지 않습니다.
조회는 현재 타임라인의 스냅샷을 반환하고, 정리는 별도 메서드가 담당합니다.

```js
import { performance } from 'node:perf_hooks';

performance.mark('report:start');
performance.mark('report:end');
performance.measure('report:total', 'report:start', 'report:end');

console.log(performance.getEntriesByType('measure').length); // 1
console.log(performance.getEntriesByType('measure').length); // 1

performance.clearMeasures('report:total');
performance.clearMarks('report:start');
performance.clearMarks('report:end');

console.log(performance.getEntriesByType('measure').length); // 0
```

현재 항목을 유형별로 조회하고 요약하는 방법은 [Node.js performance.getEntriesByType 가이드](/development/blog/seo/2026/06/14/nodejs-performance-getentriesbytype-timeline-query-guide.html)에서 자세히 확인할 수 있습니다.

### H3. mark와 measure를 각각 정리해야 한다

measure를 지워도 해당 구간을 만들 때 사용한 mark가 함께 삭제되지는 않습니다.
반대로 mark를 지워도 이미 생성된 measure는 남습니다.

따라서 두 유형을 모두 생성했다면 각각 정리해야 합니다.

```js
performance.clearMeasures('report:total');
performance.clearMarks('report:start');
performance.clearMarks('report:end');
```

정리 순서는 일반적으로 측정값을 조회한 뒤 measure와 mark를 제거하는 방식이 이해하기 쉽습니다.
measure를 만들기 전에 시작 mark를 지우면 의도한 구간을 측정할 수 없으므로 측정 완료 시점을 정리 경계로 삼아야 합니다.

## 전체 정리와 이름별 정리

### H3. 이름을 생략하면 해당 유형 전체를 제거한다

`clearMarks()`와 `clearMeasures()`에 이름을 전달하지 않으면 현재 타임라인의 해당 유형 항목을 모두 제거합니다.

```js
performance.clearMeasures();
performance.clearMarks();
```

전체 정리는 하나의 테스트가 타임라인을 독점하거나, 프로세스 종료 직전 진단 데이터를 초기화하는 상황에서는 간단합니다.
하지만 여러 모듈이 같은 타임라인을 사용하는 애플리케이션에서 호출하면 다른 기능이 아직 소비하지 않은 항목까지 삭제할 수 있습니다.

### H3. 공유 프로세스에서는 이름별 정리를 우선한다

서버, 워커, 통합 테스트처럼 여러 기능이 같은 프로세스에서 실행된다면 자신이 만든 이름만 정리하는 편이 안전합니다.
기능별 고정 접두사를 정하면 소유 범위를 파악하기 쉽습니다.

```js
function clearReportTiming() {
  performance.clearMeasures('report:build');
  performance.clearMarks('report:build:start');
  performance.clearMarks('report:build:end');
}
```

사용자 ID, 이메일, 전체 URL처럼 종류가 계속 늘거나 민감할 수 있는 값을 항목 이름에 포함하지 않아야 합니다.
`report:build`, `cache:warm`처럼 제한된 고정 이름을 사용하면 정보 노출과 높은 카디널리티 문제를 함께 줄일 수 있습니다.

## 측정과 정리를 하나의 함수로 묶기

### H3. finally에서 정리해 예외 경로를 다룬다

측정 대상 함수가 실패하면 정상 흐름의 마지막에 둔 정리 코드가 실행되지 않을 수 있습니다.
`finally`에서 항목을 정리하면 성공과 실패 경로 모두에서 정리 책임을 지킬 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

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

    const [entry] = performance.getEntriesByName(measureName, 'measure');

    console.log({
      event: measureName,
      durationMs: Number(entry.duration.toFixed(2))
    });

    performance.clearMeasures(measureName);
    performance.clearMarks(startMark);
    performance.clearMarks(endMark);
  }
}
```

로그에는 측정 이름과 소요 시간처럼 진단에 필요한 최소 정보만 남겨야 합니다.
오류 객체나 작업 입력 전체를 그대로 출력하면 토큰, 개인정보, 연결 문자열 같은 민감정보가 섞일 수 있습니다.

### H3. 정리 자체도 항상 실행되게 중첩 finally를 사용한다

측정 결과를 로그나 외부 메트릭 시스템으로 보내는 코드가 실패할 수 있다면, 전송과 정리를 같은 `finally` 블록에 순서대로 두는 것만으로는 충분하지 않을 수 있습니다.
정리 코드를 중첩된 `finally`에 두면 결과 처리 실패 여부와 관계없이 실행할 수 있습니다.

```js
function consumeAndClear(measureName, startMark, endMark, consume) {
  try {
    const [entry] = performance.getEntriesByName(measureName, 'measure');

    if (entry) {
      consume(entry);
    }
  } finally {
    performance.clearMeasures(measureName);
    performance.clearMarks(startMark);
    performance.clearMarks(endMark);
  }
}
```

`consume()`의 실패를 무시하라는 의미는 아닙니다.
호출자는 오류를 기록하거나 재시도하되, 성능 타임라인 정리 책임이 누락되지 않도록 경계를 분리해야 합니다.

## 동시 실행에서 이름 충돌 피하기

### H3. 같은 mark 이름을 동시에 재사용하지 않는다

동시에 실행되는 두 작업이 같은 mark 이름을 사용하면 어떤 시작점과 종료점을 측정하는지 모호해질 수 있습니다.
한 작업이 이름별 정리를 수행하는 동안 다른 작업의 항목까지 제거할 수도 있습니다.

요청마다 mark를 만드는 대신, 호출별 지연 분포를 지속 수집하는 목적이라면 `performance.timerify()`와 히스토그램 같은 전용 계측 방식을 검토하는 편이 좋습니다.
[Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)는 함수 호출별 실행 시간을 수집하는 방법을 설명합니다.

### H3. 고유 이름이 필요하면 수명과 노출 범위를 제한한다

동시 작업별로 반드시 별도 mark가 필요하다면 추측 가능한 순번처럼 민감하지 않은 짧은 식별자를 사용하고, 작업이 끝나는 즉시 이름별로 정리해야 합니다.
외부 입력을 그대로 이름에 넣거나 타임라인 전체를 외부 응답으로 노출해서는 안 됩니다.

```js
let sequence = 0;

async function measureExclusiveTask(runTask) {
  const id = sequence++;
  const startMark = `task:${id}:start`;
  const endMark = `task:${id}:end`;
  const measureName = `task:${id}`;

  performance.mark(startMark);

  try {
    return await runTask();
  } finally {
    performance.mark(endMark);
    performance.measure(measureName, startMark, endMark);

    const [entry] = performance.getEntriesByName(measureName, 'measure');
    console.log({ event: 'task', durationMs: Number(entry.duration.toFixed(2)) });

    performance.clearMeasures(measureName);
    performance.clearMarks(startMark);
    performance.clearMarks(endMark);
  }
}
```

이 예시는 단일 프로세스 안에서 이름 충돌을 줄이는 최소 패턴입니다.
대규모 동시 요청을 모두 타임라인 항목으로 만들기 전에 메모리 사용량, 관찰 목적, 샘플링 전략을 함께 검토해야 합니다.

## 테스트에서 타임라인 격리하기

### H3. 테스트 시작과 종료에서 관련 항목을 제거한다

같은 Node.js 프로세스에서 여러 테스트가 실행되면 이전 테스트가 남긴 항목 때문에 개수 검증이 실패할 수 있습니다.
테스트가 타임라인을 독점한다면 시작과 종료에서 전체 정리를 적용할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { performance } from 'node:perf_hooks';

test('clears report timing entries', (t) => {
  performance.clearMarks();
  performance.clearMeasures();

  t.after(() => {
    performance.clearMeasures();
    performance.clearMarks();
  });

  performance.mark('report:start');
  performance.mark('report:end');
  performance.measure('report:total', 'report:start', 'report:end');

  assert.equal(performance.getEntriesByName('report:total', 'measure').length, 1);

  performance.clearMeasures('report:total');
  performance.clearMarks('report:start');
  performance.clearMarks('report:end');

  assert.equal(performance.getEntriesByName('report:total', 'measure').length, 0);
});
```

병렬 테스트가 같은 타임라인을 공유한다면 전체 정리는 다른 테스트에 영향을 줄 수 있습니다.
그 경우 테스트를 격리된 프로세스로 실행하거나 테스트별 이름과 이름별 정리를 사용해야 합니다.

### H3. 실제 시간 숫자보다 생성과 정리를 검증한다

단위 테스트에서 정확한 실행 시간을 고정하면 시스템 부하에 따라 결과가 흔들릴 수 있습니다.
성능 항목이 생성됐는지, `duration`이 유효한 범위인지, 정리 후 해당 이름이 사라졌는지를 검증하는 편이 안정적입니다.

실제 성능 기준과 회귀 여부는 별도 벤치마크와 운영 메트릭으로 관리해야 합니다.

## 운영 적용 체크리스트

### H3. 타임라인 소유권을 코드에 드러낸다

성능 타임라인 정리는 단순한 마지막 한 줄이 아니라 계측 설계의 일부입니다.
다음 항목을 배포 전에 확인하면 다른 모듈의 데이터를 지우거나 항목을 계속 누적하는 문제를 줄일 수 있습니다.

- 측정값을 언제, 어디에서 소비할지 정했는가?
- 생성한 mark와 measure를 각각 정리하는가?
- 공유 프로세스에서 전체 정리 대신 이름별 정리를 사용하는가?
- 예외와 결과 전송 실패가 발생해도 정리 코드가 실행되는가?
- 이름에 개인정보, 토큰, 전체 URL 같은 민감정보가 포함되지 않는가?
- 동시 실행에서 같은 이름이 충돌하지 않는가?
- 대량 호출에는 타임라인 대신 집계용 메트릭이 더 적합하지 않은가?

## 자주 묻는 질문

### H3. clearMarks를 호출하면 measure도 삭제되나요?

아닙니다.
`clearMarks()`는 mark 항목만 제거하며, 이미 만들어진 measure는 `clearMeasures()`로 별도 정리해야 합니다.

### H3. clearMeasures에 존재하지 않는 이름을 전달하면 오류가 발생하나요?

존재하지 않는 이름을 지정해도 제거할 항목이 없을 뿐입니다.
정리 함수는 생성 여부가 불확실한 예외 경로의 `finally`에서도 사용할 수 있습니다.

### H3. 서버 요청마다 clearMarks와 clearMeasures를 호출해도 되나요?

호출 자체보다 이름 충돌과 타임라인 설계가 더 중요합니다.
동시 요청이 같은 이름을 공유하면 측정과 정리가 서로 간섭할 수 있으므로, 요청별 mark가 꼭 필요한지 먼저 검토하고 지속적인 지연 수집에는 전용 메트릭 방식을 우선 고려하세요.

## 마무리

`performance.clearMarks()`와 `performance.clearMeasures()`는 Node.js 성능 타임라인을 예측 가능한 상태로 유지하는 기본 도구입니다.
조회만으로 항목이 사라지지 않고 mark와 measure가 독립적으로 남는다는 점을 이해해야 올바른 정리 경계를 만들 수 있습니다.

공유 프로세스에서는 이름별 정리를 우선하고, 측정 결과 소비와 정리를 `try`와 `finally`로 묶으세요.
동시 실행과 테스트 격리까지 함께 설계하면 성능 계측이 다른 기능을 방해하지 않으면서 필요한 진단 정보를 안정적으로 제공할 수 있습니다.

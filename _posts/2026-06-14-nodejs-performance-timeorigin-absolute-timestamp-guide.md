---
layout: post
title: "Node.js performance.timeOrigin 가이드: 상대 시간을 절대 타임스탬프로 변환하기"
date: 2026-06-14 08:00:00 +0900
lang: ko
translation_key: nodejs-performance-timeorigin-absolute-timestamp-guide
permalink: /development/blog/seo/2026/06/14/nodejs-performance-timeorigin-absolute-timestamp-guide.html
alternates:
  ko: /development/blog/seo/2026/06/14/nodejs-performance-timeorigin-absolute-timestamp-guide.html
  x_default: /development/blog/seo/2026/06/14/nodejs-performance-timeorigin-absolute-timestamp-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, timeorigin, timestamp, monitoring, observability]
description: "Node.js performance.timeOrigin과 performance.now()를 조합해 상대 시간을 절대 타임스탬프로 변환하고 로그·트레이스·성능 측정 시간을 일관되게 기록하는 방법을 설명합니다."
---

Node.js에서 실행 시간을 측정할 때 `Date.now()`만 사용하면 시스템 시계 보정의 영향을 받을 수 있고, `performance.now()`만 사용하면 다른 시스템의 로그와 시각을 맞추기 어렵습니다.
성능 측정에는 안정적인 상대 시간이 필요하지만, 장애 분석과 분산 추적에는 사람이 읽고 비교할 수 있는 절대 시각도 필요합니다.

`node:perf_hooks`의 `performance.timeOrigin`은 현재 프로세스 성능 타임라인의 기준이 되는 Unix 타임스탬프를 밀리초 단위로 제공합니다.
여기에 `performance.now()`를 더하면 성능 측정의 상대 시간 값을 로그와 트레이스에서 사용할 수 있는 절대 타임스탬프로 변환할 수 있습니다.

이 글에서는 `performance.timeOrigin`의 의미, 상대 시간과 절대 시간의 역할 분리, 성능 항목의 타임스탬프 변환, 운영 환경에서의 주의점을 정리합니다.
코드 구간의 실행 시간을 직접 측정하려면 [Node.js performance.mark·measure 가이드](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)도 함께 참고하세요.

## performance.timeOrigin이 필요한 이유

### H3. 성능 측정 시간과 로그 시간을 연결한다

Node.js 애플리케이션에서는 보통 두 종류의 시간이 필요합니다.

- 처리 시간이 얼마나 걸렸는지 계산하는 상대 시간
- 이벤트가 실제로 언제 발생했는지 표시하는 절대 시간

`performance.now()`는 성능 타임라인 기준점부터 흐른 시간을 반환하므로 구간별 소요 시간을 계산하는 데 적합합니다.
반면 `performance.timeOrigin`은 그 기준점이 Unix 시간으로 언제인지 알려 줍니다.

두 값을 더하면 현재 성능 타임라인의 시각을 절대 타임스탬프로 바꿀 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

const relativeNowMs = performance.now();
const absoluteNowMs = performance.timeOrigin + relativeNowMs;

console.log({
  relativeNowMs: Number(relativeNowMs.toFixed(2)),
  absoluteNowMs: Math.round(absoluteNowMs),
  occurredAt: new Date(absoluteNowMs).toISOString()
});
```

`relativeNowMs`는 프로세스 안에서 실행 시간을 비교할 때 사용하고, `occurredAt`은 다른 서비스의 로그와 사건 순서를 맞출 때 사용할 수 있습니다.

### H3. Date.now와 performance.now는 목적이 다르다

`Date.now()`는 현재 시스템 시각을 Unix 타임스탬프로 반환합니다.
사람이 읽는 로그 시각이나 외부 시스템과 교환하는 타임스탬프에는 적합하지만, 실행 중 운영체제의 시계가 보정되면 두 시점의 차이가 실제 경과 시간과 달라질 수 있습니다.

`performance.now()`는 단조 증가하는 고해상도 시간 값을 제공하므로 구간 소요 시간을 측정하는 데 적합합니다.
성능 측정에는 `performance.now()`를 사용하고, 필요한 시점에 `performance.timeOrigin`을 더해 절대 시각으로 변환하는 방식이 역할을 분명하게 만듭니다.

## 기본 사용법

### H3. 프로세스 기준 시각을 ISO 문자열로 확인한다

`performance.timeOrigin`은 밀리초 단위의 숫자이므로 `Date`에 전달해 읽기 쉬운 형식으로 바꿀 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

console.log({
  timeOriginMs: performance.timeOrigin,
  timeOriginIso: new Date(performance.timeOrigin).toISOString()
});
```

이 시각은 애플리케이션의 준비 완료 시각이나 HTTP 서버가 포트를 연 시각과 같지 않습니다.
런타임 시작 단계와 준비 시간을 구분하려면 [Node.js performance.nodeTiming 가이드](/development/blog/seo/2026/06/13/nodejs-performance-nodetiming-startup-time-guide.html)를 함께 활용해야 합니다.

### H3. 공통 변환 함수로 단위를 명확하게 만든다

상대 시간을 여러 곳에서 절대 시간으로 바꾼다면 변환 함수를 하나로 모으는 편이 좋습니다.
함수 이름과 반환값에 밀리초 단위를 드러내면 초와 밀리초를 혼동하는 오류도 줄일 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

function relativeToUnixMs(relativeMs) {
  if (!Number.isFinite(relativeMs) || relativeMs < 0) {
    throw new TypeError('relativeMs must be a non-negative finite number');
  }

  return performance.timeOrigin + relativeMs;
}

const observedAtMs = relativeToUnixMs(performance.now());

console.log({
  observedAtMs: Math.round(observedAtMs),
  observedAt: new Date(observedAtMs).toISOString()
});
```

외부 API가 초 단위를 요구한다면 변환 지점에서 명시적으로 `Math.floor(observedAtMs / 1000)`을 적용해야 합니다.
필드 이름도 `observedAtMs`, `observedAtSeconds`처럼 단위를 구분하는 편이 안전합니다.

## 성능 항목의 절대 시각 계산하기

### H3. PerformanceEntry의 startTime을 변환한다

`performance.mark()`와 `performance.measure()`로 생성한 성능 항목의 `startTime`은 성능 타임라인 기준의 상대 시간입니다.
항목이 시작된 절대 시각은 `performance.timeOrigin + entry.startTime`으로 계산할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

performance.mark('cache:warm:start');
await Promise.resolve();
performance.mark('cache:warm:end');
performance.measure('cache:warm', 'cache:warm:start', 'cache:warm:end');

const [entry] = performance.getEntriesByName('cache:warm', 'measure');
const startedAtMs = performance.timeOrigin + entry.startTime;

console.log({
  event: entry.name,
  startedAt: new Date(startedAtMs).toISOString(),
  durationMs: Number(entry.duration.toFixed(2))
});

performance.clearMarks('cache:warm:start');
performance.clearMarks('cache:warm:end');
performance.clearMeasures('cache:warm');
```

`duration`에는 `timeOrigin`을 더하면 안 됩니다.
`startTime`은 기준점에서 시작 시점까지의 상대 위치이고, `duration`은 시작부터 종료까지의 길이이기 때문입니다.

### H3. 시작 시각과 종료 시각을 함께 기록한다

성능 항목의 종료 시각은 `startTime + duration`으로 계산한 뒤 `timeOrigin`을 더할 수 있습니다.

```js
function serializePerformanceEntry(entry) {
  const startedAtMs = performance.timeOrigin + entry.startTime;
  const endedAtMs = startedAtMs + entry.duration;

  return {
    name: entry.name,
    entryType: entry.entryType,
    startedAt: new Date(startedAtMs).toISOString(),
    endedAt: new Date(endedAtMs).toISOString(),
    durationMs: Number(entry.duration.toFixed(2))
  };
}
```

시작과 종료 시각은 로그 검색과 사건 순서 분석에 유용하고, `durationMs`는 성능 회귀 비교에 유용합니다.
세 값을 함께 기록하면 절대 시각과 경과 시간을 각각의 목적에 맞게 사용할 수 있습니다.

## 로그와 트레이스에 연결하기

### H3. 한 번 측정한 값을 재사용한다

로그 객체를 만들 때 `performance.now()`를 여러 번 호출하면 각 필드가 미세하게 다른 시점을 나타냅니다.
하나의 사건을 기록할 때는 상대 시간을 한 번만 읽고 필요한 표현으로 변환해야 합니다.

```js
import { performance } from 'node:perf_hooks';

function createEventTiming() {
  const relativeMs = performance.now();
  const unixMs = performance.timeOrigin + relativeMs;

  return {
    relativeMs: Number(relativeMs.toFixed(3)),
    unixMs: Math.round(unixMs),
    isoTime: new Date(unixMs).toISOString()
  };
}

console.log({
  event: 'worker_ready',
  ...createEventTiming()
});
```

로그 전송 과정에서는 토큰, 연결 문자열, 사용자 개인정보를 시간 정보와 함께 섞어 보내지 않아야 합니다.
장애 분석에 필요한 이벤트 이름, 시각, 소요 시간, 고정된 서비스 식별자 정도만 기록하는 편이 안전합니다.

### H3. 다른 프로세스의 상대 시간을 직접 비교하지 않는다

`performance.now()` 값은 각 프로세스의 `timeOrigin`을 기준으로 합니다.
프로세스 A의 `performance.now()`와 프로세스 B의 `performance.now()`를 그대로 비교하면 기준점이 다르므로 의미가 없습니다.

여러 프로세스나 서비스의 사건 순서를 비교하려면 각각의 `timeOrigin`을 사용해 절대 타임스탬프로 변환해야 합니다.
다만 시스템 시계가 서로 어긋나면 절대 타임스탬프도 정확히 맞지 않을 수 있으므로, 분산 환경에서는 시계 동기화와 트레이스의 인과 관계 정보를 함께 사용해야 합니다.

## 운영 환경에서 주의할 점

### H3. 절대 시각과 경과 시간을 서로 대체하지 않는다

절대 타임스탬프는 로그 검색과 외부 시스템 연동에 필요하지만, 실행 시간 계산에는 상대 시간을 사용하는 편이 안정적입니다.
반대로 상대 시간은 프로세스 밖에서 단독으로 해석하기 어렵습니다.

운영 지표를 설계할 때 다음처럼 역할을 나누면 혼란을 줄일 수 있습니다.

- 구간 소요 시간과 백분위: `performance.now()` 또는 `PerformanceEntry.duration`
- 로그와 이벤트 발생 시각: `performance.timeOrigin + relativeMs`
- 사람이 읽는 표현: 절대 타임스탬프를 ISO 8601 문자열로 변환
- 시스템 간 사건 연결: 절대 시각과 trace ID를 함께 사용

실행 시간의 분포를 관찰하려면 [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)처럼 호출별 측정과 백분위 집계를 함께 고려할 수 있습니다.

### H3. 숫자 정밀도와 저장 형식을 점검한다

`performance.timeOrigin`과 `performance.now()`는 모두 숫자이지만, 저장소나 외부 API가 소수점 이하 값을 유지하지 않을 수 있습니다.
로그에는 ISO 문자열과 반올림한 Unix 밀리초를 기록하고, 정밀한 성능 분석에는 원래 상대 시간과 `duration`을 별도로 유지하는 방식이 실용적입니다.

JavaScript 밀리초 값을 초 단위 필드에 그대로 보내면 약 1,000배 큰 타임스탬프가 됩니다.
외부 시스템의 단위와 허용 범위를 배포 전에 테스트해야 합니다.

### H3. timeOrigin을 애플리케이션 시작 완료 시각으로 오해하지 않는다

`performance.timeOrigin`은 성능 타임라인의 기준점이지 서비스가 트래픽을 처리할 준비가 된 시각은 아닙니다.
설정 검증, 데이터베이스 연결, 캐시 준비가 끝난 시각은 애플리케이션이 별도로 측정해야 합니다.

준비 완료 시각을 기록할 때도 `performance.now()`로 상대 준비 시간을 측정하고, 필요하면 `timeOrigin`을 더해 절대 시각을 함께 남길 수 있습니다.

```js
const readyRelativeMs = performance.now();
const readyAtMs = performance.timeOrigin + readyRelativeMs;

console.log({
  event: 'application_ready',
  readyAfterMs: Number(readyRelativeMs.toFixed(2)),
  readyAt: new Date(readyAtMs).toISOString()
});
```

## 테스트로 변환 규칙 검증하기

### H3. 허용 오차 안에서 Date.now와 비교한다

`performance.timeOrigin + performance.now()`는 일반적으로 현재 절대 시각과 가까운 값을 만듭니다.
테스트에서는 실행 지연을 고려해 작은 허용 오차를 두고 비교할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { performance } from 'node:perf_hooks';

test('performance timeline converts to current unix time', () => {
  const convertedNow = performance.timeOrigin + performance.now();
  const wallClockNow = Date.now();

  assert.ok(Math.abs(convertedNow - wallClockNow) < 100);
});
```

이 테스트는 시스템 시계 상태와 실행 환경의 영향을 받을 수 있으므로 지나치게 작은 허용 오차를 사용하면 불안정해질 수 있습니다.
핵심 비즈니스 로직 테스트에서는 시각 공급자를 주입해 결정적으로 검증하고, 실제 런타임 변환 검사는 별도 통합 테스트로 두는 편이 좋습니다.

### H3. 단위 변환과 잘못된 입력을 검증한다

공통 변환 함수를 만들었다면 정상 입력뿐 아니라 `NaN`, 음수, 무한대 같은 잘못된 입력도 테스트해야 합니다.
외부 시스템으로 전송하기 전에는 초와 밀리초 필드가 의도한 단위인지 샘플 이벤트로 확인해야 합니다.

## 자주 묻는 질문

### H3. performance.timeOrigin은 Date.now와 같은 값인가요?

둘 다 Unix 기준 밀리초 값으로 표현할 수 있지만 의미가 다릅니다.
`Date.now()`는 호출 시점의 시스템 시각이고, `performance.timeOrigin`은 현재 프로세스 성능 타임라인의 기준 시각입니다.

### H3. 실행 시간 측정에도 timeOrigin을 더해야 하나요?

아닙니다.
두 시점의 차이나 `PerformanceEntry.duration`처럼 경과 시간을 계산할 때는 상대 시간만 사용하면 됩니다.
이벤트가 실제로 언제 발생했는지 절대 시각이 필요할 때만 `timeOrigin`을 더합니다.

### H3. 여러 서버의 타임스탬프를 변환하면 정확히 정렬되나요?

각 서버의 시스템 시계가 충분히 동기화되어야 의미 있는 비교가 가능합니다.
분산 시스템에서는 절대 시각만 믿기보다 trace ID, 부모·자식 span 관계, 메시지 순서 같은 인과 관계 정보도 함께 사용해야 합니다.

## 적용 체크리스트

- [ ] 경과 시간은 `performance.now()` 또는 `PerformanceEntry.duration`으로 측정한다.
- [ ] 절대 시각이 필요할 때만 `performance.timeOrigin`과 상대 시간을 더한다.
- [ ] `startTime`과 `duration`의 의미를 구분한다.
- [ ] 여러 프로세스의 상대 시간 값을 직접 비교하지 않는다.
- [ ] 외부 시스템의 초·밀리초 단위를 배포 전에 검증한다.
- [ ] 로그에 민감정보를 포함하지 않고 필요한 시간 정보만 기록한다.
- [ ] 애플리케이션 준비 완료 시각을 `timeOrigin`과 구분한다.

## 마무리

`performance.timeOrigin`은 Node.js 성능 타임라인의 상대 시간과 운영 로그의 절대 시각을 연결하는 기준점입니다.
`performance.now()`와 `PerformanceEntry`로 안정적으로 경과 시간을 측정하고, 필요한 순간에만 `timeOrigin`을 더하면 측정 정확성과 로그 활용성을 함께 확보할 수 있습니다.

시간 데이터는 값 자체보다 기준점과 단위를 명확히 하는 것이 중요합니다.
상대 시간, 절대 시간, 소요 시간을 각 목적에 맞게 분리하고 변환 규칙을 공통화하면 장애 분석과 성능 비교에서 생기는 혼란을 줄일 수 있습니다.

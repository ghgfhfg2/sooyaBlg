---
layout: post
title: "Node.js performance.toJSON 가이드: 런타임 성능 스냅샷 직렬화하기"
date: 2026-06-15 20:00:00 +0900
lang: ko
translation_key: nodejs-performance-tojson-runtime-snapshot-guide
permalink: /development/blog/seo/2026/06/15/nodejs-performance-tojson-runtime-snapshot-guide.html
alternates:
  ko: /development/blog/seo/2026/06/15/nodejs-performance-tojson-runtime-snapshot-guide.html
  x_default: /development/blog/seo/2026/06/15/nodejs-performance-tojson-runtime-snapshot-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, perf-hooks, tojson, monitoring, observability]
description: "Node.js performance.toJSON()으로 nodeTiming·timeOrigin·eventLoopUtilization 런타임 스냅샷을 직렬화하고, 로그 필드 선별·주기 수집·테스트에 적용하는 방법을 설명합니다."
---

Node.js 프로세스의 시작 상태와 이벤트 루프 활용률을 로그로 남기려면 여러 `node:perf_hooks` API의 반환값을 각각 조합해야 합니다.
필드를 직접 모으는 방식은 목적이 분명하지만, 빠르게 런타임 상태를 확인하거나 진단 스냅샷의 기본 형태를 만들 때는 반복 코드가 생길 수 있습니다.

`performance.toJSON()`은 현재 Node.js 런타임의 성능 관련 정보를 직렬화 가능한 객체로 반환합니다.
반환값에는 프로세스 시작 단계 정보인 `nodeTiming`, 성능 타임라인 기준 시각인 `timeOrigin`, 이벤트 루프 상태인 `eventLoopUtilization`이 포함됩니다.

이 글에서는 `performance.toJSON()`의 반환 구조, `JSON.stringify()`와의 관계, 운영 로그에 필요한 필드만 선별하는 방법, 주기 수집과 테스트 시 주의점을 정리합니다.
개별 시작 단계의 의미를 먼저 확인하려면 [Node.js performance.nodeTiming 가이드](/development/blog/seo/2026/06/13/nodejs-performance-nodetiming-startup-time-guide.html)를 참고하세요.

## performance.toJSON이 필요한 이유

### H3. 런타임 성능 상태를 하나의 객체로 확인한다

`performance.toJSON()`을 호출하면 현재 프로세스의 런타임 성능 스냅샷을 객체로 받을 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

const snapshot = performance.toJSON();

console.log(snapshot);
```

반환 객체의 주요 속성은 다음과 같습니다.

- `nodeTiming`: Node.js 시작 단계와 이벤트 루프 관련 타이밍
- `timeOrigin`: 성능 타임라인의 Unix 밀리초 기준 시각
- `eventLoopUtilization`: 이벤트 루프의 유휴 시간, 활성 시간, 활용률

각 값은 서로 다른 질문에 답합니다.
`nodeTiming`은 프로세스 시작 과정, `timeOrigin`은 상대 시간과 절대 시간의 연결, `eventLoopUtilization`은 이벤트 루프가 관찰 구간 동안 얼마나 바빴는지 파악하는 데 사용합니다.

### H3. 호출 시점의 스냅샷이라는 점을 이해한다

`performance.toJSON()`의 결과는 고정된 설정값이 아니라 호출 시점의 상태입니다.
프로세스가 실행되는 동안 `nodeTiming.duration`, `nodeTiming.idleTime`, `eventLoopUtilization` 같은 값은 달라질 수 있습니다.

```js
import { performance } from 'node:perf_hooks';
import { setTimeout as delay } from 'node:timers/promises';

const before = performance.toJSON();

await delay(100);

const after = performance.toJSON();

console.log({
  beforeDuration: before.nodeTiming.duration,
  afterDuration: after.nodeTiming.duration,
  beforeUtilization: before.eventLoopUtilization.utilization,
  afterUtilization: after.eventLoopUtilization.utilization
});
```

따라서 결과를 저장할 때는 스냅샷을 수집한 시각과 목적을 함께 기록해야 합니다.
두 스냅샷의 숫자가 다르다는 사실만으로 성능 회귀라고 판단해서도 안 됩니다.

## 기본 직렬화 방법

### H3. toJSON 반환값을 JSON 문자열로 변환한다

`performance.toJSON()`의 반환값은 `JSON.stringify()`로 직렬화할 수 있습니다.
로그 수집기나 진단 파일로 전달하기 전에 문자열 형태가 필요하다면 다음처럼 사용할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

const payload = JSON.stringify(performance.toJSON());

console.log(payload);
```

들여쓰기가 적용된 출력은 로컬 진단에는 읽기 쉽지만, 운영 로그에서는 데이터 크기를 늘릴 수 있습니다.

```js
const readablePayload = JSON.stringify(performance.toJSON(), null, 2);
console.log(readablePayload);
```

운영 환경에서는 한 줄 JSON이나 구조화 로그 객체를 사용하고, 사람이 직접 확인하는 진단 명령에서만 들여쓰기를 적용하는 편이 실용적입니다.

### H3. JSON.stringify가 toJSON을 자동 호출할 수 있다

JavaScript의 `JSON.stringify()`는 대상 객체에 `toJSON()` 메서드가 있으면 해당 메서드의 반환값을 직렬화에 사용합니다.
따라서 `performance` 객체 자체를 전달해도 런타임 스냅샷을 문자열로 만들 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

console.log(JSON.stringify(performance));
```

동작은 간단하지만 코드만 읽었을 때 어떤 필드가 직렬화되는지 명확하지 않을 수 있습니다.
공용 로깅 함수에서는 `performance.toJSON()`을 명시적으로 호출하고, 반환값을 선별하는 방식을 권장합니다.

## 반환값을 올바르게 해석하기

### H3. nodeTiming으로 시작 단계를 확인한다

`nodeTiming`에는 Node.js 런타임 시작 과정의 여러 상대 시간 값이 포함됩니다.
각 값은 `timeOrigin`을 기준으로 한 밀리초 단위이며, 프로세스가 아직 특정 단계에 도달하지 않았다면 일부 필드가 음수일 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

const { nodeTiming } = performance.toJSON();

console.log({
  nodeStartMs: nodeTiming.nodeStart,
  v8StartMs: nodeTiming.v8Start,
  bootstrapCompleteMs: nodeTiming.bootstrapComplete,
  loopStartMs: nodeTiming.loopStart,
  durationMs: nodeTiming.duration
});
```

`nodeTiming.duration`은 애플리케이션 요청 한 건의 처리 시간이 아닙니다.
런타임 타임라인의 지속 시간에 가까운 값이므로 API 지연 시간이나 작업 처리 시간을 대신하는 지표로 사용하면 안 됩니다.

### H3. timeOrigin으로 상대 시간을 절대 시각에 연결한다

`timeOrigin`은 현재 성능 타임라인의 기준이 되는 Unix 밀리초 값입니다.
`performance.now()`나 `PerformanceEntry.startTime` 같은 상대 시간에 더하면 절대 타임스탬프로 변환할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

const { timeOrigin } = performance.toJSON();
const capturedAtMs = timeOrigin + performance.now();

console.log({
  timeOrigin: new Date(timeOrigin).toISOString(),
  capturedAt: new Date(capturedAtMs).toISOString()
});
```

상대 시간과 절대 시간의 역할을 구분하는 방법은 [Node.js performance.timeOrigin 가이드](/development/blog/seo/2026/06/14/nodejs-performance-timeorigin-absolute-timestamp-guide.html)에서 자세히 확인할 수 있습니다.

### H3. eventLoopUtilization은 누적 상태로 해석한다

`performance.toJSON()`에 포함된 `eventLoopUtilization`은 현재 시점까지 관찰된 이벤트 루프 상태를 보여 줍니다.
한 번의 스냅샷만으로 특정 1분 구간의 부하를 정확히 판단하기는 어렵습니다.

```js
import { performance } from 'node:perf_hooks';

const { eventLoopUtilization } = performance.toJSON();

console.log({
  idleMs: eventLoopUtilization.idle,
  activeMs: eventLoopUtilization.active,
  utilization: Number(eventLoopUtilization.utilization.toFixed(4))
});
```

구간별 활용률이 필요하다면 이전 값을 보관하고 `performance.eventLoopUtilization(previous)`로 차이를 계산해야 합니다.
구간 비교와 해석 기준은 [Node.js performance.eventLoopUtilization 가이드](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)를 참고하세요.

## 운영 로그에 필요한 필드만 선별하기

### H3. 전체 객체보다 목적에 맞는 요약을 만든다

`performance.toJSON()` 결과 전체를 매 요청마다 기록하면 로그 크기가 불필요하게 커집니다.
런타임 진단 목적이라면 낮은 빈도로 수집하고, 대시보드나 알림에 필요한 필드만 안정적인 이름으로 변환하는 편이 좋습니다.

```js
import { performance } from 'node:perf_hooks';

function createRuntimePerformanceSummary() {
  const snapshot = performance.toJSON();
  const capturedAtMs = snapshot.timeOrigin + performance.now();

  return {
    event: 'runtime_performance_snapshot',
    capturedAt: new Date(capturedAtMs).toISOString(),
    processUptimeMs: Math.round(snapshot.nodeTiming.duration),
    eventLoopIdleMs: Math.round(snapshot.eventLoopUtilization.idle),
    eventLoopActiveMs: Math.round(snapshot.eventLoopUtilization.active),
    eventLoopUtilization: Number(
      snapshot.eventLoopUtilization.utilization.toFixed(4)
    )
  };
}

console.log(createRuntimePerformanceSummary());
```

필드 이름에 단위를 포함하면 초와 밀리초를 혼동하는 오류를 줄일 수 있습니다.
활용률은 원본 숫자를 내부 계산에 유지하되, 로그에서는 필요한 정밀도만 남길 수 있습니다.

### H3. 민감정보를 합쳐 넣지 않는다

`performance.toJSON()`의 기본 반환값은 런타임 성능 정보이지만, 이를 감싸는 로그 객체에 요청 헤더나 환경 변수 전체를 추가하면 민감정보가 노출될 수 있습니다.

```js
const logEntry = {
  service: 'report-worker',
  ...createRuntimePerformanceSummary()
};

console.log(logEntry);
```

서비스 이름처럼 고정되고 공개 가능한 식별자만 추가하세요.
API 키, 토큰, 연결 문자열, 사용자 이메일, 요청 본문을 진단 스냅샷에 함께 기록하지 않아야 합니다.

## 주기적으로 스냅샷 수집하기

### H3. 수집 주기와 로그 비용을 함께 관리한다

런타임 상태를 주기적으로 기록하면 장시간 실행되는 프로세스의 변화를 확인할 수 있습니다.
다만 너무 짧은 간격은 로그 비용을 늘리고 문제 분석에 필요한 신호를 오히려 흐릴 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

function createSnapshot() {
  const snapshot = performance.toJSON();

  return {
    event: 'runtime_performance_snapshot',
    observedAt: new Date(
      snapshot.timeOrigin + performance.now()
    ).toISOString(),
    nodeDurationMs: Math.round(snapshot.nodeTiming.duration),
    eventLoopUtilization: Number(
      snapshot.eventLoopUtilization.utilization.toFixed(4)
    )
  };
}

const interval = setInterval(() => {
  console.log(createSnapshot());
}, 60_000);

interval.unref();
```

`unref()`를 호출하면 이 타이머만 남았을 때 프로세스가 종료되는 것을 방해하지 않습니다.
실제 운영에서는 구조화 로거와 메트릭 시스템을 사용하고, 수집 주기는 장애 탐지 목적과 저장 비용을 기준으로 정해야 합니다.

### H3. 개별 요청 지연 시간은 별도로 측정한다

`performance.toJSON()`은 런타임 전체 상태를 요약하지만 특정 함수나 요청의 실행 시간을 제공하지 않습니다.
개별 작업 지연 시간은 `performance.mark()`와 `performance.measure()` 또는 전용 메트릭으로 측정해야 합니다.

사용자 타이밍 항목을 만들고 조회하는 기본 패턴은 [Node.js performance.mark·measure 가이드](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)에서 확인할 수 있습니다.

또한 `performance.toJSON()` 결과에는 직접 만든 mark와 measure 목록이 포함되지 않습니다.
해당 항목이 필요하다면 `performance.getEntriesByType()`이나 `PerformanceObserver`를 별도로 사용해야 합니다.

## 테스트와 검증 전략

### H3. 정확한 시간값 대신 구조와 범위를 검증한다

성능 관련 숫자는 실행 환경과 시점에 따라 달라집니다.
테스트에서는 특정 밀리초 값을 고정하기보다 필요한 필드가 존재하고 유효한 범위인지 확인해야 합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { performance } from 'node:perf_hooks';

test('performance.toJSON returns a runtime snapshot', () => {
  const snapshot = performance.toJSON();

  assert.equal(typeof snapshot.timeOrigin, 'number');
  assert.equal(typeof snapshot.nodeTiming, 'object');
  assert.equal(typeof snapshot.eventLoopUtilization, 'object');
  assert.ok(snapshot.nodeTiming.duration >= 0);
  assert.ok(snapshot.eventLoopUtilization.utilization >= 0);
  assert.ok(snapshot.eventLoopUtilization.utilization <= 1);
});
```

Node.js 버전에 따라 반환 객체의 세부 필드가 달라질 가능성을 고려해야 합니다.
외부 시스템과 엄격한 스키마 계약을 맺는다면 필요한 값만 자체 스키마로 변환하고, 런타임 업그레이드 전에 호환성 테스트를 실행하세요.

### H3. 직렬화 가능 여부를 함께 확인한다

진단 데이터를 외부 로그 시스템으로 보낼 예정이라면 직렬화 결과도 테스트할 수 있습니다.

```js
test('performance snapshot is JSON serializable', () => {
  const json = JSON.stringify(performance.toJSON());
  const parsed = JSON.parse(json);

  assert.equal(typeof parsed.timeOrigin, 'number');
  assert.equal(typeof parsed.nodeTiming.duration, 'number');
  assert.equal(
    typeof parsed.eventLoopUtilization.utilization,
    'number'
  );
});
```

이 테스트는 실제 로그 전송 성공을 보장하지는 않습니다.
전송 크기 제한, 필드 이름 규칙, 샘플링과 보존 정책은 사용하는 로그 시스템에서 별도로 검증해야 합니다.

## 운영 적용 체크리스트

### H3. 스냅샷의 목적과 경계를 먼저 정한다

`performance.toJSON()`은 런타임 상태를 빠르게 직렬화하는 출발점입니다.
다음 항목을 확인하면 스냅샷을 과도하게 수집하거나 잘못 해석하는 문제를 줄일 수 있습니다.

- 스냅샷을 시작 진단, 주기 모니터링, 장애 분석 중 어디에 사용할지 정했는가?
- 호출 시점에 따라 값이 달라지는 현재 상태라는 점을 반영했는가?
- 필요한 필드만 자체 로그 스키마로 변환했는가?
- 밀리초와 활용률 등 단위를 필드 이름에 드러냈는가?
- 개별 요청 지연 시간과 런타임 전체 상태를 구분했는가?
- mark와 measure 목록이 결과에 포함되지 않는다는 점을 확인했는가?
- 로그 객체에 토큰, 개인정보, 환경 변수 전체를 추가하지 않았는가?
- 수집 빈도와 로그 저장 비용을 함께 검토했는가?
- Node.js 업그레이드 전에 반환 구조 호환성을 테스트하는가?

## 자주 묻는 질문

### H3. JSON.stringify(performance)와 JSON.stringify(performance.toJSON())은 무엇이 다른가요?

`JSON.stringify(performance)`는 직렬화 과정에서 `performance.toJSON()`을 자동 호출할 수 있습니다.
명시적으로 `toJSON()`을 호출하면 어떤 데이터를 다루는지 코드에서 더 분명하게 드러나고, 반환값을 선별하거나 가공하기도 쉽습니다.

### H3. performance.toJSON에 mark와 measure 항목도 포함되나요?

포함되지 않습니다.
직접 만든 성능 타임라인 항목은 `performance.getEntriesByType()`, `performance.getEntriesByName()`, `PerformanceObserver` 같은 API로 별도 조회해야 합니다.

### H3. performance.toJSON만으로 이벤트 루프 부하 알림을 만들 수 있나요?

현재까지의 누적 상태를 간단히 확인하는 데는 사용할 수 있지만, 안정적인 구간 알림에는 부족할 수 있습니다.
이전 스냅샷과의 차이, 이벤트 루프 지연, CPU 사용량, 요청 지연 같은 다른 지표를 함께 관찰해야 합니다.

## 마무리

`performance.toJSON()`은 Node.js 프로세스의 `nodeTiming`, `timeOrigin`, `eventLoopUtilization`을 하나의 직렬화 가능한 런타임 스냅샷으로 확인하는 간단한 방법입니다.
빠른 진단과 기본 로그 구조를 만드는 데 유용하지만, 개별 요청 지연 시간이나 사용자 타이밍 항목을 대신하지는 않습니다.

운영 환경에서는 전체 반환값을 무조건 기록하기보다 목적에 필요한 필드만 안정적인 스키마로 변환하세요.
호출 시점과 단위를 함께 기록하고, 수집 빈도와 민감정보 경계를 관리하면 런타임 성능 스냅샷을 신뢰할 수 있는 관찰 데이터로 활용할 수 있습니다.

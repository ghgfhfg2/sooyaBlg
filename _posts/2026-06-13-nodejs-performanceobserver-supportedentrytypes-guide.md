---
layout: post
title: "Node.js PerformanceObserver.supportedEntryTypes 가이드: 지원 성능 항목 확인하기"
date: 2026-06-13 20:00:00 +0900
lang: ko
translation_key: nodejs-performanceobserver-supportedentrytypes-guide
permalink: /development/blog/seo/2026/06/13/nodejs-performanceobserver-supportedentrytypes-guide.html
alternates:
  ko: /development/blog/seo/2026/06/13/nodejs-performanceobserver-supportedentrytypes-guide.html
  x_default: /development/blog/seo/2026/06/13/nodejs-performanceobserver-supportedentrytypes-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performanceobserver, perf-hooks, monitoring, observability]
description: "Node.js PerformanceObserver.supportedEntryTypes로 현재 런타임이 지원하는 성능 항목을 확인하고, 환경 차이를 고려해 관찰기를 안전하게 구성하는 방법을 설명합니다."
---

Node.js의 `PerformanceObserver`로 성능 데이터를 수집할 때는 런타임이 원하는 항목 유형을 실제로 지원하는지 먼저 확인해야 합니다.
지원 여부를 가정한 채 관찰기를 등록하면 Node.js 버전이나 실행 환경이 바뀌었을 때 계측 코드가 실패하거나 필요한 데이터가 조용히 빠질 수 있습니다.

`PerformanceObserver.supportedEntryTypes`는 현재 런타임에서 관찰할 수 있는 성능 항목 유형 목록을 제공합니다.
이 목록으로 시작 단계에서 필수 항목을 검증하고, 선택 항목은 지원될 때만 등록하면 계측 기능의 이식성과 예측 가능성을 높일 수 있습니다.

이 글에서는 지원 항목 조회, 필수·선택 항목 구분, 관찰기 구성, 테스트와 운영 점검 방법을 정리합니다.
사용자 코드 구간을 직접 측정하려면 [Node.js performance.mark·measure 가이드](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)도 함께 참고하세요.

## supportedEntryTypes가 필요한 이유

### H3. 지원 항목은 런타임에 따라 달라질 수 있다

`PerformanceObserver`는 `mark`, `measure`, `function`, `gc`처럼 서로 다른 성능 항목을 관찰할 수 있습니다.
그러나 실제 지원 목록은 Node.js 버전과 런타임 구현에 따라 달라질 수 있으므로 특정 목록을 모든 환경에서 동일하다고 가정하면 안 됩니다.

다음 코드는 현재 프로세스가 지원하는 항목을 출력합니다.

```js
import { PerformanceObserver } from 'node:perf_hooks';

console.log(PerformanceObserver.supportedEntryTypes);
```

반환값은 문자열 배열입니다.
배포 환경에서 직접 출력하거나 테스트로 검증하면 문서나 개발 환경의 기억에 의존하지 않고 현재 런타임의 기능을 확인할 수 있습니다.

### H3. 계측 기능도 호환성 검사가 필요하다

성능 계측은 애플리케이션의 핵심 요청 처리와 분리된 보조 기능이지만, 잘못 구성하면 시작 실패나 진단 공백을 만들 수 있습니다.
특히 여러 Node.js 버전을 동시에 운영하거나 점진적으로 업그레이드할 때는 지원 항목 검사가 유용합니다.

지원 여부에 따라 행동을 구분하면 운영 정책도 명확해집니다.

- 반드시 필요한 항목이 없으면 시작 단계에서 명확한 오류를 남긴다.
- 선택 항목이 없으면 경고만 남기고 나머지 계측을 계속한다.
- 지원 항목 목록을 런타임 버전과 함께 기록해 환경 차이를 추적한다.

## 지원 항목 조회하고 검증하기

### H3. Set으로 변환해 반복 검사를 단순화한다

배열을 `Set`으로 변환하면 여러 항목의 지원 여부를 읽기 쉽게 검사할 수 있습니다.

```js
import { PerformanceObserver } from 'node:perf_hooks';

const supported = new Set(PerformanceObserver.supportedEntryTypes);

console.log({
  supportsMeasure: supported.has('measure'),
  supportsFunction: supported.has('function'),
  supportsGc: supported.has('gc')
});
```

항목 이름은 고정된 문자열을 사용해야 합니다.
사용자 입력이나 요청 데이터를 항목 이름으로 사용하면 검증이 불안정해지고 로그와 메트릭의 카디널리티도 커질 수 있습니다.

### H3. 필수 항목과 선택 항목을 분리한다

모든 성능 항목을 필수로 취급하면 부가 계측 하나가 지원되지 않는다는 이유로 서비스 전체가 시작되지 않을 수 있습니다.
반대로 필요한 항목이 없어도 무조건 계속 실행하면 관측이 중단된 사실을 늦게 발견할 수 있습니다.

```js
import { PerformanceObserver } from 'node:perf_hooks';

const supported = new Set(PerformanceObserver.supportedEntryTypes);
const requiredTypes = ['mark', 'measure'];
const optionalTypes = ['gc', 'function'];

const missingRequired = requiredTypes.filter((type) => !supported.has(type));
const enabledOptional = optionalTypes.filter((type) => supported.has(type));

if (missingRequired.length > 0) {
  throw new Error(
    `Required performance entry types are unavailable: ${missingRequired.join(', ')}`
  );
}

console.log({
  event: 'performance_observer_capabilities',
  enabledOptional
});
```

필수 여부는 계측 목적에 따라 결정해야 합니다.
성능 데이터가 없어도 요청을 안전하게 처리할 수 있다면 시작 실패 대신 경고와 상태 지표를 사용하는 편이 적절할 수 있습니다.

## 지원되는 항목만 관찰하기

### H3. entryTypes 목록을 동적으로 구성한다

관찰하려는 후보 중 현재 런타임이 지원하는 항목만 골라 `observe()`에 전달할 수 있습니다.

```js
import { PerformanceObserver, performance } from 'node:perf_hooks';

const supported = new Set(PerformanceObserver.supportedEntryTypes);
const candidates = ['measure', 'function', 'gc'];
const entryTypes = candidates.filter((type) => supported.has(type));

if (entryTypes.length > 0) {
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      console.log({
        name: entry.name,
        entryType: entry.entryType,
        durationMs: Number(entry.duration.toFixed(2))
      });
    }
  });

  observer.observe({ entryTypes });

  performance.mark('example:start');
  await Promise.resolve();
  performance.mark('example:end');
  performance.measure('example', 'example:start', 'example:end');
}
```

예제는 지원 항목 선택과 관찰기 등록 흐름을 보여 줍니다.
운영 환경에서는 개별 항목을 모두 로그로 남기기보다 집계하거나 샘플링해 로그 양과 비용을 제한해야 합니다.

### H3. 빈 목록으로 관찰기를 등록하지 않는다

후보 항목이 모두 지원되지 않는 환경에서는 관찰기를 등록할 이유가 없습니다.
빈 목록을 명시적으로 처리하면 계측이 비활성화된 이유를 코드와 로그에서 쉽게 확인할 수 있습니다.

```js
function selectSupportedEntryTypes(candidates) {
  const supported = new Set(PerformanceObserver.supportedEntryTypes);
  return candidates.filter((type) => supported.has(type));
}

const entryTypes = selectSupportedEntryTypes(['measure', 'gc']);

if (entryTypes.length === 0) {
  console.warn({
    event: 'performance_observer_disabled',
    reason: 'no_supported_entry_types'
  });
}
```

경고에는 토큰, 연결 문자열, 사용자 정보 같은 민감정보를 넣지 않아야 합니다.
지원 항목과 런타임 버전처럼 원인 파악에 필요한 최소 정보만 기록하는 것으로 충분합니다.

## 테스트로 런타임 요구사항 고정하기

### H3. CI에서 필수 항목을 검증한다

서비스가 `mark`와 `measure`에 의존한다면 CI에서 해당 항목의 지원 여부를 확인할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { PerformanceObserver } from 'node:perf_hooks';

test('required performance entry types are supported', () => {
  const supported = new Set(PerformanceObserver.supportedEntryTypes);

  assert.equal(supported.has('mark'), true);
  assert.equal(supported.has('measure'), true);
});
```

이 테스트는 실제 배포 환경과 같은 Node.js 메이저 버전에서 실행해야 의미가 있습니다.
개발 환경에서만 통과한 결과로 운영 환경의 지원 여부까지 보장할 수는 없습니다.

### H3. Node.js 업그레이드 전후 목록을 비교한다

Node.js 버전을 변경할 때는 애플리케이션 테스트뿐 아니라 지원 성능 항목 목록도 비교하는 편이 좋습니다.
목록 변화가 확인되면 관찰기 구성, 대시보드, 경고 정책이 새 환경에서도 유효한지 점검해야 합니다.

함수 호출 시간을 관찰하는 구성은 [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)에서 확인할 수 있습니다.
GC 항목을 운영 지표로 연결하려면 [Node.js PerformanceObserver GC 가이드](/development/blog/seo/2026/06/12/nodejs-performanceobserver-gc-pause-monitoring-guide.html)를 함께 참고하세요.

## 운영 환경에서의 주의점

### H3. 지원 여부와 데이터 생성 여부는 다르다

항목이 지원 목록에 있다는 사실은 해당 데이터가 항상 생성된다는 뜻이 아닙니다.
예를 들어 특정 기능을 실행하거나 측정 API를 호출해야 성능 항목이 만들어질 수 있습니다.

따라서 시작 시점의 지원 여부와 실행 중 실제 수집량을 별도로 관찰해야 합니다.
지원 항목인데 수집량이 계속 0이라면 계측 경로가 실행되는지, 관찰기 등록 시점이 적절한지 확인해야 합니다.

### H3. 계측 실패가 서비스 오류를 가리지 않게 한다

관찰기 콜백에서 무거운 처리나 동기 I/O를 실행하면 측정 대상 자체의 성능에 영향을 줄 수 있습니다.
콜백에서는 필요한 값만 빠르게 추출하고, 집계와 전송은 제한된 큐나 별도 처리 경로로 넘기는 편이 안전합니다.

계측 전송 실패를 처리할 때도 원래 애플리케이션 오류를 덮어쓰지 않아야 합니다.
성능 관찰기의 비용과 이벤트 루프 부하를 함께 살펴보려면 [Node.js performance.eventLoopUtilization 가이드](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)를 참고하세요.

## 자주 묻는 질문

### H3. supportedEntryTypes 목록을 코드에 복사해도 되나요?

권장하지 않습니다.
목록을 코드에 고정하면 런타임이 바뀌었을 때 실제 지원 상태와 어긋날 수 있으므로 현재 프로세스의 `PerformanceObserver.supportedEntryTypes`를 직접 조회하는 편이 안전합니다.

### H3. 지원되지 않는 선택 항목 때문에 서비스를 중단해야 하나요?

대부분의 경우에는 경고를 남기고 지원되는 계측만 계속하는 편이 적절합니다.
다만 규정이나 운영 요구사항상 해당 데이터가 반드시 필요하다면 시작 전 검증 단계에서 실패하도록 정책을 명확히 정할 수 있습니다.

### H3. 지원 목록에 있으면 관찰 결과가 즉시 들어오나요?

아닙니다.
지원 여부는 관찰 가능한 항목 유형이라는 뜻이며, 실제 항목이 생성되어야 콜백으로 전달됩니다.
측정 API 호출, 관찰기 등록 시점, 실행 경로를 함께 확인해야 합니다.

## 적용 체크리스트

- [ ] 시작 시 현재 런타임의 `supportedEntryTypes`를 확인한다.
- [ ] 필수 항목과 선택 항목의 실패 정책을 분리한다.
- [ ] 지원되는 항목만 관찰기에 등록한다.
- [ ] 빈 항목 목록과 계측 비활성 상태를 명시적으로 처리한다.
- [ ] 실제 배포 Node.js 버전과 같은 환경에서 CI 검증을 실행한다.
- [ ] 로그에 토큰, 연결 문자열, 사용자 정보를 넣지 않는다.
- [ ] 지원 여부와 실제 수집량을 별도로 관찰한다.

## 마무리

`PerformanceObserver.supportedEntryTypes`는 Node.js 성능 계측을 현재 런타임의 실제 기능에 맞게 구성하는 간단한 출발점입니다.
필수·선택 항목을 나누고 지원되는 유형만 등록하면 버전 차이 때문에 계측이 깨지거나 조용히 빠지는 위험을 줄일 수 있습니다.

성능 관찰 기능도 애플리케이션의 다른 의존성과 마찬가지로 검증과 운영 정책이 필요합니다.
지원 목록, 실제 수집량, 계측 비용을 함께 확인하면 런타임 업그레이드 이후에도 신뢰할 수 있는 진단 흐름을 유지할 수 있습니다.

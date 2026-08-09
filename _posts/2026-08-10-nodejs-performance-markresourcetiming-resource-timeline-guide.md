---
layout: post
title: "Node.js markResourceTiming 가이드: 외부 호출 성능을 Resource Timing으로 남기는 법"
date: 2026-08-10 08:00:00 +0900
lang: ko
translation_key: nodejs-performance-markresourcetiming-resource-timeline-guide
permalink: /development/blog/seo/2026/08/10/nodejs-performance-markresourcetiming-resource-timeline-guide.html
alternates:
  ko: /development/blog/seo/2026/08/10/nodejs-performance-markresourcetiming-resource-timeline-guide.html
  x_default: /development/blog/seo/2026/08/10/nodejs-performance-markresourcetiming-resource-timeline-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, performance, perf-hooks, markresourcetiming, resource-timing, observability, fetch, backend]
description: "Node.js performance.markResourceTiming으로 외부 API 호출과 파일 다운로드 같은 리소스 성능을 Resource Timing 항목으로 남기는 방법을 정리합니다. 버전 호환성, URL 정규화, buffer 관리, 민감정보 점검까지 실무 예제로 설명합니다."
---

외부 API 호출이 느려졌을 때 보통은 요청 시작 시각과 종료 시각을 직접 로그에 남깁니다.
이 방식도 충분히 유용하지만, 애플리케이션 내부 구간 측정은 `measure`, 함수 계측은 `function`, 리소스 호출은 `resource`처럼 성능 타임라인의 항목 유형을 맞춰 두면 이후 수집과 분석 코드가 훨씬 단순해집니다.

Node.js의 `performance.markResourceTiming()`은 수동으로 `PerformanceResourceTiming` 항목을 Resource Timeline에 추가하는 API입니다.
브라우저의 Resource Timing 모델에 맞춰 `fetch`, 이미지 변환 입력 파일, 외부 설정 다운로드처럼 "리소스를 가져오는 작업"을 하나의 성능 항목으로 남길 수 있습니다.
이 글에서는 `markResourceTiming()`을 바로 도입할 때 확인해야 할 버전 호환성, 이름 설계, buffer 관리, 민감정보 점검 기준을 정리합니다.

성능 타임라인을 조회하는 기본 흐름은 [Node.js performance.getEntriesByType 가이드](/development/blog/seo/2026/06/14/nodejs-performance-getentriesbytype-timeline-query-guide.html)를 먼저 보면 좋습니다.
사용자 코드 구간을 직접 측정해야 한다면 [Node.js performance.mark·measure 가이드](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)와 함께 설계하세요.

## markResourceTiming이 필요한 상황

### H3. 외부 호출을 일반 measure와 분리한다

`performance.measure()`는 애플리케이션이 의미를 붙인 구간을 측정하기에 좋습니다.
예를 들어 `checkout:validate`, `report:render`, `cache:warm`처럼 도메인 단계가 분명한 작업에 잘 맞습니다.

반면 외부 HTTP 호출, 원격 파일 다운로드, 내부 서비스 fetch처럼 "어떤 리소스를 가져왔는가"가 핵심인 작업은 Resource Timing 항목으로 분리하면 읽기 쉽습니다.

```js
import { performance } from 'node:perf_hooks';

performance.mark('profile:api:start');
const response = await fetch('https://api.example.com/profile');
performance.mark('profile:api:end');

performance.measure('profile:api', 'profile:api:start', 'profile:api:end');

console.log(performance.getEntriesByType('measure'));
```

이 코드는 전체 시간은 볼 수 있지만, 나중에 `resource` 항목만 골라 외부 호출 목록을 만들기는 어렵습니다.
`markResourceTiming()`을 쓰면 리소스 호출을 성능 타임라인에서 별도 유형으로 조회할 수 있습니다.

```js
const resources = performance.getEntriesByType('resource');
```

대시보드나 진단 명령에서 `measure`와 `resource`를 분리해 보여 주면 "내부 처리 지연"과 "외부 리소스 지연"을 더 빨리 나눌 수 있습니다.

### H3. 자동 계측이 없는 경계를 수동으로 보완한다

모든 라이브러리가 Node.js 성능 타임라인에 Resource Timing을 자동으로 남기는 것은 아닙니다.
직접 만든 fetch wrapper, 내부 SDK, 파일 다운로드 유틸리티, 배치 작업의 원격 설정 로더처럼 호출 경계가 애플리케이션 코드 안에 있다면 수동 계측이 필요합니다.

이때 선택지는 두 가지입니다.

- `performance.mark()`와 `performance.measure()`로 일반 구간 측정을 남긴다.
- `performance.markResourceTiming()`으로 Resource Timeline에 리소스 항목을 남긴다.

팀에서 이미 `PerformanceObserver`로 `resource` 항목을 수집하고 있다면 두 번째 방식이 더 자연스럽습니다.
반대로 아직 Resource Timing 수집 경로가 없다면 먼저 `measure`로 단순하게 시작하고, 외부 호출 분석 요구가 커졌을 때 `resource` 항목을 추가해도 됩니다.

## 기본 설계

### H3. 지원 여부와 Node.js 버전을 먼저 확인한다

`markResourceTiming()`은 Node.js의 `node:perf_hooks`가 제공하는 확장 API입니다.
공식 문서 기준으로 이 API는 브라우저에서 제공되는 표준 메서드가 아니라 Node.js 전용 확장이며, `bodyInfo`, `responseStatus`, `deliveryType` 인자는 Node.js 22.2.0 이후 추가되었습니다.

운영 코드에서는 런타임 지원 여부를 시작 단계에서 확인하는 편이 안전합니다.

```js
import { performance } from 'node:perf_hooks';

export function supportsResourceTimingMark() {
  return typeof performance.markResourceTiming === 'function';
}

if (!supportsResourceTimingMark()) {
  console.warn('performance.markResourceTiming is not available in this runtime');
}
```

여러 Node.js 버전을 동시에 운영한다면 계측 코드를 필수 경로로 두지 마세요.
외부 호출 자체는 계속 수행하고, Resource Timing 기록만 비활성화하는 fallback을 두는 편이 장애 범위를 줄입니다.

### H3. URL 이름은 정규화해서 카디널리티를 낮춘다

Resource Timing 항목의 `name`은 보통 리소스 URL입니다.
하지만 실제 요청 URL을 그대로 넣으면 사용자 ID, 검색어, 토큰성 query string, 페이지 번호가 모두 다른 항목으로 쌓일 수 있습니다.
관측 데이터에서는 이런 고카디널리티 이름이 비용과 분석 난이도를 크게 올립니다.

아래처럼 URL을 계측용 이름으로 정규화해 두는 편이 좋습니다.

```js
export function normalizeResourceName(input) {
  const url = new URL(input);

  url.username = '';
  url.password = '';
  url.search = '';
  url.hash = '';

  return `${url.origin}${url.pathname}`;
}
```

검색어 자체가 성능 분석에 필요하다면 원문 query를 이름에 넣기보다 허용된 작은 분류값만 별도 필드로 남기세요.
로그와 메트릭에서 민감정보를 줄이는 원칙은 [CLI 출력값 민감정보 제거 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)와 같습니다.

### H3. timingInfo는 가능한 값만 보수적으로 채운다

`markResourceTiming()`은 Fetch Timing Info에 해당하는 `timingInfo` 객체를 받습니다.
하지만 애플리케이션 wrapper가 DNS, TCP 연결, TLS, request start, response start를 모두 알 수 있는 경우는 많지 않습니다.
모르는 값을 억지로 추정하면 성능 데이터가 더 위험해집니다.

실무에서는 두 가지 원칙을 권합니다.

- 정확히 측정한 값만 채운다.
- 세부 단계가 없으면 전체 duration은 `measure` 또는 별도 메트릭으로 보완한다.

예를 들어 단순 fetch wrapper는 시작과 종료 시각만 확실히 압니다.
이 경우 Resource Timing은 "이 URL에 대한 호출이 있었다"는 항목으로 쓰고, 세부 latency는 wrapper 로그나 `performance.measure()`로 함께 남기는 방식이 현실적입니다.

```js
import { performance } from 'node:perf_hooks';

export async function measuredFetch(url, options) {
  const resourceName = normalizeResourceName(url);
  const start = performance.now();

  try {
    const response = await fetch(url, options);
    const end = performance.now();

    performance.measure(`fetch:${resourceName}`, {
      start,
      end,
      detail: {
        status: response.status
      }
    });

    markFetchResourceTiming({
      resourceName,
      start,
      end,
      status: response.status
    });

    return response;
  } catch (error) {
    const end = performance.now();

    performance.measure(`fetch:${resourceName}:error`, {
      start,
      end
    });

    throw error;
  }
}
```

`markFetchResourceTiming()`은 런타임별 차이를 감싸는 얇은 adapter로 두면 좋습니다.
Node.js 버전이 올라가 인자 구조가 바뀌어도 호출부 전체를 흔들지 않아도 됩니다.

## Resource Timing 기록하기

### H3. adapter 함수로 호출부를 격리한다

`markResourceTiming()`은 저수준 API에 가깝습니다.
서비스 코드 곳곳에서 직접 호출하기보다, 한 파일에 adapter를 두고 이름 정규화와 지원 여부 확인을 모으는 편이 안전합니다.

```js
import { performance } from 'node:perf_hooks';

export function markFetchResourceTiming({ resourceName, start, end, status }) {
  if (typeof performance.markResourceTiming !== 'function') {
    return null;
  }

  const timingInfo = {
    startTime: start,
    endTime: end,
    finalServiceWorkerStartTime: 0,
    redirectStartTime: 0,
    redirectEndTime: 0,
    postRedirectStartTime: start,
    finalNetworkRequestStartTime: start,
    finalNetworkResponseStartTime: end,
    encodedBodySize: 0,
    decodedBodySize: 0
  };

  const bodyInfo = {
    encodedBodySize: 0,
    decodedBodySize: 0,
    transferSize: 0
  };

  return performance.markResourceTiming(
    timingInfo,
    resourceName,
    'fetch',
    globalThis,
    '',
    bodyInfo,
    status
  );
}
```

이 예제는 fetch 세부 단계가 없는 wrapper를 전제로 합니다.
DNS나 TLS 시간을 알 수 없다면 빈 값을 추정하지 않고, 전체 호출 시각만 일관되게 넣습니다.
더 자세한 단계가 필요한 환경에서는 HTTP 클라이언트가 제공하는 timing hook을 확인한 뒤 adapter 내부에서만 확장하세요.

### H3. PerformanceObserver로 resource 항목을 수집한다

Resource Timing을 남겼다면 `PerformanceObserver`로 `resource` 항목을 수집할 수 있습니다.
지원 항목은 런타임마다 다를 수 있으므로 `supportedEntryTypes`로 확인한 뒤 등록하면 좋습니다.

```js
import { PerformanceObserver } from 'node:perf_hooks';

export function startResourceTimingObserver({ logger }) {
  if (!PerformanceObserver.supportedEntryTypes.includes('resource')) {
    logger.warn('resource timing entries are not supported');
    return () => {};
  }

  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      logger.info({
        type: 'resource_timing',
        name: entry.name,
        initiatorType: entry.initiatorType,
        durationMs: Number(entry.duration.toFixed(2)),
        responseStatus: entry.responseStatus
      });
    }
  });

  observer.observe({ type: 'resource', buffered: true });

  return () => observer.disconnect();
}
```

`buffered: true`를 쓰면 관찰기를 등록하기 전에 이미 생성된 항목도 받을 수 있습니다.
장기 실행 프로세스에서는 관찰기를 한 번만 등록하고, 종료 시 `disconnect()`를 호출하는 구조가 좋습니다.

### H3. resource buffer 크기를 관리한다

Resource Timing 항목은 전역 resource timeline buffer에 쌓입니다.
Node.js 문서 기준 기본 buffer 크기는 250개이며, 더 많은 항목을 담아야 한다면 `performance.setResourceTimingBufferSize()`로 조정할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

performance.setResourceTimingBufferSize(1_000);

performance.addEventListener('resourcetimingbufferfull', () => {
  console.warn('resource timing buffer is full');
  performance.clearResourceTimings();
});
```

무조건 buffer 크기를 크게 늘리는 것은 답이 아닙니다.
외부 호출이 많은 서버에서 모든 요청의 모든 리소스를 남기면 메모리와 로그 비용이 커질 수 있습니다.
처음에는 샘플링, 중요 endpoint 한정, 배치 작업 한정처럼 범위를 좁게 잡는 편이 좋습니다.

처리한 항목을 주기적으로 비우는 기준은 [Node.js performance clearMarks/clearMeasures 가이드](/development/blog/seo/2026/06/15/nodejs-performance-clearmarks-clearmeasures-timeline-cleanup-guide.html)와 비슷하게 생각할 수 있습니다.

## 운영에서 주의할 점

### H3. 계측 실패가 요청 실패가 되면 안 된다

성능 계측은 진단을 돕는 보조 기능입니다.
`markResourceTiming()` 호출이 실패하거나 런타임에서 지원되지 않는다고 해서 외부 API 호출 자체가 실패하면 안 됩니다.

adapter 내부에서 예외를 흡수하고, 필요한 경우 낮은 빈도의 경고만 남기세요.

```js
export function safeMarkResourceTiming(payload, logger) {
  try {
    return markFetchResourceTiming(payload);
  } catch (error) {
    logger.debug({
      event: 'resource_timing_mark_failed',
      reason: error.code ?? error.name
    });

    return null;
  }
}
```

여기서도 오류 객체 전체를 그대로 로그에 남기지 않는 편이 좋습니다.
요청 URL, header, 인증 정보가 섞여 있을 수 있기 때문입니다.

### H3. status와 URL은 공개 가능한 수준으로 줄인다

Resource Timing 로그에는 보통 `name`, `initiatorType`, `duration`, `responseStatus` 정도면 충분합니다.
원문 request body, authorization header, session cookie, 개인 식별자, 전체 query string은 남기지 마세요.

안전한 로그 예시는 아래와 같습니다.

```js
logger.info({
  event: 'resource_timing',
  resource: 'https://api.example.com/v1/profile',
  initiatorType: 'fetch',
  durationMs: 82.4,
  responseStatus: 200
});
```

위 예시는 계측용 리소스 이름이 정규화되어 있다는 전제입니다.
실제 사용자별 URL이 필요한 경우에도 저장 전에 path parameter를 템플릿으로 바꾸는 정책을 검토하세요.

```js
const resource = 'https://api.example.com/v1/users/:id/orders';
```

### H3. 자동 계측과 중복 집계를 피한다

이미 APM, OpenTelemetry instrumentation, HTTP 클라이언트 hook이 외부 호출 latency를 수집하고 있다면 `markResourceTiming()`을 추가하기 전에 중복 여부를 확인해야 합니다.
같은 호출을 두 경로에서 모두 수집하면 대시보드 숫자가 두 배로 보이거나, 서로 다른 이름 규칙 때문에 분석이 흐려질 수 있습니다.

좋은 기준은 다음과 같습니다.

- 요청 trace와 분산 추적은 OpenTelemetry 같은 tracing 계층에 맡긴다.
- Node.js 내부 성능 타임라인에서 빠르게 조회할 값은 Resource Timing으로 남긴다.
- 둘 다 남긴다면 동일한 resource naming 규칙을 공유한다.
- 샘플링 비율과 보관 기간을 문서화한다.

분산 추적과 연결하려면 [OpenTelemetry Node.js 분산 추적 가이드](/development/blog/seo/2026/03/14/opentelemetry-nodejs-distributed-tracing-practical-guide.html)를 함께 참고하세요.

## 테스트와 점검

### H3. adapter는 단위 테스트로 검증한다

서비스 테스트에서 실제 Resource Timing 항목을 많이 만들 필요는 없습니다.
adapter 함수에 `performance` 객체를 주입할 수 있게 만들고, `markResourceTiming()` 호출 인자와 fallback 동작을 검증하면 충분합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function createResourceTimingMarker(performanceLike) {
  return function markResourceTiming(payload) {
    if (typeof performanceLike.markResourceTiming !== 'function') {
      return null;
    }

    return performanceLike.markResourceTiming(payload);
  };
}

test('markResourceTiming adapter skips unsupported runtime', () => {
  const fakePerformance = {
    markResourceTiming: undefined
  };

  const result = createResourceTimingMarker(fakePerformance)({
    resourceName: 'https://api.example.com/v1/users',
    start: 10,
    end: 20,
    status: 200
  });

  assert.equal(result, null);
});
```

테스트에서 중요한 것은 Node.js 내부 구현을 복제하는 것이 아닙니다.
팀의 정책이 지켜지는지, 즉 URL이 정규화되는지, unsupported runtime에서 실패하지 않는지, 민감정보가 들어가지 않는지를 확인하는 것입니다.

### H3. 발행 전 체크리스트

- `Node.js markResourceTiming` 핵심 키워드를 제목과 설명에 포함했다.
- `performance.markResourceTiming()`을 필수 기능이 아닌 보조 계측으로 다뤘다.
- Resource Timing 이름에서 query string, hash, 인증 정보를 제거했다.
- `resource` buffer 크기와 정리 전략을 설명했다.
- 내부링크를 성능 타임라인, 로그 마스킹, 분산 추적 글과 연결했다.
- 코드 예제에 실제 토큰, 계정, 내부 호스트, 개인정보가 없다.

## FAQ

### H3. markResourceTiming은 fetch 시간을 자동으로 측정하나요?

아닙니다.
`markResourceTiming()`은 Resource Timeline에 항목을 추가하는 저수준 API입니다.
어떤 시각을 넣을지, 어떤 URL 이름을 남길지, 어떤 상태값을 기록할지는 호출 코드가 정해야 합니다.

### H3. performance.measure만 써도 되나요?

많은 경우에는 충분합니다.
외부 호출도 단순히 "이 구간이 몇 ms 걸렸나"만 필요하다면 `measure`가 더 쉽습니다.
다만 성능 타임라인에서 리소스 호출만 따로 조회하거나 `PerformanceObserver`의 `resource` 항목으로 수집하고 싶다면 `markResourceTiming()`을 검토할 수 있습니다.

### H3. 모든 외부 API 호출에 적용해야 하나요?

처음부터 모든 호출에 적용하지 않는 편이 좋습니다.
중요 endpoint, 배치 작업, 장애 분석이 자주 필요한 외부 의존성부터 시작하세요.
그 뒤 로그 비용, buffer 사용량, 대시보드 품질을 확인하며 범위를 넓히는 것이 현실적입니다.

## 마무리

`performance.markResourceTiming()`은 Node.js 애플리케이션에서 외부 리소스 호출을 성능 타임라인의 `resource` 항목으로 정리할 수 있게 해 줍니다.
잘 쓰면 일반 `measure`와 외부 호출 계측을 분리하고, 진단 명령이나 관찰기에서 리소스 성능만 빠르게 조회할 수 있습니다.

다만 이 API는 계측 보조 도구입니다.
지원 여부를 확인하고, 호출부를 adapter로 감싸고, URL을 정규화하고, buffer를 관리해야 운영에서 안전합니다.
가장 중요한 원칙은 단순합니다.
계측은 서비스 동작을 방해하지 않아야 하며, 로그와 성능 항목에는 공개 가능한 이름과 숫자만 남겨야 합니다.

## 참고 자료

- [Node.js 공식 문서: Performance measurement APIs](https://nodejs.org/api/perf_hooks.html#performancemarkresourcetimingtiminginfo-requestedurl-initiatortype-global-cachemode-bodyinfo-responsestatus-deliverytype)
- [Node.js performance.getEntriesByType 가이드](/development/blog/seo/2026/06/14/nodejs-performance-getentriesbytype-timeline-query-guide.html)
- [Node.js performance.mark·measure 가이드](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)
- [CLI 출력값 민감정보 제거 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)

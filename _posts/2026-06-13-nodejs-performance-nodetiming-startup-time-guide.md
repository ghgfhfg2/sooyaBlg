---
layout: post
title: "Node.js performance.nodeTiming 가이드: 애플리케이션 시작 시간 측정하기"
date: 2026-06-13 08:00:00 +0900
lang: ko
translation_key: nodejs-performance-nodetiming-startup-time-guide
permalink: /development/blog/seo/2026/06/13/nodejs-performance-nodetiming-startup-time-guide.html
alternates:
  ko: /development/blog/seo/2026/06/13/nodejs-performance-nodetiming-startup-time-guide.html
  x_default: /development/blog/seo/2026/06/13/nodejs-performance-nodetiming-startup-time-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, nodetiming, startup, monitoring, observability]
description: "Node.js performance.nodeTiming으로 런타임 시작 단계별 시간을 확인하고 애플리케이션 준비 시간과 함께 측정해 콜드 스타트 지연을 진단하는 방법을 설명합니다."
---

Node.js 애플리케이션의 시작이 느려지면 배포 교체, 오토스케일링, 장애 복구에 필요한 시간도 길어집니다.
프로세스가 실행된 시점부터 요청을 받을 준비가 끝날 때까지를 하나의 숫자로만 기록하면 런타임 초기화와 애플리케이션 초기화 중 어느 구간이 느린지 구분하기 어렵습니다.

`node:perf_hooks`의 `performance.nodeTiming`은 Node.js 프로세스 시작 과정의 주요 시점을 제공하는 `PerformanceNodeTiming` 항목입니다.
이를 애플리케이션 준비 완료 시점과 함께 기록하면 런타임 시작 비용과 사용자 코드의 초기화 비용을 나누어 살펴볼 수 있습니다.

이 글에서는 `performance.nodeTiming`의 주요 값, 시작 시간 요약 코드, 준비 상태와 연결하는 방법, 운영에서 비교할 때의 주의점을 정리합니다.
코드 내부 단계의 실행 시간을 직접 구분하려면 [Node.js performance.mark·measure 가이드](/development/blog/seo/2026/06/12/nodejs-performance-mark-measure-user-timing-guide.html)도 함께 참고하세요.

## performance.nodeTiming이 필요한 이유

### H3. 시작 지연을 런타임과 애플리케이션 구간으로 나눈다

서버 시작 시간에는 여러 작업이 포함됩니다.

- Node.js 런타임과 환경 초기화
- 모듈 로딩과 설정 검증
- 데이터베이스와 외부 서비스 연결
- 캐시 준비와 초기 데이터 조회
- HTTP 서버 시작과 준비 상태 전환

`performance.nodeTiming`은 이 가운데 Node.js 런타임이 기록하는 시작 단계의 시점을 보여 줍니다.
애플리케이션이 직접 측정한 준비 완료 시간과 함께 보면 런타임 초기화 이후 사용자 코드에서 소비한 시간을 구분하는 데 도움이 됩니다.

다만 `performance.nodeTiming`만으로 특정 모듈이나 데이터베이스 연결이 느린 원인을 자동으로 찾을 수는 없습니다.
세부 원인을 좁히려면 애플리케이션 초기화 단계에도 별도 측정을 추가해야 합니다.

### H3. 모든 값은 프로세스 시작을 기준으로 해석한다

`performance.nodeTiming`은 프로세스의 성능 타임라인에 포함된 항목이며 주요 속성은 밀리초 단위입니다.
기본 정보를 출력하면 현재 런타임에서 제공하는 값을 확인할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

const timing = performance.nodeTiming;

console.log({
  name: timing.name,
  entryType: timing.entryType,
  startTime: timing.startTime,
  durationMs: Number(timing.duration.toFixed(2)),
  nodeStartMs: Number(timing.nodeStart.toFixed(2)),
  v8StartMs: Number(timing.v8Start.toFixed(2)),
  bootstrapCompleteMs: Number(timing.bootstrapComplete.toFixed(2)),
  environmentMs: Number(timing.environment.toFixed(2)),
  loopStartMs: Number(timing.loopStart.toFixed(2))
});
```

이 값들은 벽시계 시각이 아니라 프로세스 성능 타임라인의 기준점에 대한 상대 시간입니다.
배포 시각 같은 절대 시각이 필요하면 별도의 타임스탬프를 함께 기록해야 합니다.

## 주요 시작 단계 해석하기

### H3. bootstrapComplete까지는 런타임 부트스트랩 구간이다

`nodeStart`, `v8Start`, `environment`, `bootstrapComplete`는 Node.js와 V8이 시작되는 과정의 주요 시점을 나타냅니다.
운영체제, Node.js 버전, 실행 옵션, 컨테이너 자원 제한이 바뀌면 이 구간의 기준선도 달라질 수 있습니다.

값 하나만 보고 정상 또는 장애를 판단하기보다 같은 환경에서 여러 번 실행한 분포를 비교해야 합니다.
특히 첫 실행은 파일 시스템 캐시나 이미지 풀 상태의 영향을 받을 수 있으므로 웜 스타트와 콜드 스타트를 구분하는 편이 좋습니다.

### H3. loopStart는 이벤트 루프가 시작된 시점이다

`loopStart`는 이벤트 루프가 시작된 시점을 보여 줍니다.
프로그램 실행 초기에 값을 조회하면 아직 이벤트 루프가 시작되지 않아 의미 있는 값이 준비되지 않았을 수 있습니다.

다음처럼 이벤트 루프가 한 번 진행된 뒤 값을 읽으면 시작 단계 요약을 안정적으로 기록할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';
import { setImmediate } from 'node:timers/promises';

await setImmediate();

const { nodeTiming } = performance;

console.log({
  bootstrapCompleteMs: nodeTiming.bootstrapComplete,
  loopStartMs: nodeTiming.loopStart,
  elapsedMs: performance.now()
});
```

`loopStart`가 빠르더라도 애플리케이션이 요청을 받을 준비가 끝났다는 뜻은 아닙니다.
외부 연결이나 초기 데이터 준비가 남아 있다면 준비 완료 시점을 별도로 측정해야 합니다.

### H3. loopExit는 종료 분석에 활용한다

`loopExit`는 이벤트 루프가 종료되는 시점과 관련된 값입니다.
프로세스가 정상적으로 실행 중일 때는 아직 종료 시점이 없으므로 시작 시간 대시보드의 핵심 지표로 사용하기 어렵습니다.

종료 단계까지 분석하려면 종료 훅에서 제한된 정보를 기록하거나 별도의 진단 보고서를 활용할 수 있습니다.
종료 처리 자체가 오래 걸리는 문제는 [Node.js 활성 리소스 진단 가이드](/development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html)를 함께 참고하세요.

## 애플리케이션 준비 시간 측정하기

### H3. 준비 완료 시점을 performance.now로 기록한다

런타임 시작 이후 애플리케이션이 실제 요청을 받을 준비가 될 때까지의 시간은 애플리케이션이 직접 정의해야 합니다.
필수 연결과 초기화가 끝난 뒤 `performance.now()`를 기록하면 프로세스 시작 기준의 준비 시간을 얻을 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

async function startApplication() {
  await validateConfiguration();
  await connectDatabase();
  await warmRequiredCache();

  const server = await startHttpServer();
  const readyMs = performance.now();

  console.log({
    event: 'application_ready',
    readyMs: Number(readyMs.toFixed(2)),
    bootstrapCompleteMs: Number(
      performance.nodeTiming.bootstrapComplete.toFixed(2)
    ),
    afterBootstrapMs: Number(
      (readyMs - performance.nodeTiming.bootstrapComplete).toFixed(2)
    )
  });

  return server;
}

await startApplication();
```

`afterBootstrapMs`가 크게 늘었다면 런타임보다 애플리케이션 초기화 경로를 우선 조사할 근거가 됩니다.
반대로 `bootstrapComplete` 자체가 함께 늘었다면 실행 환경이나 Node.js 버전 변경도 확인해야 합니다.

### H3. 준비 상태는 필수 의존성 기준으로 정의한다

준비 완료 시점을 단순히 HTTP 포트를 연 시점으로 정하면 필수 데이터베이스 연결이나 설정 검증이 끝나기 전에 트래픽을 받을 수 있습니다.
반대로 선택 기능까지 모두 기다리게 하면 필요 이상으로 준비 시간이 길어질 수 있습니다.

준비 상태에는 요청 처리에 반드시 필요한 조건만 포함하고, 선택 기능은 백그라운드에서 초기화하거나 별도 상태로 관찰하는 편이 좋습니다.
컨테이너 환경에서는 준비 상태 정의를 [Node.js readiness·liveness 프로브 가이드](/development/blog/seo/2026/04/14/nodejs-readiness-liveness-startup-probe-guide.html)와 일치시켜야 측정값과 실제 트래픽 전환 시점이 어긋나지 않습니다.

## 초기화 단계별 병목 좁히기

### H3. 느린 단계 후보에 mark와 measure를 추가한다

전체 준비 시간이 늘었다면 설정 검증, 외부 연결, 캐시 준비 같은 주요 단계에 측정을 추가해 원인을 좁힐 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

performance.mark('startup:app:start');

await validateConfiguration();
performance.mark('startup:config:done');

await connectDatabase();
performance.mark('startup:database:done');

await warmRequiredCache();
performance.mark('startup:cache:done');

performance.measure(
  'startup:database',
  'startup:config:done',
  'startup:database:done'
);
performance.measure(
  'startup:cache',
  'startup:database:done',
  'startup:cache:done'
);
performance.measure(
  'startup:app:total',
  'startup:app:start',
  'startup:cache:done'
);
```

측정 이름에는 사용자 ID, 요청 ID, 비밀 값 같은 데이터를 넣지 않아야 합니다.
고정된 단계 이름을 사용하면 배포별 분포를 비교하기 쉽고 메트릭 카디널리티도 예측할 수 있습니다.

### H3. 계측 코드가 시작 실패를 가리지 않게 한다

시작 시간 측정은 진단을 위한 보조 기능입니다.
메트릭 전송 실패 때문에 애플리케이션 시작이 실패하거나, 실제 초기화 오류가 계측 오류로 바뀌어서는 안 됩니다.

초기화 실패는 원래 오류를 유지한 채 기록하고, 메트릭 전송에는 짧은 제한 시간과 실패 처리를 적용하는 편이 안전합니다.
설정 값이나 연결 문자열 전체를 로그에 남기지 말고 단계 이름과 소요 시간처럼 조사에 필요한 최소 정보만 기록해야 합니다.

## 운영 환경에서 비교하는 방법

### H3. 단일 값보다 배포별 분포를 본다

시작 시간은 노드 상태, 이미지 다운로드, 디스크 캐시, CPU 경합에 따라 달라질 수 있습니다.
한 번의 실행만으로 회귀를 판단하지 말고 여러 시작 표본의 p50·p95·최대값을 비교해야 합니다.

배포별로 다음 값을 함께 기록하면 변화 원인을 구분하기 쉽습니다.

- Node.js 버전과 애플리케이션 버전
- `bootstrapComplete`와 `loopStart`
- 애플리케이션 준비 완료 시간
- 주요 초기화 단계별 실행 시간
- 컨테이너 CPU·메모리 제한
- 콜드 스타트와 웜 스타트 여부

이벤트 루프가 시작된 뒤의 런타임 부하는 [Node.js performance.eventLoopUtilization 가이드](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)로 별도 관찰할 수 있습니다.

### H3. 측정 조건을 고정해 전후를 비교한다

의존성 설치 상태나 테스트 데이터가 다른 환경의 숫자를 직접 비교하면 잘못된 결론을 내리기 쉽습니다.
성능 회귀 테스트에서는 같은 Node.js 버전, 실행 옵션, 자원 제한, 초기 데이터 조건을 유지해야 합니다.

시작 시간 개선 후에는 실제 배포 교체 시간과 준비 프로브 성공 시간도 함께 확인해야 합니다.
내부 측정값이 줄었더라도 이미지 다운로드나 스케줄링 대기가 길다면 사용자가 경험하는 복구 시간은 개선되지 않을 수 있습니다.

## 자주 묻는 질문

### H3. performance.nodeTiming과 performance.now는 무엇이 다른가요?

`performance.nodeTiming`은 Node.js가 기록한 런타임 시작 단계의 시점을 제공합니다.
`performance.now()`는 현재 시점이 프로세스 성능 타임라인 기준점에서 얼마나 지났는지 반환하므로 애플리케이션이 정의한 준비 완료 시간을 기록하는 데 사용할 수 있습니다.

### H3. nodeTiming만으로 콜드 스타트 전체 시간을 알 수 있나요?

아닙니다.
프로세스 실행 전의 컨테이너 스케줄링, 이미지 다운로드, 런타임 생성 시간은 포함되지 않으며, 애플리케이션 준비 완료 시점도 직접 정의해야 합니다.
플랫폼 지표와 애플리케이션 내부 지표를 함께 봐야 전체 콜드 스타트를 이해할 수 있습니다.

### H3. 시작 시간이 느리면 모듈 로딩을 모두 지연해야 하나요?

무조건 지연 로딩으로 바꾸면 첫 요청이 느려지거나 실행 중 오류가 늦게 발견될 수 있습니다.
먼저 단계별 측정으로 병목을 확인한 뒤, 요청 처리에 필수적이지 않고 지연 로딩 실패를 안전하게 처리할 수 있는 기능부터 검토해야 합니다.

## 적용 체크리스트

- [ ] `performance.nodeTiming`과 애플리케이션 준비 완료 시간을 함께 기록한다.
- [ ] 준비 상태를 실제 트래픽 처리에 필요한 필수 조건으로 정의한다.
- [ ] 주요 초기화 단계는 고정된 이름으로 측정한다.
- [ ] 로그와 메트릭에 연결 문자열, 토큰, 사용자 정보를 넣지 않는다.
- [ ] 같은 실행 환경에서 여러 표본의 분포를 비교한다.
- [ ] 내부 시작 시간과 실제 배포·준비 프로브 시간을 함께 확인한다.

## 마무리

`performance.nodeTiming`은 Node.js 런타임 시작 과정의 기준선을 만드는 데 유용합니다.
여기에 애플리케이션 준비 완료 시간과 주요 초기화 단계별 측정을 더하면 시작 지연이 런타임, 사용자 코드, 외부 의존성 중 어디에서 커졌는지 체계적으로 좁힐 수 있습니다.

시작 성능은 한 번의 숫자보다 동일한 조건에서 수집한 분포와 배포 전후 변화가 중요합니다.
필수 준비 조건을 명확히 정의하고 실제 트래픽 전환 시점과 함께 관찰하면 배포와 장애 복구의 예측 가능성을 높일 수 있습니다.


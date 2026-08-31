---
layout: post
title: "Node.js perf_hooks Histogram 가이드: CDF, CCDF, SLO burn rate로 지연 시간 읽는 법"
date: 2026-09-01 08:00:00 +0900
lang: ko
translation_key: nodejs-perf-hooks-histogram-cdf-slo-latency-guide
permalink: /development/blog/seo/2026/09/01/nodejs-perf-hooks-histogram-cdf-slo-latency-guide.html
alternates:
  ko: /development/blog/seo/2026/09/01/nodejs-perf-hooks-histogram-cdf-slo-latency-guide.html
  x_default: /development/blog/seo/2026/09/01/nodejs-perf-hooks-histogram-cdf-slo-latency-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, perf-hooks, histogram, cdf, ccdf, slo, latency, monitoring]
description: "Node.js perf_hooks Histogram의 cdf(), ccdf(), cliffsD(), cohensD(), EWMA, SLO burn rate를 이용해 지연 시간 분포와 배포 전후 성능 차이를 읽는 방법을 실무 예제로 정리합니다."
---

Node.js 서비스의 지연 시간을 볼 때 `p95`, `p99`, `max`만으로 판단하면 놓치는 부분이 있습니다.
백분위수는 "특정 비율의 요청이 이 값 이하"라는 기준을 빠르게 보여 주지만, 임계값을 넘은 요청이 얼마나 자주 발생하는지, 배포 전후 분포가 실제로 달라졌는지, SLO 예산을 얼마나 빨리 쓰고 있는지는 따로 계산해야 합니다.

Node.js `node:perf_hooks`의 `Histogram`에는 이런 질문에 답하기 위한 분포 분석 API가 추가되어 있습니다.
공식 문서 기준으로 Node.js 26.8.0부터 `histogram.cdf(value)`, `histogram.ccdf(value)`, `histogram.cliffsD(other)`, `histogram.cohensD(other)`, `histogram.countAt(value)`, EWMA 관련 값, `histogram.burnRate(sloTarget)`를 사용할 수 있습니다.

이 글에서는 `perf_hooks` Histogram으로 지연 시간 분포를 읽는 방법을 정리합니다.
기본적인 백분위수 수집은 [Node.js perf_hooks.createHistogram 가이드](/development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html), 이벤트 루프 지연 관찰은 [Node.js monitorEventLoopDelay 가이드](/development/blog/seo/2026/06/18/nodejs-monitor-event-loop-delay-alerting-guide.html), 함수 실행 시간 측정은 [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js Performance measurement APIs 공식 문서](https://nodejs.org/api/perf_hooks.html), [Histogram 클래스 공식 문서](https://nodejs.org/api/perf_hooks.html#class-histogram), [monitorEventLoopDelay 공식 문서](https://nodejs.org/api/perf_hooks.html#perf_hooksmonitoreventloopdelayoptions)

## Histogram 분포 API가 필요한 이유

### 백분위수는 기준값을 보여 주지만 임계값 질문에는 돌아간다

지연 시간 모니터링에서 가장 익숙한 값은 `p95`와 `p99`입니다.
예를 들어 p95가 180ms라면 관측된 값의 95%가 180ms 이하라는 뜻입니다.
이 값은 대시보드에 올리기 좋고 추세를 빠르게 읽기 좋습니다.

하지만 운영 중에는 질문이 조금 다르게 들어옵니다.

- "200ms SLO를 넘은 요청 비율이 얼마나 되나요?"
- "배포 이후 500ms 이상 지연이 더 자주 발생하나요?"
- "평균은 조금 늘었는데 실제 사용자 체감도 나빠졌나요?"
- "지금 에러 예산을 평소보다 빠르게 쓰고 있나요?"

이 질문에 매번 raw sample을 모두 들고 계산하면 저장 비용과 구현 부담이 커집니다.
`Histogram`이 이미 분포를 압축해 들고 있다면, 그 분포에 직접 질문하는 편이 더 단순합니다.

### cdf와 ccdf는 임계값 중심으로 읽는다

`histogram.cdf(value)`는 기록된 값이 `value` 이하일 확률을 반환합니다.
반대로 `histogram.ccdf(value)`는 기록된 값이 `value`를 초과할 확률을 반환합니다.
공식 문서에서는 `ccdf(value)`가 `1 - histogram.cdf(value)`와 같다고 설명합니다.

지연 시간 SLO를 200ms로 두었다면 운영자가 자주 보고 싶은 값은 p95 자체가 아니라 "200ms 초과 비율"일 수 있습니다.
이때 `ccdf(200_000_000)`처럼 임계값을 나노초 단위로 넣어 바로 초과 확률을 읽을 수 있습니다.

```js
import { createHistogram } from 'node:perf_hooks';

const latency = createHistogram();

latency.record(42_000_000);
latency.record(85_000_000);
latency.record(210_000_000);
latency.record(730_000_000);

const sloNs = 200_000_000;

console.log({
  p95Ms: latency.percentile(95) / 1e6,
  withinSloRatio: latency.cdf(sloNs),
  overSloRatio: latency.ccdf(sloNs)
});
```

`percentile(95)`는 "95% 지점이 어디인가"를 묻습니다.
`ccdf(sloNs)`는 "정해진 SLO를 넘은 비율이 얼마인가"를 묻습니다.
둘 다 필요하지만, 경보와 에러 예산 계산에는 후자가 더 직접적입니다.

## latency Histogram 설계하기

### 단위를 한 곳에서 고정한다

`perf_hooks` Histogram은 나노초 단위 값과 잘 맞습니다.
`performance.timerify()`에 histogram을 넘기는 경우에도 duration은 나노초로 기록됩니다.
반면 HTTP handler에서 직접 시간을 재면 `performance.now()`는 밀리초 단위입니다.
따라서 기록 시점에서 단위를 반드시 통일해야 합니다.

```js
import { createHistogram, performance } from 'node:perf_hooks';

const httpLatency = createHistogram();

function msToNs(value) {
  return Math.max(1, Math.round(value * 1e6));
}

export async function measureHttpRequest(work) {
  const startedAt = performance.now();

  try {
    return await work();
  } finally {
    const durationMs = performance.now() - startedAt;
    httpLatency.record(msToNs(durationMs));
  }
}
```

대시보드에는 밀리초로 보여 주더라도 Histogram 내부 기록 단위는 하나로 고정하는 편이 좋습니다.
단위가 섞이면 p95, cdf, burn rate가 모두 그럴듯하지만 틀린 값이 됩니다.

### snapshot 함수에서 해석용 값을 같이 만든다

운영 코드 곳곳에서 `histogram.percentile(99) / 1e6` 같은 계산을 반복하면 기준이 흩어집니다.
대신 snapshot helper를 만들어 대시보드, 로그, 테스트가 같은 해석을 공유하게 만들 수 있습니다.

```js
export function readLatencySnapshot(histogram, { sloMs }) {
  const sloNs = msToNs(sloMs);

  return {
    count: histogram.count,
    minMs: histogram.min / 1e6,
    meanMs: histogram.mean / 1e6,
    p50Ms: histogram.percentile(50) / 1e6,
    p95Ms: histogram.percentile(95) / 1e6,
    p99Ms: histogram.percentile(99) / 1e6,
    maxMs: histogram.max / 1e6,
    withinSloRatio: histogram.cdf(sloNs),
    overSloRatio: histogram.ccdf(sloNs)
  };
}
```

이 함수는 단순한 변환처럼 보이지만 운영에서는 중요합니다.
"200ms 초과 비율"과 "p95"가 같은 화면에 같이 있어야 장애 판단이 빨라집니다.
p95가 안정적이어도 극단적인 tail이 늘 수 있고, p99가 튀어도 SLO 초과 비율은 아직 작을 수 있습니다.

## 배포 전후 분포 비교하기

### 평균 차이만 보면 작은 악화를 놓칠 수 있다

성능 회귀를 볼 때 평균만 비교하면 tail latency 변화가 가려질 수 있습니다.
예를 들어 대부분의 요청은 그대로인데 일부 요청만 느려졌다면 평균은 조금만 움직입니다.
하지만 실제 사용자는 그 일부 요청에서 강한 지연을 경험합니다.

Node.js 26.8.0의 Histogram 비교 API는 두 분포 사이의 차이를 읽는 데 도움을 줍니다.
`histogram.cliffsD(other)`는 한 분포의 임의 값이 다른 분포의 임의 값보다 클 경향을 -1부터 1 사이 값으로 표현합니다.
`histogram.cohensD(other)`는 평균 차이를 pooled standard deviation 기준으로 표준화한 effect size를 반환합니다.

```js
import { createHistogram } from 'node:perf_hooks';

function recordSamples(valuesMs) {
  const histogram = createHistogram();

  for (const value of valuesMs) {
    histogram.record(msToNs(value));
  }

  return histogram;
}

const before = recordSamples([82, 91, 96, 110, 135, 180]);
const after = recordSamples([86, 95, 120, 180, 260, 420]);

console.log({
  beforeP95Ms: before.percentile(95) / 1e6,
  afterP95Ms: after.percentile(95) / 1e6,
  cliffsD: after.cliffsD(before),
  cohensD: after.cohensD(before)
});
```

실무에서는 이 값을 단독 합격/불합격 기준으로 두기보다 배포 검증의 보조 지표로 쓰는 편이 좋습니다.
샘플 수가 적거나 트래픽 구성이 다르면 effect size가 과장되거나 축소될 수 있습니다.
그래도 "평균이 8ms 늘었다"보다 "배포 후 분포가 전반적으로 느린 쪽으로 이동했다"는 판단을 더 선명하게 만들 수 있습니다.

### 비교 대상의 트래픽 조건을 맞춘다

분포 비교는 입력 조건이 비슷할 때 의미가 있습니다.
배포 전은 낮 시간 전체 트래픽이고 배포 후는 야간 배치가 섞인 트래픽이라면, Histogram 차이가 코드 변경 때문인지 부하 조건 때문인지 구분하기 어렵습니다.

비교할 때는 다음 조건을 맞추는 편이 좋습니다.

- 같은 route 또는 같은 작업 단위
- 같은 region 또는 같은 인프라 그룹
- 비슷한 트래픽 시간대
- 같은 sampling 정책
- 같은 성공/실패 필터

특히 실패 요청을 포함할지 제외할지 먼저 정해야 합니다.
실패가 빠르게 반환되는 서비스에서는 실패 요청을 포함하면 평균 지연이 낮아질 수 있습니다.
반대로 timeout 실패가 많은 서비스에서는 실패 요청이 tail을 크게 밀어 올립니다.

## EWMA와 SLO burn rate 활용하기

### 이동 평균은 최근 상태를 더 빠르게 반영한다

누적 Histogram은 긴 시간의 전체 분포를 보여 주는 데 좋습니다.
하지만 장애 감지는 최근 상태에 더 민감해야 합니다.
최근 5분 동안만 나빠졌는데 하루 누적 Histogram으로 보면 변화가 묻힐 수 있습니다.

Node.js 26.8.0 문서에는 `histogram.ewmaMean`, `histogram.ewmaStddev`, `histogram.ewmaErrorRate`, `histogram.burnRate()` 같은 SLO 관찰용 값이 포함되어 있습니다.
EWMA는 최근 샘플에 더 큰 가중치를 주는 이동 평균이므로, 누적 평균보다 변화에 빠르게 반응할 수 있습니다.

```js
import { createHistogram } from 'node:perf_hooks';

const latency = createHistogram({
  halfLife: 100,
  threshold: 200_000_000
});

for (const valueMs of [80, 92, 110, 350, 420, 95]) {
  latency.record(msToNs(valueMs));
}

console.log({
  ewmaMeanMs: latency.ewmaMean / 1e6,
  ewmaStddevMs: latency.ewmaStddev / 1e6,
  ewmaErrorRate: latency.ewmaErrorRate,
  burnRate: latency.burnRate(0.99)
});
```

여기서 `threshold`는 SLO 위반을 판단할 기준값입니다.
`burnRate(0.99)`는 99% SLO를 기준으로 error budget을 얼마나 빠르게 소모하는지 보는 값입니다.
burn rate가 1이면 예산을 SLO 기간에 맞춰 정확히 쓰는 상태이고, 1보다 크면 허용보다 빠르게 쓰고 있다는 뜻입니다.

### burn rate는 alert의 원인을 설명하기 좋다

단순히 "p99가 900ms입니다"라는 경보는 대응 우선순위를 판단하기 어렵습니다.
서비스마다 p99의 정상 범위가 다르고, 짧은 spike인지 지속 악화인지도 따로 봐야 합니다.

burn rate는 SLO 목표와 연결되어 있어서 설명력이 좋습니다.
"현재 99% latency SLO의 error budget을 허용 속도의 4배로 쓰고 있습니다"라고 말하면, 장애 대응자가 지금 멈춰서 봐야 할 이유를 바로 이해할 수 있습니다.

다만 burn rate도 완벽한 답은 아닙니다.
트래픽이 너무 적으면 몇 건의 느린 요청만으로 값이 흔들릴 수 있고, SLO threshold가 현실보다 낮으면 항상 나쁜 신호만 나옵니다.
처음에는 p95, p99, `ccdf(threshold)`, burn rate를 같이 보며 기준을 조정하는 편이 안전합니다.

## 운영 코드에 붙이는 패턴

### route별 Histogram을 분리한다

전체 HTTP 요청을 하나의 Histogram에 넣으면 서비스의 큰 그림은 볼 수 있습니다.
하지만 느린 route 하나가 전체 평균에 묻히거나, 빠른 health check가 실제 사용자 요청을 희석할 수 있습니다.
운영에서는 route나 작업 종류별로 Histogram을 나눠야 합니다.

```js
import { createHistogram, performance } from 'node:perf_hooks';

const histograms = new Map();

function getRouteHistogram(route) {
  let histogram = histograms.get(route);

  if (!histogram) {
    histogram = createHistogram({
      halfLife: 200,
      threshold: 300_000_000
    });
    histograms.set(route, histogram);
  }

  return histogram;
}

export async function withRouteLatency(route, work) {
  const histogram = getRouteHistogram(route);
  const startedAt = performance.now();

  try {
    return await work();
  } finally {
    histogram.record(msToNs(performance.now() - startedAt));
  }
}
```

route 이름은 raw URL이 아니라 템플릿 형태로 정규화해야 합니다.
`/users/1`, `/users/2`를 모두 별도 Histogram으로 만들면 메모리가 늘고 지표가 쪼개집니다.
`/users/:id`처럼 제한된 cardinality를 유지하는 것이 중요합니다.

### 주기적으로 읽고 reset할지 결정한다

Histogram을 계속 누적하면 장기 분포를 볼 수 있지만, 최근 상태 변화에는 둔감해집니다.
반대로 매번 `reset()`하면 window별 변화는 잘 보이지만 장기 비교가 어렵습니다.
둘 중 하나만 고르기보다 목적별로 나누는 편이 좋습니다.

```js
setInterval(() => {
  for (const [route, histogram] of histograms) {
    const snapshot = readLatencySnapshot(histogram, { sloMs: 300 });

    emitMetric('http_latency_p95_ms', snapshot.p95Ms, { route });
    emitMetric('http_latency_p99_ms', snapshot.p99Ms, { route });
    emitMetric('http_latency_over_slo_ratio', snapshot.overSloRatio, { route });
    emitMetric('http_latency_burn_rate', histogram.burnRate(0.99), { route });

    histogram.reset();
  }
}, 60_000).unref();
```

위 예제는 1분 window를 만드는 방식입니다.
운영 환경에서는 같은 데이터를 metrics backend로 보내고, 긴 기간의 추세는 backend의 rollup 기능으로 보는 편이 흔합니다.
애플리케이션 내부 Histogram은 가벼운 실시간 집계에 집중시키는 것이 관리하기 쉽습니다.

## 주의할 점

### 분포 지표를 개인정보 로그와 섞지 않는다

Histogram은 숫자만 기록하므로 자체로는 민감정보 위험이 낮습니다.
문제는 지연 시간과 함께 request body, query, header를 로그로 남기는 코드입니다.
성능 조사를 하다 보면 느린 요청의 입력값을 보고 싶어지지만, 토큰, 이메일, 전화번호, 결제 정보가 그대로 남을 수 있습니다.

성능 로그에는 route template, status code, duration, upstream 이름, timeout 여부처럼 진단에 필요한 최소 정보만 남기는 편이 좋습니다.
원문 URL이나 request body가 필요하다면 별도 승인된 샘플링 정책과 마스킹 규칙을 먼저 둬야 합니다.

### 작은 샘플의 effect size를 과신하지 않는다

`cliffsD()`와 `cohensD()`는 배포 전후 비교에 유용하지만, 샘플이 작으면 값이 흔들립니다.
특히 cold start, 캐시 상태, 특정 고객의 대량 요청이 섞이면 분포 차이가 코드 변경처럼 보일 수 있습니다.

배포 판정에서는 최소 샘플 수, 비교 window, 제외할 route를 정해두는 것이 좋습니다.
수치가 애매할 때는 자동 롤백보다 추가 관찰이나 canary 비율 조정이 더 나은 선택일 수 있습니다.

### Node.js 버전 차이를 명시한다

`Histogram` 자체는 오래전부터 있었지만, 이 글에서 다룬 일부 분포 분석 API는 Node.js 26.8.0 기준으로 추가된 항목입니다.
운영 런타임이 Node.js 24 LTS나 22 LTS라면 같은 메서드가 없을 수 있습니다.

공통 라이브러리에 넣을 때는 feature detection을 해두면 배포 환경 차이로 인한 장애를 줄일 수 있습니다.

```js
export function overThresholdRatio(histogram, thresholdNs) {
  if (typeof histogram.ccdf === 'function') {
    return histogram.ccdf(thresholdNs);
  }

  const p99 = histogram.percentile(99);
  return p99 > thresholdNs ? 0.01 : 0;
}
```

fallback 값은 정확한 대체가 아닙니다.
다만 오래된 런타임에서 코드가 깨지는 것보다 "정확한 SLO 초과 비율은 지원되는 Node.js 버전에서만 계산한다"는 계약을 분명히 하는 편이 낫습니다.

## FAQ

### H3. cdf와 percentile은 어떻게 다르나요?

`percentile(95)`는 "95% 지점의 값"을 반환합니다.
`cdf(value)`는 "이 값 이하에 들어온 비율"을 반환합니다.
즉 percentile은 비율에서 값을 찾고, cdf는 값에서 비율을 찾습니다.
SLO처럼 기준값이 이미 정해진 상황에서는 `cdf()`와 `ccdf()`가 더 직접적입니다.

### H3. latency SLO는 p99와 같은 뜻인가요?

같지 않습니다.
"p99가 300ms 이하"는 백분위수 기준입니다.
"99% 요청이 300ms 이하"는 사실상 `cdf(300ms) >= 0.99`라는 임계값 기준입니다.
둘은 비슷하게 들리지만 경보 구현에서는 어느 값을 기준으로 삼는지 명확히 해야 합니다.

### H3. Histogram 값을 그대로 외부 모니터링에 보내도 되나요?

보통은 snapshot으로 요약한 값을 보냅니다.
p50, p95, p99, `ccdf(threshold)`, burn rate, count 정도를 route label과 함께 보내면 대시보드와 alert에 충분한 경우가 많습니다.
원본 샘플을 모두 보내는 방식은 비용과 개인정보 위험을 별도로 검토해야 합니다.

## 정리

Node.js `perf_hooks` Histogram은 단순히 p95와 p99를 계산하는 도구에서 한 단계 더 나아가고 있습니다.
`cdf()`와 `ccdf()`는 SLO threshold를 기준으로 분포를 읽게 해주고, `cliffsD()`와 `cohensD()`는 배포 전후 분포 차이를 설명하는 데 도움을 줍니다.
EWMA와 burn rate는 최근 상태와 error budget 소모 속도를 더 운영 친화적인 언어로 보여 줍니다.

실무에서는 먼저 단위를 통일하고, route cardinality를 제한하고, snapshot helper를 만들어 해석 기준을 한 곳에 모으세요.
그다음 p95, p99, SLO 초과 비율, burn rate를 함께 보면 "느려졌다"는 감각을 더 재현 가능한 지표로 바꿀 수 있습니다.

## 내부 링크

- [Node.js perf_hooks.createHistogram 가이드: 지연 시간 백분위수 측정하기](/development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html)
- [Node.js monitorEventLoopDelay 가이드: 이벤트 루프 지연 경보 설계하기](/development/blog/seo/2026/06/18/nodejs-monitor-event-loop-delay-alerting-guide.html)
- [Node.js performance.timerify 가이드: 함수 실행 시간을 관측 가능한 지표로 만들기](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)

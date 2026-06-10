---
layout: post
title: "Node.js performance.eventLoopUtilization 가이드: 이벤트 루프 부하 측정하기"
date: 2026-06-11 08:00:00 +0900
lang: ko
translation_key: nodejs-performance-eventlooputilization-monitoring-guide
permalink: /development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html
alternates:
  ko: /development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html
  x_default: /development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, eventlooputilization, monitoring, observability, backend]
description: "Node.js performance.eventLoopUtilization()으로 이벤트 루프의 active·idle 시간과 활용률을 측정하는 방법을 정리합니다. 구간 비교, CPU 작업 재현, 운영 모니터링과 해석 기준을 예제로 설명합니다."
---

Node.js 서버의 응답이 느려졌을 때 CPU 사용률만 확인하면 원인을 충분히 설명하지 못할 수 있습니다.
프로세스 전체 CPU 사용률이 높지 않아도 메인 이벤트 루프가 동기 연산에 오래 묶이면 요청과 타이머 처리가 늦어질 수 있기 때문입니다.

`performance.eventLoopUtilization()`은 이벤트 루프가 활성 상태로 실행된 시간과 유휴 상태로 기다린 시간을 바탕으로 활용률을 계산합니다.
이 값을 일정 구간마다 비교하면 이벤트 루프가 얼마나 바빴는지 확인하고, 응답 지연이 메인 스레드 부하와 함께 발생했는지 판단하는 데 도움을 받을 수 있습니다.

이 글에서는 `performance.eventLoopUtilization()`의 기본 사용법, 구간별 활용률 측정, 동기 CPU 작업으로 부하를 재현하는 방법, 운영 모니터링과 해석 시 주의할 점을 정리합니다.
지연 시간을 백분위수로 요약하는 방법이 필요하다면 [Node.js perf_hooks.createHistogram 가이드](/development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html)도 함께 참고하세요.

## eventLoopUtilization이 필요한 이유

### H3. CPU 사용률과 이벤트 루프 활용률은 관점이 다르다

CPU 사용률은 프로세스나 시스템이 CPU 시간을 얼마나 사용했는지 보여 줍니다.
반면 이벤트 루프 활용률은 관찰 구간 중 Node.js 이벤트 루프가 활성 상태였던 비율을 보여 줍니다.

예를 들어 여러 코어가 있는 서버에서 Node.js 메인 스레드 하나가 동기 연산으로 계속 바쁘더라도 시스템 전체 CPU 사용률은 낮아 보일 수 있습니다.
하지만 해당 프로세스의 이벤트 루프는 요청을 처리할 여유가 부족할 수 있습니다.

이벤트 루프 활용률은 다음 상황을 조사할 때 유용합니다.

- API 응답 시간이 늘어난 시점에 메인 이벤트 루프도 바빴는지 확인할 때
- 동기 JSON 처리, 정규식, 압축, 템플릿 렌더링의 영향을 비교할 때
- 코드 변경이나 배포 전후의 이벤트 루프 부하를 비교할 때
- 워커 스레드나 별도 서비스로 작업을 분리할 필요가 있는지 판단할 때

활용률 하나만으로 병목 원인을 확정할 수는 없지만, 조사 방향을 정하는 운영 지표로 사용할 수 있습니다.

### H3. active와 idle 시간을 바탕으로 활용률을 계산한다

`performance.eventLoopUtilization()`은 다음 값을 포함한 객체를 반환합니다.

- `active`: 이벤트 루프가 활성 상태였던 시간
- `idle`: 이벤트 루프가 유휴 상태였던 시간
- `utilization`: 전체 관찰 시간 중 활성 시간이 차지한 비율

```js
import { performance } from 'node:perf_hooks';

console.log(performance.eventLoopUtilization());
```

`utilization`은 보통 0과 1 사이 값으로 해석합니다.
예를 들어 `0.7`이라면 관찰 구간에서 이벤트 루프가 약 70% 활성 상태였다는 의미입니다.

프로세스를 시작한 뒤 누적된 값만 한 번 확인하기보다, 일정 구간의 변화를 비교해야 운영 상황을 더 명확하게 이해할 수 있습니다.

## 구간별 이벤트 루프 활용률 측정하기

### H3. 이전 스냅샷과 비교해 구간 값을 구한다

이 API에 이전 측정 결과를 전달하면 해당 시점 이후의 차이를 계산할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

let previous = performance.eventLoopUtilization();

setInterval(() => {
  const current = performance.eventLoopUtilization();
  const interval = performance.eventLoopUtilization(previous);

  console.log({
    active: interval.active,
    idle: interval.idle,
    utilization: interval.utilization
  });

  previous = current;
}, 5_000).unref();
```

이 예제는 5초마다 직전 구간의 이벤트 루프 활용률을 출력합니다.
진단용 타이머에 `unref()`를 호출했기 때문에 이 타이머만 남았을 때 프로세스 종료를 막지 않습니다.

측정 간격이 너무 짧으면 순간적인 작업에 값이 크게 흔들리고, 너무 길면 짧은 부하 급증이 평균에 가려질 수 있습니다.
서비스의 요청 패턴과 경보 목적에 맞춰 간격을 정해야 합니다.

### H3. 두 스냅샷을 명시해 차이를 계산할 수 있다

특정 작업 전후를 비교하려면 시작과 종료 스냅샷을 보관한 뒤 차이를 계산할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

const before = performance.eventLoopUtilization();

await runBatchJob();

const after = performance.eventLoopUtilization();
const difference = performance.eventLoopUtilization(after, before);

console.log({
  job: 'daily-summary',
  eventLoopUtilization: difference.utilization
});
```

이 방식은 배치 작업이나 특정 처리 경로가 이벤트 루프를 얼마나 점유했는지 비교할 때 유용합니다.
다만 같은 프로세스에서 동시에 처리된 다른 요청과 작업도 측정값에 포함되므로, 운영 환경의 결과를 해당 작업만의 비용으로 단정하면 안 됩니다.

## 동기 CPU 작업으로 부하 재현하기

### H3. 이벤트 루프를 막는 작업은 활용률을 높인다

아래 예제는 의도적으로 동기 반복 작업을 실행해 이벤트 루프를 바쁘게 만듭니다.

```js
import { performance } from 'node:perf_hooks';
import { setTimeout as delay } from 'node:timers/promises';

function blockEventLoop(durationMs) {
  const deadline = performance.now() + durationMs;

  while (performance.now() < deadline) {
    Math.sqrt(Math.random());
  }
}

const before = performance.eventLoopUtilization();

await delay(100);
blockEventLoop(300);
await delay(100);

const interval = performance.eventLoopUtilization(before);

console.log({
  utilization: interval.utilization,
  active: interval.active,
  idle: interval.idle
});
```

이 코드는 동작 확인을 위한 재현 예제입니다.
실제 서버 요청 처리 경로에서 의도적으로 이벤트 루프를 막는 코드를 실행하면 다른 사용자 요청까지 지연될 수 있습니다.

동기 연산이 꼭 필요하다면 입력 크기를 제한하고, 처리 시간을 측정하며, 긴 CPU 작업은 워커 스레드나 별도 작업 시스템으로 분리할 수 있는지 검토해야 합니다.

### H3. 긴 반복 작업은 작은 단위로 나눌 수 있다

반복 작업을 한 번에 모두 처리하면 이벤트 루프가 오랫동안 다른 작업을 실행하지 못할 수 있습니다.
작업 특성에 따라 일정 단위마다 실행 기회를 양보하면 응답성을 개선할 수 있습니다.

```js
import { scheduler } from 'node:timers/promises';

export async function processItems(items) {
  const results = [];

  for (let index = 0; index < items.length; index += 1) {
    results.push(transform(items[index]));

    if ((index + 1) % 100 === 0) {
      await scheduler.yield();
    }
  }

  return results;
}
```

`scheduler.yield()`는 작업 총량을 줄이지는 않지만 다른 이벤트 루프 작업이 실행될 기회를 제공합니다.
작업을 나누는 기준과 효과를 실제 부하 테스트로 검증해야 하며, CPU 사용량이 큰 계산은 워커 스레드가 더 적합할 수 있습니다.

긴 반복 작업의 양보 패턴은 [Node.js scheduler.yield 가이드](/development/blog/seo/2026/05/20/nodejs-timers-promises-scheduler-yield-event-loop-guide.html)에서 더 자세히 확인할 수 있습니다.

## 운영 모니터링에 적용하기

### H3. 활용률과 응답 지연을 함께 기록한다

이벤트 루프 활용률이 높더라도 요청이 정상적으로 처리되고 있다면 즉시 장애라고 단정할 수 없습니다.
반대로 활용률이 낮아도 데이터베이스, 외부 API, 네트워크 대기 때문에 응답 시간이 길어질 수 있습니다.

운영에서는 다음 지표를 함께 비교하는 편이 좋습니다.

- 요청 처리 시간의 p50, p95, p99
- 초당 요청 수와 동시 요청 수
- 오류율과 타임아웃 수
- 프로세스 및 시스템 CPU 사용률
- 이벤트 루프 지연 시간과 활용률
- 메모리 사용량과 가비지 컬렉션 상태

활용률 상승과 응답 지연 상승이 같은 시점에 반복된다면 동기 작업, 과도한 콜백 처리, 큰 입력 처리 같은 메인 스레드 병목을 우선 조사할 근거가 됩니다.

### H3. 고정 임계값보다 서비스 기준선을 먼저 만든다

모든 서비스에 공통으로 적용할 수 있는 단일 활용률 임계값은 없습니다.
짧은 요청이 많은 API, 주기적으로 배치를 실행하는 프로세스, 연결을 오래 유지하는 서버는 정상 패턴이 서로 다릅니다.

먼저 정상 트래픽에서 시간대별 기준선을 만들고, 배포 전후와 장애 시점의 변화를 비교하는 것이 좋습니다.
경보도 활용률 하나만 사용하기보다 높은 활용률과 응답 지연 또는 오류율이 일정 시간 함께 지속될 때 발생하도록 설계하면 불필요한 알림을 줄일 수 있습니다.

```js
function shouldAlert({
  utilization,
  p95LatencyMs,
  errorRate
}) {
  return utilization > 0.85
    && (p95LatencyMs > 500 || errorRate > 0.02);
}
```

위 숫자는 예시일 뿐입니다.
실제 임계값은 서비스 수준 목표, 정상 기준선, 트래픽 패턴을 바탕으로 정해야 합니다.

### H3. 로그에는 필요한 진단 정보만 남긴다

활용률 로그에는 요청 본문, 인증 토큰, 사용자 식별 정보가 필요하지 않습니다.
프로세스 역할, 인스턴스 구분용 내부 식별자, 측정 구간, 비율처럼 조사에 필요한 최소 정보만 기록하는 편이 안전합니다.

```js
logger.info({
  event: 'event-loop-utilization',
  service: 'api',
  intervalSeconds: 10,
  utilization: Number(interval.utilization.toFixed(4))
});
```

고빈도 로그는 저장 비용을 늘리고 중요한 이벤트를 찾기 어렵게 만들 수 있습니다.
메트릭 시스템을 사용할 수 있다면 이 비율은 메트릭으로 수집하고, 로그는 임계값 초과나 진단 상황에 제한하는 방법도 고려할 수 있습니다.

## eventLoopUtilization 해석 시 주의할 점

### H3. 높은 활용률은 원인이 아니라 현상이다

높은 활용률은 이벤트 루프가 바빴다는 사실을 보여 주지만 어떤 코드가 시간을 사용했는지는 알려 주지 않습니다.
원인을 찾으려면 CPU 프로파일, 요청 추적, 함수별 처리 시간, 이벤트 루프 지연 분포 같은 추가 자료가 필요합니다.

활용률이 높을 때는 다음 항목을 순서대로 확인할 수 있습니다.

1. 같은 시점에 응답 지연과 오류율도 증가했는지 확인한다.
2. 최근 배포와 트래픽 또는 입력 크기 변화를 확인한다.
3. 동기 파일 작업, 큰 JSON 처리, 정규식, 암호화 연산을 찾는다.
4. CPU 프로파일로 실제 실행 시간이 긴 함수를 확인한다.
5. 작업 분할, 입력 제한, 캐시, 워커 분리의 효과를 부하 테스트한다.

### H3. 낮은 활용률도 서버가 건강하다는 보장은 아니다

이벤트 루프가 유휴 상태인 시간이 많아도 외부 데이터베이스나 API 응답을 오래 기다리는 중일 수 있습니다.
또한 연결 풀 고갈, 네트워크 문제, 하위 서비스 오류는 이벤트 루프 활용률을 크게 높이지 않고도 사용자 응답을 늦출 수 있습니다.

따라서 낮은 활용률은 메인 이벤트 루프의 과도한 실행 가능성을 낮추는 단서일 뿐, 전체 시스템이 정상이라는 결론은 아닙니다.
관찰 가능성 지표는 항상 요청 흐름과 의존 서비스 상태를 함께 봐야 합니다.

### H3. worker_threads는 각 스레드에서 따로 측정한다

워커 스레드에서 실행되는 작업은 메인 스레드의 이벤트 루프 활용률만으로 충분히 관찰할 수 없습니다.
각 워커가 자신의 이벤트 루프 활용률을 측정하고, 필요한 요약값을 부모 스레드에 전달하도록 설계해야 합니다.

워커를 사용하면 CPU 작업이 사라지는 것이 아니라 메인 이벤트 루프와 분리됩니다.
워커 수, 작업 큐 대기 시간, 처리량, 메모리 사용량도 함께 측정해야 과부하를 다른 위치로 옮기는 문제를 피할 수 있습니다.

## 테스트와 검증 기준

### H3. 절대값보다 상대 변화를 확인한다

개발 머신과 CI 환경은 CPU 성능과 백그라운드 작업이 다르기 때문에 활용률 절대값을 테스트에서 고정하면 불안정할 수 있습니다.
성능 검증에서는 동일한 환경과 입력에서 변경 전후를 여러 번 측정하고, 응답 시간과 처리량도 함께 비교하는 편이 좋습니다.

다음 항목을 확인할 수 있습니다.

- 측정 코드가 서비스 종료를 막지 않는가?
- 측정 간격이 순간 부하와 장기 추세를 구분하기에 적절한가?
- 활용률 상승 시 응답 지연도 함께 관찰되는가?
- 동기 작업을 분리하거나 나눈 뒤 지연과 활용률이 개선되는가?
- 로그와 메트릭에 민감정보가 포함되지 않는가?

## 자주 묻는 질문

### H3. eventLoopUtilization과 monitorEventLoopDelay는 무엇이 다른가요?

`eventLoopUtilization()`은 관찰 구간 중 이벤트 루프가 활성 상태였던 비율을 보여 줍니다.
`monitorEventLoopDelay()`는 이벤트 루프가 예정된 작업을 얼마나 늦게 실행했는지 지연 시간의 분포를 측정합니다.

둘은 서로 대체 관계가 아닙니다.
활용률과 지연 분포를 함께 보면 이벤트 루프가 바쁜지, 그 결과 실행 지연이 얼마나 발생했는지 더 잘 이해할 수 있습니다.

### H3. 활용률이 높으면 바로 워커 스레드를 사용해야 하나요?

먼저 실제 병목이 CPU 연산인지 프로파일링으로 확인해야 합니다.
불필요한 동기 코드 제거, 입력 크기 제한, 캐시, 알고리즘 개선이 더 단순하고 효과적인 해결책일 수 있습니다.
분리할 만한 CPU 작업이 확인됐을 때 워커 스레드와 작업 큐를 검토하는 것이 좋습니다.

### H3. 몇 초 간격으로 측정해야 하나요?

정답은 서비스마다 다릅니다.
짧은 부하 급증을 찾아야 하면 비교적 짧은 간격이 필요하고, 용량 계획과 장기 추세가 목적이면 더 긴 집계 구간이 적합합니다.
초기에는 5~15초 정도의 원본 측정값을 수집하고, 메트릭 시스템에서 더 긴 구간으로 집계해 패턴을 확인할 수 있습니다.

## 마무리

`performance.eventLoopUtilization()`은 Node.js 이벤트 루프가 관찰 구간 동안 얼마나 바빴는지 확인하는 가벼운 진단 도구입니다.
이전 스냅샷과 비교해 구간별 활용률을 수집하면 배포, 트래픽 변화, 동기 작업이 메인 이벤트 루프에 미치는 영향을 추적할 수 있습니다.

다만 활용률은 병목 원인을 직접 알려 주지 않으며, 낮은 값도 전체 서비스의 정상 상태를 보장하지 않습니다.
응답 지연, 오류율, CPU, 이벤트 루프 지연, 프로파일 결과와 함께 해석하고 실제 부하 테스트로 개선 효과를 검증해야 합니다.

관련 글:

- [Node.js perf_hooks.createHistogram 가이드: 지연 시간 백분위수 측정하기](/development/blog/seo/2026/05/31/nodejs-perf-hooks-createhistogram-latency-percentile-guide.html)
- [Node.js scheduler.yield 가이드: 긴 작업 중 이벤트 루프에 실행 기회 주기](/development/blog/seo/2026/05/20/nodejs-timers-promises-scheduler-yield-event-loop-guide.html)
- [Node.js process.getActiveResourcesInfo 가이드: 프로세스 종료 지연 원인 찾기](/development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html)

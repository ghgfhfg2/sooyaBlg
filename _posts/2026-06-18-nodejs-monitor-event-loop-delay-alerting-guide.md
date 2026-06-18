---
layout: post
title: "Node.js monitorEventLoopDelay 가이드: 이벤트 루프 지연 알림 설계하기"
date: 2026-06-18 20:00:00 +0900
lang: ko
translation_key: nodejs-monitor-event-loop-delay-alerting-guide
permalink: /development/blog/seo/2026/06/18/nodejs-monitor-event-loop-delay-alerting-guide.html
alternates:
  ko: /development/blog/seo/2026/06/18/nodejs-monitor-event-loop-delay-alerting-guide.html
  x_default: /development/blog/seo/2026/06/18/nodejs-monitor-event-loop-delay-alerting-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, perf_hooks, eventloop, monitoring, observability, alerting, backend]
description: "Node.js perf_hooks의 monitorEventLoopDelay로 이벤트 루프 지연을 측정하고, p95와 max 값을 활용해 운영 알림 기준을 설계하는 방법을 정리합니다. CPU 점유, 동기 작업, GC 지연을 구분하는 실무 예제를 포함합니다."
---

Node.js 서버에서 응답 시간이 갑자기 늘어났는데 CPU 사용률, 데이터베이스 지연, 외부 API 오류만 봐서는 원인이 잘 보이지 않을 때가 있습니다.
이럴 때 놓치기 쉬운 신호가 이벤트 루프 지연입니다.
이벤트 루프가 오래 막히면 새 요청을 받아도 콜백, 타이머, Promise 후속 작업이 제때 실행되지 못하고 전체 서비스가 느려집니다.

`perf_hooks.monitorEventLoopDelay()`는 이런 지연을 히스토그램으로 관찰할 수 있게 해 줍니다.
단순히 "현재 느리다"를 확인하는 도구가 아니라, p95, p99, max 같은 분포 지표를 기반으로 알림 기준을 세울 수 있는 운영 지표입니다.

이 글에서는 Node.js 서비스에서 이벤트 루프 지연을 측정하고, 너무 민감하지 않으면서도 장애 신호를 놓치지 않는 알림 기준을 설계하는 방법을 정리합니다.
CPU 시간과 함께 보고 싶다면 [Node.js process.cpuUsage CPU 시간 모니터링 가이드](/development/blog/seo/2026/06/17/nodejs-process-cpuusage-cpu-time-monitoring-guide.html)를, 런타임 리소스 스냅샷이 필요하다면 [Node.js process.resourceUsage 런타임 리소스 진단 가이드](/development/blog/seo/2026/06/17/nodejs-process-resourceusage-runtime-resource-diagnostics-guide.html)를 함께 참고하세요.

## monitorEventLoopDelay가 보여 주는 것

### H3. 이벤트 루프 지연은 응답 시간과 같은 값이 아니다

응답 시간은 네트워크, 라우팅, 비즈니스 로직, 데이터베이스, 외부 API, 직렬화까지 포함한 요청 단위 결과입니다.
반면 이벤트 루프 지연은 Node.js 프로세스가 다음 작업을 실행하기까지 얼마나 밀렸는지를 보여 줍니다.

예를 들어 데이터베이스가 느려서 응답 시간이 늘어나는 상황과, CPU를 오래 쓰는 동기 코드 때문에 이벤트 루프가 막힌 상황은 운영 대응이 다릅니다.
이벤트 루프 지연을 별도 지표로 두면 "외부 의존성이 느린가"와 "프로세스 내부가 막혔는가"를 더 빨리 나눌 수 있습니다.

```js
import { monitorEventLoopDelay } from 'node:perf_hooks';

const eventLoopDelay = monitorEventLoopDelay({
  resolution: 20
});

eventLoopDelay.enable();
```

`resolution`은 샘플링 해상도입니다.
값을 너무 낮게 잡으면 측정 비용이 늘 수 있고, 너무 높게 잡으면 짧은 지연을 놓칠 수 있습니다.
일반적인 웹 서버에서는 10ms에서 50ms 사이로 시작한 뒤 운영 부하를 보며 조정하는 편이 무난합니다.

### H3. 평균보다 백분위와 최댓값을 우선 본다

이벤트 루프 지연은 짧은 순간 튀는 값이 중요합니다.
평균이 낮아도 특정 구간에서 500ms 이상 막히면 사용자 요청 일부는 크게 느려질 수 있습니다.
그래서 평균만 대시보드에 올리면 실제 체감 지연을 놓치기 쉽습니다.

```js
function readEventLoopDelaySnapshot(histogram) {
  return {
    minMs: histogram.min / 1e6,
    meanMs: histogram.mean / 1e6,
    p95Ms: histogram.percentile(95) / 1e6,
    p99Ms: histogram.percentile(99) / 1e6,
    maxMs: histogram.max / 1e6
  };
}
```

`monitorEventLoopDelay()`의 값은 나노초 단위입니다.
운영 로그와 메트릭에는 밀리초로 변환해서 저장하는 편이 읽기 쉽습니다.

## 운영 메트릭으로 내보내기

### H3. 주기적으로 읽고 reset한다

히스토그램은 계속 누적됩니다.
운영 지표로 쓰려면 일정 주기마다 값을 읽고 `reset()`해서 최근 구간의 상태를 보아야 합니다.
그렇지 않으면 오래전 스파이크가 계속 max 값에 남아 현재 상태를 왜곡할 수 있습니다.

```js
const INTERVAL_MS = 30_000;

setInterval(() => {
  const snapshot = readEventLoopDelaySnapshot(eventLoopDelay);

  publishMetric('node_event_loop_delay_p95_ms', snapshot.p95Ms);
  publishMetric('node_event_loop_delay_p99_ms', snapshot.p99Ms);
  publishMetric('node_event_loop_delay_max_ms', snapshot.maxMs);

  eventLoopDelay.reset();
}, INTERVAL_MS).unref();
```

30초 또는 60초 단위로 시작하면 알림 노이즈와 탐지 속도의 균형을 잡기 쉽습니다.
트래픽이 매우 짧게 몰리는 서비스라면 더 짧은 구간도 검토할 수 있지만, 그만큼 알림 기준을 완화해야 합니다.

### H3. 로그에는 이상 구간만 남긴다

모든 스냅샷을 로그로 남기면 비용이 늘고 검색하기 어려워집니다.
일반 상태는 메트릭으로 보내고, 임계값을 넘은 구간만 구조화 로그로 남기는 구성이 더 실용적입니다.

```js
function logEventLoopDelayIfNeeded(snapshot) {
  if (snapshot.p95Ms < 100 && snapshot.maxMs < 500) {
    return;
  }

  console.warn(JSON.stringify({
    event: 'event_loop_delay_high',
    p95Ms: Math.round(snapshot.p95Ms),
    p99Ms: Math.round(snapshot.p99Ms),
    maxMs: Math.round(snapshot.maxMs)
  }));
}
```

이 로그에는 요청 본문, 헤더, 사용자 입력을 넣지 않습니다.
이벤트 루프 지연은 프로세스 상태 지표이므로 민감정보 없이도 충분히 진단 가치가 있습니다.
로그 샘플링과 필드 제한 원칙은 [Node.js 로그 샘플링 가이드](/development/blog/seo/2026/06/18/nodejs-log-sampling-redaction-observability-guide.html)의 방식과 같이 적용하면 됩니다.

## 알림 기준 설계

### H3. 단일 스파이크보다 지속 시간을 본다

이벤트 루프 지연은 배포 직후 초기화, 짧은 GC, 일시적인 대량 JSON 직렬화 같은 이유로 순간적으로 튈 수 있습니다.
한 번의 max 값만으로 페이지 알림을 보내면 운영 피로가 빠르게 쌓입니다.

실무에서는 다음처럼 단계별 기준을 둡니다.

1. p95가 100ms 이상인 상태가 3분 이상 지속되면 경고한다.
2. p99가 250ms 이상인 상태가 3분 이상 지속되면 장애 후보로 본다.
3. max가 1초 이상이고 동시에 오류율이 증가하면 즉시 확인한다.

이 기준은 절대값이 아닙니다.
API 서버, 배치 워커, 웹소켓 서버는 허용 가능한 지연이 다르기 때문에 정상 상태의 기준선을 먼저 수집한 뒤 조정해야 합니다.

### H3. CPU 사용률과 함께 해석한다

이벤트 루프 지연이 높고 CPU 사용률도 높다면 동기 CPU 작업, 큰 JSON 처리, 압축, 암호화, 정규식 백트래킹 같은 원인을 의심할 수 있습니다.
반대로 이벤트 루프 지연은 높은데 CPU가 낮다면 긴 GC, 네이티브 애드온, 파일 시스템 병목, 컨테이너 스케줄링 문제를 함께 봐야 합니다.

```js
import process from 'node:process';

let previousCpu = process.cpuUsage();

function readCpuUsagePercent(intervalMs) {
  const current = process.cpuUsage(previousCpu);
  previousCpu = process.cpuUsage();

  const usedMicros = current.user + current.system;
  return (usedMicros / 1000 / intervalMs) * 100;
}
```

CPU 퍼센트는 컨테이너 CPU 제한과 워커 프로세스 수에 따라 해석이 달라질 수 있습니다.
그래서 숫자 하나만 보지 말고 이벤트 루프 지연, 요청 지연, 오류율, GC 로그를 함께 보는 것이 안전합니다.

### H3. 요청 경로와 배치 작업을 분리한다

같은 Node.js 프로세스에서 HTTP 요청과 무거운 배치 작업을 같이 처리하면 이벤트 루프 지연이 섞여 보입니다.
알림이 울렸을 때 어느 작업이 원인인지 알기 어렵고, 사용자 요청의 영향도 판단하기 힘듭니다.

가능하면 요청 처리 프로세스와 배치 워커를 분리하고, 메트릭 라벨도 다르게 둡니다.

```js
publishMetric('node_event_loop_delay_p95_ms', snapshot.p95Ms, {
  service: 'api',
  role: 'http',
  instanceId: process.env.HOSTNAME
});
```

라벨에는 민감정보나 사용자 식별자를 넣지 않습니다.
서비스명, 역할, 배포 환경, 인스턴스처럼 운영에 필요한 낮은 민감도의 값만 사용합니다.

## 지연 원인을 좁히는 점검 순서

### H3. 최근 배포와 트래픽 변화를 먼저 본다

이벤트 루프 지연 알림이 울리면 가장 먼저 최근 배포, 트래픽 급증, 특정 엔드포인트의 요청 증가를 확인합니다.
새 코드가 큰 배열 정렬, 동기 파일 읽기, 대량 직렬화, 무거운 암호화 작업을 요청 경로에 넣었을 수 있습니다.

```js
function expensiveHandler(items) {
  return items
    .filter((item) => item.enabled)
    .sort((a, b) => b.score - a.score)
    .slice(0, 100);
}
```

이런 코드는 데이터가 작을 때는 문제가 없어 보이다가 입력 크기가 커지면 이벤트 루프를 오래 점유합니다.
요청 경로에서는 입력 크기 제한, 사전 집계, 스트리밍, 워커 스레드 분리 같은 선택지를 검토해야 합니다.

### H3. 동기 API 사용을 확인한다

운영 요청 경로에서 `readFileSync`, `readdirSync`, `execSync`, 큰 `JSON.stringify`를 반복하면 이벤트 루프가 쉽게 막힙니다.
설정 파일을 시작 시 한 번 읽는 정도는 괜찮지만, 요청마다 동기 I/O를 수행하면 트래픽 증가와 함께 지연이 커집니다.

```js
// 요청마다 실행하지 말고 시작 시 로드하거나 비동기 캐시로 분리한다.
import { readFile } from 'node:fs/promises';

const config = JSON.parse(
  await readFile(new URL('./runtime-config.json', import.meta.url), 'utf8')
);
```

동기 API를 모두 금지할 필요는 없습니다.
중요한 것은 요청 처리 중 반복 실행되는 동기 작업을 찾아내고, 지연이 사용자 요청에 전파되지 않도록 분리하는 것입니다.

### H3. 워커 스레드와 큐를 대안으로 둔다

CPU를 많이 쓰는 작업은 비동기 함수로 감싼다고 이벤트 루프 부담이 사라지지 않습니다.
Promise는 실행 모델을 바꾸는 것이 아니라 결과 전달 방식을 바꾸는 것이기 때문입니다.
무거운 계산, 이미지 처리, 대량 압축, 복잡한 파싱은 워커 스레드나 별도 작업 큐로 분리하는 편이 좋습니다.

```js
import { Worker } from 'node:worker_threads';

function runHeavyJob(payload) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL('./heavy-job-worker.js', import.meta.url), {
      workerData: payload
    });

    worker.once('message', resolve);
    worker.once('error', reject);
    worker.once('exit', (code) => {
      if (code !== 0) reject(new Error(`worker stopped with code ${code}`));
    });
  });
}
```

워커 스레드는 만능 해결책이 아닙니다.
데이터 복사 비용, 워커 수 제한, 실패 처리, 큐 적체를 같이 설계해야 합니다.
그래도 사용자 요청 경로의 이벤트 루프를 보호하는 데는 효과적인 선택지입니다.

## 마무리 체크리스트

### H3. 운영 적용 전에 확인할 것

`monitorEventLoopDelay()`는 Node.js 서비스의 내부 혼잡도를 보는 데 유용하지만, 단독 지표로 장애를 단정하면 안 됩니다.
응답 시간, 오류율, CPU 사용률, 메모리, GC, 다운스트림 지연과 함께 보아야 원인을 더 정확히 좁힐 수 있습니다.

운영 적용 전에는 다음 항목을 확인해 보세요.

1. 히스토그램 값을 밀리초로 변환해 저장한다.
2. 일정 주기마다 값을 읽고 `reset()`한다.
3. 평균보다 p95, p99, max를 우선 대시보드에 올린다.
4. 단일 스파이크가 아니라 지속 시간 기준으로 알림을 만든다.
5. 로그에는 이상 구간만 남기고 민감정보를 포함하지 않는다.
6. CPU 사용률과 최근 배포 정보를 함께 확인한다.

이벤트 루프 지연 알림은 "서버가 느리다"는 막연한 느낌을 "프로세스 내부가 몇 ms 동안 막혔다"는 구체적인 신호로 바꿔 줍니다.
그 신호를 적절한 임계값, 내부링크된 리소스 지표, 안전한 로그 정책과 연결하면 운영 대응 속도와 품질을 함께 높일 수 있습니다.

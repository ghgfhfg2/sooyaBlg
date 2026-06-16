---
layout: post
title: "Node.js process.cpuUsage 가이드: CPU 시간으로 병목 구분하기"
date: 2026-06-17 08:00:00 +0900
lang: ko
translation_key: nodejs-process-cpuusage-cpu-time-monitoring-guide
permalink: /development/blog/seo/2026/06/17/nodejs-process-cpuusage-cpu-time-monitoring-guide.html
alternates:
  ko: /development/blog/seo/2026/06/17/nodejs-process-cpuusage-cpu-time-monitoring-guide.html
  x_default: /development/blog/seo/2026/06/17/nodejs-process-cpuusage-cpu-time-monitoring-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, cpuusage, process, monitoring, observability, performance]
description: "Node.js process.cpuUsage()로 user·system CPU 시간을 측정하고, 구간별 CPU 사용량 계산, 이벤트 루프 지표와의 차이, 운영 로그 설계와 민감정보 없는 진단 패턴을 정리합니다."
---

Node.js 서버가 느려졌을 때 "CPU가 바쁜 문제인지", "외부 API나 데이터베이스를 기다리는 문제인지", "이벤트 루프가 막힌 문제인지"를 구분하지 못하면 대응이 늦어집니다.
응답 시간이 길다는 사실만으로는 원인을 알 수 없습니다.
같은 지연이라도 CPU 계산이 많은 경우와 네트워크 대기가 많은 경우의 해결책은 완전히 다릅니다.

`process.cpuUsage()`는 현재 Node.js 프로세스가 사용한 CPU 시간을 `user`와 `system`으로 나누어 보여 주는 내장 API입니다.
이 값은 벽시계 시간과 다릅니다.
프로세스가 실제로 CPU에서 실행된 시간을 마이크로초 단위로 누적해서 보여 주기 때문에, 구간별 차이를 계산하면 "이 작업이 CPU를 얼마나 썼는지"를 가볍게 확인할 수 있습니다.

이 글에서는 `process.cpuUsage()`의 기본 구조, 구간별 측정 패턴, 운영 로그로 남길 때의 기준, 이벤트 루프 지표와 함께 해석하는 방법을 정리합니다.
이벤트 루프의 바쁨 자체를 보고 싶다면 [Node.js performance.eventLoopUtilization 가이드](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)를 함께 참고하세요.

## process.cpuUsage가 보여 주는 것

### H3. 반환값은 user와 system CPU 시간이다

`process.cpuUsage()`는 객체 하나를 반환합니다.
`user`는 애플리케이션 코드가 사용자 영역에서 사용한 CPU 시간이고, `system`은 커널 영역에서 사용한 CPU 시간입니다.
두 값 모두 마이크로초 단위입니다.

```js
const usage = process.cpuUsage();

console.log({
  userMs: usage.user / 1000,
  systemMs: usage.system / 1000,
  totalMs: (usage.user + usage.system) / 1000
});
```

마이크로초 값은 그대로 로그에 남기면 직관적으로 읽기 어렵습니다.
대부분의 운영 로그와 대시보드에서는 밀리초로 바꿔 저장하는 편이 좋습니다.

### H3. 누적값보다 구간 차이가 더 유용하다

인자 없이 호출한 값은 프로세스 시작 이후의 누적 CPU 시간입니다.
운영 진단에서는 전체 누적값보다 특정 구간의 차이가 더 중요합니다.

```js
const start = process.cpuUsage();

await runJob();

const diff = process.cpuUsage(start);

console.log({
  jobCpuUserMs: Math.round(diff.user / 1000),
  jobCpuSystemMs: Math.round(diff.system / 1000),
  jobCpuTotalMs: Math.round((diff.user + diff.system) / 1000)
});
```

이 패턴은 배치 작업, 이미지 처리, 리포트 생성, 압축, 암호화처럼 CPU 비용이 의심되는 구간에 잘 맞습니다.
벽시계 시간과 CPU 시간을 함께 남기면 대기 시간이 큰 작업인지 계산 시간이 큰 작업인지 더 쉽게 구분할 수 있습니다.

## 벽시계 시간과 함께 측정하기

### H3. elapsedMs와 cpuMs를 같이 남긴다

CPU 시간만 보면 작업이 실제로 얼마나 오래 걸렸는지 알 수 없습니다.
반대로 `performance.now()` 같은 벽시계 시간만 보면 그 시간 동안 CPU를 썼는지 기다렸는지 알 수 없습니다.
두 값을 같이 기록해야 진단 가치가 생깁니다.

```js
import { performance } from 'node:perf_hooks';

async function measureJob(name, job) {
  const startedAt = performance.now();
  const startedCpu = process.cpuUsage();

  try {
    return await job();
  } finally {
    const elapsedMs = performance.now() - startedAt;
    const cpu = process.cpuUsage(startedCpu);
    const cpuMs = (cpu.user + cpu.system) / 1000;

    console.log({
      name,
      elapsedMs: Number(elapsedMs.toFixed(1)),
      cpuMs: Number(cpuMs.toFixed(1)),
      cpuRatio: Number((cpuMs / elapsedMs).toFixed(3))
    });
  }
}
```

`cpuRatio`는 단순한 보조 지표입니다.
단일 스레드 작업에서는 대략 1에 가까울수록 CPU를 많이 썼다고 볼 수 있고, 0에 가까울수록 대기 시간이 많았다고 해석할 수 있습니다.
다만 워커 스레드나 네이티브 작업이 섞이면 해석이 달라질 수 있으므로 절대 기준으로 쓰면 안 됩니다.

### H3. CPU 시간이 벽시계 시간보다 커질 수 있다

멀티코어 환경에서 여러 스레드가 동시에 CPU를 쓰면, 프로세스의 CPU 시간 합계가 실제 경과 시간보다 커질 수 있습니다.
예를 들어 1초 동안 두 스레드가 각각 1초씩 CPU를 쓰면 총 CPU 시간은 2초에 가까워질 수 있습니다.

```js
function summarizeCpuUsage(cpu, elapsedMs) {
  const userMs = cpu.user / 1000;
  const systemMs = cpu.system / 1000;
  const totalMs = userMs + systemMs;

  return {
    elapsedMs: Number(elapsedMs.toFixed(1)),
    userMs: Number(userMs.toFixed(1)),
    systemMs: Number(systemMs.toFixed(1)),
    totalMs: Number(totalMs.toFixed(1)),
    cpuPerElapsed: Number((totalMs / elapsedMs).toFixed(2))
  };
}
```

따라서 `cpuPerElapsed`가 1보다 크다고 해서 측정이 잘못된 것은 아닙니다.
중요한 것은 같은 서비스 안에서 같은 방식으로 반복 측정하고, 평소 기준선과 달라지는 지점을 찾는 것입니다.

## 실무에서 쓰는 측정 위치

### H3. 요청 단위 전체에 무조건 붙이지 않는다

모든 HTTP 요청마다 CPU 사용량을 자세히 계산하면 로그가 커지고 해석도 어려워집니다.
대부분의 요청은 프레임워크, 미들웨어, 외부 대기 시간이 섞여 있어 CPU 병목만 분리하기 어렵습니다.

대신 다음처럼 비용이 큰 경계에 제한적으로 붙이는 편이 좋습니다.

1. 리포트 생성, 대량 변환, 압축 같은 CPU 의심 작업
2. 정기 배치나 큐 워커의 작업 처리 함수
3. 성능 회귀가 의심되는 특정 코드 경로
4. 장애 대응 중 임시로 켠 진단 로그

측정 코드는 원인을 찾기 위한 도구입니다.
처음부터 모든 요청의 기본 로그로 넣기보다, 관찰 대상과 기간을 정해 두는 방식이 운영 비용을 줄입니다.

### H3. 큐 워커에서는 작업 종류별로 요약한다

큐 워커는 같은 프로세스 안에서 서로 다른 작업을 반복 처리합니다.
이때 작업 이름별로 CPU 시간과 경과 시간을 남기면 어떤 작업이 CPU를 많이 쓰는지 빠르게 볼 수 있습니다.

```js
async function runWorkerJob(job) {
  const wallStart = performance.now();
  const cpuStart = process.cpuUsage();

  try {
    await handlers[job.type](job.payload);
  } finally {
    const elapsedMs = performance.now() - wallStart;
    const cpu = process.cpuUsage(cpuStart);
    const cpuMs = (cpu.user + cpu.system) / 1000;

    logger.info({
      event: 'worker.job.completed',
      jobType: job.type,
      elapsedMs: Number(elapsedMs.toFixed(1)),
      cpuMs: Number(cpuMs.toFixed(1))
    });
  }
}
```

여기서 `job.payload` 전체를 로그에 남기면 안 됩니다.
작업 종류, 제한된 상태값, 숫자 지표만 남기고 사용자 입력, 토큰, 원문 데이터는 제외해야 합니다.

## eventLoopUtilization과 함께 해석하기

### H3. cpuUsage는 CPU 시간, ELU는 이벤트 루프 상태다

`process.cpuUsage()`와 `performance.eventLoopUtilization()`은 비슷해 보이지만 답하는 질문이 다릅니다.
`cpuUsage`는 프로세스가 CPU를 얼마나 사용했는지 보여 줍니다.
`eventLoopUtilization`은 이벤트 루프가 관찰 구간 동안 얼마나 바쁘게 돌았는지 보여 줍니다.

```js
import { performance } from 'node:perf_hooks';

const startCpu = process.cpuUsage();
const startElu = performance.eventLoopUtilization();
const startTime = performance.now();

await runScenario();

const elapsedMs = performance.now() - startTime;
const cpu = process.cpuUsage(startCpu);
const elu = performance.eventLoopUtilization(startElu);

console.log({
  elapsedMs: Number(elapsedMs.toFixed(1)),
  cpuMs: Number(((cpu.user + cpu.system) / 1000).toFixed(1)),
  eventLoopUtilization: Number(elu.utilization.toFixed(4))
});
```

CPU 시간이 높고 ELU도 높다면 JavaScript 실행, JSON 처리, 압축, 암호화 같은 CPU 작업을 의심할 수 있습니다.
CPU 시간은 낮은데 ELU가 높다면 짧은 콜백이 지나치게 많거나 이벤트 루프를 자주 깨우는 구조를 확인해야 합니다.
CPU 시간도 낮고 ELU도 낮은데 응답 시간이 길다면 외부 시스템 대기나 큐 대기를 먼저 살펴보는 편이 자연스럽습니다.

### H3. 함수 단위 duration은 timerify로 보완한다

`process.cpuUsage()`는 프로세스 단위 값입니다.
특정 함수 하나의 duration을 계측하려면 `performance.timerify()`나 `performance.mark()`와 `performance.measure()`를 함께 쓰는 편이 더 명확합니다.

```js
import { performance, PerformanceObserver } from 'node:perf_hooks';

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log({
      name: entry.name,
      durationMs: Number(entry.duration.toFixed(1))
    });
  }
});

observer.observe({ entryTypes: ['function'] });

const measuredBuild = performance.timerify(buildSearchIndex);
await measuredBuild();
```

함수별 duration과 프로세스 CPU 시간은 서로 보완 관계입니다.
함수 실행 시간이 길고 구간 CPU도 높다면 계산 최적화를 검토하고, 함수 실행 시간은 긴데 CPU가 낮다면 외부 대기나 잠금 경합을 의심할 수 있습니다.
함수 계측 패턴은 [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)에서 더 자세히 볼 수 있습니다.

## 운영 로그 설계 기준

### H3. 낮은 카디널리티 필드만 남긴다

CPU 진단 로그는 메트릭 시스템이나 로그 검색 도구로 이어질 가능성이 큽니다.
필드 값이 너무 다양하면 집계 비용이 커지고, 개인정보가 섞일 위험도 높아집니다.

```js
logger.info({
  event: 'cpu.sample',
  component: 'report-worker',
  operation: 'pdf-render',
  elapsedMs,
  cpuUserMs,
  cpuSystemMs,
  cpuTotalMs
});
```

`component`, `operation`처럼 미리 정해진 값은 집계에 적합합니다.
반대로 이메일, IP 주소, 전체 URL, 검색어, 주문번호 같은 값은 CPU 로그에 넣지 않는 편이 안전합니다.

### H3. 샘플링과 임계값을 둔다

CPU 사용량을 모든 작업마다 남기면 정상 상황에서도 로그가 빠르게 늘어납니다.
운영에서는 샘플링하거나 임계값을 넘는 경우에만 자세한 로그를 남기는 방식이 좋습니다.

```js
function shouldLogCpuSample({ elapsedMs, cpuMs }) {
  if (elapsedMs >= 1000) return true;
  if (cpuMs >= 250) return true;
  return Math.random() < 0.01;
}
```

임계값은 서비스 특성에 맞춰 조정해야 합니다.
API 서버, 배치 워커, CLI 도구는 정상 지연 시간과 허용 가능한 CPU 비용이 서로 다릅니다.

## 테스트와 로컬 진단에서의 활용

### H3. 성능 회귀 테스트는 느슨한 기준으로 둔다

CPU 시간은 실행 환경, 노트북 상태, CI 머신 부하에 따라 흔들립니다.
따라서 단위 테스트에서 "반드시 몇 ms 이하" 같은 딱딱한 기준을 두면 불안정한 테스트가 되기 쉽습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { performance } from 'node:perf_hooks';

test('index build does not spend excessive CPU in sample data', async () => {
  const startedAt = performance.now();
  const startedCpu = process.cpuUsage();

  await buildIndex(createSmallFixture());

  const elapsedMs = performance.now() - startedAt;
  const cpu = process.cpuUsage(startedCpu);
  const cpuMs = (cpu.user + cpu.system) / 1000;

  assert.ok(elapsedMs < 2000);
  assert.ok(cpuMs < 1500);
});
```

이런 테스트는 정확한 성능 보증보다 큰 회귀를 잡는 안전망에 가깝습니다.
실제 성능 판단은 반복 실행, 고정된 입력, 별도 벤치마크 환경에서 보는 것이 더 신뢰할 수 있습니다.

### H3. 로컬 스크립트는 전후 차이를 출력한다

운영 코드에 넣기 전 로컬 진단 스크립트로 병목 후보를 좁힐 수 있습니다.
반복 횟수를 늘리고 작업 전후의 CPU 차이를 출력하면 개선 전후 비교가 쉬워집니다.

```js
async function runBenchmark(label, fn) {
  const startedAt = performance.now();
  const startedCpu = process.cpuUsage();

  await fn();

  const elapsedMs = performance.now() - startedAt;
  const cpu = process.cpuUsage(startedCpu);

  console.log(label, summarizeCpuUsage(cpu, elapsedMs));
}

await runBenchmark('json-parse-large-fixture', async () => {
  for (let index = 0; index < 100; index += 1) {
    JSON.parse(largeJsonText);
  }
});
```

출력에는 실제 고객 데이터나 내부 토큰을 포함하지 않아야 합니다.
벤치마크 입력도 가능하면 익명화된 fixture나 합성 데이터를 사용해야 재현성과 안전성을 함께 지킬 수 있습니다.

## 실무 적용 체크리스트

### H3. cpuUsage를 붙이기 전에 확인할 것

다음 기준을 두면 `process.cpuUsage()`를 과하게 쓰지 않으면서도 진단 가치를 얻을 수 있습니다.

1. 누적값이 아니라 `process.cpuUsage(previous)`로 구간 차이를 계산한다.
2. `performance.now()`로 잰 경과 시간과 CPU 시간을 함께 남긴다.
3. 로그 필드는 작업 종류, 컴포넌트, 숫자 지표처럼 낮은 카디널리티로 제한한다.
4. 민감정보, 사용자 입력, 전체 payload를 CPU 진단 로그에 포함하지 않는다.
5. 이벤트 루프 지표, 함수 duration, 프로세스 리포트와 역할을 나눠 해석한다.

CPU 사용량은 단독으로 원인을 확정하는 지표가 아닙니다.
다만 응답 지연의 성격을 빠르게 가르는 첫 번째 단서로는 충분히 유용합니다.

### H3. 장애 대응에서는 process.report와 연결한다

CPU 사용량이 비정상적으로 높다는 사실을 확인했다면, 다음 질문은 "어떤 코드와 런타임 상태가 그 순간에 있었는가"입니다.
장애 순간의 스택, 환경, 리소스 정보를 남겨야 한다면 `process.report`를 함께 사용할 수 있습니다.

```js
if (cpuMs > 1000 && elapsedMs > 1000) {
  logger.warn({
    event: 'high.cpu.window',
    operation,
    elapsedMs,
    cpuMs
  });
}
```

진단 리포트는 내부 경로, 환경 변수, 런타임 정보를 포함할 수 있으므로 보관 위치와 접근 권한을 별도로 관리해야 합니다.
운영 장애 자료를 안전하게 남기는 기준은 [Node.js process.report 가이드](/development/blog/seo/2026/05/29/nodejs-process-report-production-diagnostics-guide.html)를 참고하세요.

## 마무리

`process.cpuUsage()`는 Node.js 프로세스의 CPU 시간을 가볍게 확인할 수 있는 내장 도구입니다.
누적값을 그대로 보는 것보다 `process.cpuUsage(previous)`로 구간 차이를 계산하고, 벽시계 시간과 함께 남길 때 실무 가치가 커집니다.

핵심은 해석의 범위를 분명히 하는 것입니다.
`cpuUsage`는 CPU 시간을 보여 주고, `eventLoopUtilization`은 이벤트 루프의 바쁨을 보여 주며, `timerify`와 `mark/measure`는 함수나 작업 단위 duration을 보여 줍니다.
이 지표들을 함께 보면 "느리다"는 증상을 CPU 병목, 이벤트 루프 포화, 외부 대기 중 어디에 가까운지 더 빠르게 좁힐 수 있습니다.

다음 글을 함께 보면 Node.js 성능 진단 흐름을 더 체계적으로 연결할 수 있습니다.

- [Node.js performance.eventLoopUtilization 가이드](/development/blog/seo/2026/06/11/nodejs-performance-eventlooputilization-monitoring-guide.html)
- [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)
- [Node.js process.report 가이드](/development/blog/seo/2026/05/29/nodejs-process-report-production-diagnostics-guide.html)

---
layout: post
title: "Node.js process.resourceUsage 가이드: 런타임 리소스 사용량 진단하기"
date: 2026-06-17 20:00:00 +0900
lang: ko
translation_key: nodejs-process-resourceusage-runtime-resource-diagnostics-guide
permalink: /development/blog/seo/2026/06/17/nodejs-process-resourceusage-runtime-resource-diagnostics-guide.html
alternates:
  ko: /development/blog/seo/2026/06/17/nodejs-process-resourceusage-runtime-resource-diagnostics-guide.html
  x_default: /development/blog/seo/2026/06/17/nodejs-process-resourceusage-runtime-resource-diagnostics-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, resourceusage, process, monitoring, observability, diagnostics]
description: "Node.js process.resourceUsage()로 CPU 시간, 최대 RSS, 파일 시스템 I/O, 컨텍스트 스위치 같은 런타임 리소스 지표를 확인하고 운영 진단 로그에 안전하게 연결하는 방법을 정리합니다."
---

Node.js 애플리케이션에서 장애가 발생했을 때 응답 시간이나 에러 로그만으로는 원인을 좁히기 어렵습니다.
CPU를 많이 쓴 것인지, 메모리 사용량이 치솟은 것인지, 파일 시스템 I/O가 갑자기 늘어난 것인지, 스케줄링 지연이 섞인 것인지가 모두 다르게 대응해야 할 신호이기 때문입니다.

`process.resourceUsage()`는 현재 Node.js 프로세스의 리소스 사용량을 한 번에 확인할 수 있는 내장 API입니다.
CPU 시간, 최대 RSS, 파일 시스템 읽기/쓰기 횟수, 자발적·비자발적 컨텍스트 스위치 같은 값을 운영 진단에 붙일 수 있습니다.
이 글에서는 `process.resourceUsage()`의 반환값을 어떻게 읽고, 어떤 지표를 로그로 남기며, `process.cpuUsage()`나 성능 타임라인 지표와 어떻게 함께 해석할지 정리합니다.

CPU 시간만 먼저 보고 싶다면 [Node.js process.cpuUsage 가이드](/development/blog/seo/2026/06/17/nodejs-process-cpuusage-cpu-time-monitoring-guide.html)를 함께 참고하세요.

## process.resourceUsage가 필요한 순간

### H3. 단일 지표로는 장애 원인을 확정하기 어렵다

운영에서 "느리다"는 증상은 여러 원인으로 나타납니다.
CPU 계산이 많아도 느려지고, 디스크 I/O가 밀려도 느려지고, 메모리 압박으로 GC가 자주 발생해도 느려집니다.

`process.resourceUsage()`는 이런 상황에서 프로세스 단위의 넓은 스냅샷을 제공합니다.
정밀 프로파일러는 아니지만, 어느 영역을 더 깊게 봐야 하는지 가르는 1차 진단 지표로 유용합니다.

```js
const usage = process.resourceUsage();

console.log({
  userCpuMs: usage.userCPUTime / 1000,
  systemCpuMs: usage.systemCPUTime / 1000,
  maxRssKb: usage.maxRSS,
  fsRead: usage.fsRead,
  fsWrite: usage.fsWrite,
  voluntaryContextSwitches: usage.voluntaryContextSwitches,
  involuntaryContextSwitches: usage.involuntaryContextSwitches
});
```

반환값은 플랫폼에 따라 해석 가능한 범위가 다를 수 있습니다.
그래서 절대값 하나로 결론을 내리기보다 같은 환경에서 같은 방식으로 반복 측정해 기준선을 잡는 편이 좋습니다.

### H3. 누적 스냅샷이라는 점을 먼저 이해한다

`process.resourceUsage()`는 호출 시점의 누적 리소스 사용량을 반환합니다.
특정 작업 하나의 비용을 알고 싶다면 작업 전후 값을 비교해야 합니다.

```js
function diffResourceUsage(before, after) {
  return {
    userCpuMs: (after.userCPUTime - before.userCPUTime) / 1000,
    systemCpuMs: (after.systemCPUTime - before.systemCPUTime) / 1000,
    fsRead: after.fsRead - before.fsRead,
    fsWrite: after.fsWrite - before.fsWrite,
    voluntaryContextSwitches:
      after.voluntaryContextSwitches - before.voluntaryContextSwitches,
    involuntaryContextSwitches:
      after.involuntaryContextSwitches - before.involuntaryContextSwitches
  };
}
```

`maxRSS`는 최대 resident set size라서 전후 차이를 단순히 빼는 방식이 항상 의미 있지는 않습니다.
작업 구간 중 최고점을 확인하는 참고값으로 보고, 현재 메모리 사용량은 `process.memoryUsage()`와 함께 보는 편이 더 실무적입니다.

## 운영 로그에 남길 핵심 필드

### H3. CPU 시간은 user와 system을 나눠 본다

`userCPUTime`은 애플리케이션 코드가 사용자 영역에서 쓴 CPU 시간이고, `systemCPUTime`은 커널 영역 작업에 쓴 CPU 시간입니다.
두 값은 마이크로초 단위이므로 로그나 대시보드에서는 밀리초로 변환하는 편이 읽기 쉽습니다.

```js
function summarizeCpuResource(usage) {
  const userCpuMs = usage.userCPUTime / 1000;
  const systemCpuMs = usage.systemCPUTime / 1000;

  return {
    userCpuMs: Number(userCpuMs.toFixed(1)),
    systemCpuMs: Number(systemCpuMs.toFixed(1)),
    totalCpuMs: Number((userCpuMs + systemCpuMs).toFixed(1))
  };
}
```

JavaScript 계산, JSON 직렬화, 암호화, 압축 같은 작업이 많으면 사용자 CPU 시간이 늘어날 수 있습니다.
반대로 파일 시스템, 네트워크, 커널 경계 작업이 많으면 시스템 CPU 시간이 함께 올라갈 수 있습니다.
다만 이 값만으로 코드 위치를 특정할 수는 없으므로, 함수 단위 측정이나 프로파일링으로 이어가는 단서로 써야 합니다.

### H3. maxRSS는 메모리 압박의 거친 신호다

`maxRSS`는 프로세스가 사용한 최대 RSS를 보여 줍니다.
Node.js heap만 보는 지표가 아니기 때문에 큰 `Buffer`, 네이티브 모듈, 이미지 처리, 압축 해제처럼 V8 heap 밖의 사용량이 의심될 때 참고할 수 있습니다.

```js
function summarizeMemoryResource(usage) {
  return {
    maxRssMb: Number((usage.maxRSS / 1024).toFixed(1)),
    heapUsedMb: Number((process.memoryUsage().heapUsed / 1024 / 1024).toFixed(1)),
    externalMb: Number((process.memoryUsage().external / 1024 / 1024).toFixed(1))
  };
}
```

`maxRSS`와 `heapUsed`를 함께 보면 "V8 heap이 늘어난 문제"인지, "프로세스 전체 메모리가 커진 문제"인지 구분하기 쉬워집니다.
메모리 여유와 작업 거절 기준은 [Node.js process.availableMemory 가이드](/development/blog/seo/2026/05/21/nodejs-process-availablememory-memory-pressure-guide.html)를 같이 보면 흐름이 이어집니다.

### H3. fsRead와 fsWrite는 I/O 급증을 찾는 보조 지표다

파일 시스템 I/O가 많은 작업은 CPU보다 대기 시간이 커질 수 있습니다.
`fsRead`와 `fsWrite`는 프로세스가 수행한 파일 시스템 읽기·쓰기 횟수를 보여 주므로, 특정 작업 전후의 증가량을 보면 I/O 성격을 대략 확인할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

async function measureResourceWindow(name, task) {
  const started = process.resourceUsage();
  const startedAt = performance.now();

  try {
    return await task();
  } finally {
    const elapsedMs = performance.now() - startedAt;
    const ended = process.resourceUsage();
    const diff = diffResourceUsage(started, ended);

    logger.info({
      event: 'resource.window',
      name,
      elapsedMs: Number(elapsedMs.toFixed(1)),
      ...diff
    });
  }
}
```

이 로그에는 파일 경로, 원문 payload, 사용자 입력을 넣지 않는 편이 안전합니다.
리소스 지표는 작업 이름, 컴포넌트, 숫자 값만으로도 충분히 집계할 수 있습니다.

## 컨텍스트 스위치 지표 해석하기

### H3. voluntaryContextSwitches는 대기와 양보의 흔적이다

자발적 컨텍스트 스위치는 프로세스가 I/O 대기나 락 대기 등으로 CPU를 양보한 흔적에 가깝습니다.
이 값이 특정 작업에서 크게 늘면 CPU 계산보다 외부 대기나 커널 대기가 섞여 있을 가능성을 검토할 수 있습니다.

```js
const before = process.resourceUsage();

await runFileHeavyJob();

const after = process.resourceUsage();

console.log({
  voluntaryContextSwitches:
    after.voluntaryContextSwitches - before.voluntaryContextSwitches
});
```

값 자체가 높다고 바로 문제는 아닙니다.
서버는 정상 상황에서도 네트워크와 파일 시스템을 기다리며 컨텍스트 스위치를 만듭니다.
중요한 것은 평소와 비교해 어느 작업, 어느 배포 이후, 어느 입력 크기에서 증가했는지입니다.

### H3. involuntaryContextSwitches는 CPU 경합의 단서가 될 수 있다

비자발적 컨텍스트 스위치는 운영체제가 프로세스를 다른 작업 때문에 밀어낸 경우를 포함합니다.
이 값이 급격히 늘고 CPU 시간도 같이 높다면 같은 호스트의 CPU 경합, 워커 수 과다, 무거운 동시 작업을 의심할 수 있습니다.

```js
function summarizeContextSwitches(before, after) {
  return {
    voluntary:
      after.voluntaryContextSwitches - before.voluntaryContextSwitches,
    involuntary:
      after.involuntaryContextSwitches - before.involuntaryContextSwitches
  };
}
```

컨텍스트 스위치는 원인 확정 지표가 아닙니다.
대신 큐 워커 동시성, CPU 바운드 작업, 컨테이너 CPU 제한, 호스트 부하를 확인해야 할지 결정하는 방향 표시로 쓰면 좋습니다.

## 배치와 큐 워커에서 적용하기

### H3. 작업 종류별로 리소스 요약을 남긴다

배치나 큐 워커는 같은 프로세스 안에서 반복적으로 작업을 처리합니다.
이때 작업 종류별 리소스 사용량을 남기면 어떤 작업이 CPU, 메모리, I/O를 많이 쓰는지 빠르게 비교할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

async function runJobWithResourceLog(job) {
  const before = process.resourceUsage();
  const startedAt = performance.now();

  try {
    return await handlers[job.type](job.payload);
  } finally {
    const after = process.resourceUsage();
    const elapsedMs = performance.now() - startedAt;
    const diff = diffResourceUsage(before, after);

    logger.info({
      event: 'worker.job.resource',
      jobType: job.type,
      elapsedMs: Number(elapsedMs.toFixed(1)),
      maxRssMb: Number((after.maxRSS / 1024).toFixed(1)),
      ...diff
    });
  }
}
```

`job.payload` 전체를 로그에 남기면 민감정보가 섞일 수 있습니다.
작업 유형, 성공 여부, 제한된 에러 코드, 숫자 지표만 남기고 상세 입력은 별도 보안 기준을 따르는 저장소에 보관해야 합니다.

### H3. 임계값을 넘을 때만 자세히 남긴다

모든 작업의 리소스 스냅샷을 상세 로그로 남기면 비용이 커집니다.
평상시에는 요약 메트릭만 남기고, 특정 임계값을 넘을 때만 자세한 진단 로그를 남기는 편이 운영에 적합합니다.

```js
function shouldLogResourceDetail({ elapsedMs, totalCpuMs, fsRead, fsWrite }) {
  if (elapsedMs >= 2000) return true;
  if (totalCpuMs >= 500) return true;
  if (fsRead + fsWrite >= 1000) return true;
  return Math.random() < 0.01;
}
```

임계값은 서비스 성격에 따라 달라집니다.
API 서버는 짧은 지연에 민감하고, 리포트 배치는 긴 경과 시간을 허용할 수 있으며, 파일 변환 워커는 I/O 증가가 정상일 수도 있습니다.

## 다른 Node.js 진단 API와 함께 보기

### H3. cpuUsage와 resourceUsage는 범위가 다르다

`process.cpuUsage()`는 CPU 시간에 집중합니다.
`process.resourceUsage()`는 CPU 시간 외에도 메모리 최고점, 파일 시스템 I/O, 컨텍스트 스위치까지 넓게 보여 줍니다.

```js
const cpuStart = process.cpuUsage();
const resourceStart = process.resourceUsage();

await runScenario();

const cpuDiff = process.cpuUsage(cpuStart);
const resourceEnd = process.resourceUsage();

console.log({
  cpuTotalMs: (cpuDiff.user + cpuDiff.system) / 1000,
  maxRssMb: resourceEnd.maxRSS / 1024,
  contextSwitches: summarizeContextSwitches(resourceStart, resourceEnd)
});
```

특정 구간의 CPU만 빠르게 확인하려면 `process.cpuUsage()`가 간단합니다.
장애 순간의 리소스 단서를 넓게 보고 싶다면 `process.resourceUsage()`가 더 적합합니다.

### H3. 성능 타임라인과 조합하면 위치를 좁힐 수 있다

`process.resourceUsage()`는 프로세스 단위 지표라서 어떤 함수가 문제인지 직접 알려 주지는 않습니다.
작업 위치를 좁히려면 `performance.mark()`, `performance.measure()`, `performance.timerify()` 같은 성능 타임라인 도구를 함께 써야 합니다.

```js
import { performance } from 'node:perf_hooks';

const resourceStart = process.resourceUsage();

performance.mark('report:start');
await buildReport();
performance.mark('report:end');
performance.measure('report', 'report:start', 'report:end');

const resourceEnd = process.resourceUsage();

console.log({
  entries: performance.getEntriesByName('report'),
  resources: diffResourceUsage(resourceStart, resourceEnd)
});
```

성능 타임라인 조회는 [Node.js performance.getEntriesByName 가이드](/development/blog/seo/2026/06/16/nodejs-performance-getentriesbyname-named-timing-query-guide.html)에서 더 자세히 정리했습니다.
프로세스 단위 리소스 지표와 함수·작업 단위 duration을 같이 보면 "무엇이 느렸는지"와 "그때 어떤 리소스를 썼는지"를 함께 볼 수 있습니다.

## 실무 적용 체크리스트

### H3. resourceUsage 로그 설계 기준

`process.resourceUsage()`를 운영 코드에 넣을 때는 다음 기준을 먼저 정하면 좋습니다.

1. 누적값을 그대로 해석하지 말고 작업 전후 차이를 계산한다.
2. CPU 시간은 마이크로초에서 밀리초로 변환해 저장한다.
3. `maxRSS`는 최고점 지표로 보고, 현재 heap 지표와 함께 해석한다.
4. 파일 경로, 사용자 입력, payload 원문 같은 민감한 값은 로그에서 제외한다.
5. 작업 유형, 컴포넌트, 숫자 지표처럼 낮은 카디널리티 필드만 남긴다.
6. 모든 요청에 무조건 붙이기보다 배치, 큐 워커, 장애 대응 구간에 우선 적용한다.

리소스 지표는 많을수록 좋은 것이 아니라, 해석 가능한 형태로 꾸준히 쌓일 때 가치가 생깁니다.
처음에는 작업 종류별 요약부터 시작하고, 문제가 반복되는 구간에만 더 자세한 로그를 붙이는 방식이 안정적입니다.

### H3. 장애 대응 메모에 남길 질문

장애 후 회고에서는 숫자만 남기지 말고 질문까지 함께 남기는 편이 좋습니다.

1. CPU 시간과 경과 시간이 함께 증가했는가?
2. `maxRSS`가 평소 기준선보다 높았는가?
3. 파일 시스템 I/O가 특정 작업에서만 급증했는가?
4. 컨텍스트 스위치 증가가 동시성 변경이나 배포 시점과 맞물렸는가?
5. 같은 구간의 성능 타임라인 entry와 연결되는가?

이 질문에 답하면 다음 액션이 더 분명해집니다.
CPU 프로파일을 떠야 하는지, 워커 동시성을 낮춰야 하는지, 메모리 사용량을 줄여야 하는지, I/O 배치를 다시 설계해야 하는지를 구분할 수 있습니다.

## 마무리

`process.resourceUsage()`는 Node.js 프로세스의 리소스 사용량을 넓게 확인하는 실무형 진단 도구입니다.
CPU 시간만 보는 것보다 더 많은 단서를 제공하고, 메모리 최고점·파일 시스템 I/O·컨텍스트 스위치를 함께 보게 해 줍니다.

다만 이 API는 원인을 직접 지목하는 도구가 아닙니다.
작업 전후 차이를 계산하고, 벽시계 시간과 함께 기록하고, 성능 타임라인과 연결할 때 진단 가치가 커집니다.
운영 로그에는 민감정보 없이 낮은 카디널리티 필드만 남기고, 기준선과 비교해 달라진 지점을 찾는 방식으로 활용하는 것이 좋습니다.

다음 글을 함께 보면 Node.js 운영 진단 흐름을 더 촘촘하게 연결할 수 있습니다.

- [Node.js process.cpuUsage 가이드](/development/blog/seo/2026/06/17/nodejs-process-cpuusage-cpu-time-monitoring-guide.html)
- [Node.js process.availableMemory 가이드](/development/blog/seo/2026/05/21/nodejs-process-availablememory-memory-pressure-guide.html)
- [Node.js performance.getEntriesByName 가이드](/development/blog/seo/2026/06/16/nodejs-performance-getentriesbyname-named-timing-query-guide.html)

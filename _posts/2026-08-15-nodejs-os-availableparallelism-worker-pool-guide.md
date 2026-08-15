---
layout: post
title: "Node.js os.availableParallelism 가이드: 워커 수를 안전하게 정하는 법"
date: 2026-08-15 20:00:00 +0900
lang: ko
translation_key: nodejs-os-availableparallelism-worker-pool-guide
permalink: /development/blog/seo/2026/08/15/nodejs-os-availableparallelism-worker-pool-guide.html
alternates:
  ko: /development/blog/seo/2026/08/15/nodejs-os-availableparallelism-worker-pool-guide.html
  x_default: /development/blog/seo/2026/08/15/nodejs-os-availableparallelism-worker-pool-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, os, availableparallelism, worker-threads, concurrency, performance, backend, javascript]
description: "Node.js os.availableParallelism()으로 워커 스레드, 큐 처리, 배치 작업의 기본 병렬도를 정하는 방법을 정리합니다. os.cpus().length와의 차이, 컨테이너 환경, 과도한 병렬 실행 방지, 설정 우선순위까지 실무 예제로 설명합니다."
---

Node.js에서 CPU를 많이 쓰는 작업을 병렬로 처리하려면 "워커를 몇 개 띄울 것인가"를 먼저 정해야 합니다.
이미지 변환, 압축, 대량 JSON 파싱, 리포트 생성, 테스트 샤딩처럼 CPU 시간이 중요한 작업에서는 워커 수가 너무 적어도 느리고, 너무 많아도 컨텍스트 스위칭과 메모리 사용량 때문에 오히려 느려질 수 있습니다.

예전 코드에서는 `os.cpus().length`를 기본값으로 쓰는 경우가 많았습니다.
하지만 이 값은 실제 프로세스가 사용할 수 있는 병렬 실행 가능량과 다를 수 있습니다.
컨테이너, CI runner, 프로세스 affinity, 운영체제 정책이 끼어드는 환경에서는 더 보수적인 기준이 필요합니다.

`os.availableParallelism()`은 현재 프로세스가 사용할 수 있는 병렬도에 맞춰 기본 워커 수를 정할 때 더 적합한 API입니다.
이 글에서는 `os.availableParallelism()`을 기준으로 워커 풀 크기를 계산하는 방법, 사용자 설정과 기본값의 우선순위, 과도한 병렬 실행을 막는 실무 패턴을 정리합니다.
워커 내부에 실행 환경을 전달하는 방식은 [Node.js worker_threads environment data 가이드](/development/blog/seo/2026/05/19/nodejs-worker-threads-environment-data-guide.html), CPU 시간 측정은 [Node.js process.cpuUsage 가이드](/development/blog/seo/2026/06/17/nodejs-process-cpuusage-cpu-time-monitoring-guide.html)와 함께 보면 좋습니다.
함수 단위 처리 시간을 측정해야 한다면 [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)도 이어서 참고할 수 있습니다.

## os.availableParallelism이 필요한 순간

### os.cpus().length를 기본 워커 수로 쓰지 않는다

`os.cpus()`는 논리 CPU 정보를 배열로 반환합니다.
서버의 하드웨어 정보를 확인할 때는 유용하지만, 워커 풀의 기본 병렬도를 정하는 값으로는 너무 직접적입니다.

```js
import os from 'node:os';

const cpuCount = os.cpus().length;
const parallelism = os.availableParallelism();

console.log({ cpuCount, parallelism });
```

`cpuCount`는 머신에 보이는 논리 CPU 수에 가깝고, `parallelism`은 현재 프로세스가 실제로 활용할 수 있는 병렬 실행 가능량에 더 가깝습니다.
로컬 노트북에서는 두 값이 같을 수 있지만, CI나 컨테이너에서는 다를 수 있습니다.
그래서 애플리케이션 기본값은 `os.availableParallelism()`에서 시작하는 편이 안전합니다.

```js
import os from 'node:os';

export function getDefaultWorkerCount() {
  return Math.max(1, os.availableParallelism() - 1);
}
```

여기서 `- 1`을 적용하는 이유는 메인 스레드와 운영체제, 다른 프로세스가 사용할 여유를 남기기 위해서입니다.
항상 최댓값까지 채우는 것보다 약간 덜 쓰는 기본값이 서비스 환경에서는 더 안정적입니다.
배치 전용 프로세스처럼 CPU를 적극적으로 써도 되는 경우에는 이 정책을 설정으로 바꿀 수 있게 열어 두면 됩니다.

### 컨테이너와 CI에서는 보이는 CPU보다 사용 가능량이 중요하다

개발자의 로컬 환경에서는 CPU 전체를 쓸 수 있는 경우가 많습니다.
하지만 배포 환경에서는 프로세스가 제한된 CPU quota 안에서 실행될 수 있습니다.
이때 하드웨어 기준으로 워커를 만들면 제한된 자원에 비해 너무 많은 작업을 동시에 실행할 수 있습니다.

```js
import os from 'node:os';

export function describeParallelism() {
  return {
    availableParallelism: os.availableParallelism(),
    visibleCpuCount: os.cpus().length
  };
}
```

이 정보를 시작 로그에 남기면 성능 이슈를 추적할 때 도움이 됩니다.
다만 로그에는 토큰, 데이터베이스 URL, 사용자 식별자 같은 민감정보를 함께 출력하지 않아야 합니다.
병렬도와 CPU 개수는 운영 진단에 필요한 값이지만, 환경변수 전체를 덤프하는 방식은 피해야 합니다.

## 워커 수 계산 패턴

### 설정값이 있으면 검증 후 우선한다

운영자는 특정 서비스의 성격을 알고 있습니다.
CPU 집약 작업이 핵심인 배치 프로세스라면 병렬도를 높이고 싶을 수 있고, API 서버와 같은 프로세스 안에서 보조 작업만 처리한다면 낮추고 싶을 수 있습니다.
따라서 기본값은 자동 계산하되, 명시 설정이 있으면 검증 후 우선하도록 만드는 것이 좋습니다.

```js
import os from 'node:os';

export function readWorkerCount(env = process.env) {
  const fallback = Math.max(1, os.availableParallelism() - 1);

  if (env.WORKER_COUNT == null || env.WORKER_COUNT === '') {
    return fallback;
  }

  const workerCount = Number(env.WORKER_COUNT);

  if (!Number.isInteger(workerCount) || workerCount < 1) {
    throw new Error(`Invalid WORKER_COUNT: ${env.WORKER_COUNT}`);
  }

  return Math.min(workerCount, os.availableParallelism());
}
```

이 함수는 세 가지 규칙을 코드에 고정합니다.
명시 설정이 없으면 안전한 기본값을 사용합니다.
설정이 있으면 양의 정수인지 검증합니다.
그리고 사용 가능한 병렬도를 넘지 않도록 상한을 둡니다.

상한을 무조건 걸지 여부는 서비스 성격에 따라 달라질 수 있습니다.
I/O 대기 시간이 긴 작업은 CPU 수보다 많은 동시 작업이 필요할 때도 있습니다.
하지만 워커 스레드처럼 CPU 작업을 병렬화하는 목적이라면 `availableParallelism`을 상한으로 삼는 편이 일반적으로 안전합니다.

### API 서버에서는 메인 이벤트 루프 여유를 남긴다

Node.js API 서버가 워커 스레드를 함께 쓸 때는 메인 이벤트 루프가 요청 수신, 라우팅, 응답 작성, 타이머 처리 등을 계속 맡습니다.
워커를 가능한 한 많이 띄우면 CPU를 전부 채우는 동안 이벤트 루프가 제때 반응하지 못할 수 있습니다.

```js
import os from 'node:os';

export function getApiWorkerPoolSize() {
  const available = os.availableParallelism();

  if (available <= 2) {
    return 1;
  }

  return Math.max(1, Math.floor(available * 0.75));
}
```

이 예시는 작은 환경에서는 워커 1개만 사용하고, CPU 여유가 있는 환경에서는 약 75%만 워커 풀에 할당합니다.
정답 숫자라기보다 정책을 명확히 표현한 코드에 가깝습니다.
중요한 것은 "왜 이 서비스는 이만큼만 병렬 실행하는가"가 코드와 설정에서 드러나는 것입니다.

## worker_threads에 적용하기

### 작은 워커 풀을 먼저 만든다

CPU 작업을 처리할 때 매 요청마다 새 워커를 만들면 생성 비용이 큽니다.
대부분의 서비스에서는 시작 시점에 워커 풀을 만들고, 작업을 큐에 넣어 재사용하는 방식이 더 낫습니다.

```js
import { Worker } from 'node:worker_threads';
import { readWorkerCount } from './worker-count.js';

export function createWorkerPool(workerUrl, env = process.env) {
  const size = readWorkerCount(env);
  const workers = [];

  for (let index = 0; index < size; index += 1) {
    workers.push(new Worker(workerUrl, {
      name: `cpu-worker-${index + 1}`
    }));
  }

  return workers;
}
```

실제 운영 코드에서는 작업 큐, idle 상태 추적, 오류 복구, 종료 처리까지 함께 설계해야 합니다.
그래도 출발점은 단순합니다.
워커 수를 `availableParallelism` 기반으로 계산하고, 설정으로 조정할 수 있게 만든 뒤, 서비스가 감당할 수 있는 만큼만 동시에 실행합니다.

### 작업 큐에는 백프레셔를 둔다

워커 수만 제한해도 CPU 동시 실행은 제어됩니다.
하지만 요청이 몰릴 때 큐가 끝없이 커지면 메모리 사용량이 늘고 응답 시간이 길어집니다.
그래서 워커 풀 앞에는 큐 길이 제한이나 거절 정책을 함께 두는 것이 좋습니다.

```js
export class JobQueue {
  #pending = [];
  #maxPending;

  constructor({ maxPending = 100 }) {
    this.#maxPending = maxPending;
  }

  push(job) {
    if (this.#pending.length >= this.#maxPending) {
      throw new Error('Worker queue is full');
    }

    this.#pending.push(job);
  }

  shift() {
    return this.#pending.shift();
  }
}
```

큐가 가득 찼을 때는 작업을 무한히 쌓기보다 명확히 실패시키는 편이 낫습니다.
API 서버라면 `503 Service Unavailable`과 재시도 안내를 반환할 수 있고, 배치 시스템이라면 나중에 다시 큐에 넣는 정책을 선택할 수 있습니다.
핵심은 병렬도 제한과 큐 제한을 함께 봐야 한다는 점입니다.

## 운영 점검 체크리스트

### 시작 로그에는 숫자만 남긴다

병렬도 문제는 배포 환경에서만 드러나는 경우가 많습니다.
그래서 시작 시점에 계산된 워커 수와 사용 가능한 병렬도를 남기면 좋습니다.

```js
import os from 'node:os';
import { readWorkerCount } from './worker-count.js';

export function logWorkerSettings(logger) {
  logger.info('worker pool configured', {
    workerCount: readWorkerCount(),
    availableParallelism: os.availableParallelism(),
    visibleCpuCount: os.cpus().length
  });
}
```

이 로그는 성능 분석에 필요한 정보를 제공합니다.
반면 `process.env` 전체, 작업 payload 원문, 사용자 입력 전체를 함께 출력하면 민감정보 노출 위험이 커집니다.
운영 로그는 필요한 숫자만 남기고, 문제 재현에 필요한 추가 정보는 별도 샘플링과 마스킹 정책을 거쳐야 합니다.

### 변경 전후를 측정한다

워커 수는 감으로만 정하면 실패하기 쉽습니다.
값을 바꿨다면 처리량, 평균 지연 시간, p95 또는 p99 지연 시간, CPU 사용률, 메모리 사용량을 함께 봐야 합니다.

```js
import { performance } from 'node:perf_hooks';

export async function measureJob(label, job) {
  const start = performance.now();

  try {
    return await job();
  } finally {
    const durationMs = performance.now() - start;
    console.log(`${label} finished in ${durationMs.toFixed(1)}ms`);
  }
}
```

작업 시간이 줄었더라도 API 응답 지연이 커졌다면 워커 수가 너무 높을 수 있습니다.
반대로 CPU 사용률이 낮고 큐가 계속 쌓인다면 병렬도를 조금 높이거나 작업 단위를 나누는 개선을 검토할 수 있습니다.
`os.availableParallelism()`은 좋은 기본값의 출발점이지, 측정을 대체하는 값은 아닙니다.

## FAQ

### os.availableParallelism은 os.cpus().length와 같은 값인가요?

항상 같지는 않습니다.
로컬 환경에서는 같게 보일 수 있지만, 컨테이너나 제한된 실행 환경에서는 현재 프로세스가 사용할 수 있는 병렬도가 더 작게 나올 수 있습니다.
워커 수 기본값은 `os.availableParallelism()`에서 시작하는 편이 더 안전합니다.

### 워커 수는 availableParallelism과 똑같이 맞추면 되나요?

배치 전용 프로세스라면 그렇게 시작할 수 있습니다.
하지만 API 서버처럼 메인 이벤트 루프의 반응성이 중요한 프로세스에서는 1개 이상 여유를 남기거나 비율 기반으로 줄이는 편이 좋습니다.

### I/O 작업도 availableParallelism으로 제한해야 하나요?

CPU 작업과 I/O 작업은 기준이 다릅니다.
파일, 네트워크, 데이터베이스 요청처럼 대기 시간이 큰 작업은 CPU 수보다 높은 동시성이 필요할 수 있습니다.
`os.availableParallelism()`은 주로 CPU를 쓰는 워커 스레드, 압축, 인코딩, 계산 작업의 기본 병렬도에 적합합니다.

## 마무리

`os.availableParallelism()`은 Node.js에서 워커 수를 정할 때 더 현실적인 출발점을 제공합니다.
`os.cpus().length`처럼 보이는 CPU 개수에 바로 기대기보다, 현재 프로세스가 사용할 수 있는 병렬도를 기준으로 기본값을 계산하는 편이 컨테이너와 CI 환경에서 안정적입니다.

실무에서는 자동 계산, 명시 설정, 상한, 큐 제한, 측정을 함께 묶어야 합니다.
기본값은 보수적으로 잡고, 운영 지표를 보며 조정하는 구조가 오래 유지됩니다.
워커 풀을 이미 쓰고 있다면 다음 점검에서 `os.cpus().length`를 직접 쓰는 코드가 있는지 확인하고, `os.availableParallelism()` 기반 정책으로 바꿀 수 있는지 살펴보세요.

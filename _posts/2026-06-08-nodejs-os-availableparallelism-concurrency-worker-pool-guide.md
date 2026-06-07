---
layout: post
title: "Node.js os.availableParallelism 가이드: 워커 수와 동시성 기본값 정하는 법"
date: 2026-06-08 08:00:00 +0900
lang: ko
translation_key: nodejs-os-availableparallelism-concurrency-worker-pool-guide
permalink: /development/blog/seo/2026/06/08/nodejs-os-availableparallelism-concurrency-worker-pool-guide.html
alternates:
  ko: /development/blog/seo/2026/06/08/nodejs-os-availableparallelism-concurrency-worker-pool-guide.html
  x_default: /development/blog/seo/2026/06/08/nodejs-os-availableparallelism-concurrency-worker-pool-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, availableparallelism, worker_threads, concurrency, performance, backend]
description: "Node.js os.availableParallelism()으로 컨테이너와 서버 환경에 맞는 동시성 기본값을 정하는 방법을 정리했습니다. os.cpus().length와의 차이, 워커 풀 크기, 작업 큐, 운영 설정 기준을 예제로 설명합니다."
---

Node.js에서 워커 수나 병렬 작업 개수를 정할 때 습관처럼 `os.cpus().length`를 쓰는 코드가 많습니다.
로컬 개발 환경에서는 그럭저럭 맞아 보이지만, 컨테이너, CI, 공유 서버, CPU 제한이 걸린 런타임에서는 실제로 사용할 수 있는 병렬 처리량과 어긋날 수 있습니다.

Node.js의 `os.availableParallelism()`은 현재 프로세스가 사용할 수 있는 병렬 처리 수준을 얻기 위한 API입니다.
CPU 바운드 작업을 `worker_threads`로 분리하거나, 이미지 변환·리포트 생성·배치 처리처럼 한 번에 몇 개까지 실행할지 정할 때 기본값으로 쓰기 좋습니다.

이 글에서는 `os.availableParallelism()`과 `os.cpus().length`의 차이, 워커 풀 크기를 정하는 기준, I/O 작업과 CPU 작업을 구분하는 방법, 운영 설정으로 덮어쓰는 패턴을 정리합니다.
CPU 작업을 워커로 분리하는 전체 구조가 먼저 필요하다면 [Node.js worker_threads 가이드](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)를 함께 참고하세요.

## os.availableParallelism이 필요한 이유

### H3. CPU 개수와 실제 사용 가능한 병렬성은 다를 수 있다

`os.cpus().length`는 시스템의 CPU 정보 배열 길이를 반환합니다.
하지만 애플리케이션이 그 CPU를 모두 자유롭게 쓸 수 있다는 뜻은 아닙니다.
컨테이너 CPU quota, 프로세스 affinity, 호스팅 환경의 제한, CI 실행기 정책에 따라 실제로 사용할 수 있는 병렬 처리 수준은 더 낮을 수 있습니다.

```js
import os from 'node:os';

console.log({
  cpuCount: os.cpus().length,
  availableParallelism: os.availableParallelism()
});
```

`availableParallelism()`은 이런 제약을 고려한 값을 얻기 위한 함수입니다.
따라서 "이 머신의 물리적 CPU 정보"가 아니라 "이 프로세스가 기본 병렬 처리 값으로 삼을 숫자"가 필요할 때 더 알맞습니다.

### H3. 잘못 큰 기본값은 장애를 키운다

동시성 기본값이 너무 크면 처리량이 늘기보다 대기열과 메모리 사용량이 먼저 커질 수 있습니다.
특히 CPU 바운드 작업에서는 워커를 많이 만든다고 선형으로 빨라지지 않습니다.
오히려 컨텍스트 스위칭, GC 부담, 메모리 압박, 하위 시스템 병목이 함께 늘어납니다.

예를 들어 2 vCPU 제한이 있는 컨테이너에서 `os.cpus().length`가 호스트 기준 16으로 보이고, 그 값을 그대로 워커 수로 쓰면 작은 요청 급증에도 프로세스가 쉽게 흔들릴 수 있습니다.
동시성 제한은 성능 최적화이면서 동시에 과부하 보호 장치입니다.
과부하 제어 관점은 [Node.js admission control 가이드](/development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html)와도 연결됩니다.

## 기본 사용법

### H3. 워커 수 기본값 계산하기

가장 단순한 패턴은 `availableParallelism()`을 기준으로 워커 수를 정하되, 최소·최대 범위를 함께 두는 것입니다.

```js
import os from 'node:os';

export function getDefaultWorkerCount() {
  const parallelism = os.availableParallelism();
  return Math.max(1, Math.min(parallelism, 8));
}
```

최대값을 두는 이유는 운영 환경이 커졌다고 워커 수를 무한히 늘리는 것이 항상 좋은 전략은 아니기 때문입니다.
각 워커가 메모리를 따로 쓰고, 작업마다 입력 데이터와 중간 결과를 들고 있을 수 있습니다.
대량 처리 시스템에서는 CPU보다 메모리, DB 연결, 파일 디스크립터, 외부 API 제한이 먼저 병목이 될 수 있습니다.

### H3. 환경 변수로 운영자가 덮어쓰게 한다

기본값은 코드가 정하되, 운영에서는 설정으로 조정할 수 있어야 합니다.
자동 계산값이 항상 최적은 아니기 때문입니다.

```js
import os from 'node:os';

function parsePositiveInteger(value) {
  if (!value) return null;

  const number = Number(value);
  if (!Number.isInteger(number) || number < 1) {
    throw new Error('WORKER_COUNT must be a positive integer');
  }

  return number;
}

export function resolveWorkerCount() {
  const configured = parsePositiveInteger(process.env.WORKER_COUNT);
  if (configured) return configured;

  const defaultCount = os.availableParallelism();
  return Math.max(1, Math.min(defaultCount, 8));
}
```

이 패턴은 세 가지 장점이 있습니다.

- 로컬과 운영에서 같은 계산 규칙을 공유한다.
- 장애 대응 중 코드 배포 없이 워커 수를 줄일 수 있다.
- 성능 테스트 결과에 맞춰 서비스별 값을 조정할 수 있다.

환경 변수 관리는 [Node.js loadEnvFile 가이드](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)처럼 부팅 시점에 검증해 두면 운영 실수를 줄이기 쉽습니다.

## worker_threads에 적용하는 패턴

### H3. 워커 풀은 사용 가능한 병렬성보다 작게 시작한다

CPU 바운드 작업은 메인 이벤트 루프에서 오래 돌리면 요청 처리와 타이머, I/O 콜백까지 밀릴 수 있습니다.
이럴 때 `worker_threads`로 분리하면 메인 스레드의 응답성을 지킬 수 있습니다.
다만 워커 수는 보수적으로 시작하는 편이 안전합니다.

```js
import { Worker } from 'node:worker_threads';
import { resolveWorkerCount } from './config/concurrency.js';

export function createWorkerPool() {
  const workerCount = resolveWorkerCount();

  return Array.from({ length: workerCount }, () => {
    return new Worker(new URL('./worker.js', import.meta.url));
  });
}
```

처음부터 `availableParallelism()` 전체를 다 쓰기보다, 서비스 특성에 맞춰 1개를 남겨 두는 전략도 자주 씁니다.
메인 스레드, 로그 처리, 네트워크 I/O, 런타임 내부 작업이 완전히 공짜는 아니기 때문입니다.

```js
import os from 'node:os';

export function getCpuWorkerCount() {
  const usable = os.availableParallelism();
  return Math.max(1, Math.min(usable - 1, 6));
}
```

작업 하나가 메모리를 많이 쓰는 경우에는 CPU 수보다 메모리 한도가 더 중요한 기준이 됩니다.
남은 메모리와 배치 크기를 함께 보려면 [Node.js process.availableMemory 가이드](/development/blog/seo/2026/05/21/nodejs-process-availablememory-memory-pressure-guide.html)를 같이 적용할 수 있습니다.

### H3. 워커 수와 큐 길이는 별도로 제한한다

워커 수만 제한해도 동시에 실행되는 CPU 작업 수는 줄일 수 있습니다.
하지만 들어오는 작업을 무제한 큐에 쌓으면 결국 메모리와 지연 시간이 터집니다.
워커 풀에는 큐 길이 제한도 함께 있어야 합니다.

```js
const MAX_QUEUE_SIZE = 200;

export class WorkerQueue {
  #queue = [];

  enqueue(job) {
    if (this.#queue.length >= MAX_QUEUE_SIZE) {
      const error = new Error('worker queue is full');
      error.code = 'QUEUE_FULL';
      throw error;
    }

    this.#queue.push(job);
  }

  dequeue() {
    return this.#queue.shift();
  }
}
```

이 코드는 단순한 예시지만 원칙은 중요합니다.
동시 실행 수와 대기열 크기는 서로 다른 보호 장치입니다.
동시 실행 수는 CPU 과부하를 줄이고, 큐 길이 제한은 메모리 증가와 응답 지연을 제한합니다.
큐가 필요한 서비스라면 [Node.js bounded queue 가이드](/development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html)처럼 거절 기준과 재시도 정책까지 함께 정하는 편이 좋습니다.

## I/O 작업에는 그대로 적용하면 안 된다

### H3. CPU 바운드와 I/O 바운드는 기준이 다르다

`availableParallelism()`은 CPU 병렬 처리 기본값을 잡는 데 유용합니다.
하지만 모든 동시성 제한을 이 값 하나로 통일하면 안 됩니다.
외부 API 호출, DB 쿼리, 파일 업로드, 메시지 소비처럼 I/O 대기가 큰 작업은 CPU 수보다 다른 제한이 더 중요합니다.

예를 들어 외부 API 호출 동시성은 다음 요소를 봐야 합니다.

- 상대 API의 rate limit
- 네트워크 timeout과 retry 정책
- 커넥션 풀 크기
- 요청당 메모리 사용량
- 실패 시 재시도 폭증 가능성

반대로 이미지 리사이즈, 압축, 암호화, 대규모 JSON 파싱처럼 CPU를 오래 쓰는 작업은 `availableParallelism()` 기반 제한이 잘 맞습니다.
하나의 서비스 안에서도 작업 종류별로 제한값을 따로 둬야 합니다.

### H3. 작업별 동시성 설정 이름을 분리한다

운영 설정을 하나로 뭉치면 나중에 튜닝이 어려워집니다.
CPU 작업과 I/O 작업은 환경 변수 이름부터 분리하는 편이 명확합니다.

```js
export const concurrency = {
  imageWorkers: resolveInteger('IMAGE_WORKER_COUNT', getCpuWorkerCount()),
  reportWorkers: resolveInteger('REPORT_WORKER_COUNT', getCpuWorkerCount()),
  apiRequests: resolveInteger('EXTERNAL_API_CONCURRENCY', 20),
  dbJobs: resolveInteger('DB_JOB_CONCURRENCY', 10)
};
```

기본값의 출처도 다르게 둡니다.
CPU 작업은 `availableParallelism()`에서 출발하고, 외부 API나 DB 작업은 계약된 제한과 실제 관측 지표에서 출발합니다.
이렇게 나누면 장애 대응 중 특정 병목만 줄일 수 있습니다.

## 운영에서 확인할 지표

### H3. CPU 사용률만 보지 않는다

워커 수를 조정할 때 CPU 사용률 하나만 보면 위험합니다.
CPU가 낮아도 큐 대기 시간이 길 수 있고, CPU가 높아도 전체 처리량이 안정적일 수 있습니다.
다음 지표를 같이 봐야 합니다.

- 작업 처리 시간 p50, p95, p99
- 큐 대기 시간과 큐 길이
- 이벤트 루프 지연
- RSS와 heap 사용량
- 워커 재시작 횟수
- 작업 실패율과 timeout 비율

특히 메인 스레드의 이벤트 루프 지연은 워커를 도입한 뒤에도 계속 확인해야 합니다.
작업 제출, 결과 병합, 직렬화 비용이 메인 스레드에 남기 때문입니다.
관련 지표는 [Node.js 이벤트 루프 지연 모니터링 가이드](/development/blog/seo/2026/04/13/nodejs-event-loop-lag-monitoring-guide.html)에서 더 자세히 다룹니다.

### H3. 성능 테스트로 상한을 찾는다

`availableParallelism()`은 출발점이지 최종 정답이 아닙니다.
서비스별로 입력 크기, 라이브러리 구현, 메모리 사용량, 배포 환경이 다르기 때문에 실제 상한은 측정해야 합니다.

실험은 한 번에 한 변수만 바꾸는 편이 좋습니다.

1. 워커 수를 1, 2, 4, 6처럼 단계적으로 올린다.
2. 같은 입력 데이터와 같은 부하 조건을 사용한다.
3. 처리량뿐 아니라 p95 지연, 큐 대기, 메모리, 실패율을 함께 기록한다.
4. 처리량 증가가 둔해지거나 실패율이 늘어나는 지점을 찾는다.
5. 그보다 한 단계 낮은 값을 운영 기본값으로 둔다.

여기서 얻은 값은 환경 변수 기본값이나 배포별 설정으로 남겨야 합니다.
코드에 "우리 서버는 8코어니까 8"처럼 박아 두면 다음 인프라 변경 때 다시 위험해집니다.

## 실무 체크리스트

### H3. 적용 전 확인할 것

- CPU 바운드 작업인지 I/O 바운드 작업인지 구분한다.
- 워커 수 기본값은 `os.availableParallelism()`에서 출발한다.
- 최소값과 최대값을 함께 둔다.
- 운영 환경 변수로 덮어쓸 수 있게 한다.
- 워커 수와 큐 길이 제한을 별도로 둔다.
- 메모리 사용량이 큰 작업은 CPU 수보다 메모리 한도를 먼저 본다.
- 이벤트 루프 지연, 큐 대기 시간, 실패율을 함께 관측한다.

### H3. 피해야 할 패턴

아래 패턴은 운영에서 문제를 만들기 쉽습니다.

- `os.cpus().length`를 워커 수로 그대로 사용한다.
- 컨테이너 CPU 제한을 고려하지 않는다.
- CPU 작업과 I/O 작업의 동시성 기본값을 하나로 통일한다.
- 워커 수만 제한하고 큐는 무제한으로 둔다.
- 성능 테스트 없이 큰 서버에서는 무조건 워커를 더 늘린다.
- 장애 대응 중 조정할 수 없는 하드코딩 값을 쓴다.

## FAQ

### H3. os.availableParallelism은 os.cpus().length를 완전히 대체하나요?

목적이 다릅니다.
CPU 모델, 속도, 상세 정보를 보고 싶다면 `os.cpus()`가 필요할 수 있습니다.
하지만 워커 수나 병렬 작업 기본값처럼 "현재 프로세스가 사용할 병렬성"을 정하려는 목적이라면 `os.availableParallelism()`을 우선 검토하는 편이 좋습니다.

### H3. availableParallelism 값만큼 워커를 만들면 항상 빠른가요?

아닙니다.
작업이 CPU 바운드인지, 워커가 쓰는 메모리가 얼마나 큰지, 결과를 메인 스레드로 되돌릴 때 직렬화 비용이 얼마나 드는지에 따라 달라집니다.
대부분의 서비스에서는 계산값을 출발점으로 삼고, 실제 부하 테스트로 운영값을 줄이거나 고정합니다.

### H3. 외부 API 호출 동시성도 이 값으로 정하면 되나요?

보통은 아닙니다.
외부 API 호출은 CPU보다 상대 서비스의 rate limit, timeout, 커넥션 풀, 재시도 정책이 더 중요합니다.
`availableParallelism()`은 CPU를 오래 쓰는 작업의 기본값에 적합하고, I/O 동시성은 별도 지표와 계약 기준으로 정하는 편이 안전합니다.

## 마무리

`os.availableParallelism()`은 작은 API이지만 Node.js 서비스의 동시성 기본값을 더 현실적으로 잡게 해 줍니다.
특히 컨테이너와 CI처럼 프로세스가 사용할 수 있는 CPU가 제한되는 환경에서는 `os.cpus().length`보다 운영 의도에 가까운 출발점이 됩니다.

다만 이 값이 자동 튜닝을 대신하지는 않습니다.
CPU 바운드 작업에 우선 적용하고, 최소·최대 범위와 환경 변수 override를 두고, 큐 길이와 메모리 지표까지 함께 보는 것이 안전합니다.
워커 수는 성능 숫자이면서 장애 반경을 정하는 운영 설정입니다.

## 함께 보면 좋은 글

- [Node.js worker_threads 가이드: CPU 바운드 작업으로 이벤트 루프 막지 않는 법](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)
- [Node.js process.availableMemory 가이드: 메모리 압박에 맞춰 작업량 조절하기](/development/blog/seo/2026/05/21/nodejs-process-availablememory-memory-pressure-guide.html)
- [Node.js bounded queue 가이드: 대기열 길이 제한으로 과부하 막기](/development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html)

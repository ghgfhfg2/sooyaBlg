---
layout: post
title: "Node.js Worker Threads 실전 가이드: CPU 바운드 병목 줄이고 API 지연 안정화하기"
date: 2026-03-18 20:00:00 +0900
lang: ko
translation_key: nodejs-worker-threads-cpu-bound-performance-guide
permalink: /development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html
alternates:
  ko: /development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html
  x_default: /development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, worker-threads, cpu-bound, performance, backend, event-loop]
description: "Node.js API가 CPU 바운드 작업 때문에 느려질 때 Worker Threads로 이벤트 루프 병목을 줄이는 방법을 정리했습니다. 적용 기준, 구현 예시, 운영 체크리스트까지 실무 관점으로 설명합니다."
---

Node.js 서비스에서 응답 지연이 갑자기 늘어날 때, 원인이 DB나 네트워크가 아니라 **CPU 바운드 작업**인 경우가 많습니다.
이미지 리사이즈, 대용량 JSON 파싱, 암호화/압축 같은 작업이 메인 스레드를 오래 점유하면 이벤트 루프가 막히고 전체 API가 느려집니다.
이 글에서는 Worker Threads를 기준으로 병목을 분리하고, 지연을 안정화하는 실전 패턴을 정리합니다.

## Worker Threads가 필요한 상황부터 구분하기

### H3. I/O 바운드와 CPU 바운드는 대응 방식이 다르다

Node.js는 I/O 처리에 강하지만, CPU 계산이 길어지면 단일 이벤트 루프 특성상 전체 처리 지연으로 번집니다.

- I/O 바운드: DB/외부 API 대기 시간이 길다 → 커넥션/타임아웃/재시도 최적화
- CPU 바운드: 계산 자체가 오래 걸린다 → 메인 스레드 분리(Worker Threads)

핵심은 "느리다"를 하나로 보지 말고, 어디에서 시간이 소모되는지 먼저 분해하는 것입니다.

### H3. Worker Threads 도입 신호

아래 신호가 반복되면 Worker Threads 검토 우선순위를 올리는 편이 안전합니다.

1. p95/p99 응답 지연이 트래픽 증가 없이도 튄다
2. CPU 사용률은 높은데 DB/네트워크 지표는 정상이다
3. 특정 엔드포인트 호출 시 다른 경량 API까지 같이 느려진다
4. 이벤트 루프 지연(event loop lag) 경고가 반복된다

## Node.js Worker Threads 기본 구현 패턴

### H3. 메인 스레드는 요청 처리, 워커는 계산 전담

아래처럼 메인 스레드는 HTTP 처리에 집중하고, 무거운 계산은 워커로 위임합니다.

```ts
// main.ts
import { Worker } from 'node:worker_threads';

export function runCpuTask(payload: unknown): Promise<unknown> {
  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL('./worker-task.js', import.meta.url), {
      workerData: payload,
    });

    worker.once('message', resolve);
    worker.once('error', reject);
    worker.once('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`));
    });
  });
}
```

```ts
// worker-task.ts
import { parentPort, workerData } from 'node:worker_threads';

function heavyCompute(input: any) {
  // CPU 집약 로직 (예: 대용량 변환/해시/압축 전처리)
  let acc = 0;
  for (let i = 0; i < 50_000_000; i++) acc += i % 7;
  return { ok: true, inputSize: JSON.stringify(input).length, acc };
}

const result = heavyCompute(workerData);
parentPort?.postMessage(result);
```

초기 단계에서는 "엔드포인트 하나"만 분리해도 체감 개선이 큽니다.

### H3. 워커 풀(Pool) 없이 남발하면 오히려 느려질 수 있다

요청마다 워커를 새로 만들면 생성/종료 오버헤드가 누적됩니다.
실무에서는 보통 아래 기준으로 시작합니다.

- 고정 크기 워커 풀(코어 수 기준 50~100%)
- 큐 길이 상한 설정(무한 대기열 금지)
- 작업별 타임아웃과 취소 처리

즉, Worker Threads는 "분리" 자체보다 **제어 가능한 실행 모델**로 운용하는 것이 핵심입니다.

## 운영에서 자주 겪는 실수와 방지책

### H3. 실수 1) 큰 객체 복사 비용을 간과

메인 ↔ 워커 간 메시지는 직렬화/복사 비용이 발생합니다.
대용량 데이터는 `Transferable`(예: `ArrayBuffer`)을 고려해 복사 비용을 줄여야 합니다.

### H3. 실수 2) 장애 전파 경로가 불명확

워커 실패 시 메인 스레드에서 에러 맥락이 사라지면 원인 파악이 어려워집니다.

대응:
- 작업 ID를 부여해 요청-워커 로그를 연결
- 에러 코드/원인/입력 크기를 구조화 로그로 기록
- 타임아웃과 강제 종료 기준을 runbook에 명시

### H3. 실수 3) "CPU 분리 = 무조건 성능 향상"으로 오해

CPU 바운드가 아닌 경로까지 워커로 보내면 통신 오버헤드만 늘어날 수 있습니다.
도입 전후를 아래 지표로 비교해야 합니다.

- API p95/p99 지연
- event loop lag
- 워커 큐 대기 시간
- 프로세스별 CPU 사용률

## 배포 전 체크리스트

### H3. 설계/성능

- CPU 집약 구간이 프로파일링으로 확인됐는가
- 워커 풀 크기와 큐 상한이 정의됐는가
- 작업 타임아웃/재시도/실패 응답 정책이 있는가

### H3. 보안/안전

- 예시 코드/로그에 API 키, 토큰, 개인정보가 없는가
- 사용자 입력을 워커로 전달하기 전 검증하는가
- 과장된 성능 주장 없이 측정값 기반으로 서술했는가

## 요약

Node.js에서 CPU 바운드 병목은 "서버 스펙 부족"보다 **실행 모델 부적합**에서 시작되는 경우가 많습니다.
Worker Threads를 선택적으로 도입하고, 워커 풀/큐/타임아웃/관측 지표를 함께 설계하면
메인 이벤트 루프를 보호하면서 API 지연을 안정적으로 낮출 수 있습니다.

## 내부 링크

- [/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)
- [/development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html](/development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html)
- [/development/blog/seo/2026/03/17/nodejs-idempotent-consumer-kafka-duplicate-message-handling-guide.html](/development/blog/seo/2026/03/17/nodejs-idempotent-consumer-kafka-duplicate-message-handling-guide.html)

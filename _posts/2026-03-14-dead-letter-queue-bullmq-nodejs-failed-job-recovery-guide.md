---
layout: post
title: "Node.js BullMQ Dead Letter Queue 설계: 실패 작업 유실 없이 복구하는 실무 가이드"
date: 2026-03-14 20:00:00 +0900
lang: ko
translation_key: dead-letter-queue-bullmq-nodejs-failed-job-recovery-guide
permalink: /development/blog/seo/2026/03/14/dead-letter-queue-bullmq-nodejs-failed-job-recovery-guide.html
alternates:
  ko: /development/blog/seo/2026/03/14/dead-letter-queue-bullmq-nodejs-failed-job-recovery-guide.html
  x_default: /development/blog/seo/2026/03/14/dead-letter-queue-bullmq-nodejs-failed-job-recovery-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, bullmq, redis, dead-letter-queue, reliability, backend]
description: "Node.js BullMQ에서 Dead Letter Queue(DLQ)를 설계해 실패 작업을 안전하게 보관하고 재처리하는 방법을 정리합니다. 재시도 정책, 백오프, 알림, 운영 체크리스트까지 실무 기준으로 확인하세요."
---

큐 기반 비동기 처리를 운영하다 보면 결국 마주치는 문제가 있습니다.
바로 **계속 실패하는 작업이 조용히 사라지거나, 무한 재시도로 시스템을 압박하는 상황**입니다.
이 글에서는 Node.js + BullMQ 환경에서 Dead Letter Queue(DLQ)를 도입해
실패 작업을 유실 없이 보관하고, 안전하게 재처리하는 구조를 정리합니다.

## 왜 BullMQ에 Dead Letter Queue가 필요한가

### 실패를 "없애는 것"보다 "관리 가능한 상태로 격리"하는 게 먼저다

재시도만 늘리면 일시적 오류에는 도움이 되지만,
데이터 포맷 불일치나 비즈니스 규칙 충돌 같은 **영구 실패(permanent failure)** 에서는 오히려 장애를 키웁니다.
DLQ는 이런 작업을 본 큐에서 분리해 서비스 전체 지연을 막아줍니다.

### 온콜 대응에서 "무엇이 실패했는지"가 즉시 보여야 한다

운영에서 중요한 건 실패 건수보다 **실패 원인 분류와 복구 가능성**입니다.
DLQ에 실패 컨텍스트를 함께 저장하면,
온콜 담당자가 로그를 뒤지지 않고도 재처리 전략을 빠르게 결정할 수 있습니다.

## BullMQ DLQ 아키텍처 기본 설계

### 1) 재시도 정책부터 분리한다 (일시적/영구 실패)

- 일시적 실패: 네트워크 타임아웃, 외부 API 5xx
- 영구 실패: 유효성 검증 실패, 존재하지 않는 참조 데이터

일시적 실패만 제한 횟수로 재시도하고,
한계 초과 시 DLQ로 보내는 식으로 정책을 분리해야 합니다.

### 2) 실패 메타데이터를 표준화한다

DLQ payload에는 최소한 아래 항목을 남기세요.

- `jobName`, `jobId`, `attemptsMade`
- `failedAt`, `errorCode`, `errorMessage`
- 재처리 판단용 `context` (민감정보 마스킹 필수)

### 3) DLQ도 "운영 대상"으로 본다

DLQ를 쌓기만 하면 또 다른 데이터 무덤이 됩니다.
주기적인 triage(분류)와 재처리 워크플로를 정의해
"쌓임 → 분류 → 수정 → 재처리 → 회고" 루프를 운영해야 합니다.

## Node.js BullMQ 구현 예시

### 큐/워커 설정과 재시도 + 백오프

```js
import { Queue, Worker } from 'bullmq';
import IORedis from 'ioredis';

const connection = new IORedis(process.env.REDIS_URL);

const mainQueue = new Queue('email-send', { connection });
const dlqQueue = new Queue('email-send-dlq', { connection });

const worker = new Worker(
  'email-send',
  async (job) => {
    // 실제 작업 로직
    await sendEmail(job.data);
  },
  {
    connection,
    // 일시적 오류를 위한 재시도 정책
    attempts: 5,
    backoff: {
      type: 'exponential',
      delay: 2000,
    },
    removeOnComplete: true,
    removeOnFail: false,
  }
);
```

### 실패 작업을 DLQ로 이동시키기

```js
worker.on('failed', async (job, err) => {
  if (!job) return;

  // 재시도 한계 초과 시 DLQ 적재
  if (job.attemptsMade >= (job.opts.attempts ?? 1)) {
    await dlqQueue.add('failed-email-job', {
      originalQueue: 'email-send',
      jobId: job.id,
      jobName: job.name,
      data: sanitize(job.data),
      attemptsMade: job.attemptsMade,
      failedAt: new Date().toISOString(),
      errorMessage: err.message,
    });
  }
});

function sanitize(data) {
  // 예시: 이메일/전화번호 등 민감정보 마스킹
  return {
    ...data,
    email: data?.email ? '***masked***' : undefined,
  };
}
```

핵심은 실패 시점에 원본 작업 컨텍스트를 최소 단위로 보존하되,
**개인정보/비밀값은 저장 전에 마스킹**하는 것입니다.

## 운영에서 자주 놓치는 포인트

### DLQ 알림 임계치가 없으면 장애를 늦게 인지한다

DLQ 건수는 "이미 사용자 영향이 시작됐을 가능성"을 뜻합니다.
예를 들어 5분 내 DLQ 20건 이상이면 즉시 알림하도록 기준을 잡으세요.

### 재처리 스크립트에 멱등성이 없으면 2차 사고가 난다

같은 작업을 재실행해도 부작용이 없도록
idempotency key 또는 처리 이력 검증 로직을 넣어야 합니다.

### 실패 원인 taxonomy가 없으면 회고가 쌓이지 않는다

`VALIDATION_ERROR`, `UPSTREAM_TIMEOUT`, `RATE_LIMITED`처럼
오류 코드를 분류해 월간 회고에 반영해야 개선 우선순위가 보입니다.

## 배포 전/후 체크리스트

### 배포 전

- 재시도 횟수/백오프가 외부 API SLA와 맞는지 확인
- DLQ payload에서 민감정보 마스킹 확인
- DLQ 건수 알림 규칙(임계치, 채널) 설정

### 배포 후 1주

- DLQ Top 오류 코드 3개 점검
- 재처리 성공률/재실패율 추적
- 영구 실패 비율이 높은 작업 스키마 개선

## 요약

BullMQ Dead Letter Queue는 "실패를 숨기는 장치"가 아니라
**실패를 통제 가능한 운영 이벤트로 바꾸는 장치**입니다.
재시도 정책, DLQ 스키마, 알림, 재처리 기준을 함께 설계하면
비동기 시스템의 장애 복구 속도와 신뢰도를 동시에 끌어올릴 수 있습니다.

## 내부 링크

- [/development/blog/seo/2026/03/13/feature-flag-rollout-strategy-nodejs-safe-release-guide.html](/development/blog/seo/2026/03/13/feature-flag-rollout-strategy-nodejs-safe-release-guide.html)
- [/development/blog/seo/2026/03/12/rate-limiting-nodejs-express-redis-abuse-protection-guide.html](/development/blog/seo/2026/03/12/rate-limiting-nodejs-express-redis-abuse-protection-guide.html)
- [/development/blog/seo/2026/03/08/idempotency-key-api-retry-consistency-guide.html](/development/blog/seo/2026/03/08/idempotency-key-api-retry-consistency-guide.html)

---
layout: post
title: "Node.js Idempotent Consumer 실전 가이드: Kafka 중복 메시지에도 데이터 정합성 지키기"
date: 2026-03-17 20:00:00 +0900
lang: ko
translation_key: nodejs-idempotent-consumer-kafka-duplicate-message-handling-guide
permalink: /development/blog/seo/2026/03/17/nodejs-idempotent-consumer-kafka-duplicate-message-handling-guide.html
alternates:
  ko: /development/blog/seo/2026/03/17/nodejs-idempotent-consumer-kafka-duplicate-message-handling-guide.html
  x_default: /development/blog/seo/2026/03/17/nodejs-idempotent-consumer-kafka-duplicate-message-handling-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, kafka, message-queue, idempotency, backend, data-consistency]
description: "Node.js 이벤트 기반 아키텍처에서 Kafka 중복 메시지를 안전하게 처리하는 Idempotent Consumer 패턴을 정리합니다. 중복 방지 키 설계, DB 트랜잭션 처리, 장애 복구 체크리스트까지 실무 기준으로 설명합니다."
---

이벤트 기반 시스템에서는 메시지가 **한 번만 전달될 것**이라고 가정하면 거의 반드시 사고가 납니다.
브로커 재시도, 컨슈머 재시작, 네트워크 타임아웃 때문에 중복 전달은 정상 동작에 가깝기 때문입니다.
이 글은 Node.js 백엔드에서 Kafka(또는 다른 MQ) 소비 로직을
`중복 허용 + 결과 멱등` 구조로 만드는 방법을 실무 기준으로 정리합니다.

## 왜 Idempotent Consumer가 필요한가

### at-least-once 전달 모델의 현실

Kafka, RabbitMQ, SQS 같은 시스템은 보통 at-least-once 전달을 기본으로 둡니다.
즉, "유실은 줄이되 중복 가능성은 열어둔다"는 설계입니다.
따라서 중복 메시지를 애플리케이션에서 안전하게 흡수해야 합니다.

### 중복 처리 실패가 만드는 대표 장애

- 주문/결제 이벤트 중복 반영으로 금액 불일치
- 포인트/재고가 두 번 차감되는 데이터 손상
- 외부 API 호출 중복으로 비용 증가 및 CS 이슈

핵심은 "메시지 중복" 자체가 아니라, **중복이 비즈니스 결과를 바꾸는 구조**입니다.

## Node.js Idempotent Consumer 설계 원칙

### H3. 멱등 키(idempotency key)부터 명확히 정한다

메시지마다 비즈니스 관점의 유일 키를 가져야 합니다.
예: `orderId + eventType + eventVersion`

브로커 offset이나 timestamp를 키로 쓰면 재처리/재발행 시 의미가 깨질 수 있습니다.
"이 작업은 이미 처리했는가?"를 판별할 수 있는 키가 필요합니다.

### H3. DB에 "처리 이력"을 원자적으로 남긴다

가장 단순하고 강력한 방식은
`processed_messages` 테이블(또는 Redis set + 영속 백업)에
멱등 키를 unique 제약으로 저장하는 방법입니다.

```sql
create table processed_messages (
  id bigint generated always as identity primary key,
  idempotency_key varchar(200) not null,
  consumer_name varchar(100) not null,
  processed_at timestamptz not null default now(),
  unique (idempotency_key, consumer_name)
);
```

비즈니스 업데이트와 처리 이력 기록을 **같은 트랜잭션**으로 묶어야
부분 성공(데이터만 반영, 이력 미기록) 같은 꼬임을 줄일 수 있습니다.

## 구현 예시: PostgreSQL + Node.js

### H3. 트랜잭션 안에서 "중복이면 무시" 처리

```ts
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export async function handleOrderPaid(event: {
  eventId: string;
  orderId: string;
  amount: number;
}) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const key = `order-paid:${event.eventId}`;

    const inserted = await client.query(
      `
      insert into processed_messages (idempotency_key, consumer_name)
      values ($1, $2)
      on conflict (idempotency_key, consumer_name) do nothing
      returning id
      `,
      [key, 'billing-consumer']
    );

    // 이미 처리된 메시지라면 안전하게 종료
    if (inserted.rowCount === 0) {
      await client.query('ROLLBACK');
      return { skipped: true };
    }

    await client.query(
      `update orders
       set paid_amount = paid_amount + $1,
           status = 'PAID'
       where id = $2`,
      [event.amount, event.orderId]
    );

    await client.query('COMMIT');
    return { skipped: false };
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

이 패턴의 장점은 단순합니다.
- 첫 처리: 이력 insert 성공 + 비즈니스 반영
- 중복 처리: 이력 insert 충돌 → 비즈니스 로직 미실행

## 운영에서 자주 놓치는 포인트

### H3. "외부 API 호출"은 별도 멱등 장치가 필요하다

DB는 트랜잭션으로 보호해도,
외부 결제/알림 API는 네트워크 타임아웃 시 결과를 확신하기 어렵습니다.
가능하면 외부 API에도 idempotency key를 전달하고,
없다면 호출 결과를 저장해 재시도 정책을 분리해야 합니다.

### H3. DLQ와 재처리 정책을 같이 설계한다

멱등 소비자는 중복에는 강하지만,
스키마 불일치/필수 데이터 누락 같은 "영구 실패"까지 해결해주진 않습니다.
실패 메시지는 DLQ(Dead Letter Queue)로 보내고
운영자가 재처리할 수 있는 runbook을 준비해야 합니다.

### H3. 관측 지표를 미리 붙인다

최소한 아래 지표는 기본으로 수집하는 편이 좋습니다.

- duplicate_detected_count (중복 감지 건수)
- consumer_lag
- processing_latency_p95
- dlq_enqueue_count

중복 감지율이 갑자기 늘면,
프로듀서 재시도 폭증이나 브로커/네트워크 문제의 신호일 수 있습니다.

## 실무 체크리스트

### H3. 배포 전

- 멱등 키 규칙이 팀 내에서 문서화되어 있는가
- 처리 이력 저장소에 unique 제약이 있는가
- 비즈니스 반영과 이력 기록이 같은 트랜잭션인가

### H3. 배포 후

- duplicate_detected_count 알림 임계치를 설정했는가
- DLQ 재처리 절차(담당자/명령어/검증 항목)가 있는가
- 장애 회고 때 중복 처리 누락 케이스를 템플릿화했는가

## 요약

이벤트 기반 아키텍처에서 "중복 메시지"는 예외가 아니라 기본값입니다.
Node.js 컨슈머를 안전하게 운영하려면
**멱등 키 설계 + 원자적 처리 이력 기록 + DLQ/관측 지표**를 세트로 가져가야 합니다.
이 3가지를 표준화하면 재시도는 더 공격적으로 가져가면서도,
데이터 정합성은 안정적으로 지킬 수 있습니다.

## 내부 링크

- [/development/blog/seo/2026/03/15/transactional-outbox-pattern-nodejs-event-driven-consistency-guide.html](/development/blog/seo/2026/03/15/transactional-outbox-pattern-nodejs-event-driven-consistency-guide.html)
- [/development/blog/seo/2026/03/14/dead-letter-queue-bullmq-nodejs-failed-job-recovery-guide.html](/development/blog/seo/2026/03/14/dead-letter-queue-bullmq-nodejs-failed-job-recovery-guide.html)
- [/development/blog/seo/2026/03/08/idempotency-key-api-retry-consistency-guide.html](/development/blog/seo/2026/03/08/idempotency-key-api-retry-consistency-guide.html)

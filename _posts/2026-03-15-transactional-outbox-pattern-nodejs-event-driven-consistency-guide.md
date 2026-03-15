---
layout: post
title: "Node.js 트랜잭셔널 아웃박스 패턴: 이벤트 유실 없이 데이터 정합성 지키는 실무 가이드"
date: 2026-03-15 20:00:00 +0900
lang: ko
translation_key: transactional-outbox-pattern-nodejs-event-driven-consistency-guide
permalink: /development/blog/seo/2026/03/15/transactional-outbox-pattern-nodejs-event-driven-consistency-guide.html
alternates:
  ko: /development/blog/seo/2026/03/15/transactional-outbox-pattern-nodejs-event-driven-consistency-guide.html
  x_default: /development/blog/seo/2026/03/15/transactional-outbox-pattern-nodejs-event-driven-consistency-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, outbox-pattern, event-driven, consistency, reliability, backend]
description: "Node.js 백엔드에서 트랜잭셔널 아웃박스 패턴을 적용해 DB 저장과 이벤트 발행 사이의 유실 문제를 해결하는 방법을 설명합니다. 스키마 설계, 폴링 발행기, 멱등 처리, 운영 체크리스트까지 실무 기준으로 정리했습니다."
---

주문/결제 같은 핵심 도메인에서 자주 터지는 문제가 있습니다.
바로 **DB 트랜잭션은 성공했는데 이벤트 발행이 실패해서 다운스트림 시스템과 데이터가 어긋나는 문제**입니다.
이 글에서는 Node.js 서비스에서 트랜잭셔널 아웃박스(Transactional Outbox) 패턴을 적용해
이벤트 유실 없이 정합성을 지키는 구조를 정리합니다.

## 왜 트랜잭셔널 아웃박스가 필요한가

### DB 커밋과 메시지 브로커 발행은 원자적이지 않다

일반적으로 서비스는 아래 순서로 동작합니다.

1. 주문 데이터를 DB에 저장
2. Kafka/RabbitMQ/SQS 등에 `OrderCreated` 이벤트 발행

문제는 1번과 2번이 서로 다른 시스템이라는 점입니다.
DB 커밋 직후 프로세스가 죽거나 네트워크가 끊기면,
주문은 저장됐지만 이벤트는 영영 발행되지 않을 수 있습니다.

### “재시도”만으로는 유실을 100% 막기 어렵다

발행 실패 시 재시도 로직을 넣어도,
재시도 전에 프로세스가 종료되면 메모리 안의 작업은 사라집니다.
핵심은 **발행 대상 이벤트를 먼저 영속화**해 두는 것입니다.

## 트랜잭셔널 아웃박스 아키텍처

### 1) 비즈니스 데이터와 아웃박스 레코드를 같은 트랜잭션으로 저장

주문 생성 시 `orders` 테이블만 저장하지 않고,
`outbox_events` 테이블에 이벤트도 같이 INSERT 합니다.
둘 중 하나라도 실패하면 롤백되므로 정합성이 맞춰집니다.

### 2) 별도 발행기(Outbox Publisher)가 미발행 이벤트를 읽어 브로커에 전송

백그라운드 워커(혹은 크론/잡)가 주기적으로 `published_at IS NULL` 레코드를 조회해
메시지 브로커로 발행합니다.
발행 성공 시 `published_at`을 업데이트합니다.

### 3) 소비자/발행기 모두 멱등성을 전제로 설계

아웃박스는 “최소 1회(at-least-once)” 발행 모델이 기본이라
중복 이벤트 가능성을 열어두고 설계해야 합니다.
이벤트 ID 기반 중복 제거는 필수입니다.

## Node.js 구현 예시 (PostgreSQL + BullMQ)

### 아웃박스 테이블 스키마

```sql
CREATE TABLE outbox_events (
  id UUID PRIMARY KEY,
  aggregate_type TEXT NOT NULL,
  aggregate_id TEXT NOT NULL,
  event_type TEXT NOT NULL,
  payload JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published_at TIMESTAMPTZ,
  retry_count INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_outbox_unpublished
  ON outbox_events (created_at)
  WHERE published_at IS NULL;
```

### 주문 생성과 이벤트 적재를 하나의 트랜잭션으로 처리

```js
import { randomUUID } from 'node:crypto';

async function createOrderWithOutbox(client, orderInput) {
  await client.query('BEGIN');

  try {
    const orderRes = await client.query(
      `INSERT INTO orders (id, user_id, amount, status)
       VALUES ($1, $2, $3, $4)
       RETURNING id, user_id, amount, status`,
      [randomUUID(), orderInput.userId, orderInput.amount, 'CREATED']
    );

    const order = orderRes.rows[0];

    await client.query(
      `INSERT INTO outbox_events
       (id, aggregate_type, aggregate_id, event_type, payload)
       VALUES ($1, $2, $3, $4, $5)`,
      [
        randomUUID(),
        'order',
        order.id,
        'OrderCreated',
        JSON.stringify({
          orderId: order.id,
          userId: order.user_id,
          amount: order.amount,
        }),
      ]
    );

    await client.query('COMMIT');
    return order;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  }
}
```

### 아웃박스 발행 워커 (락 + 배치 + 상태 업데이트)

```js
async function publishOutboxBatch({ client, broker, batchSize = 100 }) {
  await client.query('BEGIN');

  try {
    const { rows } = await client.query(
      `SELECT id, event_type, payload
       FROM outbox_events
       WHERE published_at IS NULL
       ORDER BY created_at ASC
       FOR UPDATE SKIP LOCKED
       LIMIT $1`,
      [batchSize]
    );

    for (const row of rows) {
      try {
        await broker.publish(row.event_type, row.payload, {
          messageId: row.id, // 멱등 키
        });

        await client.query(
          `UPDATE outbox_events
           SET published_at = NOW()
           WHERE id = $1`,
          [row.id]
        );
      } catch (publishErr) {
        await client.query(
          `UPDATE outbox_events
           SET retry_count = retry_count + 1
           WHERE id = $1`,
          [row.id]
        );
      }
    }

    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  }
}
```

핵심 포인트는 세 가지입니다.

- `FOR UPDATE SKIP LOCKED`로 다중 워커 충돌 방지
- `messageId`로 중복 발행 대비
- 실패 시 `retry_count` 누적으로 운영 가시성 확보

## 운영에서 자주 놓치는 함정

### 아웃박스 테이블 무한 증가

발행 완료 데이터 정리 정책이 없으면 테이블이 급격히 커집니다.
예: `published_at` 기준 7~30일 보관 후 아카이브/삭제 정책 적용.

### “발행 성공” 기준을 애매하게 정의

브로커 ack를 받기 전 `published_at`을 업데이트하면 유실이 생깁니다.
반드시 **브로커 수신 확인 이후** 상태를 바꿔야 합니다.

### 장애 시 재처리 우선순위 부재

모든 이벤트를 동일 우선순위로 재처리하면,
결제/환불처럼 중요한 이벤트가 뒤로 밀릴 수 있습니다.
도메인별 큐 분리 또는 우선순위 큐가 필요합니다.

## 배포 전/후 체크리스트

### 배포 전

- 주문/결제 트랜잭션에 아웃박스 INSERT가 함께 들어가는지 확인
- 발행 워커 중복 실행 시 락(`SKIP LOCKED`)이 동작하는지 검증
- payload 내 개인정보/토큰 등 민감정보가 없는지 점검

### 배포 후 1주

- 미발행 이벤트 누적량(`published_at IS NULL`) 모니터링
- 이벤트 발행 지연 시간 p95/p99 추적
- 재시도 상위 이벤트 타입 원인 분석 후 개선 백로그화

## 요약

트랜잭셔널 아웃박스 패턴은
“DB에는 반영됐는데 이벤트가 사라지는” 가장 치명적인 정합성 문제를 줄여주는 현실적인 방법입니다.
완벽한 분산 트랜잭션 대신,
**같은 DB 트랜잭션으로 저장하고 나중에 안전하게 발행**하는 전략으로
운영 복잡도와 신뢰성의 균형을 맞출 수 있습니다.

## 내부 링크

- [/development/blog/seo/2026/03/14/dead-letter-queue-bullmq-nodejs-failed-job-recovery-guide.html](/development/blog/seo/2026/03/14/dead-letter-queue-bullmq-nodejs-failed-job-recovery-guide.html)
- [/development/blog/seo/2026/03/13/sli-slo-error-budget-nodejs-oncall-reliability-guide.html](/development/blog/seo/2026/03/13/sli-slo-error-budget-nodejs-oncall-reliability-guide.html)
- [/development/blog/seo/2026/03/08/idempotency-key-api-retry-consistency-guide.html](/development/blog/seo/2026/03/08/idempotency-key-api-retry-consistency-guide.html)

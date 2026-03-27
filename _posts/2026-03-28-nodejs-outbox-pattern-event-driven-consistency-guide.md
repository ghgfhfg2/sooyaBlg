---
layout: post
title: "Node.js Outbox Pattern 가이드: DB와 메시지 발행을 안전하게 맞추는 실무 설계법"
date: 2026-03-28 08:00:00 +0900
lang: ko
translation_key: nodejs-outbox-pattern-event-driven-consistency-guide
permalink: /development/blog/seo/2026/03/28/nodejs-outbox-pattern-event-driven-consistency-guide.html
alternates:
  ko: /development/blog/seo/2026/03/28/nodejs-outbox-pattern-event-driven-consistency-guide.html
  x_default: /development/blog/seo/2026/03/28/nodejs-outbox-pattern-event-driven-consistency-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, outbox-pattern, event-driven, transactional-outbox, backend, reliability]
description: "Node.js 서비스에서 outbox pattern을 적용해 DB 저장과 이벤트 발행 불일치를 줄이고, 재시도와 중복 발행까지 안전하게 다루는 방법을 정리했습니다."
---

주문 생성이나 결제 완료 같은 작업을 처리할 때, 애플리케이션은 보통 두 가지를 함께 해야 합니다.
하나는 **DB에 상태를 저장하는 일**이고, 다른 하나는 **메시지 브로커나 이벤트 버스로 후속 이벤트를 발행하는 일**입니다.
문제는 이 둘이 서로 다른 시스템이라는 점입니다.
DB 저장은 성공했는데 이벤트 발행이 실패하면 다른 서비스는 변화를 모른 채 남고, 반대로 이벤트는 나갔는데 DB 반영이 실패하면 더 골치 아픈 불일치가 생깁니다.
이때 자주 쓰는 해결책이 **outbox pattern**입니다.
이 글에서는 Node.js 서비스에서 outbox pattern이 왜 필요한지, 어떤 구조로 구현하면 좋은지, 중복 발행과 재시도는 어떻게 다루는지 실무 관점에서 정리합니다.

## Node.js outbox pattern은 왜 필요한가

### H3. DB 저장과 메시지 발행은 한 번에 성공하거나 실패하지 않는다

많은 팀이 처음에는 아래처럼 단순한 흐름으로 시작합니다.

1. 주문 정보를 DB에 저장
2. Kafka, RabbitMQ, SQS 같은 시스템에 이벤트 발행

평소에는 잘 동작해 보입니다.
하지만 실제 운영에서는 아래 같은 순간이 꼭 옵니다.

- DB 커밋 직후 네트워크 문제로 이벤트 발행 실패
- 이벤트 발행은 성공했지만 응답 전에 프로세스 종료
- 메시지 브로커 지연으로 애플리케이션 timeout 발생
- 재시도 과정에서 같은 이벤트가 중복 발행

핵심은 **DB 트랜잭션과 메시지 브로커 발행은 보통 하나의 원자적 트랜잭션으로 묶이지 않는다**는 점입니다.
그래서 둘 사이를 안전하게 연결하는 별도 장치가 필요합니다.

### H3. 이벤트 기반 아키텍처일수록 작은 불일치가 크게 번진다

단일 서비스 안에서는 DB 상태만 맞으면 버틸 수 있는 경우도 있습니다.
하지만 이벤트 기반 구조에서는 다른 서비스가 이벤트를 기준으로 움직입니다.
예를 들어 주문 서비스가 `order.created` 이벤트를 못 보내면 아래 같은 문제가 연쇄적으로 생길 수 있습니다.

- 재고 서비스가 재고 차감을 시작하지 못함
- 알림 서비스가 주문 완료 메시지를 보내지 못함
- 정산 서비스가 후속 작업을 시작하지 못함
- 분석 파이프라인에서 데이터 누락 발생

즉 outbox pattern은 단순히 "메시지 하나 더 확실하게 보내자"가 아니라, **서비스 간 데이터 흐름의 신뢰도를 지키는 패턴**에 가깝습니다.

## outbox pattern은 어떻게 동작하나

### H3. 비즈니스 데이터와 outbox 레코드를 같은 DB 트랜잭션 안에 기록한다

outbox pattern의 핵심은 생각보다 단순합니다.
메시지를 브로커로 바로 보내지 말고, 먼저 **DB 안의 outbox 테이블**에 "나중에 발행할 이벤트"를 함께 저장합니다.
중요한 점은 이 outbox 레코드가 비즈니스 데이터와 **같은 DB 트랜잭션** 안에서 기록돼야 한다는 것입니다.

예를 들어 주문 생성 시 아래처럼 처리합니다.

1. `orders` 테이블에 주문 저장
2. `outbox_events` 테이블에 `order.created` 이벤트 저장
3. 두 작업을 하나의 DB 트랜잭션으로 커밋
4. 별도 worker 또는 poller가 outbox를 읽어 브로커로 발행
5. 발행 성공 시 outbox 상태를 `published`로 갱신

이 구조의 장점은 분명합니다.
주문이 DB에 저장됐다면 outbox 레코드도 반드시 남고, 주문이 롤백됐다면 이벤트도 같이 사라집니다.
즉 **DB 내부에서는 최소한 일관성이 보장**됩니다.

### H3. 발행은 비동기로 분리하고, 실패는 재시도로 흡수한다

outbox pattern은 브로커 발행을 요청 처리 경로에서 완전히 떼어내는 경우가 많습니다.
즉 API 요청은 DB 커밋까지만 책임지고, 실제 이벤트 발행은 background worker가 담당합니다.

이렇게 하면 장점이 여러 가지입니다.

- 브로커 지연이 사용자 응답 시간을 직접 끌어올리지 않음
- 일시적인 브로커 장애를 재시도로 흡수 가능
- 발행 실패 이벤트를 모아서 모니터링하기 쉬움
- 배치 발행, 백오프, rate limiting 같은 운영 전략 적용 가능

물론 그 대가로 **즉시 발행이 아니라 잠깐의 지연 허용**이 필요합니다.
하지만 대부분의 비동기 이벤트 처리에서는 이 타협이 훨씬 현실적입니다.

## Node.js에서 outbox pattern을 언제 도입하는 게 좋을까

### H3. 주문, 결제, 회원가입 후속 처리처럼 후속 이벤트가 중요한 경로가 우선순위다

모든 서비스에 outbox pattern이 필요한 것은 아닙니다.
하지만 아래처럼 "DB 저장 이후 다른 시스템이 반드시 알아야 하는 변화"가 있으면 우선순위가 높습니다.

- 주문 생성 후 재고/알림/정산이 이어지는 경우
- 결제 승인 후 영수증 발행, 포인트 적립이 이어지는 경우
- 회원가입 후 이메일 발송, CRM 적재, 분석 이벤트가 이어지는 경우
- 콘텐츠 등록 후 검색 인덱싱, 캐시 갱신, 피드 반영이 필요한 경우

반대로 단일 DB 안에서만 끝나는 단순 CRUD라면 outbox가 과할 수 있습니다.
핵심은 **데이터 변경과 외부 전파가 둘 다 중요한 경로인가**를 보는 것입니다.

### H3. dual write가 이미 사고 포인트라면 빨리 정리하는 편이 낫다

실무에서 자주 보는 나쁜 냄새는 아래 패턴입니다.

- `DB 저장 -> 이벤트 발행`을 한 함수 안에서 순차 실행
- 실패하면 그냥 로그만 남기고 끝냄
- 이벤트 발행 실패를 운영자가 수동 복구
- 중복 발행을 소비자 쪽에서만 어설프게 막음

이 구조는 트래픽이 적을 때는 잠복해 있다가, 장애나 재배포 순간에 문제를 크게 터뜨립니다.
이미 "어쩌다 한 번 이벤트가 빠진다" 같은 증상이 있었다면 outbox pattern 도입 시점이 꽤 지난 경우가 많습니다.

## Node.js outbox pattern은 어떻게 구현할까

### H3. outbox 테이블에 이벤트 타입, payload, 상태, 재시도 메타를 함께 저장한다

기본 테이블은 대개 아래 정보가 필요합니다.

- 이벤트 ID
- aggregate type / aggregate ID
- event type
- payload JSON
- status(`pending`, `published`, `failed` 등)
- retry count
- next retry at
- created at / published at

간단한 예시는 아래처럼 잡을 수 있습니다.

```sql
create table outbox_events (
  id uuid primary key,
  aggregate_type varchar(100) not null,
  aggregate_id varchar(100) not null,
  event_type varchar(150) not null,
  payload jsonb not null,
  status varchar(30) not null default 'pending',
  retry_count integer not null default 0,
  next_retry_at timestamptz,
  published_at timestamptz,
  created_at timestamptz not null default now()
);

create index idx_outbox_pending on outbox_events (status, next_retry_at, created_at);
```

이때 payload에는 소비자가 정말 필요한 데이터만 넣는 편이 좋습니다.
민감정보를 통째로 넣으면 로그, 재처리, 분석 과정에서 노출 범위가 커집니다.
이 가이드는 공통 원칙에 맞게 **개인정보나 토큰, 결제 민감 필드 전체 원문 저장은 피하는 방향**을 권장합니다.

### H3. 비즈니스 트랜잭션 안에서 주문 데이터와 outbox 레코드를 같이 쓴다

Node.js에서 ORM이나 query builder를 쓰든, 핵심은 같습니다.
비즈니스 데이터와 outbox 레코드를 같은 트랜잭션에 넣어야 합니다.

아래는 개념 예시입니다.

```ts
import { randomUUID } from 'node:crypto';

async function createOrder(db, input) {
  return db.transaction(async (tx) => {
    const order = await tx.orders.insert({
      id: randomUUID(),
      userId: input.userId,
      amount: input.amount,
      status: 'created',
    });

    await tx.outboxEvents.insert({
      id: randomUUID(),
      aggregateType: 'order',
      aggregateId: order.id,
      eventType: 'order.created',
      payload: {
        orderId: order.id,
        userId: input.userId,
        amount: input.amount,
      },
      status: 'pending',
      nextRetryAt: new Date(),
    });

    return order;
  });
}
```

이 구조에서는 DB 커밋만 성공하면 최소한 "이벤트를 나중에라도 발행할 근거"가 남습니다.
그게 바로 dual write 문제를 크게 줄여주는 이유입니다.

## outbox 발행 worker는 어떻게 설계할까

### H3. poller는 작은 배치로 읽고, 발행 성공 후 상태를 명확히 갱신해야 한다

가장 흔한 구현은 poller 방식입니다.
주기적으로 `pending` 상태의 이벤트를 읽어 브로커로 발행하고, 성공 시 `published`로 바꿉니다.

대략적인 흐름은 아래와 같습니다.

1. 발행 대상 이벤트를 created 순서로 소량 조회
2. 브로커에 발행
3. 성공하면 `published_at` 기록 + `status=published`
4. 실패하면 `retry_count` 증가 + `next_retry_at` 갱신

Node.js 개념 예시는 아래와 같습니다.

```ts
async function publishPendingOutboxEvents(db, broker) {
  const events = await db.outboxEvents.findPendingBatch({
    now: new Date(),
    limit: 50,
  });

  for (const event of events) {
    try {
      await broker.publish(event.eventType, event.payload, {
        messageId: event.id,
      });

      await db.outboxEvents.markPublished({
        id: event.id,
        publishedAt: new Date(),
      });
    } catch (error) {
      await db.outboxEvents.markRetry({
        id: event.id,
        retryCount: event.retryCount + 1,
        nextRetryAt: calculateBackoffTime(event.retryCount + 1),
        lastError: String(error),
      });
    }
  }
}
```

실전에서는 다중 worker 환경에서 같은 레코드를 동시에 집지 않도록 `SELECT ... FOR UPDATE SKIP LOCKED` 같은 방식이나 상태 전이 잠금을 같이 쓰는 편이 안전합니다.

### H3. 재시도는 무한 루프가 아니라 백오프와 관측성을 함께 가져가야 한다

발행 실패는 outbox pattern에서 자연스러운 일입니다.
문제는 실패 자체보다 **실패를 어떻게 관리하느냐**입니다.

권장하는 방향은 아래와 같습니다.

- 즉시 재시도보다 backoff 사용
- retry count 상한 또는 dead-letter 전략 준비
- 마지막 에러 원인 저장
- 오래 쌓인 pending 이벤트 알림 설정
- 발행 지연 시간, 재시도 횟수, backlog 크기 메트릭 수집

이 부분은 이전에 다룬 <a href="/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html">Node.js Exponential Backoff + Jitter 가이드</a>와도 직접 연결됩니다.
재시도는 필요하지만, 몰아서 두드리면 브로커나 다운스트림을 더 아프게 만들 수 있기 때문입니다.

## 중복 발행은 어떻게 다뤄야 할까

### H3. outbox pattern은 at-least-once 성격을 갖기 때문에 소비자 idempotency가 여전히 중요하다

outbox pattern을 도입했다고 해서 중복 발행이 완전히 사라지는 것은 아닙니다.
예를 들어 브로커 발행은 성공했는데, 애플리케이션이 `published` 상태 갱신 전에 죽으면 다음 poller가 같은 이벤트를 다시 보낼 수 있습니다.

즉 outbox pattern은 보통 **at-least-once delivery**에 가깝습니다.
그래서 소비자 쪽에서도 아래 준비가 필요합니다.

- 이벤트 ID 기반 중복 처리 방지
- 동일 이벤트 재수신 시 안전한 no-op 처리
- side effect가 있는 소비 로직의 idempotency 보장

이 점은 어제 정리한 <a href="/development/blog/seo/2026/03/27/nodejs-idempotency-key-api-duplicate-request-guide.html">Node.js Idempotency Key 가이드</a>와 관점이 닿아 있습니다.
생산자와 소비자 모두에서 "같은 의도는 여러 번 와도 결과가 망가지지 않게" 설계해야 안정성이 올라갑니다.

### H3. 이벤트 ID와 natural key를 같이 쓰면 복구가 쉬워진다

소비자 입장에서는 메시지의 `eventId`만 믿기보다 도메인 natural key를 같이 보는 편이 유리할 때가 많습니다.
예를 들어 `order.created` 이벤트라면 `orderId`가 이미 한 번 처리된 주문인지 같이 확인할 수 있습니다.

이렇게 하면 아래 상황에서 복구가 수월합니다.

- 메시지가 중복 전송된 경우
- 재처리 배치를 수동 실행한 경우
- 운영 중 이벤트 순서가 살짝 뒤틀린 경우

결국 outbox pattern은 발행 신뢰도를 높여주지만, 시스템 전체 신뢰도는 **소비자 idempotency와 함께 설계할 때** 비로소 올라갑니다.

## outbox pattern과 다른 패턴은 어떻게 연결될까

### H3. circuit breaker, retry, webhook 검증과 역할을 나눠서 보는 편이 좋다

outbox pattern은 이벤트 발행 불일치를 줄이는 패턴이지, 모든 신뢰성 문제를 혼자 해결하지는 않습니다.
그래서 주변 패턴과 역할을 분리해서 봐야 합니다.

- retry: 일시적 발행 실패 복구
- circuit breaker: 브로커 또는 다운스트림 장애 전파 완화
- idempotency: 중복 처리 안전성 확보
- webhook 검증: 외부 이벤트 입력 신뢰성 확보

관련해서 아래 글을 함께 보면 흐름이 더 입체적으로 잡힙니다.

- <a href="/development/blog/seo/2026/03/27/nodejs-circuit-breaker-failure-isolation-guide.html">Node.js Circuit Breaker 패턴 가이드</a>
- <a href="/development/blog/seo/2026/03/23/nodejs-webhook-signature-verification-security-guide.html">Node.js Webhook Signature Verification 가이드</a>
- <a href="/development/blog/seo/2026/03/27/nodejs-idempotency-key-api-duplicate-request-guide.html">Node.js Idempotency Key 가이드</a>

즉 outbox는 "저장과 발행의 간격"을 안전하게 메우는 역할이고, 다른 패턴은 그 앞뒤에서 실패를 다루는 역할이라고 보면 이해가 쉽습니다.

## 운영에서 자주 하는 실수

### H3. outbox 테이블을 그냥 로그 테이블처럼 방치하는 경우

outbox 테이블은 한 번 만들고 끝나는 테이블이 아닙니다.
운영이 길어지면 행이 빠르게 쌓이고, 인덱스와 조회 성능에도 영향을 줍니다.
그래서 아래 전략이 필요합니다.

- 발행 완료 데이터 보관 기간 정의
- 주기적 archive 또는 cleanup
- pending/failed 이벤트 우선 조회 인덱스 유지
- backlog 급증 시 알림 설정

즉 outbox는 패턴이면서 동시에 **운영 대상 데이터 구조**입니다.
관리하지 않으면 나중에는 신뢰성 장치가 병목이 됩니다.

### H3. payload에 모든 모델을 통째로 넣는 경우

이벤트 payload를 설계할 때 ORM 결과 전체를 그대로 넣는 경우가 있습니다.
처음에는 편하지만 시간이 갈수록 문제가 생깁니다.

- 불필요한 개인정보 노출 가능성 증가
- 메시지 크기 증가
- 스키마 변경 영향 확대
- 소비자 결합도 상승

그래서 payload는 "이 이벤트를 처리하는 데 꼭 필요한 최소 정보" 위주로 잡는 편이 낫습니다.
이 원칙은 민감정보 보호와 유지보수성 두 측면 모두에서 중요합니다.

## 실무 체크리스트

### H3. 도입 전에 이 항목부터 정리하면 시행착오를 줄일 수 있다

- DB 변경과 함께 반드시 외부로 알려야 하는 이벤트가 무엇인가
- outbox 레코드가 비즈니스 데이터와 같은 트랜잭션에 저장되는가
- poller가 중복 집기를 피하도록 설계됐는가
- 재시도 백오프와 최대 재시도 정책이 있는가
- 발행 성공 후 상태 갱신 실패에 대비한 중복 처리 전략이 있는가
- 소비자가 event ID 또는 natural key 기준으로 idempotent한가
- outbox backlog, 실패율, 최대 지연 시간을 관측하는가
- 발행 완료 데이터 정리 정책이 있는가

## FAQ

### H3. Kafka의 transaction 기능이 있으면 outbox pattern이 필요 없을까

상황에 따라 줄어들 수는 있지만, 많은 서비스에서는 여전히 outbox가 더 단순하고 현실적인 선택입니다.
애플리케이션 DB와 메시지 시스템을 완전히 하나의 트랜잭션 경계로 맞추기 어렵기 때문입니다.
특히 여러 저장소가 섞이거나 운영 복잡도를 낮추고 싶다면 outbox pattern이 여전히 유효합니다.

### H3. CDC 기반으로 outbox 없이 바로 갈 수는 없을까

가능합니다.
예를 들어 DB 변경 로그를 읽는 CDC 파이프라인을 잘 갖추고 있다면 대안이 될 수 있습니다.
다만 초기 구축과 운영 난이도가 더 높을 수 있고, 팀 역량에 따라 오히려 단순한 outbox poller가 더 빠르게 안정화되기도 합니다.
핵심은 멋진 구조보다 **팀이 실제로 안정적으로 운영할 수 있는 구조**를 고르는 것입니다.

## 마무리

Node.js outbox pattern은 이벤트 기반 시스템에서 자주 터지는 dual write 문제를 줄이는 매우 실용적인 방법입니다.
핵심은 브로커 발행을 억지로 즉시 성공시키려 하기보다, **DB 안에 발행 의도를 안전하게 남기고 나중에 확실히 전달하는 흐름**을 만드는 데 있습니다.

특히 주문, 결제, 회원가입 같은 핵심 경로에서 DB 저장과 후속 이벤트 전파가 모두 중요하다면 outbox pattern은 꽤 높은 확률로 투자 대비 효과가 좋습니다.
이벤트 기반 Node.js 서비스를 운영 중인데 아직 "가끔 이벤트가 빠진다"는 냄새가 있다면, outbox pattern부터 정리하는 편이 훨씬 덜 아프게 오래 갑니다.

---
layout: post
title: "Node.js Saga Pattern 가이드: 분산 트랜잭션과 보상 처리 실무 설계법"
date: 2026-03-28 20:00:00 +0900
lang: ko
translation_key: nodejs-saga-pattern-distributed-transaction-compensation-guide
permalink: /development/blog/seo/2026/03/28/nodejs-saga-pattern-distributed-transaction-compensation-guide.html
alternates:
  ko: /development/blog/seo/2026/03/28/nodejs-saga-pattern-distributed-transaction-compensation-guide.html
  x_default: /development/blog/seo/2026/03/28/nodejs-saga-pattern-distributed-transaction-compensation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, saga-pattern, distributed-transaction, compensation, backend, reliability]
description: "Node.js 서비스에서 saga pattern을 적용해 분산 트랜잭션을 다루고, 실패 시 보상 처리와 재시도 전략을 안전하게 설계하는 방법을 정리했습니다."
---

주문, 결제, 재고, 알림처럼 여러 서비스가 순서대로 움직이는 구조에서는 "중간에 하나라도 실패하면 어떻게 되지?"라는 질문이 항상 따라옵니다.
단일 DB 트랜잭션이라면 롤백으로 끝낼 수 있지만, 마이크로서비스나 이벤트 기반 구조에서는 이야기가 달라집니다.
이미 결제는 승인됐는데 재고 차감이 실패하거나, 주문은 생성됐는데 배송 준비 단계가 멈추면 사용자는 애매한 상태를 보게 됩니다.
이럴 때 자주 꺼내는 설계가 **saga pattern**입니다.
이 글에서는 Node.js 서비스 기준으로 saga pattern이 왜 필요한지, choreography와 orchestration은 어떻게 다르고, 보상 처리와 idempotency는 어떻게 함께 설계해야 하는지 실무 관점에서 정리합니다.

## Node.js saga pattern은 왜 필요한가

### H3. 분산 환경에서는 하나의 ACID 트랜잭션으로 끝나지 않는다

모놀리식 애플리케이션에서는 주문 생성, 결제 상태 업데이트, 재고 차감 같은 작업을 하나의 DB 트랜잭션으로 묶을 수 있습니다.
하지만 서비스가 나뉘면 각 단계가 서로 다른 저장소와 네트워크 경계를 가집니다.
즉 아래 같은 흐름을 한 번에 커밋하거나 롤백하기 어렵습니다.

1. 주문 서비스에서 주문 생성
2. 결제 서비스에서 결제 승인
3. 재고 서비스에서 수량 차감
4. 배송 서비스에서 출고 준비

이때 한 단계가 실패하면 전체 흐름의 **비즈니스 일관성**이 깨집니다.
DB 수준의 원자성 대신, 서비스 간 협업으로 최종 일관성을 맞추는 패턴이 필요합니다.
그 역할을 saga pattern이 맡습니다.

### H3. 실패를 없애는 것이 아니라 실패 이후의 복구 흐름을 설계하는 패턴이다

saga pattern의 핵심은 "절대 실패하지 않는 흐름"을 만드는 게 아닙니다.
실패가 생길 것을 전제로, 각 단계가 실패했을 때 **어떤 보상 작업(compensation)** 으로 되돌릴지 미리 설계하는 데 있습니다.

예를 들어 주문 처리 saga에서는 아래처럼 볼 수 있습니다.

- 주문 생성 성공
- 결제 승인 성공
- 재고 차감 실패
- 결제 취소 보상 실행
- 주문 상태를 `cancelled`로 변경

즉 saga는 기술적 롤백이 아니라 **비즈니스 관점의 되돌리기 절차**에 가깝습니다.
그래서 설계가 애매하면 장애 때 더 큰 혼란을 부를 수 있고, 반대로 잘 설계하면 분산 시스템 운영 난도를 꽤 낮춰줍니다.

## saga pattern의 두 가지 방식: choreography와 orchestration

### H3. choreography는 이벤트 중심으로 느슨하게 연결된다

choreography 방식은 중앙 지휘자 없이 각 서비스가 이벤트를 구독하고 다음 행동을 이어가는 구조입니다.
예를 들어 아래처럼 진행할 수 있습니다.

1. 주문 서비스가 `order.created` 발행
2. 결제 서비스가 이를 구독해 결제 승인 후 `payment.completed` 발행
3. 재고 서비스가 이를 구독해 재고 차감 후 `inventory.reserved` 발행
4. 배송 서비스가 이를 구독해 출고 준비 시작

장점은 서비스 결합도가 낮고, 이벤트 기반 구조와 잘 맞는다는 점입니다.
하지만 흐름이 길어질수록 전체 상태를 추적하기 어려워집니다.
장애 시 "지금 어느 단계까지 갔는지"를 한눈에 보기 힘들고, 보상 이벤트도 여기저기 퍼질 수 있습니다.

### H3. orchestration은 중앙에서 흐름과 보상을 관리한다

orchestration 방식은 별도 saga orchestrator가 전체 단계를 관리합니다.
즉 "다음으로 무엇을 호출할지", "어느 단계에서 실패했는지", "어떤 보상을 실행할지"를 한 곳에서 정리합니다.

예를 들면 아래와 같습니다.

1. orchestrator가 주문 생성 요청
2. 성공하면 결제 승인 요청
3. 성공하면 재고 예약 요청
4. 실패하면 이전 단계에 대한 보상 호출

이 방식은 흐름 추적과 운영 가시성이 좋아집니다.
반면 orchestrator가 복잡해질 수 있고, 중앙 조정 로직이 병목이 되지 않도록 신경 써야 합니다.

실무적으로는 **단순한 도메인은 choreography**, **단계가 길고 보상 규칙이 복잡한 핵심 플로우는 orchestration**이 더 잘 맞는 경우가 많습니다.

## Node.js에서 saga pattern을 언제 도입해야 할까

### H3. 주문·결제·예약처럼 서비스 간 상태 일치가 중요한 경우가 우선이다

아래처럼 여러 서비스가 하나의 사용자 행동을 완성하는 구조라면 saga pattern 우선순위가 높습니다.

- 이커머스 주문 생성 → 결제 승인 → 재고 예약 → 배송 준비
- 여행 예약 → 좌석 확보 → 결제 승인 → 예약 확정
- 구독 결제 → 권한 부여 → 영수증 발행 → CRM 반영
- 광고 집행 → 예산 차감 → 캠페인 활성화 → 알림 발송

공통점은 분명합니다.
하나라도 중간 단계가 실패하면 사용자가 체감하는 상태가 꼬이기 쉽고, 수동 복구 비용도 커집니다.

### H3. 단순 CRUD나 짧은 동기 호출 체인에는 과할 수 있다

반대로 모든 흐름에 saga를 붙이는 건 과합니다.
단일 서비스 안에서 끝나는 작업, 실패 시 그냥 요청 전체를 에러 처리하면 되는 작업이라면 일반 트랜잭션과 재시도로 충분할 수 있습니다.

즉 도입 기준은 "분산 시스템이라서"가 아니라 아래 질문에 가깝습니다.

- 여러 서비스가 함께 성공해야 비즈니스가 완료되는가
- 중간 실패 시 부분 완료 상태가 실제 문제를 만드는가
- 단순 재시도만으로는 일관성을 맞추기 어려운가

이 세 가지에 모두 가깝다면 saga pattern 검토 가치가 큽니다.

## Node.js saga workflow는 어떻게 설계할까

### H3. 각 단계의 성공 조건과 보상 조건을 먼저 문서화해야 한다

코드부터 짜기보다 먼저 할 일은 **상태 전이 표를 만드는 것**입니다.
예를 들어 주문 saga라면 아래 정도는 선명해야 합니다.

- 주문 생성 성공 시 다음 단계는 무엇인가
- 결제 실패 시 주문 상태는 어떻게 바뀌는가
- 재고 예약 실패 시 결제 취소가 필요한가
- 배송 준비 실패 시 재고 해제와 결제 취소 중 어디까지 되돌릴 것인가
- 보상 작업이 또 실패하면 누가 재시도하는가

이 부분이 모호하면 실제 구현 단계에서 서비스별 예외 처리 로직이 제각각 커집니다.
결국 saga 설계는 코드보다 먼저 **업무 규칙을 명시하는 작업**에 가깝습니다.

### H3. saga 상태는 별도 저장소에서 추적하는 편이 운영에 유리하다

orchestration 방식이라면 saga 인스턴스 상태를 저장하는 테이블이나 문서 저장소를 두는 편이 좋습니다.
최소한 아래 정보는 남기는 편이 운영에 유리합니다.

- saga ID
- 비즈니스 키(orderId, reservationId 등)
- 현재 단계
- 전체 상태(`pending`, `completed`, `compensating`, `failed` 등)
- 단계별 실행 시각
- 최근 에러 원인
- 재시도 횟수

간단한 예시는 아래처럼 설계할 수 있습니다.

```sql
create table order_sagas (
  id uuid primary key,
  order_id uuid not null unique,
  current_step varchar(50) not null,
  status varchar(30) not null,
  retry_count integer not null default 0,
  last_error text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

이력이 남아 있으면 장애 대응 시 "무슨 단계에서 멈췄는지"를 바로 찾을 수 있습니다.
운영자가 수동 복구를 하더라도 기준점이 생긴다는 점이 큽니다.

## Node.js에서 orchestration 예시는 어떻게 생길까

### H3. 각 단계는 명시적 명령과 결과로 나누는 편이 읽기 쉽다

아래는 개념을 단순화한 예시입니다.
실제 구현에서는 HTTP 호출 대신 메시지 브로커, workflow engine, job queue를 쓸 수도 있습니다.

```ts
async function runOrderSaga({ orderId, payment, items }, deps) {
  await deps.sagaStore.markStep(orderId, 'create-order');
  await deps.orderService.createOrder({ orderId, items });

  try {
    await deps.sagaStore.markStep(orderId, 'charge-payment');
    await deps.paymentService.charge(payment);

    await deps.sagaStore.markStep(orderId, 'reserve-inventory');
    await deps.inventoryService.reserve(items);

    await deps.sagaStore.markCompleted(orderId);
  } catch (error) {
    await deps.sagaStore.markCompensating(orderId, String(error));

    await compensateOrderSaga({ orderId, payment, items }, deps);
    throw error;
  }
}

async function compensateOrderSaga({ orderId, payment, items }, deps) {
  await deps.inventoryService.release(items).catch(() => {});
  await deps.paymentService.refund(payment).catch(() => {});
  await deps.orderService.cancelOrder({ orderId }).catch(() => {});
}
```

이 코드는 개념 설명용이라 단순하지만, 메시지는 분명합니다.
성공 경로만큼 **보상 경로를 일급 로직으로 다뤄야** saga가 의미를 가집니다.

### H3. 보상 순서는 일반적으로 성공 단계의 역순이 자연스럽다

보상은 대개 앞에서 성공한 단계를 뒤에서부터 되돌리는 방식이 자연스럽습니다.
예를 들어 아래처럼요.

1. 재고 예약 성공
2. 결제 승인 성공
3. 배송 준비 실패

이 경우 보상은 보통 아래 순서를 탑니다.

1. 배송 리소스 정리
2. 재고 예약 해제
3. 결제 취소 또는 환불
4. 주문 상태 취소 처리

항상 완벽한 역순이 답은 아니지만, 의존성이 있는 자원을 정리할 때는 역순 보상이 가장 이해하기 쉽고 사고를 줄입니다.

## 보상 처리에서 자주 놓치는 포인트

### H3. compensation은 rollback이 아니라 또 다른 비즈니스 트랜잭션이다

이 부분을 헷갈리면 운영에서 크게 다칩니다.
보상 작업은 "없던 일로 만드는 마법"이 아니라 **새로운 상태 변경**입니다.
예를 들어 결제 취소는 단순 rollback이 아니라 결제 시스템에 남는 별도 거래일 수 있습니다.
재고 해제도 이미 다른 주문이 끼어든 뒤라면 원래 상태와 100% 똑같아지지 않을 수 있습니다.

그래서 compensation 로직은 아래 원칙으로 보는 편이 좋습니다.

- 되도록 명시적인 상태 전이를 남긴다
- 사용자가 보게 될 최종 상태를 분명히 정의한다
- 회계/정산/알림 관점에서 부작용을 같이 검토한다

### H3. 보상 실패 자체도 재시도와 관측 대상이어야 한다

실무에서는 본작업 실패보다 보상 실패가 더 아픈 경우가 있습니다.
결제 취소가 실패하면 금전 문제가 되고, 재고 해제가 실패하면 판매 가능 수량이 왜곡됩니다.
그래서 보상 호출도 아래처럼 다뤄야 합니다.

- idempotent하게 설계
- retry + backoff 적용
- 마지막 실패 원인 기록
- 오래 걸리는 보상 작업 알림 발송
- 수동 개입용 운영 도구 준비

재시도 전략은 <a href="/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html">Node.js Exponential Backoff + Jitter 가이드</a>와 연결해서 보면 좋습니다.
보상 실패에 즉시 무한 재시도를 걸면 장애를 복구하기보다 더 키울 수 있습니다.

## saga pattern과 idempotency는 왜 같이 가야 할까

### H3. 네트워크 재시도와 중복 메시지는 분산 시스템의 기본값에 가깝다

saga 단계는 보통 네트워크 호출이나 비동기 메시지를 포함합니다.
이 말은 곧 중복 실행 가능성을 항상 생각해야 한다는 뜻입니다.
예를 들어 orchestrator가 결제 승인 응답을 못 받으면, 실제 승인은 됐는데 타임아웃만 난 상황일 수 있습니다.
이 상태에서 같은 결제 요청을 다시 보내면 중복 결제가 날 수 있습니다.

그래서 각 단계는 가능한 한 idempotent해야 합니다.

- 결제 승인 요청에 idempotency key 사용
- 재고 예약 요청에 reservation key 사용
- 주문 취소 요청이 여러 번 와도 같은 결과 유지
- 보상 요청이 중복돼도 상태가 망가지지 않게 처리

이 부분은 <a href="/development/blog/seo/2026/03/27/nodejs-idempotency-key-api-duplicate-request-guide.html">Node.js Idempotency Key 가이드</a>를 같이 보면 감이 더 선명해집니다.
사가는 흐름을 관리하고, idempotency는 그 흐름이 여러 번 흔들려도 결과를 안전하게 지켜줍니다.

### H3. outbox pattern과 함께 쓰면 상태 전파의 신뢰도를 높일 수 있다

사가에서 단계 완료 이벤트를 다른 서비스로 전파해야 한다면, 이벤트 발행 자체의 신뢰성도 중요합니다.
이때 <a href="/development/blog/seo/2026/03/28/nodejs-outbox-pattern-event-driven-consistency-guide.html">Node.js Outbox Pattern 가이드</a>와 연결하면 설계가 더 안정적입니다.

예를 들어 결제 완료 후 `payment.completed` 이벤트를 발행할 때 outbox를 사용하면 아래 장점이 있습니다.

- DB 상태와 이벤트 발행 의도를 함께 저장 가능
- 브로커 일시 장애를 재시도로 흡수 가능
- saga 단계 전이를 더 안정적으로 다른 서비스에 전달 가능

즉 saga는 **분산 비즈니스 흐름의 일관성**, outbox는 **상태 전파의 신뢰성**을 담당한다고 보면 이해가 쉽습니다.

## 장애 대응 관점에서 어떤 메트릭을 봐야 할까

### H3. 성공률보다 지연 중인 saga와 compensating 상태 비중이 더 중요할 때가 많다

운영 대시보드에서 단순 성공 건수만 보면 실제 문제를 늦게 발견할 수 있습니다.
실무적으로는 아래 메트릭이 더 유용한 경우가 많습니다.

- 단계별 saga 시작 대비 완료 비율
- `compensating` 상태 개수
- 특정 단계 체류 시간 p95/p99
- 보상 작업 실패율
- 수동 개입이 필요한 `failed` saga 개수
- 동일 비즈니스 키에 대한 중복 실행 감지 건수

특히 `pending`이나 `compensating` 상태가 오래 쌓이는 순간은 운영 냄새가 강합니다.
사용자 체감 문제로 이어질 가능성이 높기 때문입니다.

### H3. circuit breaker와 timeout을 섞어 장애 전파를 줄여야 한다

사가 한 단계가 느려지면 뒤 단계 전체가 밀릴 수 있습니다.
특정 서비스 장애가 전체 주문 흐름을 잡아먹지 않게 하려면 timeout, retry, circuit breaker를 함께 써야 합니다.

관련해서 <a href="/development/blog/seo/2026/03/27/nodejs-circuit-breaker-failure-isolation-guide.html">Node.js Circuit Breaker 패턴 가이드</a>를 같이 보는 편을 권합니다.
보상과 재시도만으로는 장애 전파를 막기 어렵고, 아예 불안정한 의존성을 빨리 차단하는 전략도 필요하기 때문입니다.

## 실무 체크리스트

### H3. 도입 전에 이 항목을 먼저 정리하면 시행착오를 줄일 수 있다

- saga 단계별 성공 조건과 실패 조건이 문서화됐는가
- 각 단계에 대응하는 보상 작업이 정의됐는가
- 보상 작업이 idempotent한가
- 중복 메시지나 timeout 이후 재시도에 안전한가
- saga 상태를 조회하고 운영자가 추적할 수 있는가
- 실패한 saga를 재시도하거나 수동 종결할 운영 절차가 있는가
- timeout, retry, circuit breaker 정책이 단계별로 다른가
- 사용자에게 보여줄 최종 상태 문구가 일관적인가

## FAQ

### H3. saga pattern이 있으면 2PC 같은 분산 트랜잭션은 필요 없을까

항상 그렇지는 않습니다.
다만 많은 웹 서비스 환경에서는 2PC가 구현과 운영 복잡도에 비해 부담이 크고, 외부 서비스까지 포함하면 적용도 어렵습니다.
그럴 때 saga는 완벽한 원자성 대신 현실적인 최종 일관성을 주는 대안이 됩니다.

### H3. choreography와 orchestration 중 무엇이 더 좋을까

정답은 없습니다.
단계 수가 적고 서비스 간 결합을 낮추고 싶다면 choreography가 잘 맞을 수 있습니다.
반대로 핵심 결제 흐름처럼 추적성, 보상 규칙, 운영 가시성이 중요하면 orchestration이 보통 더 관리하기 쉽습니다.
중요한 건 유행이 아니라 **팀이 장애 때 이해하고 복구할 수 있는 구조인지**입니다.

## 마무리

Node.js saga pattern은 마이크로서비스나 이벤트 기반 구조에서 자주 마주치는 "부분 성공" 문제를 다루는 실용적인 설계입니다.
핵심은 모든 걸 한 번에 성공시키려 애쓰기보다, **단계를 나누고 실패 시 어떤 보상으로 비즈니스 일관성을 회복할지 미리 정하는 것**입니다.

특히 주문, 결제, 예약처럼 여러 서비스가 함께 움직이는 핵심 플로우를 운영하고 있다면 saga pattern은 단순한 이론이 아니라 사고 비용을 줄이는 장치에 가깝습니다.
Outbox, idempotency, circuit breaker 같은 패턴과 함께 묶어 설계하면 훨씬 현실적인 안정성을 만들 수 있습니다.

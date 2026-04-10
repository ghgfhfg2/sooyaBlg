---
layout: post
title: "Node.js Connection Pool Exhaustion 가이드: DB 연결 고갈을 예방하는 실무 전략"
date: 2026-04-11 08:00:00 +0900
lang: ko
translation_key: nodejs-connection-pool-exhaustion-prevention-guide
permalink: /development/blog/seo/2026/04/11/nodejs-connection-pool-exhaustion-prevention-guide.html
alternates:
  ko: /development/blog/seo/2026/04/11/nodejs-connection-pool-exhaustion-prevention-guide.html
  x_default: /development/blog/seo/2026/04/11/nodejs-connection-pool-exhaustion-prevention-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, connection-pool, database, postgres, mysql, prisma, overload, backend, resilience]
description: "Node.js에서 connection pool exhaustion이 발생하는 원인과 징후, pool size 설정, timeout, transaction 관리, 백프레셔와 관측 포인트까지 실무 기준으로 정리했습니다."
---

트래픽이 늘었는데 CPU는 여유가 있고, 애플리케이션 인스턴스도 살아 있는데 요청이 갑자기 느려지는 경우가 있습니다.
이때 의외로 자주 숨어 있는 원인이 **connection pool exhaustion**, 즉 DB 연결 풀 고갈입니다.
쿼리 자체가 아주 무겁지 않아도, 요청 처리 방식이 pool보다 빠르게 연결을 점유하면 전체 서비스가 대기열처럼 막히기 시작합니다.

Node.js 백엔드에서는 이 문제가 더 헷갈릴 수 있습니다.
이벤트 루프는 잘 돌고 있고 서버 프로세스도 정상처럼 보이는데, 실제 병목은 애플리케이션 바깥인 DB 연결 수 한도에서 생기기 때문입니다.
이 글에서는 Node.js에서 connection pool exhaustion이 왜 발생하는지, pool size를 어떻게 잡아야 하는지, timeout과 transaction을 어떻게 관리해야 하는지 실무 관점에서 정리합니다.

## Node.js Connection Pool Exhaustion이 왜 위험한가

### H3. 연결이 모자라면 에러보다 먼저 대기 시간이 폭증한다

connection pool 문제는 처음부터 명확한 에러로 드러나지 않는 경우가 많습니다.
대개는 아래 순서로 악화됩니다.

- 일부 요청이 연결 반환을 기다리며 지연됨
- 평균 latency보다 p95, p99가 먼저 튀기 시작함
- upstream timeout과 재시도가 늘어남
- 애플리케이션 동시성이 더 높아지며 pool 대기가 더 심해짐
- 결국 connection timeout, acquire timeout, too many clients 같은 에러가 발생함

즉 핵심 위험은 단순 접속 실패가 아니라 **대기 시간 전파**입니다.
요청 하나가 느려지면 같은 워커에서 더 오래 자원을 붙잡고, 그것이 다시 pool 대기로 이어져 전체 응답성을 무너뜨립니다.
이런 관점은 [Node.js Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와도 연결됩니다.
DB 대기를 무한정 허용하면 상위 계층 timeout 정책도 함께 무너집니다.

### H3. pool exhaustion은 DB 성능 문제와 애플리케이션 동시성 문제가 함께 만든다

많은 팀이 pool 고갈을 보면 먼저 DB 튜닝만 떠올립니다.
물론 느린 쿼리는 큰 원인이지만, 그게 전부는 아닙니다.
실무에서는 아래 조합이 자주 문제를 만듭니다.

- pool size에 비해 애플리케이션 인스턴스 수가 과도함
- transaction이 길어서 연결 점유 시간이 길어짐
- 외부 API 호출이나 파일 I/O를 transaction 내부에서 수행함
- 실패 요청이 재시도로 폭증함
- batch job과 사용자 요청이 같은 pool을 경쟁함

즉 pool exhaustion은 단순한 DB 튜닝 이슈가 아니라 **동시성 제어와 자원 분배 문제**입니다.
이 점은 [Node.js Adaptive Concurrency Limit 가이드](/development/blog/seo/2026/03/26/nodejs-adaptive-concurrency-limit-downstream-protection-guide.html)에서 다룬 하류 시스템 보호 전략과도 같은 맥락입니다.

## 어떤 상황에서 connection pool exhaustion이 잘 발생할까

### H3. 인스턴스 수만 늘리고 DB 최대 연결 수를 함께 계산하지 않은 경우

오토스케일링 이후 문제가 터지는 대표적인 패턴입니다.
예를 들어 인스턴스당 pool size를 20으로 잡고, 애플리케이션을 10대로 늘렸다면 이론상 200개 연결을 요구합니다.
그런데 DB의 `max_connections`가 150이고, 모니터링 에이전트나 관리 세션도 연결을 사용한다면 실제 여유는 더 적습니다.

이 상태에서는 개별 인스턴스가 정상처럼 보여도 전체 시스템은 항상 연결 부족 압력을 받습니다.
그래서 pool size는 앱 한 대 기준이 아니라 아래처럼 **전체 배치 기준**으로 계산해야 합니다.

- 인스턴스 최대 개수
- 인스턴스당 pool size
- 읽기/쓰기 분리 여부
- migration, admin tool, background worker가 쓰는 연결 수
- DB 자체가 안전하게 처리 가능한 동시 연결 수

### H3. 긴 transaction이 연결을 오래 붙잡는 경우

connection pool은 연결 개수만의 문제가 아니라 **점유 시간** 문제이기도 합니다.
pool size가 20이어도 transaction 하나가 3초씩 연결을 잡고 있으면 초당 처리량은 빠르게 제한됩니다.
특히 아래 패턴은 위험합니다.

- transaction 내부에서 외부 API 호출
- transaction 내부에서 대용량 loop 처리
- 사용자 입력 검증을 너무 늦게 수행
- 불필요하게 넓은 row 범위 lock
- 커밋 직전까지 많은 비즈니스 로직 수행

실무에서는 "쿼리 수가 많다"보다 "연결을 오래 놓지 않는다"가 더 자주 문제를 만듭니다.
그래서 transaction 내부 코드는 가능하면 짧고 결정적으로 유지해야 합니다.

### H3. 실패 재시도가 pool을 더 빨리 고갈시키는 경우

다운스트림 응답이 느릴수록 클라이언트와 gateway는 재시도를 시작합니다.
문제는 재시도가 이미 바쁜 DB에 더 많은 연결 경쟁을 얹는다는 점입니다.
이렇게 되면 원래는 버틸 수 있던 지연이 재시도 폭증 때문에 장애로 커집니다.

이럴 때는 단순히 pool size를 키우기보다 아래 조치가 먼저 필요합니다.

- acquire timeout을 짧게 둬 빠르게 실패시키기
- 요청 동시성 제한 적용
- 쓰기 요청 우선순위 분리
- 재시도 budget 제한
- 과부하 시 일부 기능을 갈색화하거나 지연 허용 범위를 낮추기

재시도 제어는 [Node.js Retry Budget 가이드](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html)와 함께 보면 설계가 더 쉬워집니다.

## Pool Size는 어떻게 정해야 할까

### H3. 큰 pool이 항상 좋은 것은 아니다

처음 겪는 팀은 종종 "대기가 생기니 pool을 100까지 늘리자"고 반응합니다.
하지만 pool을 크게 키우면 애플리케이션 입장에서는 잠깐 덜 막히는 것처럼 보여도, DB 입장에서는 context switching과 메모리 사용이 늘고 lock 경합이 심해질 수 있습니다.

즉 pool size는 병목을 해결하는 마법 숫자가 아니라 **부하를 어디서 흡수할지 정하는 설정**입니다.
대개는 무제한 확장보다 아래 원칙이 안전합니다.

- DB가 안정적으로 감당 가능한 연결 수를 먼저 정함
- 전체 앱/워커 수로 연결 예산을 나눔
- 여유 버퍼를 남김
- 나머지 초과 요청은 앱 계층에서 대기, 제한, 실패 처리함

애플리케이션이 DB보다 훨씬 많은 동시 요청을 받을 수 있다면, 병목은 결국 DB에서 고정됩니다.
그렇다면 그 한계를 인정하고 앞단에서 제어하는 편이 서비스 전체 안정성에 유리합니다.

### H3. 인스턴스 최대 개수를 기준으로 연결 예산을 나눈다

간단한 출발점은 아래 방식입니다.

1. DB에서 앱이 사용할 총 연결 예산을 정한다
2. 예약 연결 수를 뺀다
3. 최대 앱 인스턴스 수와 워커 수로 나눈다
4. 읽기/쓰기나 API/배치 풀을 따로 둘지 결정한다

예를 들어 DB에서 앱에 120개 연결까지 허용하고, 운영상 20개는 예약하고 싶고, 앱 인스턴스 최대 개수가 5대라면 인스턴스당 대략 20개 수준에서 시작할 수 있습니다.
여기에 배치 워커가 별도 pool을 쓰면 그 몫을 따로 빼야 합니다.

중요한 점은 **평균 인스턴스 수가 아니라 최대 인스턴스 수 기준**으로 설계해야 한다는 것입니다.
오토스케일이 붙은 환경에서는 피크 순간의 총합이 실제 장애를 결정합니다.

## Node.js에서 바로 적용할 수 있는 예방 전략

### H3. acquire timeout과 query timeout을 분리해서 둔다

많은 서비스가 timeout을 한 가지로만 생각합니다.
하지만 pool 문제를 다루려면 최소 두 가지를 구분하는 편이 좋습니다.

- acquire timeout: 연결을 빌리는 최대 대기 시간
- query timeout: 쿼리 자체가 실행될 최대 시간

이 둘을 분리해야 어디서 시간이 새는지 보입니다.
acquire timeout이 자주 발생하면 pool 경쟁 문제일 가능성이 높고, query timeout이 자주 발생하면 DB 실행 성능이나 lock 대기 문제가 더 의심됩니다.

예를 들면 아래처럼 생각할 수 있습니다.

- 전체 요청 timeout: 2초
- DB acquire timeout: 100~200ms
- query timeout: 500~800ms
- 남는 시간: 애플리케이션 로직과 응답 직렬화

이렇게 해야 요청 전체 예산 안에서 병목 위치를 판단할 수 있습니다.

### H3. transaction 안에서는 DB에 꼭 필요한 일만 한다

아래는 피해야 할 패턴입니다.

```js
await db.transaction(async (tx) => {
  const order = await tx.order.create({ data: input });

  // 좋지 않은 예시: 외부 API 호출이 transaction 안에 있음
  const payment = await paymentClient.approve(order);

  await tx.payment.create({
    data: { orderId: order.id, paymentId: payment.id },
  });
});
```

이 구조에서는 외부 결제 API가 느릴수록 DB 연결이 오래 점유됩니다.
가능하면 아래 원칙을 따르는 편이 안전합니다.

- validation은 transaction 전에 끝낸다
- 외부 API 호출은 가능하면 transaction 밖으로 뺀다
- transaction 내부는 짧은 read/write와 commit에 집중한다
- 후속 작업은 outbox, queue, 비동기 처리로 넘긴다

이런 분리는 [Node.js Outbox Pattern 가이드](/development/blog/seo/2026/03/28/nodejs-outbox-pattern-event-driven-consistency-guide.html)와도 잘 맞습니다.

### H3. pool 앞단에 요청 동시성 제한을 둔다

DB 연결 풀은 마지막 안전장치이지, 첫 번째 안전장치가 아닙니다.
애플리케이션 레벨에서 동시에 수행할 DB 작업 수를 제한하면 pool이 갑자기 바닥나는 상황을 줄일 수 있습니다.

간단한 예시는 아래와 같습니다.

```js
class Semaphore {
  constructor(limit) {
    this.limit = limit;
    this.active = 0;
    this.queue = [];
  }

  async use(fn) {
    if (this.active >= this.limit) {
      await new Promise((resolve) => this.queue.push(resolve));
    }

    this.active += 1;

    try {
      return await fn();
    } finally {
      this.active -= 1;
      this.queue.shift()?.();
    }
  }
}

const dbGate = new Semaphore(30);

app.post('/orders', async (req, res, next) => {
  try {
    const result = await dbGate.use(() => createOrder(req.body));
    res.status(201).json(result);
  } catch (error) {
    next(error);
  }
});
```

실제 운영에서는 라이브러리나 미들웨어, 큐, 우선순위 분리까지 함께 고려할 수 있습니다.
핵심은 **DB가 감당 가능한 수준보다 앞단 동시성이 먼저 커지지 않게 막는 것**입니다.
이 접근은 [Node.js Semaphore Pattern 가이드](/development/blog/seo/2026/04/08/nodejs-semaphore-pattern-shared-resource-concurrency-guide.html)와 자연스럽게 이어집니다.

## 어떤 지표를 보면 조기 징후를 잡을 수 있을까

### H3. 단순 에러율보다 pool wait time과 in-use connection을 먼저 본다

pool exhaustion은 에러율이 튀기 전에 징후가 보이는 경우가 많습니다.
아래 지표를 함께 보는 편이 좋습니다.

- pool in-use connection 수
- idle connection 수
- connection acquire 대기 시간
- acquire timeout 건수
- query latency와 transaction duration
- 인스턴스 수 증가와 동시 요청 수 변화
- DB CPU, lock wait, active session 수

특히 **pool wait time**이 계속 올라가는데 DB CPU가 낮다면, 단순한 쿼리 성능보다는 연결 사용 방식이나 transaction 설계 문제가 숨어 있을 가능성이 큽니다.

### H3. 요청 유형별로 pool 사용량을 나눠 봐야 원인을 찾기 쉽다

전체 요청 평균만 보면 어떤 엔드포인트가 연결을 오래 붙잡는지 놓치기 쉽습니다.
그래서 아래처럼 분해해 보는 것이 좋습니다.

- 읽기 API vs 쓰기 API
- 사용자 요청 vs 배치 작업
- 일반 transaction vs 장기 transaction
- 정상 요청 vs 재시도 요청

가능하면 로그나 trace에 아래 정보를 남기면 도움이 됩니다.

- route 또는 operation name
- acquire duration
- query count
- transaction duration
- timeout 여부
- retry 여부

단, SQL 원문이나 민감한 파라미터를 그대로 남기면 안 됩니다.
쿼리 템플릿 수준이나 마스킹된 식별자만 남기는 편이 안전합니다.

## 운영 중 장애가 났을 때는 어떻게 대응할까

### H3. pool size를 급하게 키우기 전에 긴 transaction과 재시도 폭증부터 본다

장애 상황에서 pool size를 바로 늘리면 체감상 잠깐 나아질 수 있습니다.
하지만 느린 transaction이나 재시도 폭증이 원인이라면 DB 전체를 더 힘들게 만들 수 있습니다.
우선순위는 보통 아래가 안전합니다.

1. acquire timeout, transaction duration, 느린 route 확인
2. 배치성 작업 또는 저우선 트래픽 차단
3. 재시도 축소 또는 일시 중단
4. 필요 시 앱 동시성 제한 강화
5. 마지막 수단으로 pool 조정 검토

즉 장애 대응의 초점은 "더 많은 연결 공급"보다 **연결 점유 시간을 줄이고 경쟁을 낮추는 것**에 있어야 합니다.

### H3. 사용자 요청과 백그라운드 작업을 같은 pool에 두지 않는 것도 효과적이다

실무에서 자주 효과를 보는 방법 중 하나가 pool 분리입니다.
예를 들어 아래를 나눌 수 있습니다.

- 실시간 사용자 API용 pool
- 배치/집계 작업용 pool
- 읽기 replica 전용 pool

이렇게 하면 배치 작업이 갑자기 몰려도 사용자 요청 전체가 같이 잠기는 상황을 줄일 수 있습니다.
물론 분리만으로 끝나지는 않고, 각 pool의 예산과 timeout 정책도 함께 관리해야 합니다.

## 정리

Node.js connection pool exhaustion은 단순히 DB 연결 수가 적어서 생기는 문제가 아닙니다.
대개는 **긴 transaction, 과도한 동시성, 느린 하류 시스템, 재시도 폭증, 잘못된 연결 예산 계산**이 함께 만든 결과입니다.

실무에서는 아래 다섯 가지부터 점검하면 효과가 큽니다.

- 최대 인스턴스 수 기준으로 전체 연결 예산 다시 계산하기
- acquire timeout과 query timeout 분리하기
- transaction 내부에서 외부 작업 빼기
- 앱 앞단에 동시성 제한 두기
- pool wait time, transaction duration, retry 급증을 함께 관측하기

DB는 결국 공유 자원입니다.
그래서 connection pool 전략의 핵심은 더 많이 여는 것이 아니라, **적절한 수의 연결을 더 짧고 안정적으로 쓰는 것**입니다.
그 기준이 잡히면 트래픽이 늘어도 훨씬 예측 가능한 운영이 가능해집니다.

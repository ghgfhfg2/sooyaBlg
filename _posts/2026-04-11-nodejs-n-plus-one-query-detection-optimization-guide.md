---
layout: post
title: "Node.js N+1 Query 문제 가이드: DB 부하와 응답 지연을 줄이는 실무 점검법"
date: 2026-04-11 20:00:00 +0900
lang: ko
translation_key: nodejs-n-plus-one-query-detection-optimization-guide
permalink: /development/blog/seo/2026/04/11/nodejs-n-plus-one-query-detection-optimization-guide.html
alternates:
  ko: /development/blog/seo/2026/04/11/nodejs-n-plus-one-query-detection-optimization-guide.html
  x_default: /development/blog/seo/2026/04/11/nodejs-n-plus-one-query-detection-optimization-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, n-plus-one, database, orm, prisma, sequelize, performance, backend, query-optimization]
description: "Node.js에서 N+1 Query 문제가 왜 생기는지, 로그와 APM으로 어떻게 발견하는지, join, eager loading, batching, 캐시 전략으로 어떻게 최적화하는지 실무 기준으로 정리했습니다."
---

API는 멀쩡히 동작하는데 특정 목록 화면만 유난히 느린 경우가 있습니다.
CPU도 크게 높지 않고 에러율도 낮은데, DB 쿼리 수만 조용히 폭증하는 패턴이라면 **N+1 Query 문제**를 먼저 의심할 만합니다.
이 문제는 처음에는 작은 비효율처럼 보이지만, 트래픽이 늘면 응답 지연과 DB 부하를 함께 키우며 운영 비용까지 밀어 올립니다.

특히 ORM을 쓰는 Node.js 서비스에서는 코드가 읽기 쉬운 대신, 관계 데이터를 무심코 반복 조회하면서 N+1이 숨어들기 쉽습니다.
이 글에서는 Node.js에서 N+1 Query가 왜 발생하는지, 어떤 로그와 지표로 발견할 수 있는지, 그리고 eager loading, batching, 캐시로 어떻게 줄일 수 있는지 실무 관점에서 정리합니다.

## Node.js N+1 Query 문제가 왜 위험한가

### H3. 요청 한 번이 끝날 때까지 불필요한 DB 왕복이 계속 늘어난다

N+1 Query는 보통 아래처럼 생깁니다.

1. 부모 목록을 가져오는 쿼리 1번 실행
2. 목록의 각 항목마다 자식 데이터를 가져오는 쿼리 N번 실행

예를 들어 주문 100건을 조회한 뒤, 각 주문의 사용자나 상품 정보를 별도 쿼리로 가져오면 요청 하나에 101개 이상의 쿼리가 생길 수 있습니다.
개발 환경 데이터가 작을 때는 잘 티가 안 나지만, 운영 데이터가 커질수록 쿼리 수와 네트워크 왕복 비용이 급격히 증가합니다.

이 문제는 단순히 "쿼리가 조금 많다" 수준에서 끝나지 않습니다.
DB 연결을 오래 점유하고, 같은 시간에 처리할 수 있는 요청 수를 줄이며, 결국 [Node.js Connection Pool Exhaustion 가이드](/development/blog/seo/2026/04/11/nodejs-connection-pool-exhaustion-prevention-guide.html)에서 설명한 연결 고갈 문제까지 불러올 수 있습니다.

### H3. 평균 latency보다 p95, p99가 먼저 나빠지는 경우가 많다

N+1 Query는 트래픽이 낮을 때는 평균 응답 시간에 크게 드러나지 않을 수 있습니다.
하지만 동시 요청이 늘어나면 각 요청이 더 많은 쿼리를 만들어 DB 경쟁이 심해지고, 그 영향이 상위 백분위 지연으로 먼저 나타납니다.

실무에서는 아래 징후가 함께 보이는 경우가 많습니다.

- 특정 목록 API만 p95, p99 latency가 급증함
- 같은 엔드포인트의 DB query count가 비정상적으로 높음
- DB CPU보다 connection acquire wait가 먼저 증가함
- ORM debug log를 켜면 비슷한 select 문이 반복됨
- 페이지네이션 크기를 늘릴수록 응답 시간이 거의 선형으로 악화됨

즉 N+1은 단순 코드 스타일 문제가 아니라 **동시성, 비용, 안정성 문제**입니다.

## 어떤 코드에서 N+1 Query가 자주 생길까

### H3. 목록을 순회하며 관계 데이터를 개별 조회하는 패턴

가장 흔한 형태는 아래와 같습니다.

```js
const orders = await prisma.order.findMany({
  where: { status: 'PAID' },
  take: 50,
});

const result = [];

for (const order of orders) {
  const user = await prisma.user.findUnique({
    where: { id: order.userId },
  });

  result.push({
    ...order,
    user,
  });
}
```

겉으로 보면 이해하기 쉬운 코드지만, 주문 수가 50건이면 사용자 조회도 최대 50번 발생합니다.
여기에 상품, 쿠폰, 배송지까지 붙기 시작하면 쿼리 수는 더 빠르게 늘어납니다.

### H3. 템플릿 직전 직렬화 단계에서 숨겨진 조회가 발생하는 패턴

N+1은 서비스 계층에서만 생기지 않습니다.
응답 DTO를 만들거나 템플릿에 넘길 데이터를 직렬화하는 단계에서 getter, resolver, serializer가 추가 조회를 일으키는 경우도 많습니다.

특히 아래 구조는 주의할 만합니다.

- GraphQL resolver가 필드마다 DB 조회를 수행함
- ORM relation accessor를 루프 안에서 호출함
- view model 생성 시 보조 데이터를 개별 조회함
- admin 화면에서 컬럼마다 lookup 쿼리를 추가함

코드를 읽을 때는 한 번의 조회처럼 보여도, 실제 런타임에서는 요청당 수십 번의 쿼리가 숨어 있을 수 있습니다.

## Node.js에서 N+1 Query를 발견하는 방법

### H3. 엔드포인트별 query count와 SQL 반복 패턴을 먼저 본다

가장 실용적인 출발점은 요청 하나가 끝날 때까지 실행된 쿼리 수를 세는 것입니다.
ORM이나 드라이버 로그를 구조화해 두면 특정 엔드포인트에서 같은 SQL이 반복되는지 금방 확인할 수 있습니다.

예를 들어 아래 항목을 같이 남기면 원인 파악이 쉬워집니다.

- request id
- route path
- 총 query count
- 총 DB 시간
- 가장 많이 반복된 SQL 패턴
- rows returned, duration 분포

이런 요청 단위 상관관계는 [Node.js AsyncLocalStorage 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)처럼 request context를 로그에 묶어 두면 훨씬 보기 좋아집니다.

### H3. APM에서 span 개수와 waterfall 모양을 함께 확인한다

APM을 쓰고 있다면 한 요청 안에 DB span이 비정상적으로 많이 늘어나는 패턴을 쉽게 찾을 수 있습니다.
특히 waterfall에서 비슷한 쿼리가 길게 반복되면 N+1 가능성이 높습니다.

실무에서는 아래 질문으로 점검하면 좋습니다.

- 같은 route에서 요청당 DB span 수가 page size와 함께 증가하는가
- 같은 SQL template이 짧은 간격으로 반복되는가
- 쿼리 실행 시간보다 round trip 횟수가 병목인가
- cache miss 직후만 느린지, 항상 느린지 구분되는가

분산 추적을 이미 쓰고 있다면 [OpenTelemetry Node.js 가이드](/development/blog/seo/2026/03/14/opentelemetry-nodejs-distributed-tracing-practical-guide.html)와 연결해 보는 것이 좋습니다.
DB 한 번이 느린 문제인지, DB를 너무 많이 부르는 문제인지 구분이 빨라집니다.

## N+1 Query를 줄이는 실무 전략

### H3. eager loading이나 join으로 필요한 관계를 한 번에 가져온다

가장 먼저 검토할 방법은 **한 번에 읽을 수 있는 관계는 한 번에 읽는 것**입니다.
ORM마다 표현은 다르지만 보통 include, populate, preload, eager loading 같은 기능을 제공합니다.

Prisma 예시는 아래처럼 바꿀 수 있습니다.

```js
const orders = await prisma.order.findMany({
  where: { status: 'PAID' },
  take: 50,
  include: {
    user: {
      select: {
        id: true,
        name: true,
        email: true,
      },
    },
  },
});
```

이 방식의 장점은 구현이 단순하고 즉시 효과가 크다는 점입니다.
다만 무조건 많은 relation을 한 번에 붙이면 row duplication이나 과도한 payload로 다른 문제가 생길 수 있으니, 정말 필요한 필드만 선택하는 편이 안전합니다.

### H3. 여러 개별 조회가 불가피하면 batching으로 묶는다

항상 join 하나로 끝나는 것은 아닙니다.
권한, 외부 소스, 다단계 조합처럼 관계가 복잡하면 조회를 완전히 합치기 어려울 수 있습니다.
이럴 때는 개별 조회를 그대로 두기보다 **batching**으로 묶는 편이 낫습니다.

예를 들어 주문 목록에서 필요한 `userId`를 먼저 모은 뒤, 한 번의 `in` 조회로 사용자 정보를 가져와 맵으로 조립할 수 있습니다.

```js
const orders = await prisma.order.findMany({
  where: { status: 'PAID' },
  take: 50,
});

const userIds = [...new Set(orders.map((order) => order.userId))];

const users = await prisma.user.findMany({
  where: { id: { in: userIds } },
  select: { id: true, name: true, email: true },
});

const userMap = new Map(users.map((user) => [user.id, user]));

const result = orders.map((order) => ({
  ...order,
  user: userMap.get(order.userId) ?? null,
}));
```

GraphQL 환경이라면 DataLoader 같은 배치 계층을 두는 것도 효과적입니다.
핵심은 루프 안의 즉시 조회를, 요청 범위 안에서 합칠 수 있는 조회로 바꾸는 것입니다.

### H3. 캐시는 마지막 단계로 쓰고, 먼저 쿼리 수를 줄인다

캐시는 분명 도움이 되지만, N+1의 1차 해결책은 아닙니다.
근본적으로 쿼리 구조가 비효율적이면 cache miss 순간마다 같은 문제가 반복됩니다.
그래서 보통 우선순위는 아래가 안전합니다.

1. 루프 안 개별 조회 제거
2. eager loading 또는 batching 적용
3. 필요한 필드만 select
4. 그다음 반복 조회가 많은 데이터에 캐시 적용

이 순서를 지키면 캐시가 성능 보조 수단이 되고, 구조적 비효율을 가리는 임시 처방이 되지 않습니다.
관련해서 동일 키 조회가 몰릴 때는 [Node.js Request Coalescing 가이드](/development/blog/seo/2026/04/05/nodejs-request-coalescing-cache-stampede-prevention-guide.html)를 함께 참고할 만합니다.

## N+1 최적화할 때 같이 점검해야 할 것

### H3. 페이지 크기와 응답 필드를 제한하지 않으면 최적화 효과가 줄어든다

N+1을 해결했다고 해서 무조건 빨라지는 것은 아닙니다.
한 번에 1000건을 읽고, 각 건마다 큰 relation을 여러 개 붙이면 이제는 쿼리 수 대신 결과 크기가 병목이 될 수 있습니다.

그래서 아래 항목을 함께 봐야 합니다.

- 기본 page size가 과도하지 않은가
- relation에 꼭 필요한 필드만 포함했는가
- 상세 화면용 데이터를 목록 API에 과하게 싣고 있지 않은가
- 정렬, 필터, 인덱스가 join 패턴과 맞는가

즉 N+1 최적화는 "쿼리 개수 줄이기"에서 끝나는 작업이 아니라, **한 요청의 데이터 예산을 다시 설계하는 작업**에 가깝습니다.

### H3. 동시 요청이 많은 엔드포인트는 앞단 동시성 제한도 같이 검토한다

N+1을 줄여도 트래픽 급증 상황에서는 여전히 DB가 부담을 받을 수 있습니다.
특히 인기 목록 API나 백오피스 집계 화면은 동시에 많이 호출되기 쉽습니다.
이런 엔드포인트에는 애플리케이션 레벨 동시성 제한이나 캐시 전략을 함께 두는 편이 안전합니다.

요청 폭주 제어 자체는 [Node.js Semaphore Pattern 가이드](/development/blog/seo/2026/04/08/nodejs-semaphore-pattern-shared-resource-concurrency-guide.html)처럼 앞단에서 제한을 거는 방식이 실무에서 자주 쓰입니다.
N+1 제거는 기본기이고, 과부하 제어는 별도 안전장치입니다.

## 운영에서 바로 쓰는 N+1 Query 점검 체크리스트

### H3. 배포 전과 장애 후에 같은 기준으로 확인한다

아래 체크리스트를 정해 두면 회귀를 줄이기 좋습니다.

- [ ] 목록 API의 요청당 query count를 측정하고 있는가
- [ ] page size 증가에 따라 query count가 함께 선형 증가하지 않는가
- [ ] 루프 안에서 ORM relation 조회가 일어나지 않는가
- [ ] include, preload, join에 필요한 필드만 선택했는가
- [ ] GraphQL resolver에 batching 계층이 있는가
- [ ] APM에서 같은 SQL span이 반복되는지 확인했는가
- [ ] 캐시 적용 전에 쿼리 구조부터 정리했는가

이 정도만 꾸준히 확인해도, 운영 중에 뒤늦게 "왜 이 API만 DB를 이렇게 많이 쓰지?"라는 상황을 크게 줄일 수 있습니다.

## 마무리

Node.js에서 N+1 Query는 작은 비효율처럼 시작하지만, 결국 DB 부하, 응답 지연, 연결 고갈을 함께 키우는 문제로 번지기 쉽습니다.
특히 ORM이 생산성을 높여 주는 만큼, 관계 조회가 실제로 몇 번의 쿼리로 번역되는지 계속 확인하는 습관이 중요합니다.

핵심은 단순합니다.
**루프 안 개별 조회를 줄이고, 요청 단위 query count를 측정하고, 한 번에 가져올 수 있는 데이터는 한 번에 가져오는 것**입니다.
이 기준만 잘 지켜도 목록 API와 백오피스 화면의 체감 성능이 꽤 크게 좋아집니다.

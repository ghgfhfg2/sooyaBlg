---
layout: post
title: "Node.js Request Coalescing 가이드: 캐시 스탬피드를 줄이는 실무 설계법"
date: 2026-04-05 20:00:00 +0900
lang: ko
translation_key: nodejs-request-coalescing-cache-stampede-prevention-guide
permalink: /development/blog/seo/2026/04/05/nodejs-request-coalescing-cache-stampede-prevention-guide.html
alternates:
  ko: /development/blog/seo/2026/04/05/nodejs-request-coalescing-cache-stampede-prevention-guide.html
  x_default: /development/blog/seo/2026/04/05/nodejs-request-coalescing-cache-stampede-prevention-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, request-coalescing, cache-stampede, caching, promise-sharing, resilience, concurrency, performance]
description: "Node.js 서비스에서 request coalescing을 적용해 캐시 스탬피드와 중복 호출을 줄이는 방법, Promise 공유 패턴, 실패 처리, TTL 설계 포인트를 실무 관점에서 정리했습니다."
---

캐시를 붙였는데도 트래픽이 몰리는 순간 서비스가 갑자기 느려지는 경우가 있습니다.
원인은 의외로 캐시 미스 자체보다 **같은 키를 향한 중복 요청이 한꺼번에 쏟아지는 현상**인 경우가 많습니다.
특정 인기 상품, 홈 피드, 대시보드 통계 API처럼 많은 사용자가 동시에 같은 데이터를 요청하면, 캐시가 비는 짧은 순간에도 애플리케이션과 다운스트림이 같은 계산을 여러 번 반복하게 됩니다.

이 문제를 줄이는 대표적인 방법이 **request coalescing**입니다.
동일한 키에 대한 동시 요청이 들어오면 첫 번째 요청만 실제로 실행하고, 나머지는 그 결과를 함께 기다리게 만드는 방식입니다.
이 글에서는 Node.js 환경에서 request coalescing이 왜 필요한지, Promise 공유 방식으로 어떻게 구현하는지, 그리고 캐시 TTL·실패 처리·오버로드 보호와 어떻게 연결해야 하는지 정리합니다.

## Node.js Request Coalescing이 왜 필요한가

### H3. 캐시가 있어도 캐시 스탬피드는 생길 수 있다

많이 오해하는 부분이 있습니다.
캐시를 쓴다고 해서 자동으로 중복 부하가 사라지는 것은 아닙니다.
오히려 TTL이 딱 끝나는 시점이나 cold start 직후에는 수십, 수백 개 요청이 같은 키를 동시에 다시 계산하려고 달려들 수 있습니다.

예를 들어 이런 상황을 생각해 볼 수 있습니다.

- 상품 상세 캐시 TTL이 60초다
- 60초가 지난 직후 인기 상품 페이지로 요청이 몰린다
- 서버 인스턴스마다 같은 DB 조회를 동시에 실행한다
- DB, Redis, 외부 API까지 연쇄적으로 부하가 커진다

이때 문제는 단순한 캐시 미스가 아니라 **같은 일을 여러 번 하는 중복 실행**입니다.
request coalescing은 바로 이 지점을 줄입니다.

### H3. 오버로드 상황에서는 중복 요청 제거가 지연 최적화보다 더 중요하다

트래픽이 안정적일 때는 중복 호출이 조금 있어도 티가 덜 납니다.
하지만 피크 시간대나 다운스트림이 느려진 시점에는 중복 호출 하나하나가 큽니다.
같은 데이터 하나를 가져오기 위해 Promise 50개가 각각 DB를 치는 구조와, Promise 1개만 실행하고 나머지가 결과를 공유하는 구조는 체감 차이가 큽니다.

이런 관점은 [Node.js Bounded Queue 가이드](/development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html)에서 다룬 큐 길이 제한, 그리고 [Node.js Concurrency Limit 가이드](/development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)에서 설명한 동시성 제어와도 맞닿아 있습니다.
즉 request coalescing은 캐시 기법이면서 동시에 **오버로드 완화 전략**이기도 합니다.

## Request Coalescing은 무엇이고 memoization과 무엇이 다를까

### H3. 핵심은 같은 시점의 같은 작업을 하나로 합치는 것이다

request coalescing은 일정 시간 동안의 결과를 저장하는 캐시와 비슷해 보이지만, 초점이 약간 다릅니다.
핵심은 **이미 진행 중인 동일 작업을 하나로 합치는 것**입니다.

간단히 구분하면 아래와 같습니다.

- 캐시: 이미 계산이 끝난 결과를 재사용
- memoization: 함수 입력과 결과를 메모리에 보관해 재사용
- request coalescing: 현재 진행 중인 동일 요청을 하나로 합쳐 공유

실무에서는 셋을 같이 쓰는 경우가 많습니다.
예를 들어 캐시에 값이 있으면 바로 반환하고, 캐시에 없으면 coalescing으로 한 번만 원본 조회를 실행한 뒤 결과를 다시 캐시에 넣는 식입니다.

### H3. Promise 공유 패턴이 Node.js와 특히 잘 맞는다

Node.js는 비동기 작업을 Promise로 다루기 때문에 coalescing 구현이 비교적 자연스럽습니다.
같은 키에 대해 이미 진행 중인 Promise가 있다면 그 Promise를 그대로 반환하면 됩니다.
그러면 요청자 수가 1명이든 100명이든 실제 실행은 1번만 일어납니다.

다만 단순해 보여도 몇 가지 함정이 있습니다.

- 실패한 Promise를 맵에 오래 남겨 두면 이후 요청도 계속 실패를 재사용할 수 있다
- key 설계가 부정확하면 서로 다른 요청이 잘못 합쳐질 수 있다
- 너무 많은 in-flight 항목을 허용하면 맵 자체가 메모리 부담이 될 수 있다

즉 request coalescing은 코드 몇 줄로 시작할 수 있지만, 운영하려면 **수명 관리와 실패 정책**을 함께 설계해야 합니다.

## Node.js에서 Request Coalescing을 어떻게 구현할까

### H3. 가장 단순한 형태는 in-flight Promise 맵이다

아래 예시는 동일한 캐시 키에 대한 동시 요청을 하나의 Promise로 묶는 가장 단순한 형태입니다.

```js
const inflight = new Map();
const cache = new Map();

async function getProduct(productId) {
  const cacheKey = `product:${productId}`;
  const cached = cache.get(cacheKey);

  if (cached && cached.expiresAt > Date.now()) {
    return cached.value;
  }

  if (inflight.has(cacheKey)) {
    return inflight.get(cacheKey);
  }

  const promise = (async () => {
    try {
      const product = await fetchProductFromDB(productId);
      cache.set(cacheKey, {
        value: product,
        expiresAt: Date.now() + 60_000
      });
      return product;
    } finally {
      inflight.delete(cacheKey);
    }
  })();

  inflight.set(cacheKey, promise);
  return promise;
}
```

이 구조의 장점은 명확합니다.

- 같은 키를 향한 동시 요청이 하나로 합쳐진다
- 첫 요청이 끝나면 이후 요청은 캐시에서 빠르게 반환된다
- 구현 복잡도가 낮아 빠르게 적용할 수 있다

하지만 이것만으로는 충분하지 않습니다.
실패 처리, timeout, stale 데이터 활용, key 폭증 방어가 빠져 있기 때문입니다.

### H3. 실패와 timeout을 함께 설계해야 다운스트림 보호 효과가 커진다

coalescing된 단일 요청이 너무 오래 걸리면 기다리는 모든 요청이 같이 묶여서 느려질 수 있습니다.
그래서 실무에서는 원본 조회에 timeout을 걸고, 실패 시 in-flight 상태를 즉시 정리해야 합니다.

```js
async function withTimeout(promise, timeoutMs) {
  return Promise.race([
    promise,
    new Promise((_, reject) => {
      setTimeout(() => reject(new Error(`timeout after ${timeoutMs}ms`)), timeoutMs);
    })
  ]);
}

async function getDashboardStats(teamId) {
  const key = `dashboard:${teamId}`;

  if (inflight.has(key)) {
    return inflight.get(key);
  }

  const promise = (async () => {
    try {
      return await withTimeout(fetchStatsFromOrigin(teamId), 800);
    } finally {
      inflight.delete(key);
    }
  })();

  inflight.set(key, promise);
  return promise;
}
```

여기서 중요한 포인트는 두 가지입니다.

1. 느린 요청을 무한정 공유하지 않는다
2. 실패 후에는 새 요청이 다시 정상 경로를 시도할 수 있게 정리한다

이 원칙은 [Node.js Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와 같이 보면 더 이해가 쉽습니다.
coalescing은 중복 실행을 줄여 주지만, **느린 단일 실행의 위험까지 해결해 주지는 않기 때문**입니다.

## 캐시 스탬피드를 줄이려면 TTL 설계가 같이 가야 한다

### H3. TTL이 동시에 만료되면 coalescing만으로는 충분하지 않을 수 있다

request coalescing을 넣었다고 해서 모든 stampede가 사라지지는 않습니다.
특히 서버 인스턴스가 여러 대이고 각 인스턴스가 독립 메모리를 쓰는 구조라면, 인스턴스마다 같은 키에 대한 첫 요청이 따로 실행될 수 있습니다.
또 TTL이 모든 키에서 같은 규칙으로 만료되면 특정 시점에 미스가 몰릴 수 있습니다.

그래서 아래 보완책을 같이 고려하는 편이 좋습니다.

- TTL에 jitter를 넣어 만료 시점을 분산한다
- hot key는 더 긴 TTL 또는 별도 갱신 정책을 둔다
- stale 데이터를 잠시 허용해 원본 조회 압력을 줄인다
- 인스턴스 간 공유 락 또는 분산 캐시 전략을 검토한다

### H3. stale-while-revalidate와 같이 쓰면 사용자 체감이 더 좋아진다

사용자가 꼭 최신값을 실시간으로 볼 필요가 없는 데이터라면 **stale-while-revalidate** 패턴과 조합하는 것이 특히 좋습니다.
오래된 값을 잠시 먼저 반환하고, 백그라운드에서 한 번만 새 값을 가져오도록 만들 수 있기 때문입니다.

이 방식의 장점은 분명합니다.

- 사용자 응답 속도가 안정적이다
- 원본 시스템에 급격한 갱신 부하가 덜 간다
- hot key에서 캐시 만료 충격을 완화하기 쉽다

관련 흐름은 [Node.js Stale-While-Revalidate 가이드](/development/blog/seo/2026/03/26/nodejs-stale-while-revalidate-cache-strategy-guide.html)와 이어서 보면 설계 감이 더 잘 잡힙니다.
request coalescing이 같은 순간의 중복 요청을 합친다면, stale-while-revalidate는 **갱신 타이밍의 사용자 충격을 줄이는 보완책**입니다.

## 운영에서 자주 놓치는 설계 포인트

### H3. key를 너무 넓게 잡아도 문제고 너무 세밀하게 잡아도 문제다

coalescing 성공률은 key 설계에 크게 좌우됩니다.
같은 결과를 반환해야 하는 요청인데도 쿼리 파라미터 순서, 불필요한 헤더, 추적용 값까지 key에 포함하면 공유율이 급격히 떨어집니다.
반대로 사용자 권한이나 로캘 차이를 무시하고 key를 너무 넓게 잡으면 잘못된 데이터가 섞일 수 있습니다.

실무에서는 아래 기준이 무난합니다.

- 응답 결과를 실제로 바꾸는 요소만 key에 포함
- 사용자별 권한 차이가 있으면 반드시 분리
- 정렬 순서, 필터, 로캘 등 결과 차이에 영향 주는 값만 반영
- timestamp 같은 노이즈 값은 제외

### H3. 메모리 보호 장치가 없으면 hot key보다 key 폭증이 더 위험할 수 있다

많은 팀이 hot key만 걱정하지만, 실제 운영에서는 드문 키가 짧은 시간에 대량으로 들어와 in-flight 맵이 비대해지는 경우도 있습니다.
특히 검색, 추천, 사용자 생성 필터처럼 key 공간이 큰 API는 주의가 필요합니다.

그래서 아래 같은 보호 장치를 고려할 수 있습니다.

- in-flight 맵 최대 크기 제한
- 특정 키 패턴만 coalescing 적용
- 고비용 API만 선택적으로 적용
- 일정 이상 대기자가 붙는 키는 별도 모니터링

즉 request coalescing은 모든 엔드포인트에 일괄 적용하는 만능 패턴이 아니라, **중복 계산 비용이 크고 동시성이 높은 경로에 우선 적용할 때** 효과가 좋습니다.

## 언제 쓰면 좋고 언제 과한가

### H3. 특히 잘 맞는 케이스

다음처럼 동일한 읽기 요청이 짧은 시간에 몰리기 쉬운 경로에는 효과가 큽니다.

- 홈 화면 집계 데이터
- 인기 상품/콘텐츠 상세
- 대시보드 통계 카드
- 외부 API 기반 환율·시세 조회
- 동일 파라미터로 반복 호출되는 내부 BFF API

### H3. 과하거나 신중해야 하는 케이스

반대로 아래 경우에는 coalescing보다 다른 전략이 더 중요할 수 있습니다.

- 사용자별 데이터 차이가 커서 공유 여지가 낮은 요청
- 쓰기 요청처럼 멱등성과 순서 보장이 중요한 작업
- 요청 결과가 매우 짧은 시간에도 민감하게 바뀌는 실시간 데이터
- 이미 강한 캐시와 큐잉, rate limit이 잘 갖춰진 경로

핵심은 단순합니다.
request coalescing은 **같은 일을 동시에 여러 번 하지 않도록 만드는 기술**이지, 모든 성능 문제를 해결하는 만능 버튼은 아닙니다.
하지만 캐시 스탬피드와 중복 호출이 문제인 구간에서는 생각보다 작은 구현으로 큰 효과를 낼 수 있습니다.

## 실무 적용 체크리스트

### H3. 적용 전 확인할 것

- 이 엔드포인트는 정말 같은 키 요청이 몰리는가?
- 결과를 공유해도 안전한 읽기 요청인가?
- timeout, stale 허용, 실패 정책이 정의돼 있는가?
- 인스턴스 간 중복 실행도 문제라면 분산 전략이 필요한가?

### H3. 적용 후 모니터링할 것

- cache hit ratio
- coalescing hit ratio
- hot key별 대기자 수
- 원본 조회 횟수 감소율
- timeout, error rate, p95/p99 latency 변화

숫자를 보면 금방 감이 옵니다.
coalescing이 잘 동작하면 같은 시간대 요청 수 대비 원본 호출 수가 줄고, 피크 구간 지연도 더 완만해집니다.
반대로 대기 시간이 너무 길어지거나 실패 전파가 커지면 timeout과 stale 정책을 다시 조정해야 합니다.

## 마무리

Node.js에서 request coalescing은 복잡한 분산 락부터 도입하지 않아도 시작할 수 있는 실용적인 최적화입니다.
같은 키에 대한 동시 요청을 하나의 Promise로 묶는 것만으로도 캐시 스탬피드, 중복 DB 조회, 외부 API 호출 폭증을 눈에 띄게 줄일 수 있습니다.

다만 중요한 건 "합친다" 자체보다 **어디에 적용할지, 실패 시 어떻게 풀지, stale 데이터와 timeout을 어떻게 조합할지**입니다.
실무에서는 request coalescing 하나만 따로 보지 말고 캐시 전략, 동시성 제한, 큐 길이 보호, timeout budget과 함께 운영해야 진짜 효과가 납니다.

캐시가 있는데도 피크 시간마다 같은 데이터 조회 때문에 시스템이 흔들린다면, 다음 최적화 후보는 캐시 교체가 아니라 **중복 요청 합치기**일 가능성이 높습니다.

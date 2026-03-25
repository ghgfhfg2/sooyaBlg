---
layout: post
title: "Node.js Adaptive Concurrency Limit 가이드: 다운스트림 보호와 p99 안정화를 함께 잡는 법"
date: 2026-03-26 08:00:00 +0900
lang: ko
translation_key: nodejs-adaptive-concurrency-limit-downstream-protection-guide
permalink: /development/blog/seo/2026/03/26/nodejs-adaptive-concurrency-limit-downstream-protection-guide.html
alternates:
  ko: /development/blog/seo/2026/03/26/nodejs-adaptive-concurrency-limit-downstream-protection-guide.html
  x_default: /development/blog/seo/2026/03/26/nodejs-adaptive-concurrency-limit-downstream-protection-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, adaptive-concurrency, concurrency-limit, backend, resilience, performance]
description: "Node.js 서비스에서 adaptive concurrency limit를 적용해 다운스트림 과부하를 줄이고 p99 지연을 안정화하는 실무 설계 방법을 정리했습니다."
---

트래픽이 늘어날 때 많은 팀이 먼저 생각하는 건 서버 증설입니다.
물론 필요한 선택입니다.
하지만 실제 운영에서는 **서버 자체보다 DB, 캐시, 외부 API 같은 다운스트림이 먼저 버거워지는 순간**이 더 자주 문제를 만듭니다.
이때 Node.js 애플리케이션이 무작정 요청을 계속 흘려보내면, 평균 응답 시간보다 p99가 먼저 무너지고 timeout·retry·에러율이 함께 치솟습니다.
그래서 필요한 것이 **adaptive concurrency limit** 입니다.
현재 시스템 상태를 보면서 동시에 처리할 요청 수를 동적으로 조절해, 다운스트림을 보호하고 전체 지연을 안정화하는 방식입니다.
이 글에서는 adaptive concurrency limit가 왜 필요한지, 고정 동시성 제한과 무엇이 다른지, Node.js에서 어떤 신호로 제한값을 조절해야 하는지 실무 중심으로 정리합니다.

## Node.js adaptive concurrency limit는 왜 필요한가

### H3. 트래픽보다 다운스트림 병목이 먼저 한계를 만든다

Node.js API 서버가 아무리 가볍게 보여도, 실제 병목은 뒤쪽에서 생기는 경우가 많습니다.
대표적으로 아래 자원이 먼저 포화됩니다.

- DB connection pool
- Redis 연결 수와 command latency
- 외부 결제·메시징·인증 API 동시 호출 한도
- 내부 마이크로서비스 worker 또는 thread pool

문제는 이 한계를 애플리케이션이 모른 채 계속 요청을 밀어 넣으면, 한동안은 처리량이 늘어나는 것처럼 보이다가 어느 순간 급격히 무너진다는 점입니다.
특히 p95보다 p99가 먼저 흔들리고, 이후 timeout과 재시도가 겹치며 더 큰 장애로 번집니다.
adaptive concurrency limit는 이 구간에서 **지금 시스템이 감당 가능한 만큼만 흘려보내는 장치** 역할을 합니다.

### H3. 고정 상한만으로는 시간대별·상황별 변화에 대응하기 어렵다

예를 들어 새벽에는 동시 80개가 안전하지만, 점심 피크에는 같은 80개도 DB latency를 급격히 악화시킬 수 있습니다.
반대로 캐시 적중률이 높거나 특정 기능이 비활성화된 시간에는 더 높은 동시성도 충분히 소화할 수 있습니다.

이런 상황에서 고정값 하나로만 운영하면 두 가지 문제가 생깁니다.

1. 평소에는 너무 보수적이라 처리량을 낭비한다
2. 피크에는 너무 공격적이라 다운스트림을 압박한다

adaptive concurrency limit는 응답 시간, 에러율, queue depth 같은 신호를 보고 상한을 올리거나 내리기 때문에 고정 임계치보다 훨씬 현실적인 운영이 가능합니다.

## 고정 concurrency limit와 adaptive limit는 무엇이 다를까

### H3. 고정 limit는 단순하고 안전하지만 환경 변화에 둔감하다

고정 동시성 제한은 구현이 단순하고 예측 가능하다는 장점이 있습니다.
예를 들어 최대 50개 요청만 동시에 처리하도록 두면, 메모리와 커넥션 사용량이 갑자기 폭주하는 것을 꽤 잘 막을 수 있습니다.

다만 아래 상황에는 한계가 뚜렷합니다.

- 특정 시간대만 유독 느려지는 다운스트림
- 외부 API 품질 저하로 latency가 순간적으로 악화되는 경우
- 일부 엔드포인트만 유독 무거워지는 배포 직후 상황
- 캐시 hit ratio 변화에 따라 체감 처리량이 달라지는 경우

즉 고정 limit는 시작점으로는 좋지만, 운영이 복잡해질수록 자동 조절 장치가 필요해집니다.

### H3. adaptive limit는 관측값을 기준으로 안전 범위를 계속 재조정한다

adaptive limit의 핵심은 단순합니다.
**지연과 실패가 늘어나면 limit를 줄이고, 안정적이면 조금씩 다시 늘리는 것**입니다.

개념적으로는 아래 흐름에 가깝습니다.

- 최근 요청의 latency가 목표보다 낮다
- timeout·5xx 비율이 안정적이다
- queue가 과도하게 쌓이지 않는다
- 그러면 동시성 상한을 소폭 올린다

반대로 아래 신호가 보이면 limit를 줄입니다.

- p95/p99 응답 시간이 급격히 증가한다
- upstream timeout 비율이 올라간다
- DB pool wait 시간이 길어진다
- 503 또는 5xx가 빠르게 증가한다

이 방식은 어제 다룬 <a href="/development/blog/seo/2026/03/25/nodejs-load-shedding-overload-protection-guide.html">Node.js Load Shedding 가이드</a>와도 잘 연결됩니다.
adaptive concurrency limit가 선제적으로 유입량을 조절하고, 그래도 위험할 때는 load shedding이 마지막 방어선이 되는 구조가 좋습니다.

## 어떤 신호로 concurrency limit를 조절해야 할까

### H3. 평균 latency보다 p95, p99와 timeout 비율을 더 중요하게 봐야 한다

평균 응답 시간만 보면 상황을 지나치게 낙관적으로 해석하기 쉽습니다.
운영 장애는 대부분 꼬리 지연(tail latency)에서 먼저 드러나기 때문입니다.

실무에서는 아래 신호를 우선적으로 보는 편이 낫습니다.

- p95, p99 latency
- timeout 비율
- upstream 5xx 비율
- DB pool wait time
- event loop lag
- queue depth

예를 들어 평균은 120ms로 멀쩡해 보여도 p99가 2.8초까지 튄다면 이미 사용자는 느린 서비스를 체감하고 있을 가능성이 큽니다.
이때 동시성을 조금만 줄여도 tail latency가 크게 안정되는 경우가 많습니다.

### H3. 한 가지 지표보다 복합 신호가 더 안전하다

동시성 조절을 latency 하나에만 걸면, 일시적인 네트워크 흔들림에도 limit가 과하게 출렁일 수 있습니다.
반대로 에러율만 보면 느려지기만 하고 아직 실패하지 않은 구간을 놓칠 수 있습니다.

그래서 보통은 아래처럼 묶어서 판단합니다.

- 최근 1분 p99가 목표치 초과
- 동시에 DB pool wait 상승
- 그리고 timeout 비율 증가

이 세 가지가 함께 보이면 상한을 내리고, 일정 시간 안정되면 아주 천천히 다시 올리는 식입니다.
핵심은 민감하게 내리고, 보수적으로 올리는 것입니다.

## Node.js에서 adaptive concurrency limit를 어떻게 구현할까

### H3. 먼저 semaphore 기반으로 현재 동시 실행 수를 제어한다

Node.js에서는 무제한 `Promise.all` 이나 과도한 병렬 처리로 다운스트림을 쉽게 압박할 수 있습니다.
따라서 adaptive 설계 이전에, 최소한 현재 동시 실행 수를 제어하는 구조가 있어야 합니다.

아래는 단순화한 개념 예시입니다.

```ts
class AdaptiveLimiter {
  private inFlight = 0;
  private limit = 32;

  constructor(
    private readonly minLimit = 8,
    private readonly maxLimit = 128,
  ) {}

  canRun() {
    return this.inFlight < this.limit;
  }

  async run<T>(task: () => Promise<T>): Promise<T> {
    if (!this.canRun()) {
      const error = new Error('ADAPTIVE_LIMIT_REACHED');
      (error as any).statusCode = 503;
      throw error;
    }

    this.inFlight += 1;
    try {
      return await task();
    } finally {
      this.inFlight -= 1;
    }
  }

  decrease() {
    this.limit = Math.max(this.minLimit, Math.floor(this.limit * 0.8));
  }

  increase() {
    this.limit = Math.min(this.maxLimit, this.limit + 1);
  }

  getCurrentLimit() {
    return this.limit;
  }
}
```

실제 서비스에서는 요청마다 바로 올리고 내리기보다, 10초~1분 단위 윈도우에서 관측값을 집계한 뒤 limit를 조절하는 편이 안정적입니다.
그래야 노이즈에 덜 흔들립니다.

### H3. AIMD처럼 줄일 때는 크게, 늘릴 때는 천천히 가는 편이 실용적이다

가장 이해하기 쉬운 방식은 **AIMD(Additive Increase, Multiplicative Decrease)** 입니다.
네트워크 제어에서 자주 쓰는 패턴인데, 서비스 운영에도 꽤 잘 맞습니다.

- 안정적일 때: limit를 1씩 천천히 증가
- 위험 신호가 보일 때: limit를 20~30% 정도 한 번에 감소

이 방식이 좋은 이유는 간단합니다.
문제가 생길 때는 빨리 보호해야 하고, 회복할 때는 다시 과도하게 밀어붙이지 않아야 하기 때문입니다.

예를 들어 현재 limit가 60일 때 상황이 안정적이면 61, 62, 63처럼 천천히 올립니다.
반대로 p99와 timeout이 튀면 60을 48이나 42 정도로 줄여 바로 압력을 낮춥니다.
이 방식은 <a href="/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html">Exponential Backoff + Jitter 가이드</a>처럼 급격한 재압박을 피하는 철학과도 잘 맞습니다.

## 어떤 요청에 adaptive limit를 먼저 적용해야 할까

### H3. 외부 의존성이 있는 고비용 경로부터 적용하는 편이 효과가 크다

처음부터 전체 API에 복잡한 동적 제한을 넣을 필요는 없습니다.
아래처럼 다운스트림 비용이 크고 변동성이 높은 경로부터 시작하는 편이 좋습니다.

- 외부 결제·인증 API를 호출하는 엔드포인트
- 무거운 집계 쿼리를 실행하는 리포트 API
- 캐시 miss 시 DB 부하가 큰 조회 경로
- 파일 처리·이미지 변환 같은 CPU/IO 혼합 작업

반대로 아주 가볍고 캐시 hit 비중이 높은 엔드포인트는 고정 limit만으로도 충분할 수 있습니다.
핵심은 **같은 제한값을 모든 엔드포인트에 똑같이 쓰지 않는 것**입니다.

### H3. 자원 격리 없이 한 풀로 묶으면 중요한 요청이 같이 흔들린다

추천 API와 주문 API가 같은 concurrency pool을 공유하면, 저우선순위 요청이 limit를 먼저 점유해 중요한 경로가 지연될 수 있습니다.
그래서 adaptive limit를 도입할 때는 엔드포인트 그룹 또는 의존성별로 풀을 나누는 것이 좋습니다.

예를 들면 아래처럼 구분할 수 있습니다.

- 결제/주문: 높은 보호, 낮은 허용 실패율
- 검색/집계: 별도 풀, 빠른 제한
- 추천/배너: 작은 풀 + fallback 우선
- 운영성 배치: 사용자 요청과 분리

이 부분은 <a href="/development/blog/seo/2026/03/24/nodejs-bulkhead-pattern-resource-isolation-resilience-guide.html">Node.js Bulkhead Pattern 가이드</a>와 함께 봐야 실전에서 더 잘 적용됩니다.
adaptive limit는 조절 장치이고, bulkhead는 섞이지 않게 막는 장치입니다.

## 운영에서 자주 하는 실수

### H3. limit 조절 주기를 너무 짧게 둬서 시스템이 출렁이는 경우

매 요청마다 limit를 바꾸면 시스템이 오히려 더 불안정해질 수 있습니다.
짧은 순간의 느려짐을 과잉 해석해 계속 limit가 오르내리면, 실제 병목보다 제어 로직 자체가 변동성을 키웁니다.

그래서 아래 원칙이 중요합니다.

- 집계 윈도우를 둔다
- 최근 N초 또는 N개 요청 기준으로 판단한다
- 증가 조건과 감소 조건을 다르게 둔다
- 최소 limit와 최대 limit를 강제한다

즉 적응형이라고 해서 실시간으로 정신없이 흔들리게 만들면 안 됩니다.
운영 가능한 범위 안에서 천천히 반응해야 합니다.

### H3. 관측 없이 limit만 자동 조정하면 원인 분석이 어려워진다

adaptive limit를 넣은 뒤 지연이 좋아졌는지, 단지 요청을 덜 받아서 숫자가 예뻐진 건지 구분이 안 되면 운영 가치가 떨어집니다.
반드시 아래 항목을 같이 기록하는 편이 좋습니다.

- 현재 limit 값
- in-flight 수
- limit 감소/증가 이벤트 횟수
- shed 또는 reject 비율
- 다운스트림 latency 변화
- 핵심 엔드포인트 성공률

이 데이터를 남겨야 "limit를 64에서 40으로 낮췄더니 p99가 3.2초에서 900ms로 내려갔다" 같은 식의 운영 판단이 가능해집니다.

## 실무 체크리스트

### H3. 도입 전에 이 항목을 먼저 정리하면 실패 확률이 줄어든다

- 어떤 다운스트림이 실제 병목인지 확인했는가
- 평균이 아니라 p95/p99와 timeout을 보고 있는가
- 엔드포인트별 중요도와 비용이 구분돼 있는가
- 고정 limit와 adaptive limit의 적용 범위를 나눴는가
- 최소/최대 limit를 안전하게 정의했는가
- 감소는 빠르게, 증가는 천천히 하도록 설계했는가
- limit 변화와 성공률 변화를 함께 관측하는가
- 과부하 시에는 load shedding이나 fallback과 연결되는가

## 마무리

Node.js adaptive concurrency limit는 단순히 병렬 요청 수를 줄이는 기법이 아닙니다.
**다운스트림이 감당할 수 있는 속도에 애플리케이션이 스스로 보폭을 맞추는 운영 방식**에 가깝습니다.

고정 concurrency limit만으로도 많은 문제를 막을 수 있지만, 실제 서비스는 시간대·기능·외부 의존성 상태에 따라 한계가 계속 바뀝니다.
그럴수록 적응형 제어가 더 잘 맞습니다.

특히 load shedding, bulkhead, retry 정책과 함께 설계하면 "많이 받다가 한 번에 무너지는 서비스"보다 "조금 덜 받아도 오래 버티는 서비스"에 가까워집니다.
장애를 완전히 없애지는 못해도, 장애 반경과 tail latency를 확실히 줄이는 데는 꽤 효과적인 선택입니다.

---
layout: post
title: "Node.js Hedged Requests 가이드: Tail Latency를 줄이되 중복 요청 비용은 통제하는 법"
date: 2026-04-08 08:00:00 +0900
lang: ko
translation_key: nodejs-hedged-requests-tail-latency-reduction-guide
permalink: /development/blog/seo/2026/04/08/nodejs-hedged-requests-tail-latency-reduction-guide.html
alternates:
  ko: /development/blog/seo/2026/04/08/nodejs-hedged-requests-tail-latency-reduction-guide.html
  x_default: /development/blog/seo/2026/04/08/nodejs-hedged-requests-tail-latency-reduction-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, hedged-requests, tail-latency, timeout, resilience, backend, distributed-systems, performance]
description: "Node.js에서 hedged requests를 적용해 tail latency를 줄이는 방법과, 중복 요청 비용·멱등성·timeout budget을 함께 관리하는 실무 패턴을 정리했습니다."
---

평균 응답 시간은 괜찮은데 p95, p99 구간만 유독 길게 튀는 서비스가 있습니다.
대부분의 요청은 빠르게 끝나지만, 일부 요청이 느린 replica·일시적 네트워크 지연·큐 적체에 걸리면서 사용자 체감 성능을 크게 망치는 경우입니다.
이때 단순 재시도만으로는 이미 느려진 뒤에야 대응하게 되고, timeout만 줄이면 정상 요청까지 같이 잘릴 수 있습니다.

이럴 때 검토할 수 있는 패턴이 **hedged requests**입니다.
핵심은 첫 요청을 보낸 뒤 일정 시간 안에 응답이 오지 않으면 **동일한 요청을 다른 경로로 한 번 더 보내고, 먼저 끝난 결과를 채택하는 것**입니다.
잘 쓰면 tail latency를 꽤 줄일 수 있지만, 잘못 쓰면 중복 트래픽과 부작용만 키웁니다.
이 글에서는 Node.js 환경에서 hedged requests가 왜 필요한지, 어디에 적용해야 하는지, 멱등성과 timeout budget을 어떻게 함께 설계해야 하는지 실무 관점에서 정리합니다.

## Node.js Hedged Requests가 필요한 이유

### H3. 평균보다 tail latency가 사용자 경험을 더 크게 망가뜨리는 경우가 많다

사용자는 평균값을 체감하지 않습니다.
검색, 추천, 결제, API 집계처럼 화면 하나를 만들기 위해 여러 내부 호출이 묶이는 시스템에서는 몇 개의 느린 요청이 전체 응답을 늦춥니다.
특히 아래 같은 상황에서 문제가 두드러집니다.

- 여러 replica 중 일부 인스턴스만 간헐적으로 느릴 때
- 캐시 미스가 특정 요청에서만 크게 튈 때
- 네트워크 hop이 많은 내부 서비스 호출이 있을 때
- 같은 API라도 백엔드 shard 상태에 따라 편차가 클 때

이런 경우 평균 latency를 조금 더 줄이는 것보다 **tail latency를 줄이는 것**이 실제 사용자 체감에는 더 중요할 수 있습니다.
hedged requests는 바로 이 꼬리 구간을 겨냥하는 패턴입니다.

### H3. 단순 재시도와 다른 점은 “느려지기 전에 병렬 대안을 준비한다”는 것이다

일반 재시도는 실패나 timeout 이후에 다음 시도를 시작합니다.
즉 이미 시간이 꽤 지난 뒤입니다.
반면 hedged requests는 완전히 실패하기 전, **느려질 조짐이 보이는 시점**에 보조 요청을 추가합니다.

예를 들어 첫 요청을 보낸 뒤 80ms 안에 응답이 없으면 다른 replica에 같은 요청을 한 번 더 보낼 수 있습니다.
그리고 둘 중 먼저 끝나는 응답을 채택합니다.
이 방식은 아래 상황에서 유리합니다.

- 특정 인스턴스 한 대만 느린 경우
- 일시적인 큐 적체가 개별 노드에만 생긴 경우
- 동일한 결과를 여러 replica 중 어디서 받아도 되는 read-heavy 요청인 경우

다만 hedged requests는 “빠른 길을 하나 더 깐다”는 발상이라서, 무분별하게 적용하면 시스템 전체 비용을 올릴 수 있습니다.
그래서 어디에 써야 하고 어디엔 쓰지 말아야 하는지 구분이 중요합니다.

## 어떤 요청에 Hedged Requests를 적용해야 할까

### H3. 읽기 요청, 멱등 요청, replica 간 결과 차이가 없는 요청에 먼저 적용하는 편이 안전하다

가장 안전한 대상은 아래와 같습니다.

- 검색 조회
- 상품 목록 조회
- 추천 후보 조회
- 캐시 가능한 API 조회
- 동일 스냅샷 기준으로 읽는 내부 read API

이런 요청은 보통 **중복 호출이 일어나도 부작용이 작고**, 먼저 온 결과를 채택해도 의미가 크게 변하지 않습니다.
특히 read replica를 여러 대 두고 있는 환경에서는 tail latency 완화 효과가 잘 드러납니다.

반대로 아래 요청은 신중해야 합니다.

- 결제 승인
- 재고 차감
- 쿠폰 발급
- 이메일 전송
- 외부 파트너 API에 비용이 발생하는 호출

이런 요청은 중복 실행 자체가 문제를 일으킬 수 있습니다.
hedged requests를 적용하려면 최소한 **멱등 키, deduplication, side effect 통제**가 먼저 준비돼 있어야 합니다.

### H3. 모든 느린 요청에 적용하기보다 “상위 일부 tail 구간”에만 제한하는 것이 좋다

hedged requests의 목표는 모든 요청을 이중화하는 것이 아닙니다.
그렇게 하면 평균은 조금 좋아질 수 있어도 비용과 부하가 훨씬 빨리 증가합니다.
실무에서는 보통 아래처럼 제한을 둡니다.

- 전체 요청 중 일부 경로에만 적용
- p95 또는 p99 기준을 넘는 구간에서만 hedge 시작
- 최대 1회의 보조 요청만 허용
- 시스템 부하가 높을 때는 hedge 자체를 비활성화

즉 hedged requests는 기본값이 아니라 **선별적 최적화 도구**에 가깝습니다.
이 점은 [Node.js Load Shedding 가이드](/development/blog/seo/2026/03/31/nodejs-load-shedding-overload-protection-guide.html)와도 연결됩니다.
시스템이 이미 과부하라면 추가 보조 요청은 구조적으로 위험할 수 있기 때문입니다.

## Node.js에서 Hedged Requests를 구현하는 기본 흐름

### H3. 첫 요청 후 짧은 hedge delay를 두고, 먼저 끝난 응답 하나만 채택한다

Node.js에서는 `AbortController`, `Promise.any`, 명시적인 deadline 관리로 기본 구조를 구현할 수 있습니다.
핵심 순서는 단순합니다.

1. 첫 요청을 보낸다
2. 정한 hedge delay 안에 응답이 없으면 두 번째 요청을 보낸다
3. 둘 중 먼저 성공한 응답을 채택한다
4. 나머지 요청은 가능한 한 빨리 취소한다

아래는 개념을 설명하기 위한 단순 예시입니다.

```js
async function fetchWithHedge(makeRequest, {
  hedgeDelayMs = 80,
  totalDeadlineMs = 300,
} = {}) {
  const startedAt = Date.now();

  const primaryController = new AbortController();
  const hedgeController = new AbortController();

  const primaryPromise = makeRequest({ signal: primaryController.signal, attempt: 'primary' });

  const hedgePromise = new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      makeRequest({ signal: hedgeController.signal, attempt: 'hedge' })
        .then(resolve)
        .catch(reject);
    }, hedgeDelayMs);

    primaryPromise.finally(() => clearTimeout(timer));
  });

  const deadlineTimer = setTimeout(() => {
    primaryController.abort();
    hedgeController.abort();
  }, totalDeadlineMs);

  try {
    const winner = await Promise.any([primaryPromise, hedgePromise]);
    primaryController.abort();
    hedgeController.abort();
    return winner;
  } finally {
    clearTimeout(deadlineTimer);
    const elapsed = Date.now() - startedAt;
    if (elapsed > totalDeadlineMs) {
      throw new Error('deadline exceeded');
    }
  }
}
```

실서비스에서는 이보다 더 많은 보호 장치가 필요하지만, 핵심 아이디어는 같습니다.
**보조 요청은 늦게 시작하고, 먼저 끝난 것만 채택하며, 패자는 즉시 취소하는 것**입니다.

### H3. hedge delay를 0으로 두면 거의 항상 과한 중복 요청이 된다

hedged requests를 처음 도입할 때 흔한 실수는 첫 요청과 두 번째 요청을 거의 동시에 보내는 것입니다.
이렇게 하면 tail latency는 줄어들 수 있지만, 사실상 모든 요청이 두 배 가까운 비용을 내게 됩니다.

hedge delay는 보통 아래 기준으로 잡습니다.

- 해당 API의 p90 또는 p95 latency 기준
- 사용자 체감 임계값보다 조금 이른 시점
- replica 간 성능 편차가 실제로 드러나는 구간

예를 들어 p95가 90ms이고 p99가 220ms라면, hedge delay를 70~100ms 수준에서 실험해볼 수 있습니다.
중요한 건 “무조건 빨리”가 아니라 **느린 요청만 골라 추가 요청을 붙이는 것**입니다.

## Hedged Requests에서 가장 중요한 건 멱등성과 중복 비용 통제다

### H3. 읽기 요청이어도 캐시, 로그, 과금, rate limit에 숨은 부작용이 있을 수 있다

겉으로 보기엔 단순 조회여도 실제로는 부작용이 숨어 있을 수 있습니다.
예를 들면 아래와 같습니다.

- 요청 1건당 외부 API 과금이 발생한다
- 조회 시 access log가 별도 비용으로 집계된다
- downstream rate limit을 빨리 소진한다
- warm-up성 쿼리나 캐시 적재 부하를 두 번 만든다

그래서 hedged requests는 단순히 “읽기니까 괜찮다”로 끝내면 안 됩니다.
적용 전에는 최소한 아래를 확인해야 합니다.

- 호출 1건당 비용 구조
- downstream의 rate limit 정책
- 중복 요청 시 캐시/DB/queue 부하 변화
- 실패 대비 취소가 실제로 잘 전달되는지 여부

특히 취소 신호가 전달되지 않는 외부 의존성이라면, 애플리케이션은 응답 한 개만 채택해도 downstream에서는 이미 **두 요청 모두 끝까지 처리**할 수 있습니다.
이 점은 반드시 관측으로 검증해야 합니다.

### H3. side effect가 있는 요청이라면 멱등 키 없이 hedge를 붙이지 않는 편이 낫다

쓰기 요청이나 외부 결제성 호출에 hedged requests를 붙이려면 멱등성 설계가 먼저입니다.
대표적으로 아래가 필요합니다.

- 요청별 idempotency key
- 서버 측 deduplication 저장소
- 중복 실행 감지 후 같은 결과 반환
- 외부 파트너와의 멱등 계약 확인

이 준비가 없다면 hedged requests는 성능 최적화가 아니라 **중복 실행 버그 생성기**가 될 수 있습니다.
이 주제는 [Node.js Retry Budget 가이드](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html)와도 밀접합니다.
재시도와 hedge는 모두 “추가 요청”을 만든다는 점에서 같은 예산 관점으로 다뤄야 합니다.

## Timeout Budget, Retry Budget과 함께 설계해야 하는 이유

### H3. hedge를 붙여도 전체 deadline은 더 짧아져야지 길어지면 안 된다

hedged requests를 도입했다고 해서 사용자 요청의 전체 timeout을 같이 늘리면 안 됩니다.
그렇게 하면 tail latency를 줄이는 대신 시스템이 더 오래 자원을 붙잡게 됩니다.
실무에서는 보통 아래 원칙을 둡니다.

- 사용자 요청 전체 deadline은 고정
- hedge delay는 그 deadline 안에서만 사용
- 첫 요청과 hedge 요청이 같은 budget을 공유
- 남은 시간이 너무 적으면 hedge 자체를 시작하지 않음

예를 들어 전체 deadline이 300ms라면, 100ms에 hedge를 시작하더라도 두 번째 요청이 새로 300ms를 또 쓰게 두면 안 됩니다.
둘 다 **같은 요청 예산 안에서 경쟁**해야 합니다.

이 점은 [Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와 함께 보면 더 명확합니다.
hedged requests는 timeout 정책의 대체제가 아니라, timeout budget 안에서 작동하는 보조 전략입니다.

### H3. retry와 hedge를 동시에 쓰면 요청 증폭이 생각보다 빨리 커질 수 있다

예를 들어 1차 요청 + hedge 1회 + 실패 후 재시도 1회를 허용하면, 한 사용자 요청이 downstream에 최대 3~4개의 호출을 만들 수 있습니다.
트래픽이 큰 서비스에서는 이 증폭이 금방 부담이 됩니다.

그래서 실무에서는 아래처럼 합산 예산을 둡니다.

- request당 추가 호출 총량 제한
- hedge와 retry를 동시에 켜지 않거나 우선순위 부여
- 과부하 시 hedge 비활성화
- 에러율 상승 시 retry 축소

핵심은 hedge와 retry를 별개 기능으로 보지 않고, **같은 추가 요청 예산을 쓰는 메커니즘**으로 보는 것입니다.

## 관측과 실험 없이 Hedged Requests를 켜면 안 되는 이유

### H3. p50보다 p95, p99, downstream QPS 변화, 취소 성공률을 같이 봐야 한다

hedged requests의 목표는 평균값 미세 개선이 아니라 tail latency 개선입니다.
따라서 아래 지표를 같이 봐야 합니다.

- p50, p90, p95, p99 latency
- hedge 시작 비율
- hedge 승리 비율
- downstream QPS 증가율
- abort 후 실제 작업 중단 비율
- error rate와 timeout rate 변화

만약 hedge 시작 비율은 높은데 승리 비율이 낮다면, delay가 너무 짧거나 구조상 효과가 적을 수 있습니다.
반대로 승리 비율은 높지만 downstream 비용이 과하게 오르면 적용 범위를 줄여야 합니다.

### H3. A/B 테스트나 단계적 rollout으로 적용 범위를 천천히 넓히는 편이 안전하다

가장 안전한 도입 방식은 전면 적용이 아니라 제한된 경로에서의 실험입니다.
예를 들면 아래 순서가 현실적입니다.

1. 읽기 전용 API 1개에만 적용
2. 트래픽 일부에만 rollout
3. p95/p99 개선과 downstream 비용 증가 비교
4. 효과가 확인되면 비슷한 read API로 확장

이 방식은 실패 비용을 작게 만들고, hedge delay나 적용 조건을 실제 데이터 기반으로 조정할 수 있게 해줍니다.
처음부터 전 서비스 공통 미들웨어로 박아두는 방식은 대개 과합니다.

## Node.js Hedged Requests 도입 체크리스트

### H3. 적용 전 체크리스트

- 이 요청은 멱등적인가
- replica 간 결과 차이가 허용 가능한가
- downstream 비용 증가를 감당할 수 있는가
- abort/cancel 신호가 실제로 전달되는가
- 전체 timeout budget 안에서 hedge가 동작하는가
- retry와 합산한 추가 요청 예산이 관리되는가

### H3. 적용 후 체크리스트

- p95, p99가 실제로 개선됐는가
- 평균 latency보다 tail latency 개선이 뚜렷한가
- downstream QPS 증가 폭이 허용 범위인가
- hedge 승리 비율이 의미 있게 나오는가
- 과부하 상황에서 자동 비활성화 장치가 있는가

## 마무리

hedged requests는 “요청을 한 번 더 보내면 빨라진다” 수준의 단순한 트릭이 아닙니다.
정확히는 **느린 꼬리 구간만 겨냥해 병렬 대안을 준비하고, 그 대신 발생하는 중복 비용을 예산 안에서 통제하는 패턴**입니다.

그래서 잘 맞는 곳에 쓰면 p95, p99를 꽤 눈에 띄게 줄일 수 있지만, 아무 데나 붙이면 rate limit·비용·중복 부작용만 키웁니다.
Node.js 서비스에서 hedged requests를 검토한다면, 먼저 읽기 요청부터 좁게 적용하고, timeout budget·retry budget·load shedding과 함께 한 세트로 설계하는 쪽이 훨씬 안전합니다.

관련해서 함께 보면 좋은 글:

- [Node.js Timeout Budget 가이드: Deadline 전파로 연쇄 타임아웃과 Tail Latency를 줄이는 법](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)
- [Node.js Retry Budget 가이드: 재시도는 늘리기보다 예산 안에서 통제해야 하는 이유](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.md)
- [Node.js Load Shedding 가이드: 과부하 상황에서 무엇을 버려야 서비스가 살아남는가](/development/blog/seo/2026/03/31/nodejs-load-shedding-overload-protection-guide.md)

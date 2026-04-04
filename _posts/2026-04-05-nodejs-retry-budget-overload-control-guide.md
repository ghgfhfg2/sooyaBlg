---
layout: post
title: "Node.js Retry Budget 가이드: 재시도로 장애를 키우지 않는 실무 설계법"
date: 2026-04-05 08:00:00 +0900
lang: ko
translation_key: nodejs-retry-budget-overload-control-guide
permalink: /development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html
alternates:
  ko: /development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html
  x_default: /development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, retry-budget, retry, overload-control, resilience, backoff, timeout, circuit-breaker]
description: "Node.js 서비스에서 retry budget을 적용해 재시도가 장애를 증폭시키지 않도록 제어하는 방법과 timeout·backoff·circuit breaker 조합 전략을 실무 관점에서 정리했습니다."
---

분산 시스템에서는 재시도가 늘 좋은 해법처럼 보입니다.
일시적인 네트워크 오류나 짧은 타임아웃에는 실제로 retry가 꽤 잘 듣습니다.
문제는 **이미 느려지거나 과부하가 시작된 시스템에 재시도를 습관처럼 붙일 때**입니다.
실패한 요청 하나가 두 번, 세 번 다시 들어가면 원래 버티고 있던 다운스트림은 더 빨리 포화되고, 상류 서비스도 연결·메모리·워커 자원을 같이 잃기 쉽습니다.

이때 필요한 개념이 **retry budget**입니다.
재시도를 아예 금지하는 것이 아니라, 시스템이 감당할 수 있는 범위 안에서만 retry를 허용하는 운영 규칙입니다.
이 글에서는 Node.js 환경에서 retry budget이 왜 필요한지, timeout·backoff·circuit breaker와 어떻게 조합해야 하는지, 그리고 실무에서 어떤 지표를 봐야 하는지 정리합니다.

## Node.js Retry Budget이 왜 필요한가

### H3. 재시도는 회복 도구이기도 하지만 트래픽 증폭기이기도 하다

retry의 위험은 실패 자체보다 **추가 요청을 자동 생성한다는 점**에 있습니다.
예를 들어 초당 1,000건 요청이 들어오는 API가 있고, 실패 시 최대 2회 재시도를 허용한다고 가정해 보겠습니다.
다운스트림이 흔들리는 순간 원래 1,000건이어야 할 부하가 2,000건, 3,000건 수준으로 불어날 수 있습니다.
이 부하는 새로운 사용자 유입이 아니라, 이미 실패한 요청이 다시 몰려드는 형태라 더 아깝습니다.

특히 아래 상황에서 retry는 문제를 더 키우기 쉽습니다.

- timeout이 지나치게 길어서 실패 판단이 늦을 때
- 모든 클라이언트가 같은 규칙으로 동시에 재시도할 때
- 백그라운드 작업과 사용자 요청이 같은 다운스트림을 두드릴 때
- 이미 queue가 쌓인 상황에서도 계속 재시도할 때

즉 retry budget은 "재시도를 얼마나 허용할지"보다, **언제부터 재시도가 위험해지는지 명시하는 장치**에 가깝습니다.

### H3. 장애 시기에 중요한 것은 성공률보다 자원 보호다

운영에서는 일시적으로 일부 요청을 포기하는 편이 전체 시스템을 살리는 경우가 많습니다.
모든 요청을 끝까지 구제하려고 하면 오히려 전체 응답성이 무너집니다.
retry budget은 바로 이 균형점을 잡습니다.

- 성공 가능성이 높은 일시 오류에는 제한적으로 재시도
- 이미 실패가 누적되는 구간에서는 빠르게 포기
- 남은 자원을 핵심 요청 경로에 우선 배분

이 철학은 [Node.js Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)에서 다룬 deadline 관리와 연결되고, [Node.js Circuit Breaker 가이드](/development/blog/seo/2026/04/04/nodejs-circuit-breaker-failure-isolation-guide.html)에서 설명한 실패 격리와도 자연스럽게 이어집니다.
결국 중요한 것은 "몇 번 더 시도할까"가 아니라 **"시스템 전체가 버틸 수 있는가"**입니다.

## Retry Budget은 무엇이고 일반 retry 정책과 무엇이 다른가

### H3. 최대 재시도 횟수는 요청 단위 규칙이고, retry budget은 시스템 단위 규칙이다

많이 쓰는 방식은 요청별로 `maxRetries = 2` 같은 값을 두는 것입니다.
이 방식은 단순하지만 시스템 상황을 잘 반영하지 못합니다.
정상 시기와 장애 시기에 같은 규칙이 적용되기 때문입니다.

반면 retry budget은 전체 성공 요청, 실패 요청, 추가 재시도 요청의 비율을 보고 **재시도 총량 자체를 제한**합니다.
예를 들면 이런 식입니다.

- 일정 시간 동안 성공 요청 1,000건당 재시도는 최대 200건까지만 허용
- 최근 윈도우에서 retry 비율이 20%를 넘으면 추가 retry 차단
- 비핵심 기능의 retry 예산을 핵심 API보다 더 낮게 설정

즉 retry budget은 "한 요청당 몇 번"보다 **"전체 시스템이 지금 얼마나 더 시도해도 되는가"**를 묻는 개념입니다.

### H3. 재시도는 공짜가 아니므로 예산처럼 다뤄야 운영 판단이 쉬워진다

budget이라는 표현이 유용한 이유는 팀이 같은 언어로 의사결정하기 쉬워지기 때문입니다.
retry를 허용할지 말지 감각적으로 정하는 대신, 다음처럼 운영 기준을 명확히 세울 수 있습니다.

- 로그인 API는 retry 1회까지만 허용
- 결제 승인처럼 중복 위험이 큰 쓰기 요청은 기본 무재시도
- 캐시 조회나 읽기 전용 내부 API는 짧은 backoff와 함께 제한적 retry 허용
- 다운스트림 에러율이 임계치를 넘으면 retry budget 즉시 축소

이렇게 하면 장애 대응 때 "왜 재시도를 끊었는가"를 설명하기 쉬워지고, 서비스별 정책 차등화도 수월해집니다.

## Node.js에서 Retry Budget을 어떻게 설계할까

### H3. 재시도 대상부터 구분해야 한다

모든 에러를 재시도 대상으로 보면 거의 반드시 부작용이 생깁니다.
Node.js 서비스에서는 먼저 **재시도 가능한 실패와 재시도하면 안 되는 실패를 분리**해야 합니다.

대체로 아래 기준이 실무에서 무난합니다.

재시도를 고려할 수 있는 경우

- 짧은 네트워크 단절
- 일시적 timeout
- 502, 503, 504 같은 일시적 서버 오류
- 멱등성이 보장된 읽기 요청

재시도하면 안 되거나 매우 신중해야 하는 경우

- 입력값 오류 같은 4xx
- 중복 실행이 치명적인 쓰기 요청
- 이미 queue 적체가 심한 상황
- circuit breaker가 열린 다운스트림

중복 쓰기 위험이 있는 API라면 [Node.js Idempotency Key 가이드](/development/blog/seo/2026/03/27/nodejs-idempotency-key-api-duplicate-request-guide.html)를 함께 보는 편이 좋습니다.
재시도보다 먼저 **중복 안전성**이 설계돼 있어야 하기 때문입니다.

### H3. timeout, backoff, jitter를 같이 둬야 retry budget이 의미가 생긴다

retry budget은 단독으로 쓰기보다 timeout과 backoff 위에 올라가는 정책이어야 합니다.
그렇지 않으면 느린 실패를 오래 기다린 뒤 다시 같은 부하를 보내는 구조가 됩니다.

실무에서는 보통 아래 조합이 안정적입니다.

1. 요청별 timeout을 짧고 명확하게 둔다
2. retry 횟수는 1~2회 수준으로 작게 시작한다
3. exponential backoff를 적용한다
4. 모든 클라이언트가 동시에 몰리지 않도록 jitter를 섞는다
5. 최근 retry 총량이 budget을 넘으면 추가 retry를 중단한다

특히 jitter가 없으면 같은 시점에 실패한 요청들이 거의 동시에 다시 들어가면서 다운스트림을 한 번 더 세게 때리게 됩니다.
retry budget은 총량을 줄이고, jitter는 **타이밍을 분산**합니다.
둘은 같이 가야 효과가 큽니다.

## Node.js Retry Budget 구현 예시

### H3. sliding window 기반으로 retry 총량을 제한하는 단순한 형태부터 시작할 수 있다

아래 예시는 개념 설명용으로, 최근 1분 윈도우에서 성공 요청 대비 retry 비율이 20%를 넘으면 추가 재시도를 막는 단순한 방식입니다.
프로덕션에서는 메트릭 저장소, 프로세스 간 공유 상태, 서비스별 정책 분리가 더 필요하지만 출발점으로는 충분합니다.

```js
class RetryBudget {
  constructor({ windowMs = 60_000, ratio = 0.2 }) {
    this.windowMs = windowMs;
    this.ratio = ratio;
    this.successEvents = [];
    this.retryEvents = [];
  }

  prune(now) {
    const min = now - this.windowMs;
    this.successEvents = this.successEvents.filter((t) => t >= min);
    this.retryEvents = this.retryEvents.filter((t) => t >= min);
  }

  recordSuccess(now = Date.now()) {
    this.prune(now);
    this.successEvents.push(now);
  }

  canRetry(now = Date.now()) {
    this.prune(now);
    const allowedRetries = Math.max(1, Math.floor(this.successEvents.length * this.ratio));
    return this.retryEvents.length < allowedRetries;
  }

  recordRetry(now = Date.now()) {
    this.prune(now);
    this.retryEvents.push(now);
  }
}

async function fetchWithRetryBudget(url, { budget, maxRetries = 2, timeoutMs = 800 }) {
  let attempt = 0;

  while (true) {
    try {
      const response = await fetch(url, {
        signal: AbortSignal.timeout(timeoutMs)
      });

      if (!response.ok && [502, 503, 504].includes(response.status)) {
        throw new Error(`retryable status: ${response.status}`);
      }

      budget.recordSuccess();
      return response;
    } catch (error) {
      if (attempt >= maxRetries || !budget.canRetry()) {
        throw error;
      }

      budget.recordRetry();
      attempt += 1;

      const backoffMs = Math.min(100 * 2 ** attempt, 1000);
      const jitterMs = Math.floor(Math.random() * 100);
      await new Promise((resolve) => setTimeout(resolve, backoffMs + jitterMs));
    }
  }
}
```

이 예시의 핵심은 세 가지입니다.

- retry 여부를 요청 하나의 의지만으로 결정하지 않음
- 성공 요청 규모에 비례해 retry 허용량을 조정함
- budget을 넘으면 더 버티기보다 빠르게 실패함

즉 retry budget은 성공률 집착보다 **시스템 보호를 우선하는 정책 레이어**라고 볼 수 있습니다.

### H3. 서비스별·기능별로 예산을 분리해야 한쪽 폭주가 전체 정책을 오염시키지 않는다

retry budget을 전역 하나로만 두면 의도치 않은 문제가 생길 수 있습니다.
예를 들어 비핵심 추천 API의 retry가 많아졌다는 이유로 결제나 로그인 같은 핵심 경로의 retry까지 함께 막히면 곤란합니다.
그래서 보통은 아래처럼 분리하는 편이 낫습니다.

- 다운스트림별 budget 분리
- 읽기/쓰기 경로 budget 분리
- 핵심 기능/비핵심 기능 budget 분리
- 사용자 요청/배치 작업 budget 분리

이 구조는 [Node.js Bulkhead Pattern 가이드](/development/blog/seo/2026/03/29/nodejs-bulkhead-pattern-resource-isolation-guide.html)와 잘 맞습니다.
리소스만 분리할 것이 아니라, **재시도 예산도 같이 분리**해야 진짜 격리가 됩니다.

## Circuit Breaker, Concurrency Limit과는 어떻게 같이 써야 할까

### H3. retry budget은 실패 증폭을 줄이고, circuit breaker는 위험 구간 자체를 차단한다

두 패턴은 비슷해 보이지만 초점이 다릅니다.

- retry budget: 재시도 총량을 제한해 트래픽 증폭 방지
- circuit breaker: 실패 구간을 감지하면 호출 자체를 잠시 차단
- concurrency limit: 동시에 붙는 실행 수를 제한해 자원 포화 방지

실무에서는 셋을 따로 두기보다 함께 설계하는 편이 안정적입니다.
예를 들어 다운스트림이 느려지는 상황이라면 다음 순서가 자연스럽습니다.

1. timeout으로 오래 붙잡히는 요청을 정리
2. concurrency limit으로 동시에 붙는 수를 제한
3. retry budget으로 추가 요청 증폭을 제어
4. 실패가 계속되면 circuit breaker를 열어 fast-fail 전환

이 흐름은 [Node.js Concurrency Limit 가이드](/development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)와도 연결됩니다.
즉 retry를 잘 설계하는 일은 retry 코드만 손보는 문제가 아니라, **과부하 제어 체계 전체를 연결하는 일**입니다.

### H3. queue가 이미 길어졌다면 retry보다 드롭 또는 지연이 더 나을 수 있다

운영에서 놓치기 쉬운 포인트는, queue 적체가 심한데도 retry를 계속 허용하는 경우입니다.
이때 재시도는 복구 전략이 아니라 대기열 생성기 역할을 합니다.
이미 처리 지연이 심하다면 아래 선택지가 더 낫습니다.

- 비핵심 요청은 빠르게 실패
- 백그라운드 작업은 지연 실행
- stale cache 또는 fallback 응답 사용
- 사용량이 큰 테넌트는 별도 rate limit 적용

즉 시스템 상태가 안 좋을수록 "한 번 더"보다 **"지금은 그만"**이 더 건강한 결정일 수 있습니다.

## 운영하면서 꼭 봐야 할 지표

### H3. 단순 retry 횟수보다 retry가 만든 추가 트래픽 비율을 봐야 한다

retry budget을 운영한다면 최소한 아래 지표는 함께 보는 편이 좋습니다.

- 전체 요청 대비 retry 요청 비율
- 다운스트림별 retry 성공률
- retry 후 최종 성공까지 걸린 추가 지연 시간
- timeout 비율과 queue depth 변화
- circuit breaker open 횟수

이 지표를 보면 "retry가 실제로 회복에 기여하는지"와 "단지 부하만 늘리는지"를 구분하기 쉬워집니다.
retry 성공률이 낮고 지연만 커진다면 예산을 줄이거나 정책 자체를 바꿔야 합니다.

### H3. 정상 시기와 장애 시기의 기준선을 따로 관리해야 한다

정상 시기에는 retry 1회가 꽤 잘 먹히는 서비스도, 장애 시기에는 같은 정책이 독이 될 수 있습니다.
그래서 운영 기준은 고정값 하나보다 **상태 기반 조정**이 더 낫습니다.
예를 들면 이런 식입니다.

- 에러율 상승 시 retry budget 자동 축소
- latency 급증 시 timeout과 retry 동시 강화
- 배치 작업 시간대에는 비핵심 retry budget 축소
- 특정 다운스트림 장애 시 관련 기능만 fast-fail 전환

이런 정책이 있어야 retry가 단순 습관이 아니라 **상황 인식형 제어 장치**가 됩니다.

## 결론

Node.js 서비스에서 retry는 필요하지만, 무조건 많을수록 좋은 장치는 아닙니다.
특히 과부하와 부분 장애가 시작된 순간에는 retry가 복구보다 증폭 역할을 하기가 더 쉽습니다.
그래서 실무에서는 최대 재시도 횟수만 보는 것보다, **retry budget으로 전체 허용량을 제어하고 timeout·backoff·concurrency limit·circuit breaker를 함께 설계하는 접근**이 훨씬 안정적입니다.

정리하면 아래 네 가지를 먼저 챙기면 됩니다.

1. 재시도 가능한 실패와 금지해야 할 실패를 구분하기
2. timeout, backoff, jitter 없이 retry만 두지 않기
3. 시스템 전체 기준의 retry budget을 운영 지표와 함께 관리하기
4. 핵심/비핵심 경로와 다운스트림별로 예산을 분리하기

재시도는 친절한 안전장치가 될 수도 있고, 장애를 키우는 증폭기가 될 수도 있습니다.
차이는 보통 retry 코드 한 줄보다 **예산을 두고 절제하는 설계**에서 갈립니다.

## FAQ

### H3. retry budget이 있으면 max retry는 필요 없을까

아닙니다.
보통은 둘 다 필요합니다.
`max retry`는 개별 요청의 상한이고, retry budget은 시스템 전체의 상한입니다.
둘 중 하나만 있으면 보호 범위가 좁아집니다.

### H3. 쓰기 API에도 retry budget을 적용해도 될까

가능은 하지만 매우 신중해야 합니다.
결제, 주문 생성, 포인트 적립처럼 중복 실행 위험이 있는 요청은 idempotency key 같은 안전장치가 먼저 필요합니다.
그 전에는 무재시도 정책이 더 안전한 경우가 많습니다.

### H3. retry budget 비율은 몇 퍼센트가 적당할까

정답은 서비스마다 다르지만, 처음에는 보수적으로 시작하는 편이 좋습니다.
예를 들어 성공 요청 대비 10~20% 수준에서 시작하고, 실제 retry 성공률과 지연 비용을 보면서 조정하는 방식이 현실적입니다.

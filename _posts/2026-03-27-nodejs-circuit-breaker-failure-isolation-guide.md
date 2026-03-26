---
layout: post
title: "Node.js Circuit Breaker 패턴 가이드: 장애 전파를 막고 복구를 더 빠르게 만드는 법"
date: 2026-03-27 08:00:00 +0900
lang: ko
translation_key: nodejs-circuit-breaker-failure-isolation-guide
permalink: /development/blog/seo/2026/03/27/nodejs-circuit-breaker-failure-isolation-guide.html
alternates:
  ko: /development/blog/seo/2026/03/27/nodejs-circuit-breaker-failure-isolation-guide.html
  x_default: /development/blog/seo/2026/03/27/nodejs-circuit-breaker-failure-isolation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, circuit-breaker, resilience, backend, fault-tolerance, performance]
description: "Node.js 서비스에서 circuit breaker 패턴을 적용해 장애 전파를 막고 timeout 폭증을 줄이는 실무 설계 방법을 정리했습니다."
---

Node.js 서비스는 평소에는 빠르게 보이지만, 외부 API나 내부 마이크로서비스 하나가 흔들리기 시작하면 생각보다 빠르게 전체가 느려질 수 있습니다.
특히 실패한 요청을 계속 같은 방식으로 재시도하거나, timeout이 길게 쌓이는 구조라면 문제는 한 서비스에서 끝나지 않고 **장애 전파**로 번집니다.
이럴 때 필요한 대표적인 보호 장치가 **circuit breaker 패턴**입니다.
문제가 생긴 다운스트림으로의 호출을 잠시 끊고, 시스템이 회복할 시간을 벌어주는 방식입니다.
이 글에서는 Node.js circuit breaker가 왜 필요한지, timeout·retry·fallback과 어떤 관계인지, 실무에서 어떤 기준으로 open/half-open/closed 상태를 운영하면 좋은지 정리합니다.

## Node.js circuit breaker 패턴은 왜 필요한가

### H3. 느린 실패는 빠른 실패보다 훨씬 비싸다

많은 팀이 에러 자체보다 **느리게 실패하는 요청**의 파괴력을 과소평가합니다.
예를 들어 외부 결제 API가 정상 응답 대신 8초 timeout을 반복하면, 애플리케이션은 그 8초 동안 connection, 메모리, worker, 사용자 대기 시간을 모두 붙잡히게 됩니다.
한두 건은 버틸 수 있지만 트래픽이 붙으면 in-flight 요청이 쌓이고, 결국 다른 정상 경로까지 같이 느려집니다.

circuit breaker는 이런 상황에서 아래 역할을 합니다.

- 실패율이나 timeout이 임계치를 넘으면 호출을 잠시 차단
- 더 많은 요청이 같은 실패를 반복하지 않게 보호
- 다운스트림이 회복할 시간을 확보
- 상위 서비스는 fallback 또는 명확한 에러 응답으로 빠르게 종료

즉 핵심은 실패를 숨기는 것이 아니라 **비싼 실패를 짧고 통제 가능한 실패로 바꾸는 것**입니다.

### H3. retry만 늘리면 복구가 아니라 압박이 되는 경우가 많다

장애 상황에서 흔히 하는 실수는 retry 횟수를 늘리는 것입니다.
정상 상태에서는 retry가 유용할 수 있습니다.
하지만 다운스트림이 이미 포화 상태라면 재시도는 복구가 아니라 추가 압박이 됩니다.

예를 들어 요청 1건이 실패할 때마다 2번 더 retry하면, 원래 100건이던 호출이 순식간에 300건까지 불어날 수 있습니다.
이때 circuit breaker가 없으면 시스템은 "지금 멈춰야 하는지" 판단하지 못한 채 계속 두드립니다.

이 문제는 이전에 다룬 <a href="/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html">Node.js Exponential Backoff + Jitter 가이드</a>와 함께 봐야 합니다.
retry는 필요하지만, circuit breaker 없이 쓰면 장애 전파를 키울 수 있습니다.

## circuit breaker의 상태는 어떻게 동작할까

### H3. Closed, Open, Half-Open 세 상태를 이해하면 설계가 쉬워진다

circuit breaker는 보통 세 가지 상태로 설명합니다.

1. **Closed**: 평소 상태로 요청을 정상 전달
2. **Open**: 실패가 임계치를 넘어서 요청을 즉시 차단
3. **Half-Open**: 일부 요청만 시험적으로 통과시켜 회복 여부를 확인

Closed 상태에서는 평소처럼 다운스트림을 호출합니다.
하지만 일정 기간 동안 실패율, timeout 비율, 5xx 비율이 높아지면 Open 상태로 전환합니다.
Open 상태에서는 더 이상 모든 요청을 보내지 않고 즉시 실패 또는 fallback으로 처리합니다.
그리고 일정 시간이 지나면 Half-Open 상태로 바뀌어, 소수의 탐침 요청만 보내봅니다.
이 요청들이 안정적으로 성공하면 다시 Closed로 복귀하고, 다시 실패하면 Open으로 돌아갑니다.

이 구조의 장점은 단순합니다.
장애가 발생했을 때 모두가 동시에 "괜찮아졌나?" 하고 몰려들지 않게 만든다는 점입니다.

### H3. failure count만 보지 말고 시간 창 기반으로 판단하는 편이 안전하다

간단한 구현은 연속 실패 횟수만 세서 breaker를 엽니다.
하지만 실무에서는 시간 창(window) 기반으로 보는 편이 더 낫습니다.
연속 실패만 보면 트래픽이 낮은 서비스에서는 너무 둔감하고, 트래픽이 높은 서비스에서는 너무 민감할 수 있기 때문입니다.

보통 아래 조합이 더 현실적입니다.

- 최근 30초 또는 60초 요청 수
- 같은 구간의 실패율
- timeout 비율
- 특정 상태 코드 비율(예: 502, 503, 504)
- 최소 샘플 수

예를 들어 최근 1분 동안 100건 이상 호출이 있었고, 그중 50% 이상이 timeout 또는 5xx라면 Open으로 전환하는 식입니다.
이렇게 해야 일시적 노이즈와 구조적 장애를 조금 더 구분할 수 있습니다.

## Node.js에서 circuit breaker를 언제 적용해야 할까

### H3. 외부 API, DB 앞단보다 변동성이 큰 의존성부터 적용하는 편이 좋다

모든 함수 호출에 breaker를 둘 필요는 없습니다.
실제로 효과가 큰 곳부터 적용하는 편이 낫습니다.
대표적으로 아래 경로가 우선순위가 높습니다.

- 외부 결제, 인증, 메시징 API
- 다른 팀이 운영하는 내부 마이크로서비스 호출
- 응답 지연 편차가 큰 추천/검색 API
- 순간 장애 시 timeout 누적이 큰 I/O 경로

반대로 매우 안정적이고 짧게 끝나는 로컬 연산이나 메모리 조회에는 breaker가 과할 수 있습니다.
핵심은 **실패가 느리고, 자주 전파되며, 상위 경로에 큰 비용을 만드는 의존성**에 먼저 적용하는 것입니다.

### H3. 엔드포인트별 중요도와 fallback 가능성을 같이 봐야 한다

같은 외부 API라도 어떤 요청은 실패 시 대체 응답이 가능하고, 어떤 요청은 그렇지 않습니다.
예를 들어 추천 배너는 fallback으로 빈 리스트를 주거나 이전 캐시를 보여줄 수 있지만, 결제 승인 API는 그렇게 처리하면 안 됩니다.

그래서 breaker를 붙일 때는 아래를 함께 정리해야 합니다.

- 이 의존성이 실패하면 사용자 경험이 어떻게 바뀌는가
- 빠른 실패가 나은가, 짧은 대기가 나은가
- fallback이 가능한가
- stale cache나 기본값으로 대체 가능한가
- 장애 시 이 기능을 비활성화해도 되는가

이 판단이 있어야 breaker가 단순한 차단기가 아니라 **서비스 정책의 일부**가 됩니다.

## Node.js circuit breaker를 어떻게 구현할까

### H3. timeout, semaphore, metrics를 함께 묶어야 실전에서 잘 버틴다

breaker는 혼자만으로 완전하지 않습니다.
Node.js에서 실무적으로 쓰려면 최소한 아래 세 가지와 함께 가야 합니다.

- 호출 timeout
- 동시 실행 수 제어(semaphore 또는 pool)
- 메트릭 수집

timeout이 없으면 실패를 늦게 감지하고, concurrency 제어가 없으면 Open 전까지 이미 과부하가 누적될 수 있습니다.
메트릭이 없으면 breaker가 너무 자주 열리는지, 실제로 tail latency를 줄였는지 판단하기 어렵습니다.

아래는 개념을 단순화한 예시입니다.

```ts
interface CircuitWindow {
  total: number;
  failures: number;
  timeouts: number;
}

type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

class CircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private openedAt = 0;
  private halfOpenInFlight = 0;
  private window: CircuitWindow = { total: 0, failures: 0, timeouts: 0 };

  constructor(
    private readonly failureThreshold = 0.5,
    private readonly minSamples = 20,
    private readonly openDurationMs = 15000,
    private readonly halfOpenMaxRequests = 3,
  ) {}

  async execute<T>(task: () => Promise<T>): Promise<T> {
    this.refreshStateByTime();

    if (this.state === 'OPEN') {
      throw new Error('CIRCUIT_OPEN');
    }

    if (this.state === 'HALF_OPEN' && this.halfOpenInFlight >= this.halfOpenMaxRequests) {
      throw new Error('CIRCUIT_HALF_OPEN_LIMIT');
    }

    if (this.state === 'HALF_OPEN') {
      this.halfOpenInFlight += 1;
    }

    this.window.total += 1;

    try {
      const result = await task();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure(error);
      throw error;
    } finally {
      if (this.state === 'HALF_OPEN' && this.halfOpenInFlight > 0) {
        this.halfOpenInFlight -= 1;
      }
    }
  }

  private onSuccess() {
    if (this.state === 'HALF_OPEN') {
      this.state = 'CLOSED';
      this.resetWindow();
    }
  }

  private onFailure(error: unknown) {
    this.window.failures += 1;

    if ((error as Error)?.name === 'TimeoutError') {
      this.window.timeouts += 1;
    }

    const failureRate = this.window.total > 0 ? this.window.failures / this.window.total : 0;

    if (this.window.total >= this.minSamples && failureRate >= this.failureThreshold) {
      this.state = 'OPEN';
      this.openedAt = Date.now();
    }
  }

  private refreshStateByTime() {
    if (this.state === 'OPEN' && Date.now() - this.openedAt >= this.openDurationMs) {
      this.state = 'HALF_OPEN';
      this.halfOpenInFlight = 0;
      this.resetWindow();
    }
  }

  private resetWindow() {
    this.window = { total: 0, failures: 0, timeouts: 0 };
  }
}
```

실서비스에서는 요청 단위 윈도우 집계, 라벨별 메트릭, 에러 분류, fallback 정책을 더 명확히 분리하는 편이 좋습니다.
하지만 위 구조만 이해해도 breaker의 핵심 동작은 충분히 잡힙니다.

### H3. Half-Open에서는 탐침 요청 수를 작게 유지해야 한다

실무에서 많이 놓치는 부분이 Half-Open 상태입니다.
Open 상태가 끝났다고 갑자기 모든 요청을 다시 풀어주면, 방금 회복하려던 다운스트림을 다시 눌러버릴 수 있습니다.
그래서 Half-Open에서는 아주 적은 수의 요청만 통과시키는 것이 중요합니다.

예를 들어 아래처럼 운영할 수 있습니다.

- 10초 또는 15초 동안 Open 유지
- 이후 Half-Open으로 전환
- 1~3개 정도의 탐침 요청만 통과
- 이 요청들이 일정 횟수 연속 성공하면 Closed 복귀
- 다시 실패하면 즉시 Open 재전환

이 접근은 어제 다룬 <a href="/development/blog/seo/2026/03/26/nodejs-adaptive-concurrency-limit-downstream-protection-guide.html">Node.js Adaptive Concurrency Limit 가이드</a>와도 잘 연결됩니다.
adaptive concurrency가 전체 압력을 낮추는 역할이라면, circuit breaker는 아예 위험한 호출을 잠시 끊는 역할에 가깝습니다.

## circuit breaker와 fallback은 어떻게 같이 써야 할까

### H3. breaker는 차단 장치이고 fallback은 사용자 경험 완화 장치다

breaker만 두면 장애 전파는 줄일 수 있지만, 사용자는 여전히 에러를 볼 수 있습니다.
그래서 fallback과 함께 설계하는 경우가 많습니다.

예를 들어 아래 조합이 가능합니다.

- 추천 API 실패 시: 빈 리스트 또는 캐시된 추천값 반환
- 통계 API 실패 시: 마지막 성공 스냅샷 반환
- 개인화 배너 실패 시: 기본 배너 반환
- 외부 환율 API 실패 시: 최근 캐시값 + 시각 표시

다만 fallback이 있다고 해서 아무 기능에나 붙이면 안 됩니다.
결제, 주문 확정, 권한 확인처럼 **틀리면 안 되는 경로**에서는 잘못된 fallback이 더 위험할 수 있습니다.
이때는 명확한 에러와 빠른 실패가 오히려 안전합니다.

### H3. stale cache와 연결하면 읽기 경로 복원력이 더 좋아진다

읽기 중심 경로에서는 breaker와 stale cache를 함께 쓰는 조합이 강력합니다.
다운스트림이 잠시 죽었을 때 breaker가 불필요한 재호출을 막고, 애플리케이션은 예전 캐시라도 빠르게 응답할 수 있기 때문입니다.

이 패턴은 <a href="/development/blog/seo/2026/03/26/nodejs-stale-while-revalidate-cache-strategy-guide.html">Node.js Stale-While-Revalidate 캐시 전략 가이드</a>와 자연스럽게 이어집니다.
즉 "새 데이터를 못 받아오면 오래된 값이라도 잠시 보여주고, 시스템이 회복되면 다시 갱신한다"는 구조입니다.
이렇게 하면 장애가 곧바로 빈 화면이나 전체 timeout으로 번지는 일을 꽤 줄일 수 있습니다.

## 운영에서 자주 하는 실수

### H3. 모든 4xx, 5xx를 똑같이 failure로 처리하는 경우

breaker는 "다운스트림 장애"를 감지해야지, 모든 비정상 응답을 무조건 같은 실패로 취급하면 안 됩니다.
예를 들어 잘못된 요청으로 인한 400, 404까지 breaker failure에 포함하면 실제 장애가 아닌데도 회로가 열릴 수 있습니다.

보통은 아래처럼 구분합니다.

- 5xx, network error, timeout: breaker failure 후보
- 429: 상황에 따라 별도 정책 적용
- 4xx: 대개 호출 자체의 문제로 보고 breaker failure에서 제외

어떤 상태 코드를 failure로 볼지 명확히 하지 않으면 오탐이 많아집니다.

### H3. Open 시간과 timeout 값을 감으로만 정하면 튜닝이 길어진다

timeout이 너무 길면 breaker가 늦게 반응하고, Open 시간이 너무 짧으면 회복 전에 계속 두드리게 됩니다.
반대로 Open 시간이 너무 길면 이미 회복된 시스템도 오래 차단하게 됩니다.

그래서 아래 데이터를 먼저 보고 시작하는 편이 좋습니다.

- 정상 시 p95, p99 latency
- 장애 시 timeout 비율 상승 패턴
- 다운스트림 평균 복구 시간
- 요청 중요도와 허용 대기 시간

예를 들어 정상 p99가 300ms인데 timeout을 10초로 두는 경우는 대체로 과합니다.
보통은 사용자 기대 시간과 다운스트림 특성에 맞춰 훨씬 짧게 잡고, breaker가 그 짧은 실패를 바탕으로 빠르게 반응하게 만드는 편이 낫습니다.

## 실무 체크리스트

### H3. circuit breaker 도입 전 이 항목부터 정리해두면 좋다

- 어떤 의존성이 장애 전파를 가장 크게 만드는가
- timeout 기준이 실제 사용자 기대 시간에 맞는가
- failure로 볼 상태 코드와 에러 유형이 정의돼 있는가
- Open, Half-Open, Closed 전환 기준이 있는가
- Half-Open 탐침 요청 수가 충분히 작은가
- fallback 가능한 경로와 절대 불가능한 경로가 구분돼 있는가
- 현재 breaker open 횟수와 duration을 관측하는가
- retry, load shedding, adaptive concurrency와 역할 분담이 돼 있는가

## FAQ

### H3. circuit breaker가 있으면 retry는 필요 없을까

아닙니다.
retry 자체는 여전히 유용할 수 있습니다.
다만 **짧은 timeout, 제한된 retry, backoff + jitter, 그리고 breaker**를 함께 써야 안전합니다.
즉 retry는 일시적 흔들림 복구용이고, breaker는 구조적 장애 확산 방지용이라고 보는 편이 맞습니다.

### H3. 라이브러리를 써야 할까, 직접 구현해도 될까

팀 규모와 복잡도에 따라 다릅니다.
작은 서비스라면 직접 구현도 가능합니다.
다만 다수의 의존성, 정교한 메트릭, 세밀한 상태 전환이 필요하면 검증된 라이브러리나 공통 모듈로 묶는 편이 운영에 유리합니다.
중요한 것은 라이브러리 사용 여부보다 **전환 기준과 failure 분류가 팀 안에서 명확한가**입니다.

## 마무리

Node.js circuit breaker 패턴은 단지 에러를 빨리 내는 기법이 아닙니다.
**망가진 의존성을 계속 두드리지 않음으로써 전체 시스템이 같이 무너지는 것을 막는 운영 장치**에 가깝습니다.

특히 timeout, retry, fallback, adaptive concurrency와 함께 설계하면 장애가 생겼을 때 "모두가 같이 느려지는 서비스"에서 "문제가 있는 부분만 빠르게 격리되는 서비스"로 한 단계 올라갈 수 있습니다.
장애를 완전히 없애지는 못해도, 장애 반경과 tail latency를 줄이는 데는 매우 실용적인 패턴입니다.

---
layout: post
title: "Node.js Circuit Breaker 가이드: 장애 전파를 끊고 다운스트림 실패를 격리하는 방법"
date: 2026-04-04 20:00:00 +0900
lang: ko
translation_key: nodejs-circuit-breaker-failure-isolation-guide
permalink: /development/blog/seo/2026/04/04/nodejs-circuit-breaker-failure-isolation-guide.html
alternates:
  ko: /development/blog/seo/2026/04/04/nodejs-circuit-breaker-failure-isolation-guide.html
  x_default: /development/blog/seo/2026/04/04/nodejs-circuit-breaker-failure-isolation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, circuit-breaker, resilience, fault-tolerance, failure-isolation, timeout, retry, overload-control]
description: "Node.js 서비스에서 circuit breaker를 적용해 느린 외부 API와 장애 전파를 차단하고, timeout·retry·fallback을 함께 설계하는 실무 방법을 정리했습니다."
---

외부 API나 내부 마이크로서비스를 호출하는 Node.js 서비스는 보통 정상일 때보다 **부분 장애가 시작될 때** 더 위험합니다.
성공률이 100%에서 0%로 바로 떨어지면 오히려 눈치채기 쉽지만, 실제 운영에서는 응답이 조금씩 느려지고 일부 요청만 실패하면서 시스템 전체가 같이 지치는 경우가 많습니다.
이때 타임아웃은 길어지고, 재시도는 늘어나며, 워커와 커넥션 풀이 낭비되고, 결국 원래는 멀쩡하던 상류 서비스까지 같이 흔들릴 수 있습니다.

이런 상황에서 중요한 장치가 **circuit breaker**입니다.
전기 차단기처럼 다운스트림이 비정상일 때 호출을 계속 밀어 넣지 않고, 일정 조건에서 **일시적으로 호출을 끊어 장애 전파를 막는 패턴**입니다.
이 글에서는 Node.js 환경에서 circuit breaker가 왜 필요한지, timeout·retry와 어떤 관계인지, 그리고 운영에서 어떻게 튜닝해야 하는지 실무 관점에서 정리합니다.

## Node.js Circuit Breaker가 왜 필요한가

### H3. 느린 실패는 빠른 실패보다 더 비싸다

다운스트림이 완전히 죽은 상황보다 더 곤란한 경우는 느리게 실패하는 상황입니다.
예를 들어 평소 150ms 안에 끝나던 외부 API가 갑자기 3초, 5초씩 걸리기 시작하면 요청 하나하나가 오래 붙잡히게 됩니다.
그 사이 Node.js 프로세스는 promise와 소켓, 메모리, 워커 자원을 계속 점유합니다.
그리고 상위 계층에서는 timeout 이후 retry를 시작하면서 같은 문제를 더 크게 만듭니다.

이때 문제는 단순한 에러율이 아닙니다.

- 응답 지연이 전체 요청 경로로 전파됨
- 커넥션과 실행 슬롯이 느린 호출에 묶임
- retry가 유입량을 더 키움
- 정상 요청도 같이 늦어짐
- 장애 원인이 한 서비스였는데 전체 시스템 문제처럼 번짐

즉 circuit breaker는 "호출을 포기하는 장치"라기보다, **비정상 구간에서 정상 자원을 보호하는 장치**에 가깝습니다.
이 점은 [Node.js Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)에서 설명한 deadline 관리와도 직접 연결됩니다.
타임아웃만 두고 계속 재도전하면 보호가 약하고, circuit breaker까지 있어야 호출 단위가 아니라 시스템 단위로 방어가 됩니다.

### H3. 다운스트림 장애를 감지해도 계속 호출하면 실패가 누적된다

운영에서는 종종 "에러가 나면 어차피 로그에 보이니까 괜찮다"고 생각하기 쉽습니다.
하지만 감지와 제어는 다른 문제입니다.
다운스트림 장애를 이미 알고 있어도 모든 요청이 계속 같은 실패 경로를 밟게 두면, 그 실패 자체가 비용이 됩니다.

예를 들어 결제 부가 API, 추천 API, 알림 API 중 하나가 느려졌다고 가정해 보겠습니다.
그런데 상위 서비스가 매 요청마다 여전히 그 API를 호출한다면, 실패를 아는 상태에서도 다음 비용이 반복됩니다.

- 매 호출마다 timeout 대기
- 실패 로그와 메트릭 폭증
- 재시도 정책에 따른 중복 부하
- 사용자 응답 지연 증가

circuit breaker는 이런 불필요한 반복을 끊습니다.
실패율이나 지연이 임계치를 넘으면 breaker를 열고, 일정 시간 동안은 **실제 호출 대신 즉시 실패 혹은 fallback**으로 전환합니다.

## Circuit Breaker와 Timeout, Retry는 무엇이 다를까

### H3. timeout은 한 번의 호출을 제한하고, circuit breaker는 실패 구간 전체를 제한한다

timeout은 개별 요청이 얼마나 오래 기다릴지 정합니다.
반면 circuit breaker는 최근 호출들이 일정 수준 이상 실패하거나 느려졌을 때, 그 구간 전체를 위험 상태로 보고 호출 자체를 멈춥니다.

- timeout: 요청 1건이 오래 걸리지 않게 제한
- circuit breaker: 최근 연속 실패/에러율/지연을 기준으로 호출 경로를 차단

둘 중 하나만으로는 부족한 경우가 많습니다.
타임아웃만 있으면 느린 실패를 계속 반복할 수 있고, circuit breaker만 있으면 개별 요청이 너무 오래 붙잡힐 수 있습니다.
실무에서는 둘을 함께 둬야 합니다.

### H3. retry는 회복된 시스템에 도움이 되지만, 망가진 시스템에는 독이 될 수 있다

retry는 일시적 네트워크 오류나 드문 타임아웃에는 유용합니다.
하지만 다운스트림이 이미 과부하 상태라면 retry는 문제를 해결하기보다 확대하기 쉽습니다.
같은 요청을 한 번 더 보내는 순간, 상대 시스템과 내 시스템 모두에 추가 비용이 생기기 때문입니다.

그래서 일반적으로는 아래 순서가 안정적입니다.

1. 짧고 명확한 timeout 설정
2. 제한된 retry 횟수와 backoff 적용
3. 실패가 누적되면 circuit breaker open
4. open 상태에서는 fallback 또는 fast-fail

이 흐름은 [Node.js Load Shedding 가이드](/development/blog/seo/2026/03/31/nodejs-load-shedding-overload-protection-guide.html), [Node.js Admission Control 가이드](/development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html)와도 이어집니다.
입구에서 너무 많이 받지 않는 것과, 위험한 다운스트림을 계속 두드리지 않는 것은 결국 같은 운영 철학입니다.

## Node.js Circuit Breaker 상태는 어떻게 동작하나

### H3. closed, open, half-open 세 상태를 이해하면 구조가 단순해진다

대부분의 circuit breaker는 아래 세 상태를 중심으로 동작합니다.

- **closed**: 정상 상태. 요청을 그대로 보냄
- **open**: 차단 상태. 일정 시간 동안 실제 호출을 막음
- **half-open**: 시험 상태. 일부 요청만 통과시켜 회복 여부를 확인함

흐름은 보통 이렇습니다.

1. 평소에는 closed 상태로 요청을 보냄
2. 실패율, 연속 실패 수, 지연 시간 등이 임계치를 넘으면 open으로 전환
3. open 동안은 즉시 실패시키거나 fallback 수행
4. cool-down 시간이 지나면 half-open으로 전환
5. 소수 요청이 성공하면 다시 closed, 실패하면 다시 open

핵심은 단순합니다.
**"지금은 계속 시도할 때가 아니라, 잠깐 멈추고 상태를 확인할 때"를 코드로 명시하는 것**입니다.

### H3. half-open에서 트래픽을 너무 많이 열면 다시 무너질 수 있다

half-open은 회복을 확인하는 단계이지, 즉시 정상 복귀를 선언하는 단계가 아닙니다.
여기서 흔한 실수는 대기 중이던 요청을 한꺼번에 다시 흘려보내는 것입니다.
그러면 아직 덜 회복된 다운스트림이 다시 무너질 수 있습니다.

그래서 half-open에서는 보통 아래 같은 제약이 필요합니다.

- 시험 요청 수 제한
- 짧은 관찰 윈도우 설정
- 성공률 기준 충족 시에만 closed 복귀
- 다시 느려지면 즉시 open 재전환

즉 half-open은 "조심스럽게 문을 열어 보는 상태"여야 합니다.
완전한 회복 전까지는 점진적으로 확인하는 편이 훨씬 안전합니다.

## Node.js에서 Circuit Breaker를 어떻게 구현할까

### H3. 가장 먼저 해야 할 일은 실패 기준을 코드 밖의 운영 개념으로 정의하는 것이다

구현보다 먼저 정해야 할 것은 "언제 실패로 볼 것인가"입니다.
단순 500 에러만 세면 부족한 경우가 많습니다.
실무에서는 아래 항목을 함께 보게 됩니다.

- timeout 발생 비율
- HTTP 5xx 비율
- 네트워크 오류 수
- 최근 N건 중 실패 비율
- p95, p99 지연 급증 여부

예를 들어 외부 API가 200을 주더라도 8초가 걸리면 서비스에는 사실상 실패일 수 있습니다.
따라서 circuit breaker의 실패 정의는 단순 상태코드가 아니라 **사용자 경험과 자원 보호 기준**에 맞춰야 합니다.

### H3. 작은 래퍼부터 시작해도 효과는 충분하다

아래는 개념 설명용 단순 예시입니다.
프로덕션에서는 메트릭, 분산 환경, 세밀한 윈도우 계산을 더 보강해야 하지만, 동작 원리는 이 정도로 이해할 수 있습니다.

```js
class SimpleCircuitBreaker {
  constructor({ failureThreshold = 5, cooldownMs = 10000, requestTimeoutMs = 1500 }) {
    this.failureThreshold = failureThreshold;
    this.cooldownMs = cooldownMs;
    this.requestTimeoutMs = requestTimeoutMs;
    this.failures = 0;
    this.state = 'CLOSED';
    this.nextAttemptAt = 0;
  }

  async execute(task, fallback) {
    const now = Date.now();

    if (this.state === 'OPEN') {
      if (now < this.nextAttemptAt) {
        if (fallback) return fallback();
        const error = new Error('Circuit is open');
        error.code = 'CIRCUIT_OPEN';
        throw error;
      }

      this.state = 'HALF_OPEN';
    }

    try {
      const result = await Promise.race([
        task(),
        new Promise((_, reject) =>
          setTimeout(() => reject(new Error('Request timeout')), this.requestTimeoutMs)
        )
      ]);

      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      if (fallback) return fallback(error);
      throw error;
    }
  }

  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failures += 1;
    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttemptAt = Date.now() + this.cooldownMs;
    }
  }
}
```

이 예시의 포인트는 세 가지입니다.

- timeout과 breaker를 함께 둠
- open 상태에서는 실제 호출을 멈춤
- fallback 경로를 별도로 제공함

즉 회복력이 중요한 코드는 "실패했을 때 어떻게 덜 망가질 것인가"까지 함께 설계해야 합니다.

### H3. 라이브러리를 쓰더라도 fallback 전략은 직접 설계해야 한다

Node.js에서는 circuit breaker 관련 라이브러리를 활용할 수 있습니다.
하지만 라이브러리가 있다고 해서 운영 전략이 자동으로 생기지는 않습니다.
오히려 중요한 것은 **open 상태에서 무엇을 반환할지**입니다.

예를 들면 다음처럼 나눌 수 있습니다.

- 추천 API 실패 → 빈 추천 목록 또는 캐시된 결과 반환
- 알림 API 실패 → 비동기 재처리 큐로 전환
- 통계 API 실패 → 최신성이 조금 떨어지는 캐시 사용
- 결제 승인 실패 → 절대 임의 성공 처리 금지, 명시적 실패 응답

즉 fallback은 "아무 값이나 대신 주는 것"이 아니라, 도메인별로 **안전하게 축소된 동작**이어야 합니다.
[Node.js Brownout Pattern 가이드](/development/blog/seo/2026/04/01/nodejs-brownout-pattern-feature-toggle-overload-resilience-guide.html)처럼 비핵심 기능을 잠시 줄여 핵심 경로를 지키는 전략과도 잘 맞습니다.

## Circuit Breaker 임계치는 어떻게 정해야 할까

### H3. 평균이 아니라 장애 전조 구간을 기준으로 잡아야 한다

임계치를 정할 때 흔한 실수는 평소 평균값만 보고 설정하는 것입니다.
하지만 breaker는 정상 구간보다 **비정상으로 넘어가는 경계**에서 작동해야 의미가 있습니다.
그래서 아래 질문을 중심으로 잡는 편이 좋습니다.

- 이 다운스트림은 몇 초 이상 느려지면 상위 서비스가 위험해지는가
- 연속 몇 번 실패하면 우연이 아니라 장애 징후로 봐야 하는가
- 실패율이 몇 퍼센트일 때 fast-fail이 전체 시스템에 더 이로운가
- 얼마나 쉬고 나서 재확인해야 하는가

예를 들어 호출량이 낮은 서비스에서는 "최근 20건 중 50% 실패"보다 "연속 5회 timeout"이 더 실용적일 수 있습니다.
반대로 호출량이 높은 서비스는 rolling window 기반 실패율과 latency percentile을 함께 보는 편이 정확할 수 있습니다.

### H3. 서비스 중요도에 따라 공격성과 보수성을 다르게 가져가야 한다

모든 다운스트림에 같은 breaker 정책을 적용하면 운영이 거칠어집니다.
핵심 결제 시스템과 부가 추천 시스템은 의미가 다르기 때문입니다.

- 핵심 트랜잭션 경로: 더 보수적으로 관찰하고 fallback도 엄격하게 설계
- 비핵심 부가 기능: 더 공격적으로 open 전환하고 빠르게 축소 동작
- 읽기 전용 캐시 가능 API: 비교적 짧은 cool-down과 캐시 fallback 활용
- 쓰기/중복 위험 API: retry와 half-open 정책을 특히 신중히 설정

결국 circuit breaker 튜닝은 기술 설정이면서 동시에 제품 정책입니다.
"무엇을 지키기 위해 무엇을 잠시 포기할지"를 정하는 일이기 때문입니다.

## Node.js Circuit Breaker 도입 시 자주 하는 실수

### H3. breaker를 넣었는데 timeout이 여전히 너무 긴 경우

회로 차단기를 넣어도 개별 호출 timeout이 10초, 15초처럼 길면 이미 자원이 너무 오래 잡혀 있습니다.
breaker가 열리기 전까지 서비스는 계속 지칠 수밖에 없습니다.
따라서 timeout은 breaker보다 앞단의 1차 방어선으로 짧고 명확해야 합니다.

### H3. open 상태에서 의미 없는 fallback을 반환하는 경우

사용자에게는 성공처럼 보이지만 실제로는 중요한 데이터가 빠져 있거나, 비즈니스적으로 틀린 값을 주는 fallback은 더 위험할 수 있습니다.
특히 결제, 재고, 인증처럼 정확성이 중요한 영역에서는 "애매한 성공"보다 **명시적 실패**가 낫습니다.

### H3. 메트릭 없이 breaker 상태만 로그로 보는 경우

운영에서는 아래 지표를 함께 봐야 의미가 있습니다.

- closed/open/half-open 전환 횟수
- circuit open 비율
- fallback 사용률
- timeout 비율
- retry 발생량
- 다운스트림별 p95, p99 latency

breaker가 자주 열린다는 사실만으로는 충분하지 않습니다.
왜 열렸는지, open이 실제로 상위 시스템 보호에 도움이 됐는지까지 봐야 합니다.

## 실무 체크리스트

### H3. 발행 전이 아니라 배포 전에 확인해야 할 질문

circuit breaker를 코드에 넣기 전에 아래 질문을 먼저 정리하면 시행착오를 줄일 수 있습니다.

- 실패로 간주할 이벤트는 무엇인가
- timeout 값은 사용자 기대 시간 안에 들어오는가
- retry는 몇 번까지 허용할 것인가
- open 상태에서 fallback, queue, 캐시, 명시적 실패 중 무엇을 택할 것인가
- half-open에서 몇 건만 시험할 것인가
- 어떤 메트릭과 알림 기준을 둘 것인가

이 질문에 답이 없으면 라이브러리만 붙여도 운영은 쉽게 안정되지 않습니다.

## 마무리

Node.js에서 circuit breaker는 장애를 완전히 없애는 기술이 아닙니다.
대신 장애가 생겼을 때 **그 실패가 어디까지 번질지를 통제하는 기술**입니다.
느린 외부 API, 부분적으로 흔들리는 내부 서비스, 재시도로 증폭되는 부하가 반복된다면, 이제는 단순 timeout이나 retry만으로는 부족할 가능성이 큽니다.

특히 timeout budget, admission control, load shedding을 이미 고민하고 있다면 circuit breaker는 그 다음에 붙는 자연스러운 조각입니다.
핵심은 간단합니다.
다운스트림이 위험 신호를 보낼 때 끝까지 버티며 같이 지치는 대신, **짧게 끊고 핵심 경로를 지키는 편이 전체 시스템에는 더 건강하다**는 점입니다.

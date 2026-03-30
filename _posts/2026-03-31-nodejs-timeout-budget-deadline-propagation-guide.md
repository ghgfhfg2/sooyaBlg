---
layout: post
title: "Node.js Timeout Budget 가이드: Deadline Propagation으로 연쇄 타임아웃 줄이는 설계법"
date: 2026-03-31 08:00:00 +0900
lang: ko
translation_key: nodejs-timeout-budget-deadline-propagation-guide
permalink: /development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html
alternates:
  ko: /development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html
  x_default: /development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, timeout-budget, deadline-propagation, latency, backend, resilience, api]
description: "Node.js 서비스에서 timeout budget과 deadline propagation을 적용해 연쇄 타임아웃, 불필요한 재시도, 다운스트림 과부하를 줄이는 실무 설계 방법을 정리했습니다."
---

Node.js 백엔드에서 timeout은 보통 "몇 초로 할까" 수준에서 끝나기 쉽습니다.
하지만 실제 장애는 timeout 숫자 하나보다 **요청 전체에 허용된 시간 예산을 서비스 간에 어떻게 나눠 쓰는가**에서 더 자주 터집니다.
사용자 요청은 3초 안에 끝나야 하는데 API gateway, BFF, 내부 서비스, DB, 외부 API가 모두 제각각 3초 timeout을 들고 있으면 결국 마지막에는 연쇄 대기와 중복 재시도만 남습니다.
이때 필요한 개념이 **timeout budget**과 **deadline propagation**입니다.
이 글에서는 Node.js 환경에서 timeout budget이 왜 중요한지, 서비스 간 deadline을 어떻게 전달해야 하는지, retry·circuit breaker·backpressure와는 어떻게 연결되는지 실무 관점에서 정리합니다.

## Node.js timeout budget이 왜 중요한가

### H3. 개별 timeout만 맞추면 전체 요청 시간은 쉽게 무너진다

실무에서 흔한 구조는 아래와 같습니다.

- 클라이언트는 3초 안에 응답을 기대함
- API gateway도 3초 timeout
- 내부 BFF도 3초 timeout
- downstream 서비스도 각각 3초 timeout
- 외부 API 호출도 3초 timeout

겉보기에는 모두 "3초 안에 끝내자"처럼 보이지만, 실제로는 그렇지 않습니다.
상위 요청이 이미 2.4초를 쓴 상태에서 하위 서비스가 또 3초를 기다리면, 그 호출은 성공해도 사용자 입장에서는 이미 늦은 결과가 됩니다.
문제는 이 늦은 작업이 서버 자원과 연결 풀, worker 슬롯을 계속 점유한다는 점입니다.
즉 timeout budget 없이 각 계층이 독립적으로 기다리면 **유효하지 않은 일에 자원을 계속 쓰게 되는 구조**가 됩니다.

### H3. deadline이 없으면 연쇄 타임아웃과 쓸모없는 재시도가 겹친다

timeout budget이 없는 시스템에서는 보통 이런 일이 벌어집니다.

- 상위 계층은 이미 포기했는데 하위 계층은 계속 처리함
- 실패 원인이 느린 downstream인데 여러 계층이 각각 재시도함
- 응답은 이미 버려졌는데 DB 조회나 외부 API 호출은 끝까지 실행됨
- tail latency가 길어질수록 in-flight request 수가 급증함

이 구조는 단순히 느린 것에서 끝나지 않습니다.
결국 **대기 증가 → 자원 점유 증가 → timeout 증가 → 재시도 증가 → 추가 과부하**의 악순환으로 이어집니다.
그래서 timeout budget은 성능 미세 조정보다 **불필요한 일을 빨리 멈추는 운영 규율**에 가깝습니다.

## timeout budget과 deadline propagation은 무엇인가

### H3. timeout budget은 요청 전체에 허용된 총 시간을 뜻한다

timeout budget은 각 호출마다 따로 주는 시간이 아니라, **사용자 요청이 끝날 때까지 허용되는 총 시간**입니다.
예를 들어 전체 SLA가 2500ms라면, 그 안에서 각 계층은 남은 시간을 계산해 사용해야 합니다.

- API gateway: 인증과 라우팅에 150ms 사용
- BFF: 조합 로직에 250ms 사용
- 내부 서비스 A 호출: 최대 600ms
- 내부 서비스 B 호출: 최대 700ms
- fallback 또는 후처리: 남은 시간 내에서만 수행

핵심은 "각 단계가 고정 timeout을 갖는다"가 아니라 **남은 예산 안에서만 다음 단계를 진행한다**는 점입니다.

### H3. deadline propagation은 남은 시간을 하위 호출에 계속 전달하는 방식이다

deadline propagation은 상위 계층이 가진 deadline 또는 남은 시간을 하위 서비스에 전달하는 설계입니다.
보통은 아래 둘 중 하나로 구현합니다.

- 절대 시각 deadline 전달: `x-deadline-at: 1711849205123`
- 남은 시간 전달: `x-timeout-ms: 640`

실무에서는 절대 시각이 더 안전한 경우가 많습니다.
중간 hop이 많아도 각 서비스가 **현재 시각 기준으로 남은 시간을 다시 계산**할 수 있기 때문입니다.
반면 남은 시간을 숫자로 계속 넘기면 네트워크 지연, 큐 대기, 중간 처리 시간을 반영하기 어렵거나 누락하기 쉽습니다.

## Node.js에서 어디에 먼저 적용해야 할까

### H3. 서비스 간 HTTP 호출과 외부 API 연동이 1차 대상이다

timeout budget은 아래 구간에서 가장 먼저 효과가 납니다.

- API gateway → BFF → 내부 서비스로 이어지는 HTTP 체인
- 결제, 메시징, 인증, 추천 같은 외부 API 호출
- 여러 downstream 응답을 fan-out으로 조합하는 aggregator API
- background job이 다시 외부 시스템을 호출하는 구간

특히 fan-out 구조에서는 개별 호출 하나가 아니라 **여러 개의 느린 호출이 전체 deadline을 같이 잠식**하는 문제가 큽니다.
이때 각 호출이 남은 예산을 모르고 고정 timeout만 들고 있으면 tail latency가 빠르게 커집니다.

### H3. retry와 fallback이 있는 구간일수록 deadline 없이 운영하면 손해가 커진다

처음엔 timeout만 잘 넣으면 된다고 생각하기 쉽습니다.
하지만 실전에서는 retry, fallback, cache miss, queue 대기까지 함께 얽혀 있습니다.
예를 들면 아래처럼 됩니다.

- 추천 API가 800ms 안에 응답하지 않음
- BFF가 retry 한 번 더 시도함
- 그 사이 상위 요청은 이미 거의 deadline에 도달함
- fallback 캐시 조회까지 붙으면서 결국 전체 응답이 timeout 남

이때 retry가 아예 나쁜 것은 아닙니다.
문제는 **남은 시간이 충분한지 확인하지 않고 retry를 실행하는 것**입니다.
retry와 fallback은 항상 deadline-aware 해야 합니다.

## Node.js timeout budget 구현 예시

### H3. 요청 시작 시 deadline을 만들고 남은 시간을 계산하는 helper를 두면 관리가 쉬워진다

아래처럼 요청 단위 deadline helper를 두면 각 계층에서 같은 기준을 쓸 수 있습니다.

```js
function createDeadline(totalMs) {
  const deadlineAt = Date.now() + totalMs;

  return {
    deadlineAt,
    remainingMs() {
      return Math.max(0, deadlineAt - Date.now());
    }
  };
}

const deadline = createDeadline(2500);
console.log(deadline.remainingMs());
```

중요한 점은 남은 시간이 0에 가깝거나 이미 소진됐다면 다음 작업을 억지로 시작하지 않는 것입니다.
"혹시 되면 좋고"라는 마음으로 던지는 호출이 운영에서는 가장 비싼 호출이 되기 쉽습니다.

### H3. fetch 호출에는 남은 시간 기준으로 AbortSignal을 붙여야 한다

Node.js의 `fetch`와 `AbortSignal.timeout()`을 이용하면 남은 예산을 비교적 간단하게 반영할 수 있습니다.

```js
async function fetchWithDeadline(url, deadline) {
  const remaining = deadline.remainingMs();

  if (remaining < 100) {
    throw new Error('deadline exceeded before downstream call');
  }

  const response = await fetch(url, {
    signal: AbortSignal.timeout(remaining)
  });

  if (!response.ok) {
    throw new Error(`downstream error: ${response.status}`);
  }

  return response.json();
}
```

여기서 100ms 같은 최소 여유 시간을 두는 이유는, 이미 사실상 끝난 요청을 downstream에 넘겨도 성공 확률보다 비용이 더 큰 경우가 많기 때문입니다.
네트워크 왕복, JSON 파싱, 로깅 비용까지 생각하면 **실행 자체를 생략하는 판단**이 꽤 중요합니다.

### H3. 서비스 간 호출에서는 deadline header를 함께 전달해야 체인이 끊기지 않는다

deadline-aware 한 서비스가 하나만 있어서는 효과가 제한적입니다.
호출 체인 전체가 같은 deadline 개념을 공유해야 합니다.

```js
async function callInventoryService(itemId, deadline) {
  const remaining = deadline.remainingMs();

  if (remaining < 120) {
    throw new Error('not enough budget for inventory call');
  }

  const response = await fetch(`https://inventory.internal/items/${itemId}`, {
    headers: {
      'x-deadline-at': String(deadline.deadlineAt)
    },
    signal: AbortSignal.timeout(remaining)
  });

  return response.json();
}
```

그리고 수신 측에서는 이 헤더를 읽어 새로운 고정 timeout을 만드는 대신, **전달받은 deadline 기준으로 남은 시간을 다시 계산**해야 합니다.
이렇게 해야 gateway에서 시작한 시간 예산이 여러 hop을 지나도 일관되게 유지됩니다.

## retry, circuit breaker, backpressure와는 어떻게 연결될까

### H3. deadline 없는 retry는 복구 전략이 아니라 과부하 증폭기가 되기 쉽다

retry는 네트워크 순간 장애나 일시적 5xx에 분명 유효합니다.
하지만 deadline을 고려하지 않으면 아래처럼 망가지기 쉽습니다.

- 첫 호출이 700ms 사용
- 두 번째 retry가 700ms 사용
- 세 번째 fallback이 400ms 사용
- 이미 상위 요청은 2초 budget을 모두 잃음

이 구조에서는 성공 여부보다 **다른 요청을 처리할 자원까지 같이 잃는 것**이 더 큰 문제입니다.
retry는 항상 아래 조건과 함께 가는 편이 좋습니다.

- 남은 budget이 최소 retry 기준보다 클 때만 실행
- exponential backoff와 jitter 적용
- 사용자 요청 경로와 background retry 분리
- 실패 비용이 큰 쓰기 연산은 idempotency와 함께 설계

재시도 전략 자체를 따로 정리한 글은 [Node.js Exponential Backoff & Jitter 가이드](/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html)에서 이어서 볼 수 있습니다.

### H3. circuit breaker와 backpressure도 deadline 정보를 알아야 더 정교하게 동작한다

circuit breaker는 반복 실패를 빠르게 차단하고, backpressure는 감당 가능한 양만 흘려보내는 패턴입니다.
그런데 둘 다 deadline 정보가 있으면 훨씬 현실적으로 동작합니다.

예를 들어 남은 시간이 거의 없는 요청은 아래처럼 처리할 수 있습니다.

- circuit breaker fallback 경로로 즉시 우회
- low-priority downstream 호출 생략
- queue 적재 대신 fast-fail
- partial response 허용

즉 deadline propagation은 timeout에만 쓰는 정보가 아니라, **어떤 일을 지금 계속할 가치가 있는가**를 결정하는 기준이 됩니다.
과부하 제어 관점은 [Node.js Backpressure 가이드](/development/blog/seo/2026/03/30/nodejs-backpressure-overload-control-guide.html), 실패 격리 관점은 [Node.js Circuit Breaker 가이드](/development/blog/seo/2026/03/27/nodejs-circuit-breaker-failure-isolation-guide.html)와 함께 보면 더 잘 연결됩니다.

## 운영하면서 꼭 봐야 할 지표

### H3. timeout 횟수보다 예산 소진 위치를 보는 편이 원인 파악에 더 유리하다

단순히 "timeout이 몇 건 났는가"만 보면 어디서 시간이 사라졌는지 알기 어렵습니다.
실제로는 아래 지표가 더 중요합니다.

- 요청 시작 대비 각 hop의 elapsed time
- downstream 호출 시작 시점의 remaining budget
- deadline exceeded 비율
- retry가 실제로 수행된 시점의 remaining budget
- 499, 504, upstream timeout 비율
- queue wait time과 in-flight request 수

이 지표를 보면 "외부 API가 느리다"보다 더 구체적으로, **어느 계층이 남은 시간을 거의 다 먹고 넘기는지**를 볼 수 있습니다.
이게 보여야 timeout 숫자를 조정하는 대신 구조를 바꿀 수 있습니다.

### H3. 요청 우선순위별로 최소 보장 시간을 다르게 두는 편이 현실적이다

모든 요청에 같은 timeout budget을 주는 것이 항상 정답은 아닙니다.
예를 들면 아래처럼 나누는 편이 더 운영 친화적입니다.

- 로그인, 결제 승인: 짧지만 엄격한 budget
- 대시보드 조합 API: 부분 응답 허용 + 중간 budget
- 관리자 export: 긴 budget 허용, 대신 별도 경로 처리
- 추천/부가 정보: budget 부족 시 생략 가능

핵심은 timeout budget이 단순 숫자 정책이 아니라 **우선순위 정책과 연결된 운영 기준**이라는 점입니다.

## Node.js timeout budget 적용 체크리스트

### H3. 아래 항목에 많이 해당되면 deadline propagation을 도입할 시점이다

배포 전이나 장애 회고 때 아래 질문을 빠르게 점검해볼 수 있습니다.

- 서비스마다 고정 timeout만 있고 요청 전체 deadline 개념이 없다
- 상위 요청이 끝난 뒤에도 하위 호출이 계속 실행된다
- retry가 남은 시간과 무관하게 수행된다
- fan-out API에서 일부 downstream 지연이 전체 응답을 자주 망친다
- timeout이 났는데 어느 hop에서 시간이 소진됐는지 추적하기 어렵다
- low-priority 기능이 핵심 응답 시간을 같이 잠식한다

두세 개 이상 해당되면 timeout budget과 deadline propagation을 설계 우선순위에 올릴 만합니다.

## 자주 묻는 질문

### H3. timeout budget과 일반 timeout 설정은 무엇이 다른가요?

일반 timeout은 개별 호출 기준이고, timeout budget은 **요청 전체 수명주기 기준**입니다.
실무에서는 개별 timeout만 있으면 각 계층이 따로 오래 기다려 전체 응답을 망치는 경우가 많습니다.

### H3. deadline은 절대 시각과 남은 시간 중 무엇으로 전달하는 게 좋나요?

둘 다 가능하지만, hop이 여러 개인 시스템에서는 절대 시각 deadline이 더 관리하기 쉽습니다.
각 서비스가 현재 시각 기준으로 남은 시간을 다시 계산할 수 있어서 중간 지연을 더 정확히 반영할 수 있습니다.

### H3. 모든 API가 deadline propagation을 해야 하나요?

처음부터 전부 바꿀 필요는 없습니다.
먼저 fan-out이 많은 API, 외부 의존성이 많은 경로, timeout과 retry가 자주 문제 되는 경로부터 적용하는 편이 현실적입니다.

## 마무리

Node.js에서 timeout budget은 단순히 timeout 값을 줄이는 기법이 아닙니다.
그보다 중요한 것은 **이미 늦은 요청에 자원을 더 쓰지 않도록 시스템 전체가 같은 시계를 보게 만드는 것**입니다.
서비스가 커질수록 장애는 한 번의 큰 실패보다, 작은 대기와 느린 retry가 여러 계층에서 겹치며 만들어집니다.
그래서 deadline propagation은 성능 최적화라기보다 **쓸모없는 일을 빨리 포기해 전체 시스템을 지키는 설계**에 가깝습니다.

고정 timeout만으로 운영이 자꾸 불안하다면 숫자를 더 만지기 전에 먼저 질문해볼 만합니다.
이 요청에 정말 남은 시간이 있는가.
그 질문을 코드와 헤더 수준에서 일관되게 다루기 시작하면 연쇄 타임아웃 문제를 훨씬 덜 아프게 만들 수 있습니다.

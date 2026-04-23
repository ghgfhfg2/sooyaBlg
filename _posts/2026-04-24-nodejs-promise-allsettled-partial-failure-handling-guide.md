---
layout: post
title: "Node.js Promise.allSettled 가이드: 부분 실패를 허용하면서 병렬 작업을 안전하게 처리하는 법"
date: 2026-04-24 08:00:00 +0900
lang: ko
translation_key: nodejs-promise-allsettled-partial-failure-handling-guide
permalink: /development/blog/seo/2026/04/24/nodejs-promise-allsettled-partial-failure-handling-guide.html
alternates:
  ko: /development/blog/seo/2026/04/24/nodejs-promise-allsettled-partial-failure-handling-guide.html
  x_default: /development/blog/seo/2026/04/24/nodejs-promise-allsettled-partial-failure-handling-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, promise-allsettled, partial-failure, concurrency, backend, reliability, async, javascript]
description: "Node.js에서 Promise.allSettled를 사용해 부분 실패를 허용하는 병렬 처리 패턴, 결과 정리 방식, 재시도 기준, 실무 체크포인트를 예제와 함께 정리했습니다."
---

Node.js에서 여러 비동기 작업을 병렬로 처리할 때, 가장 흔한 실수 중 하나는 **하나만 실패해도 전체를 실패로 간주해버리는 것**입니다.
실제로는 모든 작업이 반드시 동시에 성공해야 하는 경우보다, 일부 실패를 감수하고도 처리 가능한 결과를 먼저 확보하는 편이 더 현실적인 경우가 많습니다.

이럴 때 유용한 도구가 `Promise.allSettled()`입니다.
결론부터 말하면 `Promise.allSettled()`는 **성공과 실패를 함께 수집해 후처리할 수 있게 해주는 병렬 처리 도구**이고, Node.js에서는 알림 발송, 다중 외부 API 조회, 배치 집계 같은 작업에서 특히 실용적입니다.

## Promise.all 대신 allSettled를 봐야 하는 순간

### H3. 하나의 실패가 전체 비즈니스 실패를 뜻하지 않는 경우가 많다

`Promise.all()`은 하나라도 reject되면 전체가 즉시 실패합니다.
이 동작은 강하게 묶인 작업에는 적절하지만, 실무에서는 아래처럼 부분 성공이 허용되는 작업이 자주 있습니다.

- 여러 외부 API에서 데이터를 모아 가능한 만큼 먼저 보여주는 경우
- 여러 알림 채널 중 일부만 실패해도 핵심 처리는 계속해야 하는 경우
- 배치 작업에서 실패한 항목만 별도로 재처리하면 되는 경우
- 캐시 warm-up, 검색 인덱싱, 부가 통계 생성처럼 완전 성공이 필수는 아닌 경우

이런 상황에서 `Promise.all()`만 고집하면 실패 전파가 과해집니다.
반면 `Promise.allSettled()`는 성공 결과와 실패 이유를 분리해서 볼 수 있어, **부분 실패를 운영 가능한 형태로 다룰 수 있게 해줍니다**.

### H3. 빠른 실패보다 실패 분류가 더 중요할 때가 있다

장애 대응에서는 “실패했다”는 사실보다 **무엇이 얼마나 실패했는지**가 더 중요할 때가 많습니다.
특히 대량 병렬 작업에서는 전부를 한 덩어리 예외로 던져버리면 다음 판단이 어려워집니다.

예를 들어 아래처럼 나눠야 하는 경우가 있습니다.

- 일시적 오류라 재시도 가능한 실패
- 입력값 문제라 즉시 제외해야 하는 실패
- 핵심 데이터라 전체 요청을 degraded 상태로 표시해야 하는 실패
- 부가 데이터라 로그만 남기고 넘어가도 되는 실패

이때 `Promise.allSettled()`는 결과를 일괄 수집해 후속 정책을 붙이기 좋습니다.
이 감각은 [Node.js Retry Budget 가이드](/development/blog/seo/2026/03/30/nodejs-retry-budget-overload-control-guide.html)와도 이어집니다.
실패를 세분화해야 재시도도 과해지지 않습니다.

## Promise.allSettled는 무엇을 반환하나

### H3. fulfilled와 rejected가 같은 배열 안에 함께 들어온다

`Promise.allSettled()`는 입력 순서를 유지한 채 각 작업의 최종 상태를 배열로 돌려줍니다.
각 원소는 아래 둘 중 하나입니다.

- `status: 'fulfilled'` 와 `value`
- `status: 'rejected'` 와 `reason`

즉 “누가 성공했고 누가 실패했는지”를 따로 try/catch 중첩 없이 한 번에 정리할 수 있습니다.

```js
const results = await Promise.allSettled([
  fetchUserProfile(),
  fetchUserOrders(),
  fetchUserCoupons(),
]);

console.log(results);
```

이 구조 덕분에 병렬 실행과 결과 분류를 분리할 수 있습니다.
성공한 결과만 먼저 조립하고, 실패한 결과는 재시도 큐나 로그로 보내는 흐름이 자연스럽습니다.

### H3. 입력 순서는 유지되지만 의미 있는 이름 매핑이 더 중요하다

실무에서 자주 놓치는 점은 배열 인덱스만 믿고 처리하는 것입니다.
작업 수가 늘면 `results[3]`, `results[5]` 같은 코드는 금방 읽기 어려워집니다.

그래서 아래처럼 이름을 붙여 처리하는 편이 운영에 유리합니다.

```js
const tasks = {
  profile: fetchUserProfile(userId),
  orders: fetchUserOrders(userId),
  coupons: fetchUserCoupons(userId),
};

const entries = Object.entries(tasks);
const settled = await Promise.allSettled(entries.map(([, task]) => task));

const mapped = entries.map(([name], index) => ({
  name,
  ...settled[index],
}));
```

이렇게 하면 실패 로그와 메트릭도 훨씬 분명해집니다.
단순히 “병렬 작업 실패”가 아니라 “coupons API 실패”처럼 남길 수 있기 때문입니다.

## Node.js에서 어떻게 쓰면 안전할까

### H3. 성공한 값과 실패한 값을 먼저 분리한다

가장 기본적인 패턴은 결과를 성공과 실패로 나누는 것입니다.
그러면 응답 조립, degraded 응답, 재시도 판단이 쉬워집니다.

```js
function partitionSettledResults(results) {
  const fulfilled = [];
  const rejected = [];

  for (const result of results) {
    if (result.status === 'fulfilled') {
      fulfilled.push(result.value);
    } else {
      rejected.push(result.reason);
    }
  }

  return { fulfilled, rejected };
}
```

핵심은 실패를 숨기지 않는 것입니다.
`allSettled`를 쓴다고 해서 실패를 무시하라는 뜻은 아닙니다.
오히려 **실패를 남기되 전체 흐름을 끊지 않는 것**에 가깝습니다.

### H3. 응답 모델에서 필수 데이터와 선택 데이터를 구분한다

부분 실패를 허용하려면, 어떤 데이터가 필수인지 먼저 정리돼 있어야 합니다.
이 경계가 없으면 `allSettled`를 써도 결국 판단이 흐려집니다.

예를 들어 사용자 상세 화면이라면 아래처럼 나눌 수 있습니다.

- 필수: 기본 프로필, 권한 정보
- 선택: 추천 상품, 쿠폰, 활동 통계

이 경우 필수 데이터가 실패하면 전체 요청을 실패시키고, 선택 데이터가 실패하면 degraded 응답으로 내려보낼 수 있습니다.
이런 식의 자원 경계는 [Node.js Bulkhead Pattern 가이드](/development/blog/seo/2026/03/29/nodejs-bulkhead-pattern-resource-isolation-guide.html)와 함께 보면 더 이해가 쉽습니다.

## 언제 Promise.allSettled가 특히 잘 맞을까

### H3. 외부 API fan-out 후 가능한 결과만 먼저 합쳐야 할 때

한 요청 안에서 여러 upstream을 동시에 조회하는 fan-out 구조는 부분 실패가 흔합니다.
이때 모든 소스를 필수로 묶으면 느리거나 불안정한 한 곳이 전체 응답을 망칠 수 있습니다.

예를 들어 아래처럼 쓸 수 있습니다.

- 사용자 프로필 API
- 추천 API
- 포인트 API
- 최근 활동 API

이 중 추천이나 최근 활동은 빠져도 페이지 자체는 보여줄 수 있습니다.
이럴 때 `Promise.allSettled()`는 “핵심은 유지하고 부가 정보만 유연하게 붙이는” 구조에 잘 맞습니다.
단, 무제한 fan-out은 위험하므로 [Node.js Concurrency Limit, Promise Pool 가이드](/development/blog/seo/2026/03/27/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)처럼 동시성 상한과 함께 쓰는 편이 안전합니다.

### H3. 배치 작업에서 실패한 항목만 선별 재처리할 때

배치에서는 전체 중 일부 항목이 깨지는 일이 흔합니다.
이때 하나 실패했다고 전체 작업을 되돌리는 것보다, 실패한 항목만 별도로 수집해 재처리하는 편이 효율적입니다.

예를 들어 이메일 발송 배치, 썸네일 생성, 외부 동기화 작업은 아래처럼 다룰 수 있습니다.

- 성공 항목은 완료 처리
- 일시 실패 항목은 재시도 큐로 이동
- 영구 실패 항목은 DLQ나 운영 알림으로 분리

이 흐름은 [Node.js Dead Letter Queue 가이드](/development/blog/seo/2026/03/25/nodejs-dead-letter-queue-dlq-failure-recovery-guide.html)와도 잘 맞습니다.
중요한 것은 실패를 없애는 게 아니라, **실패를 운영 가능한 단위로 정리하는 것**입니다.

## 실무에서 자주 생기는 실수

### H3. allSettled를 썼는데도 실제로는 실패를 삼켜버리는 경우

`Promise.allSettled()`를 도입한 뒤 가장 흔한 부작용은 “이제 실패해도 괜찮다”는 식으로 흘러가는 것입니다.
하지만 실패를 관측하지 않으면 조용히 품질이 떨어집니다.

아래는 꼭 남기는 편이 좋습니다.

- 실패 개수와 실패 비율
- 어떤 작업 이름이 실패했는지
- 재시도 대상인지 영구 실패인지
- degraded 응답으로 내려간 횟수

즉 `allSettled`는 예외를 숨기는 도구가 아니라, **예외를 구조화하는 도구**로 써야 합니다.

### H3. 병렬 수를 제한하지 않으면 allSettled도 쉽게 과부하를 만든다

`Promise.allSettled()`는 결과 수집 방식일 뿐, 동시성 제어 도구는 아닙니다.
작업 1만 개를 한 번에 넣으면 성공/실패와 별개로 메모리와 외부 자원을 과하게 밀어붙일 수 있습니다.

그래서 아래 원칙이 중요합니다.

- 작업 수가 많으면 pool 또는 batch로 나눈다
- 외부 API 호출은 timeout과 retry budget을 함께 둔다
- DB, Redis, HTTP 호출은 각자 다른 병목을 가진다는 점을 고려한다
- 필요하면 semaphore나 bounded queue로 상한을 둔다

이 부분은 [Node.js Semaphore Pattern 가이드](/development/blog/seo/2026/04/08/nodejs-semaphore-pattern-shared-resource-concurrency-guide.html)와 같이 보면 더 명확합니다.
병렬 결과 수집과 동시성 제한은 다른 문제입니다.

### H3. 실패 사유를 그대로 사용자 응답에 노출하면 위험할 수 있다

실패한 `reason`에는 내부 에러 메시지, upstream 응답, 스택 정보가 섞일 수 있습니다.
그래서 운영 로그와 사용자 응답은 분리하는 편이 안전합니다.

예를 들어 사용자에게는 아래 정도만 내려도 충분합니다.

- 일부 부가 정보가 일시적으로 보이지 않을 수 있음
- 잠시 후 다시 시도해달라는 안내
- 핵심 기능 자체는 사용 가능하다는 안내

반면 내부에서는 구체적인 `reason`을 구조화 로그로 남겨 원인 분석에 써야 합니다.
민감정보와 과도한 에러 노출을 막는다는 점에서 이 구분은 꽤 중요합니다.

## 빠르게 적용할 때 쓰는 체크리스트

### H3. 아래 다섯 가지가 맞으면 allSettled 도입 가치가 크다

- 하나의 실패가 전체 요청 실패를 뜻하지 않는다.
- 필수 데이터와 선택 데이터를 구분할 수 있다.
- 실패한 항목만 재시도하거나 별도 큐로 보낼 수 있다.
- 실패 비율과 degraded 응답을 관측할 수 있다.
- 병렬 수를 별도 제한할 장치가 있다.

## 마무리

Node.js의 `Promise.allSettled()`는 단순히 “실패해도 계속 가는 방법”이 아닙니다.
이 도구의 핵심은 성공과 실패를 함께 수집해, 부분 실패를 **비즈니스적으로 해석 가능한 결과**로 바꾸는 데 있습니다.

실무에서는 모든 작업을 무조건 같은 운명으로 묶기보다, 무엇이 필수이고 무엇이 선택인지 먼저 나누는 편이 좋습니다.
그다음 `Promise.allSettled()`로 결과를 모으고, 실패 항목만 재시도하거나 degraded 응답으로 처리하면 병렬 작업의 안정성이 훨씬 좋아집니다.

관련해서 함께 보면 좋은 글은 아래입니다.

- [Node.js Retry Budget 가이드](/development/blog/seo/2026/03/30/nodejs-retry-budget-overload-control-guide.html)
- [Node.js Concurrency Limit, Promise Pool 가이드](/development/blog/seo/2026/03/27/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)
- [Node.js Bulkhead Pattern 가이드](/development/blog/seo/2026/03/29/nodejs-bulkhead-pattern-resource-isolation-guide.html)
- [Node.js Semaphore Pattern 가이드](/development/blog/seo/2026/04/08/nodejs-semaphore-pattern-shared-resource-concurrency-guide.html)
- [Node.js Dead Letter Queue 가이드](/development/blog/seo/2026/03/25/nodejs-dead-letter-queue-dlq-failure-recovery-guide.html)

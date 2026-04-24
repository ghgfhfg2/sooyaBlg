---
layout: post
title: "Node.js Promise.any 가이드: 첫 성공 결과로 fallback 체인을 빠르게 구성하는 법"
date: 2026-04-24 20:00:00 +0900
lang: ko
translation_key: nodejs-promise-any-fallback-first-success-guide
permalink: /development/blog/seo/2026/04/24/nodejs-promise-any-fallback-first-success-guide.html
alternates:
  ko: /development/blog/seo/2026/04/24/nodejs-promise-any-fallback-first-success-guide.html
  x_default: /development/blog/seo/2026/04/24/nodejs-promise-any-fallback-first-success-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, promise-any, fallback, resilience, async, javascript, backend, latency]
description: "Node.js에서 Promise.any를 활용해 첫 성공 결과를 빠르게 반환하는 fallback 패턴, AggregateError 처리, timeout 조합, 실무 적용 기준을 예제와 함께 정리했습니다."
---

Node.js에서 여러 데이터 소스 중 **하나만 빨리 성공하면 되는 요청**은 생각보다 많습니다.
예를 들어 캐시와 원본 저장소를 함께 조회하거나, 여러 리전 엔드포인트 중 먼저 응답한 곳을 쓰거나, 여러 추천 소스 중 가능한 결과 하나만 먼저 받아도 되는 경우가 그렇습니다.

이럴 때 `Promise.race()`를 바로 쓰는 경우가 많은데, 실무에서는 그 선택이 꽤 자주 문제를 만듭니다.
결론부터 말하면 **실패보다 성공을 우선해서 기다려야 하는 상황**이라면 `Promise.any()`가 더 잘 맞습니다.
Node.js의 `Promise.any()`는 가장 먼저 성공한 값을 반환하고, 모두 실패했을 때만 에러를 던지기 때문에 fallback 체인을 단순하게 만들 수 있습니다.

## Promise.race 대신 Promise.any를 고려해야 하는 이유

### H3. race는 첫 결과가 실패여도 그대로 끝난다

`Promise.race()`는 가장 먼저 settle된 결과를 그대로 반환합니다.
즉 가장 먼저 도착한 것이 reject라면, 뒤에 성공 가능한 후보가 남아 있어도 전체가 실패합니다.

반면 `Promise.any()`는 **가장 먼저 fulfilled 된 값**을 반환합니다.
이 차이 때문에 아래 같은 상황에서는 `Promise.any()`가 더 자연스럽습니다.

- 캐시 조회 실패 후 원본 조회 성공을 허용해야 하는 경우
- 여러 CDN 또는 리전 엔드포인트 중 하나만 살아 있으면 되는 경우
- 복수 검색 소스 중 최소 한 군데 결과만 있어도 되는 경우
- 외부 API 공급자가 여러 개라 첫 정상 응답만 받아도 되는 경우

핵심은 “누가 먼저 끝났는가”가 아니라 “누가 먼저 성공했는가”입니다.
이 기준이 분명한 요청이라면 `Promise.any()`가 의도를 더 정확히 드러냅니다.

### H3. fallback 로직을 중첩 try/catch 없이 평평하게 유지할 수 있다

`Promise.any()` 없이 fallback을 구현하면 보통 이렇게 됩니다.

- A 호출
- 실패하면 B 호출
- 실패하면 C 호출
- 마지막까지 실패하면 전체 오류 처리

이 구조는 직렬이라 느릴 수 있고, 코드도 길어집니다.
반면 병렬로 띄워두고 첫 성공만 채택하면 평균 응답 시간을 줄이기 쉽습니다.
이 접근은 [Node.js Hedged Requests 가이드](/development/blog/seo/2026/04/08/nodejs-hedged-requests-tail-latency-reduction-guide.html)와도 닿아 있습니다.
다만 hedged request가 동일 자원에 대한 중복 요청 최적화에 가깝다면, `Promise.any()`는 **서로 다른 대안 경로 중 첫 성공 선택**에 더 가깝습니다.

## Promise.any는 어떻게 동작하나

### H3. 첫 번째 fulfilled 값을 반환하고, 전부 실패하면 AggregateError를 던진다

기본 형태는 단순합니다.

```js
const result = await Promise.any([
  fetchFromRedis(key),
  fetchFromPrimaryDb(key),
  fetchFromReplicaDb(key),
]);
```

위 코드는 세 작업 중 하나라도 성공하면 즉시 그 값을 반환합니다.
대신 전부 실패하면 일반 `Error`가 아니라 `AggregateError`가 발생합니다.
즉 실패를 다룰 때는 “마지막 하나의 에러”가 아니라 **전체 후보가 왜 실패했는지**를 함께 보는 습관이 필요합니다.

### H3. 모두 실패했을 때는 errors 배열을 구조적으로 다뤄야 한다

`AggregateError`에는 각 실패 사유가 `errors` 배열에 담깁니다.
이 정보를 그냥 문자열로 뭉개지 말고 분리해두는 편이 좋습니다.

```js
try {
  const profile = await Promise.any([
    fetchProfileFromCache(userId),
    fetchProfileFromApi(userId),
    fetchProfileFromReplica(userId),
  ]);

  return profile;
} catch (error) {
  if (error instanceof AggregateError) {
    logger.error('all profile sources failed', {
      userId,
      errors: error.errors.map((item) => item.message),
    });
  }

  throw error;
}
```

여기서 중요한 점은 사용자 응답과 내부 로그를 분리하는 것입니다.
내부 원인 메시지를 그대로 외부에 노출하면 운영 정보가 새어 나갈 수 있습니다.
이 부분은 [Node.js Promise.allSettled 가이드](/development/blog/seo/2026/04/24/nodejs-promise-allsettled-partial-failure-handling-guide.html)에서 다룬 실패 구조화 원칙과 비슷합니다.

## Node.js에서 안전하게 쓰는 패턴

### H3. 후보별 timeout을 먼저 분리해야 느린 대안이 전체를 끌지 않는다

`Promise.any()`를 쓴다고 해서 느린 후보가 자동으로 정리되지는 않습니다.
각 후보에 timeout이 없으면, 실패는 아니지만 끝없이 늦는 작업이 계속 남아 자원을 잡아둘 수 있습니다.

Node.js에서는 `AbortSignal.timeout()`과 함께 쓰는 패턴이 실용적입니다.

```js
async function fetchJsonWithTimeout(url, timeoutMs) {
  const response = await fetch(url, {
    signal: AbortSignal.timeout(timeoutMs),
  });

  if (!response.ok) {
    throw new Error(`upstream ${response.status}`);
  }

  return response.json();
}

const data = await Promise.any([
  fetchJsonWithTimeout(primaryUrl, 300),
  fetchJsonWithTimeout(replicaUrl, 500),
  fetchJsonWithTimeout(backupUrl, 800),
]);
```

이렇게 해야 “첫 성공” 전략이 실제 응답 시간 개선으로 이어집니다.
관련해서 timeout 조합은 [Node.js AbortSignal.any, timeout 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)도 함께 보면 좋습니다.

### H3. 성공 후 남은 작업을 취소할 수 있으면 더 좋다

Node.js의 기본 Promise 자체는 다른 작업을 자동 취소하지 않습니다.
그래서 fetch, DB client, 커스텀 래퍼가 취소 신호를 지원한다면 성공 후 나머지 후보를 중단하는 편이 낫습니다.

특히 아래 상황에서는 중요합니다.

- 동일한 대용량 데이터를 여러 경로로 중복 조회할 때
- 비용이 드는 외부 API를 여러 개 동시에 때릴 때
- 하나만 성공하면 충분한데 나머지가 계속 CPU/네트워크를 쓰는 경우

즉 `Promise.any()`는 선택 정책이고, **취소는 별도 설계**입니다.
이 둘을 분리해서 봐야 실제 운영 비용을 줄일 수 있습니다.

## 언제 잘 맞고, 언제 오히려 위험할까

### H3. 데이터 동등성이 높을수록 잘 맞는다

`Promise.any()`는 여러 후보가 사실상 같은 의미의 결과를 반환할 때 특히 좋습니다.
예를 들면 아래와 같습니다.

- 같은 데이터를 가진 캐시와 원본 저장소
- 동일 API의 멀티 리전 엔드포인트
- 같은 파일의 서로 다른 미러 서버
- 동일 추천 모델의 읽기 전용 복제본

반대로 후보마다 결과 의미가 조금씩 다르면, 첫 성공이 꼭 올바른 선택이 아닐 수 있습니다.
예를 들어 A는 최신성 우선, B는 캐시 우선, C는 축약 데이터를 주는 식이라면 먼저 온 결과를 그대로 쓰기 전에 정책을 더 정교하게 잡아야 합니다.

### H3. 장애 은폐가 아니라 장애 분리를 목표로 해야 한다

`Promise.any()`를 잘못 쓰면 한 소스가 오래 망가져도 다른 소스가 대신 성공하니 문제가 가려질 수 있습니다.
이건 꽤 위험합니다.
장애를 숨기는 순간 나중에는 fallback 자체가 주 경로가 되어버립니다.

그래서 최소한 아래는 남겨두는 편이 좋습니다.

- 어떤 후보가 최종 승자가 되었는지
- 주 경로가 아닌 fallback이 선택된 비율
- 모든 후보 실패 비율
- 후보별 latency와 timeout 비율

fallback은 복원력을 높이는 장치이지, 관측을 포기하는 장치가 아닙니다.

## 실무 체크리스트

### H3. 아래 질문에 예가 많다면 Promise.any 도입 가치가 높다

- 여러 후보 중 하나만 성공하면 비즈니스 목표를 달성할 수 있는가?
- 가장 빠른 실패보다 가장 빠른 성공이 더 중요한가?
- 각 후보에 timeout과 취소 전략을 붙일 수 있는가?
- fallback 선택 비율을 메트릭으로 관측할 수 있는가?
- 후보들이 비슷한 의미의 결과를 반환하는가?

## 마무리

Node.js의 `Promise.any()`는 “여러 개 중 아무거나”를 대충 고르는 도구가 아닙니다.
이 메서드의 핵심은 **실패를 견디면서 첫 성공을 빠르게 확보하는 것**입니다.

실무에서는 `Promise.race()`와 헷갈리기 쉽지만, 성공 중심 fallback이 필요하다면 `Promise.any()`가 의도에 더 잘 맞습니다.
다만 timeout, 취소, 메트릭 없이 쓰면 느린 자원 낭비나 장애 은폐로 이어질 수 있습니다.
그래서 첫 성공 전략을 도입할 때는 속도와 함께 관측 가능성까지 같이 설계하는 편이 좋습니다.

관련해서 함께 보면 좋은 글은 아래입니다.

- [Node.js Promise.allSettled 가이드](/development/blog/seo/2026/04/24/nodejs-promise-allsettled-partial-failure-handling-guide.html)
- [Node.js AbortSignal.any, timeout 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js Hedged Requests 가이드](/development/blog/seo/2026/04/08/nodejs-hedged-requests-tail-latency-reduction-guide.html)
- [Node.js Exponential Backoff, Jitter 가이드](/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html)

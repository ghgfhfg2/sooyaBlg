---
layout: post
title: "Node.js fetch dispatcher 가이드: Undici 연결 풀을 서비스 단위로 제어하는 법"
date: 2026-08-29 20:00:00 +0900
lang: ko
translation_key: nodejs-undici-dispatcher-fetch-connection-pool-guide
permalink: /development/blog/seo/2026/08/29/nodejs-undici-dispatcher-fetch-connection-pool-guide.html
alternates:
  ko: /development/blog/seo/2026/08/29/nodejs-undici-dispatcher-fetch-connection-pool-guide.html
  x_default: /development/blog/seo/2026/08/29/nodejs-undici-dispatcher-fetch-connection-pool-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fetch, undici, dispatcher, connection-pool, timeout, backend]
description: "Node.js fetch에서 Undici dispatcher와 Agent를 활용해 외부 API별 연결 풀, timeout, 자원 격리 기준을 설계하는 방법을 실무 예제로 정리합니다."
---

Node.js에서 `fetch()`를 쓰면 HTTP 클라이언트 코드가 단순해집니다.
하지만 서비스가 커질수록 단순한 호출 문법만으로는 부족합니다.
결제 API, 검색 API, 추천 API, 사내 인증 API가 모두 같은 방식으로 외부 요청을 보내면 느린 upstream 하나가 전체 서버의 연결과 대기 시간을 잠식할 수 있습니다.

이때 확인해야 하는 축이 **Undici dispatcher**입니다.
Node.js의 내장 `fetch()`는 Undici 기반으로 동작하며, dispatcher를 통해 요청이 어떤 연결 풀과 정책을 사용할지 제어할 수 있습니다.
이 글에서는 Node.js fetch에서 dispatcher를 왜 분리해야 하는지, Agent를 서비스 단위로 어떻게 잡아야 하는지, timeout과 장애 격리 기준을 어떻게 문서화하면 좋은지 정리합니다.

기본적인 요청 timeout 설계는 [Node.js fetch AbortSignal timeout 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html), 전체 요청 예산 전파는 [Node.js timeout budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html), 기능별 자원 분리는 [Node.js Bulkhead Pattern 가이드](/development/blog/seo/2026/03/24/nodejs-bulkhead-pattern-resource-isolation-resilience-guide.html)와 함께 보면 좋습니다.

참고 문서: [Undici 공식 문서](https://undici.nodejs.org/)와 [Node.js fetch 공식 문서](https://nodejs.org/api/globals.html#fetch)

## Node.js fetch dispatcher가 필요한 이유

### 기본 fetch는 호출부가 간단하지만 운영 정책이 잘 보이지 않는다

작은 서버에서는 아래 코드만으로도 충분해 보입니다.

```js
export async function loadProfile(userId, { signal } = {}) {
  const response = await fetch(`https://profile.example.com/users/${userId}`, {
    signal
  });

  if (!response.ok) {
    throw new Error(`Profile API failed: ${response.status}`);
  }

  return response.json();
}
```

문제는 운영 정책이 코드에 거의 드러나지 않는다는 점입니다.
어떤 연결 풀을 쓰는지, 최대 연결 수는 어느 정도인지, 느린 서비스가 다른 API 호출에 영향을 주는지, timeout 기준은 어디서 관리하는지 알기 어렵습니다.

서비스가 작을 때는 "그냥 fetch"가 장점입니다.
하지만 외부 의존성이 늘어나면 HTTP 클라이언트는 단순 호출 도구가 아니라 **장애 전파를 막는 경계**가 됩니다.
dispatcher는 이 경계를 코드로 분명히 표현하는 수단입니다.

### 외부 API별 병목은 서로 격리해야 한다

예를 들어 한 API 서버가 아래 세 가지 upstream을 호출한다고 가정해 봅니다.

- `auth`: 로그인과 권한 확인에 필요
- `billing`: 결제와 구독 상태 확인에 필요
- `recommendation`: 홈 화면 개인화에 필요

recommendation API가 느려졌을 때 auth나 billing까지 같이 느려지는 구조라면 장애 영향 범위가 너무 큽니다.
연결 풀과 timeout을 서비스별로 나누면 낮은 우선순위 기능이 핵심 경로를 끌어내리는 상황을 줄일 수 있습니다.

## Undici Agent로 서비스별 dispatcher 만들기

### H3. Agent를 upstream 단위로 선언한다

Undici의 `Agent`는 여러 origin에 대한 연결 관리 정책을 담을 수 있는 dispatcher입니다.
실무에서는 외부 API별로 Agent를 나누고, 호출 wrapper에서 해당 dispatcher를 명시하는 방식이 읽기 좋습니다.

```js
import { Agent } from 'undici';

export const authDispatcher = new Agent({
  connections: 20,
  keepAliveTimeout: 10_000
});

export const recommendationDispatcher = new Agent({
  connections: 5,
  keepAliveTimeout: 5_000
});
```

핵심은 숫자 자체보다 의도입니다.
인증처럼 핵심 경로에 가까운 API는 충분한 연결 여유를 두고, 추천처럼 내려놓을 수 있는 기능은 더 작은 풀로 제한할 수 있습니다.
이렇게 하면 특정 기능의 느린 요청이 전체 프로세스의 HTTP 자원을 무제한으로 차지하기 어렵습니다.

### H3. fetch 호출부에는 dispatcher를 명시한다

서비스별 wrapper를 만들면 호출부가 매번 Agent를 고민하지 않아도 됩니다.

```js
import { authDispatcher, recommendationDispatcher } from './dispatchers.js';

export async function fetchAuth(path, { signal } = {}) {
  return fetch(`https://auth.example.com${path}`, {
    dispatcher: authDispatcher,
    signal
  });
}

export async function fetchRecommendation(path, { signal } = {}) {
  return fetch(`https://recommendation.example.com${path}`, {
    dispatcher: recommendationDispatcher,
    signal
  });
}
```

이 구조의 장점은 운영 정책이 호출 이름에 묻어난다는 점입니다.
`fetchAuth()`와 `fetchRecommendation()`은 단순 URL 조합 함수가 아니라 서로 다른 연결 예산을 가진 HTTP 경계가 됩니다.

## timeout과 dispatcher는 함께 설계한다

### H3. 연결 풀 제한만으로는 느린 요청을 끝낼 수 없다

dispatcher로 연결 수를 제한해도 이미 시작된 요청이 오래 붙잡히면 대기열과 사용자 지연은 계속 늘어납니다.
따라서 연결 풀 제한은 `AbortSignal.timeout()`이나 요청 단위 deadline과 함께 써야 합니다.

```js
export async function fetchJson(url, {
  dispatcher,
  signal,
  timeoutMs = 800
} = {}) {
  const timeoutSignal = AbortSignal.timeout(timeoutMs);
  const signals = signal
    ? AbortSignal.any([signal, timeoutSignal])
    : timeoutSignal;

  const response = await fetch(url, {
    dispatcher,
    signal: signals,
    headers: {
      accept: 'application/json'
    }
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }

  return response.json();
}
```

이 helper는 dispatcher와 timeout을 같은 호출 경계에 둡니다.
중요한 API라면 timeout을 조금 길게 잡을 수 있고, 부가 기능이라면 더 짧게 실패시켜 fallback으로 넘길 수 있습니다.

### H3. timeout 값은 기능 우선순위를 반영해야 한다

모든 외부 요청에 같은 timeout을 쓰면 운영 의도가 흐려집니다.
예를 들어 결제 상태 확인과 추천 위젯 조회가 둘 다 3초를 기다린다면, 사용자가 실제로 기다려도 되는 작업과 빨리 포기해야 하는 작업이 구분되지 않습니다.

실무에서는 대략 아래처럼 나눠볼 수 있습니다.

- 인증·결제: 짧지만 너무 공격적이지 않은 timeout, 충분한 연결 예산
- 주문·계정 조회: 사용자 요청 예산 안에서 안정적인 timeout
- 추천·통계·배너: 짧은 timeout, 작은 연결 풀, fallback 필수
- 배치·백오피스: 사용자 요청과 분리된 별도 프로세스나 별도 정책

이 기준은 코드보다 문서와 리뷰에서 먼저 합의해야 합니다.
그래야 새 외부 API를 붙일 때 "어느 dispatcher를 쓰지?"라는 질문이 자연스럽게 생깁니다.

## 전역 dispatcher를 바꿀 때의 주의점

### H3. setGlobalDispatcher는 편하지만 영향 범위가 넓다

Undici는 `setGlobalDispatcher()`로 기본 dispatcher를 바꿀 수 있습니다.
애플리케이션 전체의 기본 HTTP 정책을 통일할 때는 편리하지만, 영향 범위가 넓기 때문에 신중해야 합니다.

```js
import { Agent, setGlobalDispatcher } from 'undici';

setGlobalDispatcher(new Agent({
  connections: 50,
  keepAliveTimeout: 10_000
}));
```

전역 설정은 "모든 fetch 기본값"을 바꾸는 선택입니다.
라이브러리 내부 호출, 테스트, 관리용 스크립트까지 영향을 받을 수 있으므로 서비스별 격리가 목적이라면 개별 dispatcher를 명시하는 편이 더 안전합니다.

### H3. 테스트에서는 dispatcher 주입이 더 검증하기 쉽다

전역 dispatcher를 쓰면 테스트 간 상태가 섞이기 쉽습니다.
반대로 함수 인자로 dispatcher를 주입하면 테스트에서 mock dispatcher나 작은 연결 정책을 넣어 동작을 분리해 확인할 수 있습니다.

```js
export function createUserClient({ dispatcher, baseUrl }) {
  return {
    async getUser(userId, { signal } = {}) {
      const response = await fetch(`${baseUrl}/users/${userId}`, {
        dispatcher,
        signal
      });

      if (!response.ok) {
        throw new Error(`User API failed: ${response.status}`);
      }

      return response.json();
    }
  };
}
```

이런 형태는 운영 코드와 테스트 코드가 같은 경계를 공유합니다.
테스트 안정화 관점에서는 [Node.js test runner async activity 정리 가이드](/development/blog/seo/2026/08/29/nodejs-test-runner-async-activity-cleanup-guide.html)처럼 요청과 리소스를 테스트 생명주기에 맞춰 닫는 기준도 함께 필요합니다.

## 운영 체크리스트

### H3. 관측 지표를 dispatcher 단위로 나눈다

dispatcher를 나눴다면 로그와 지표도 같은 단위로 나눠야 합니다.
연결 풀을 분리했는데 모든 외부 요청을 `external_api` 하나로만 집계하면 병목을 찾기 어렵습니다.

최소한 아래 값은 upstream별로 볼 수 있어야 합니다.

- 요청 수와 에러율
- p95, p99 지연 시간
- timeout 발생 수
- fallback 발생 수
- 429, 503, 504 응답 비율
- 재시도 횟수와 최종 실패율

지표 이름에는 서비스명, 기능 우선순위, 에러 코드를 포함하는 편이 좋습니다.
관측 이벤트 설계는 [Node.js diagnostics_channel 계측 가이드](/development/blog/seo/2026/06/19/nodejs-diagnostics-channel-instrumentation-guide.html)와 연결해서 볼 수 있습니다.

### H3. dispatcher 정책은 설정값만이 아니라 계약이다

HTTP 연결 풀 설정은 성능 튜닝 값처럼 보이지만, 실제로는 서비스 간 계약에 가깝습니다.
"recommendation은 최대 연결 5개, 700ms 안에 실패, 실패하면 기본 리스트 노출"처럼 적어두면 장애 상황에서 판단이 빨라집니다.

새 API를 추가할 때는 아래 질문을 체크합니다.

- 이 upstream은 핵심 경로인가, 부가 기능인가?
- 실패하면 사용자에게 어떤 fallback을 보여줄 수 있는가?
- timeout은 전체 요청 예산의 몇 퍼센트를 써도 되는가?
- 연결 풀을 기존 dispatcher와 공유해도 되는가?
- 장애 시 어떤 로그와 지표로 원인을 찾을 수 있는가?

이 질문에 답하지 못한 채 외부 요청을 추가하면, 작은 기능 하나가 전체 응답 시간을 흔드는 지점이 됩니다.

## FAQ

### Node.js fetch는 반드시 Undici Agent를 직접 써야 하나요?

아닙니다.
작은 서비스나 호출이 적은 스크립트에서는 기본 `fetch()`만으로도 충분합니다.
다만 외부 API가 여러 개이고 장애 영향 범위를 줄여야 한다면 dispatcher를 명시하는 편이 운영 의도를 더 잘 드러냅니다.

### dispatcher와 AbortSignal 중 무엇이 더 중요한가요?

역할이 다릅니다.
dispatcher는 연결과 요청 처리 경계를 제어하고, `AbortSignal`은 개별 요청을 언제 끝낼지 결정합니다.
실무에서는 둘을 함께 써야 느린 upstream과 무제한 대기 문제를 같이 줄일 수 있습니다.

### 모든 API마다 Agent를 따로 만들면 되나요?

무조건 많이 나누는 것도 답은 아닙니다.
장애 영향 범위, 트래픽 규모, 우선순위가 비슷한 API는 같은 dispatcher를 공유할 수 있습니다.
중요한 것은 "왜 공유하는지"와 "언제 분리할지"를 팀이 설명할 수 있는 상태입니다.

## 마무리

Node.js fetch dispatcher는 단순한 옵션처럼 보이지만, 운영 관점에서는 외부 의존성의 영향 범위를 정하는 중요한 설계 지점입니다.
기본 `fetch()`로 시작하더라도 서비스가 커지면 upstream별 Agent, timeout, fallback, 관측 지표를 함께 정리해야 합니다.

핵심은 모든 외부 요청을 똑같이 대하지 않는 것입니다.
중요한 경로는 충분히 보호하고, 부가 기능은 빠르게 실패하고, 느린 upstream이 전체 서버의 연결과 시간을 독점하지 못하게 만드세요.
그 기준이 코드에 드러날 때 fetch 호출은 단순한 네트워크 요청을 넘어 안정성 경계가 됩니다.

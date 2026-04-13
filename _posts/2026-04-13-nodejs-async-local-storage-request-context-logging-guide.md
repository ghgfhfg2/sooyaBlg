---
layout: post
title: "Node.js AsyncLocalStorage 가이드: 요청 컨텍스트와 로그 추적을 안정적으로 연결하는 법"
date: 2026-04-13 20:00:00 +0900
lang: ko
translation_key: nodejs-async-local-storage-request-context-logging-guide
permalink: /development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html
alternates:
  ko: /development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html
  x_default: /development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, asynclocalstorage, request-context, logging, tracing, observability, backend, debugging]
description: "Node.js AsyncLocalStorage를 사용해 requestId, userId, trace 정보를 비동기 흐름 전체에 안전하게 전달하고, 로그 상관관계와 장애 분석 품질을 높이는 실무 방법을 정리했습니다."
---

운영 중인 Node.js 서비스에서 가장 답답한 순간 중 하나는 에러 로그는 많은데 **어느 요청에서 시작된 문제인지 연결이 안 될 때**입니다.
특히 비동기 함수가 여러 계층으로 흩어져 있으면 requestId, userId, trace 정보를 매번 함수 인자로 넘기는 방식은 금방 지저분해집니다.

이럴 때 실무에서 꽤 유용한 도구가 **AsyncLocalStorage**입니다.
이 기능을 잘 쓰면 요청 단위 컨텍스트를 비동기 흐름 전체에 유지하면서, 로그 상관관계 분석과 장애 추적 품질을 크게 높일 수 있습니다.
이 글에서는 Node.js AsyncLocalStorage가 왜 필요한지, 어떤 식으로 구성해야 안정적인지, 그리고 운영에서 흔히 겪는 함정을 어떻게 피할지 정리합니다.

## Node.js AsyncLocalStorage가 필요한 이유

### H3. 비동기 흐름이 길어질수록 로그 문맥이 쉽게 사라진다

간단한 API는 함수 인자로 `requestId`를 계속 넘겨도 버틸 수 있습니다.
하지만 실제 서비스는 controller, service, repository, queue producer, 외부 API client, logger가 여러 계층으로 나뉘어 있습니다.
이 구조에서 모든 함수 시그니처에 컨텍스트를 추가하기 시작하면 코드가 빠르게 무거워집니다.

특히 아래 같은 문제가 자주 생깁니다.

- 어떤 계층에서 `requestId` 전달을 빠뜨림
- background task나 callback에서 사용자 문맥이 끊김
- 로그는 남았지만 서로 같은 요청인지 연결이 어려움
- 장애 분석 시 시간만 오래 쓰고 원인 수렴은 느림

즉 AsyncLocalStorage는 편의 기능이라기보다, **비동기 환경에서 문맥 보존을 위한 운영 도구**에 가깝습니다.

### H3. requestId를 일관되게 묶어야 장애 분석 속도가 빨라진다

운영에서는 에러 한 줄보다 그 앞뒤 문맥이 더 중요합니다.
같은 요청 안에서 남은 access log, business log, DB 에러, 외부 API 실패 로그가 하나의 requestId로 묶여 있으면 분석 속도가 눈에 띄게 빨라집니다.

특히 아래 상황에서 효과가 큽니다.

- timeout 원인을 계층별로 좁혀야 할 때
- 특정 사용자 요청에서만 간헐 오류가 날 때
- 외부 API 호출 실패와 내부 재시도 로그를 연결해야 할 때
- queue 적체나 tail latency 원인을 따라가야 할 때

이런 관점은 [Node.js Deadline Exceeded 에러 처리 가이드](/development/blog/seo/2026/04/12/nodejs-deadline-exceeded-error-handling-guide.html)나 [Node.js Event Loop Lag 모니터링 가이드](/development/blog/seo/2026/04/13/nodejs-event-loop-lag-monitoring-guide.html)처럼 운영 지표를 읽는 글과도 잘 이어집니다.

## AsyncLocalStorage는 어떻게 동작하나

### H3. 요청 시작 시 store를 만들고 이후 비동기 체인에서 꺼내 쓴다

AsyncLocalStorage는 현재 실행 흐름에 연결된 저장소를 제공하는 API입니다.
보통 HTTP 요청이 들어오는 시점에 store를 만들고, 그 안에 requestId, traceId, userId 같은 값을 넣습니다.
그 뒤 같은 비동기 체인 안에서는 어느 계층에서든 store를 읽을 수 있습니다.

기본 구조는 아래처럼 잡을 수 있습니다.

```js
const { AsyncLocalStorage } = require('node:async_hooks');
const crypto = require('node:crypto');

const requestContext = new AsyncLocalStorage();

function contextMiddleware(req, res, next) {
  const store = {
    requestId: req.headers['x-request-id'] || crypto.randomUUID(),
    userId: req.user?.id || null,
    path: req.path,
    method: req.method,
  };

  requestContext.run(store, () => next());
}

function getContext() {
  return requestContext.getStore() || {};
}
```

핵심은 요청 시작점에서 `run()`으로 store를 만들고, 이후에는 `getStore()`로 읽는 방식입니다.
이 패턴을 기준으로 logger를 감싸면 로그마다 공통 필드를 자동으로 붙일 수 있습니다.

### H3. logger와 묶어야 실무 가치가 커진다

AsyncLocalStorage를 넣어도 로그 시스템과 연결하지 않으면 효과가 반쯤 줄어듭니다.
실무에서는 보통 logger wrapper를 하나 두고, 컨텍스트를 자동 병합합니다.

```js
const pino = require('pino');
const logger = pino();

function logInfo(message, extra = {}) {
  const context = requestContext.getStore() || {};
  logger.info({ ...context, ...extra }, message);
}

async function fetchOrder(orderId) {
  logInfo('fetch order start', { orderId });
  // ...
  logInfo('fetch order end', { orderId });
}
```

이렇게 하면 서비스 계층에서 매번 requestId를 인자로 넘기지 않아도 됩니다.
대신 로깅 규칙이 한 곳으로 모여서 유지보수도 훨씬 쉬워집니다.

## Express나 Fastify에서 적용할 때의 기본 패턴

### H3. 가장 앞단 미들웨어에서 컨텍스트를 심는다

Express 기준으로는 request lifecycle 초반에 미들웨어를 두는 편이 안전합니다.
인증, 비즈니스 로직, DB 접근 전에 컨텍스트가 먼저 생성돼야 하기 때문입니다.

```js
const express = require('express');
const app = express();

app.use(contextMiddleware);

app.get('/orders/:id', async (req, res) => {
  const context = getContext();
  logInfo('handle order request', { orderId: req.params.id });
  res.json({ ok: true, requestId: context.requestId });
});
```

가능하면 아래 정보까지 함께 넣는 것이 실무적으로 유용합니다.

- requestId
- traceId 또는 correlationId
- userId 또는 tenantId
- route path
- method
- 배포 버전 또는 instance id

이 정보가 있어야 장애를 볼 때 로그를 단순 나열이 아니라 맥락 있는 사건 흐름으로 읽을 수 있습니다.

### H3. 외부 호출과 DB 로그에도 같은 문맥을 붙인다

진짜 효과는 애플리케이션 로그만이 아니라 외부 연동 구간까지 문맥이 이어질 때 나옵니다.
예를 들어 HTTP client wrapper나 query logger에서도 store를 읽도록 만들면, 어느 요청이 어떤 downstream 호출을 만들었는지 한 번에 추적할 수 있습니다.

이 방식은 [Node.js Connection Pool Exhaustion 방지 가이드](/development/blog/seo/2026/04/11/nodejs-connection-pool-exhaustion-prevention-guide.html)처럼 DB 대기 시간 문제를 분석할 때도 꽤 유용합니다.
단순히 쿼리가 느렸다는 사실보다, **어느 요청 패턴이 풀 고갈을 만들었는지**를 보기 쉬워지기 때문입니다.

## AsyncLocalStorage를 쓸 때 흔한 실수

### H3. store에 너무 많은 데이터를 넣지 않는다

가장 흔한 실수는 컨텍스트 저장소를 만능 캐시처럼 쓰는 것입니다.
requestId, traceId 정도는 가볍지만, 큰 객체나 응답 payload 전체를 넣기 시작하면 메모리 부담과 디버깅 복잡성이 같이 커집니다.

보통 store에는 아래 정도만 두는 편이 적절합니다.

- 짧은 식별자 값
- 로깅과 추적에 꼭 필요한 메타데이터
- 전역처럼 읽되 요청 단위로 달라지는 값

반대로 아래는 피하는 편이 좋습니다.

- 대형 객체 전체
- 민감한 개인정보 원문
- access token, session secret 같은 비밀값
- 요청 바디 전체 복사본

가이드라인 관점에서도 민감정보를 남기지 않는 것이 중요합니다.
로그 추적을 강화하겠다고 해서 개인정보나 인증정보를 컨텍스트에 넣는 건 좋은 설계가 아닙니다.

### H3. 비동기 경계마다 항상 유지된다고 과신하면 안 된다

최근 Node.js에서는 AsyncLocalStorage 안정성이 많이 좋아졌지만, 라이브러리 조합이나 우회적인 비동기 경계에서 예상과 다르게 문맥이 비는 경우는 여전히 만날 수 있습니다.
그래서 운영 전에는 꼭 검증이 필요합니다.

추천하는 점검 방식은 아래와 같습니다.

- HTTP 요청 시작부터 응답 직전까지 requestId가 유지되는지 확인
- DB client, message queue, 외부 API client wrapper에서 문맥이 보이는지 확인
- 에러 핸들러와 retry 로직에서도 같은 ID가 이어지는지 테스트
- 배치나 worker 프로세스는 별도 컨텍스트 시작점을 명시적으로 설계

특히 재시도 로직이 섞이면 로그가 혼란스러워질 수 있습니다.
이 부분은 [Node.js Exponential Backoff와 Jitter 가이드](/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html)처럼 재시도 정책과 함께 보면 더 명확합니다.

## 운영에서 더 잘 쓰는 방법

### H3. requestId만 말고 trace 관점으로 확장한다

처음에는 requestId 하나만으로 시작해도 괜찮습니다.
하지만 서비스가 커지면 내부 서비스 호출, 비동기 작업, 메시지 큐 소비까지 이어지므로 correlationId 또는 traceId 개념으로 확장하는 편이 좋습니다.

예를 들어 아래처럼 구분할 수 있습니다.

- requestId: 현재 HTTP 요청 단위 식별자
- traceId: 여러 서비스나 작업 흐름을 묶는 상위 식별자
- userId 또는 tenantId: 비즈니스 분석용 문맥

이 구분이 있으면 단일 요청 추적뿐 아니라, 한 사용자 행동이 여러 시스템에서 어떻게 번지는지도 훨씬 잘 보입니다.

### H3. 구조화 로그와 함께 써야 검색성이 좋아진다

텍스트 로그에 requestId를 문자열로 억지 삽입하는 방식보다, JSON 구조화 로그에 필드를 분리해서 넣는 편이 훨씬 낫습니다.
그래야 Kibana, Datadog, Cloud Logging 같은 도구에서 필터링과 집계가 쉬워집니다.

실무적으로는 아래 기준을 추천합니다.

- requestId, traceId는 최상위 필드로 고정
- userId는 있으면 넣고 없으면 생략 또는 null 처리
- 에러 로그에는 service, operation, downstream 정보도 함께 기록
- 로그 포맷은 전 서비스에서 최대한 일관되게 유지

이 원칙이 있어야 검색 품질이 올라가고, 나중에 관측성 도구를 바꿔도 손해가 적습니다.

## 실무 체크리스트

### H3. 도입 전에 이 항목부터 확인한다

Node.js AsyncLocalStorage를 운영 환경에 넣기 전에 아래 항목을 점검하면 시행착오를 많이 줄일 수 있습니다.

- 요청 시작 미들웨어에서 store를 생성하는가
- logger wrapper가 store를 자동으로 병합하는가
- 민감정보를 store에 넣지 않도록 규칙이 있는가
- DB, 외부 API, retry 로직까지 같은 문맥이 이어지는가
- requestId와 traceId 역할을 팀 내에서 구분했는가

이 다섯 가지가 정리되면 AsyncLocalStorage는 단순한 Node.js API가 아니라, 장애 분석 시간을 줄여 주는 운영 기반이 됩니다.

## 마무리

Node.js AsyncLocalStorage는 코드 몇 줄로 끝나는 작은 기능처럼 보이지만, 실제로는 로그 품질과 디버깅 속도를 크게 바꾸는 도구입니다.
requestId를 함수 인자에 억지로 흘려보내는 구조가 한계에 부딪혔다면, 요청 컨텍스트를 런타임 차원에서 관리하는 쪽이 훨씬 안정적입니다.

핵심은 간단합니다.
**비동기 코드가 복잡해질수록 문맥 보존은 선택이 아니라 필수에 가깝습니다.**
AsyncLocalStorage를 logger, 외부 호출, 에러 처리 흐름과 함께 묶어 두면 Node.js 운영에서 로그를 훨씬 믿을 만하게 읽을 수 있습니다.

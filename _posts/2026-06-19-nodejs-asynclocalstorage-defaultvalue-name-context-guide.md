---
layout: post
title: "Node.js AsyncLocalStorage defaultValue·name 가이드: 컨텍스트 누락을 안전하게 다루기"
date: 2026-06-19 08:00:00 +0900
lang: ko
translation_key: nodejs-asynclocalstorage-defaultvalue-name-context-guide
permalink: /development/blog/seo/2026/06/19/nodejs-asynclocalstorage-defaultvalue-name-context-guide.html
alternates:
  ko: /development/blog/seo/2026/06/19/nodejs-asynclocalstorage-defaultvalue-name-context-guide.html
  x_default: /development/blog/seo/2026/06/19/nodejs-asynclocalstorage-defaultvalue-name-context-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, asynclocalstorage, async_hooks, logging, observability, context, backend]
description: "Node.js AsyncLocalStorage의 defaultValue와 name 옵션을 활용해 요청 컨텍스트 누락을 줄이고, 로그·관측성 코드를 더 명확하게 운영하는 방법을 실무 예제로 정리합니다."
---

Node.js 서비스에서 요청 ID, trace ID, 작업 이름 같은 값을 비동기 흐름 전체에 전파할 때 `AsyncLocalStorage`는 매우 유용합니다.
하지만 실제 운영 코드에서는 컨텍스트가 항상 존재한다고 가정하기 어렵습니다.
HTTP 요청 안에서는 값이 있지만, 테스트 코드·배치 초기화·헬스체크·일부 이벤트 핸들러에서는 `getStore()`가 `undefined`를 반환할 수 있습니다.

Node.js의 `AsyncLocalStorage` 생성자는 `defaultValue`와 `name` 옵션을 지원합니다.
[Node.js 공식 문서](https://nodejs.org/api/async_context.html#new-asynclocalstorageoptions) 기준으로 `defaultValue`는 스토어가 없을 때 사용할 기본값이고, `name`은 인스턴스에 붙이는 이름입니다.
이 글에서는 두 옵션을 로그·관측성 코드에 어떻게 적용하면 좋은지, 그리고 어디까지 기대해야 안전한지 정리합니다.

기본적인 요청 컨텍스트 전파 패턴이 먼저 필요하다면 [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)를 함께 참고하세요.
로그 필드 정책과 샘플링 전략은 [Node.js 로그 샘플링 가이드](/development/blog/seo/2026/06/18/nodejs-log-sampling-redaction-observability-guide.html)와 같이 보면 좋습니다.

## AsyncLocalStorage defaultValue가 해결하는 문제

### H3. 컨텍스트가 없는 구간에서도 안전한 기본 구조를 반환한다

기존에는 `getStore()`를 호출할 때마다 옵셔널 체이닝이나 fallback 객체를 반복해서 붙이는 경우가 많았습니다.

```js
const store = requestContext.getStore();

logger.info({
  requestId: store?.requestId ?? 'unknown',
  traceId: store?.traceId ?? 'none',
}, 'job started');
```

이 방식은 단순하지만 코드가 커질수록 누락이 생기기 쉽습니다.
어떤 파일은 `unknown`을 쓰고, 어떤 파일은 `undefined`를 그대로 남기고, 어떤 파일은 빈 문자열을 남기면 로그 검색 기준도 흔들립니다.

`defaultValue`를 사용하면 컨텍스트가 없을 때의 기본 모양을 한 곳에서 정할 수 있습니다.

```js
import { AsyncLocalStorage } from 'node:async_hooks';

const DEFAULT_CONTEXT = Object.freeze({
  requestId: 'no-request',
  traceId: 'no-trace',
  actorType: 'system',
});

export const requestContext = new AsyncLocalStorage({
  defaultValue: DEFAULT_CONTEXT,
  name: 'request-context',
});
```

이제 컨텍스트 밖에서 `getStore()`를 호출해도 로그 코드가 기대하는 기본 필드를 받을 수 있습니다.
핵심은 "값이 항상 의미 있다"가 아니라 "로그 구조가 항상 일정하다"는 점입니다.

### H3. 로거와 메트릭 코드의 분기 처리를 줄인다

관측성 코드는 애플리케이션 곳곳에서 호출됩니다.
그래서 매번 `if (!store)` 같은 방어 코드를 반복하면 읽기 어려워지고, 정작 중요한 로그 필드 설계가 흐려집니다.

```js
export function logInfo(message, fields = {}) {
  const ctx = requestContext.getStore();

  logger.info({
    requestId: ctx.requestId,
    traceId: ctx.traceId,
    actorType: ctx.actorType,
    ...fields,
  }, message);
}
```

이 패턴의 장점은 호출부가 단순해지는 것입니다.
서비스 함수는 컨텍스트가 있는지 없는지 매번 판단하지 않아도 되고, 로거는 항상 같은 필드 이름을 남깁니다.

다만 `defaultValue`가 비즈니스 로직의 입력 검증을 대신하지는 않습니다.
권한 판단, 결제 처리, 사용자별 데이터 조회처럼 실제 사용자 식별이 필요한 곳에서는 기본값을 신뢰하면 안 됩니다.

## name 옵션으로 운영 의도를 드러내기

### H3. 여러 컨텍스트 저장소를 구분하기 쉬워진다

큰 Node.js 서비스에는 요청 컨텍스트 하나만 있는 것이 아닙니다.
요청 단위 컨텍스트, 잡 실행 컨텍스트, 테스트 실행 컨텍스트처럼 목적이 다른 저장소가 생길 수 있습니다.

```js
export const requestContext = new AsyncLocalStorage({
  defaultValue: { requestId: 'no-request' },
  name: 'request-context',
});

export const jobContext = new AsyncLocalStorage({
  defaultValue: { jobId: 'no-job', queueName: 'unknown' },
  name: 'job-context',
});
```

`name`은 코드를 읽는 사람에게 이 저장소의 목적을 알려주는 작은 문서 역할을 합니다.
특히 공통 유틸이나 테스트 헬퍼에서 어떤 `AsyncLocalStorage` 인스턴스를 다루는지 명확히 보이는 장점이 있습니다.

### H3. 이름은 관측성 설계의 보조 정보로만 다룬다

`name`이 있다고 해서 자동으로 모든 로그나 추적 도구에 표시되는 것은 아닙니다.
운영 로그에 컨텍스트 이름을 남기고 싶다면 로거 코드에서 명시적으로 필드를 추가하는 편이 더 예측 가능합니다.

```js
export function getContextSnapshot() {
  const ctx = requestContext.getStore();

  return {
    contextName: requestContext.name,
    requestId: ctx.requestId,
    traceId: ctx.traceId,
    actorType: ctx.actorType,
  };
}
```

이때도 `contextName`을 과하게 남길 필요는 없습니다.
대부분의 요청 로그에는 `requestId`, `traceId`, `route`, `statusCode`, `elapsedMs` 같은 실행 분석 필드가 더 중요합니다.
`name`은 디버깅 헬퍼나 공통 진단 로그에서만 사용해도 충분합니다.

## 실무 적용 패턴

### H3. 기본값은 작고 불변에 가깝게 만든다

`defaultValue`에는 큰 객체나 변경 가능한 상태를 넣지 않는 편이 좋습니다.
기본값 객체가 여러 호출에서 공유될 수 있다고 생각하고, 읽기 전용 상수처럼 다뤄야 합니다.

```js
const DEFAULT_REQUEST_CONTEXT = Object.freeze({
  requestId: 'no-request',
  traceId: 'no-trace',
  userHash: null,
  route: 'unknown',
});
```

배열이나 중첩 객체처럼 나중에 수정될 수 있는 값을 넣으면 의도하지 않은 공유 상태처럼 보일 수 있습니다.
컨텍스트 기본값은 "로그 구조를 맞추기 위한 최소 필드"로 제한하는 편이 안전합니다.

### H3. 요청 안에서는 run()으로 명시적인 store를 넣는다

`defaultValue`를 도입해도 요청 시작 지점에서 실제 store를 만드는 일은 그대로 필요합니다.

```js
import crypto from 'node:crypto';

export function withRequestContext(req, handler) {
  const store = {
    requestId: req.headers['x-request-id'] ?? crypto.randomUUID(),
    traceId: req.headers['traceparent'] ?? 'local-trace',
    userHash: req.user ? hashUserId(req.user.id) : null,
    route: `${req.method} ${req.route?.path ?? req.url}`,
  };

  return requestContext.run(store, handler);
}
```

여기서 중요한 원칙은 민감정보를 컨텍스트에 넣지 않는 것입니다.
이메일, 전화번호, 원문 토큰, 세션 쿠키, 결제 식별자처럼 노출되면 안 되는 값은 store에 직접 저장하지 않습니다.
필요하다면 마스킹이나 해시를 거친 요약값만 넣습니다.

### H3. 테스트에서는 기본값과 실제 store를 모두 검증한다

컨텍스트 코드는 장애 상황에서 로그 품질에 직접 영향을 줍니다.
따라서 테스트에서는 두 가지 흐름을 확인하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { requestContext, getContextSnapshot } from './request-context.js';

test('returns default context outside request scope', () => {
  assert.equal(getContextSnapshot().requestId, 'no-request');
});

test('returns request context inside request scope', () => {
  requestContext.run({ requestId: 'req-123', traceId: 'trace-123', actorType: 'user' }, () => {
    assert.equal(getContextSnapshot().requestId, 'req-123');
  });
});
```

첫 번째 테스트는 컨텍스트 밖에서 로거가 깨지지 않는지 확인합니다.
두 번째 테스트는 실제 요청 범위에서 명시적으로 넣은 store가 기본값보다 우선하는지 확인합니다.

## defaultValue를 쓰면 안 되는 경우

### H3. 누락을 반드시 실패로 드러내야 하는 로직에는 맞지 않는다

컨텍스트가 없으면 바로 실패해야 하는 코드도 있습니다.
예를 들어 사용자 권한 확인, 테넌트별 데이터 분리, 감사 로그의 필수 주체 기록처럼 누락이 위험한 영역입니다.

이런 곳에서는 기본값으로 조용히 넘어가기보다 명시적으로 예외를 던지는 함수가 더 적합합니다.

```js
export function requireRequestContext() {
  const ctx = requestContext.getStore();

  if (ctx.requestId === 'no-request') {
    throw new Error('Request context is required');
  }

  return ctx;
}
```

즉 `defaultValue`는 운영 로그를 안정화하는 도구이지, 필수 실행 조건을 숨기는 도구가 아닙니다.
실패해야 하는 곳과 fallback이 허용되는 곳을 함수 이름으로 분리하면 리뷰하기 쉬워집니다.

### H3. 기본값이 실제 데이터처럼 집계되지 않게 한다

`no-request`, `no-trace`, `unknown` 같은 값은 운영 대시보드에서 실제 사용자 흐름처럼 보이면 안 됩니다.
메트릭 집계나 로그 분석에서는 별도 필터를 두는 편이 좋습니다.

```js
function shouldCountAsRequestMetric(ctx) {
  return ctx.requestId !== 'no-request';
}
```

기본값이 자주 보인다면 그것도 하나의 신호입니다.
어떤 코드 경로가 컨텍스트 없이 실행되는지 확인하고, HTTP 요청·큐 작업·배치 작업 중 어디에 별도 컨텍스트 초기화가 필요한지 점검해야 합니다.

## 체크리스트

### H3. 도입 전 확인할 것

- `defaultValue`에 민감정보나 큰 객체를 넣지 않는다
- 기본값은 로그 구조를 맞추는 최소 필드로 제한한다
- 실제 요청·잡 실행 범위에서는 `run()`으로 명시적인 store를 넣는다
- 권한·감사·테넌트 분리처럼 필수 컨텍스트가 필요한 곳은 별도 `require...()` 함수로 실패시킨다
- 기본값이 운영 지표에서 실제 트래픽처럼 집계되지 않도록 필터를 둔다

## FAQ

### H3. defaultValue가 있으면 getStore()가 항상 안전한가요?

로그와 진단 코드에서는 더 안전해질 수 있습니다.
하지만 비즈니스 로직에서 사용자나 테넌트가 반드시 필요한 경우에는 기본값을 안전한 값으로 보면 안 됩니다.
그런 영역은 컨텍스트 누락을 예외로 드러내는 편이 좋습니다.

### H3. name 옵션은 꼭 넣어야 하나요?

필수는 아니지만 공통 모듈이나 대형 서비스에서는 넣는 편이 좋습니다.
요청 컨텍스트, 잡 컨텍스트, 테스트 컨텍스트처럼 여러 저장소가 있을 때 코드의 의도를 더 빨리 파악할 수 있습니다.

### H3. 기존 AsyncLocalStorage 코드에 바로 적용해도 되나요?

먼저 `getStore()`가 `undefined`일 때 실패해야 하는 코드와 fallback이 허용되는 코드를 나누는 것이 좋습니다.
그다음 로거·메트릭·진단 헬퍼처럼 구조적 기본값이 도움이 되는 영역부터 적용하면 리스크가 작습니다.

## 함께 읽기

- [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)
- [Node.js AsyncLocalStorage snapshot 컨텍스트 전달 가이드](/development/blog/seo/2026/05/01/nodejs-asynclocalstorage-snapshot-context-handoff-guide.html)
- [Node.js 로그 샘플링 가이드](/development/blog/seo/2026/06/18/nodejs-log-sampling-redaction-observability-guide.html)

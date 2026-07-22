---
layout: post
title: "Node.js AsyncLocalStorage defaultValue와 name 가이드: 요청 컨텍스트 기본값을 안전하게 다루기"
date: 2026-07-23 08:00:00 +0900
lang: ko
translation_key: nodejs-asynclocalstorage-defaultvalue-name-request-context-guide
permalink: /development/blog/seo/2026/07/23/nodejs-asynclocalstorage-defaultvalue-name-request-context-guide.html
alternates:
  ko: /development/blog/seo/2026/07/23/nodejs-asynclocalstorage-defaultvalue-name-request-context-guide.html
  x_default: /development/blog/seo/2026/07/23/nodejs-asynclocalstorage-defaultvalue-name-request-context-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, asynclocalstorage, async-context, request-context, logging, observability]
description: "Node.js AsyncLocalStorage의 defaultValue와 name 옵션으로 요청 컨텍스트 기본값, 로깅, 테스트 격리를 안전하게 설계하는 방법을 설명합니다. run(), getStore(), snapshot()과 함께 사용할 때의 실무 주의점까지 정리합니다."
---

요청 단위 로그를 남기거나 트레이스 ID를 전파할 때 `AsyncLocalStorage`는 편리합니다.
하지만 컨텍스트가 없는 곳에서 `getStore()`를 호출하면 `undefined`가 돌아오고, 이 값을 방어하지 않은 로깅 코드나 감사 로그 코드가 쉽게 흔들립니다.
반대로 기본값을 아무렇게나 넣으면 요청 밖에서 만든 로그가 실제 요청 로그처럼 보이는 문제도 생깁니다.

Node.js의 `AsyncLocalStorage` 생성자 옵션에는 `defaultValue`와 `name`을 줄 수 있습니다.
이 글에서는 두 옵션을 요청 컨텍스트, 구조화 로그, 테스트 코드에서 어떻게 써야 안전한지 정리합니다.
[Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html), [AsyncLocalStorage snapshot 컨텍스트 전달 가이드](/development/blog/seo/2026/05/01/nodejs-asynclocalstorage-snapshot-context-handoff-guide.html), [AsyncLocalStorage bind 콜백 컨텍스트 가이드](/development/blog/seo/2026/05/09/nodejs-asynclocalstorage-bind-callback-context-guide.html)와 함께 보면 비동기 컨텍스트 설계를 단계적으로 정리할 수 있습니다.

## defaultValue와 name이 필요한 상황

### H3. 컨텍스트가 없는 로그도 같은 모양으로 남기고 싶다

많은 서비스는 요청 안팎에서 같은 logger를 사용합니다.
요청 처리 중에는 `requestId`, `userId`, `route` 같은 값이 있고, 배치 작업이나 부팅 로그에는 그런 값이 없습니다.
이때 매번 `storage.getStore() ?? fallback`을 반복하면 코드가 지저분해지고, 누락된 방어 코드가 생기기 쉽습니다.

```js
import { AsyncLocalStorage } from 'node:async_hooks';

const requestContext = new AsyncLocalStorage({
  name: 'request-context',
  defaultValue: {
    requestId: 'system',
    route: 'background',
    userId: null
  }
});

export function getContext() {
  return requestContext.getStore();
}
```

이렇게 기본값을 두면 logger는 항상 같은 shape의 객체를 받을 수 있습니다.
다만 `defaultValue`는 "요청이 있다"는 뜻이 아닙니다.
기본값은 요청 밖의 코드가 깨지지 않도록 돕는 안전한 빈 컨텍스트에 가깝습니다.

### H3. name은 디버깅과 관찰 가능성의 라벨이다

`name`은 `AsyncLocalStorage` 인스턴스가 무엇을 담는지 설명하는 이름입니다.
한 프로세스 안에 요청 컨텍스트, 작업 컨텍스트, 테스트 컨텍스트처럼 여러 저장소가 있으면 변수명만으로 추적하기 어려워집니다.
`name`을 명확히 두면 진단 코드나 내부 도구에서 어떤 저장소를 보고 있는지 식별하기 쉽습니다.

```js
const requestContext = new AsyncLocalStorage({
  name: 'request-context',
  defaultValue: { requestId: 'system' }
});

const jobContext = new AsyncLocalStorage({
  name: 'job-context',
  defaultValue: { jobId: 'none' }
});
```

이름은 짧고 안정적인 문자열이 좋습니다.
환경별로 바뀌는 값이나 요청별 ID를 넣는 용도로 쓰지 말고, 인스턴스의 역할을 나타내는 라벨로 두는 편이 관리하기 쉽습니다.

## 요청 컨텍스트에 적용하기

### H3. 요청 경계에서는 run()으로 실제 값을 넣는다

HTTP 요청이 들어온 순간에는 `run()`으로 실제 요청 컨텍스트를 명확히 시작해야 합니다.
기본값이 있다고 해서 요청 경계를 생략하면 모든 로그가 `system` 같은 기본값으로 찍혀 추적성이 사라집니다.

```js
import http from 'node:http';
import { randomUUID } from 'node:crypto';
import { AsyncLocalStorage } from 'node:async_hooks';

const requestContext = new AsyncLocalStorage({
  name: 'request-context',
  defaultValue: {
    requestId: 'system',
    route: 'background',
    userId: null
  }
});

function log(message, extra = {}) {
  const context = requestContext.getStore();
  console.log(JSON.stringify({
    level: 'info',
    message,
    requestId: context.requestId,
    route: context.route,
    ...extra
  }));
}

http.createServer((req, res) => {
  const context = {
    requestId: req.headers['x-request-id'] ?? randomUUID(),
    route: req.url ?? 'unknown',
    userId: null
  };

  requestContext.run(context, async () => {
    log('request.start');

    res.setHeader('content-type', 'application/json');
    res.end(JSON.stringify({ ok: true }));

    log('request.finish');
  });
}).listen(3000);
```

여기서 logger는 요청 밖에서도 동작하지만, 요청 안에서는 반드시 `run()`으로 들어온 값이 우선합니다.
운영 로그에서 `requestId: "system"`이 자주 보인다면 요청 경계가 빠졌거나 비동기 컨텍스트가 끊긴 지점을 의심해야 합니다.

### H3. defaultValue 객체는 불변처럼 다룬다

`defaultValue`에 객체를 넣을 때 가장 조심해야 할 부분은 변경입니다.
기본 객체를 직접 수정하면 요청 밖 코드들이 같은 객체를 공유하는 것처럼 보일 수 있고, 테스트 간 상태가 섞인 것처럼 보이는 혼란이 생깁니다.

```js
// 피해야 할 패턴
requestContext.getStore().requestId = 'manual-change';
```

컨텍스트 값은 생성 시점에 완성하고, 이후에는 읽기 전용처럼 다루는 편이 안전합니다.
값을 바꿔야 한다면 기존 객체를 수정하지 말고 새 객체로 `run()` 범위를 다시 시작하는 방식을 선택합니다.

```js
function withUser(userId, callback) {
  const current = requestContext.getStore();

  return requestContext.run({
    ...current,
    userId
  }, callback);
}
```

이 방식은 요청 처리 도중 인증 정보가 확정되는 흐름에 잘 맞습니다.
처음에는 `userId: null`로 시작하고, 인증 이후에는 더 구체적인 컨텍스트를 가진 하위 범위에서 코드를 실행할 수 있습니다.

## 로깅과 테스트에서의 실무 기준

### H3. 로그에서는 기본값과 실제 요청값을 구분한다

기본값을 쓰면 로그 shape는 안정적이지만, 분석 단계에서 요청 로그와 시스템 로그를 구분할 수 있어야 합니다.
`source`나 `contextType` 같은 필드를 두면 쿼리할 때 편합니다.

```js
const requestContext = new AsyncLocalStorage({
  name: 'request-context',
  defaultValue: {
    contextType: 'system',
    requestId: 'system',
    route: 'background',
    userId: null
  }
});

function startRequestContext(req, callback) {
  return requestContext.run({
    contextType: 'request',
    requestId: req.headers['x-request-id'] ?? randomUUID(),
    route: req.url ?? 'unknown',
    userId: null
  }, callback);
}
```

이렇게 하면 대시보드에서 `contextType: "request"`만 모아 요청 지연 시간과 에러율을 볼 수 있습니다.
배치 작업이나 서버 부팅 로그는 같은 logger를 쓰더라도 별도의 시스템 이벤트로 남습니다.

### H3. 테스트에서는 기본값을 명시적으로 검증한다

`defaultValue`를 도입하면 "컨텍스트가 없으면 undefined"라는 오래된 가정이 깨질 수 있습니다.
따라서 logger, 인증 미들웨어, 테스트 helper처럼 `getStore()`를 직접 호출하는 코드에는 기본값 동작을 테스트로 고정해 두는 편이 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('requestContext returns a system context outside request scope', () => {
  assert.deepStrictEqual(getContext(), {
    contextType: 'system',
    requestId: 'system',
    route: 'background',
    userId: null
  });
});

test('requestContext prefers the scoped request value', async () => {
  await requestContext.run({
    contextType: 'request',
    requestId: 'req-123',
    route: '/health',
    userId: null
  }, async () => {
    assert.equal(getContext().requestId, 'req-123');
    assert.equal(getContext().contextType, 'request');
  });
});
```

테스트 이름에는 기본값과 요청 범위 값을 분리해서 드러내는 것이 좋습니다.
나중에 `defaultValue`를 제거하거나 컨텍스트 shape를 바꿀 때 어떤 기대가 깨졌는지 바로 알 수 있습니다.

## 도입 전 체크리스트

### H3. 기본값은 안전하지만 조용한 실패를 만들 수도 있다

`defaultValue`는 방어 코드를 줄여 주지만, 컨텍스트 누락을 숨길 수도 있습니다.
요청 처리 코드에서 `system` 기본값이 찍히면 정상으로 넘기지 말고 경고나 메트릭으로 잡아야 합니다.

```js
function logRequestOnlyEvent(message) {
  const context = requestContext.getStore();

  if (context.contextType !== 'request') {
    console.warn(JSON.stringify({
      level: 'warn',
      message: 'request-context.missing',
      event: message
    }));
    return;
  }

  log(message);
}
```

모든 코드에 이 검사를 넣을 필요는 없습니다.
다만 결제, 권한 변경, 감사 로그처럼 반드시 요청 주체가 있어야 하는 이벤트에는 기본값을 그대로 통과시키지 않는 것이 안전합니다.

### H3. 민감정보는 컨텍스트에 넣지 않는다

요청 컨텍스트는 여러 함수와 라이브러리를 지나가므로 민감정보를 넣지 않는 편이 좋습니다.
액세스 토큰, 세션 쿠키, 원본 이메일, 주민등록번호 같은 값은 컨텍스트가 아니라 보안 경계가 있는 저장소에서 다뤄야 합니다.
컨텍스트에는 추적에 필요한 최소 식별자만 둡니다.

권장하는 값은 다음 정도입니다.

- `requestId`: 로그 상관관계용 난수 ID
- `traceId`: 분산 추적 시스템과 맞춘 ID
- `route`: 정규화된 라우트 이름
- `userId`: 내부 식별자 또는 마스킹된 값
- `contextType`: `request`, `job`, `system` 같은 분류

개인정보가 될 수 있는 값은 로그 출력 직전에 마스킹하는 것보다 처음부터 컨텍스트에 넣지 않는 방식이 더 안전합니다.

## 마무리

`AsyncLocalStorage`의 `defaultValue`는 요청 밖 코드도 안정적인 컨텍스트 shape를 받게 해 줍니다.
`name`은 여러 컨텍스트 저장소를 운영할 때 인스턴스의 역할을 분명히 해 줍니다.
하지만 기본값은 요청 경계를 대신하지 않습니다.
HTTP 요청, 작업 실행, 테스트 범위에서는 여전히 `run()`으로 실제 값을 넣어야 하고, 기본값이 찍히는 로그는 컨텍스트 누락 신호로 다뤄야 합니다.

실무에서는 먼저 컨텍스트 shape를 작게 정하고, 민감정보를 제외하고, 기본값과 실제 요청값을 테스트로 고정하는 순서가 좋습니다.
이렇게 해 두면 로깅 코드는 단순해지고, 비동기 컨텍스트가 끊긴 지점도 더 빨리 찾을 수 있습니다.

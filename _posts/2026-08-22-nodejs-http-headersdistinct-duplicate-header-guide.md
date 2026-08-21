---
layout: post
title: "Node.js headersDistinct 가이드: 중복 HTTP 헤더를 안전하게 검사하는 법"
date: 2026-08-22 08:00:00 +0900
lang: ko
translation_key: nodejs-http-headersdistinct-duplicate-header-guide
permalink: /development/blog/seo/2026/08/22/nodejs-http-headersdistinct-duplicate-header-guide.html
alternates:
  ko: /development/blog/seo/2026/08/22/nodejs-http-headersdistinct-duplicate-header-guide.html
  x_default: /development/blog/seo/2026/08/22/nodejs-http-headersdistinct-duplicate-header-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, http, headersdistinct, duplicate-headers, rawheaders, api-security, backend, javascript]
description: "Node.js message.headersDistinct로 중복 HTTP 헤더를 안전하게 검사하는 방법을 정리합니다. message.headers, rawHeaders, joinDuplicateHeaders와의 차이, 인증·프록시 환경에서의 검증 패턴까지 실무 예제로 설명합니다."
---

HTTP 헤더는 보통 하나의 이름에 하나의 값이 있다고 생각하기 쉽습니다.
하지만 실제 요청에서는 같은 이름의 헤더가 여러 번 들어올 수 있습니다.
`Cookie`, `Set-Cookie`, 캐시 관련 헤더처럼 중복이 자연스러운 경우도 있고, `Authorization`, `Host`, `Content-Length`처럼 중복이 들어오면 의도를 의심해야 하는 경우도 있습니다.

Node.js의 `message.headers`는 개발자가 편하게 쓰기 좋도록 헤더 이름을 소문자로 정리하고, 일부 중복 헤더를 병합하거나 버립니다.
이 동작은 일반적인 라우팅과 로깅에는 편하지만, 중복 헤더 자체를 검사해야 하는 보안·프록시·인증 코드에서는 정보가 이미 가공된 뒤일 수 있습니다.
이럴 때 `message.headersDistinct`를 보면 각 헤더 값을 배열로 확인할 수 있습니다.

이 글에서는 `headersDistinct`가 필요한 상황, `headers`, `rawHeaders`, `joinDuplicateHeaders`와의 차이, 그리고 실무에서 중복 헤더를 거절하거나 로그로 남기는 패턴을 정리합니다.
헤더 이름과 값 자체의 유효성 검사는 [Node.js http.validateHeaderName/Value 가이드](/development/blog/seo/2026/06/09/nodejs-http-validate-header-name-value-guide.html), 서버 타임아웃 기준은 [Node.js requestTimeout, timeout, headersTimeout 차이 가이드](/development/blog/seo/2026/04/16/nodejs-requesttimeout-timeout-headerstimeout-difference-guide.html), 프록시 뒤 keep-alive 설정은 [Node.js server.keepAliveTimeoutBuffer 가이드](/development/blog/seo/2026/08/17/nodejs-server-keepalivetimeoutbuffer-econnreset-guide.html)와 함께 보면 좋습니다.

## headersDistinct가 필요한 이유

### message.headers는 애플리케이션 편의용이다

Node.js HTTP 서버에서 요청을 받으면 가장 자주 보는 속성은 `req.headers`입니다.
이 객체는 헤더 이름을 소문자로 바꾸고, 헤더별 규칙에 따라 값을 문자열 또는 배열로 제공합니다.

```js
import http from 'node:http';

const server = http.createServer((req, res) => {
  console.log(req.headers);
  res.end('ok');
});

server.listen(3000);
```

대부분의 API 서버에서는 이 정도면 충분합니다.
`content-type`, `accept`, `user-agent`, `authorization`처럼 필요한 값을 읽고 라우터나 인증 미들웨어로 넘길 수 있습니다.

문제는 "중복으로 들어왔는가" 자체가 중요한 경우입니다.
공식 Node.js HTTP 문서 기준으로 `message.headers`는 헤더 이름에 따라 중복을 다르게 처리합니다.
일부 헤더의 중복은 버려지고, `set-cookie`는 배열로 유지되며, `cookie`는 세미콜론으로 합쳐지고, 그 밖의 헤더는 쉼표로 합쳐질 수 있습니다.
따라서 `req.headers.authorization`만 보고는 클라이언트가 `Authorization`을 두 번 보냈는지 확실히 알기 어렵습니다.

### headersDistinct는 항상 배열로 보여준다

`req.headersDistinct`는 `req.headers`와 비슷하지만 병합 로직을 적용하지 않습니다.
각 헤더 이름의 값은 항상 문자열 배열입니다.
헤더가 한 번만 들어와도 배열이고, 여러 번 들어오면 값이 여러 개 들어갑니다.

```js
const server = http.createServer((req, res) => {
  const distinct = req.headersDistinct;

  console.log({
    authorization: distinct.authorization,
    host: distinct.host,
    cookie: distinct.cookie
  });

  res.end('ok');
});
```

이 구조는 보안 검증에 유리합니다.
예를 들어 `authorization` 배열 길이가 2 이상이면 요청을 거절할 수 있습니다.
`host`가 여러 개라면 프록시 라우팅과 가상 호스트 판별이 애매해질 수 있으므로 명확한 정책을 적용할 수 있습니다.

`headersDistinct`를 쓰면 애플리케이션이 "최종으로 해석된 값"과 "수신된 값의 개수"를 분리해서 볼 수 있습니다.
일반 비즈니스 로직은 `req.headers`를 써도 되고, 경계 검증은 `req.headersDistinct`를 쓰는 방식이 읽기 좋습니다.

## 중복 헤더 정책 세우기

### 단일 값이어야 하는 헤더를 먼저 정한다

모든 중복 헤더를 무조건 거절하면 현실적인 요청까지 막을 수 있습니다.
반대로 모든 중복을 허용하면 인증, 라우팅, 캐시 판단에서 애매한 요청이 지나갈 수 있습니다.
먼저 프로젝트에서 단일 값이어야 하는 헤더 목록을 정하는 편이 좋습니다.

```js
const SINGLE_VALUE_HEADERS = [
  'authorization',
  'host',
  'content-length',
  'content-type',
  'x-request-id',
  'x-forwarded-proto'
];

export function findDuplicateSingleValueHeaders(headersDistinct) {
  return SINGLE_VALUE_HEADERS.filter((name) => {
    const values = headersDistinct[name];
    return Array.isArray(values) && values.length > 1;
  });
}
```

이 함수는 정책을 코드로 드러냅니다.
어떤 헤더를 단일 값으로 보는지 리뷰하기 쉽고, 프록시나 API gateway에서 추가하는 헤더도 별도로 검토할 수 있습니다.

단일 값 정책은 서비스 구조에 따라 달라질 수 있습니다.
예를 들어 `x-forwarded-for`는 프록시 체인 때문에 쉼표로 이어진 목록이 정상일 수 있습니다.
반면 `x-request-id`는 하나의 요청 추적 ID로 고정하는 편이 로그 상관관계에 좋습니다.

### 요청 초입에서 애매한 헤더를 거절한다

중복 헤더를 거절하려면 라우팅, 인증, body parsing보다 앞에서 처리해야 합니다.
뒤쪽 미들웨어가 이미 다른 방식으로 헤더를 해석한 뒤라면 정책이 흔들립니다.

```js
import http from 'node:http';

const server = http.createServer((req, res) => {
  const duplicates = findDuplicateSingleValueHeaders(req.headersDistinct);

  if (duplicates.length > 0) {
    res.writeHead(400, { 'content-type': 'application/json; charset=utf-8' });
    res.end(JSON.stringify({
      error: 'duplicate_single_value_header',
      headers: duplicates
    }));
    return;
  }

  res.writeHead(200, { 'content-type': 'text/plain; charset=utf-8' });
  res.end('ok');
});

server.listen(3000);
```

응답에는 실제 헤더 값을 넣지 않는 편이 좋습니다.
헤더에는 토큰, 쿠키, 내부 라우팅 정보가 들어갈 수 있기 때문입니다.
클라이언트에게는 어떤 헤더 이름이 정책 위반인지 정도만 알려도 충분합니다.

운영 로그도 마찬가지입니다.
중복된 헤더 이름, 요청 ID, 경로, 원격 주소 정도는 남기되 `authorization`이나 `cookie` 원문을 출력하지 않아야 합니다.

## headers, headersDistinct, rawHeaders 비교

### headers는 최종 해석용이다

`req.headers`는 애플리케이션 코드에서 가장 쓰기 좋은 형태입니다.
대부분의 헤더가 문자열로 들어오고, `set-cookie`처럼 배열이 필요한 경우는 배열로 유지됩니다.
헤더 이름도 소문자로 정리되어 있으므로 대소문자 차이를 신경 쓰지 않아도 됩니다.

```js
function readJsonRequest(req) {
  const contentType = req.headers['content-type'] ?? '';

  if (!contentType.includes('application/json')) {
    throw new Error('Expected JSON request');
  }
}
```

이 코드는 단순하고 실용적입니다.
다만 중복 여부가 중요한 경계 코드라면 `headers`만으로 판단하지 않는 것이 좋습니다.
중복이 병합되거나 일부 값이 버려진 뒤일 수 있기 때문입니다.

### headersDistinct는 검증용이다

`req.headersDistinct`는 헤더별 값 개수를 보고 싶을 때 적합합니다.
값이 항상 배열이므로 검사 함수도 예측 가능하게 작성할 수 있습니다.

```js
function readSingleHeader(req, name) {
  const values = req.headersDistinct[name.toLowerCase()] ?? [];

  if (values.length === 0) {
    return undefined;
  }

  if (values.length > 1) {
    throw new Error(`Duplicate header is not allowed: ${name}`);
  }

  return values[0];
}
```

이 함수는 인증 토큰, 요청 ID, 서명 헤더처럼 하나만 있어야 하는 값을 읽을 때 유용합니다.
값을 읽는 동시에 중복 정책을 적용하기 때문에 뒤쪽 코드가 애매한 상태를 받지 않습니다.

### rawHeaders는 원문 재구성용이다

`req.rawHeaders`는 Node.js가 받은 헤더 이름과 값을 순서대로 담은 배열입니다.
헤더 이름의 대소문자와 중복 순서를 그대로 보고 싶을 때 쓸 수 있습니다.

```js
function toHeaderPairs(rawHeaders) {
  const pairs = [];

  for (let index = 0; index < rawHeaders.length; index += 2) {
    pairs.push({
      name: rawHeaders[index],
      value: rawHeaders[index + 1]
    });
  }

  return pairs;
}
```

디버깅이나 프록시 호환성 분석에는 `rawHeaders`가 도움이 됩니다.
하지만 일반 검증에서는 직접 파싱해야 할 일이 늘어납니다.
중복 개수만 확인하면 된다면 `headersDistinct`가 더 안전하고 단순합니다.

## joinDuplicateHeaders와의 관계

### joinDuplicateHeaders는 해석 정책을 바꾼다

`http.createServer()`에는 `joinDuplicateHeaders` 옵션이 있습니다.
이 옵션을 켜면 일부 중복 헤더를 버리는 대신 쉼표로 결합하도록 `message.headers`의 동작이 달라집니다.

```js
const server = http.createServer({
  joinDuplicateHeaders: true
}, (req, res) => {
  console.log(req.headers.authorization);
  res.end('ok');
});
```

이 옵션은 RFC 기준의 헤더 결합 동작과 맞추고 싶을 때 고려할 수 있습니다.
하지만 보안 경계에서는 "결합해서 하나의 문자열로 만들기"가 항상 좋은 답은 아닙니다.
특히 인증, 서명, 라우팅에 영향을 주는 헤더라면 중복을 합치기보다 요청 자체를 거절하는 편이 더 명확합니다.

즉 `joinDuplicateHeaders`는 중복을 어떻게 해석할지에 대한 옵션이고, `headersDistinct`는 중복이 실제로 들어왔는지 확인하는 관찰 도구에 가깝습니다.
둘의 목적을 분리해서 봐야 합니다.

### 프록시 뒤에서는 더 보수적으로 본다

Node.js 서버가 Nginx, CDN, API gateway 뒤에 있다면 헤더는 이미 한 번 이상 가공되었을 수 있습니다.
이 환경에서는 클라이언트가 보낸 헤더와 애플리케이션이 보는 헤더가 다를 수 있고, 프록시가 추가한 헤더도 섞입니다.

따라서 프록시 뒤에서는 아래 기준을 문서화하는 것이 좋습니다.

- 애플리케이션이 직접 신뢰하는 헤더 이름
- 프록시가 덮어쓰는 헤더 이름
- 클라이언트가 보내면 거절할 헤더 이름
- 중복을 허용하는 헤더와 거절하는 헤더
- 로그에 남겨도 되는 헤더 이름과 마스킹할 헤더 이름

예를 들어 `x-forwarded-proto`를 프록시가 항상 덮어쓴다면 애플리케이션은 중복 여부보다 프록시 설정의 신뢰 경계를 먼저 봐야 합니다.
반대로 외부 클라이언트가 임의의 `x-internal-user-id`를 보낼 수 있다면, 애플리케이션 초입에서 제거하거나 거절해야 합니다.

## 실무 검증 미들웨어 예시

### HTTP 서버 공통 필터로 분리한다

Node.js 기본 `http` 서버만 쓰는 코드라면 요청 핸들러 초입에 작은 필터를 둘 수 있습니다.
프레임워크를 쓰더라도 같은 원칙으로 가장 앞쪽 미들웨어에 배치하면 됩니다.

```js
const SENSITIVE_HEADERS = new Set([
  'authorization',
  'cookie',
  'proxy-authorization'
]);

const SINGLE_VALUE_HEADERS = new Set([
  'authorization',
  'host',
  'content-length',
  'content-type',
  'x-request-id'
]);

export function validateRequestHeaders(req, logger) {
  const duplicates = [];

  for (const [name, values] of Object.entries(req.headersDistinct)) {
    if (SINGLE_VALUE_HEADERS.has(name) && values.length > 1) {
      duplicates.push(name);
    }
  }

  if (duplicates.length === 0) {
    return { ok: true };
  }

  logger.warn('duplicate single-value headers rejected', {
    path: req.url,
    method: req.method,
    duplicateHeaders: duplicates,
    containsSensitiveHeader: duplicates.some((name) => SENSITIVE_HEADERS.has(name))
  });

  return {
    ok: false,
    statusCode: 400,
    code: 'duplicate_single_value_header',
    headers: duplicates
  };
}
```

로그에는 값 대신 이름만 남깁니다.
민감 헤더가 포함되었는지도 boolean으로 남기면, 실제 토큰을 노출하지 않고도 위험도를 파악할 수 있습니다.

핸들러에서는 결과만 보고 일관된 오류 응답을 보냅니다.

```js
const server = http.createServer((req, res) => {
  const headerValidation = validateRequestHeaders(req, logger);

  if (!headerValidation.ok) {
    res.writeHead(headerValidation.statusCode, {
      'content-type': 'application/json; charset=utf-8'
    });
    res.end(JSON.stringify({
      error: headerValidation.code,
      headers: headerValidation.headers
    }));
    return;
  }

  routeRequest(req, res);
});
```

이렇게 분리하면 테스트도 쉬워집니다.
실제 네트워크 요청을 만들지 않아도 `headersDistinct` 모양의 객체를 넣어 정책을 검증할 수 있습니다.

### 테스트는 허용과 거절 케이스를 함께 둔다

중복 헤더 정책은 나중에 프록시 설정이나 인증 방식이 바뀌면서 흔들릴 수 있습니다.
작은 단위 테스트로 의도를 고정해 두면 회귀를 줄일 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('rejects duplicate authorization header', () => {
  const result = validateRequestHeaders({
    method: 'GET',
    url: '/me',
    headersDistinct: {
      authorization: ['Bearer first', 'Bearer second'],
      host: ['api.example.com']
    }
  }, silentLogger);

  assert.equal(result.ok, false);
  assert.deepEqual(result.headers, ['authorization']);
});

test('allows multiple cookie header values when policy permits it', () => {
  const result = validateRequestHeaders({
    method: 'GET',
    url: '/settings',
    headersDistinct: {
      cookie: ['a=1', 'b=2'],
      host: ['api.example.com']
    }
  }, silentLogger);

  assert.equal(result.ok, true);
});
```

테스트 이름에는 정책이 드러나야 합니다.
단순히 "works"라고 쓰기보다 어떤 헤더를 왜 허용하거나 거절하는지 남기는 편이 나중에 읽기 좋습니다.

## 운영 체크리스트

### 경계에서 한 번만 정리한다

헤더 검증은 여러 계층에 흩어지면 오히려 위험합니다.
라우터, 인증 모듈, 서비스 함수가 각자 다른 기준으로 `authorization`이나 `host`를 읽으면 요청 해석이 달라질 수 있습니다.

추천 흐름은 아래와 같습니다.

1. HTTP 서버 또는 가장 앞쪽 미들웨어에서 중복 헤더 정책을 적용한다.
2. 단일 값 헤더는 검증 후 정규화된 request context에 넣는다.
3. 뒤쪽 비즈니스 로직은 원본 헤더를 반복해서 해석하지 않는다.
4. 거절 로그에는 헤더 값이 아니라 헤더 이름과 요청 메타데이터만 남긴다.
5. 프록시가 추가하는 헤더는 인프라 문서와 함께 관리한다.

이 구조는 책임을 명확히 합니다.
중복 헤더는 경계에서 다루고, 애플리케이션 내부는 이미 검증된 값을 사용합니다.

### 관측 지표를 작게 남긴다

중복 헤더 거절이 갑자기 늘어나면 클라이언트 버그, 프록시 설정 변경, 보안 스캐너 트래픽을 의심할 수 있습니다.
운영 지표는 과하게 자세할 필요는 없지만 아래 정도는 도움이 됩니다.

- `duplicate_header_rejected_total`
- 중복 헤더 이름별 카운트
- 라우트 또는 서비스별 카운트
- 400 응답 비율
- 프록시 배포 직후 변화 여부

값 원문을 metric label로 넣는 것은 피해야 합니다.
카디널리티가 폭발하고 민감정보가 관측 시스템에 남을 수 있습니다.
헤더 이름도 필요한 목록으로 제한하는 편이 안전합니다.

## FAQ

### headersDistinct만 쓰면 rawHeaders는 필요 없나요?

대부분의 중복 검사에는 `headersDistinct`가 더 단순합니다.
하지만 헤더 이름의 원래 대소문자, 수신 순서, 프록시가 보낸 실제 원문 형태를 분석해야 한다면 `rawHeaders`가 필요할 수 있습니다.
운영 검증은 `headersDistinct`, 호환성 디버깅은 `rawHeaders`로 나누면 좋습니다.

### 모든 중복 헤더를 거절해도 되나요?

권장하지 않습니다.
`cookie`처럼 중복이 현실적으로 들어올 수 있는 헤더가 있고, 프록시나 클라이언트 라이브러리의 동작도 다양합니다.
인증, 라우팅, 본문 해석, 추적 ID처럼 단일 값이어야 하는 헤더부터 명시적으로 거절하는 방식이 안전합니다.

### joinDuplicateHeaders를 켜면 보안이 좋아지나요?

그 자체로 보안 기능이라고 보기는 어렵습니다.
`joinDuplicateHeaders`는 중복 헤더를 버릴지 결합할지에 대한 해석 옵션입니다.
보안상 애매한 요청을 줄이고 싶다면 단일 값 헤더의 중복을 `headersDistinct`로 확인한 뒤 거절하는 정책이 더 명확합니다.

## 마무리

Node.js HTTP 서버에서 `req.headers`는 편리하지만, 중복 헤더를 검사해야 하는 경계 코드에는 충분하지 않을 수 있습니다.
`req.headersDistinct`를 함께 보면 헤더 값이 몇 번 들어왔는지 배열로 확인할 수 있고, 인증·라우팅·프록시 관련 애매한 요청을 초기에 거절할 수 있습니다.

핵심은 모든 헤더를 복잡하게 다루는 것이 아닙니다.
단일 값이어야 하는 헤더를 정하고, 요청 초입에서 검증하고, 로그에는 값 대신 이름과 메타데이터만 남기세요.
그 정도만 해도 중복 헤더 때문에 발생하는 해석 차이와 보안 리스크를 꽤 줄일 수 있습니다.

## 함께 읽기

- [Node.js http.validateHeaderName/Value 가이드: HTTP 헤더 검증을 안전하게 처리하는 법](/development/blog/seo/2026/06/09/nodejs-http-validate-header-name-value-guide.html)
- [Node.js requestTimeout, timeout, headersTimeout 차이 가이드](/development/blog/seo/2026/04/16/nodejs-requesttimeout-timeout-headerstimeout-difference-guide.html)
- [Node.js server.keepAliveTimeoutBuffer 가이드: ECONNRESET을 줄이는 HTTP keep-alive 설정법](/development/blog/seo/2026/08/17/nodejs-server-keepalivetimeoutbuffer-econnreset-guide.html)
- [Node.js HTTP 공식 문서: message.headersDistinct와 중복 헤더 동작](https://nodejs.org/api/http.html#messageheadersdistinct)

---
layout: post
title: "Node.js URLPattern 가이드: API 라우트 매칭과 URL 검증을 구조적으로 다루는 법"
date: 2026-08-26 20:00:00 +0900
lang: ko
translation_key: nodejs-urlpattern-api-route-matching-guide
permalink: /development/blog/seo/2026/08/26/nodejs-urlpattern-api-route-matching-guide.html
alternates:
  ko: /development/blog/seo/2026/08/26/nodejs-urlpattern-api-route-matching-guide.html
  x_default: /development/blog/seo/2026/08/26/nodejs-urlpattern-api-route-matching-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, urlpattern, url, routing, api, edge, javascript, validation]
description: "Node.js URLPattern으로 API 라우트, 웹훅 경로, 외부 URL allowlist를 구조적으로 매칭하는 방법을 정리합니다. test와 exec 차이, 그룹 추출, baseURL, 실험적 API 주의점까지 실무 예제로 설명합니다."
---

API 서버나 프록시 코드를 만들다 보면 URL 경로를 문자열로 비교하는 코드가 금방 늘어납니다.
`pathname.startsWith('/api/users/')`, `pathname.includes('/webhook')`, 정규식 몇 개를 조합하다 보면 처음에는 간단했던 라우팅 코드가 점점 읽기 어려워집니다.
특히 경로 파라미터를 추출하거나, 특정 호스트만 허용하거나, 쿼리 문자열까지 조건에 넣어야 할 때는 문자열 분기만으로 정책을 설명하기 어렵습니다.

Node.js에는 WHATWG URL 계열 API와 함께 `URLPattern`이 문서화되어 있습니다.
Node.js v26.7.0 문서 기준으로 `URLPattern`은 v23.8.0에 추가되었고, 안정도는 Experimental로 표시되어 있습니다.
즉, 모든 운영 환경에서 무조건 기본 선택지로 삼기보다는 런타임 버전과 안정성 요구를 확인한 뒤 적용해야 합니다.
공식 문서는 `URLPattern`을 URL 또는 URL 구성 요소를 패턴과 매칭하는 인터페이스로 설명합니다.

이 글에서는 Node.js `URLPattern`을 사용해 API 라우트 매칭, 웹훅 경로 검증, 외부 URL allowlist를 구조적으로 다루는 방법을 정리합니다.
URL 파싱의 기본기는 [Node.js URL.canParse 가이드](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html), HTTP 헤더 검증은 [Node.js HTTP validateHeaderName/Value 가이드](/development/blog/seo/2026/06/09/nodejs-http-validate-header-name-value-guide.html), Content-Type 검증은 [Node.js MIMEType 가이드](/development/blog/seo/2026/08/26/nodejs-util-mimetype-content-type-validation-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js URL API 공식 문서](https://nodejs.org/api/url.html)

## URLPattern을 고려할 만한 상황

### 라우트 조건이 URL 구조를 따라갈 때

문자열 비교는 조건이 하나일 때는 충분합니다.
하지만 실제 서비스에서는 경로, 호스트, 프로토콜, 포트, 검색 파라미터 조건이 함께 등장합니다.

```js
const url = new URL(request.url);

if (
  url.protocol === 'https:' &&
  url.hostname === 'api.example.com' &&
  url.pathname.startsWith('/v1/users/')
) {
  // user route
}
```

이 정도 코드는 아직 읽을 수 있습니다.
문제는 같은 스타일의 조건이 여러 라우트에 반복될 때입니다.
어떤 조건은 `startsWith`, 어떤 조건은 정규식, 어떤 조건은 호스트 비교를 쓰면 정책의 일관성이 흐려집니다.

`URLPattern`은 URL을 구성 요소 단위로 나눠 패턴을 선언하게 해 줍니다.
라우트가 URL의 구조를 기준으로 정해진다면, 패턴 객체를 미리 만들어 두고 요청마다 `test` 또는 `exec`를 호출하는 방식이 더 명확합니다.

```js
const userRoute = new URLPattern({
  protocol: 'https',
  hostname: 'api.example.com',
  pathname: '/v1/users/:userId'
});

console.log(userRoute.test('https://api.example.com/v1/users/42'));
// true
```

여기서 중요한 점은 `URLPattern`이 라우터 프레임워크를 대체하는 거대한 도구가 아니라는 점입니다.
Express, Fastify, Hono 같은 프레임워크를 쓰고 있다면 해당 프레임워크의 라우팅 기능을 먼저 따르는 편이 자연스럽습니다.
`URLPattern`은 작은 프록시, 웹훅 검증, 런타임 공통 유틸, 엣지 스타일 핸들러처럼 라우팅 계층이 얇은 코드에서 특히 유용합니다.

### 정규식보다 의도를 드러내고 싶을 때

URL 매칭을 정규식으로만 작성하면 정확한 조건을 표현할 수는 있지만, 리뷰 비용이 올라갑니다.
예를 들어 아래 정규식은 동작하더라도 어느 부분이 호스트이고 어느 부분이 경로인지 한눈에 들어오지 않습니다.

```js
const pattern = /^https:\/\/api\.example\.com\/v1\/users\/([^/]+)$/;
```

반면 `URLPattern`은 구성 요소가 분리되어 있어 코드 리뷰에서 정책을 확인하기 쉽습니다.

```js
const pattern = new URLPattern({
  protocol: 'https',
  hostname: 'api.example.com',
  pathname: '/v1/users/:userId'
});
```

보안 검토에서도 장점이 있습니다.
호스트 allowlist, 프로토콜 제한, 경로 파라미터 추출이 한 객체에 담기므로 "어떤 URL을 허용하는가"를 더 짧게 설명할 수 있습니다.
다만 `URLPattern` 역시 패턴을 잘못 쓰면 의도보다 넓게 매칭될 수 있으므로, 중요한 경계에서는 테스트 케이스를 반드시 함께 두어야 합니다.

## test와 exec를 나눠 쓰기

### 매칭 여부만 필요하면 test를 쓴다

`test`는 입력 URL이 패턴과 맞는지 boolean으로 알려줍니다.
프록시에서 특정 경로만 통과시키거나, 웹훅 엔드포인트인지 확인하는 단계에는 `test`가 적합합니다.

```js
const webhookPattern = new URLPattern({
  pathname: '/webhooks/:provider/events'
}, 'https://example.com');

export function isWebhookUrl(input) {
  try {
    return webhookPattern.test(input, 'https://example.com');
  } catch {
    return false;
  }
}

console.log(isWebhookUrl('/webhooks/github/events'));
// true
```

상대 URL을 다룰 때는 기준이 되는 `baseURL`을 명확히 두어야 합니다.
요청 객체가 이미 절대 URL을 제공하는 런타임도 있지만, Node.js의 기본 HTTP 서버에서는 직접 호스트와 프로토콜을 조합해야 하는 경우가 많습니다.
프로젝트 안에서 "요청 URL은 항상 절대 URL로 정규화한 뒤 매칭한다" 같은 규칙을 세워 두면 혼란이 줄어듭니다.

```js
export function toRequestUrl(req) {
  const proto = req.headers['x-forwarded-proto'] ?? 'http';
  const host = req.headers.host;

  if (!host) {
    return null;
  }

  return new URL(req.url, `${proto}://${host}`);
}
```

프록시 뒤에서 `x-forwarded-proto`를 신뢰할지는 배포 환경에 따라 다릅니다.
인터넷에서 직접 들어오는 요청이라면 이 헤더를 그대로 믿으면 안 됩니다.
로드밸런서나 게이트웨이가 신뢰 가능한 경계에서 설정한 값인지 먼저 확인해야 합니다.

### 파라미터가 필요하면 exec를 쓴다

`exec`는 매칭 결과와 함께 URL 구성 요소별 그룹을 반환합니다.
라우트 파라미터를 꺼내야 한다면 `test`보다 `exec`가 어울립니다.

```js
const articlePattern = new URLPattern({
  pathname: '/articles/:slug'
}, 'https://blog.example.com');

export function getArticleSlug(input) {
  const match = articlePattern.exec(input, 'https://blog.example.com');

  if (!match) {
    return null;
  }

  return match.pathname.groups.slug;
}

console.log(getArticleSlug('/articles/nodejs-urlpattern-guide'));
// nodejs-urlpattern-guide
```

파라미터 이름은 라우트 계약의 일부입니다.
`slug`, `userId`, `provider`처럼 도메인 의미가 드러나는 이름을 쓰면 이후 핸들러 코드가 단순해집니다.
반대로 `id`만 반복하면 어떤 리소스의 id인지 컨텍스트를 따라가야 합니다.

`exec` 결과를 그대로 외부 응답에 넣는 것은 피하세요.
매칭 입력 전체가 포함될 수 있으므로, 로그나 응답에는 필요한 그룹 값만 골라서 남기는 편이 안전합니다.

## API 라우트 테이블 만들기

### 패턴과 핸들러를 함께 선언한다

작은 서버에서는 라우트 테이블을 배열로 관리할 수 있습니다.
각 항목에 HTTP 메서드, URLPattern, 핸들러를 함께 두면 라우팅 규칙이 한 곳에 모입니다.

```js
const routes = [
  {
    method: 'GET',
    pattern: new URLPattern({ pathname: '/api/users/:userId' }, 'https://app.local'),
    handler: getUser
  },
  {
    method: 'POST',
    pattern: new URLPattern({ pathname: '/api/users' }, 'https://app.local'),
    handler: createUser
  }
];

export function matchRoute(method, url) {
  for (const route of routes) {
    if (route.method !== method) {
      continue;
    }

    const match = route.pattern.exec(url);

    if (match) {
      return { route, params: match.pathname.groups };
    }
  }

  return null;
}
```

이 방식은 프레임워크가 없는 테스트 서버, 로컬 mock API, 빌드 도구의 내부 HTTP 엔드포인트에 잘 맞습니다.
라우트 수가 많아지거나 미들웨어, 스키마 검증, 인증 계층이 복잡해진다면 전용 프레임워크로 옮기는 편이 낫습니다.

### 라우트 순서를 테스트로 고정한다

패턴 기반 라우팅에서는 구체적인 라우트와 넓은 라우트의 순서가 중요합니다.
예를 들어 `/api/users/me`와 `/api/users/:userId`가 함께 있으면 어느 쪽을 먼저 검사하느냐에 따라 결과가 달라질 수 있습니다.

```js
const routes = [
  {
    name: 'current-user',
    method: 'GET',
    pattern: new URLPattern({ pathname: '/api/users/me' }, 'https://app.local')
  },
  {
    name: 'user-detail',
    method: 'GET',
    pattern: new URLPattern({ pathname: '/api/users/:userId' }, 'https://app.local')
  }
];
```

구체적인 경로를 먼저 두고, 변수 경로를 뒤에 두는 규칙을 정하세요.
그리고 이 규칙은 주석보다 테스트가 더 잘 지켜 줍니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

test('specific user route wins before parameter route', () => {
  const result = matchRoute('GET', 'https://app.local/api/users/me');

  assert.equal(result.route.name, 'current-user');
});
```

라우터는 작은 코드처럼 보여도 사용자 요청의 입구입니다.
경로 하나가 잘못 매칭되면 권한 검사, 캐시 키, 감사 로그까지 영향을 받을 수 있습니다.
라우트 테이블을 만들었다면 대표 경로와 거부해야 할 경로를 함께 테스트하는 편이 좋습니다.

## 외부 URL allowlist에 적용하기

### protocol과 hostname을 분리해서 제한한다

외부 URL을 입력받는 기능에서는 "우리 서비스가 호출해도 되는 주소인가"를 확인해야 합니다.
예를 들어 웹훅 대상, 이미지 프록시, 문서 import 기능은 SSRF 같은 위험을 피하기 위해 URL allowlist가 필요할 수 있습니다.

`URLPattern`을 쓰면 프로토콜과 호스트 조건을 분리해 표현할 수 있습니다.

```js
const allowedDocsUrl = new URLPattern({
  protocol: 'https',
  hostname: 'docs.example.com',
  pathname: '/projects/:projectId/*'
});

export function isAllowedDocsUrl(input) {
  try {
    return allowedDocsUrl.test(input);
  } catch {
    return false;
  }
}
```

이런 검증은 문자열 접두사 검사보다 안전합니다.
`https://docs.example.com.evil.test/...` 같은 값을 막으려면 호스트를 URL 구조로 파싱해서 비교해야 합니다.
패턴을 사용하더라도 로컬 네트워크 차단, 리다이렉트 검증, DNS 재해석 같은 SSRF 방어는 별도 계층에서 다뤄야 합니다.

### search 조건은 너무 많은 의미를 넣지 않는다

`URLPattern`은 검색 문자열 조건도 다룰 수 있습니다.
하지만 쿼리 파라미터는 순서, 인코딩, 중복 키 같은 변수가 많습니다.
복잡한 비즈니스 규칙까지 `search` 패턴에 넣기보다는 URL 매칭 후 `URLSearchParams`로 검증하는 편이 읽기 쉽습니다.

```js
const reportPattern = new URLPattern({
  pathname: '/reports/:reportId'
}, 'https://app.local');

export function parseReportRequest(input) {
  const url = new URL(input, 'https://app.local');
  const match = reportPattern.exec(url.href);

  if (!match) {
    return null;
  }

  const format = url.searchParams.get('format') ?? 'html';

  if (!['html', 'csv'].includes(format)) {
    return null;
  }

  return {
    reportId: match.pathname.groups.reportId,
    format
  };
}
```

URL의 위치를 찾는 일과 쿼리 값의 정책을 검증하는 일을 분리하면 테스트가 단순해집니다.
라우트 매칭 실패와 파라미터 검증 실패를 다른 에러 코드로 남길 수도 있습니다.

## 운영 적용 체크리스트

### 런타임 지원 범위를 먼저 확인한다

Node.js 문서 기준 `URLPattern`은 Experimental API입니다.
서비스가 LTS 버전을 엄격히 쓰거나 런타임을 여러 버전으로 운영한다면, 배포 대상에서 실제로 사용할 수 있는지 먼저 확인해야 합니다.

```js
export function assertUrlPatternSupport() {
  if (typeof URLPattern !== 'function') {
    throw new Error('URLPattern is not available in this Node.js runtime');
  }
}
```

라이브러리 코드라면 import 시점에 바로 실패시키기보다, 기능을 쓰는 경로에서 명확한 에러를 내는 편이 사용자에게 친절할 수 있습니다.
애플리케이션 코드라면 부팅 단계에서 지원 여부를 확인해 배포 실수를 빠르게 잡는 방식도 좋습니다.

### 보안 경계에서는 거부 케이스를 더 많이 테스트한다

URL 매칭 코드는 성공 케이스보다 실패 케이스가 중요합니다.
허용해야 할 URL만 통과하는지, 비슷해 보이는 악성 URL은 거부하는지 테스트해야 합니다.

- `http`가 `https` 조건을 통과하지 않는지 확인한다.
- 비슷한 하위 도메인이나 접미사 도메인이 통과하지 않는지 확인한다.
- 상대 URL을 허용할 때 기준 `baseURL`이 의도와 맞는지 확인한다.
- 경로 파라미터에 빈 값이나 슬래시가 들어갈 때 결과를 확인한다.
- 매칭 결과 전체를 로그나 응답으로 노출하지 않는지 확인한다.

특히 외부 호출 allowlist라면 `URLPattern` 하나만으로 보안이 끝났다고 보면 안 됩니다.
URL 파싱, 리다이렉트 제한, IP 대역 차단, 응답 크기 제한, 타임아웃 정책을 함께 설계해야 합니다.

## FAQ

### URLPattern은 Express 라우터를 대체하나요?

대부분의 웹 애플리케이션에서는 대체하지 않는 편이 좋습니다.
Express, Fastify, Hono 같은 프레임워크를 이미 쓰고 있다면 해당 라우터가 미들웨어, 에러 처리, 파라미터 처리와 함께 동작합니다.
`URLPattern`은 얇은 핸들러, 프록시, URL allowlist, 테스트 유틸처럼 작은 경계에 적용하기 좋습니다.

### 정규식보다 항상 안전한가요?

항상 그렇지는 않습니다.
`URLPattern`은 URL 구성 요소를 분리해 의도를 표현하기 쉽게 만들지만, 넓은 패턴을 쓰면 의도보다 많은 URL을 허용할 수 있습니다.
보안 경계에서는 허용 케이스와 거부 케이스를 모두 테스트해야 합니다.

### 지금 운영 서비스에 바로 써도 되나요?

Node.js v26.7.0 문서 기준 `URLPattern`은 Experimental로 표시되어 있습니다.
운영 서비스에서는 런타임 버전, 장애 허용도, 대체 구현 가능성을 먼저 확인하세요.
작은 내부 도구나 Node 버전이 고정된 서비스라면 실험 범위를 제한해 적용해 볼 수 있습니다.

## 마무리

`URLPattern`의 핵심 가치는 URL 매칭 정책을 문자열 분기가 아니라 구조화된 선언으로 바꾸는 데 있습니다.
경로, 호스트, 프로토콜, 파라미터 추출이 섞이기 시작했다면 `URLPattern`을 검토해 볼 만합니다.

다만 현재 문서상 Experimental API라는 점을 잊지 말아야 합니다.
런타임 지원을 확인하고, 라우트 순서와 거부 케이스를 테스트로 고정한 뒤, 프레임워크 라우터가 과한 작은 경계부터 적용하는 방식이 가장 현실적입니다.

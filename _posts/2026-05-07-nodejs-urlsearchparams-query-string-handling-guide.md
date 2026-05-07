---
layout: post
title: "Node.js URLSearchParams 가이드: 쿼리 문자열을 안전하게 조립하고 읽는 법"
date: 2026-05-07 20:00:00 +0900
lang: ko
translation_key: nodejs-urlsearchparams-query-string-handling-guide
permalink: /development/blog/seo/2026/05/07/nodejs-urlsearchparams-query-string-handling-guide.html
alternates:
  ko: /development/blog/seo/2026/05/07/nodejs-urlsearchparams-query-string-handling-guide.html
  x_default: /development/blog/seo/2026/05/07/nodejs-urlsearchparams-query-string-handling-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, url, urlsearchparams, querystring, backend]
description: "Node.js URLSearchParams로 쿼리 문자열을 안전하게 만들고 읽는 방법을 정리했습니다. 검색 조건 직렬화, 중복 파라미터, 인코딩 주의사항, 실무 패턴까지 함께 설명합니다."
---

검색, 필터, 페이지네이션, 외부 API 호출처럼 HTTP 요청을 다루다 보면 결국 쿼리 문자열을 조립해야 합니다.
이때 문자열을 직접 이어 붙이면 `?`, `&`, `=` 처리와 인코딩 규칙이 섞이면서 작은 버그가 자주 생깁니다.

`URLSearchParams`는 이런 흐름을 단순하게 만들어 줍니다.
핵심은 **쿼리 문자열을 문자열 조각이 아니라 key-value 구조로 다루고, 직렬화와 인코딩은 표준 API에 맡길 수 있다**는 점입니다.

## URLSearchParams가 필요한 이유

### H3. 쿼리 문자열 직접 조립은 쉽게 깨진다

아래처럼 문자열을 직접 만들면 처음에는 간단해 보입니다.

```js
const keyword = 'node js';
const page = 2;

const url = `/search?q=${keyword}&page=${page}`;
console.log(url); // /search?q=node js&page=2
```

하지만 실제 서비스에서는 입력값에 공백, 한글, `&`, `=` 같은 문자가 섞일 수 있습니다.
이 값을 직접 붙이면 검색어가 잘리거나, 의도하지 않은 다른 파라미터로 해석될 수 있습니다.

```js
const keyword = 'node&javascript';
const url = `/search?q=${keyword}`;

console.log(url); // /search?q=node&javascript
```

위 문자열은 `q` 값이 `node&javascript`가 아니라 `node`처럼 해석될 수 있습니다.
그래서 사용자 입력을 URL에 넣을 때는 직접 조립보다 안전한 직렬화 API를 쓰는 편이 좋습니다.

사용자 입력 URL 자체를 먼저 검증해야 하는 상황이라면 [Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)도 함께 보면 흐름이 더 명확해집니다.

### H3. 조건이 늘어날수록 객체처럼 다루는 편이 읽기 쉽다

검색 페이지에서는 보통 조건이 하나로 끝나지 않습니다.

- 검색어 `q`
- 페이지 번호 `page`
- 정렬 기준 `sort`
- 태그 `tag`
- 기간 필터 `from`, `to`

이런 값을 문자열 템플릿으로 계속 이어 붙이면 조건 분기와 구분자 처리가 뒤섞입니다.
`URLSearchParams`를 쓰면 쿼리 문자열을 조립하는 의도를 더 직접적으로 표현할 수 있습니다.

## URLSearchParams 기본 사용법

### H3. 객체에서 쿼리 문자열 만들기

가장 단순한 사용법은 객체를 넘겨 쿼리 문자열을 만드는 방식입니다.

```js
const params = new URLSearchParams({
  q: 'node js',
  page: '2',
  sort: 'latest',
});

console.log(params.toString());
// q=node+js&page=2&sort=latest
```

`toString()`을 호출하면 필요한 인코딩이 적용된 문자열이 만들어집니다.
이제 URL에는 아래처럼 붙이면 됩니다.

```js
const path = `/search?${params.toString()}`;
console.log(path);
```

숫자나 불리언 값은 명시적으로 문자열로 바꿔 넣는 습관이 좋습니다.
나중에 값 변환 규칙이 바뀌어도 쿼리 생성 위치에서 의도를 확인하기 쉽기 때문입니다.

### H3. URL 객체와 함께 searchParams를 바로 수정하기

이미 전체 URL을 다루고 있다면 `URL` 객체의 `searchParams`를 사용하는 방식이 더 자연스럽습니다.

```js
const url = new URL('https://api.example.com/posts');

url.searchParams.set('q', 'node js');
url.searchParams.set('page', String(2));
url.searchParams.set('sort', 'latest');

console.log(url.toString());
// https://api.example.com/posts?q=node+js&page=2&sort=latest
```

이 방식은 외부 API 호출 코드를 만들 때 특히 깔끔합니다.
경로, 호스트, 쿼리 조립을 각각 문자열로 관리하지 않아도 되기 때문입니다.

라우팅이나 경로 패턴 검증까지 함께 다룬다면 [Node.js URLPattern 가이드: 라우팅과 URL 검증을 선언적으로 처리하는 법](/development/blog/seo/2026/05/03/nodejs-urlpattern-routing-validation-guide.html)과 연결해서 생각하기 좋습니다.

## 실무에서 자주 쓰는 패턴

### H3. 빈 값은 넣지 않고 필요한 조건만 추가하기

검색 조건은 선택값이 많습니다.
값이 없는데도 `q=&tag=&page=1`처럼 전부 넣으면 로그와 캐시 키가 지저분해질 수 있습니다.

```js
function buildSearchUrl({ q, tag, page = 1 }) {
  const url = new URL('https://api.example.com/posts');

  if (q?.trim()) {
    url.searchParams.set('q', q.trim());
  }

  if (tag) {
    url.searchParams.set('tag', tag);
  }

  if (page > 1) {
    url.searchParams.set('page', String(page));
  }

  return url.toString();
}

console.log(buildSearchUrl({ q: ' node ', tag: 'backend', page: 2 }));
```

이 패턴의 장점은 분명합니다.

- 기본값과 실제 필터를 구분할 수 있다
- 빈 문자열 파라미터가 줄어든다
- 캐시 키와 로그가 더 읽기 쉬워진다

### H3. 같은 이름의 파라미터를 여러 번 넣기

태그, 카테고리, 체크박스 필터처럼 같은 키가 여러 번 나와야 할 때가 있습니다.
이때는 `set()`이 아니라 `append()`를 사용합니다.

```js
const params = new URLSearchParams();

params.append('tag', 'nodejs');
params.append('tag', 'backend');
params.append('tag', 'performance');

console.log(params.toString());
// tag=nodejs&tag=backend&tag=performance

console.log(params.getAll('tag'));
// ['nodejs', 'backend', 'performance']
```

`set()`은 같은 이름의 기존 값을 대체하고, `append()`는 값을 추가합니다.
이 차이를 명확히 알고 있어야 필터 조건이 조용히 사라지는 버그를 줄일 수 있습니다.

### H3. 기존 URL의 쿼리 값을 읽고 수정하기

프록시, 리다이렉트, 검색 조건 보존 같은 기능에서는 기존 URL을 읽은 뒤 일부 값만 바꾸는 일이 많습니다.

```js
const url = new URL('https://example.com/search?q=node&page=1');

const keyword = url.searchParams.get('q');
const page = Number(url.searchParams.get('page') ?? '1');

url.searchParams.set('page', String(page + 1));

console.log(keyword); // node
console.log(url.toString());
// https://example.com/search?q=node&page=2
```

이런 코드는 문자열 치환보다 안전합니다.
파라미터 순서가 바뀌거나 다른 조건이 추가되어도 특정 키만 읽고 수정할 수 있기 때문입니다.

동적으로 검색 패턴을 만들어야 한다면 쿼리 값 조립과 정규식 조립을 섞지 않는 편이 좋습니다.
정규식 입력 처리는 [Node.js RegExp.escape 가이드: 사용자 입력을 정규식에 안전하게 넣는 법](/development/blog/seo/2026/05/05/nodejs-regexp-escape-safe-dynamic-pattern-guide.html)에서 다룬 방식처럼 별도로 분리하는 것이 안전합니다.

## URLSearchParams 사용 시 주의할 점

### H3. 객체 생성자는 배열 값을 자동으로 반복 파라미터로 만들지 않는다

아래 코드는 기대와 다르게 동작할 수 있습니다.

```js
const params = new URLSearchParams({
  tag: ['nodejs', 'backend'],
});

console.log(params.toString());
// tag=nodejs%2Cbackend
```

반복 파라미터가 필요하면 명시적으로 `append()`를 쓰는 편이 안전합니다.

```js
const params = new URLSearchParams();

for (const tag of ['nodejs', 'backend']) {
  params.append('tag', tag);
}
```

배열을 어떤 방식으로 표현할지는 서버와 클라이언트가 같은 규칙을 써야 합니다.
예를 들어 `tag=nodejs&tag=backend`, `tag[]=nodejs&tag[]=backend`, `tag=nodejs,backend` 중 무엇을 쓸지 API 계약에서 먼저 정해야 합니다.

### H3. 이미 인코딩된 값을 다시 넣으면 이중 인코딩될 수 있다

`URLSearchParams`에는 원본 값을 넣는 것이 기본입니다.
이미 `encodeURIComponent()`로 인코딩한 값을 다시 넣으면 `%` 문자까지 인코딩되어 결과가 달라질 수 있습니다.

```js
const encoded = encodeURIComponent('node js');
const params = new URLSearchParams();

params.set('q', encoded);

console.log(params.toString());
// q=node%2520js
```

대부분의 경우에는 직접 인코딩하지 말고 원본 문자열을 넣는 편이 맞습니다.
직렬화는 `URLSearchParams`가 담당하게 두면 됩니다.

### H3. 쿼리 값은 검증된 입력이 아니다

`URLSearchParams`는 문자열을 안전하게 직렬화하고 파싱하는 도구입니다.
하지만 값의 의미까지 검증해 주지는 않습니다.

예를 들어 `page=-100`, `sort=unknown`, `redirect=https://evil.example` 같은 값은 문법적으로는 쿼리 문자열일 수 있습니다.
그래도 서비스 정책상 허용하면 안 되는 값일 수 있습니다.

```js
function readPage(params) {
  const page = Number(params.get('page') ?? '1');

  if (!Number.isInteger(page) || page < 1 || page > 100) {
    return 1;
  }

  return page;
}
```

쿼리 문자열 처리는 인코딩과 파싱 문제를 해결하고, 입력 검증은 별도 단계로 두는 것이 좋습니다.

## URLSearchParams를 쓰면 좋은 상황

### H3. 외부 API 클라이언트 만들기

외부 API를 호출할 때는 쿼리 조건이 늘어나기 쉽습니다.
`URL`과 `URLSearchParams`를 함께 쓰면 endpoint와 조건을 안정적으로 관리할 수 있습니다.

```js
function createPostsApiUrl({ baseUrl, keyword, limit }) {
  const url = new URL('/posts', baseUrl);

  if (keyword) {
    url.searchParams.set('q', keyword);
  }

  url.searchParams.set('limit', String(limit ?? 20));

  return url;
}

const url = createPostsApiUrl({
  baseUrl: 'https://api.example.com',
  keyword: 'nodejs',
  limit: 10,
});

const response = await fetch(url);
```

문자열 조립이 줄어들면 API 변경에도 대응하기 쉬워집니다.
특히 테스트에서 `url.searchParams.get('limit')`처럼 특정 조건만 확인할 수 있어 유지보수성이 좋아집니다.

### H3. 리다이렉트 URL에 추적 파라미터 붙이기

캠페인 파라미터나 내부 추적값을 붙일 때도 유용합니다.
다만 리다이렉트 목적지는 반드시 허용된 도메인인지 검증해야 합니다.

```js
function addTracking(rawUrl) {
  const url = new URL(rawUrl);

  if (url.hostname !== 'example.com') {
    throw new Error('허용되지 않은 리다이렉트 대상입니다.');
  }

  url.searchParams.set('utm_source', 'blog');
  url.searchParams.set('utm_medium', 'post');

  return url.toString();
}
```

여기서 중요한 점은 `URLSearchParams`가 오픈 리다이렉트 문제를 해결해 주지는 않는다는 것입니다.
쿼리를 안전하게 조립하는 것과 목적지를 안전하게 검증하는 것은 별개의 책임입니다.

## 자주 묻는 질문

### H3. querystring 모듈 대신 URLSearchParams를 써도 될까?

새 코드라면 보통 `URLSearchParams`를 먼저 고려해도 좋습니다.
표준 Web API와 맞물려 있고, `URL` 객체와 함께 쓰기 자연스럽기 때문입니다.
다만 기존 코드가 `querystring`의 특정 직렬화 규칙에 의존한다면 한 번에 바꾸기보다 테스트를 먼저 보강하는 편이 안전합니다.

### H3. 파라미터 순서는 믿어도 될까?

`URLSearchParams`는 추가한 순서를 대체로 유지하지만, 서비스 로직이 파라미터 순서에 의존하도록 만들지는 않는 편이 좋습니다.
캐시 키나 서명 문자열처럼 순서가 중요한 경우에는 별도 정렬 규칙을 명시적으로 적용해야 합니다.

```js
params.sort();
const cacheKey = params.toString();
```

정렬 규칙이 필요한지 여부는 API 계약과 캐시 전략에 따라 결정하면 됩니다.

## 마무리

`URLSearchParams`는 화려한 기능은 아니지만, Node.js에서 URL 쿼리 문자열을 다루는 코드를 훨씬 안전하고 읽기 좋게 만들어 줍니다.
직접 문자열을 이어 붙이는 대신 key-value 구조로 값을 추가하고, 인코딩과 직렬화는 표준 API에 맡기는 것이 핵심입니다.

실무에서는 아래 원칙만 지켜도 대부분의 쿼리 문자열 버그를 줄일 수 있습니다.

- 원본 값을 넣고 인코딩은 `URLSearchParams`에 맡긴다
- 같은 키를 여러 번 넣어야 하면 `append()`를 쓴다
- 쿼리 파싱과 입력 검증을 분리한다
- 리다이렉트나 외부 URL은 별도로 허용 목록을 검증한다

## 함께 보면 좋은 글

- [Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)
- [Node.js URLPattern 가이드: 라우팅과 URL 검증을 선언적으로 처리하는 법](/development/blog/seo/2026/05/03/nodejs-urlpattern-routing-validation-guide.html)
- [Node.js RegExp.escape 가이드: 사용자 입력을 정규식에 안전하게 넣는 법](/development/blog/seo/2026/05/05/nodejs-regexp-escape-safe-dynamic-pattern-guide.html)

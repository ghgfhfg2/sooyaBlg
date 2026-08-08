---
layout: post
title: "Node.js URLSearchParams.sort 가이드: 쿼리스트링 캐시 키를 안정적으로 만드는 법"
date: 2026-08-08 20:00:00 +0900
lang: ko
translation_key: nodejs-urlsearchparams-sort-cache-key-guide
permalink: /development/blog/seo/2026/08/08/nodejs-urlsearchparams-sort-cache-key-guide.html
alternates:
  ko: /development/blog/seo/2026/08/08/nodejs-urlsearchparams-sort-cache-key-guide.html
  x_default: /development/blog/seo/2026/08/08/nodejs-urlsearchparams-sort-cache-key-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, urlsearchparams, query-string, cache-key, url, api, backend, javascript]
description: "Node.js URLSearchParams.sort로 쿼리스트링 순서를 정규화하고 캐시 키, API 중복 제거 키, 서명 입력값을 안정적으로 만드는 실무 패턴을 정리합니다."
---

API 서버나 프론트엔드 라우터를 만들다 보면 같은 요청인데도 URL 문자열이 다르게 보이는 경우가 자주 생깁니다.
예를 들어 `?page=1&sort=latest`와 `?sort=latest&page=1`은 사람이 보기에는 같은 조건입니다.
하지만 문자열 그대로 캐시 키나 중복 제거 키로 쓰면 서로 다른 요청처럼 취급됩니다.

이 문제를 줄이는 기본 도구가 Node.js와 브라우저가 함께 제공하는 `URLSearchParams`입니다.
특히 `URLSearchParams.sort()`를 이용하면 쿼리 파라미터의 이름 순서를 안정적으로 정렬할 수 있어, 캐시 키·로그 집계 키·서명 입력값을 더 예측 가능하게 만들 수 있습니다.
이 글에서는 `URLSearchParams.sort()`의 사용법, 다중 값 파라미터 처리, 캐시 키 설계, 보안 점검 기준을 실무 예제로 정리합니다.

쿼리스트링을 처음 다룬다면 [Node.js URLSearchParams 가이드: 쿼리 문자열을 안전하게 다루는 법](/development/blog/seo/2026/05/07/nodejs-urlsearchparams-query-string-handling-guide.html)을 먼저 보면 좋습니다.
URL 입력값 검증이 함께 필요하다면 [Node.js URL.canParse 가이드: 안전한 URL 검증 패턴](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)도 이어서 참고할 수 있습니다.

## 쿼리스트링 정규화가 필요한 이유

### H3. 순서만 다른 요청이 캐시를 낭비한다

HTTP 쿼리 파라미터는 순서가 바뀌어도 애플리케이션 의미가 같을 때가 많습니다.
검색, 목록 필터, 페이지네이션처럼 key-value 조건으로 해석되는 API가 대표적입니다.

```txt
/articles?page=1&sort=latest&tag=nodejs
/articles?tag=nodejs&sort=latest&page=1
```

두 URL이 같은 결과를 반환한다면 캐시 키도 같아야 합니다.
하지만 원문 URL 전체를 그대로 키로 쓰면 캐시가 나뉘고, 같은 데이터를 여러 번 계산하거나 내려받게 됩니다.

```js
const cacheKey = request.url;
```

이 방식은 구현은 쉽지만 운영 비용을 숨깁니다.
파라미터 순서가 호출자마다 달라질 수 있는 공개 API, 프록시 뒤의 서버, 여러 클라이언트가 붙는 서비스라면 정규화된 키를 따로 만드는 편이 안전합니다.

### H3. 문자열 비교 전에 의미를 먼저 정해야 한다

정규화는 모든 URL을 무조건 같은 규칙으로 바꾸는 일이 아닙니다.
먼저 서비스에서 쿼리 파라미터를 어떤 의미로 해석하는지 정해야 합니다.

- `page`, `sort`, `tag`처럼 순서가 의미 없는 필터인지
- `ids=3&ids=1`처럼 값의 순서가 의미 있는지
- 빈 값과 누락을 같게 볼지 다르게 볼지
- 추적용 파라미터를 캐시 키에서 제외할지
- 대소문자를 보존해야 하는지

이 기준 없이 정렬만 적용하면 오히려 버그가 생길 수 있습니다.
예를 들어 `ids=3&ids=1`이 사용자가 지정한 우선순서라면 값 순서를 함부로 바꾸면 안 됩니다.
반대로 단순 필터라면 같은 값 집합을 하나의 키로 모으는 것이 유리합니다.

## URLSearchParams.sort 기본 사용법

### H3. 파라미터 이름 기준으로 정렬한다

`URLSearchParams.sort()`는 현재 객체를 제자리에서 정렬합니다.
정렬 기준은 파라미터 이름입니다.

```js
const params = new URLSearchParams('tag=nodejs&page=1&sort=latest');

params.sort();

console.log(params.toString());
// page=1&sort=latest&tag=nodejs
```

이 결과를 캐시 키 일부로 사용하면 호출자가 파라미터를 어떤 순서로 보냈는지에 덜 흔들립니다.

```js
export function createListCacheKey(pathname, rawQuery) {
  const params = new URLSearchParams(rawQuery);
  params.sort();

  const query = params.toString();
  return query ? `${pathname}?${query}` : pathname;
}
```

사용 예시는 아래와 같습니다.

```js
console.log(createListCacheKey('/articles', 'tag=nodejs&page=1&sort=latest'));
console.log(createListCacheKey('/articles', 'sort=latest&page=1&tag=nodejs'));

// 둘 다 /articles?page=1&sort=latest&tag=nodejs
```

이 함수는 URL 전체를 직접 문자열 파싱하지 않습니다.
쿼리스트링만 `URLSearchParams`에 맡기기 때문에 인코딩 처리도 더 일관적입니다.

### H3. 원본 URL을 보존해야 할 때는 복사본을 정렬한다

`sort()`는 원본 객체를 바꿉니다.
요청 처리 중 원본 순서가 필요한 코드가 있다면 복사본을 만들어 정렬해야 합니다.

```js
export function normalizeQuery(params) {
  const normalized = new URLSearchParams(params);
  normalized.sort();
  return normalized.toString();
}

const original = new URLSearchParams('b=2&a=1');
const normalized = normalizeQuery(original);

console.log(original.toString()); // b=2&a=1
console.log(normalized); // a=1&b=2
```

작은 차이지만 공용 유틸리티에서는 중요합니다.
함수가 인자로 받은 객체를 몰래 바꾸면 호출부에서 재현하기 어려운 버그가 생깁니다.
특히 요청 로깅, 인증 검증, 라우팅이 같은 `URLSearchParams` 객체를 공유하는 코드에서는 복사본을 쓰는 습관이 안전합니다.

## 캐시 키 설계 패턴

### H3. 캐시에서 제외할 파라미터를 명시한다

마케팅 추적 파라미터나 화면 상태 파라미터는 API 결과에 영향을 주지 않을 수 있습니다.
이런 값까지 캐시 키에 넣으면 같은 응답이 여러 키로 갈라집니다.

```js
const IGNORED_CACHE_PARAMS = new Set([
  'utm_source',
  'utm_medium',
  'utm_campaign',
  'fbclid',
  'gclid'
]);

export function createApiCacheKey(inputUrl) {
  const url = new URL(inputUrl, 'https://example.com');
  const params = new URLSearchParams();

  for (const [key, value] of url.searchParams) {
    if (!IGNORED_CACHE_PARAMS.has(key)) {
      params.append(key, value);
    }
  }

  params.sort();

  const query = params.toString();
  return query ? `${url.pathname}?${query}` : url.pathname;
}
```

이 예제는 origin을 캐시 키에 넣지 않습니다.
같은 서버 안에서 pathname과 query만으로 리소스를 식별한다는 전제가 있기 때문입니다.
여러 host를 같은 캐시 저장소에서 다룬다면 host도 키에 포함해야 합니다.

```js
export function createHostAwareCacheKey(inputUrl) {
  const url = new URL(inputUrl);
  const params = new URLSearchParams(url.searchParams);

  params.sort();

  const query = params.toString();
  return query ? `${url.host}${url.pathname}?${query}` : `${url.host}${url.pathname}`;
}
```

캐시 키는 짧고 안정적이어야 하지만, 서로 다른 리소스를 같은 키로 합치면 안 됩니다.
제외 목록은 “응답에 영향을 주지 않는 파라미터”로만 제한하는 것이 좋습니다.

### H3. 다중 값 파라미터는 정책을 분리한다

`URLSearchParams.sort()`는 이름 기준으로 정렬하지만, 같은 이름을 가진 값들의 의미까지 판단하지는 않습니다.
예를 들어 아래 두 쿼리는 이름 기준 정렬 후에도 값 순서가 다릅니다.

```txt
tag=nodejs&tag=backend&page=1
tag=backend&tag=nodejs&page=1
```

서비스에서 `tag`의 순서가 의미 없다면 값도 정렬해야 같은 키가 됩니다.

```js
const ORDERLESS_PARAMS = new Set(['tag', 'category']);

export function createSearchCacheKey(rawQuery) {
  const source = new URLSearchParams(rawQuery);
  const grouped = new Map();

  for (const [key, value] of source) {
    if (!grouped.has(key)) {
      grouped.set(key, []);
    }

    grouped.get(key).push(value);
  }

  const normalized = new URLSearchParams();

  for (const key of [...grouped.keys()].sort()) {
    const values = grouped.get(key);
    const orderedValues = ORDERLESS_PARAMS.has(key) ? [...values].sort() : values;

    for (const value of orderedValues) {
      normalized.append(key, value);
    }
  }

  return normalized.toString();
}
```

이 함수는 key를 정렬하고, `tag`와 `category`처럼 순서가 의미 없는 값만 별도로 정렬합니다.
반대로 `ids`, `steps`, `fields`처럼 순서가 응답 형태에 영향을 줄 수 있는 파라미터는 원래 값 순서를 보존합니다.

## API 서명과 중복 제거에 적용하기

### H3. 서명 입력값은 양쪽이 같은 정규화 규칙을 가져야 한다

웹훅이나 내부 API 서명에서는 쿼리스트링을 서명 입력값에 포함하는 경우가 있습니다.
이때 클라이언트와 서버가 서로 다른 문자열을 만들면 같은 요청도 검증에 실패합니다.

```js
import { createHmac } from 'node:crypto';

export function signRequest({ method, pathname, query, bodyHash, secret }) {
  const params = new URLSearchParams(query);
  params.sort();

  const canonical = [
    method.toUpperCase(),
    pathname,
    params.toString(),
    bodyHash
  ].join('\n');

  return createHmac('sha256', secret).update(canonical).digest('hex');
}
```

여기서 중요한 것은 `sort()` 자체가 아니라 **canonical 문자열 규칙을 문서화하는 것**입니다.
메서드는 대문자로 바꾸는지, pathname은 trailing slash를 보존하는지, 빈 query는 빈 줄로 남기는지, body hash는 어떤 알고리즘을 쓰는지까지 양쪽이 같아야 합니다.

서명 검증을 구현할 때는 [Node.js Web Crypto HMAC 가이드](/development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html)와 [Node.js 웹훅 서명 검증 보안 가이드](/development/blog/seo/2026/03/23/nodejs-webhook-signature-verification-security-guide.html)를 함께 참고하면 좋습니다.
서명 키나 토큰은 로그에 남기지 말고, 실패 응답에서도 상세한 canonical 문자열을 그대로 노출하지 않는 것이 안전합니다.

### H3. 짧은 시간 중복 요청을 묶을 수 있다

동일한 API 요청이 짧은 시간에 여러 번 들어오는 경우, 정규화된 쿼리스트링을 중복 제거 키로 사용할 수 있습니다.
검색 자동완성, 목록 새로고침, 프론트엔드 retry, 서버 간 재시도에서 유용합니다.

```js
const inFlightRequests = new Map();

export async function dedupeByQuery({ pathname, query, load }) {
  const params = new URLSearchParams(query);
  params.sort();

  const key = `${pathname}?${params.toString()}`;

  if (inFlightRequests.has(key)) {
    return inFlightRequests.get(key);
  }

  const promise = load().finally(() => {
    inFlightRequests.delete(key);
  });

  inFlightRequests.set(key, promise);
  return promise;
}
```

이 패턴은 같은 프로세스 안의 동시 요청을 줄이는 데 적합합니다.
프로세스가 여러 개인 서버, 서버리스, 여러 인스턴스 환경에서는 Redis 같은 외부 저장소나 HTTP 캐시 계층이 필요할 수 있습니다.
또한 사용자 권한에 따라 응답이 달라지는 API라면 사용자 식별자나 권한 범위를 키에 포함해야 합니다.

```js
const key = `${user.id}:${pathname}?${params.toString()}`;
```

단, 사용자 이메일이나 액세스 토큰 같은 민감정보를 그대로 키에 넣는 것은 피해야 합니다.
필요하다면 내부 사용자 ID처럼 노출 위험이 낮은 식별자를 쓰거나, 저장소 특성에 맞게 해시를 검토합니다.

## 테스트와 운영 점검

### H3. 순서가 달라도 같은 키가 나오는지 테스트한다

정규화 유틸리티는 작은 함수처럼 보이지만 캐시 적중률과 보안 검증에 영향을 줍니다.
그래서 순서 변경, 빈 값, 다중 값, 제외 파라미터를 테스트로 고정해두는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { createApiCacheKey } from './cache-key.js';

test('normalizes query parameter order', () => {
  assert.equal(
    createApiCacheKey('/articles?tag=nodejs&page=1&sort=latest'),
    '/articles?page=1&sort=latest&tag=nodejs'
  );

  assert.equal(
    createApiCacheKey('/articles?sort=latest&tag=nodejs&page=1'),
    '/articles?page=1&sort=latest&tag=nodejs'
  );
});

test('ignores tracking parameters', () => {
  assert.equal(
    createApiCacheKey('/articles?page=1&utm_source=newsletter'),
    '/articles?page=1'
  );
});
```

테스트 러너의 기본 패턴은 [Node.js test runner 가이드: 테스트 구조와 실행 전략](/development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html)을 참고할 수 있습니다.
캐시 키처럼 작은 유틸리티는 단위 테스트로도 회귀를 충분히 잡을 수 있습니다.

### H3. 로그에는 정규화 결과와 원문을 구분해 남긴다

운영 중 문제를 찾으려면 원문 URL과 정규화된 키를 모두 보고 싶을 때가 있습니다.
하지만 원문 query에는 검색어, 이메일, 토큰, 임시 코드가 들어올 수 있습니다.
따라서 로그에는 필요한 정보만 제한적으로 남기는 것이 좋습니다.

```js
logger.info({
  pathname: url.pathname,
  cacheKey,
  queryParamCount: [...url.searchParams.keys()].length
}, 'api cache key created');
```

원문 전체를 남기는 대신 pathname, 정규화된 캐시 키, 파라미터 개수처럼 운영 판단에 필요한 값만 기록합니다.
캐시 키 자체에도 민감정보가 들어갈 수 있다면 마스킹하거나 해시한 값을 기록합니다.
로그 위생은 캐시 성능만큼 중요합니다.

## 정리

`URLSearchParams.sort()`는 작은 API지만 쿼리스트링을 다루는 코드의 예측 가능성을 크게 높여줍니다.
파라미터 순서가 다른 요청을 같은 캐시 키로 모으고, API 서명 입력값을 안정화하며, 짧은 시간 중복 요청을 줄이는 데 사용할 수 있습니다.

다만 정렬은 도구일 뿐입니다.
어떤 파라미터가 응답에 영향을 주는지, 다중 값의 순서를 보존해야 하는지, 추적용 파라미터를 제외해도 되는지, 사용자 권한을 키에 포함해야 하는지를 먼저 정해야 합니다.
그 기준을 테스트로 고정하면 캐시 적중률과 보안 검증을 함께 안정화할 수 있습니다.

## FAQ

### H3. URLSearchParams.sort는 값까지 정렬하나요?

아니요.
기본적으로 파라미터 이름 기준으로 정렬합니다.
같은 이름을 가진 다중 값의 순서를 별도로 바꾸고 싶다면 서비스 정책에 맞춰 값을 그룹화한 뒤 직접 정렬해야 합니다.

### H3. 캐시 키에서 utm 파라미터를 항상 제거해도 되나요?

항상은 아닙니다.
응답 결과에 영향을 주지 않는다는 것이 명확할 때만 제거해야 합니다.
분석용 랜딩 페이지처럼 `utm_*` 값에 따라 화면이나 실험군이 달라지는 구조라면 캐시 키에 포함해야 할 수 있습니다.

### H3. 정규화된 쿼리스트링을 서명 검증에 써도 안전한가요?

클라이언트와 서버가 같은 canonical 규칙을 공유하고, 비밀 키를 로그나 응답에 노출하지 않으며, 안전한 HMAC 검증을 사용한다면 실무적으로 유용합니다.
단, 기존 외부 API 규격이 이미 정한 서명 규칙이 있다면 그 규칙을 우선해야 합니다.

---
layout: post
title: "Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법"
date: 2026-05-04 20:00:00 +0900
lang: ko
translation_key: nodejs-url-canparse-safe-url-validation-guide
permalink: /development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html
alternates:
  ko: /development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html
  x_default: /development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, url, validation, backend, web]
description: "Node.js의 URL.canParse로 사용자 입력 URL을 예외 없이 검증하는 방법을 정리했습니다. new URL()과의 차이, 실무 검증 패턴, 상대 경로 처리 팁까지 예제와 함께 설명합니다."
---

폼 입력, 환경변수, 외부 API 응답처럼 **신뢰할 수 없는 문자열을 URL로 해석해야 하는 상황**은 생각보다 자주 나옵니다.
이때 `new URL(value)`만 바로 호출하면 잘못된 입력에서 예외가 터지고, 검증 로직과 에러 처리 로직이 뒤섞이기 쉽습니다.

Node.js의 `URL.canParse()`는 이런 문제를 훨씬 단정하게 정리해 줍니다.
핵심은 **"먼저 파싱 가능한지 확인하고, 그다음 안전하게 URL 객체를 만든다"**는 흐름으로 바꿀 수 있다는 점입니다.

## URL.canParse가 필요한 이유

### H3. new URL()은 간단하지만 검증 단계에서는 예외 비용이 생긴다

`new URL()`은 URL 객체를 만들 때 매우 유용하지만, 잘못된 값이 들어오면 즉시 예외를 던집니다.
문제는 사용자 입력 검증처럼 실패가 흔한 경로에서는 이 예외가 "이상 상황"이 아니라 **정상적으로 자주 발생하는 분기**라는 점입니다.

```js
function parseWebsite(input) {
  return new URL(input);
}
```

위 코드는 정상 입력에서는 간단하지만, 아래 같은 경우에 취약합니다.

- 입력이 빈 문자열인 경우
- 프로토콜이 빠진 경우
- 상대 경로인데 base URL이 없는 경우
- 공백이나 잘못된 문자가 섞인 경우

검증이 주목적인 코드라면 `try/catch`를 광범위하게 두르는 것보다, 애초에 **파싱 가능 여부를 불리언으로 분리**하는 편이 더 읽기 쉽습니다.

### H3. 실패를 예외가 아니라 조건문으로 다룰 수 있다

`URL.canParse(input, base?)`는 파싱 가능하면 `true`, 아니면 `false`를 반환합니다.
즉, 아래처럼 흐름을 단순화할 수 있습니다.

```js
function isValidAbsoluteUrl(input) {
  return URL.canParse(input);
}
```

이 패턴은 입력을 먼저 검사하고 나중에 실제 동작을 수행하는 [Node.js util.parseArgs 가이드: CLI 인자 파싱을 표준 기능으로 정리하는 법](/development/blog/seo/2026/04/28/nodejs-util-parseargs-cli-argument-parsing-guide.html)과도 잘 맞습니다.

## 기본 사용법

### H3. 절대 URL 검증은 가장 단순하게 시작할 수 있다

```js
const values = [
  'https://sooyadev.com',
  'http://localhost:3000',
  'not a url',
  ''
];

for (const value of values) {
  console.log(value, URL.canParse(value));
}
```

이 접근의 장점은 분명합니다.

- 예외를 잡지 않아도 된다
- 대량 입력 검증 루프에서 코드가 단순해진다
- 검증 단계와 객체 생성 단계를 분리할 수 있다

실무에서는 `true`가 나온 뒤에만 `new URL()`을 호출하는 2단계 패턴이 가장 무난합니다.

```js
function parseIfValid(input) {
  if (!URL.canParse(input)) {
    return null;
  }

  return new URL(input);
}
```

### H3. 상대 경로는 base URL과 함께 확인해야 한다

상대 경로는 기준 URL 없이는 해석할 수 없습니다.
이 점을 놓치면 분명히 정상인 값도 잘못된 값처럼 처리하게 됩니다.

```js
const base = 'https://example.com';

console.log(URL.canParse('/docs', base)); // true
console.log(URL.canParse('../about', base)); // true
console.log(URL.canParse('/docs')); // false
```

특히 링크 조합, 리다이렉트 처리, 크롤링, 라우팅 유틸리티에서는 이 base 인자가 중요합니다.
관련해서 패턴 기반 매칭이 필요하다면 [Node.js URLPattern 가이드: 라우팅과 URL 검증을 더 읽기 쉽게 다루는 법](/development/blog/seo/2026/05/03/nodejs-urlpattern-routing-validation-guide.html)도 함께 보면 좋습니다.

## 실무에서 유용한 검증 패턴

### H3. 프로토콜 허용 목록을 함께 검사한다

파싱 가능하다고 해서 곧바로 안전하거나 허용 가능한 URL은 아닙니다.
예를 들어 `javascript:` 같은 스킴은 목적에 따라 차단해야 할 수 있습니다.

```js
const ALLOWED_PROTOCOLS = new Set(['http:', 'https:']);

function parsePublicUrl(input) {
  if (!URL.canParse(input)) {
    return null;
  }

  const url = new URL(input);

  if (!ALLOWED_PROTOCOLS.has(url.protocol)) {
    return null;
  }

  return url;
}
```

즉, `URL.canParse()`는 **1차 문법 검증**, 프로토콜/호스트/포트 점검은 **2차 정책 검증**으로 나누는 편이 좋습니다.

### H3. 환경변수 검증에도 잘 맞는다

서비스 시작 시 환경변수 URL이 잘못 들어오면 나중에 애매한 장애로 번질 수 있습니다.
초기 부팅 단계에서 명확하게 걸러 두는 편이 안전합니다.

```js
function getRequiredServiceUrl(name) {
  const value = process.env[name];

  if (!value || !URL.canParse(value)) {
    throw new Error(`${name} must be a valid URL`);
  }

  return new URL(value);
}
```

이런 식으로 시작 시점 검증을 고정해 두면, 실제 요청 처리 경로에서는 URL 형태를 다시 의심하지 않아도 됩니다.
환경 설정을 초기에 정리하는 관점은 [Node.js loadEnvFile 가이드: built-in 환경변수 파일 관리를 단순하게 하는 법](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)과도 연결됩니다.

## URL.canParse를 쓸 때 주의할 점

### H3. 검증 성공은 비즈니스 규칙 통과를 의미하지 않는다

`URL.canParse()`는 어디까지나 파싱 가능 여부를 알려 줄 뿐입니다.
아래 조건은 별도로 검사해야 할 수 있습니다.

- 사내 도메인만 허용할지
- 특정 포트만 허용할지
- 쿼리 파라미터 길이를 제한할지
- 업로드/리다이렉트 경로에서 로컬 주소를 차단할지

즉, 문법적으로 맞는 URL과 **서비스 정책상 허용 가능한 URL**은 다른 문제입니다.

### H3. base URL을 잘못 넘기면 검증 기준도 함께 흔들린다

상대 경로 검증은 base 값이 바뀌면 결과가 달라집니다.
그래서 아래처럼 기준 URL을 코드 전역에서 제각각 만들면 유지보수가 어려워집니다.

```js
function resolvePath(pathname) {
  const base = process.env.PUBLIC_BASE_URL;

  if (!base || !URL.canParse(pathname, base)) {
    return null;
  }

  return new URL(pathname, base);
}
```

이럴 때는 base URL 생성 규칙을 한곳으로 모으는 편이 낫습니다.
경로와 URL 문맥을 명확히 나누는 습관은 [Node.js import.meta.dirname 가이드: ESM에서 파일 경로를 안정적으로 다루는 법](/development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html)처럼 "기준이 되는 위치"를 분명히 다루는 글과도 통하는 부분이 있습니다.

## 추천 적용 방식

### H3. 검증 함수와 파싱 함수를 분리하면 재사용성이 좋아진다

처음부터 모든 곳에서 `new URL()`을 직접 호출하기보다, 아래처럼 작은 유틸리티를 두는 편이 실용적입니다.

```js
export function canParseHttpUrl(input) {
  if (!URL.canParse(input)) {
    return false;
  }

  const url = new URL(input);
  return url.protocol === 'http:' || url.protocol === 'https:';
}

export function parseHttpUrl(input) {
  if (!canParseHttpUrl(input)) {
    return null;
  }

  return new URL(input);
}
```

이 구조의 장점은 세 가지입니다.

1. 컨트롤러, 배치, CLI에서 같은 규칙을 재사용할 수 있다
2. 테스트에서 허용/비허용 케이스를 분리하기 쉽다
3. 나중에 정책이 바뀌어도 수정 범위가 작다

### H3. 예외는 정말 예외적인 상황에만 남겨 두는 편이 낫다

사용자 입력이 틀릴 수 있다는 사실은 정상입니다.
따라서 입력 검증 코드에서 너무 많은 `try/catch`를 쓰기보다, 예외는 아래 같은 곳에 남기는 편이 좋습니다.

- 반드시 존재해야 하는 설정값이 깨졌을 때
- 내부 불변식이 무너졌을 때
- 검증 후에도 도달하면 안 되는 상태가 생겼을 때

에러를 어디서 던지고 어디서 흡수할지 구분하는 습관은 [Node.js error cause 가이드: 래핑된 에러에서도 원인을 잃지 않는 법](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)과 함께 보면 더 도움이 됩니다.

## 실전 체크리스트

### H3. 도입 전에 이 다섯 가지를 먼저 확인한다

`URL.canParse()`를 넣기 전에 아래를 점검하면 시행착오를 줄일 수 있습니다.

1. 이 검증이 절대 URL용인지, 상대 경로용인지 구분했는가?
2. 상대 경로라면 base URL의 출처가 일관적인가?
3. 파싱 가능 여부와 허용 정책을 분리했는가?
4. `http:`/`https:` 외 스킴 허용 여부를 명시했는가?
5. 검증 후에만 `new URL()`을 호출하도록 흐름을 정리했는가?

이 다섯 가지만 지켜도 URL 처리 로직이 훨씬 덜 깨지고, 장애 원인도 빨리 좁힐 수 있습니다.

## 마무리

`URL.canParse()`는 거창한 기능은 아니지만, **자주 실패하는 입력 검증 경로를 훨씬 덜 거칠게 만들어 주는 도구**입니다.
특히 폼 입력, 환경변수, 리다이렉트 URL, 외부 API 응답처럼 신뢰할 수 없는 문자열을 자주 다뤄야 한다면 체감 효과가 큽니다.

핵심은 단순합니다.
**예외를 검증 수단으로 남용하지 말고, 파싱 가능 여부를 먼저 확인한 뒤 객체를 생성하는 흐름으로 바꾸는 것**입니다.

## 함께 보면 좋은 글

- [Node.js URLPattern 가이드: 라우팅과 URL 검증을 더 읽기 쉽게 다루는 법](/development/blog/seo/2026/05/03/nodejs-urlpattern-routing-validation-guide.html)
- [Node.js util.parseArgs 가이드: CLI 인자 파싱을 표준 기능으로 정리하는 법](/development/blog/seo/2026/04/28/nodejs-util-parseargs-cli-argument-parsing-guide.html)
- [Node.js loadEnvFile 가이드: built-in 환경변수 파일 관리를 단순하게 하는 법](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)

---
layout: post
title: "Node.js RegExp.escape 가이드: 사용자 입력을 정규식에 안전하게 넣는 법"
date: 2026-05-05 08:00:00 +0900
lang: ko
translation_key: nodejs-regexp-escape-safe-dynamic-pattern-guide
permalink: /development/blog/seo/2026/05/05/nodejs-regexp-escape-safe-dynamic-pattern-guide.html
alternates:
  ko: /development/blog/seo/2026/05/05/nodejs-regexp-escape-safe-dynamic-pattern-guide.html
  x_default: /development/blog/seo/2026/05/05/nodejs-regexp-escape-safe-dynamic-pattern-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, regexp, regex, validation, backend, security]
description: "Node.js의 RegExp.escape로 사용자 입력을 정규식 패턴에 안전하게 포함하는 방법을 정리했습니다. 수동 이스케이프의 한계, 검색·필터링 실무 예제, 주의사항까지 함께 설명합니다."
---

검색어 강조, 파일명 필터링, 로그 매칭, 부분 검색처럼 **문자열을 바탕으로 정규식을 동적으로 만드는 일**은 생각보다 자주 생깁니다.
그런데 이때 사용자 입력을 그대로 `new RegExp()`에 넣으면, 단순 검색을 만들려다가 의도하지 않은 패턴 확장이나 매칭 오류를 만들기 쉽습니다.

이럴 때 `RegExp.escape()`는 꽤 실용적입니다.
핵심은 **사용자 입력을 정규식 문법이 아니라 "그대로 찾을 문자열"로 바꿔서 안전하게 조합할 수 있다**는 점입니다.

## RegExp.escape가 필요한 이유

### H3. 사용자 입력은 검색어이지 정규식이 아닐 때가 많다

예를 들어 사용자가 아래 같은 값을 입력했다고 가정해 보겠습니다.

- `node.js`
- `a+b`
- `(test)`
- `foo|bar`

이 값을 그대로 정규식에 넣으면 마침표, 더하기, 괄호, 파이프가 전부 **정규식 메타문자**로 해석됩니다.
즉, 사용자는 단순 문자열 검색을 원했는데 코드가 멋대로 패턴 검색으로 바뀌는 셈입니다.

```js
const keyword = 'node.js';
const regex = new RegExp(keyword, 'i');

console.log(regex.test('nodeXjs')); // true
```

위 결과는 종종 개발자가 기대한 것과 다릅니다.
`.`이 "아무 문자 하나"로 해석되기 때문입니다.

### H3. 수동 이스케이프는 자주 빠뜨리고 유지보수도 어렵다

기존에는 아래처럼 직접 치환하는 코드가 흔했습니다.

```js
function escapeRegex(value) {
  return value.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```

이 방식도 쓸 수는 있지만, 프로젝트마다 중복 구현되기 쉽고, 코드만 보고 의도를 바로 이해하기도 어렵습니다.
이럴 때 표준 API인 `RegExp.escape()`를 쓰면 **"정규식에 안전하게 넣기 위한 이스케이프"**라는 의도가 더 분명해집니다.

## RegExp.escape 기본 사용법

### H3. 동적 검색 패턴을 만들 때 가장 먼저 감싼다

```js
const keyword = 'node.js';
const safePattern = RegExp.escape(keyword);
const regex = new RegExp(safePattern, 'i');

console.log(regex.test('node.js guide')); // true
console.log(regex.test('nodeXjs guide')); // false
```

이 패턴의 장점은 단순합니다.

- 입력이 메타문자를 포함해도 검색 의미가 바뀌지 않는다
- 검색 기능과 정규식 문법 기능을 분리할 수 있다
- 사용자 입력 기반 매칭 로직이 덜 깨진다

입력을 먼저 안전한 형태로 정리한 뒤 실제 동작으로 넘기는 구조는 [Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)에서 다룬 검증 흐름과도 닮아 있습니다.

### H3. 접두사·접미사 조건과 함께 조합하기도 쉽다

실무에서는 완전 일치보다 일부 조건과 함께 쓰는 경우가 많습니다.

```js
function createStartsWithRegex(input) {
  return new RegExp(`^${RegExp.escape(input)}`, 'i');
}

const regex = createStartsWithRegex('api/v1.');

console.log(regex.test('api/v1.users')); // true
console.log(regex.test('apiXv1.users')); // false
```

즉, 정규식의 구조는 개발자가 만들고, **가변 입력 부분만 안전하게 escape**하는 식으로 역할을 나누는 게 핵심입니다.

## 어떤 상황에서 특히 유용한가

### H3. 검색어 하이라이트나 목록 필터링

```js
function containsKeyword(text, keyword) {
  const regex = new RegExp(RegExp.escape(keyword), 'i');
  return regex.test(text);
}
```

블로그 검색, 관리자 페이지 목록 필터, 로그 검색처럼 "입력값이 문자열 그대로 포함되는지" 보고 싶은 상황에서 특히 잘 맞습니다.
정규식 문법을 허용하지 않겠다는 정책이 분명할수록 더 유용합니다.

### H3. 파일명·경로·도메인 같은 특수문자 많은 문자열 비교

파일명, URL 경로, 도메인, npm 패키지 이름은 점(`.`), 하이픈(`-`), 슬래시(`/`)처럼 메타문자와 헷갈리기 쉬운 문자가 자주 들어갑니다.

```js
function hasExactPackageName(text, packageName) {
  const regex = new RegExp(`\\b${RegExp.escape(packageName)}\\b`, 'i');
  return regex.test(text);
}
```

이런 경우 수동 이스케이프를 빼먹으면 검색 결과가 미묘하게 틀어질 수 있습니다.
URL 입력처럼 특수문자가 많은 값을 안전하게 다루는 감각은 [Node.js URLPattern 가이드: 라우팅과 URL 검증을 더 읽기 쉽게 다루는 법](/development/blog/seo/2026/05/03/nodejs-urlpattern-routing-validation-guide.html)과도 연결됩니다.

## RegExp.escape를 쓸 때 주의할 점

### H3. 정규식 기능 자체를 허용하려는 경우에는 쓰면 안 된다

모든 입력을 escape하면 와일드카드나 그룹 같은 정규식 기능도 함께 무력화됩니다.
즉, 아래 두 요구사항은 구분해야 합니다.

1. 사용자가 입력한 문자열을 그대로 찾고 싶은가?
2. 사용자가 직접 정규식 문법을 쓸 수 있게 할 것인가?

첫 번째라면 `RegExp.escape()`가 맞고, 두 번째라면 escape 없이 별도의 검증·권한 정책이 필요합니다.
둘을 섞으면 검색 UX도 보안 정책도 애매해집니다.

### H3. escape는 안전한 검색 보조 도구이지 전체 보안 정책의 대체재는 아니다

`RegExp.escape()`는 정규식 메타문자 문제를 줄여 주지만, 아래까지 자동으로 해결해 주지는 않습니다.

- 너무 긴 입력으로 인한 성능 문제
- 대량 반복 검색의 비용
- 플래그 설정 실수(`g`, `m`, `u` 등)
- 정규식 외의 SQL, 쉘, HTML 인젝션 문제

즉, 이 함수는 **정규식 조합 단계의 안정성**을 높여 주는 도구입니다.
입력 길이 제한이나 전체 처리량 제어는 별도로 챙겨야 합니다.
운영 보호 장치를 같이 보는 관점은 [Node.js scheduler.yield 가이드: 긴 작업 중 이벤트 루프 응답성을 지키는 법](/development/blog/seo/2026/05/02/nodejs-scheduler-yield-event-loop-responsiveness-guide.html)과도 통합니다.

## 추천 적용 패턴

### H3. 정규식 생성 책임을 작은 유틸 함수로 모은다

아래처럼 정규식 생성 함수를 한곳으로 모아 두면 코드베이스가 훨씬 단정해집니다.

```js
export function createLiteralSearchRegex(input, flags = 'i') {
  return new RegExp(RegExp.escape(input), flags);
}

export function createExactMatchRegex(input, flags = 'i') {
  return new RegExp(`^${RegExp.escape(input)}$`, flags);
}
```

이 구조의 장점은 분명합니다.

- escape 누락을 줄일 수 있다
- 테스트 대상을 좁히기 쉽다
- 팀 내 규칙을 일관되게 유지할 수 있다

### H3. "고정 패턴 + 가변 입력" 구조를 먼저 정한다

예를 들어 아래처럼 고정 구조와 사용자 입력을 분리하면 의도가 잘 드러납니다.

```js
function createTagSearchRegex(tag) {
  return new RegExp(`(^|\\s)#${RegExp.escape(tag)}(\\s|$)`, 'i');
}
```

여기서 정규식 구조는 개발자가 통제하고, `tag`만 escape합니다.
이 방식은 문자열을 먼저 안전하게 정리한 뒤 조합하는 [Node.js process.getBuiltinModule 가이드: 런타임 선택 의존성을 더 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-process-getbuiltinmodule-runtime-optional-dependency-guide.html)처럼, **변동성이 큰 부분만 따로 다루는 설계**와 잘 맞습니다.

## 실전 체크리스트

### H3. 도입 전에 이 다섯 가지를 먼저 확인한다

`RegExp.escape()`를 넣기 전에 아래를 점검하면 시행착오를 줄일 수 있습니다.

1. 이 입력은 문자열 검색용인가, 정규식 문법 허용용인가?
2. 정규식 구조와 사용자 입력을 분리했는가?
3. escape 책임을 공용 유틸로 모을 수 있는가?
4. 너무 긴 입력에 대한 길이 제한이 있는가?
5. `i`, `m`, `u` 같은 플래그 선택 이유가 분명한가?

이 다섯 가지만 정리해도 검색·필터링 코드의 오동작이 꽤 줄어듭니다.

## 마무리

`RegExp.escape()`는 화려한 기능은 아니지만, **동적 정규식 생성에서 가장 흔한 실수를 줄여 주는 안전장치**입니다.
특히 검색어, 파일명, 경로, 태그처럼 사용자가 넣는 문자열을 그대로 찾아야 하는 상황이라면 체감 효과가 큽니다.

핵심은 단순합니다.
**정규식 문법은 개발자가 만들고, 사용자 입력은 `RegExp.escape()`로 문자열 그대로 안전하게 넣는 것**입니다.
이 원칙만 지켜도 동적 검색 로직이 훨씬 덜 놀랍게 동작합니다.

## 함께 보면 좋은 글

- [Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)
- [Node.js URLPattern 가이드: 라우팅과 URL 검증을 더 읽기 쉽게 다루는 법](/development/blog/seo/2026/05/03/nodejs-urlpattern-routing-validation-guide.html)
- [Node.js process.getBuiltinModule 가이드: 런타임 선택 의존성을 더 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-process-getbuiltinmodule-runtime-optional-dependency-guide.html)

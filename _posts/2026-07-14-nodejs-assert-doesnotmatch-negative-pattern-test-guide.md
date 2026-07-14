---
layout: post
title: "Node.js assert.doesNotMatch 가이드: 없어야 할 문자열 패턴을 테스트하는 법"
date: 2026-07-14 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-doesnotmatch-negative-pattern-test-guide
permalink: /development/blog/seo/2026/07/14/nodejs-assert-doesnotmatch-negative-pattern-test-guide.html
alternates:
  ko: /development/blog/seo/2026/07/14/nodejs-assert-doesnotmatch-negative-pattern-test-guide.html
  x_default: /development/blog/seo/2026/07/14/nodejs-assert-doesnotmatch-negative-pattern-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, doesnotmatch, regex, test-runner, testing, javascript, validation]
description: "Node.js assert.doesNotMatch()로 에러 메시지, 로그, 응답 문자열에 포함되면 안 되는 패턴을 검증하는 방법을 정리합니다. 정규식 범위, 민감정보 노출 방지, assert.match와의 역할 분리를 예제로 설명합니다."
---

테스트는 기대한 값이 있는지 확인하는 일만 하지 않습니다.
때로는 절대 드러나면 안 되는 문자열이 없는지 확인해야 합니다.
예를 들어 에러 메시지에 내부 경로가 섞이면 안 되고, API 응답에 토큰이 노출되면 안 되며, 사용자에게 보여줄 문구에 디버그 플래그가 남아 있으면 안 됩니다.

Node.js의 `node:assert/strict`가 제공하는 `assert.doesNotMatch()`는 문자열이 특정 정규식과 매칭되지 않아야 한다는 조건을 표현합니다.
반대로 포함되어야 할 패턴은 `assert.match()`가 담당합니다.
두 assertion을 나누어 쓰면 테스트 이름만 읽어도 "반드시 있어야 하는 것"과 "있으면 안 되는 것"이 분명해집니다.

이 글에서는 `assert.doesNotMatch()`를 언제 쓰면 좋은지, 정규식을 과하게 넓게 잡지 않는 방법, 민감정보 노출 방지 테스트를 작성하는 기준을 정리합니다.
포함되어야 할 문자열 패턴은 [Node.js assert.match 문자열 패턴 테스트 가이드](/development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html), 기본 truthy 검증은 [Node.js assert.ok 가이드](/development/blog/seo/2026/06/29/nodejs-assert-ok-truthy-condition-guide.html), 테스트 실행 구조는 [Node.js test runner built-in testing 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)와 함께 보면 좋습니다.

## assert.doesNotMatch가 필요한 상황

### H3. 민감정보가 출력되지 않는지 확인한다

운영 코드에서 가장 위험한 문자열 회귀는 성공 응답보다 실패 응답에서 자주 생깁니다.
예외를 그대로 직렬화하거나, 디버깅을 위해 붙인 값을 지우지 않으면 로그와 응답에 내부 정보가 흘러나올 수 있습니다.

아래 예시는 사용자에게 보여줄 에러 메시지에서 토큰 모양의 문자열이 빠졌는지 확인합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function toPublicError(error) {
  return `request failed: ${error.code}`;
}

test('does not expose bearer token in public error message', () => {
  const message = toPublicError({
    code: 'UPSTREAM_TIMEOUT',
    token: 'Bearer example-token-value'
  });

  assert.equal(message, 'request failed: UPSTREAM_TIMEOUT');
  assert.doesNotMatch(message, /Bearer\s+[A-Za-z0-9._-]+/);
});
```

여기서 핵심은 테스트 fixture에도 실제 토큰을 넣지 않는 것입니다.
테스트용 문자열은 실제 인증정보처럼 보이지 않는 값으로 만들고, 정규식은 "실제로 막고 싶은 노출 형태"를 표현하는 데 집중합니다.

### H3. 내부 구현 세부 사항이 사용자 문구에 섞이지 않게 한다

에러 메시지나 UI 문구에는 사용자에게 필요한 정보만 남기는 편이 좋습니다.
스택 트레이스, 절대 경로, SQL 조각, 내부 서비스 이름 같은 값은 디버깅에는 유용하지만 공개 메시지에는 부적절할 수 있습니다.

```js
import assert from 'node:assert/strict';

function formatUploadError() {
  return '파일 업로드에 실패했습니다. 잠시 후 다시 시도해 주세요.';
}

const message = formatUploadError();

assert.doesNotMatch(message, /\/Users\/|\/home\/|C:\\\\/);
assert.doesNotMatch(message, /SELECT\s+.+\s+FROM/i);
assert.doesNotMatch(message, /stack trace/i);
```

이런 테스트는 보안 테스트라기보다 회귀 방지 테스트에 가깝습니다.
코드 리뷰에서 놓치기 쉬운 문구 변경을 자동으로 잡아 주고, 공개 영역과 내부 영역의 경계를 문서화합니다.

## 정규식을 안전하게 좁히는 방법

### H3. 너무 넓은 패턴은 정상 문구까지 막는다

`assert.doesNotMatch()`는 부정 assertion이기 때문에 정규식이 너무 넓으면 유지보수 비용이 커집니다.
예를 들어 `/error/i`처럼 흔한 단어를 금지하면 정상적인 장애 안내 문구도 테스트를 깨뜨릴 수 있습니다.
정말 금지하려는 표현이 "raw stack trace"라면 그 형태에 가까운 패턴을 사용해야 합니다.

```js
const publicMessage = '요청을 처리하지 못했습니다. 다시 시도해 주세요.';

assert.doesNotMatch(publicMessage, /\bat\s+\w+\s+\(.+:\d+:\d+\)/);
assert.doesNotMatch(publicMessage, /Error:\s+\w+/);
```

위 테스트는 단순히 "오류"라는 단어를 금지하지 않습니다.
대신 JavaScript 스택 프레임이나 원본 예외명이 그대로 노출되는 상황을 겨냥합니다.
검색 의도와 테스트 의도를 모두 생각하면, 부정 검증일수록 패턴을 좁게 잡는 편이 오래 갑니다.

### H3. 실패 메시지로 의도를 남긴다

부정 assertion은 실패했을 때 "무엇이 없어야 했는지"를 바로 알기 어렵습니다.
그래서 세 번째 인자로 짧은 실패 메시지를 넣어 두면 CI 로그에서 원인을 빠르게 파악할 수 있습니다.

```js
assert.doesNotMatch(
  publicMessage,
  /\b(?:password|secret|token)\b/i,
  'public message must not contain sensitive field names'
);
```

실패 메시지에는 실제 민감정보 값을 넣지 않습니다.
검증 대상 문자열이 CI 로그에 출력될 수 있으므로, 테스트 데이터 자체도 공개 가능한 값으로 구성해야 합니다.
민감정보를 막기 위한 테스트가 오히려 민감정보를 남기지 않도록 주의해야 합니다.

## assert.match와 역할을 분리하기

### H3. 있어야 할 것과 없어야 할 것을 한 테스트에서 함께 본다

사용자에게 보여줄 메시지는 긍정 조건과 부정 조건을 함께 가질 때가 많습니다.
예를 들어 장애 안내에는 요청 ID가 있어야 하지만 내부 호스트명은 없어야 할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function formatIncidentMessage(requestId) {
  return `처리에 실패했습니다. 문의 시 요청 ID ${requestId}를 알려 주세요.`;
}

test('formats a safe incident message', () => {
  const message = formatIncidentMessage('req_01J2EXAMPLE');

  assert.match(message, /요청 ID req_[A-Z0-9]+/);
  assert.doesNotMatch(message, /internal-api|localhost|127\.0\.0\.1/i);
});
```

`assert.match()`는 사용자에게 필요한 단서를 확인하고, `assert.doesNotMatch()`는 노출되면 안 되는 내부 단서를 차단합니다.
이렇게 역할을 나누면 테스트가 단순한 문자열 스냅샷보다 덜 취약해집니다.
문구가 조금 바뀌어도 핵심 계약만 유지하면 테스트가 계속 통과하기 때문입니다.

### H3. 전체 문장 고정보다 계약 중심으로 검증한다

문구 전체를 `assert.equal()`로 고정하면 작은 카피 변경에도 테스트가 자주 깨집니다.
반대로 정규식만 쓰면 너무 느슨해질 수 있습니다.
그래서 공개 메시지 테스트는 보통 핵심 문구, 요청 식별자, 금지 패턴을 조합하는 방식이 좋습니다.

```js
assert.match(message, /잠시 후 다시 시도/);
assert.match(message, /요청 ID/);
assert.doesNotMatch(message, /Bearer|password|secret/i);
assert.doesNotMatch(message, /\b\d{4}-\d{4}-\d{4}-\d{4}\b/);
```

마지막 패턴은 예시용 카드 번호 형태를 막는 검증입니다.
실제 결제 정보나 개인정보를 fixture에 넣지 않고도 "그런 형태가 공개 메시지에 남으면 안 된다"는 계약을 표현할 수 있습니다.

## 실무 체크리스트

### H3. 공개 문자열 경계에 우선 적용한다

`assert.doesNotMatch()`는 모든 문자열에 붙일 필요가 없습니다.
가장 먼저 적용할 곳은 외부로 나가는 경계입니다.
API 응답, 웹훅 응답, 사용자 알림, 공개 로그 샘플, 문서화된 에러 메시지가 좋은 후보입니다.

체크할 만한 패턴은 다음과 같습니다.

- 인증정보 이름: `token`, `secret`, `password`, `apiKey`
- 내부 주소: `localhost`, 사설 IP, 내부 서비스 호스트명
- 원본 예외: `Error:`, `Traceback`, JavaScript stack frame
- 로컬 경로: `/Users/`, `/home/`, `C:\\`
- SQL 조각: `SELECT`, `INSERT`, `UPDATE`가 원문으로 섞인 메시지

이 목록은 그대로 복사하기보다 프로젝트의 실제 노출 경계에 맞게 줄이는 편이 좋습니다.
너무 많은 금지어를 한 번에 넣으면 정상 문구까지 막고, 팀이 assertion을 우회하게 됩니다.

### H3. 테스트 데이터도 공개 가능한 값만 사용한다

민감정보 검증 테스트의 기본 원칙은 테스트 데이터가 유출되어도 문제가 없어야 한다는 것입니다.
`example.test`, `req_01J2EXAMPLE`, `test-token-placeholder`처럼 의도를 알 수 있지만 실제 값이 아닌 문자열을 사용합니다.
환경 변수에서 실제 값을 읽어 assertion에 넣는 방식은 피해야 합니다.

또한 실패 로그를 확인할 때 테스트 러너가 실제 문자열을 출력할 수 있다는 점을 고려합니다.
검증 대상 메시지에 민감정보가 들어간 상태로 실패하면, 테스트가 그 값을 그대로 노출할 수 있습니다.
따라서 production-like secret을 테스트 입력으로 쓰지 않는 규칙이 가장 중요합니다.

## 마무리

`assert.doesNotMatch()`는 "문자열이 특정 패턴과 일치하지 않아야 한다"는 계약을 간결하게 표현합니다.
공개 메시지, API 응답, 로그 샘플처럼 외부로 나가는 문자열에서 특히 유용합니다.
`assert.match()`로 필요한 단서를 확인하고, `assert.doesNotMatch()`로 민감하거나 내부적인 단서를 차단하면 전체 문장을 과하게 고정하지 않으면서도 중요한 회귀를 막을 수 있습니다.

실무에서는 정규식을 넓게 잡기보다 금지하려는 형태를 좁게 표현하고, 테스트 fixture에는 실제 개인정보나 토큰을 넣지 않는 기준을 지켜야 합니다.
그렇게 작성한 부정 검증은 보안 리뷰와 운영 장애 대응 사이에서 작지만 오래 가는 안전장치가 됩니다.

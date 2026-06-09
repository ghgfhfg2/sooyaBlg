---
layout: post
title: "Node.js HTTP 헤더 검증 가이드: validateHeaderName과 validateHeaderValue 사용법"
date: 2026-06-09 20:00:00 +0900
lang: ko
translation_key: nodejs-http-validate-header-name-value-guide
permalink: /development/blog/seo/2026/06/09/nodejs-http-validate-header-name-value-guide.html
alternates:
  ko: /development/blog/seo/2026/06/09/nodejs-http-validate-header-name-value-guide.html
  x_default: /development/blog/seo/2026/06/09/nodejs-http-validate-header-name-value-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, http, headers, validation, security, backend]
description: "Node.js http.validateHeaderName()과 validateHeaderValue()로 동적 HTTP 헤더를 미리 검증하는 방법을 정리합니다. 에러 처리, 허용 목록, 민감정보 로그 방지까지 실무 예제로 설명합니다."
---

Node.js에서 HTTP 요청이나 응답 헤더를 직접 구성할 때 헤더 이름과 값이 항상 안전한 형식이라고 가정하면 안 됩니다.
설정 파일, 프록시 입력, 사용자 지정 메타데이터처럼 외부에서 받은 값에는 잘못된 문자나 줄바꿈이 섞일 수 있습니다.

Node.js의 `http.validateHeaderName()`과 `http.validateHeaderValue()`는 헤더를 전송하기 전에 이름과 값이 유효한지 검사하는 내장 함수입니다.
검증에 실패하면 예외를 던지므로, 네트워크 요청 직전이 아니라 입력 경계에서 문제를 발견하고 명확한 에러로 바꿀 수 있습니다.

이 글에서는 두 함수의 기본 사용법, 허용 목록과 조합하는 패턴, 에러 처리, 로그에 민감한 헤더 값을 남기지 않는 방법을 정리합니다.
외부 요청에 시간 제한과 취소 정책도 적용해야 한다면 [HTTP 요청 타임아웃과 fail-fast 가이드](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)를 함께 참고하세요.

## HTTP 헤더를 미리 검증해야 하는 이유

### H3. 잘못된 헤더는 요청 전송 시점에 실패한다

HTTP 헤더 이름에는 허용되지 않는 문자를 사용할 수 없고, 헤더 값에도 줄바꿈처럼 위험하거나 잘못된 문자가 들어가면 안 됩니다.
Node.js HTTP API도 이런 값을 거부하지만, 검증 없이 바로 요청을 만들면 실패 지점이 네트워크 코드 깊숙한 곳으로 밀릴 수 있습니다.

```js
import http from 'node:http';

const request = http.request({
  hostname: 'example.com',
  headers: {
    'x-report-name': reportName
  }
});
```

`reportName`이 외부 입력이라면 요청 생성 또는 헤더 설정 과정에서 예외가 발생할 수 있습니다.
입력 경계에서 먼저 검증하면 어떤 설정이나 입력이 잘못됐는지 더 빠르게 설명할 수 있습니다.

### H3. 형식 검증과 업무 허용 정책은 서로 다르다

유효한 HTTP 헤더라고 해서 애플리케이션이 받아도 되는 헤더라는 뜻은 아닙니다.
예를 들어 `authorization`, `cookie`, `host`는 형식상 유효하지만, 외부 입력이 임의로 덮어쓰게 허용하면 보안과 라우팅 문제가 생길 수 있습니다.

따라서 헤더 처리에는 두 단계가 필요합니다.

1. Node.js 내장 함수로 HTTP 형식이 유효한지 검사한다.
2. 애플리케이션 허용 목록으로 사용할 수 있는 헤더인지 검사한다.

## validateHeaderName과 validateHeaderValue 기본 사용법

### H3. 헤더 이름을 검증한다

`validateHeaderName()`은 헤더 이름이 유효하면 반환값 없이 끝나고, 유효하지 않으면 예외를 던집니다.

```js
import { validateHeaderName } from 'node:http';

validateHeaderName('x-request-id');
validateHeaderName('content-type');
```

동적 헤더 이름을 처리할 때는 `try/catch`로 감싸 애플리케이션 에러로 변환할 수 있습니다.

```js
import { validateHeaderName } from 'node:http';

export function assertValidHeaderName(name) {
  try {
    validateHeaderName(name);
  } catch {
    throw new TypeError('유효하지 않은 HTTP 헤더 이름입니다.');
  }
}
```

원본 입력을 에러 메시지에 그대로 넣지 않으면 로그에 예상하지 못한 제어 문자나 민감한 데이터가 남는 위험을 줄일 수 있습니다.

### H3. 헤더 값을 검증한다

`validateHeaderValue()`는 검사할 헤더 이름과 값을 함께 받습니다.
값이 유효하지 않거나 정의되지 않은 경우 예외를 던질 수 있으므로, 선택적 값은 먼저 존재 여부를 확인해야 합니다.

```js
import { validateHeaderValue } from 'node:http';

validateHeaderValue('x-request-id', 'req-1234');
validateHeaderValue('content-type', 'application/json');
```

설정에서 선택적으로 받는 헤더라면 다음처럼 처리합니다.

```js
import { validateHeaderValue } from 'node:http';

export function validateOptionalHeader(name, value) {
  if (value === undefined || value === null) {
    return;
  }

  validateHeaderValue(name, String(value));
}
```

숫자나 boolean을 문자열로 바꾸는 정책은 애플리케이션에서 명시적으로 결정해야 합니다.
객체를 무조건 `String()`으로 변환하면 의도하지 않은 `[object Object]`가 전송될 수 있으므로 허용 타입도 함께 제한하는 편이 좋습니다.

## 동적 헤더를 안전하게 구성하는 패턴

### H3. 허용 목록을 먼저 적용한다

외부 입력으로 헤더 이름과 값을 모두 받는 기능은 범위를 최대한 좁혀야 합니다.
분석용 메타데이터처럼 필요한 헤더만 허용 목록에 넣고, 비교할 때는 이름을 소문자로 정규화합니다.

```js
import {
  validateHeaderName,
  validateHeaderValue
} from 'node:http';

const allowedHeaders = new Set([
  'x-request-id',
  'x-client-version',
  'x-report-type'
]);

export function buildAllowedHeaders(input) {
  const headers = {};

  for (const [rawName, rawValue] of Object.entries(input)) {
    const name = rawName.toLowerCase();

    if (!allowedHeaders.has(name)) {
      throw new Error('허용되지 않은 HTTP 헤더입니다.');
    }

    if (typeof rawValue !== 'string') {
      throw new TypeError('HTTP 헤더 값은 문자열이어야 합니다.');
    }

    validateHeaderName(name);
    validateHeaderValue(name, rawValue);
    headers[name] = rawValue;
  }

  return headers;
}
```

이 패턴은 형식상 유효하지만 애플리케이션이 허용하지 않은 헤더가 전달되는 것을 막습니다.
특히 `authorization`, `cookie`, `host`, 프록시 관련 헤더는 신뢰 경계를 검토하지 않은 채 외부 입력으로 받지 않는 편이 안전합니다.

### H3. 검증한 헤더만 요청 객체에 전달한다

검증 함수가 성공한 뒤 생성한 새 객체만 HTTP 클라이언트에 전달하면 원본 입력이 섞이는 실수를 줄일 수 있습니다.

```js
import https from 'node:https';

const headers = buildAllowedHeaders({
  'x-request-id': 'req-1234',
  'x-client-version': '2.1.0'
});

const request = https.request({
  hostname: 'api.example.com',
  path: '/reports',
  method: 'POST',
  headers
});

request.end();
```

운영 코드에서는 목적지 호스트도 고정하거나 허용 목록으로 관리해야 합니다.
헤더 검증은 잘못된 헤더 형식을 막아 주지만, 임의 목적지 요청이나 SSRF 같은 네트워크 정책 문제까지 해결하지는 않습니다.

## 에러 처리와 로그 정책

### H3. 내부 예외를 사용자용 검증 에러로 바꾼다

내장 검증 함수가 던지는 예외는 개발자에게 유용하지만, 외부 API 응답에 그대로 노출할 필요는 없습니다.
클라이언트에는 짧은 검증 메시지를 반환하고, 내부 로그에는 입력 원문 대신 필드 위치와 에러 코드만 남기는 편이 좋습니다.

```js
export function parseCustomHeaders(input) {
  try {
    return buildAllowedHeaders(input);
  } catch (error) {
    console.warn({
      event: 'invalid-custom-headers',
      errorCode: error.code ?? 'APP_HEADER_VALIDATION_FAILED'
    });

    throw new Error('사용자 지정 헤더 설정을 확인해 주세요.');
  }
}
```

예외 객체 전체를 직렬화하면 런타임이나 프레임워크에 따라 입력값이 함께 기록될 수 있습니다.
로그 정책에 맞춰 필요한 코드와 이벤트 이름만 선택적으로 남겨야 합니다.

### H3. 민감한 헤더 값은 정상 입력이어도 기록하지 않는다

`authorization`과 `cookie`는 형식이 정상이어도 민감정보입니다.
디버깅 로그에 전체 헤더 객체를 출력하지 말고, 헤더 이름이나 마스킹된 존재 여부만 기록합니다.

```js
function summarizeHeaders(headers) {
  return Object.keys(headers).sort();
}

console.info({
  event: 'outbound-request',
  headerNames: summarizeHeaders(headers)
});
```

CLI와 운영 로그에서 비밀값을 제거하는 기준은 [민감정보 로그 마스킹 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)에서도 확인할 수 있습니다.

## 테스트로 검증 정책 고정하기

### H3. 정상 입력과 거부 입력을 함께 테스트한다

허용 목록과 형식 검증은 테스트로 고정해야 나중에 정책이 느슨해지는 것을 막을 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

test('허용된 사용자 지정 헤더를 구성한다', () => {
  const headers = buildAllowedHeaders({
    'X-Request-ID': 'req-1234'
  });

  assert.deepEqual(headers, {
    'x-request-id': 'req-1234'
  });
});

test('허용 목록에 없는 헤더를 거부한다', () => {
  assert.throws(
    () => buildAllowedHeaders({ authorization: 'secret' }),
    /허용되지 않은 HTTP 헤더/
  );
});

test('줄바꿈이 포함된 헤더 값을 거부한다', () => {
  assert.throws(() => {
    buildAllowedHeaders({
      'x-request-id': 'req-1234\r\nx-extra: value'
    });
  });
});
```

보안 관련 검증은 성공 사례만 확인하면 부족합니다.
거부해야 하는 헤더 이름, 값, 타입을 테스트에 명시해 애플리케이션의 신뢰 경계를 문서화해야 합니다.

## 실무 적용 체크리스트

### H3. 입력 경계에서 확인할 항목

- 동적 헤더 이름과 값에 내장 검증 함수를 적용한다.
- 형식 검증과 별도로 애플리케이션 허용 목록을 둔다.
- 외부 입력이 인증, 쿠키, 호스트, 프록시 헤더를 덮어쓰지 못하게 한다.
- 헤더 값에 허용할 타입과 최대 길이를 별도로 정한다.
- 검증한 값으로 새 헤더 객체를 만들어 요청에 전달한다.

### H3. 운영 로그에서 확인할 항목

- 전체 헤더 객체를 로그에 출력하지 않는다.
- 토큰, 쿠키, 세션 식별자 같은 민감한 값을 기록하지 않는다.
- 실패 로그에는 원본 입력보다 이벤트 이름과 에러 코드를 남긴다.
- 정상 요청 로그에서도 필요한 헤더 이름만 기록한다.

웹훅처럼 서명 헤더를 처리한다면 값의 형식 검증뿐 아니라 원문 본문과 서명을 함께 검증해야 합니다.
관련 구현은 [Node.js Web Crypto HMAC 웹훅 서명 검증 가이드](/development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html)를 참고하세요.

## FAQ

### H3. validateHeaderName과 validateHeaderValue만 쓰면 보안 문제가 해결되나요?

아닙니다.
두 함수는 HTTP 헤더 형식이 유효한지 검사합니다.
허용할 헤더 종류, 값 길이, 인증 정보 처리, 목적지 호스트 같은 애플리케이션 보안 정책은 별도로 적용해야 합니다.

### H3. 헤더 이름은 소문자로 바꿔야 하나요?

HTTP 헤더 이름은 대소문자를 구분하지 않으므로 허용 목록 비교와 중복 방지를 위해 소문자로 정규화하는 것이 편리합니다.
다만 원본 표기 보존이 필요한 특별한 요구가 있다면 사용하는 HTTP 클라이언트의 동작도 확인해야 합니다.

### H3. 검증 실패 예외 메시지를 그대로 API 응답에 보내도 되나요?

권장하지 않습니다.
내부 구현 정보나 입력 일부가 노출될 수 있으므로, 외부 응답에는 일반화된 검증 메시지를 사용하고 내부 로그에도 민감한 헤더 값을 남기지 않아야 합니다.

## 마무리

`http.validateHeaderName()`과 `http.validateHeaderValue()`는 동적으로 구성한 HTTP 헤더가 전송 가능한 형식인지 미리 확인하는 간단한 내장 도구입니다.
네트워크 요청 직전에 실패하게 두는 대신 입력 경계에서 검증하면 에러 처리와 테스트가 명확해집니다.

다만 형식 검증은 첫 단계일 뿐입니다.
허용 목록, 타입과 길이 제한, 민감정보 로그 방지, 목적지 네트워크 정책을 함께 적용해야 동적 헤더 기능을 안전하게 운영할 수 있습니다.

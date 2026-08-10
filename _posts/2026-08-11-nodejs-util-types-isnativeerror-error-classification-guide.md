---
layout: post
title: "Node.js util.types.isNativeError 가이드: 에러 객체를 안전하게 분류하는 법"
date: 2026-08-11 08:00:00 +0900
lang: ko
translation_key: nodejs-util-types-isnativeerror-error-classification-guide
permalink: /development/blog/seo/2026/08/11/nodejs-util-types-isnativeerror-error-classification-guide.html
alternates:
  ko: /development/blog/seo/2026/08/11/nodejs-util-types-isnativeerror-error-classification-guide.html
  x_default: /development/blog/seo/2026/08/11/nodejs-util-types-isnativeerror-error-classification-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, util, util-types, isnativeerror, error-handling, logging, backend, javascript]
description: "Node.js util.types.isNativeError로 catch 값과 외부 경계에서 넘어온 값을 안전하게 에러 객체로 분류하는 방법을 정리합니다. instanceof Error의 한계, 에러 정규화, cause 보존, 민감정보 로그 점검까지 실무 예제로 설명합니다."
---

Node.js 애플리케이션에서 에러 처리는 생각보다 자주 흐트러집니다.
`catch (error)` 안에 들어온 값이 항상 `Error` 객체라고 믿고 `error.message`, `error.stack`, `error.code`를 바로 읽으면, 문자열을 throw한 코드나 외부 라이브러리의 특이한 예외 때문에 로깅 코드가 다시 실패할 수 있습니다.
운영 장애 상황에서는 원래 문제보다 "에러를 기록하다가 난 에러"가 더 골치 아플 때도 있습니다.

Node.js의 `util.types.isNativeError()`는 값이 네이티브 Error 객체인지 확인할 수 있는 작은 판별 도구입니다.
`instanceof Error`보다 의도가 분명하고, 여러 실행 컨텍스트가 섞이는 코드에서도 에러 분류 기준을 한 곳으로 모으기 좋습니다.
이 글에서는 `isNativeError()`를 어디에 쓰면 좋은지, 문자열·객체·시스템 에러를 어떻게 정규화할지, 로그에 민감정보를 남기지 않기 위한 기준을 정리합니다.

에러 원인을 감싸서 보존하는 패턴은 [Node.js error cause 가이드](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)를 함께 보면 좋습니다.
외부 API 호출 실패를 timeout과 재시도로 분류해야 한다면 [Node.js fetch 에러 분류 가이드](/development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html)도 이어서 참고할 수 있습니다.

## isNativeError가 필요한 이유

### H3. catch 값은 항상 Error가 아니다

JavaScript에서는 `throw` 뒤에 거의 아무 값이나 올 수 있습니다.
팀 코드에서는 `Error`를 던지기로 약속했더라도, 오래된 코드나 외부 패키지, 테스트 fixture에서는 문자열이나 plain object가 들어올 수 있습니다.

```js
try {
  throw 'upload failed';
} catch (error) {
  console.log(error.message); // undefined
}
```

이 코드는 바로 터지지는 않지만, 에러 메시지가 비어 버립니다.
더 나쁜 경우는 로거나 분류 함수가 `error.stack.split('\n')`처럼 값을 확정적으로 다루다가 추가 예외를 만드는 상황입니다.

그래서 에러 처리의 첫 단계는 "이 값이 정말 Error 계열인가"를 확인하는 일입니다.

```js
import { types } from 'node:util';

export function isErrorLike(value) {
  return types.isNativeError(value);
}
```

`isNativeError()`는 이 판별을 코드 의도로 드러냅니다.
`typeof error === 'object'`나 `'message' in error` 같은 느슨한 검사보다 읽는 사람이 더 빨리 이해할 수 있습니다.

### H3. instanceof Error만으로는 경계가 흐릴 수 있다

일반적인 애플리케이션 코드에서는 `error instanceof Error`도 자주 충분합니다.
하지만 VM, Worker, 테스트 격리, 번들러가 만든 다른 실행 컨텍스트가 섞이면 `instanceof` 기준이 기대와 다르게 보일 수 있습니다.

```js
if (error instanceof Error) {
  logger.error({ stack: error.stack }, error.message);
}
```

이 코드는 간단하지만 "현재 전역 객체의 Error 생성자"를 기준으로 판단합니다.
대부분의 서버 코드에서는 문제가 없더라도, 여러 컨텍스트를 다루는 라이브러리나 테스트 도구에서는 더 명확한 판별 함수를 두는 편이 안전합니다.

`util.types.isNativeError()`는 Node.js가 제공하는 타입 판별 API입니다.
에러 분류 유틸리티에서 이 함수를 사용하면 "에러 객체로 다룰 수 있는 값"의 기준을 한 곳에 고정할 수 있습니다.

## 기본 사용법

### H3. 에러 정규화 함수를 먼저 만든다

실무에서는 `isNativeError()`를 호출부마다 직접 쓰기보다, 로깅과 분류에 맞는 정규화 함수를 만드는 편이 좋습니다.

```js
import { types } from 'node:util';

export function normalizeCaughtError(value) {
  if (types.isNativeError(value)) {
    return value;
  }

  if (typeof value === 'string') {
    return new Error(value);
  }

  return new Error('Non-error value was thrown', {
    cause: value
  });
}
```

이 함수의 목적은 모든 실패를 같은 모양으로 다루게 만드는 것입니다.
이후 로깅, 재시도 판단, HTTP 응답 변환 코드는 `Error` 객체를 받는다고 가정할 수 있습니다.

```js
try {
  await runJob();
} catch (caught) {
  const error = normalizeCaughtError(caught);

  logger.error({
    name: error.name,
    message: error.message,
    stack: error.stack
  }, 'job failed');
}
```

정규화 단계에서 원본 값을 `cause`로 남기면 디버깅 단서를 잃지 않습니다.
다만 원본 값이 외부 입력을 포함할 수 있다면 그대로 로그에 펼치지 않는 것이 좋습니다.

### H3. public message와 diagnostic detail을 분리한다

에러 객체라고 해서 모든 필드를 사용자 응답이나 외부 로그에 그대로 노출해도 되는 것은 아닙니다.
`message`에는 파일 경로, SQL 조각, 내부 서비스 이름, 사용자 입력이 섞일 수 있습니다.
`stack`은 운영 로그에는 유용하지만 클라이언트 응답에는 보통 과합니다.

아래처럼 사용자에게 보여 줄 메시지와 내부 진단 정보를 분리하세요.

```js
export function toErrorResponse(caught) {
  const error = normalizeCaughtError(caught);

  return {
    status: 500,
    body: {
      code: 'internal_error',
      message: '요청을 처리하지 못했습니다.'
    },
    diagnostic: {
      name: error.name,
      message: error.message,
      stack: error.stack
    }
  };
}
```

이 구조에서는 응답 본문이 안정적이고, 내부 로그에는 원인 분석에 필요한 정보가 남습니다.
민감정보 제거 기준이 필요하다면 [CLI 출력값 민감정보 제거 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)의 원칙을 로그에도 그대로 적용할 수 있습니다.

## 에러 분류 패턴

### H3. 시스템 에러 code는 별도로 읽는다

Node.js 시스템 에러에는 `code`, `errno`, `syscall`, `path` 같은 필드가 붙는 경우가 많습니다.
하지만 TypeScript나 런타임 관점에서 모든 `Error`가 이 필드를 가진 것은 아닙니다.
먼저 네이티브 에러인지 확인하고, 그다음 좁은 형태로 분류하는 편이 안전합니다.

```js
import { types } from 'node:util';

export function getSystemErrorCode(value) {
  if (!types.isNativeError(value)) {
    return null;
  }

  if (
    'code' in value &&
    typeof value.code === 'string'
  ) {
    return value.code;
  }

  return null;
}
```

파일 시스템 자동화에서는 `ENOENT`, `EACCES`, `EISDIR` 같은 code에 따라 사용자 메시지나 재시도 여부를 나눌 수 있습니다.
중요한 점은 code가 없을 때도 기본 경로가 실패하지 않도록 만드는 것입니다.

```js
try {
  await readConfigFile();
} catch (caught) {
  const code = getSystemErrorCode(caught);

  if (code === 'ENOENT') {
    throw new Error('설정 파일을 찾을 수 없습니다.', {
      cause: normalizeCaughtError(caught)
    });
  }

  throw normalizeCaughtError(caught);
}
```

파일 경로나 권한 문제를 다루는 흐름은 [Node.js fs.promises.access 가이드](/development/blog/seo/2026/07/30/nodejs-fspromises-access-file-permission-check-guide.html)와 함께 보면 더 자연스럽습니다.

### H3. 재시도 가능한 에러와 아닌 에러를 나눈다

외부 호출이나 배치 작업에서는 모든 에러를 똑같이 재시도하면 안 됩니다.
네트워크 timeout처럼 다시 시도할 가치가 있는 실패도 있지만, 입력 검증 실패나 인증 실패는 반복해도 성공하지 않습니다.

`isNativeError()`는 첫 판별일 뿐이고, 실제 재시도 정책은 도메인 규칙과 함께 정해야 합니다.

```js
export function classifyError(caught) {
  const error = normalizeCaughtError(caught);
  const code = getSystemErrorCode(error);

  if (code === 'ECONNRESET' || code === 'ETIMEDOUT') {
    return {
      retryable: true,
      reason: code,
      error
    };
  }

  if (error.name === 'AbortError' || error.name === 'TimeoutError') {
    return {
      retryable: true,
      reason: error.name,
      error
    };
  }

  return {
    retryable: false,
    reason: error.name,
    error
  };
}
```

이렇게 분류 객체를 만들면 호출부가 더 단순해집니다.

```js
const result = classifyError(caught);

logger.warn({
  retryable: result.retryable,
  reason: result.reason
}, 'request failed');

if (!result.retryable) {
  throw result.error;
}
```

에러 이름과 code만으로 완벽한 정책을 만들 수는 없습니다.
HTTP status, 요청 메서드, idempotency key, 남은 deadline까지 함께 봐야 하는 호출도 있습니다.
그래도 기본 에러 정규화가 되어 있으면 정책을 확장하기가 훨씬 쉽습니다.

## 로깅과 관측성에 적용하기

### H3. 에러 로그 스키마를 고정한다

장애 대응에서는 로그 필드가 매번 달라지는 것이 큰 비용입니다.
`message`가 있으면 `message`, 없으면 `error`, 어떤 곳은 `err`, 어떤 곳은 문자열 한 줄로 남기면 검색과 집계가 어려워집니다.

에러 정규화 함수를 기준으로 로그 스키마를 고정하세요.

```js
export function serializeErrorForLog(caught) {
  const error = normalizeCaughtError(caught);
  const code = getSystemErrorCode(error);

  return {
    name: error.name,
    message: redactErrorMessage(error.message),
    code,
    stack: error.stack,
    causeName: error.cause instanceof Error ? error.cause.name : undefined
  };
}

function redactErrorMessage(message) {
  return message
    .replaceAll(/Bearer\s+[A-Za-z0-9._~+/=-]+/g, 'Bearer <redacted>')
    .replaceAll(/token=[^&\s]+/g, 'token=<redacted>');
}
```

이 예제는 최소한의 마스킹만 보여 줍니다.
실서비스에서는 API 키 형식, 내부 계정 ID, 이메일, 경로 정책 등 프로젝트별 규칙을 별도 유틸리티로 관리하는 편이 좋습니다.

### H3. trace와 metric에는 낮은 카디널리티 값만 넣는다

에러 메시지를 metric label이나 trace attribute에 그대로 넣으면 카디널리티가 급격히 늘어납니다.
사용자 ID, URL query, 파일 경로가 매번 다른 값으로 들어가기 때문입니다.

관측성 데이터에는 아래처럼 제한된 값만 넣는 편이 좋습니다.

- `error.name`
- 정규화된 `code`
- 재시도 가능 여부
- 라우트 이름이나 작업 이름
- 내부에서 정의한 에러 분류값

상세 메시지와 stack은 로그나 별도 진단 자료에 남기고, metric에는 `reason: "ECONNRESET"`처럼 작은 분류값만 사용하세요.
이 기준은 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/08/07/nodejs-diagnostics-channel-observability-guide.html)의 channel 설계와도 잘 맞습니다.

## 테스트 기준

### H3. 문자열 throw와 plain object를 함께 검증한다

에러 정규화 함수는 작은 코드지만 운영 영향이 큽니다.
정상적인 `Error`만 테스트하면 실제로 필요한 방어 경로가 비어 있을 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { normalizeCaughtError } from './errors.js';

test('normalizeCaughtError keeps native Error values', () => {
  const original = new TypeError('invalid input');
  const normalized = normalizeCaughtError(original);

  assert.equal(normalized, original);
});

test('normalizeCaughtError converts strings to Error', () => {
  const normalized = normalizeCaughtError('failed');

  assert.equal(normalized.name, 'Error');
  assert.equal(normalized.message, 'failed');
});

test('normalizeCaughtError preserves non-error value as cause', () => {
  const value = { status: 503 };
  const normalized = normalizeCaughtError(value);

  assert.equal(normalized.message, 'Non-error value was thrown');
  assert.equal(normalized.cause, value);
});
```

테스트 이름은 입력 형태와 기대 동작을 드러내는 것이 좋습니다.
`normalizes error`보다 `converts strings to Error`가 실패 원인을 더 빨리 알려 줍니다.

### H3. 로그 직렬화 테스트에는 민감정보 예시를 넣는다

로그 직렬화 함수가 있다면 토큰성 문자열을 마스킹하는지도 테스트해야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { serializeErrorForLog } from './errors.js';

test('serializeErrorForLog redacts bearer tokens', () => {
  const fields = serializeErrorForLog(
    new Error('request failed: Bearer abc.def.ghi')
  );

  assert.equal(
    fields.message,
    'request failed: Bearer <redacted>'
  );
});
```

이런 테스트는 보안 기능이라기보다 회귀 방지 장치입니다.
나중에 로깅 포맷을 바꿀 때 실수로 민감값이 다시 노출되는 것을 막아 줍니다.

## 실무 체크리스트

### H3. isNativeError 도입 전에 확인할 것

`util.types.isNativeError()`를 쓰기 전에 아래 기준을 정하면 코드가 더 오래 갑니다.

- `catch` 값은 반드시 정규화 함수로 통과시키는가?
- 문자열 throw와 plain object throw를 어떻게 기록할지 정했는가?
- 사용자 응답 메시지와 내부 진단 메시지를 분리했는가?
- `cause`를 보존하되 원본 값을 무제한 로그에 펼치지 않는가?
- metric label에는 낮은 카디널리티 분류값만 넣는가?
- 시스템 에러의 `code`가 없을 때도 fallback이 있는가?

이 중 가장 중요한 것은 정규화 함수의 위치입니다.
서비스 파일마다 제각각 에러 판별을 넣기보다 `errors.js`, `error-normalizer.ts` 같은 작은 모듈로 모으는 편이 유지보수에 좋습니다.

### H3. 모든 문제를 타입 판별로 해결하려 하지 않는다

`isNativeError()`는 유용하지만 에러 처리 전체를 대신하지 않습니다.
값이 Error인지 확인하는 것과, 그 실패가 재시도 가능한지, 사용자에게 어떤 메시지를 줄지, 어떤 로그를 남길지는 다른 문제입니다.

좋은 흐름은 보통 이렇게 나뉩니다.

1. 알 수 없는 `caught` 값을 `Error`로 정규화한다.
2. 시스템 code, 에러 name, 도메인 필드로 분류한다.
3. 사용자 응답과 내부 로그를 분리한다.
4. `cause`와 stack은 진단용으로 보존한다.
5. 민감정보와 고카디널리티 값을 로그·메트릭에서 줄인다.

이렇게 나누면 에러 처리 코드가 과하게 똑똑해지지 않습니다.
작은 함수들이 각자 한 가지 책임만 갖기 때문에 테스트도 쉬워집니다.

## FAQ

### H3. util.types.isNativeError와 instanceof Error 중 무엇을 써야 하나요?

단순한 애플리케이션 내부 코드에서는 `instanceof Error`도 충분한 경우가 많습니다.
하지만 에러 정규화 유틸리티, 라이브러리, Worker나 VM처럼 실행 컨텍스트가 섞일 수 있는 코드에서는 `util.types.isNativeError()`가 의도를 더 명확히 드러냅니다.

### H3. isNativeError가 true면 안전하게 로그에 남겨도 되나요?

아닙니다.
에러 객체 여부와 로그 공개 가능성은 별개입니다.
`message`, `stack`, `cause`에는 내부 경로나 토큰, 사용자 입력이 섞일 수 있으므로 로그 직렬화 단계에서 마스킹 기준을 적용해야 합니다.

### H3. non-error throw는 전부 버그로 봐야 하나요?

대부분의 애플리케이션 코드에서는 `Error`를 throw하는 기준이 좋습니다.
다만 외부 라이브러리나 오래된 코드에서 다른 값이 넘어올 수 있으므로, 경계에서는 방어적으로 정규화하고 내부 코드에서는 lint나 리뷰로 `throw new Error()` 패턴을 유지하는 편이 현실적입니다.

## 마무리

`util.types.isNativeError()`는 작은 API지만, Node.js 에러 처리의 첫 단추를 단단하게 만드는 데 도움이 됩니다.
핵심은 이 함수를 여기저기 흩뿌리는 것이 아니라, `catch` 값을 정규화하는 공통 경계에 넣는 것입니다.

정규화, 분류, 응답 메시지, 진단 로그, 민감정보 마스킹을 분리해 두면 에러 처리는 훨씬 예측 가능해집니다.
운영 중 실패가 늘어났을 때도 "무슨 값이 넘어왔는지"부터 다시 추측하지 않고, 일관된 로그와 분류값으로 원인을 좁힐 수 있습니다.

## 내부링크

- [Node.js error cause 가이드](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)
- [Node.js fetch 에러 분류 가이드](/development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html)
- [CLI 출력값 민감정보 제거 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)

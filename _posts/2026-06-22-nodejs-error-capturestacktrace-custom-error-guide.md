---
layout: post
title: "Node.js Error.captureStackTrace 가이드: 커스텀 에러의 스택 트레이스를 깔끔하게 만드는 법"
date: 2026-06-22 20:00:00 +0900
lang: ko
translation_key: nodejs-error-capturestacktrace-custom-error-guide
permalink: /development/blog/seo/2026/06/22/nodejs-error-capturestacktrace-custom-error-guide.html
alternates:
  ko: /development/blog/seo/2026/06/22/nodejs-error-capturestacktrace-custom-error-guide.html
  x_default: /development/blog/seo/2026/06/22/nodejs-error-capturestacktrace-custom-error-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, error, capturestacktrace, stacktrace, customerror, debugging, backend]
description: "Node.js Error.captureStackTrace()로 커스텀 에러의 생성자 프레임을 숨기고, Error.stackTraceLimit과 error.cause를 함께 사용해 운영 로그에서 읽기 쉬운 스택 트레이스를 남기는 방법을 정리합니다."
---

Node.js에서 커스텀 에러 클래스를 만들면 로그가 더 읽기 쉬워집니다.
`code`, `statusCode`, `retryable`, `details` 같은 필드를 붙여 장애 원인과 처리 방식을 분리할 수 있기 때문입니다.
하지만 커스텀 에러를 잘못 만들면 정작 가장 중요한 스택 트레이스가 생성자, 팩토리 함수, 래퍼 함수로 흐려질 수 있습니다.

이때 사용할 수 있는 도구가 `Error.captureStackTrace()`입니다.
[Node.js 공식 문서](https://nodejs.org/api/errors.html)에 따르면 이 API는 대상 객체에 `.stack` 속성을 만들고, 두 번째 인자로 넘긴 함수 위쪽 프레임을 스택에서 제외할 수 있습니다.
MDN도 이 API를 V8 기반 환경에서 제공되는 비표준 함수로 설명하며, 주된 용도는 커스텀 에러 객체에 스택 트레이스를 설치하는 것이라고 정리합니다.
이 글에서는 `Error.captureStackTrace()`, `Error.stackTraceLimit`, `error.cause`를 함께 사용해 운영 로그에 적합한 에러 객체를 설계하는 방법을 정리합니다.

에러를 감싸서 원인을 보존하는 패턴은 [Node.js Error cause 가이드](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)와 연결됩니다.
요청 단위 로그에 에러를 묶어 보고 싶다면 [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)를 함께 보면 좋습니다.
로그에 민감한 값을 남기지 않는 원칙은 [로그 예제 sanitization 가이드](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)를 기준으로 잡을 수 있습니다.

## Error.captureStackTrace가 필요한 이유

### H3. 커스텀 에러의 구현 세부사항을 숨긴다

커스텀 에러 클래스는 보통 생성자 안에서 여러 필드를 정리합니다.
문제는 스택 트레이스에 생성자 자체가 섞이면 실제 실패 지점보다 에러 객체를 만드는 코드가 더 크게 보일 수 있다는 점입니다.

```js
class AppError extends Error {
  constructor(message, options = {}) {
    super(message, options);

    this.name = 'AppError';
    this.code = options.code ?? 'APP_ERROR';
    this.statusCode = options.statusCode ?? 500;
  }
}

function readRequiredConfig(name) {
  throw new AppError(`Missing required config: ${name}`, {
    code: 'CONFIG_REQUIRED',
    statusCode: 500
  });
}
```

이 코드도 동작은 합니다.
하지만 에러 클래스를 여러 단계로 감싸거나 팩토리 함수를 쓰기 시작하면 로그에서 중요한 호출 위치가 아래로 밀릴 수 있습니다.
운영 로그를 읽는 사람은 "어떤 헬퍼가 에러를 만들었는가"보다 "어떤 업무 코드에서 실패가 시작됐는가"를 먼저 알아야 합니다.

### H3. stack은 사용자를 위한 메시지가 아니라 개발자를 위한 진단 정보다

스택 트레이스는 사용자에게 보여 줄 텍스트가 아닙니다.
파일 경로, 함수명, 내부 구조가 포함될 수 있으므로 API 응답에 그대로 내려보내면 보안과 사용성 모두에서 좋지 않습니다.
대신 내부 로그, 에러 리포팅 도구, 장애 분석용 이벤트에 남기는 진단 정보로 다뤄야 합니다.

`Error.captureStackTrace()`는 이 진단 정보를 더 읽기 좋게 만드는 API입니다.
에러를 더 안전하게 만드는 API가 아니고, 에러 처리를 대신해 주는 API도 아닙니다.
사용자 응답에는 안정적인 `code`와 짧은 메시지를 쓰고, 내부 로그에는 스택과 원인을 남기는 식으로 역할을 나누는 것이 좋습니다.

## 기본 사용법

### H3. 생성자를 스택에서 제외한다

가장 흔한 패턴은 커스텀 에러 생성자 안에서 `Error.captureStackTrace(this, AppError)`를 호출하는 것입니다.
두 번째 인자로 생성자 함수를 넘기면 그 함수와 그 위쪽 프레임이 스택에서 제외됩니다.

```js
export class AppError extends Error {
  constructor(message, options = {}) {
    super(message, { cause: options.cause });

    this.name = 'AppError';
    this.code = options.code ?? 'APP_ERROR';
    this.statusCode = options.statusCode ?? 500;
    this.retryable = options.retryable ?? false;

    Error.captureStackTrace?.(this, AppError);
  }
}
```

`Error.captureStackTrace?.`처럼 optional chaining을 쓴 이유는 이 API가 표준 ECMAScript API가 아니기 때문입니다.
Node.js에서는 사용할 수 있지만, 브라우저나 다른 JavaScript 런타임까지 공유하는 패키지라면 존재 여부를 확인하는 편이 안전합니다.

### H3. 팩토리 함수까지 숨기려면 제외 기준을 바꾼다

에러 생성을 함수로 감싸는 코드도 많습니다.
예를 들어 입력 검증 실패를 일관된 형태로 만들기 위해 `validationError()` 같은 팩토리를 둘 수 있습니다.
이 경우 생성자만 숨기면 팩토리 함수가 스택에 남습니다.

```js
class AppError extends Error {
  constructor(message, options = {}) {
    super(message, { cause: options.cause });

    this.name = 'AppError';
    this.code = options.code ?? 'APP_ERROR';
    this.statusCode = options.statusCode ?? 500;

    Error.captureStackTrace?.(
      this,
      options.stackStartFn ?? AppError
    );
  }
}

function validationError(message, details) {
  return new AppError(message, {
    code: 'VALIDATION_FAILED',
    statusCode: 400,
    details,
    stackStartFn: validationError
  });
}
```

이렇게 하면 스택 트레이스는 `validationError()` 내부가 아니라 그 팩토리를 호출한 업무 코드부터 시작됩니다.
로그를 보는 입장에서는 훨씬 직접적입니다.
다만 너무 많은 래퍼를 숨기면 실제 에러 생성 흐름을 추적하기 어려울 수 있으므로, 반복적으로 노이즈가 되는 공통 헬퍼에만 적용하는 편이 좋습니다.

## 운영용 커스텀 에러 설계

### H3. code와 statusCode는 안정적인 계약으로 둔다

에러 메시지는 시간이 지나며 바뀔 수 있습니다.
한국어 문구를 다듬거나, 더 친절한 설명으로 바꾸거나, 내부 용어를 제거할 수 있습니다.
반면 `code`는 모니터링, 테스트, 클라이언트 분기에서 쓰일 수 있으므로 안정적인 값으로 관리해야 합니다.

```js
export class HttpError extends Error {
  constructor(message, options = {}) {
    super(message, { cause: options.cause });

    this.name = 'HttpError';
    this.code = options.code;
    this.statusCode = options.statusCode;
    this.expose = options.expose ?? this.statusCode < 500;

    Error.captureStackTrace?.(this, options.stackStartFn ?? HttpError);
  }
}

export function notFound(resourceName) {
  return new HttpError(`${resourceName} not found`, {
    code: 'NOT_FOUND',
    statusCode: 404,
    stackStartFn: notFound
  });
}
```

API 응답에서는 `statusCode`, `code`, 공개 가능한 짧은 메시지만 내려보내고, 내부 로그에서는 `stack`, `cause`, 안전하게 정리한 `details`를 남깁니다.
이 구분을 해 두면 디버깅 품질과 외부 노출 통제를 동시에 챙길 수 있습니다.

### H3. details에는 재현 가능한 단서를 넣되 민감정보는 빼야 한다

운영 에러에는 추가 맥락이 필요합니다.
예를 들어 어떤 입력 필드가 실패했는지, 어느 외부 API 단계에서 실패했는지, 재시도 가능한 오류인지 같은 정보가 있으면 장애 대응이 빨라집니다.

```js
throw new HttpError('Invalid checkout request', {
  code: 'CHECKOUT_VALIDATION_FAILED',
  statusCode: 400,
  details: {
    fields: ['shippingAddress.postalCode', 'items'],
    itemCount: items.length
  }
});
```

여기서 주의할 점은 `details`가 로그로 흘러갈 수 있다는 것입니다.
비밀번호, 인증 토큰, 전체 카드번호, 주민등록번호, 개인 연락처 같은 값은 넣지 않아야 합니다.
필요한 경우 필드 이름, 개수, 길이, 마스킹된 접미사처럼 재현에 필요한 최소 정보만 남깁니다.

## error.cause와 함께 쓰기

### H3. 원본 에러를 잃지 않고 메시지를 바꾼다

운영 코드에서는 낮은 수준의 에러를 그대로 던지기보다 업무 맥락을 붙여 다시 던지는 경우가 많습니다.
예를 들어 데이터베이스 드라이버 에러를 `ORDER_LOOKUP_FAILED`로 감싸면 호출자는 더 안정적인 코드로 분기할 수 있습니다.
이때 원본 에러를 `cause`에 넣으면 디버깅 단서를 잃지 않습니다.

```js
async function loadOrder(orderId) {
  try {
    return await orderRepository.findById(orderId);
  } catch (cause) {
    throw new AppError('Failed to load order', {
      code: 'ORDER_LOOKUP_FAILED',
      statusCode: 500,
      cause
    });
  }
}
```

`Error.captureStackTrace()`는 새 에러의 스택을 보기 좋게 만들고, `cause`는 원본 실패를 보존합니다.
둘은 대체 관계가 아니라 보완 관계입니다.
겉 에러는 업무 문맥을 설명하고, 안쪽 cause는 실제 하위 시스템의 실패를 설명합니다.

### H3. cause 체인은 로깅할 때 깊이를 제한한다

`cause`를 계속 감싸다 보면 에러 체인이 길어질 수 있습니다.
모든 cause의 전체 stack을 매번 JSON 로그에 넣으면 로그 비용이 커지고, 같은 정보가 반복될 수 있습니다.
따라서 로거에서 깊이 제한과 필드 제한을 두는 편이 좋습니다.

```js
function serializeError(error, depth = 0) {
  if (!(error instanceof Error) || depth > 3) {
    return undefined;
  }

  return {
    name: error.name,
    message: error.message,
    code: error.code,
    statusCode: error.statusCode,
    stack: error.stack,
    cause: serializeError(error.cause, depth + 1)
  };
}
```

운영 환경에서는 이 함수에 redaction 정책을 추가해야 합니다.
`details`, `request`, `response` 같은 객체를 그대로 펼치지 말고 허용 목록 방식으로 필요한 값만 골라 남기는 것이 안전합니다.

## Error.stackTraceLimit 다루기

### H3. 기본 프레임 수는 충분하지 않을 수 있다

Node.js 문서에 따르면 `Error.stackTraceLimit`은 `new Error().stack`과 `Error.captureStackTrace()`가 수집하는 스택 프레임 수를 제어합니다.
기본값은 보통 짧은 편이라, 복잡한 프레임워크나 테스트 환경에서는 원인을 찾기에 부족할 수 있습니다.

```js
Error.stackTraceLimit = 30;
```

이 설정은 설정 이후에 캡처되는 스택에 영향을 줍니다.
애플리케이션 시작 시점에 한 번 정하고, 요청 처리 중에 자주 바꾸지 않는 편이 좋습니다.
테스트나 로컬 디버깅에서는 값을 크게 잡고, 운영에서는 로그 비용과 가독성을 고려해 적당한 수준으로 제한합니다.

### H3. 특정 구간에서만 임시 조정할 수 있다

드물게 스택을 만들되 현재 함수의 스택 비용을 줄이고 싶은 경우가 있습니다.
Node.js 문서 예시처럼 임시로 `stackTraceLimit`을 바꾼 뒤 원래 값을 복구하는 패턴을 사용할 수 있습니다.

```js
function createErrorWithoutInitialStack(message) {
  const previousLimit = Error.stackTraceLimit;

  try {
    Error.stackTraceLimit = 0;
    return new Error(message);
  } finally {
    Error.stackTraceLimit = previousLimit;
  }
}
```

다만 전역 상태를 바꾸는 방식이므로 일반적인 웹 요청 코드에서는 신중해야 합니다.
동시에 여러 요청을 처리하는 서버에서는 예상하지 못한 위치의 에러 캡처에 영향을 줄 수 있습니다.
대부분의 서비스에서는 시작 시점에 고정값을 설정하는 정도가 더 단순하고 안전합니다.

## 로깅과 응답을 분리하는 패턴

### H3. 내부 로그에는 stack을 남긴다

장애 대응을 위해서는 내부 로그에 스택이 필요합니다.
단순히 `error.message`만 남기면 같은 메시지가 여러 위치에서 발생할 때 원인을 구분하기 어렵습니다.

```js
function logError(logger, error, context = {}) {
  logger.error({
    ...context,
    error: serializeError(error)
  }, 'request failed');
}
```

여기서 `context`에는 요청 ID, route 이름, 사용자 식별자의 해시값, 배포 버전처럼 진단에 필요한 값을 넣을 수 있습니다.
개인정보와 인증 정보는 제외해야 합니다.
스택 트레이스 자체에도 파일 경로나 내부 함수명이 들어가므로 외부 응답으로 재사용하지 않습니다.

### H3. 사용자 응답에는 안정적인 형태만 내려보낸다

API 응답은 에러를 디버깅하는 사람보다 요청을 보낸 클라이언트를 위해 설계해야 합니다.
따라서 내부 스택 대신 안정적인 `code`와 공개 가능한 메시지를 내려보냅니다.

```js
function toErrorResponse(error) {
  const statusCode = error.statusCode ?? 500;
  const expose = error.expose === true || statusCode < 500;

  return {
    statusCode,
    body: {
      error: {
        code: error.code ?? 'INTERNAL_SERVER_ERROR',
        message: expose ? error.message : 'Internal server error'
      }
    }
  };
}
```

이 패턴을 쓰면 운영자는 내부 로그에서 상세 스택을 확인하고, 클라이언트는 문서화된 에러 코드로 대응할 수 있습니다.
`Error.captureStackTrace()`는 내부 로그의 품질을 높이고, 응답 설계는 외부 계약을 안정화합니다.

## 테스트 포인트

### H3. 생성자 프레임이 숨겨졌는지 확인한다

커스텀 에러는 단위 테스트로 의도를 고정해 두는 것이 좋습니다.
특히 공통 에러 클래스는 여러 모듈에서 쓰이므로 스택 형태가 의도와 다르게 바뀌면 로그 품질이 떨어질 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

test('AppError hides its constructor from stack trace', () => {
  function createFailure() {
    return new AppError('boom');
  }

  const error = createFailure();

  assert.match(error.stack, /createFailure/);
  assert.doesNotMatch(error.stack, /new AppError/);
});
```

스택 문자열은 Node.js 버전이나 실행 방식에 따라 조금씩 달라질 수 있습니다.
그래서 전체 문자열을 snapshot으로 고정하기보다, 꼭 필요한 함수명이 포함되거나 제외되는지만 검사하는 편이 덜 깨집니다.

### H3. code와 cause가 유지되는지 확인한다

에러 클래스에서 더 중요한 계약은 구조화된 필드입니다.
`captureStackTrace`를 적용하면서 `cause`, `code`, `statusCode` 같은 값이 빠지지 않았는지 테스트합니다.

```js
test('AppError keeps operational fields and cause', () => {
  const cause = new Error('database unavailable');

  const error = new AppError('Failed to load order', {
    code: 'ORDER_LOOKUP_FAILED',
    statusCode: 500,
    cause
  });

  assert.equal(error.code, 'ORDER_LOOKUP_FAILED');
  assert.equal(error.statusCode, 500);
  assert.equal(error.cause, cause);
  assert.match(error.stack, /Failed to load order/);
});
```

이 테스트는 에러 객체가 운영 로그와 API 변환 계층에서 기대한 형태로 동작하는지 확인합니다.
스택 트레이스는 진단 품질이고, 구조화 필드는 시스템 간 계약입니다.
둘을 함께 지켜야 에러 처리가 안정적입니다.

## 자주 하는 실수

### H3. Error를 상속하지 않는 객체에만 필요하다고 오해한다

`Error.captureStackTrace()`는 Error를 상속하지 않는 객체에도 스택을 붙일 수 있습니다.
하지만 실무에서는 `Error`를 상속한 커스텀 에러에서도 생성자나 팩토리 프레임을 숨기기 위해 자주 사용합니다.
즉 "Error를 상속하지 않을 때만 쓰는 API"로 볼 필요는 없습니다.

다만 특별한 이유가 없다면 커스텀 에러는 `Error`를 상속하는 편이 좋습니다.
`instanceof Error`, `message`, `cause`, 일반 로거와의 호환성이 더 자연스럽기 때문입니다.

### H3. 스택을 파싱해 업무 로직을 만들면 안 된다

스택 문자열은 디버깅용 표현입니다.
파일 경로, 줄 번호, 함수명 형식은 런타임과 빌드 도구, sourcemap 설정에 따라 달라질 수 있습니다.
따라서 스택 문자열을 파싱해 사용자에게 보여 줄 코드나 비즈니스 분기를 만들면 깨지기 쉽습니다.

업무 분기는 `code`, `statusCode`, `retryable`, `category` 같은 명시적 필드로 처리해야 합니다.
스택은 사람이 장애를 분석하기 위한 보조 자료로 남기는 것이 맞습니다.

### H3. 모든 에러에 details를 무제한으로 붙이면 로그가 위험해진다

에러 객체에 많은 정보를 붙이고 싶은 유혹이 있습니다.
하지만 요청 body 전체, 외부 API 응답 전체, 사용자 프로필 전체를 `details`에 넣으면 민감정보와 로그 비용 문제가 생깁니다.

좋은 에러 details는 작고, 재현 가능하고, 마스킹되어 있습니다.
예를 들어 `fieldNames`, `limit`, `receivedCount`, `providerCode`, `retryAfterMs`처럼 원인을 좁히는 값만 남깁니다.
원본 payload가 필요하다면 접근 통제된 별도 저장소와 보존 정책을 먼저 검토해야 합니다.

## 실무 체크리스트

### H3. 커스텀 에러 클래스에 넣을 것

- `name`: 로그와 스택 첫 줄에서 에러 종류를 구분한다.
- `code`: 모니터링과 클라이언트 분기에 사용할 안정적인 문자열이다.
- `statusCode`: HTTP 계층에서 응답 상태를 결정한다.
- `cause`: 원본 에러를 보존한다.
- `captureStackTrace`: 생성자나 팩토리 프레임을 숨겨 스택을 읽기 쉽게 만든다.
- `details`: 허용 목록 기반의 안전한 진단 정보만 담는다.

### H3. 운영 로그 전에 확인할 것

- 스택 트레이스를 외부 응답에 그대로 노출하지 않는다.
- `details`에 토큰, 비밀번호, 개인정보, 결제 정보가 들어가지 않는다.
- `cause` 체인을 직렬화할 때 깊이와 필드를 제한한다.
- 스택 전체 문자열을 테스트 snapshot으로 고정하지 않는다.
- 브라우저 공유 코드라면 `Error.captureStackTrace` 존재 여부를 확인한다.

## 마무리

`Error.captureStackTrace()`는 작은 API지만 Node.js 에러 로그의 가독성을 크게 바꿀 수 있습니다.
커스텀 에러의 생성자와 팩토리 프레임을 숨기면 실제 실패 지점이 더 빨리 보이고, `code`, `statusCode`, `cause`를 함께 설계하면 응답 계약과 내부 진단을 분리할 수 있습니다.

핵심은 스택 트레이스를 "외부 메시지"가 아니라 "내부 진단 정보"로 다루는 것입니다.
사용자에게는 안정적인 에러 코드와 안전한 메시지를 주고, 운영자에게는 깔끔한 스택과 원인을 남기면 장애 대응 속도와 보안 기준을 함께 지킬 수 있습니다.

## FAQ

### H3. Error.captureStackTrace는 표준 JavaScript API인가요?

아닙니다.
MDN은 `Error.captureStackTrace()`를 비표준 API로 분류합니다.
Node.js처럼 V8 기반 런타임에서는 사용할 수 있지만, 여러 런타임에서 공유하는 라이브러리라면 `Error.captureStackTrace?.(...)`처럼 존재 여부를 확인하는 편이 안전합니다.

### H3. Error.captureStackTrace를 쓰면 성능 비용이 있나요?

스택 트레이스를 캡처하는 작업에는 비용이 있습니다.
일반적인 예외 경로에서는 문제가 되지 않는 경우가 많지만, 정상 제어 흐름에서 대량으로 에러 객체를 만들거나 반복 루프 안에서 스택을 계속 캡처하는 방식은 피해야 합니다.
에러는 예외적 상황과 진단이 필요한 실패를 표현하는 데 쓰는 편이 좋습니다.

### H3. 커스텀 에러마다 별도 클래스를 만들어야 하나요?

항상 그럴 필요는 없습니다.
도메인별로 `ValidationError`, `AuthError`, `ExternalServiceError`처럼 의미가 분명한 클래스는 도움이 됩니다.
반대로 너무 많은 클래스를 만들면 관리가 어려워질 수 있으므로, 안정적인 `code`와 공통 `AppError`만으로 충분한지 먼저 검토하는 편이 좋습니다.

## 함께 보면 좋은 글

- [Node.js Error cause 가이드: 감싼 에러의 원인을 잃지 않는 디버깅 패턴](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)
- [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)
- [로그 예제 sanitization 가이드: 신뢰 가능한 개발 글을 위한 민감정보 제거](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)

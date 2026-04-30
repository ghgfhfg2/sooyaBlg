---
layout: post
title: "Node.js Error.cause 가이드: 래핑된 에러의 원인을 잃지 않고 디버깅하는 법"
date: 2026-05-01 08:00:00 +0900
lang: ko
translation_key: nodejs-error-cause-wrapped-errors-debugging-guide
permalink: /development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html
alternates:
  ko: /development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html
  x_default: /development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, error, debugging, observability, javascript, backend]
description: "Node.js에서 Error.cause를 사용해 상위 에러에 맥락을 추가하면서도 원본 실패 원인을 보존하는 방법, 실무 패턴, 로깅 주의사항을 예제와 함께 정리했습니다."
---

Node.js 서비스에서 장애를 추적하다 보면 이런 상황이 자주 나옵니다.
상위 계층은 “결제 처리 실패”, “파일 업로드 실패”, “사용자 조회 실패”처럼 비즈니스 맥락을 잘 설명하는데, 정작 로그를 열어보면 **원래 어떤 예외가 시작점이었는지 사라져 있는 경우**입니다.

이럴 때 핵심은 **에러를 새로 던지더라도 `Error.cause`로 원본 원인을 함께 보존하는 것**입니다.
결론부터 말하면 `Error.cause`를 쓰면 서비스 레이어별로 의미 있는 에러 메시지를 유지하면서도, 실제 장애 원인까지 끊기지 않게 연결할 수 있습니다.

## 왜 Error.cause가 중요한가

### H3. 상위 메시지만 남기면 디버깅 정보가 끊긴다

실무 코드에서는 원본 예외를 그대로 노출할 수 없어서 보통 더 읽기 쉬운 에러로 감싸게 됩니다.
문제는 이 과정에서 원본 스택과 실패 맥락이 끊기기 쉽다는 점입니다.

예를 들어 아래 코드는 의도는 좋지만 디버깅에는 약합니다.

```js
try {
  await savePayment(payment);
} catch (error) {
  throw new Error('결제 저장 실패');
}
```

이렇게 작성하면 로그에는 “결제 저장 실패”만 남고, 실제 원인이 DB timeout인지, validation 오류인지, 네트워크 문제인지 분간하기 어려워집니다.

### H3. 원본 에러와 도메인 맥락을 같이 가져가야 운영이 쉬워진다

운영에서는 보통 두 가지가 모두 필요합니다.

- 사람에게 읽히는 상위 맥락
- 실제 실패를 만든 하위 원인

`Error.cause`는 바로 이 두 가지를 한 객체 체인으로 연결해 줍니다.
즉 “무엇이 실패했는가”와 “왜 실패했는가”를 동시에 남길 수 있습니다.

## Node.js Error.cause 기본 사용법

### H3. 새 Error를 만들 때 cause 옵션으로 원본 예외를 넣는다

가장 기본 패턴은 아래와 같습니다.

```js
try {
  await savePayment(payment);
} catch (error) {
  throw new Error('결제 저장 실패', { cause: error });
}
```

이 패턴의 장점은 단순합니다.

- 상위 계층 메시지를 더 명확하게 만들 수 있음
- 원본 예외 객체를 버리지 않음
- 로깅 시 연쇄 원인을 따라가기 쉬움
- API 에러 변환, 배치 실패 처리, 백그라운드 작업 추적에 모두 재사용 가능

특히 비동기 경로에서 에러가 누락되지 않게 관리하려면 [Node.js EventEmitter captureRejections 가이드](/development/blog/seo/2026/04/25/nodejs-eventemitter-capturerejections-unhandled-async-errors-guide.html)에서 다룬 패턴과 함께 보는 편이 좋습니다.

### H3. 여러 계층에서 단계별 맥락을 추가할 수 있다

`cause`의 진짜 장점은 한 번만 쓰는 데 있지 않습니다.
계층이 올라갈 때마다 필요한 맥락을 더할 수 있습니다.

```js
async function loadUser(userId) {
  try {
    return await repository.findUserById(userId);
  } catch (error) {
    throw new Error(`사용자 조회 실패: ${userId}`, { cause: error });
  }
}

async function buildDashboard(userId) {
  try {
    const user = await loadUser(userId);
    return createDashboard(user);
  } catch (error) {
    throw new Error('대시보드 생성 실패', { cause: error });
  }
}
```

이렇게 하면 최상위에서는 “대시보드 생성 실패”를 보되, 필요하면 내부 원인 체인을 따라가며 “사용자 조회 실패”, 그 아래의 실제 DB 오류까지 확인할 수 있습니다.

## 어떤 상황에서 특히 유용한가

### H3. 외부 API, DB, 파일 시스템 예외를 도메인 에러로 바꿀 때 좋다

서비스 코드는 인프라 에러를 그대로 노출하기보다 도메인 언어로 바꾸는 경우가 많습니다.
이때 `cause`가 가장 자연스럽습니다.

```js
try {
  const response = await fetch(endpoint, { signal });

  if (!response.ok) {
    throw new Error(`upstream returned ${response.status}`);
  }
} catch (error) {
  throw new Error('사용자 프로필 동기화 실패', { cause: error });
}
```

이 패턴은 timeout, 취소, 재시도 같은 제어 흐름과도 잘 맞습니다.
상위 작업이 취소 신호를 어떻게 다루는지는 [Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)와 함께 보면 더 깔끔하게 정리할 수 있습니다.

### H3. 배치 작업에서 실패 요약과 근본 원인을 함께 남길 수 있다

배치나 큐 워커는 실패 건수를 요약해야 하지만, 동시에 각 건의 실제 실패 이유도 잃으면 안 됩니다.

```js
try {
  await processInvoiceJob(job);
} catch (error) {
  throw new Error(`invoice job failed: ${job.id}`, { cause: error });
}
```

이 방식이면 운영 대시보드에는 job 단위 문맥을 남기고, 상세 로그에서는 실제 SQL 에러나 파일 파싱 예외까지 추적할 수 있습니다.

## 로깅할 때 주의할 점

### H3. error.message만 찍으면 cause 체인이 보이지 않을 수 있다

`Error.cause`를 썼더라도 로거가 `message`만 출력하면 효과가 반쯤 사라집니다.
그래서 아래처럼 전체 에러 객체나 스택을 함께 남기는 습관이 중요합니다.

```js
logger.error({ err: error }, 'request failed');
```

혹은 최소한 아래처럼 원인 체인을 펼쳐서 기록해야 합니다.

```js
function getErrorChain(error) {
  const chain = [];
  let current = error;

  while (current) {
    chain.push({
      name: current.name,
      message: current.message,
    });
    current = current.cause;
  }

  return chain;
}
```

관측성 체계를 정리하고 있다면 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)처럼 에러 이벤트와 메타데이터를 함께 설계하는 편이 운영에서 훨씬 유리합니다.

### H3. 사용자 응답에는 내부 cause를 그대로 노출하지 않는 편이 안전하다

`cause`에 들어 있는 원본 에러는 내부 경로, SQL, upstream 응답, 파일 위치 같은 운영 정보가 담길 수 있습니다.
그래서 API 응답에는 보통 안전한 상위 메시지만 내보내고, 상세 cause는 서버 로그에서만 보는 편이 맞습니다.

즉 권장 분리는 이렇습니다.

- 사용자 응답: 일반화된 실패 메시지
- 내부 로그: 전체 에러 객체 + cause 체인
- 알림/모니터링: 서비스 문맥 + 핵심 원인 요약

## 커스텀 에러 클래스와 함께 쓰는 패턴

### H3. 상태 코드나 에러 코드를 유지하면서 cause를 넣을 수 있다

실무에서는 커스텀 에러 클래스를 쓰는 경우가 많습니다.
그럴 때도 `cause`를 그대로 활용할 수 있습니다.

```js
class AppError extends Error {
  constructor(message, { code, status, cause } = {}) {
    super(message, { cause });
    this.name = 'AppError';
    this.code = code;
    this.status = status;
  }
}

try {
  await repository.insertUser(input);
} catch (error) {
  throw new AppError('사용자 생성 실패', {
    code: 'USER_CREATE_FAILED',
    status: 500,
    cause: error,
  });
}
```

이 구조는 API 계층에서 특히 편합니다.
에러 코드와 HTTP 상태는 유지하면서, 내부 원인까지 잃지 않을 수 있기 때문입니다.

### H3. 너무 많은 래핑은 오히려 가독성을 해칠 수 있다

다만 모든 계층에서 기계적으로 래핑하는 것은 권장하지 않습니다.
의미가 달라지지 않는 지점까지 계속 감싸면 체인만 길어지고 신호 대 잡음비가 나빠집니다.

아래 기준이 실용적입니다.

- 도메인 맥락이 새로 생길 때만 래핑
- 같은 의미의 pass-through 계층은 그대로 rethrow
- 사용자에게 보여 줄 메시지 경계에서 래핑
- 로깅 기준점에서 한 번 더 문맥을 추가

## 실무 체크리스트

### H3. Error.cause를 도입할 때 최소한 이것부터 맞춘다

팀 코드베이스에 적용할 때는 아래 네 가지를 먼저 맞추면 효과가 큽니다.

1. 원본 예외를 새 `Error`로 덮어쓸 때는 `cause`를 기본값으로 사용하기
2. 로거가 `err` 객체 전체를 남기도록 설정하기
3. API 응답에는 내부 cause를 그대로 노출하지 않기
4. 커스텀 에러 클래스에서도 `super(message, { cause })`를 지원하기

이 정도만 정리해도 “메시지는 친절해졌는데 원인 추적은 더 어려워진다”는 흔한 문제를 꽤 줄일 수 있습니다.

## 마무리

Node.js의 `Error.cause`는 작은 문법 추가처럼 보여도, 실제 운영에서는 **맥락 있는 에러 메시지와 근본 원인 추적을 동시에 가능하게 해 주는 도구**에 가깝습니다.

특히 서비스 계층이 많은 코드베이스일수록 “무슨 작업이 실패했는지”와 “진짜 무엇이 문제였는지”를 분리해서 보존하는 것이 중요합니다.
새 에러를 던져야 할 때 원본 예외를 버리지 말고 `cause`로 이어 두면, 장애 대응 속도와 로그 품질이 눈에 띄게 좋아집니다.

## 함께 보면 좋은 글

- [Node.js EventEmitter captureRejections 가이드: 비동기 리스너 에러를 놓치지 않는 법](/development/blog/seo/2026/04/25/nodejs-eventemitter-capturerejections-unhandled-async-errors-guide.html)
- [Node.js AbortSignal.any 가이드: timeout과 사용자 취소를 함께 처리하는 법](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js diagnostics_channel 가이드: 관측성 이벤트를 낮은 결합도로 수집하는 법](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)

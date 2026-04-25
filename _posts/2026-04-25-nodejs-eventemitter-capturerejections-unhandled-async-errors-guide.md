---
layout: post
title: "Node.js EventEmitter captureRejections 가이드: async 리스너 에러를 놓치지 않는 법"
date: 2026-04-25 20:00:00 +0900
lang: ko
translation_key: nodejs-eventemitter-capturerejections-unhandled-async-errors-guide
permalink: /development/blog/seo/2026/04/25/nodejs-eventemitter-capturerejections-unhandled-async-errors-guide.html
alternates:
  ko: /development/blog/seo/2026/04/25/nodejs-eventemitter-capturerejections-unhandled-async-errors-guide.html
  x_default: /development/blog/seo/2026/04/25/nodejs-eventemitter-capturerejections-unhandled-async-errors-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, eventemitter, captureRejections, async, error-handling, backend, javascript]
description: "Node.js EventEmitter의 captureRejections 옵션을 사용해 async 리스너에서 새는 에러를 줄이는 방법, 동작 방식, 실수하기 쉬운 지점을 예제와 함께 정리했습니다."
---

Node.js에서 `EventEmitter`는 익숙하지만, **async 리스너에서 발생한 예외가 예상과 다르게 흘러가는 문제**는 꽤 자주 놓칩니다.
특히 이벤트 핸들러를 `async` 함수로 바꾼 뒤 `await` 이후 예외가 발생하면, 기존의 `try/catch`나 `error` 이벤트 처리만으로는 충분하지 않은 경우가 있습니다.

이 글은 검색 의도가 분명한 질문, 즉 **"Node.js EventEmitter에서 async 리스너 에러를 어떻게 안전하게 처리할까?"**에 답하는 글입니다.
결론부터 말하면 `captureRejections`는 `async` 리스너의 rejected promise를 `error` 흐름으로 연결해, 누락되기 쉬운 예외를 더 일관되게 다루도록 돕는 옵션입니다.

## Node.js EventEmitter에서 async 에러가 헷갈리는 이유

### H3. 동기 리스너의 throw와 async 리스너의 rejection은 같은 것처럼 보여도 다르게 흐른다

동기 리스너는 보통 `throw`가 즉시 현재 호출 흐름으로 전파됩니다.
반면 `async` 리스너는 내부적으로 promise를 반환하므로, `await` 이후 발생한 에러는 rejection으로 처리됩니다.
이 차이 때문에 아래처럼 코드를 읽고도 실제 런타임 동작을 잘못 예상하는 경우가 많습니다.

```js
const { EventEmitter } = require('node:events');

const bus = new EventEmitter();

bus.on('job:done', async () => {
  await Promise.resolve();
  throw new Error('listener failed');
});

bus.emit('job:done');
```

겉으로 보면 단순하지만, 핵심은 `emit()` 호출부가 이 rejection을 직접 잡아주지 않는다는 점입니다.
그래서 상황에 따라 unhandled rejection처럼 보이거나, 기대한 `error` 이벤트 흐름으로 이어지지 않아 장애 분석이 늦어질 수 있습니다.

### H3. 이벤트 기반 구조일수록 예외 누락이 운영 리스크로 이어진다

실무에서는 이벤트를 아래처럼 많이 씁니다.

- 작업 완료 후 후행 처리 실행
- 캐시 무효화 트리거
- 알림 발송 시작
- 메트릭/로그 기록
- 비즈니스 도메인 이벤트 전달

문제는 이벤트 핸들러가 늘어날수록 “어느 리스너에서 실패했는지”가 흐려지기 쉽다는 점입니다.
이때 async 리스너 에러까지 제각각 처리하면 운영 로그가 지저분해지고 재현도 어려워집니다.
이런 맥락에서는 [Node.js unhandledRejection / uncaughtException 가이드](/development/blog/seo/2026/04/15/nodejs-unhandledrejection-uncaughtexception-shutdown-guide.html)처럼 프로세스 단위 예외 전략과 함께 봐야 전체 그림이 정리됩니다.

## captureRejections는 무엇을 해결하나

### H3. async 리스너의 rejected promise를 error 이벤트 흐름에 연결한다

`EventEmitter`를 만들 때 `captureRejections: true`를 주면, async 리스너가 반환한 promise가 reject될 때 이를 `error` 이벤트 쪽으로 넘길 수 있습니다.
예시는 아래와 같습니다.

```js
const { EventEmitter } = require('node:events');

const bus = new EventEmitter({ captureRejections: true });

bus.on('job:done', async (payload) => {
  await Promise.resolve();
  throw new Error(`job failed: ${payload.jobId}`);
});

bus.on('error', (err) => {
  console.error('captured error:', err.message);
});

bus.emit('job:done', { jobId: 'A-101' });
```

이 구조의 장점은 분명합니다.
`async` 리스너 실패를 제각각 `try/catch`로 감싸지 않아도, 최소한 공통 에러 관문을 만들 수 있습니다.
특히 여러 이벤트 리스너를 운영하는 서비스에서 **누락 방지용 안전망**으로 유용합니다.

### H3. 모든 에러 처리를 대체하는 만능 기능은 아니다

여기서 중요한 오해를 먼저 막아야 합니다.
`captureRejections`는 rejected promise를 더 잘 포착하게 도와주지만, 그 자체가 완전한 장애 복구 전략은 아닙니다.

예를 들어 아래까지 자동으로 해결해주지는 않습니다.

- 재시도 정책 설계
- 보상 트랜잭션 수행
- 실패 이벤트를 큐로 옮기는 작업
- 외부 API 장애에 대한 서킷 브레이커 처리

즉 `captureRejections`는 **예외 누락을 줄이는 메커니즘**이지, 실패 자체를 비즈니스적으로 복구하는 메커니즘은 아닙니다.

## 언제 특히 유용한가

### H3. 후행 작업 리스너가 많고, 공통 에러 로깅 지점이 필요할 때

한 이벤트에 여러 리스너가 붙는 구조라면, 각 리스너가 제각각 에러를 삼키지 않도록 통제점이 필요합니다.
예를 들어 주문 완료 이벤트 뒤에 아래 작업이 붙을 수 있습니다.

- CRM 동기화
- 분석 로그 적재
- 푸시 알림 발송
- 추천 캐시 갱신

이때 모든 리스너에 개별 `try/catch`만 의존하면 누락이 생기기 쉽습니다.
반면 `captureRejections`와 `error` 리스너를 함께 두면 최소한 “실패 사실을 중앙에서 감지”하는 구조를 만들 수 있습니다.

### H3. Promise 기반 코드로 전환 중인 레거시 이벤트 시스템에서 효과가 크다

기존에는 콜백 위주였던 이벤트 핸들러가 점점 `async/await`로 바뀌는 경우가 많습니다.
이 전환기에는 팀이 여전히 `throw`와 rejection의 차이를 섞어 이해하는 경우가 많아 사고가 납니다.
이럴 때 `captureRejections`를 켜면 동작 모델을 조금 더 단순하게 맞출 수 있습니다.

다만 이런 구조에서도 [Node.js Promise.allSettled 가이드](/development/blog/seo/2026/04/24/nodejs-promise-allsettled-partial-failure-handling-guide.html)처럼 “어떤 실패는 허용하고 어떤 실패는 승격할지”를 따로 정의하는 편이 좋습니다.
모든 비동기 실패를 같은 심각도로 보면 운영 노이즈가 커질 수 있기 때문입니다.

## 실무 적용 패턴

### H3. 공통 error 리스너에서 로깅·메트릭·알림 기준을 분리한다

실전에서는 `error` 이벤트를 받았다고 해서 무조건 같은 방식으로 처리하지 않는 편이 좋습니다.
예를 들면 아래처럼 나눌 수 있습니다.

- 운영 로그에는 항상 기록
- 메트릭은 이벤트 타입별 카운트만 증가
- 페이징 알림은 특정 중요 이벤트에만 발송
- 사용자 입력 오류는 경고 수준으로만 남김

```js
const { EventEmitter } = require('node:events');

const bus = new EventEmitter({ captureRejections: true });

bus.on('error', (err) => {
  logger.error('event listener failed', {
    message: err.message,
    name: err.name,
  });

  metrics.counter('event_listener_error_total').inc();
});
```

이 패턴은 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)와도 잘 어울립니다.
에러를 직접 여러 군데 흩뿌리기보다, 중앙 관문에서 관측 신호를 다시 발행하는 방식이 코드 오염을 줄이는 데 도움이 됩니다.

### H3. 그래도 리스너 내부에서 맥락을 보강하는 것은 필요하다

`captureRejections`만 믿고 리스너 내부 정보를 전혀 남기지 않으면, 나중에 “어떤 payload에서 왜 실패했는지” 찾기 어려울 수 있습니다.
그래서 아래처럼 맥락을 보강한 에러를 던지는 편이 좋습니다.

```js
bus.on('invoice:publish', async (payload) => {
  try {
    await publishInvoice(payload.invoiceId);
  } catch (err) {
    err.message = `invoice publish failed: invoiceId=${payload.invoiceId} / ${err.message}`;
    throw err;
  }
});
```

단, 여기서도 원문 payload 전체를 던지거나 로그로 남기지는 않는 편이 안전합니다.
민감한 필드가 섞이면 운영 로그가 곧 리스크가 됩니다.

## 자주 하는 실수

### H3. error 리스너 없이 captureRejections만 켜두는 경우

이건 꽤 위험합니다.
`captureRejections`를 켰더라도 `error` 이벤트를 받아줄 리스너가 없다면, 결국 더 큰 런타임 문제로 번질 수 있습니다.
즉 옵션만 켜고 끝내면 안 되고, **반드시 에러를 소비하는 기준점**을 같이 둬야 합니다.

### H3. 비즈니스 필수 흐름을 EventEmitter 에러 처리에만 의존하는 경우

핵심 정합성이 필요한 작업이라면 이벤트 리스너 실패를 단순 로깅으로만 끝내면 안 됩니다.
예를 들어 결제 확정, 재고 차감, 정산 기록 같은 흐름은 명시적 트랜잭션이나 작업 큐, 재시도 정책이 더 적절합니다.

안전한 기준은 이렇습니다.

- 핵심 정합성: 서비스 로직, 트랜잭션, 큐, 명시적 재시도
- 후행 처리/관측: EventEmitter + captureRejections

### H3. 민감한 객체를 통째로 에러 메시지에 붙이는 경우

실패 분석이 급하다고 아래 데이터를 그대로 로그에 싣는 경우가 있습니다.

- 액세스 토큰
- 사용자 이메일, 전화번호, 주소
- 결제 응답 원문
- 내부 시스템 식별자 전체

하지만 이벤트 시스템의 에러 로그는 수집·전송·보관 범위가 넓습니다.
따라서 식별자 일부, 이벤트 타입, 실패 코드 정도로 최소화하는 편이 안전합니다.

## 도입 전 체크리스트

### H3. 아래 항목에 많이 해당하면 captureRejections를 검토할 만하다

- EventEmitter 리스너를 `async` 함수로 자주 작성하는가?
- 리스너 실패가 누락돼 장애 파악이 늦어진 적이 있는가?
- 공통 `error` 리스너에서 운영 로깅을 일관되게 하고 싶은가?
- 레거시 이벤트 코드를 promise 기반으로 옮기는 중인가?
- 핵심 정합성보다는 후행 처리 안정성이 더 중요한 이벤트가 많은가?

## 마무리

Node.js `EventEmitter`의 `captureRejections`는 화려한 기능은 아니지만, **async 리스너 예외가 조용히 새는 문제를 줄이는 데 꽤 실용적인 기본기**입니다.
특히 이벤트 기반 후행 작업이 많은 서비스라면, 이 옵션 하나만으로도 장애 탐지 속도와 코드 일관성이 좋아질 수 있습니다.

다만 이것을 만능 에러 처리로 생각하면 곤란합니다.
핵심 비즈니스 정합성은 별도 메커니즘으로 지키고, `captureRejections`는 리스너 실패를 더 잘 드러내는 안전망으로 쓰는 편이 가장 현실적입니다.

함께 보면 좋은 글:

- [Node.js unhandledRejection / uncaughtException 가이드](/development/blog/seo/2026/04/15/nodejs-unhandledrejection-uncaughtexception-shutdown-guide.html)
- [Node.js Promise.allSettled 가이드](/development/blog/seo/2026/04/24/nodejs-promise-allsettled-partial-failure-handling-guide.html)
- [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)

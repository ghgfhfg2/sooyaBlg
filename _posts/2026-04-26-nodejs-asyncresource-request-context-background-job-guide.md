---
layout: post
title: "Node.js AsyncResource 가이드: 백그라운드 작업에서도 요청 컨텍스트를 이어가는 법"
date: 2026-04-26 20:00:00 +0900
lang: ko
translation_key: nodejs-asyncresource-request-context-background-job-guide
permalink: /development/blog/seo/2026/04/26/nodejs-asyncresource-request-context-background-job-guide.html
alternates:
  ko: /development/blog/seo/2026/04/26/nodejs-asyncresource-request-context-background-job-guide.html
  x_default: /development/blog/seo/2026/04/26/nodejs-asyncresource-request-context-background-job-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, AsyncResource, AsyncLocalStorage, observability, tracing, backend, background-jobs]
description: "Node.js AsyncResource로 큐·타이머·커스텀 비동기 작업에서도 요청 컨텍스트를 유지하는 방법, AsyncLocalStorage와 함께 쓰는 패턴과 주의사항을 실무 예제로 정리했습니다."
---

Node.js에서 요청 단위 로그를 잘 남기고 있었는데, 큐 작업이나 커스텀 비동기 경계로 넘어가는 순간 `requestId`가 갑자기 사라지는 일이 자주 생깁니다.
특히 `AsyncLocalStorage`를 도입한 뒤에도 일부 백그라운드 작업에서 컨텍스트가 끊기면, 운영 중 장애를 추적할 때 로그가 서로 연결되지 않아 꽤 답답해집니다.

이럴 때 검토할 수 있는 도구가 `AsyncResource`입니다.
결론부터 말하면 `AsyncResource`는 **직접 만든 비동기 경계에 “이 작업도 같은 흐름이다”라는 힌트를 주는 도구**에 가깝습니다.
`AsyncLocalStorage`만으로 충분한 경우도 많지만, 커스텀 큐·이벤트 브리지·콜백 래퍼처럼 런타임이 자동으로 컨텍스트를 이어주지 않는 지점에서는 `AsyncResource`가 실무적으로 꽤 유용합니다.

## 왜 요청 컨텍스트가 중간에 끊어질까

### H3. 프레임워크 바깥으로 나가는 순간 추적이 흐려지기 쉽다

HTTP 요청 안에서는 `AsyncLocalStorage`가 비교적 잘 동작합니다.
하지만 아래처럼 “한 번 더 감싼” 경계가 생기면 문맥이 끊기는 경우가 있습니다.

- 직접 구현한 작업 큐
- 이벤트 에미터를 감싼 커스텀 래퍼
- 콜백을 나중에 실행하는 스케줄러
- 외부 라이브러리와 내부 로깅 규약을 연결하는 브리지

이런 구간에서는 "누가 이 작업을 시작했는가"가 사라지기 쉽습니다.
결국 로그에는 에러 메시지가 남아도, 어떤 요청에서 시작된 작업인지 연결이 안 됩니다.

### H3. AsyncLocalStorage만 써도 되는 상황과 아닌 상황이 있다

많은 경우 `AsyncLocalStorage.run()`이나 `enterWith()`만으로 충분합니다.
문제는 비동기 작업 생성 시점과 실행 시점 사이에 프레임워크가 보장하지 않는 커스텀 경계가 끼어드는 경우입니다.

예를 들어 아래처럼 간단한 인메모리 큐를 생각해볼 수 있습니다.

```js
const { AsyncLocalStorage } = require('node:async_hooks');

const als = new AsyncLocalStorage();
const queue = [];

function enqueue(job) {
  queue.push(job);
}

function processNext() {
  const job = queue.shift();
  if (!job) return;

  setTimeout(() => {
    console.log('requestId:', als.getStore()?.requestId);
    job();
  }, 10);
}
```

겉으로는 단순하지만, 실제 구조가 복잡해질수록 컨텍스트가 기대대로 이어지지 않는 지점이 생길 수 있습니다.
이때 어디서 흐름이 끊기는지 명시적으로 표현해야 합니다.

## AsyncResource는 무엇을 해 주는가

### H3. 커스텀 비동기 작업에 실행 컨텍스트를 연결해 준다

`AsyncResource`는 `node:async_hooks` 모듈에서 제공됩니다.
핵심은 `runInAsyncScope()`입니다.
이 메서드로 콜백을 실행하면, 해당 콜백이 특정 비동기 리소스 문맥 안에서 실행되도록 연결할 수 있습니다.

```js
const { AsyncLocalStorage, AsyncResource } = require('node:async_hooks');

const als = new AsyncLocalStorage();

class JobResource extends AsyncResource {
  constructor() {
    super('job-resource');
  }

  run(fn) {
    return this.runInAsyncScope(fn);
  }
}
```

즉 `AsyncResource`는 “이 콜백은 그냥 나중에 호출되는 함수”가 아니라, **특정 비동기 작업의 일부**라고 런타임에 알려주는 역할을 합니다.

### H3. 로그 상관관계와 트레이싱 연결이 쉬워진다

운영 관점에서 가장 큰 장점은 상관관계 유지입니다.
아래처럼 요청에서 만든 문맥을 큐 처리 시점까지 이어가면, 로그·메트릭·추적 데이터를 더 자연스럽게 묶을 수 있습니다.

```js
const { AsyncLocalStorage, AsyncResource } = require('node:async_hooks');

const als = new AsyncLocalStorage();
const queue = [];

class JobResource extends AsyncResource {
  constructor(store) {
    super('job-resource');
    this.store = store;
  }

  run(job) {
    return als.run(this.store, () => this.runInAsyncScope(job));
  }
}

function enqueue(task) {
  const store = als.getStore();
  queue.push({
    task,
    resource: new JobResource(store),
  });
}

function processNext() {
  const item = queue.shift();
  if (!item) return;

  setTimeout(() => {
    item.resource.run(() => {
      console.log('requestId:', als.getStore()?.requestId);
      item.task();
      item.resource.emitDestroy();
    });
  }, 10);
}
```

이 패턴의 핵심은 작업을 넣는 시점의 store를 저장해 두고, 실행 시점에 같은 문맥으로 복원하는 것입니다.
그 결과 백그라운드 작업 안에서도 `requestId`, `tenantId`, `traceId` 같은 값을 잃지 않기 쉬워집니다.

## 언제 AsyncResource를 고려하면 좋은가

### H3. 인메모리 큐·스케줄러·이벤트 브리지처럼 직접 경계를 만들 때

실무에서 `AsyncResource`가 특히 빛나는 구간은 아래와 같습니다.

- 자체 구현한 작업 큐
- 이벤트를 다른 레이어로 재전달하는 브리지
- 콜백을 저장했다가 나중에 실행하는 스케줄러
- 추상화 레이어가 두꺼운 SDK 래퍼

이런 경우에는 런타임이 자동으로 문맥을 이어주리라 기대하기보다, 아예 경계를 명시하는 편이 안전합니다.
특히 요청 기반 로깅은 [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)와 같이 설계해 두었더라도, 커스텀 작업 실행기에서 마지막 한 끗이 빠지면 효과가 반감됩니다.

### H3. observability 데이터를 한 요청 단위로 묶고 싶을 때

에러가 발생했을 때 단순 메시지만 보는 것과, 같은 `requestId` 아래에서 로그·메트릭·후속 작업 이력을 함께 보는 것은 차이가 큽니다.
`AsyncResource`는 이런 운영 가시성 품질을 올리는 데 도움이 됩니다.

이미 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)처럼 이벤트 계측을 하고 있다면, 컨텍스트가 끊기지 않도록 보강하는 용도로 잘 맞습니다.

## 구현할 때 자주 하는 실수

### H3. store 자체를 너무 무겁게 만드는 경우

컨텍스트를 유지하고 싶다고 해서 store 안에 큰 객체를 통째로 넣는 건 좋지 않습니다.
보통 아래 정도만 넣는 편이 현실적입니다.

- `requestId`
- `traceId`
- `tenantId`
- 사용자 식별에 필요한 최소 키

로그 상관관계에 필요한 최소한의 값만 담아야 메모리 부담과 의도치 않은 정보 노출을 줄일 수 있습니다.
민감한 payload 전체를 store에 넣는 방식은 피하는 편이 좋습니다.

### H3. 작업 종료 후 리소스 정리를 잊는 경우

`AsyncResource`를 만들었다면 수명 관리도 같이 봐야 합니다.
특히 오래 사는 큐나 반복 작업에서는 `emitDestroy()` 호출 시점을 정리해 두는 편이 좋습니다.

물론 모든 케이스에서 수동 정리가 절대적으로 필요한 것은 아니지만, 커스텀 리소스를 많이 만들수록 “언제 끝나는 작업인지”를 코드에서 분명하게 표현하는 것이 디버깅에 유리합니다.

### H3. AsyncResource를 만능 해결책처럼 쓰는 경우

문맥 전파가 안 된다고 해서 모든 비동기 지점에 `AsyncResource`를 덕지덕지 붙일 필요는 없습니다.
오히려 아래 순서가 더 낫습니다.

1. 기본 `AsyncLocalStorage`만으로 해결되는지 확인
2. 어떤 커스텀 경계에서 끊기는지 재현
3. 그 지점에만 `AsyncResource`를 적용

즉 `AsyncResource`는 기본값이라기보다 **정말 필요한 경계에 정밀하게 넣는 보강재**에 가깝습니다.

## 운영에서 확인하면 좋은 체크포인트

### H3. 로그 상관관계가 실제로 복원됐는지 먼저 검증한다

도입 후에는 단순히 에러가 안 난다는 이유로 끝내지 말고, 아래를 확인하는 편이 좋습니다.

- HTTP 요청 로그와 후속 큐 로그가 같은 `requestId`를 가지는가
- 실패 재시도 시에도 같은 상관관계 키 전략이 유지되는가
- 타이머·이벤트·워커 경계마다 값이 일관되게 보이는가

이 검증이 빠지면 “도입은 했는데 실제 운영에서는 안 이어지는” 상태가 남을 수 있습니다.

### H3. 성능보다 먼저 정확한 적용 지점을 찾는 게 중요하다

`AsyncResource` 자체를 과하게 두려워할 필요는 없지만, 불필요한 남용도 피해야 합니다.
핫패스 전체에 일괄 적용하기보다 컨텍스트 손실이 운영 문제로 이어지는 경계부터 우선 보강하는 편이 현실적입니다.
비동기 장애 처리까지 함께 점검하려면 [Node.js EventEmitter captureRejections 가이드](/development/blog/seo/2026/04/25/nodejs-eventemitter-capturerejections-unhandled-async-errors-guide.html)도 같이 보는 것이 좋습니다.

## 도입 전 체크리스트

### H3. 아래 항목에 해당하면 AsyncResource 검토 가치가 크다

- `AsyncLocalStorage`를 쓰는데 일부 백그라운드 작업에서 값이 사라진다
- 직접 만든 큐나 스케줄러가 있다
- 요청 단위 로그 상관관계가 운영에서 중요하다
- 추적이 끊겨 장애 분석 시간이 길어지고 있다
- 민감정보 대신 최소 식별 키만 컨텍스트에 담을 수 있다

## 마무리

Node.js `AsyncResource`는 자주 쓰는 API는 아니지만, 커스텀 비동기 경계에서 요청 문맥이 끊기는 문제를 다룰 때 꽤 강력한 도구입니다.
특히 `AsyncLocalStorage`를 이미 잘 쓰고 있는데도 일부 큐·콜백·브리지에서 상관관계가 무너진다면, 그 경계에 `AsyncResource`를 명시적으로 두는 편이 효과적입니다.

중요한 건 모든 곳에 적용하는 게 아니라, **문맥이 실제로 끊기는 지점을 찾아 최소한으로 연결하는 것**입니다.
그렇게 해야 로그 품질은 올리고 복잡도는 과하게 늘리지 않을 수 있습니다.

함께 보면 좋은 글:

- [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)
- [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)
- [Node.js EventEmitter captureRejections 가이드](/development/blog/seo/2026/04/25/nodejs-eventemitter-capturerejections-unhandled-async-errors-guide.html)

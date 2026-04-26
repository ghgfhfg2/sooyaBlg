---
layout: post
title: "Node.js Promise.withResolvers 가이드: deferred 패턴을 더 깔끔하게 다루는 법"
date: 2026-04-27 08:00:00 +0900
lang: ko
translation_key: nodejs-promise-withresolvers-deferred-pattern-guide
permalink: /development/blog/seo/2026/04/27/nodejs-promise-withresolvers-deferred-pattern-guide.html
alternates:
  ko: /development/blog/seo/2026/04/27/nodejs-promise-withresolvers-deferred-pattern-guide.html
  x_default: /development/blog/seo/2026/04/27/nodejs-promise-withresolvers-deferred-pattern-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, promise-withresolvers, deferred, javascript, async, backend, error-handling]
description: "Node.js Promise.withResolvers로 deferred 패턴을 더 읽기 쉽게 구현하는 방법, 기존 Promise 생성자 패턴과의 차이, 실무 사용 예시와 주의사항을 정리했습니다."
---

Node.js에서 비동기 흐름을 연결하다 보면 Promise를 지금 만들고, `resolve`나 `reject`는 나중에 호출해야 하는 순간이 꽤 자주 나옵니다.
예를 들어 이벤트가 올 때까지 기다리거나, 큐에서 다음 작업을 받을 때까지 대기하거나, 콜백 기반 API를 한 번 더 감싸야 하는 경우가 그렇습니다.

이때 흔히 쓰는 방식이 이른바 deferred 패턴입니다.
문제는 전통적인 `new Promise((resolve, reject) => { ... })` 패턴이 길어지기 시작하면 의도가 흐려지고, `resolve`와 `reject`를 바깥 변수에 빼는 코드가 금방 지저분해진다는 점입니다.

이럴 때 볼 만한 선택지가 `Promise.withResolvers()`입니다.
결론부터 말하면 `Promise.withResolvers()`는 **Promise 본체와 `resolve`·`reject` 함수를 한 번에 안전하게 꺼내는 표준 API**라서, deferred 패턴을 더 읽기 쉽게 정리할 때 꽤 유용합니다.
다만 아무 곳에나 남발하면 상태 관리가 분산돼 오히려 추적이 어려워질 수 있으니, “누가 끝낼 책임을 가지는지”가 분명한 경계에서 쓰는 편이 좋습니다.

## Promise.withResolvers가 필요한 이유

### H3. 전통적인 deferred 패턴은 금방 장황해진다

기존에는 아래처럼 Promise 바깥으로 `resolve`와 `reject`를 꺼내는 코드가 흔했습니다.

```js
let resolveNext;
let rejectNext;

const nextItemPromise = new Promise((resolve, reject) => {
  resolveNext = resolve;
  rejectNext = reject;
});
```

동작 자체는 단순하지만 실무에서는 몇 가지 불편이 생깁니다.

- 변수 선언과 Promise 생성이 분리돼 읽기 흐름이 끊김
- 초기화 누락 시 `undefined is not a function` 같은 실수 가능
- 타입 시스템이나 리뷰 관점에서 의도가 덜 명확함
- 여러 군데에서 `resolve`를 만지기 시작하면 책임 경계가 흐려짐

즉 문법 문제라기보다, **코드 의도를 드러내는 힘이 약하다**는 쪽이 더 큰 문제입니다.

### H3. Promise와 resolver를 한 번에 꺼내면 의도가 선명해진다

`Promise.withResolvers()`를 쓰면 같은 코드를 아래처럼 정리할 수 있습니다.

```js
const { promise, resolve, reject } = Promise.withResolvers();
```

이 한 줄만으로 “이 코드는 나중에 완료될 Promise를 하나 만들고, 완료 핸들을 같이 보관한다”는 의도가 바로 드러납니다.
특히 큐, 이벤트 브리지, 상태 전이 같은 코드에서는 가독성 차이가 꽤 큽니다.

## Promise.withResolvers는 어떻게 쓰는가

### H3. 가장 기본적인 형태는 간단한 deferred 객체다

아래 예시는 외부 이벤트가 올 때까지 기다렸다가 Promise를 완료하는 가장 단순한 구조입니다.

```js
function createDeferred() {
  return Promise.withResolvers();
}

const deferred = createDeferred();

setTimeout(() => {
  deferred.resolve({ ok: true, receivedAt: Date.now() });
}, 100);

const result = await deferred.promise;
console.log(result);
```

핵심은 `promise`, `resolve`, `reject`가 한 객체에 같이 묶여 나온다는 점입니다.
그래서 생성 지점과 완료 지점을 나눠야 하는 코드에서도 구조를 비교적 깔끔하게 유지할 수 있습니다.

### H3. 이벤트 한 번을 기다리는 래퍼를 읽기 쉽게 만들 수 있다

예를 들어 `EventEmitter`에서 특정 이벤트를 한 번만 기다리고 싶다면 아래처럼 감쌀 수 있습니다.

```js
import { EventEmitter } from 'node:events';

function onceData(emitter) {
  const deferred = Promise.withResolvers();

  const onData = (payload) => {
    cleanup();
    deferred.resolve(payload);
  };

  const onError = (error) => {
    cleanup();
    deferred.reject(error);
  };

  function cleanup() {
    emitter.off('data', onData);
    emitter.off('error', onError);
  }

  emitter.once('data', onData);
  emitter.once('error', onError);

  return deferred.promise;
}
```

이 패턴은 콜백 기반 이벤트를 Promise 기반 흐름으로 연결할 때 유용합니다.
다만 에러 이벤트와 cleanup을 같이 다루지 않으면 리스너 누수가 생길 수 있으니, `resolve`만 신경 쓰고 끝내면 위험합니다.
이 부분은 [Node.js EventEmitter captureRejections 가이드](/development/blog/seo/2026/04/25/nodejs-eventemitter-capturerejections-unhandled-async-errors-guide.html)와 함께 보면 더 감이 잘 옵니다.

## 실무에서 어디에 잘 맞는가

### H3. 큐 소비자나 워커 풀에서 “다음 아이템 대기”를 표현할 때

메모리 큐나 커스텀 채널을 구현할 때는 “지금은 비어 있지만, 다음 아이템이 오면 깨워라” 같은 흐름이 자주 나옵니다.
이때 `Promise.withResolvers()`는 꽤 자연스럽습니다.

```js
class AsyncQueue {
  constructor() {
    this.items = [];
    this.waiters = [];
  }

  push(item) {
    const waiter = this.waiters.shift();

    if (waiter) {
      waiter.resolve(item);
      return;
    }

    this.items.push(item);
  }

  async shift() {
    if (this.items.length > 0) {
      return this.items.shift();
    }

    const deferred = Promise.withResolvers();
    this.waiters.push(deferred);
    return deferred.promise;
  }
}
```

이 코드는 “빈 큐면 기다렸다가, 생산자가 들어오면 resolve한다”는 모델이 비교적 직관적으로 보입니다.
다만 waiter가 너무 많이 쌓이면 메모리와 지연 문제가 생길 수 있으니, 대기열 제한은 [Node.js Bounded Queue 가이드](/development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html)와 같이 설계하는 편이 안전합니다.

### H3. 콜백 API를 임시로 Promise로 감쌀 때

레거시 라이브러리를 다루다 보면 `util.promisify()`로 바로 바꾸기 애매한 경우가 있습니다.
예를 들어 성공 콜백과 실패 콜백이 따로 있거나, 완료 시점이 조금 특이한 API라면 직접 래핑해야 할 수 있습니다.

```js
function saveFileLegacy(client, input) {
  const deferred = Promise.withResolvers();

  client.save(
    input,
    (result) => deferred.resolve(result),
    (error) => deferred.reject(error),
  );

  return deferred.promise;
}
```

이런 코드는 마이그레이션 중간 단계에서 특히 실용적입니다.
다만 장기적으로는 래퍼가 계속 쌓이지 않게 인터페이스를 정리하는 편이 낫습니다.

## Promise.withResolvers를 쓸 때 주의할 점

### H3. “끝내지 못한 Promise”가 숨어들기 쉬운 구조다

이 API가 편한 만큼, `resolve`나 `reject`가 절대 호출되지 않는 버그도 만들기 쉬워집니다.
특히 아래 상황은 자주 문제 됩니다.

- 타임아웃 없이 영원히 대기함
- 예외 경로에서 `reject`가 빠짐
- cleanup 전에 참조가 사라져 waiter만 남음
- 종료 신호를 받았는데 대기 Promise를 정리하지 않음

그래서 실무에서는 대기 Promise에 취소나 타임아웃 경계를 같이 두는 편이 좋습니다.
이 원칙은 [Node.js AbortController 실전 가이드](/development/blog/seo/2026/03/19/nodejs-abortcontroller-timeout-cancellation-pattern-guide.html)에서 다룬 취소 표준화와도 자연스럽게 연결됩니다.

### H3. 상태 전이를 여러 군데서 만지면 오히려 디버깅이 어려워진다

`Promise.withResolvers()`는 강력하지만, 그만큼 바깥에서 상태를 끝낼 수 있는 권한도 커집니다.
예를 들어 하나의 deferred를 여러 모듈이 공유하면 아래 문제가 생길 수 있습니다.

- 누가 먼저 완료했는지 추적 어려움
- 중복 resolve/reject 시도가 숨어 있음
- 성공과 실패 책임 경계가 불명확함
- 테스트에서 타이밍 의존성이 커짐

그래서 저는 아래 기준을 권합니다.

- 생성한 모듈이 완료 책임도 같이 가진다
- 외부에는 가능하면 `promise`만 노출한다
- `resolve`·`reject`는 가장 좁은 범위에 둔다
- 타임아웃, abort, cleanup을 한 묶음으로 설계한다

즉 deferred 패턴을 더 예쁘게 만드는 도구이지, 상태 관리 복잡도를 없애 주는 도구는 아닙니다.

## 기존 Promise 생성자 패턴과 어떻게 구분하면 좋을까

### H3. 즉시 실행 로직이면 기존 Promise 생성자만으로 충분하다

Promise를 만드는 자리에서 바로 비동기 작업을 시작하고, 그 스코프 안에서 `resolve`와 `reject`를 끝낼 수 있다면 기존 방식도 충분히 명확합니다.

```js
import fs from 'node:fs';

const result = await new Promise((resolve, reject) => {
  fs.readFile('config.json', 'utf8', (error, data) => {
    if (error) {
      reject(error);
      return;
    }

    resolve(JSON.parse(data));
  });
});
```

이 경우에는 굳이 resolver를 바깥으로 뺄 이유가 없습니다.
오히려 기존 패턴이 생명주기를 더 좁게 묶어 줍니다.

### H3. 생성 시점과 완료 시점이 분리될 때 Promise.withResolvers가 빛난다

반대로 아래 조건이라면 `Promise.withResolvers()`가 더 잘 맞습니다.

- Promise는 지금 만들지만 완료는 나중에 일어남
- 이벤트, 큐, 상태머신처럼 외부 자극이 필요함
- resolver를 객체와 함께 보관해야 함
- 테스트에서 완료 시점을 명시적으로 제어하고 싶음

그리고 실제 도입 전에는 **운영 중인 Node.js 버전에서 `Promise.withResolvers()`를 지원하는지** 먼저 확인하는 편이 좋습니다.
런타임 버전이 섞여 있는 환경이라면 문법 자체보다 배포 호환성이 더 큰 이슈가 될 수 있습니다.

즉 핵심 판단 기준은 “Promise 생명주기가 한 스코프에 갇혀 있는가”입니다.
그게 아니라면 `Promise.withResolvers()`가 코드 의도를 더 잘 드러낼 가능성이 큽니다.

## 마무리

`Promise.withResolvers()`는 거창한 새 패턴이라기보다, Node.js에서 오래 쓰던 deferred 패턴을 **더 분명하고 덜 어색하게 표현하는 표준화된 문법**에 가깝습니다.
특히 이벤트 대기, 큐 소비, 콜백 브리지 같은 코드에서는 읽는 사람의 부담을 꽤 줄여 줍니다.

다만 resolver를 바깥으로 꺼낸다는 사실 자체는 여전히 같은 만큼, timeout·abort·cleanup 없이 쓰면 “영원히 끝나지 않는 Promise”를 더 쉽게 만들 수도 있습니다.
그래서 이 API는 편의성보다도 **책임 경계를 더 명확히 쓰기 위한 도구**로 보는 편이 실무적으로 맞습니다.

관련 글:

- [Node.js Promise.any 가이드: 여러 후보 중 가장 먼저 성공한 결과만 안전하게 쓰는 법](/development/blog/seo/2026/04/24/nodejs-promise-any-fallback-first-success-guide.html)
- [Node.js AbortController 실전 가이드: 타임아웃·취소 표준화로 장애 반경 줄이기](/development/blog/seo/2026/03/19/nodejs-abortcontroller-timeout-cancellation-pattern-guide.html)
- [Node.js EventEmitter captureRejections 가이드: 비동기 리스너 에러를 놓치지 않는 법](/development/blog/seo/2026/04/25/nodejs-eventemitter-capturerejections-unhandled-async-errors-guide.html)

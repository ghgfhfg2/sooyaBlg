---
layout: post
title: "Node.js Array.fromAsync 가이드: 비동기 이터러블을 배열로 안전하게 모으는 법"
date: 2026-05-05 20:00:00 +0900
lang: ko
translation_key: nodejs-array-fromasync-async-iterable-collection-guide
permalink: /development/blog/seo/2026/05/05/nodejs-array-fromasync-async-iterable-collection-guide.html
alternates:
  ko: /development/blog/seo/2026/05/05/nodejs-array-fromasync-async-iterable-collection-guide.html
  x_default: /development/blog/seo/2026/05/05/nodejs-array-fromasync-async-iterable-collection-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, async, iterable, stream, promise]
description: "Node.js에서 Array.fromAsync로 async iterable과 스트림 데이터를 배열로 수집하는 방법을 정리했습니다. Promise.all과의 차이, 실무 예제, 메모리 주의사항까지 함께 설명합니다."
---

API 페이지네이션 결과, 스트림 청크, 비동기 제너레이터처럼 **값이 한 번에 오지 않고 순서대로 도착하는 데이터**를 다룰 때가 많습니다.
이때 배열이 필요하다고 해서 직접 `for await...of` 루프를 돌며 `push()`를 반복하면, 코드가 금방 장황해지고 변환 로직도 중간에 흩어지기 쉽습니다.

`Array.fromAsync()`는 이런 흐름을 훨씬 단정하게 만들어 줍니다.
핵심은 **async iterable이나 promise 기반 입력을 "배열로 모으는 단계"를 표준 API 하나로 표현할 수 있다**는 점입니다.

## Array.fromAsync가 필요한 이유

### H3. 비동기 입력을 배열로 모으는 코드는 자주 반복된다

실무에서는 아래 같은 패턴이 자주 나옵니다.

- 페이지 단위로 읽어 오는 API 결과를 하나의 리스트로 합치기
- 스트림이나 async generator에서 받은 값을 후처리하기
- 비동기적으로 계산되는 값을 모아서 템플릿이나 응답에 넘기기

보통은 아래처럼 작성합니다.

```js
const results = [];

for await (const item of source) {
  results.push(item);
}
```

이 코드 자체는 나쁘지 않지만, 중간에 매핑·정규화·필터링이 들어가기 시작하면 수집 로직과 비즈니스 로직이 섞이기 쉽습니다.

### H3. Promise.all과는 해결하는 문제가 다르다

`Promise.all()`은 **이미 준비된 Promise 목록을 병렬로 기다리는 데** 강합니다.
반면 `Array.fromAsync()`는 **순차적으로 도착하는 입력을 소비하며 배열을 만드는 데** 더 잘 맞습니다.

예를 들어 async generator는 값이 필요할 때마다 다음 값을 만들어 냅니다.
이런 입력은 처음부터 완성된 배열이 아니기 때문에 `Promise.all()`보다 `Array.fromAsync()`가 더 자연스럽습니다.

부분 실패를 따로 모아야 하는 상황이라면 [Node.js Promise.allSettled 가이드: 부분 실패를 버리지 않고 처리하는 법](/development/blog/seo/2026/04/24/nodejs-promise-allsettled-partial-failure-handling-guide.html)도 함께 보는 편이 좋습니다.

## Array.fromAsync 기본 사용법

### H3. async iterable을 가장 짧게 배열로 바꾸기

```js
async function* getJobs() {
  yield { id: 1, status: 'queued' };
  yield { id: 2, status: 'running' };
  yield { id: 3, status: 'done' };
}

const jobs = await Array.fromAsync(getJobs());

console.log(jobs);
```

이 패턴의 장점은 분명합니다.

- 수집 의도가 코드에서 바로 보인다
- `push()`용 임시 배열을 직접 관리하지 않아도 된다
- 이후 변환 단계를 이어 붙이기 쉽다

### H3. mapFn으로 수집과 변환을 한 번에 처리할 수 있다

```js
async function* getUsers() {
  yield { id: 1, email: 'Admin@Example.com ' };
  yield { id: 2, email: 'Team@Example.com ' };
}

const emails = await Array.fromAsync(
  getUsers(),
  async (user) => user.email.trim().toLowerCase()
);

console.log(emails);
```

별도 루프를 열지 않아도 되기 때문에, **"무엇을 모으는지"와 "어떻게 바꾸는지"를 가까운 위치에서 함께 표현**할 수 있습니다.

입력값을 사전에 안전하게 정리해야 하는 경우에는 [Node.js RegExp.escape 가이드: 사용자 입력을 정규식에 안전하게 넣는 법](/development/blog/seo/2026/05/05/nodejs-regexp-escape-safe-dynamic-pattern-guide.html)처럼 변환 단계의 책임을 명확히 나누는 접근도 잘 맞습니다.

## 실무에서 자주 쓰는 패턴

### H3. 페이지네이션 API를 한 배열로 합치기

```js
async function* fetchAllProducts(fetchPage) {
  let page = 1;

  while (true) {
    const response = await fetchPage(page);

    for (const product of response.items) {
      yield product;
    }

    if (!response.nextPage) {
      return;
    }

    page = response.nextPage;
  }
}

const products = await Array.fromAsync(fetchAllProducts(fetchPage));
```

이 구조는 API 소비 코드에서 꽤 읽기 좋습니다.
페이지를 넘기는 책임은 generator가 맡고, 최종적으로 배열이 필요한 호출부는 `Array.fromAsync()` 한 줄로 끝낼 수 있기 때문입니다.

### H3. 스트림 데이터를 후처리 가능한 배열로 모으기

```js
import { Readable } from 'node:stream';

async function* lines() {
  yield 'alpha\n';
  yield 'beta\n';
  yield 'gamma\n';
}

const stream = Readable.from(lines());
const chunks = await Array.fromAsync(stream);
const text = chunks.join('');

console.log(text);
```

스트림을 웹 스트림 또는 다른 형태와 함께 다뤄야 한다면 [Node.js Readable.fromWeb/toWeb 가이드: Web Stream과 Node Stream을 안전하게 연결하는 법](/development/blog/seo/2026/04/30/nodejs-readable-fromweb-toweb-stream-bridge-guide.html)도 같이 참고할 만합니다.

### H3. 비동기 제너레이터에서 필요한 필드만 추출하기

```js
async function* readAuditLogs() {
  yield { id: 'a1', action: 'login', userId: 10 };
  yield { id: 'a2', action: 'download', userId: 11 };
  yield { id: 'a3', action: 'logout', userId: 10 };
}

const actions = await Array.fromAsync(
  readAuditLogs(),
  ({ action, userId }) => `${userId}:${action}`
);

console.log(actions);
```

후속 처리를 위해 전체 객체가 아니라 일부 필드만 배열로 빼고 싶을 때 특히 편합니다.

## Promise.all과 Array.fromAsync를 어떻게 구분할까

### H3. 이미 배열이 있고 병렬 대기가 목적이면 Promise.all이 더 맞다

```js
const tasks = urls.map((url) => fetch(url).then((res) => res.json()));
const results = await Promise.all(tasks);
```

이 경우에는 입력이 처음부터 배열이고, 각 작업을 최대한 동시에 처리하고 싶기 때문에 `Promise.all()`이 더 적합합니다.

### H3. 입력이 순차적으로 생산되면 Array.fromAsync가 더 자연스럽다

```js
async function* readInBatches() {
  for (let page = 1; page <= 3; page += 1) {
    yield fetchPage(page);
  }
}

const pages = await Array.fromAsync(readInBatches(), (promise) => promise);
```

여기서 중요한 점은 `Array.fromAsync()`가 **비동기 입력을 소비하며 배열을 만든다**는 것입니다.
즉, 시작점이 "배열"이 아니라 "비동기적으로 열리는 데이터 소스"일 때 더 읽기 좋은 선택이 됩니다.

## 사용할 때 주의할 점

### H3. 결국 전부 배열에 담기므로 메모리 사용량은 커질 수 있다

`Array.fromAsync()`는 편하지만, 이름 그대로 최종 결과를 배열에 모두 담습니다.
따라서 입력 데이터가 매우 크다면 아래를 먼저 점검해야 합니다.

- 정말 전체 결과를 메모리에 올려야 하는가?
- 중간 처리 후 바로 버릴 수는 없는가?
- 배치 단위 처리나 스트림 처리로 바꾸는 편이 낫지 않은가?

큰 입력을 다룰 때는 [Node.js Stream Backpressure 가이드: 메모리 급증 없이 처리량 높이는 법](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)처럼 전체 수집 대신 흐름 제어 중심으로 설계하는 편이 더 안전할 수 있습니다.

### H3. 지연이 긴 변환 로직은 전체 수집 시간을 늘릴 수 있다

`mapFn` 안에서 무거운 비동기 작업을 수행하면, 전체 배열 수집 시간도 그만큼 길어집니다.
특히 변환 과정이 네트워크 요청이나 디스크 I/O에 의존한다면, 수집 단계와 확장 단계의 책임을 분리하는 편이 더 나을 수 있습니다.

필요하다면 먼저 원본 데이터를 모은 뒤, 이후 단계에서 병렬 처리 전략을 별도로 적용하세요.

## 이런 상황에서 특히 추천한다

### H3. 추천하는 경우

- async generator 결과를 그대로 배열로 받아야 할 때
- 페이지네이션 결과를 최종 응답용 배열로 합칠 때
- 수집과 단순 변환을 한 번에 표현하고 싶을 때
- `for await...of` 보일러플레이트를 줄이고 싶을 때

### H3. 다른 선택지가 더 나은 경우

- 데이터를 끝까지 모으지 않고 흘려보내야 할 때
- 매우 큰 입력이라 메모리 압박이 걱정될 때
- 처음부터 Promise 배열이 준비되어 있어 병렬 대기가 핵심일 때

## 마무리

`Array.fromAsync()`는 화려한 기능이라기보다, **비동기 입력을 배열로 바꾸는 흔한 작업을 더 짧고 명확하게 만드는 표준 도구**에 가깝습니다.
특히 async iterable, 스트림, 페이지네이션 generator를 자주 다루는 코드베이스라면 `for await...of` 수집 루프를 꽤 깔끔하게 정리할 수 있습니다.

정리하면 아래처럼 기억하면 편합니다.

- 이미 Promise 배열이 있다 → `Promise.all()` 계열 검토
- 비동기적으로 생성되는 값을 모아야 한다 → `Array.fromAsync()` 검토
- 전체를 모으면 부담이 크다 → 스트림/배치 처리 우선 검토

---
layout: post
title: "Node.js structuredClone 가이드: JSON 없이 안전하게 깊은 복사하는 법"
date: 2026-05-15 08:00:00 +0900
lang: ko
translation_key: nodejs-structuredclone-deep-copy-guide
permalink: /development/blog/seo/2026/05/15/nodejs-structuredclone-deep-copy-guide.html
alternates:
  ko: /development/blog/seo/2026/05/15/nodejs-structuredclone-deep-copy-guide.html
  x_default: /development/blog/seo/2026/05/15/nodejs-structuredclone-deep-copy-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, structuredclone, javascript, deep-copy, backend, data-integrity]
description: "Node.js structuredClone으로 JSON.stringify 없이 객체를 안전하게 깊은 복사하는 방법을 정리했습니다. Date, Map, Set, ArrayBuffer 처리, transfer 옵션, 한계와 실무 적용 기준까지 예제로 설명합니다."
---

JavaScript에서 객체를 깊은 복사할 때 아직도 `JSON.parse(JSON.stringify(value))`를 습관처럼 쓰는 코드가 많습니다.
간단한 plain object에서는 동작하지만, 운영 코드에서는 곧 한계가 드러납니다.
`Date`는 문자열로 바뀌고, `Map`과 `Set`은 의도대로 보존되지 않으며, `undefined`나 순환 참조가 섞이면 데이터가 조용히 사라지거나 예외가 납니다.

Node.js의 `structuredClone()`은 이런 문제를 줄여 주는 내장 깊은 복사 API입니다.
브라우저의 structured clone algorithm과 같은 계열의 동작을 제공하며, JSON 직렬화보다 더 많은 타입을 원래 의미에 가깝게 복사할 수 있습니다.
이 글에서는 `structuredClone()`의 기본 사용법, JSON 복사와의 차이, `Map`·`Set`·`Date` 처리, `ArrayBuffer` transfer 옵션, 실무에서 피해야 할 경우까지 정리합니다.
데이터를 안전하게 검증하는 흐름까지 함께 보려면 [Node.js OpenAPI + Zod 가이드: API 계약 검증으로 응답 일관성 지키기](/development/blog/seo/2026/03/21/nodejs-openapi-zod-contract-validation-api-consistency-guide.html)도 참고하면 좋습니다.

## structuredClone이 필요한 이유

### H3. JSON 기반 깊은 복사는 타입 정보를 잃는다

가장 흔한 깊은 복사 예제는 다음과 같습니다.

```js
const copied = JSON.parse(JSON.stringify(original));
```

이 방식은 단순해 보이지만 “JSON으로 표현 가능한 값만 남긴다”는 제약이 있습니다.
운영 코드에서는 이 제약이 버그로 이어질 수 있습니다.

```js
const original = {
  createdAt: new Date('2026-05-15T08:00:00+09:00'),
  ids: new Set(['a', 'b']),
  metadata: new Map([['source', 'admin']]),
  optional: undefined,
};

const copied = JSON.parse(JSON.stringify(original));

console.log(copied.createdAt instanceof Date); // false
console.log(copied.ids); // {}
console.log(copied.metadata); // {}
console.log('optional' in copied); // false
```

이 코드는 에러 없이 실행되지만 결과가 이미 원본과 다릅니다.
특히 캐시 스냅샷, 작업 큐 payload, 관리자 설정 복사처럼 데이터 의미가 중요한 곳에서는 조용한 변환이 더 위험합니다.
타입이 바뀐 채 다음 단계로 넘어가면 실제 문제 지점과 에러가 발생한 지점이 멀어져 추적이 어려워집니다.

### H3. structuredClone은 더 많은 내장 타입을 보존한다

`structuredClone()`은 이름 그대로 구조화된 데이터를 복제합니다.
plain object와 array뿐 아니라 `Date`, `Map`, `Set`, `ArrayBuffer`, typed array 같은 값도 복사할 수 있습니다.

```js
const original = {
  createdAt: new Date('2026-05-15T08:00:00+09:00'),
  ids: new Set(['a', 'b']),
  metadata: new Map([['source', 'admin']]),
};

const copied = structuredClone(original);

console.log(copied === original); // false
console.log(copied.createdAt instanceof Date); // true
console.log(copied.ids instanceof Set); // true
console.log(copied.metadata instanceof Map); // true
console.log(copied.metadata.get('source')); // admin
```

중요한 점은 “모든 JavaScript 값을 복사한다”가 아니라 “structured clone algorithm이 지원하는 값은 의미를 유지해 복사한다”입니다.
함수, DOM 노드, 일부 호스트 객체처럼 복사할 수 없는 값은 `DataCloneError`가 납니다.
따라서 입력이 무엇인지 모르는 경계에서는 예외 처리를 함께 두는 편이 안전합니다.
입력 URL처럼 예외 없는 검증이 필요한 값은 [Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)과 같은 방식으로 별도 검증 계층을 두는 것이 좋습니다.

## 기본 사용법: 원본을 보호하는 복사본 만들기

### H3. 설정 객체를 복사해 부작용을 차단한다

`structuredClone()`을 쓰기 좋은 대표 사례는 설정 객체를 넘겨받은 뒤 내부에서 안전하게 수정해야 하는 경우입니다.
외부에서 전달한 객체를 그대로 변경하면 호출자 쪽 상태까지 바뀌어 예측하기 어려운 버그가 생깁니다.

```js
const defaultRetryPolicy = {
  attempts: 3,
  delayMs: 200,
  retryableStatus: new Set([408, 429, 500, 502, 503, 504]),
};

export function createClient(options = {}) {
  const retryPolicy = structuredClone(
    options.retryPolicy ?? defaultRetryPolicy,
  );

  retryPolicy.attempts = Math.max(1, retryPolicy.attempts);

  return {
    request(path) {
      return fetchWithRetry(path, retryPolicy);
    },
  };
}
```

이렇게 복사본을 만든 뒤 정규화하면, 기본 설정이나 호출자가 가진 원본 객체를 건드리지 않습니다.
특히 `Set` 같은 컬렉션이 포함된 설정에서는 JSON 복사보다 의도가 분명합니다.
재시도 정책을 더 체계적으로 설계하려면 [Node.js exponential backoff + jitter 가이드: 재시도 폭주를 막는 실전 전략](/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html)를 함께 볼 만합니다.

### H3. 캐시 입력값을 스냅샷으로 고정한다

비동기 코드에서는 객체가 참조로 공유되면서 예상치 못한 시점에 값이 바뀔 수 있습니다.
큐에 넣은 작업 payload, 캐시에 저장한 옵션, 감사 로그용 이벤트는 “그 순간의 값”을 보존하는 것이 중요합니다.

```js
const auditEvents = [];

export function recordAuditEvent(event) {
  const snapshot = structuredClone({
    ...event,
    recordedAt: new Date(),
  });

  auditEvents.push(snapshot);
}

const event = {
  type: 'role.updated',
  actorId: 'user_123',
  roles: new Set(['editor']),
};

recordAuditEvent(event);
event.roles.add('admin');

console.log(auditEvents[0].roles.has('admin')); // false
```

이 패턴은 감사 로그나 이벤트 처리에서 특히 유용합니다.
다만 `actorId`처럼 예시 식별자는 실제 개인정보나 토큰을 그대로 넣지 않도록 주의해야 합니다.
로그 예제의 민감정보 처리 기준은 [로그 예제 정제 가이드: 믿을 수 있는 개발 글을 위한 마스킹 원칙](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)에 정리해 두었습니다.

## JSON 복사와 structuredClone 비교

### H3. Date, Map, Set, 순환 참조에서 차이가 크다

두 방식의 차이는 작은 예제로도 확인할 수 있습니다.

```js
const original = {
  now: new Date(),
  tags: new Set(['nodejs', 'backend']),
  counts: new Map([['success', 10]]),
};

original.self = original;

try {
  JSON.parse(JSON.stringify(original));
} catch (error) {
  console.error('json copy failed:', error.message);
}

const copied = structuredClone(original);

console.log(copied.self === copied); // true
console.log(copied.now instanceof Date); // true
console.log(copied.tags.has('nodejs')); // true
console.log(copied.counts.get('success')); // 10
```

순환 참조를 포함한 객체는 JSON 직렬화로 복사할 수 없습니다.
반면 `structuredClone()`은 지원 타입 안에서 순환 구조를 유지합니다.
그래서 복잡한 상태 객체를 다루는 테스트, 작업 큐 중간 처리, in-memory 인덱스 스냅샷에서 더 안전한 선택이 될 수 있습니다.

하지만 순환 구조를 복사할 수 있다고 해서 순환 구조를 남발해도 된다는 뜻은 아닙니다.
데이터 모델이 지나치게 얽혀 있으면 직렬화, 로깅, 디버깅, 캐시 무효화가 어려워집니다.
가능하면 경계에서 plain data transfer object로 정리하고, 내부 구현에서만 복잡한 참조를 유지하는 편이 좋습니다.

### H3. 함수와 클래스 인스턴스는 설계를 다시 봐야 한다

`structuredClone()`은 함수를 복사하지 않습니다.
클래스 인스턴스도 기대한 방식으로 메서드까지 보존하는 용도가 아닙니다.

```js
const job = {
  name: 'send-email',
  run() {
    console.log('running');
  },
};

structuredClone(job); // DataCloneError
```

이 에러는 오히려 좋은 신호일 수 있습니다.
큐 payload나 API 응답처럼 경계를 넘어가는 데이터에 함수가 들어간다면, 데이터와 동작이 섞였다는 뜻이기 때문입니다.
실무에서는 다음처럼 분리하는 편이 안전합니다.

```js
const jobPayload = {
  type: 'send-email',
  to: 'masked@example.com',
  templateId: 'welcome',
};

const snapshot = structuredClone(jobPayload);
```

동작은 `type`에 따라 실행기에서 선택하고, payload는 복사 가능한 데이터로 유지합니다.
이 구조는 실패한 작업을 재처리하거나 dead letter queue로 보낼 때도 유리합니다.
관련 복구 패턴은 [Node.js Dead Letter Queue 가이드: BullMQ 실패 작업을 안전하게 복구하기](/development/blog/seo/2026/03/14/dead-letter-queue-bullmq-nodejs-failed-job-recovery-guide.html)를 참고하세요.

## ArrayBuffer와 transfer 옵션 사용하기

### H3. 큰 바이너리는 복사 대신 소유권 이전을 고려한다

`structuredClone()`은 두 번째 인자로 `transfer` 옵션을 받을 수 있습니다.
`ArrayBuffer` 같은 transferable 객체를 복사하지 않고 새 객체로 소유권을 넘길 때 사용합니다.
큰 바이너리 데이터를 worker thread로 넘기거나, 원본을 더 이상 쓰지 않는 상황에서 유용합니다.

```js
const buffer = new ArrayBuffer(1024);
const view = new Uint8Array(buffer);
view[0] = 42;

const copied = structuredClone({ buffer }, {
  transfer: [buffer],
});

console.log(copied.buffer.byteLength); // 1024
console.log(buffer.byteLength); // 0
```

transfer된 원본 `ArrayBuffer`는 detached 상태가 되어 더 이상 정상적으로 사용할 수 없습니다.
따라서 transfer는 “원본을 이후에 쓰지 않는다”는 소유권 규칙이 명확할 때만 써야 합니다.
성능 최적화를 위해 무심코 적용하면, 뒤쪽 코드에서 빈 버퍼를 읽는 버그가 생길 수 있습니다.

worker thread와 함께 CPU 작업을 분리하는 경우라면 [Node.js worker_threads 가이드: CPU 바운드 작업을 안전하게 분리하기](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)와 함께 설계하는 것이 좋습니다.

### H3. 성능 최적화 전에 복사 범위를 줄인다

깊은 복사는 공짜가 아닙니다.
객체가 클수록 CPU와 메모리를 사용하고, 복사 시점에 이벤트 루프 지연을 만들 수 있습니다.
그래서 `structuredClone()`을 쓰더라도 “무조건 전체 상태를 복사”하는 패턴은 피해야 합니다.

```js
function snapshotOrderForAudit(order) {
  return structuredClone({
    orderId: order.id,
    status: order.status,
    items: order.items.map((item) => ({
      sku: item.sku,
      quantity: item.quantity,
      price: item.price,
    })),
    updatedAt: order.updatedAt,
  });
}
```

필요한 필드만 골라 작은 객체를 만든 뒤 복사하면 비용과 정보 노출 위험을 동시에 줄일 수 있습니다.
운영 로그, 감사 이벤트, 외부 API payload에서는 이 방식이 특히 중요합니다.
외부 API 응답을 안정적으로 다루는 관점은 [HTTP 요청 타임아웃과 fail-fast 가이드: 느린 API에 끌려가지 않는 법](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)와도 이어집니다.

## 실무 적용 체크리스트

### H3. structuredClone을 쓰기 좋은 경우

`structuredClone()`은 다음 상황에서 좋은 기본 선택지가 됩니다.

- JSON으로 표현하기 어려운 `Date`, `Map`, `Set`이 포함된 객체를 복사할 때
- 호출자가 넘긴 옵션 객체를 내부에서 정규화해야 할 때
- 작업 큐나 감사 로그에 “현재 시점의 스냅샷”을 남겨야 할 때
- 순환 참조가 있을 수 있는 테스트 fixture를 복사해야 할 때
- `ArrayBuffer`를 worker thread로 넘기며 transfer를 명시하고 싶을 때

반대로 데이터가 이미 JSON API 경계에 있고, 의도적으로 JSON 직렬화 가능성만 허용해야 한다면 JSON 변환이 더 명확할 수도 있습니다.
예를 들어 외부 API 응답 계약을 검증하는 단계에서는 “JSON으로 표현 가능한 DTO”만 통과시키는 편이 유지보수에 좋습니다.

### H3. 피해야 할 사용 방식

다음 패턴은 조심해야 합니다.

- 민감정보가 섞인 전체 요청 객체를 통째로 복사해 로그에 저장하기
- 함수나 복잡한 클래스 인스턴스를 복사 가능한 데이터처럼 다루기
- 큰 상태 객체를 요청마다 반복 복사하기
- transfer 후 원본 버퍼를 계속 사용할 수 있다고 가정하기
- 복사 실패 예외를 잡지 않은 채 사용자 입력 경계에서 호출하기

특히 요청 객체 전체를 복사하는 코드는 개인정보, 인증 헤더, 쿠키, 내부 토큰을 함께 보관할 위험이 있습니다.
필요한 필드만 선별하고, 값이 민감할 수 있으면 마스킹한 뒤 저장해야 합니다.
이 원칙은 기술 블로그 예제에도 그대로 적용됩니다.
실제 토큰, 실제 사용자 이메일, 내부 호스트명을 예제에 넣지 않는 것이 안전합니다.

## FAQ

### H3. structuredClone은 lodash cloneDeep을 대체할 수 있나요?

많은 경우 대체할 수 있지만 완전한 1:1 대체는 아닙니다.
`structuredClone()`은 플랫폼 내장 API이고 지원 타입이 명확하지만, 함수나 특수 객체 처리 방식은 lodash의 `cloneDeep`과 다릅니다.
기존 코드에서 교체할 때는 테스트로 타입 보존, 예외 발생, 클래스 인스턴스 처리 방식을 확인해야 합니다.

### H3. structuredClone은 JSON 복사보다 항상 빠른가요?

항상 그렇지는 않습니다.
데이터 모양, 크기, 포함 타입, Node.js 버전에 따라 달라집니다.
성능만 보고 선택하기보다 타입 보존과 실패 방식까지 함께 봐야 합니다.
큰 객체를 자주 복사한다면 먼저 복사 범위를 줄이고, 필요한 경우 실제 payload로 벤치마크하는 편이 안전합니다.
측정 기준을 잡을 때는 [Node.js performance.now 가이드: console.time보다 정확한 실행 시간 측정](/development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html)을 활용할 수 있습니다.

### H3. structuredClone이 실패하면 어떻게 처리해야 하나요?

입력 경계에서 호출한다면 `try...catch`로 `DataCloneError`를 잡고, 복사 가능한 데이터만 받도록 검증을 추가하는 것이 좋습니다.
내부 코드에서 실패한다면 payload 설계가 잘못됐을 가능성이 큽니다.
함수, 스트림, 소켓, 요청 객체처럼 동작이나 외부 리소스를 가진 값은 복사 대상에서 제외하고 plain data로 분리하세요.

## 마무리

`structuredClone()`은 Node.js에서 깊은 복사를 더 안전하고 명확하게 처리할 수 있는 내장 도구입니다.
JSON 기반 복사보다 타입 보존 범위가 넓고, 순환 참조와 transferable 객체도 다룰 수 있습니다.
하지만 모든 값을 복사하는 만능 API는 아니며, 함수·리소스·민감정보가 섞인 객체를 그대로 복제하는 설계는 피해야 합니다.

실무 기준은 단순합니다.
복사해야 하는 값이 데이터인지 먼저 확인하고, 필요한 필드만 작게 만들고, 타입 보존이 중요하면 `structuredClone()`을 사용하세요.
그렇게 하면 깊은 복사가 조용히 데이터를 바꾸는 문제를 줄이고, 설정·캐시·감사 로그 같은 코드의 안정성을 높일 수 있습니다.

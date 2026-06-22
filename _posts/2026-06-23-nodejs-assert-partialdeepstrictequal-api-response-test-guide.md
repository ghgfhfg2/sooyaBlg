---
layout: post
title: "Node.js assert.partialDeepStrictEqual 가이드: API 응답 테스트를 덜 깨지게 만드는 부분 검증"
date: 2026-06-23 08:00:00 +0900
lang: ko
translation_key: nodejs-assert-partialdeepstrictequal-api-response-test-guide
permalink: /development/blog/seo/2026/06/23/nodejs-assert-partialdeepstrictequal-api-response-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/23/nodejs-assert-partialdeepstrictequal-api-response-test-guide.html
  x_default: /development/blog/seo/2026/06/23/nodejs-assert-partialdeepstrictequal-api-response-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, partialDeepStrictEqual, test, api, backend]
description: "Node.js assert.partialDeepStrictEqual()로 API 응답과 이벤트 payload의 핵심 필드만 안전하게 검증하는 방법을 정리합니다. deepStrictEqual과의 차이, 테스트 설계 기준, 과검증을 줄이는 패턴을 예제로 설명합니다."
---

API 응답 테스트가 자주 깨지는 이유가 항상 버그 때문은 아닙니다.
응답에 `createdAt`, `requestId`, `links`, `metadata` 같은 필드가 조금씩 늘어났을 뿐인데, 테스트가 전체 객체를 엄격하게 비교하고 있으면 정상적인 변경도 실패로 보일 수 있습니다.
반대로 너무 느슨하게 `ok === true`만 확인하면 실제 계약이 깨져도 놓치기 쉽습니다.

이 사이에서 쓸 수 있는 선택지가 Node.js `node:assert/strict`의 `assert.partialDeepStrictEqual()`입니다.
[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)에 따르면 이 메서드는 `expected`에 존재하는 속성만 깊게 비교하는 부분 검증을 수행합니다.
즉 전체 객체를 모두 고정하지 않고도, 테스트가 정말 지켜야 하는 계약을 명확하게 표현할 수 있습니다.

내장 테스트 러너의 기본 구조가 필요하다면 [Node.js built-in test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 참고하세요.
외부 의존성 호출을 검증하는 테스트는 [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html)와 함께 보면 좋습니다.
CI 출력과 리포터 구성을 다듬고 있다면 [Node.js test runner reporter 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-reporter-ci-junit-guide.html)도 연결됩니다.

## partialDeepStrictEqual이 필요한 이유

### H3. 전체 응답 비교는 변경에 약하다

`assert.deepStrictEqual()`은 실제 값과 기대 값을 깊게 비교합니다.
API 계약 전체를 고정해야 하는 경우에는 좋은 선택입니다.
하지만 응답 객체가 커지고, 부가 필드가 자주 늘어나는 서비스에서는 작은 확장에도 테스트가 깨질 수 있습니다.

```js
import assert from 'node:assert/strict';

const response = {
  id: 'order_123',
  status: 'paid',
  amount: 39000,
  currency: 'KRW',
  createdAt: '2026-06-23T08:00:00.000Z',
  requestId: 'req_abc'
};

assert.deepStrictEqual(response, {
  id: 'order_123',
  status: 'paid',
  amount: 39000,
  currency: 'KRW'
});
```

위 테스트는 핵심 계약이 맞아도 실패합니다.
`createdAt`과 `requestId`가 실제 응답에 추가되어 있기 때문입니다.
테스트의 의도가 "주문 상태와 금액이 맞는가"라면 전체 객체 비교는 의도보다 많은 것을 고정합니다.

### H3. 부분 검증은 핵심 계약만 고정한다

`assert.partialDeepStrictEqual()`은 `expected`에 적은 속성만 비교합니다.
실제 객체에 추가 속성이 있어도, 기대한 핵심 필드가 맞으면 통과합니다.

```js
import assert from 'node:assert/strict';

const response = {
  id: 'order_123',
  status: 'paid',
  amount: 39000,
  currency: 'KRW',
  createdAt: '2026-06-23T08:00:00.000Z',
  requestId: 'req_abc'
};

assert.partialDeepStrictEqual(response, {
  id: 'order_123',
  status: 'paid',
  amount: 39000,
  currency: 'KRW'
});
```

이 테스트는 응답 확장에는 덜 민감하지만, 중요한 값이 바뀌면 여전히 실패합니다.
그래서 API 응답, 이벤트 payload, 설정 객체처럼 필드가 늘어날 수 있는 구조에서 특히 유용합니다.

## deepStrictEqual과의 차이

### H3. expected에 있는 속성만 비교한다

부분 검증의 기준은 `expected`입니다.
`actual`에 어떤 속성이 더 있느냐가 아니라, `expected`에 적은 속성이 `actual` 안에서 같은 구조와 값으로 존재하는지가 핵심입니다.

```js
import assert from 'node:assert/strict';

const actual = {
  user: {
    id: 'user_1',
    name: 'Soo',
    role: 'admin',
    profile: {
      locale: 'ko-KR',
      timezone: 'Asia/Seoul'
    }
  }
};

assert.partialDeepStrictEqual(actual, {
  user: {
    id: 'user_1',
    profile: {
      timezone: 'Asia/Seoul'
    }
  }
});
```

중첩 객체에서도 같은 원칙이 적용됩니다.
위 예시는 `name`, `role`, `profile.locale`을 고정하지 않습니다.
하지만 `user.id`와 `user.profile.timezone`은 테스트 계약으로 남습니다.

### H3. 값 비교는 strict 계열의 규칙을 따른다

이 메서드는 이름 그대로 strict 계열입니다.
문자열 `'1'`과 숫자 `1`을 같은 값으로 보지 않습니다.
테스트에서 타입까지 계약으로 다뤄야 하는 API 응답에는 이 점이 중요합니다.

```js
import assert from 'node:assert/strict';

assert.partialDeepStrictEqual(
  { page: 1, pageSize: 20 },
  { page: '1' }
);
```

위 코드는 실패해야 정상입니다.
페이지 번호가 숫자라는 계약을 지키고 싶은 테스트에서 문자열 변환 버그를 놓치지 않기 때문입니다.

## API 응답 테스트에 적용하기

### H3. 성공 응답은 상태와 핵심 데이터만 고정한다

서비스 응답에는 추적용 필드, pagination 정보, 링크, 실험 플래그가 추가될 수 있습니다.
테스트가 모든 부가 필드를 고정하면 기능 변경보다 테스트 수정이 더 많아집니다.
성공 응답에서는 사용자에게 중요한 계약을 먼저 고정하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { createOrder } from '../src/orders.js';

test('createOrder returns paid order summary', async () => {
  const result = await createOrder({
    productId: 'prod_1',
    quantity: 2
  });

  assert.partialDeepStrictEqual(result, {
    ok: true,
    order: {
      status: 'paid',
      quantity: 2,
      currency: 'KRW'
    }
  });
});
```

이 테스트는 `order.id`, `createdAt`, `payment.provider`, `links.self` 같은 필드가 추가되어도 계속 통과할 수 있습니다.
하지만 결제 상태, 수량, 통화가 바뀌면 실패합니다.
테스트가 지켜야 할 계약이 더 선명해집니다.

### H3. 오류 응답은 코드와 복구 힌트를 검증한다

오류 응답에서도 전체 메시지를 고정하면 문구 수정만으로 테스트가 깨질 수 있습니다.
대신 클라이언트가 분기하는 `code`, 재시도 가능 여부, 사용자 조치 힌트처럼 실제 계약이 되는 필드를 검증합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { reserveStock } from '../src/stock.js';

test('reserveStock returns stable error contract', async () => {
  const result = await reserveStock({
    productId: 'prod_sold_out',
    quantity: 1
  });

  assert.partialDeepStrictEqual(result, {
    ok: false,
    error: {
      code: 'OUT_OF_STOCK',
      retryable: false
    }
  });
});
```

에러 메시지 전체를 고정해야 하는 경우도 있습니다.
예를 들어 외부 공개 API 문서에 정확한 문구까지 계약으로 명시했다면 별도 테스트로 확인할 수 있습니다.
다만 대부분의 내부 서비스에서는 코드와 구조를 고정하고 문구는 덜 엄격하게 두는 편이 유지보수에 유리합니다.

## 이벤트와 큐 메시지 검증

### H3. 이벤트 payload는 소비자 계약 중심으로 본다

이벤트 기반 시스템에서는 producer가 payload를 확장하는 일이 흔합니다.
새 필드 추가가 기존 consumer를 깨뜨리지 않는다면, consumer 테스트는 자신이 읽는 필드만 검증하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';

const event = {
  type: 'order.paid',
  version: 2,
  data: {
    orderId: 'order_123',
    userId: 'user_1',
    amount: 39000,
    currency: 'KRW'
  },
  metadata: {
    traceId: 'trace_abc',
    publishedAt: '2026-06-23T08:00:00.000Z'
  }
};

assert.partialDeepStrictEqual(event, {
  type: 'order.paid',
  data: {
    orderId: 'order_123',
    amount: 39000
  }
});
```

이 방식은 이벤트 확장성을 보존합니다.
동시에 consumer가 실제로 의존하는 필드가 빠지거나 값이 바뀌면 테스트가 실패합니다.
이벤트 중복 처리와 일관성 설계는 [Node.js transactional outbox pattern 가이드](/development/blog/seo/2026/03/15-transactional-outbox-pattern-nodejs-event-driven-consistency-guide.html)와 함께 검토할 수 있습니다.

### H3. 배열은 필요한 항목을 별도로 찾아 검증한다

부분 검증을 배열 전체에 바로 적용하면 테스트 의도가 흐려질 수 있습니다.
목록 응답에서는 먼저 검증 대상 항목을 찾고, 그 항목에 대해 부분 검증을 적용하는 편이 읽기 쉽습니다.

```js
import assert from 'node:assert/strict';

const users = await listUsers();
const admin = users.find((user) => user.email === 'admin@example.test');

assert.ok(admin);
assert.partialDeepStrictEqual(admin, {
  role: 'admin',
  active: true
});
```

목록 순서가 계약이라면 순서 테스트를 따로 둡니다.
순서가 계약이 아니라면 특정 index를 고정하지 않는 편이 변경에 강합니다.

## 과검증을 줄이는 기준

### H3. 공개 계약과 내부 구현을 분리한다

테스트에서 모든 필드를 검증하고 싶어지는 순간이 있습니다.
하지만 공개 계약과 내부 구현 정보를 구분하지 않으면 테스트가 코드를 보호하기보다 변경을 방해합니다.

검증 대상으로 삼기 좋은 필드는 다음과 같습니다.

- 클라이언트가 분기하는 상태값
- API 문서에 명시한 식별자와 타입
- 금액, 수량, 권한처럼 비즈니스 의미가 큰 값
- 재시도 가능 여부, 에러 코드처럼 운영 흐름에 영향을 주는 값

반대로 `requestId`, `debug`, 내부 계산 단계, 실험용 metadata처럼 자주 바뀌는 값은 별도 목적이 있을 때만 고정합니다.
로그와 테스트 데이터를 다룰 때 민감정보가 섞이지 않도록 [로그 예제 sanitization 가이드](/development/blog/seo/2026/03/03-log-example-sanitization-for-trustworthy-dev-posts.html)의 기준도 함께 적용하세요.

### H3. 스냅샷 테스트와 역할을 나눈다

Node.js test runner는 snapshot 테스트도 제공합니다.
[Node.js test runner 공식 문서](https://nodejs.org/api/test.html)는 snapshot 파일을 테스트 파일별로 생성하고, `--test-update-snapshots` 플래그로 갱신할 수 있다고 설명합니다.
스냅샷은 큰 출력물의 회귀를 확인할 때 편하지만, 작은 API 계약까지 모두 스냅샷으로 고정하면 변경 검토 비용이 커질 수 있습니다.

실무에서는 역할을 나누는 편이 좋습니다.
핵심 계약은 `partialDeepStrictEqual()`로 명확하게 검증하고, 큰 렌더링 결과나 사람이 검토할 구조는 snapshot으로 다룹니다.
이렇게 하면 테스트 실패 메시지가 "어떤 계약이 깨졌는지"를 더 직접적으로 보여 줍니다.

## 실무 체크리스트

### H3. 테스트 작성 전 확인할 질문

부분 검증은 느슨한 테스트를 만들기 위한 도구가 아닙니다.
무엇을 고정할지 더 의식적으로 선택하기 위한 도구입니다.
테스트를 작성하기 전에 아래 질문을 확인하면 기준이 선명해집니다.

- 이 필드는 클라이언트나 consumer가 실제로 의존하는가?
- 값뿐 아니라 타입도 계약인가?
- 새 필드가 추가되어도 기존 테스트가 통과해야 하는가?
- 순서가 중요한가, 아니면 항목의 존재가 중요한가?
- 오류 문구 전체가 계약인가, 아니면 오류 코드가 계약인가?

이 질문에 답하고 나면 `deepStrictEqual()`, `partialDeepStrictEqual()`, `assert.match()`, snapshot 중 어떤 도구가 맞는지 고르기 쉬워집니다.

### H3. 실패 메시지가 읽히도록 기대 값을 작게 유지한다

`expected` 객체가 너무 커지면 부분 검증을 쓰더라도 실패 메시지가 복잡해집니다.
한 테스트에서 여러 계약을 한꺼번에 검증하기보다, 의미 있는 단위로 나누는 편이 낫습니다.

```js
assert.partialDeepStrictEqual(result, {
  ok: true,
  order: {
    status: 'paid'
  }
});

assert.partialDeepStrictEqual(result, {
  invoice: {
    currency: 'KRW',
    totalAmount: 39000
  }
});
```

위처럼 상태 계약과 청구 금액 계약을 나누면 실패 원인이 더 빨리 보입니다.
테스트 이름도 각각의 계약을 설명하도록 붙이면 CI 로그에서 원인을 찾기 쉽습니다.

## FAQ

### H3. partialDeepStrictEqual은 deepStrictEqual을 대체하나요?

아닙니다.
전체 객체가 정확히 같아야 하는 계약에는 `deepStrictEqual()`이 더 적합합니다.
`partialDeepStrictEqual()`은 실제 객체가 더 많은 정보를 포함해도 괜찮고, 그중 핵심 필드만 고정하고 싶을 때 쓰는 도구입니다.

### H3. API 응답에 새 필드가 추가되면 테스트가 항상 통과해야 하나요?

항상 그렇지는 않습니다.
새 필드가 기존 계약을 바꾸지 않는 확장이라면 통과하는 것이 자연스럽습니다.
하지만 새 필드 추가가 보안, 권한, 금액 계산, 공개 API 문서와 연결된다면 별도 테스트로 명시하는 편이 좋습니다.

### H3. 부분 검증이 테스트를 너무 느슨하게 만들지는 않나요?

그럴 수 있습니다.
그래서 `expected`에는 실제 소비자가 의존하는 필드를 빠뜨리지 않아야 합니다.
핵심 필드를 검증하지 않고 단순히 테스트를 통과시키기 위해 쓰면 오히려 회귀를 놓칠 수 있습니다.

## 마무리

`assert.partialDeepStrictEqual()`은 Node.js 테스트에서 "중요한 계약은 엄격하게, 확장 가능한 필드는 유연하게" 다루기 위한 도구입니다.
API 응답, 이벤트 payload, 오류 객체처럼 구조가 커질 수 있는 값을 테스트할 때 특히 효과적입니다.
전체 비교가 필요한 곳에는 `deepStrictEqual()`을 쓰고, 소비자 계약 중심의 검증에는 `partialDeepStrictEqual()`을 적용하면 테스트가 덜 흔들리면서도 더 정확한 신호를 줄 수 있습니다.

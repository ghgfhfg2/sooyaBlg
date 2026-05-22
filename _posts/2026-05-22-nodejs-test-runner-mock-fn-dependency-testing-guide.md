---
layout: post
title: "Node.js test runner mock.fn 가이드: 외부 의존성 호출을 안전하게 검증하는 법"
date: 2026-05-22 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-mock-fn-dependency-testing-guide
permalink: /development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html
alternates:
  ko: /development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html
  x_default: /development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, mock-fn, unit-test, dependency-injection, javascript, testing]
description: "Node.js 내장 test runner의 mock.fn으로 외부 API, 저장소, 큐 발행 같은 의존성 호출을 검증하는 방법을 정리했습니다. 테스트 격리, 호출 인자 확인, reset/restore 기준, 실무 체크리스트까지 다룹니다."
---

단위 테스트에서 가장 자주 막히는 지점은 비즈니스 로직 자체보다 외부 의존성입니다.
결제 API를 실제로 호출할 수는 없고, 메일을 매번 발송할 수도 없으며, 데이터베이스 상태에 따라 테스트 결과가 흔들리면 신뢰하기 어렵습니다.
이럴 때 필요한 것은 무작정 모든 것을 가짜로 바꾸는 일이 아니라, 코드가 의존성을 어떤 인자로 몇 번 호출했는지 안전하게 확인하는 기준입니다.

Node.js 내장 test runner의 `mock.fn`은 별도 mocking 라이브러리 없이 함수 호출을 기록하고, 원하는 반환값이나 구현을 주입할 수 있게 해 줍니다.
테스트 러너 자체를 처음 정리한다면 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 보고, 시간 의존 테스트는 [Node.js mock timers 가이드](/development/blog/seo/2026/04/29/nodejs-test-runner-mock-timers-guide.html)와 함께 보면 좋습니다.
이 글에서는 `mock.fn`을 외부 의존성 검증에 쓰는 실무 패턴을 정리합니다.

## Node.js mock.fn이 필요한 이유

### 외부 호출은 테스트를 느리고 불안정하게 만든다

단위 테스트는 입력과 출력이 빠르게 검증되어야 합니다.
하지만 테스트 안에서 실제 HTTP API, 메시지 큐, 파일 시스템, 메일 서버를 직접 호출하면 네트워크 지연과 환경 차이에 영향을 받습니다.
운영 장애를 줄이기 위한 테스트가 오히려 불안정한 신호가 되는 셈입니다.

```js
export async function createInvoice(order, paymentClient) {
  if (order.total <= 0) {
    throw new RangeError('order total must be positive');
  }

  const payment = await paymentClient.charge({
    orderId: order.id,
    amount: order.total,
  });

  return {
    invoiceId: `inv_${order.id}`,
    paymentId: payment.id,
  };
}
```

위 함수에서 중요한 것은 실제 결제사가 응답하는지가 아닙니다.
`createInvoice`가 유효하지 않은 주문을 거부하고, 유효한 주문일 때 결제 의존성을 올바른 인자로 호출하는지가 핵심입니다.
이 지점을 `mock.fn`으로 작게 고립하면 테스트가 빨라지고 실패 원인도 명확해집니다.

### 호출 기록은 계약 테스트의 가장 작은 단위다

mock은 단순히 결과를 가짜로 만드는 도구가 아닙니다.
함수 호출 횟수, 호출 인자, 호출 순서, throw 여부를 기록해 코드의 계약을 검증하는 장치입니다.
특히 외부 API를 감싼 어댑터나 저장소 계층에서는 “무엇을 호출했는가”가 곧 중요한 동작입니다.

관측 가능성 관점에서는 테스트 로그도 운영 로그처럼 재현 가능해야 합니다.
테스트 실패를 구조적으로 읽는 흐름은 [Node.js assert.partialDeepStrictEqual 가이드](/development/blog/seo/2026/05/18/nodejs-assert-partialdeepstrictequal-object-subset-test-guide.html)와 연결해서 정리하면 좋습니다.

## mock.fn 기본 사용법

### 함수 호출 횟수와 인자를 확인한다

`mock.fn`은 `node:test` 모듈의 `mock` 객체에서 사용할 수 있습니다.
가장 기본적인 패턴은 의존성 함수를 mock으로 만들고, 테스트 대상 함수에 주입한 뒤 호출 기록을 확인하는 방식입니다.

```js
import assert from 'node:assert/strict';
import test, { mock } from 'node:test';
import { createInvoice } from './invoice.js';

test('결제 클라이언트를 주문 금액으로 호출한다', async () => {
  const charge = mock.fn(async () => ({ id: 'pay_123' }));
  const paymentClient = { charge };

  const invoice = await createInvoice(
    { id: 'order_1', total: 30000 },
    paymentClient,
  );

  assert.equal(invoice.invoiceId, 'inv_order_1');
  assert.equal(invoice.paymentId, 'pay_123');
  assert.equal(charge.mock.callCount(), 1);
  assert.deepEqual(charge.mock.calls[0].arguments[0], {
    orderId: 'order_1',
    amount: 30000,
  });
});
```

이 테스트는 외부 결제 서버 없이도 핵심 계약을 검증합니다.
실패했을 때도 네트워크 문제인지, 비즈니스 로직 문제인지 헷갈리지 않습니다.

### 반환값과 예외를 테스트별로 바꾼다

외부 의존성은 항상 성공하지 않습니다.
`mock.fn`에 async 구현을 전달하면 성공과 실패 케이스를 테스트별로 분리할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test, { mock } from 'node:test';
import { createInvoice } from './invoice.js';

test('결제 실패를 호출자에게 전달한다', async () => {
  const charge = mock.fn(async () => {
    throw new Error('payment gateway unavailable');
  });

  await assert.rejects(
    createInvoice({ id: 'order_2', total: 5000 }, { charge }),
    /payment gateway unavailable/,
  );

  assert.equal(charge.mock.callCount(), 1);
});
```

실패 테스트에서는 에러 메시지 전체보다 의미 있는 패턴을 검증하는 편이 좋습니다.
에러를 감싸서 원인을 보존하는 구조는 [Node.js Error cause 가이드](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)를 참고하면 테스트 기대값을 더 안정적으로 만들 수 있습니다.

## 의존성 주입으로 mock 범위를 줄이기

### 전역 모듈 교체보다 인자 주입이 단순하다

mock이 복잡해지는 가장 흔한 이유는 테스트 대상 코드가 내부에서 직접 의존성을 생성하기 때문입니다.
예를 들어 함수 안에서 HTTP 클라이언트를 바로 만들면 테스트가 그 내부 구현을 가로채야 합니다.
반대로 의존성을 인자로 받으면 `mock.fn`만으로 충분합니다.

```js
export function makeUserService({ userRepository, eventPublisher }) {
  return {
    async activateUser(userId) {
      const user = await userRepository.findById(userId);

      if (!user) {
        throw new Error('user not found');
      }

      await userRepository.updateStatus(userId, 'active');
      await eventPublisher.publish('user.activated', { userId });

      return { userId, status: 'active' };
    },
  };
}
```

이 구조에서는 저장소와 이벤트 발행기를 모두 mock으로 바꿀 수 있습니다.
테스트 대상은 `activateUser`의 흐름이고, 실제 데이터베이스나 브로커 연결은 통합 테스트에서 별도로 확인하면 됩니다.

### 여러 의존성의 호출 순서를 검증한다

상태 변경 후 이벤트를 발행해야 하는 코드에서는 호출 순서도 중요합니다.
`mock.fn`의 호출 기록을 활용하면 의도한 순서를 비교할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test, { mock } from 'node:test';
import { makeUserService } from './user-service.js';

test('사용자 상태를 변경한 뒤 이벤트를 발행한다', async () => {
  const steps = [];
  const userRepository = {
    findById: mock.fn(async () => ({ id: 'user_1' })),
    updateStatus: mock.fn(async () => {
      steps.push('update');
    }),
  };
  const eventPublisher = {
    publish: mock.fn(async () => {
      steps.push('publish');
    }),
  };

  const service = makeUserService({ userRepository, eventPublisher });

  await service.activateUser('user_1');

  assert.deepEqual(steps, ['update', 'publish']);
  assert.equal(userRepository.updateStatus.mock.callCount(), 1);
  assert.equal(eventPublisher.publish.mock.callCount(), 1);
});
```

모든 내부 구현 순서를 세세하게 고정하면 리팩터링이 어려워집니다.
하지만 데이터 변경 후 이벤트 발행처럼 깨지면 장애로 이어지는 순서는 명시적으로 검증할 가치가 있습니다.
이벤트 발행 안정성은 [Node.js outbox pattern 가이드](/development/blog/seo/2026/03/28/nodejs-outbox-pattern-event-driven-consistency-guide.html)와 함께 설계하면 더 안전합니다.

## mock 상태 관리 기준

### 테스트 간 호출 기록을 공유하지 않는다

mock 함수는 호출 기록을 갖고 있기 때문에 테스트 간에 같은 인스턴스를 공유하면 결과가 오염될 수 있습니다.
가장 안전한 방식은 테스트 안에서 mock을 새로 만드는 것입니다.
공통 준비 코드가 필요하다면 팩토리 함수를 만들어 매번 새 객체를 반환하게 합니다.

```js
import { mock } from 'node:test';

export function createMockUserDependencies() {
  return {
    userRepository: {
      findById: mock.fn(async () => ({ id: 'user_1' })),
      updateStatus: mock.fn(async () => undefined),
    },
    eventPublisher: {
      publish: mock.fn(async () => undefined),
    },
  };
}
```

테스트 파일 상단에 만든 mock을 여러 테스트가 같이 쓰면 첫 번째 테스트의 호출 기록이 두 번째 테스트에 남을 수 있습니다.
이 문제는 테스트가 병렬화될수록 더 찾기 어려워집니다.
테스트 격리와 병렬 실행 기준은 [Node.js test runner coverage 가이드](/development/blog/seo/2026/05/10/nodejs-test-runner-coverage-v8-guide.html)처럼 CI 실행 흐름과 함께 관리하는 편이 좋습니다.

### mock.method는 복구 기준을 분명히 한다

객체의 기존 메서드를 잠깐 바꿔야 한다면 `mock.method`를 사용할 수 있습니다.
이 경우 테스트가 끝난 뒤 원래 구현으로 복구하는 기준이 특히 중요합니다.

```js
import assert from 'node:assert/strict';
import test, { mock } from 'node:test';

const clock = {
  now() {
    return Date.now();
  },
};

test('현재 시간을 고정해 만료 여부를 계산한다', () => {
  mock.method(clock, 'now', () => 1_800_000);

  const expiresAt = 1_700_000;
  const expired = clock.now() > expiresAt;

  assert.equal(expired, true);
  mock.restoreAll();
});
```

`mock.restoreAll()`은 편리하지만, 테스트 중간에 호출하면 같은 테스트 안의 다른 mock까지 복구될 수 있습니다.
가능하면 테스트별로 작은 범위에서 mock을 만들고, 전역 객체를 바꾸는 방식은 최소화하는 것이 좋습니다.
시간 자체를 제어해야 한다면 mock function보다 mock timers가 더 적합합니다.

## 실무에서 자주 하는 실수

### 구현 세부사항까지 과하게 검증한다

mock 테스트는 호출 기록을 볼 수 있기 때문에 내부 구현까지 쉽게 고정합니다.
하지만 모든 private helper 호출을 검증하면 리팩터링 때 테스트가 불필요하게 깨집니다.
검증 대상은 외부로 드러나는 계약에 가까워야 합니다.

좋은 기준은 다음과 같습니다.

- 외부 API, 저장소, 큐, 메일처럼 경계 밖으로 나가는 호출인가?
- 잘못 호출되면 비용, 데이터 오염, 장애로 이어지는가?
- 반환값 검증만으로는 중요한 부작용을 확인할 수 없는가?

이 질문에 해당하지 않는 내부 함수라면 mock보다 입력과 출력 중심 테스트가 더 낫습니다.

### mock 데이터가 실제 계약과 멀어진다

mock 응답이 실제 API 응답과 달라지면 테스트는 통과하지만 운영에서 실패할 수 있습니다.
예를 들어 결제 API가 `payment_id`를 반환하는데 mock만 `id`를 반환하도록 만들면 어댑터의 변환 버그를 놓칠 수 있습니다.

이 문제를 줄이려면 외부 응답 샘플을 작은 fixture로 관리하고, 민감정보는 마스킹해야 합니다.
로그와 예제 데이터를 안전하게 정리하는 기준은 [CLI 출력 sanitizing 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)를 그대로 적용할 수 있습니다.
개인정보, 토큰, 계정 식별자는 테스트 fixture에도 남기지 않는 편이 안전합니다.

## CI에서 mock 테스트를 안정화하는 체크리스트

### 빠른 단위 테스트와 느린 통합 테스트를 분리한다

mock 기반 테스트는 빠르게 자주 실행하는 것이 목적입니다.
반대로 실제 데이터베이스, 메시지 브로커, 외부 API sandbox를 사용하는 테스트는 통합 테스트로 분리하는 편이 좋습니다.

```bash
node --test test/unit/**/*.test.js
node --test test/integration/**/*.test.js
```

CI에서는 pull request마다 unit 테스트를 먼저 실행하고, 통합 테스트는 병렬 또는 후속 단계로 분리할 수 있습니다.
실패 원인이 빨리 드러나야 개발자가 테스트를 신뢰합니다.

### 호출 인자는 부분 검증으로 안정화한다

외부 요청 객체가 커질수록 전체 객체를 매번 `deepEqual`로 검증하면 작은 필드 추가에도 테스트가 깨집니다.
핵심 필드만 검증하고 나머지는 계약 테스트나 통합 테스트에서 다루는 편이 유지보수에 좋습니다.

```js
import assert from 'node:assert/strict';

const request = {
  orderId: 'order_1',
  amount: 30000,
  metadata: {
    traceId: 'trace-test',
    source: 'unit-test',
  },
};

assert.partialDeepStrictEqual(request, {
  orderId: 'order_1',
  amount: 30000,
});
```

핵심 계약만 작게 검증하면 테스트가 구현 변화에 덜 흔들립니다.
동시에 중요한 필드 누락은 놓치지 않습니다.

## Node.js mock.fn 적용 체크리스트

### 새 테스트를 추가하기 전 확인할 것

- 테스트 대상 함수가 외부 의존성을 인자로 받을 수 있는가?
- 실제 네트워크, 메일, 큐, 데이터베이스 호출을 단위 테스트에서 제거했는가?
- 호출 횟수와 핵심 인자만 검증하고 있는가?
- 테스트마다 새 mock 인스턴스를 만들었는가?
- mock fixture에 토큰, 이메일, 내부 호스트명 같은 민감정보가 없는가?

### 운영 코드 구조와 함께 개선할 것

`mock.fn`을 잘 쓰려면 테스트 코드만 바꾸는 것으로는 부족합니다.
운영 코드도 의존성 주입이 가능한 구조여야 하고, 외부 호출 경계가 명확해야 합니다.
HTTP 요청 취소와 재시도까지 다루는 서비스라면 [Node.js fetch timeout 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)를 함께 연결해 외부 API 어댑터를 작게 유지하는 것이 좋습니다.

## FAQ

### mock.fn은 Jest mock과 같은 역할인가요?

큰 방향은 비슷합니다.
함수 호출을 기록하고 원하는 구현을 주입한다는 점에서는 같은 문제를 풉니다.
다만 Node.js 내장 test runner의 API와 Jest API는 이름과 세부 동작이 다르므로 그대로 섞어 쓰기보다 한 테스트 파일 안에서는 한 가지 스타일을 유지하는 것이 좋습니다.

### 모든 외부 의존성을 mock으로만 테스트해도 되나요?

아니요.
단위 테스트에서는 mock으로 빠르게 검증하고, 실제 연결과 스키마 호환성은 통합 테스트나 계약 테스트로 확인해야 합니다.
mock은 외부 시스템을 대체하는 완전한 보증이 아니라, 특정 코드 경로를 빠르게 검증하기 위한 도구입니다.

### mock 호출 인자를 어디까지 검증해야 하나요?

장애나 데이터 오류로 이어지는 핵심 필드는 명시적으로 검증하는 것이 좋습니다.
반대로 단순한 부가 메타데이터나 내부 구현에 가까운 필드는 과하게 고정하지 않는 편이 리팩터링에 유리합니다.

## 마무리

Node.js `mock.fn`은 작은 기능처럼 보이지만, 외부 의존성이 많은 서비스에서 테스트 신뢰도를 크게 올려 줍니다.
핵심은 mock을 많이 쓰는 것이 아니라 경계 밖 호출만 작게 고립하고, 호출 횟수와 핵심 인자를 재현 가능하게 검증하는 것입니다.

테스트가 빨라지고 실패 원인이 선명해지면 배포 전 피드백 루프도 짧아집니다.
새로운 서비스 코드를 작성할 때는 의존성을 직접 생성하기보다 주입 가능한 구조로 만들고, `mock.fn`으로 핵심 계약을 먼저 고정해 보세요.

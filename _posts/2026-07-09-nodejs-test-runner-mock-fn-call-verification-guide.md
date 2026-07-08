---
layout: post
title: "Node.js test runner mock.fn 가이드: 함수 호출과 인자를 검증하는 법"
date: 2026-07-09 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-mock-fn-call-verification-guide
permalink: /development/blog/seo/2026/07/09/nodejs-test-runner-mock-fn-call-verification-guide.html
alternates:
  ko: /development/blog/seo/2026/07/09/nodejs-test-runner-mock-fn-call-verification-guide.html
  x_default: /development/blog/seo/2026/07/09/nodejs-test-runner-mock-fn-call-verification-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, mock, mock-fn, testing, javascript, unit-test, ci]
description: "Node.js test runner의 mock.fn으로 함수 호출 횟수, 인자, 반환값, 비동기 콜백을 검증하는 방법을 정리합니다. t.mock 범위, mock.calls 사용법, 과도한 구현 검증을 피하는 기준을 예제로 설명합니다."
---

단위 테스트에서 중요한 것은 결과값만이 아닙니다.
이벤트를 발행했는지, 로거가 필요한 컨텍스트를 받았는지, retry 함수가 예상 횟수만큼 호출됐는지처럼 "어떤 함수를 어떻게 불렀는가"를 확인해야 할 때가 있습니다.
Node.js 내장 test runner의 `mock.fn()`은 이런 호출 검증을 외부 테스트 프레임워크 없이 처리할 수 있게 해 줍니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#mocking)는 `node:test` 모듈이 top-level `mock` 객체를 통해 mocking 기능을 제공한다고 설명합니다.
또한 각 테스트 컨텍스트에도 `t.mock`이 있어 테스트별 mock 생명주기를 더 좁게 관리할 수 있습니다.
이 글에서는 `mock.fn()`으로 spy 함수를 만들고, 호출 횟수와 인자를 검증하며, 테스트가 구현 세부사항에 너무 묶이지 않도록 기준을 정리합니다.

테스트 timeout과 취소 정리는 [Node.js test runner timeout 가이드](/development/blog/seo/2026/07/07/nodejs-test-runner-timeout-hanging-test-guide.html)와 [Node.js test runner signal 가이드](/development/blog/seo/2026/07/08/nodejs-test-runner-signal-abort-cleanup-guide.html)를 함께 참고하세요.
시간 의존 테스트를 고정하는 방법은 [Node.js test runner mock timers 가이드](/development/blog/seo/2026/07/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html)에서 다뤘습니다.
테스트 파일 격리 기준은 [Node.js test runner isolation 가이드](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html)와 이어집니다.

## mock.fn이 필요한 순간

### H3. 결과보다 상호작용이 더 중요한 코드를 검증한다

`mock.fn()`은 호출 가능한 함수를 만들고, 그 함수가 어떻게 호출됐는지 기록합니다.
테스트 대상 코드가 콜백, 이벤트 발행기, 알림 전송 함수, 로거처럼 외부 동작을 위임한다면 반환값만으로는 충분하지 않을 수 있습니다.
이때 mock 함수는 "이 의존성이 실제로 호출됐는가"를 확인하는 관찰 지점이 됩니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function publishUserCreated(user, publish) {
  publish('user.created', {
    id: user.id,
    email: user.email
  });
}

test('publishes user.created event', (t) => {
  const publish = t.mock.fn();

  publishUserCreated({
    id: 'user_123',
    email: 'dev@example.com'
  }, publish);

  assert.equal(publish.mock.callCount(), 1);
  assert.deepEqual(publish.mock.calls[0].arguments, [
    'user.created',
    {
      id: 'user_123',
      email: 'dev@example.com'
    }
  ]);
});
```

이 예시는 실제 메시지 브로커를 호출하지 않습니다.
대신 이벤트 이름과 payload가 올바른지 검증합니다.
단위 테스트에서는 네트워크나 큐 같은 느린 의존성을 빼고, 코드가 외부 경계에 어떤 요청을 보냈는지만 확인하는 편이 안정적입니다.

### H3. spy와 stub의 역할을 구분한다

`mock.fn()`을 구현 없이 만들면 호출 기록만 남기는 spy처럼 쓸 수 있습니다.
구현 함수를 넘기면 실제 반환값을 제어하는 stub처럼 사용할 수 있습니다.
테스트가 무엇을 검증하려는지에 따라 둘 중 하나를 선택하면 됩니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function calculateDiscount(total, readRate) {
  const rate = readRate(total);
  return Math.round(total * rate);
}

test('uses configured discount rate', (t) => {
  const readRate = t.mock.fn((total) => {
    return total >= 100_000 ? 0.1 : 0.03;
  });

  const discount = calculateDiscount(120_000, readRate);

  assert.equal(discount, 12_000);
  assert.equal(readRate.mock.callCount(), 1);
  assert.deepEqual(readRate.mock.calls[0].arguments, [120_000]);
});
```

여기서는 `readRate`가 fake 구현을 갖습니다.
테스트 대상 함수는 설정 저장소나 원격 API를 몰라도 할인 계산을 검증할 수 있습니다.
반환값이 필요한 의존성은 구현을 넣고, 호출 사실만 필요한 의존성은 빈 mock으로 두면 테스트 의도가 더 분명해집니다.

## 호출 횟수와 인자 검증하기

### H3. callCount로 호출 계약을 좁힌다

호출 횟수 검증은 retry, fallback, idempotency 테스트에서 특히 유용합니다.
예를 들어 첫 번째 전송이 실패하고 두 번째 전송이 성공해야 한다면, 전송 함수가 정확히 두 번 호출됐는지 확인해야 합니다.
이 검증이 없으면 코드가 불필요하게 더 많이 호출돼도 결과값만 보고 통과할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

async function sendWithRetry(payload, send) {
  try {
    return await send(payload);
  } catch {
    return send(payload);
  }
}

test('retries once after transient failure', async (t) => {
  let attempt = 0;

  const send = t.mock.fn(async (payload) => {
    attempt += 1;

    if (attempt === 1) {
      throw new Error('temporary unavailable');
    }

    return { ok: true, id: payload.id };
  });

  const result = await sendWithRetry({ id: 'job_123' }, send);

  assert.deepEqual(result, { ok: true, id: 'job_123' });
  assert.equal(send.mock.callCount(), 2);
});
```

이 테스트는 retry가 한 번만 발생한다는 계약을 검증합니다.
운영 코드에서 retry 횟수는 비용, 지연, 중복 처리와 직접 연결되므로 결과값보다 호출 횟수가 더 중요한 신호가 될 수 있습니다.
다만 호출 횟수를 과하게 검증하면 리팩터링에 약해지므로 외부 부작용이 있는 경계에 우선 적용하는 것이 좋습니다.

### H3. calls 배열로 각 호출의 인자를 확인한다

`mock.calls`에는 호출별 정보가 순서대로 쌓입니다.
각 항목의 `arguments`를 확인하면 첫 번째 호출과 두 번째 호출의 인자가 달랐는지 검증할 수 있습니다.
배치 처리나 단계별 상태 전이를 테스트할 때 유용합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function flushMetrics(metrics, write) {
  for (const metric of metrics) {
    write(metric.name, metric.value);
  }
}

test('writes metrics in order', (t) => {
  const write = t.mock.fn();

  flushMetrics([
    { name: 'queue.depth', value: 12 },
    { name: 'job.failed', value: 1 }
  ], write);

  assert.equal(write.mock.callCount(), 2);
  assert.deepEqual(write.mock.calls.map((call) => call.arguments), [
    ['queue.depth', 12],
    ['job.failed', 1]
  ]);
});
```

순서가 의미 있는 코드에서는 인자 배열을 모아 한 번에 비교하면 읽기 쉽습니다.
반대로 순서가 의미 없다면 정렬하거나 `Set`으로 바꿔 검증하는 편이 테스트를 덜 취약하게 만듭니다.
테스트가 실제 요구사항의 순서를 검증하는지, 우연한 구현 순서를 검증하는지 먼저 구분해야 합니다.

## t.mock으로 테스트 범위 관리하기

### H3. 테스트별 mock tracker를 우선 사용한다

`node:test`는 top-level `mock` export도 제공하지만, 일반적인 테스트에서는 `t.mock`을 우선 쓰는 편이 좋습니다.
테스트 컨텍스트에 속한 mock은 테스트 단위로 관리하기 쉬워서 병렬 실행과 격리 관점에서 더 안전합니다.
특히 같은 파일 안에서 여러 테스트가 서로 다른 mock 구현을 가져야 한다면 `t.mock`이 의도를 더 잘 드러냅니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function createWelcomeMessage(user, now) {
  return {
    message: `hello ${user.name}`,
    createdAt: now().toISOString()
  };
}

test('creates deterministic welcome message', (t) => {
  const now = t.mock.fn(() => new Date('2026-07-09T00:00:00.000Z'));

  const result = createWelcomeMessage({ name: 'sooya' }, now);

  assert.deepEqual(result, {
    message: 'hello sooya',
    createdAt: '2026-07-09T00:00:00.000Z'
  });
  assert.equal(now.mock.callCount(), 1);
});
```

이 방식은 날짜를 고정하면서도 전역 `Date`를 바꾸지 않습니다.
전역 상태를 건드리지 않는 테스트는 순서가 바뀌거나 병렬로 실행돼도 깨질 가능성이 낮습니다.
mock 함수는 가능하면 의존성 주입 형태로 넘기고, 전역 monkey patch는 마지막 선택지로 남겨두세요.

### H3. mock.method가 필요한 경우와 구분한다

`mock.fn()`은 새 함수를 만들어 넘길 수 있을 때 가장 단순합니다.
이미 존재하는 객체의 메서드를 잠깐 바꿔야 한다면 `mock.method()`가 더 적합합니다.
즉 함수가 인자로 주입되는 구조라면 `mock.fn()`, 객체 메서드를 감시하거나 대체해야 한다면 `mock.method()`를 고려하면 됩니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function saveAuditLog(auditLogger, event) {
  auditLogger.write({
    type: event.type,
    userId: event.userId
  });
}

test('writes audit log through injected logger', (t) => {
  const auditLogger = {
    write: t.mock.fn()
  };

  saveAuditLog(auditLogger, {
    type: 'LOGIN',
    userId: 'user_123'
  });

  assert.equal(auditLogger.write.mock.callCount(), 1);
  assert.deepEqual(auditLogger.write.mock.calls[0].arguments, [
    {
      type: 'LOGIN',
      userId: 'user_123'
    }
  ]);
});
```

이 예시는 객체 자체를 새로 만들 수 있으므로 `mock.fn()`만으로 충분합니다.
반면 실제 모듈이나 기존 인스턴스의 메서드를 바꿔야 한다면 복원까지 고려해야 합니다.
테스트 설계 관점에서는 의존성을 주입할 수 있는 작은 함수가 mock도 단순하게 만듭니다.

## 비동기 콜백과 에러 경로 테스트

### H3. async mock은 await로 결과를 검증한다

mock 구현은 비동기 함수가 될 수 있습니다.
이 경우 테스트 대상 코드가 `await`을 제대로 하는지, 실패 경로를 어떻게 처리하는지 함께 확인할 수 있습니다.
비동기 mock을 쓸 때는 호출 횟수뿐 아니라 최종 결과나 예외도 같이 검증해야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

async function notifyUser(user, sendEmail) {
  await sendEmail(user.email, 'Welcome');
  return { notified: true };
}

test('awaits email delivery', async (t) => {
  const sendEmail = t.mock.fn(async (email, subject) => {
    assert.equal(email, 'dev@example.com');
    assert.equal(subject, 'Welcome');
  });

  const result = await notifyUser({
    email: 'dev@example.com'
  }, sendEmail);

  assert.deepEqual(result, { notified: true });
  assert.equal(sendEmail.mock.callCount(), 1);
});
```

mock 내부에 assertion을 넣을 수는 있지만, 너무 많은 검증을 mock 구현 안에 숨기면 테스트 흐름이 읽기 어려워집니다.
간단한 guard 정도는 괜찮지만 핵심 검증은 테스트 마지막에 모아두는 편이 유지보수에 좋습니다.
특히 호출 인자는 `mock.calls`로 확인하면 실패 메시지를 더 명확하게 만들 수 있습니다.

### H3. 실패 경로에서는 호출되지 않아야 할 함수도 검증한다

좋은 테스트는 "호출됐다"뿐 아니라 "호출되지 않았다"도 확인합니다.
검증 실패, 권한 부족, 입력 거부 같은 경로에서는 외부 부작용이 없어야 합니다.
이런 코드는 `callCount()`가 0인지 확인하는 것이 핵심 요구사항입니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function createOrder(input, chargePayment) {
  if (!input.userId) {
    throw new Error('userId is required');
  }

  chargePayment(input.amount);
  return { ok: true };
}

test('does not charge payment for invalid order', (t) => {
  const chargePayment = t.mock.fn();

  assert.throws(() => {
    createOrder({ amount: 30_000 }, chargePayment);
  }, /userId is required/);

  assert.equal(chargePayment.mock.callCount(), 0);
});
```

결제, 이메일, 메시지 발행처럼 되돌리기 어려운 동작은 실패 경로에서 호출되지 않는지 반드시 확인하는 편이 좋습니다.
이 검증은 테스트를 더 장황하게 만들지만, 실제 장애 비용을 줄이는 데 도움이 됩니다.
성공 경로보다 실패 경로의 부작용 방지가 더 중요한 경우도 많습니다.

## 과도한 구현 검증을 피하는 기준

### H3. 외부 경계에 가까운 호출만 자세히 본다

mock 호출 검증은 강력하지만 남용하면 리팩터링을 어렵게 만듭니다.
내부 helper가 몇 번 호출됐는지까지 전부 검증하면 코드 구조를 조금만 바꿔도 테스트가 깨집니다.
호출 검증은 외부 API, 큐, 메일, 결제, 로그, metric처럼 부작용이나 비용이 있는 경계에 우선 적용하세요.

```text
좋은 호출 검증 후보:
- payment.charge가 실패 입력에서 호출되지 않아야 한다
- publish가 정확한 event name과 payload를 받아야 한다
- retry 가능한 요청이 최대 2번까지만 호출돼야 한다
- audit log가 사용자 id와 action을 포함해야 한다

피하는 편이 좋은 검증:
- 내부 format 함수가 정확히 한 번 호출된다
- private helper 호출 순서가 현재 구현과 같다
- 단순 계산 중간 단계가 모두 mock으로 대체된다
```

테스트는 구현이 아니라 관찰 가능한 계약을 보호해야 합니다.
함수 호출 자체가 제품 요구사항이나 운영 안정성과 연결될 때만 자세히 검증하는 것이 좋습니다.
그 외에는 입력과 출력 중심 테스트가 더 오래 갑니다.

### H3. mock보다 작은 순수 함수를 먼저 고려한다

mock이 많아지는 코드는 의존성이 복잡하다는 신호일 수 있습니다.
계산, 변환, 필터링 같은 로직은 순수 함수로 분리하면 mock 없이도 쉽게 테스트할 수 있습니다.
mock은 외부 경계와 비결정적 동작을 다룰 때 쓰고, 순수 로직은 값 비교로 검증하는 편이 단순합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function buildAuditPayload(event) {
  return {
    type: event.type,
    userId: event.userId,
    source: 'web'
  };
}

function writeAudit(event, write) {
  write(buildAuditPayload(event));
}

test('builds audit payload without mocks', () => {
  assert.deepEqual(buildAuditPayload({
    type: 'LOGOUT',
    userId: 'user_123'
  }), {
    type: 'LOGOUT',
    userId: 'user_123',
    source: 'web'
  });
});
```

이렇게 payload 생성은 순수 함수로 검증하고, `writeAudit()`에서는 write 함수가 한 번 호출되는지만 확인하면 됩니다.
mock은 줄어들고 테스트 실패 원인은 더 분명해집니다.
테스트하기 어려운 코드를 무리하게 mock으로 덮기보다, 테스트 가능한 경계로 코드를 나누는 것이 먼저입니다.

## 실무 적용 체크리스트

### H3. mock.fn 테스트를 작성하기 전에 확인할 것

`mock.fn()`을 쓰기 전에 이 호출 검증이 실제 요구사항을 보호하는지 확인하세요.
외부 부작용, retry 횟수, 이벤트 payload, 실패 시 미호출 보장처럼 운영상 의미가 있다면 좋은 후보입니다.
반대로 내부 구현 순서를 고정하는 목적이라면 값 중심 테스트로 바꿀 수 있는지 먼저 보는 편이 낫습니다.

```text
checklist:
- 테스트 대상 함수가 의존성을 인자로 받을 수 있는가?
- 호출 횟수가 실제 요구사항과 연결되는가?
- 인자 검증이 개인정보나 민감정보를 노출하지 않는가?
- 실패 경로에서 호출되지 않아야 할 함수를 확인했는가?
- mock 없이 순수 함수 테스트로 분리할 부분은 없는가?
```

테스트 double은 코드를 빠르고 안정적으로 검증하게 해 주지만, 설계를 대신해 주지는 않습니다.
의존성을 명확히 주입하고, 부작용 있는 경계에만 호출 검증을 집중하면 테스트는 훨씬 덜 흔들립니다.
Node.js test runner의 `mock.fn()`은 그 기준 안에서 사용할 때 가장 실용적입니다.


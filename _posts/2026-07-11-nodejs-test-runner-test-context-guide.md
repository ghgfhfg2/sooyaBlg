---
layout: post
title: "Node.js test runner test context 가이드: t 객체로 테스트를 더 안전하게 관리하는 법"
date: 2026-07-11 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-test-context-guide
permalink: /development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html
alternates:
  ko: /development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html
  x_default: /development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, test-context, testing, javascript, unit-test, mock, ci]
description: "Node.js test runner의 test context t 객체로 테스트 이름, 진단 로그, 하위 테스트, mock과 cleanup 흐름을 관리하는 방법을 정리합니다. t.test, t.diagnostic, t.mock 사용 기준과 CI에서 읽기 좋은 테스트 작성 패턴을 예제로 설명합니다."
---

Node.js 내장 test runner를 쓰다 보면 `test()` 콜백에 전달되는 `t` 객체를 그냥 지나치기 쉽습니다.
작은 단위 테스트에서는 `assert`만으로도 충분해 보이지만, 테스트가 늘어나면 하위 테스트를 묶고, 실패 원인을 남기고, mock을 테스트 범위 안에서 관리하는 기준이 필요해집니다.
이때 test context인 `t` 객체를 활용하면 테스트 파일이 더 명확해지고 CI 리포트도 읽기 쉬워집니다.

`t`는 현재 실행 중인 테스트의 문맥입니다.
하위 테스트를 만들고, 진단 메시지를 출력하고, mock tracker에 접근하고, 테스트별 실행 흐름을 좁히는 데 사용합니다.
전역 helper를 늘리는 대신 테스트가 필요한 정보를 자기 문맥 안에 갖게 만들면, 병렬 실행과 실패 분석이 훨씬 단순해집니다.

이 글에서는 Node.js test runner의 test context를 언제 쓰면 좋은지, `t.test()`, `t.diagnostic()`, `t.mock`을 어떤 기준으로 적용할지 실무 예제로 정리합니다.
테스트 스위트 구조는 [Node.js test runner describe/it 가이드](/development/blog/seo/2026/07/10/nodejs-test-runner-describe-it-suite-structure-guide.html)를 함께 참고하세요.
mock 복원 기준은 [Node.js test runner mock.method restore 가이드](/development/blog/seo/2026/07/09/nodejs-test-runner-mock-method-restore-guide.html), 테스트별 준비와 정리는 [Node.js test runner beforeEach/afterEach 가이드](/development/blog/seo/2026/07/10/nodejs-test-runner-beforeeach-aftereach-hooks-guide.html)와 이어집니다.

## test context가 해결하는 문제

### H3. 테스트의 실행 문맥을 코드에 드러낸다

Node.js test runner의 `test()` 콜백은 첫 번째 인자로 test context를 받을 수 있습니다.
보통 변수 이름은 짧게 `t`를 사용합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('normalizes an email address', (t) => {
  t.diagnostic('input includes surrounding whitespace and mixed case');

  const result = ' USER@example.COM '.trim().toLowerCase();

  assert.equal(result, 'user@example.com');
});
```

`t.diagnostic()`은 테스트 결과 스트림에 보조 정보를 남깁니다.
일반적인 성공 경로에서는 꼭 필요하지 않지만, 실패했을 때 입력 조건이나 외부 상태를 확인해야 하는 테스트라면 유용합니다.

중요한 점은 `console.log()` 대신 테스트 문맥에 맞는 진단 로그를 쓰는 것입니다.
테스트 리포터가 해당 메시지를 테스트 결과와 함께 다룰 수 있고, CI 로그에서도 어떤 테스트에서 나온 메시지인지 추적하기 쉽습니다.

### H3. 전역 상태보다 테스트 범위를 우선한다

테스트 코드가 커질수록 전역 helper, 파일 상단 변수, 공유 mock이 늘어나기 쉽습니다.
하지만 이런 방식은 병렬 실행, 재시도, 부분 실행에서 예상 밖의 상태 공유를 만들 수 있습니다.

test context는 현재 테스트와 관련된 동작을 그 테스트 안에 붙잡아 두는 도구입니다.
하위 테스트, mock, 진단 메시지를 `t`를 통해 관리하면 테스트가 실패했을 때 확인해야 할 범위가 줄어듭니다.

```js
test('creates a user profile', async (t) => {
  t.diagnostic('repository is replaced with an in-memory test double');

  const saved = [];
  const repository = {
    async save(user) {
      saved.push(user);
      return { ...user, id: 'user_123' };
    }
  };

  const result = await repository.save({ email: 'user@example.com' });

  assert.equal(result.id, 'user_123');
  assert.equal(saved.length, 1);
});
```

이 예제는 단순하지만 의도는 분명합니다.
테스트에 필요한 저장소 대역과 검증 대상이 테스트 내부에 있고, 진단 메시지도 같은 문맥에 있습니다.
파일의 다른 테스트가 추가되더라도 이 상태가 섞일 여지가 줄어듭니다.

## t.test로 하위 테스트를 구성하는 법

### H3. 한 흐름 안의 조건을 묶을 때 사용한다

`t.test()`는 현재 테스트 아래에 하위 테스트를 만듭니다.
`describe()`와 `it()`으로 파일 전체 구조를 잡을 수도 있지만, 하나의 큰 사용자 흐름 안에서 작은 조건을 순서대로 검증하고 싶을 때 `t.test()`가 잘 맞습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function calculateShipping({ country, total }) {
  if (country !== 'KR') return 15_000;
  if (total >= 50_000) return 0;
  return 3_000;
}

test('calculateShipping', async (t) => {
  await t.test('returns free shipping for domestic orders over threshold', () => {
    assert.equal(calculateShipping({ country: 'KR', total: 60_000 }), 0);
  });

  await t.test('charges domestic shipping below threshold', () => {
    assert.equal(calculateShipping({ country: 'KR', total: 30_000 }), 3_000);
  });

  await t.test('charges international shipping separately', () => {
    assert.equal(calculateShipping({ country: 'US', total: 80_000 }), 15_000);
  });
});
```

하위 테스트를 `await`하면 부모 테스트가 자식 테스트의 완료를 기다립니다.
비동기 테스트에서는 이 습관이 중요합니다.
기다리지 않은 하위 테스트는 리포트 순서와 실패 처리가 의도와 달라질 수 있습니다.

### H3. 너무 깊은 중첩은 피한다

`t.test()`도 중첩할 수 있지만, 테스트 구조가 너무 깊어지면 실패 리포트를 읽기 어려워집니다.
대부분의 경우 부모 테스트 하나와 하위 테스트 한 단계면 충분합니다.

```js
test('checkout discount policy', async (t) => {
  await t.test('applies welcome coupon before membership discount', () => {
    // ...
  });

  await t.test('ignores expired coupon codes', () => {
    // ...
  });
});
```

하위 테스트 이름은 부모 이름을 반복하지 않는 편이 좋습니다.
`checkout discount policy applies...`처럼 모든 이름에 같은 문맥을 붙이면 리포트가 길어집니다.
부모 테스트가 문맥을 맡고, 자식 테스트는 조건과 기대 결과를 짧게 표현하면 됩니다.

## t.diagnostic으로 실패 분석 정보를 남기기

### H3. 실패했을 때 필요한 정보만 남긴다

진단 로그는 많을수록 좋은 정보가 아닙니다.
항상 같은 값을 반복하거나, 테스트 성공에도 의미 없는 로그를 많이 남기면 CI 출력이 금방 시끄러워집니다.

좋은 진단 메시지는 실패 원인을 좁히는 데 직접 도움을 줍니다.
예를 들어 랜덤 seed, 테스트용 설정 값, 외부 API를 대체한 fake 이름, fixture 버전 같은 정보가 여기에 해당합니다.

```js
test('selects a rollout variant', (t) => {
  const seed = 'user_42';
  const rolloutRate = 0.25;

  t.diagnostic(`seed=${seed}, rolloutRate=${rolloutRate}`);

  const bucket = hashToBucket(seed);

  assert.equal(bucket >= 0 && bucket < 1, true);
});

function hashToBucket(value) {
  let hash = 0;

  for (const character of value) {
    hash = (hash * 31 + character.charCodeAt(0)) % 1000;
  }

  return hash / 1000;
}
```

이런 메시지는 테스트가 실패했을 때 재현 단서를 제공합니다.
반대로 민감정보, 실제 사용자 이메일, 토큰, 세션 값은 진단 로그에 남기면 안 됩니다.
테스트 로그도 운영 로그처럼 보관되거나 외부 시스템으로 전송될 수 있기 때문입니다.

### H3. console.log를 기본 디버깅 수단으로 두지 않는다

임시 디버깅 중에는 `console.log()`를 넣을 수 있습니다.
하지만 커밋되는 테스트 코드에는 가능하면 `t.diagnostic()`을 사용하세요.

```js
test('parses a webhook event', (t) => {
  const event = parseWebhookEvent('{"type":"payment.succeeded"}');

  t.diagnostic(`event.type=${event.type}`);

  assert.equal(event.type, 'payment.succeeded');
});

function parseWebhookEvent(payload) {
  return JSON.parse(payload);
}
```

`console.log()`는 테스트 리포터와 분리된 일반 출력입니다.
여러 테스트가 동시에 실행되면 메시지의 출처를 구분하기 어렵습니다.
`t.diagnostic()`을 쓰면 메시지가 테스트 결과와 더 가깝게 묶입니다.

## t.mock을 테스트 범위 안에서 사용하기

### H3. mock tracker를 현재 테스트에 묶는다

Node.js test runner는 mock 기능을 제공합니다.
test context의 `t.mock`을 사용하면 mock을 현재 테스트 문맥에서 다루는 의도가 분명해집니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('calls the notifier after signup', (t) => {
  const notifier = {
    sendWelcomeEmail(email) {
      return `sent:${email}`;
    }
  };

  const sendWelcomeEmail = t.mock.method(notifier, 'sendWelcomeEmail');

  const result = notifier.sendWelcomeEmail('user@example.com');

  assert.equal(result, 'sent:user@example.com');
  assert.equal(sendWelcomeEmail.mock.callCount(), 1);
  assert.equal(sendWelcomeEmail.mock.calls[0].arguments[0], 'user@example.com');
});
```

이 예제에서는 실제 메서드 동작을 유지하면서 호출 여부를 검증합니다.
테스트에서 검증하고 싶은 것이 반환값인지, 호출 횟수인지, 호출 인자인지 먼저 정하면 mock의 역할이 과해지지 않습니다.

mock을 복잡하게 만들기 전에 의존성을 함수 인자로 주입할 수 있는지도 확인하세요.
단순한 의존성 주입이 가능한 코드라면 mock보다 테스트 대역 객체가 더 읽기 쉬울 때가 많습니다.

### H3. 복원 기준을 명확히 둔다

메서드 mock은 테스트가 끝난 뒤 원래 구현으로 돌아가야 합니다.
테스트 범위 밖까지 mock이 남아 있으면 이후 테스트가 잘못된 구현을 바라볼 수 있습니다.

```js
test('replaces Date.now for a deterministic timestamp', (t) => {
  t.mock.method(Date, 'now', () => 1_788_710_400_000);

  assert.equal(Date.now(), 1_788_710_400_000);
});
```

현재 테스트 안에서만 mock을 만들고 검증하면 복원 범위를 추적하기 쉽습니다.
여러 테스트가 같은 전역 객체를 mock해야 한다면 병렬 실행과 순서 의존성도 함께 점검해야 합니다.
전역 메서드를 다루는 자세한 기준은 [Node.js test runner mock.method restore 가이드](/development/blog/seo/2026/07/09/nodejs-test-runner-mock-method-restore-guide.html)를 참고하면 좋습니다.

## context 기반 테스트 작성 패턴

### H3. Arrange, Act, Assert를 문맥 안에 유지한다

test context를 쓴다고 해서 테스트 구조가 복잡해질 필요는 없습니다.
오히려 준비, 실행, 검증을 한 테스트 안에 더 분명히 배치하는 것이 좋습니다.

```js
test('rejects a duplicate idempotency key', async (t) => {
  const requests = new Set(['key_123']);

  t.diagnostic('existing key: key_123');

  async function reserveKey(key) {
    if (requests.has(key)) {
      return { ok: false, reason: 'duplicate' };
    }

    requests.add(key);
    return { ok: true };
  }

  const result = await reserveKey('key_123');

  assert.deepEqual(result, { ok: false, reason: 'duplicate' });
});
```

테스트를 읽는 사람은 위에서 아래로 준비 조건, 진단 정보, 실행 결과를 확인할 수 있습니다.
테스트 helper를 만들 때도 이 흐름을 깨지 않는 선에서만 추출하는 편이 좋습니다.

### H3. 반복 조건은 하위 테스트로 표처럼 표현한다

여러 입력과 기대 결과를 반복 검증해야 한다면 `for...of`와 `t.test()`를 함께 사용할 수 있습니다.
각 케이스가 독립적인 하위 테스트로 표시되기 때문에 실패한 입력을 찾기 쉽습니다.

```js
test('validates usernames', async (t) => {
  const cases = [
    { value: 'sooya', expected: true },
    { value: 'soo', expected: true },
    { value: 'x', expected: false },
    { value: 'name with space', expected: false }
  ];

  for (const testCase of cases) {
    await t.test(`value=${JSON.stringify(testCase.value)}`, () => {
      assert.equal(isValidUsername(testCase.value), testCase.expected);
    });
  }
});

function isValidUsername(value) {
  return /^[a-z0-9_]{2,20}$/.test(value);
}
```

테스트 이름에 입력값을 넣을 때는 민감정보가 아닌지 확인해야 합니다.
사용자 이메일, 전화번호, 토큰, 실제 주문 번호 같은 값은 마스킹하거나 의미 있는 별칭으로 바꾸는 편이 안전합니다.

## CI에서 읽기 좋은 test context 운영 기준

### H3. 리포트 검색 키워드를 먼저 정한다

CI에서 테스트가 실패했을 때 가장 먼저 하는 일은 로그 검색입니다.
따라서 테스트 이름과 진단 메시지에는 팀이 실제로 검색할 키워드가 들어가야 합니다.

예를 들어 결제 서비스라면 `payment`, `refund`, `authorization`, `idempotency` 같은 도메인 단어가 좋습니다.
반대로 `case 1`, `happy path`, `util test`처럼 문맥이 약한 이름은 실패 원인을 좁히는 데 도움이 적습니다.

```js
test('payment authorization keeps cart when gateway declines', async (t) => {
  t.diagnostic('gateway fake returns decline code');

  // ...
});
```

이름이 길어지는 것이 항상 나쁜 것은 아닙니다.
다만 이름이 길다면 중복 문맥을 줄이고, 조건과 기대 결과가 드러나는지 확인해야 합니다.

### H3. 병렬 실행에서 공유 상태를 의심한다

test context를 잘 써도 전역 상태를 많이 바꾸는 테스트는 병렬 실행에서 흔들릴 수 있습니다.
특히 `Date`, `process.env`, singleton client, module cache, 임시 파일 경로를 바꾸는 테스트는 주의가 필요합니다.

공유 상태를 만지는 테스트는 다음 기준으로 점검하세요.

- mock이나 환경 변수 변경이 테스트 안에서 끝나는가
- 임시 파일과 포트 번호가 테스트마다 분리되는가
- 하위 테스트를 `await`해 완료 순서가 명확한가
- 진단 로그에 재현 가능한 입력이 남아 있는가
- 실제 비밀 값이나 사용자 데이터가 로그에 섞이지 않았는가

테스트 격리 전략은 [Node.js test runner isolation 가이드](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html), CI 병렬 실행은 [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html)와 함께 보면 흐름이 이어집니다.

## 실무 체크리스트

### H3. t 객체를 도입하기 전 확인할 것

test context는 모든 테스트에 억지로 넣을 필요가 없습니다.
단순한 검증 하나로 끝나는 테스트라면 `test('name', () => {})`만으로도 충분합니다.
다음 조건 중 하나가 있을 때 `t`를 적극적으로 도입하세요.

- 하위 테스트로 여러 조건을 독립 리포트하고 싶다
- 실패했을 때 재현 단서를 남겨야 한다
- mock 호출 횟수나 인자를 현재 테스트 범위에서 검증한다
- 반복 케이스를 각 테스트 결과로 분리하고 싶다
- 병렬 실행에서 테스트 문맥을 더 명확히 드러내고 싶다

도구를 쓰는 목적은 테스트 코드를 화려하게 만드는 것이 아니라 실패 분석 비용을 줄이는 것입니다.
`t`를 추가했는데 테스트가 더 읽기 어려워졌다면 구조를 다시 나누는 편이 낫습니다.

### H3. 발행 전 테스트 코드 리뷰 포인트

커밋 전에 다음 항목을 빠르게 확인하면 test context를 안정적으로 운영할 수 있습니다.

- `await t.test()`를 빠뜨리지 않았는가
- `t.diagnostic()`에 민감정보가 들어가지 않았는가
- mock이 테스트 밖 상태를 오염시키지 않는가
- 하위 테스트 이름만 봐도 실패 조건을 알 수 있는가
- 반복 케이스의 입력값이 안전하고 재현 가능한가
- `console.log()`가 커밋된 테스트 코드에 남아 있지 않은가

Node.js test runner의 test context는 작은 기능처럼 보이지만, 테스트가 많아질수록 효과가 커집니다.
현재 테스트의 문맥을 `t`에 모아 두면 스위트 구조, 진단 로그, mock 관리, 반복 케이스 리포트가 한결 일관됩니다.
테스트 실패를 빨리 읽고 안전하게 고치는 팀일수록 이런 작은 문맥 관리가 장기적인 유지보수 비용을 줄여 줍니다.

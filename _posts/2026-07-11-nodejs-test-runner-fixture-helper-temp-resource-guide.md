---
layout: post
title: "Node.js test runner fixture helper 가이드: 임시 리소스를 안전하게 만들고 정리하는 법"
date: 2026-07-11 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-fixture-helper-temp-resource-guide
permalink: /development/blog/seo/2026/07/11/nodejs-test-runner-fixture-helper-temp-resource-guide.html
alternates:
  ko: /development/blog/seo/2026/07/11/nodejs-test-runner-fixture-helper-temp-resource-guide.html
  x_default: /development/blog/seo/2026/07/11/nodejs-test-runner-fixture-helper-temp-resource-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, fixture, temp-file, cleanup, testing, javascript, ci]
description: "Node.js test runner에서 fixture helper로 임시 디렉터리, 테스트 데이터, fake client 같은 리소스를 만들고 afterEach로 안전하게 정리하는 방법을 정리합니다. 테스트 독립성, cleanup 순서, CI 충돌 방지 기준을 예제로 설명합니다."
---

Node.js 내장 test runner로 테스트를 작성하다 보면 같은 준비 코드가 여러 파일에 반복됩니다.
임시 디렉터리를 만들고, 테스트용 JSON 파일을 쓰고, fake client를 준비하고, 테스트가 끝난 뒤 리소스를 정리하는 흐름입니다.
처음에는 테스트 본문에 직접 써도 되지만, 반복이 늘어나면 검증 의도보다 준비와 정리 코드가 더 크게 보이기 시작합니다.

fixture helper는 이 문제를 줄이는 실용적인 방법입니다.
테스트마다 필요한 리소스를 만들되, 공유 상태를 숨기지 않고, 정리 책임까지 한곳에 묶어 둡니다.
특히 파일 시스템, clock, mock, in-memory store처럼 테스트 간 오염이 생기기 쉬운 리소스는 helper의 범위와 수명을 명확히 해야 합니다.

이 글에서는 Node.js test runner에서 fixture helper를 설계하는 기준과 임시 리소스 정리 패턴을 정리합니다.
테스트별 준비와 정리 hook은 [Node.js test runner beforeEach/afterEach 가이드](/development/blog/seo/2026/07/10/nodejs-test-runner-beforeeach-aftereach-hooks-guide.html)를 먼저 참고하면 좋습니다.
test context의 `t` 객체 활용은 [Node.js test runner test context 가이드](/development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html), 순서 의존 문제를 찾는 방법은 [Node.js test runner randomize/seed 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-randomize-seed-order-dependent-test-guide.html)와 이어집니다.

## fixture helper가 필요한 순간

### H3. 테스트 준비 코드가 본문을 가릴 때

테스트 본문은 가능하면 입력, 실행, 검증이 바로 보이는 구조가 좋습니다.
하지만 fixture 준비 코드가 길어지면 테스트가 실제로 무엇을 검증하는지 읽기 어려워집니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp, readFile, rm, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

test('reads a generated profile', async () => {
  const dir = await mkdtemp(join(tmpdir(), 'profile-test-'));

  try {
    const filePath = join(dir, 'profile.json');
    await writeFile(filePath, JSON.stringify({ id: 'user_1', role: 'admin' }));

    const body = await readFile(filePath, 'utf8');

    assert.deepEqual(JSON.parse(body), { id: 'user_1', role: 'admin' });
  } finally {
    await rm(dir, { recursive: true, force: true });
  }
});
```

이 코드는 안전하지만 테스트 하나 안에 준비, 실행, 정리가 모두 섞여 있습니다.
같은 패턴이 여러 테스트에 반복된다면 helper로 분리할 수 있습니다.
단, helper가 너무 많은 동작을 숨기면 테스트 입력이 보이지 않으므로 반복되는 리소스 생성과 정리만 옮기는 편이 좋습니다.

### H3. 테스트마다 고립된 리소스가 필요할 때

임시 파일, in-memory store, fake server, message queue 대역은 테스트마다 새로 만드는 편이 안전합니다.
하나의 리소스를 여러 테스트가 공유하면 테스트 실행 순서나 병렬성에 따라 결과가 흔들릴 수 있습니다.

fixture helper의 기본 원칙은 단순합니다.
호출할 때마다 새 리소스를 만들고, 정리 함수도 함께 반환합니다.

```js
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

async function createProfileFixture(profile) {
  const dir = await mkdtemp(join(tmpdir(), 'profile-fixture-'));
  const filePath = join(dir, 'profile.json');

  await writeFile(filePath, JSON.stringify(profile));

  return {
    dir,
    filePath,
    async cleanup() {
      await rm(dir, { recursive: true, force: true });
    }
  };
}
```

이 helper는 테스트 데이터를 파일로 만들고, 테스트가 읽을 경로와 정리 함수를 함께 제공합니다.
전역 변수에 경로를 저장하지 않기 때문에 여러 테스트가 동시에 실행되어도 충돌 가능성이 낮습니다.

## helper와 afterEach를 함께 쓰는 법

### H3. cleanup 목록을 테스트별로 관리한다

테스트마다 여러 리소스를 만들 수 있다면 cleanup 함수를 배열로 모아 두는 패턴이 유용합니다.
`afterEach()`는 테스트가 끝날 때마다 실행되므로, 현재 테스트가 만든 리소스만 정리하도록 범위를 좁힐 수 있습니다.

```js
import { afterEach, test } from 'node:test';
import assert from 'node:assert/strict';
import { readFile } from 'node:fs/promises';

const cleanups = [];

afterEach(async () => {
  while (cleanups.length > 0) {
    const cleanup = cleanups.pop();
    await cleanup();
  }
});

test('loads profile fixture', async () => {
  const fixture = await createProfileFixture({ id: 'user_1', role: 'admin' });
  cleanups.push(fixture.cleanup);

  const body = await readFile(fixture.filePath, 'utf8');

  assert.equal(JSON.parse(body).role, 'admin');
});
```

정리 함수는 만든 순서의 반대로 실행하는 편이 안전합니다.
예를 들어 파일, 디렉터리, 서버처럼 의존 관계가 있다면 나중에 만든 리소스부터 닫아야 충돌이 적습니다.
`pop()`을 사용하면 이런 후입선출 정리 흐름을 자연스럽게 만들 수 있습니다.

### H3. 정리 실패가 원래 실패를 덮지 않게 한다

cleanup이 실패하면 테스트 실패 원인을 더 복잡하게 만들 수 있습니다.
그래도 정리 실패를 조용히 삼키면 CI에 찌꺼기가 남아 다음 테스트에 영향을 줄 수 있습니다.
따라서 정리 함수는 가능한 한 멱등적으로 만들고, 이미 삭제된 리소스에는 `force: true` 같은 방어 옵션을 사용합니다.

```js
async function cleanupDirectory(dir) {
  await rm(dir, {
    recursive: true,
    force: true
  });
}
```

정리 함수 안에서 실제 사용자 홈 디렉터리나 프로젝트 루트 같은 넓은 경로를 지우지 않도록 prefix도 좁게 둬야 합니다.
테스트가 만든 임시 디렉터리만 정리하고, 경로를 문자열 조합으로 추측하지 않는 편이 안전합니다.

## fixture helper 설계 기준

### H3. 입력은 명시적으로 받고 기본값은 작게 둔다

좋은 fixture helper는 테스트가 바꾸고 싶은 값을 인자로 드러냅니다.
반대로 모든 값을 내부 기본값으로 숨기면 테스트가 왜 통과하는지 읽기 어려워집니다.

```js
function createUserFixture(overrides = {}) {
  return {
    id: 'user_1',
    email: 'user@example.com',
    role: 'member',
    ...overrides
  };
}

test('allows admin users to export reports', () => {
  const user = createUserFixture({ role: 'admin' });

  assert.equal(canExportReports(user), true);
});

function canExportReports(user) {
  return user.role === 'admin';
}
```

이 패턴은 테스트 의도를 잘 드러냅니다.
기본 fixture는 최소한의 정상 객체만 만들고, 테스트가 검증하는 차이는 `overrides`로 표시합니다.
단, `overrides`가 너무 커진다면 별도의 named fixture를 만드는 편이 더 읽기 쉽습니다.

### H3. 비즈니스 규칙을 helper 안에 넣지 않는다

fixture helper는 테스트 데이터를 준비하는 도구이지, 실제 도메인 로직을 다시 구현하는 곳이 아닙니다.
예를 들어 할인 정책을 테스트하면서 helper 안에서 할인 금액까지 계산하면 테스트가 구현과 같은 실수를 반복할 수 있습니다.

```js
function createOrderFixture(overrides = {}) {
  return {
    id: 'order_1',
    total: 50_000,
    country: 'KR',
    couponCode: null,
    ...overrides
  };
}
```

이 정도의 helper는 안전합니다.
주문 객체의 모양만 만들고, 배송비나 할인 계산은 테스트 대상 함수가 담당합니다.
테스트는 helper가 만든 입력을 사용해 실제 결과를 검증하면 됩니다.

## 임시 리소스와 CI 충돌 방지

### H3. 고정 파일명 대신 고유 디렉터리를 쓴다

CI에서는 테스트 파일이 병렬로 실행될 수 있습니다.
`/tmp/profile.json`처럼 고정 파일명을 사용하면 두 테스트가 같은 파일을 동시에 읽고 쓸 수 있습니다.
`mkdtemp()`로 고유 디렉터리를 만들고 그 안에 파일을 두면 충돌을 줄일 수 있습니다.

```js
import { mkdtemp, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

async function writeTempJson(name, value) {
  const dir = await mkdtemp(join(tmpdir(), 'json-fixture-'));
  const filePath = join(dir, name);

  await writeFile(filePath, JSON.stringify(value));

  return { dir, filePath };
}
```

파일명은 테스트 의미를 담아도 되지만, 디렉터리 자체는 실행마다 달라야 합니다.
이렇게 하면 `--test-shard`, randomize, 병렬 실행을 켜도 서로의 파일을 덮어쓸 가능성이 낮아집니다.

### H3. 외부 의존성은 fake와 contract를 나눠 본다

fixture helper가 외부 API 응답까지 모두 가짜로 만들 수는 있습니다.
하지만 모든 테스트가 fake 응답에만 의존하면 실제 계약 변화는 놓칠 수 있습니다.
단위 테스트에서는 fake client로 빠르게 검증하고, 별도 통합 테스트에서 계약을 확인하는 식으로 역할을 나누는 편이 좋습니다.

```js
function createPaymentClientFixture({ status = 'paid' } = {}) {
  return {
    async getPayment(paymentId) {
      return {
        id: paymentId,
        status
      };
    }
  };
}

test('marks paid payment as completed', async () => {
  const client = createPaymentClientFixture({ status: 'paid' });

  const result = await resolvePaymentStatus('pay_123', client);

  assert.equal(result, 'completed');
});

async function resolvePaymentStatus(paymentId, client) {
  const payment = await client.getPayment(paymentId);
  return payment.status === 'paid' ? 'completed' : 'pending';
}
```

fake client는 테스트를 빠르고 결정적으로 만듭니다.
다만 실제 API 스키마, 인증, 네트워크 오류 처리는 통합 테스트나 contract test에서 별도로 확인해야 합니다.
fixture helper 하나에 모든 테스트 목적을 넣으려 하면 오히려 유지보수가 어려워집니다.

## 실무 체크리스트

### H3. fixture helper를 추가하기 전 확인할 것

- 같은 준비 코드가 세 번 이상 반복되는가?
- 테스트가 바꾸고 싶은 입력이 helper 인자로 드러나는가?
- helper가 도메인 로직을 다시 구현하지 않는가?
- 호출할 때마다 새 리소스를 만드는가?
- cleanup 함수가 있고, 실패해도 다시 실행 가능한가?
- 임시 파일명과 포트, 식별자가 병렬 실행에서 충돌하지 않는가?
- 테스트 로그에 토큰, 실제 이메일, 개인정보가 남지 않는가?

체크리스트의 핵심은 fixture helper가 테스트를 더 짧게 만드는 데서 끝나지 않는다는 점입니다.
테스트의 독립성, 실패 분석, 정리 책임까지 함께 좋아져야 좋은 helper입니다.

### H3. 피해야 할 신호

fixture helper가 커질수록 테스트는 더 편해 보일 수 있습니다.
하지만 helper 내부에 조건문이 많아지고, 테스트마다 어떤 기본값이 들어가는지 외워야 한다면 이미 비용이 커진 상태입니다.

```js
// 피하는 편이 좋은 방향
const user = createEverythingFixture('admin-paid-premium-with-expired-coupon');
```

이런 문자열 기반 fixture는 빠르게 늘어나고, 실제 입력 구조를 감춥니다.
차라리 작은 helper를 조합하고 테스트에서 중요한 차이를 직접 드러내는 편이 유지보수에 유리합니다.

```js
const user = createUserFixture({ role: 'admin' });
const subscription = createSubscriptionFixture({ plan: 'premium', status: 'paid' });
const coupon = createCouponFixture({ expiresAt: '2026-01-01T00:00:00.000Z' });
```

각 객체의 역할이 분리되어 있으면 실패한 테스트를 읽을 때 원인을 좁히기 쉽습니다.
fixture helper는 테스트를 숨기는 도구가 아니라, 테스트가 중요한 차이에 집중하도록 돕는 도구여야 합니다.

## 마무리

Node.js test runner에서 fixture helper는 반복되는 준비 코드를 줄이는 동시에 테스트 리소스의 수명을 명확하게 만드는 장치입니다.
임시 디렉터리, fake client, 테스트 데이터 객체처럼 반복되는 준비 작업은 helper로 묶되, 테스트가 검증하는 핵심 입력은 본문에 드러내야 합니다.

가장 안전한 기본형은 호출할 때마다 새 리소스를 만들고 cleanup 함수를 함께 반환하는 구조입니다.
그 cleanup을 `afterEach()`에서 실행하면 테스트 실패 후에도 다음 테스트로 오염이 번질 가능성을 줄일 수 있습니다.
fixture helper를 작게 유지하면 CI 병렬 실행, randomize, shard 환경에서도 테스트가 더 안정적으로 읽히고 실행됩니다.

## 관련 글

- [Node.js test runner beforeEach/afterEach 가이드](/development/blog/seo/2026/07/10/nodejs-test-runner-beforeeach-aftereach-hooks-guide.html)
- [Node.js test runner test context 가이드](/development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html)
- [Node.js test runner randomize/seed 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-randomize-seed-order-dependent-test-guide.html)

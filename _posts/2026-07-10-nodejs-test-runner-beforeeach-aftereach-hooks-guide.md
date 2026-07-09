---
layout: post
title: "Node.js test runner beforeEach/afterEach 가이드: 테스트별 준비와 정리를 안정적으로 나누는 법"
date: 2026-07-10 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-beforeeach-aftereach-hooks-guide
permalink: /development/blog/seo/2026/07/10/nodejs-test-runner-beforeeach-aftereach-hooks-guide.html
alternates:
  ko: /development/blog/seo/2026/07/10/nodejs-test-runner-beforeeach-aftereach-hooks-guide.html
  x_default: /development/blog/seo/2026/07/10/nodejs-test-runner-beforeeach-aftereach-hooks-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, beforeeach, aftereach, hooks, testing, javascript, unit-test, ci]
description: "Node.js test runner의 beforeEach와 afterEach로 테스트별 fixture 준비, 상태 초기화, 리소스 정리를 안정적으로 나누는 방법을 정리합니다. before/after와의 차이, t.mock 정리, 파일과 서버 정리 패턴을 예제로 설명합니다."
---

테스트가 하나둘 늘어날 때는 각 테스트 본문에 준비 코드를 직접 써도 크게 불편하지 않습니다.
하지만 같은 fixture 생성, mock 초기화, 임시 파일 정리 코드가 반복되기 시작하면 테스트의 핵심 검증이 흐려집니다.
Node.js 내장 test runner의 `beforeEach()`와 `afterEach()`는 테스트마다 필요한 준비와 정리를 가까운 위치에서 일관되게 실행할 수 있게 해 줍니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#hooks)는 `before()`, `after()`, `beforeEach()`, `afterEach()` hook을 제공한다고 설명합니다.
이 중 `beforeEach()`와 `afterEach()`는 각 테스트를 기준으로 반복 실행되므로, 테스트 간 상태가 섞이면 안 되는 단위 테스트와 잘 맞습니다.
이 글에서는 per-test hook을 언제 쓰고, 언제 helper 함수나 global setup으로 나눌지 실무 기준을 정리합니다.

테스트 실행 전체의 준비와 정리는 [Node.js test runner global setup 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-global-setup-teardown-guide.html)를 참고하세요.
hook이 멈추는 문제를 빠르게 드러내는 방법은 [Node.js test runner timeout 가이드](/development/blog/seo/2026/07/07/nodejs-test-runner-timeout-hanging-test-guide.html)와 이어집니다.
mock 생명주기와 호출 검증은 [Node.js test runner mock.method 가이드](/development/blog/seo/2026/07/09/nodejs-test-runner-mock-method-restore-guide.html)도 함께 보면 좋습니다.

## beforeEach와 afterEach가 필요한 순간

### H3. 테스트마다 새 상태가 필요하다

`beforeEach()`는 각 테스트가 시작되기 전에 실행됩니다.
테스트가 같은 배열, Map, in-memory store, 임시 객체를 공유하면 실행 순서에 따라 결과가 달라질 수 있습니다.
이때 `beforeEach()`에서 새 상태를 만들면 테스트가 독립적으로 읽힙니다.

```js
import { beforeEach, test } from 'node:test';
import assert from 'node:assert/strict';

function createUserStore() {
  const users = new Map();

  return {
    add(user) {
      users.set(user.id, user);
    },
    find(id) {
      return users.get(id);
    },
    count() {
      return users.size;
    }
  };
}

let store;

beforeEach(() => {
  store = createUserStore();
});

test('adds a user', () => {
  store.add({ id: 'user_1', name: 'Kim' });

  assert.equal(store.count(), 1);
});

test('starts from an empty store again', () => {
  assert.equal(store.count(), 0);
});
```

두 번째 테스트가 첫 번째 테스트의 영향을 받지 않는다는 점이 중요합니다.
상태 초기화를 테스트 본문마다 반복하지 않아도 되고, 테스트 이름은 실제 검증 의도에 집중할 수 있습니다.
단, `beforeEach()`에 너무 많은 일을 넣으면 테스트가 어떤 입력으로 실행되는지 숨겨질 수 있으므로 공통 바닥 상태만 준비하는 편이 좋습니다.

### H3. 정리 작업은 테스트와 같은 수명에 둔다

`afterEach()`는 각 테스트가 끝난 뒤 실행됩니다.
임시 파일, fake server, timer, 커스텀 mock처럼 테스트마다 만든 리소스는 같은 범위에서 정리해야 합니다.
정리 코드를 `after()` 하나에 몰아두면 중간 테스트 실패 후 다음 테스트에 오염이 남을 수 있습니다.

```js
import { afterEach, beforeEach, test } from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp, readFile, rm, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

let fixtureDir;

beforeEach(async () => {
  fixtureDir = await mkdtemp(join(tmpdir(), 'node-test-hooks-'));
});

afterEach(async () => {
  await rm(fixtureDir, { recursive: true, force: true });
});

test('reads generated fixture', async () => {
  const filePath = join(fixtureDir, 'profile.json');

  await writeFile(filePath, JSON.stringify({ id: 'user_1' }));

  const body = await readFile(filePath, 'utf8');
  assert.deepEqual(JSON.parse(body), { id: 'user_1' });
});
```

이 예시는 테스트별 임시 디렉터리를 만들고 테스트가 끝나면 바로 삭제합니다.
파일 시스템을 쓰는 테스트는 실패했을 때 찌꺼기가 남기 쉬우므로 `force: true`와 좁은 임시 경로를 함께 쓰는 편이 안전합니다.
CI에서 병렬 실행을 켜더라도 테스트마다 다른 디렉터리를 쓰면 충돌 가능성이 낮아집니다.

## before/after와 구분하는 기준

### H3. 파일 전체에서 한 번이면 before와 after를 쓴다

`before()`와 `after()`는 같은 테스트 파일 안에서 한 번씩 실행되는 hook입니다.
비싼 준비 작업을 매 테스트마다 반복할 필요가 없고, 읽기 전용으로 공유해도 안전하다면 `before()`가 적합합니다.
예를 들어 큰 fixture 파일을 한 번 읽어 메모리에 올리는 작업은 테스트마다 다시 할 필요가 없습니다.

```js
import { before, test } from 'node:test';
import assert from 'node:assert/strict';

let productCatalog;

before(() => {
  productCatalog = Object.freeze([
    { id: 'book', price: 15_000 },
    { id: 'pen', price: 2_000 }
  ]);
});

test('finds book price', () => {
  const book = productCatalog.find((item) => item.id === 'book');

  assert.equal(book.price, 15_000);
});
```

공유 상태를 쓸 때는 읽기 전용인지 확인해야 합니다.
테스트 중 하나가 `productCatalog`를 수정한다면 다음 테스트 결과가 달라질 수 있습니다.
수정 가능한 데이터라면 `beforeEach()`에서 새 복사본을 만드는 편이 더 안정적입니다.

### H3. 테스트별 변경은 beforeEach에 둔다

데이터를 수정하거나 호출 기록을 쌓거나 clock을 고정하는 작업은 테스트마다 새로 시작해야 합니다.
이런 작업을 `before()`에 넣으면 테스트가 서로 같은 객체를 보게 됩니다.
테스트 순서를 바꾸거나 `--test-randomize`를 켰을 때 깨진다면 공유 상태를 의심해야 합니다.

```js
import { beforeEach, test } from 'node:test';
import assert from 'node:assert/strict';

let cart;

beforeEach(() => {
  cart = {
    items: [],
    add(item) {
      this.items.push(item);
    },
    total() {
      return this.items.reduce((sum, item) => sum + item.price, 0);
    }
  };
});

test('adds one item', () => {
  cart.add({ name: 'keyboard', price: 80_000 });

  assert.equal(cart.total(), 80_000);
});

test('adds another item from a clean cart', () => {
  cart.add({ name: 'mouse', price: 30_000 });

  assert.equal(cart.total(), 30_000);
});
```

이 패턴은 단순하지만 강력합니다.
각 테스트가 자기 입력만 설명하면 되므로 실패 메시지도 해석하기 쉽습니다.
상태가 복잡해지면 `createCartFixture()` 같은 helper 함수로 옮기고, `beforeEach()`는 helper 호출만 담당하게 만드는 편이 좋습니다.

## mock과 hook을 함께 쓰는 법

### H3. t.mock은 가능하면 테스트 본문 가까이에 둔다

Node.js test runner의 `t.mock`은 테스트 컨텍스트에 묶여 있어 mock 수명 관리에 유리합니다.
따라서 mock은 `beforeEach()`에서 전역 변수로 만들기보다 테스트 본문에서 직접 만드는 편이 더 명확한 경우가 많습니다.
특히 어떤 테스트는 성공 응답을, 어떤 테스트는 실패 응답을 원한다면 본문 가까이에 mock 구현을 두세요.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

async function loadProfile(userId, client) {
  const response = await client.get(`/users/${userId}`);
  return response.data;
}

test('loads a user profile', async (t) => {
  const client = {
    async get() {
      throw new Error('real client must not run');
    }
  };

  const get = t.mock.method(client, 'get', async () => {
    return { data: { id: 'user_1', name: 'Kim' } };
  });

  const profile = await loadProfile('user_1', client);

  assert.deepEqual(profile, { id: 'user_1', name: 'Kim' });
  assert.deepEqual(get.mock.calls[0].arguments, ['/users/user_1']);
});
```

이 방식은 mock 구현과 검증이 한 테스트 안에 모입니다.
`beforeEach()`에서 공통 client만 만들고, 실제 mock 구현은 각 테스트가 정하는 식으로 나누면 읽기 쉽습니다.
mock을 공통 hook에 숨기면 테스트가 어떤 응답 조건에서 실행되는지 찾기 어려워질 수 있습니다.

### H3. 공통 객체는 hook에서 만들고 세부 동작은 테스트에서 정한다

반복되는 객체 생성은 `beforeEach()`로 줄일 수 있습니다.
다만 테스트마다 달라지는 응답, 에러, 호출 검증은 테스트 본문에 남기는 편이 좋습니다.
아래처럼 공통 client 형태만 준비하고 메서드 mock은 각 테스트에서 설정하면 균형이 맞습니다.

```js
import { beforeEach, test } from 'node:test';
import assert from 'node:assert/strict';

let client;

beforeEach(() => {
  client = {
    async get() {
      throw new Error('unexpected network call');
    }
  };
});

test('returns true for active user', async (t) => {
  t.mock.method(client, 'get', async () => {
    return { data: { active: true } };
  });

  const response = await client.get('/users/user_1');

  assert.equal(response.data.active, true);
});

test('returns false for inactive user', async (t) => {
  t.mock.method(client, 'get', async () => {
    return { data: { active: false } };
  });

  const response = await client.get('/users/user_2');

  assert.equal(response.data.active, false);
});
```

공통 준비와 테스트별 차이를 분리하면 hook이 과하게 똑똑해지는 일을 막을 수 있습니다.
hook은 "모든 테스트가 반드시 필요로 하는 기본 상태"만 담당하고, 시나리오 조건은 테스트에 남기세요.
이 원칙을 지키면 나중에 실패한 테스트 하나만 읽어도 필요한 맥락을 대부분 파악할 수 있습니다.

## CI에서 hook을 안정적으로 운영하기

### H3. hook에도 timeout을 둔다

준비나 정리 단계가 멈추면 테스트 본문은 시작도 못 하고 CI 시간이 낭비됩니다.
Node.js test runner hook은 옵션 객체를 받을 수 있으므로, 느린 외부 리소스를 다루는 hook에는 timeout을 명시하는 편이 좋습니다.
테스트 timeout과 hook timeout을 분리하면 실패 위치도 더 분명해집니다.

```js
import { afterEach, beforeEach, test } from 'node:test';
import assert from 'node:assert/strict';

let server;

beforeEach(async () => {
  server = await startTestServer();
}, { timeout: 3_000 });

afterEach(async () => {
  await server.close();
}, { timeout: 3_000 });

test('calls test server', async () => {
  const response = await fetch(server.url);

  assert.equal(response.status, 200);
});
```

예시의 `startTestServer()`는 프로젝트 안의 테스트 helper라고 가정한 코드입니다.
핵심은 서버 시작과 종료가 오래 걸릴 때 hook 단계에서 실패하도록 만든다는 점입니다.
CI 로그에서 "테스트가 느림"과 "준비가 멈춤"을 구분할 수 있으면 재시도와 원인 분석이 빨라집니다.

### H3. 공유 리소스 이름에 테스트 고유값을 넣는다

CI shard나 병렬 실행을 사용하면 여러 테스트가 동시에 같은 리소스를 만들 수 있습니다.
임시 디렉터리, 포트, 파일 이름, 데이터베이스 record key가 겹치면 테스트가 서로를 방해합니다.
`beforeEach()`에서 리소스를 만들 때는 가능한 한 자동 생성된 고유값을 사용하세요.

```js
import { beforeEach, test } from 'node:test';
import assert from 'node:assert/strict';
import { randomUUID } from 'node:crypto';

let tenantId;

beforeEach(() => {
  tenantId = `test_${randomUUID()}`;
});

test('creates tenant scoped cache key', () => {
  const cacheKey = `${tenantId}:profile:user_1`;

  assert.match(cacheKey, /^test_[0-9a-f-]+:profile:user_1$/);
});
```

실제 서비스 테스트라면 `tenantId` 같은 스코프 값을 요청 payload나 fixture key에 넣을 수 있습니다.
테스트가 끝난 뒤에는 같은 스코프 기준으로 정리하면 됩니다.
고유값을 쓰면 테스트 순서, 병렬성, 재시도 여부가 바뀌어도 충돌 위험이 줄어듭니다.

## 실무 체크리스트

### H3. hook을 넣기 전에 확인할 것

`beforeEach()`와 `afterEach()`는 중복을 줄여주지만, 테스트의 중요한 맥락을 숨길 수도 있습니다.
아래 기준을 통과하는 공통 작업만 hook으로 옮기는 편이 좋습니다.

```text
beforeEach에 넣기 좋은 작업:
- 테스트마다 새로 필요한 빈 store나 fixture 생성
- 모든 테스트가 공통으로 쓰는 기본 객체 준비
- 테스트별 임시 경로, 고유 id, 기본 clock 생성

afterEach에 넣기 좋은 작업:
- 테스트마다 만든 파일, 서버, 연결 정리
- 커스텀 전역 변경 복원
- 외부 리소스 스코프 단위 삭제

본문에 남기는 편이 좋은 작업:
- 테스트별 성공/실패 응답 설정
- 핵심 입력 데이터 생성
- 호출 횟수와 인자 검증
```

hook이 길어지면 helper 함수로 추출하되, 이름은 구체적으로 붙이세요.
`setup()`보다 `createEmptyUserStore()`나 `createTempFixtureDir()`가 테스트 의도를 더 잘 드러냅니다.
테스트 본문을 읽는 사람이 숨은 전제 조건을 추적하지 않아도 되게 만드는 것이 좋은 hook 설계입니다.

### H3. 실패해도 정리 가능한 구조로 둔다

테스트 본문이 실패해도 `afterEach()`는 정리 기회를 제공합니다.
하지만 `beforeEach()`가 중간에 실패하면 정리할 대상이 일부만 만들어졌을 수 있습니다.
그래서 정리 코드는 값이 없거나 이미 닫힌 상태에서도 안전하게 실행되도록 작성하는 편이 좋습니다.

```js
import { afterEach, beforeEach, test } from 'node:test';
import assert from 'node:assert/strict';

let resource;

beforeEach(async () => {
  resource = await createResource();
});

afterEach(async () => {
  if (resource) {
    await resource.close();
    resource = undefined;
  }
});

test('uses resource', async () => {
  const result = await resource.run();

  assert.equal(result.ok, true);
});
```

정리 hook은 여러 번 실행돼도 안전한 형태가 좋습니다.
`close()`가 실패할 수 있다면 로그를 남기거나 명확한 에러로 실패시키는 기준도 정해야 합니다.
CI에서는 정리 실패를 조용히 무시하면 다음 job의 이상 현상으로 돌아올 수 있습니다.

## 마무리

Node.js test runner의 `beforeEach()`와 `afterEach()`는 테스트별 준비와 정리를 같은 수명으로 묶는 기본 도구입니다.
테스트마다 새 상태가 필요하면 `beforeEach()`, 테스트마다 만든 리소스를 치워야 하면 `afterEach()`를 쓰면 됩니다.

다만 모든 공통 코드를 hook으로 숨기는 것이 목표는 아닙니다.
공통 바닥 상태는 hook에 두고, 테스트별 입력과 기대 결과는 본문에 남겨야 실패한 테스트를 빠르게 이해할 수 있습니다.
전체 실행에서 한 번이면 충분한 준비는 global setup으로, 각 테스트의 독립성에 필요한 준비는 per-test hook으로 나누는 것이 가장 유지보수하기 쉬운 기준입니다.

## 관련 글

- [Node.js test runner global setup 가이드: 테스트 전후 공통 준비를 한곳에서 관리하는 법](/development/blog/seo/2026/07/05/nodejs-test-runner-global-setup-teardown-guide.html)
- [Node.js test runner timeout 가이드: 멈춘 테스트와 느린 hook을 빠르게 실패시키는 법](/development/blog/seo/2026/07/07/nodejs-test-runner-timeout-hanging-test-guide.html)
- [Node.js test runner mock.method 가이드: 객체 메서드를 안전하게 대체하고 복원하는 법](/development/blog/seo/2026/07/09/nodejs-test-runner-mock-method-restore-guide.html)

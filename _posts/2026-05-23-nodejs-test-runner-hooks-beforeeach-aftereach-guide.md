---
layout: post
title: "Node.js test runner hooks 가이드: beforeEach/afterEach로 테스트 준비와 정리 표준화하기"
date: 2026-05-23 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-hooks-beforeeach-aftereach-guide
permalink: /development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html
alternates:
  ko: /development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html
  x_default: /development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, beforeeach, aftereach, hooks, testing, javascript]
description: "Node.js 내장 test runner의 before, beforeEach, afterEach, after 훅으로 테스트 준비와 정리를 표준화하는 방법을 정리했습니다. fixture 생성, mock 초기화, 임시 파일 정리, CI에서 흔들리지 않는 테스트 구조까지 다룹니다."
---

테스트가 늘어날수록 실패 원인은 코드 버그만이 아닙니다.
이전 테스트가 남긴 파일, 재사용된 mock 호출 기록, 공유 객체의 변경, 닫히지 않은 리소스가 다음 테스트를 흔들 수 있습니다.
테스트가 서로 영향을 주기 시작하면 한 번은 통과하고 다음 번에는 실패하는 flaky test가 생깁니다.

Node.js 내장 test runner는 `before`, `beforeEach`, `afterEach`, `after` 훅으로 테스트 준비와 정리 과정을 구조화할 수 있습니다.
테스트 러너 기본기는 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)에서 다뤘고, CI 출력 관리는 [Node.js test runner reporter 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-reporter-ci-junit-guide.html)에서 정리했습니다.
이 글에서는 테스트 훅을 실무에서 안전하게 쓰는 기준을 정리합니다.

## Node.js test runner hooks가 필요한 이유

### 테스트 준비 코드는 중복되기 쉽다

테스트는 보통 비슷한 준비 과정을 반복합니다.
사용자 fixture를 만들고, 저장소를 초기화하고, 외부 API 클라이언트를 mock으로 바꾸고, 임시 디렉터리를 준비합니다.
이 코드를 각 테스트마다 직접 쓰면 처음에는 명확해 보이지만 테스트가 늘어날수록 수정 지점이 흩어집니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function createUser(overrides = {}) {
  return {
    id: 'user_1',
    email: 'dev@example.com',
    plan: 'free',
    ...overrides,
  };
}

test('무료 사용자는 기본 한도를 가진다', () => {
  const user = createUser();

  assert.equal(user.plan, 'free');
});

test('유료 사용자는 플랜을 바꿀 수 있다', () => {
  const user = createUser({ plan: 'pro' });

  assert.equal(user.plan, 'pro');
});
```

위 예시는 아직 간단합니다.
하지만 준비 과정이 데이터베이스 연결, 임시 파일, mock 초기화까지 커지면 각 테스트가 무엇을 검증하는지 흐려집니다.
훅은 반복되는 준비와 정리 작업을 한곳으로 모아 테스트 본문을 더 읽기 쉽게 만듭니다.

### 테스트 격리는 신뢰도의 핵심이다

테스트 격리는 “각 테스트가 독립적으로 실행되어도 같은 결과를 내는가”의 문제입니다.
실행 순서에 따라 결과가 달라진다면 테스트는 품질 신호가 아니라 운에 가까워집니다.
특히 CI에서는 테스트 파일 실행 순서나 병렬 실행 방식이 로컬과 다를 수 있어 격리 문제가 더 빨리 드러납니다.

`beforeEach`와 `afterEach`는 테스트마다 새 상태를 만들고 사용한 리소스를 정리하는 데 적합합니다.
외부 의존성 호출 기록은 [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html)처럼 테스트 단위로 초기화해야 실패 원인을 정확히 읽을 수 있습니다.

## before와 beforeEach 차이 이해하기

### before는 테스트 묶음 전체에 한 번 실행된다

`before`는 같은 suite 안에서 테스트가 시작되기 전에 한 번 실행됩니다.
비용이 크지만 여러 테스트가 공유해도 안전한 준비 작업에 적합합니다.
예를 들어 정적인 설정 파일을 읽거나, 한 번만 필요한 테스트 서버를 띄우는 작업이 여기에 들어갈 수 있습니다.

```js
import assert from 'node:assert/strict';
import { before, describe, test } from 'node:test';

let config;

before(() => {
  config = {
    region: 'ap-northeast-2',
    featureFlag: true,
  };
});

describe('설정 기반 기능', () => {
  test('리전을 읽는다', () => {
    assert.equal(config.region, 'ap-northeast-2');
  });

  test('기능 플래그를 읽는다', () => {
    assert.equal(config.featureFlag, true);
  });
});
```

공유 상태를 `before`에서 만든다면 테스트가 그 상태를 변경하지 않도록 주의해야 합니다.
한 테스트가 `config.featureFlag = false`처럼 값을 바꾸면 뒤 테스트에 영향을 줄 수 있습니다.
읽기 전용 데이터에 가깝게 유지하거나, 테스트마다 복사본을 만들어 쓰는 편이 안전합니다.

### beforeEach는 각 테스트 전에 실행된다

`beforeEach`는 테스트마다 새 준비 상태가 필요할 때 사용합니다.
공유 객체를 직접 재사용하지 않고 매번 새 객체를 만들면 테스트 순서에 덜 민감해집니다.

```js
import assert from 'node:assert/strict';
import { beforeEach, describe, test } from 'node:test';

let cart;

beforeEach(() => {
  cart = [];
});

describe('장바구니', () => {
  test('상품을 추가한다', () => {
    cart.push({ id: 'book', price: 15000 });

    assert.equal(cart.length, 1);
  });

  test('새 테스트는 빈 장바구니에서 시작한다', () => {
    assert.deepEqual(cart, []);
  });
});
```

이 패턴의 장점은 단순합니다.
첫 번째 테스트가 장바구니를 바꿔도 두 번째 테스트는 새 배열에서 시작합니다.
테스트 격리 관점에서는 `beforeEach`가 기본 선택지이고, 성능상 꼭 필요한 경우에만 `before`로 올리는 편이 안전합니다.

## afterEach와 after로 정리 책임 분리하기

### afterEach는 테스트마다 남긴 흔적을 지운다

`afterEach`는 각 테스트가 끝난 뒤 실행됩니다.
mock 호출 기록, 임시 파일, 타이머, 환경 변수 변경처럼 테스트 단위로 원복해야 하는 작업에 적합합니다.

```js
import assert from 'node:assert/strict';
import { afterEach, beforeEach, describe, mock, test } from 'node:test';

let sendEmail;

afterEach(() => {
  mock.reset();
});

beforeEach(() => {
  sendEmail = mock.fn(async () => ({ accepted: true }));
});

describe('메일 발송', () => {
  test('가입 메일을 한 번 보낸다', async () => {
    await sendEmail('dev@example.com');

    assert.equal(sendEmail.mock.callCount(), 1);
  });

  test('다음 테스트는 새 mock에서 시작한다', () => {
    assert.equal(sendEmail.mock.callCount(), 0);
  });
});
```

mock을 테스트마다 새로 만들더라도 전역 mock 상태를 정리하는 습관은 유용합니다.
특히 여러 파일에서 mock timer나 method mock을 섞어 쓴다면 `afterEach` 정리가 빠진 테스트가 뒤 테스트를 흔들 수 있습니다.

### after는 묶음 전체 리소스를 마지막에 정리한다

`after`는 suite가 끝난 뒤 한 번 실행됩니다.
테스트 서버 종료, 데이터베이스 연결 해제, 전체 테스트 묶음에서 공유한 큰 리소스 정리에 적합합니다.

```js
import assert from 'node:assert/strict';
import { after, before, describe, test } from 'node:test';

function createFakeServer() {
  return {
    closed: false,
    close() {
      this.closed = true;
    },
    getStatus() {
      return this.closed ? 503 : 200;
    },
  };
}

let server;

before(() => {
  server = createFakeServer();
});

after(() => {
  server.close();
});

describe('테스트 서버', () => {
  test('서버가 응답한다', () => {
    assert.equal(server.getStatus(), 200);
  });
});
```

여기서 중요한 기준은 “이 리소스가 테스트마다 새로 필요할까, 묶음 전체에서 공유해도 될까”입니다.
공유해도 되는 리소스는 `before`와 `after`로 묶고, 테스트별 상태는 `beforeEach`와 `afterEach`로 관리하면 구조가 선명해집니다.

## fixture 생성 패턴 설계하기

### 기본값이 있는 factory를 둔다

테스트 훅이 모든 준비 값을 직접 만들기 시작하면 훅 자체가 비대해질 수 있습니다.
이럴 때는 fixture factory를 두고, `beforeEach`에서는 테스트마다 필요한 기본 상태만 생성하게 만드는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { beforeEach, describe, test } from 'node:test';

function createOrder(overrides = {}) {
  return {
    id: 'order_1',
    userId: 'user_1',
    status: 'created',
    total: 30000,
    ...overrides,
  };
}

let order;

beforeEach(() => {
  order = createOrder();
});

describe('주문 상태', () => {
  test('기본 주문은 created 상태다', () => {
    assert.equal(order.status, 'created');
  });

  test('결제 완료 주문을 만들 수 있다', () => {
    order = createOrder({ status: 'paid' });

    assert.equal(order.status, 'paid');
  });
});
```

factory는 테스트 데이터의 의미를 드러냅니다.
무작위 값이나 긴 객체 literal을 테스트 본문에 반복하는 것보다, 독자가 “기본 주문을 만들고 이 필드만 바꿨구나”라고 바로 읽을 수 있습니다.

### 공유 fixture를 직접 변경하지 않는다

테스트 fixture에서 가장 흔한 실수는 공유 객체를 만든 뒤 여러 테스트가 그 객체를 직접 변경하는 것입니다.
객체가 깊은 구조를 가진다면 얕은 복사만으로는 부족할 수 있습니다.
Node.js의 `structuredClone`을 활용하면 테스트마다 독립적인 복사본을 만들기 쉽습니다.
관련 내용은 [Node.js structuredClone 가이드](/development/blog/seo/2026/05/15/nodejs-structuredclone-deep-copy-guide.html)에서 더 자세히 볼 수 있습니다.

```js
import assert from 'node:assert/strict';
import { beforeEach, describe, test } from 'node:test';

const baseProfile = {
  id: 'user_1',
  settings: {
    theme: 'light',
    alerts: true,
  },
};

let profile;

beforeEach(() => {
  profile = structuredClone(baseProfile);
});

describe('프로필 설정', () => {
  test('테마를 바꾼다', () => {
    profile.settings.theme = 'dark';

    assert.equal(profile.settings.theme, 'dark');
  });

  test('다음 테스트는 기본 테마를 유지한다', () => {
    assert.equal(profile.settings.theme, 'light');
  });
});
```

핵심은 훅이 공유 상태를 숨기는 도구가 아니라 격리를 명시하는 도구여야 한다는 점입니다.
`beforeEach`에서 매번 새 객체를 만든다는 사실이 보이면 테스트를 읽는 사람도 안심할 수 있습니다.

## 임시 파일과 디렉터리 정리하기

### 테스트마다 임시 디렉터리를 만든다

파일 시스템을 사용하는 테스트는 특히 정리 누락에 취약합니다.
고정 경로를 여러 테스트가 공유하면 로컬에서는 우연히 통과해도 CI 병렬 실행에서 충돌할 수 있습니다.
테스트마다 고유한 임시 디렉터리를 만들고 끝나면 삭제하는 패턴이 안전합니다.

```js
import assert from 'node:assert/strict';
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { afterEach, beforeEach, describe, test } from 'node:test';

let workDir;

beforeEach(async () => {
  workDir = await mkdtemp(join(tmpdir(), 'node-test-'));
});

afterEach(async () => {
  await rm(workDir, { recursive: true, force: true });
});

describe('파일 생성 기능', () => {
  test('결과 파일을 쓴다', async () => {
    const filePath = join(workDir, 'result.txt');

    await writeFile(filePath, 'ok', 'utf8');

    assert.match(filePath, /result\.txt$/);
  });
});
```

임시 디렉터리는 테스트 실패 시에도 `afterEach`에서 정리되어야 합니다.
그래야 실패한 테스트가 다음 테스트의 환경을 오염시키지 않습니다.
임시 리소스 정리 관점은 [Node.js mkdtempDisposable 가이드](/development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html)와 함께 보면 더 좋습니다.

### 테스트 산출물은 CI 리포트와 구분한다

테스트 중간 산출물과 CI 리포트 파일은 목적이 다릅니다.
임시 파일은 테스트가 끝나면 지워야 하고, JUnit XML이나 coverage 같은 리포트는 CI 아티팩트로 남길 수 있습니다.
둘을 같은 디렉터리에 섞으면 정리 스크립트가 리포트를 지우거나, 오래된 임시 파일이 리포트로 업로드될 수 있습니다.

권장 구조는 단순합니다.
테스트 중간 파일은 OS 임시 디렉터리 아래에 만들고, 리포트는 `reports/`처럼 명시적인 프로젝트 경로에 저장합니다.
리포트 저장 방식은 [Node.js test runner reporter 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-reporter-ci-junit-guide.html)에서 다룬 것처럼 CI 아티팩트 업로드 단계와 연결하면 됩니다.

## 비동기 훅을 안전하게 다루기

### async 훅은 반드시 await 가능한 작업을 반환한다

Node.js test runner 훅은 async 함수를 사용할 수 있습니다.
중요한 것은 준비 작업이 끝나기 전에 테스트가 시작되지 않도록 `await`를 빠뜨리지 않는 것입니다.

```js
import assert from 'node:assert/strict';
import { beforeEach, describe, test } from 'node:test';

async function createSession() {
  return {
    id: 'session_1',
    active: true,
  };
}

let session;

beforeEach(async () => {
  session = await createSession();
});

describe('세션', () => {
  test('활성 세션을 준비한다', () => {
    assert.equal(session.active, true);
  });
});
```

비동기 훅 안에서 promise를 만들고 반환하지 않으면 테스트가 준비되지 않은 상태로 실행될 수 있습니다.
테스트가 가끔만 실패한다면 훅 내부의 누락된 `await`부터 확인하는 것이 좋습니다.

### 타임아웃과 취소 기준을 둔다

외부 서비스나 느린 리소스를 준비하는 훅은 테스트 전체를 멈추게 만들 수 있습니다.
가능하면 단위 테스트에서는 외부 네트워크를 피하고, 통합 테스트라면 타임아웃과 취소 기준을 명확히 둡니다.
취소 흐름은 [Node.js fetch AbortSignal timeout/retry 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)와 [Node.js AbortSignal.any timeout 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)의 패턴을 테스트 준비 코드에도 적용할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { beforeEach, describe, test } from 'node:test';

async function loadFixture({ signal }) {
  signal.throwIfAborted();

  return { ready: true };
}

let fixture;

beforeEach(async () => {
  const signal = AbortSignal.timeout(1000);

  fixture = await loadFixture({ signal });
});

describe('fixture 로딩', () => {
  test('제한 시간 안에 준비한다', () => {
    assert.equal(fixture.ready, true);
  });
});
```

테스트 준비 단계도 운영 코드처럼 실패 방식을 설계해야 합니다.
준비가 끝나지 않는 테스트는 실패한 테스트보다 더 나쁩니다.
원인을 알려 주지 않고 CI 시간을 소모하기 때문입니다.

## 훅을 과하게 쓰지 않는 기준

### 테스트 본문에서 중요한 조건은 숨기지 않는다

훅은 반복을 줄이는 도구이지만, 테스트의 핵심 조건까지 숨기면 가독성이 떨어집니다.
예를 들어 “관리자 사용자는 삭제할 수 있다”를 검증하는 테스트에서 관리자 권한 부여가 `beforeEach` 깊숙이 숨어 있으면 독자가 테스트 본문만 보고 의도를 파악하기 어렵습니다.

좋은 기준은 “모든 테스트에 공통인 준비는 훅으로, 이 테스트만의 조건은 본문으로”입니다.

```js
import assert from 'node:assert/strict';
import { beforeEach, describe, test } from 'node:test';

let user;

beforeEach(() => {
  user = { id: 'user_1', role: 'member' };
});

describe('권한 검사', () => {
  test('관리자는 삭제할 수 있다', () => {
    user.role = 'admin';

    assert.equal(user.role, 'admin');
  });
});
```

이렇게 쓰면 기본 상태는 훅에서 보장하고, 테스트의 핵심 조건은 본문에 드러납니다.
테스트를 읽는 사람은 준비와 검증을 한 화면에서 이해할 수 있습니다.

### 훅 순서에 의존하는 테스트를 만들지 않는다

`before`, `beforeEach`, `afterEach`, `after`의 실행 순서를 이해하는 것은 필요합니다.
하지만 테스트가 훅의 부작용 순서에 지나치게 의존하면 유지보수가 어려워집니다.
특히 중첩된 `describe` 안에 훅을 많이 두면 실제 준비 상태를 추적하기 어려워질 수 있습니다.

중첩 suite는 도메인 구분이 명확할 때만 사용하고, 훅은 가까운 범위에 둡니다.
한 파일에 전역 훅이 너무 많다면 테스트 파일을 기능별로 나누는 편이 더 나을 수 있습니다.

## CI에서 흔들리지 않는 테스트 구조 만들기

### 테스트 파일은 독립 실행 가능해야 한다

좋은 테스트 파일은 단독으로 실행해도 통과해야 합니다.
특정 파일이 먼저 실행되어야 하거나, 다른 테스트가 만든 데이터를 기대한다면 CI 병렬화와 캐시 환경에서 깨질 가능성이 높습니다.

```bash
node --test test/order.test.js
```

이 명령으로 개별 파일을 실행했을 때도 통과해야 합니다.
테스트 전체 실행에서만 통과한다면 숨은 의존성이 있다는 신호입니다.
파일 단위 독립성은 테스트 리포트와도 연결됩니다.
리포트에서 특정 파일만 실패했을 때 해당 파일을 바로 재실행할 수 있어야 디버깅 속도가 빨라집니다.

### 정리 실패도 테스트 실패로 본다

`afterEach`에서 정리 작업이 실패한다면 그것도 중요한 신호입니다.
파일 삭제가 실패하거나 서버 close가 실패했다면 테스트 환경이 오염되었을 수 있습니다.
정리 실패를 조용히 무시하면 다음 테스트가 더 이상한 방식으로 실패합니다.

불가피하게 `force: true`를 쓰더라도, 어떤 리소스를 정리하고 있는지는 코드에서 명확히 보여야 합니다.
민감한 경로나 실제 운영 디렉터리를 테스트 정리 대상으로 삼지 않도록 임시 경로 prefix를 제한하는 것도 좋은 습관입니다.

## 실무 체크리스트

### hooks 적용 전 확인할 질문

- 모든 테스트에 공통인 준비 작업인가?
- 테스트마다 새로 만들어야 하는 상태인가, 묶음 전체에서 공유해도 되는 상태인가?
- 테스트 본문의 핵심 조건을 훅이 숨기고 있지는 않은가?
- 실패해도 `afterEach` 또는 `after`에서 리소스가 정리되는가?
- 개별 테스트 파일을 단독 실행해도 통과하는가?

### 추천 기본 구조

작은 프로젝트라면 다음 구조만으로도 충분합니다.
공유해도 안전한 설정은 `before`, 테스트별 상태는 `beforeEach`, 테스트별 흔적은 `afterEach`, 묶음 전체 리소스는 `after`에서 관리합니다.

```js
import { after, afterEach, before, beforeEach, describe, test } from 'node:test';

before(() => {
  // suite 전체에서 한 번만 필요한 준비
});

beforeEach(() => {
  // 각 테스트마다 새로 필요한 상태
});

afterEach(() => {
  // 각 테스트가 남긴 흔적 정리
});

after(() => {
  // suite 전체 리소스 정리
});

describe('기능 이름', () => {
  test('기대 동작을 구체적으로 설명한다', () => {
    // arrange, act, assert
  });
});
```

여기서 더 복잡해진다면 훅을 더 늘리기보다 fixture factory, 테스트 파일 분리, helper 함수 추출을 먼저 검토하는 것이 좋습니다.

## FAQ

### beforeEach를 많이 쓰면 테스트가 느려지지 않나요?

느려질 수 있습니다.
하지만 상태가 공유되어 flaky test가 생기는 비용이 더 큰 경우가 많습니다.
처음에는 `beforeEach`로 격리를 우선하고, 정말 병목으로 확인된 준비 작업만 `before`로 올리는 순서가 안전합니다.

### before에서 만든 객체를 테스트가 수정해도 되나요?

권장하지 않습니다.
`before`에서 만든 객체는 suite 전체가 공유하므로 한 테스트의 변경이 다른 테스트에 영향을 줄 수 있습니다.
수정이 필요하다면 `beforeEach`에서 복사본을 만들거나 fixture factory로 새 객체를 생성하세요.

### afterEach에서 mock.reset을 항상 호출해야 하나요?

mock을 쓰는 테스트 파일에서는 좋은 기본값입니다.
호출 기록, mock timer, method mock이 남아 뒤 테스트를 흔들 수 있기 때문입니다.
다만 파일 안에서 어떤 mock을 쓰는지 명확하게 유지하고, 테스트별 mock은 `beforeEach`에서 새로 만드는 것이 더 읽기 좋습니다.

### 훅과 helper 함수 중 무엇을 먼저 써야 하나요?

테스트마다 반드시 필요한 공통 준비와 정리는 훅이 적합합니다.
특정 테스트에서 선택적으로 쓰는 데이터 생성이나 조건 설정은 helper 함수가 더 명확합니다.
훅은 자동 실행되고 helper는 명시 호출되므로, 독자가 놓치면 안 되는 조건은 helper로 드러내는 편이 좋습니다.

## 마무리

Node.js test runner의 훅은 테스트 코드를 짧게 만드는 기능이 아니라 테스트 격리를 표준화하는 장치입니다.
`beforeEach`로 새 상태를 만들고, `afterEach`로 흔적을 지우며, 공유해도 안전한 리소스만 `before`와 `after`로 묶으면 테스트가 실행 순서에 덜 흔들립니다.

테스트가 신뢰할 수 있어야 CI 결과도 의미가 있습니다.
훅으로 준비와 정리 기준을 명확히 세우고, reporter와 mock 패턴까지 함께 정리하면 실패 로그를 읽는 시간이 줄어듭니다.
다음에 테스트가 가끔만 실패한다면 코드 로직만 보지 말고, 먼저 훅과 정리 책임이 제대로 나뉘어 있는지 확인해 보세요.

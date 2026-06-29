---
layout: post
title: "Node.js test runner concurrency 가이드: 병렬 테스트를 안전하게 실행하는 법"
date: 2026-06-30 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-concurrency-parallel-tests-guide
permalink: /development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html
alternates:
  ko: /development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html
  x_default: /development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, concurrency, parallel-test, testing, ci, javascript]
description: "Node.js test runner의 concurrency 옵션과 --test-concurrency 플래그로 병렬 테스트를 안전하게 실행하는 방법을 정리합니다. 파일 단위 병렬성, 하위 테스트 동시성, 공유 상태 격리, CI 적용 기준을 예제로 설명합니다."
---

테스트 스위트가 커지면 실행 시간을 줄이기 위해 병렬 실행을 고민하게 됩니다.
Node.js 내장 test runner는 파일 단위 실행과 테스트 내부 실행을 각각 제어할 수 있지만, 두 개념을 섞어 이해하면 공유 상태 문제나 불안정한 테스트가 생기기 쉽습니다.
병렬 테스트는 단순히 더 빠르게 돌리는 옵션이 아니라, 테스트가 서로 간섭하지 않는 구조인지 확인하는 압박 테스트이기도 합니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html)는 기본 프로세스 격리 모드에서 각 테스트 파일이 별도 child process로 실행되고, 동시에 실행되는 child process 수를 `--test-concurrency`로 제어한다고 설명합니다.
반면 `test()`의 `concurrency` 옵션은 같은 파일 안에서 하위 테스트를 동시에 실행할지 정하는 데 더 가깝습니다.
이 글에서는 두 레벨을 나눠 보고, CI에서 병렬 테스트를 켤 때 확인해야 할 실무 기준을 정리합니다.

기본 실행법은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 참고하세요.
테스트 준비와 정리 패턴은 [Node.js test runner hooks 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html)와 이어집니다.
불안정한 테스트를 좁혀야 한다면 [Node.js test runner 실패 테스트 재실행 가이드](/development/blog/seo/2026/05/26/nodejs-test-rerun-failures-flaky-test-debugging-guide.html)도 함께 보면 좋습니다.

## Node.js test runner 병렬성의 두 레벨

### H3. 파일 단위 병렬성은 --test-concurrency가 제어한다

`node --test`는 여러 테스트 파일을 찾아 실행합니다.
기본 프로세스 격리 모드에서는 테스트 파일마다 별도 child process가 만들어지고, 동시에 몇 개의 파일을 실행할지는 `--test-concurrency` 플래그가 제어합니다.
테스트 파일이 많고 파일 사이에 공유 상태가 없다면 이 값이 실행 시간에 큰 영향을 줍니다.

```bash
node --test --test-concurrency=4
```

이 명령은 한 번에 최대 4개 테스트 파일을 실행하도록 제한합니다.
CPU 코어가 많아도 외부 데이터베이스, 파일 시스템, API mock 서버 같은 공유 리소스가 병목이면 값을 무작정 높이지 않는 편이 좋습니다.
처음에는 낮은 값에서 시작해 CI 실행 시간과 실패 패턴을 함께 관찰하세요.

### H3. 파일 내부 하위 테스트는 test concurrency 옵션을 사용한다

같은 파일 안에서 하위 테스트를 병렬로 실행하려면 `test()` 옵션의 `concurrency`를 사용할 수 있습니다.
이 옵션은 파일 단위 child process 개수와 다릅니다.
한 파일 안에서 독립적인 케이스를 동시에 실행할 때 의미가 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

async function calculateShipping(region) {
  await new Promise((resolve) => setTimeout(resolve, 10));

  return region === 'local' ? 3000 : 7000;
}

test('shipping fee by region', { concurrency: true }, async (t) => {
  await t.test('local region', async () => {
    assert.strictEqual(await calculateShipping('local'), 3000);
  });

  await t.test('remote region', async () => {
    assert.strictEqual(await calculateShipping('remote'), 7000);
  });
});
```

이 구조에서는 두 하위 테스트가 서로 같은 mutable 상태를 공유하지 않아야 합니다.
예제처럼 입력과 출력이 독립적인 순수 계산 또는 읽기 전용 조회라면 병렬 실행에 잘 맞습니다.
반대로 같은 배열, 같은 mock, 같은 임시 파일을 수정한다면 순차 실행이 더 안전합니다.

## 병렬 테스트가 깨지는 흔한 이유

### H3. 전역 상태를 공유하면 실행 순서에 의존한다

병렬 테스트에서 가장 흔한 문제는 테스트들이 같은 전역 상태를 읽고 쓰는 것입니다.
순차 실행에서는 우연히 통과하던 테스트가 병렬 실행에서 실패한다면, 테스트가 실행 순서에 의존하고 있을 가능성이 큽니다.
공유 상태는 테스트마다 새로 만들거나, 하위 테스트 안으로 좁히는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function createCart() {
  const items = [];

  return {
    add(item) {
      items.push(item);
    },
    count() {
      return items.length;
    }
  };
}

test('cart behavior', { concurrency: true }, async (t) => {
  await t.test('adds one item', () => {
    const cart = createCart();

    cart.add('book');

    assert.strictEqual(cart.count(), 1);
  });

  await t.test('starts empty', () => {
    const cart = createCart();

    assert.strictEqual(cart.count(), 0);
  });
});
```

핵심은 `cart`를 부모 스코프에 하나만 만들지 않는 것입니다.
각 하위 테스트가 자기 fixture를 만들면 실행 순서가 바뀌어도 결과가 흔들리지 않습니다.
이 원칙은 메모리 객체뿐 아니라 임시 디렉터리, mock 서버 포트, 환경 변수에도 똑같이 적용됩니다.

### H3. mock과 타이머는 테스트마다 정리한다

`mock.method()`, fake timer, 환경 변수 변경처럼 런타임 상태를 바꾸는 테스트는 병렬 실행에서 특히 조심해야 합니다.
하나의 테스트가 바꾼 상태가 다른 테스트에 보이면 실패 원인이 매우 흐려집니다.
가능하면 테스트별로 mock을 만들고, `afterEach` 또는 테스트 컨텍스트 정리 흐름에서 복구하세요.

```js
import assert from 'node:assert/strict';
import { afterEach, mock, test } from 'node:test';

const clock = {
  now() {
    return Date.now();
  }
};

afterEach(() => {
  mock.restoreAll();
});

test('formats current day with injected clock', () => {
  mock.method(clock, 'now', () => Date.UTC(2026, 5, 30));

  const day = new Date(clock.now()).toISOString().slice(0, 10);

  assert.strictEqual(day, '2026-06-30');
});
```

이 예제는 테스트가 끝난 뒤 mock을 복구합니다.
파일 내부 병렬 실행을 적극적으로 쓴다면 같은 객체 메서드를 여러 하위 테스트에서 동시에 mock하지 않는 구조가 더 안전합니다.
공유 객체를 바꿔야 한다면 해당 테스트 묶음은 순차 실행으로 남겨 두는 판단도 필요합니다.

## CI에서 concurrency 값을 정하는 기준

### H3. 로컬과 CI의 병목이 다를 수 있다

로컬에서는 `--test-concurrency=8`이 빠르게 보이더라도 CI에서는 오히려 느려질 수 있습니다.
CI runner의 CPU, 메모리, 파일 시스템 성능, 서비스 컨테이너 개수, 데이터베이스 connection 제한이 다르기 때문입니다.
병렬도는 개발자 노트북 기준이 아니라 실제 CI 환경 기준으로 측정해야 합니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "node --test --test-concurrency=4"
  }
}
```

`test`는 기본값으로 두고, CI용 스크립트에 명시적인 값을 고정하면 재현성이 좋아집니다.
프로젝트 규모가 커지면 2, 4, 8 같은 후보를 실제 실행 시간과 실패율로 비교하세요.
가장 빠른 값보다 안정적으로 반복 통과하는 값을 선택하는 편이 좋습니다.

### H3. 외부 리소스가 있으면 병렬도를 낮춘다

테스트가 실제 데이터베이스, Redis, 파일 시스템, 로컬 HTTP 서버를 건드린다면 병렬 실행은 비용을 키울 수 있습니다.
connection pool이 작거나 포트가 고정되어 있으면 테스트 파일끼리 충돌할 수 있습니다.
이런 테스트는 파일을 분리하고, 리소스 이름에 고유 suffix를 붙이거나, 병렬도를 낮춰야 합니다.

```js
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import assert from 'node:assert/strict';
import test from 'node:test';

test('writes report to an isolated temp directory', async (t) => {
  const dir = await mkdtemp(join(tmpdir(), 'report-test-'));

  t.after(async () => {
    await rm(dir, { recursive: true, force: true });
  });

  const reportPath = join(dir, 'summary.json');

  await writeFile(reportPath, JSON.stringify({ ok: true }));

  assert.match(reportPath, /report-test-/);
});
```

임시 디렉터리를 테스트마다 만들면 같은 파일명을 두고 충돌할 가능성이 줄어듭니다.
외부 리소스를 공유해야 한다면 테스트 데이터 prefix, transaction rollback, cleanup hook 같은 격리 전략을 먼저 준비하세요.
격리 없이 concurrency만 올리면 테스트 속도보다 디버깅 비용이 더 커질 수 있습니다.

## 병렬 실행을 켜기 전 체크리스트

### H3. 순서 의존성을 먼저 제거한다

병렬 실행을 켜기 전에 전체 테스트가 순차 실행에서 안정적으로 통과해야 합니다.
이미 flaky test가 있는 상태에서 concurrency를 올리면 원인 분석이 더 어려워집니다.
먼저 실패를 재현하고, 공유 상태와 정리 누락을 줄인 뒤 병렬도를 조정하는 순서가 좋습니다.

- 테스트마다 fixture를 새로 만드는가?
- 전역 변수, process.env, mock, timer를 바꾼 뒤 복구하는가?
- 임시 파일과 포트가 테스트별로 분리되는가?
- 외부 서비스 connection limit를 넘지 않는가?
- CI에서 같은 명령을 여러 번 반복해도 통과하는가?

이 체크리스트는 병렬 테스트뿐 아니라 일반적인 테스트 신뢰도에도 직접 연결됩니다.
특히 `process.env`를 바꾸는 테스트는 이전 값을 저장했다가 반드시 되돌려야 합니다.
테스트가 서로 독립적이라는 확신이 생긴 뒤에 concurrency 값을 올리는 편이 안전합니다.

### H3. 느린 테스트와 병렬 테스트를 구분한다

모든 느린 테스트가 병렬 실행으로 해결되는 것은 아닙니다.
느린 이유가 외부 API 대기, 불필요한 sleep, 큰 fixture 생성, 반복적인 빌드 작업이라면 병렬도보다 테스트 설계를 먼저 봐야 합니다.
병렬 실행은 독립적인 작업이 많을 때 효과가 큽니다.

예를 들어 20개 파일이 각각 순수 함수와 가벼운 I/O를 테스트한다면 파일 단위 병렬성이 잘 맞습니다.
반대로 하나의 통합 테스트 파일이 실제 서버와 데이터베이스를 오래 붙잡고 있다면, 파일을 잘게 나누거나 fixture를 줄이는 것이 먼저일 수 있습니다.
속도를 올리는 방법과 테스트를 안정화하는 방법을 분리해서 판단하세요.

## 마무리

Node.js test runner의 concurrency는 테스트 시간을 줄일 수 있는 강력한 옵션입니다.
하지만 파일 단위 `--test-concurrency`와 파일 내부 `concurrency` 옵션은 서로 다른 레벨의 도구입니다.
파일 사이 병렬성은 CI 처리량을 높이고, 하위 테스트 병렬성은 독립적인 케이스를 빠르게 확인하는 데 적합합니다.

좋은 병렬 테스트의 기준은 단순합니다.
각 테스트가 자기 데이터를 만들고, 자기 흔적을 지우며, 다른 테스트의 순서를 기대하지 않아야 합니다.
이 조건을 만족한 뒤 concurrency 값을 조정하면 테스트 스위트는 더 빨라지면서도 신뢰도를 잃지 않습니다.

## 함께 보면 좋은 글

- [Node.js test runner 가이드: 내장 테스트 러너로 테스트 시작하기](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)
- [Node.js test runner hooks 가이드: beforeEach/afterEach로 테스트 준비와 정리 표준화하기](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html)
- [Node.js test runner 실패 테스트 재실행 가이드: --test-rerun-failures로 디버깅 시간을 줄이는 법](/development/blog/seo/2026/05/26/nodejs-test-rerun-failures-flaky-test-debugging-guide.html)

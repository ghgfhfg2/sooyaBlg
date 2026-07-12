---
layout: post
title: "Node.js test runner concurrency 가이드: 병렬 테스트를 안전하게 빠르게 돌리는 법"
date: 2026-07-13 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-concurrency-parallel-test-guide
permalink: /development/blog/seo/2026/07/13/nodejs-test-runner-concurrency-parallel-test-guide.html
alternates:
  ko: /development/blog/seo/2026/07/13/nodejs-test-runner-concurrency-parallel-test-guide.html
  x_default: /development/blog/seo/2026/07/13/nodejs-test-runner-concurrency-parallel-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, concurrency, parallel-test, testing, javascript, ci, flaky-test]
description: "Node.js test runner concurrency 옵션으로 병렬 테스트를 빠르고 안전하게 운영하는 방법을 정리합니다. --test-concurrency, t.test concurrency, 공유 자원 격리, flaky test 방지 기준을 예제로 설명합니다."
---

테스트 시간이 길어지면 개발자는 테스트를 덜 자주 실행하게 됩니다.
CI에서도 같은 문제가 생깁니다.
테스트가 너무 느리면 PR 피드백이 늦어지고, 배포 전 확인 단계가 병목이 됩니다.
Node.js 내장 test runner의 concurrency 설정은 이 시간을 줄일 수 있는 실용적인 선택지입니다.

하지만 병렬 실행은 단순히 숫자를 키우는 문제가 아닙니다.
파일, 포트, 데이터베이스, 환경 변수처럼 공유 자원이 섞인 테스트를 무작정 동시에 돌리면 빠른 테스트가 아니라 불안정한 테스트가 됩니다.
concurrency는 테스트 격리 기준과 함께 설계해야 효과가 있습니다.

이 글에서는 Node.js test runner에서 `--test-concurrency`와 하위 테스트의 `concurrency` 옵션을 어떻게 바라봐야 하는지, 어떤 테스트를 병렬화해도 좋은지, CI에서 flaky test를 만들지 않으려면 어떤 안전장치가 필요한지 정리합니다.
테스트 파일 격리 전략은 [Node.js test runner isolation 가이드](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html), CI 분산 실행은 [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html), 반복 케이스 정리는 [Node.js test runner table-driven tests 가이드](/development/blog/seo/2026/07/12/nodejs-test-runner-table-driven-tests-guide.html)와 함께 보면 좋습니다.

## Node.js test runner concurrency를 쓰는 이유

### H3. 느린 테스트를 기다리는 시간을 줄인다

테스트 시간이 늘어나는 이유는 대개 테스트 수만이 아닙니다.
네트워크 흉내, 파일 입출력, 임시 리소스 생성, 비동기 작업 대기처럼 서로 독립적인 시간이 쌓이기 때문입니다.
서로 영향을 주지 않는 테스트라면 순서대로 실행할 필요가 없습니다.

예를 들어 20개의 독립적인 서비스 함수 테스트가 각각 1초씩 걸린다면 순차 실행은 20초에 가깝습니다.
일부를 병렬로 실행하면 전체 피드백 시간을 줄일 수 있습니다.
이때 목표는 모든 테스트를 동시에 실행하는 것이 아니라, 격리된 테스트를 적절한 수준으로 동시에 실행하는 것입니다.

```bash
node --test --test-concurrency=4
```

위 명령은 test runner가 동시에 실행할 테스트 작업 수를 제한합니다.
숫자가 높을수록 항상 빠른 것은 아닙니다.
CPU, 메모리, 파일 시스템, 외부 서비스 mock의 성격에 따라 오히려 느려지거나 불안정해질 수 있습니다.

### H3. 병렬화 대상과 순차 실행 대상을 나눈다

concurrency를 적용하기 전에 테스트를 두 그룹으로 나누는 것이 좋습니다.
첫 번째는 순서와 공유 상태에 영향을 받지 않는 테스트입니다.
순수 함수, 독립적인 validator, 임시 디렉터리를 케이스별로 만드는 파일 테스트가 여기에 가깝습니다.

두 번째는 공유 자원을 쓰는 테스트입니다.
같은 포트, 같은 데이터베이스 스키마, 전역 환경 변수, 고정 파일 경로, 싱글톤 캐시를 건드리는 테스트는 병렬 실행에서 충돌할 가능성이 큽니다.
이런 테스트는 먼저 격리 구조를 만들거나 순차 실행으로 남겨야 합니다.

```text
병렬화하기 좋은 테스트
- 입력과 출력이 명확한 순수 함수 테스트
- 케이스마다 독립 임시 디렉터리를 쓰는 파일 테스트
- mock 객체를 테스트 내부에서 새로 만드는 단위 테스트

주의가 필요한 테스트
- process.env를 직접 바꾸는 테스트
- 같은 포트나 같은 파일명을 공유하는 테스트
- 데이터베이스 seed와 cleanup 순서에 의존하는 테스트
```

테스트를 빠르게 만들고 싶다면 먼저 의존성을 드러내야 합니다.
병렬 실행은 숨은 공유 상태를 찾아내는 계기가 되기도 합니다.

## --test-concurrency로 전체 실행 폭 조절하기

### H3. CI에서는 고정된 숫자로 시작한다

로컬 머신은 개발자마다 성능이 다릅니다.
CI도 runner 종류와 동시 job 수에 따라 가용 자원이 달라질 수 있습니다.
그래서 처음에는 공격적인 값보다 보수적인 고정값으로 시작하는 편이 좋습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:ci": "node --test --test-concurrency=4"
  }
}
```

이렇게 두면 로컬 기본 실행은 단순하게 유지하고, CI에서는 병렬 폭을 명시적으로 관리할 수 있습니다.
팀에서 테스트 시간이 충분히 안정적이라는 판단이 서면 값을 조정합니다.
반대로 flaky test가 늘어난다면 숫자를 키우기 전에 공유 자원 충돌부터 확인해야 합니다.

### H3. 속도보다 재현성을 먼저 측정한다

concurrency 값을 바꿀 때는 평균 시간만 보면 안 됩니다.
실패 재현성도 함께 봐야 합니다.
테스트 시간이 30% 줄었지만 간헐 실패가 생겼다면 운영 관점에서는 손해입니다.

간단한 확인 방법은 같은 명령을 여러 번 반복해 보는 것입니다.

```bash
for i in 1 2 3 4 5; do
  echo "run $i"
  node --test --test-concurrency=4
done
```

반복 실행에서 특정 테스트가 가끔 실패한다면 concurrency가 문제를 만든 것이 아니라, 기존 테스트가 공유 상태에 의존하고 있었을 가능성이 큽니다.
이때는 실패 테스트를 따로 떼어 격리하거나, 임시 리소스 이름을 고유하게 만들거나, 전역 상태 변경을 테스트 안에서 복구해야 합니다.

## t.test concurrency로 하위 테스트 제어하기

### H3. 독립 케이스는 하위 테스트 단위로 병렬화한다

부모 테스트 안에서 여러 하위 테스트를 만들 때도 concurrency를 고려할 수 있습니다.
케이스별 입력과 기대 결과가 완전히 독립적이라면 하위 테스트를 동시에 실행해 시간을 줄일 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('renderInvoicePreview', { concurrency: true }, async (t) => {
  const cases = [
    { name: 'basic plan', plan: 'basic', expected: 'Basic' },
    { name: 'pro plan', plan: 'pro', expected: 'Pro' },
    { name: 'enterprise plan', plan: 'enterprise', expected: 'Enterprise' }
  ];

  for (const testCase of cases) {
    t.test(testCase.name, async () => {
      const preview = await renderInvoicePreview({ plan: testCase.plan });
      assert.match(preview.title, new RegExp(testCase.expected));
    });
  }
});

async function renderInvoicePreview({ plan }) {
  return { title: `${plan[0].toUpperCase()}${plan.slice(1)} invoice` };
}
```

이 예시는 각 케이스가 독립된 입력으로 결과를 검증합니다.
공유 배열을 수정하거나 같은 파일에 쓰지 않습니다.
이런 구조에서는 concurrency가 테스트 의미를 바꾸지 않습니다.

### H3. 공유 setup이 있으면 병렬화 범위를 줄인다

모든 하위 테스트가 같은 fixture를 수정한다면 병렬 실행은 위험합니다.
특히 객체 하나를 만들고 여러 테스트가 그 객체를 바꾸는 구조는 순서 의존성을 만들기 쉽습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('cart discounts', async (t) => {
  await t.test('member discount', () => {
    const cart = createCart();
    cart.add({ sku: 'plan-member', price: 10000 });
    assert.equal(cart.totalFor('member'), 9000);
  });

  await t.test('vip discount', () => {
    const cart = createCart();
    cart.add({ sku: 'plan-vip', price: 10000 });
    assert.equal(cart.totalFor('vip'), 8000);
  });
});

function createCart() {
  const items = [];

  return {
    add(item) {
      items.push(item);
    },
    totalFor(role) {
      const total = items.reduce((sum, item) => sum + item.price, 0);
      if (role === 'vip') return total * 0.8;
      if (role === 'member') return total * 0.9;
      return total;
    }
  };
}
```

위 코드처럼 하위 테스트마다 fixture를 새로 만들면 병렬화 가능성이 높아집니다.
반대로 부모 테스트에서 `const cart = createCart()`를 한 번만 만들고 모든 하위 테스트가 공유한다면 순차 실행으로 두는 편이 안전합니다.

## 공유 자원 충돌을 줄이는 패턴

### H3. 임시 파일과 포트는 케이스마다 고유하게 만든다

병렬 테스트에서 가장 흔한 충돌은 고정 경로입니다.
`./tmp/output.json` 같은 파일을 여러 테스트가 동시에 쓰면 마지막에 쓴 테스트가 앞선 테스트 결과를 덮어쓸 수 있습니다.
임시 디렉터리는 테스트마다 새로 만드는 기준을 세우는 편이 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp, readFile, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

test('writes a report file', async () => {
  const dir = await mkdtemp(join(tmpdir(), 'report-test-'));
  const reportPath = join(dir, 'report.json');

  await writeFile(reportPath, JSON.stringify({ passed: true }), 'utf8');

  const saved = JSON.parse(await readFile(reportPath, 'utf8'));
  assert.equal(saved.passed, true);
});
```

임시 경로를 고유하게 만들면 테스트가 동시에 실행되어도 서로의 결과를 덮어쓰지 않습니다.
DB나 메시지 큐를 쓰는 통합 테스트도 같은 원칙을 따릅니다.
스키마 이름, 큐 이름, tenant ID처럼 충돌 가능한 식별자는 테스트 실행 단위별로 분리합니다.

### H3. process.env 변경은 복구 기준을 둔다

`process.env`는 프로세스 전역 상태입니다.
한 테스트가 환경 변수를 바꾸는 동안 다른 테스트가 읽으면 예상하지 못한 결과가 나올 수 있습니다.
환경 변수 테스트는 가능한 한 순차 실행으로 두거나, 변경 전 값을 저장하고 반드시 복구해야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('reads feature flag from env', () => {
  const previous = process.env.FEATURE_CHECKOUT_V2;

  try {
    process.env.FEATURE_CHECKOUT_V2 = 'enabled';
    assert.equal(isCheckoutV2Enabled(), true);
  } finally {
    if (previous === undefined) {
      delete process.env.FEATURE_CHECKOUT_V2;
    } else {
      process.env.FEATURE_CHECKOUT_V2 = previous;
    }
  }
});

function isCheckoutV2Enabled() {
  return process.env.FEATURE_CHECKOUT_V2 === 'enabled';
}
```

복구 코드는 테스트가 실패해도 실행되어야 하므로 `finally`에 둡니다.
이런 테스트가 많다면 환경 변수를 직접 읽는 함수보다 설정 객체를 주입받는 구조로 바꾸는 편이 더 테스트하기 쉽습니다.

## CI 운영 체크리스트

### H3. concurrency 변경은 작은 PR로 분리한다

테스트 병렬화는 코드 변경처럼 다뤄야 합니다.
한 번에 concurrency 값, 테스트 구조, CI shard, reporter를 모두 바꾸면 실패 원인을 찾기 어렵습니다.
먼저 `--test-concurrency` 값만 작게 적용하고, 반복 실행으로 안정성을 확인한 뒤 범위를 넓히는 편이 좋습니다.

```yaml
name: test

on:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run test:ci
```

CI 설정에서는 concurrency 값을 package script 안에 두면 로컬 재현이 쉬워집니다.
실패한 개발자는 같은 `npm run test:ci` 명령으로 문제를 확인할 수 있습니다.

### H3. 실패 로그에는 케이스 이름이 보여야 한다

병렬 테스트에서는 실행 순서가 매번 같지 않을 수 있습니다.
그래서 실패 로그에서 어떤 케이스가 깨졌는지 바로 보여야 합니다.
하위 테스트 이름을 `case 1`처럼 두면 병렬 실행 후 로그를 읽기가 더 어려워집니다.

좋은 이름은 테스트가 검증하는 규칙을 말합니다.
예를 들어 `returns cached profile for repeated user id`, `creates isolated temp report path`, `restores env flag after failure`처럼 실패 원인을 좁히는 이름이 좋습니다.
이 기준은 table-driven test와도 같습니다.

## 마무리

Node.js test runner concurrency는 테스트 시간을 줄이는 강력한 방법이지만, 격리되지 않은 테스트를 빠르게 실행한다고 좋은 테스트가 되는 것은 아닙니다.
먼저 병렬화해도 되는 테스트와 순차 실행이 필요한 테스트를 나누고, 파일 경로·포트·환경 변수·DB 상태 같은 공유 자원을 케이스별로 분리해야 합니다.

실무에서는 `--test-concurrency=4`처럼 보수적인 값으로 시작하고, 반복 실행으로 재현성을 확인한 뒤 조금씩 조정하는 흐름이 안전합니다.
하위 테스트 병렬화는 케이스가 독립적일 때만 적용하고, 실패 로그에는 규칙 중심의 이름을 남기세요.
빠른 테스트보다 중요한 것은 믿을 수 있는 테스트이고, 좋은 concurrency 설정은 그 신뢰를 해치지 않는 범위에서 피드백 시간을 줄이는 선택입니다.

## FAQ

### H3. --test-concurrency 값을 CPU 코어 수와 같게 두면 되나요?

항상 그렇지는 않습니다.
CPU 중심 테스트라면 코어 수가 참고값이 될 수 있지만, 파일 입출력이나 통합 테스트가 많으면 더 낮은 값이 안정적일 수 있습니다.
처음에는 2나 4처럼 작은 값으로 시작해 반복 실행 결과를 보고 조정하는 편이 좋습니다.

### H3. flaky test가 생기면 concurrency를 끄는 것이 답인가요?

임시 대응으로 값을 낮출 수는 있지만, 근본 원인은 공유 상태일 가능성이 큽니다.
고정 파일 경로, 전역 환경 변수, DB seed, 싱글톤 캐시처럼 테스트 간에 공유되는 자원을 먼저 확인하세요.
격리가 끝난 뒤 다시 concurrency를 적용하면 속도와 안정성을 함께 얻을 수 있습니다.

### H3. shard와 concurrency는 같이 써도 되나요?

같이 쓸 수 있지만 역할이 다릅니다.
shard는 테스트 묶음을 여러 CI job으로 나누고, concurrency는 한 job 안에서 동시에 실행할 폭을 조절합니다.
둘을 동시에 조정하면 원인 분석이 어려우므로 먼저 하나씩 적용하고 결과를 확인하는 편이 안전합니다.

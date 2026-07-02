---
layout: post
title: "Node.js test runner tags 가이드: 느린 테스트와 통합 테스트를 안전하게 분리하는 법"
date: 2026-07-02 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-tags-filter-guide
permalink: /development/blog/seo/2026/07/02/nodejs-test-runner-tags-filter-guide.html
alternates:
  ko: /development/blog/seo/2026/07/02/nodejs-test-runner-tags-filter-guide.html
  x_default: /development/blog/seo/2026/07/02/nodejs-test-runner-tags-filter-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, tags, filtering, testing, ci, javascript]
description: "Node.js test runner의 tags와 --experimental-test-tag-filter로 unit, integration, slow, db 테스트를 분리 실행하는 방법을 정리합니다. 태그 설계, CI 명령, name pattern과의 차이, 운영 체크리스트를 예제로 설명합니다."
---

테스트가 많아질수록 모든 검증을 매번 같은 방식으로 실행하기 어렵습니다.
빠른 단위 테스트는 커밋 전마다 돌리고 싶지만, 데이터베이스나 외부 서비스가 필요한 통합 테스트는 CI의 별도 단계에서만 실행하고 싶을 수 있습니다.
이때 테스트 이름에 `[slow]`, `[db]` 같은 문자열을 억지로 넣으면 이름이 점점 실행 정책을 담는 메타데이터가 됩니다.

Node.js test runner는 이런 문제를 줄이기 위해 test tags 기능을 제공합니다.
[Node.js test runner 공식 문서](https://nodejs.org/api/test.html)에 따르면 tag는 테스트와 suite에 붙이는 문자열 라벨이며, `--experimental-test-tag-filter` 플래그로 특정 tag를 가진 테스트만 선택할 수 있습니다.
다만 공식 문서에서 early development 상태로 표시되는 기능이므로, 팀 표준으로 도입할 때는 Node.js 버전과 CI 명령을 함께 고정하는 편이 안전합니다.

Node.js 내장 테스트 러너의 기본 흐름은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 참고하세요.
이름 기반 필터링과 집중 실행은 [Node.js test runner skip, todo, only 가이드](/development/blog/seo/2026/07/01/nodejs-test-runner-skip-todo-only-focused-tests-guide.html)와 함께 보면 차이가 분명해집니다.
병렬 실행 전략은 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html)에서 이어서 정리했습니다.

## test tags가 필요한 상황

### H3. 테스트 이름이 아니라 실행 조건을 분리한다

테스트 이름은 독자가 실패 원인을 이해하도록 쓰는 문장입니다.
반면 tag는 실행 정책을 표현하는 메타데이터입니다.
예를 들어 `payment refunds stale authorization`이라는 테스트 이름에 `db`, `integration`, `slow` 같은 정보를 끼워 넣으면 이름이 길어지고 검색 기준도 불안정해집니다.

```js
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';

describe('payment refund flow', { tags: ['payment'] }, () => {
  it('refunds an authorized charge', { tags: ['integration', 'db'] }, async () => {
    const refund = await refundAuthorizedCharge('charge_123');

    assert.strictEqual(refund.status, 'succeeded');
  });
});
```

이 예제에서 테스트 이름은 동작을 설명하고, tag는 실행 축을 설명합니다.
`payment`는 도메인, `integration`은 테스트 종류, `db`는 필요한 환경을 나타냅니다.
이렇게 나누면 이름을 바꾸더라도 CI 필터 기준은 덜 흔들립니다.

### H3. suite tag는 하위 테스트로 상속된다

공식 문서에 따르면 suite에 붙인 tag는 자식 테스트로 이어지고, 자식 테스트의 tag와 합쳐집니다.
공통 도메인이나 공통 환경은 `describe()`에 붙이고, 개별 테스트의 성격만 `it()`이나 `test()`에 추가하면 중복을 줄일 수 있습니다.

```js
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';

describe('catalog search', { tags: ['search'] }, () => {
  it('returns exact title matches', () => {
    assert.deepStrictEqual(searchCatalog('node'), ['node handbook']);
  });

  it('reads indexed documents', { tags: ['integration', 'db'] }, async () => {
    const results = await searchIndexedDocuments('test runner');

    assert.ok(results.length > 0);
  });
});
```

첫 번째 테스트는 `search` tag만 갖습니다.
두 번째 테스트는 `search`, `integration`, `db` tag를 함께 갖습니다.
suite 단위 상속을 활용하면 "이 파일은 어떤 영역의 테스트인가"와 "이 테스트는 어떤 실행 조건이 필요한가"를 동시에 표현할 수 있습니다.

## 태그 필터 실행법

### H3. --experimental-test-tag-filter로 필요한 묶음만 실행한다

tag 필터는 literal tag 이름을 기준으로 동작합니다.
공식 문서 기준으로 `--experimental-test-tag-filter`를 한 번 지정하면 해당 tag를 가진 테스트만 실행됩니다.
tag가 없는 테스트는 필터가 있는 실행에서 제외됩니다.

```bash
node --test --experimental-test-tag-filter=integration
```

```bash
node --test --experimental-test-tag-filter=db
```

이 명령은 각각 `integration` 또는 `db` tag가 포함된 테스트만 실행합니다.
로컬 개발에서는 빠른 테스트만 기본 명령으로 돌리고, 통합 테스트는 명시적인 명령으로 실행하는 식으로 나눌 수 있습니다.
중요한 점은 tag가 없는 테스트가 필터 실행에 포함되지 않는다는 점입니다.

### H3. 필터를 여러 번 쓰면 모두 만족해야 한다

공식 문서에 따르면 `--experimental-test-tag-filter`를 여러 번 지정하면 테스트는 모든 필터를 만족해야 실행됩니다.
즉 여러 tag는 OR가 아니라 AND 조건으로 해석됩니다.
이 동작은 "db 테스트 전체"와 "db이면서 integration인 테스트"를 분리할 때 유용합니다.

```bash
node --test \
  --experimental-test-tag-filter=db \
  --experimental-test-tag-filter=integration
```

위 명령은 `db`와 `integration`을 모두 가진 테스트만 실행합니다.
`db`만 가진 테스트나 `integration`만 가진 테스트는 제외됩니다.
팀 문서에는 이 규칙을 분명히 적어 두는 편이 좋습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:integration": "node --test --experimental-test-tag-filter=integration",
    "test:db": "node --test --experimental-test-tag-filter=db",
    "test:db:integration": "node --test --experimental-test-tag-filter=db --experimental-test-tag-filter=integration"
  }
}
```

스크립트 이름을 명확히 만들면 개발자는 매번 플래그를 기억하지 않아도 됩니다.
특히 experimental 플래그는 오타가 나기 쉬우므로 `package.json`에 고정해 두는 편이 실무적으로 안전합니다.
CI에서도 같은 스크립트를 호출하면 로컬과 배포 파이프라인의 실행 기준이 일치합니다.

## name pattern과 tags의 차이

### H3. 이름 검색은 임시 디버깅에 적합하다

`--test-name-pattern`은 테스트 이름을 JavaScript 정규식으로 필터링합니다.
특정 실패를 빠르게 재현하거나, 이름에 포함된 기능명을 기준으로 잠시 좁혀 실행할 때 유용합니다.
하지만 테스트 이름은 리팩터링이나 문장 개선 과정에서 바뀔 수 있으므로 장기적인 CI 정책으로 쓰기에는 취약합니다.

```bash
node --test --test-name-pattern="/refund/i"
```

이 명령은 이름에 `refund`가 들어간 테스트를 찾는 데 좋습니다.
하지만 환불 테스트가 `returns money to the customer`처럼 이름이 바뀌면 필터가 깨질 수 있습니다.
이런 실행 정책은 `tags: ['refund']` 또는 `tags: ['payment']`처럼 별도 메타데이터로 옮기는 편이 낫습니다.

### H3. tag는 반복되는 실행 정책에 적합하다

tag는 테스트 이름보다 안정적인 분류 기준입니다.
팀이 자주 사용하는 축을 미리 정하면 CI 구성이 단순해집니다.
예를 들어 다음처럼 tag vocabulary를 작게 유지할 수 있습니다.

```text
recommended tags:
- unit: 빠르게 실행되는 순수 단위 테스트
- integration: 여러 모듈이나 외부 자원을 함께 검증하는 테스트
- db: 데이터베이스가 필요한 테스트
- slow: 기본 로컬 루프에서 제외하고 싶은 테스트
- flaky: 안정화 전까지 별도 감시가 필요한 테스트
```

tag를 너무 많이 만들면 오히려 분류가 어려워집니다.
`api`, `service`, `domain`처럼 의미가 겹치는 tag가 늘어나면 어떤 필터를 써야 할지 팀마다 다르게 해석할 수 있습니다.
처음에는 실행 비용, 외부 의존성, 도메인 정도만 작게 시작하는 것이 좋습니다.

## CI에서 운영하는 방법

### H3. 기본 테스트와 통합 테스트 단계를 나눈다

CI에서는 빠른 피드백과 충분한 검증을 모두 챙겨야 합니다.
기본 단계에서는 tag 필터 없이 빠른 테스트를 포함해 전체 기본 검증을 돌리고, 통합 테스트는 환경 준비 후 별도 단계에서 실행하는 방식이 실용적입니다.
단, 기본 명령에 모든 테스트가 포함되면 느린 테스트까지 매번 실행될 수 있으므로 프로젝트 정책을 명확히 정해야 합니다.

```yaml
name: test

on:
  pull_request:

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 26
      - run: npm ci
      - run: npm test

  integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 26
      - run: npm ci
      - run: npm run test:integration
```

이 예시는 구조를 보여 주기 위한 최소 형태입니다.
실제 프로젝트에서는 데이터베이스 서비스, 환경 변수, seed 데이터, 캐시 전략을 함께 구성해야 합니다.
tag 필터는 실행 대상을 고르는 도구일 뿐, 테스트 환경을 자동으로 준비해 주지는 않습니다.

### H3. flaky tag는 숨기기보다 격리와 추적에 쓴다

`flaky` tag는 실패를 무시하기 위한 면죄부가 아닙니다.
불안정한 테스트를 기본 경로에서 잠시 분리하더라도, 별도 CI 단계에서 계속 실행하고 원인을 추적해야 합니다.
그렇지 않으면 중요한 회귀가 조용히 방치될 수 있습니다.

```json
{
  "scripts": {
    "test:flaky": "node --test --experimental-test-tag-filter=flaky",
    "test:stable": "node --test --test-skip-pattern=\"\\\\[flaky\\\\]\""
  }
}
```

위 예시처럼 이름 패턴과 tag를 섞는 방식은 과도기에는 쓸 수 있습니다.
하지만 최종적으로는 `flaky`도 tag 기준으로 관리하고, 안정화되면 tag를 제거하는 흐름이 더 깔끔합니다.
불안정한 테스트에는 이슈 번호나 원인 메모를 남겨 "언젠가 고칠 것"이 아니라 추적 가능한 작업으로 만들어야 합니다.

## 도입할 때 주의할 점

### H3. Node.js 버전을 고정한다

test tags는 early development 상태로 문서화된 기능입니다.
따라서 도입할 때는 로컬 개발 환경과 CI의 Node.js 버전을 맞추는 것이 중요합니다.
한 개발자는 tag 필터가 동작하고 다른 개발자는 플래그를 인식하지 못하면 테스트 명령 자체가 신뢰를 잃습니다.

```bash
node --version
```

```json
{
  "engines": {
    "node": ">=26.2.0"
  }
}
```

프로젝트 정책에 따라 `engines`, `.nvmrc`, Volta, mise 같은 도구 중 하나로 버전을 명시하세요.
CI의 `node-version`도 같은 기준으로 맞추는 편이 좋습니다.
experimental 기능을 쓰는 동안에는 Node.js 릴리스 노트를 확인하고, 변경이 생기면 테스트 스크립트를 함께 갱신해야 합니다.

### H3. tag 이름은 소문자와 짧은 단어로 통일한다

공식 문서 기준으로 tag 값은 비어 있지 않은 문자열이어야 하며, 매칭은 대소문자를 구분하지 않습니다.
그래도 팀 내부 규칙은 소문자 kebab-case나 짧은 단어로 고정하는 편이 좋습니다.
`DB`, `database`, `db-test`가 섞이면 사람에게는 다른 의미처럼 보이기 때문입니다.

```js
// good
test('creates a user profile', { tags: ['db', 'integration'] }, async () => {});

// avoid
test('creates a user profile', { tags: ['DB', 'database-test', 'IntegrationTest'] }, async () => {});
```

기계가 대소문자를 맞춰 준다고 해서 사람이 읽기 좋은 규칙까지 해결되는 것은 아닙니다.
tag는 테스트 실행 명령과 문서, CI 화면에 반복해서 등장합니다.
처음부터 작고 일관된 vocabulary를 정해 두면 이후 정리가 훨씬 쉬워집니다.

## 실무 도입 체크리스트

### H3. Node.js test runner tags를 적용하기 전 확인할 것

- tag 기능을 지원하는 Node.js 버전을 로컬과 CI에서 모두 쓰는가?
- tag가 테스트 이름을 대신하지 않고 실행 정책만 표현하는가?
- `unit`, `integration`, `db`, `slow`, `flaky`처럼 작은 vocabulary로 시작했는가?
- `package.json` scripts에 자주 쓰는 tag 필터 명령을 고정했는가?
- tag 필터가 AND 조건으로 동작한다는 점을 팀 문서에 남겼는가?
- `flaky` tag를 무시가 아니라 격리와 추적 용도로 쓰는가?

test tags는 테스트를 더 많이 나누기 위한 장식이 아니라, 실행 비용과 환경 조건을 명확히 관리하기 위한 도구입니다.
이름 기반 필터는 디버깅에 남겨 두고, 반복되는 CI 정책은 tag로 옮기면 테스트 명령이 더 안정적으로 유지됩니다.
experimental 상태라는 점만 잊지 말고 Node.js 버전과 스크립트를 함께 고정하면, 큰 테스트 스위트에서도 빠른 피드백과 깊은 검증을 균형 있게 운영할 수 있습니다.

## 함께 보면 좋은 글

- [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)
- [Node.js test runner skip, todo, only 가이드: 테스트 실행 범위를 안전하게 관리하기](/development/blog/seo/2026/07/01/nodejs-test-runner-skip-todo-only-focused-tests-guide.html)
- [Node.js test runner concurrency 가이드: 병렬 테스트로 CI 시간을 줄이는 법](/development/blog/seo/2026/06/30/nodejs-test-runner-concurrency-parallel-tests-guide.html)

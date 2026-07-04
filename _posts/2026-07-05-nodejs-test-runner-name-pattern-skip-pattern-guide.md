---
layout: post
title: "Node.js test runner name pattern 가이드: 테스트 이름으로 실행 범위를 좁히는 방법"
date: 2026-07-05 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-name-pattern-skip-pattern-guide
permalink: /development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html
alternates:
  ko: /development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html
  x_default: /development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, name-pattern, skip-pattern, testing, javascript, ci]
description: "Node.js test runner의 --test-name-pattern과 --test-skip-pattern으로 테스트 이름을 기준으로 실행 범위를 좁히는 방법을 정리합니다. 정규식 규칙, nested test 주의점, CI 재현 명령, tags 필터와의 차이를 예제로 설명합니다."
---

테스트 스위트가 커질수록 실패 원인을 빠르게 좁히는 능력이 중요해집니다.
항상 전체 테스트를 다시 실행하면 가장 정확할 수는 있지만, 작은 수정 하나를 확인하기에는 피드백이 너무 느립니다.
이때 테스트 이름을 기준으로 실행 범위를 줄이면 특정 기능, 특정 실패 케이스, 특정 하위 테스트만 빠르게 확인할 수 있습니다.

Node.js 내장 test runner는 이 용도로 `--test-name-pattern`과 `--test-skip-pattern` 옵션을 제공합니다.
[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#filtering-tests-by-name)에 따르면 두 옵션은 테스트 이름을 JavaScript 정규식으로 해석하고, 실행되는 테스트에 해당하는 hook도 함께 실행합니다.
또 [Node.js CLI 공식 문서](https://nodejs.org/api/cli.html#--test-name-pattern)는 `--test-name-pattern`이 이름과 일치하는 테스트만 실행하며, `--test-skip-pattern`을 함께 쓰면 두 조건을 모두 만족해야 실행된다고 설명합니다.

장기적인 테스트 분류 기준은 [Node.js test runner tags filter 가이드](/development/blog/seo/2026/07/02/nodejs-test-runner-tags-filter-guide.html)를 참고하세요.
실패한 테스트를 다시 실행하는 흐름은 [Node.js test runner rerun failures 가이드](/development/blog/seo/2026/07/03/nodejs-test-runner-rerun-failures-ci-guide.html)와 연결됩니다.
테스트 파일을 CI 작업으로 나누는 전략은 [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html)에서 이어서 볼 수 있습니다.

## name pattern이 필요한 순간

### H3. 실패한 테스트 이름만 빠르게 다시 실행한다

CI 로그에서 실패한 테스트 이름을 확인했다면 전체 스위트를 다시 돌리기 전에 해당 이름만 좁혀 볼 수 있습니다.
`--test-name-pattern`은 테스트 이름이 패턴과 맞는 항목만 실행합니다.
실패 재현 속도가 중요한 상황에서는 이 옵션이 가장 가벼운 첫 단계입니다.

```bash
node --test --test-name-pattern="creates user profile"
```

이 명령은 이름에 `creates user profile`이 맞는 테스트만 실행합니다.
테스트 파일 수가 많아도 실행 대상 테스트가 줄어들기 때문에 원인 확인이 빨라질 수 있습니다.
단, 공식 문서가 설명하듯 이 옵션은 테스트 파일 탐색 범위 자체를 바꾸지는 않습니다.

파일 탐색까지 줄이고 싶다면 테스트 파일 경로를 함께 넘기는 편이 좋습니다.
이렇게 하면 test runner가 확인해야 할 파일 수와 실행할 테스트 수를 동시에 줄일 수 있습니다.

```bash
node --test test/users/profile.test.js --test-name-pattern="creates user profile"
```

이 명령은 특정 파일 안에서 이름이 맞는 테스트만 실행합니다.
로컬 디버깅에서는 이 조합이 가장 직관적입니다.
CI에서는 실패 로그에 파일명과 테스트 이름이 함께 남도록 reporter 설정을 정리해 두면 재현 명령을 만들기 쉽습니다.

### H3. 특정 느린 테스트를 잠시 제외한다

`--test-skip-pattern`은 이름이 패턴과 맞는 테스트를 실행 대상에서 제외합니다.
로컬에서 빠른 확인이 필요하거나, 별도 통합 테스트 job으로 옮긴 케이스를 임시로 빼고 싶을 때 사용할 수 있습니다.

```bash
node --test --test-skip-pattern="integration"
```

이 명령은 이름이 `integration`과 맞는 테스트를 제외하고 실행합니다.
다만 장기 운영 정책을 테스트 이름 문자열에 계속 의존하는 것은 약합니다.
반복적으로 같은 범주를 제외해야 한다면 이름 패턴보다 tags나 별도 npm script로 옮기는 편이 안전합니다.

## 정규식 규칙 이해하기

### H3. 문자열은 JavaScript 정규식처럼 해석된다

`--test-name-pattern`에 넘기는 값은 단순 포함 검색처럼 보일 수 있지만 실제로는 정규식 패턴입니다.
따라서 `.`이나 `[]` 같은 문자는 정규식 의미를 가질 수 있습니다.
테스트 이름에 특수 문자가 들어간다면 의도와 다르게 넓게 매칭되지 않는지 확인해야 합니다.

```bash
node --test --test-name-pattern="GET /users/[0-9]+"
```

이 명령은 숫자 id가 붙은 사용자 조회 테스트 이름을 찾는 식으로 사용할 수 있습니다.
정규식을 의도적으로 쓰면 유연하지만, 팀원이 읽기 어려운 패턴은 재현성을 떨어뜨립니다.
CI 문서나 pull request 코멘트에 남길 명령은 가능한 한 명확한 문자열을 우선하세요.

대소문자를 무시해야 한다면 정규식 literal 형태를 사용할 수 있습니다.
공식 문서는 `/pattern/i`처럼 flag가 포함된 패턴도 사용할 수 있다고 설명합니다.

```bash
node --test --test-name-pattern="/payment/i"
```

이 명령은 `payment`, `Payment`, `PAYMENT`처럼 대소문자가 다른 이름을 함께 찾습니다.
테스트 이름 표기가 오래된 프로젝트에서 섞여 있다면 유용할 수 있습니다.
하지만 새 테스트를 작성할 때는 이름 표기 자체를 일관되게 맞추는 편이 더 낫습니다.

### H3. 여러 패턴을 넘길 수 있다

공식 문서에 따르면 `--test-name-pattern`과 `--test-skip-pattern`은 여러 번 지정할 수 있습니다.
여러 이름을 한 번에 확인하거나 nested test를 좁혀야 할 때 사용할 수 있습니다.

```bash
node --test \
  --test-name-pattern="creates user profile" \
  --test-name-pattern="updates user profile"
```

이 방식은 실패한 테스트가 두세 개일 때 빠르게 재현 범위를 만들기 좋습니다.
패턴 하나에 복잡한 정규식을 몰아넣는 것보다 명령이 읽기 쉬울 때도 많습니다.
다만 너무 많은 패턴을 CLI에 늘어놓아야 한다면 별도 script나 실패 재실행 파일을 쓰는 편이 낫습니다.

## nested test에서 조심할 점

### H3. 부모 테스트가 매칭되지 않으면 자식 테스트도 실행되지 않을 수 있다

nested test를 사용할 때는 부모 테스트 이름까지 고려해야 합니다.
공식 문서는 부모 테스트가 name pattern에 맞지 않으면 자식 테스트 이름이 맞더라도 실행되지 않을 수 있다고 설명합니다.
즉 하위 테스트 이름만 보고 패턴을 만들면 의도한 테스트가 빠질 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('user profile', async (t) => {
  await t.test('creates profile', () => {
    assert.equal(1 + 1, 2);
  });

  await t.test('updates profile', () => {
    assert.equal(2 + 2, 4);
  });
});
```

이 구조에서 하위 테스트 하나만 실행하고 싶다면 부모 이름을 함께 포함하는 패턴이 더 안전합니다.
테스트 이름이 계층 구조를 드러내도록 작성하면 CLI 필터링도 쉬워집니다.

```bash
node --test --test-name-pattern="user profile creates profile"
```

공식 문서는 조상 테스트 이름을 공백으로 이어 붙이면 특정 하위 테스트를 더 고유하게 지정할 수 있다고 설명합니다.
같은 이름의 `creates profile` 테스트가 여러 suite에 있을 때 특히 중요합니다.
테스트 이름은 사람이 읽는 문서이면서 동시에 CLI 필터링 키가 됩니다.

### H3. 중복 이름은 디버깅 비용을 키운다

여러 suite에 같은 하위 테스트 이름이 반복되면 name pattern으로 하나만 고르기 어렵습니다.
`works`, `returns error`, `handles empty input` 같은 이름은 단독으로는 너무 넓습니다.
실패 로그와 재현 명령을 읽기 좋게 만들려면 테스트 이름에 행위와 조건을 함께 넣는 편이 좋습니다.

```js
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';

describe('profile email validation', () => {
  it('rejects missing email', () => {
    assert.throws(() => validateEmail(''));
  });

  it('accepts verified email address', () => {
    assert.equal(validateEmail('hello@example.com'), true);
  });
});

function validateEmail(value) {
  if (!value.includes('@')) {
    throw new Error('invalid email');
  }

  return true;
}
```

이름이 구체적이면 `--test-name-pattern="profile email validation rejects missing email"`처럼 재현 명령도 명확해집니다.
테스트 이름을 짧게만 쓰는 습관은 초기에는 편하지만, 실패가 누적되면 오히려 시간을 더 쓰게 만듭니다.
테스트 이름은 CI 로그의 검색 키워드라고 생각하는 편이 실무에 맞습니다.

## CI에서 사용하는 기준

### H3. pull request 재현 명령을 남긴다

CI에서 실패한 테스트를 로컬에서 빠르게 재현하려면 명령이 그대로 복사 가능해야 합니다.
실패 로그에는 Node.js 버전, 테스트 파일, name pattern, skip pattern 여부가 함께 남는 것이 좋습니다.
GitHub Actions에서는 실패한 테스트를 수동 재현할 때 사용할 예시 명령을 job summary에 남길 수 있습니다.

```yaml
name: focused-test

on:
  workflow_dispatch:
    inputs:
      pattern:
        description: "Test name pattern"
        required: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 26
      - run: npm ci
      - run: npm run test:ci -- --test-name-pattern="${{ inputs.pattern }}"
```

이 workflow는 실패한 테스트 이름을 입력으로 받아 focused test를 실행합니다.
항상 필요한 구성은 아니지만 테스트 스위트가 큰 저장소에서는 재현 시간을 줄이는 데 도움이 됩니다.
입력값은 정규식으로 해석되므로 팀 내부 문서에 예시와 주의점을 같이 남겨 두세요.

### H3. skip pattern을 영구 우회로 쓰지 않는다

`--test-skip-pattern`은 문제를 잠시 우회하는 데 유용하지만, 실패 테스트를 장기간 숨기는 장치가 되면 위험합니다.
특정 테스트가 자주 실패해서 제외해야 한다면 이슈를 만들고 원인, 제외 범위, 복구 조건을 남겨야 합니다.
skip pattern은 임시 운영 도구이지 품질 기준을 낮추는 방법이 아닙니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:fast": "node --test --test-skip-pattern=\"/integration|slow/i\""
  }
}
```

이런 script는 로컬 빠른 확인용으로는 괜찮습니다.
하지만 main 브랜치 필수 체크가 `test:fast`만 실행한다면 중요한 통합 테스트가 계속 빠질 수 있습니다.
필수 CI에서는 전체 테스트, tag 기반 통합 테스트, shard 실행을 조합해 실제 품질 신호가 남도록 설계하세요.

## tags 필터와의 차이

### H3. name pattern은 임시 검색에 강하다

name pattern은 테스트 이름만 있으면 바로 쓸 수 있습니다.
새 메타데이터를 추가하지 않아도 되고, 실패 로그에서 본 이름을 그대로 재현 명령으로 바꿀 수 있습니다.
따라서 특정 실패를 좁히는 디버깅 작업에 잘 맞습니다.

```bash
node --test --test-name-pattern="refund"
```

이 명령은 이름에 `refund`가 들어간 테스트를 빠르게 찾습니다.
문제는 이름이 바뀌면 실행 범위도 같이 바뀐다는 점입니다.
테스트 분류 정책을 이름 문자열에 오래 묶어 두면 리팩터링 때 CI 동작이 조용히 달라질 수 있습니다.

### H3. 반복 실행 정책은 tags로 옮긴다

반복적으로 실행 범위를 나눠야 한다면 tags가 더 적합합니다.
예를 들어 `db`, `integration`, `slow`, `payment`처럼 성격이 명확한 축은 이름보다 메타데이터로 표현하는 편이 안정적입니다.
이름은 테스트가 무엇을 검증하는지 설명하고, tag는 어떤 실행 정책에 속하는지 설명하게 나누면 됩니다.

```js
import { describe, it } from 'node:test';

describe('payment refund', { tags: ['payment'] }, () => {
  it('refunds captured charge', { tags: ['integration', 'db'] }, async () => {
    // integration test body
  });
});
```

```bash
node --test --experimental-test-tag-filter=integration
```

name pattern과 tag filter는 서로 경쟁하는 기능이 아닙니다.
디버깅에는 name pattern, 반복 운영에는 tags를 쓰는 식으로 역할을 나누면 됩니다.
특히 CI 필수 체크는 이름 변경에 덜 흔들리는 tags나 파일 경로 기준을 우선 검토하세요.

## 운영 체크리스트

### H3. name pattern을 도입할 때 확인할 것

테스트 이름 필터링은 가벼운 기능이지만, 팀 단위로 쓰려면 이름 작성 규칙이 필요합니다.
아래 기준을 맞추면 실패 로그와 로컬 재현 명령이 훨씬 읽기 쉬워집니다.

- 테스트 이름에 기능, 조건, 기대 결과가 드러나는가?
- nested test는 부모 이름까지 이어도 자연스럽게 읽히는가?
- 중복된 하위 테스트 이름이 너무 많지 않은가?
- CI 로그에서 파일명과 테스트 이름을 함께 확인할 수 있는가?
- 반복 실행 정책을 name pattern에 과하게 의존하지 않는가?

이 기준은 테스트 자체의 품질에도 영향을 줍니다.
좋은 테스트 이름은 실패했을 때 무엇이 깨졌는지 바로 알려 주고, name pattern은 그 이름을 실행 도구로 바꿔 줍니다.
결국 중요한 것은 빠른 실행보다 빠른 이해입니다.

### H3. 안전하게 쓰는 기본 조합

로컬에서는 파일 경로와 name pattern을 함께 쓰고, CI에서는 tags나 shard와 조합하는 방식이 현실적입니다.
실패한 테스트 하나를 볼 때는 좁게 실행하고, 병합 전에는 전체 품질 신호를 다시 확인해야 합니다.

```bash
# local focused debugging
node --test test/payment/refund.test.js --test-name-pattern="refunds captured charge"

# CI category run
node --test --experimental-test-tag-filter=integration

# CI parallel run
node --test --test-shard=1/4
```

`--test-name-pattern`은 원인 분석을 빠르게 시작하는 도구입니다.
`--test-skip-pattern`은 잠시 범위를 줄이는 도구입니다.
둘 다 편리하지만, 최종 품질 판단은 전체 테스트와 명확한 CI 정책 위에서 내려야 합니다.

## 마무리

Node.js test runner의 `--test-name-pattern`과 `--test-skip-pattern`은 큰 테스트 스위트에서 실패 범위를 빠르게 줄이는 실용적인 옵션입니다.
정규식 규칙, nested test의 부모 이름, skip pattern의 임시성만 이해하면 로컬 디버깅과 CI 재현 흐름을 훨씬 단순하게 만들 수 있습니다.

장기 운영 기준은 이름 문자열에만 기대지 말고 tags, shard, rerun failures 같은 기능과 나누어 설계하세요.
테스트 이름은 사람이 실패를 이해하는 문장이고, name pattern은 그 문장을 실행 가능한 검색 조건으로 바꾸는 연결점입니다.

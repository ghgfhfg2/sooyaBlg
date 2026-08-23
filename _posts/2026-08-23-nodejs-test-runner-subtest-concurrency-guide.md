---
layout: post
title: "Node.js test runner subtest 가이드: 테스트 구조와 병렬 실행을 안정적으로 나누는 법"
date: 2026-08-23 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-subtest-concurrency-guide
permalink: /development/blog/seo/2026/08/23/nodejs-test-runner-subtest-concurrency-guide.html
alternates:
  ko: /development/blog/seo/2026/08/23/nodejs-test-runner-subtest-concurrency-guide.html
  x_default: /development/blog/seo/2026/08/23/nodejs-test-runner-subtest-concurrency-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, subtest, concurrency, testing, javascript, ci, backend]
description: "Node.js test runner에서 subtest로 테스트 구조를 나누고 병렬 실행을 안전하게 관리하는 방법을 정리합니다. 중첩 테스트, 공유 상태 제거, CI 안정성, 실패 로그 가독성까지 실무 기준으로 설명합니다."
---

테스트가 많아질수록 어려운 문제는 "테스트를 어떻게 작성하느냐"보다 **어떻게 읽히고, 어떻게 실패하게 만들 것인가**입니다.
파일 하나에 `test()`가 길게 늘어서 있으면 처음에는 단순하지만, 기능 단위와 엣지 케이스가 섞이면서 실패 원인을 찾기 어려워집니다.
반대로 구조를 지나치게 잘게 쪼개면 실행 순서와 공유 상태가 숨어 CI에서만 깨지는 테스트가 생길 수 있습니다.

Node.js 내장 test runner는 `test()` 안에서 다시 테스트를 만드는 subtest 패턴을 지원합니다.
subtest를 잘 쓰면 기능 단위, 시나리오 단위, 입력 케이스 단위를 계층적으로 표현할 수 있고, 실패 로그도 더 읽기 쉬워집니다.
이 글에서는 Node.js test runner에서 subtest를 구성하는 기준과 병렬 실행을 다룰 때 피해야 할 함정을 정리합니다.
시간 의존 테스트는 [Node.js test runner mock timers 가이드](/development/blog/seo/2026/08/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html), 객체 속성 교체 테스트는 [Node.js test runner mock.property 가이드](/development/blog/seo/2026/08/18/nodejs-test-runner-mock-property-config-test-guide.html), 커버리지 운영은 [Node.js test runner coverage 가이드](/development/blog/seo/2026/05/10/nodejs-test-runner-coverage-v8-guide.html)와 함께 보면 좋습니다.

## Subtest를 쓰는 이유

### 실패 로그에 제품 구조를 남긴다

subtest의 가장 큰 장점은 테스트 결과에 기능 구조가 그대로 남는다는 점입니다.
예를 들어 사용자 설정 API를 테스트한다면 `validation`, `authorization`, `persistence` 같은 큰 묶음을 만들고 그 아래에 세부 케이스를 둘 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function normalizeSettings(input) {
  return {
    theme: input.theme ?? 'system',
    emailEnabled: input.emailEnabled !== false
  };
}

test('normalizeSettings', async (t) => {
  await t.test('uses defaults when fields are missing', () => {
    assert.deepEqual(normalizeSettings({}), {
      theme: 'system',
      emailEnabled: true
    });
  });

  await t.test('keeps explicit opt-out values', () => {
    assert.deepEqual(normalizeSettings({ emailEnabled: false }), {
      theme: 'system',
      emailEnabled: false
    });
  });
});
```

이 구조는 테스트 이름만 봐도 어떤 기능의 어떤 조건이 깨졌는지 알려줍니다.
특히 CI 로그에서 여러 테스트가 동시에 실패할 때, 상위 테스트 이름은 실패 범위를 빠르게 좁히는 힌트가 됩니다.

### 준비 코드와 검증 코드를 가까이 둔다

subtest는 공통 준비 코드를 상위 테스트 안에 두고, 세부 검증을 하위 테스트로 나누기 좋습니다.
다만 이때도 공유 객체를 직접 변경하는 방식은 조심해야 합니다.
하위 테스트가 같은 객체를 바꾸면 실행 순서에 따라 결과가 달라질 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function buildPayload(overrides = {}) {
  return {
    name: 'demo',
    retries: 3,
    timeoutMs: 1000,
    ...overrides
  };
}

test('build job payload', async (t) => {
  await t.test('accepts a custom timeout', () => {
    const payload = buildPayload({ timeoutMs: 5000 });
    assert.equal(payload.timeoutMs, 5000);
  });

  await t.test('keeps the default retry count', () => {
    const payload = buildPayload();
    assert.equal(payload.retries, 3);
  });
});
```

핵심은 "상위 테스트에 공유 상태를 둔다"가 아니라 "상위 테스트에 공유 맥락을 둔다"입니다.
데이터는 각 subtest 안에서 새로 만들고, 설명과 헬퍼만 공유하는 편이 안정적입니다.

## 병렬 실행에서 주의할 점

### await 없이 만든 subtest는 흐름을 흐리게 만든다

Node.js test runner에서 subtest를 만들 때는 `await t.test(...)` 형태를 기본값으로 두는 것이 읽기 쉽습니다.
하위 테스트의 완료를 상위 테스트가 기다린다는 의도가 명확해지고, cleanup이나 mock 복구 시점도 예측하기 쉬워집니다.

```js
test('report formatter', async (t) => {
  await t.test('formats success rows', () => {
    assert.equal(formatStatus('ok'), 'SUCCESS');
  });

  await t.test('formats failure rows', () => {
    assert.equal(formatStatus('failed'), 'FAILED');
  });
});
```

작은 테스트에서는 `await`를 빼도 당장 문제 없어 보일 수 있습니다.
하지만 상위 테스트에 비동기 cleanup, mock 복구, 임시 파일 삭제가 들어가면 완료 시점이 섞여 추적하기 어려워집니다.
팀 규칙으로 `async (t)`와 `await t.test()`를 통일해두면 리뷰 비용이 줄어듭니다.

### 병렬 테스트는 공유 자원을 먼저 제거한다

병렬 실행은 테스트 시간을 줄이는 데 도움이 되지만, 파일 시스템, 포트, 환경 변수, 전역 mock, 임시 데이터베이스처럼 공유 자원이 있으면 불안정해집니다.
subtest를 병렬로 돌리기 전에 아래 조건을 먼저 확인하세요.

- 각 테스트가 고유한 임시 디렉터리를 쓰는가
- 환경 변수 변경을 테스트 안에서 복구하는가
- 같은 포트를 동시에 열지 않는가
- mock, timer, global 객체 변경이 테스트 밖으로 새지 않는가
- 외부 API나 실제 데이터베이스에 의존하지 않는가

아래처럼 케이스별 입력과 결과만 검증하는 순수 함수 테스트는 병렬화하기 쉽습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

const cases = [
  ['retryable timeout', { code: 'ETIMEDOUT' }, true],
  ['retryable reset', { code: 'ECONNRESET' }, true],
  ['non retryable validation', { code: 'VALIDATION_ERROR' }, false]
];

test('isRetryableError', async (t) => {
  for (const [name, error, expected] of cases) {
    await t.test(String(name), () => {
      assert.equal(isRetryableError(error), expected);
    });
  }
});
```

먼저 순수한 케이스 테이블을 만들고, 그 다음 실행 시간을 보고 병렬 옵션을 검토하는 순서가 안전합니다.
테스트가 느린 이유가 외부 의존성이라면 병렬화보다 의존성 제거가 먼저입니다.

## Subtest 설계 기준

### 기능 단위와 케이스 단위를 섞지 않는다

좋은 subtest 구조는 깊이가 많아서 좋은 것이 아니라, 계층마다 의미가 분명해서 좋습니다.
상위 테스트는 기능이나 공개 API를 표현하고, 하위 테스트는 관찰 가능한 동작을 표현하는 식으로 나누면 유지보수가 쉽습니다.

권장 구조는 아래와 같습니다.

- 1단계: 모듈, 함수, 엔드포인트 같은 테스트 대상
- 2단계: 성공, 실패, 권한, 경계값 같은 시나리오
- 3단계: 입력 케이스 테이블이 필요할 때만 추가

반대로 구현 세부사항을 그대로 테스트 이름에 넣으면 리팩터링 때 테스트가 과하게 흔들립니다.
`uses Map internally`보다 `deduplicates repeated ids`가 더 오래 살아남는 이름입니다.

### 테스트 이름은 실패 메시지처럼 쓴다

테스트 이름은 문서 제목이 아니라 실패 로그의 첫 줄입니다.
따라서 "works"나 "case 1"처럼 짧기만 한 이름은 나중에 비용이 됩니다.

```js
await t.test('rejects negative retry count', () => {
  assert.throws(
    () => parseRetryCount('-1'),
    /retry count must be greater than or equal to 0/
  );
});
```

이름에는 입력 조건과 기대 동작을 함께 넣는 편이 좋습니다.
모든 세부 값을 다 넣을 필요는 없지만, 로그만 보고도 어떤 조건을 다시 실행해야 하는지 알 수 있어야 합니다.

### 헬퍼는 테스트 파일 안에서 작게 시작한다

subtest가 늘어나면 `setupUser()`, `createRequest()`, `expectValidationError()` 같은 헬퍼가 자연스럽게 생깁니다.
처음부터 공용 테스트 유틸 패키지로 빼기보다 해당 테스트 파일 안에서 작게 시작하는 것이 좋습니다.

공유 헬퍼로 승격할 기준은 명확합니다.
세 개 이상의 파일에서 같은 준비 코드가 반복되고, 그 코드가 도메인 규칙을 안정적으로 표현한다면 분리할 가치가 있습니다.
그 전에는 테스트 파일 가까이에 두는 편이 실패 원인을 추적하기 쉽습니다.

## CI에서 안정적으로 운영하는 방법

### 실패 재현 명령을 문서화한다

CI에서 깨진 테스트는 로컬에서 빠르게 재현할 수 있어야 합니다.
Node.js test runner는 파일 단위로 실행할 수 있으므로, 실패한 테스트 파일을 바로 실행하는 명령을 README나 기여 가이드에 남기면 좋습니다.

```bash
node --test test/settings.test.js
```

테스트 이름 패턴으로 좁힐 수 있는 환경이라면, 팀에서 쓰는 명령을 별도로 고정해두세요.
중요한 것은 CI 로그를 본 사람이 "어떤 명령을 복사하면 되는지" 고민하지 않게 만드는 것입니다.

### 전역 상태 변경은 cleanup과 한 세트로 둔다

환경 변수, 현재 작업 디렉터리, mock, fake timer, 임시 파일은 모두 테스트 밖으로 새기 쉽습니다.
subtest 안에서 전역 상태를 바꾼다면 같은 블록 안에 복구 코드를 둬야 합니다.

```js
await t.test('reads feature flag from env', () => {
  const previous = process.env.FEATURE_PREVIEW;
  process.env.FEATURE_PREVIEW = 'enabled';

  try {
    assert.equal(readPreviewFlag(), true);
  } finally {
    if (previous === undefined) {
      delete process.env.FEATURE_PREVIEW;
    } else {
      process.env.FEATURE_PREVIEW = previous;
    }
  }
});
```

cleanup이 테스트 바깥에 흩어져 있으면 subtest가 늘어날수록 누락이 생깁니다.
상태를 바꾸는 코드와 복구하는 코드를 가까이 두는 것이 가장 단순한 안정화 전략입니다.

## 실무 체크리스트

### 새 테스트 파일을 만들 때 확인할 것

아래 기준을 통과하면 subtest 구조가 지나치게 복잡하거나 불안정해질 가능성이 줄어듭니다.

- 상위 테스트 이름이 공개 API나 기능 단위를 나타낸다
- 하위 테스트 이름이 입력 조건과 기대 동작을 함께 설명한다
- 각 subtest가 필요한 데이터를 직접 만든다
- 전역 상태 변경에는 복구 코드가 붙어 있다
- 병렬 실행 전에 공유 자원 의존성을 제거했다
- CI에서 파일 단위 재현 명령을 바로 알 수 있다

subtest는 테스트를 예쁘게 접는 도구가 아닙니다.
실패 원인을 좁히고, 변경 범위를 드러내고, CI 로그를 읽을 수 있게 만드는 구조화 도구입니다.
작게 시작하되 테스트 이름과 상태 격리를 엄격하게 지키면 Node.js 내장 test runner만으로도 꽤 단단한 테스트 체계를 만들 수 있습니다.

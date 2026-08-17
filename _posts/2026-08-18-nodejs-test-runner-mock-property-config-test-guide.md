---
layout: post
title: "Node.js test runner mock.property 가이드: 설정값과 전역 상태 테스트를 안전하게 격리하는 법"
date: 2026-08-18 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-mock-property-config-test-guide
permalink: /development/blog/seo/2026/08/18/nodejs-test-runner-mock-property-config-test-guide.html
alternates:
  ko: /development/blog/seo/2026/08/18/nodejs-test-runner-mock-property-config-test-guide.html
  x_default: /development/blog/seo/2026/08/18/nodejs-test-runner-mock-property-config-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, node-test, mock-property, test-runner, configuration, testing, javascript, backend]
description: "Node.js test runner의 mock.property로 설정 객체, feature flag, 전역 상태를 테스트 안에서 안전하게 바꾸는 방법을 정리합니다. 직접 대입과의 차이, 자동 복원, 접근 횟수 검증, process.env 대체 패턴까지 실무 예제로 설명합니다."
---

테스트에서 설정값을 바꾸는 일은 흔합니다.
예를 들어 결제 기능이 feature flag에 따라 꺼지는지, 운영 환경에서만 로그 레벨이 달라지는지, 특정 옵션이 없을 때 기본값으로 돌아가는지 확인해야 합니다.
문제는 이런 값이 대개 모듈 전역 객체나 공유 설정 객체에 들어 있다는 점입니다.

값을 직접 대입하고 테스트 끝에서 원래 값으로 되돌리는 방식은 처음에는 단순해 보입니다.
하지만 실패한 assertion, 중간 throw, 병렬 테스트, 누락된 cleanup이 겹치면 다음 테스트까지 상태가 새어 나갑니다.
Node.js test runner의 `t.mock.property()`는 이런 공유 속성 변경을 테스트 컨텍스트 안에 묶어 두고, 접근 기록과 복원까지 함께 다룰 수 있게 해 줍니다.

Node.js 공식 문서 기준으로 `mock.property()`는 `v24.3.0`과 `v22.20.0`에 추가된 API입니다.
이미 `node:test`를 쓰고 있다면 별도 mocking 라이브러리 없이 설정 객체의 속성값을 잠깐 바꾸고, 테스트가 끝난 뒤 원래 값으로 복원하는 구조를 만들 수 있습니다.
기본 test runner 흐름은 [Node.js test runner 기본 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html), 함수 mock 검증은 [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html), 테스트 컨텍스트 활용은 [Node.js test runner TestContext 가이드](/development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html)와 함께 보면 좋습니다.

## mock.property가 필요한 상황

### 설정 객체를 직접 대입하면 복원이 흐려진다

가장 흔한 패턴은 설정 객체의 속성을 테스트 안에서 바꾸는 것입니다.
아래 코드는 기능 플래그에 따라 베타 메뉴를 노출할지 결정합니다.

```js
// config.js
export const config = {
  enableBetaMenu: false,
  logLevel: 'info'
};

// navigation.js
import { config } from './config.js';

export function getNavigationItems() {
  const items = ['home', 'settings'];

  if (config.enableBetaMenu) {
    items.push('beta');
  }

  return items;
}
```

직접 대입으로 테스트하면 다음처럼 보일 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { config } from './config.js';
import { getNavigationItems } from './navigation.js';

test('shows beta menu when the flag is enabled', () => {
  const previous = config.enableBetaMenu;
  config.enableBetaMenu = true;

  assert.deepStrictEqual(getNavigationItems(), ['home', 'settings', 'beta']);

  config.enableBetaMenu = previous;
});
```

이 코드는 정상 경로에서는 동작합니다.
하지만 assertion 전에 예외가 발생하면 마지막 복원 코드가 실행되지 않습니다.
`try/finally`를 매번 쓰면 해결할 수 있지만, 테스트가 많아질수록 반복이 커지고 리뷰에서 놓치기 쉽습니다.

### 테스트 컨텍스트에 mock을 묶어 둔다

`t.mock.property()`를 쓰면 같은 테스트를 더 분명하게 쓸 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { config } from './config.js';
import { getNavigationItems } from './navigation.js';

test('shows beta menu when the flag is enabled', (t) => {
  t.mock.property(config, 'enableBetaMenu', true);

  assert.deepStrictEqual(getNavigationItems(), ['home', 'settings', 'beta']);
});
```

테스트 컨텍스트의 mock tracker는 테스트가 끝난 뒤 자신이 만든 mock을 정리합니다.
따라서 테스트 본문은 "이 테스트에서 어떤 값을 보고 싶은가"에 집중할 수 있습니다.
복원 코드가 사라지기 때문에, 설정 변경이 테스트 밖으로 새어 나갈 가능성도 줄어듭니다.

전역 `mock` export도 있지만, 공유 상태를 다루는 테스트에서는 가능하면 `t.mock`을 우선으로 두는 편이 안전합니다.
테스트 단위의 수명과 mock의 수명이 같아져서 cleanup 책임이 더 명확해집니다.

## 직접 대입과 다른 점

### 접근 횟수까지 검증할 수 있다

`mock.property()`가 반환하는 proxy에는 `mock` 컨텍스트가 붙습니다.
이를 통해 속성이 몇 번 읽혔는지, 어떤 값으로 쓰였는지 확인할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('reads logLevel while building logger options', (t) => {
  const runtime = { logLevel: 'warn' };
  const mocked = t.mock.property(runtime, 'logLevel', 'debug');

  function buildLoggerOptions() {
    return {
      level: runtime.logLevel,
      pretty: runtime.logLevel === 'debug'
    };
  }

  assert.deepStrictEqual(buildLoggerOptions(), {
    level: 'debug',
    pretty: true
  });

  assert.strictEqual(mocked.mock.accessCount(), 2);
  assert.strictEqual(mocked.mock.accesses[0].type, 'get');
  assert.strictEqual(mocked.mock.accesses[0].value, 'debug');
});
```

일반적인 테스트에서는 결과값만 검증해도 충분합니다.
하지만 성능상 반복 접근을 줄여야 하거나, lazy getter가 예상보다 여러 번 호출되는 문제를 추적할 때는 접근 기록이 유용합니다.
단, 접근 횟수 검증은 구현 세부에 가까우므로 모든 테스트에 남발하지 않는 것이 좋습니다.

### 쓰기까지 감시할 수 있다

설정 객체를 읽기만 하는 코드라면 문제가 단순합니다.
반대로 초기화 함수가 설정 객체를 보정하거나 migration하는 구조라면 "어떤 값으로 썼는지"도 확인하고 싶을 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function normalizeRuntime(runtime) {
  if (runtime.region == null) {
    runtime.region = 'ap-northeast-2';
  }

  return runtime;
}

test('fills default region when runtime region is missing', (t) => {
  const runtime = { region: undefined };
  const mocked = t.mock.property(runtime, 'region');

  normalizeRuntime(runtime);

  assert.strictEqual(runtime.region, 'ap-northeast-2');
  assert.strictEqual(mocked.mock.accesses.at(-1).type, 'get');
  assert.strictEqual(
    mocked.mock.accesses.some((access) => access.type === 'set' && access.value === 'ap-northeast-2'),
    true
  );
});
```

이 예시는 쓰기 이벤트를 확인하기 위해 access log를 봅니다.
실무에서는 이런 테스트를 핵심 설정 보정 로직에만 제한하는 편이 좋습니다.
일반 비즈니스 로직 테스트까지 내부 접근 순서를 촘촘히 고정하면 리팩터링 비용이 커집니다.

## 설정 테스트 구조 잡기

### process.env를 직접 mock하기보다 읽기 계층을 분리한다

환경 변수 테스트에서 가장 먼저 떠오르는 대상은 `process.env`입니다.
하지만 `process.env`는 Node.js 런타임 전체가 공유하는 특수 객체입니다.
테스트마다 직접 값을 넣고 지우는 방식은 병렬 실행과 섞이면 불안정해질 수 있습니다.

더 나은 구조는 환경 변수 읽기를 작은 계층으로 감싸고, 애플리케이션 코드는 정규화된 설정 객체만 보게 만드는 것입니다.

```js
// runtime-config.js
export function readRuntimeConfig(env = process.env) {
  return {
    nodeEnv: env.NODE_ENV || 'development',
    apiBaseUrl: env.API_BASE_URL || 'http://localhost:3000',
    enableDebugPanel: env.ENABLE_DEBUG_PANEL === 'true'
  };
}
```

이 함수는 테스트에서 plain object로 검증할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { readRuntimeConfig } from './runtime-config.js';

test('reads debug panel flag from env-like object', () => {
  const config = readRuntimeConfig({
    NODE_ENV: 'test',
    API_BASE_URL: 'https://api.example.test',
    ENABLE_DEBUG_PANEL: 'true'
  });

  assert.deepStrictEqual(config, {
    nodeEnv: 'test',
    apiBaseUrl: 'https://api.example.test',
    enableDebugPanel: true
  });
});
```

이 방식은 `mock.property()`가 필요 없는 좋은 설계입니다.
의존성을 함수 인자로 받을 수 있다면, 공유 전역을 mock하지 않는 쪽이 더 단순합니다.

### 이미 공유 객체가 있다면 mock.property로 경계를 좁힌다

현실의 코드베이스에는 이미 공유 설정 객체가 존재할 수 있습니다.
한 번 import된 객체를 여러 모듈이 바라보고 있고, 당장 구조를 바꾸기 어렵다면 `mock.property()`로 테스트 경계를 좁힐 수 있습니다.

```js
// runtime.js
export const runtime = {
  env: 'production',
  region: 'ap-northeast-2'
};

// cache-key.js
import { runtime } from './runtime.js';

export function buildCacheKey(userId) {
  return `${runtime.env}:${runtime.region}:user:${userId}`;
}
```

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { runtime } from './runtime.js';
import { buildCacheKey } from './cache-key.js';

test('builds cache key for staging runtime', (t) => {
  t.mock.property(runtime, 'env', 'staging');
  t.mock.property(runtime, 'region', 'us-east-1');

  assert.strictEqual(buildCacheKey('42'), 'staging:us-east-1:user:42');
});
```

핵심은 mock 대상을 좁게 잡는 것입니다.
`process.env` 전체를 바꾸는 대신 `runtime.env`처럼 애플리케이션이 실제로 의존하는 작은 객체를 바꿉니다.
이렇게 하면 테스트 의도가 선명해지고, 민감정보가 섞인 환경 전체를 로그나 assertion failure에 노출할 위험도 줄어듭니다.

## 병렬 테스트에서 주의할 점

### 같은 객체를 동시에 바꾸는 테스트는 격리한다

`t.mock.property()`가 테스트 종료 후 복원을 도와줘도, 같은 객체의 같은 속성을 여러 테스트가 동시에 바꾸면 충돌할 수 있습니다.
병렬 테스트에서는 공유 객체 자체가 경쟁 자원이 됩니다.

다음처럼 같은 전역 runtime 객체를 여러 테스트가 동시에 바꾸는 구조는 피하는 편이 좋습니다.

```js
test('staging cache key', (t) => {
  t.mock.property(runtime, 'env', 'staging');
  assert.match(buildCacheKey('1'), /^staging:/);
});

test('production cache key', (t) => {
  t.mock.property(runtime, 'env', 'production');
  assert.match(buildCacheKey('1'), /^production:/);
});
```

테스트 runner 설정에 따라 이 두 테스트가 같은 프로세스에서 겹치면 결과가 흔들릴 수 있습니다.
공유 객체를 mock해야 한다면 해당 파일의 테스트를 순차 실행하도록 구성하거나, 더 좋은 방법으로 함수 인자 주입 구조를 추가하는 것이 좋습니다.

```js
export function buildCacheKey(userId, runtimeConfig = runtime) {
  return `${runtimeConfig.env}:${runtimeConfig.region}:user:${userId}`;
}
```

이제 테스트는 공유 객체를 건드리지 않고 독립 입력을 넘길 수 있습니다.

```js
test('staging cache key without shared mutation', () => {
  assert.strictEqual(
    buildCacheKey('1', { env: 'staging', region: 'us-east-1' }),
    'staging:us-east-1:user:1'
  );
});
```

`mock.property()`는 레거시 경계나 전역 값이 필요한 테스트에 좋은 도구입니다.
하지만 모든 설정 테스트를 mock으로 풀어야 한다는 뜻은 아닙니다.
새 코드라면 의존성 주입이 더 오래 버티는 구조가 될 때가 많습니다.

### import 시점에 읽힌 값은 나중에 바꿔도 반영되지 않을 수 있다

속성 mock에서 자주 헷갈리는 부분은 import 시점입니다.
모듈이 로드될 때 설정값을 읽어 상수로 저장했다면, 테스트 중에 원본 객체의 속성을 바꿔도 이미 계산된 값은 바뀌지 않습니다.

```js
// bad-cache-key.js
import { runtime } from './runtime.js';

const prefix = `${runtime.env}:${runtime.region}`;

export function buildCacheKey(userId) {
  return `${prefix}:user:${userId}`;
}
```

이 구조에서는 `runtime.env`를 mock해도 `prefix`는 그대로입니다.
테스트가 실패하면 `mock.property()`가 동작하지 않는 것처럼 보이지만, 실제 원인은 값이 너무 이른 시점에 캐시된 것입니다.

운영 설정처럼 배포 중 바뀌지 않는 값은 import 시점 캐싱이 맞을 수 있습니다.
반대로 테스트에서 환경별 동작을 자주 확인해야 하거나 런타임 중 갱신될 수 있는 값이라면, 함수 호출 시점에 읽도록 설계하는 편이 좋습니다.

## 실무 적용 체크리스트

### 구현 전 확인할 것

- 새 코드라면 공유 객체 mock보다 함수 인자 주입으로 테스트할 수 있는지 먼저 본다.
- 기존 공유 설정 객체를 건드려야 한다면 `t.mock.property()`로 테스트 컨텍스트에 묶는다.
- `process.env` 전체를 출력하거나 assertion failure에 민감값이 섞이지 않게 한다.
- 같은 속성을 바꾸는 테스트가 병렬로 실행될 가능성이 있는지 확인한다.
- import 시점에 값이 캐시되는 구조인지, 호출 시점에 읽는 구조인지 구분한다.

### 리뷰에서 볼 것

- 직접 대입 후 복원하는 코드가 `try/finally` 없이 남아 있지 않은가?
- 전역 `mock` 대신 테스트 컨텍스트의 `t.mock`을 쓸 수 있지 않은가?
- 접근 횟수 검증이 꼭 필요한 테스트에만 쓰였는가?
- 공유 상태 변경 테스트가 다른 테스트와 순서 의존성을 만들지 않는가?
- 설정 테스트가 비밀값이나 실제 운영 엔드포인트를 요구하지 않는가?

## 자주 묻는 질문

### mock.property는 객체에 없는 속성도 만들 수 있나요?

테스트 대상 객체의 속성값을 mock하는 용도로 보는 것이 안전합니다.
없는 속성을 임시로 만드는 테스트는 실제 코드의 계약을 흐리게 만들 수 있습니다.
새 옵션의 존재 여부를 테스트해야 한다면, 설정 파서 함수에 plain object 입력을 넘겨 검증하는 구조가 더 읽기 쉽습니다.

### process.env에도 바로 써도 되나요?

기술적으로는 객체 속성을 다루는 방식으로 접근할 수 있지만, 추천 우선순위는 낮습니다.
`process.env`는 프로세스 전체가 공유하는 상태라서 병렬 테스트와 섞이면 흔들리기 쉽습니다.
환경 변수를 읽는 함수를 따로 두고, 테스트에서는 env-like object를 넘기는 편이 더 안정적입니다.

### mock.property와 mock.method는 언제 나누나요?

값 속성을 바꿀 때는 `mock.property()`를 씁니다.
함수 호출을 감시하거나 구현을 바꿔야 한다면 `mock.method()`나 `mock.fn()`이 맞습니다.
설정값, feature flag, region, mode처럼 "읽는 값"은 property mock이 더 자연스럽고, 외부 API 호출 함수나 logger 메서드는 method mock이 더 자연스럽습니다.

## 마무리

`mock.property()`는 테스트 안에서 공유 속성값을 바꾸는 일을 더 안전하게 만들어 줍니다.
직접 대입과 수동 복원 대신 테스트 컨텍스트에 변경을 묶고, 필요하면 접근 기록까지 확인할 수 있습니다.

다만 이 API가 공유 상태 설계를 모두 해결해 주지는 않습니다.
새 코드에서는 설정을 함수 인자로 주입하거나 작은 설정 읽기 계층으로 분리하는 편이 더 단단합니다.
이미 전역 설정 객체가 널리 쓰이는 코드라면 `t.mock.property()`로 변경 범위를 줄이고, 병렬 실행과 import 시점 캐싱을 함께 점검하는 것이 현실적인 출발점입니다.

## 함께 보면 좋은 글

- [Node.js test runner 기본 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)
- [Node.js test runner mock.fn 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html)
- [Node.js test runner TestContext 가이드](/development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html)

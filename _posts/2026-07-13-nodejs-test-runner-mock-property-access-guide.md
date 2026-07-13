---
layout: post
title: "Node.js test runner mock.property 가이드: getter와 설정값 접근을 안전하게 검증하는 법"
date: 2026-07-13 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-mock-property-access-guide
permalink: /development/blog/seo/2026/07/13/nodejs-test-runner-mock-property-access-guide.html
alternates:
  ko: /development/blog/seo/2026/07/13/nodejs-test-runner-mock-property-access-guide.html
  x_default: /development/blog/seo/2026/07/13/nodejs-test-runner-mock-property-access-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, mock-property, getter, setter, testing, javascript, unit-test]
description: "Node.js test runner의 mock.property로 getter, setter, 설정 객체 접근을 안전하게 검증하는 방법을 정리합니다. 접근 횟수, mockImplementationOnce, restore 기준과 테스트 격리 패턴을 예제로 설명합니다."
---

테스트에서 함수 호출만 검증하면 충분한 경우가 많습니다.
하지만 실제 애플리케이션 코드는 함수보다 설정 객체, getter, 런타임 플래그, lazy property에 의존하는 경우도 자주 만납니다.
이때 값을 직접 덮어쓰면 테스트는 간단해 보이지만, 원래 동작을 복구하지 못하거나 접근 여부를 확인하기 어려워질 수 있습니다.

Node.js 내장 test runner의 `mock.property()`는 이런 상황을 다룰 수 있는 도구입니다.
객체의 property 값을 테스트 안에서 바꾸고, 읽기와 쓰기 접근이 몇 번 일어났는지 확인하며, 테스트가 끝난 뒤 원래 상태로 되돌리는 흐름을 만들 수 있습니다.
특히 설정값을 읽는 코드, getter를 통해 계산된 값을 노출하는 객체, setter 호출 여부가 중요한 코드에서 유용합니다.

이 글에서는 `mock.property()`로 property 접근을 검증하는 기준, `accesses`와 `accessCount()`를 읽는 법, `mockImplementationOnce()`를 사용할 때의 주의점, 테스트 격리를 지키는 패턴을 정리합니다.
함수 호출 검증은 [Node.js test runner mock.fn call verification 가이드](/development/blog/seo/2026/07/09/nodejs-test-runner-mock-fn-call-verification-guide.html), 메서드 복구 흐름은 [Node.js test runner mock.method restore 가이드](/development/blog/seo/2026/07/09/nodejs-test-runner-mock-method-restore-guide.html), 전역 상태 정리는 [Node.js test runner beforeEach/afterEach 가이드](/development/blog/seo/2026/07/10/nodejs-test-runner-beforeeach-aftereach-hooks-guide.html)와 함께 보면 좋습니다.

## mock.property가 필요한 순간

### H3. 설정 객체를 읽는 코드를 검증할 때

애플리케이션 코드는 보통 설정 객체를 직접 참조합니다.
예를 들어 기능 플래그가 켜졌을 때만 새 로직을 타는 함수가 있다고 가정해 보겠습니다.

```js
export const runtimeConfig = {
  enableFastCheckout: false
};

export function selectCheckoutFlow() {
  return runtimeConfig.enableFastCheckout ? 'fast' : 'classic';
}
```

이 코드를 테스트할 때 `runtimeConfig.enableFastCheckout = true`처럼 직접 값을 바꿀 수도 있습니다.
하지만 테스트 중간에 실패하면 원래 값 복구가 빠질 수 있고, 다른 테스트가 같은 객체를 참조하면 순서 의존성이 생길 수 있습니다.
`mock.property()`를 쓰면 테스트 스코프 안에서 의도를 더 분명하게 표현할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

const runtimeConfig = {
  enableFastCheckout: false
};

function selectCheckoutFlow() {
  return runtimeConfig.enableFastCheckout ? 'fast' : 'classic';
}

test('uses fast checkout when the feature flag is enabled', (t) => {
  t.mock.property(runtimeConfig, 'enableFastCheckout', true);

  assert.equal(selectCheckoutFlow(), 'fast');
});
```

테스트 컨텍스트의 `t.mock`으로 만든 mock은 해당 테스트가 끝날 때 정리됩니다.
그래도 공유 객체를 많이 다루는 코드라면 테스트가 서로 같은 객체를 만지지 않도록 구조를 작게 유지하는 편이 좋습니다.

### H3. getter 접근 자체가 의미 있을 때

일부 객체는 property를 읽는 순간 계산이나 캐시 조회를 수행합니다.
이런 객체를 테스트할 때는 결과값뿐 아니라 실제로 property를 읽었는지도 확인하고 싶을 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function buildInvoiceSummary(settings) {
  if (settings.locale === 'ko-KR') {
    return '원화 청구서';
  }

  return 'Invoice';
}

test('reads locale setting while building invoice summary', (t) => {
  const settings = { locale: 'en-US' };
  const locale = t.mock.property(settings, 'locale', 'ko-KR');

  const summary = buildInvoiceSummary(settings);

  assert.equal(summary, '원화 청구서');
  assert.equal(locale.mock.accessCount(), 1);
});
```

여기서 `accessCount()`는 property가 읽히거나 쓰인 횟수를 반환합니다.
단순히 최종 결과만 보는 테스트보다, 설정값을 실제로 참조했는지까지 확인할 수 있습니다.
다만 접근 횟수 검증은 구현 세부 사항에 가까울 수 있으므로 남용하지 않는 것이 좋습니다.

## accesses로 읽기와 쓰기 흐름 확인하기

### H3. 어떤 값이 읽혔는지 확인한다

`mock.property()`가 반환하는 context에는 접근 기록이 남습니다.
`accesses`를 보면 각 접근의 `type`과 `value`를 확인할 수 있습니다.
읽기 접근은 `type`이 `get`이고, 쓰기 접근은 `set`입니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function renderCurrencyLabel(preferences) {
  return preferences.currency === 'KRW' ? '원' : '$';
}

test('records property get access', (t) => {
  const preferences = { currency: 'USD' };
  const currency = t.mock.property(preferences, 'currency', 'KRW');

  assert.equal(renderCurrencyLabel(preferences), '원');

  assert.equal(currency.mock.accesses.length, 1);
  assert.equal(currency.mock.accesses[0].type, 'get');
  assert.equal(currency.mock.accesses[0].value, 'KRW');
});
```

이 방식은 설정 객체를 읽는 분기 로직을 확인할 때 유용합니다.
다만 `accesses`는 배열 복사본을 반환하므로 단순 횟수만 필요하다면 `accessCount()`가 더 간결합니다.
로그나 assertion 메시지에 실제 사용자 설정값을 그대로 남기지 않는 기준도 유지해야 합니다.

### H3. setter 호출을 검증한다

property가 쓰였는지 확인해야 하는 코드도 있습니다.
예를 들어 초기화 함수가 런타임 상태를 갱신하는지 검증할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function markReady(state) {
  state.ready = true;
}

test('records property set access', (t) => {
  const state = { ready: false };
  const ready = t.mock.property(state, 'ready', false);

  markReady(state);

  assert.equal(ready.mock.accessCount(), 1);
  assert.equal(ready.mock.accesses[0].type, 'set');
  assert.equal(ready.mock.accesses[0].value, true);
});
```

setter 검증은 상태 변경이 핵심 계약일 때만 사용하는 편이 좋습니다.
사용자에게 보이는 결과가 더 중요한 함수라면 최종 결과를 먼저 검증하고, 접근 기록은 보조 근거로 두는 것이 읽기 좋습니다.

## mockImplementationOnce 사용 기준

### H3. 한 번만 다른 값을 반환한다

특정 접근에서만 다른 값을 돌려줘야 할 때는 `mockImplementationOnce()`를 사용할 수 있습니다.
예를 들어 첫 번째 읽기에서는 기능 플래그가 꺼져 있고, 두 번째 읽기에서는 켜져 있는 상황을 재현할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function collectFlowNames(config) {
  return [
    config.enableFastCheckout ? 'fast' : 'classic',
    config.enableFastCheckout ? 'fast' : 'classic'
  ];
}

test('changes a property value for one access', (t) => {
  const config = { enableFastCheckout: false };
  const flag = t.mock.property(config, 'enableFastCheckout', false);

  flag.mock.mockImplementationOnce(true);

  assert.deepEqual(collectFlowNames(config), ['fast', 'classic']);
});
```

이 테스트는 첫 번째 접근에서만 `true`를 반환하고 이후에는 기본 mock 값인 `false`로 돌아옵니다.
일시적인 런타임 변화를 검증할 때 편하지만, 읽는 사람이 흐름을 바로 이해하기 어렵다면 케이스를 나눠 쓰는 편이 더 낫습니다.

### H3. get과 set이 모두 접근으로 기록된다는 점을 기억한다

`mockImplementationOnce()`는 다음 읽기만 대상으로 삼는 도구가 아닙니다.
property 쓰기도 접근으로 계산됩니다.
따라서 한 번만 다른 값을 반환하려고 했는데 중간에 setter가 먼저 실행되면 그 setter가 once 값을 소비할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function updateThenRead(config) {
  config.mode = 'manual';
  return config.mode;
}

test('set access also consumes one-time property behavior', (t) => {
  const config = { mode: 'auto' };
  const mode = t.mock.property(config, 'mode', 'auto');

  mode.mock.mockImplementationOnce('preview');

  assert.equal(updateThenRead(config), 'manual');
  assert.equal(mode.mock.accessCount(), 2);
  assert.equal(mode.mock.accesses[0].type, 'set');
  assert.equal(mode.mock.accesses[1].type, 'get');
});
```

이런 테스트는 일부러 동작을 보여주기 위한 예시입니다.
실무 코드에서는 get과 set이 섞인 property에 once 동작을 넣기보다, 테스트 대상을 더 작은 함수로 나누거나 명시적인 의존성을 주입하는 편이 안정적입니다.

## 테스트 격리를 지키는 패턴

### H3. 전역 설정 객체는 테스트마다 복구 기준을 둔다

공유 객체를 mock할 때 가장 중요한 기준은 복구입니다.
테스트 컨텍스트의 `t.mock`을 사용하면 테스트 종료 시 mock이 정리되지만, 여러 하위 테스트가 같은 객체를 동시에 다루는 구조라면 여전히 충돌할 수 있습니다.
가능하면 테스트마다 새 객체를 만들거나, 공유 객체를 직접 넘기는 함수 형태로 바꾸는 것이 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function createPaymentLabel(config) {
  return config.currency === 'KRW' ? '국내 결제' : '해외 결제';
}

test('creates a Korean payment label', (t) => {
  const config = { currency: 'USD' };
  t.mock.property(config, 'currency', 'KRW');

  assert.equal(createPaymentLabel(config), '국내 결제');
});

test('creates a default payment label', () => {
  const config = { currency: 'USD' };

  assert.equal(createPaymentLabel(config), '해외 결제');
});
```

각 테스트가 자신의 `config` 객체를 만들면 병렬 실행에서도 서로 영향을 주지 않습니다.
이 구조는 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/07/13/nodejs-test-runner-concurrency-parallel-test-guide.html)에서 다룬 공유 자원 격리 기준과도 이어집니다.

### H3. 구현 세부 사항보다 계약을 먼저 검증한다

property mock은 강력하지만, 모든 property 접근을 테스트해야 한다는 뜻은 아닙니다.
접근 횟수와 접근 순서는 구현 방식이 조금만 바뀌어도 달라질 수 있습니다.
테스트가 너무 세밀하면 리팩터링을 방해합니다.

다음 순서로 검증 범위를 정하면 과한 테스트를 줄일 수 있습니다.

```text
1. 사용자에게 의미 있는 결과를 먼저 검증한다.
2. 설정값 접근이 핵심 계약일 때만 accessCount나 accesses를 확인한다.
3. get과 set이 섞인 property는 once 동작보다 명시적인 테스트 데이터로 분리한다.
4. 공유 객체를 mock했다면 테스트 종료 후 복구와 병렬 실행 영향을 확인한다.
```

예를 들어 `config.locale`을 두 번 읽는지 한 번 읽는지는 대개 중요하지 않습니다.
하지만 캐시 warm-up 함수가 lazy getter를 정확히 한 번만 호출해야 성능 계약을 지킨다면 접근 횟수 검증이 의미를 가질 수 있습니다.
테스트의 목적이 결과인지, 부수효과인지, 성능상 접근 제한인지 먼저 정해야 합니다.

## CI에서 property mock을 운영하는 체크리스트

### H3. Node.js 버전과 API 안정성을 확인한다

`mock.property()`와 property access 기록은 비교적 최근 Node.js test runner API에 속합니다.
팀의 로컬 Node.js와 CI Node.js 버전이 다르면 로컬에서는 통과하지만 CI에서 실패할 수 있습니다.
`package.json`의 `engines`, `.nvmrc`, CI setup action 버전을 맞춰 두는 편이 안전합니다.

```json
{
  "engines": {
    "node": ">=22.20.0"
  },
  "scripts": {
    "test": "node --test"
  }
}
```

버전 기준은 프로젝트 상황에 맞게 정해야 합니다.
중요한 점은 테스트 코드가 요구하는 Node.js 기능과 CI 런타임이 같은 방향을 보도록 만드는 것입니다.

### H3. 실패 로그에 민감정보를 남기지 않는다

property mock은 설정 객체를 자주 다룹니다.
설정 객체에는 API 엔드포인트, tenant ID, 토큰 이름, 사용자 식별자가 섞이기 쉽습니다.
테스트용 값이라도 실패 메시지에 그대로 출력하면 CI 로그에 남습니다.

```js
assert.equal(access.mock.accessCount(), 1, 'config value should be read once');
```

위처럼 메시지는 의도를 설명하는 수준으로 충분합니다.
실제 토큰, 이메일, 세션 값, 내부 URL을 assertion 메시지에 넣지 않는 기준을 지키는 것이 좋습니다.
테스트가 실패했을 때 필요한 정보는 케이스 이름, 검증 대상, 기대한 동작 정도로 제한해도 대부분의 문제를 좁힐 수 있습니다.

## 정리

`mock.property()`는 Node.js test runner에서 객체 property 의존성을 테스트하기 위한 실용적인 도구입니다.
설정값을 일시적으로 바꾸고, getter와 setter 접근을 기록하며, 테스트 종료 후 원래 동작으로 복구하는 흐름을 만들 수 있습니다.

다만 property 접근 검증은 구현 세부 사항에 가까워지기 쉽습니다.
먼저 사용자에게 의미 있는 결과를 검증하고, 접근 횟수나 순서가 실제 계약일 때만 `accesses`와 `accessCount()`를 사용하세요.
공유 설정 객체를 다룬다면 테스트마다 객체를 분리하고, CI Node.js 버전을 맞추고, 실패 로그에 민감정보가 남지 않도록 점검하는 것이 안전합니다.

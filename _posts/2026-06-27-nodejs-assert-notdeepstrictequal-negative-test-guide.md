---
layout: post
title: "Node.js assert.notDeepStrictEqual 가이드: 같으면 안 되는 객체를 검증하는 테스트 패턴"
date: 2026-06-27 08:00:00 +0900
lang: ko
translation_key: nodejs-assert-notdeepstrictequal-negative-test-guide
permalink: /development/blog/seo/2026/06/27/nodejs-assert-notdeepstrictequal-negative-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/27/nodejs-assert-notdeepstrictequal-negative-test-guide.html
  x_default: /development/blog/seo/2026/06/27/nodejs-assert-notdeepstrictequal-negative-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, notDeepStrictEqual, test, assertion, object, backend]
description: "Node.js assert.notDeepStrictEqual()로 두 객체나 배열이 같아지면 안 되는 상황을 검증하는 방법을 정리합니다. deepStrictEqual과의 차이, 부정 검증이 필요한 사례, 깨지기 쉬운 테스트를 피하는 기준을 예제로 설명합니다."
---

테스트는 "기대한 값과 같다"만 확인하지 않습니다.
토큰이 매번 새로 발급되는지, 원본 객체가 의도치 않게 변경되지 않았는지, 서로 다른 입력이 같은 결과로 접히지 않는지도 확인해야 합니다.
이런 상황에서는 두 값이 같으면 실패해야 하므로 부정 assertion이 필요합니다.

Node.js `node:assert/strict`의 `assert.notDeepStrictEqual()`은 객체와 배열의 중첩 값이 deep strict equality 기준으로 같지 않은지 검증합니다.
[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)는 이 API가 두 값이 깊은 엄격 동등성을 만족하지 않아야 한다는 조건을 확인한다고 설명합니다.
값이 "다르다"는 사실 자체가 테스트 계약일 때 사용할 수 있는 도구입니다.

객체와 배열이 정확히 같아야 하는 테스트는 [Node.js assert.deepStrictEqual 가이드](/development/blog/seo/2026/06/26/nodejs-assert-deepstrictequal-object-comparison-test-guide.html)를 참고하세요.
일부 필드만 확인해야 한다면 [Node.js assert.partialDeepStrictEqual 가이드](/development/blog/seo/2026/06/23/nodejs-assert-partialdeepstrictequal-api-response-test-guide.html)가 더 적합합니다.
테스트에서 명시적인 실패 경로를 남기는 방법은 [Node.js assert.fail 가이드](/development/blog/seo/2026/06/26/nodejs-assert-fail-explicit-test-failure-guide.html)와 연결됩니다.

## notDeepStrictEqual이 필요한 이유

### H3. 객체가 같아지면 안 되는 회귀를 잡는다

서로 다른 입력이 우연히 같은 출력으로 변하면 비즈니스 로직의 구분이 사라질 수 있습니다.
예를 들어 권한별 메뉴, 요금제별 제한, 지역별 설정처럼 입력 차이가 결과 차이로 이어져야 하는 코드가 있습니다.
이때 결과가 완전히 같아지는 회귀를 `assert.notDeepStrictEqual()`로 잡을 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function buildPlanLimits(plan) {
  if (plan === 'pro') {
    return {
      projects: 50,
      seats: 10,
      exportFormats: ['csv', 'json']
    };
  }

  return {
    projects: 3,
    seats: 1,
    exportFormats: ['csv']
  };
}

test('free and pro plans expose different limits', () => {
  const freeLimits = buildPlanLimits('free');
  const proLimits = buildPlanLimits('pro');

  assert.notDeepStrictEqual(freeLimits, proLimits);
});
```

이 테스트는 세부 필드 하나하나를 설명하지 않습니다.
대신 두 요금제가 같은 제한값을 가지면 안 된다는 큰 계약을 표현합니다.
차이가 중요한 도메인에서는 이런 부정 검증이 간결한 안전장치가 됩니다.

### H3. 원본 객체 변형 여부를 확인한다

정렬, 정규화, 마스킹 함수는 새 객체를 반환해야 할 때가 많습니다.
원본 데이터가 그대로 유지되어야 하는 함수라면 결과 객체와 원본 객체가 달라지는지 함께 확인할 수 있습니다.
다만 이 경우에는 "참조가 다른지"와 "값이 다른지"를 구분해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function redactUser(user) {
  return {
    ...user,
    email: '[redacted]',
    token: '[redacted]'
  };
}

test('redactUser returns a redacted copy', () => {
  const original = {
    id: 'user_1',
    email: 'soo@example.com',
    token: 'test-token'
  };

  const redacted = redactUser(original);

  assert.notDeepStrictEqual(redacted, original);
  assert.deepStrictEqual(original, {
    id: 'user_1',
    email: 'soo@example.com',
    token: 'test-token'
  });
});
```

첫 번째 assertion은 마스킹 결과가 원본과 같은 값이 아니어야 한다는 점을 확인합니다.
두 번째 assertion은 원본 객체가 변경되지 않았다는 점을 확인합니다.
부정 검증만으로는 어떤 값이 안전한지까지 설명하기 어렵기 때문에, 중요한 계약은 긍정 검증과 함께 두는 편이 좋습니다.

## deepStrictEqual과 함께 쓰는 기준

### H3. 부정 검증은 의도를 좁게 써야 한다

`notDeepStrictEqual()`은 두 값이 다르기만 하면 통과합니다.
그래서 너무 넓게 쓰면 테스트가 실제 요구사항을 제대로 설명하지 못합니다.
예를 들어 "두 응답이 달라야 한다"만 확인하면 어떤 필드가 달라야 하는지는 놓칠 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function createSession(userId, issuedAt) {
  return {
    userId,
    sessionId: `${userId}-${issuedAt.getTime()}`,
    issuedAt: issuedAt.toISOString()
  };
}

test('new sessions use different session ids', () => {
  const first = createSession('user_1', new Date('2026-06-27T00:00:00.000Z'));
  const second = createSession('user_1', new Date('2026-06-27T00:01:00.000Z'));

  assert.notDeepStrictEqual(first, second);
  assert.notEqual(first.sessionId, second.sessionId);
});
```

전체 객체가 다르다는 검증은 보조 신호입니다.
핵심 계약은 `sessionId`가 달라야 한다는 점이므로 `assert.notEqual()`로 한 번 더 좁혀 줍니다.
부정 assertion은 구체적인 필드 검증과 함께 쓸수록 유지보수하기 쉽습니다.

### H3. 같아야 하는 구조는 deepStrictEqual로 고정한다

부정 검증이 필요한 테스트에서도 같아야 하는 부분은 명확히 고정해야 합니다.
다른 값이 생겼다는 사실만 확인하면, 의도하지 않은 필드까지 바뀌어도 테스트가 통과할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function rotateApiKey(account) {
  return {
    accountId: account.id,
    keyPrefix: account.keyPrefix,
    apiKey: `${account.keyPrefix}_new`
  };
}

test('rotateApiKey changes only the secret value', () => {
  const account = {
    id: 'acct_1',
    keyPrefix: 'demo_prefix'
  };

  const rotated = rotateApiKey(account);

  assert.notDeepStrictEqual(rotated, {
    accountId: 'acct_1',
    keyPrefix: 'demo_prefix',
    apiKey: 'demo_prefix_old'
  });

  assert.deepStrictEqual(
    {
      accountId: rotated.accountId,
      keyPrefix: rotated.keyPrefix
    },
    {
      accountId: 'acct_1',
      keyPrefix: 'demo_prefix'
    }
  );
});
```

새 키가 예전 키와 같으면 안 된다는 점은 부정 검증으로 표현합니다.
계정 식별자와 키 prefix가 유지되어야 한다는 점은 긍정 검증으로 고정합니다.
이렇게 나누면 테스트 실패 메시지를 보고 어떤 계약이 깨졌는지 빠르게 구분할 수 있습니다.

## 실무에서 조심할 점

### H3. 랜덤 값 테스트는 고정 가능한 입력으로 만든다

랜덤 토큰, 시간, UUID를 테스트할 때는 매번 다른 값을 기대하기 쉽습니다.
하지만 테스트가 진짜 랜덤에 의존하면 재현성이 떨어집니다.
가능하면 생성기를 주입하거나 시간 값을 고정해서 같은 입력에서는 같은 결과, 다른 입력에서는 다른 결과가 나오는지 확인하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function buildTraceContext(requestId, nonce) {
  return {
    requestId,
    traceId: `${requestId}:${nonce}`
  };
}

test('trace contexts differ when nonce changes', () => {
  const first = buildTraceContext('req_1', 'a');
  const second = buildTraceContext('req_1', 'b');

  assert.notDeepStrictEqual(first, second);
  assert.equal(first.requestId, second.requestId);
  assert.notEqual(first.traceId, second.traceId);
});
```

이 테스트는 진짜 난수를 만들지 않습니다.
대신 nonce를 명시적으로 바꿔 차이를 재현합니다.
불안정한 테스트를 줄이려면 무작위성 자체보다 무작위성을 받는 경계를 테스트하는 것이 좋습니다.

### H3. 스냅샷 대체용으로 남용하지 않는다

`notDeepStrictEqual()`은 스냅샷 테스트의 반대 버전이 아닙니다.
"예전 스냅샷과 다르면 된다"는 검증은 대부분 요구사항을 흐리게 만듭니다.
값이 달라져야 하는 이유를 코드로 표현할 수 있을 때만 사용하는 것이 좋습니다.

```js
import assert from 'node:assert/strict';

function assertDifferentPlanLimits(before, after) {
  assert.notDeepStrictEqual(before, after);
  assert.notEqual(before.projects, after.projects);
}
```

위 헬퍼처럼 전체 차이를 확인한 뒤 핵심 필드 차이를 좁혀 주면 의도가 분명해집니다.
반대로 어떤 필드가 달라야 하는지 설명할 수 없다면 테스트 설계를 다시 보는 편이 낫습니다.
부정 검증은 작은 보조 장치이지, 요구사항을 대신 설명하는 도구는 아닙니다.

## 마무리 체크리스트

`assert.notDeepStrictEqual()`은 같으면 안 되는 객체와 배열을 검증할 때 유용합니다.
서로 다른 입력이 같은 결과로 합쳐지는 회귀, 원본 객체와 마스킹 결과의 차이, 토큰이나 trace context처럼 값이 새로 만들어져야 하는 흐름을 확인할 수 있습니다.
다만 이 API는 "다르면 통과"하는 도구이므로, 중요한 필드는 `assert.notEqual()`이나 `assert.deepStrictEqual()`로 더 좁게 검증하는 것이 좋습니다.

발행 전에는 다음 기준으로 점검하면 안전합니다.

- 두 값이 달라야 하는 이유가 본문과 코드에 드러나는가?
- 같아야 하는 필드는 별도 assertion으로 고정했는가?
- 랜덤, 시간, 토큰 예시는 재현 가능한 입력으로 구성했는가?
- 민감정보처럼 보이는 값은 예시용 더미 문자열로만 작성했는가?
- 내부링크가 관련된 Node.js assert 글로 자연스럽게 연결되는가?

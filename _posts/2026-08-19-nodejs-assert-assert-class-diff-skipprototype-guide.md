---
layout: post
title: "Node.js Assert 클래스 가이드: 테스트 diff와 prototype 비교 옵션을 분리하는 법"
date: 2026-08-19 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-assert-class-diff-skipprototype-guide
permalink: /development/blog/seo/2026/08/19/nodejs-assert-assert-class-diff-skipprototype-guide.html
alternates:
  ko: /development/blog/seo/2026/08/19/nodejs-assert-assert-class-diff-skipprototype-guide.html
  x_default: /development/blog/seo/2026/08/19/nodejs-assert-assert-class-diff-skipprototype-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, node-test, testing, deepstrictequal, skipprototype, diff, javascript]
description: "Node.js node:assert의 Assert 클래스로 테스트별 assertion 설정을 분리하는 방법을 정리합니다. diff 옵션, strict 모드, skipPrototype, deepStrictEqual 비교 기준, 메서드 구조분해 주의점을 실무 예제로 설명합니다."
---

테스트 실패 메시지는 코드 품질만큼 중요합니다.
실패 원인이 빠르게 보이면 수정이 빨라지고, 반대로 diff가 너무 짧거나 비교 기준이 애매하면 작은 실패도 오래 끌립니다.
Node.js의 `node:assert/strict`는 기본 테스트에는 충분히 단순하지만, 어떤 테스트에서는 더 긴 diff가 필요하고, 어떤 테스트에서는 클래스 인스턴스의 prototype보다 직렬화된 데이터 모양을 비교하고 싶을 수 있습니다.

`node:assert`의 `Assert` 클래스는 이런 assertion 정책을 독립 인스턴스로 분리하는 API입니다.
Node.js 공식 문서 기준으로 `Assert` 클래스는 `v24.6.0`과 `v22.19.0`에 추가되었고, `skipPrototype` 옵션은 `v24.9.0`에 추가되었습니다.
`diff`, `strict`, `skipPrototype` 같은 설정을 한 곳에 묶어 특정 테스트 파일이나 helper에서만 사용할 수 있습니다.

이 글에서는 `Assert` 클래스를 언제 쓰면 좋은지, `diff: 'full'`과 `skipPrototype: true`가 어떤 문제를 해결하는지, 그리고 메서드 구조분해가 왜 위험한지 정리합니다.
객체 비교의 기본 기준은 [Node.js assert.deepStrictEqual 가이드](/development/blog/seo/2026/06/26/nodejs-assert-deepstrictequal-object-comparison-test-guide.html), 부분 객체 검증은 [Node.js assert.partialDeepStrictEqual 가이드](/development/blog/seo/2026/06/23/nodejs-assert-partialdeepstrictequal-api-response-test-guide.html), 커스텀 실패 메시지는 [Node.js AssertionError 가이드](/development/blog/seo/2026/07/15/nodejs-assert-assertionerror-custom-assertion-guide.html)와 함께 보면 좋습니다.

## Assert 클래스가 필요한 상황

### 전역 assert 정책을 바꾸고 싶지 않을 때

대부분의 테스트는 `node:assert/strict`를 그대로 쓰는 편이 좋습니다.
팀 전체가 같은 비교 기준을 공유하고, 코드도 짧습니다.

```js
import assert from 'node:assert/strict';

assert.deepStrictEqual(actualUser, expectedUser);
```

하지만 특정 테스트 묶음에서는 실패 diff를 더 자세히 보고 싶을 수 있습니다.
예를 들어 설정 파일 변환, API 응답 스냅샷, 큰 JSON 구조 비교처럼 어느 필드가 달라졌는지 바로 봐야 하는 경우입니다.
이때 모든 테스트의 출력 정책을 바꾸기보다 해당 파일에서만 별도 assertion 인스턴스를 만들 수 있습니다.

```js
import { Assert } from 'node:assert';

const assertVerbose = new Assert({
  diff: 'full'
});

assertVerbose.deepStrictEqual(actualConfig, expectedConfig);
```

이 코드는 "이 비교에서는 전체 diff가 필요하다"는 의도를 테스트 근처에 남깁니다.
기본 assert 사용 방식은 유지하면서, 진단이 필요한 테스트에만 더 자세한 실패 출력을 적용할 수 있습니다.

### 테스트 helper마다 비교 정책을 나눌 수 있다

공용 helper가 많아질수록 assertion 정책도 섞이기 쉽습니다.
도메인 객체 비교 helper, API 응답 비교 helper, 마이그레이션 결과 비교 helper가 모두 같은 `assert`를 직접 쓰면 나중에 출력 정책을 바꾸기 어렵습니다.

```js
import { Assert } from 'node:assert';

export const assertApi = new Assert({
  diff: 'full',
  strict: true
});

export const assertPlainData = new Assert({
  diff: 'simple',
  strict: true,
  skipPrototype: true
});
```

이렇게 나누면 helper 이름이 비교 목적을 드러냅니다.
`assertApi`는 API 응답처럼 실패 diff가 중요한 곳에 쓰고, `assertPlainData`는 class instance와 plain object를 데이터 모양 기준으로 비교해야 하는 좁은 구간에만 사용할 수 있습니다.

중요한 점은 이 패턴을 남발하지 않는 것입니다.
기본값에서 벗어난 assertion은 테스트를 읽는 사람에게 추가 해석 비용을 줍니다.
따라서 프로젝트 전역 규칙이 아니라, 명확한 이유가 있는 helper나 테스트 파일에만 두는 편이 좋습니다.

## diff 옵션으로 실패 원인 보기

### simple diff는 대부분의 테스트에 충분하다

`Assert` 인스턴스의 `diff` 옵션은 assertion 실패 메시지의 diff 길이를 조절합니다.
기본값은 `simple`입니다.
작은 객체나 원시값 비교에서는 기본 diff가 오히려 읽기 쉽습니다.

```js
import { Assert } from 'node:assert';

const assert = new Assert();

assert.deepStrictEqual(
  { id: 1, status: 'ready' },
  { id: 1, status: 'done' }
);
```

이런 비교에서는 실패 위치가 작고 분명합니다.
모든 assertion을 `diff: 'full'`로 바꾸면 CI 로그가 불필요하게 길어질 수 있습니다.
특히 병렬 테스트에서 여러 실패가 한 번에 나오면 긴 diff가 오히려 핵심 신호를 밀어낼 수 있습니다.

### full diff는 큰 구조 비교에 제한해서 쓴다

큰 JSON, 설정 객체, 코드 생성 결과처럼 중첩 구조가 큰 값은 실패 지점이 생략되면 조사 시간이 늘어납니다.
이때 `diff: 'full'`을 별도 인스턴스로 두면 필요한 테스트만 자세히 볼 수 있습니다.

```js
import { Assert } from 'node:assert';
import { test } from 'node:test';

const assertSnapshot = new Assert({
  diff: 'full'
});

test('builds deployment manifest', () => {
  const manifest = buildManifest({
    service: 'billing',
    region: 'ap-northeast-2'
  });

  assertSnapshot.deepStrictEqual(manifest, {
    service: 'billing',
    region: 'ap-northeast-2',
    healthCheck: {
      path: '/health',
      timeoutMs: 1000
    },
    rollout: {
      strategy: 'rolling',
      maxUnavailable: 1
    }
  });
});
```

이 예시에서 `full` diff는 manifest의 어느 하위 필드가 달라졌는지 확인하는 데 도움이 됩니다.
반대로 단순 boolean, 숫자, 짧은 배열 비교에는 굳이 적용할 필요가 없습니다.

### 실패 메시지에 민감정보가 섞이지 않게 한다

긴 diff는 편하지만, 출력되는 값도 많아집니다.
API 토큰, Authorization 헤더, 세션 쿠키, 사용자 이메일, 로컬 파일 경로가 비교 대상에 들어 있으면 CI 로그나 외부 리포트에 그대로 남을 수 있습니다.

```js
function redactRuntimeConfig(config) {
  return {
    ...config,
    databaseUrl: config.databaseUrl ? '[redacted]' : undefined,
    apiToken: config.apiToken ? '[redacted]' : undefined
  };
}

assertSnapshot.deepStrictEqual(
  redactRuntimeConfig(actualConfig),
  redactRuntimeConfig(expectedConfig)
);
```

테스트 출력은 개발자만 보는 것처럼 느껴지지만, 실제로는 CI artifact, PR comment, 실패 알림으로 복사될 수 있습니다.
`diff: 'full'`을 쓰는 helper에는 민감 필드를 제거하거나 fixture 자체를 안전한 값으로 만드는 규칙을 같이 둬야 합니다.

## skipPrototype 옵션 이해하기

### 기본 deepStrictEqual은 prototype도 비교한다

`deepStrictEqual()`은 객체의 열거 가능한 속성만 대충 비교하는 함수가 아닙니다.
엄격 비교에서는 객체의 prototype과 constructor 차이도 의미 있는 차이로 봅니다.

```js
import assert from 'node:assert/strict';

class UserDto {
  constructor(id, name) {
    this.id = id;
    this.name = name;
  }
}

const dto = new UserDto(1, 'Soo');
const plain = { id: 1, name: 'Soo' };

assert.deepStrictEqual(dto, plain);
```

두 값의 속성은 같아 보여도 이 assertion은 실패합니다.
하나는 `UserDto` 인스턴스이고, 다른 하나는 plain object이기 때문입니다.
도메인 모델의 타입이 계약에 포함된다면 이 실패는 좋은 신호입니다.
서비스 내부 객체를 검증하는 테스트에서는 prototype 차이를 무시하면 오히려 버그를 놓칠 수 있습니다.

### 직렬화된 데이터 비교에는 skipPrototype이 맞을 수 있다

반대로 어떤 테스트에서는 prototype이 계약이 아닙니다.
예를 들어 DB row mapper, JSON 직렬화 결과, 메시지 큐 payload처럼 "외부로 나가는 데이터 모양"만 확인하고 싶은 경우입니다.
이때 `skipPrototype: true`를 설정한 `Assert` 인스턴스를 사용할 수 있습니다.

```js
import { Assert } from 'node:assert';
import { test } from 'node:test';

const assertData = new Assert({
  strict: true,
  skipPrototype: true
});

class UserDto {
  constructor(id, name) {
    this.id = id;
    this.name = name;
  }
}

test('maps user record to public payload shape', () => {
  const payload = new UserDto(1, 'Soo');

  assertData.deepStrictEqual(payload, {
    id: 1,
    name: 'Soo'
  });
});
```

이 테스트의 관심사는 `payload`가 어떤 클래스인지가 아니라, 공개 payload에 필요한 필드가 맞는지입니다.
이런 경우에는 `skipPrototype`이 테스트 의도를 더 잘 표현할 수 있습니다.

그래도 기본값으로 두기에는 위험합니다.
prototype 차이는 보안 경계, validation 결과, Date와 plain object 구분, class invariant에서 중요한 의미를 가질 수 있습니다.
따라서 `skipPrototype: true`는 데이터 전송 객체, JSON shape, 테스트 fixture 비교처럼 범위를 좁힌 helper에만 두는 편이 안전합니다.

### util.isDeepStrictEqual과 기준을 맞춘다

Node.js `util.isDeepStrictEqual()`도 prototype 비교를 건너뛰는 옵션을 제공합니다.
애플리케이션 코드에서 boolean 비교가 필요하고, 테스트에서는 assertion error diff가 필요하다면 두 기준을 맞춰야 합니다.

```js
import { isDeepStrictEqual } from 'node:util';
import { Assert } from 'node:assert';

const assertPlainData = new Assert({
  strict: true,
  skipPrototype: true
});

export function hasSamePayloadShape(left, right) {
  return isDeepStrictEqual(left, right, true);
}

export function assertSamePayloadShape(left, right) {
  assertPlainData.deepStrictEqual(left, right);
}
```

이렇게 이름을 맞춰 두면 "prototype을 무시하는 비교"라는 정책이 숨지 않습니다.
`deepStrictEqual()`을 아무 옵션 없이 호출하는 코드와 섞이면 테스트마다 비교 기준이 달라져 헷갈릴 수 있으므로, helper 이름에 의도를 드러내는 편이 좋습니다.

## 구조분해 주의점

### Assert 메서드는 인스턴스 설정에 묶여 있다

`Assert` 클래스에서 가장 놓치기 쉬운 부분은 메서드 구조분해입니다.
공식 문서도 `Assert` 인스턴스의 assertion 메서드를 구조분해하면 인스턴스 설정과의 연결을 잃는다고 주의합니다.

아래 코드는 보기에는 자연스럽지만 피하는 편이 좋습니다.

```js
import { Assert } from 'node:assert';

const assertVerbose = new Assert({
  diff: 'full',
  skipPrototype: true
});

const { deepStrictEqual } = assertVerbose;

deepStrictEqual(actual, expected);
```

이렇게 메서드를 꺼내면 `diff`, `strict`, `skipPrototype` 같은 인스턴스 옵션이 기대대로 적용되지 않을 수 있습니다.
따라서 `Assert` 인스턴스를 쓸 때는 메서드를 직접 호출합니다.

```js
assertVerbose.deepStrictEqual(actual, expected);
```

조금 길어 보여도 이 방식이 더 안전합니다.
테스트 helper에 넘길 때도 메서드만 넘기기보다 wrapper 함수를 만드는 편이 낫습니다.

```js
export function assertGeneratedConfig(actual, expected) {
  assertVerbose.deepStrictEqual(actual, expected);
}
```

### import 이름으로 정책을 표현한다

파일 안에서 여러 assertion 정책을 섞어야 한다면 변수 이름을 구체적으로 짓는 것이 중요합니다.

```js
import assert from 'node:assert/strict';
import { Assert } from 'node:assert';

const assertFullDiff = new Assert({ diff: 'full' });
const assertPayloadShape = new Assert({
  strict: true,
  skipPrototype: true
});
```

이런 이름은 리뷰어에게 비교 기준을 알려 줍니다.
`assert2`, `customAssert`, `looseAssert`처럼 의미가 흐린 이름은 피하는 편이 좋습니다.
특히 `skipPrototype`은 느슨한 비교처럼 오해될 수 있으므로, "payload shape"나 "plain data"처럼 의도한 비교 대상을 이름에 넣는 것이 좋습니다.

## 실무 적용 기준

### 기본 assert를 먼저 유지한다

`Assert` 클래스는 테스트 전반을 새 방식으로 바꾸라는 신호가 아닙니다.
기본값은 여전히 `node:assert/strict`를 직접 쓰는 것입니다.

```js
import assert from 'node:assert/strict';

assert.equal(statusCode, 200);
assert.deepStrictEqual(body, expectedBody);
```

아래 상황에서만 별도 `Assert` 인스턴스를 검토하면 충분합니다.

- 큰 객체 비교에서 생략 없는 diff가 필요하다.
- class instance와 plain object를 데이터 모양 기준으로 비교해야 한다.
- 특정 테스트 helper에만 assertion 정책을 고정하고 싶다.
- 실패 출력에서 민감정보를 제거하는 wrapper를 함께 제공하고 싶다.

테스트 정책이 늘어나면 문서화도 필요합니다.
프로젝트의 `test/helpers/assertions.js` 같은 파일에 helper를 모으고, 왜 기본 assert 대신 별도 인스턴스를 쓰는지 짧게 남기면 됩니다.

### prototype 무시는 계약을 흐릴 수 있다

`skipPrototype: true`는 편리하지만, 잘못 쓰면 중요한 차이를 감춥니다.
예를 들어 `Date` 객체가 문자열로 바뀌었거나, 도메인 class가 plain object로 변했거나, validation 결과 객체가 다른 constructor를 갖게 되었는데도 테스트가 지나갈 수 있습니다.

```js
const assertPayloadShape = new Assert({
  strict: true,
  skipPrototype: true
});

// 공개 JSON shape만 검증하는 테스트라면 괜찮다.
assertPayloadShape.deepStrictEqual(publicPayload, expectedPublicPayload);

// 도메인 객체 타입 자체가 중요하다면 기본 assert를 쓴다.
assert.deepStrictEqual(domainUser, expectedDomainUser);
```

비교 대상이 내부 모델인지, 외부 전송 데이터인지 먼저 나누세요.
내부 모델은 prototype까지 비교하는 기본값이 더 안전하고, 외부 전송 데이터는 JSON으로 직렬화된 결과를 비교하는 방식도 좋은 선택입니다.

### CI 로그 길이와 보안 기준을 같이 본다

`diff: 'full'`은 실패 조사에는 좋지만 로그 양을 늘립니다.
CI가 실패할 때마다 큰 객체 diff가 여러 개 출력되면 필요한 정보가 묻힐 수 있습니다.
또 긴 diff에는 민감정보가 섞일 가능성이 커집니다.

실무에서는 아래 기준을 권장합니다.

- 기본 assertion은 `node:assert/strict`를 유지한다.
- 큰 생성 결과 비교에만 `diff: 'full'` helper를 쓴다.
- full diff 대상 fixture에는 토큰, 이메일, 내부 URL을 넣지 않는다.
- `skipPrototype` helper는 payload, plain data, serialized shape처럼 이름을 좁힌다.
- `Assert` 인스턴스의 메서드는 구조분해하지 않는다.

## FAQ

### Assert 클래스는 node:test 전용인가요?

아닙니다.
`Assert` 클래스는 `node:assert` 모듈의 API입니다.
Node.js 내장 test runner와 함께 쓰기 좋지만, 일반 스크립트나 다른 테스트 실행 환경에서도 사용할 수 있습니다.
다만 테스트 실패 출력과 CI 로그에서 가장 큰 가치가 나타납니다.

### skipPrototype은 deepStrictEqual을 느슨하게 만드는 옵션인가요?

일부 기준에서는 그렇지만, 정확히는 prototype과 constructor 비교를 건너뛰는 옵션입니다.
열거 가능한 속성의 deep strict 비교까지 사라지는 것은 아닙니다.
그래도 타입 차이를 숨길 수 있으므로 외부 payload shape 비교처럼 목적이 분명한 곳에만 쓰는 편이 좋습니다.

### full diff를 전역 기본값으로 두면 안 되나요?

가능은 하지만 권장하지 않습니다.
작은 실패에도 로그가 길어지고, CI 출력에 민감정보가 섞일 가능성이 커집니다.
큰 구조 비교가 필요한 테스트에만 별도 `Assert` 인스턴스를 쓰는 방식이 더 관리하기 쉽습니다.

## 정리

Node.js `Assert` 클래스는 assertion 정책을 테스트 단위로 분리할 수 있게 해 줍니다.
`diff: 'full'`은 큰 객체 실패를 더 빨리 읽게 만들고, `skipPrototype: true`는 class instance와 plain object를 데이터 shape 기준으로 비교해야 하는 좁은 상황에 쓸 수 있습니다.

다만 기본값을 바꾸는 도구인 만큼 범위를 작게 잡아야 합니다.
대부분의 테스트는 `node:assert/strict`를 유지하고, 진단 출력이나 payload shape 비교처럼 이유가 분명한 곳에만 별도 `Assert` 인스턴스를 두세요.
그리고 인스턴스 메서드는 구조분해하지 말고, helper 이름에 비교 정책을 드러내는 것이 안전합니다.

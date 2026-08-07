---
layout: post
title: "Node.js crypto.randomInt 가이드: 편향 없는 정수 난수를 안전하게 만드는 법"
date: 2026-08-08 08:00:00 +0900
lang: ko
translation_key: nodejs-crypto-randomint-unbiased-random-number-guide
permalink: /development/blog/seo/2026/08/08/nodejs-crypto-randomint-unbiased-random-number-guide.html
alternates:
  ko: /development/blog/seo/2026/08/08/nodejs-crypto-randomint-unbiased-random-number-guide.html
  x_default: /development/blog/seo/2026/08/08/nodejs-crypto-randomint-unbiased-random-number-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, crypto, randomint, random-number, security, javascript, backend, token]
description: "Node.js crypto.randomInt로 편향 없는 정수 난수를 만드는 방법을 정리합니다. Math.random과의 차이, min/max 범위, 동기·비동기 사용법, 토큰 코드와 테스트 전략까지 실무 예제로 설명합니다."
---

Node.js에서 임의의 숫자가 필요할 때 가장 먼저 떠오르는 코드는 `Math.random()`입니다.
UI 샘플링, 애니메이션, 비보안 테스트 데이터에는 충분히 쓸 수 있습니다.
하지만 인증 코드, 초대 코드, 추첨 로직, 분산 작업 샤딩처럼 결과의 공정성이나 예측 불가능성이 중요한 곳에서는 기준이 달라집니다.

`crypto.randomInt()`는 Node.js 기본 `node:crypto` 모듈이 제공하는 정수 난수 API입니다.
공식 문서 기준으로 지정한 범위 안에서 `min <= n < max` 조건을 만족하는 정수를 반환하며, 단순 나머지 연산에서 생길 수 있는 modulo bias를 피하도록 구현되어 있습니다.
이 글에서는 `crypto.randomInt()`의 기본 사용법, `Math.random()`과의 차이, 범위 설계, 토큰 코드 생성, 테스트에서 난수를 다루는 방법을 실무 기준으로 정리합니다.

안전한 고유 ID 생성이 목적이라면 [Node.js crypto.randomUUID 가이드: 안전한 ID 생성과 추적 키 설계](/development/blog/seo/2026/05/08/nodejs-crypto-randomuuid-safe-id-generation-guide.html)를 함께 볼 수 있습니다.
웹훅이나 서명 검증처럼 난수보다 검증 로직이 중요하다면 [Node.js Web Crypto HMAC 가이드](/development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html)가 이어서 도움이 됩니다.

## crypto.randomInt가 필요한 이유

### H3. Math.random은 보안용 난수가 아니다

`Math.random()`은 편하고 빠릅니다.
하지만 보안 토큰, 인증 코드, 권한 검증, 결제성 추첨처럼 결과를 공격자가 맞히면 안 되는 곳에 쓰면 안 됩니다.
런타임 구현에 따라 결과 품질과 예측 가능성 기준이 다르고, 암호학적으로 안전한 난수를 제공하는 API가 아니기 때문입니다.

아래 코드는 보기에는 간단하지만 보안성 있는 6자리 코드를 만들기에는 적합하지 않습니다.

```js
const code = String(Math.floor(Math.random() * 1_000_000)).padStart(6, '0');
```

비슷한 모양의 코드를 `crypto.randomInt()`로 바꾸면 의도가 더 분명해집니다.

```js
import { randomInt } from 'node:crypto';

const code = String(randomInt(0, 1_000_000)).padStart(6, '0');

console.log(code);
```

이 코드는 `000000`부터 `999999`까지의 정수 중 하나를 만듭니다.
상한값 `1_000_000`은 포함되지 않는다는 점이 중요합니다.
즉 `randomInt(0, 10)`은 0 이상 10 미만의 정수, 다시 말해 0부터 9까지를 반환합니다.

### H3. 나머지 연산은 분포를 치우치게 만들 수 있다

난수를 직접 바이트로 만든 뒤 `%` 연산으로 범위를 줄이는 코드를 종종 볼 수 있습니다.

```js
import { randomBytes } from 'node:crypto';

function badRandomDigit() {
  return randomBytes(1)[0] % 10;
}
```

이 코드는 짧지만 균등하지 않습니다.
1바이트 값은 0부터 255까지 256가지이고, 10으로 나누어 떨어지지 않습니다.
그 결과 일부 숫자가 다른 숫자보다 조금 더 자주 나올 수 있습니다.
작은 차이처럼 보여도 추첨, 샤딩, rate limit bucket 선택처럼 반복 횟수가 많은 로직에서는 신뢰를 떨어뜨립니다.

`crypto.randomInt()`는 이런 modulo bias를 직접 피하도록 만들어진 API입니다.
정수 범위가 필요하다면 바이트를 직접 조합하기보다 이 API를 먼저 선택하는 편이 좋습니다.

## 기본 사용법

### H3. max만 넘기면 0부터 max 직전까지 나온다

가장 단순한 형태는 `randomInt(max)`입니다.

```js
import { randomInt } from 'node:crypto';

const shard = randomInt(16);

console.log(shard); // 0부터 15까지
```

샤드 번호, A/B 테스트 bucket, 작은 샘플링 그룹처럼 0부터 시작하는 범위에는 이 형태가 깔끔합니다.
다만 보안 판단이 필요한 샤딩이라면 난수만으로 책임을 끝내지 말고, 요청자 식별자와 권한 검증을 별도로 유지해야 합니다.

작업 분배 코드에서는 범위가 코드 곳곳에 흩어지지 않도록 이름 있는 상수로 빼두는 편이 좋습니다.

```js
import { randomInt } from 'node:crypto';

const WORKER_BUCKET_COUNT = 32;

export function pickWorkerBucket() {
  return randomInt(WORKER_BUCKET_COUNT);
}
```

이렇게 하면 bucket 수를 바꿀 때 호출부를 일일이 찾아다닐 필요가 없습니다.
관측성 지표에도 같은 상수 이름을 쓰면 운영 중 분포를 확인하기가 쉬워집니다.

### H3. min과 max를 넘기면 시작값을 바꿀 수 있다

주사위처럼 1부터 시작하는 범위가 필요하다면 `randomInt(min, max)`를 씁니다.
이때 `min`은 포함되고 `max`는 포함되지 않습니다.

```js
import { randomInt } from 'node:crypto';

export function rollDice() {
  return randomInt(1, 7);
}
```

상한이 exclusive라는 규칙은 배열 인덱스와 잘 맞습니다.

```js
import { randomInt } from 'node:crypto';

export function pickOne(items) {
  if (items.length === 0) {
    throw new Error('Cannot pick from an empty array');
  }

  return items[randomInt(items.length)];
}
```

빈 배열을 먼저 막는 이유는 `randomInt(0)`이 유효한 범위가 아니기 때문입니다.
난수 함수 안에서 애매한 예외가 나는 것보다, 도메인에 맞는 오류 메시지를 직접 던지는 편이 디버깅에 좋습니다.

## 인증 코드와 초대 코드에 적용하기

### H3. 숫자 코드는 짧을수록 추측에 약하다

6자리 숫자 코드는 사용성이 좋지만 경우의 수는 100만 개입니다.
따라서 로그인 OTP, 이메일 인증 코드, 초대 코드처럼 외부 입력으로 검증되는 값에는 아래 방어선을 함께 둬야 합니다.

- 만료 시간을 짧게 둔다
- 시도 횟수를 제한한다
- 사용자나 목적별로 코드를 분리한다
- 성공과 실패 로그에 원문 코드를 남기지 않는다
- 저장할 때는 원문 대신 해시나 별도 토큰 저장 전략을 검토한다

코드 생성 자체는 간단합니다.

```js
import { randomInt } from 'node:crypto';

export function createNumericCode(length = 6) {
  if (!Number.isInteger(length) || length < 4 || length > 12) {
    throw new RangeError('length must be an integer between 4 and 12');
  }

  const max = 10 ** length;
  return String(randomInt(0, max)).padStart(length, '0');
}
```

`length`에 상한을 둔 이유는 실수로 지나치게 큰 범위를 만들지 않기 위해서입니다.
Node.js의 `randomInt()`는 안전한 정수 범위와 내부 난수 생성 한계를 전제로 하므로, 사람이 입력한 숫자를 그대로 넘기지 않는 편이 좋습니다.

해시 기반 저장 전략이 필요하다면 [Node.js crypto.hash 가이드: 짧은 데이터 해시를 한 줄로 안전하게 만드는 법](/development/blog/seo/2026/05/12/nodejs-crypto-hash-one-shot-digest-guide.html)을 참고하세요.
다만 짧은 숫자 코드는 해시해도 경우의 수가 작기 때문에, 만료 시간과 시도 제한이 같이 있어야 합니다.

### H3. 문자 코드에는 알파벳 선택 규칙을 먼저 정한다

초대 코드나 쿠폰 코드에는 숫자만으로 부족할 때가 많습니다.
이때는 사람이 읽기 어려운 문자를 제외한 alphabet을 정하고, 각 위치마다 `randomInt()`로 문자를 고를 수 있습니다.

```js
import { randomInt } from 'node:crypto';

const ALPHABET = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';

export function createInviteCode(length = 10) {
  if (!Number.isInteger(length) || length < 6 || length > 32) {
    throw new RangeError('length must be an integer between 6 and 32');
  }

  let code = '';

  for (let index = 0; index < length; index += 1) {
    code += ALPHABET[randomInt(ALPHABET.length)];
  }

  return code;
}
```

여기서는 `I`, `O`, `0`, `1`처럼 혼동하기 쉬운 문자를 제외했습니다.
사용자가 직접 입력하는 코드라면 보안성만큼 입력 경험도 중요합니다.
대문자로 정규화할지, 하이픈을 넣을지, 복사 붙여넣기에서 공백을 제거할지도 함께 정해야 합니다.

```js
export function normalizeInviteCode(input) {
  return input.replaceAll('-', '').replaceAll(' ', '').toUpperCase();
}
```

단, 정규화 로직은 검증 전에만 적용해야 합니다.
로그나 오류 응답에 원문 코드를 그대로 남기면 민감정보 노출이 될 수 있습니다.
운영 로그에서 값 노출을 줄이는 기준은 [CLI 출력값 민감정보 제거 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)와 같은 원칙을 따르면 됩니다.

## 동기와 비동기 사용 기준

### H3. 짧은 단발 생성은 동기 API가 읽기 쉽다

`randomInt()`는 callback을 넘기지 않으면 동기적으로 값을 반환합니다.
대부분의 짧은 코드 생성, 샤드 선택, 배열 샘플링에는 동기 형태가 충분히 읽기 좋습니다.

```js
import { randomInt } from 'node:crypto';

export function shouldSample(percent) {
  if (percent < 0 || percent > 100) {
    throw new RangeError('percent must be between 0 and 100');
  }

  return randomInt(100) < percent;
}
```

샘플링에는 이 패턴이 간단합니다.
다만 보안 감사 로그나 결제성 추첨처럼 결과 설명 가능성이 중요한 로직에서는 무작위 선택만 기록하지 말고, 선택 시각과 정책 버전, 대상 집합 기준도 함께 남겨야 합니다.

### H3. 대량 생성은 async 흐름으로 분리할 수 있다

콜백을 넘기면 비동기 형태로 사용할 수 있습니다.
Promise 기반 코드베이스에서는 `node:util`의 `promisify`로 감싸는 방식이 자연스럽습니다.

```js
import { randomInt } from 'node:crypto';
import { promisify } from 'node:util';

const randomIntAsync = promisify(randomInt);

export async function createCodes(count) {
  const codes = [];

  for (let index = 0; index < count; index += 1) {
    const value = await randomIntAsync(0, 1_000_000);
    codes.push(String(value).padStart(6, '0'));
  }

  return codes;
}
```

대량 생성에서는 난수 API보다 중복 처리 정책이 더 중요해질 수 있습니다.
코드 공간이 작으면 충돌이 생길 수 있으므로, DB unique constraint나 재시도 로직을 같이 설계해야 합니다.
발급 수가 많고 충돌 비용이 크다면 숫자 6자리보다 더 긴 alphabet 코드나 UUID 계열 ID가 더 적합합니다.

## 테스트하기 쉬운 난수 코드 만들기

### H3. 난수 함수를 주입하면 테스트가 안정된다

테스트에서 진짜 난수를 직접 호출하면 결과가 매번 달라집니다.
로직 자체를 검증하려면 난수 생성 함수를 주입할 수 있게 만드는 편이 좋습니다.

```js
import { randomInt } from 'node:crypto';

export function createPicker({ randomInteger = randomInt } = {}) {
  return function pickOne(items) {
    if (items.length === 0) {
      throw new Error('Cannot pick from an empty array');
    }

    return items[randomInteger(items.length)];
  };
}
```

테스트에서는 고정된 값을 돌려주는 함수를 넣으면 됩니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { createPicker } from './picker.js';

test('pickOne returns the selected item', () => {
  const pickOne = createPicker({
    randomInteger: () => 1
  });

  assert.equal(pickOne(['red', 'green', 'blue']), 'green');
});
```

이 방식은 난수 품질을 테스트하려는 것이 아닙니다.
난수를 사용하는 비즈니스 로직이 범위와 예외를 제대로 처리하는지 확인하기 위한 구조입니다.
Node.js 내장 테스트 러너 패턴은 [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)을 참고할 수 있습니다.

### H3. 분포 테스트는 보조 신호로만 둔다

난수 분포를 테스트하고 싶어서 "100번 뽑았을 때 모든 숫자가 한 번 이상 나와야 한다" 같은 테스트를 만들면 CI가 불안정해집니다.
무작위 결과는 정상이어도 특정 숫자가 안 나올 수 있습니다.

대신 애플리케이션 테스트에서는 아래 정도만 확인하는 편이 좋습니다.

- 반환값이 `min <= n < max` 범위를 지킨다
- 빈 배열, 잘못된 길이, 음수 percent 같은 입력을 거부한다
- 코드 길이와 alphabet 제약을 만족한다
- 중복 처리 로직이 DB unique constraint 실패를 재시도한다

분포 품질은 Node.js crypto 구현을 신뢰하고, 서비스 차원에서는 운영 지표로 이상 징후를 보는 방식이 현실적입니다.
예를 들어 초대 코드 발급 결과를 원문 없이 길이, prefix 정책 버전, 충돌 횟수 정도로 집계하면 문제가 생겼을 때 원인을 좁히기 쉽습니다.

## 실무 체크리스트

### H3. randomInt 적용 전 확인할 것

`crypto.randomInt()`를 쓰기 전에 아래 기준을 확인하세요.

- 범위의 상한이 exclusive임을 코드와 테스트에 반영했는가?
- `Math.random()`을 보안성 있는 코드 생성에 사용하지 않았는가?
- `%` 연산으로 난수 범위를 줄이는 코드를 남겨두지 않았는가?
- 사용자 입력으로 받은 `min`, `max`, `length`를 검증했는가?
- 짧은 인증 코드에는 만료 시간과 시도 제한을 함께 적용했는가?
- 로그에 인증 코드, 초대 코드, 토큰 원문을 남기지 않았는가?
- 테스트에서는 난수 함수를 주입해 결과를 안정화했는가?

### H3. randomInt를 쓰지 않는 편이 나은 경우

정수 하나가 아니라 긴 토큰 바이트가 필요하다면 `randomBytes()`나 Web Crypto API가 더 잘 맞을 수 있습니다.
정렬 가능한 고유 ID가 필요하면 UUID v7 같은 별도 식별자 전략을 검토할 수 있고, 사람이 보지 않는 내부 추적 ID라면 `randomUUID()`가 더 단순합니다.

반대로 배열에서 하나 고르기, bucket 선택, 짧은 숫자 코드 생성, 공정한 정수 범위 선택에는 `crypto.randomInt()`가 좋은 기본값입니다.
핵심은 "난수가 필요하다"가 아니라 "어떤 범위와 어떤 보안 수준의 난수가 필요한가"를 먼저 정하는 것입니다.

## 마무리

`crypto.randomInt()`는 작지만 중요한 API입니다.
`Math.random()`보다 의도가 분명하고, 직접 `%` 연산으로 범위를 줄일 때 생길 수 있는 편향도 피할 수 있습니다.
특히 인증 코드, 초대 코드, 샤드 선택, 추첨성 로직처럼 정수 범위의 공정성과 예측 불가능성이 필요한 곳에서는 기본 선택지로 삼을 만합니다.

다만 난수 API 하나가 전체 보안을 완성하지는 않습니다.
코드 길이, 만료 시간, 시도 제한, 충돌 처리, 로그 마스킹, 테스트 주입 구조까지 함께 설계해야 실무에서 안전하게 운영할 수 있습니다.

참고: [Node.js 공식 crypto.randomInt 문서](https://nodejs.org/api/crypto.html#cryptorandomintmin-max-callback)

---
layout: post
title: "Node.js crypto.argon2 가이드: 비밀번호 해싱을 내장 API로 설계하는 법"
date: 2026-08-22 20:00:00 +0900
lang: ko
translation_key: nodejs-crypto-argon2-password-hashing-guide
permalink: /development/blog/seo/2026/08/22/nodejs-crypto-argon2-password-hashing-guide.html
alternates:
  ko: /development/blog/seo/2026/08/22/nodejs-crypto-argon2-password-hashing-guide.html
  x_default: /development/blog/seo/2026/08/22/nodejs-crypto-argon2-password-hashing-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, crypto, argon2, password-hashing, authentication, security, backend, javascript]
description: "Node.js crypto.argon2로 비밀번호 해싱을 설계하는 방법을 정리합니다. argon2id 선택, salt와 pepper 관리, 저장 포맷, 검증 코드, 운영 튜닝 기준까지 실무 예제로 설명합니다."
---

비밀번호를 저장할 때 가장 피해야 할 실수는 빠른 해시 함수만 쓰는 것입니다.
SHA-256, SHA-512 같은 일반 해시는 무결성 확인에는 좋지만 너무 빠릅니다.
공격자가 해시 목록을 얻으면 대량 추측을 빠르게 반복할 수 있습니다.

비밀번호 해싱에는 계산 비용과 메모리 비용을 의도적으로 높이는 전용 알고리즘이 필요합니다.
Node.js는 `crypto.argon2()`와 `crypto.argon2Sync()`를 제공하며, 공식 문서 기준으로 Argon2d, Argon2i, Argon2id 변형을 지원합니다.
실무 인증 서버에서는 대체로 `argon2id`를 우선 검토하는 편이 무난합니다.

이 글에서는 Node.js 내장 `crypto.argon2()`로 비밀번호 해시를 만들고 검증하는 구조를 정리합니다.
단순 해시와의 차이는 [Node.js crypto.hash 가이드](/development/blog/seo/2026/05/12/nodejs-crypto-hash-one-shot-digest-guide.html), 랜덤 salt 생성은 [Node.js randomBytes와 base64url 토큰 가이드](/development/blog/seo/2026/05/16/nodejs-buffer-base64url-encoding-guide.html), 웹훅 서명 검증은 [Node.js Web Crypto HMAC 가이드](/development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html)와 함께 보면 좋습니다.

## crypto.argon2를 쓰는 이유

### 일반 해시는 비밀번호 저장용이 아니다

일반 해시 함수는 입력이 같으면 같은 결과를 빠르게 계산하도록 만들어졌습니다.
파일 무결성 확인, 캐시 키, 짧은 식별자 생성에는 이 성질이 유용합니다.
하지만 비밀번호 저장에서는 바로 이 속도가 문제가 됩니다.

```js
import { createHash } from 'node:crypto';

function unsafePasswordHash(password) {
  return createHash('sha256').update(password).digest('hex');
}
```

위 코드는 짧고 이해하기 쉽지만 비밀번호 저장에는 맞지 않습니다.
salt가 없기 때문에 같은 비밀번호가 같은 해시로 드러나고, 계산이 너무 빨라 추측 공격 비용을 충분히 높이지 못합니다.

비밀번호 해싱의 목표는 "원문 비밀번호를 다시 알 수 없게 한다"에서 끝나지 않습니다.
공격자가 해시와 salt를 얻더라도 후보 비밀번호 하나를 검증하는 비용이 충분히 크도록 만들어야 합니다.
Argon2는 이 목적을 위해 계산량과 메모리 사용량을 함께 조절합니다.

### Argon2는 비용을 설정하는 해싱이다

Node.js의 `crypto.argon2()`는 `message`, `nonce`, `parallelism`, `tagLength`, `memory`, `passes` 같은 파라미터를 받습니다.
여기서 `message`는 비밀번호, `nonce`는 비밀번호 해싱에서 흔히 말하는 salt 역할입니다.
공식 문서는 `nonce`가 최소 8바이트 이상이어야 하고, 랜덤하면서 16바이트 이상이면 좋다고 설명합니다.

```js
import { argon2, randomBytes } from 'node:crypto';
import { promisify } from 'node:util';

const argon2Async = promisify(argon2);

const parameters = {
  message: 'correct horse battery staple',
  nonce: randomBytes(16),
  parallelism: 4,
  tagLength: 32,
  memory: 65536,
  passes: 3
};

const derivedKey = await argon2Async('argon2id', parameters);
console.log(derivedKey.toString('base64url'));
```

`memory`는 1KiB 블록 단위입니다.
예를 들어 `65536`은 약 64MiB 수준의 메모리 비용을 의미합니다.
이 값은 보안과 서버 처리량 사이의 균형을 직접 바꾸므로, 문서 예시를 그대로 운영 기본값으로 고정하기보다 실제 로그인 경로에서 측정해야 합니다.

## 저장 포맷 설계하기

### salt와 파라미터를 함께 저장한다

비밀번호 검증은 가입 시점과 로그인 시점에 같은 파라미터로 Argon2를 다시 계산한 뒤 결과를 비교하는 방식입니다.
따라서 salt와 비용 파라미터는 해시와 함께 저장해야 합니다.
salt는 비밀이 아니므로 데이터베이스에 저장해도 됩니다.

```js
import { argon2, randomBytes, timingSafeEqual } from 'node:crypto';
import { promisify } from 'node:util';

const argon2Async = promisify(argon2);

const PASSWORD_HASH_VERSION = 1;

const DEFAULT_ARGON2 = {
  algorithm: 'argon2id',
  parallelism: 4,
  tagLength: 32,
  memory: 65536,
  passes: 3
};

export async function hashPassword(password) {
  const salt = randomBytes(16);
  const hash = await argon2Async(DEFAULT_ARGON2.algorithm, {
    message: password,
    nonce: salt,
    parallelism: DEFAULT_ARGON2.parallelism,
    tagLength: DEFAULT_ARGON2.tagLength,
    memory: DEFAULT_ARGON2.memory,
    passes: DEFAULT_ARGON2.passes
  });

  return [
    `v=${PASSWORD_HASH_VERSION}`,
    `alg=${DEFAULT_ARGON2.algorithm}`,
    `p=${DEFAULT_ARGON2.parallelism}`,
    `m=${DEFAULT_ARGON2.memory}`,
    `t=${DEFAULT_ARGON2.passes}`,
    `l=${DEFAULT_ARGON2.tagLength}`,
    `salt=${salt.toString('base64url')}`,
    `hash=${hash.toString('base64url')}`
  ].join('$');
}
```

이 예시는 PHC 문자열 포맷을 완전히 구현한 것은 아닙니다.
핵심은 저장 문자열 안에 버전, 알고리즘, 비용 파라미터, salt, 해시를 함께 넣는 것입니다.
나중에 비용을 올리거나 알고리즘을 바꾸더라도 기존 사용자의 해시를 계속 검증할 수 있습니다.

저장 포맷은 팀에서 이미 쓰는 인증 라이브러리나 기존 사용자 테이블 규칙을 우선해야 합니다.
새로 설계한다면 사람이 읽을 수 있고, 파싱하기 쉽고, 버전 변경을 담을 수 있는 구조가 좋습니다.

### pepper는 저장 위치가 다르다

Node.js `crypto.argon2()`에는 `secret` 파라미터가 있습니다.
비밀번호 해싱 맥락에서는 pepper로 볼 수 있습니다.
salt와 달리 pepper는 데이터베이스에 함께 저장하지 않습니다.
애플리케이션의 secret store나 배포 환경 변수처럼 별도 경로에서 관리해야 합니다.

```js
function readPasswordPepper() {
  const value = process.env.PASSWORD_PEPPER;

  if (!value) {
    throw new Error('PASSWORD_PEPPER is required');
  }

  return Buffer.from(value, 'base64url');
}
```

pepper를 쓰면 데이터베이스만 유출된 상황에서 공격 비용을 더 높일 수 있습니다.
하지만 운영 복잡도도 생깁니다.
pepper가 사라지면 기존 비밀번호 검증이 불가능해질 수 있고, 교체 전략도 별도로 필요합니다.

처음부터 pepper를 넣을지 여부는 서비스 위험도와 secret 운영 체계에 따라 결정하세요.
간단한 내부 도구라면 충분한 salt와 비용 파라미터부터 안정적으로 운영하는 것이 먼저일 수 있습니다.

## 비밀번호 검증 코드

### 저장 문자열을 파싱한다

검증 함수는 저장된 파라미터를 읽고 같은 조건으로 Argon2를 다시 계산해야 합니다.
입력 문자열이 깨졌을 때는 인증 실패로 처리하되, 로그에 원문 비밀번호나 전체 해시를 그대로 남기지 않는 편이 좋습니다.

```js
function parsePasswordHash(encoded) {
  const fields = Object.fromEntries(
    encoded.split('$').map((part) => {
      const index = part.indexOf('=');

      if (index === -1) {
        throw new Error('Invalid password hash field');
      }

      return [part.slice(0, index), part.slice(index + 1)];
    })
  );

  const required = ['v', 'alg', 'p', 'm', 't', 'l', 'salt', 'hash'];

  for (const key of required) {
    if (!fields[key]) {
      throw new Error('Invalid password hash payload');
    }
  }

  return {
    version: Number.parseInt(fields.v, 10),
    algorithm: fields.alg,
    parallelism: Number.parseInt(fields.p, 10),
    memory: Number.parseInt(fields.m, 10),
    passes: Number.parseInt(fields.t, 10),
    tagLength: Number.parseInt(fields.l, 10),
    salt: Buffer.from(fields.salt, 'base64url'),
    hash: Buffer.from(fields.hash, 'base64url')
  };
}
```

실제 서비스에서는 허용 알고리즘과 숫자 범위를 더 엄격하게 검사하세요.
데이터베이스 값도 외부 입력처럼 다루는 편이 안전합니다.
예를 들어 `memory`가 비정상적으로 큰 값이면 로그인 요청 하나가 서버 자원을 과도하게 잡아먹을 수 있습니다.

### timingSafeEqual로 결과를 비교한다

계산한 해시와 저장된 해시는 길이가 같을 때 `timingSafeEqual()`로 비교할 수 있습니다.
길이가 다르면 바로 실패시키되, 예외가 외부로 새지 않게 감싸는 것이 좋습니다.

```js
export async function verifyPassword(password, encodedHash) {
  let parsed;

  try {
    parsed = parsePasswordHash(encodedHash);
  } catch {
    return false;
  }

  if (parsed.algorithm !== 'argon2id' || parsed.version !== 1) {
    return false;
  }

  const candidate = await argon2Async(parsed.algorithm, {
    message: password,
    nonce: parsed.salt,
    parallelism: parsed.parallelism,
    tagLength: parsed.tagLength,
    memory: parsed.memory,
    passes: parsed.passes
  });

  if (candidate.length !== parsed.hash.length) {
    return false;
  }

  return timingSafeEqual(candidate, parsed.hash);
}
```

로그인 API에서는 이 함수의 결과만 보고 성공과 실패를 나누면 됩니다.
사용자에게는 "이메일 또는 비밀번호가 올바르지 않습니다"처럼 같은 메시지를 보여 주고, 내부 로그에는 사용자 ID, 요청 ID, 실패 유형 정도만 남기세요.
비밀번호 원문, 전체 해시, pepper 값은 로그에 절대 남기지 않아야 합니다.

## 운영 튜닝 기준

### 비용 파라미터는 실제 경로에서 측정한다

Argon2 비용을 높이면 공격자는 힘들어지지만 서버도 같은 비용을 냅니다.
로그인, 회원가입, 비밀번호 변경, 관리자 강제 초기화처럼 비밀번호 해싱이 발생하는 경로의 동시성을 기준으로 측정해야 합니다.

```js
import { performance } from 'node:perf_hooks';

async function measureArgon2(parameters, rounds = 10) {
  const durations = [];

  for (let index = 0; index < rounds; index += 1) {
    const started = performance.now();
    await argon2Async('argon2id', {
      ...parameters,
      message: `password-${index}`,
      nonce: randomBytes(16)
    });
    durations.push(performance.now() - started);
  }

  durations.sort((a, b) => a - b);

  return {
    min: durations[0],
    median: durations[Math.floor(durations.length / 2)],
    max: durations[durations.length - 1]
  };
}
```

벤치마크는 개발자 노트북보다 실제 배포 환경에 가까운 곳에서 돌려야 의미가 있습니다.
서버리스, 컨테이너, 작은 VPS, CI runner는 CPU와 메모리 특성이 다릅니다.
특히 `memory`를 높이면 동시 로그인 요청 수에 따라 메모리 압박이 빠르게 커질 수 있습니다.

### 동기 API는 요청 경로에서 피한다

`crypto.argon2Sync()`도 있지만 일반적인 HTTP 요청 처리 경로에서는 비동기 `crypto.argon2()`를 우선하세요.
동기 API는 호출 중 이벤트 루프를 막기 때문에 동시에 들어온 요청의 지연을 키울 수 있습니다.

동기 API가 어울리는 곳도 있습니다.
예를 들어 CLI 초기화 스크립트, 일회성 마이그레이션, 테스트 fixture 생성처럼 사용자 요청을 직접 처리하지 않는 코드에서는 단순함이 장점일 수 있습니다.
하지만 로그인 API, 회원가입 API, OAuth 콜백 같은 온라인 경로에서는 비동기 함수로 분리하는 편이 낫습니다.

비밀번호 해싱은 의도적으로 무거운 작업입니다.
rate limiting, 로그인 실패 잠금, IP·계정 단위 지연 정책과 함께 설계해야 전체 인증 경로가 버팁니다.

## 마이그레이션 전략

### 기존 해시는 로그인 성공 시점에 교체한다

이미 bcrypt, scrypt, PBKDF2, 혹은 오래된 SHA 계열 해시를 쓰는 서비스라면 한 번에 모든 사용자를 바꾸기 어렵습니다.
일반적으로는 저장된 해시의 버전이나 prefix를 보고 검증 함수를 나눈 뒤, 사용자가 정상 로그인했을 때 새 Argon2 해시로 교체합니다.

```js
async function verifyAndUpgradePassword(user, password) {
  const verified = await verifyLegacyOrCurrentPassword(user.passwordHash, password);

  if (!verified) {
    return false;
  }

  if (needsPasswordHashUpgrade(user.passwordHash)) {
    const upgradedHash = await hashPassword(password);
    await updateUserPasswordHash(user.id, upgradedHash);
  }

  return true;
}
```

이 방식은 사용자에게 비밀번호 변경을 강제하지 않아도 점진적으로 저장 포맷을 바꿀 수 있습니다.
다만 비활성 계정은 오래된 해시가 계속 남을 수 있으므로, 일정 기간 뒤 재인증이나 비밀번호 재설정을 요구하는 정책도 검토할 수 있습니다.

마이그레이션 중에는 어떤 알고리즘을 검증했는지, 업그레이드가 성공했는지, 실패율이 변했는지 지표를 봐야 합니다.
하지만 지표에도 비밀번호 원문이나 해시 값을 넣으면 안 됩니다.

### 비용 상향도 버전 변경으로 다룬다

처음 정한 `memory`와 `passes`가 영원히 충분하다고 볼 수는 없습니다.
서버 사양과 공격 비용은 시간이 지나며 바뀝니다.
비용을 올릴 때도 해시 버전이나 파라미터 비교로 업그레이드 여부를 판단하면 됩니다.

```js
function needsPasswordHashUpgrade(encodedHash) {
  try {
    const parsed = parsePasswordHash(encodedHash);

    return (
      parsed.version < PASSWORD_HASH_VERSION ||
      parsed.algorithm !== DEFAULT_ARGON2.algorithm ||
      parsed.memory < DEFAULT_ARGON2.memory ||
      parsed.passes < DEFAULT_ARGON2.passes ||
      parsed.tagLength !== DEFAULT_ARGON2.tagLength
    );
  } catch {
    return true;
  }
}
```

비용 상향은 배포 전 부하 테스트가 필요합니다.
평균 로그인 시간만 보지 말고 p95, p99 지연과 메모리 사용량을 함께 보세요.
인증 장애는 서비스 전체 진입을 막기 때문에 작은 튜닝 실수도 크게 보일 수 있습니다.

## 체크리스트

### 구현 전 확인할 것

- Node.js 런타임 버전에서 `crypto.argon2()`를 지원하는가?
- 새 가입과 로그인 검증이 같은 저장 포맷을 공유하는가?
- salt는 사용자마다 랜덤으로 생성되는가?
- 비용 파라미터가 운영 환경에서 측정됐는가?
- 로그인 실패 응답이 계정 존재 여부를 과하게 드러내지 않는가?
- 로그와 지표에 비밀번호, 전체 해시, pepper가 남지 않는가?

### 운영 중 확인할 것

- 로그인 p95, p99 지연 시간이 허용 범위 안에 있는가?
- 인증 서버 메모리 사용량이 동시 요청 증가에도 안정적인가?
- 실패율 급증, 특정 IP 반복 실패, 계정 잠금 증가를 볼 수 있는가?
- 파라미터 변경 시 구버전 해시를 검증하고 업그레이드하는가?
- pepper를 쓰는 경우 백업, 교체, 사고 대응 절차가 있는가?

## FAQ

### crypto.argon2와 scrypt 중 무엇을 써야 하나요?

둘 다 비밀번호 해싱에 쓸 수 있는 비용 기반 KDF입니다.
새로 설계하는 Node.js 서비스에서 내장 Argon2를 사용할 수 있다면 `argon2id`를 우선 검토할 만합니다.
다만 이미 검증된 scrypt 운영 코드와 마이그레이션 정책이 있다면 무리해서 즉시 바꿀 필요는 없습니다.

### salt는 암호화해서 저장해야 하나요?

보통 salt는 비밀이 아닙니다.
사용자마다 다른 랜덤 salt를 해시와 함께 저장하면 됩니다.
비밀로 관리해야 하는 값은 salt가 아니라 pepper이며, pepper를 쓴다면 데이터베이스와 분리된 secret store에서 관리해야 합니다.

### 비밀번호 해시는 복호화할 수 있나요?

아니요.
비밀번호 해싱은 복호화가 아니라 재계산과 비교로 검증하는 구조입니다.
사용자가 비밀번호를 잊은 경우에는 기존 비밀번호를 알려 주는 것이 아니라 재설정 링크나 인증 절차를 통해 새 비밀번호를 받는 방식으로 처리해야 합니다.

### Argon2 파라미터는 어느 값이 정답인가요?

모든 서비스에 맞는 하나의 정답은 없습니다.
공식 문서 예시처럼 `memory`, `passes`, `parallelism`, `tagLength`를 정할 수 있지만, 최종 값은 실제 배포 환경의 지연 시간과 메모리 사용량을 측정해 결정해야 합니다.
보안 요구가 높은 서비스라면 주기적으로 비용 상향 여지도 점검하세요.

## 마무리

Node.js의 `crypto.argon2()`를 사용하면 별도 네이티브 패키지 없이도 Argon2 기반 비밀번호 해싱을 구현할 수 있습니다.
핵심은 `argon2id` 같은 적절한 알고리즘 선택보다 저장 포맷, salt 생성, 비용 튜닝, 검증 비교, 마이그레이션 정책을 함께 설계하는 것입니다.

비밀번호 해시는 인증 시스템의 가장 낮은 층에 있습니다.
여기서 빠른 일반 해시를 쓰지 않고, 사용자별 salt와 측정된 비용 파라미터, 안전한 비교, 로그 마스킹을 적용하면 계정 보안의 기본선을 크게 높일 수 있습니다.

## 함께 읽기

- [Node.js crypto.hash 가이드: 짧은 데이터 해시를 간단하게 계산하는 법](/development/blog/seo/2026/05/12/nodejs-crypto-hash-one-shot-digest-guide.html)
- [Node.js Buffer base64url 인코딩 가이드: URL에 안전한 토큰 만들기](/development/blog/seo/2026/05/16/nodejs-buffer-base64url-encoding-guide.html)
- [Node.js Web Crypto HMAC 가이드: 웹훅 서명 검증을 안전하게 구현하는 법](/development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html)
- [Node.js Crypto 공식 문서: crypto.argon2와 crypto.argon2Sync](https://nodejs.org/api/crypto.html#cryptoargon2algorithm-parameters-callback)

---
layout: post
title: "Node.js Buffer base64url 가이드: URL에 안전한 토큰 인코딩하는 법"
date: 2026-05-16 08:00:00 +0900
lang: ko
translation_key: nodejs-buffer-base64url-encoding-guide
permalink: /development/blog/seo/2026/05/16/nodejs-buffer-base64url-encoding-guide.html
alternates:
  ko: /development/blog/seo/2026/05/16/nodejs-buffer-base64url-encoding-guide.html
  x_default: /development/blog/seo/2026/05/16/nodejs-buffer-base64url-encoding-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, buffer, base64url, encoding, token, backend]
description: "Node.js Buffer의 base64url 인코딩으로 URL과 쿠키에 안전한 토큰을 다루는 방법을 정리했습니다. base64와 base64url 차이, 랜덤 토큰 생성, 디코딩 검증, 실무 보안 주의점까지 예제로 설명합니다."
---

API 토큰, 이메일 인증 링크, 비밀번호 재설정 링크, OAuth `state` 값처럼 문자열을 URL에 넣어야 하는 순간이 자주 있습니다.
이때 아무 문자열이나 그대로 붙이면 `+`, `/`, `=` 같은 문자가 쿼리스트링·경로·쿠키에서 다르게 해석되어 예기치 않은 오류를 만들 수 있습니다.
그래서 운영 코드에서는 URL에 넣기 쉬운 인코딩 형식을 명확히 정해 두는 것이 좋습니다.

Node.js의 `Buffer`는 `base64url` 인코딩을 지원합니다.
`base64url`은 일반 base64와 비슷하지만 URL에서 특별한 의미를 갖는 문자를 피하도록 설계된 표현입니다.
이 글에서는 `Buffer`로 base64url 문자열을 만들고 다시 읽는 방법, 랜덤 토큰 생성 예시, 입력 검증 기준, 실무에서 조심해야 할 보안 포인트를 정리합니다.
토큰 값 자체를 안전하게 생성하는 기준은 [Node.js crypto.randomUUID 가이드: 안전한 ID 생성과 실무 적용 기준](/development/blog/seo/2026/05/08/nodejs-crypto-randomuuid-safe-id-generation-guide.html)도 함께 보면 좋습니다.

## base64url이 필요한 이유

### H3. 일반 base64는 URL에서 불편한 문자를 포함한다

일반 base64는 바이너리 데이터를 문자열로 표현할 때 널리 쓰입니다.
하지만 결과에 `+`, `/`, `=` 문자가 들어갈 수 있습니다.
이 문자들은 URL 쿼리스트링, 경로 segment, 폼 인코딩, 일부 쿠키 처리 흐름에서 별도 의미를 가질 수 있습니다.

```js
const value = Buffer.from('hello?node=js');

console.log(value.toString('base64'));
// aGVsbG8/bm9kZT1qcw==
```

이 값은 짧은 예시라 비교적 안전해 보이지만, 실제 랜덤 바이트를 base64로 바꾸면 `+`와 `/`가 자연스럽게 섞입니다.
문자열을 URL에 넣을 때 매번 `encodeURIComponent()`를 정확히 적용하면 해결할 수 있지만, 여러 서비스와 언어가 오가는 시스템에서는 누락이 생기기 쉽습니다.

```js
import { randomBytes } from 'node:crypto';

const token = randomBytes(16).toString('base64');
console.log(token); // 예: v1zY+WjS/0K6dQxkU4YQ0g==
```

이 토큰을 그대로 `/verify/${token}` 같은 경로에 넣으면 `/`가 경로 구분자로 해석될 수 있습니다.
쿼리스트링에서도 `+`가 공백처럼 처리되는 환경을 만나면 원래 토큰과 다른 값이 서버에 도착할 수 있습니다.

### H3. base64url은 URL 친화적인 문자 집합을 사용한다

base64url은 일반 base64의 `+`를 `-`로, `/`를 `_`로 바꾼 변형입니다.
패딩 문자 `=`도 생략하는 경우가 많아 링크나 쿠키 값으로 다루기 편합니다.
Node.js에서는 `Buffer#toString('base64url')`을 사용하면 이 형식으로 바로 변환할 수 있습니다.

```js
import { randomBytes } from 'node:crypto';

const token = randomBytes(32).toString('base64url');

console.log(token);
// 예: 0t9O2DKWZgA3G5U3bAVZCE5m_K3thp6S1qf8y3kC6Hg
```

이 문자열은 URL 경로, 쿼리스트링, 쿠키 값에서 다루기 쉽습니다.
그렇다고 검증이 필요 없어지는 것은 아닙니다.
외부에서 받은 값은 길이, 문자 집합, 만료 시간, 사용 여부를 서버에서 다시 확인해야 합니다.
입력값 검증 흐름은 [Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)처럼 경계에서 먼저 정리하는 습관이 중요합니다.

## Buffer로 base64url 인코딩하고 디코딩하기

### H3. 문자열과 바이너리를 base64url로 바꾼다

`Buffer.from(value)`로 바이트 배열을 만든 뒤 `toString('base64url')`을 호출하면 됩니다.
텍스트를 인코딩할 때는 기본값이 UTF-8이므로 한글도 처리됩니다.

```js
const payload = JSON.stringify({
  type: 'email-verification',
  userId: 'user_123',
  issuedAt: '2026-05-16T08:00:00+09:00',
});

const encoded = Buffer.from(payload, 'utf8').toString('base64url');
console.log(encoded);

const decoded = Buffer.from(encoded, 'base64url').toString('utf8');
console.log(JSON.parse(decoded));
```

다만 이 예시는 “인코딩”일 뿐 “보안”이 아닙니다.
base64url 문자열은 누구나 다시 디코딩할 수 있습니다.
사용자에게 보이면 안 되는 개인정보, 권한, 내부 식별자, 가격 정책 같은 값은 그대로 넣지 말아야 합니다.
민감한 값이 필요한 경우에는 서버 저장소에 토큰 식별자만 저장하고, 토큰 자체는 추측하기 어려운 랜덤 값으로 만드는 편이 안전합니다.
로그 예제에서 민감정보를 다루는 기준은 [로그 예제 정제 가이드: 믿을 수 있는 개발 글을 위한 마스킹 원칙](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)에 정리되어 있습니다.

### H3. 디코딩 실패와 잘못된 입력을 분리한다

외부에서 받은 토큰은 항상 의심해야 합니다.
`Buffer.from(value, 'base64url')`은 일부 비정상 입력도 관대하게 처리할 수 있으므로, 토큰 형식이 정해져 있다면 먼저 정규식과 길이로 걸러내는 것이 좋습니다.

```js
const BASE64URL_RE = /^[A-Za-z0-9_-]+$/;

export function parseToken(value) {
  if (typeof value !== 'string') {
    return null;
  }

  if (value.length < 32 || value.length > 128) {
    return null;
  }

  if (!BASE64URL_RE.test(value)) {
    return null;
  }

  try {
    return Buffer.from(value, 'base64url');
  } catch {
    return null;
  }
}
```

토큰이 “랜덤 바이트를 base64url로 표현한 값”이라면 디코딩한 바이트 길이까지 확인하는 편이 더 좋습니다.
예를 들어 `randomBytes(32)`로 만든 토큰만 허용한다면 디코딩 결과가 정확히 32바이트인지 확인합니다.

```js
export function parseRandomToken(value) {
  const bytes = parseToken(value);

  if (!bytes || bytes.byteLength !== 32) {
    return null;
  }

  return bytes;
}
```

이렇게 하면 너무 짧은 값, 너무 긴 값, 다른 인코딩에서 온 값을 초기에 배제할 수 있습니다.
API 계약을 명확히 유지하는 검증 계층은 [Node.js OpenAPI + Zod 가이드: API 계약 검증으로 응답 일관성 지키기](/development/blog/seo/2026/03/21/nodejs-openapi-zod-contract-validation-api-consistency-guide.html)와도 같은 방향입니다.

## 실무 예제: 이메일 인증 토큰 만들기

### H3. 토큰은 랜덤 값으로 만들고 서버에 해시로 저장한다

이메일 인증 링크를 만들 때 토큰 안에 사용자 정보를 직접 담는 방식은 간단하지만 위험합니다.
링크가 전달되거나 로그에 남으면 내부 정보가 노출될 수 있고, 만료·폐기·1회 사용 처리도 까다로워집니다.
더 안전한 기본 구조는 다음과 같습니다.

1. 충분히 긴 랜덤 바이트를 만든다.
2. 토큰 원문은 사용자에게 한 번만 보여준다.
3. 서버에는 토큰 원문이 아니라 해시를 저장한다.
4. 검증 시 입력 토큰을 다시 해시해 저장값과 비교한다.

```js
import { createHash, randomBytes } from 'node:crypto';

function sha256Base64url(value) {
  return createHash('sha256').update(value).digest('base64url');
}

export function createEmailVerificationToken() {
  const token = randomBytes(32).toString('base64url');
  const tokenHash = sha256Base64url(token);

  return {
    token,
    tokenHash,
    expiresAt: new Date(Date.now() + 30 * 60 * 1000),
  };
}
```

해시는 원문 복구가 어렵기 때문에 DB가 일부 노출되더라도 피해를 줄일 수 있습니다.
물론 토큰이 충분히 길고 무작위라는 전제가 필요합니다.
짧은 토큰을 해시해 저장해도 무차별 대입에 취약할 수 있으므로, 인증 링크나 재설정 링크에는 최소 128비트 이상, 가능하면 256비트 수준의 난수를 사용하는 편이 좋습니다.
해시 API의 기본 사용법은 [Node.js crypto.hash 가이드: 짧은 데이터 해시를 간단하게 계산하는 법](/development/blog/seo/2026/05/12/nodejs-crypto-hash-one-shot-digest-guide.html)도 참고할 수 있습니다.

### H3. 비교는 timingSafeEqual로 길이 확인 후 처리한다

토큰 검증에서는 단순 문자열 비교도 대부분의 웹 서비스에서 큰 문제가 되지 않는 경우가 많습니다.
하지만 인증·서명·초대 링크처럼 보안 경계에 가까운 값이라면 비교 방식까지 일관되게 가져가는 편이 좋습니다.
Node.js의 `timingSafeEqual()`은 같은 길이의 버퍼를 상수 시간에 가깝게 비교합니다.

```js
import { createHash, timingSafeEqual } from 'node:crypto';

function digest(value) {
  return createHash('sha256').update(value).digest();
}

export function verifyToken(inputToken, storedTokenHashBase64url) {
  const inputDigest = digest(inputToken);
  const storedDigest = Buffer.from(storedTokenHashBase64url, 'base64url');

  if (inputDigest.byteLength !== storedDigest.byteLength) {
    return false;
  }

  return timingSafeEqual(inputDigest, storedDigest);
}
```

중요한 점은 `timingSafeEqual()`이 토큰 설계 전체를 대신해 주지는 않는다는 것입니다.
만료 시간, 1회 사용 여부, 재시도 제한, 감사 로그, 알림 정책을 함께 설계해야 합니다.
특히 인증 링크 검증 엔드포인트가 반복 호출될 수 있다면 [Node.js rate limiting token bucket 가이드: Redis로 API 남용 막기](/development/blog/seo/2026/03/23/nodejs-rate-limiting-token-bucket-redis-api-protection-guide.html)를 함께 적용하는 것이 좋습니다.

## base64url 사용 시 흔한 실수

### H3. 인코딩을 암호화로 오해하지 않는다

base64url은 암호화가 아닙니다.
사용자가 볼 수 있는 링크에 다음처럼 payload를 그대로 넣으면 누구나 디코딩할 수 있습니다.

```js
const unsafePayload = Buffer.from(JSON.stringify({
  email: 'masked@example.com',
  role: 'admin',
})).toString('base64url');
```

예시에서는 이메일을 마스킹했지만, 실제 코드에서는 이런 구조를 피하는 편이 좋습니다.
사용자에게 보여도 되는 값만 넣고, 권한 판단은 반드시 서버 상태를 기준으로 해야 합니다.
클라이언트가 보낸 인코딩된 값만 믿고 권한을 부여하면 취약점이 됩니다.

### H3. 토큰 길이와 수명을 정책으로 고정한다

토큰 생성 함수가 여러 곳에 흩어져 있으면 서비스마다 길이와 만료 시간이 달라집니다.
처음에는 사소해 보이지만, 장애 대응이나 보안 점검 때 큰 혼란을 만듭니다.
공통 유틸리티로 정책을 고정해 두면 운영이 쉬워집니다.

```js
import { randomBytes } from 'node:crypto';

export const TOKEN_BYTES = 32;
export const EMAIL_VERIFICATION_TTL_MS = 30 * 60 * 1000;

export function createBase64urlToken(bytes = TOKEN_BYTES) {
  return randomBytes(bytes).toString('base64url');
}

export function getExpiresAt(now = Date.now()) {
  return new Date(now + EMAIL_VERIFICATION_TTL_MS);
}
```

이렇게 정책을 한 곳에 모으면 테스트도 쉬워집니다.
만료 시간 검증은 mock timer를 사용해 안정적으로 테스트할 수 있고, 토큰 길이는 정규식과 바이트 길이로 단위 테스트를 만들 수 있습니다.
시간 의존 테스트는 [Node.js test runner mock timers 가이드: 시간 의존 코드를 안정적으로 테스트하기](/development/blog/seo/2026/04/29/nodejs-test-runner-mock-timers-guide.html)를 참고하세요.

## 정리: base64url은 토큰 전달 형식, 보안은 별도 설계

`Buffer`의 `base64url` 인코딩은 URL과 쿠키에 안전한 문자열을 만들 때 간단하고 실용적인 선택입니다.
일반 base64의 `+`, `/`, `=` 문제를 줄이고, Node.js 내장 API만으로 인코딩과 디코딩을 처리할 수 있습니다.

다만 base64url은 데이터를 숨기거나 보호하지 않습니다.
운영 코드에서는 충분히 긴 랜덤 토큰, 서버 측 해시 저장, 만료 시간, 1회 사용 처리, 입력 길이 검증, 재시도 제한을 함께 설계해야 합니다.
즉 base64url은 “전달 형식”이고, 보안은 토큰 생명주기 전체에서 만들어집니다.

## FAQ

### H3. base64url과 encodeURIComponent는 둘 다 필요한가요?

랜덤 토큰을 처음부터 `base64url`로 만들었다면 보통 `encodeURIComponent()` 없이도 URL에 넣기 쉽습니다.
다만 토큰 외의 일반 사용자 입력이나 제목, 검색어 같은 값은 여전히 URL 인코딩이 필요합니다.
base64url은 바이너리 토큰 표현에 적합한 형식이지 모든 URL 값의 검증을 대체하지 않습니다.

### H3. base64url 토큰은 몇 바이트로 만들면 좋나요?

이메일 인증, 초대 링크, 비밀번호 재설정처럼 보안성이 필요한 토큰은 최소 16바이트 이상을 권장하고, 운영 기본값으로는 32바이트를 많이 사용합니다.
32바이트 랜덤 값은 base64url 문자열로 약 43자 정도가 됩니다.
서비스 정책에 따라 더 길게 만들 수 있지만, 너무 긴 토큰은 URL 길이와 로그 가독성도 함께 고려해야 합니다.

### H3. JWT도 base64url을 쓰나요?

네. JWT의 header, payload, signature도 base64url 형식을 사용합니다.
하지만 JWT는 단순 인코딩이 아니라 서명 검증, 만료 시간, issuer/audience 검증 같은 규칙이 함께 있는 토큰 형식입니다.
직접 만든 base64url 문자열을 JWT처럼 신뢰하면 안 되고, JWT를 쓸 때도 반드시 검증 라이브러리와 정책을 명확히 적용해야 합니다.

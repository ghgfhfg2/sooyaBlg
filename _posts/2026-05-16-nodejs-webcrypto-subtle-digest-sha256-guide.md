---
layout: post
title: "Node.js Web Crypto API 가이드: 브라우저와 서버에서 같은 SHA-256 해시 계산하기"
date: 2026-05-16 20:00:00 +0900
lang: ko
translation_key: nodejs-webcrypto-subtle-digest-sha256-guide
permalink: /development/blog/seo/2026/05/16/nodejs-webcrypto-subtle-digest-sha256-guide.html
alternates:
  ko: /development/blog/seo/2026/05/16/nodejs-webcrypto-subtle-digest-sha256-guide.html
  x_default: /development/blog/seo/2026/05/16/nodejs-webcrypto-subtle-digest-sha256-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, webcrypto, sha256, crypto, hash, backend]
description: "Node.js Web Crypto API의 crypto.subtle.digest로 브라우저와 서버에서 같은 SHA-256 해시를 계산하는 방법을 정리했습니다. TextEncoder, ArrayBuffer 변환, base64url 출력, 실무 검증 포인트까지 예제로 설명합니다."
---

브라우저와 Node.js 양쪽에서 같은 해시 값을 계산해야 하는 일이 있습니다.
예를 들어 클라이언트에서 파일 체크섬을 미리 계산하거나, 서버와 프런트엔드가 동일한 서명 전처리 규칙을 공유하거나, 테스트 코드에서 브라우저 런타임과 서버 런타임 결과를 비교하는 경우입니다.
이때 Node.js 전용 `createHash()`만 사용하면 서버 코드는 간단하지만 브라우저와 코드를 공유하기 어렵습니다.

Node.js는 표준 Web Crypto API를 제공합니다.
그중 `crypto.subtle.digest()`를 사용하면 브라우저와 비슷한 방식으로 SHA-256 같은 해시를 계산할 수 있습니다.
이 글에서는 Node.js Web Crypto API로 문자열과 바이너리 데이터를 해시하는 방법, `ArrayBuffer` 결과를 hex·base64url 문자열로 바꾸는 방법, 실무에서 혼동하기 쉬운 인코딩 기준을 정리합니다.
Node.js 전용 해시 API가 더 알맞은 상황은 [Node.js crypto.hash 가이드: 짧은 데이터 해시를 간단하게 계산하는 법](/development/blog/seo/2026/05/12/nodejs-crypto-hash-one-shot-digest-guide.html)도 함께 참고하세요.

## Web Crypto API를 쓰면 좋은 상황

### H3. 브라우저와 Node.js에서 같은 코드를 공유한다

`crypto.subtle.digest()`는 Web Crypto 표준 API입니다.
브라우저에서도 쓰이고 Node.js에서도 `globalThis.crypto` 또는 `node:crypto`의 `webcrypto`를 통해 사용할 수 있습니다.
서버 전용 배치 작업이라면 `createHash()`가 더 익숙할 수 있지만, 프런트엔드와 공유되는 유틸리티라면 Web Crypto API가 중복 구현을 줄여 줍니다.

```js
const encoder = new TextEncoder();
const bytes = encoder.encode('hello node');

const digest = await crypto.subtle.digest('SHA-256', bytes);
console.log(digest instanceof ArrayBuffer); // true
```

여기서 반환값은 문자열이 아니라 `ArrayBuffer`입니다.
그래서 로그, URL, DB 컬럼, 테스트 스냅샷에 쓰려면 hex나 base64url 같은 문자열 형식으로 한 번 더 변환해야 합니다.
이 변환 규칙을 프로젝트 안에서 통일해 두면 런타임이 달라도 같은 입력에서 같은 출력이 나옵니다.

### H3. 해시는 암호화가 아니라 무결성 확인 도구다

SHA-256 해시는 입력을 고정 길이 요약값으로 바꾸는 기능입니다.
해시 값만 보고 원문을 복구하기 어렵지만, 이것이 곧 암호화라는 뜻은 아닙니다.
입력이 이메일, 전화번호, 짧은 코드처럼 추측 가능한 값이면 공격자가 후보 값을 많이 넣어 같은 해시를 찾을 수 있습니다.

따라서 해시를 개인정보 보호 수단으로 과신하면 안 됩니다.
민감정보는 가능한 한 저장하지 않고, 꼭 저장해야 한다면 별도의 보안 설계와 접근 통제를 적용해야 합니다.
개발 글 예제에서도 실제 토큰이나 개인정보가 노출되지 않도록 [로그 예제 정제 가이드: 믿을 수 있는 개발 글을 위한 마스킹 원칙](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)을 기준으로 샘플 값을 다듬는 편이 안전합니다.

## 문자열을 SHA-256으로 해시하기

### H3. TextEncoder로 UTF-8 바이트를 만든다

`subtle.digest()`는 문자열을 직접 받지 않습니다.
반드시 `BufferSource`, 즉 `ArrayBuffer`, `TypedArray`, `DataView` 같은 바이트 데이터를 넘겨야 합니다.
문자열을 해시할 때는 `TextEncoder`로 UTF-8 바이트를 만드는 과정을 명시합니다.

```js
export async function sha256ArrayBuffer(text) {
  const bytes = new TextEncoder().encode(text);
  return crypto.subtle.digest('SHA-256', bytes);
}

const digest = await sha256ArrayBuffer('안녕하세요');
console.log(digest.byteLength); // 32
```

이 방식은 한글, 이모지, 공백, 줄바꿈까지 UTF-8 기준으로 처리합니다.
중요한 점은 입력 문자열을 만들기 전 단계입니다.
객체를 해시하려면 JSON 직렬화 순서, 날짜 형식, 공백 처리, 숫자 포맷을 먼저 안정화해야 합니다.
API 계약이 흔들리면 해시 함수가 같아도 서로 다른 결과가 나옵니다.
계약 검증은 [Node.js OpenAPI + Zod 가이드: API 계약 검증으로 응답 일관성 지키기](/development/blog/seo/2026/03/21/nodejs-openapi-zod-contract-validation-api-consistency-guide.html)처럼 경계에서 정리하는 습관이 좋습니다.

### H3. ArrayBuffer를 hex 문자열로 변환한다

해시 결과를 사람이 읽기 쉬운 로그나 테스트 값으로 남길 때는 hex 문자열이 편합니다.
`ArrayBuffer`를 `Uint8Array`로 감싼 뒤 각 바이트를 16진수 두 자리로 바꾸면 됩니다.

```js
export function arrayBufferToHex(buffer) {
  return [...new Uint8Array(buffer)]
    .map((byte) => byte.toString(16).padStart(2, '0'))
    .join('');
}

export async function sha256Hex(text) {
  const digest = await sha256ArrayBuffer(text);
  return arrayBufferToHex(digest);
}

console.log(await sha256Hex('hello node'));
// 예: 2f4f... 형태의 64자 hex 문자열
```

SHA-256은 32바이트 결과를 만들고, hex로 표현하면 64자가 됩니다.
테스트에서는 길이와 문자 집합을 함께 확인하면 실수를 빨리 잡을 수 있습니다.
예를 들어 대문자 hex와 소문자 hex가 섞이면 문자열 비교가 실패할 수 있으므로 프로젝트 표준을 하나로 정해야 합니다.

## URL에 넣을 해시는 base64url로 표현하기

### H3. Buffer를 이용해 base64url 문자열로 바꾼다

해시 값을 URL, 쿠키, 파일명 일부에 넣어야 한다면 hex보다 base64url이 짧고 다루기 편합니다.
Node.js에서는 Web Crypto 결과인 `ArrayBuffer`를 `Buffer.from()`으로 감싼 뒤 `toString('base64url')`을 호출할 수 있습니다.

```js
export function arrayBufferToBase64url(buffer) {
  return Buffer.from(buffer).toString('base64url');
}

export async function sha256Base64url(text) {
  const digest = await sha256ArrayBuffer(text);
  return arrayBufferToBase64url(digest);
}

console.log(await sha256Base64url('hello node'));
// URL에 넣기 쉬운 짧은 문자열
```

base64url은 `+`, `/`, `=` 같은 URL 처리에 불편한 문자를 피합니다.
인증 링크, 캐시 키, 파일 체크섬 URL을 설계할 때 유용합니다.
더 자세한 인코딩 차이는 [Node.js Buffer base64url 가이드: URL에 안전한 토큰 인코딩하는 법](/development/blog/seo/2026/05/16/nodejs-buffer-base64url-encoding-guide.html)에 정리해 두었습니다.

### H3. 랜덤 토큰과 입력 해시는 목적이 다르다

해시는 “입력에서 결정적으로 나온 값”입니다.
같은 입력은 항상 같은 해시가 됩니다.
반대로 인증 토큰, 비밀번호 재설정 링크, 초대 코드처럼 추측이 어려워야 하는 값은 해시가 아니라 충분히 긴 랜덤 바이트에서 시작해야 합니다.

```js
import { randomBytes } from 'node:crypto';

const token = randomBytes(32).toString('base64url');
```

토큰 원문을 저장하지 않기 위해 서버에 토큰 해시만 저장하는 패턴은 괜찮습니다.
하지만 짧은 사용자 입력을 단순히 SHA-256으로 바꾼 값을 토큰처럼 쓰는 것은 안전하지 않습니다.
랜덤 식별자 설계는 [Node.js crypto.randomUUID 가이드: 안전한 ID 생성과 실무 적용 기준](/development/blog/seo/2026/05/08/nodejs-crypto-randomuuid-safe-id-generation-guide.html)과 함께 구분해서 보는 편이 좋습니다.

## 실무에서 확인할 체크리스트

### H3. 런타임 지원 범위를 먼저 고정한다

현대 Node.js에서는 `globalThis.crypto`를 바로 사용할 수 있습니다.
다만 오래된 런타임이나 테스트 환경에서는 전역 `crypto`가 없을 수 있습니다.
라이브러리 코드라면 의존하는 Node.js 최소 버전을 문서화하고, 필요하면 `node:crypto`에서 `webcrypto`를 가져오는 방식을 준비합니다.

```js
import { webcrypto } from 'node:crypto';

const subtle = globalThis.crypto?.subtle ?? webcrypto.subtle;

export async function digestSha256(bytes) {
  return subtle.digest('SHA-256', bytes);
}
```

이렇게 해 두면 테스트 환경 차이로 생기는 실패를 줄일 수 있습니다.
운영 코드에서는 런타임 버전, 인코딩 규칙, 출력 형식을 README나 API 문서에 같이 남기는 것이 좋습니다.

### H3. 큰 파일 해시는 스트리밍 전략을 따로 검토한다

`subtle.digest()`는 입력 전체를 한 번에 받아 해시합니다.
짧은 문자열, 작은 JSON, 작은 파일 체크섬에는 편하지만 대용량 파일을 통째로 메모리에 올려 처리하는 구조에는 맞지 않을 수 있습니다.
Node.js 서버에서 큰 파일을 다룬다면 스트림 기반 처리나 업로드 경로의 메모리 제한을 함께 설계해야 합니다.

```js
// 작은 입력에는 간단하다.
const digest = await crypto.subtle.digest('SHA-256', new Uint8Array([1, 2, 3]));

// 큰 파일은 전체를 ArrayBuffer로 올리기 전에 메모리 사용량을 검토한다.
```

파일 업로드처럼 바이너리 데이터를 다루는 흐름은 [Node.js fs.openAsBlob 가이드: 파일을 Blob처럼 읽어 업로드하기](/development/blog/seo/2026/05/06/nodejs-fs-openasblob-file-to-blob-upload-guide.html)와 연결해서 보면 좋습니다.
해시 계산은 간단한 함수처럼 보이지만, 입력 크기와 처리 위치에 따라 성능 특성이 크게 달라집니다.

## FAQ

### H3. Web Crypto API와 node:crypto 중 무엇을 써야 하나요?

브라우저와 공유할 코드이거나 표준 API 기준으로 맞추고 싶다면 Web Crypto API가 좋습니다.
Node.js 서버 전용 코드이고 스트림 처리, HMAC, 다양한 해시 옵션이 필요하다면 `node:crypto`가 더 익숙하고 넓은 선택지를 제공합니다.
중요한 것은 한 프로젝트 안에서 입력 인코딩과 출력 포맷을 일관되게 정하는 것입니다.

### H3. SHA-256 해시는 비밀번호 저장에 써도 되나요?

일반 SHA-256만으로 비밀번호를 저장하는 것은 적절하지 않습니다.
비밀번호 저장에는 bcrypt, scrypt, Argon2처럼 느리게 설계된 비밀번호 해싱 알고리즘과 salt가 필요합니다.
`subtle.digest('SHA-256')`는 체크섬, 캐시 키, 서명 전처리처럼 목적이 분명한 곳에 사용하세요.

### H3. base64url 해시 값은 안전한가요?

base64url은 표현 형식일 뿐 보안 기능이 아닙니다.
해시가 어떤 입력에서 만들어졌는지, 입력이 충분히 예측 불가능한지, 결과를 어디에 노출하는지가 더 중요합니다.
URL에 넣기 쉬운 문자열이 필요할 때 base64url을 선택하되, 민감한 값은 별도 보안 설계를 적용해야 합니다.

## 마무리

Node.js Web Crypto API의 `crypto.subtle.digest()`를 사용하면 브라우저와 서버에서 같은 방식으로 SHA-256 해시를 계산할 수 있습니다.
핵심은 문자열을 `TextEncoder`로 UTF-8 바이트로 바꾸고, 결과 `ArrayBuffer`를 프로젝트 표준에 맞게 hex나 base64url로 변환하는 것입니다.

실무에서는 해시와 암호화, 해시와 랜덤 토큰을 구분해야 합니다.
작은 입력의 무결성 확인에는 Web Crypto API가 간결하지만, 대용량 파일·서버 전용 암호 기능·비밀번호 저장에는 각각 다른 도구와 설계가 필요합니다.
출력 형식과 입력 정규화 규칙을 문서화해 두면 런타임이 달라도 재현 가능한 해시 값을 유지할 수 있습니다.

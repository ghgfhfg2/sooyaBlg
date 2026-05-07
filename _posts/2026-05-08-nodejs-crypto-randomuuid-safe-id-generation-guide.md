---
layout: post
title: "Node.js crypto.randomUUID 가이드: 안전한 식별자를 직접 만드는 법"
date: 2026-05-08 08:00:00 +0900
lang: ko
translation_key: nodejs-crypto-randomuuid-safe-id-generation-guide
permalink: /development/blog/seo/2026/05/08/nodejs-crypto-randomuuid-safe-id-generation-guide.html
alternates:
  ko: /development/blog/seo/2026/05/08/nodejs-crypto-randomuuid-safe-id-generation-guide.html
  x_default: /development/blog/seo/2026/05/08/nodejs-crypto-randomuuid-safe-id-generation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, crypto, uuid, security, backend]
description: "Node.js crypto.randomUUID로 안전한 UUID 식별자를 만드는 방법을 정리했습니다. 임시 파일명, 요청 추적 ID, idempotency key, 사용 시 주의사항까지 실무 예제로 설명합니다."
---

서비스를 만들다 보면 작지만 중요한 식별자가 계속 필요합니다.
요청 추적 ID, 임시 파일명, 작업 ID, 외부 API 재시도 키처럼 **겹치면 곤란하지만 데이터베이스의 순차 ID까지는 필요 없는 값**들이 대표적입니다.

이때 시간을 붙이거나 `Math.random()`을 조합해 문자열을 만들 수도 있지만, 운영 환경에서는 충돌 가능성과 예측 가능성을 함께 생각해야 합니다.
Node.js의 `crypto.randomUUID()`는 이런 상황에서 표준 UUID를 간단하고 안전하게 만들 수 있는 기본 API입니다.

## crypto.randomUUID가 필요한 이유

### H3. 직접 만든 랜덤 문자열은 생각보다 취약하다

아래 코드는 흔히 볼 수 있는 임시 식별자 생성 방식입니다.

```js
const id = `${Date.now()}-${Math.random().toString(36).slice(2)}`;

console.log(id);
```

개발 중에는 충분해 보이지만, 실제 서비스에서는 몇 가지 문제가 있습니다.

- 생성 규칙이 일정해 예측하기 쉽다
- 짧은 랜덤 문자열은 트래픽이 늘면 충돌 가능성이 커진다
- 보안 목적 식별자처럼 오해해서 재사용할 위험이 있다
- 테스트와 운영 코드에 제각각 다른 생성 규칙이 퍼질 수 있다

`crypto.randomUUID()`를 쓰면 최소한 UUID 생성 방식은 표준 API에 맡길 수 있습니다.
직접 문자열 규칙을 설계하고 검증하는 부담이 줄어듭니다.

### H3. 식별자 생성은 보안과 운영 안정성의 경계에 있다

식별자는 단순한 문자열처럼 보이지만, 잘못 쓰면 운영 문제로 이어집니다.
예를 들어 요청 ID가 자주 겹치면 로그 추적이 어려워지고, 임시 파일명이 겹치면 기존 파일을 덮어쓸 수 있습니다.
외부 API 재시도 키가 불안정하면 중복 요청 방지 흐름도 흔들릴 수 있습니다.

중복 요청 방지 설계가 필요하다면 [Node.js Idempotency Key 가이드: 중복 요청을 안전하게 막는 법](/development/blog/seo/2026/04/10/nodejs-idempotency-key-duplicate-request-prevention-guide.html)과 함께 보는 것이 좋습니다.
`randomUUID()`는 이런 패턴에서 키 후보를 만들 때 유용하지만, 저장·만료·중복 검증 정책은 별도로 설계해야 합니다.

## crypto.randomUUID 기본 사용법

### H3. UUID 문자열 만들기

사용법은 매우 단순합니다.

```js
import { randomUUID } from 'node:crypto';

const requestId = randomUUID();

console.log(requestId);
// 예: 8f6f8f1e-7d4c-4c1e-9d62-1f3b9f0c8b7a
```

`randomUUID()`는 문자열을 반환합니다.
별도 패키지를 설치하지 않아도 되고, Node.js 표준 모듈만으로 사용할 수 있습니다.

### H3. CommonJS에서도 사용할 수 있다

CommonJS 프로젝트라면 아래처럼 가져오면 됩니다.

```js
const { randomUUID } = require('node:crypto');

const jobId = randomUUID();
console.log(jobId);
```

새 프로젝트라면 ESM과 `node:` 접두어를 함께 쓰는 방식을 추천하지만, 기존 코드베이스에서는 현재 모듈 시스템에 맞춰 일관성을 유지하는 편이 더 중요합니다.
ESM에서 경로와 실행 기준을 함께 다룬다면 [Node.js import.meta.dirname 가이드: ESM에서 현재 파일 경로를 안전하게 다루는 법](/development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html)도 이어서 참고할 만합니다.

## 실무에서 자주 쓰는 패턴

### H3. 요청 추적 ID 만들기

API 서버에서는 요청마다 추적 ID를 붙여 두면 로그를 훨씬 찾기 쉬워집니다.

```js
import { randomUUID } from 'node:crypto';

function withRequestId(req, res, next) {
  req.requestId = req.headers['x-request-id'] || randomUUID();
  res.setHeader('x-request-id', req.requestId);
  next();
}
```

외부에서 이미 `x-request-id`를 넘겨 주는 환경이라면 그 값을 이어받고, 없을 때만 새로 만드는 방식이 실용적입니다.
다만 외부 입력을 그대로 신뢰하기보다 길이와 문자 집합을 제한하는 검증을 추가하는 편이 좋습니다.

요청 단위 컨텍스트에 ID를 보관하려면 [Node.js AsyncLocalStorage 가이드: 요청 컨텍스트 로깅을 안전하게 관리하는 법](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)과 연결해서 설계할 수 있습니다.

### H3. 임시 파일명 충돌 줄이기

파일 업로드나 변환 작업에서는 임시 파일명이 겹치지 않도록 만들어야 합니다.

```js
import { randomUUID } from 'node:crypto';
import path from 'node:path';

function makeTempImagePath(dir, ext = '.png') {
  const safeExt = ext.startsWith('.') ? ext : `.${ext}`;
  return path.join(dir, `${randomUUID()}${safeExt}`);
}

console.log(makeTempImagePath('/tmp/uploads', 'webp'));
```

여기서 중요한 점은 UUID가 파일 경로 검증을 대신하지 않는다는 것입니다.
업로드 원본 파일명이나 확장자를 사용자 입력에서 가져온다면, 허용 확장자 목록과 저장 디렉터리 제한을 별도로 적용해야 합니다.

### H3. 작업 큐의 jobId 후보로 사용하기

백그라운드 작업을 큐에 넣을 때도 UUID는 좋은 기본값이 될 수 있습니다.

```js
import { randomUUID } from 'node:crypto';

function createExportJob(userId) {
  return {
    id: randomUUID(),
    type: 'export-report',
    userId,
    status: 'pending',
    createdAt: new Date().toISOString(),
  };
}
```

다만 작업을 정확히 한 번만 처리해야 한다면 UUID만으로는 부족합니다.
큐 저장소의 유니크 제약, 재시도 정책, 실패 작업 보관 전략이 함께 필요합니다.
실패 작업을 따로 다루는 흐름은 [Node.js Dead Letter Queue 가이드: 실패한 작업을 안전하게 복구하는 법](/development/blog/seo/2026/04/02/nodejs-dead-letter-queue-dlq-failure-recovery-guide.html)과도 이어집니다.

## crypto.randomUUID 사용 시 주의할 점

### H3. UUID는 비밀번호나 인증 토큰이 아니다

`randomUUID()`는 안전한 난수 기반 UUID를 만들 때 유용하지만, 그 자체를 모든 보안 토큰의 정답처럼 쓰면 안 됩니다.
세션 토큰, 비밀번호 재설정 토큰, 이메일 인증 토큰처럼 노출 시 피해가 큰 값은 요구 보안 수준과 만료 정책을 따로 정해야 합니다.

이런 값이 필요하다면 보통 다음 요소를 함께 봅니다.

- 충분한 엔트로피
- 서버 측 저장 또는 해시 저장 여부
- 짧은 만료 시간
- 1회 사용 처리
- 전송 경로와 로그 마스킹

UUID는 식별자에는 적합하지만, 인증·인가의 근거로 단독 사용하기에는 설계 검토가 더 필요합니다.

### H3. 사용자에게 노출되는 ID라면 의미를 분리한다

UUID를 공개 URL에 넣는 경우도 있습니다.
예를 들어 `/share/uuid` 같은 링크를 만들 수 있습니다.
하지만 UUID가 길고 무작위라는 이유만으로 접근 제어를 생략하면 위험합니다.

공개 링크라면 최소한 아래를 분리해서 생각해야 합니다.

1. 리소스의 내부 기본키
2. 외부에 노출할 공유 식별자
3. 공유 링크의 만료 또는 회수 정책
4. 접근 권한 검증

식별자를 어렵게 추측하게 만드는 것과 권한을 검증하는 것은 다른 문제입니다.

### H3. 정렬이 필요한 ID에는 맞지 않을 수 있다

UUID v4 형태의 랜덤 ID는 시간순 정렬에 유리하지 않습니다.
생성 순서대로 정렬해야 하거나 데이터베이스 인덱스 지역성이 중요한 시스템이라면 다른 ID 전략을 검토할 수 있습니다.

예를 들어 아래처럼 목적에 따라 선택지가 달라집니다.

- 로그 추적 ID: `randomUUID()`가 좋은 기본값
- 공개 공유 ID: 권한·만료 정책과 함께 검토
- 시간순 정렬 ID: ULID, UUID v7 등 별도 전략 검토
- 내부 DB 기본키: 데이터베이스 특성과 조회 패턴 기준으로 결정

블로그 예제나 자동화 스크립트처럼 운영 부담이 낮은 곳에서는 `randomUUID()`가 충분히 실용적인 기본값입니다.

## 적용 체크리스트

### H3. 도입 전에 이 다섯 가지를 확인한다

1. 이 값이 단순 식별자인가, 인증 토큰인가?
2. 충돌 시 어떤 장애가 생기는가?
3. 외부에 노출되는 값인가?
4. 정렬 가능성이 필요한가?
5. 로그와 에러 메시지에 노출되어도 괜찮은가?

이 질문에 답하면 `randomUUID()`를 써도 되는 곳과 별도 토큰 설계가 필요한 곳을 더 쉽게 구분할 수 있습니다.

## 마무리

`crypto.randomUUID()`는 Node.js에서 안전한 UUID 식별자를 만들 때 가장 먼저 고려할 만한 기본 도구입니다.
패키지 의존성을 늘리지 않고도 요청 ID, 작업 ID, 임시 파일명 같은 값을 일관되게 생성할 수 있습니다.

정리하면 이렇게 기억하면 됩니다.

- 직접 만든 랜덤 문자열보다 표준 API를 우선한다
- UUID와 인증 토큰의 역할을 구분한다
- 식별자 생성보다 저장·검증·만료 정책이 더 중요할 때가 있다

## 함께 보면 좋은 글

- [Node.js Idempotency Key 가이드: 중복 요청을 안전하게 막는 법](/development/blog/seo/2026/04/10/nodejs-idempotency-key-duplicate-request-prevention-guide.html)
- [Node.js AsyncLocalStorage 가이드: 요청 컨텍스트 로깅을 안전하게 관리하는 법](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)
- [Node.js Dead Letter Queue 가이드: 실패한 작업을 안전하게 복구하는 법](/development/blog/seo/2026/04/02/nodejs-dead-letter-queue-dlq-failure-recovery-guide.html)

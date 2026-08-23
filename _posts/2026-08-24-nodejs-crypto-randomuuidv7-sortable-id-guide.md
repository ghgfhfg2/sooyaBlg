---
layout: post
title: "Node.js crypto.randomUUIDv7 가이드: 정렬 가능한 UUID를 안전하게 쓰는 법"
date: 2026-08-24 08:00:00 +0900
lang: ko
translation_key: nodejs-crypto-randomuuidv7-sortable-id-guide
permalink: /development/blog/seo/2026/08/24/nodejs-crypto-randomuuidv7-sortable-id-guide.html
alternates:
  ko: /development/blog/seo/2026/08/24/nodejs-crypto-randomuuidv7-sortable-id-guide.html
  x_default: /development/blog/seo/2026/08/24/nodejs-crypto-randomuuidv7-sortable-id-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, crypto, randomuuidv7, uuidv7, sortable-id, database, backend, javascript]
description: "Node.js crypto.randomUUIDv7로 시간 기반 정렬이 가능한 UUID를 생성하는 방법을 정리합니다. UUID v4와 v7의 차이, 데이터베이스 키 설계, 단조 증가 한계, entropy cache 옵션과 운영 체크리스트까지 실무 예제로 설명합니다."
---

서비스에서 ID를 만들 때 가장 먼저 떠올리는 선택지는 보통 숫자 시퀀스, UUID v4, 혹은 별도 ID 생성기입니다.
숫자 시퀀스는 정렬과 인덱스 효율이 좋지만 여러 리전, 여러 서비스, 오프라인 생성이 섞이면 운영 부담이 커집니다.
UUID v4는 충돌 가능성이 매우 낮고 어디서든 만들기 쉽지만, 값이 무작위라 데이터베이스 인덱스와 시간순 조회에는 불리할 수 있습니다.

Node.js에는 `crypto.randomUUIDv7()` API가 추가되어 시간 기반 정렬이 가능한 UUID v7을 생성할 수 있습니다.
UUID v7은 앞쪽 비트에 밀리초 단위 Unix timestamp를 담고 뒤쪽에 암호학적으로 안전한 랜덤 값을 채우는 방식이라, 대략적인 생성 시간 순서로 정렬하기 좋습니다.
다만 Node.js 공식 문서 기준으로 포함된 timestamp는 단조 증가를 보장하지 않습니다.
따라서 "정렬에 유리한 ID"로 이해해야지, "항상 순번처럼 증가하는 ID"로 기대하면 안 됩니다.

이 글에서는 Node.js에서 `crypto.randomUUIDv7()`을 쓰는 기본 구조와 UUID v4와의 선택 기준, 데이터베이스 키 설계, 민감정보 없는 로그 운영 방법을 정리합니다.
비밀번호 해싱처럼 보안성 있는 랜덤 값과 파라미터 관리가 필요한 작업은 [Node.js crypto.argon2 비밀번호 해싱 가이드](/development/blog/seo/2026/08/22/nodejs-crypto-argon2-password-hashing-guide.html), 일반 UUID v4와 토큰 생성 기준은 [Node.js Web Crypto SHA-256 가이드](/development/blog/seo/2026/05/16/nodejs-webcrypto-subtle-digest-sha256-guide.html), 중복 요청 방지는 [Node.js Idempotency-Key 가이드](/development/blog/seo/2026/04/10/nodejs-idempotency-key-duplicate-request-prevention-guide.html)와 함께 보면 좋습니다.

## UUID v7을 검토할 때

### 무작위 UUID v4의 장점과 한계

`crypto.randomUUID()`가 생성하는 UUID v4는 거의 전부가 랜덤 값입니다.
중앙 서버 없이 여러 프로세스에서 동시에 만들어도 충돌 위험이 낮고, 외부에 노출해도 순번을 쉽게 추측하기 어렵습니다.
그래서 공개 API의 리소스 ID, 작업 ID, 파일 ID처럼 넓은 범위에서 무난한 기본값이 됩니다.

하지만 v4는 시간 정보가 없기 때문에 값 자체로 생성 순서를 알 수 없습니다.
데이터베이스 기본 키로 쓰면 새 레코드가 인덱스의 임의 위치에 들어가고, 최근 생성 항목을 조회할 때도 별도의 `created_at` 컬럼이 필요합니다.
물론 `created_at`은 어차피 있어야 하는 컬럼이지만, 대량 삽입과 범위 조회가 많은 테이블에서는 ID의 정렬 특성이 성능과 운영 편의성에 영향을 줄 수 있습니다.

UUID v7은 이 지점에서 선택지가 됩니다.
ID만 정렬해도 대략적인 생성 시간 순서가 나오고, 데이터베이스 인덱스에 새 값이 비교적 뒤쪽으로 들어가므로 v4보다 쓰기 패턴을 예측하기 쉽습니다.

### UUID v7은 시간순에 가깝지만 순번은 아니다

UUID v7은 timestamp를 포함하므로 문자열 정렬만으로도 시간순에 가까운 순서를 얻을 수 있습니다.
그러나 같은 밀리초 안에서 여러 ID가 만들어질 수 있고, 시스템 시계가 뒤로 움직일 수도 있습니다.
Node.js 문서도 `crypto.randomUUIDv7()`의 timestamp가 비단조 clock에 의존하며 엄격한 증가 순서를 보장하지 않는다고 설명합니다.

따라서 아래 용도에는 잘 맞습니다.

- 최근 생성 데이터가 많은 테이블의 기본 키 후보
- 로그, 이벤트, 작업 ID처럼 대략적인 시간 정렬이 도움이 되는 값
- 여러 프로세스에서 중앙 조정 없이 생성해야 하는 공개 식별자
- UUID 형식이 필요한 외부 연동 ID

반대로 아래 요구사항이라면 별도 설계가 필요합니다.

- 결제 승인 번호처럼 법적, 회계적으로 연속 번호가 필요한 값
- 같은 밀리초 안에서도 반드시 생성 순서가 보존되어야 하는 큐 순번
- 사용자가 생성량을 추정하면 안 되는 민감한 비즈니스 도메인
- 오래된 Node.js 런타임까지 반드시 지원해야 하는 라이브러리

ID는 단순한 문자열처럼 보여도 운영 계약입니다.
나중에 바꾸기 어려우므로 정렬, 노출, 보존 기간, 외부 연동 요구사항을 먼저 정리한 뒤 선택해야 합니다.

## Node.js에서 UUID v7 생성하기

### 기본 사용법

Node.js에서 UUID v7은 `node:crypto` 모듈의 `randomUUIDv7()`로 생성합니다.
런타임 버전이 충분히 최신인지 확인한 뒤 사용하세요.

```js
import { randomUUIDv7 } from 'node:crypto';

const orderId = randomUUIDv7();

console.log(orderId);
```

반환값은 UUID 문자열입니다.
기존 UUID 컬럼, 로그 필드, JSON 응답에서 다루기 쉽고, 별도 바이너리 인코딩을 도입하지 않아도 됩니다.

CommonJS 코드라면 아래처럼 가져올 수 있습니다.

```js
const { randomUUIDv7 } = require('node:crypto');

const jobId = randomUUIDv7();
```

라이브러리 코드라면 지원 Node.js 버전을 명확히 문서화하세요.
애플리케이션 코드라면 `engines.node`, Docker 이미지 태그, CI Node.js 버전이 실제 배포 버전과 일치하는지 확인하는 편이 좋습니다.

### 런타임 지원 여부를 방어적으로 확인한다

사내 패키지나 여러 서비스가 함께 쓰는 유틸리티에서는 런타임이 기대보다 낮을 수 있습니다.
이때 애플리케이션 시작 시점에 명확한 오류를 내면 장애 원인을 빨리 찾을 수 있습니다.

```js
import * as crypto from 'node:crypto';

export function createSortableId() {
  if (typeof crypto.randomUUIDv7 !== 'function') {
    throw new Error('crypto.randomUUIDv7 requires a Node.js runtime that supports UUID v7');
  }

  return crypto.randomUUIDv7();
}
```

조용히 UUID v4로 fallback하는 방식은 신중해야 합니다.
호출자는 정렬 가능한 ID를 기대하는데 일부 환경에서만 v4가 섞이면 정렬, 테스트, 인덱스 특성이 달라집니다.
fallback을 넣어야 한다면 함수 이름과 문서에서 "정렬 가능한 경우도 있는 ID"가 아니라 정확한 보장 범위를 드러내야 합니다.

### entropy cache 옵션을 이해한다

`randomUUIDv7()`은 `randomUUID()`와 마찬가지로 `disableEntropyCache` 옵션을 받습니다.
기본값에서는 성능을 위해 여러 UUID를 만들 수 있는 랜덤 데이터를 캐시합니다.

```js
import { randomUUIDv7 } from 'node:crypto';

const id = randomUUIDv7({
  disableEntropyCache: true
});
```

대부분의 웹 애플리케이션에서는 기본값을 유지하는 편이 좋습니다.
캐시는 UUID 생성을 빠르게 하기 위한 런타임 내부 최적화이며, 일반적인 요청 ID나 레코드 ID 생성에서 매번 비활성화할 이유는 많지 않습니다.

다만 보안 감사나 특정 컴플라이언스 요구사항에서 "각 ID 생성마다 캐시된 랜덤 데이터를 쓰지 않는다"는 운영 기준을 요구할 수 있습니다.
그런 경우에만 성능 영향을 측정한 뒤 옵션을 켜는 것이 안전합니다.

## 데이터베이스 키로 쓸 때의 설계

### 기본 키와 생성 시간 컬럼을 함께 둔다

UUID v7 안에 timestamp가 들어 있더라도 `created_at` 컬럼은 따로 두는 편이 좋습니다.
ID는 식별자이고, `created_at`은 도메인 이벤트 시간입니다.
둘을 같은 것으로 취급하면 나중에 데이터 이관, 재처리, 백필, 타임존 표시, 감사 로그에서 애매해집니다.

```sql
CREATE TABLE jobs (
  id TEXT PRIMARY KEY,
  status TEXT NOT NULL,
  payload_json TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

애플리케이션에서는 ID와 생성 시간을 같은 시점에 만들되, 의미는 분리합니다.

```js
import { randomUUIDv7 } from 'node:crypto';

export function createJobRecord(payload) {
  const now = new Date().toISOString();

  return {
    id: randomUUIDv7(),
    status: 'queued',
    payloadJson: JSON.stringify(payload),
    createdAt: now,
    updatedAt: now
  };
}
```

정렬 쿼리는 요구사항에 따라 선택하세요.
최근 생성순 화면이라면 `created_at DESC`가 더 명확합니다.
샤딩, 페이지네이션, 인덱스 지역성이 중요하다면 `id` 정렬도 검토할 수 있습니다.
핵심은 ID에 시간이 들어 있다는 이유만으로 업무 시간을 대체하지 않는 것입니다.

### 커서 페이지네이션에서 사용할 수 있다

UUID v7은 커서 페이지네이션에도 잘 어울립니다.
최신순으로 정렬할 때 `created_at`과 `id`를 함께 쓰면 같은 밀리초에 생성된 데이터도 안정적으로 넘길 수 있습니다.

```sql
SELECT id, title, created_at
FROM posts
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

단일 `id`만 커서로 쓰는 방식도 가능하지만, 비즈니스 시간 기준 정렬이 필요한 화면에서는 `created_at`을 함께 쓰는 편이 의도가 더 분명합니다.
반대로 내부 이벤트 저장소처럼 ID 생성 시간이 곧 이벤트 수집 시간인 시스템에서는 `id` 기반 범위 조회가 단순할 수 있습니다.

페이지네이션에서 중요한 것은 "정렬 기준이 유일하고 안정적인가"입니다.
`created_at`만 쓰면 같은 시간의 행 사이에서 순서가 흔들릴 수 있으므로 보조 정렬 키로 `id`를 붙이는 습관을 들이면 좋습니다.

### 문자열 저장과 바이너리 저장을 구분한다

UUID는 문자열로 저장하면 다루기 쉽고 디버깅이 편합니다.
로그, 관리자 화면, API 응답에서도 그대로 읽을 수 있습니다.
대부분의 중소 규모 서비스에서는 문자열 저장만으로 충분합니다.

반면 초대형 테이블에서 저장 공간과 인덱스 크기가 중요한 경우에는 UUID를 바이너리로 저장하는 전략을 검토할 수 있습니다.
다만 이 경우 애플리케이션 코드, 마이그레이션, 운영 도구, 데이터 분석 쿼리의 복잡도가 올라갑니다.

처음부터 바이너리 최적화를 선택하기보다 아래 질문에 답해보세요.

- 테이블이 실제로 수억 행 이상으로 커질 예정인가
- 문자열 UUID 인덱스가 병목이라는 측정 결과가 있는가
- 운영자가 바이너리 값을 사람이 읽을 수 있는 형태로 쉽게 변환할 수 있는가
- 백업, 복구, 데이터 이관 도구가 같은 형식을 이해하는가

측정 없이 복잡한 저장 형식을 먼저 도입하면 성능보다 유지보수 비용이 먼저 늘어납니다.

## 운영 코드에서 주의할 점

### ID에 의미를 너무 많이 싣지 않는다

UUID v7은 생성 시간을 일부 드러냅니다.
이 점은 정렬에는 장점이지만, 공개 URL이나 외부 API에서 생성 시점이 노출된다는 뜻이기도 합니다.
대부분의 게시글, 작업, 이벤트 ID에서는 문제가 되지 않지만, 민감한 도메인에서는 검토가 필요합니다.

예를 들어 사용자 가입 ID, 보안 이벤트 ID, 내부 조사 케이스 ID처럼 생성 시점 자체가 민감할 수 있는 값은 별도 공개 ID를 두는 방법이 더 안전할 수 있습니다.
내부 기본 키와 외부 공개 식별자를 분리하면 운영자는 정렬 가능한 키를 쓰면서도 사용자에게 노출되는 정보는 줄일 수 있습니다.

```js
import { randomUUID, randomUUIDv7 } from 'node:crypto';

export function createCaseRecord(input) {
  return {
    id: randomUUIDv7(),
    publicId: randomUUID(),
    subject: input.subject,
    createdAt: new Date().toISOString()
  };
}
```

모든 테이블에 공개 ID를 둘 필요는 없습니다.
하지만 "이 값이 URL, 로그, 고객 지원 화면, 외부 파트너에게 노출되는가"는 ID 설계 때 반드시 확인해야 합니다.

### 로그에는 원문보다 맥락을 남긴다

ID는 보통 민감정보는 아니지만, 사용자 행동이나 내부 사건과 연결되면 추적 가능한 정보가 됩니다.
로그에는 필요한 맥락을 남기되, 불필요하게 요청 본문이나 인증 값을 함께 기록하지 않도록 분리하세요.

```js
logger.info('job created', {
  jobId,
  queue: 'thumbnail',
  status: 'queued'
});
```

좋은 로그는 장애를 찾을 만큼 충분하지만, 데이터 유출 시 피해를 키우지 않습니다.
UUID v7 자체에서 생성 시간을 대략 추정할 수 있으므로, 필요 이상으로 사용자 이메일, 토큰, 원문 payload를 같은 로그에 묶지 않는 편이 좋습니다.

### 테스트에서는 형식과 정렬 기대를 분리한다

테스트에서 UUID v7을 다룰 때는 두 가지를 나눠 검증해야 합니다.
하나는 형식이 UUID처럼 생겼는지, 다른 하나는 애플리케이션이 ID 정렬에 과도하게 의존하지 않는지입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { createSortableId } from '../src/ids.js';

test('createSortableId returns a UUID-like value', () => {
  const id = createSortableId();

  assert.match(
    id,
    /^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/
  );
});
```

반대로 "나중에 만든 ID가 항상 더 크다"는 테스트는 피하는 편이 좋습니다.
동일 밀리초, 시스템 clock, 런타임 구현 세부사항 때문에 깨질 수 있습니다.
정렬이 필요한 도메인 로직은 `created_at`과 보조 키를 함께 넣은 샘플 데이터로 검증하세요.

## UUID v4, v7, 숫자 시퀀스 선택 기준

### 공개 리소스 ID라면 노출 특성을 먼저 본다

공개 URL에 들어가는 ID는 충돌 가능성뿐 아니라 추측 가능성과 정보 노출을 함께 봐야 합니다.
숫자 시퀀스는 가장 읽기 쉽지만, 생성량과 순서를 쉽게 추정할 수 있습니다.
UUID v4는 시간 정보가 없어 노출 면에서 단순합니다.
UUID v7은 정렬성이 좋지만 생성 시점이 일부 드러납니다.

선택 기준은 아래처럼 잡을 수 있습니다.

- 내부 테이블 기본 키: UUID v7 또는 데이터베이스 시퀀스
- 공개 URL 리소스 ID: UUID v4 또는 별도 slug
- 이벤트 저장소 ID: UUID v7
- 회계상 순번: 데이터베이스 시퀀스나 별도 번호 발급기
- 외부 파트너 연동 ID: 계약에서 요구하는 형식 우선

정답은 하나가 아닙니다.
한 서비스 안에서도 내부 기본 키는 UUID v7, 사용자에게 보이는 공개 ID는 UUID v4를 쓸 수 있습니다.

### 마이그레이션은 새 테이블부터 시작한다

이미 운영 중인 테이블의 기본 키를 UUID v4에서 v7로 바꾸는 일은 비용이 큽니다.
외래 키, 캐시 키, 검색 인덱스, URL, 데이터 분석 쿼리까지 영향을 받습니다.
성능 문제가 측정된 것이 아니라면 기존 ID를 유지하고, 새 테이블이나 새 이벤트 저장소부터 UUID v7을 적용하는 편이 안전합니다.

마이그레이션이 필요하다면 아래 순서로 접근하세요.

1. 새 `sortable_id` 컬럼을 추가한다
2. 신규 데이터부터 값을 채운다
3. 백필 작업으로 기존 데이터를 채운다
4. 읽기 경로에서 새 ID를 보조적으로 사용한다
5. 충분히 검증한 뒤 외부 계약과 기본 키 변경을 별도로 진행한다

기본 키 변경은 "컬럼 하나 바꾸기"가 아니라 시스템 식별자 계약 변경입니다.
작은 단계로 나누고 롤백 경로를 남기는 편이 좋습니다.

## 발행 전 체크리스트

### 코드와 운영 기준을 함께 확인한다

`crypto.randomUUIDv7()`을 도입하기 전에 아래 항목을 점검하세요.

- 배포 Node.js 버전이 `randomUUIDv7()`을 지원하는가
- ID가 외부에 노출될 때 생성 시점 노출이 문제 없는가
- `created_at` 같은 도메인 시간 컬럼을 별도로 유지하는가
- 단조 증가나 회계상 연속 번호를 기대하고 있지 않은가
- 테스트가 런타임 구현 세부사항에 과하게 의존하지 않는가
- 로그에 ID와 민감정보가 함께 남지 않는가

이 기준을 통과하면 UUID v7은 Node.js 백엔드에서 꽤 실용적인 기본 키 후보가 됩니다.
특히 이벤트, 작업 큐, 최근순 조회가 많은 데이터에서는 UUID v4보다 운영 감각이 좋습니다.

## 함께 읽기

- [Node.js crypto.argon2 비밀번호 해싱 가이드](/development/blog/seo/2026/08/22/nodejs-crypto-argon2-password-hashing-guide.html)
- [Node.js Idempotency-Key 중복 요청 방지 가이드](/development/blog/seo/2026/04/10/nodejs-idempotency-key-duplicate-request-prevention-guide.html)
- [Node.js 커서 페이지네이션 일관성 가이드](/development/blog/seo/2026/03/22/nodejs-cursor-pagination-infinite-scroll-consistency-guide.html)

## 참고 자료

- [Node.js Crypto 공식 문서](https://nodejs.org/api/crypto.html)

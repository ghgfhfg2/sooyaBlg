---
layout: post
title: "Node.js 낙관적 락 실전 가이드: ETag와 If-Match로 동시 수정 충돌 줄이기"
date: 2026-03-21 20:00:00 +0900
lang: ko
translation_key: nodejs-etag-if-match-optimistic-locking-concurrency-guide
permalink: /development/blog/seo/2026/03/21/nodejs-etag-if-match-optimistic-locking-concurrency-guide.html
alternates:
  ko: /development/blog/seo/2026/03/21/nodejs-etag-if-match-optimistic-locking-concurrency-guide.html
  x_default: /development/blog/seo/2026/03/21/nodejs-etag-if-match-optimistic-locking-concurrency-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, etag, if-match, optimistic-locking, concurrency, api]
description: "Node.js API에서 동시 수정으로 데이터가 덮어써지는 문제를 줄이기 위해 ETag, If-Match, 버전 필드 기반 낙관적 락을 어떻게 설계하고 운영할지 실무 기준으로 정리했습니다."
---

같은 문서를 두 사람이 거의 동시에 수정했는데, 나중에 저장한 사람의 내용만 남고 먼저 저장한 변경이 조용히 사라지는 경우가 있습니다.
이건 UI 문제가 아니라 **서버가 동시 수정 충돌을 감지하지 못한 것**에 가깝습니다.
이 글에서는 Node.js API에서 **낙관적 락(optimistic locking)** 을 어떻게 적용하면 좋은지, `ETag`와 `If-Match` 헤더를 중심으로 동시 업데이트 충돌을 감지하고 안전하게 처리하는 방법을 실무 기준으로 정리합니다.

## 왜 동시 수정 충돌이 문제인가

### H3. 마지막 저장이 무조건 이기는 구조는 조용히 데이터를 망가뜨린다

초기 CRUD API는 보통 단순합니다.
클라이언트가 `PUT /posts/123` 으로 전체 데이터를 보내면 서버는 그대로 덮어씁니다.
이 흐름은 구현은 쉽지만, 동시성 상황에서는 아래 문제가 생깁니다.

- A 사용자가 문서를 열어 수정한다
- B 사용자도 같은 시점의 문서를 열어 수정한다
- A가 먼저 저장한다
- B가 나중에 저장하면서 A의 변경을 모른 채 덮어쓴다

장애처럼 크게 보이지 않아도, 운영에서는 이런 조용한 데이터 손실이 더 위험할 때가 많습니다.
로그에는 정상 요청 두 건만 남고, 사용자는 "분명 저장했는데 왜 내용이 사라졌지?"를 겪게 됩니다.

### H3. 특히 관리자 화면과 협업 기능에서 자주 터진다

동시 수정 충돌은 아래 같은 서비스에서 흔합니다.

- 관리자 CMS에서 공지·상품·배너를 편집할 때
- 고객 정보나 주문 메모를 여러 운영자가 수정할 때
- 문서, 노트, 설정 화면처럼 사람이 내용을 편집하는 기능
- 모바일과 웹이 같은 자원을 서로 다른 시점에 갱신할 때

읽기보다 쓰기가 적더라도, 한번 충돌이 나면 신뢰를 크게 떨어뜨립니다.
그래서 락을 아주 무겁게 걸지 않더라도 **업데이트 전 버전 확인** 정도는 두는 편이 안전합니다.

## 낙관적 락이란 무엇인가

### H3. 충돌이 드물다고 가정하고, 저장 순간에만 검증한다

낙관적 락은 "항상 충돌할 것"이라고 보고 자원을 미리 잠그는 방식이 아닙니다.
대신 **대부분은 충돌하지 않지만, 저장 직전에 버전이 바뀌었는지 확인하자**는 접근입니다.

개념은 단순합니다.

1. 클라이언트가 자원을 조회한다
2. 서버는 현재 버전을 함께 내려준다
3. 클라이언트는 수정 후 저장할 때 그 버전을 다시 보낸다
4. 서버는 현재 버전과 비교해 같을 때만 업데이트한다

같지 않다면 서버는 업데이트를 거절하고, 클라이언트는 최신 상태를 다시 보여주거나 충돌 해결 UI를 띄울 수 있습니다.

### H3. 비관적 락보다 운영 부담이 낮다

데이터베이스 row lock 같은 비관적 락은 특정 상황에서 필요하지만, 웹 API 전체에 광범위하게 적용하면 비용이 큽니다.
반면 낙관적 락은 아래 장점이 있습니다.

- HTTP API와 잘 맞는다
- 장시간 편집 화면에 적용하기 쉽다
- DB 락 유지 시간이 길어지지 않는다
- 충돌 감지 책임이 명확하다

즉, 협업 편집기가 아니더라도 일반 백오피스 API에서 충분히 현실적인 기본값이 됩니다.

## HTTP에서 ETag와 If-Match를 쓰는 이유

### H3. ETag는 현재 표현의 버전 표식으로 이해하면 쉽다

`ETag`는 원래 HTTP 캐시 맥락에서 자주 보이지만, 동시 수정 제어에도 잘 맞습니다.
서버가 자원을 응답할 때 현재 버전을 식별할 수 있는 값을 `ETag`로 내려주고, 클라이언트는 수정 요청 시 `If-Match` 헤더에 그 값을 넣어 보냅니다.

예를 들면 아래와 같습니다.

```http
GET /api/posts/123 HTTP/1.1
```

```http
HTTP/1.1 200 OK
ETag: "post-123:v17"
Content-Type: application/json

{
  "id": "123",
  "title": "동시성 제어",
  "content": "초안 내용",
  "version": 17
}
```

이후 저장 요청은 이렇게 갑니다.

```http
PUT /api/posts/123 HTTP/1.1
If-Match: "post-123:v17"
Content-Type: application/json

{
  "title": "동시성 제어",
  "content": "수정된 내용"
}
```

서버의 현재 버전이 아직 `v17`이면 저장하고, 이미 다른 요청이 `v18`로 바꿨다면 충돌로 처리합니다.

### H3. 412 Precondition Failed를 쓰면 의미가 분명해진다

`If-Match`가 맞지 않을 때는 보통 `412 Precondition Failed`를 반환합니다.
이 상태 코드는 "업데이트 요청 형식은 맞지만, 전제 조건이 더 이상 성립하지 않는다"는 의미라서 동시 수정 충돌과 잘 맞습니다.

예시 응답은 아래처럼 구성할 수 있습니다.

```json
{
  "code": "PRECONDITION_FAILED",
  "message": "리소스가 다른 사용자에 의해 먼저 수정되었습니다.",
  "resourceId": "123"
}
```

실무에서는 `409 Conflict`를 쓰는 팀도 있지만, `If-Match` 기반 전제 조건 실패라면 `412`가 더 직관적인 경우가 많습니다.
중요한 것은 팀 내에서 기준을 통일하는 것입니다.

## Node.js에서 구현하는 기본 패턴

### H3. DB 버전 필드를 ETag로 노출한다

가장 단순한 방법은 테이블에 `version` 컬럼이나 `updatedAt` 기반 버전 값을 두고, 이를 ETag로 매핑하는 것입니다.
예를 들면 다음 흐름입니다.

```ts
function toEtag(post: { id: string; version: number }) {
  return `"post-${post.id}:v${post.version}"`;
}
```

조회 시:

```ts
app.get('/api/posts/:id', async (req, res) => {
  const post = await postRepository.findById(req.params.id);

  if (!post) {
    return res.status(404).json({
      code: 'NOT_FOUND',
      message: '게시글을 찾을 수 없습니다.',
    });
  }

  res.setHeader('ETag', toEtag(post));

  return res.json({
    id: post.id,
    title: post.title,
    content: post.content,
    version: post.version,
  });
});
```

저장 시:

```ts
app.put('/api/posts/:id', async (req, res) => {
  const ifMatch = req.header('If-Match');
  const current = await postRepository.findById(req.params.id);

  if (!current) {
    return res.status(404).json({
      code: 'NOT_FOUND',
      message: '게시글을 찾을 수 없습니다.',
    });
  }

  const currentEtag = toEtag(current);

  if (!ifMatch || ifMatch !== currentEtag) {
    return res.status(412).json({
      code: 'PRECONDITION_FAILED',
      message: '다른 변경사항이 먼저 반영되었습니다. 최신 데이터를 다시 확인해주세요.',
    });
  }

  const updated = await postRepository.updateWithVersion(req.params.id, {
    title: req.body.title,
    content: req.body.content,
    expectedVersion: current.version,
  });

  res.setHeader('ETag', toEtag(updated));

  return res.json({
    id: updated.id,
    title: updated.title,
    content: updated.content,
    version: updated.version,
  });
});
```

핵심은 `If-Match` 확인만으로 끝내지 말고, 실제 DB 업데이트도 **기대 버전(expectedVersion)** 을 조건으로 수행하는 것입니다.
그래야 애플리케이션 레벨과 저장소 레벨이 함께 안전해집니다.

### H3. SQL 업데이트 자체를 조건부로 만들어야 진짜로 막힌다

아래처럼 `WHERE id = ? AND version = ?` 조건으로 업데이트해야 경쟁 상태(race condition)를 줄일 수 있습니다.

```sql
UPDATE posts
SET title = ?, content = ?, version = version + 1
WHERE id = ? AND version = ?;
```

영향받은 row 수가 0이면, 누군가 먼저 수정했다는 뜻입니다.
이 신호를 받아 `412` 또는 팀 규약에 맞는 충돌 응답을 반환하면 됩니다.

즉, 낙관적 락은 헤더 처리만의 문제가 아니라 **DB 쓰기 조건까지 포함한 패턴**이어야 합니다.

## ETag 설계에서 자주 헷갈리는 포인트

### H3. 해시 기반 ETag와 버전 기반 ETag 중 무엇이 나을까

두 방식 모두 가능하지만 실무에서는 버전 기반이 운영하기 쉬운 편입니다.

- **버전 기반 ETag**: `v17`, `updatedAt`, revision number 등으로 계산
- **해시 기반 ETag**: 응답 body 전체를 hash 해서 생성

버전 기반은 계산이 단순하고 DB 조건과 직접 연결하기 쉽습니다.
해시 기반은 표현 그 자체를 반영한다는 장점이 있지만, 필드 순서나 직렬화 정책, 불필요한 계산 비용을 함께 고민해야 합니다.

사람이 편집하는 자원이라면 보통 버전 컬럼 기반이 더 단순하고 예측 가능합니다.

### H3. Weak ETag보다 strong 비교 기준이 안전하다

캐시 최적화 문맥에서는 weak ETag가 쓰일 수 있지만, 동시 수정 제어에서는 strong 비교가 더 안전합니다.
낙관적 락 목적으로는 "이 버전이 정확히 같을 때만 저장"이 중요하기 때문입니다.

즉, 편집 API에서는 모호한 동등성보다 **정확한 버전 일치**가 우선입니다.

## 운영에서 놓치기 쉬운 예외 처리

### H3. PATCH와 부분 수정도 같은 방식으로 보호해야 한다

일부 팀은 `PUT`만 충돌 제어를 넣고 `PATCH`는 자유롭게 두는데, 실제로는 부분 수정도 덮어쓰기 문제를 만들 수 있습니다.
예를 들어 제목만 바꾼다고 생각했지만, 서버 내부 병합 로직 때문에 다른 필드 상태에 영향을 줄 수 있습니다.

그래서 아래 원칙이 좋습니다.

- 같은 자원을 바꾸는 쓰기 요청은 `PUT`, `PATCH`, `DELETE` 모두 버전 체크 대상에 포함
- 관리자 화면과 공개 API의 규칙을 가능하면 통일
- 문서에 `If-Match` 필수 여부를 명시

### H3. 클라이언트 UX까지 설계해야 실제로 덜 아프다

서버가 `412`를 잘 준다고 끝나지 않습니다.
사용자 입장에서는 왜 저장이 실패했는지, 다음에 무엇을 해야 하는지가 중요합니다.

실무에서는 아래 대응이 자주 쓰입니다.

- 최신 데이터를 다시 불러오고 차이를 보여준다
- 내가 입력한 초안은 로컬에 임시 보존한다
- 충돌 발생 시 "다른 사용자가 먼저 수정했습니다"를 분명하게 안내한다
- 자동 재시도 대신 사용자 확인을 거친다

동시 수정 문제는 기술 이슈이면서 동시에 UX 이슈입니다.
충돌을 감지하는 것만큼 **충돌 후 복구 경험**도 중요합니다.

## 언제 낙관적 락이 특히 잘 맞나

### H3. 사람이 편집하는 자원, 관리자 화면, 설정 변경 API

아래 같은 경우라면 도입 효과가 큽니다.

- 관리자 페이지의 게시글, 상품, 정책 설정
- 사용자 프로필, 주소, 결제 수단 메타데이터 수정
- 내부 운영 도구에서 여러 사람이 동시에 만지는 리소스
- 외부 파트너가 API를 통해 같은 객체를 갱신하는 경우

반대로 고빈도 카운터 증가처럼 "마지막 값만 맞으면 되는" 업데이트는 별도 원자 연산 전략이 더 맞을 수 있습니다.
중요한 것은 모든 쓰기에 같은 규칙을 강제하는 게 아니라, **충돌 시 손실 비용이 큰 자원**부터 적용하는 것입니다.

## 배포 전 체크리스트

### H3. API 계약과 구현 점검

- 조회 응답에 현재 버전을 표현하는 `ETag` 또는 동등한 버전 필드가 있는가
- 쓰기 요청에 `If-Match` 또는 기대 버전을 요구하는가
- DB 업데이트가 `WHERE version = ?` 같은 조건부 쓰기로 구현됐는가
- 충돌 시 상태 코드와 에러 포맷이 일관적인가
- OpenAPI 문서에 `412 Precondition Failed` 응답과 헤더 요구사항이 반영됐는가

### H3. 보안·운영 점검

- ETag 값에 내부 인프라 정보나 민감 식별자가 들어가지 않는가
- 코드 예시에 실제 토큰, 내부 도메인, 개인정보가 포함되지 않았는가
- 자동 재시도로 중복 저장을 유발하지 않는가
- 관리자 화면에서 충돌 발생 시 사용자에게 다음 행동이 명확히 안내되는가

## 요약

Node.js API에서 동시 수정 충돌은 흔하지만, 그대로 두면 조용한 데이터 손실로 이어지기 쉽습니다.
`ETag`와 `If-Match`를 이용한 낙관적 락은 구현 난이도 대비 효과가 큰 편이고, 특히 관리자 화면·협업 기능·설정 변경 API에서 유용합니다.
핵심은 헤더만 흉내 내는 것이 아니라 **버전 기반 조회, 조건부 업데이트, 일관된 충돌 응답, 클라이언트 UX**까지 함께 설계하는 것입니다.

## 내부 링크

- [Node.js OpenAPI 계약 검증 가이드: Zod와 Swagger로 API 불일치 줄이기](/development/blog/seo/2026/03/21/nodejs-openapi-zod-contract-validation-api-consistency-guide.html)
- [API 버저닝 실전 가이드: Node.js에서 하위 호환성 지키며 배포하기](/development/blog/seo/2026/03/16/api-versioning-nodejs-backward-compatibility-guide.html)
- [Circuit Breaker 패턴 실전 가이드: Node.js 외부 API 장애 전파 줄이기](/development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html)

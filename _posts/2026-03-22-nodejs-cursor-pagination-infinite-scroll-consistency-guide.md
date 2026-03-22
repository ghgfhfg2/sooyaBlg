---
layout: post
title: "Node.js 커서 페이지네이션 가이드: Offset 대신 Cursor로 무한 스크롤 일관성 지키기"
date: 2026-03-22 20:00:00 +0900
lang: ko
translation_key: nodejs-cursor-pagination-infinite-scroll-consistency-guide
permalink: /development/blog/seo/2026/03/22/nodejs-cursor-pagination-infinite-scroll-consistency-guide.html
alternates:
  ko: /development/blog/seo/2026/03/22/nodejs-cursor-pagination-infinite-scroll-consistency-guide.html
  x_default: /development/blog/seo/2026/03/22/nodejs-cursor-pagination-infinite-scroll-consistency-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, pagination, cursor-pagination, infinite-scroll, api, backend]
description: "Node.js API에서 offset pagination의 한계와 cursor pagination의 장점을 비교하고, 무한 스크롤과 대용량 목록 조회에서 일관성을 높이는 구현 기준을 정리했습니다."
---

목록 API를 처음 만들 때는 보통 `page=1&limit=20` 같은 offset 방식부터 붙입니다.
처음엔 단순하고 익숙해서 좋아 보이지만, 데이터가 계속 추가되거나 삭제되는 서비스에서는 곧 문제가 드러납니다.
이 글에서는 Node.js API에서 **offset pagination** 이 왜 흔들리기 쉬운지, 그리고 **cursor pagination** 으로 무한 스크롤과 대용량 목록 조회의 일관성을 어떻게 높일 수 있는지 실무 기준으로 정리합니다.

## 왜 offset pagination이 자주 문제를 만들까

### H3. 데이터가 중간에 바뀌면 중복 조회와 누락이 생기기 쉽다

offset 방식은 보통 아래처럼 동작합니다.

- 1페이지: 0개 건너뛰고 20개 조회
- 2페이지: 20개 건너뛰고 20개 조회
- 3페이지: 40개 건너뛰고 20개 조회

문제는 사용자가 1페이지를 본 뒤 2페이지를 요청하는 사이에 새 글이 앞쪽에 끼어들 수 있다는 점입니다.
그러면 원래 2페이지에 있어야 할 항목이 밀리거나 당겨지면서 아래 같은 문제가 생깁니다.

- 방금 본 항목이 다음 페이지에 또 나온다
- 봐야 할 항목 하나가 통째로 건너뛰어진다
- 무한 스크롤에서 사용자가 "왜 같은 게 또 나오지?"를 겪는다

정적인 데이터셋에서는 괜찮아 보여도, 실서비스의 피드·로그·주문 내역·알림 목록처럼 계속 변하는 데이터에서는 offset이 쉽게 흔들립니다.

### H3. 페이지가 뒤로 갈수록 성능 비용도 커질 수 있다

offset 방식의 또 다른 문제는 성능입니다.
예를 들어 `OFFSET 100000 LIMIT 20` 같은 쿼리는 최종적으로 20개만 돌려주더라도, 앞의 많은 행을 건너뛰는 비용이 생길 수 있습니다.
DB 엔진과 인덱스 상태에 따라 차이는 있지만, 대체로 아래 상황에서 부담이 커집니다.

- 오래된 로그나 피드의 깊은 페이지 조회
- 관리자 화면에서 대량 데이터 탐색
- 정렬 조건이 복잡한 목록 API
- 모바일 무한 스크롤에서 연속 호출이 잦은 경우

즉 offset은 구현 난도는 낮지만, **데이터 변화에 약하고 깊은 페이지에서 비싸질 수 있는 방식**입니다.

## cursor pagination은 무엇이 다른가

### H3. 위치 번호가 아니라 마지막으로 본 기준점을 넘긴다

cursor pagination은 "몇 번째 페이지인가"보다 **마지막으로 본 항목의 기준점이 무엇인가**를 넘기는 방식입니다.
예를 들어 최신순 목록이라면 `createdAt DESC, id DESC` 같은 정렬 기준을 두고, 마지막 항목의 값을 cursor로 사용합니다.

흐름은 대략 이렇습니다.

1. 첫 요청은 cursor 없이 최신 20개를 조회한다
2. 응답에는 목록과 함께 `nextCursor`를 내려준다
3. 다음 요청은 `?cursor=...` 를 보내 그 기준 이후 항목을 조회한다

이 방식의 핵심은 **앞에서 몇 개를 건너뛸지 세는 게 아니라, 어디서부터 이어서 볼지를 지정하는 것**입니다.
그래서 중간에 새 데이터가 들어와도 이미 본 구간이 비교적 덜 흔들립니다.

### H3. 무한 스크롤과 이벤트성 데이터에 특히 잘 맞는다

cursor pagination은 아래 같은 목록에 특히 잘 맞습니다.

- SNS 피드, 댓글, 알림 목록
- 주문 내역, 활동 로그, 감사 로그
- 채팅 메시지, 타임라인, 이벤트 스트림 근접 목록
- 관리자 백오피스의 최신순 목록

반대로 "정확히 123페이지로 바로 이동" 같은 요구가 강하면 offset이 더 직관적일 수 있습니다.
즉 cursor가 모든 경우의 정답은 아니지만, **연속 탐색과 데이터 변화 대응**에서는 대개 더 유리합니다.

## Node.js API에서 cursor를 설계하는 기본 원칙

### H3. 정렬 기준은 안정적이고 유일해야 한다

cursor pagination에서 가장 중요한 건 정렬 기준입니다.
정렬 기준이 흔들리면 cursor도 의미가 없어집니다.
실무에서는 보통 아래처럼 갑니다.

- 1차 정렬: `created_at DESC`
- 2차 정렬: `id DESC`

`created_at`만 쓰면 같은 시각에 생성된 행들의 순서가 불안정할 수 있습니다.
그래서 보통은 **동점 해소용 tie-breaker** 로 `id` 같은 유일 키를 함께 둡니다.

정리하면 좋은 기준은 이렇습니다.

- 정렬이 예측 가능해야 한다
- 같은 조건에서 결과 순서가 안정적이어야 한다
- 인덱스로 뒷받침 가능해야 한다
- cursor에 필요한 값이 응답에서 복원 가능해야 한다

### H3. cursor는 노출용 토큰으로 감싸는 편이 안전하다

cursor를 그냥 `createdAt`과 `id` 문자열 그대로 노출해도 동작은 합니다.
하지만 실무에서는 보통 Base64 같은 방식으로 묶어서 토큰처럼 다룹니다.

예를 들면 아래 같은 payload를 인코딩할 수 있습니다.

```json
{
  "createdAt": "2026-03-22T10:12:30.000Z",
  "id": 98123
}
```

이렇게 하면 API 계약이 조금 더 깔끔해지고, 클라이언트는 cursor 내부 구조를 굳이 알 필요가 없습니다.
물론 이것이 보안 기능은 아닙니다.
민감정보를 넣으면 안 되고, 단지 **계약 안정성과 파라미터 관리 편의성**을 위한 포장으로 보는 편이 맞습니다.

## SQL 조건은 어떻게 잡을까

### H3. 정렬 순서와 where 조건이 정확히 맞아야 한다

최신순(`created_at DESC, id DESC`) 목록이라면 다음 페이지는 보통 "마지막 항목보다 더 오래된 것"을 가져와야 합니다.
예를 들어 마지막 항목이 `(created_at='2026-03-22 10:12:30', id=98123)` 이라면 조건은 보통 아래처럼 갑니다.

```sql
SELECT id, title, created_at
FROM posts
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 21;
```

또는 DB별 제약에 따라 아래처럼 풀어서 쓸 수도 있습니다.

```sql
SELECT id, title, created_at
FROM posts
WHERE created_at < $1
   OR (created_at = $1 AND id < $2)
ORDER BY created_at DESC, id DESC
LIMIT 21;
```

핵심은 단순합니다.
`ORDER BY`와 `WHERE` 비교 기준이 어긋나면 중복이나 누락이 생깁니다.
cursor pagination은 개념보다 이 부분의 세부 정합성이 더 중요합니다.

### H3. hasNextPage 판단을 위해 하나 더 조회하는 패턴이 실용적이다

실무에서는 `LIMIT 20` 대신 `LIMIT 21`로 한 건 더 조회하는 패턴이 자주 쓰입니다.
이렇게 하면 다음 페이지 존재 여부를 쉽게 판단할 수 있습니다.

- 21건이 왔다면 `hasNextPage = true`
- 마지막 1건은 잘라내고 20건만 응답
- 잘라낸 직전 항목 기준으로 `nextCursor` 생성

이 방식은 별도 count 쿼리 없이도 무한 스크롤에 필요한 정보를 주기 좋아서 많이 씁니다.
전체 개수까지 꼭 필요한 화면이 아니라면, cursor 방식에서 `COUNT(*)`를 매번 같이 구하는 건 오히려 비쌀 수 있습니다.

## Node.js 예시 구현

### H3. Express 기준으로 단순한 cursor 인코딩/디코딩 유틸을 둔다

아래는 개념을 설명하기 위한 단순 예시입니다.

```ts
function encodeCursor(input: { createdAt: string; id: number }) {
  return Buffer.from(JSON.stringify(input)).toString('base64url');
}

function decodeCursor(cursor?: string) {
  if (!cursor) return null;

  const json = Buffer.from(cursor, 'base64url').toString('utf8');
  const parsed = JSON.parse(json);

  return {
    createdAt: parsed.createdAt,
    id: Number(parsed.id),
  };
}
```

목록 핸들러는 대략 아래처럼 갈 수 있습니다.

```ts
app.get('/api/posts', async (req, res) => {
  const limit = Math.min(Number(req.query.limit ?? 20), 50);
  const cursor = decodeCursor(String(req.query.cursor ?? ''));

  const rows = await postRepository.listByCursor({
    limit: limit + 1,
    cursor,
  });

  const hasNextPage = rows.length > limit;
  const items = hasNextPage ? rows.slice(0, limit) : rows;

  const lastItem = items[items.length - 1];

  return res.json({
    items,
    pageInfo: {
      hasNextPage,
      nextCursor:
        hasNextPage && lastItem
          ? encodeCursor({
              createdAt: lastItem.createdAt.toISOString(),
              id: lastItem.id,
            })
          : null,
    },
  });
});
```

이 예시의 핵심은 복잡한 라이브러리보다 **정렬 기준, limit 상한, cursor 검증, 다음 페이지 판단 방식**입니다.
이 네 가지가 맞아야 실제 운영에서 덜 흔들립니다.

## offset에서 cursor로 바꿀 때 자주 부딪히는 문제

### H3. 임의 페이지 점프 UX와는 잘 안 맞을 수 있다

cursor 방식은 "다음으로", "이어서 더 보기"에는 강하지만, "57페이지로 바로 가기"에는 불편합니다.
그래서 모든 목록을 한 번에 cursor로 바꾸기보다, 화면 목적에 따라 나누는 편이 현실적입니다.

- 무한 스크롤 피드: cursor 우선
- 검색 결과에서 특정 페이지 이동 필요: offset 유지 가능
- 관리자 목록: 최신순 탐색 중심이면 cursor 검토

즉 기술적으로 더 좋아 보여도, UX 요구와 어긋나면 오히려 불편해질 수 있습니다.

### H3. 정렬 필드가 자주 바뀌는 화면은 계약을 더 엄격히 잡아야 한다

예를 들어 사용자가 `latest`, `popular`, `comments` 같은 정렬을 바꿀 수 있다면, cursor는 정렬 방식마다 별개로 취급해야 합니다.
`latest`에서 받은 cursor를 `popular`에 재사용하면 당연히 의미가 어긋납니다.

그래서 아래 원칙이 좋습니다.

- cursor는 특정 정렬 조건에만 유효하다고 본다
- 정렬 방식이 바뀌면 cursor를 폐기한다
- 서버에서 허용되지 않는 조합은 400으로 명확히 거절한다

cursor는 만능 포인터가 아니라 **특정 쿼리 맥락에 종속된 위치 토큰**입니다.

## 운영에서 놓치기 쉬운 포인트

### H3. 삭제·삽입이 잦아도 "일관된 탐색"이 목표이지 "완전 고정 목록"은 아니다

cursor pagination이 offset보다 안정적이라고 해서, 목록 스냅샷을 영구히 얼리는 건 아닙니다.
새 글이 올라오고, 일부 항목이 삭제되면 사용자가 보는 전체 집합은 계속 바뀝니다.
중요한 것은 그런 변화 속에서도 다음 페이지 탐색이 최대한 자연스럽고 예측 가능하게 유지되는가입니다.

즉 목표는 다음과 같습니다.

- 중복을 줄인다
- 누락 가능성을 낮춘다
- 깊은 페이지 성능을 개선한다
- 무한 스크롤 경험을 덜 흔들리게 만든다

### H3. OpenAPI 문서와 클라이언트 계약을 함께 정리해야 한다

cursor pagination은 단순 쿼리 파라미터 하나 추가로 끝나지 않습니다.
클라이언트가 알아야 할 계약이 분명히 있습니다.

- `cursor`는 선택 파라미터인가
- `limit` 최대값은 얼마인가
- `pageInfo.hasNextPage`와 `nextCursor` 형식은 무엇인가
- 정렬 변경 시 cursor 재사용이 가능한가
- 잘못된 cursor는 어떤 에러로 반환하는가

문서화가 약하면 서버는 맞게 구현돼도, 앱·웹 클라이언트가 제각각 해석하면서 버그가 생깁니다.
실무에서는 코드보다 **계약 명시 부족**이 더 흔한 문제입니다.

## 배포 전 체크리스트

### H3. 구현 점검

- 정렬 기준이 안정적이고 유일 키로 보조되고 있는가
- `ORDER BY`와 cursor 조건이 정확히 일치하는가
- limit 상한이 있는가
- `hasNextPage`, `nextCursor` 계산 방식이 일관적인가
- 잘못된 cursor 입력에 대해 명확한 검증과 에러 응답이 있는가

### H3. 보안·운영·콘텐츠 점검

- cursor 토큰에 개인정보나 내부 식별 규칙을 과도하게 노출하지 않는가
- 코드 예시에 실제 토큰, 내부 URL, 민감정보가 포함되지 않았는가
- 무한 스크롤과 관리자 목록처럼 실제 사용 맥락이 분명히 설명됐는가
- 내부링크와 메타 설명, 태그가 일관되게 설정됐는가

## 요약

Node.js 목록 API에서 offset pagination은 구현은 쉽지만, 데이터가 자주 바뀌는 환경에서는 중복·누락·깊은 페이지 성능 문제를 만들기 쉽습니다.
cursor pagination은 마지막으로 본 기준점을 넘기는 방식이라 무한 스크롤과 최신순 목록에서 더 안정적으로 동작합니다.
핵심은 멋진 추상화보다 **안정적인 정렬 기준, 일치하는 SQL 조건, 명확한 API 계약, 제한된 limit 정책**을 함께 설계하는 것입니다.

## 내부 링크

- [Node.js OpenAPI 계약 검증 가이드: Zod와 Swagger로 API 불일치 줄이기](/development/blog/seo/2026/03/21/nodejs-openapi-zod-contract-validation-api-consistency-guide.html)
- [API 버저닝 실전 가이드: Node.js에서 하위 호환성 지키며 배포하기](/development/blog/seo/2026/03/16/api-versioning-nodejs-backward-compatibility-guide.html)
- [Node.js 캐시 스탬피드 방지 가이드: Redis와 Singleflight로 트래픽 폭주 줄이기](/development/blog/seo/2026/03/20/nodejs-cache-stampede-prevention-redis-singleflight-guide.html)

---
layout: post
title: "Node.js + Prisma 무중단 데이터베이스 마이그레이션 가이드: 운영 장애 없이 스키마 변경하기"
date: 2026-03-16 08:00:00 +0900
lang: ko
translation_key: zero-downtime-database-migration-nodejs-prisma-guide
permalink: /development/blog/seo/2026/03/16/zero-downtime-database-migration-nodejs-prisma-guide.html
alternates:
  ko: /development/blog/seo/2026/03/16/zero-downtime-database-migration-nodejs-prisma-guide.html
  x_default: /development/blog/seo/2026/03/16/zero-downtime-database-migration-nodejs-prisma-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, prisma, database-migration, zero-downtime, backend, reliability]
description: "Node.js와 Prisma 환경에서 무중단 데이터베이스 마이그레이션을 수행하는 실무 방법을 정리했습니다. Expand-Contract 패턴, 배포 순서, 롤백 전략, 점검 체크리스트까지 운영 관점으로 설명합니다."
---

서비스 트래픽이 있는 상태에서 DB 스키마를 바꿀 때 가장 무서운 건,
**마이그레이션 자체보다 애플리케이션 버전 불일치로 인한 장애**입니다.
이 글에서는 Node.js + Prisma 기준으로 무중단(Zero-Downtime) 마이그레이션을 수행하는
실전 패턴을 정리합니다.

## 왜 일반적인 "한 번에 변경" 방식이 위험한가

### 앱 버전이 동시에 바뀌지 않는다

실서비스는 롤링 배포를 하므로 구버전/신버전 앱이 잠시 공존합니다.
이때 컬럼을 삭제하거나 타입을 즉시 변경하면 구버전 코드가 바로 깨질 수 있습니다.

### DDL 잠금과 긴 트랜잭션이 지연을 만든다

DB 엔진에 따라 `ALTER TABLE`이 잠금을 유발할 수 있고,
트래픽 시간대에 실행하면 API 지연이나 타임아웃으로 이어집니다.

## 무중단 마이그레이션 핵심: Expand → Migrate → Contract

### 1) Expand: 먼저 "호환 가능한 확장"만 적용

초기 단계에서는 기존 코드가 그대로 동작해야 합니다.
예를 들어 `users.display_name`을 `users.nickname`으로 바꾸고 싶다면,
먼저 `nickname` 컬럼을 **추가만** 합니다(삭제/강제 변경 금지).

```sql
-- 1) 확장 단계: 새 컬럼 추가 (nullable)
ALTER TABLE users ADD COLUMN nickname TEXT;

-- 2) 조회 성능이 필요하면 인덱스 추가
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_nickname ON users (nickname);
```

### 2) Migrate: 애플리케이션을 이중 쓰기/이중 읽기로 전환

배포 1차에서는 쓰기 시 두 컬럼을 함께 채우고,
읽기 시 새 컬럼 우선 + 기존 컬럼 fallback을 사용합니다.

```ts
// Prisma + Node.js 예시
await prisma.user.update({
  where: { id: userId },
  data: {
    displayName: input.name,
    nickname: input.name,
  },
});

const profileName = user.nickname ?? user.displayName;
```

이 상태를 며칠 유지하면서 데이터 누락/예외 로그를 확인합니다.

### 3) Backfill: 과거 데이터를 배치로 천천히 메우기

기존 레코드가 많다면 한 번에 업데이트하지 말고,
배치 크기(예: 500~2000건) 단위로 처리해야 락과 I/O 급증을 줄일 수 있습니다.

```ts
const batchSize = 1000;
let cursor = 0;

while (true) {
  const users = await prisma.user.findMany({
    where: { nickname: null },
    orderBy: { id: 'asc' },
    take: batchSize,
    skip: cursor,
    select: { id: true, displayName: true },
  });

  if (!users.length) break;

  await prisma.$transaction(
    users.map((u) =>
      prisma.user.update({
        where: { id: u.id },
        data: { nickname: u.displayName },
      })
    )
  );

  cursor += users.length;
}
```

### 4) Contract: 충분한 검증 후에만 기존 컬럼 제거

관측 지표(에러율, p95 지연, 누락 건수)가 안정적일 때,
마지막 배포에서만 `display_name` 의존 코드를 제거하고 컬럼을 삭제합니다.

## Prisma 환경에서 실수 줄이는 배포 순서

### H3. 권장 배포 플로우

1. DB Expand 마이그레이션 반영 (`nickname` 추가)
2. 앱 v1 배포 (이중 쓰기/이중 읽기)
3. 백필 작업 실행 + 검증
4. 앱 v2 배포 (신규 컬럼만 사용)
5. DB Contract 마이그레이션 반영 (기존 컬럼 제거)

### H3. 롤백 전략을 먼저 준비

- 앱 롤백: v1로 즉시 되돌릴 수 있게 이미지 보관
- DB 롤백: Contract 전까지는 복구가 쉬움
- 운영 롤백 기준: 에러율 급증, 지연 임계치 초과, 백필 실패율 증가

## 운영 체크리스트 (발행 전/배포 전)

### H3. 변경 전

- 트래픽 피크 시간 회피 배포 계획 수립
- 쿼리/인덱스 영향도 사전 측정
- 민감정보가 로그에 남지 않도록 마스킹 정책 적용

### H3. 변경 중

- API 에러율, DB lock wait, p95 지연 모니터링
- 배치 백필 속도와 실패율 추적
- 타임아웃 설정(애플리케이션/DB 드라이버) 재확인

### H3. 변경 후

- 구버전 인스턴스 0대 확인 후 Contract 진행
- 불필요 인덱스/코드 제거
- 회고 문서화(다음 마이그레이션 템플릿으로 재사용)

## 요약

무중단 마이그레이션의 핵심은 기술 스택보다 **순서 관리**입니다.
"추가(Expand) → 전환/백필(Migrate) → 제거(Contract)" 원칙을 지키면,
Node.js + Prisma에서도 운영 장애 없이 안전하게 스키마를 진화시킬 수 있습니다.

## 내부 링크

- [/development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html](/development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html)
- [/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)
- [/development/blog/seo/2026/03/14/opentelemetry-nodejs-distributed-tracing-practical-guide.html](/development/blog/seo/2026/03/14/opentelemetry-nodejs-distributed-tracing-practical-guide.html)

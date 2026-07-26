---
layout: post
title: "Node.js sqlite changeset 가이드: 내장 SQLite로 변경분 동기화하기"
date: 2026-07-27 08:00:00 +0900
lang: ko
translation_key: nodejs-sqlite-changeset-sync-guide
permalink: /development/blog/seo/2026/07/27/nodejs-sqlite-changeset-sync-guide.html
alternates:
  ko: /development/blog/seo/2026/07/27/nodejs-sqlite-changeset-sync-guide.html
  x_default: /development/blog/seo/2026/07/27/nodejs-sqlite-changeset-sync-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, sqlite, changeset, database, sync, offline-first, backend, javascript]
description: "Node.js node:sqlite의 changeset과 applyChangeset으로 SQLite 변경분을 추적하고 다른 데이터베이스에 안전하게 반영하는 방법을 정리합니다. 세션 생성, 충돌 처리, 마이그레이션 기준, 백업 전략까지 실무 예제로 설명합니다."
---

작은 Node.js 서비스나 자동화 도구에서는 SQLite 하나로 충분한 경우가 많습니다.
로컬 캐시, 데스크톱 앱, 내부 운영 도구, 테스트 fixture, 오프라인 우선 기능처럼 중앙 데이터베이스까지는 부담스럽지만 파일 기반 저장소가 필요한 상황이 대표적입니다.
문제는 데이터가 한 곳에만 머물지 않을 때 시작됩니다.
로컬에서 만든 변경분을 다른 파일에 반영하거나, 임시 데이터베이스에서 검증한 결과만 실제 데이터베이스에 적용해야 하는 흐름이 생깁니다.

Node.js의 `node:sqlite` 모듈은 `DatabaseSync`와 함께 SQLite session 기반 changeset 기능을 제공합니다.
공식 문서 기준으로 `node:sqlite`는 release candidate 상태이며, `createSession()`, `session.changeset()`, `database.applyChangeset()`을 사용해 변경분을 바이너리 형태로 추출하고 다른 데이터베이스에 적용할 수 있습니다.
이 글에서는 Node.js에서 SQLite changeset을 언제 쓰면 좋은지, 기본 구현은 어떻게 잡는지, 충돌과 운영 리스크는 어떤 기준으로 다뤄야 하는지 정리합니다.
파일 처리와 배포 전 점검 자동화를 함께 다룬다면 [Node.js fs.promises.opendir 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html), 디스크 여유 공간 확인은 [Node.js fs.promises.statfs 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-statfs-disk-space-check-guide.html)를 함께 보면 좋습니다.

## SQLite changeset이 필요한 상황

### H3. 전체 파일 복사보다 변경분만 옮기고 싶을 때

SQLite 데이터베이스는 파일 하나로 다룰 수 있어서 단순 복사가 쉽습니다.
하지만 파일 전체를 매번 복사하는 방식은 데이터가 커지거나 동기화 주기가 짧아질수록 부담이 커집니다.
또 어느 시점의 변경만 반영하고 싶은데 파일 단위 복사만 있으면 적용 범위를 세밀하게 제어하기 어렵습니다.

changeset은 세션이 열린 뒤 발생한 변경을 별도 바이너리 데이터로 뽑아냅니다.
이 변경분을 다른 데이터베이스에 적용하면 원본과 대상 사이의 차이를 더 명시적으로 다룰 수 있습니다.

예를 들어 아래 같은 흐름에 잘 맞습니다.

- 로컬 작업 데이터베이스에서 검증된 변경만 실제 데이터베이스에 반영
- 테스트 중 만든 fixture 변경을 다른 in-memory 데이터베이스에 복제
- 오프라인 작업 결과를 서버 또는 동기화 파일에 전달
- 배치 작업이 만든 변경분을 적용 전에 로그와 함께 보관

changeset은 복잡한 분산 데이터베이스를 자동으로 만들어 주는 기능은 아닙니다.
대신 SQLite가 이미 제공하는 변경 추적 단위를 Node.js 코드에서 직접 다룰 수 있게 해 주는 도구에 가깝습니다.

### H3. 동기화 경계와 검증 단계를 분리하고 싶을 때

실무에서는 "바로 실제 DB에 쓰기"보다 한 번 더 안전한 경계를 두고 싶은 순간이 많습니다.
예를 들어 CSV import, 크롤링 결과 정제, 대량 설정 변경, 사용자 업로드 검증처럼 실패 가능성이 큰 작업은 먼저 임시 데이터베이스에서 처리한 뒤 결과만 반영하는 편이 안전합니다.

이때 changeset을 사용하면 아래처럼 단계를 나눌 수 있습니다.

1. 임시 데이터베이스에서 스키마와 입력값을 검증한다.
2. 세션을 열고 실제 변경 작업을 실행한다.
3. changeset을 추출한다.
4. 대상 데이터베이스에 적용하기 전에 백업과 충돌 정책을 확인한다.
5. 적용 결과를 로그로 남긴다.

이 구조는 장애 복구에도 유리합니다.
변경을 만든 코드, 변경분 파일, 적용 대상, 충돌 처리 결과를 분리해서 기록할 수 있기 때문입니다.

## 기본 사용법

### H3. createSession으로 변경 추적을 시작한다

`createSession()`은 데이터베이스 연결에 세션을 만들고, 이후 변경 내용을 추적할 수 있게 합니다.
세션은 너무 오래 열어 두기보다 하나의 작업 단위에 맞춰 열고 닫는 편이 좋습니다.

```js
import { DatabaseSync } from 'node:sqlite';

const sourceDb = new DatabaseSync(':memory:');
const targetDb = new DatabaseSync(':memory:');

const schema = `
  CREATE TABLE notes(
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    body TEXT NOT NULL
  ) STRICT
`;

sourceDb.exec(schema);
targetDb.exec(schema);

const session = sourceDb.createSession();

try {
  const insert = sourceDb.prepare(`
    INSERT INTO notes (id, title, body)
    VALUES (?, ?, ?)
  `);

  insert.run(1, 'release checklist', 'backup, migrate, verify');
  insert.run(2, 'incident note', 'record what changed');

  const changeset = session.changeset();
  targetDb.applyChangeset(changeset);
} finally {
  session.close();
  sourceDb.close();
  targetDb.close();
}
```

핵심은 세션을 만든 뒤 실행한 변경만 changeset에 들어간다는 점입니다.
따라서 스키마 생성이나 초기 데이터 적재까지 변경분에 포함할지, 실제 업무 데이터 변경만 포함할지 먼저 정해야 합니다.

### H3. 테이블 단위로 추적 범위를 좁힌다

작업 범위가 분명하다면 세션 생성 시 특정 테이블만 추적하도록 제한하는 편이 좋습니다.
동기화 대상이 아닌 로그 테이블, 임시 테이블, 캐시 테이블까지 changeset에 섞이면 적용 단계가 불필요하게 복잡해집니다.

```js
const session = sourceDb.createSession({
  table: 'notes'
});
```

테이블 범위를 좁히면 변경분이 작아지고, 적용 전 검토도 쉬워집니다.
특히 사용자 설정, 문서 초안, 작업 큐 상태처럼 일부 테이블만 이동해야 하는 구조라면 처음부터 changeset 경계를 작게 잡는 편이 안전합니다.

## 변경분 적용과 충돌 처리

### H3. applyChangeset은 대상 데이터베이스의 현재 상태와 충돌할 수 있다

changeset은 "이 변경을 적용하라"는 기록이지만, 대상 데이터베이스가 항상 같은 상태라는 보장은 없습니다.
대상에 이미 같은 기본 키가 있거나, update 대상 row가 사라졌거나, foreign key 제약을 깨는 변경이 들어올 수 있습니다.

그래서 `applyChangeset()`에는 충돌 처리 정책이 필요합니다.
Node.js는 SQLite changeset 충돌 상수를 제공하고, `onConflict` 콜백에서 충돌별 결정을 내릴 수 있습니다.
기본값은 충돌이 나면 적용을 중단하는 쪽에 가깝게 잡는 것이 운영상 안전합니다.

```js
import {
  DatabaseSync,
  SQLITE_CHANGESET_ABORT,
  SQLITE_CHANGESET_CONFLICT,
  SQLITE_CHANGESET_DATA,
  SQLITE_CHANGESET_NOTFOUND
} from 'node:sqlite';

function applySafely(targetDb, changeset) {
  return targetDb.applyChangeset(changeset, {
    onConflict(conflictType) {
      if (conflictType === SQLITE_CHANGESET_CONFLICT) {
        return SQLITE_CHANGESET_ABORT;
      }

      if (conflictType === SQLITE_CHANGESET_DATA) {
        return SQLITE_CHANGESET_ABORT;
      }

      if (conflictType === SQLITE_CHANGESET_NOTFOUND) {
        return SQLITE_CHANGESET_ABORT;
      }

      return SQLITE_CHANGESET_ABORT;
    }
  });
}
```

처음부터 자동 병합을 넓게 허용하면 데이터가 조용히 덮어써질 수 있습니다.
특히 사용자 입력, 결제 상태, 권한, 재고처럼 정합성이 중요한 데이터는 충돌 시 중단하고 사람이 원인을 확인하게 만드는 편이 낫습니다.

### H3. 자동 반영보다 사전 검증을 먼저 둔다

changeset 적용 전에는 대상 데이터베이스가 예상한 스키마인지 확인해야 합니다.
테이블은 같아 보여도 컬럼 제약, 기본 키, foreign key 설정, 트리거가 다르면 적용 결과가 달라질 수 있습니다.

간단한 내부 도구라면 최소한 아래 기준은 점검하는 편이 좋습니다.

- `PRAGMA foreign_keys = ON` 상태인지 확인
- 마이그레이션 버전 테이블이 기대값인지 확인
- 적용 전 대상 DB 백업 또는 스냅샷 생성
- changeset 생성 코드 버전과 적용 코드 버전 기록
- 충돌 발생 시 원본 changeset을 보존

Node.js에서 환경 변수를 함께 관리한다면 [Node.js loadEnvFile 가이드](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)처럼 데이터베이스 경로와 실행 모드를 분리해 두는 것도 도움이 됩니다.

## 운영 패턴

### H3. changeset 파일에는 민감정보가 들어갈 수 있다

changeset은 바이너리라서 사람이 바로 읽기 어렵지만, 안전한 파일이라는 뜻은 아닙니다.
변경된 row 값이 들어갈 수 있으므로 사용자 이메일, 토큰, 내부 설정, 원문 입력값이 포함될 수 있습니다.
따라서 changeset 파일은 로그 파일보다 데이터베이스 백업에 가까운 민감도로 다뤄야 합니다.

운영 기준은 아래처럼 잡을 수 있습니다.

- changeset 저장 디렉터리를 저장소에 커밋하지 않는다.
- 파일명에 사용자 식별자나 민감한 업무 정보를 넣지 않는다.
- 보관 기간을 짧게 두고 자동 삭제한다.
- 접근 권한을 배포 계정 또는 운영 계정으로 제한한다.
- 장애 분석용으로 공유할 때는 샘플 데이터로 재현한다.

로그와 마스킹 정책은 [Node.js 로그 샘플링 가이드](/development/blog/seo/2026/06/18/nodejs-log-sampling-redaction-observability-guide.html)와 같은 기준으로 함께 정리해 두면 좋습니다.

### H3. 적용 전 백업과 디스크 여유 공간을 확인한다

SQLite는 파일 기반이라 백업이 쉬운 편이지만, 그만큼 적용 전후 파일 상태를 명확히 남기는 것이 중요합니다.
changeset을 적용하기 전에는 대상 파일을 복사하거나 SQLite 백업 API를 사용해 되돌릴 수 있는 지점을 만들어야 합니다.
또 디스크 공간이 부족한 상태에서 변경을 적용하면 백업도, 적용도 애매하게 실패할 수 있습니다.

자동화 스크립트에서는 아래 순서를 권장합니다.

1. 대상 데이터베이스 경로를 확인한다.
2. 디스크 여유 공간을 확인한다.
3. 대상 데이터베이스를 백업한다.
4. changeset을 적용한다.
5. 검증 쿼리를 실행한다.
6. 성공 로그와 백업 보관 만료 시점을 남긴다.

디스크 공간 확인은 [Node.js fs.promises.statfs 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-statfs-disk-space-check-guide.html), 대량 파일 백업 구조는 [Node.js fs.cp 가이드](/development/blog/seo/2026/05/17/nodejs-fs-cp-recursive-copy-guide.html)와 연결해서 설계할 수 있습니다.

### H3. 장기 동기화 시스템이라면 버전과 소유권을 명시한다

changeset 기반 동기화를 반복 운영하려면 "누가 만든 변경인가"와 "어떤 버전에서 만든 변경인가"가 중요해집니다.
동일한 테이블을 여러 프로세스가 동시에 수정하는 구조라면 충돌은 예외가 아니라 정상 상황이 됩니다.

그래서 changeset 자체와 별도로 작은 메타데이터를 함께 저장하는 편이 좋습니다.

```json
{
  "changesetId": "2026-07-27T08-00-00Z-notes-import",
  "schemaVersion": 12,
  "source": "notes-import-worker",
  "createdAt": "2026-07-26T23:00:00.000Z",
  "tables": ["notes"],
  "conflictPolicy": "abort"
}
```

이 메타데이터는 충돌이 났을 때 원인을 좁히는 데 도움이 됩니다.
또 오래된 changeset을 최신 스키마에 잘못 적용하는 사고를 줄일 수 있습니다.

## 테스트 기준

### H3. 정상 적용보다 충돌 케이스를 먼저 테스트한다

changeset 코드는 정상 경로만 보면 간단해 보입니다.
하지만 실제 위험은 충돌, 중복 적용, 스키마 불일치, 일부 테이블 제외 같은 경계에서 생깁니다.
테스트는 아래 케이스부터 잡는 것이 좋습니다.

- 빈 대상 DB에 changeset을 적용하면 기대 row가 생기는가?
- 같은 changeset을 두 번 적용할 때 충돌이 명확히 발생하는가?
- 대상 row가 이미 다른 값으로 바뀐 경우 중단되는가?
- 추적 대상이 아닌 테이블 변경은 changeset에 포함되지 않는가?
- foreign key 위반이 발생하면 적용이 롤백되는가?

Node.js 내장 test runner를 쓴다면 in-memory 데이터베이스로 빠르게 검증할 수 있습니다.
테스트 실행 범위를 좁히는 방법은 [Node.js test runner name pattern 가이드](/development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html)를 참고하세요.

### H3. 임시 파일 테스트는 정리까지 확인한다

파일 기반 데이터베이스로 테스트할 때는 임시 디렉터리 정리가 중요합니다.
테스트가 끝난 뒤 SQLite 파일, WAL 파일, changeset 파일이 남으면 다음 테스트에 영향을 줄 수 있습니다.

테스트 helper는 아래처럼 정리 책임을 명확히 두는 편이 좋습니다.

```js
import { mkdtemp, rm } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

export async function withTempSqliteFile(fn) {
  const directory = await mkdtemp(join(tmpdir(), 'sqlite-sync-'));

  try {
    return await fn({
      dbPath: join(directory, 'app.db'),
      changesetPath: join(directory, 'sync.changeset')
    });
  } finally {
    await rm(directory, { recursive: true, force: true });
  }
}
```

테스트가 프로세스를 붙잡는다면 열린 데이터베이스 연결이나 파일 handle이 남아 있는지 먼저 확인해야 합니다.
이 관점은 [Node.js test runner force exit 가이드](/development/blog/seo/2026/07/15/nodejs-test-runner-force-exit-hanging-process-guide.html)와도 연결됩니다.

## 도입 체크리스트

Node.js SQLite changeset을 도입하기 전에는 아래 항목을 확인하세요.

- `node:sqlite`를 사용할 Node.js 버전과 배포 런타임이 맞는가?
- 변경 추적을 시작할 지점과 끝낼 지점이 명확한가?
- 적용 대상 데이터베이스의 스키마 버전을 검증하는가?
- 충돌 정책이 자동 덮어쓰기보다 보수적으로 설정돼 있는가?
- changeset 파일을 백업 데이터 수준으로 보호하는가?
- 적용 전 백업과 적용 후 검증 쿼리가 있는가?
- 중복 적용, 충돌, foreign key 실패 테스트가 있는가?

## 마무리

`node:sqlite`의 changeset 기능은 SQLite를 단순한 로컬 저장소에서 한 단계 더 실용적인 동기화 단위로 끌어올립니다.
전체 파일을 복사하지 않고 변경분을 추출하고, 대상 데이터베이스에 적용하며, 충돌 정책을 코드로 표현할 수 있기 때문입니다.

다만 changeset은 자동 병합 시스템이 아닙니다.
스키마 버전, 충돌 정책, 백업, 민감정보 보호, 테스트 기준이 함께 있어야 운영 가능한 동기화 구조가 됩니다.
작게 시작한다면 in-memory 데이터베이스 두 개로 정상 적용과 충돌 케이스를 먼저 테스트하고, 그다음 파일 백업과 검증 쿼리를 자동화하는 순서가 가장 안전합니다.

## 함께 보면 좋은 글

- [Node.js fs.promises.opendir 가이드: 대량 파일 디렉터리 순회를 안전하게 처리하는 법](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html)
- [Node.js fs.promises.statfs 가이드: 디스크 여유 공간을 배포 전에 확인하는 법](/development/blog/seo/2026/07/26/nodejs-fspromises-statfs-disk-space-check-guide.html)
- [Node.js 로그 샘플링 가이드: 운영 비용과 민감정보 리스크 줄이기](/development/blog/seo/2026/06/18/nodejs-log-sampling-redaction-observability-guide.html)
- [Node.js test runner name pattern 가이드: 테스트 이름으로 실행 범위를 좁히는 방법](/development/blog/seo/2026/07/05/nodejs-test-runner-name-pattern-skip-pattern-guide.html)

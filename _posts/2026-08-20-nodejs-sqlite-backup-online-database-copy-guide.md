---
layout: post
title: "Node.js sqlite.backup 가이드: 서비스 중 SQLite 데이터베이스를 안전하게 복사하는 법"
date: 2026-08-20 08:00:00 +0900
lang: ko
translation_key: nodejs-sqlite-backup-online-database-copy-guide
permalink: /development/blog/seo/2026/08/20/nodejs-sqlite-backup-online-database-copy-guide.html
alternates:
  ko: /development/blog/seo/2026/08/20/nodejs-sqlite-backup-online-database-copy-guide.html
  x_default: /development/blog/seo/2026/08/20/nodejs-sqlite-backup-online-database-copy-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, sqlite, backup, databasesync, database-backup, node-sqlite, durability, backend, javascript]
description: "Node.js node:sqlite의 sqlite.backup()으로 실행 중인 SQLite 데이터베이스를 안전하게 백업하는 방법을 정리합니다. DatabaseSync 연결, rate와 progress 옵션, 임시 파일 교체, 잠금과 민감정보 점검까지 실무 예제로 설명합니다."
---

SQLite를 파일 데이터베이스로 쓰는 서비스에서는 백업이 단순해 보입니다.
데이터베이스가 파일 하나라면 그냥 복사하면 될 것 같기 때문입니다.
하지만 실제 운영 중인 데이터베이스 파일을 일반 파일 복사로 다루면, 쓰기 도중의 상태나 WAL 파일 조합을 잘못 가져가 깨진 백업을 만들 수 있습니다.

Node.js의 `node:sqlite` 모듈은 이 문제를 다루기 위해 `sqlite.backup()` API를 제공합니다.
공식 문서 기준으로 `backup(sourceDb, path, options)`는 열린 `DatabaseSync` 연결을 대상으로 SQLite 백업 절차를 수행하고, 완료되면 백업된 페이지 수를 담은 Promise를 반환합니다.
`rate`로 한 번에 복사할 페이지 수를 조절하고, `progress` 콜백으로 남은 페이지 수를 관찰할 수 있습니다.

이 글에서는 Node.js `sqlite.backup()`을 운영 백업 작업에 어떻게 붙일지, 임시 파일과 원자적 교체를 어떻게 조합할지, 백업 로그에 민감정보가 섞이지 않게 하려면 무엇을 봐야 하는지 정리합니다.
기본 SQLite 연결은 [Node.js 내장 SQLite 가이드](/development/blog/seo/2026/05/29/nodejs-sqlite-built-in-database-guide.html), 변경 동기화 흐름은 [Node.js SQLite changeset 가이드](/development/blog/seo/2026/07/27/nodejs-sqlite-changeset-sync-guide.html), prepared statement 캐시는 [Node.js sqlTagStore 가이드](/development/blog/seo/2026/08/11/nodejs-sqlite-sqltagstore-prepared-statement-cache-guide.html)와 함께 보면 좋습니다.

## sqlite.backup이 필요한 이유

### 파일 복사는 운영 데이터베이스에 충분하지 않을 수 있다

SQLite 데이터베이스는 파일 기반이라 배포와 로컬 개발이 편합니다.
하지만 실행 중인 데이터베이스를 단순히 `cp app.db backup.db`로 복사하는 방식은 백업 전략으로 부족할 수 있습니다.
서비스가 쓰기 트랜잭션을 처리하는 중이라면 백업 파일이 일관된 시점의 데이터인지 보장하기 어렵고, 저널 모드나 WAL 파일 운영 방식에 따라 필요한 파일 조합도 달라집니다.

`sqlite.backup()`은 SQLite의 백업 메커니즘을 Node.js API로 감싼 함수입니다.
애플리케이션이 사용하는 열린 데이터베이스 연결을 넘기면, SQLite가 데이터베이스 페이지를 읽어 대상 파일로 복사합니다.

```js
import { backup, DatabaseSync } from 'node:sqlite';

const db = new DatabaseSync('./data/app.db', {
  timeout: 5000
});

const pages = await backup(db, './backups/app.db');

console.log('backup completed', { pages });
```

이 코드는 가장 작은 형태의 백업입니다.
실무에서는 대상 파일명, 임시 파일, 진행 로그, 실패 시 정리까지 함께 설계해야 합니다.

### 백업은 애플리케이션 작업과 분리해서 실행한다

`backup()`은 Promise를 반환하지만, 원본 데이터베이스 객체는 `DatabaseSync`입니다.
즉 쿼리 API 자체는 동기식이고, 같은 프로세스 안에서 무거운 작업을 섞으면 응답 지연을 만들 수 있습니다.
웹 요청 처리 경로에서 바로 백업을 수행하기보다 별도 관리 작업, 예약 작업, 운영 CLI에서 실행하는 편이 안전합니다.

```js
import { backup, DatabaseSync } from 'node:sqlite';

export async function runBackupJob({ sourcePath, targetPath, logger }) {
  const db = new DatabaseSync(sourcePath, {
    readOnly: true,
    timeout: 5000
  });

  try {
    const pages = await backup(db, targetPath);
    logger.info({ pages, target: targetPath }, 'sqlite backup completed');
    return { pages };
  } finally {
    db.close();
  }
}
```

읽기 전용 연결로 백업을 수행하면 백업 작업이 실수로 데이터를 수정할 가능성을 줄일 수 있습니다.
다만 운영 애플리케이션이 같은 파일을 쓰는 상황에서는 잠금과 타임아웃 정책을 함께 봐야 합니다.

## 안전한 백업 파일 만들기

### 임시 파일에 먼저 쓰고 완료 후 교체한다

백업 대상 경로에 바로 쓰면 실패한 백업이 정상 백업 파일을 덮어쓸 수 있습니다.
더 안전한 방식은 같은 디렉터리에 임시 파일을 만들고, 백업이 성공했을 때만 최종 파일명으로 바꾸는 것입니다.

```js
import { backup, DatabaseSync } from 'node:sqlite';
import { mkdir, rename, rm } from 'node:fs/promises';
import { dirname, join } from 'node:path';

export async function createSqliteBackup(sourcePath, backupDir) {
  await mkdir(backupDir, { recursive: true });

  const timestamp = new Date().toISOString().replaceAll(':', '-');
  const finalPath = join(backupDir, `app-${timestamp}.db`);
  const tempPath = `${finalPath}.tmp`;
  const db = new DatabaseSync(sourcePath, {
    readOnly: true,
    timeout: 5000
  });

  try {
    const pages = await backup(db, tempPath, {
      rate: 100
    });

    await rename(tempPath, finalPath);

    return {
      path: finalPath,
      pages
    };
  } catch (error) {
    await rm(tempPath, { force: true });
    throw error;
  } finally {
    db.close();
  }
}
```

`rename()`은 같은 파일 시스템 안에서는 원자적 교체에 가깝게 동작합니다.
그래서 임시 파일과 최종 파일은 같은 디렉터리에 두는 편이 좋습니다.
백업 파일을 다른 디스크나 오브젝트 스토리지로 옮기는 작업은 이 단계가 끝난 뒤 별도 업로드 작업으로 분리합니다.

### 백업 파일명에는 기준 시간을 남긴다

백업 파일명은 사람이 정렬하고 조사하기 쉬워야 합니다.
`latest.db` 하나만 유지하면 사고 시 어느 시점의 백업인지 판단하기 어렵습니다.
반대로 날짜와 시간을 포함한 파일명을 남기면 복구 후보를 빠르게 고를 수 있습니다.

```js
function buildBackupName(prefix, now = new Date()) {
  const stamp = now.toISOString()
    .replaceAll(':', '-')
    .replace(/\.\d{3}Z$/, 'Z');

  return `${prefix}-${stamp}.db`;
}
```

운영 지역 시간이 중요하다면 파일명에는 UTC를 쓰고, 로그에는 서비스 지역 시간을 별도 필드로 남기는 방식도 좋습니다.
중요한 것은 "언제 만든 백업인지"와 "어떤 코드가 만든 백업인지"를 나중에 추적할 수 있게 하는 것입니다.

## rate와 progress 옵션 사용하기

### rate로 한 번에 복사할 페이지 수를 조절한다

`backup()`의 `rate` 옵션은 백업 단계마다 전송할 페이지 수를 의미합니다.
기본값은 한 번에 100페이지입니다.
값을 작게 잡으면 진행 상황을 더 자주 관찰할 수 있지만, 콜백 호출이 늘고 전체 시간이 길어질 수 있습니다.
값을 크게 잡으면 단순한 작업에서는 빠르지만, 큰 데이터베이스에서 한 번의 단계가 길어질 수 있습니다.

```js
const pages = await backup(db, tempPath, {
  rate: 50,
  progress({ remainingPages, totalPages }) {
    logger.info({
      remainingPages,
      totalPages
    }, 'sqlite backup progress');
  }
});
```

처음에는 기본값으로 시작하고, 실제 데이터베이스 크기와 백업 시간이 확인된 뒤 조정하는 편이 안전합니다.
진행 로그가 너무 많이 찍히면 로그 비용이 늘어나므로 일정 간격으로 샘플링하는 구조를 추가할 수 있습니다.

### progress 로그에는 경로와 페이지 수만 남긴다

백업 진행 로그는 운영자가 상태를 보는 데 유용합니다.
하지만 로그에 SQL, 레코드 값, 사용자 식별자, 파일 시스템의 절대 경로 전체를 남길 필요는 없습니다.
백업 작업 식별자와 페이지 수 정도면 충분한 경우가 많습니다.

```js
function createProgressLogger(logger, jobId) {
  let lastLoggedPercent = -1;

  return ({ remainingPages, totalPages }) => {
    if (!totalPages) return;

    const copiedPages = totalPages - remainingPages;
    const percent = Math.floor((copiedPages / totalPages) * 100);

    if (percent < lastLoggedPercent + 10 && remainingPages !== 0) {
      return;
    }

    lastLoggedPercent = percent;

    logger.info({
      jobId,
      percent,
      remainingPages,
      totalPages
    }, 'sqlite backup progress');
  };
}
```

10% 단위처럼 적당히 샘플링하면 큰 데이터베이스에서도 로그가 과하게 늘지 않습니다.
운영 알림은 매 진행률이 아니라 실패, 장시간 지연, 성공 완료 이벤트에 집중하는 편이 좋습니다.

## 잠금과 실패 처리 기준

### timeout을 명시해 잠금 대기를 제한한다

SQLite는 파일 기반 데이터베이스이므로 잠금 상태를 고려해야 합니다.
`DatabaseSync`를 만들 때 `timeout` 옵션을 주면 데이터베이스 잠금이 풀리기를 기다릴 최대 시간을 정할 수 있습니다.
백업 작업이 영원히 기다리지 않게 하려면 이 값을 운영 기준으로 정해야 합니다.

```js
const db = new DatabaseSync('./data/app.db', {
  readOnly: true,
  timeout: 10_000
});
```

짧은 백업이라면 몇 초 정도로 충분할 수 있고, 쓰기가 많은 서비스에서는 더 긴 값이나 재시도 정책이 필요할 수 있습니다.
다만 timeout을 길게 잡는다고 백업 전략이 안정해지는 것은 아닙니다.
백업이 자주 잠금에 막힌다면 쓰기 부하가 낮은 시간대, 별도 복제 파일, 작업 큐 제한을 검토해야 합니다.

### 실패한 백업은 성공으로 취급하지 않는다

백업 작업에서 가장 위험한 실패는 실패했는데도 성공처럼 보이는 상태입니다.
임시 파일이 남아 있거나, 빈 파일이 최종 파일명으로 바뀌었거나, 업로드만 성공하고 SQLite 백업은 실패한 경우가 여기에 속합니다.

```js
export async function backupWithResult(options) {
  try {
    const result = await createSqliteBackup(options.sourcePath, options.backupDir);

    return {
      ok: true,
      ...result
    };
  } catch (error) {
    options.logger.error({
      code: error?.code,
      name: error?.name,
      message: error?.message
    }, 'sqlite backup failed');

    return {
      ok: false,
      reason: 'backup_failed'
    };
  }
}
```

에러 객체를 그대로 직렬화하면 내부 경로와 쿼리 정보가 예상보다 많이 나올 수 있습니다.
운영 로그에는 `code`, `name`, 제한된 `message`, 작업 ID 정도를 남기고, 민감한 설정값은 별도 redaction 규칙을 적용합니다.

## 복구 가능성까지 확인하기

### 백업 직후 무결성 검사를 실행한다

백업 파일을 만들었다면 최소한 열 수 있는지 확인해야 합니다.
가능하면 별도 `DatabaseSync` 연결로 백업 파일을 열고 간단한 무결성 검사를 실행합니다.

```js
import { DatabaseSync } from 'node:sqlite';

export function verifyBackupFile(path) {
  const db = new DatabaseSync(path, {
    readOnly: true,
    timeout: 5000
  });

  try {
    const row = db.prepare('PRAGMA quick_check').get();
    const value = Object.values(row)[0];

    if (value !== 'ok') {
      throw new Error('SQLite backup quick_check failed');
    }

    return true;
  } finally {
    db.close();
  }
}
```

`quick_check`는 완전한 복구 리허설을 대신하지는 않습니다.
그래도 백업 파일을 열 수 있는지, SQLite가 즉시 문제를 감지하는지 확인하는 작은 안전장치가 됩니다.
중요한 서비스라면 정기적으로 별도 환경에 복원해 주요 쿼리를 실행하는 훈련까지 포함해야 합니다.

### 보관 정책을 코드로 고정한다

백업은 만들기만 하면 끝이 아닙니다.
보관 기간, 삭제 기준, 암호화 여부, 접근 권한이 함께 정해져야 합니다.
개발용 로컬 DB 백업과 운영 고객 데이터 백업은 전혀 다른 기준을 가져야 합니다.

```js
import { readdir, rm } from 'node:fs/promises';
import { join } from 'node:path';

export async function pruneOldBackups(backupDir, keepCount = 14) {
  const entries = await readdir(backupDir, { withFileTypes: true });
  const files = entries
    .filter((entry) => entry.isFile() && entry.name.endsWith('.db'))
    .map((entry) => entry.name)
    .sort()
    .reverse();

  const staleFiles = files.slice(keepCount);

  await Promise.all(
    staleFiles.map((name) => rm(join(backupDir, name), { force: true }))
  );
}
```

이 예시는 최신 14개만 남기는 단순 정책입니다.
실제 운영에서는 일별, 주별, 월별 보관 정책을 분리하고, 삭제 로그와 접근 권한 점검을 함께 남기는 편이 좋습니다.

## 운영 체크리스트

### 발행 전 코드 점검

- 백업은 요청 처리 경로가 아니라 별도 작업에서 실행하는가?
- 원본 연결은 `readOnly`와 `timeout` 기준을 명시하는가?
- 임시 파일에 먼저 쓰고 성공 후 최종 파일명으로 바꾸는가?
- 실패 시 임시 파일을 정리하고 성공 알림을 보내지 않는가?
- 백업 파일을 다시 열어 최소 검사를 수행하는가?
- 로그에 데이터 값, 토큰, 사용자 개인정보, 절대 경로 전체가 남지 않는가?
- 보관 기간과 삭제 정책이 문서 또는 코드로 고정되어 있는가?

### 작은 서비스부터 자동화한다

처음부터 복잡한 백업 플랫폼을 만들 필요는 없습니다.
작은 SQLite 기반 서비스라면 하루 한 번 `sqlite.backup()`을 실행하고, 성공 파일을 검증한 뒤, 보관 개수만 제한해도 수동 파일 복사보다 훨씬 나은 출발점이 됩니다.
그다음 단계에서 오브젝트 스토리지 업로드, 암호화, 복원 리허설, 실패 알림을 차례로 붙이면 됩니다.

## FAQ

### sqlite.backup은 실행 중인 데이터베이스에도 사용할 수 있나요?

네. `backup()`은 열린 `DatabaseSync` 원본을 대상으로 백업을 수행합니다.
다만 같은 데이터베이스에 쓰기 부하가 높으면 잠금 대기나 재시작 비용이 생길 수 있으므로, `timeout`, 작업 시간대, 재시도 정책을 함께 설계해야 합니다.

### 백업 파일을 만들면 복구가 보장되나요?

아니요. 백업 파일 생성은 복구 가능성의 일부일 뿐입니다.
최소한 백업 파일을 다시 열어 `PRAGMA quick_check` 같은 검사를 수행하고, 중요한 서비스는 정기적으로 별도 환경에서 복원 리허설을 해야 합니다.

### 백업 로그에는 무엇을 남기는 게 좋나요?

작업 ID, 페이지 수, 진행률, 결과, 제한된 오류 코드 정도면 충분합니다.
SQL 결과값, 사용자 데이터, 토큰, 세션, 운영 절대 경로는 로그에 남기지 않는 편이 안전합니다.

## 마무리

`sqlite.backup()`은 Node.js에서 SQLite 파일을 더 안전하게 복사하기 위한 실용적인 기본 도구입니다.
단순 파일 복사 대신 열린 `DatabaseSync` 연결을 기준으로 백업을 만들고, 임시 파일 교체, 진행 로그, 잠금 timeout, 무결성 확인을 함께 붙이면 작은 서비스에도 충분히 믿을 만한 백업 루프를 만들 수 있습니다.

핵심은 백업을 "파일 하나 생성"으로 보지 않는 것입니다.
성공 판정, 검증, 보관, 삭제, 복원 리허설까지 이어져야 실제 장애 상황에서 쓸 수 있는 백업이 됩니다.

## 관련 글

- [Node.js 내장 SQLite 가이드: node:sqlite로 가벼운 로컬 DB 시작하기](/development/blog/seo/2026/05/29/nodejs-sqlite-built-in-database-guide.html)
- [Node.js SQLite changeset 가이드: 로컬 데이터 변경을 안전하게 동기화하는 법](/development/blog/seo/2026/07/27/nodejs-sqlite-changeset-sync-guide.html)
- [Node.js sqlTagStore 가이드: prepared statement 캐시로 SQLite 쿼리 관리하기](/development/blog/seo/2026/08/11/nodejs-sqlite-sqltagstore-prepared-statement-cache-guide.html)
- [Node.js 공식 문서: SQLite backup](https://nodejs.org/api/sqlite.html#sqlitebackupsource-db-path-options)

---
layout: post
title: "Node.js node:sqlite 가이드: 작은 도구와 테스트 DB를 내장 기능으로 다루는 법"
date: 2026-05-29 08:00:00 +0900
lang: ko
translation_key: nodejs-sqlite-built-in-database-guide
permalink: /development/blog/seo/2026/05/29/nodejs-sqlite-built-in-database-guide.html
alternates:
  ko: /development/blog/seo/2026/05/29/nodejs-sqlite-built-in-database-guide.html
  x_default: /development/blog/seo/2026/05/29/nodejs-sqlite-built-in-database-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, sqlite, database, testing, backend]
description: "Node.js의 node:sqlite 모듈로 작은 내부 도구와 테스트용 데이터베이스를 단순하게 다루는 방법을 예제와 함께 정리했습니다. 도입 기준, prepared statement, 테스트 격리, 운영 주의점을 살펴봅니다."
---

작은 Node.js 도구를 만들 때마다 데이터 저장소 선택은 생각보다 번거롭습니다.
JSON 파일은 금방 한계가 오고, 별도 데이터베이스 서버는 준비 비용이 큽니다.
테스트 코드에서도 매번 외부 DB를 띄우면 실행 속도와 격리 문제가 따라옵니다.

이럴 때 `node:sqlite`는 한 번 검토할 만한 선택지입니다.
핵심은 **간단한 로컬 데이터 저장과 테스트용 DB를 Node.js 기본 모듈 흐름 안에서 다룰 수 있다**는 점입니다.

## Node.js node:sqlite가 잘 맞는 상황

### H3. 작은 내부 도구에는 서버형 DB보다 단순한 저장소가 나을 때가 있다

모든 데이터 저장이 PostgreSQL이나 MySQL까지 필요하진 않습니다.
아래 같은 경우에는 파일 기반 SQLite가 더 실용적일 수 있습니다.

- 개인 자동화 도구의 작업 이력 저장
- 크론 작업의 마지막 실행 상태 기록
- CLI 도구의 로컬 캐시
- 테스트에서 빠르게 만들고 버리는 임시 DB
- 작은 관리자 도구의 설정 테이블

이런 작업은 네트워크 연결, 계정, 마이그레이션 서버를 먼저 준비하기보다 파일 하나로 시작하는 편이 가볍습니다.
특히 `:memory:` 데이터베이스를 쓰면 테스트마다 깨끗한 저장소를 만들 수 있어 반복 실행이 쉬워집니다.

### H3. 내장 모듈을 쓰면 의존성과 설치 변수가 줄어든다

SQLite 패키지를 따로 설치하는 방식도 충분히 좋지만, 네이티브 확장 빌드나 런타임 환경 차이가 부담이 되는 경우가 있습니다.
`node:sqlite`를 검토하는 이유는 이 지점을 줄이는 데 있습니다.

다만 모든 프로젝트에서 바로 기존 DB 레이어를 바꿔야 한다는 뜻은 아닙니다.
이미 ORM, 마이그레이션, 커넥션 관리, 관측성이 잘 잡힌 서비스라면 기존 구조를 유지하는 편이 낫습니다.
`node:sqlite`는 **작고 독립적인 저장 문제를 단순하게 풀 때** 가장 빛납니다.

## 기본 사용법

### H3. DatabaseSync로 메모리 DB를 빠르게 만들 수 있다

가장 단순한 예시는 메모리 DB를 열고 테이블을 만든 뒤 값을 읽는 흐름입니다.

```js
import { DatabaseSync } from 'node:sqlite';

const db = new DatabaseSync(':memory:');

db.exec(`
  CREATE TABLE tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    done INTEGER NOT NULL DEFAULT 0
  ) STRICT
`);

const insertTask = db.prepare(`
  INSERT INTO tasks (title)
  VALUES (?)
`);

insertTask.run('write daily report');

const selectTasks = db.prepare(`
  SELECT id, title, done
  FROM tasks
  ORDER BY id DESC
`);

console.log(selectTasks.all());

db.close();
```

이 예제의 장점은 외부 서버나 별도 설정 없이도 데이터 흐름을 확인할 수 있다는 점입니다.
테스트 러너와 함께 쓰는 작은 검증 코드라면 [Node.js test runner 가이드: 내장 테스트 러너로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)과도 잘 맞습니다.

### H3. prepared statement를 기본값으로 두는 편이 안전하다

SQL을 문자열로 이어 붙이면 입력값 처리에서 실수하기 쉽습니다.
사용자 입력, 파일명, URL 파라미터처럼 외부에서 들어온 값은 prepared statement에 바인딩하는 습관을 들이는 편이 좋습니다.

```js
function findTaskByTitle(db, title) {
  const statement = db.prepare(`
    SELECT id, title, done
    FROM tasks
    WHERE title = ?
    LIMIT 1
  `);

  return statement.get(title);
}
```

이 구조는 코드 리뷰에서도 의도가 명확합니다.
SQL 구조와 입력값이 분리되어 있어, 나중에 컬럼이 늘어나도 위험한 문자열 조합이 퍼질 가능성이 줄어듭니다.

## 테스트 DB로 활용하는 패턴

### H3. 테스트마다 새 메모리 DB를 만들면 격리가 쉽다

테스트 실패 원인 중 하나는 이전 테스트가 남긴 데이터입니다.
`node:sqlite`의 메모리 DB는 테스트마다 새 인스턴스를 만들고 닫는 구조를 잡기 좋습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { DatabaseSync } from 'node:sqlite';

function createTestDb() {
  const db = new DatabaseSync(':memory:');

  db.exec(`
    CREATE TABLE tasks (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      done INTEGER NOT NULL DEFAULT 0
    ) STRICT
  `);

  return db;
}

test('inserts a task', () => {
  const db = createTestDb();

  try {
    db.prepare('INSERT INTO tasks (title) VALUES (?)').run('ship post');
    const task = db.prepare('SELECT title FROM tasks LIMIT 1').get();

    assert.equal(task.title, 'ship post');
  } finally {
    db.close();
  }
});
```

리소스 정리를 더 구조적으로 다루고 싶다면 [Node.js using 가이드: Disposable과 AsyncDisposable로 정리 누락을 줄이는 법](/development/blog/seo/2026/05/02/nodejs-using-disposable-and-asyncdisposable-guide.html)도 함께 볼 만합니다.

### H3. 파일 DB는 임시 디렉터리와 함께 쓰면 흔적을 줄일 수 있다

실제 파일 DB 동작을 확인해야 하는 테스트도 있습니다.
이때는 프로젝트 루트에 테스트 파일을 남기기보다 임시 디렉터리에 DB 파일을 만들고 정리하는 편이 안전합니다.

임시 디렉터리 정리 패턴은 [Node.js mkdtempDisposable 가이드: 임시 디렉터리 정리를 안전하게 처리하는 법](/development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html)처럼 별도 유틸리티로 묶어 두면 재사용하기 좋습니다.

## 운영에서 주의할 점

### H3. 동기 API는 작은 작업에 적합하지만 요청 경로에서는 신중해야 한다

`DatabaseSync`는 이름 그대로 동기 API입니다.
CLI, 배치, 테스트처럼 실행 흐름이 짧고 단순한 곳에서는 오히려 읽기 쉽습니다.
하지만 고트래픽 HTTP 요청 경로에서 무거운 쿼리를 동기 실행하면 이벤트 루프를 막을 수 있습니다.

운영 서버에서 쓰려면 아래 기준을 먼저 정해야 합니다.

1. 쿼리 시간이 짧고 데이터 규모가 예측 가능한가?
2. 요청 경로가 아니라 초기화, 캐시, 내부 도구에 가까운가?
3. DB 파일 위치와 백업 정책이 명확한가?
4. 장애 시 재생성 가능한 데이터인가, 보존해야 하는 데이터인가?
5. 동시 쓰기와 잠금 상황을 테스트했는가?

이벤트 루프 영향이 걱정된다면 [Node.js scheduler.yield 가이드: 이벤트 루프 응답성을 지키는 법](/development/blog/seo/2026/05/02/nodejs-scheduler-yield-event-loop-responsiveness-guide.html)처럼 실행 흐름을 함께 점검하는 편이 좋습니다.

### H3. SQL 로그에는 개인정보와 토큰을 남기지 않는다

DB 예제에서 자주 놓치는 부분이 로그입니다.
디버깅을 위해 입력값을 그대로 출력하다 보면 이메일, 토큰, 내부 식별자 같은 정보가 남을 수 있습니다.

안전한 기본값은 아래에 가깝습니다.

- SQL 구조와 실행 시간은 남기되 바인딩 값은 최소화한다.
- 필요한 경우 값 전체가 아니라 길이, 타입, 마스킹된 일부만 기록한다.
- 테스트 fixture에도 실제 사용자 정보나 운영 토큰을 넣지 않는다.
- DB 파일을 저장소에 커밋하지 않도록 `.gitignore`를 확인한다.

개발 블로그 예제 역시 실제 운영 데이터가 아니라 재현 가능한 더미 값을 쓰는 편이 안전합니다.

## 실무 도입 체크리스트

### H3. 먼저 작은 경계부터 적용한다

`node:sqlite`를 도입할 때는 아래 순서가 현실적입니다.

1. CLI, 크론, 테스트처럼 독립적인 영역을 먼저 고른다.
2. 스키마 생성 코드를 한 파일에 모은다.
3. 모든 입력값은 prepared statement로 바인딩한다.
4. 테스트마다 새 DB를 만들거나 초기화 순서를 명확히 한다.
5. DB 파일 저장 위치, 백업, 로그 마스킹 기준을 문서화한다.

이렇게 시작하면 기존 서비스 구조를 크게 흔들지 않고도 SQLite의 장점을 확인할 수 있습니다.

## 마무리

`node:sqlite`는 대형 서비스의 DB 레이어를 한 번에 대체하는 도구라기보다, **작은 저장 문제를 빠르고 단정하게 해결하는 기본 도구**에 가깝습니다.
로컬 도구, 테스트, 크론 상태 저장처럼 범위가 명확한 곳에서는 의존성을 줄이고 실행 환경을 단순하게 만드는 효과가 큽니다.

중요한 건 “SQLite를 쓸 수 있는가”보다 **어디까지 SQLite로 책임지고, 어디서부터 별도 DB 전략이 필요한지 선을 긋는 것**입니다.
작게 시작하고, prepared statement와 정리 규칙을 기본값으로 두면 오래 유지되는 코드가 됩니다.

## 함께 보면 좋은 글

- [Node.js test runner 가이드: 내장 테스트 러너로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)
- [Node.js using 가이드: Disposable과 AsyncDisposable로 정리 누락을 줄이는 법](/development/blog/seo/2026/05/02/nodejs-using-disposable-and-asyncdisposable-guide.html)
- [Node.js mkdtempDisposable 가이드: 임시 디렉터리 정리를 안전하게 처리하는 법](/development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html)

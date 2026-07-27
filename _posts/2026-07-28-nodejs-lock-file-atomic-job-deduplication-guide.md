---
layout: post
title: "Node.js lock file 가이드: 중복 실행을 원자적으로 막는 작업 잠금 설계"
date: 2026-07-28 08:00:00 +0900
lang: ko
translation_key: nodejs-lock-file-atomic-job-deduplication-guide
permalink: /development/blog/seo/2026/07/28/nodejs-lock-file-atomic-job-deduplication-guide.html
alternates:
  ko: /development/blog/seo/2026/07/28/nodejs-lock-file-atomic-job-deduplication-guide.html
  x_default: /development/blog/seo/2026/07/28/nodejs-lock-file-atomic-job-deduplication-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, lock-file, atomic, cron, automation, concurrency, reliability, javascript]
description: "Node.js에서 lock file로 크론, 배치, 파일 처리 작업의 중복 실행을 원자적으로 막는 방법을 정리합니다. fs.open의 wx 플래그, stale lock 처리, finally 정리, CI 점검 기준까지 실무 예제로 설명합니다."
---

크론이나 배치 작업은 한 번만 실행된다고 믿기 쉽습니다.
하지만 실제 운영에서는 이전 실행이 오래 걸리는 동안 다음 스케줄이 시작되거나, 배포 직후 같은 작업이 두 프로세스에서 동시에 뜨거나, 수동 재실행과 자동 실행이 겹치는 일이 생깁니다.
이때 작업이 같은 파일을 쓰거나 같은 외부 API를 호출하거나 같은 데이터베이스 row를 갱신하면 결과가 두 번 반영될 수 있습니다.

Node.js에서 이런 중복 실행을 줄이는 가장 단순한 방법 중 하나가 lock file입니다.
락 파일은 "이 작업은 지금 누군가 실행 중이다"라는 표시를 파일 시스템에 남기고, 다른 프로세스가 같은 파일을 만들려고 할 때 실패하게 만드는 방식입니다.
핵심은 파일 존재 여부를 먼저 확인한 뒤 만드는 것이 아니라, `fs.open()`의 exclusive 생성 플래그인 `wx`로 원자적으로 선점하는 것입니다.
작업 취소와 정리 흐름을 함께 설계한다면 [Node.js child_process AbortSignal 가이드](/development/blog/seo/2026/07/25/nodejs-child-process-spawn-abortsignal-timeout-guide.html), 파일 시스템 자동화의 탐색 범위는 [Node.js fs.promises.opendir 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html)를 함께 보면 좋습니다.

## lock file이 필요한 상황

### H3. 중복 실행은 조용히 데이터 품질을 망친다

중복 실행은 항상 에러로 보이지 않습니다.
같은 작업이 두 번 돌았는데 둘 다 성공 로그를 남기면 오히려 더 위험합니다.
예를 들어 RSS 생성, 사이트맵 갱신, 통계 집계, 알림 발송, 파일 업로드 후처리처럼 부작용이 있는 작업은 한 번 더 실행되는 순간 결과가 달라질 수 있습니다.

대표적인 위험은 아래와 같습니다.

- 같은 포스트나 리포트 파일을 두 프로세스가 동시에 쓴다.
- 외부 API에 같은 요청을 두 번 보내 사용량과 비용이 늘어난다.
- 큐 작업이 중복으로 완료 처리되어 원인을 추적하기 어렵다.
- 임시 파일 정리 작업이 다른 실행의 작업 디렉터리를 지운다.
- 배치 결과가 마지막으로 끝난 프로세스 기준으로 덮어써진다.

작업이 완전히 멱등적으로 설계되어 있다면 중복 실행의 피해가 작을 수 있습니다.
하지만 모든 부작용을 완벽하게 멱등화하기는 어렵습니다.
그래서 실행 진입점에서 한 번 더 잠금 경계를 두는 것이 실무적으로 유용합니다.

### H3. exists 확인 후 생성은 경쟁 조건이 있다

락 파일을 만들 때 가장 흔한 실수는 `access()`나 `existsSync()`로 파일 존재 여부를 확인한 뒤, 없으면 파일을 만드는 방식입니다.
이 구조에는 짧지만 중요한 틈이 있습니다.
두 프로세스가 거의 동시에 "파일이 없다"고 판단한 뒤 둘 다 파일을 만들 수 있기 때문입니다.

문제 흐름은 단순합니다.

1. 프로세스 A가 lock 파일이 없다고 확인한다.
2. 프로세스 B도 lock 파일이 없다고 확인한다.
3. 프로세스 A가 lock 파일을 만든다.
4. 프로세스 B도 lock 파일을 덮어쓰거나 새로 만든다.

락의 목적은 "확인과 생성"을 하나의 원자적 동작으로 묶는 것입니다.
Node.js에서는 `fs.promises.open(lockPath, 'wx')`를 사용하면 파일이 이미 있을 때 실패하고, 없을 때만 새 파일을 만들 수 있습니다.

## 기본 구현

### H3. wx 플래그로 lock file을 원자적으로 만든다

아래 예제는 작업 시작 시 lock 파일을 만들고, 작업이 끝나면 `finally`에서 제거합니다.
이미 lock 파일이 있으면 다른 실행이 진행 중이라고 보고 바로 종료합니다.

```js
import { open, rm } from 'node:fs/promises';
import process from 'node:process';

const lockPath = new URL('./tmp/report-job.lock', import.meta.url);

async function acquireLock() {
  try {
    const handle = await open(lockPath, 'wx');

    await handle.writeFile(JSON.stringify({
      pid: process.pid,
      startedAt: new Date().toISOString(),
      job: 'report-job'
    }));

    return handle;
  } catch (error) {
    if (error?.code === 'EEXIST') {
      return null;
    }

    throw error;
  }
}

async function main() {
  const lock = await acquireLock();

  if (!lock) {
    console.log('report job is already running');
    return;
  }

  try {
    await runReportJob();
  } finally {
    await lock.close();
    await rm(lockPath, { force: true });
  }
}

async function runReportJob() {
  // 실제 작업을 여기에 둔다.
}

await main();
```

`wx`는 write와 exclusive create를 함께 의미합니다.
파일이 이미 있으면 기존 파일을 열지 않고 실패하므로 lock 선점에 사용할 수 있습니다.
중요한 점은 실패를 정상적인 분기 중 하나로 다뤄야 한다는 것입니다.
중복 실행이 감지된 상황은 장애가 아니라 보호 장치가 작동한 상황일 수 있습니다.

### H3. lock 파일에는 추적용 메타데이터만 넣는다

락 파일은 문제를 조사할 때 도움이 되는 작은 메타데이터를 담을 수 있습니다.
하지만 민감정보를 넣어서는 안 됩니다.
작업 대상 사용자 ID, API 토큰, 원문 요청 payload, 내부 URL 같은 값이 lock 파일에 남으면 로그와 비슷한 유출 위험이 생깁니다.

권장 메타데이터는 아래 정도로 충분합니다.

- 프로세스 ID
- 작업 이름
- 시작 시각
- 배포 버전 또는 git commit
- 호스트명 또는 실행 환경 이름

예를 들어 아래처럼 파일 내용을 제한할 수 있습니다.

```json
{
  "pid": 4312,
  "job": "daily-content-build",
  "startedAt": "2026-07-27T23:00:00.000Z",
  "commit": "8f3a1c2"
}
```

민감한 값은 lock 파일이 아니라 권한이 통제된 로그 시스템이나 감사 기록에 별도로 남기는 편이 안전합니다.
로그 마스킹 기준은 [Node.js 로그 샘플링과 redaction 가이드](/development/blog/seo/2026/06/18/nodejs-log-sampling-redaction-observability-guide.html)와 같은 운영 규칙으로 함께 관리할 수 있습니다.

## stale lock 처리

### H3. 프로세스가 죽으면 lock 파일이 남을 수 있다

`finally` 정리는 정상 종료와 대부분의 예외 상황에서 잘 동작합니다.
하지만 프로세스가 강제 종료되거나, 머신이 재부팅되거나, 작업 컨테이너가 중간에 사라지면 lock 파일이 남을 수 있습니다.
이 오래된 lock을 stale lock이라고 부릅니다.

stale lock 처리가 없으면 다음 실행은 계속 "이미 실행 중"이라고 판단하고 아무 일도 하지 않습니다.
반대로 stale lock을 너무 쉽게 지우면 실제로 오래 걸리는 정상 작업을 중복 실행할 수 있습니다.
그래서 작업별 최대 실행 시간을 기준으로 보수적인 만료 시간을 정해야 합니다.

예를 들어 평소 2분 안에 끝나는 작업이라도 네트워크 지연과 재시도를 고려해 30분 이상 지난 lock만 stale로 처리할 수 있습니다.
데이터 마이그레이션처럼 오래 걸릴 수 있는 작업은 자동 삭제보다 사람이 확인하게 만드는 편이 낫습니다.

### H3. stale 기준은 작업 시간보다 길게 잡는다

아래 예제는 lock 파일의 수정 시각을 확인해 오래된 lock만 삭제한 뒤 다시 선점합니다.

```js
import { open, rm, stat } from 'node:fs/promises';

const staleAfterMs = 30 * 60 * 1000;

async function removeStaleLock(lockPath) {
  try {
    const info = await stat(lockPath);
    const ageMs = Date.now() - info.mtimeMs;

    if (ageMs > staleAfterMs) {
      await rm(lockPath, { force: true });
      return true;
    }

    return false;
  } catch (error) {
    if (error?.code === 'ENOENT') {
      return true;
    }

    throw error;
  }
}

async function acquireLockWithStaleCheck(lockPath) {
  try {
    return await open(lockPath, 'wx');
  } catch (error) {
    if (error?.code !== 'EEXIST') {
      throw error;
    }
  }

  const removed = await removeStaleLock(lockPath);

  if (!removed) {
    return null;
  }

  try {
    return await open(lockPath, 'wx');
  } catch (error) {
    if (error?.code === 'EEXIST') {
      return null;
    }

    throw error;
  }
}
```

stale lock을 삭제한 뒤에도 다시 `open(lockPath, 'wx')`를 호출하는 점이 중요합니다.
삭제와 재생성 사이에도 다른 프로세스가 lock을 선점할 수 있기 때문입니다.
락 파일을 지웠다는 사실만으로 작업을 시작하면 다시 경쟁 조건이 생깁니다.

## 운영 설계 기준

### H3. lock 디렉터리와 작업 디렉터리를 분리한다

락 파일은 작업 결과물과 같은 디렉터리에 두지 않는 편이 좋습니다.
작업 중간 산출물을 정리하는 코드가 lock 파일까지 지우면 보호 장치가 사라질 수 있습니다.
반대로 lock 디렉터리 정리 정책이 결과 파일을 건드려도 곤란합니다.

권장 구조는 아래처럼 분리하는 것입니다.

```text
project/
  tmp/
    locks/
      daily-build.lock
  output/
    reports/
  scripts/
    daily-build.mjs
```

`tmp/locks`는 저장소에 커밋하지 않고, 배포 환경에서는 실행 계정만 쓸 수 있게 제한합니다.
여러 작업이 같은 디렉터리를 공유한다면 lock 파일명에 작업 이름과 환경을 포함하되, 사용자 입력값을 그대로 넣지는 않습니다.

### H3. 단일 머신과 분산 환경을 구분한다

파일 기반 lock은 같은 파일 시스템을 보는 프로세스 사이에서 유용합니다.
단일 서버의 크론, 한 컨테이너 안의 작업, 로컬 자동화 스크립트에는 충분히 단순하고 효과적입니다.

하지만 여러 서버, 여러 컨테이너, 여러 리전에서 동시에 작업이 뜨는 구조라면 파일 lock만으로는 부족합니다.
각 인스턴스가 서로 다른 파일 시스템을 쓰면 같은 lock 파일을 보지 못하기 때문입니다.
이 경우에는 데이터베이스 unique constraint, Redis 기반 lock, 큐의 visibility timeout, 작업 테이블의 상태 전이처럼 공용 저장소를 기준으로 잠금을 잡아야 합니다.

중복 요청 자체를 업무 키 기준으로 막아야 한다면 [Node.js Idempotency Key 가이드](/development/blog/seo/2026/03/27/nodejs-idempotency-key-api-duplicate-request-guide.html)를 함께 적용하는 것이 좋습니다.
파일 lock은 실행 진입점을 좁히고, idempotency key는 업무 결과의 중복 반영을 막는 역할에 가깝습니다.

## 테스트와 점검

### H3. 두 프로세스가 동시에 들어와도 하나만 성공해야 한다

lock file 구현은 단위 테스트만으로는 충분하지 않습니다.
동시 실행을 실제로 만들어 보고 하나만 lock을 얻는지 확인해야 합니다.
간단한 검증 스크립트는 같은 작업을 여러 번 동시에 실행한 뒤 성공 횟수를 세는 방식으로 만들 수 있습니다.

```js
import { spawn } from 'node:child_process';

function runOnce() {
  return new Promise((resolve) => {
    const child = spawn(process.execPath, ['./scripts/daily-build.mjs'], {
      stdio: ['ignore', 'pipe', 'pipe']
    });

    let output = '';

    child.stdout.on('data', (chunk) => {
      output += chunk;
    });

    child.on('close', (code) => {
      resolve({ code, output });
    });
  });
}

const results = await Promise.all([
  runOnce(),
  runOnce(),
  runOnce()
]);

const started = results.filter((result) => result.output.includes('started'));

if (started.length !== 1) {
  throw new Error(`expected exactly one started job, got ${started.length}`);
}
```

이 검증은 CI에서 매번 돌리기보다 lock 구현을 바꿨을 때 실행하는 통합 테스트로 두면 충분합니다.
프로세스 격리와 병렬 테스트 기준은 [Node.js test runner isolation 가이드](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html)와 연결해 관리할 수 있습니다.

### H3. 실패 로그는 조용한 스킵과 진짜 장애를 나눠야 한다

중복 실행 때문에 lock을 얻지 못한 경우와 파일 시스템 권한 오류 때문에 lock을 만들지 못한 경우는 다르게 다뤄야 합니다.
`EEXIST`는 보통 정상적인 스킵입니다.
하지만 `EACCES`, `ENOENT`, 디스크 부족, 경로 설정 오류는 작업 자체가 실행될 수 없는 장애입니다.

운영 로그에는 아래 필드를 남기면 조사 시간이 줄어듭니다.

- 작업 이름
- lock 파일 경로
- lock 획득 여부
- 기존 lock의 나이
- 실행 PID
- 종료 코드
- 실패한 시스템 에러 코드

단, 경로에 사용자명이나 내부 디렉터리 구조가 과하게 드러나는 환경이라면 외부 공유용 로그에서는 경로를 축약해야 합니다.
민감정보를 직접 남기지 않는다는 원칙은 lock 파일과 작업 로그 모두에 적용됩니다.

## 발행 전 체크리스트

### H3. 작은 자동화부터 적용한다

처음부터 모든 서비스에 lock file을 넣을 필요는 없습니다.
중복 실행이 실제 피해를 만들 수 있고, 같은 파일 시스템 안에서 실행되는 작은 자동화부터 적용하는 편이 좋습니다.

적용 전에는 아래 항목을 확인합니다.

- 작업이 같은 시간에 두 번 실행될 수 있는가?
- 중복 실행 시 파일, 데이터베이스, 외부 API에 부작용이 생기는가?
- lock 파일을 만들 디렉터리가 저장소 밖이거나 `.gitignore` 대상인가?
- `wx`처럼 원자적 생성 방식을 사용했는가?
- 정상 종료와 예외 종료 모두에서 lock을 정리하는가?
- stale lock 기준이 작업의 최악 실행 시간보다 충분히 긴가?
- 단일 머신 lock으로 충분한 구조인가, 분산 lock이 필요한 구조인가?

lock file은 대단한 분산 시스템 기능이 아닙니다.
하지만 크론, 정적 사이트 빌드, 파일 변환, 리포트 생성 같은 자동화에서는 가장 비용이 낮은 보호 장치가 될 수 있습니다.
`exists` 확인 후 생성하는 습관을 버리고 원자적 생성, 보수적인 stale 처리, 민감정보 없는 메타데이터를 기준으로 잡으면 중복 실행으로 인한 운영 사고를 크게 줄일 수 있습니다.


---
layout: post
title: "Node.js mkdtemp 가이드: 임시 작업 디렉터리를 안전하게 만들고 정리하는 법"
date: 2026-07-28 20:00:00 +0900
lang: ko
translation_key: nodejs-mkdtemp-temporary-directory-cleanup-guide
permalink: /development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html
  x_default: /development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, mkdtemp, temporary-directory, cleanup, filesystem, automation, reliability, javascript]
description: "Node.js에서 fs.mkdtemp로 임시 작업 디렉터리를 안전하게 만들고, finally와 fs.rm으로 정리하는 방법을 정리합니다. 충돌 없는 경로 생성, 민감정보 없는 파일명, 실패 복구, CI 점검 기준까지 실무 예제로 설명합니다."
---

파일 변환, 정적 사이트 빌드, 압축 해제, 리포트 생성, 이미지 후처리 같은 자동화 작업은 중간 산출물이 필요합니다.
처음에는 `tmp/output`처럼 고정된 경로 하나를 쓰기 쉽지만, 작업이 동시에 실행되거나 이전 실행의 파일이 남아 있으면 결과가 섞일 수 있습니다.
특히 크론, CI, 큐 worker처럼 같은 스크립트가 반복 실행되는 환경에서는 임시 작업 디렉터리를 매번 새로 만들고 끝난 뒤 정리하는 습관이 중요합니다.

Node.js에서는 `fs.promises.mkdtemp()`로 충돌 가능성이 낮은 임시 디렉터리를 만들 수 있습니다.
핵심은 작업별 prefix를 정하고, 그 안에서만 파일을 만들고, 성공과 실패 모두에서 `fs.promises.rm()`으로 정리하는 것입니다.
중복 실행 자체를 막아야 한다면 [Node.js lock file 가이드](/development/blog/seo/2026/07/28/nodejs-lock-file-atomic-job-deduplication-guide.html)를 함께 적용하고, 임시 디렉터리 내부를 순회해야 한다면 [Node.js fs.promises.opendir 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html)를 같이 보면 좋습니다.

## 임시 디렉터리가 필요한 이유

### H3. 고정 경로는 자동화가 겹칠 때 위험하다

고정된 작업 디렉터리는 구현이 단순합니다.
하지만 두 실행이 같은 경로를 공유하면 한 실행이 만든 파일을 다른 실행이 읽거나 지울 수 있습니다.
이 문제는 로컬에서는 잘 보이지 않다가 CI 병렬 실행, 크론 재시도, 배포 직후 수동 재실행이 겹치는 순간 드러납니다.

예를 들어 아래 같은 구조는 충돌에 취약합니다.

```text
project/
  tmp/
    build/
      extracted/
      report.json
      output.zip
```

첫 번째 프로세스가 `report.json`을 쓰는 동안 두 번째 프로세스가 같은 파일을 덮어쓰면 결과가 마지막 writer 기준으로 바뀝니다.
한쪽이 정리 작업으로 `tmp/build`를 지우면 다른 쪽은 중간에 `ENOENT`를 만날 수 있습니다.
임시 경로는 "작업 하나당 하나"로 분리하는 편이 안전합니다.

### H3. 임시 파일명에 사용자 입력을 그대로 넣지 않는다

임시 디렉터리 이름은 디버깅에 도움이 되어야 하지만, 민감정보를 담으면 안 됩니다.
이메일, 사용자 ID, 주문 번호, 토큰 일부, 원본 파일명 같은 값은 로그와 파일 시스템 스냅샷에 남을 수 있습니다.
특히 CI artifact나 장애 조사 로그에 경로가 노출되는 조직에서는 파일명도 데이터 유출 표면이 됩니다.

좋은 prefix는 작업 성격만 드러냅니다.

- `report-`
- `image-resize-`
- `site-build-`
- `import-job-`

반대로 아래처럼 업무 데이터를 직접 넣는 방식은 피합니다.

```text
tmp/user-email-invoice-20260728/
tmp/service-token-export/
tmp/customer-id-backup/
```

필요한 추적 정보는 별도 로그에 마스킹해서 남기고, 임시 경로에는 작업명과 무작위 suffix만 둡니다.

## mkdtemp 기본 사용법

### H3. os.tmpdir와 path.join으로 prefix를 만든다

`mkdtemp()`는 전달한 prefix 뒤에 무작위 문자를 붙여 새 디렉터리를 만듭니다.
운영체제의 임시 디렉터리를 기준으로 삼으려면 `node:os`의 `tmpdir()`와 `node:path`의 `join()`을 함께 사용합니다.

```js
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

async function buildReport() {
  const workDir = await mkdtemp(join(tmpdir(), 'report-build-'));

  try {
    const reportPath = join(workDir, 'report.json');

    await writeFile(reportPath, JSON.stringify({
      generatedAt: new Date().toISOString(),
      status: 'ok'
    }));

    return await uploadReport(reportPath);
  } finally {
    await rm(workDir, {
      recursive: true,
      force: true
    });
  }
}

async function uploadReport(reportPath) {
  // 실제 업로드나 후처리 로직을 여기에 둔다.
  return reportPath;
}
```

`finally`에 정리를 두면 작업이 성공해도 실패해도 임시 디렉터리 제거를 시도합니다.
`rm()`의 `recursive: true`는 디렉터리 안의 파일까지 제거하고, `force: true`는 이미 지워진 경로에서 불필요한 예외를 줄여 줍니다.
Node.js 공식 문서 기준으로 `fsPromises.rm()`은 파일과 디렉터리를 제거하는 API이며, 재귀 삭제에는 `recursive` 옵션을 사용합니다.

### H3. prefix 끝의 구분자를 의도적으로 둔다

`mkdtemp()`는 prefix 뒤에 suffix를 붙입니다.
그래서 prefix를 어떻게 만들었는지가 최종 경로에 그대로 반영됩니다.
작업 이름 뒤에 `-` 같은 구분자를 넣으면 로그를 볼 때 작업명과 무작위 suffix를 구분하기 쉽습니다.

```js
const good = await mkdtemp(join(tmpdir(), 'site-build-'));
// /var/folders/.../site-build-a1B2c3

const confusing = await mkdtemp(join(tmpdir(), 'site-build'));
// /var/folders/.../site-builda1B2c3
```

작은 차이지만 장애 조사 때는 경로 가독성이 중요합니다.
prefix는 짧고 일관되게 정하고, 코드 여러 곳에서 직접 문자열을 반복하기보다 작업 진입점 근처의 상수로 관리합니다.

## 안전한 정리 패턴

### H3. 정리 실패를 원래 에러보다 크게 만들지 않는다

`finally` 안의 `rm()`도 실패할 수 있습니다.
Windows에서는 파일 핸들이 아직 열려 있거나 바이러스 검사, 인덱싱, 다른 프로세스 접근 때문에 `EPERM` 또는 `EBUSY`가 날 수 있습니다.
Linux와 macOS에서도 권한, 열린 파일, 예기치 않은 하위 경로 때문에 정리가 실패할 수 있습니다.

중요한 기준은 원래 작업 실패와 정리 실패를 구분하는 것입니다.
작업이 이미 실패했는데 정리 실패가 덮어쓰면 실제 원인을 놓칠 수 있습니다.
아래처럼 정리 함수를 분리해 로그만 남기고 원래 흐름을 보존할 수 있습니다.

```js
import { rm } from 'node:fs/promises';

async function cleanupWorkDir(workDir, logger = console) {
  try {
    await rm(workDir, {
      recursive: true,
      force: true,
      maxRetries: 3,
      retryDelay: 100
    });
  } catch (error) {
    logger.warn({
      event: 'temporary_directory_cleanup_failed',
      code: error?.code,
      workDir: redactTmpPath(workDir)
    });
  }
}

function redactTmpPath(path) {
  return path.replace(/^.*[/\\](report-build-[^/\\]+)$/, '$1');
}
```

`rm()`은 recursive 모드에서 일부 오류에 대해 재시도 옵션을 지원합니다.
짧은 재시도는 일시적인 파일 잠금에 도움이 될 수 있지만, 계속 실패하는 권한 오류를 무한히 숨기면 안 됩니다.
운영 로그에는 정리 실패 횟수를 남기고, 오래된 임시 디렉터리를 별도 청소 작업으로 점검하는 편이 좋습니다.

### H3. 파일 핸들과 스트림을 먼저 닫는다

임시 디렉터리를 지우기 전에 내부 파일을 쓰던 핸들과 스트림이 닫혔는지 확인해야 합니다.
`writeFile()`처럼 Promise가 완료되면 작업이 끝나는 API는 비교적 단순합니다.
하지만 `createWriteStream()`, `FileHandle`, child process, 압축 라이브러리처럼 별도 리소스를 여는 경우에는 종료 이벤트나 close를 기다려야 합니다.

```js
import { createWriteStream } from 'node:fs';
import { finished } from 'node:stream/promises';
import { join } from 'node:path';

async function writeLargeFile(workDir, readable) {
  const output = createWriteStream(join(workDir, 'large.ndjson'));

  readable.pipe(output);

  await finished(output);
}
```

스트림 완료를 기다리지 않고 곧바로 `rm(workDir)`을 호출하면 플랫폼에 따라 정리가 실패하거나, 더 나쁘게는 일부 파일만 남을 수 있습니다.
파일 생성, 압축, 업로드, 정리 순서를 명확히 나누면 실패 지점을 추적하기 쉬워집니다.

## 최신 Node.js의 disposable 흐름

### H3. mkdtempDisposable은 await using과 잘 맞는다

최신 Node.js에는 `fs.promises.mkdtempDisposable()`도 제공됩니다.
이 API는 임시 디렉터리 경로와 함께 정리 메서드를 가진 disposable 객체를 반환해 `await using` 패턴과 결합할 수 있습니다.
팀이 Explicit Resource Management 문법을 사용할 수 있는 Node.js 버전을 표준으로 정했다면, 임시 디렉터리 정리를 더 선언적으로 표현할 수 있습니다.

```js
import { mkdtempDisposable, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

async function renderPreview() {
  await using temp = await mkdtempDisposable(join(tmpdir(), 'preview-'));

  const htmlPath = join(temp.path, 'index.html');

  await writeFile(htmlPath, '<main>preview</main>');
  await runRenderer(htmlPath);
}
```

이 방식은 스코프를 벗어날 때 정리 의도를 코드 구조로 드러낼 수 있다는 장점이 있습니다.
다만 프로젝트의 Node.js 지원 버전, transpiler 설정, lint 규칙이 이 문법을 받아들이는지 먼저 확인해야 합니다.
공개 라이브러리나 오래된 LTS까지 지원하는 패키지라면 `try`와 `finally`를 쓰는 편이 호환성이 좋습니다.

### H3. 팀 표준은 지원 버전 기준으로 정한다

새 API가 있다고 해서 모든 코드에 바로 적용할 필요는 없습니다.
운영 런타임, 로컬 개발 버전, CI 이미지가 같은 수준인지 확인해야 합니다.
문법 지원이 엇갈리면 임시 디렉터리 정리를 개선하려다가 빌드 자체가 깨질 수 있습니다.

현실적인 기준은 아래처럼 나눌 수 있습니다.

- 여러 Node.js 버전을 지원하는 패키지: `mkdtemp()`와 `try/finally`를 기본값으로 둔다.
- 사내 서비스처럼 런타임을 통제할 수 있는 코드: 버전 고정 후 `mkdtempDisposable()` 도입을 검토한다.
- CLI 도구: 사용자의 설치 환경을 고려해 보수적으로 작성한다.
- 테스트 코드: CI Node.js 버전이 충분히 최신이면 disposable 패턴을 먼저 실험할 수 있다.

임시 디렉터리 정리는 기능 자체보다 실패했을 때 예측 가능한지가 더 중요합니다.
팀원이 읽었을 때 언제 만들어지고 언제 지워지는지 한눈에 보여야 합니다.

## 운영 점검 기준

### H3. 오래된 임시 디렉터리를 관찰한다

정리를 아무리 잘해도 프로세스 강제 종료, 머신 재시작, 컨테이너 중단 같은 상황에서는 임시 디렉터리가 남을 수 있습니다.
그래서 임시 경로를 만들 때 prefix를 통일하고, 오래된 작업 디렉터리를 관찰할 수 있게 해야 합니다.

예를 들어 하루에 한 번 아래 기준으로 점검할 수 있습니다.

- prefix가 팀 표준과 맞는가?
- 24시간 이상 지난 임시 디렉터리가 있는가?
- 정리 실패 로그가 최근 배포 이후 늘었는가?
- 디스크 사용량이 일정 기준을 넘었는가?
- 임시 디렉터리 안에 민감정보가 들어간 파일명이 남아 있지 않은가?

디스크 여유 공간을 먼저 확인해야 하는 작업이라면 [Node.js statfs 디스크 공간 체크 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-statfs-disk-space-check-guide.html)를 함께 적용할 수 있습니다.
압축 해제나 이미지 변환처럼 많은 파일을 만드는 작업에서는 생성 전 여유 공간 확인과 종료 후 정리 검증을 한 흐름으로 묶는 편이 좋습니다.

### H3. 테스트에서는 고유 경로를 강제한다

테스트 코드에서도 임시 디렉터리 정책은 중요합니다.
모든 테스트가 같은 `tmp/test` 경로를 쓰면 병렬 실행이나 순서 랜덤화에서 쉽게 깨집니다.
각 테스트가 자기 디렉터리를 만들고 정리하면 테스트 간 결합을 줄일 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp, rm, writeFile, readFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

test('writes a generated manifest', async () => {
  const workDir = await mkdtemp(join(tmpdir(), 'manifest-test-'));

  try {
    const manifestPath = join(workDir, 'manifest.json');

    await writeFile(manifestPath, JSON.stringify({ version: 1 }));

    const content = await readFile(manifestPath, 'utf8');

    assert.match(content, /"version":1/);
  } finally {
    await rm(workDir, { recursive: true, force: true });
  }
});
```

이 패턴은 테스트 순서가 바뀌어도 파일 시스템 상태가 섞이지 않게 합니다.
순서 의존 테스트를 찾는 흐름은 [Node.js test runner randomize 가이드](/development/blog/seo/2026/07/27/nodejs-test-runner-randomize-seed-order-dependent-guide.html)와 함께 운영하면 좋습니다.

## 발행 전 체크리스트

### H3. 임시 디렉터리는 작게 만들고 확실히 지운다

임시 디렉터리 설계는 화려한 기능이 아닙니다.
하지만 자동화가 자주 돌수록 결과 품질과 장애 조사 시간을 크게 좌우합니다.
고정 경로를 공유하지 않고, 민감정보 없는 prefix를 쓰고, 리소스를 닫은 뒤 정리하는 것만으로도 많은 파일 시스템 사고를 줄일 수 있습니다.

적용 전에는 아래 항목을 확인합니다.

- 작업 실행마다 고유한 임시 디렉터리를 만드는가?
- prefix에 사용자 입력, 토큰, 이메일, 내부 식별자가 들어가지 않는가?
- 모든 중간 산출물이 임시 디렉터리 안에만 생성되는가?
- 성공과 실패 모두에서 `finally` 또는 disposable 흐름으로 정리하는가?
- 스트림, 파일 핸들, child process가 끝난 뒤 삭제하는가?
- `rm()` 정리 실패를 로그로 남기되 원래 실패 원인을 덮어쓰지 않는가?
- 오래된 임시 디렉터리와 디스크 사용량을 운영 지표로 확인하는가?
- 테스트도 각 실행마다 고유한 임시 경로를 사용하는가?

`mkdtemp()`는 작은 API지만 자동화의 격리 경계를 분명하게 만들어 줍니다.
임시 작업 디렉터리를 매번 새로 만들고 확실히 정리하면, 크론과 CI가 겹치는 환경에서도 파일 충돌, 누락, 오래된 산출물 재사용을 훨씬 줄일 수 있습니다.

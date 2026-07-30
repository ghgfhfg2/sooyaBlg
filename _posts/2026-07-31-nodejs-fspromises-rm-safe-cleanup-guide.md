---
layout: post
title: "Node.js fs.promises.rm 가이드: 임시 파일과 빌드 산출물을 안전하게 정리하는 법"
date: 2026-07-31 08:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-rm-safe-cleanup-guide
permalink: /development/blog/seo/2026/07/31/nodejs-fspromises-rm-safe-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/07/31/nodejs-fspromises-rm-safe-cleanup-guide.html
  x_default: /development/blog/seo/2026/07/31/nodejs-fspromises-rm-safe-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, rm, cleanup, filesystem, temporary-files, build, javascript]
description: "Node.js fs.promises.rm으로 임시 파일, 캐시, 빌드 산출물을 안전하게 정리하는 방법을 정리합니다. recursive와 force 옵션, maxRetries와 retryDelay, 경로 가드, 운영 로그와 테스트 기준까지 실무 예제로 설명합니다."
---

파일을 만드는 자동화에는 반드시 정리 코드가 따라옵니다.
테스트가 만든 fixture, 빌드가 남긴 `dist/`, 이미지 변환 중간 산출물, 오래된 캐시, 실패한 배치의 임시 디렉터리는 그냥 두면 다음 실행의 원인이 됩니다.
디스크를 천천히 채우고, 오래된 파일을 최신 결과로 오해하게 만들고, 재실행할 때 충돌을 일으킵니다.

Node.js에서는 `fs.promises.rm()`으로 파일과 디렉터리를 삭제할 수 있습니다.
공식 문서 기준 `recursive: true`와 `force: true`를 함께 쓰면 Unix의 `rm -rf`에 가까운 동작을 만들 수 있고, 재귀 삭제 중 일부 오류에는 `maxRetries`와 `retryDelay`로 재시도 정책도 둘 수 있습니다.
다만 이 편리함은 위험하기도 합니다.
경로 검증 없이 `rm()`을 호출하면 잘못 계산된 경로 하나로 중요한 산출물을 지울 수 있습니다.

이 글에서는 `fs.promises.rm()`을 임시 파일 정리, 빌드 산출물 교체, 크론 작업 cleanup에 안전하게 쓰는 기준을 정리합니다.
삭제 전에 디스크 상태를 함께 점검해야 한다면 [Node.js statfs 가이드](/development/blog/seo/2026/07/30/nodejs-fspromises-statfs-disk-space-check-guide.html)를 참고하고, 임시 작업 디렉터리 생성은 [Node.js mkdtemp 가이드](/development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html)와 함께 설계하면 좋습니다.
파일 존재와 권한 점검 기준은 [Node.js fs.promises.access 가이드](/development/blog/seo/2026/07/30/nodejs-fspromises-access-file-permission-check-guide.html)에서 더 자세히 다뤘습니다.

## fs.promises.rm이 필요한 상황

### 임시 디렉터리를 실행 단위로 정리한다

빌드, 테스트, 변환 작업은 중간 산출물을 한곳에 모아 두고 마지막에 한 번에 지우는 구조가 다루기 쉽습니다.
파일 하나씩 지우는 것보다 실패 복구 범위가 명확하고, 다음 실행이 이전 산출물에 오염될 가능성도 줄어듭니다.

```js
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

export async function renderPreview() {
  const workDir = await mkdtemp(join(tmpdir(), 'preview-'));

  try {
    await writeFile(join(workDir, 'input.json'), '{"ok":true}\n', 'utf8');
    return await createPreviewImage(workDir);
  } finally {
    await rm(workDir, {
      recursive: true,
      force: true
    });
  }
}
```

`finally`에서 정리하면 성공과 실패 모두 같은 cleanup 경로를 탑니다.
`force: true`는 이미 삭제된 경로를 다시 지우려 할 때 생기는 불필요한 실패를 줄입니다.
임시 디렉터리는 작업마다 새로 만들고, 공유 디렉터리를 통째로 지우지 않는 것이 핵심입니다.

### 빌드 산출물은 교체 경계를 명확히 둔다

정적 사이트나 프론트엔드 앱을 빌드할 때 `dist/`를 매번 지우고 다시 만드는 방식은 흔합니다.
하지만 운영 배포 경로를 직접 지우는 코드는 위험합니다.
가능하면 작업용 출력 디렉터리에 새 결과를 만들고, 검증이 끝난 뒤 배포 단계로 넘기는 구조가 안전합니다.

```js
import { rm } from 'node:fs/promises';
import { resolve } from 'node:path';

const projectRoot = resolve(import.meta.dirname, '..');
const outputDir = resolve(projectRoot, 'dist');

export async function cleanBuildOutput() {
  assertInsideProject(outputDir, projectRoot);

  await rm(outputDir, {
    recursive: true,
    force: true,
    maxRetries: 3,
    retryDelay: 100
  });
}
```

삭제 대상이 프로젝트 루트 내부인지 확인하는 `assertInsideProject()` 같은 가드를 두면 실수를 줄일 수 있습니다.
예를 들어 환경변수 `OUTPUT_DIR`이 비어 있거나 `/`에 가깝게 계산되는 상황을 초기에 막을 수 있습니다.

## recursive와 force 옵션 기준

### recursive는 디렉터리 삭제 의도를 드러낸다

`recursive: true`는 디렉터리 안의 파일과 하위 디렉터리까지 삭제하겠다는 뜻입니다.
임시 작업 디렉터리, 테스트 fixture 출력, 빌드 캐시처럼 "전체를 버려도 되는 영역"에만 사용해야 합니다.
사용자 업로드 저장소, 운영 로그 루트, 프로젝트 루트처럼 여러 책임이 섞인 위치에는 바로 적용하지 않는 편이 좋습니다.

좋은 삭제 대상은 보통 아래 조건을 만족합니다.

- 애플리케이션이 직접 만든 경로다.
- 경로 이름에 작업 ID나 고정 prefix가 있다.
- 삭제해도 원본 데이터가 사라지지 않는다.
- 다음 실행에서 다시 만들 수 있다.
- 상위 디렉터리와 명확히 분리되어 있다.

반대로 "일단 비우면 되겠지"라는 이유로 넓은 경로에 `recursive`를 붙이면 장애가 됩니다.
삭제는 생성보다 되돌리기 어렵기 때문에, 대상 범위를 코드로 좁혀 두는 편이 중요합니다.

### force는 없어도 되는 파일을 허용한다

`force: true`는 경로가 없을 때 실패하지 않게 해 줍니다.
cleanup 코드에서는 유용합니다.
이전 단계가 실패해서 임시 디렉터리를 만들지 못했거나, 다른 정리 루틴이 먼저 지웠더라도 전체 작업을 불필요하게 실패시키지 않을 수 있습니다.

하지만 `force`가 모든 오류를 조용히 삼킨다는 뜻은 아닙니다.
권한 문제, 사용 중인 파일, 잘못된 파일 시스템 상태 같은 오류는 여전히 실패할 수 있습니다.
그래서 운영 코드에서는 삭제 실패를 완전히 무시하지 말고, 작업 종류와 오류 코드를 남겨야 합니다.

```js
import { rm } from 'node:fs/promises';

export async function cleanupPath(path, logger) {
  try {
    await rm(path, {
      recursive: true,
      force: true,
      maxRetries: 3,
      retryDelay: 100
    });

    return { ok: true };
  } catch (error) {
    logger.warn({
      event: 'cleanup_failed',
      code: error?.code
    });

    return {
      ok: false,
      retryable: true,
      reason: 'cleanup_failed'
    };
  }
}
```

로그에는 전체 경로를 그대로 남기지 않는 편이 좋습니다.
경로에는 사용자명, 프로젝트명, 고객 식별자, 내부 작업 ID가 들어갈 수 있습니다.
필요하면 경로 대신 작업 이름, 오류 코드, 산출물 종류, 실행 ID처럼 진단에 필요한 최소 정보만 남깁니다.

## 재시도 옵션을 언제 쓸까

### maxRetries와 retryDelay는 일시적 파일 시스템 오류에 맞다

`rm()`은 `maxRetries`와 `retryDelay` 옵션을 받을 수 있습니다.
공식 문서 기준 재귀 삭제 중 `EBUSY`, `EMFILE`, `ENFILE`, `ENOTEMPTY`, `EPERM` 같은 오류가 발생하면 선형 backoff로 재시도할 수 있습니다.
이 옵션은 `recursive: true`가 아닐 때는 의미가 없습니다.

재시도는 특히 Windows 환경, 백신 프로그램이 파일을 잠깐 잡는 환경, 테스트 runner가 파일 핸들을 늦게 닫는 환경에서 도움이 됩니다.
다만 재시도를 크게 잡으면 cleanup이 오래 걸리고, 실제 권한 문제를 늦게 발견하게 됩니다.
처음에는 2~3회 정도로 작게 시작하고 실패 로그를 보며 늘리는 편이 현실적입니다.

```js
await rm(cacheDir, {
  recursive: true,
  force: true,
  maxRetries: 3,
  retryDelay: 200
});
```

이 설정은 첫 실패 뒤 조금 기다렸다가 다시 지우는 정도의 완충 장치입니다.
운영 배치라면 cleanup 실패를 작업 전체 실패로 볼지, 다음 실행에서 다시 정리할 수 있는 경고로 볼지 정책도 함께 정해야 합니다.

### 재시도해도 안 되는 오류는 빨리 드러낸다

모든 삭제 실패가 재시도 대상은 아닙니다.
경로가 프로젝트 밖으로 벗어난 경우, 권한이 구조적으로 없는 경우, 파일 시스템이 read-only로 마운트된 경우는 반복해도 해결되지 않습니다.
이런 오류는 숨기지 말고 배포나 배치 상태에 명확히 남겨야 합니다.

```js
const RETRYABLE_CLEANUP_CODES = new Set([
  'EBUSY',
  'EMFILE',
  'ENFILE',
  'ENOTEMPTY',
  'EPERM'
]);

export function classifyCleanupError(error) {
  if (RETRYABLE_CLEANUP_CODES.has(error?.code)) {
    return 'retryable_cleanup_error';
  }

  return 'non_retryable_cleanup_error';
}
```

분류 함수가 있으면 테스트도 쉬워집니다.
삭제 구현 자체를 매번 실제 파일 시스템으로 검증하지 않아도, 오류 코드별 정책은 빠르게 단위 테스트할 수 있습니다.

## 경로 가드 만들기

### 삭제 대상은 기준 디렉터리 안에 있어야 한다

`rm()`을 안전하게 쓰려면 삭제 대상 경로를 먼저 검증해야 합니다.
문자열 prefix 비교만으로는 부족합니다.
`/app/dist2`가 `/app/dist`로 시작하는 것처럼 보이는 문제, 상대 경로의 `..`, 심볼릭 링크, 빈 문자열 같은 경우를 고려해야 합니다.

아래 예제는 실제 운영 코드의 시작점으로 쓸 수 있는 단순한 기준입니다.

```js
import { relative, resolve } from 'node:path';

export function assertInsideProject(targetPath, projectRoot) {
  const root = resolve(projectRoot);
  const target = resolve(targetPath);
  const rel = relative(root, target);

  if (rel === '' || rel.startsWith('..') || rel.startsWith('/')) {
    throw new Error('refusing to remove path outside project root');
  }
}
```

이 함수는 프로젝트 루트 자체를 삭제하는 것도 막습니다.
빌드 산출물을 지우려는 코드가 루트 전체를 지우는 일은 대부분 버그이기 때문입니다.
정말 루트 삭제가 필요한 테스트 fixture가 있다면 별도의 fixture 전용 루트를 만들어 그 안에서만 허용하는 편이 낫습니다.

### 환경변수 입력은 한 번 더 검증한다

CI나 크론에서는 삭제 대상이 환경변수로 들어오는 경우가 많습니다.
`BUILD_DIR`, `CACHE_DIR`, `OUTPUT_DIR`이 비어 있거나 예상과 다른 값이면 cleanup 코드가 가장 위험한 코드가 됩니다.
환경변수는 존재 여부, 절대 경로 변환 결과, 허용된 상위 디렉터리, 금지된 이름을 함께 확인합니다.

```js
const FORBIDDEN_BASENAMES = new Set(['', '.', '..']);

export function requireCleanupDir(name, value) {
  if (!value || FORBIDDEN_BASENAMES.has(value.trim())) {
    throw new Error(`invalid cleanup directory: ${name}`);
  }

  return value;
}
```

에러 메시지에는 환경변수 이름 정도만 넣고 실제 값은 그대로 노출하지 않는 편이 좋습니다.
값에 내부 경로나 사용자 식별자가 들어갈 수 있기 때문입니다.

## 운영에서 쓰는 cleanup 흐름

### 작업 결과와 cleanup 결과를 분리한다

cleanup 실패가 항상 작업 실패와 같은 의미는 아닙니다.
예를 들어 리포트 생성과 업로드가 이미 성공했고, 임시 디렉터리 삭제만 실패했다면 사용자에게 전달할 결과는 성공일 수 있습니다.
대신 운영 알림이나 다음 실행 전 청소 작업으로 넘기면 됩니다.

```js
export async function runJobWithCleanup({ workDir, logger }) {
  let result;

  try {
    result = await runMainJob(workDir);
  } finally {
    const cleanup = await cleanupPath(workDir, logger);

    if (!cleanup.ok) {
      logger.warn({
        event: 'job_finished_with_cleanup_warning'
      });
    }
  }

  return result;
}
```

반대로 빌드 시작 전 `dist/`를 지우지 못했다면 오래된 파일이 새 결과에 섞일 수 있습니다.
이 경우에는 빌드를 계속 진행하지 않는 편이 안전합니다.
정리 실패의 위치가 "작업 전"인지 "작업 후"인지에 따라 정책을 다르게 둬야 합니다.

### 오래된 파일 삭제는 dry-run부터 만든다

캐시나 백업처럼 보관 기간 기준으로 오래된 파일을 지우는 코드는 dry-run 모드를 먼저 만드는 것이 좋습니다.
실제 삭제 전에 어떤 파일이 삭제 대상인지 로그나 리포트로 확인할 수 있어야 합니다.

```js
export async function removeExpiredEntries(entries, { dryRun, logger }) {
  const selected = entries.filter((entry) => entry.expired);

  if (dryRun) {
    logger.info({
      event: 'expired_entries_selected',
      count: selected.length
    });

    return { removed: 0, selected: selected.length };
  }

  for (const entry of selected) {
    await cleanupPath(entry.path, logger);
  }

  return {
    removed: selected.length,
    selected: selected.length
  };
}
```

처음부터 삭제를 실행하면 정책 버그를 발견했을 때 되돌리기 어렵습니다.
특히 날짜 계산, timezone, 파일명 parsing이 들어간 삭제 정책은 dry-run 결과를 한 번 확인한 뒤 실제 삭제로 전환하는 편이 안전합니다.

## 테스트 체크리스트

### 삭제 코드는 성공보다 실패를 더 많이 테스트한다

`rm()`을 감싼 함수는 정상 삭제뿐 아니라 잘못된 경로, 없는 경로, 재시도 가능한 오류, 재시도해도 안 되는 오류를 함께 테스트해야 합니다.
실제 파일 시스템 테스트와 순수 함수 테스트를 나누면 부담이 줄어듭니다.

- 임시 디렉터리 안의 파일과 하위 디렉터리가 삭제되는가?
- 없는 경로를 `force: true`로 정리해도 성공 처리되는가?
- 프로젝트 루트 자체는 삭제 대상에서 거부되는가?
- `..`가 포함된 상대 경로가 기준 디렉터리 밖으로 벗어나지 않는가?
- 오류 코드 분류가 retryable과 non-retryable을 구분하는가?
- cleanup 실패 로그에 민감한 전체 경로가 그대로 남지 않는가?

파일 삭제 테스트는 반드시 테스트 전용 임시 디렉터리 안에서만 실행해야 합니다.
테스트 데이터 생성과 정리 모두 같은 helper를 통하게 만들면 범위를 통제하기 쉽습니다.

### 삭제 전후 상태를 명확히 검증한다

삭제 함수가 promise를 resolve했다는 사실만 확인하면 부족합니다.
테스트에서는 삭제 전 파일을 실제로 만들고, 삭제 후 접근이 실패하는지 확인해야 합니다.
경로 가드 테스트는 반대로 파일을 만들지 않고도 순수 함수로 빠르게 검증할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { access, mkdtemp, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

test('cleanup removes a temporary directory', async () => {
  const dir = await mkdtemp(join(tmpdir(), 'cleanup-test-'));
  await writeFile(join(dir, 'result.txt'), 'ok\n', 'utf8');

  await cleanupPath(dir, console);

  await assert.rejects(
    access(dir),
    { code: 'ENOENT' }
  );
});
```

운영 코드에서는 이 테스트 패턴에 timeout을 두는 것도 좋습니다.
삭제가 특정 환경에서 오래 걸리기 시작하면 테스트가 빠르게 알려 줍니다.

## 마무리 체크리스트

`fs.promises.rm()`은 정리 코드를 간단하게 만들지만, 삭제 대상이 넓어질수록 위험도 같이 커집니다.
아래 기준을 만족하면 임시 파일과 빌드 산출물 정리를 훨씬 안정적으로 운영할 수 있습니다.

- 삭제 대상은 애플리케이션이 만든 전용 디렉터리인가?
- `recursive: true`를 쓰는 경로가 프로젝트 루트나 사용자 데이터 루트가 아닌가?
- `force: true`를 cleanup 편의로 쓰되 다른 오류를 숨기지 않는가?
- `maxRetries`와 `retryDelay`를 작은 값부터 적용했는가?
- 환경변수로 받은 경로를 검증했는가?
- 작업 전 cleanup 실패와 작업 후 cleanup 실패를 다르게 처리하는가?
- dry-run이 필요한 보관 기간 삭제 로직에는 미리보기 경로가 있는가?
- 로그에 민감한 전체 경로나 사용자 입력이 그대로 남지 않는가?

삭제 코드는 눈에 띄지 않을 때 가장 위험합니다.
작게 보이는 cleanup 함수라도 경로 가드, 오류 분류, 로그 정책, 테스트를 같이 두면 자동화가 반복 실행될수록 안정성이 쌓입니다.

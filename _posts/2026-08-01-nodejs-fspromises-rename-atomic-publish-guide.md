---
layout: post
title: "Node.js fs.promises.rename 가이드: 파일 교체와 배포 반영을 원자적으로 처리하는 법"
date: 2026-08-01 08:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-rename-atomic-publish-guide
permalink: /development/blog/seo/2026/08/01/nodejs-fspromises-rename-atomic-publish-guide.html
alternates:
  ko: /development/blog/seo/2026/08/01/nodejs-fspromises-rename-atomic-publish-guide.html
  x_default: /development/blog/seo/2026/08/01/nodejs-fspromises-rename-atomic-publish-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, rename, atomic-write, deployment, filesystem, javascript]
description: "Node.js fs.promises.rename으로 파일 교체, 캐시 갱신, 정적 산출물 배포 반영을 안전하게 처리하는 방법을 정리합니다. 같은 파일 시스템 조건, 임시 파일 작성, fsync, 롤백 경계, 오류 처리 기준을 실무 예제로 설명합니다."
---

파일을 갱신하는 코드는 단순해 보이지만 운영에서는 자주 문제를 만듭니다.
설정 파일을 쓰는 중 프로세스가 종료되거나, JSON 캐시를 덮어쓰는 순간 읽기 요청이 들어오거나, 정적 산출물을 복사하던 중 배포가 반영되면 사용자는 절반만 완성된 파일을 보게 됩니다.
이런 문제는 "완성된 결과를 먼저 다른 이름으로 만들고, 마지막에 한 번에 교체한다"는 방식으로 줄일 수 있습니다.

Node.js에서는 `fs.promises.rename()`이 이 마지막 교체 단계에 자주 쓰입니다.
같은 파일 시스템 안에서 파일 이름을 바꾸는 작업은 일반적으로 원자적으로 처리되므로, 읽는 쪽은 이전 파일 또는 새 파일 중 하나를 보게 됩니다.
다만 디렉터리 경계, 운영체제별 동작, 대상 파일 존재 여부, fsync 여부를 모르면 안전하다고 착각하기 쉽습니다.

이 글에서는 `fs.promises.rename()`으로 설정 파일, 캐시 파일, 정적 배포 산출물을 안전하게 교체하는 기준을 정리합니다.
복사 단계가 필요하다면 [Node.js fs.promises.cp 가이드](/development/blog/seo/2026/07/31/nodejs-fspromises-cp-directory-copy-guide.html)를 먼저 보고, 교체 전 이전 산출물 정리는 [Node.js fs.promises.rm 가이드](/development/blog/seo/2026/07/31/nodejs-fspromises-rm-safe-cleanup-guide.html)와 함께 설계하면 좋습니다.
쓰기 내구성까지 챙겨야 한다면 [Node.js fs.promises.writeFile flush 가이드](/development/blog/seo/2026/07/29/nodejs-fspromises-writefile-flush-durable-write-guide.html)도 연결해서 볼 만합니다.

## rename이 필요한 상황

### 완성된 파일만 공개한다

캐시나 설정 파일을 바로 대상 경로에 쓰면 쓰기 도중 읽기 요청이 들어올 수 있습니다.
특히 JSON, manifest, sitemap처럼 파일 하나가 완전한 문서여야 하는 경우에는 부분 쓰기가 바로 장애가 됩니다.
이럴 때는 임시 파일에 먼저 쓰고, 검증이 끝난 뒤 `rename()`으로 최종 경로에 반영합니다.

```js
import { rename, writeFile } from 'node:fs/promises';
import { dirname, join } from 'node:path';
import { randomUUID } from 'node:crypto';

export async function writeJsonAtomically(targetPath, value) {
  const dir = dirname(targetPath);
  const tempPath = join(dir, `.${randomUUID()}.tmp`);
  const body = `${JSON.stringify(value, null, 2)}\n`;

  await writeFile(tempPath, body, 'utf8');
  await rename(tempPath, targetPath);
}
```

핵심은 임시 파일을 최종 파일과 같은 디렉터리에 만드는 것입니다.
그래야 대부분의 환경에서 같은 파일 시스템 안에서 이름만 바꾸는 작업이 되고, 교체 단계가 짧고 예측 가능해집니다.
임시 파일을 `/tmp`에 만들고 프로젝트 디렉터리로 옮기면 다른 파일 시스템을 건널 수 있고, 이때는 `EXDEV` 오류가 발생할 수 있습니다.

### 배포 산출물 반영 단계를 분리한다

정적 파일 배포에서도 같은 원칙을 쓸 수 있습니다.
새 산출물을 `public.next/` 같은 임시 디렉터리에 만들고, 검증이 끝나면 현재 공개 디렉터리를 교체합니다.
단, 디렉터리 교체는 파일 하나 교체보다 운영체제와 파일 상태의 영향을 더 받으므로 롤백 경계를 명확히 둬야 합니다.

```js
import { cp, rename, rm } from 'node:fs/promises';

export async function publishStaticDirectory(sourceDir, publicDir) {
  const nextDir = `${publicDir}.next`;
  const previousDir = `${publicDir}.previous`;

  await rm(nextDir, { recursive: true, force: true });
  await cp(sourceDir, nextDir, {
    recursive: true,
    force: false,
    errorOnExist: true
  });

  await rm(previousDir, { recursive: true, force: true });

  await rename(publicDir, previousDir).catch((error) => {
    if (error?.code !== 'ENOENT') throw error;
  });

  await rename(nextDir, publicDir);
}
```

이 구조는 "복사", "검증", "공개 반영"을 섞지 않습니다.
`cp()`가 실패하면 공개 디렉터리는 그대로 남고, `rename(nextDir, publicDir)` 단계까지 도달한 결과만 사용자에게 보입니다.
실제 운영에서는 이전 디렉터리를 언제 지울지, 교체 중 파일을 읽고 있는 프로세스가 있는지, 실패 시 어떤 알림을 보낼지까지 정해야 합니다.

## 원자적 교체의 조건

### 같은 파일 시스템 안에서 움직여야 한다

`rename()`을 안전한 교체 도구로 쓰려면 원본과 대상이 같은 파일 시스템에 있어야 합니다.
다른 디스크, 다른 마운트, 컨테이너 볼륨 경계를 넘으면 단순한 이름 변경이 아니라 복사와 삭제에 가까운 작업이 필요합니다.
Node.js의 `rename()`은 이런 경우 보통 `EXDEV` 오류를 냅니다.

```js
import { rename } from 'node:fs/promises';

export async function replaceFile(tempPath, targetPath) {
  try {
    await rename(tempPath, targetPath);
  } catch (error) {
    if (error?.code === 'EXDEV') {
      throw new Error('atomic rename failed because paths are on different devices');
    }

    throw error;
  }
}
```

`EXDEV`가 났을 때 조용히 `copyFile()`과 `unlink()`로 대체하면 원자적 교체라는 보장이 사라집니다.
그 대체 동작이 필요하다면 함수 이름과 로그에 "비원자적 fallback"임을 드러내야 합니다.
설정 파일, 인덱스 파일, manifest처럼 부분 상태가 위험한 파일은 fallback보다 같은 디렉터리에 임시 파일을 만드는 쪽이 낫습니다.

### 대상 경로를 먼저 좁힌다

교체 작업도 삭제 작업처럼 경로 가드가 필요합니다.
환경변수나 사용자 입력으로 대상 경로를 받는다면 프로젝트 루트 밖으로 벗어나지 않는지 확인해야 합니다.
잘못된 `rename()`은 중요한 파일을 덮어쓰거나 공개 경로를 엉뚱한 위치로 바꿀 수 있습니다.

```js
import { relative, resolve } from 'node:path';

export function assertInside(baseDir, targetPath) {
  const base = resolve(baseDir);
  const target = resolve(targetPath);
  const rel = relative(base, target);

  if (rel === '' || rel.startsWith('..') || rel.startsWith('/')) {
    throw new Error('path is outside the allowed directory');
  }
}
```

대상 파일만 검사하지 말고 임시 파일 경로도 함께 검사하는 편이 좋습니다.
임시 파일 이름을 만들 때는 고정된 prefix와 무작위 suffix를 함께 쓰면 충돌 가능성을 줄일 수 있습니다.
로그에는 전체 경로 대신 산출물 종류, 작업 ID, 오류 코드처럼 필요한 정보만 남깁니다.

## 내구성까지 고려한 파일 쓰기

### rename만으로 디스크 반영까지 보장하지는 않는다

`rename()`은 교체 시점을 짧고 일관되게 만드는 데 도움이 됩니다.
하지만 프로세스가 완료됐다고 해서 파일 내용과 디렉터리 변경이 모두 영구 저장 장치에 확실히 내려갔다고 단정하면 안 됩니다.
전원 장애나 파일 시스템 설정까지 고려해야 하는 환경이라면 파일 내용을 flush하고, 더 엄격하게는 디렉터리 엔트리 동기화까지 검토해야 합니다.

Node.js의 `writeFile()`에는 `flush` 옵션을 사용할 수 있는 버전이 있습니다.
간단한 캐시 파일은 과할 수 있지만, 작업 재시작 후 반드시 남아 있어야 하는 상태 파일이라면 쓰기 내구성 정책을 분명히 하는 편이 좋습니다.

```js
import { rename, writeFile } from 'node:fs/promises';
import { dirname, join } from 'node:path';
import { randomUUID } from 'node:crypto';

export async function writeStateFile(targetPath, state) {
  const tempPath = join(dirname(targetPath), `.state-${randomUUID()}.json`);
  const body = `${JSON.stringify(state)}\n`;

  await writeFile(tempPath, body, {
    encoding: 'utf8',
    flush: true
  });

  await rename(tempPath, targetPath);
}
```

`flush: true`가 모든 운영 환경의 내구성 요구를 자동으로 해결하는 것은 아닙니다.
그래도 "성공 응답을 보낸 뒤 파일이 비어 있었다" 같은 사고를 줄이는 좋은 출발점이 됩니다.
중요한 상태라면 데이터베이스나 내구성을 보장하는 큐를 쓰는 편이 더 적합할 수 있습니다.

### 검증 후 교체한다

임시 파일을 썼다면 바로 `rename()`하기 전에 최소 검증을 넣을 수 있습니다.
JSON이라면 다시 파싱하고, manifest라면 필수 키를 확인하고, HTML이라면 빈 파일이 아닌지 확인하는 정도만으로도 많은 실수를 막습니다.

```js
import { readFile, rename, writeFile } from 'node:fs/promises';

export async function publishManifest(tempPath, targetPath, manifest) {
  await writeFile(tempPath, `${JSON.stringify(manifest, null, 2)}\n`, 'utf8');

  const parsed = JSON.parse(await readFile(tempPath, 'utf8'));

  if (!parsed.version || !Array.isArray(parsed.assets)) {
    throw new Error('invalid manifest shape');
  }

  await rename(tempPath, targetPath);
}
```

검증은 복잡할 필요가 없습니다.
중요한 것은 공개 경로에 반영되기 전에 실패할 수 있는 지점을 최대한 앞당기는 것입니다.
쓰기, 검증, 교체를 함수 하나에 모아 두면 호출하는 쪽에서도 실패 의미를 이해하기 쉽습니다.

## 오류 처리와 복구 전략

### 임시 파일은 실패 후 정리한다

원자적 쓰기 함수는 실패했을 때 임시 파일을 남길 수 있습니다.
진단을 위해 잠시 남기는 것도 방법이지만, 크론이나 배치가 자주 실행된다면 오래된 임시 파일 정리 정책이 필요합니다.
보통은 `finally`에서 지우되, 삭제 실패는 원래 오류를 덮지 않게 처리합니다.

```js
import { rm } from 'node:fs/promises';

export async function cleanupTempFile(tempPath, logger) {
  try {
    await rm(tempPath, { force: true });
  } catch (error) {
    logger.warn({
      event: 'temp_cleanup_failed',
      code: error?.code
    });
  }
}
```

삭제 실패를 무시해도 된다는 뜻은 아닙니다.
다만 원래의 쓰기 실패 원인을 숨기면 디버깅이 어려워집니다.
cleanup 실패는 경고로 남기고, 호출자에게는 원래 실패를 전달하는 편이 보통 더 유용합니다.

### 읽는 쪽은 재시도를 짧게 둔다

교체하는 쪽이 `rename()`을 잘 써도 읽는 쪽에서 파일을 여는 순간 일시적 오류를 만날 수 있습니다.
특히 Windows 환경이나 네트워크 파일 시스템에서는 파일 잠금과 권한 문제가 더 자주 드러납니다.
읽기 경로에는 짧은 재시도를 두되, 오래 기다리며 장애를 숨기지 않는 것이 좋습니다.

```js
import { readFile } from 'node:fs/promises';

export async function readJsonWithRetry(path, attempts = 3) {
  let lastError;

  for (let attempt = 1; attempt <= attempts; attempt += 1) {
    try {
      return JSON.parse(await readFile(path, 'utf8'));
    } catch (error) {
      lastError = error;
      if (!['ENOENT', 'EBUSY', 'EPERM'].includes(error?.code)) {
        throw error;
      }
      await new Promise((resolve) => setTimeout(resolve, attempt * 50));
    }
  }

  throw lastError;
}
```

재시도 횟수는 작게 시작하는 편이 좋습니다.
파일이 계속 없다면 배포 산출물 생성이 실패했거나 경로가 틀린 것입니다.
이런 오류는 빠르게 드러나야 배포 파이프라인에서 바로 고칠 수 있습니다.

## 실무 체크리스트

### 파일 교체 전 확인할 것

`fs.promises.rename()`을 쓰기 전에 아래 기준을 점검하면 운영 사고를 줄일 수 있습니다.

- 임시 파일과 최종 파일이 같은 디렉터리에 있는가?
- 대상 경로가 허용된 루트 내부로 제한되는가?
- 임시 파일 이름에 충돌 방지 suffix가 있는가?
- 공개 반영 전 최소 검증을 수행하는가?
- `EXDEV` 오류를 조용히 fallback하지 않는가?
- 실패 후 임시 파일 cleanup 정책이 있는가?
- 로그에 민감한 전체 경로를 과하게 남기지 않는가?

이 체크리스트는 파일 하나를 쓰는 코드에도 적용할 수 있고, 정적 디렉터리 배포에도 응용할 수 있습니다.
중요한 파일일수록 "쓰기 성공"과 "공개 반영"을 같은 단계로 취급하지 않는 편이 안전합니다.

### 테스트로 확인할 것

테스트는 실제 파일 시스템을 조금 사용하는 편이 좋습니다.
임시 디렉터리에 기존 파일을 만들고, 새 임시 파일을 `rename()`한 뒤 최종 내용만 보이는지 확인합니다.
또한 잘못된 경로가 거부되는지, `EXDEV` 같은 오류가 별도 분류되는지, cleanup 실패가 원래 오류를 덮지 않는지도 테스트할 수 있습니다.

```js
import { mkdtemp, readFile, rename, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';
import test from 'node:test';
import assert from 'node:assert/strict';

test('atomic replacement publishes complete file', async () => {
  const dir = await mkdtemp(join(tmpdir(), 'rename-guide-'));
  const target = join(dir, 'cache.json');
  const temp = join(dir, '.cache.tmp');

  await writeFile(target, '{"version":1}\n', 'utf8');
  await writeFile(temp, '{"version":2}\n', 'utf8');
  await rename(temp, target);

  assert.equal(await readFile(target, 'utf8'), '{"version":2}\n');
});
```

테스트에서도 임시 디렉터리를 실행 단위로 만들고 정리하면 반복 실행이 안정적입니다.
파일 시스템 API는 운영체제별 차이가 있으므로 CI 환경을 하나로만 보지 말고, 배포 대상과 가까운 환경에서 한 번 더 확인하는 것이 좋습니다.

## 마무리

`fs.promises.rename()`은 작은 API지만 파일 공개 시점을 통제하는 데 매우 중요합니다.
임시 파일에 먼저 쓰고, 검증하고, 같은 디렉터리 안에서 최종 경로로 교체하면 부분 쓰기와 중간 상태 노출을 크게 줄일 수 있습니다.

다만 `rename()`만 붙였다고 모든 문제가 해결되지는 않습니다.
같은 파일 시스템 조건, 경로 가드, `EXDEV` 처리, 쓰기 내구성, 실패 후 cleanup까지 함께 설계해야 실무에서 믿을 수 있는 파일 교체 흐름이 됩니다.

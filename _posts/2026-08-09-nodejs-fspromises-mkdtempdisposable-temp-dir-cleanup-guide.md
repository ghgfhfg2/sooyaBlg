---
layout: post
title: "Node.js fsPromises.mkdtempDisposable 가이드: 임시 디렉터리 자동 정리 패턴"
date: 2026-08-09 20:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-mkdtempdisposable-temp-dir-cleanup-guide
permalink: /development/blog/seo/2026/08/09/nodejs-fspromises-mkdtempdisposable-temp-dir-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/08/09/nodejs-fspromises-mkdtempdisposable-temp-dir-cleanup-guide.html
  x_default: /development/blog/seo/2026/08/09/nodejs-fspromises-mkdtempdisposable-temp-dir-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs, fspromises, mkdtempdisposable, temporary-directory, cleanup, javascript, backend, automation]
description: "Node.js fsPromises.mkdtempDisposable로 임시 디렉터리를 만들고 자동 정리하는 방법을 정리합니다. await using, remove(), 오류 처리, 테스트와 배치 작업에서의 안전한 cleanup 패턴을 예제로 설명합니다."
---

테스트, 이미지 변환, 압축 파일 생성, 배치 작업 중간 산출물 처리에는 임시 디렉터리가 자주 필요합니다.
문제는 만드는 코드보다 지우는 코드가 더 쉽게 빠진다는 점입니다.
성공 경로에서는 정리되지만 예외가 발생하면 남거나, 병렬 작업 중 같은 경로를 공유해 예상치 못한 파일을 지우는 실수가 생길 수 있습니다.

`fsPromises.mkdtempDisposable()`은 Node.js의 `node:fs/promises` 모듈이 제공하는 임시 디렉터리 생성 API입니다.
생성된 디렉터리 경로와 제거 함수를 함께 담은 async-disposable 객체를 반환하므로, `await using` 구문과 조합하면 스코프가 끝날 때 임시 파일을 자동으로 정리할 수 있습니다.
이 글에서는 `mkdtempDisposable()`의 기본 사용법, `remove()` 직접 호출 기준, 실무에서 놓치기 쉬운 cleanup 오류 처리 방법을 정리합니다.

임시 파일을 만들기 전에 경로 조작 기준을 먼저 잡아야 한다면 [Node.js path.resolve/normalize 가이드](/development/blog/seo/2026/08/06/nodejs-path-resolve-normalize-traversal-prevention-guide.html)를 함께 참고하세요.
임시 산출물을 최종 위치로 옮기는 배포 흐름은 [Node.js fsPromises.rename 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-rename-atomic-publish-guide.html)와 함께 보면 좋습니다.
파일 삭제 정책을 더 넓게 정리하고 싶다면 [Node.js fsPromises.rm 가이드](/development/blog/seo/2026/07/31/nodejs-fspromises-rm-safe-cleanup-guide.html)도 이어서 도움이 됩니다.

## fsPromises.mkdtempDisposable이 해결하는 문제

### H3. 임시 디렉터리는 생성보다 정리가 어렵다

기존에는 `mkdtemp()`로 임시 디렉터리를 만들고 `finally`에서 직접 지우는 패턴을 많이 썼습니다.

```js
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

const dir = await mkdtemp(join(tmpdir(), 'report-'));

try {
  await writeFile(join(dir, 'summary.json'), JSON.stringify({ ok: true }));
  // 압축, 업로드, 변환 같은 본 작업 수행
} finally {
  await rm(dir, { recursive: true, force: true });
}
```

이 방식은 명시적이라 이해하기 쉽지만, 반복될수록 정리 코드가 흩어집니다.
중간에 `return`이 늘어나거나 여러 임시 리소스를 함께 다루면 `finally` 블록이 길어지고, cleanup 실패를 어떤 수준으로 처리할지도 매번 결정해야 합니다.

`mkdtempDisposable()`은 이 반복을 리소스 수명 관리 패턴으로 옮깁니다.
디렉터리 생성과 제거 책임이 하나의 객체에 묶이기 때문에 "이 스코프에서만 쓰고 끝나면 지운다"는 의도를 코드 구조로 표현할 수 있습니다.

### H3. await using은 스코프 기반 cleanup을 만든다

`mkdtempDisposable()`은 `path`, `remove()`, `[Symbol.asyncDispose]()`를 가진 객체를 반환합니다.
`await using`으로 선언하면 블록을 빠져나갈 때 비동기 dispose가 실행되고, 디렉터리와 내부 파일이 제거됩니다.

```js
import { mkdtempDisposable, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

export async function buildReportArchive(report) {
  await using workspace = await mkdtempDisposable(join(tmpdir(), 'report-'));

  const manifestPath = join(workspace.path, 'manifest.json');
  await writeFile(manifestPath, JSON.stringify(report, null, 2));

  return createArchiveFromDirectory(workspace.path);
}
```

핵심은 `workspace.path`를 작업 공간으로만 쓰고, 함수 밖으로 "나중에 계속 쓸 경로"처럼 넘기지 않는 것입니다.
스코프를 벗어나면 디렉터리가 삭제될 수 있으므로, 반환값은 파일 내용이나 최종 산출물처럼 임시 디렉터리에 의존하지 않는 형태여야 합니다.

## 기본 사용법

### H3. prefix는 tmpdir과 구분자를 함께 만든다

`mkdtempDisposable()`은 기존 `mkdtemp()`와 마찬가지로 prefix 뒤에 무작위 문자열을 붙여 고유한 디렉터리를 만듭니다.
따라서 운영체제의 임시 디렉터리 안에 만들려면 `path.join(tmpdir(), 'prefix-')`처럼 부모 경로와 prefix를 안전하게 조합하는 편이 좋습니다.

```js
import { mkdtempDisposable } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

await using tempDir = await mkdtempDisposable(join(tmpdir(), 'image-job-'));

console.log(tempDir.path);
```

`tmpdir()` 문자열에 직접 prefix를 이어 붙이면 의도와 다른 위치에 디렉터리가 만들어질 수 있습니다.
크로스 플랫폼 스크립트라면 경로 구분자를 직접 하드코딩하지 말고 `join()`을 사용하는 편이 안전합니다.

### H3. remove()는 조기 정리가 필요할 때만 직접 호출한다

대부분의 경우 `await using`에 맡기면 충분합니다.
다만 큰 임시 산출물을 만든 뒤 함수의 남은 작업이 오래 걸린다면, 디스크 사용량을 줄이기 위해 `remove()`를 직접 호출할 수 있습니다.

```js
import { mkdtempDisposable, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

export async function convertAndUpload(items) {
  await using tempDir = await mkdtempDisposable(join(tmpdir(), 'convert-'));

  for (const item of items) {
    await writeFile(join(tempDir.path, `${item.id}.json`), JSON.stringify(item));
  }

  const archive = await createArchiveFromDirectory(tempDir.path);
  await tempDir.remove();

  return uploadArchive(archive);
}
```

직접 `remove()`를 호출한 뒤에는 같은 경로에 다시 쓰지 않아야 합니다.
또한 cleanup이 완료된 경로를 로그나 이벤트에 남길 때는 내부 파일명, 사용자 식별자, 임시 토큰이 포함되지 않도록 주의해야 합니다.

## 실무 패턴

### H3. 테스트 격리는 임시 디렉터리 단위로 잡는다

파일 시스템을 다루는 테스트는 테스트마다 독립된 작업 디렉터리를 가져야 합니다.
공유된 `tmp/test` 디렉터리를 쓰면 병렬 실행, 실패 후 잔여 파일, 로컬 환경 차이 때문에 테스트가 흔들립니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { mkdtempDisposable, readFile, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

test('writes normalized config file', async () => {
  await using tempDir = await mkdtempDisposable(join(tmpdir(), 'config-test-'));

  const inputPath = join(tempDir.path, 'input.json');
  const outputPath = join(tempDir.path, 'output.json');

  await writeFile(inputPath, JSON.stringify({ port: '3000' }));
  await normalizeConfigFile(inputPath, outputPath);

  const output = JSON.parse(await readFile(outputPath, 'utf8'));
  assert.deepEqual(output, { port: 3000 });
});
```

이렇게 하면 테스트가 성공하든 실패하든 작업 디렉터리 수명이 테스트 블록과 함께 끝납니다.
테스트 실패 시 파일을 남겨 디버깅하고 싶다면 환경변수로 cleanup을 끄는 별도 디버그 모드를 두는 편이 좋습니다.

### H3. 최종 산출물은 임시 디렉터리 밖으로 확정한다

임시 디렉터리는 중간 작업 공간입니다.
사용자에게 전달할 파일이나 배포에 사용할 파일은 임시 디렉터리 안에 그대로 두지 말고, 검증 후 최종 위치로 이동해야 합니다.

```js
import { mkdtempDisposable, rename, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

export async function publishSnapshot(snapshot) {
  await using tempDir = await mkdtempDisposable(join(tmpdir(), 'snapshot-'));

  const draftPath = join(tempDir.path, 'snapshot.json');
  const finalPath = join(process.cwd(), 'public', 'snapshot.json');

  await writeFile(draftPath, JSON.stringify(snapshot, null, 2));
  await validateSnapshotFile(draftPath);
  await rename(draftPath, finalPath);

  return finalPath;
}
```

이 패턴은 "임시 위치에서 만들고 검증한 뒤 최종 위치로 확정한다"는 흐름을 분명하게 만듭니다.
다만 `rename()`은 파일 시스템과 운영체제 조건에 따라 원자성이 달라질 수 있으므로, 같은 볼륨 안에서 이동하는 구조인지 확인하는 것이 좋습니다.

## 오류 처리 기준

### H3. cleanup 실패도 운영 신호로 본다

`mkdtempDisposable()`의 dispose 과정에서 디렉터리를 지우지 못하면 오류가 발생할 수 있습니다.
권한 문제, 열려 있는 파일 핸들, 백신 또는 인덱서의 파일 점유, 잘못된 경로 사용 등이 원인이 될 수 있습니다.

```js
import { mkdtempDisposable, open } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

export async function processWithFileHandle() {
  await using tempDir = await mkdtempDisposable(join(tmpdir(), 'handle-'));

  const file = await open(join(tempDir.path, 'data.txt'), 'w');

  try {
    await file.writeFile('hello');
  } finally {
    await file.close();
  }
}
```

임시 디렉터리 안의 파일 핸들을 닫지 않으면 cleanup이 실패할 수 있습니다.
따라서 파일 핸들, 네트워크 스트림, 압축 스트림처럼 별도의 수명을 가진 리소스는 임시 디렉터리보다 먼저 닫히도록 코드를 배치해야 합니다.

### H3. 오래 가는 작업에는 cleanup 로그를 남긴다

짧은 테스트에서는 cleanup 오류가 그대로 테스트 실패로 드러나면 충분합니다.
하지만 배치 작업이나 CI 작업에서는 임시 디렉터리 정리 실패가 디스크 고갈의 전조일 수 있으므로 별도 로그를 남기는 편이 좋습니다.

```js
export async function runBatchJob(logger) {
  try {
    await using tempDir = await createJobWorkspace();
    await runPipeline(tempDir.path);
  } catch (error) {
    logger.error({
      event: 'batch_failed',
      message: error.message
    });

    throw error;
  }
}
```

로그에는 전체 임시 경로나 파일 목록을 그대로 남기지 않는 편이 안전합니다.
사용자 업로드 파일명, 내부 프로젝트 경로, 토큰이 포함된 산출물 이름이 섞일 수 있기 때문입니다.
필요하다면 작업 ID처럼 추적 가능한 최소 정보만 남깁니다.

## 도입 전 체크리스트

### H3. 런타임 버전을 먼저 확인한다

`fsPromises.mkdtempDisposable()`은 최신 Node.js 런타임에서 사용할 수 있는 API입니다.
프로덕션, CI, 로컬 개발 환경의 Node.js 버전이 서로 다르면 일부 환경에서만 실패할 수 있습니다.
도입 전에는 `node --version`과 배포 런타임 정책을 함께 확인하세요.

```sh
node --version
```

지원 버전이 섞여 있다면 기존 `mkdtemp()`와 `finally` 기반 cleanup을 유지하거나, 런타임 업그레이드 계획과 함께 적용하는 편이 안전합니다.

### H3. 임시 디렉터리 수명은 짧게 유지한다

임시 디렉터리를 함수 밖으로 넘겨 오래 보관하면 자동 정리의 장점이 사라집니다.
다음 기준을 지키면 사고 가능성을 줄일 수 있습니다.

- 임시 디렉터리 경로를 데이터베이스나 큐 메시지에 저장하지 않는다.
- 최종 산출물은 검증 후 별도 저장소나 최종 경로로 옮긴다.
- 병렬 작업은 각자 고유한 임시 디렉터리를 사용한다.
- cleanup 실패는 조용히 무시하지 말고 로그나 테스트 실패로 드러낸다.
- 임시 파일명에 토큰, 이메일, 주민번호 같은 민감정보를 넣지 않는다.

## FAQ

### H3. mkdtempDisposable과 mkdtemp의 차이는 무엇인가요?

`mkdtemp()`는 생성된 디렉터리 경로 문자열을 반환하고, 삭제는 호출자가 직접 처리해야 합니다.
`mkdtempDisposable()`은 경로와 삭제 함수를 가진 async-disposable 객체를 반환해 `await using` 스코프 종료 시 자동 정리할 수 있습니다.

### H3. await using을 쓰지 않고도 사용할 수 있나요?

가능합니다.
반환된 객체의 `remove()`를 직접 호출하면 됩니다.
다만 직접 호출 방식은 cleanup 누락 가능성이 있으므로, 스코프 기반 정리가 맞는 작업이라면 `await using`을 우선 고려하는 편이 좋습니다.

### H3. 임시 디렉터리 안에 파일이 있어도 삭제되나요?

공식 문서 기준으로 dispose 시 디렉터리와 그 안의 내용이 비동기로 제거됩니다.
다만 열려 있는 파일 핸들, 권한 문제, 운영체제별 파일 잠금 상태에 따라 실패할 수 있으므로 리소스 닫기 순서를 명확히 해야 합니다.

## 마무리

`fsPromises.mkdtempDisposable()`은 임시 디렉터리를 "만들고 잊는" 코드가 아니라 "스코프 안에서 쓰고 자동으로 정리하는" 코드로 바꿔 줍니다.
테스트 격리, 변환 파이프라인, 배치 작업처럼 임시 산출물이 많은 코드에서 특히 유용합니다.

도입할 때는 Node.js 런타임 버전, `await using` 지원 여부, 파일 핸들 정리 순서를 함께 확인하세요.
임시 디렉터리 cleanup은 사소한 편의 기능처럼 보이지만, 장기 운영에서는 디스크 안정성과 개인정보 노출 위험을 줄이는 기본 위생에 가깝습니다.

참고: [Node.js File system 공식 문서](https://nodejs.org/api/fs.html)

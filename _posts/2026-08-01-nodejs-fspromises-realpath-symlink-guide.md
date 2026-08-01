---
layout: post
title: "Node.js realpath 가이드: 심볼릭 링크와 실제 경로를 안전하게 구분하는 법"
date: 2026-08-01 20:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-realpath-symlink-guide
permalink: /development/blog/seo/2026/08/01/nodejs-fspromises-realpath-symlink-guide.html
alternates:
  ko: /development/blog/seo/2026/08/01/nodejs-fspromises-realpath-symlink-guide.html
  x_default: /development/blog/seo/2026/08/01/nodejs-fspromises-realpath-symlink-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, realpath, symlink, lstat, readlink, filesystem, javascript]
description: "Node.js fs.promises.realpath, lstat, readlink로 심볼릭 링크와 실제 파일 경로를 안전하게 구분하는 방법을 정리합니다. 경로 정규화, 프로젝트 루트 가드, 링크 루프와 권한 오류 처리, 테스트 기준까지 실무 예제로 설명합니다."
---

파일 경로를 다루는 코드는 보이는 문자열만 믿으면 쉽게 흔들립니다.
`./uploads/image.png`처럼 보이는 경로가 실제로는 심볼릭 링크를 거쳐 다른 디렉터리를 가리킬 수 있고, 배포 환경에서는 `current -> releases/2026-08-01` 같은 링크 구조로 실제 산출물 위치를 바꾸기도 합니다.
캐시, 업로드, 플러그인, 정적 파일 서버처럼 경로가 권한 경계와 연결되는 코드라면 "입력 경로"와 "실제 파일 시스템 경로"를 구분해야 합니다.

Node.js에서는 `fs.promises.realpath()`로 경로의 실제 위치를 해석할 수 있고, `fs.promises.lstat()`과 `fs.promises.readlink()`로 심볼릭 링크 자체를 검사할 수 있습니다.
공식 문서 기준 `realpath()`는 실제 경로를 확인해 문자열이나 버퍼로 반환하고, `lstat()`은 링크를 따라가지 않고 링크 자체의 `Stats`를 가져옵니다.
이 글에서는 세 API를 어떤 기준으로 나눠 쓰는지, 프로젝트 루트 밖으로 빠져나가는 경로를 어떻게 막는지, 운영 코드에서 어떤 오류를 별도로 다뤄야 하는지 정리합니다.

파일을 최종 위치에 반영하는 단계는 [Node.js rename 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-rename-atomic-publish-guide.html)와 함께 보면 좋습니다.
경로를 쓰기 전에 존재와 권한을 점검하는 기준은 [Node.js access 가이드](/development/blog/seo/2026/07/30/nodejs-fspromises-access-file-permission-check-guide.html)를 참고하고, 디렉터리 전체를 옮기는 배포 흐름은 [Node.js cp 가이드](/development/blog/seo/2026/07/31/nodejs-fspromises-cp-directory-copy-guide.html)와 연결해서 설계할 수 있습니다.

## realpath가 필요한 상황

### 심볼릭 링크 뒤의 실제 위치를 확인한다

`realpath(path)`는 경로에 포함된 심볼릭 링크를 해석해 실제 경로를 반환합니다.
사용자가 입력한 경로가 프로젝트 내부처럼 보여도, 중간에 링크가 있으면 최종 위치가 달라질 수 있습니다.
그래서 파일을 공개하거나 쓰기 전에 "최종적으로 어디를 가리키는가"를 확인해야 하는 경우가 있습니다.

```js
import { realpath } from 'node:fs/promises';

export async function resolveRealFilePath(path) {
  return realpath(path, 'utf8');
}
```

이 함수는 단순하지만 의미가 큽니다.
`path.resolve()`가 문자열 수준에서 `.`과 `..`를 정리한다면, `realpath()`는 파일 시스템을 실제로 조회해 링크를 따라갑니다.
경로 보안, 배포 링크, 캐시 디렉터리 검증처럼 실제 위치가 중요한 코드에서는 둘을 같은 도구로 보면 안 됩니다.

### 프로젝트 루트 안에 남아 있는지 검증한다

정적 파일 서버나 업로드 처리기는 "프로젝트 내부 파일만 접근한다"는 경계를 가져야 합니다.
하지만 입력 경로를 `resolve()`로만 정규화하면 심볼릭 링크가 루트 밖을 가리키는 상황을 놓칠 수 있습니다.
루트와 대상 경로를 모두 `realpath()`로 바꾼 뒤 상대 경로를 계산하면 더 현실적인 검증을 할 수 있습니다.

```js
import { realpath } from 'node:fs/promises';
import { relative, sep } from 'node:path';

export async function assertRealPathInside(rootDir, targetPath) {
  const realRoot = await realpath(rootDir, 'utf8');
  const realTarget = await realpath(targetPath, 'utf8');
  const rel = relative(realRoot, realTarget);

  if (rel === '' || rel.startsWith('..') || rel.startsWith(sep)) {
    throw new Error('target path is outside the allowed root');
  }

  return realTarget;
}
```

이 검증은 읽기 대상 파일이 이미 존재하는 경우에 적합합니다.
새 파일을 만들기 전에는 대상 파일이 아직 없어서 `realpath(targetPath)`가 `ENOENT`로 실패할 수 있습니다.
그때는 부모 디렉터리의 실제 경로를 확인하고, 새 파일 이름은 별도로 검증하는 방식이 더 맞습니다.

## lstat과 readlink를 함께 쓰는 기준

### 링크 자체를 검사하려면 lstat을 쓴다

`stat()`은 기본적으로 심볼릭 링크를 따라간 대상의 정보를 봅니다.
반대로 `lstat()`은 링크를 따라가지 않고 링크 자체의 정보를 반환합니다.
"이 경로가 링크인지 아닌지"를 판단해야 한다면 `lstat()`이 필요합니다.

```js
import { lstat } from 'node:fs/promises';

export async function describeEntry(path) {
  const stats = await lstat(path);

  if (stats.isSymbolicLink()) {
    return { type: 'symlink' };
  }

  if (stats.isDirectory()) {
    return { type: 'directory' };
  }

  if (stats.isFile()) {
    return { type: 'file' };
  }

  return { type: 'other' };
}
```

이 구분은 파일 탐색기, 백업 도구, 배포 스크립트에서 중요합니다.
링크를 파일처럼 복사할지, 링크 자체를 보존할지, 링크 대상까지 따라갈지에 따라 결과가 달라지기 때문입니다.
특히 배포 산출물을 복사할 때는 링크 정책을 명시하지 않으면 로컬에서는 괜찮던 코드가 CI나 서버에서 다른 결과를 만들 수 있습니다.

### 링크 대상 문자열은 readlink로 읽는다

링크가 어디를 가리키는지 보고 싶다면 `readlink(path)`를 사용합니다.
반환되는 값은 실제 경로로 정규화된 결과가 아니라 링크에 저장된 대상 문자열입니다.
상대 링크라면 상대 문자열 그대로 나올 수 있으므로, 링크가 위치한 디렉터리를 기준으로 해석해야 합니다.

```js
import { readlink } from 'node:fs/promises';
import { dirname, resolve } from 'node:path';

export async function readSymlinkTarget(linkPath) {
  const target = await readlink(linkPath, 'utf8');

  return {
    rawTarget: target,
    resolvedFromLinkDir: resolve(dirname(linkPath), target)
  };
}
```

`readlink()` 결과를 곧바로 신뢰하면 안 됩니다.
상대 경로, 절대 경로, 깨진 링크, 루프가 모두 가능하기 때문입니다.
운영 코드에서는 `readlink()`로 사람이 이해할 수 있는 진단 정보를 만들고, 실제 접근 허용 여부는 `realpath()`나 상위 정책으로 다시 판단하는 편이 안전합니다.

## 경로 보안 패턴

### 문자열 정규화와 실제 경로 확인을 분리한다

경로 검증은 보통 두 단계로 나누면 이해하기 쉽습니다.
먼저 문자열 수준에서 빈 값, 절대 경로 허용 여부, `..` 사용 여부를 제한합니다.
그다음 실제 파일 시스템 조회가 필요한 지점에서 `realpath()`로 심볼릭 링크까지 확인합니다.

```js
import { realpath } from 'node:fs/promises';
import { resolve, relative, sep } from 'node:path';

function assertSafeRelativePath(input) {
  if (!input || input.includes('\0')) {
    throw new Error('invalid path');
  }

  if (input.startsWith('/') || input.includes('..')) {
    throw new Error('path must stay relative');
  }
}

export async function resolvePublicAsset(rootDir, inputPath) {
  assertSafeRelativePath(inputPath);

  const candidate = resolve(rootDir, inputPath);
  const realRoot = await realpath(rootDir, 'utf8');
  const realCandidate = await realpath(candidate, 'utf8');
  const rel = relative(realRoot, realCandidate);

  if (rel === '' || rel.startsWith('..') || rel.startsWith(sep)) {
    throw new Error('asset escaped public root');
  }

  return realCandidate;
}
```

이 예제는 이미 존재하는 정적 파일을 읽는 흐름에 맞습니다.
업로드처럼 새 파일을 만드는 흐름이라면 파일 이름을 allowlist로 제한하고, 부모 디렉터리만 `realpath()`로 확인해야 합니다.
새 파일 경로까지 `realpath()`로 확인하려 하면 아직 파일이 없어서 실패하는 것이 정상입니다.

### 링크를 무조건 금지할지 따라갈지 정한다

모든 서비스가 심볼릭 링크를 허용해야 하는 것은 아닙니다.
사용자 업로드 디렉터리, 플러그인 샌드박스, 빌드 캐시처럼 경계가 중요한 곳에서는 링크 자체를 금지하는 편이 단순할 수 있습니다.
반대로 배포 릴리스 디렉터리처럼 링크가 운영 구조의 일부라면 허용하되 최종 실제 경로를 검증해야 합니다.

```js
import { lstat } from 'node:fs/promises';

export async function rejectSymlink(path) {
  const stats = await lstat(path);

  if (stats.isSymbolicLink()) {
    throw new Error('symbolic links are not allowed here');
  }
}
```

정책은 코드 이름에 드러나는 것이 좋습니다.
`rejectSymlink()`는 링크를 허용하지 않는다는 뜻이고, `assertRealPathInside()`는 링크를 따라가되 최종 위치를 검사한다는 뜻입니다.
두 정책을 섞어 놓으면 나중에 보안 리뷰나 장애 분석에서 의도를 파악하기 어려워집니다.

## 오류 처리와 운영 로그

### ENOENT와 ELOOP를 구분한다

`realpath()`는 파일이 없거나, 경로 중간이 없거나, 링크가 깨졌을 때 실패할 수 있습니다.
이때 `ENOENT`는 흔히 "대상이 없다"는 의미로 다룹니다.
심볼릭 링크가 순환 구조를 만들면 `ELOOP` 같은 오류가 발생할 수 있습니다.

```js
import { realpath } from 'node:fs/promises';

export async function tryResolveRealPath(path, logger) {
  try {
    return await realpath(path, 'utf8');
  } catch (error) {
    if (error?.code === 'ENOENT') {
      logger.info({ path }, 'path does not exist');
      return null;
    }

    if (error?.code === 'ELOOP') {
      logger.warn({ path }, 'symbolic link loop detected');
      throw new Error('invalid symbolic link chain', { cause: error });
    }

    throw error;
  }
}
```

사용자에게는 내부 절대 경로를 그대로 노출하지 않는 편이 좋습니다.
로그에는 진단에 필요한 경로, 오류 코드, 작업 ID를 남기되, 응답 메시지는 "파일을 찾을 수 없습니다"나 "허용되지 않은 경로입니다"처럼 경계를 지키는 문장으로 제한합니다.

### TOCTOU 가능성은 여전히 남는다

`realpath()`로 확인한 뒤 실제로 파일을 읽기 전까지의 짧은 시간에도 파일 시스템 상태는 바뀔 수 있습니다.
따라서 `realpath()` 검증은 최종 방어선이 아니라 위험을 줄이는 단계입니다.
실제 `readFile()`, `open()`, `rename()` 호출의 실패도 반드시 처리해야 합니다.

이 점은 `access()`와 비슷합니다.
사전 검사를 통과했다고 해서 다음 I/O가 성공한다고 보장하지 않습니다.
검증 단계는 의도를 좁히고 오류 메시지를 좋게 만들기 위한 장치이고, 실제 작업 단계는 여전히 오류 코드 기반으로 복구해야 합니다.

## 테스트 체크리스트

### 정상 경로와 링크 경로를 함께 검증한다

경로 처리 코드는 운영체제와 파일 시스템 상태의 영향을 많이 받습니다.
단위 테스트에서는 임시 디렉터리를 만들고, 일반 파일, 내부 링크, 외부 링크, 깨진 링크를 나눠 확인하는 편이 좋습니다.

```js
import { mkdtemp, symlink, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';
import { test } from 'node:test';
import assert from 'node:assert/strict';

test('rejects symlink that escapes root', async () => {
  const root = await mkdtemp(join(tmpdir(), 'asset-root-'));
  const outside = await mkdtemp(join(tmpdir(), 'outside-'));

  await writeFile(join(outside, 'outside.txt'), 'outside\n', 'utf8');
  await symlink(join(outside, 'outside.txt'), join(root, 'link.txt'));

  await assert.rejects(
    () => resolvePublicAsset(root, 'link.txt'),
    /escaped public root/
  );
});
```

실제 프로젝트에서는 Windows 권한, 개발자 머신의 심볼릭 링크 생성 권한, CI 환경의 파일 시스템 차이도 고려해야 합니다.
테스트가 특정 운영체제에서만 가능한 경우에는 조건부 skip을 두고, 핵심 경로 검증 함수는 순수 문자열 테스트와 파일 시스템 통합 테스트를 나눠 관리하면 안정적입니다.

## 마무리

`realpath()`는 경로 문자열을 더 예쁘게 만드는 함수가 아니라, 파일 시스템의 실제 위치를 확인하는 함수입니다.
`lstat()`은 링크 자체를 볼 때, `readlink()`는 링크에 저장된 대상 문자열을 볼 때 사용합니다.
세 API를 섞어 쓰면 심볼릭 링크를 무조건 두려워하지 않으면서도, 프로젝트 루트 밖으로 빠져나가는 경로를 더 일찍 잡아낼 수 있습니다.

운영 코드의 기준은 단순합니다.
링크를 허용할지 금지할지 먼저 정하고, 허용한다면 최종 실제 경로가 신뢰할 수 있는 루트 안에 있는지 확인합니다.
그리고 검증 이후의 실제 I/O에서도 `ENOENT`, `EACCES`, `EPERM`, `ELOOP` 같은 오류를 계속 처리해야 합니다.
이렇게 나누면 파일 경로 처리는 더 예측 가능해지고, 배포와 자동화 코드도 덜 불안해집니다.

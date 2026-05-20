---
layout: post
title: "Node.js mkdtempDisposable 가이드: 임시 디렉터리를 자동 정리하는 법"
date: 2026-05-20 20:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide
permalink: /development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html
  x_default: /development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, mkdtempdisposable, temporary-directory, cleanup, javascript]
description: "Node.js fs/promises의 mkdtempDisposable()로 임시 디렉터리를 만들고 using 블록 종료 시 자동 정리하는 방법을 정리했습니다. 업로드 검증, 테스트 격리, 에러 처리, permission model 주의점까지 실무 예제로 설명합니다."
---

임시 디렉터리는 개발 중 자주 등장합니다.
업로드 파일을 잠깐 풀어 검증하거나, 테스트 케이스마다 독립된 작업 공간을 만들거나, 이미지·문서 변환 결과를 배포 전에 임시로 저장할 때 필요합니다.
문제는 작업이 끝난 뒤 정리 코드가 빠지기 쉽다는 점입니다.
정상 흐름에서는 지워도, 중간에 예외가 발생하면 `/tmp`나 작업 디렉터리에 파일이 계속 쌓일 수 있습니다.

Node.js `node:fs/promises`의 `mkdtempDisposable()`은 임시 디렉터리 생성과 정리를 하나의 자원 생명주기로 묶어 줍니다.
`await using`과 함께 쓰면 블록이 끝날 때 자동으로 디렉터리를 제거할 수 있어, `try/finally` 정리 코드를 줄이고 누락 위험을 낮출 수 있습니다.
이 글에서는 Node.js `mkdtempDisposable()` 기본 사용법, `mkdtemp()`와의 차이, 업로드·테스트 예제, 에러 처리, Permission Model에서 확인할 점을 정리합니다.
자원 해제 문법이 낯설다면 먼저 [Node.js using Disposable과 AsyncDisposable 가이드: 자원 정리를 안전하게 자동화하는 법](/development/blog/seo/2026/05/02/nodejs-using-disposable-and-asyncdisposable-guide.html)을 함께 보면 좋습니다.

## mkdtempDisposable이 필요한 이유

### 임시 파일 정리는 실패 경로에서 자주 누락된다

가장 흔한 패턴은 `mkdtemp()`로 디렉터리를 만든 뒤 `finally`에서 `rm()`을 호출하는 방식입니다.
이 방식은 명확하지만, 함수가 길어질수록 정리 책임이 여러 곳으로 흩어지기 쉽습니다.

```js
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

export async function writePreviewFile(content) {
  const dir = await mkdtemp(join(tmpdir(), 'preview-'));

  try {
    const filePath = join(dir, 'preview.txt');
    await writeFile(filePath, content, 'utf8');
    return filePath;
  } finally {
    await rm(dir, { recursive: true, force: true });
  }
}
```

작은 함수에서는 충분히 괜찮습니다.
하지만 검증, 압축 해제, 변환, 업로드 같은 단계가 늘어나면 `finally`가 너무 많은 일을 떠안게 됩니다.
또 중간에 임시 디렉터리를 여러 개 만들면 어떤 경로를 어디서 지워야 하는지 추적하기 어려워집니다.

### 디렉터리 생명주기를 코드 블록으로 제한할 수 있다

`mkdtempDisposable()`은 임시 디렉터리를 disposable 자원으로 반환합니다.
`await using` 변수로 받으면 해당 스코프를 벗어날 때 정리 메서드가 호출됩니다.

```js
import { mkdtempDisposable, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

export async function buildPreview(content) {
  await using temp = await mkdtempDisposable(join(tmpdir(), 'preview-'));

  const filePath = join(temp.path, 'preview.txt');
  await writeFile(filePath, content, 'utf8');

  return await readPreviewMetadata(filePath);
}

async function readPreviewMetadata(filePath) {
  return {
    filePath,
    generatedAt: new Date().toISOString(),
  };
}
```

핵심은 `temp.path`가 블록 안에서만 유효한 작업 공간이라는 점입니다.
함수가 끝나면 디렉터리는 정리되므로, 반환값에 임시 파일 경로를 그대로 담아 외부에서 나중에 읽게 만들면 안 됩니다.
필요한 결과는 블록 안에서 읽어 메모리, 영구 저장소, 또는 별도 출력 경로로 옮긴 뒤 반환해야 합니다.

## 기본 사용법

### node:fs/promises에서 가져온다

`mkdtempDisposable()`은 `node:fs/promises` 모듈의 Promise 기반 API입니다.
접두사(prefix)를 넘기면 Node.js가 뒤에 임의 문자열을 붙여 고유한 디렉터리를 만듭니다.

```js
import { mkdtempDisposable } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

await using temp = await mkdtempDisposable(join(tmpdir(), 'job-'));

console.log(temp.path);
```

`tmpdir()`를 사용할 때는 `join(tmpdir(), 'prefix-')`처럼 경로 구분자를 직접 다루지 않는 편이 안전합니다.
운영체제마다 임시 디렉터리 위치와 경로 구분자가 다르기 때문입니다.
파일 경로 조합 규칙이 헷갈린다면 [Node.js path.matchesGlob 가이드: 파일 패턴을 안전하게 검사하는 법](/development/blog/seo/2026/05/17/nodejs-path-matchesglob-file-pattern-guide.html)에서 경로 처리 시 주의점을 함께 확인해도 좋습니다.

### 명시적으로 remove를 호출할 수도 있다

반환 객체에는 임시 디렉터리 경로와 정리 메서드가 들어 있습니다.
대부분은 `await using`에 맡기는 편이 좋지만, 특정 시점에 먼저 정리해야 한다면 `remove()`를 직접 호출할 수 있습니다.

```js
import { mkdtempDisposable, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

export async function createThenCleanup() {
  await using temp = await mkdtempDisposable(join(tmpdir(), 'manual-'));

  await writeFile(join(temp.path, 'result.json'), '{"ok":true}', 'utf8');
  await temp.remove();

  return 'cleaned';
}
```

직접 정리를 호출한 뒤에는 같은 경로를 다시 사용하지 않는 것이 원칙입니다.
코드 리뷰에서는 “이 임시 디렉터리는 어느 스코프까지 살아 있는가?”를 기준으로 보면 됩니다.
스코프가 짧을수록 버그가 줄고, 테스트도 단순해집니다.

## 업로드 검증 작업에 적용하기

### 압축 해제와 스캔을 임시 디렉터리 안에서 끝낸다

사용자가 올린 압축 파일을 검증하는 흐름을 생각해 보겠습니다.
압축을 임시 디렉터리에 풀고, 파일 목록을 검사한 뒤, 통과한 결과만 영구 저장소로 옮기는 구조가 안전합니다.

```js
import { cp, mkdtempDisposable } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

export async function inspectUpload(archivePath, outputDir) {
  await using temp = await mkdtempDisposable(join(tmpdir(), 'upload-'));

  await extractArchive(archivePath, temp.path);
  const report = await scanExtractedFiles(temp.path);

  if (!report.safe) {
    return { accepted: false, reason: report.reason };
  }

  await cp(temp.path, outputDir, {
    recursive: true,
    force: false,
    errorOnExist: true,
  });

  return { accepted: true, files: report.files };
}

async function extractArchive(archivePath, targetDir) {
  console.log('extract', archivePath, 'to', targetDir);
}

async function scanExtractedFiles(dir) {
  console.log('scan', dir);
  return { safe: true, files: [] };
}
```

영구 보존이 필요한 파일만 `cp()`로 별도 디렉터리에 복사하고, 나머지 임시 파일은 블록 종료 시 정리합니다.
재귀 복사 옵션은 실수하면 기존 파일을 덮어쓸 수 있으므로, 운영 코드에서는 목적에 맞게 `force`, `errorOnExist`, 대상 경로 정책을 명확히 정해야 합니다.
복사 옵션은 [Node.js fs.cp 가이드: 디렉터리 복사를 안전하게 처리하는 법](/development/blog/seo/2026/05/17/nodejs-fs-cp-recursive-copy-guide.html)을 참고하면 좋습니다.

### 임시 경로를 외부 API 응답에 노출하지 않는다

임시 디렉터리 경로는 서버 내부 구현 세부 사항입니다.
로그에는 디버깅 목적으로 남길 수 있지만, 사용자 응답이나 클라이언트가 저장하는 데이터에는 넣지 않는 편이 안전합니다.

```js
export async function handleUpload(filePath, permanentDir) {
  const result = await inspectUpload(filePath, permanentDir);

  if (!result.accepted) {
    return {
      status: 400,
      body: { message: '업로드 파일을 처리할 수 없습니다.' },
    };
  }

  return {
    status: 201,
    body: { fileCount: result.files.length },
  };
}
```

경로를 응답에 그대로 담으면 내부 디렉터리 구조가 드러날 수 있습니다.
또 블록 종료 후 사라지는 경로를 클라이언트가 재사용하려고 하면 장애 원인이 됩니다.
업로드 결과는 파일 개수, 검증 상태, 영구 리소스 ID처럼 오래 유지되는 값으로 표현하는 편이 좋습니다.

## 테스트 격리에 활용하기

### 테스트마다 독립된 작업 디렉터리를 만든다

파일 시스템을 다루는 테스트는 이전 테스트가 남긴 파일 때문에 흔들리기 쉽습니다.
`mkdtempDisposable()`을 테스트 본문 안에서 사용하면 케이스마다 깨끗한 작업 공간을 만들고 자동으로 치울 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { mkdtempDisposable, readFile, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

async function saveConfig(dir, config) {
  const filePath = join(dir, 'config.json');
  await writeFile(filePath, JSON.stringify(config), 'utf8');
  return filePath;
}

test('saveConfig writes json into isolated temp directory', async () => {
  await using temp = await mkdtempDisposable(join(tmpdir(), 'config-test-'));

  const filePath = await saveConfig(temp.path, { retries: 3 });
  const saved = JSON.parse(await readFile(filePath, 'utf8'));

  assert.deepEqual(saved, { retries: 3 });
});
```

이 패턴은 테스트 실패 시에도 정리가 실행된다는 장점이 있습니다.
물론 디버깅을 위해 실패한 테스트의 산출물을 일부러 남기고 싶을 때는 자동 삭제가 오히려 불편할 수 있습니다.
그럴 때는 `KEEP_TEMP=1` 같은 환경 변수를 두고, 디버그 모드에서만 별도 경로를 쓰도록 분리하는 편이 좋습니다.
Node 내장 테스트 러너 흐름은 [Node.js test runner 가이드: 내장 테스트로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)도 참고할 수 있습니다.

### 테스트 병렬 실행에서도 이름 충돌을 줄인다

수동으로 `tmp/test` 같은 고정 경로를 쓰면 병렬 테스트에서 충돌하기 쉽습니다.
`mkdtempDisposable()`은 접두사 뒤에 고유 문자열을 붙이므로 같은 테스트가 동시에 실행돼도 디렉터리 이름이 겹칠 가능성을 낮춥니다.

```js
import { mkdtempDisposable } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

export async function withTempWorkspace(fn) {
  await using temp = await mkdtempDisposable(join(tmpdir(), 'workspace-'));
  return await fn(temp.path);
}
```

테스트 헬퍼로 감싸 두면 실제 테스트는 “임시 작업 공간이 필요하다”는 의도만 표현하면 됩니다.
정리 로직을 각 테스트에 반복해서 쓰지 않아도 되어 유지보수성이 좋아집니다.

## 에러 처리와 주의점

### 정리 실패도 관찰 가능하게 남긴다

임시 디렉터리 삭제는 일반적으로 성공하지만, 파일 핸들이 열려 있거나 권한 문제가 있으면 실패할 수 있습니다.
자동 정리에만 의존하면 정리 실패 로그를 놓칠 수 있으므로, 운영 환경에서는 상위 에러 로깅 정책을 갖춰야 합니다.

```js
export async function runJobWithLogging(job) {
  try {
    await using temp = await createJobTempDirectory();
    return await job(temp.path);
  } catch (error) {
    console.error('job failed', {
      name: error.name,
      message: error.message,
    });
    throw error;
  }
}

async function createJobTempDirectory() {
  const { mkdtempDisposable } = await import('node:fs/promises');
  const { tmpdir } = await import('node:os');
  const { join } = await import('node:path');

  return await mkdtempDisposable(join(tmpdir(), 'job-'));
}
```

중요한 것은 원래 작업 실패와 정리 실패를 모두 관찰할 수 있게 만드는 것입니다.
로그에는 민감한 파일 내용이나 사용자 업로드 원문을 남기지 말고, 작업 ID와 오류 이름처럼 문제 추적에 필요한 최소 정보만 남기는 편이 안전합니다.

### 열린 파일 핸들을 먼저 닫는다

임시 디렉터리 안의 파일을 열어 둔 상태에서 삭제를 시도하면 운영체제별로 동작 차이가 생길 수 있습니다.
파일 핸들, 스트림, 압축 해제 프로세스는 임시 디렉터리 스코프 안에서 끝내고 닫아야 합니다.

```js
import { mkdtempDisposable, open } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

export async function writeWithFileHandle() {
  await using temp = await mkdtempDisposable(join(tmpdir(), 'handle-'));

  const file = await open(join(temp.path, 'data.txt'), 'w');
  try {
    await file.writeFile('hello', 'utf8');
  } finally {
    await file.close();
  }
}
```

정리 자동화는 “아무렇게나 열어 둔 리소스까지 모두 알아서 해결한다”는 뜻이 아닙니다.
각 자원은 자신에게 맞는 생명주기가 있고, 임시 디렉터리는 그중 파일 트리의 생명주기를 담당합니다.

## Permission Model에서 확인할 점

### 임시 디렉터리 생성·삭제 권한이 필요하다

Node.js Permission Model을 사용하는 환경에서는 임시 디렉터리를 만들 위치와 삭제할 위치에 파일 시스템 권한이 있어야 합니다.
권한이 좁게 잡힌 컨테이너나 CI에서는 `/tmp` 쓰기가 막혀 있을 수 있으므로, 작업 전용 임시 루트를 명시하는 편이 더 예측 가능합니다.

```js
import { mkdtempDisposable } from 'node:fs/promises';
import { join } from 'node:path';

const tempRoot = process.env.APP_TEMP_DIR ?? '/tmp';

export async function runInAppTemp(fn) {
  await using temp = await mkdtempDisposable(join(tempRoot, 'app-'));
  return await fn(temp.path);
}
```

권한 제한을 적용한다면 실행 옵션, 컨테이너 볼륨, 애플리케이션 설정이 서로 맞아야 합니다.
Permission Model의 큰 그림은 [Node.js Permission Model 가이드: 런타임 접근 제어로 피해 범위 줄이기](/development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html)를 참고하면 좋습니다.

### 임시 디렉터리 루트는 설정으로 분리한다

운영 서버, 로컬 개발, CI는 임시 파일을 둘 위치가 다를 수 있습니다.
코드에 `/tmp`를 고정하기보다 `APP_TEMP_DIR` 같은 설정으로 분리하면 배포 환경별 조정이 쉬워집니다.
다만 설정값을 그대로 신뢰하지 말고, 서비스가 소유한 안전한 경로인지 확인해야 합니다.

```js
import { mkdir, mkdtempDisposable } from 'node:fs/promises';
import { join, resolve } from 'node:path';

export async function createSafeTemp(prefix = 'work-') {
  const root = resolve(process.env.APP_TEMP_DIR ?? './tmp');
  await mkdir(root, { recursive: true });

  return await mkdtempDisposable(join(root, prefix));
}
```

상대 경로를 허용한다면 `resolve()`로 실제 기준 위치를 명확히 하고, 배포 시 작업 디렉터리가 어디인지 문서화해야 합니다.
임시 파일이 애플리케이션 루트 아래에 생긴다면 `.gitignore`, 백업 정책, 정적 파일 서빙 경로도 함께 확인해야 합니다.

## 실무 체크리스트

### 적용하기 좋은 경우

`mkdtempDisposable()`은 임시 디렉터리의 생명주기가 짧고 명확할 때 가장 잘 맞습니다.
예를 들면 다음과 같습니다.

- 업로드 파일을 잠깐 풀어 검사한 뒤 통과한 결과만 저장한다.
- 테스트 케이스마다 독립된 파일 시스템 작업 공간이 필요하다.
- 이미지, PDF, CSV 변환 중간 산출물을 블록 안에서만 사용한다.
- 실패해도 남기면 안 되는 캐시성 파일을 다룬다.

반대로 작업 결과를 디렉터리째 오래 보존해야 한다면 임시 디렉터리보다 명시적인 영구 저장 경로가 맞습니다.
자동 정리되는 경로를 나중에 다른 작업이 참조하게 만들면 장애가 생깁니다.

### 코드 리뷰에서 볼 포인트

코드 리뷰에서는 다음 질문을 던지면 좋습니다.

- 임시 디렉터리 경로가 블록 밖으로 새어 나가지 않는가?
- 필요한 결과는 삭제 전에 영구 저장소나 메모리로 옮겼는가?
- 파일 핸들, 스트림, 외부 프로세스가 정리 전에 닫히는가?
- 임시 루트 경로가 환경별로 설정 가능하고 권한 정책과 맞는가?
- 로그에 사용자 파일 내용, 토큰, 개인정보가 남지 않는가?

특히 업로드 처리 코드에서는 파일명과 경로를 그대로 신뢰하지 않아야 합니다.
임시 디렉터리는 격리 수단 중 하나일 뿐이고, 파일 확장자 검증, 크기 제한, 콘텐츠 검사, 영구 저장소 권한 분리도 함께 필요합니다.
파일을 Blob으로 다루는 업로드 흐름은 [Node.js fs.openAsBlob 가이드: 파일을 Blob으로 다뤄 업로드하는 법](/development/blog/seo/2026/05/06/nodejs-fs-openasblob-file-to-blob-upload-guide.html)과 연결해서 설계할 수 있습니다.

## FAQ

### mkdtempDisposable은 mkdtemp를 완전히 대체하나요?

완전한 대체라기보다 “자동 정리가 필요한 짧은 생명주기 임시 디렉터리”에 더 잘 맞는 API입니다.
임시 디렉터리를 만든 뒤 다른 프로세스나 긴 백그라운드 작업이 계속 사용해야 한다면 `mkdtemp()`와 명시적 정리 정책이 더 적합할 수 있습니다.

### await using을 쓰지 않고도 사용할 수 있나요?

반환 객체의 `remove()`를 직접 호출할 수 있습니다.
다만 직접 호출 방식은 정리 누락 위험이 다시 커지므로, 가능하면 `await using`으로 스코프 기반 정리를 표현하는 편이 좋습니다.
팀 코드 스타일에서 `using` 문법 지원 여부와 Node.js 버전도 함께 확인해야 합니다.

### 임시 디렉터리 안의 파일을 반환해도 되나요?

경로 문자열을 그대로 반환하는 것은 피해야 합니다.
블록이 끝나면 디렉터리가 삭제될 수 있기 때문입니다.
필요한 데이터는 블록 안에서 읽어 반환하거나, 영구 저장소로 옮긴 뒤 영구 리소스 ID 또는 URL을 반환하는 구조가 안전합니다.

### 정리 실패가 발생하면 어떻게 해야 하나요?

먼저 파일 핸들, 스트림, 외부 프로세스가 모두 닫혔는지 확인해야 합니다.
그다음 권한, 백신·보안 에이전트, 컨테이너 볼륨 정책처럼 운영체제와 배포 환경의 영향을 점검합니다.
정리 실패는 디스크 사용량 문제로 이어질 수 있으므로 작업 ID 중심의 로그와 주기적인 임시 파일 모니터링을 함께 두는 것이 좋습니다.

## 마무리

`mkdtempDisposable()`은 임시 디렉터리를 “만들고 잊어버리는 경로”가 아니라 “스코프가 끝나면 정리되는 자원”으로 다루게 해 줍니다.
업로드 검증, 테스트 격리, 변환 중간 산출물처럼 짧게 쓰고 반드시 지워야 하는 작업에 잘 맞습니다.

실무에서는 `await using`으로 생명주기를 좁히고, 임시 경로를 외부 응답에 노출하지 않으며, 필요한 결과는 삭제 전에 영구 저장소로 옮기는 원칙을 지키면 됩니다.
여기에 Permission Model, 파일 핸들 닫기, 로그 마스킹까지 함께 챙기면 임시 파일로 인한 디스크 누수와 정보 노출 위험을 훨씬 줄일 수 있습니다.

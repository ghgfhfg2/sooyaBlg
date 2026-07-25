---
layout: post
title: "Node.js fs.promises.opendir 가이드: 대량 파일 디렉터리 순회를 안전하게 처리하는 법"
date: 2026-07-26 08:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-opendir-directory-walk-guide
permalink: /development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html
alternates:
  ko: /development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html
  x_default: /development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, opendir, directory-walk, file-system, backend, automation]
description: "Node.js fs.promises.opendir()로 대량 파일 디렉터리를 메모리 부담 없이 순회하는 방법을 정리했습니다. Dirent 기반 재귀 탐색, AbortSignal 취소, glob과의 선택 기준, 테스트와 보안 체크리스트까지 실무 예제로 설명합니다."
---

Node.js 자동화 스크립트와 백엔드 작업에서는 디렉터리 안의 파일을 훑어야 하는 일이 자주 생깁니다.
빌드 산출물 점검, 로그 파일 정리, 업로드 파일 검증, 문서 링크 검사, 테스트 fixture 수집처럼 파일 목록을 만드는 작업은 작아 보이지만 운영 코드에서는 쉽게 커집니다.
파일이 수천 개를 넘거나 하위 디렉터리가 깊어지면 단순한 `readdir()` 한 번으로 끝내기 어렵고, 전체 목록을 메모리에 올리는 방식도 부담이 됩니다.

이 글에서는 Node.js `fs.promises.opendir()`로 디렉터리를 비동기 이터레이션하며 안전하게 순회하는 패턴을 정리합니다.
핵심은 파일 목록을 한꺼번에 만들기보다 **Dir 객체를 열고, 필요한 항목을 하나씩 처리하며, 취소와 권한 경계를 함께 설계하는 것**입니다.
[Node.js fs.promises.glob 가이드](/development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html), [Node.js FileHandle.readLines 가이드](/development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html), [Node.js Permission Model 가이드](/development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html)와 함께 보면 파일 검색, 대용량 처리, 런타임 권한을 한 흐름으로 이해하기 좋습니다.

## fs.promises.opendir가 필요한 이유

### H3. 대량 파일 작업은 목록 생성부터 병목이 된다

가장 단순한 디렉터리 읽기 코드는 `readdir()`입니다.

```js
import { readdir } from 'node:fs/promises';

const entries = await readdir('logs');

for (const entry of entries) {
  console.log(entry);
}
```

작은 디렉터리라면 이 방식이 충분합니다.
하지만 다음 조건이 붙으면 이야기가 달라집니다.

- 하위 디렉터리를 재귀적으로 순회해야 한다.
- 파일 개수가 많아 전체 배열을 만들고 싶지 않다.
- 파일을 찾는 즉시 처리하고 싶다.
- 중간에 timeout이나 사용자 취소를 반영해야 한다.
- 권한 오류가 나는 하위 경로를 건너뛰어야 한다.

`opendir()`는 디렉터리를 열고 `Dir` 객체를 반환합니다.
이 객체는 비동기 이터레이터로 사용할 수 있어서 항목을 하나씩 읽고 처리하는 흐름을 만들 수 있습니다.

```js
import { opendir } from 'node:fs/promises';

const dir = await opendir('logs');

for await (const dirent of dir) {
  console.log(dirent.name, dirent.isFile());
}
```

`Dirent`는 파일명뿐 아니라 파일인지, 디렉터리인지, 심볼릭 링크인지 같은 기본 정보를 제공합니다.
따라서 매 항목마다 `stat()`을 호출하지 않아도 되는 경우가 많고, 파일 시스템 접근 횟수를 줄일 수 있습니다.

### H3. glob은 검색에 좋고 opendir는 제어에 좋다

Node.js에는 `fs.promises.glob()`도 있습니다.
패턴에 맞는 파일을 찾는 목적이라면 `glob()`이 더 간단합니다.

```js
import { glob } from 'node:fs/promises';

for await (const file of glob('src/**/*.test.js')) {
  console.log(file);
}
```

반면 `opendir()`는 순회 정책을 직접 제어해야 할 때 더 잘 맞습니다.

- 디렉터리마다 다른 skip 규칙을 적용한다.
- 파일을 찾는 즉시 검증하거나 업로드한다.
- 일정 개수 이상 찾으면 순회를 멈춘다.
- 권한 오류, 심볼릭 링크, 숨김 파일 정책을 직접 나눈다.
- 순회 중 취소 신호를 확인한다.

즉 `glob()`은 "이 패턴에 맞는 파일을 찾아줘"에 가깝고, `opendir()`는 "내 규칙대로 파일 트리를 걸어갈게"에 가깝습니다.
블로그 링크 검사나 배포 전 산출물 점검처럼 조건이 많은 자동화에서는 `opendir()`가 더 읽기 쉬운 구조가 될 수 있습니다.

## opendir 기본 사용법

### H3. Dirent로 파일과 디렉터리를 구분한다

`opendir()`로 받은 `Dir` 객체는 `for await...of`로 순회할 수 있습니다.
각 항목은 `Dirent`입니다.

```js
import { opendir } from 'node:fs/promises';

export async function listTopLevelFiles(directory) {
  const files = [];
  const dir = await opendir(directory);

  for await (const entry of dir) {
    if (entry.isFile()) {
      files.push(entry.name);
    }
  }

  return files;
}
```

여기서 `entry.name`은 전체 경로가 아니라 디렉터리 안의 이름입니다.
실제 파일 경로가 필요하면 기준 디렉터리와 조합해야 합니다.

```js
import path from 'node:path';
import { opendir } from 'node:fs/promises';

export async function listFiles(directory) {
  const files = [];
  const dir = await opendir(directory);

  for await (const entry of dir) {
    if (!entry.isFile()) {
      continue;
    }

    files.push(path.join(directory, entry.name));
  }

  return files;
}
```

이때 사용자 입력이 경로 일부로 들어간다면 반드시 허용된 루트 아래에 머무르는지 확인해야 합니다.
파일 순회는 단순 조회처럼 보여도 실제로는 서버 파일 시스템을 탐색하는 작업이기 때문입니다.

### H3. try/finally 없이도 비동기 이터레이션이 닫힘을 돕는다

`for await...of`로 `Dir` 객체를 끝까지 순회하면 Node.js가 디렉터리 핸들을 닫습니다.
그래도 운영 코드에서는 오류 경로를 신경 써야 합니다.
특히 직접 `dir.read()`를 호출하는 방식으로 바꾸면 `close()` 책임이 더 명확해집니다.

일반적인 순회에는 아래처럼 `for await...of`를 우선 사용하는 편이 읽기 쉽습니다.

```js
import { opendir } from 'node:fs/promises';

export async function countEntries(directory) {
  let count = 0;

  for await (const entry of await opendir(directory)) {
    if (entry.isFile() || entry.isDirectory()) {
      count += 1;
    }
  }

  return count;
}
```

파일 핸들을 직접 다루는 작업에서는 정리 책임이 더 중요합니다.
대량 로그를 줄 단위로 읽는 경우에는 [Node.js FileHandle.readLines 가이드](/development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html)처럼 파일 단위 자원 수명도 함께 설계해야 합니다.

## 재귀 디렉터리 순회 패턴

### H3. Async generator로 찾은 파일을 즉시 yield한다

대량 파일 순회에서는 전체 배열을 반환하기보다 async generator로 파일을 하나씩 내보내는 패턴이 유용합니다.

```js
import path from 'node:path';
import { opendir } from 'node:fs/promises';

export async function* walkFiles(root) {
  const dir = await opendir(root);

  for await (const entry of dir) {
    const fullPath = path.join(root, entry.name);

    if (entry.isDirectory()) {
      yield* walkFiles(fullPath);
      continue;
    }

    if (entry.isFile()) {
      yield fullPath;
    }
  }
}
```

사용하는 쪽은 파일을 찾는 즉시 처리할 수 있습니다.

```js
for await (const filePath of walkFiles('content')) {
  console.log(filePath);
}
```

이 구조의 장점은 소비자가 원하는 시점에 멈출 수 있다는 것입니다.
예를 들어 첫 번째 오류 파일만 찾으면 되는 검사라면 전체 트리를 끝까지 돌 필요가 없습니다.

```js
for await (const filePath of walkFiles('content')) {
  if (filePath.endsWith('.broken.md')) {
    console.error('broken file:', filePath);
    break;
  }
}
```

전체 목록을 만들어 `filter()`하는 코드보다 조금 길지만, 큰 디렉터리에서는 메모리 사용량과 제어 흐름이 더 안정적입니다.

### H3. 숨김 디렉터리와 빌드 산출물은 명시적으로 건너뛴다

재귀 순회에서 가장 흔한 실수는 `node_modules`, `.git`, 빌드 결과물까지 모두 훑는 것입니다.
작은 프로젝트에서는 티가 나지 않지만 CI나 배포 서버에서는 순회 시간이 갑자기 길어질 수 있습니다.

skip 규칙을 함수로 분리하면 테스트하기 쉽습니다.

```js
const SKIP_DIRECTORIES = new Set([
  '.git',
  'node_modules',
  '.next',
  'dist',
  'coverage'
]);

function shouldSkipDirectory(name) {
  return SKIP_DIRECTORIES.has(name);
}
```

순회 함수에 적용하면 다음과 같습니다.

```js
import path from 'node:path';
import { opendir } from 'node:fs/promises';

const SKIP_DIRECTORIES = new Set(['.git', 'node_modules', 'dist', 'coverage']);

export async function* walkProjectFiles(root) {
  for await (const entry of await opendir(root)) {
    const fullPath = path.join(root, entry.name);

    if (entry.isDirectory()) {
      if (SKIP_DIRECTORIES.has(entry.name)) {
        continue;
      }

      yield* walkProjectFiles(fullPath);
      continue;
    }

    if (entry.isFile()) {
      yield fullPath;
    }
  }
}
```

문서 사이트의 Markdown 파일만 검사한다면 확장자 조건도 함께 둘 수 있습니다.

```js
for await (const filePath of walkProjectFiles(process.cwd())) {
  if (!filePath.endsWith('.md')) {
    continue;
  }

  console.log('check markdown:', filePath);
}
```

단순 패턴 매칭이 목적이라면 [Node.js fs.promises.glob 가이드](/development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html)의 `glob('**/*.md')`가 더 짧습니다.
하지만 skip 규칙, 파일 내용 검사, 중간 종료 조건이 섞이면 `opendir()` 기반 순회가 더 명시적입니다.

## 취소 가능한 디렉터리 순회 만들기

### H3. AbortSignal을 루프 안에서 직접 확인한다

`opendir()` 자체가 긴 계산을 수행하는 API는 아니지만, 재귀 순회 전체는 오래 걸릴 수 있습니다.
따라서 호출자가 넘긴 `AbortSignal`을 루프 안에서 확인하는 편이 좋습니다.

```js
import path from 'node:path';
import { opendir } from 'node:fs/promises';

export async function* walkFiles(root, { signal } = {}) {
  signal?.throwIfAborted();

  for await (const entry of await opendir(root)) {
    signal?.throwIfAborted();

    const fullPath = path.join(root, entry.name);

    if (entry.isDirectory()) {
      yield* walkFiles(fullPath, { signal });
    } else if (entry.isFile()) {
      yield fullPath;
    }
  }
}
```

호출부에서는 timeout과 외부 취소를 함께 묶을 수 있습니다.

```js
const signal = AbortSignal.timeout(5_000);

for await (const filePath of walkFiles('content', { signal })) {
  console.log(filePath);
}
```

여러 취소 조건을 합쳐야 한다면 [Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)에서 다룬 방식처럼 호출자 signal과 timeout signal을 하나로 묶을 수 있습니다.
파일 순회도 네트워크 요청이나 외부 명령처럼 작업 상한을 갖는 것이 운영에서 안전합니다.

### H3. 권한 오류는 정책에 따라 실패 또는 skip으로 나눈다

디렉터리 순회 중에는 `EACCES`, `ENOENT`, `ENOTDIR` 같은 오류가 날 수 있습니다.
예를 들어 순회하는 동안 다른 프로세스가 파일을 지웠거나, 특정 하위 디렉터리에 읽기 권한이 없을 수 있습니다.

운영 자동화에서는 오류를 모두 무시하면 문제를 숨기게 됩니다.
반대로 모든 오류를 즉시 실패로 처리하면 부분 검사도 못 하고 멈출 수 있습니다.
그래서 "이 작업에서 허용 가능한 오류"를 정해야 합니다.

```js
function canSkipDirectoryError(error) {
  return error?.code === 'EACCES' || error?.code === 'ENOENT';
}
```

순회 함수에 적용하면 다음처럼 만들 수 있습니다.

```js
import path from 'node:path';
import { opendir } from 'node:fs/promises';

export async function* walkFiles(root, { signal, onSkip } = {}) {
  signal?.throwIfAborted();

  let dir;

  try {
    dir = await opendir(root);
  } catch (error) {
    if (error.code === 'EACCES' || error.code === 'ENOENT') {
      onSkip?.({ path: root, error });
      return;
    }

    throw error;
  }

  for await (const entry of dir) {
    signal?.throwIfAborted();

    const fullPath = path.join(root, entry.name);

    if (entry.isDirectory()) {
      yield* walkFiles(fullPath, { signal, onSkip });
      continue;
    }

    if (entry.isFile()) {
      yield fullPath;
    }
  }
}
```

skip 이벤트를 로그나 메트릭으로 남기면 나중에 "검사 대상이 줄어든 이유"를 추적할 수 있습니다.
단, 로그에는 사용자 홈 경로나 내부 서버 경로처럼 민감할 수 있는 정보를 그대로 남기지 않도록 주의해야 합니다.

## 실무 예제: Markdown 링크 검사 대상 수집

### H3. 순회와 파일 검사를 분리한다

블로그나 문서 저장소에서는 Markdown 파일만 모아 링크 검사, front matter 점검, 금칙어 검사를 수행할 수 있습니다.
이때 디렉터리 순회와 파일 내용 검사를 한 함수에 모두 넣으면 테스트하기 어려워집니다.

먼저 순회 함수는 파일 경로만 담당하게 둡니다.

```js
export async function* walkMarkdownFiles(root, options = {}) {
  for await (const filePath of walkProjectFiles(root, options)) {
    if (filePath.endsWith('.md')) {
      yield filePath;
    }
  }
}
```

검사 함수는 파일 하나만 다룹니다.

```js
import { readFile } from 'node:fs/promises';

export async function inspectMarkdown(filePath) {
  const source = await readFile(filePath, 'utf8');

  return {
    filePath,
    hasTitle: /^title:/m.test(source),
    linkCount: (source.match(/\]\(/g) ?? []).length
  };
}
```

마지막으로 실행 흐름에서 둘을 조합합니다.

```js
const signal = AbortSignal.timeout(10_000);
const results = [];

for await (const filePath of walkMarkdownFiles('content', { signal })) {
  results.push(await inspectMarkdown(filePath));
}

console.table(results);
```

이 구조는 작지만 유지보수하기 쉽습니다.
순회 정책을 바꿔도 Markdown 검사 로직은 그대로 두고, 검사 로직을 늘려도 디렉터리 탐색 코드는 흔들리지 않습니다.

### H3. 동시에 너무 많은 파일을 읽지 않는다

파일을 하나씩 읽으면 안정적이지만 느릴 수 있습니다.
그렇다고 모든 파일을 `Promise.all()`로 한 번에 읽으면 파일 디스크립터나 메모리 사용량이 튈 수 있습니다.
중간 크기의 프로젝트라면 작은 concurrency limit을 두는 편이 안전합니다.

```js
export async function mapWithConcurrency(items, limit, mapper) {
  const results = [];
  const executing = new Set();

  for await (const item of items) {
    const task = Promise.resolve()
      .then(() => mapper(item))
      .finally(() => executing.delete(task));

    results.push(task);
    executing.add(task);

    if (executing.size >= limit) {
      await Promise.race(executing);
    }
  }

  return Promise.all(results);
}
```

사용 예시는 다음과 같습니다.

```js
const reports = await mapWithConcurrency(
  walkMarkdownFiles('content', { signal: AbortSignal.timeout(10_000) }),
  8,
  inspectMarkdown
);
```

동시성 제한은 정답이 하나로 정해져 있지 않습니다.
로컬 SSD, CI 환경, 네트워크 파일 시스템, 파일 크기에 따라 적절한 값이 달라집니다.
중요한 것은 제한이 아예 없는 상태를 기본값으로 두지 않는 것입니다.

## 보안과 운영 체크리스트

### H3. 순회 루트는 고정하고 입력 경로는 검증한다

사용자 입력을 받아 특정 디렉터리를 순회해야 한다면 루트를 고정하고, 입력 경로가 그 아래에 있는지 확인해야 합니다.

```js
import path from 'node:path';

function resolveInsideRoot(root, requestedPath) {
  const rootPath = path.resolve(root);
  const targetPath = path.resolve(rootPath, requestedPath);

  if (!targetPath.startsWith(rootPath + path.sep) && targetPath !== rootPath) {
    throw new Error('Path is outside of allowed root');
  }

  return targetPath;
}
```

이 검사는 완벽한 보안 모델을 대신하지 않습니다.
심볼릭 링크, 권한, 컨테이너 마운트, 런타임 사용자 권한도 함께 봐야 합니다.
Node.js Permission Model을 적용할 수 있는 환경이라면 읽기 가능한 경로를 좁혀 실행하는 것도 좋은 방어선입니다.

```bash
node --permission --allow-fs-read=./content ./scripts/check-content.mjs
```

권한 경계는 자동화 스크립트일수록 더 중요합니다.
스크립트가 실수로 프로젝트 루트 밖을 훑거나, 홈 디렉터리의 민감한 파일명을 로그로 남기는 일을 줄일 수 있습니다.

### H3. 심볼릭 링크 정책을 미리 정한다

`Dirent`에는 `isSymbolicLink()`가 있습니다.
재귀 순회에서 심볼릭 링크를 따라갈지 말지는 반드시 명시해야 합니다.

대부분의 검사 스크립트에서는 심볼릭 링크를 기본적으로 따라가지 않는 편이 안전합니다.
링크를 따라가면 프로젝트 밖으로 나가거나, 순환 구조를 만들거나, 같은 파일을 여러 번 검사할 수 있습니다.

```js
if (entry.isSymbolicLink()) {
  continue;
}
```

심볼릭 링크를 따라가야 하는 도구라면 방문한 실제 경로를 기록하고 순환을 끊는 장치가 필요합니다.
하지만 일반적인 블로그 문서 검사, 산출물 검증, fixture 수집에서는 skip 정책이 더 단순하고 예측 가능합니다.

## 테스트 전략

### H3. 임시 디렉터리 fixture로 순회 규칙을 검증한다

디렉터리 순회 함수는 실제 프로젝트 구조에 의존하면 테스트가 쉽게 깨집니다.
테스트 안에서 임시 디렉터리를 만들고 필요한 파일만 배치하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { mkdtemp, mkdir, writeFile, rm } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

test('walkMarkdownFiles returns markdown files only', async () => {
  const root = await mkdtemp(join(tmpdir(), 'walk-md-'));

  try {
    await mkdir(join(root, 'docs'));
    await writeFile(join(root, 'README.md'), '# Home');
    await writeFile(join(root, 'docs', 'guide.md'), '# Guide');
    await writeFile(join(root, 'docs', 'image.png'), 'fake');

    const files = [];

    for await (const file of walkMarkdownFiles(root)) {
      files.push(file.replace(root, ''));
    }

    assert.deepEqual(files.sort(), [
      '/README.md',
      '/docs/guide.md'
    ]);
  } finally {
    await rm(root, { recursive: true, force: true });
  }
});
```

`mkdtempDisposable()`을 사용할 수 있는 환경이라면 정리 코드를 더 간단히 만들 수 있습니다.
[Node.js fsPromises.mkdtempDisposable 가이드](/development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html)를 함께 보면 테스트 fixture 수명 주기를 더 명확히 잡을 수 있습니다.

### H3. 취소와 skip 경로도 테스트한다

파일 순회 테스트는 성공 케이스만 검증하면 부족합니다.
다음 경로를 함께 확인해야 운영에서 덜 흔들립니다.

- 숨김 디렉터리나 `node_modules`를 건너뛰는가?
- `AbortSignal`이 중단되면 빠르게 예외가 나는가?
- 권한 오류 또는 사라진 디렉터리를 정책대로 처리하는가?
- 심볼릭 링크를 의도대로 skip하는가?
- 파일 순서가 달라도 결과 검증이 안정적인가?

특히 파일 시스템 순서는 플랫폼과 상황에 따라 달라질 수 있습니다.
테스트에서는 결과를 정렬한 뒤 비교하거나, 순서가 중요한 로직이라면 코드에서 명시적으로 정렬해야 합니다.

## opendir 선택 기준

### H3. opendir가 잘 맞는 경우

`fs.promises.opendir()`는 다음 상황에 잘 맞습니다.

- 파일을 찾는 즉시 처리해야 한다.
- 전체 파일 목록을 메모리에 올리고 싶지 않다.
- 재귀 순회 중 skip, 취소, 오류 정책을 직접 제어해야 한다.
- `Dirent` 정보로 파일과 디렉터리를 빠르게 구분하고 싶다.
- glob 패턴보다 절차적인 순회 규칙이 더 읽기 쉽다.

반대로 단순히 `**/*.md` 같은 패턴으로 파일을 찾는 작업이라면 `fs.promises.glob()`가 더 간결합니다.
상위 몇 개 항목만 읽는다면 `readdir()`도 충분합니다.
도구 선택의 기준은 "가장 강력한 API"가 아니라 "현재 작업의 제어 수준에 맞는 API"입니다.

### H3. 운영 코드에 넣기 전 체크리스트

발행 전 또는 배포 전 자동화에 `opendir()` 순회를 넣는다면 아래 항목을 확인하세요.

- 순회 루트가 명확하게 고정되어 있는가?
- `node_modules`, `.git`, 빌드 산출물 skip 규칙이 있는가?
- 심볼릭 링크 정책을 정했는가?
- `AbortSignal`이나 timeout으로 중단 가능한가?
- 권한 오류를 실패로 볼지 skip으로 볼지 정했는가?
- 로그에 민감한 절대 경로나 사용자 정보를 남기지 않는가?
- 테스트 fixture로 재귀, skip, 취소 경로를 검증했는가?

이 기준을 먼저 정하면 파일 순회 코드는 단순한 유틸이 아니라 운영 가능한 작은 시스템 경계가 됩니다.

## FAQ

### fs.promises.opendir와 readdir 중 무엇을 먼저 써야 하나요?

작은 디렉터리를 한 번 읽는 정도라면 `readdir()`이 더 단순합니다.
대량 파일, 재귀 탐색, 중간 처리, 취소 정책이 필요하다면 `opendir()`가 더 적합합니다.

### fs.promises.glob가 있는데 opendir가 필요한 이유는 무엇인가요?

`glob()`은 패턴 기반 검색에 좋습니다.
`opendir()`는 순회 중에 직접 판단하고, 건너뛰고, 멈추고, 오류를 분류해야 할 때 좋습니다.
검색 조건이 단순하면 `glob()`, 순회 정책이 복잡하면 `opendir()`를 선택하면 됩니다.

### 파일 순회 결과 순서는 믿어도 되나요?

파일 시스템 순서는 플랫폼과 상태에 따라 달라질 수 있습니다.
순서가 중요한 결과라면 코드에서 명시적으로 정렬해야 합니다.
테스트에서도 결과 배열을 정렬한 뒤 비교하는 편이 안정적입니다.

### opendir로 심볼릭 링크를 따라가도 되나요?

가능은 하지만 기본값으로 추천하지 않습니다.
프로젝트 밖 경로로 빠지거나 순환 구조를 만들 수 있기 때문입니다.
정말 따라가야 한다면 실제 경로 기준 방문 집합을 기록하고, 허용 루트 밖으로 나가지 않는 검사를 추가해야 합니다.

## 마무리

Node.js `fs.promises.opendir()`는 파일 시스템을 한 번에 배열로 읽는 대신, 디렉터리를 열고 항목을 하나씩 처리하게 해 주는 실용적인 API입니다.
대량 파일 순회, 문서 검사, 빌드 산출물 점검처럼 조건이 많은 자동화에서는 `readdir()`보다 제어하기 쉽고, 단순 glob 검색보다 운영 정책을 드러내기 좋습니다.

처음부터 거창한 파일 인덱서를 만들 필요는 없습니다.
루트를 고정하고, skip 규칙을 두고, `AbortSignal`을 확인하고, 심볼릭 링크와 권한 오류 정책을 명시하는 것만으로도 파일 순회 코드는 훨씬 안전해집니다.
관련해서 패턴 기반 파일 검색은 [Node.js fs.promises.glob 가이드](/development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html), 임시 테스트 디렉터리는 [Node.js fsPromises.mkdtempDisposable 가이드](/development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html), 런타임 파일 접근 제한은 [Node.js Permission Model 가이드](/development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html)를 이어서 참고하면 좋습니다.

---
layout: post
title: "Node.js fs.promises.glob 가이드: 파일 검색을 의존성 없이 처리하는 법"
date: 2026-05-28 20:00:00 +0900
lang: ko
translation_key: nodejs-fs-promises-glob-file-search-guide
permalink: /development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html
alternates:
  ko: /development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html
  x_default: /development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, glob, file-search, automation, javascript, cli]
description: "Node.js fs/promises의 glob()으로 파일 목록을 의존성 없이 찾는 방법을 정리했습니다. AsyncIterator 사용법, cwd와 exclude 옵션, withFileTypes, 테스트 자동화와 운영 스크립트 적용 기준까지 실무 예제로 설명합니다."
---

빌드 스크립트, 문서 생성기, 테스트 보조 도구를 만들다 보면 특정 패턴에 맞는 파일 목록이 필요합니다.
예전에는 이런 작업에 `glob`, `fast-glob`, `globby` 같은 패키지를 바로 추가하는 경우가 많았습니다.
하지만 단순한 파일 검색이라면 Node.js의 `fs/promises`에서 제공하는 `glob()`만으로도 충분히 읽기 좋은 구조를 만들 수 있습니다.

`fsPromises.glob()`은 패턴에 맞는 파일 경로를 비동기 반복자로 순회하게 해 줍니다.
한 번에 모든 파일을 배열로 모으지 않고 `for await...of`로 처리할 수 있으므로, 내부 자동화나 작은 CLI에서 의존성을 줄이고 메모리 사용도 예측하기 쉽습니다.
이 글에서는 Node.js `fs/promises.glob()` 기본 사용법, `cwd`와 `exclude` 옵션, `withFileTypes` 활용, 테스트와 배포 스크립트에서 안전하게 쓰는 기준을 정리합니다.
경로 하나가 패턴과 맞는지만 확인하고 싶다면 [Node.js path.matchesGlob 가이드](/development/blog/seo/2026/05/17/nodejs-path-matchesglob-file-pattern-guide.html)를 먼저 보면 차이를 이해하기 쉽습니다.

## fs.promises.glob가 해결하는 문제

### 파일 목록 수집 코드를 표준 API로 단순화한다

작은 자동화 스크립트에서 외부 패키지 하나를 추가하는 일은 생각보다 비용이 큽니다.
버전 업데이트, 보안 알림, ESM과 CommonJS 호환성, 옵션 차이까지 따라옵니다.
프로젝트 전체에서 이미 파일 검색 라이브러리를 쓰고 있다면 그대로 유지할 수 있지만, 새 스크립트 하나를 위해 의존성을 늘리는 것은 부담스러울 수 있습니다.

`fsPromises.glob()`은 이런 상황에서 좋은 기본 선택지가 됩니다.

```js
import { glob } from 'node:fs/promises';

for await (const file of glob('src/**/*.js')) {
  console.log(file);
}
```

반환값은 배열이 아니라 비동기 반복자입니다.
그래서 파일이 많을 때도 순회하면서 하나씩 처리하는 흐름을 만들기 쉽습니다.
파일을 모두 모은 뒤 한 번에 처리해야 하는 작업이 아니라면 이 방식이 더 자연스럽습니다.

### 패턴 매칭과 파일 탐색의 책임을 분리한다

`path.matchesGlob()`는 문자열 하나가 glob 패턴과 맞는지 검사합니다.
반면 `fsPromises.glob()`은 실제 파일 시스템을 탐색해 패턴에 맞는 항목을 찾아 줍니다.
두 API는 이름이 비슷해 보여도 역할이 다릅니다.

```js
import { glob } from 'node:fs/promises';
import path from 'node:path';

console.log(path.matchesGlob('src/app.test.js', '**/*.test.js'));

for await (const file of glob('src/**/*.test.js')) {
  console.log(file);
}
```

첫 번째 코드는 이미 알고 있는 경로를 검사합니다.
두 번째 코드는 파일 시스템에서 경로를 찾아냅니다.
실무에서는 “찾기”와 “검사”를 섞지 않는 편이 유지보수에 좋습니다.
예를 들어 배포 대상 파일 목록을 만들 때는 `glob()`으로 후보를 찾고, 금지 경로나 예외 규칙은 별도 함수에서 검사하게 만들 수 있습니다.

## 기본 사용법

### for await로 파일을 하나씩 처리한다

가장 단순한 형태는 패턴 하나를 넘기고 비동기 반복자로 순회하는 방식입니다.

```js
import { glob } from 'node:fs/promises';

for await (const markdownFile of glob('docs/**/*.md')) {
  console.log(markdownFile);
}
```

이 코드는 현재 작업 디렉터리를 기준으로 `docs` 아래의 Markdown 파일을 찾습니다.
Node.js 스크립트에서 현재 작업 디렉터리는 실행 위치에 따라 달라질 수 있으므로, 운영 스크립트에서는 `cwd`를 명시하는 편이 안전합니다.

```js
import { glob } from 'node:fs/promises';

const projectRoot = new URL('../', import.meta.url);

for await (const file of glob('docs/**/*.md', { cwd: projectRoot })) {
  console.log(file);
}
```

`cwd`를 고정하면 스크립트를 어디서 실행하든 같은 기준으로 파일을 찾을 수 있습니다.
ESM에서 파일 위치를 기준으로 경로를 잡는 법은 [Node.js import.meta.dirname 가이드](/development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html)와 함께 보면 좋습니다.

### 배열이 필요할 때만 명시적으로 모은다

파일 목록을 정렬하거나 개수를 비교해야 한다면 배열로 모아도 됩니다.
다만 모든 작업을 처음부터 배열 중심으로 만들 필요는 없습니다.

```js
import { glob } from 'node:fs/promises';

const files = [];

for await (const file of glob('src/**/*.js')) {
  files.push(file);
}

files.sort();
console.log(`${files.length} files`);
```

작은 저장소라면 이 방식도 충분합니다.
하지만 로그 파일, 이미지, 빌드 산출물처럼 수천 개 이상을 다룰 가능성이 있다면 가능한 한 순회 중에 처리하는 구조를 먼저 고려하세요.
메모리를 아끼는 것뿐 아니라 실패 지점도 더 빨리 파악할 수 있습니다.

## cwd와 exclude 옵션 설계하기

### 작업 기준 디렉터리를 명확히 고정한다

자동화 스크립트에서 흔한 버그는 “내 컴퓨터에서는 됐는데 CI에서는 파일을 못 찾는” 상황입니다.
대부분 현재 작업 디렉터리가 달라졌기 때문입니다.

```js
import { glob } from 'node:fs/promises';

const root = new URL('../', import.meta.url);

for await (const file of glob('**/*.md', {
  cwd: root,
  exclude: ['node_modules/**', '_site/**', 'dist/**'],
})) {
  console.log(file);
}
```

`cwd`는 탐색 기준을 고정하고, `exclude`는 검색하지 않을 영역을 분리합니다.
`node_modules`, 빌드 산출물, 캐시 디렉터리는 대부분의 파일 검색에서 제외하는 편이 좋습니다.
검색 범위를 줄이면 속도도 좋아지고, 실수로 생성 파일을 다시 처리하는 문제도 줄어듭니다.

### exclude는 의도를 드러내는 이름으로 관리한다

여러 스크립트에서 같은 제외 규칙을 반복한다면 상수로 빼는 편이 낫습니다.

```js
const DEFAULT_EXCLUDES = [
  'node_modules/**',
  'dist/**',
  'coverage/**',
  '.git/**',
];

export async function collectSourceFiles(patterns, { cwd }) {
  const files = [];

  for await (const file of glob(patterns, {
    cwd,
    exclude: DEFAULT_EXCLUDES,
  })) {
    files.push(file);
  }

  return files.sort();
}
```

제외 규칙은 보안과도 연결됩니다.
예를 들어 문서 생성기가 `.env`, 인증 키, 빌드 캐시를 실수로 읽거나 출력하지 않게 하려면 애초에 탐색 범위를 좁혀야 합니다.
민감한 로그와 환경값을 다루는 기준은 [CLI 출력 sanitizing 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)처럼 출력 단계에서도 다시 확인하는 것이 좋습니다.

## withFileTypes로 파일과 디렉터리 구분하기

### 경로 문자열만으로 판단하지 않는다

파일인지 디렉터리인지 알아야 하는 작업이 있습니다.
예를 들어 문서 파일만 읽고, 디렉터리는 건너뛰어야 하는 경우입니다.
이때 경로 문자열의 확장자만 보고 판단하면 예외가 생깁니다.

`withFileTypes` 옵션을 사용하면 디렉터리 엔트리 정보를 함께 받을 수 있습니다.

```js
import { glob } from 'node:fs/promises';

for await (const entry of glob('content/**/*', {
  withFileTypes: true,
})) {
  if (!entry.isFile()) {
    continue;
  }

  console.log(entry.name);
}
```

이 방식은 파일 시스템의 실제 정보를 기준으로 분기하므로 더 명확합니다.
다만 `withFileTypes`를 쓰면 반환 항목의 형태가 문자열이 아니라 디렉터리 엔트리 객체가 됩니다.
팀 코드에서는 함수 이름이나 반환 타입을 분명히 드러내야 합니다.

### 읽기 작업은 필요한 파일에만 수행한다

파일 검색과 파일 읽기를 붙일 때는 순서를 조심해야 합니다.
먼저 후보를 좁히고, 필요한 파일만 읽는 구조가 안전합니다.

```js
import { readFile } from 'node:fs/promises';
import { glob } from 'node:fs/promises';

for await (const file of glob('posts/**/*.md', {
  exclude: ['posts/drafts/**'],
})) {
  const markdown = await readFile(file, 'utf8');

  if (markdown.includes('TODO')) {
    console.log(`TODO found: ${file}`);
  }
}
```

이 예제는 단순하지만 실무에서 자주 쓰입니다.
문서 품질 검사, 릴리스 노트 생성, 블로그 포스트 메타데이터 점검 같은 작업은 대부분 이런 형태에서 시작합니다.
Jekyll 포스트처럼 날짜와 slug 규칙이 중요한 파일을 검사할 때도 같은 흐름을 적용할 수 있습니다.

## 테스트 자동화에 적용하기

### 테스트 파일 목록을 명시적으로 만든다

Node.js 내장 test runner는 기본 탐색 규칙을 제공하지만, 프로젝트마다 테스트 파일 위치가 다를 수 있습니다.
특정 디렉터리의 테스트만 모아 별도 점검을 하고 싶다면 `glob()`으로 목록을 만들 수 있습니다.

```js
import { glob } from 'node:fs/promises';

export async function collectTestFiles({ cwd }) {
  const files = [];

  for await (const file of glob(['*.test.js', 'test/**/*.test.js', 'src/**/*.test.js'], {
    cwd,
    exclude: ['node_modules/**', 'dist/**'],
  })) {
    files.push(file);
  }

  return files.sort();
}
```

이 함수는 테스트 실행 자체를 대신하지 않습니다.
대신 리포트 생성, 변경 파일 기반 점검, 누락 테스트 감지 같은 보조 작업에 적합합니다.
테스트 러너 기본 실행은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 기준으로 두는 편이 좋습니다.

### 임시 디렉터리에서 glob 동작을 검증한다

파일 검색 코드는 테스트 fixture의 영향을 많이 받습니다.
실제 프로젝트 디렉터리를 그대로 읽는 테스트는 환경에 따라 흔들릴 수 있으므로, 임시 디렉터리를 만들고 필요한 파일만 배치하는 방식이 안정적입니다.

```js
import assert from 'node:assert/strict';
import { mkdtemp, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import test from 'node:test';
import { collectTestFiles } from '../src/collect-test-files.js';

test('collects JavaScript test files only', async () => {
  const cwd = await mkdtemp(join(tmpdir(), 'glob-example-'));

  await writeFile(join(cwd, 'app.test.js'), '');
  await writeFile(join(cwd, 'app.md'), '');

  const files = await collectTestFiles({ cwd });

  assert.deepEqual(files, ['app.test.js']);
});
```

테스트가 끝난 뒤 정리까지 자동화하려면 [Node.js fsPromises.mkdtempDisposable 가이드](/development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html)를 참고해 fixture 수명 주기를 관리할 수 있습니다.
테스트 준비와 정리 흐름은 [Node.js test runner hooks 가이드](/development/blog/seo/2026/05/23/nodejs-test-runner-hooks-beforeeach-aftereach-guide.html)와도 잘 이어집니다.

## 운영 스크립트에서 주의할 점

### 검색 범위를 넓게 잡을수록 비용도 커진다

`**/*` 같은 패턴은 편하지만 프로젝트 루트 전체를 훑을 수 있습니다.
작은 저장소에서는 문제가 없어 보여도, 모노레포나 빌드 산출물이 많은 프로젝트에서는 느려질 수 있습니다.

운영 스크립트에서는 다음 순서로 범위를 줄이세요.

1. `cwd`를 프로젝트 루트 또는 대상 디렉터리로 고정한다.
2. 패턴을 `**/*`보다 구체적으로 작성한다.
3. `exclude`로 캐시와 산출물 디렉터리를 제외한다.
4. 파일 내용을 읽기 전에 경로와 파일 타입으로 후보를 줄인다.

파일 검색은 빠른 편이어도 결국 파일 시스템 작업입니다.
요청 처리 경로 안에서 매번 실행하기보다, 배치 작업이나 시작 시점 캐시, 개발용 CLI에 두는 것이 더 자연스럽습니다.

### 버전 조건을 문서화한다

`fsPromises.glob()`은 비교적 최근 Node.js에서 안정화된 API입니다.
팀 프로젝트에서 사용한다면 `package.json`의 `engines`나 README에 최소 Node.js 버전을 남겨야 합니다.

```json
{
  "engines": {
    "node": ">=22.17.0"
  }
}
```

실제 최소 버전은 배포 환경과 CI 이미지에 맞춰 정해야 합니다.
새 API를 쓰는 글이나 사내 문서에서는 “로컬 Node 버전이 낮으면 동작하지 않을 수 있다”는 점을 함께 적어 두는 편이 좋습니다.
런타임 버전 차이를 줄이는 운영 기준은 [dependency version pinning 가이드](/development/blog/seo/2026/03/04/dependency-version-pinning-guide-for-seo-trust.html)와도 연결됩니다.

## fs.promises.glob 선택 기준

### 내장 glob이 잘 맞는 경우

`fsPromises.glob()`은 다음 상황에 잘 맞습니다.

- 내부 자동화 스크립트에서 파일 목록을 찾는다.
- 빌드 산출물, 문서, 테스트 파일을 간단히 검사한다.
- 외부 glob 라이브러리의 고급 기능이 필요 없다.
- 비동기 반복자로 파일을 하나씩 처리하고 싶다.
- Node.js 버전을 충분히 통제할 수 있다.

특히 작은 CLI나 운영 보조 도구에서는 의존성을 줄이는 장점이 큽니다.
인자 파싱까지 내장 API로 맞추고 싶다면 [Node.js util.parseArgs 가이드](/development/blog/seo/2026/04/28/nodejs-util-parseargs-cli-argument-parsing-guide.html)를 함께 적용하면 단일 파일 도구를 꽤 깔끔하게 만들 수 있습니다.

### 외부 라이브러리를 유지하는 편이 나은 경우

반대로 다음 조건이라면 기존 glob 라이브러리를 유지하는 편이 나을 수 있습니다.

- 오래된 Node.js 버전을 지원해야 한다.
- 라이브러리별 고급 ignore 규칙이나 정렬 규칙에 의존한다.
- 기존 빌드 도구가 특정 glob 패키지와 깊게 통합되어 있다.
- 매우 큰 모노레포에서 검증된 성능 튜닝이 이미 있다.
- 팀 전체가 같은 파일 검색 추상화를 오래 써 왔다.

내장 API가 생겼다고 해서 모든 코드를 즉시 바꿀 필요는 없습니다.
새 스크립트부터 적용하고, 기존 코드는 의존성 업데이트 비용이나 유지보수 부담이 실제로 커질 때 천천히 정리하는 방식이 안전합니다.

## 자주 묻는 질문

### fs.promises.glob는 glob 패턴 문자열만 검사하나요?

아닙니다.
`fsPromises.glob()`은 파일 시스템을 탐색해 패턴에 맞는 항목을 찾습니다.
이미 가지고 있는 경로 문자열이 패턴과 맞는지만 확인하려면 `path.matchesGlob()`가 더 직접적입니다.

### glob 결과 순서는 항상 같나요?

파일 시스템과 환경에 따라 순서를 전제로 두지 않는 편이 안전합니다.
출력이나 테스트에서 순서가 중요하다면 배열로 모은 뒤 명시적으로 정렬하세요.

```js
const files = [];

for await (const file of glob('src/**/*.js')) {
  files.push(file);
}

files.sort();
```

### 민감한 파일이 검색 결과에 섞일 수 있나요?

검색 범위를 넓게 잡으면 가능합니다.
그래서 `cwd`, 구체적인 패턴, `exclude`를 함께 사용해야 합니다.
특히 `.env`, 인증 키, 로그, 빌드 캐시, 사용자 업로드 디렉터리는 자동화 출력에 섞이지 않도록 별도 제외 규칙을 두는 것이 좋습니다.

## 마무리

Node.js `fs/promises.glob()`은 파일 검색을 위한 무거운 추상화가 아니라, 작은 자동화와 검증 스크립트를 단정하게 만드는 기본 도구입니다.
`cwd`로 기준 디렉터리를 고정하고, `exclude`로 불필요한 영역을 줄이고, 필요한 경우 `withFileTypes`로 실제 파일 타입을 확인하면 실무에서 충분히 안정적인 흐름을 만들 수 있습니다.

관련해서 경로 패턴 검사 자체는 [Node.js path.matchesGlob 가이드](/development/blog/seo/2026/05/17/nodejs-path-matchesglob-file-pattern-guide.html), 테스트 파일 검증은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html), 임시 fixture 정리는 [Node.js fsPromises.mkdtempDisposable 가이드](/development/blog/seo/2026/05/20/nodejs-fspromises-mkdtempdisposable-temp-directory-cleanup-guide.html)를 이어서 참고하면 좋습니다.

공식 API 세부 옵션은 [Node.js File system 문서](https://nodejs.org/api/fs.html)를 기준으로 프로젝트의 Node.js 버전에 맞춰 확인하세요.

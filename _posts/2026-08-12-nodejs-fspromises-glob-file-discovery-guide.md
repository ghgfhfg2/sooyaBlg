---
layout: post
title: "Node.js fs.promises.glob 가이드: 빌드 스크립트에서 파일을 안전하게 찾는 법"
date: 2026-08-12 08:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-glob-file-discovery-guide
permalink: /development/blog/seo/2026/08/12/nodejs-fspromises-glob-file-discovery-guide.html
alternates:
  ko: /development/blog/seo/2026/08/12/nodejs-fspromises-glob-file-discovery-guide.html
  x_default: /development/blog/seo/2026/08/12/nodejs-fspromises-glob-file-discovery-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, glob, file-system, build-script, automation, backend, javascript]
description: "Node.js fs.promises.glob로 빌드 스크립트와 자동화 도구에서 파일을 안전하게 찾는 방법을 정리합니다. AsyncIterator 사용법, exclude 패턴, withFileTypes, 심볼릭 링크 주의점과 민감정보 점검까지 실무 예제로 설명합니다."
---

빌드 스크립트나 운영 자동화 코드를 만들다 보면 "특정 확장자의 파일을 모두 찾아라"는 요구가 자주 나옵니다.
Markdown 포스트 목록을 만들거나, 테스트 fixture를 수집하거나, 변환 대상 JSON 파일을 찾는 식입니다.
예전에는 이런 일을 위해 외부 glob 패키지를 바로 추가하는 경우가 많았지만, 최신 Node.js에서는 `node:fs/promises`의 `glob()`를 먼저 검토할 수 있습니다.

공식 Node.js 문서 기준으로 `fsPromises.glob()`는 파일 패턴을 받아 `AsyncIterator`를 반환합니다.
`cwd`, `exclude`, `withFileTypes`, `followSymlinks` 같은 옵션을 제공하므로 작은 빌드 도구와 내부 자동화에서는 의존성을 줄이면서도 충분히 읽기 좋은 코드를 만들 수 있습니다.
이 글에서는 `glob()`를 파일 탐색의 기본 도구로 쓰는 방법, 제외 규칙을 안전하게 두는 기준, 심볼릭 링크와 민감정보 파일을 점검하는 습관을 정리합니다.

경로 문자열을 정리하는 기준은 [Node.js path resolve/normalize 가이드](/development/blog/seo/2026/08/06/nodejs-path-resolve-normalize-traversal-prevention-guide.html)를 함께 보면 좋습니다.
파일 변경 감지가 필요하다면 [Node.js fs.promises.watch 가이드](/development/blog/seo/2026/08/02/nodejs-fspromises-watch-file-change-monitoring-guide.html), 단순 패턴 매칭 기준은 [Node.js path.matchesGlob 가이드](/development/blog/seo/2026/08/02/nodejs-path-matchesglob-file-filter-guide.html)도 이어서 참고할 수 있습니다.

## fs.promises.glob가 필요한 이유

### H3. 파일 탐색 코드를 한 줄짜리 재귀 함수로 만들지 않는다

디렉터리를 재귀로 돌면서 확장자를 비교하는 코드는 직접 만들기 쉽습니다.
하지만 실제 운영 스크립트에서는 제외할 디렉터리, 상대 경로 기준, 심볼릭 링크, 숨김 파일, 결과 정렬 같은 세부 조건이 금방 붙습니다.

```js
import { readdir } from 'node:fs/promises';
import { join } from 'node:path';

async function findMarkdownFiles(dir) {
  const entries = await readdir(dir, { withFileTypes: true });
  const files = [];

  for (const entry of entries) {
    const fullPath = join(dir, entry.name);

    if (entry.isDirectory()) {
      files.push(...await findMarkdownFiles(fullPath));
    } else if (entry.name.endsWith('.md')) {
      files.push(fullPath);
    }
  }

  return files;
}
```

작은 예제에서는 충분해 보이지만, `node_modules`, `.git`, 빌드 출력물, 임시 폴더를 제외하려면 코드가 계속 커집니다.
이때 `glob()`는 파일 탐색 의도를 패턴과 옵션으로 드러내는 쪽에 가깝습니다.

```js
import { glob } from 'node:fs/promises';

for await (const file of glob('**/*.md', {
  cwd: 'content',
  exclude: ['**/node_modules/**', '**/.git/**', '**/dist/**']
})) {
  console.log(file);
}
```

이 코드는 `content` 아래의 Markdown 파일을 찾되, 불필요한 폴더는 건너뜁니다.
탐색 기준이 함수 내부 로직에 숨어 있지 않고 호출부에 모여 있기 때문에 리뷰하기도 쉽습니다.

### H3. AsyncIterator라서 큰 결과를 한 번에 들고 있지 않아도 된다

`glob()`는 배열을 바로 반환하지 않고 비동기 이터레이터를 반환합니다.
파일 수가 많을 때 전체 목록을 먼저 만들고 처리하기보다, 발견되는 항목을 순차적으로 다루기 좋습니다.

```js
import { glob } from 'node:fs/promises';

export async function collectPostSlugs() {
  const slugs = [];

  for await (const file of glob('_posts/**/*.md')) {
    slugs.push(file.replace(/^_posts\//, '').replace(/\.md$/, ''));
  }

  return slugs;
}
```

물론 결과가 작고 정렬이 필요하면 배열로 모아도 됩니다.
중요한 점은 탐색과 처리를 분리할 수 있다는 것입니다.
대량 파일 변환, lint 대상 수집, static site build 전처리처럼 결과를 하나씩 처리해도 되는 작업에서는 메모리 사용과 진행 로그를 더 자연스럽게 관리할 수 있습니다.

## 기본 사용법

### H3. cwd를 고정해서 결과 경로를 예측 가능하게 만든다

자동화 스크립트는 어디에서 실행되는지에 따라 동작이 달라지면 위험합니다.
`process.cwd()`에 암묵적으로 기대기보다, 탐색 기준 디렉터리를 명시하는 편이 좋습니다.

```js
import { glob } from 'node:fs/promises';
import { fileURLToPath } from 'node:url';
import { dirname, resolve } from 'node:path';

const __dirname = dirname(fileURLToPath(import.meta.url));
const projectRoot = resolve(__dirname, '..');

export async function findSourceFiles() {
  const files = [];

  for await (const file of glob('src/**/*.js', {
    cwd: projectRoot
  })) {
    files.push(file);
  }

  return files.sort();
}
```

`cwd`를 고정하면 CI, 로컬 터미널, npm script에서 같은 기준으로 동작합니다.
결과가 상대 경로로 나오므로 로그와 캐시 키도 안정적으로 유지하기 쉽습니다.

### H3. 여러 패턴은 목적별로 명확하게 나눈다

`glob()`의 `pattern`은 문자열 하나 또는 문자열 배열을 받을 수 있습니다.
예를 들어 소스 파일과 테스트 파일을 함께 찾되 출력물은 제외하려면 아래처럼 작성할 수 있습니다.

```js
import { glob } from 'node:fs/promises';

const patterns = [
  'src/**/*.js',
  'test/**/*.js'
];

const exclude = [
  '**/node_modules/**',
  '**/coverage/**',
  '**/dist/**'
];

for await (const file of glob(patterns, { exclude })) {
  console.log(file);
}
```

패턴 배열은 편리하지만 너무 많은 의미를 한 번에 넣으면 읽기 어려워집니다.
빌드 대상, 테스트 대상, 문서 대상처럼 목적이 다르면 함수도 나누는 편이 낫습니다.
그래야 제외 규칙이 과하게 넓어져 필요한 파일까지 놓치는 일을 줄일 수 있습니다.

## 제외 규칙과 보안 기준

### H3. exclude는 민감한 위치를 기본 차단하는 데 쓴다

파일 탐색 자동화에서 가장 흔한 실수는 "찾으면 안 되는 파일"까지 결과에 섞는 것입니다.
`.env`, 인증서, 개인 키, 임시 백업 파일은 검색 결과나 로그에 노출되지 않게 기본 제외 목록에 넣는 편이 좋습니다.

```js
export const defaultExclude = [
  '**/.git/**',
  '**/node_modules/**',
  '**/dist/**',
  '**/coverage/**',
  '**/.env',
  '**/.env.*',
  '**/*.pem',
  '**/*.key'
];
```

이 목록은 보안 장치의 전부가 아니라 최소 안전망입니다.
자동화 도구가 파일 내용을 읽거나 외부 API로 업로드한다면, 파일명 기준 제외만 믿지 말고 내용 처리 단계에서도 토큰과 개인정보를 마스킹해야 합니다.

### H3. 사용자 입력을 glob 패턴으로 그대로 쓰지 않는다

관리 화면이나 CLI에서 사용자가 검색 패턴을 넘길 수 있게 만들 때는 특히 조심해야 합니다.
입력값을 그대로 `glob(userPattern)`에 넣으면 예상보다 넓은 범위를 훑거나, 내부 디렉터리 구조를 노출하는 로그를 만들 수 있습니다.

```js
const allowedTargets = {
  posts: '_posts/**/*.md',
  pages: 'pages/**/*.md',
  assets: 'assets/**/*.{css,js,png,jpg,webp}'
};

export function getAllowedPattern(target) {
  return allowedTargets[target] ?? allowedTargets.posts;
}
```

사용자에게는 `posts`, `pages`, `assets` 같은 선택지만 받고 실제 패턴은 코드에서 고정하세요.
경로 traversal을 막아야 하는 입력이라면 `path.resolve()`와 기준 디렉터리 검사를 함께 적용하는 것이 좋습니다.

## withFileTypes와 followSymlinks

### H3. 파일 종류가 필요하면 withFileTypes를 켠다

결과 경로 문자열만 필요하면 기본값으로 충분합니다.
하지만 파일인지 디렉터리인지, 심볼릭 링크인지에 따라 처리 방식이 달라진다면 `withFileTypes: true`를 사용할 수 있습니다.

```js
import { glob } from 'node:fs/promises';

for await (const entry of glob('content/**', {
  withFileTypes: true
})) {
  if (entry.isFile() && entry.name.endsWith('.md')) {
    console.log(entry.name);
  }
}
```

`withFileTypes`를 켜면 결과를 단순 문자열처럼 다루면 안 됩니다.
이 옵션을 쓰는 함수는 반환 타입과 후속 처리까지 함께 바뀌므로, 문자열 경로가 필요한 함수와 섞지 않는 편이 안전합니다.

### H3. followSymlinks는 의도적으로만 켠다

`followSymlinks` 옵션은 `**` 패턴을 확장할 때 디렉터리 심볼릭 링크를 따라갈지 결정합니다.
기본값은 `false`입니다.
심볼릭 링크를 따라가면 모노레포나 공유 폴더에서 편할 수 있지만, 탐색 범위가 예상보다 커질 수 있습니다.

```js
for await (const file of glob('packages/**/*.js', {
  cwd: projectRoot,
  followSymlinks: false
})) {
  console.log(file);
}
```

대부분의 빌드 스크립트에서는 기본값을 유지하는 편이 안전합니다.
정말 링크를 따라가야 한다면 결과 개수 제한, 제외 목록, 로그 마스킹 기준을 함께 점검하세요.
공식 문서에 따르면 심볼릭 링크 순환은 재귀적으로 순회하지 않도록 처리되지만, 그렇다고 탐색 범위 관리 책임이 사라지는 것은 아닙니다.

## 실무 적용 예제

### H3. 블로그 포스트 품질 점검 대상 모으기

아래 예제는 Jekyll 블로그의 포스트 파일을 찾아 기본 검사를 수행하는 구조입니다.
파일 탐색은 `glob()`가 맡고, 실제 품질 검사는 별도 함수가 담당합니다.

```js
import { readFile } from 'node:fs/promises';
import { glob } from 'node:fs/promises';

const exclude = [
  '**/node_modules/**',
  '**/.git/**',
  '**/_site/**'
];

export async function auditPosts(root) {
  const results = [];

  for await (const file of glob('_posts/**/*.md', {
    cwd: root,
    exclude
  })) {
    const markdown = await readFile(new URL(file, `file://${root}/`), 'utf8');

    results.push({
      file,
      hasDescription: /^description:/m.test(markdown),
      hasInternalLink: /\]\(\/development\/blog\/seo\//.test(markdown)
    });
  }

  return results;
}
```

이런 구조에서는 검사 항목을 늘리기 쉽습니다.
예를 들어 `description` 누락, 내부 링크 부족, 금칙 표현 포함 여부를 같은 루프에서 확인할 수 있습니다.
단, 파일 내용을 로그로 그대로 출력하지 않는 규칙은 꼭 지켜야 합니다.

### H3. 결과는 정렬해서 재현성을 높인다

파일 시스템 탐색 순서는 환경에 따라 달라질 수 있습니다.
빌드 산출물이나 리포트가 매번 같은 순서로 나오게 하려면 결과를 정렬하세요.

```js
export async function listMarkdownFiles(root) {
  const files = [];

  for await (const file of glob('**/*.md', {
    cwd: root,
    exclude: defaultExclude
  })) {
    files.push(file);
  }

  return files.sort((a, b) => a.localeCompare(b));
}
```

정렬은 작지만 중요한 습관입니다.
CI diff가 안정되고, 캐시 키가 흔들리지 않으며, 테스트 snapshot도 불필요하게 바뀌지 않습니다.

## 마무리 점검

`fs.promises.glob()`는 Node.js 빌드 스크립트와 자동화 도구에서 파일 탐색 코드를 단순하게 만들어 줍니다.
핵심은 패턴을 명확히 쓰고, `cwd`를 고정하며, 제외 목록으로 민감한 위치를 기본 차단하는 것입니다.
파일 종류가 필요하면 `withFileTypes`를 사용하고, 심볼릭 링크는 기본값을 존중하면서 필요한 경우에만 따라가세요.

오늘 바로 적용할 체크리스트는 다음과 같습니다.

- `glob()` 호출마다 `cwd`를 명시했는가?
- `node_modules`, `.git`, 빌드 출력물, 민감정보 파일을 제외했는가?
- 사용자 입력을 glob 패턴으로 직접 전달하지 않는가?
- 결과 순서가 중요한 작업에서는 `sort()`를 적용했는가?
- 파일 내용이나 경로 로그에 토큰, 키, 개인정보가 섞이지 않는가?

작은 자동화일수록 파일 탐색 코드는 쉽게 방치됩니다.
하지만 탐색 범위와 제외 규칙을 처음부터 명확히 해 두면 빌드, 검사, 배포 스크립트가 훨씬 오래 안정적으로 유지됩니다.

---
layout: post
title: "Node.js path.matchesGlob 가이드: 파일 필터링을 표준 API로 단순하게 만드는 법"
date: 2026-08-02 20:00:00 +0900
lang: ko
translation_key: nodejs-path-matchesglob-file-filter-guide
permalink: /development/blog/seo/2026/08/02/nodejs-path-matchesglob-file-filter-guide.html
alternates:
  ko: /development/blog/seo/2026/08/02/nodejs-path-matchesglob-file-filter-guide.html
  x_default: /development/blog/seo/2026/08/02/nodejs-path-matchesglob-file-filter-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, path, matchesglob, glob, file-filter, cli, filesystem, javascript]
description: "Node.js path.matchesGlob로 파일 include/exclude 필터를 만들 때 알아야 할 패턴 설계, 경로 정규화, 재귀 탐색 결합, 보안상 주의점을 실무 예제로 정리합니다."
---

파일 필터링은 작지만 자주 반복되는 작업입니다.
테스트 대상 파일을 고르거나, 빌드 입력에서 임시 파일을 제외하거나, 변경 감시 이벤트 중 관심 있는 파일만 처리할 때 glob 패턴이 필요합니다.
예전에는 이 용도로 별도 glob 라이브러리를 설치하는 경우가 많았지만, 단순한 패턴 매칭이라면 Node.js의 `path.matchesGlob()`만으로 충분한 상황이 늘었습니다.

`path.matchesGlob(path, pattern)`은 문자열 경로가 glob 패턴과 맞는지 `true` 또는 `false`로 알려주는 표준 API입니다.
디렉터리를 직접 탐색하거나 파일 목록을 만들어 주는 함수는 아니지만, 이미 얻은 경로 목록을 include/exclude 규칙으로 거르는 데 유용합니다.
Node.js 공식 문서 기준으로 이 API는 v22.20.0과 v24.8.0에서 안정화되었으므로, 런타임 버전을 맞출 수 있는 프로젝트라면 작은 CLI 도구와 빌드 스크립트에서 의존성을 줄이는 선택지가 됩니다.

이 글에서는 `path.matchesGlob()`로 실무형 파일 필터를 만드는 방법을 정리합니다.
파일 목록을 재귀적으로 읽는 흐름은 [Node.js opendir 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html)를 함께 보면 좋고, 변경 감시 이벤트와 결합할 때는 [Node.js watch 가이드](/development/blog/seo/2026/08/02/nodejs-fspromises-watch-file-change-monitoring-guide.html)를 참고할 수 있습니다.
심볼릭 링크나 실제 경로 검증이 필요한 도구라면 [Node.js realpath 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-realpath-symlink-guide.html)와 같이 설계하는 편이 안전합니다.

## path.matchesGlob가 맞는 상황

### 이미 가진 파일 목록을 필터링한다

`path.matchesGlob()`는 파일 시스템을 걷는 API가 아닙니다.
인자로 받은 경로 문자열이 패턴과 일치하는지만 판단합니다.
따라서 디렉터리 탐색은 `fs.promises.opendir()`, `readdir()`, 변경 감시 이벤트, Git 변경 파일 목록 같은 다른 입력에서 만들고, 마지막 필터 단계에 `matchesGlob()`를 두는 구조가 자연스럽습니다.

```js
import path from 'node:path';

const includePatterns = [
  'src/**/*.js',
  'scripts/**/*.mjs'
];

const excludePatterns = [
  '**/*.test.js',
  '**/node_modules/**',
  '**/.*/**'
];

export function matchesAny(filePath, patterns) {
  return patterns.some((pattern) => path.matchesGlob(filePath, pattern));
}

export function shouldProcess(filePath) {
  return matchesAny(filePath, includePatterns) &&
    !matchesAny(filePath, excludePatterns);
}
```

이 예제의 핵심은 include와 exclude를 분리한 점입니다.
먼저 처리할 수 있는 후보를 넓게 고르고, 그다음 테스트 파일, 의존성 폴더, 숨김 디렉터리처럼 제외해야 하는 대상을 빼면 규칙을 읽기 쉬워집니다.
패턴이 늘어날수록 하나의 복잡한 glob로 모든 조건을 표현하기보다 작은 규칙 여러 개를 조합하는 편이 유지보수에 좋습니다.

### 작은 CLI 도구의 의존성을 줄인다

사내 스크립트나 저장소 안의 작은 CLI는 설치 비용과 의존성 업데이트 비용이 더 크게 느껴질 때가 있습니다.
이때 파일 목록은 직접 만들고 glob 매칭만 표준 API에 맡기면 패키지 수를 줄일 수 있습니다.
특히 빌드 전 검사, 마이그레이션 후보 선택, 문서 파일 검증처럼 단순 필터링이 목적이라면 충분히 실용적입니다.

```js
import path from 'node:path';
import { opendir } from 'node:fs/promises';

async function walk(dir, root = dir) {
  const results = [];
  const handle = await opendir(dir);

  for await (const entry of handle) {
    const absolute = path.join(dir, entry.name);

    if (entry.isDirectory()) {
      results.push(...await walk(absolute, root));
      continue;
    }

    if (entry.isFile()) {
      results.push(path.relative(root, absolute));
    }
  }

  return results;
}

export async function findMarkdownFiles(rootDir) {
  const files = await walk(rootDir);
  return files.filter((file) => path.matchesGlob(file, 'docs/**/*.md'));
}
```

이 코드는 `docs` 아래의 Markdown 파일만 골라냅니다.
파일 수가 많고 성능이 중요하다면 탐색 단계에서 제외 디렉터리를 먼저 건너뛰는 최적화가 필요합니다.
하지만 작은 저장소라면 "탐색 후 필터" 구조가 읽기 쉽고 테스트하기도 편합니다.

## 패턴 설계 기준

### include는 좁게, exclude는 명확하게 쓴다

glob 규칙은 시간이 지나면 복잡해지기 쉽습니다.
처음부터 모든 파일을 대상으로 삼고 exclude를 계속 추가하면, 새 디렉터리가 생겼을 때 의도하지 않은 파일까지 처리될 수 있습니다.
반대로 include를 업무 범위에 맞게 좁히면 실수의 범위가 줄어듭니다.

```js
const rules = {
  include: [
    'src/**/*.js',
    'src/**/*.mjs',
    'tools/**/*.js'
  ],
  exclude: [
    '**/*.test.js',
    '**/*.fixture.js',
    '**/dist/**'
  ]
};

export function createFileFilter({ include, exclude }) {
  return (relativePath) => {
    const included = include.some((pattern) => {
      return path.matchesGlob(relativePath, pattern);
    });

    if (!included) return false;

    return !exclude.some((pattern) => {
      return path.matchesGlob(relativePath, pattern);
    });
  };
}
```

이런 구조는 테스트 케이스로 검증하기 쉽습니다.
`src/server.js`는 포함되어야 하고, `src/server.test.js`는 제외되어야 하며, `dist/server.js`는 제외되어야 한다는 식으로 대표 경로를 고정하면 됩니다.
필터 규칙은 운영 사고를 막는 작은 정책이므로 코드 리뷰에서 눈에 잘 들어오게 작성하는 편이 좋습니다.

### 경로 구분자를 한 가지 규칙으로 맞춘다

저장소 안의 파일 패턴은 보통 `/` 기준으로 작성됩니다.
하지만 Windows에서 `path.relative()`를 사용하면 `src\app.js` 같은 문자열이 나올 수 있습니다.
팀에서 여러 운영체제를 쓴다면 glob 매칭 전에 경로 표현을 통일하는 작은 함수를 두는 편이 안전합니다.

```js
import path from 'node:path';

export function toPortablePath(filePath) {
  return filePath.split(path.sep).join('/');
}

export function relativePortablePath(rootDir, absolutePath) {
  return toPortablePath(path.relative(rootDir, absolutePath));
}

export function shouldBuild(rootDir, absolutePath) {
  const relative = relativePortablePath(rootDir, absolutePath);

  return path.matchesGlob(relative, 'src/**/*.js') &&
    !path.matchesGlob(relative, '**/*.test.js');
}
```

이 함수는 패턴 파일, 설정 파일, 테스트 스냅샷에서 모두 같은 문자열을 보게 해 줍니다.
크로스 플랫폼 도구에서 경로 문자열이 흔들리면 테스트는 통과했는데 CI나 동료 환경에서 다른 결과가 나올 수 있습니다.
경로 정규화는 작지만 빌드 도구의 재현성을 높이는 기본 작업입니다.

## 변경 감시와 결합하기

### watcher 이벤트를 바로 필터링한다

`fs.promises.watch()`와 `path.matchesGlob()`를 함께 쓰면 변경 이벤트 중 관심 있는 파일만 골라낼 수 있습니다.
다만 watcher 이벤트의 `filename`은 환경에 따라 비어 있거나 예상과 다를 수 있으므로, 이벤트 payload만 믿지 말고 필요하면 재스캔으로 보정해야 합니다.

```js
import path from 'node:path';
import { watch } from 'node:fs/promises';

export async function watchSourceFiles(rootDir, { signal, onChange }) {
  for await (const event of watch(rootDir, { signal, recursive: true })) {
    if (!event.filename) {
      await onChange({ type: 'rescan-required' });
      continue;
    }

    const relative = toPortablePath(String(event.filename));

    if (!path.matchesGlob(relative, 'src/**/*.{js,mjs,cjs}')) {
      continue;
    }

    await onChange({
      type: event.eventType,
      path: relative
    });
  }
}
```

이 구조는 빠른 피드백이 필요한 개발 서버에 적합합니다.
파일 하나가 바뀌면 먼저 glob으로 후보를 거르고, 빌드 대상이면 후속 작업을 실행합니다.
정확도가 더 중요하다면 이벤트를 받은 뒤 전체 파일 목록을 다시 만들고 동일한 필터를 적용하는 방식으로 보강할 수 있습니다.

### 설정 파일로 필터를 외부화한다

팀마다 빌드 대상, 문서 대상, 검사 제외 경로가 다르다면 패턴을 코드에 박아두기보다 설정으로 빼는 편이 좋습니다.
단, 외부 입력을 그대로 신뢰하면 너무 넓은 범위를 처리하거나 숨김 파일까지 읽을 수 있으므로 기본 제한을 함께 둬야 합니다.

```js
const defaultExclude = [
  '**/node_modules/**',
  '**/.git/**',
  '**/dist/**',
  '**/coverage/**'
];

export function compileFilter(config) {
  const include = config.include?.length ? config.include : ['src/**/*.js'];
  const exclude = [...defaultExclude, ...(config.exclude ?? [])];

  return createFileFilter({ include, exclude });
}
```

설정 파일을 지원하면 도구는 유연해지지만, 책임도 늘어납니다.
패턴이 비어 있을 때의 기본값, 너무 넓은 패턴을 허용할지 여부, 숨김 디렉터리 처리 기준을 문서화해야 합니다.
실무 도구에서는 "기본적으로 안전하게 제외하고, 필요할 때 명시적으로 포함"하는 정책이 관리하기 쉽습니다.

## 보안과 안정성 체크

### glob 매칭은 경로 접근 제어가 아니다

`path.matchesGlob()`는 문자열 매칭 함수입니다.
어떤 경로가 패턴에 맞는지 확인할 수는 있지만, 그 경로가 실제로 허용된 루트 안에 있는지 보장하지 않습니다.
사용자 입력 경로를 열거나 삭제하거나 업로드 대상으로 삼는다면 별도의 루트 검증이 필요합니다.

```js
import path from 'node:path';
import { realpath } from 'node:fs/promises';

export async function assertInsideRoot(rootDir, candidatePath) {
  const root = await realpath(rootDir);
  const target = await realpath(candidatePath);
  const relative = path.relative(root, target);

  if (relative.startsWith('..') || path.isAbsolute(relative)) {
    throw new Error('Path is outside the allowed root');
  }

  return target;
}
```

glob 필터와 루트 검증은 역할이 다릅니다.
glob은 "처리 대상 파일 종류"를 고르는 데 쓰고, `realpath()`와 `path.relative()` 검증은 "허용된 위치 안에 있는가"를 확인하는 데 씁니다.
둘을 분리해야 패턴 실수 하나가 파일 접근 문제로 이어지는 일을 줄일 수 있습니다.

### 패턴 실패를 조용히 넘기지 않는다

`path.matchesGlob()`는 인자가 문자열이 아니면 `TypeError`를 던질 수 있습니다.
외부 설정에서 패턴을 읽는다면 배열인지, 문자열만 들어 있는지 먼저 검증해야 합니다.
잘못된 설정을 무시하고 기본값으로 넘어가면 사용자는 필터가 적용된 줄 알고 위험한 작업을 실행할 수 있습니다.

```js
export function assertPatternList(name, value) {
  if (!Array.isArray(value)) {
    throw new TypeError(`${name} must be an array`);
  }

  for (const pattern of value) {
    if (typeof pattern !== 'string' || pattern.length === 0) {
      throw new TypeError(`${name} must contain non-empty strings`);
    }
  }
}
```

빌드나 배포 전 단계의 필터라면 실패를 빨리 드러내는 편이 낫습니다.
특히 삭제, 업로드, 배포 대상 선택처럼 되돌리기 어려운 작업에서는 잘못된 glob 설정을 경고 수준으로 처리하지 말고 명확한 오류로 멈춰야 합니다.
작은 검증 코드가 큰 사고를 막습니다.

## 실무 체크리스트

### 구현 전 확인할 것

- Node.js 런타임이 `path.matchesGlob()`를 안정적으로 지원하는 버전인지 확인한다.
- 파일 탐색과 glob 매칭의 책임을 분리한다.
- include는 업무 범위에 맞게 좁게 잡고 exclude는 명확히 둔다.
- 크로스 플랫폼 실행이 필요하면 상대 경로 문자열을 한 가지 형식으로 맞춘다.
- 사용자 입력 경로에는 glob과 별개로 루트 검증을 적용한다.

### 코드 리뷰에서 볼 것

- 패턴 설정이 비어 있거나 잘못됐을 때 조용히 넘어가지 않는가?
- 숨김 디렉터리, `node_modules`, 빌드 산출물이 기본 제외되는가?
- `**/*` 같은 넓은 패턴이 삭제나 배포 대상 선택에 바로 쓰이지 않는가?
- watcher 이벤트의 `filename` 누락과 재스캔 필요성을 처리하는가?
- 내부 경로, 사용자명, 민감한 파일명이 로그에 과하게 남지 않는가?

## 마무리

`path.matchesGlob()`는 glob 생태계 전체를 대체하는 거대한 도구가 아닙니다.
하지만 이미 확보한 파일 경로를 표준 API로 빠르게 include/exclude하고 싶을 때는 충분히 유용합니다.
작은 CLI, 빌드 전 검사, 변경 감시 필터처럼 범위가 명확한 작업에서는 의존성을 줄이고 코드를 단순하게 만들 수 있습니다.

좋은 파일 필터는 패턴 몇 줄보다 운영 기준이 더 중요합니다.
경로 문자열을 통일하고, include와 exclude를 분리하고, glob을 접근 제어로 오해하지 않아야 합니다.
그 기준만 지키면 `path.matchesGlob()`는 Node.js 프로젝트의 작은 자동화 도구를 더 가볍고 예측 가능하게 만드는 선택지가 됩니다.

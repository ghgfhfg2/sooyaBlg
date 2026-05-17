---
layout: post
title: "Node.js path.matchesGlob 가이드: 파일 경로가 glob 패턴과 맞는지 내장 API로 확인하기"
date: 2026-05-17 20:00:00 +0900
lang: ko
translation_key: nodejs-path-matchesglob-file-pattern-guide
permalink: /development/blog/seo/2026/05/17/nodejs-path-matchesglob-file-pattern-guide.html
alternates:
  ko: /development/blog/seo/2026/05/17/nodejs-path-matchesglob-file-pattern-guide.html
  x_default: /development/blog/seo/2026/05/17/nodejs-path-matchesglob-file-pattern-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, path, glob, filesystem, automation, backend]
description: "Node.js path.matchesGlob로 파일 경로가 glob 패턴과 일치하는지 확인하는 방법을 정리했습니다. 경로 정규화, include/exclude 규칙, 테스트·배포 자동화에서 안전하게 쓰는 체크리스트를 예제로 설명합니다."
---

파일을 스캔한 뒤 특정 패턴만 골라야 하는 작업은 생각보다 자주 나옵니다.
예를 들어 `*.test.js` 파일만 테스트 대상으로 분리하거나, 빌드 산출물에서 이미지 파일만 검사하거나, 배포 전에 `.env` 같은 민감 파일이 섞이지 않았는지 확인해야 할 수 있습니다.
이때 단순 문자열 비교만으로는 `src/**/*.js`처럼 하위 디렉터리를 포함하는 규칙을 표현하기 어렵습니다.

Node.js의 `path.matchesGlob()`는 경로가 glob 패턴과 일치하는지 확인하는 내장 API입니다.
파일 목록을 직접 찾는 도구라기보다, 이미 수집한 경로를 include/exclude 규칙으로 판정하는 데 알맞습니다.
이 글에서는 Node.js path.matchesGlob 기본 사용법, 경로 정규화 기준, 실무에서 흔한 필터링 패턴, 안전한 배포 체크리스트를 정리합니다.
파일 목록 자체를 먼저 모아야 한다면 [Node.js fsPromises.glob 가이드: 파일 검색을 내장 API로 처리하는 법](/development/blog/seo/2026/05/07/nodejs-fspromises-glob-file-discovery-guide.html)을 함께 참고하세요.

## path.matchesGlob가 필요한 상황

### H3. 수집과 판정을 분리한다

`path.matchesGlob()`는 파일 시스템을 탐색하지 않습니다.
대신 문자열 경로 하나와 glob 패턴 하나를 받아서 둘이 맞는지 `true` 또는 `false`로 알려 줍니다.
그래서 “파일을 어떻게 찾을 것인가”와 “찾은 파일을 어떤 규칙으로 통과시킬 것인가”를 분리하기 좋습니다.

```js
import path from 'node:path';

console.log(path.matchesGlob('src/app.test.js', '**/*.test.js'));
// true

console.log(path.matchesGlob('src/app.js', '**/*.test.js'));
// false
```

이 구조는 테스트 러너, 문서 빌더, 배포 전 검사 스크립트처럼 경로 규칙이 자주 바뀌는 자동화에 잘 맞습니다.
파일 탐색 단계는 별도 함수로 두고, 패턴 판정 단계만 작은 순수 함수처럼 테스트할 수 있기 때문입니다.
내장 테스트 러너와 조합하는 흐름은 [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하기](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)와도 잘 어울립니다.

### H3. 의존성을 늘리지 않고 간단한 glob 판정을 처리한다

기존에는 glob 판정을 위해 외부 패키지를 추가하는 경우가 많았습니다.
복잡한 워크스페이스 스캔이나 고급 패턴이 필요하면 여전히 전문 패키지가 더 편할 수 있습니다.
하지만 내부 스크립트에서 “이 경로가 이 패턴에 맞는가” 정도를 확인하려면 내장 API만으로 충분한 경우가 많습니다.

```js
const includePatterns = ['src/**/*.js', 'scripts/**/*.mjs'];
const file = 'scripts/publish.mjs';

const included = includePatterns.some((pattern) =>
  path.matchesGlob(file, pattern)
);

console.log(included); // true
```

의존성이 줄면 설치 시간, 보안 점검 범위, 버전 충돌 가능성도 함께 줄어듭니다.
특히 CI에서 실행되는 작은 검증 스크립트라면 표준 API로 표현할 수 있는지 먼저 확인해 보는 편이 좋습니다.
다만 팀의 Node.js 최소 버전이 이 API를 지원하는지는 `engines`, CI 이미지, 로컬 개발 환경에서 함께 확인해야 합니다.

## 기본 사용법과 경로 기준

### H3. 경로와 패턴은 같은 기준으로 맞춘다

경로 매칭에서 가장 흔한 실수는 비교 기준이 섞이는 것입니다.
한쪽은 절대 경로이고 다른 한쪽은 프로젝트 루트 기준 상대 경로라면, 패턴이 맞아도 결과가 예상과 다를 수 있습니다.
자동화 스크립트에서는 먼저 기준 디렉터리를 정하고 모든 파일을 그 기준의 상대 경로로 바꾼 뒤 판정하는 편이 안전합니다.

```js
import path from 'node:path';

const root = process.cwd();
const absoluteFile = path.resolve(root, 'src/app.test.js');
const relativeFile = path.relative(root, absoluteFile);

console.log(relativeFile); // src/app.test.js
console.log(path.matchesGlob(relativeFile, '**/*.test.js')); // true
```

이렇게 하면 로컬과 CI에서 작업 디렉터리만 같다면 같은 규칙을 적용할 수 있습니다.
로그에는 전체 홈 디렉터리나 사용자명까지 노출하기보다 프로젝트 기준 상대 경로를 남기는 편이 더 안전합니다.
실제 로그 예제를 글이나 문서에 넣을 때는 [로그 예제 정제 가이드: 믿을 수 있는 개발 글을 위한 마스킹 원칙](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)을 기준으로 민감한 경로를 정리하세요.

### H3. 운영체제별 경로 구분자를 의식한다

Windows는 기본 경로 구분자로 역슬래시를 쓰고, macOS와 Linux는 슬래시를 씁니다.
패턴을 팀 공통으로 관리한다면 구분자 차이가 테스트 실패나 누락으로 이어질 수 있습니다.
프로젝트 내부 규칙에서는 가능하면 슬래시(`/`) 기준의 상대 경로로 정규화한 뒤 매칭하는 방식을 권합니다.

```js
function toPosixPath(filePath) {
  return filePath.split(path.sep).join('/');
}

const normalized = toPosixPath(path.relative(process.cwd(), 'src/app.test.js'));

console.log(path.matchesGlob(normalized, 'src/**/*.test.js'));
```

단순해 보이지만 이 단계가 빠지면 CI 운영체제가 바뀌었을 때만 실패하는 문제가 생길 수 있습니다.
경로를 만드는 쪽에서는 `path.resolve()`와 `path.relative()`를 쓰고, 패턴을 적용하는 쪽에서는 프로젝트가 정한 문자열 기준으로 한 번 더 정규화하면 원인을 줄일 수 있습니다.

## include와 exclude 규칙 만들기

### H3. include 후보를 먼저 좁힌다

여러 패턴을 다룰 때는 include 규칙으로 후보를 먼저 좁히고, 그다음 exclude 규칙으로 위험하거나 불필요한 파일을 빼는 흐름이 읽기 쉽습니다.
이 방식은 테스트 대상, 린트 대상, 문서 생성 대상처럼 “대부분 포함하되 일부 제외”하는 작업에 잘 맞습니다.

```js
import path from 'node:path';

const include = ['src/**/*.js', 'scripts/**/*.mjs'];
const exclude = ['**/*.test.js', '**/__fixtures__/**'];

export function shouldProcess(file) {
  const normalized = file.split(path.sep).join('/');

  const isIncluded = include.some((pattern) =>
    path.matchesGlob(normalized, pattern)
  );

  const isExcluded = exclude.some((pattern) =>
    path.matchesGlob(normalized, pattern)
  );

  return isIncluded && !isExcluded;
}
```

규칙이 많아지면 배열 이름만으로 의도가 드러나야 합니다.
`patterns1`, `patterns2`처럼 애매한 이름보다 `sourceCodePatterns`, `testFilePatterns`, `sensitiveFilePatterns`처럼 목적을 담는 편이 유지보수에 유리합니다.

### H3. 민감 파일은 exclude보다 allowlist를 우선한다

배포나 공개 아카이브를 만들 때는 제외 목록만 믿으면 위험합니다.
새로운 민감 파일 이름이 추가됐는데 exclude 규칙에 반영되지 않으면 그대로 복사되거나 업로드될 수 있습니다.
이런 경우에는 허용할 확장자와 경로를 먼저 정하고, 그 밖의 파일은 기본적으로 제외하는 allowlist 전략이 안전합니다.

```js
const publicAssets = [
  'public/**/*.html',
  'public/**/*.css',
  'public/**/*.js',
  'public/**/*.{png,jpg,jpeg,svg,webp}',
];

export function isPublicAsset(file) {
  const normalized = file.split(path.sep).join('/');
  return publicAssets.some((pattern) =>
    path.matchesGlob(normalized, pattern)
  );
}
```

`.env`, 인증서, 개인 키, 원본 로그처럼 공개되면 안 되는 파일이 있는 저장소라면 allowlist가 특히 중요합니다.
복사 단계에서는 [Node.js fs.cp 가이드: 디렉터리와 파일을 안전하게 복사하는 법](/development/blog/seo/2026/05/17/nodejs-fs-cp-recursive-copy-guide.html)의 `filter` 옵션과 조합해 허용된 파일만 옮기도록 만들 수 있습니다.

## 배포 전 검증 스크립트 예시

### H3. 공개 디렉터리에 금지 파일이 없는지 확인한다

정적 사이트 배포 전에 `dist`나 `_site`에 민감 파일이 섞였는지 검사하면 사고를 줄일 수 있습니다.
아래 예시는 이미 수집한 파일 목록에서 금지 패턴과 일치하는 항목을 찾으면 프로세스를 실패시키는 구조입니다.
실무에서는 파일 수집 단계에 `fs.promises.glob()`이나 재귀 순회 함수를 붙이면 됩니다.

```js
import path from 'node:path';

const forbidden = [
  '**/.env',
  '**/.env.*',
  '**/*.pem',
  '**/*.key',
  '**/debug.log',
];

export function findForbiddenFiles(files) {
  return files.filter((file) => {
    const normalized = file.split(path.sep).join('/');
    return forbidden.some((pattern) => path.matchesGlob(normalized, pattern));
  });
}

const leakedFiles = findForbiddenFiles([
  'dist/index.html',
  'dist/assets/app.js',
  'dist/.env',
]);

if (leakedFiles.length > 0) {
  console.error('Forbidden files found:', leakedFiles);
  process.exitCode = 1;
}
```

에러 메시지에는 실제 비밀값을 출력하지 말고 파일 경로와 규칙 이름 정도만 남기는 편이 좋습니다.
운영 로그와 CI 로그는 생각보다 오래 남고 여러 사람이 볼 수 있으므로, 실패 원인을 설명하되 민감정보를 드러내지 않는 균형이 필요합니다.

### H3. 패턴 규칙 자체를 테스트한다

glob 규칙은 문자열이라서 리팩터링 중 깨져도 바로 알아차리기 어렵습니다.
그래서 중요한 include/exclude 규칙은 몇 개의 대표 경로로 테스트해 두는 편이 좋습니다.
새 폴더 구조를 추가하거나 파일명을 바꿀 때 테스트가 먼저 알려 주면 배포 누락을 줄일 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { shouldProcess } from './file-rules.js';

test('source files are included', () => {
  assert.equal(shouldProcess('src/index.js'), true);
  assert.equal(shouldProcess('scripts/publish.mjs'), true);
});

test('test fixtures are excluded', () => {
  assert.equal(shouldProcess('src/index.test.js'), false);
  assert.equal(shouldProcess('src/__fixtures__/sample.js'), false);
});
```

패턴 테스트는 빠르고 유지 비용이 낮습니다.
특히 배포 대상 파일, 문서 생성 대상 파일, 코드 생성 대상 파일처럼 누락되면 장애로 이어지는 규칙은 테스트로 고정해 두는 편이 좋습니다.
실행 시간 측정이나 CI 병목 확인이 필요하다면 [Node.js performance.now 가이드: 코드 실행 시간을 정확하게 측정하는 법](/development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html)도 연결해 볼 수 있습니다.

## 실무 체크리스트

### H3. 패턴을 설정값으로 분리한다

처음에는 코드 안에 패턴을 바로 적어도 괜찮습니다.
하지만 규칙이 여러 스크립트에서 공유되기 시작하면 `file-patterns.js` 같은 설정 모듈로 분리하는 편이 낫습니다.
그래야 테스트, 빌드, 배포 검사에서 같은 기준을 재사용할 수 있습니다.

```js
export const sourceFilePatterns = ['src/**/*.js', 'scripts/**/*.mjs'];
export const ignoredFilePatterns = ['**/*.test.js', '**/__fixtures__/**'];
export const forbiddenPublishPatterns = ['**/.env', '**/*.pem', '**/*.key'];
```

설정 파일에는 패턴의 의도를 주석으로 짧게 남기세요.
“왜 이 파일을 제외하는가”가 보이지 않으면 나중에 누군가 성능이나 편의성을 이유로 규칙을 지울 수 있습니다.
자동화 규칙은 코드보다 운영 맥락이 더 중요할 때가 많습니다.

### H3. 실패 메시지는 짧고 재현 가능하게 만든다

패턴 검증 스크립트가 실패했을 때는 어떤 파일이 어떤 규칙에 걸렸는지 바로 보여 줘야 합니다.
다만 파일 내용, 토큰, 환경 변수 값은 절대 출력하지 않습니다.
경로와 패턴 이름만으로도 대부분의 문제는 충분히 재현할 수 있습니다.

```js
function formatPatternError(file, patternName) {
  return `File pattern check failed: ${file} matched ${patternName}`;
}
```

이런 메시지는 CI 로그에서 검색하기 쉽고, 개발자가 로컬에서 같은 파일 목록으로 재현하기도 쉽습니다.
셸 명령과 함께 쓰는 검증 스크립트라면 [셸 명령 안전 가이드: 개발 글에서 위험한 명령을 다루는 기준](/development/blog/seo/2026/03/03/shell-command-safety-context-for-dev-guides.html)의 원칙처럼 삭제·덮어쓰기 명령을 예시로 넣을 때 특히 보수적으로 작성하는 편이 좋습니다.

## FAQ

### H3. path.matchesGlob만으로 파일 목록을 찾을 수 있나요?

아니요.
`path.matchesGlob()`는 경로 하나가 패턴과 맞는지 판정하는 API입니다.
파일 목록을 찾으려면 `fs.promises.glob()` 같은 탐색 단계가 별도로 필요합니다.

### H3. 외부 glob 패키지를 모두 대체해도 되나요?

간단한 판정에는 내장 API가 유용합니다.
하지만 대규모 워크스페이스 탐색, 복잡한 ignore 규칙, 오래된 Node.js 버전 지원이 필요하다면 기존 패키지가 더 적합할 수 있습니다.
팀의 런타임 버전과 필요한 패턴 기능을 먼저 확인하세요.

### H3. 배포 검사에서는 include와 exclude 중 무엇을 우선해야 하나요?

공개 대상 파일을 다룰 때는 allowlist 방식의 include를 우선하는 것이 안전합니다.
제외 목록은 보조 안전망으로 두고, 기본값은 “허용하지 않으면 공개하지 않는다”에 가깝게 설계하는 편이 좋습니다.

## 함께 보면 좋은 글

- [Node.js fsPromises.glob 가이드: 파일 검색을 내장 API로 처리하는 법](/development/blog/seo/2026/05/07/nodejs-fspromises-glob-file-discovery-guide.html)
- [Node.js fs.cp 가이드: 디렉터리와 파일을 안전하게 복사하는 법](/development/blog/seo/2026/05/17/nodejs-fs-cp-recursive-copy-guide.html)
- [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하기](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)
- [로그 예제 정제 가이드: 믿을 수 있는 개발 글을 위한 마스킹 원칙](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)

---
layout: post
title: "Node.js path.resolve 가이드: 경로 탐색 공격을 막는 파일 경로 검증법"
date: 2026-08-06 08:00:00 +0900
lang: ko
translation_key: nodejs-path-resolve-normalize-traversal-prevention-guide
permalink: /development/blog/seo/2026/08/06/nodejs-path-resolve-normalize-traversal-prevention-guide.html
alternates:
  ko: /development/blog/seo/2026/08/06/nodejs-path-resolve-normalize-traversal-prevention-guide.html
  x_default: /development/blog/seo/2026/08/06/nodejs-path-resolve-normalize-traversal-prevention-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, path, path-traversal, filesystem, security, backend, javascript]
description: "Node.js에서 path.resolve와 path.normalize로 파일 경로를 검증하고 경로 탐색 공격을 줄이는 방법을 정리합니다. base directory 제한, symlink 주의점, 로그 마스킹, 테스트 체크리스트까지 실무 예제로 설명합니다."
---

Node.js 서비스에서 사용자 입력으로 파일을 읽거나 내려주는 기능은 생각보다 자주 등장합니다.
첨부파일 다운로드, 정적 리포트 조회, 변환 결과 확인, 로컬 캐시 접근 같은 기능이 모두 파일 경로와 연결됩니다.
문제는 입력값을 그대로 파일 경로에 붙이는 순간 `../` 같은 경로 탐색 공격에 노출될 수 있다는 점입니다.

`path.resolve()`와 `path.normalize()`는 이런 위험을 줄일 때 기본으로 확인해야 하는 도구입니다.
핵심은 **사용자 입력을 신뢰하지 않고, 항상 허용된 기준 디렉터리 안의 절대 경로인지 검증한 뒤 파일 작업을 실행하는 것**입니다.
이 글에서는 Node.js에서 파일 경로를 안전하게 조합하고 검증하는 실무 패턴을 정리합니다.

경로 매칭 규칙을 함께 다룬다면 [Node.js path.matchesGlob 파일 필터 가이드](/development/blog/seo/2026/08/02/nodejs-path-matchesglob-file-filter-guide.html)를 참고할 만합니다.
심볼릭 링크까지 확인해야 하는 저장소라면 [Node.js fsPromises.realpath 심볼릭 링크 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-realpath-symlink-guide.html)와 같이 보는 편이 좋습니다.
실제 파일 접근 실패 처리는 [Node.js 시스템 오류 코드 가이드](/development/blog/seo/2026/08/05/nodejs-system-error-code-handling-guide.html)에서 이어서 정리했습니다.

## 파일 경로 검증이 필요한 이유

### 문자열 연결만으로는 기준 디렉터리를 지킬 수 없다

가장 위험한 패턴은 사용자 입력을 그대로 파일 경로 뒤에 붙이는 방식입니다.

```js
import { readFile } from 'node:fs/promises';

const uploadRoot = '/srv/app/uploads';

export async function readUserFile(name) {
  return readFile(`${uploadRoot}/${name}`, 'utf8');
}
```

겉으로는 `/srv/app/uploads` 아래의 파일만 읽는 것처럼 보입니다.
하지만 `name`에 `../../etc/passwd` 같은 값이 들어오면 의도한 디렉터리 밖으로 빠져나갈 수 있습니다.
운영체제는 문자열의 의도를 보지 않고 최종 경로를 해석하기 때문입니다.

파일 이름처럼 보이는 값도 입력입니다.
업로드 ID, 리포트 이름, 캐시 키, 이미지 경로, 압축 해제된 내부 파일명 모두 검증 대상입니다.
특히 URL query나 route parameter에서 온 값은 반드시 경계 밖의 데이터로 취급해야 합니다.

### normalize는 정리 도구이지 보안 검증의 끝이 아니다

`path.normalize()`는 중복 구분자나 `.`·`..` 조각을 정리합니다.
하지만 normalize만 호출했다고 안전해지는 것은 아닙니다.
정리된 결과가 여전히 허용된 디렉터리 밖을 가리킬 수 있기 때문입니다.

```js
import path from 'node:path';

console.log(path.normalize('/srv/app/uploads/../../etc/passwd'));
// /srv/etc/passwd
```

이 결과는 문자열이 깔끔해졌을 뿐, 접근해도 되는 경로가 아닙니다.
따라서 normalize는 보조 도구로 보고, 실제 판단은 "최종 절대 경로가 기준 디렉터리 안에 있는가"로 해야 합니다.

## path.resolve로 기준 디렉터리 안에 묶기

### base directory와 입력 경로를 모두 절대 경로로 만든다

실무에서 가장 많이 쓰는 흐름은 아래와 같습니다.

1. 기준 디렉터리를 절대 경로로 만든다.
2. 사용자 입력을 기준 디렉터리와 함께 resolve한다.
3. 결과가 기준 디렉터리 내부인지 확인한다.
4. 통과한 경로만 파일 API에 넘긴다.

```js
import path from 'node:path';

const uploadRoot = path.resolve('/srv/app/uploads');

export function resolveInside(baseDir, unsafeInput) {
  const base = path.resolve(baseDir);
  const target = path.resolve(base, unsafeInput);
  const relative = path.relative(base, target);

  if (relative === '' || (!relative.startsWith('..') && !path.isAbsolute(relative))) {
    return target;
  }

  throw new Error('Path is outside of the allowed directory');
}

console.log(resolveInside(uploadRoot, '2026/report.txt'));
```

`path.relative(base, target)`가 `..`로 시작하면 target이 base 밖에 있다는 뜻입니다.
또 Windows 환경에서는 다른 드라이브 경로가 들어올 수 있으므로 `path.isAbsolute(relative)`도 함께 확인하는 편이 안전합니다.

### startsWith만 쓰면 경로 접두어 착각이 생길 수 있다

단순히 문자열 `startsWith(base)`만 확인하는 코드는 피하는 편이 좋습니다.
예를 들어 기준 디렉터리가 `/srv/app/upload`일 때 `/srv/app/upload-backup/file.txt`도 같은 접두어로 시작합니다.
문자열 접두어만 보면 통과하지만 실제로는 다른 디렉터리입니다.

```js
const base = '/srv/app/upload';
const target = '/srv/app/upload-backup/report.txt';

console.log(target.startsWith(base));
// true, but this is not inside /srv/app/upload
```

그래서 `path.relative()`를 기준으로 판단하는 편이 낫습니다.
경로를 문자열이 아니라 파일 시스템의 계층 관계로 해석하기 때문입니다.

## 사용자 입력을 더 좁게 제한하기

### 파일명만 받아도 된다면 basename 정책이 더 단순하다

기능 요구사항이 "특정 디렉터리 안의 파일 하나를 읽는다"라면 하위 디렉터리 경로까지 허용하지 않는 편이 더 단순합니다.
이 경우 입력값을 파일명으로 제한하고, `/`, `\`, `..` 같은 경로 조각을 거부할 수 있습니다.

```js
const SAFE_FILE_NAME = /^[a-zA-Z0-9._-]{1,120}$/;

export function assertSafeFileName(fileName) {
  if (!SAFE_FILE_NAME.test(fileName)) {
    throw new Error('Invalid file name');
  }

  if (fileName === '.' || fileName === '..') {
    throw new Error('Invalid file name');
  }

  return fileName;
}
```

허용 문자를 명시하면 예측 가능성이 높아집니다.
다국어 파일명을 허용해야 하는 서비스라면 정책을 따로 잡아야 하지만, 내부 리포트나 캐시 파일처럼 시스템이 만든 이름이라면 좁은 ASCII 규칙이 운영하기 쉽습니다.

### 하위 디렉터리를 허용할 때는 세그먼트 단위로 검증한다

사용자가 `2026/08/report.json`처럼 하위 경로를 선택해야 하는 기능도 있습니다.
이때는 전체 문자열을 한 번에 정규식으로 뭉개기보다 경로 세그먼트를 나눠 검증하는 편이 읽기 쉽습니다.

```js
const SAFE_SEGMENT = /^[a-zA-Z0-9._-]{1,80}$/;

export function assertSafeRelativePath(input) {
  const segments = input.split(/[\\/]+/);

  if (segments.length === 0) {
    throw new Error('Empty path');
  }

  for (const segment of segments) {
    if (!SAFE_SEGMENT.test(segment) || segment === '.' || segment === '..') {
      throw new Error('Invalid path segment');
    }
  }

  return segments.join('/');
}
```

이렇게 하면 `/`, `\`, 빈 세그먼트, `..`를 명확하게 거를 수 있습니다.
그 뒤에도 최종적으로 `resolveInside()`를 한 번 더 통과시키는 편이 좋습니다.
입력 검증과 최종 경로 검증은 서로 보완 관계입니다.

## 실제 파일 읽기 흐름에 적용하기

### 검증과 파일 접근을 한 함수에 묶어둔다

파일 경로 검증은 호출부마다 흩어지면 빠뜨리기 쉽습니다.
허용 디렉터리별로 작은 유틸리티를 만들고, 파일 API 호출 전에 항상 거치도록 만드는 편이 안전합니다.

```js
import { readFile } from 'node:fs/promises';
import path from 'node:path';

const REPORT_ROOT = path.resolve('/srv/app/reports');

export async function readReport(relativePath) {
  const safePath = assertSafeRelativePath(relativePath);
  const absolutePath = resolveInside(REPORT_ROOT, safePath);

  return readFile(absolutePath, 'utf8');
}
```

이 구조의 장점은 호출부가 파일 시스템 규칙을 몰라도 된다는 점입니다.
API handler는 `readReport(req.params.name)`처럼 호출하고, 경로 보안 정책은 한곳에서 유지할 수 있습니다.

### 오류 메시지는 내부 경로를 그대로 드러내지 않는다

경로 검증 실패나 파일 접근 실패가 발생했을 때 전체 서버 경로를 사용자에게 보여 주지 않는 편이 좋습니다.
서버 디렉터리 구조, 사용자 홈 경로, 배포 환경 정보가 노출될 수 있기 때문입니다.

```js
export function toFileAccessResponse(error) {
  if (error?.message === 'Invalid path segment') {
    return {
      status: 400,
      body: { message: '요청한 파일 경로가 올바르지 않습니다.' }
    };
  }

  if (error?.code === 'ENOENT') {
    return {
      status: 404,
      body: { message: '요청한 파일을 찾을 수 없습니다.' }
    };
  }

  return {
    status: 500,
    body: { message: '파일을 처리하는 중 오류가 발생했습니다.' }
  };
}
```

운영 로그에는 원인 분류와 요청 ID를 남기되, 사용자 입력 전체나 절대 경로는 줄이는 편이 좋습니다.
파일명 자체가 개인정보일 수 있는 서비스라면 해시나 내부 ID로 바꿔 남기는 방식도 고려해야 합니다.

## symlink와 realpath를 함께 봐야 하는 경우

### 심볼릭 링크는 문자열 검증만으로 막기 어렵다

`path.resolve()` 기반 검증은 문자열 경로 기준으로 매우 유용합니다.
하지만 기준 디렉터리 안에 있는 파일이 심볼릭 링크이고, 그 링크가 기준 디렉터리 밖을 가리키는 경우는 별도로 생각해야 합니다.

예를 들어 `/srv/app/uploads/link-to-secret`이 실제로 `/srv/secret`을 가리킨다면 문자열상으로는 uploads 안에 있어도 실제 접근 대상은 밖일 수 있습니다.
사용자가 업로드 디렉터리에 symlink를 만들 수 있는 구조라면 이 위험이 커집니다.

이 경우에는 실제 파일 시스템이 해석한 경로를 확인하는 `realpath()` 계열 검증이 필요합니다.

```js
import { realpath } from 'node:fs/promises';
import path from 'node:path';

export async function resolveRealPathInside(baseDir, unsafeInput) {
  const base = await realpath(baseDir);
  const target = await realpath(path.resolve(base, unsafeInput));
  const relative = path.relative(base, target);

  if (relative === '' || (!relative.startsWith('..') && !path.isAbsolute(relative))) {
    return target;
  }

  throw new Error('Resolved path is outside of the allowed directory');
}
```

단, `realpath()`는 파일이 실제로 존재해야 합니다.
새 파일을 생성하는 흐름에서는 임시 파일 생성 위치, 디렉터리 권한, 원자적 rename 전략까지 함께 설계해야 합니다.
파일을 완성한 뒤 위치를 바꾸는 패턴은 [Node.js fsPromises.rename 원자적 배포 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-rename-atomic-publish-guide.html)와 잘 맞습니다.

### 업로드와 압축 해제 기능은 더 엄격하게 다룬다

압축 파일을 풀 때 내부 파일명이 `../../outside.txt`처럼 들어오는 경우가 있습니다.
이때도 각 entry 이름을 상대 경로로 검증하고, 최종 출력 경로가 추출 디렉터리 안인지 확인해야 합니다.

안전한 압축 해제 흐름은 대략 아래와 같습니다.

1. entry 이름의 절대 경로, 빈 세그먼트, `..`를 거부한다.
2. 출력 기준 디렉터리와 entry 이름을 `path.resolve()`로 합친다.
3. 결과가 기준 디렉터리 안인지 `path.relative()`로 확인한다.
4. symlink entry는 허용하지 않거나 별도 정책으로 제한한다.
5. 쓰기 전 파일 크기와 전체 추출 크기 한도를 확인한다.

경로 검증은 압축 해제 보안의 일부일 뿐입니다.
파일 개수, 총 크기, 덮어쓰기 정책, 확장자 정책도 같이 잡아야 운영 사고를 줄일 수 있습니다.

## 테스트로 고정해야 할 경계 사례

### 정상 경로보다 실패 경로를 더 많이 테스트한다

경로 검증 함수는 성공 케이스보다 실패 케이스가 중요합니다.
아래 입력은 최소한 테스트로 고정하는 편이 좋습니다.

- `../secret.txt`
- `subdir/../../secret.txt`
- `/etc/passwd`
- `C:\Windows\System32`
- `valid/../secret.txt`
- `valid//report.txt`
- `valid/%2e%2e/secret.txt`
- 빈 문자열과 너무 긴 파일명

URL에서 온 값이라면 디코딩 순서도 분명히 해야 합니다.
라우터나 프레임워크가 이미 decode한 값을 넘기는지, 애플리케이션 코드에서 직접 decode하는지에 따라 테스트 입력이 달라질 수 있습니다.

### 경로 정책은 운영체제 차이를 의식한다

Windows와 POSIX 환경은 경로 구분자와 절대 경로 표현이 다릅니다.
Linux 서버에서만 운영하는 서비스라도 개발자의 로컬 테스트가 Windows에서 실행될 수 있다면 `path.win32`, `path.posix` 테스트를 일부 추가하는 것도 좋습니다.

```js
import path from 'node:path';
import assert from 'node:assert/strict';

assert.equal(
  path.posix.relative('/srv/app/uploads', '/srv/app/secret').startsWith('..'),
  true
);

assert.equal(
  path.win32.isAbsolute('C:\\Windows\\System32'),
  true
);
```

실제 제품 코드는 현재 운영체제의 `path` 모듈을 쓰되, 테스트에서는 정책상 막아야 하는 입력을 명확히 남기는 것이 핵심입니다.
경로 처리는 사소해 보이지만 한 번 새면 영향이 크기 때문에, 실패 사례를 문서처럼 테스트로 남겨 두는 편이 오래 갑니다.

## 실무 적용 체크리스트

### 작은 다운로드 API부터 점검한다

이미 운영 중인 서비스라면 아래 순서로 점검해 보세요.

1. 사용자 입력이 파일 경로에 들어가는 모든 지점을 찾는다.
2. 문자열 연결로 경로를 만드는 코드를 `path.resolve()` 기반 함수로 모은다.
3. 파일명만 필요한 기능은 세그먼트 없는 파일명 정책으로 좁힌다.
4. 하위 디렉터리를 허용하는 기능은 세그먼트 단위 검증을 추가한다.
5. symlink 가능성이 있는 저장소는 `realpath()` 기준 검증을 검토한다.
6. 사용자 응답과 운영 로그에서 절대 경로 노출을 줄인다.
7. 경계 입력을 테스트로 고정한다.

이 정도만 해도 파일 다운로드, 리포트 조회, 압축 해제 같은 기능의 기본 방어선은 훨씬 단단해집니다.

## 마무리

Node.js의 `path.resolve()`와 `path.normalize()`는 파일 경로를 다룰 때 꼭 알아야 하는 기본 도구입니다.
하지만 보안의 핵심은 함수 이름 자체가 아니라, **최종 절대 경로가 허용된 기준 디렉터리 안에 있는지 확인하는 습관**입니다.

문자열 연결로 파일을 읽는 코드가 있다면 먼저 기준 디렉터리, 상대 경로 검증, `path.relative()` 확인을 한곳에 모아 보세요.
그다음 symlink, 로그 노출, 테스트 경계 사례까지 정리하면 파일 경로 처리의 위험을 크게 줄일 수 있습니다.

## 함께 보면 좋은 글

- [Node.js path.matchesGlob 파일 필터 가이드](/development/blog/seo/2026/08/02/nodejs-path-matchesglob-file-filter-guide.html)
- [Node.js fsPromises.realpath 심볼릭 링크 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-realpath-symlink-guide.html)
- [Node.js 시스템 오류 코드 가이드](/development/blog/seo/2026/08/05/nodejs-system-error-code-handling-guide.html)

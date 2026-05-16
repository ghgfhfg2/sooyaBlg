---
layout: post
title: "Node.js fs.cp 가이드: 디렉터리와 파일을 안전하게 복사하는 법"
date: 2026-05-17 08:00:00 +0900
lang: ko
translation_key: nodejs-fs-cp-recursive-copy-guide
permalink: /development/blog/seo/2026/05/17/nodejs-fs-cp-recursive-copy-guide.html
alternates:
  ko: /development/blog/seo/2026/05/17/nodejs-fs-cp-recursive-copy-guide.html
  x_default: /development/blog/seo/2026/05/17/nodejs-fs-cp-recursive-copy-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, filesystem, fs, copy, automation, backend]
description: "Node.js fs.cp와 fs.promises.cp로 파일·디렉터리를 재귀 복사하는 방법을 정리했습니다. recursive, force, errorOnExist, filter 옵션과 배포 스크립트에서 안전하게 쓰는 체크리스트를 예제로 설명합니다."
---

빌드 산출물을 배포 폴더로 옮기거나, 템플릿 디렉터리를 새 프로젝트로 복사하거나, 테스트 fixture를 매번 깨끗한 위치에 준비해야 할 때가 있습니다.
예전에는 `cp -R` 같은 셸 명령을 호출하거나 직접 재귀 순회를 구현하는 경우가 많았습니다.
하지만 Node.js 안에서 복사 흐름을 관리해야 한다면 운영체제 명령에 기대기보다 `fs.cp()` 또는 `fs.promises.cp()`를 쓰는 편이 더 일관적입니다.

`fs.cp()`는 파일과 디렉터리를 복사하는 Node.js 내장 API입니다.
특히 `recursive`, `force`, `errorOnExist`, `filter` 같은 옵션을 조합하면 배포·스캐폴딩·테스트 자동화에서 반복되는 복사 작업을 코드로 명확하게 표현할 수 있습니다.
이 글에서는 Node.js fs.cp로 디렉터리를 재귀 복사하는 기본 패턴, 덮어쓰기 정책을 안전하게 정하는 법, 복사 대상 필터링과 실무 체크리스트를 정리합니다.
파일 목록을 먼저 수집해야 하는 상황은 [Node.js fsPromises.glob 가이드: 파일 검색을 내장 API로 처리하는 법](/development/blog/seo/2026/05/07/nodejs-fspromises-glob-file-discovery-guide.html)도 함께 참고하세요.

## Node.js fs.cp가 필요한 상황

### H3. 셸 명령 대신 Node.js 코드로 복사를 관리한다

배포 스크립트에서 `cp -R source dist`를 실행하면 간단해 보입니다.
하지만 Windows, macOS, Linux가 섞인 환경에서는 셸 문법과 경로 처리 차이가 문제를 만들 수 있습니다.
Node.js API를 쓰면 같은 JavaScript 코드 안에서 경로 계산, 로깅, 예외 처리, 옵션 제어를 함께 관리할 수 있습니다.

```js
import { cp } from 'node:fs/promises';

await cp('public', 'dist/public', {
  recursive: true,
});
```

위 코드는 `public` 디렉터리를 `dist/public`으로 복사합니다.
`recursive: true`가 없으면 디렉터리 복사에서 오류가 날 수 있으므로, 폴더 전체를 옮기는 스크립트라면 옵션을 명시하는 습관이 좋습니다.
이렇게 복사 흐름을 코드로 남기면 CI 로그에서도 어떤 경로가 복사됐는지 추적하기 쉽습니다.

### H3. 복사는 삭제보다 안전하지만 덮어쓰기는 여전히 조심한다

복사 작업은 삭제보다 덜 위험해 보이지만, 기존 파일을 덮어쓰면 되돌리기 어려운 문제가 생길 수 있습니다.
특히 배포 산출물, 사용자 업로드, 운영 설정 파일이 섞인 경로에서는 `force`와 `errorOnExist` 정책을 먼저 정해야 합니다.

```js
await cp('template', 'project-a', {
  recursive: true,
  force: false,
  errorOnExist: true,
});
```

이 설정은 대상 파일이 이미 있으면 조용히 덮어쓰지 않고 오류를 냅니다.
처음 생성하는 프로젝트 템플릿, 마이그레이션 전 백업, 운영 설정 복사처럼 보수적으로 처리해야 하는 작업에 어울립니다.
반대로 빌드 산출물처럼 매번 새로 만들어도 되는 파일은 별도 임시 디렉터리에서 준비한 뒤 교체하는 전략이 더 안전합니다.

## fs.promises.cp 기본 사용법

### H3. 파일 하나를 복사한다

파일 하나를 복사할 때는 `recursive`가 필요하지 않습니다.
대상 디렉터리가 이미 존재한다는 전제에서 원본 파일과 대상 파일 경로를 지정하면 됩니다.

```js
import { cp, mkdir } from 'node:fs/promises';
import path from 'node:path';

const outputDir = 'dist/assets';
await mkdir(outputDir, { recursive: true });

await cp(
  'assets/logo.svg',
  path.join(outputDir, 'logo.svg'),
  { force: true }
);
```

`cp()`는 부모 디렉터리를 자동으로 모두 만들어 주는 도구가 아닙니다.
그래서 복사 전에 `mkdir(..., { recursive: true })`로 대상 폴더를 준비하는 편이 명확합니다.
경로를 문자열 결합으로 만들기보다 `path.join()`을 쓰면 운영체제별 구분자 차이도 줄일 수 있습니다.

### H3. 디렉터리를 재귀 복사한다

정적 파일, 템플릿, fixture 폴더처럼 디렉터리 전체를 복사할 때는 `recursive: true`를 사용합니다.
이 옵션은 하위 파일과 하위 디렉터리를 함께 복사하겠다는 의도를 드러냅니다.

```js
await cp('fixtures/base-project', 'tmp/project-under-test', {
  recursive: true,
  force: true,
});
```

테스트 준비 단계에서는 기존 임시 폴더를 지우고 다시 만드는 방식이 흔합니다.
이때 복사 전에 어떤 경로를 정리하는지 로그를 남기고, 작업 경로가 프로젝트 내부의 안전한 임시 경로인지 확인해야 합니다.
파일 시스템 작업은 작은 오타 하나로 엉뚱한 위치에 영향을 줄 수 있으므로, `process.cwd()`와 절대 경로를 함께 출력해 두면 디버깅이 쉬워집니다.

## 덮어쓰기와 충돌 정책 정하기

### H3. force와 errorOnExist를 함께 이해한다

`force`는 대상 파일이 있을 때 덮어쓸지 결정합니다.
기본적으로는 덮어쓰는 동작에 가까운 흐름으로 이해하기 쉬우므로, 중요한 스크립트에서는 원하는 값을 명시하는 편이 좋습니다.
`errorOnExist`는 `force: false`일 때 기존 대상이 있으면 오류를 낼지 제어합니다.

```js
await cp('docs', 'backup/docs', {
  recursive: true,
  force: false,
  errorOnExist: true,
});
```

백업이나 초안 보존 목적이라면 위처럼 실패를 명확하게 드러내는 설정이 낫습니다.
이미 대상이 있다는 사실은 누군가 먼저 작업했거나 이전 실행 결과가 남아 있다는 신호일 수 있습니다.
그 상태를 자동 덮어쓰기로 숨기면 원인 추적이 어려워집니다.

### H3. 배포 산출물은 임시 경로에서 검증한 뒤 교체한다

정적 사이트나 프런트엔드 빌드 결과를 복사할 때는 기존 공개 디렉터리에 바로 덮어쓰기보다 임시 폴더를 활용하는 편이 안전합니다.
예를 들어 `dist-next`에 먼저 복사하고 검증한 뒤, 문제가 없을 때만 실제 `dist`로 교체하는 방식입니다.

```js
await cp('build', 'dist-next', {
  recursive: true,
  force: true,
});

// 여기서 파일 개수, 필수 파일 존재 여부, index.html 등을 검증한다.
```

검증 없이 바로 덮어쓰면 빌드 중간 실패나 누락된 파일이 그대로 배포될 수 있습니다.
배포 자동화에서는 복사 자체보다 복사 전후의 확인 절차가 더 중요합니다.
작업 시간을 측정하고 병목을 확인하려면 [Node.js performance.now 가이드: 코드 실행 시간을 정확하게 측정하는 법](/development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html)을 연결해 볼 수 있습니다.

## filter 옵션으로 복사 대상 제한하기

### H3. 불필요한 파일을 복사하지 않는다

`filter` 옵션을 사용하면 원본 경로와 대상 경로를 기준으로 복사 여부를 결정할 수 있습니다.
예를 들어 `.DS_Store`, 로그 파일, 임시 파일, 테스트 출력물처럼 배포에 필요 없는 파일을 제외할 수 있습니다.

```js
await cp('public', 'dist/public', {
  recursive: true,
  filter(source) {
    return !source.endsWith('.DS_Store') && !source.includes('/__snapshots__/');
  },
});
```

필터 함수는 단순해야 합니다.
복잡한 비즈니스 규칙을 넣기 시작하면 어떤 파일이 왜 빠졌는지 파악하기 어려워집니다.
배포 제외 규칙이 많다면 별도의 목록 파일이나 glob 수집 단계로 분리하고, 최종 복사 단계에서는 이미 결정된 파일만 옮기는 구조가 더 읽기 쉽습니다.

### H3. 민감한 파일은 allowlist로 다룬다

`.env`, 인증서, 개인 키, 원본 로그처럼 민감한 파일이 섞일 수 있는 디렉터리를 통째로 복사하는 것은 위험합니다.
이런 경우에는 제외 목록보다 허용 목록이 안전합니다.
복사해도 되는 확장자와 폴더를 먼저 정하고 나머지는 복사하지 않는 방식입니다.

```js
const allowedExtensions = new Set(['.html', '.css', '.js', '.svg', '.png']);

await cp('site-output', 'deploy/site-output', {
  recursive: true,
  filter(source) {
    const ext = source.includes('.') ? source.slice(source.lastIndexOf('.')) : '';
    return ext === '' || allowedExtensions.has(ext);
  },
});
```

실무에서는 위 예시보다 경로 파싱을 더 엄격하게 다듬는 것이 좋습니다.
핵심은 민감 파일을 “나중에 제외”하는 것이 아니라, 애초에 복사 대상에 들어오지 않게 설계하는 것입니다.
환경 변수 파일 관리 기준은 [Node.js loadEnvFile 가이드: 내장 API로 .env 파일 읽는 법](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)과 함께 보면 좋습니다.

## 실무 체크리스트

### H3. 경로를 절대 경로로 확인한다

자동화 스크립트는 실행 위치가 바뀌면 상대 경로가 달라질 수 있습니다.
CI, 로컬 터미널, 패키지 매니저 스크립트에서 `cwd`가 서로 다르면 같은 코드가 다른 위치를 복사할 수 있습니다.

```js
import path from 'node:path';

const root = process.cwd();
const source = path.resolve(root, 'public');
const target = path.resolve(root, 'dist/public');

console.log({ source, target });
await cp(source, target, { recursive: true, force: true });
```

로그에는 토큰이나 개인정보가 아니라 경로와 실행 단계만 남기는 편이 안전합니다.
샘플 로그를 글이나 문서에 넣을 때는 [로그 예제 정제 가이드: 믿을 수 있는 개발 글을 위한 마스킹 원칙](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)을 기준으로 실제 사용자명, 내부 서버명, 비밀 경로를 정리하세요.

### H3. 복사 후 필수 파일을 검증한다

복사가 성공했다는 것은 API 호출이 오류 없이 끝났다는 뜻입니다.
서비스가 기대하는 파일이 모두 존재한다는 뜻은 아닙니다.
배포나 릴리스 스크립트에서는 복사 후 필수 파일을 확인하는 단계를 추가하는 것이 좋습니다.

```js
import { access } from 'node:fs/promises';

await access('dist/public/index.html');
await access('dist/public/assets/app.js');
```

필수 파일 검증은 단순하지만 효과가 큽니다.
누락을 빠르게 발견하면 잘못된 산출물이 배포되는 일을 줄일 수 있습니다.
복사 작업을 함수로 감싸고, 복사 전 준비·복사·검증·정리 단계를 분리하면 장애 원인도 더 빨리 찾을 수 있습니다.

## FAQ

### H3. fs.cp와 fs.copyFile은 무엇이 다른가요?

`fs.copyFile()`은 파일 하나를 복사하는 데 초점이 있습니다.
반면 `fs.cp()`는 파일뿐 아니라 디렉터리 재귀 복사와 필터링 옵션까지 다룹니다.
단일 파일만 확실히 복사하면 된다면 `copyFile()`도 충분하지만, 폴더 전체나 배포 산출물을 다룬다면 `cp()`가 더 적합합니다.

### H3. fs.cp가 부모 디렉터리도 자동으로 만들어 주나요?

대상 구조에 따라 필요한 디렉터리가 만들어질 수 있지만, 스크립트 의도를 분명히 하려면 복사 전에 `mkdir(targetParent, { recursive: true })`로 부모 폴더를 준비하는 편이 좋습니다.
특히 파일 하나를 깊은 경로로 복사할 때는 부모 디렉터리 존재 여부를 직접 관리하는 습관이 안전합니다.

### H3. 복사 전에 기존 dist 폴더를 지워도 될까요?

가능은 하지만 삭제는 복사보다 위험합니다.
반드시 프로젝트 내부의 임시·빌드 경로인지 확인하고, 운영 데이터나 업로드 폴더가 섞이지 않았는지 점검해야 합니다.
더 안전한 방식은 새 임시 경로에 복사하고 검증한 뒤 교체하는 것입니다.

## 마무리

Node.js `fs.cp()`는 파일과 디렉터리 복사를 JavaScript 코드 안에서 일관되게 처리하게 해 주는 실용적인 API입니다.
`recursive`로 디렉터리 복사 의도를 명시하고, `force`와 `errorOnExist`로 덮어쓰기 정책을 정하며, `filter`로 불필요하거나 민감한 파일을 복사 대상에서 제외할 수 있습니다.

실무에서 중요한 것은 복사 코드 한 줄보다 안전한 운영 흐름입니다.
원본·대상 경로를 확인하고, 덮어쓰기 정책을 명시하고, 복사 후 필수 파일을 검증하세요.
이 세 가지만 지켜도 배포 스크립트와 테스트 자동화에서 파일 복사로 생기는 사고를 크게 줄일 수 있습니다.

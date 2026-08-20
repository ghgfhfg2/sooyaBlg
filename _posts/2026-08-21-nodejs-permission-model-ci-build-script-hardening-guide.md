---
layout: post
title: "Node.js Permission Model 가이드: CI 빌드 스크립트 권한을 최소화하는 법"
date: 2026-08-21 08:00:00 +0900
lang: ko
translation_key: nodejs-permission-model-ci-build-script-hardening-guide
permalink: /development/blog/seo/2026/08/21/nodejs-permission-model-ci-build-script-hardening-guide.html
alternates:
  ko: /development/blog/seo/2026/08/21/nodejs-permission-model-ci-build-script-hardening-guide.html
  x_default: /development/blog/seo/2026/08/21/nodejs-permission-model-ci-build-script-hardening-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, permission-model, ci, build-security, least-privilege, filesystem, child-process, javascript]
description: "Node.js Permission Model로 CI 빌드 스크립트의 파일 읽기, 파일 쓰기, child_process 권한을 최소화하는 방법을 정리합니다. --permission, --allow-fs-read, --allow-fs-write, process.permission.has 점검과 운영 체크리스트까지 실무 예제로 설명합니다."
---

CI 빌드 스크립트는 생각보다 많은 권한을 가집니다.
소스 파일을 읽고, 산출물을 쓰고, 테스트 도구를 실행하고, 때로는 외부 명령까지 호출합니다.
이 권한이 넓게 열려 있으면 작은 스크립트 변경이나 의존성 동작 하나가 저장소 밖의 파일을 읽거나 예상하지 못한 위치에 파일을 남길 수 있습니다.

Node.js Permission Model은 이런 위험을 줄이기 위한 실행 시점 권한 제한 기능입니다.
공식 문서 기준으로 `--permission`을 켜면 파일 시스템 접근, 자식 프로세스, 워커, 네이티브 애드온, WASI, 인스펙터 같은 기능이 제한되고 필요한 권한만 별도 플래그로 열 수 있습니다.
특히 빌드 스크립트처럼 입력과 출력 경로가 비교적 분명한 작업에는 "필요한 파일만 읽고, 필요한 디렉터리에만 쓰기"라는 기준을 적용하기 좋습니다.

이 글에서는 Node.js Permission Model을 CI 빌드 단계에 붙이는 방법을 정리합니다.
빌드 재현성은 [npm ci와 lockfile 가이드](/development/blog/seo/2026/03/06/npm-ci-lockfile-reproducible-build-guide.html), 환경 변수 정리는 [env.example과 필수 환경 변수 가이드](/development/blog/seo/2026/03/08/env-example-and-required-variables-guide.html), CLI 입력 검증은 [Node.js util.parseArgs 가이드](/development/blog/seo/2026/08/09/nodejs-util-parseargs-cli-options-guide.html)와 함께 보면 좋습니다.

## Permission Model을 CI에 쓰는 이유

### 빌드 스크립트는 권한 경계가 흐려지기 쉽다

로컬에서 `node scripts/build.js`를 실행하면 Node.js 프로세스는 기본적으로 현재 사용자가 접근할 수 있는 파일을 읽고 쓸 수 있습니다.
CI에서도 마찬가지입니다.
빌드 스크립트가 `./src`만 읽어야 한다고 생각했더라도 실제 코드가 `../`, 홈 디렉터리, 임시 디렉터리, 캐시 디렉터리를 건드리는지 실행 전에는 놓치기 쉽습니다.

Permission Model을 켜면 이런 접근이 명시적 계약으로 바뀝니다.
허용하지 않은 파일 읽기나 쓰기는 `ERR_ACCESS_DENIED`로 실패합니다.
즉, 스크립트가 무엇을 필요로 하는지 로그와 CI 실패로 확인할 수 있습니다.

```bash
node --permission scripts/build.js
```

이 명령은 거의 바로 실패할 가능성이 높습니다.
엔트리 파일 자체는 읽을 수 있더라도 빌드가 필요한 설정 파일, 소스 디렉터리, 출력 디렉터리는 아직 허용하지 않았기 때문입니다.
처음 실패는 문제가 아니라 권한 목록을 좁게 만들기 위한 출발점입니다.

### 최소 권한은 사고 범위를 줄인다

CI 작업은 토큰, 설정 파일, 빌드 산출물, 캐시를 함께 다룹니다.
모든 파일 읽기와 쓰기를 열어 두면 실수로 민감한 파일을 읽어 로그에 남기거나, 잘못된 경로에 결과물을 쓰는 문제를 알아차리기 어렵습니다.

반대로 빌드 명령을 아래처럼 제한하면 스크립트의 행동 범위가 명확해집니다.

```bash
node \
  --permission \
  --allow-fs-read=./src \
  --allow-fs-read=./package.json \
  --allow-fs-read=./tsconfig.json \
  --allow-fs-write=./dist \
  scripts/build.js
```

핵심은 `--allow-fs-read=*`와 `--allow-fs-write=*`를 기본값처럼 쓰지 않는 것입니다.
빌드가 실제로 읽는 입력과 실제로 쓰는 출력만 먼저 허용하고, 실패가 발생할 때마다 필요한 경로인지 검토한 뒤 추가합니다.

## 파일 읽기 권한 설계하기

### 소스와 설정 파일을 분리해서 허용한다

빌드 스크립트가 읽는 대상은 보통 세 종류입니다.
애플리케이션 소스, 패키지 메타데이터, 빌드 도구 설정입니다.
이 셋을 한 번에 저장소 전체 읽기로 열기보다 각각 명시하면 나중에 권한이 왜 필요한지 리뷰하기 쉽습니다.

```bash
node \
  --permission \
  --allow-fs-read=./src \
  --allow-fs-read=./package.json \
  --allow-fs-read=./package-lock.json \
  --allow-fs-read=./tsconfig.json \
  --allow-fs-read=./vite.config.ts \
  --allow-fs-write=./dist \
  scripts/build.js
```

이 방식은 약간 길어 보이지만, CI 설정 자체가 빌드 입력 목록 역할을 합니다.
새 설정 파일이 추가되어 빌드가 실패하면 "이 파일을 읽어도 되는가"를 코드 리뷰에서 확인할 수 있습니다.

### 존재하지 않는 디렉터리에는 와일드카드를 명시한다

Node.js 문서에 따르면 허용 경로가 실제 디렉터리로 존재하면 디렉터리 하위까지 포함되도록 처리됩니다.
하지만 아직 존재하지 않는 디렉터리를 허용할 때는 의도한 범위를 더 명확히 적는 편이 안전합니다.
예를 들어 빌드가 `./generated`를 새로 만들고 그 아래 파일을 읽어야 한다면 아래처럼 하위 경로를 명시합니다.

```bash
node \
  --permission \
  --allow-fs-read=./src \
  --allow-fs-read=./generated/* \
  --allow-fs-write=./generated \
  --allow-fs-write=./dist \
  scripts/build.js
```

경로 권한을 정할 때는 CI 작업 디렉터리를 기준으로 생각해야 합니다.
로컬에서는 통과하지만 CI에서는 실패하는 경우가 있다면 `pwd`, checkout 경로, 빌드 도구가 쓰는 임시 경로를 먼저 확인합니다.

## 파일 쓰기 권한 설계하기

### 산출물 디렉터리만 쓰기 허용한다

쓰기 권한은 읽기 권한보다 더 보수적으로 잡는 편이 좋습니다.
빌드가 소스 디렉터리에 파일을 다시 쓰거나 설정 파일을 수정한다면 재현 가능한 빌드인지 다시 봐야 합니다.
일반적인 웹 빌드라면 `dist`, `build`, `.next`, `coverage`처럼 산출물 디렉터리만 쓰기 허용합니다.

```bash
node \
  --permission \
  --allow-fs-read=./src \
  --allow-fs-read=./package.json \
  --allow-fs-read=./tsconfig.json \
  --allow-fs-write=./dist \
  --allow-fs-write=./coverage \
  scripts/build-and-test.js
```

테스트가 스냅샷을 갱신하거나 코드 생성 결과를 저장하는 작업은 별도 CI job으로 분리하는 것이 좋습니다.
일반 검증 job에서는 쓰기 위치를 제한하고, 갱신 job에서만 필요한 경로를 열면 변경 의도가 더 분명해집니다.

### 임시 파일 경로를 별도로 관리한다

빌드 도구는 임시 파일을 만들 수 있습니다.
이때 시스템 전체 임시 디렉터리를 넓게 열기보다 프로젝트 내부의 임시 디렉터리를 지정하는 편이 관리하기 쉽습니다.

```bash
mkdir -p .tmp-build

TMPDIR="$PWD/.tmp-build" node \
  --permission \
  --allow-fs-read=./src \
  --allow-fs-read=./package.json \
  --allow-fs-write=./dist \
  --allow-fs-write=./.tmp-build \
  scripts/build.js
```

임시 디렉터리는 CI 종료 후 지워도 되는 데이터만 담아야 합니다.
로그에는 임시 파일의 내용이나 환경 변수 값을 남기지 말고, 경로와 작업 식별자 정도만 남깁니다.

## child_process와 도구 실행 권한

### 외부 명령 호출은 별도 권한으로 검토한다

빌드 스크립트가 `child_process`로 `git`, `npm`, `docker`, `esbuild` 같은 명령을 실행한다면 파일 권한만으로는 충분하지 않습니다.
Permission Model에서는 자식 프로세스 실행도 제한 대상이므로 필요한 경우 `--allow-child-process`를 명시해야 합니다.

```bash
node \
  --permission \
  --allow-fs-read=./src \
  --allow-fs-read=./package.json \
  --allow-fs-write=./dist \
  --allow-child-process \
  scripts/build.js
```

다만 이 권한은 영향 범위가 큽니다.
자식 프로세스는 별도의 실행 환경을 만들 수 있으므로, 단순히 빌드를 통과시키기 위해 바로 추가하기보다 스크립트가 왜 외부 명령을 호출하는지 먼저 확인해야 합니다.
가능하다면 Node.js API나 빌드 도구의 라이브러리 인터페이스로 대체하고, 꼭 필요한 호출만 남깁니다.

### 런타임 권한 확인을 방어 코드로 활용한다

Node.js는 `process.permission.has()`로 현재 프로세스가 특정 권한을 갖는지 확인할 수 있습니다.
빌드 스크립트 초반에 필요한 권한을 점검하면 실패 원인을 더 친절하게 보여줄 수 있습니다.

```js
function requirePermission(scope, reference) {
  if (!process.permission?.has(scope, reference)) {
    throw new Error(`Missing permission: ${scope} ${reference ?? ''}`.trim());
  }
}

requirePermission('fs.read', './src');
requirePermission('fs.read', './package.json');
requirePermission('fs.write', './dist');
```

이 코드는 보안 장벽 자체라기보다 운영자 경험을 돕는 검증입니다.
실제 제한은 `node --permission` 실행 플래그가 담당하고, 스크립트 내부 검증은 어떤 권한이 빠졌는지 빠르게 알려주는 역할을 합니다.

## CI에 단계적으로 도입하는 절차

### 먼저 audit 성격의 job으로 관찰한다

기존 CI에 Permission Model을 바로 강제하면 예상보다 많은 빌드가 깨질 수 있습니다.
처음에는 별도 job으로 추가해 실패 로그를 모으고, 실제로 필요한 경로 목록을 정리합니다.
권한 목록이 안정되면 필수 검증 job으로 승격합니다.

```yaml
name: permission-model-check

on:
  pull_request:

jobs:
  build-with-permissions:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 26
          cache: npm
      - run: npm ci
      - run: |
          node \
            --permission \
            --allow-fs-read=./src \
            --allow-fs-read=./package.json \
            --allow-fs-read=./package-lock.json \
            --allow-fs-read=./tsconfig.json \
            --allow-fs-write=./dist \
            scripts/build.js
```

이 예시는 가장 단순한 형태입니다.
실제 프로젝트에서는 테스트 설정, public assets, 코드 생성 입력, 커버리지 출력 등 필요한 경로를 더 추가할 수 있습니다.

### 실패 로그를 권한 계약으로 바꾼다

권한 부족으로 실패한 로그가 나오면 바로 `*`를 추가하지 않습니다.
아래 순서로 판단합니다.

1. 접근한 파일이 빌드 입력이나 출력으로 타당한가?
2. 저장소 밖 경로라면 캐시, 임시 파일, 도구 설치 위치 중 무엇인가?
3. 민감정보나 개인 경로가 로그에 노출되지 않았는가?
4. 더 좁은 경로로 허용할 수 있는가?
5. 권한 추가가 반복된다면 빌드 스크립트 구조를 나눌 수 있는가?

이 과정을 거치면 CI 설정이 단순한 실행 명령이 아니라 빌드 보안 문서가 됩니다.
새로운 도구나 설정 파일이 들어올 때도 권한 변경 diff를 통해 영향 범위를 확인할 수 있습니다.

## 한계와 주의할 점

### Permission Model은 샌드박스 전체를 대체하지 않는다

Permission Model은 Node.js 프로세스의 권한을 줄이는 데 유용하지만, 운영체제 수준의 컨테이너 격리나 CI secret 정책을 대체하지 않습니다.
공식 문서도 파일 시스템 접근 제한이 `node:fs` 중심으로 동작하며 모든 우회 가능성을 보장하지 않는다고 설명합니다.
따라서 민감한 secret은 애초에 불필요한 job에 주입하지 않는 것이 우선입니다.

또한 `--allow-child-process`처럼 넓은 권한을 열면 제한 효과가 줄어들 수 있습니다.
외부 명령이 꼭 필요하다면 별도 job으로 분리하고, 해당 job에만 필요한 secret과 파일 권한을 줍니다.

### Node.js 버전에 맞춰 플래그를 확인한다

Permission Model은 Node.js 버전에 따라 플래그 이름과 안정성 설명이 달라질 수 있습니다.
예전 문서에서는 `--experimental-permission` 형태가 보이고, 최신 문서에서는 `--permission` 사용을 기준으로 설명됩니다.
CI에서는 `actions/setup-node`, Docker 이미지, 로컬 개발 버전을 맞추고, 버전 변경 시 권한 관련 릴리스 노트를 확인해야 합니다.

```bash
node --version
node --help | grep -E "permission|allow-fs"
```

이 두 줄만 CI 진단 로그에 남겨도 나중에 "어느 Node.js 버전에서 어떤 권한 플래그를 썼는지"를 확인하기 쉬워집니다.

## 실무 체크리스트

### 도입 전에 확인할 것

- 빌드 입력 디렉터리와 설정 파일 목록을 정리한다.
- 빌드 출력 디렉터리와 임시 파일 디렉터리를 분리한다.
- `child_process`, worker, native addon 사용 여부를 확인한다.
- CI job별로 필요한 secret을 최소화한다.
- 실패 로그에 토큰, 쿠키, 개인 경로가 그대로 남지 않게 한다.

### 도입 후에 유지할 것

- 권한 목록 변경은 코드 리뷰에서 별도 확인한다.
- `--allow-fs-read=*`, `--allow-fs-write=*`는 예외 사유가 있을 때만 쓴다.
- Node.js 버전 업그레이드 시 Permission Model 문서를 다시 확인한다.
- 테스트, 빌드, 코드 생성 job을 분리해 권한 범위를 작게 유지한다.
- 권한 부족 실패를 무시하지 말고 빌드 계약 변화로 기록한다.

## FAQ

### Permission Model을 켜면 모든 보안 문제가 해결되나?

아닙니다.
Permission Model은 Node.js 프로세스의 일부 권한을 제한하는 실행 옵션입니다.
CI secret 주입 범위, 브랜치 보호, 의존성 검증, 컨테이너 격리, 로그 마스킹과 함께 써야 합니다.

### 빌드가 자꾸 실패하면 `--allow-fs-read=*`를 써도 되나?

짧은 진단 단계에서는 쓸 수 있지만, 최종 CI 설정에는 좁은 경로를 권장합니다.
계속 넓은 권한이 필요하다면 빌드 스크립트가 너무 많은 일을 하고 있는지 확인하는 편이 좋습니다.

### 로컬 개발에서도 같은 명령을 써야 하나?

가능하면 로컬에서도 같은 명령을 제공하는 것이 좋습니다.
예를 들어 `npm run build:restricted`처럼 별도 스크립트를 만들면 CI에서 깨지기 전에 개발자가 권한 문제를 재현할 수 있습니다.

## 마무리

Node.js Permission Model은 CI 빌드 스크립트를 더 명시적으로 만드는 도구입니다.
가장 큰 장점은 "이 빌드는 무엇을 읽고 어디에 쓰는가"를 실행 명령에 드러낸다는 점입니다.
처음부터 완벽한 권한 목록을 만들 필요는 없습니다.
작은 빌드 job 하나에 `--permission`을 적용하고, 실패 로그를 보며 읽기와 쓰기 경로를 좁혀가면 됩니다.

빌드 보안은 거창한 장치 하나로 끝나지 않습니다.
lockfile로 입력을 고정하고, 환경 변수를 정리하고, CI job 권한을 나누고, Permission Model로 실행 범위를 제한할 때 재현 가능성과 안전성이 함께 올라갑니다.

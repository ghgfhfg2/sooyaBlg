---
layout: post
title: "Node.js --run 가이드: npm 없이 package.json 스크립트 빠르게 실행하는 법"
date: 2026-09-01 20:00:00 +0900
lang: ko
translation_key: nodejs-run-package-script-guide
permalink: /development/blog/seo/2026/09/01/nodejs-run-package-script-guide.html
alternates:
  ko: /development/blog/seo/2026/09/01/nodejs-run-package-script-guide.html
  x_default: /development/blog/seo/2026/09/01/nodejs-run-package-script-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, cli, package-json, npm-scripts, node-run, ci, build]
description: "Node.js --run으로 package.json scripts를 실행하는 방법을 정리합니다. npm run과의 차이, pre/post 스크립트 제한, 환경 변수, CI 적용 기준을 실무 예제로 설명합니다."
---

Node.js 프로젝트에서 `package.json`의 `scripts`를 실행할 때 보통 `npm run test`, `npm run build`를 사용합니다.
하지만 단순히 현재 패키지의 스크립트 하나를 빠르게 실행하려는 목적이라면 Node.js 자체 CLI의 `--run` 옵션도 선택지가 됩니다.

공식 문서 기준으로 `node --run <script>`는 `package.json`의 `scripts` 항목을 찾아 실행합니다.
Node.js 22.9.0과 20.18.0에서 더 이상 experimental이 아닌 옵션으로 표시되었고, npm이나 pnpm 같은 패키지 매니저의 모든 동작을 대체하려는 기능은 아닙니다.
이 글에서는 `node --run`을 어디에 쓰면 좋은지, `npm run`과 무엇이 다른지, CI와 로컬 개발 스크립트에서 어떤 기준으로 고르면 좋은지 정리합니다.

런타임 플래그 확인은 [Node.js process.execArgv 가이드](/development/blog/seo/2026/08/13/nodejs-process-execargv-runtime-flags-guide.html), `.env` 파일 로딩은 [Node.js process.loadEnvFile 가이드](/development/blog/seo/2026/08/14/nodejs-process-loadenvfile-env-file-guide.html), CLI 옵션 파싱은 [Node.js util.parseArgs 가이드](/development/blog/seo/2026/08/09/nodejs-util-parseargs-cli-options-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js Command-line API 공식 문서](https://nodejs.org/api/cli.html), [package.json scripts 실행 옵션 문서](https://nodejs.org/api/cli.html#--run)

## node --run이 하는 일

### package.json scripts를 Node.js CLI에서 바로 실행한다

`node --run`은 현재 작업 디렉터리 기준으로 `package.json`을 읽고, `scripts`에 정의된 이름을 실행합니다.
예를 들어 아래처럼 `test`, `lint`, `build`가 있다면 각 스크립트를 Node.js 명령으로 직접 호출할 수 있습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "lint": "eslint .",
    "build": "vite build"
  }
}
```

```bash
node --run test
node --run lint
node --run build
```

이 방식의 장점은 의도가 선명하다는 점입니다.
"패키지 매니저 기능 전체"가 아니라 "현재 패키지의 스크립트 하나 실행"이 목적일 때 명령이 짧고 예측 가능해집니다.

특히 Node.js 자체 기능으로 구성된 스크립트라면 잘 어울립니다.
예를 들어 `node --test`, `node --watch`, `node --env-file`, `node --check`처럼 Node.js CLI 옵션만 조합한 프로젝트에서는 패키지 매니저 호출을 한 단계 줄일 수 있습니다.

### 인자는 -- 뒤에 붙인다

스크립트에 추가 인자를 넘길 때는 `--` 뒤에 작성합니다.
이 규칙은 npm script를 사용할 때와 비슷하게 보이지만, 실제로는 Node.js CLI가 `--run` 뒤의 script 이름과 추가 인자를 구분하기 위한 방식입니다.

```bash
node --run test -- --test-name-pattern "user service"
node --run lint -- --fix
node --run build -- --mode production
```

`package.json`에는 공통 명령을 짧게 두고, 실행 시점마다 필요한 세부 옵션만 덧붙이면 됩니다.

```json
{
  "scripts": {
    "test": "node --test",
    "check": "node --check src/index.js"
  }
}
```

```bash
node --run test -- --test-reporter spec
```

팀 문서에는 "추가 인자는 `--` 뒤에 둔다"는 규칙을 명시하는 편이 좋습니다.
작은 차이지만 CI 설정, README, 로컬 실행 예제가 서로 어긋나지 않게 만드는 기준이 됩니다.

## npm run과 다른 점

### pre/post 스크립트를 자동 실행하지 않는다

`node --run`은 의도적으로 제한된 기능입니다.
공식 문서에서도 `npm run`이나 다른 패키지 매니저의 run 명령과 같은 동작을 목표로 하지 않는다고 설명합니다.
대표적인 차이는 `pre<script>`와 `post<script>`를 자동으로 실행하지 않는다는 점입니다.

아래 구성을 보겠습니다.

```json
{
  "scripts": {
    "prebuild": "node scripts/clean.js",
    "build": "vite build",
    "postbuild": "node scripts/report-build.js"
  }
}
```

`npm run build`에 익숙한 팀이라면 `prebuild`, `build`, `postbuild`가 순서대로 실행된다고 기대할 수 있습니다.
하지만 `node --run build`는 `build` 스크립트 자체를 실행하는 데 초점을 둡니다.
사전 정리와 후속 리포트가 필요하다면 `build` 명령 안에서 명시적으로 연결해야 합니다.

```json
{
  "scripts": {
    "build": "node scripts/clean.js && vite build && node scripts/report-build.js"
  }
}
```

이 차이는 장점이 될 수도 있습니다.
숨은 자동 실행이 줄어들면 CI에서 어떤 명령이 실제로 실행되는지 추적하기 쉽습니다.
반대로 기존 프로젝트가 `pre`와 `post` 훅에 많이 의존한다면 바로 바꾸지 않는 편이 안전합니다.

### 패키지 매니저 전용 환경 변수를 기대하면 안 된다

`node --run`은 일부 실행 환경 변수를 제공합니다.
문서 기준으로 실행 중인 스크립트 이름은 `NODE_RUN_SCRIPT_NAME`, 처리된 `package.json` 경로는 `NODE_RUN_PACKAGE_JSON_PATH`로 전달됩니다.

```js
console.log({
  scriptName: process.env.NODE_RUN_SCRIPT_NAME,
  packageJson: process.env.NODE_RUN_PACKAGE_JSON_PATH
});
```

반면 `npm_package_name`, `npm_lifecycle_event`처럼 npm이 넣어 주는 전용 환경 변수에 의존하는 스크립트는 그대로 옮기기 어렵습니다.
빌드 스크립트가 그런 값을 읽고 있다면 먼저 의존성을 확인해야 합니다.

```js
const scriptName =
  process.env.NODE_RUN_SCRIPT_NAME ??
  process.env.npm_lifecycle_event ??
  'unknown';
```

마이그레이션 초기에는 위처럼 양쪽 환경 변수를 모두 허용할 수 있습니다.
다만 장기적으로는 특정 패키지 매니저 전용 값에 기대기보다 스크립트 인자나 명시적인 환경 변수로 계약을 정리하는 편이 좋습니다.

## 실무 적용 패턴

### Node.js 내장 테스트 러너와 잘 맞는다

`node --run`이 가장 깔끔하게 맞는 곳은 Node.js 내장 도구를 그대로 호출하는 스크립트입니다.
테스트 러너를 예로 들면 `package.json`을 아래처럼 단순하게 유지할 수 있습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:watch": "node --watch --test",
    "test:coverage": "node --test --experimental-test-coverage"
  }
}
```

```bash
node --run test
node --run test:watch
node --run test -- --test-name-pattern "auth"
```

테스트 스크립트가 Node.js CLI 옵션 중심이면 실행 경로가 짧습니다.
패키지 매니저가 꼭 필요한 의존성 설치 단계와, 이미 설치된 의존성 위에서 스크립트를 실행하는 단계를 분리하기도 쉽습니다.

CI에서는 다음처럼 설치는 패키지 매니저가 맡고, 실행은 Node.js CLI가 맡는 구조를 둘 수 있습니다.

```yaml
steps:
  - run: npm ci
  - run: node --run test
  - run: node --run build
```

이 구성이 항상 더 낫다는 뜻은 아닙니다.
워크스페이스, 캐시, 패키지 매니저 플러그인, 사내 레지스트리 정책이 복잡한 저장소라면 기존 run 명령을 유지하는 편이 더 자연스러울 수 있습니다.
핵심은 `node --run`을 "간단한 단일 패키지 스크립트 실행기"로 보는 것입니다.

### 로컬 개발 명령을 짧게 고정한다

개발자가 매일 치는 명령은 짧고 일관될수록 좋습니다.
`node --run dev`, `node --run test`, `node --run lint`처럼 기본 명령을 통일하면 README와 자동화 문서도 단순해집니다.

```json
{
  "scripts": {
    "dev": "node --env-file=.env.local server.js",
    "lint": "eslint src test",
    "format": "prettier --check .",
    "typecheck": "tsc --noEmit"
  }
}
```

```bash
node --run dev
node --run format
node --run typecheck
```

여기서 주의할 점은 `.env` 파일 로딩입니다.
공식 문서 기준으로 `--env-file`로 로드한 환경 변수는 `--run`으로 실행되는 명령에 적용되지 않습니다.
따라서 환경 파일이 필요한 경우에는 `node --env-file=.env.local --run dev`처럼 바깥 Node.js 명령에 붙이기보다, 실제 실행되는 script 안에서 필요한 로딩 방식을 명확히 정해야 합니다.

```json
{
  "scripts": {
    "dev": "node --env-file=.env.local server.js"
  }
}
```

이렇게 두면 `dev` 스크립트의 계약이 파일 안에 남습니다.
누가 어떤 도구로 실행하든 개발 서버가 같은 환경 파일을 사용한다는 점이 분명해집니다.

## 도입 전 점검 체크리스트

### 기존 scripts의 숨은 의존성을 확인한다

기존 프로젝트에 바로 `npm run`을 `node --run`으로 치환하기 전에 아래 항목을 확인해야 합니다.

- `pre<script>` 또는 `post<script>`에 필수 작업이 있는가?
- `npm_lifecycle_event`, `npm_package_*` 환경 변수를 읽는가?
- 워크스페이스 루트에서 하위 패키지 스크립트를 실행하는가?
- 패키지 매니저 플러그인이나 shell 설정에 기대는가?
- `npm run`의 출력 형식이나 종료 코드 동작을 CI가 전제로 삼는가?

하나라도 해당된다면 전체 치환보다 일부 스크립트부터 적용하는 편이 좋습니다.
예를 들어 `test`, `check`, `format`처럼 독립적인 명령만 먼저 바꾸고, 배포나 릴리스처럼 부작용이 큰 스크립트는 기존 방식을 유지할 수 있습니다.

### 문서에는 선택 기준을 같이 남긴다

팀에서 두 가지 실행 방식을 함께 쓰면 "언제 무엇을 써야 하는지"가 중요합니다.
README에는 명령만 나열하기보다 기준을 짧게 남기는 편이 좋습니다.

```md
## Scripts

- Use `node --run test` for local test runs.
- Use `node --run lint -- --fix` when passing extra script arguments.
- Keep `npm run release` for release automation because it depends on npm lifecycle behavior.
```

이 정도만 있어도 새로 합류한 개발자가 실행 방식을 추측하지 않아도 됩니다.
자동화 스크립트가 의도적으로 `npm run`을 유지하는 이유도 함께 드러납니다.

## FAQ

### node --run은 npm run을 완전히 대체하나요?

아닙니다.
`node --run`은 package.json scripts를 실행하는 가벼운 Node.js CLI 기능입니다.
패키지 매니저의 lifecycle 훅, 워크스페이스 기능, 전용 환경 변수까지 모두 대체한다고 보면 안 됩니다.

### CI에서 바로 써도 되나요?

단일 패키지의 단순 스크립트라면 CI에서도 쓸 수 있습니다.
다만 기존 CI가 `pre`/`post` 스크립트나 npm 전용 환경 변수에 기대고 있다면 먼저 확인해야 합니다.

### 추가 인자는 어떻게 넘기나요?

스크립트 이름 뒤에 `--`를 쓰고 그 뒤에 인자를 붙입니다.
예를 들어 `node --run test -- --test-name-pattern "user"`처럼 실행합니다.

## 마무리

`node --run`은 복잡한 패키지 매니저 기능을 대신하는 도구가 아니라, 단순한 `package.json` 스크립트를 Node.js 자체 CLI로 실행하는 선택지입니다.
Node.js 내장 테스트 러너, 문법 검사, 로컬 개발 서버처럼 실행 계약이 단순한 스크립트에는 잘 맞습니다.

도입할 때는 `npm run`과의 차이를 먼저 확인해야 합니다.
`pre`/`post` 훅, npm 전용 환경 변수, 워크스페이스 실행 흐름이 없다면 `node --run`은 개발자 명령과 CI 스텝을 조금 더 직접적으로 만들 수 있습니다.

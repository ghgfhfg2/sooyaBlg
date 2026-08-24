---
layout: post
title: "Node.js TypeScript type stripping 안정화 가이드: 빌드 없는 .ts 실행을 운영에 적용하는 기준"
date: 2026-08-25 08:00:00 +0900
lang: ko
translation_key: nodejs-typescript-type-stripping-stable-runtime-guide
permalink: /development/blog/seo/2026/08/25/nodejs-typescript-type-stripping-stable-runtime-guide.html
alternates:
  ko: /development/blog/seo/2026/08/25/nodejs-typescript-type-stripping-stable-runtime-guide.html
  x_default: /development/blog/seo/2026/08/25/nodejs-typescript-type-stripping-stable-runtime-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, typescript, type-stripping, tsconfig, runtime, build, ci, javascript]
description: "Node.js TypeScript type stripping 안정화 이후 .ts 파일을 빌드 없이 실행할 때의 적용 기준을 정리합니다. erasable syntax, tsconfig 설정, import type, CI 타입 검사, 운영 배포 체크리스트까지 실무 예제로 설명합니다."
---

Node.js에서 TypeScript 파일을 실행하는 선택지가 더 단순해졌습니다.
공식 문서 기준으로 Node.js의 TypeScript type stripping은 Node.js 24.12.0과 25.2.0에서 안정화되었고, 타입만 제거할 수 있는 `.ts`, `.mts`, `.cts` 파일은 별도 로더 없이 실행할 수 있습니다.
작은 자동화 스크립트나 내부 CLI를 다룰 때는 `ts-node`, `tsx`, 번들러를 붙이기 전에 이 기본 기능만으로 충분한지 검토할 만합니다.

하지만 type stripping은 TypeScript 컴파일러가 아닙니다.
Node.js는 타입 문법을 공백으로 바꿔 실행할 뿐이고, 타입 검사를 하지 않으며, `tsconfig.json`의 path alias나 downlevel transform도 적용하지 않습니다.
따라서 "빌드가 필요 없는 실행 경로"로는 유용하지만, "TypeScript 프로젝트 전체를 Node.js가 대신 빌드한다"로 이해하면 운영에서 깨지기 쉽습니다.

이 글에서는 Node.js TypeScript type stripping을 운영 코드에 적용할 때의 기준을 정리합니다.
기본 실행법은 [Node.js TypeScript 타입 스트리핑 가이드](/development/blog/seo/2026/05/13/nodejs-typescript-type-stripping-runtime-guide.html), 코드 문자열 변환 API는 [Node.js stripTypeScriptTypes 가이드](/development/blog/seo/2026/07/22/nodejs-module-striptypescripttypes-runtime-transform-guide.html), 개발 중 자동 재시작은 [Node.js watch 모드 가이드](/development/blog/seo/2026/05/13/nodejs-watch-mode-auto-restart-guide.html)와 함께 보면 좋습니다.

## Type stripping을 선택해도 되는 경우

### 작은 실행 도구에서 빌드 단계를 줄인다

Type stripping이 가장 잘 맞는 곳은 애플리케이션 본체보다 주변 도구입니다.
예를 들어 릴리스 노트 생성, fixture 정리, 로컬 데이터 점검, 배포 전 검증처럼 짧은 스크립트는 타입을 붙이고 싶지만 별도 `dist` 디렉터리까지 만들고 싶지 않은 경우가 많습니다.

```ts
// scripts/check-release.ts
type ReleaseCheck = {
  version: string;
  requiredFiles: string[];
};

const check: ReleaseCheck = {
  version: '1.8.0',
  requiredFiles: ['CHANGELOG.md', 'package.json']
};

for (const file of check.requiredFiles) {
  console.log(`check ${check.version}: ${file}`);
}
```

최신 Node.js 런타임에서는 이런 파일을 바로 실행할 수 있습니다.

```bash
node scripts/check-release.ts
```

이 흐름의 장점은 실행 경로가 짧다는 점입니다.
개발자는 TypeScript 문법으로 의도를 표시하고, Node.js는 실행에 필요 없는 타입 부분만 제거합니다.
빌드 결과물이 없으니 스크립트 위치, sourcemap, 출력 디렉터리 동기화 문제도 줄어듭니다.

### 서비스 본체에는 더 보수적으로 적용한다

서비스 본체를 모두 `.ts` 직접 실행으로 바꾸는 것은 별도 판단이 필요합니다.
배포 환경의 Node.js 버전, 테스트 러너, 번들링 전략, sourcemap 정책, 성능 측정 방식이 함께 바뀔 수 있기 때문입니다.
특히 서버리스, 컨테이너, 로컬 개발 환경의 Node.js 버전이 다르면 "내 컴퓨터에서는 실행되지만 배포에서는 실패"하는 상황이 생길 수 있습니다.

처음 도입한다면 아래 순서가 안전합니다.

- `scripts/` 같은 내부 자동화부터 적용한다.
- CI에서 Node.js 버전을 고정한다.
- `tsc --noEmit`을 별도 타입 검사 단계로 유지한다.
- 애플리케이션 엔트리포인트는 기존 빌드 흐름을 유지한다.
- 운영 런타임에서 직접 실행할 파일만 명확히 문서화한다.

핵심은 적용 범위를 좁히는 것입니다.
Type stripping은 "작은 실행 경로를 가볍게 만드는 기능"으로 시작하고, 서비스 본체로 확장할지는 측정과 배포 조건을 보고 결정하는 편이 좋습니다.

## 지원되는 TypeScript 문법을 구분하기

### erasable syntax만 통과한다고 생각한다

Node.js type stripping은 JavaScript 코드 생성이 필요 없는 TypeScript 문법에 맞춰져 있습니다.
공식 문서에서는 이런 범위를 erasable TypeScript syntax로 설명합니다.
즉 타입 annotation, type alias, interface처럼 지워도 JavaScript 실행 의미가 유지되는 문법은 잘 맞습니다.

```ts
type User = {
  id: string;
  name: string;
};

interface Logger {
  info(message: string): void;
}

export function formatUser(user: User, logger?: Logger): string {
  logger?.info(`format user ${user.id}`);
  return `${user.name} (${user.id})`;
}
```

이 코드는 타입 정보만 제거하면 JavaScript로 실행할 수 있습니다.
Node.js가 타입을 검사하지는 않지만, 실행에 필요한 값 코드는 그대로 남습니다.

반대로 TypeScript 문법이 JavaScript 값으로 변환되어야 한다면 조심해야 합니다.
대표적으로 `enum`, 런타임 값을 담는 `namespace`, parameter property, import alias 같은 문법은 단순 삭제만으로는 의미가 유지되지 않습니다.

```ts
// type stripping 실행 경로에는 부적합한 예
enum Status {
  Ready = 'ready',
  Done = 'done'
}

class Job {
  constructor(public id: string) {}
}
```

이런 문법을 계속 쓰고 싶다면 `tsx`, `tsc`, `swc`, `esbuild` 같은 변환 도구가 더 맞습니다.
Node.js 기본 type stripping으로 운영하려면 "타입만 지울 수 있는 코드"라는 제약을 팀 규칙으로 받아들여야 합니다.

### import type을 습관으로 만든다

Type stripping에서는 타입 전용 import를 명확히 표시해야 합니다.
Node.js는 `import` 문을 값 import로 해석하므로, 타입만 가져오는 이름에 `type` 키워드를 붙이지 않으면 런타임에서 실제 모듈 export를 찾으려 할 수 있습니다.

```ts
import type { User } from './types.ts';
import { createLogger, type LoggerOptions } from './logger.ts';

const logger = createLogger({
  level: 'info'
} satisfies LoggerOptions);

export function label(user: User): string {
  logger.info(user.id);
  return user.id;
}
```

이 패턴은 빌드 도구를 쓰는 프로젝트에서도 좋은 습관입니다.
타입 의존성과 런타임 의존성이 분리되어 번들링, 테스트, 순환 참조 진단이 쉬워집니다.

`tsconfig.json`에서는 `verbatimModuleSyntax`를 켜서 TypeScript 컴파일러의 해석도 Node.js 실행 방식에 가깝게 맞출 수 있습니다.
타입 import를 빠뜨렸을 때 CI에서 더 빨리 잡기 위한 장치입니다.

## tsconfig는 실행이 아니라 점검을 위해 둔다

### Node.js는 tsconfig를 읽지 않는다

중요한 운영 포인트는 Node.js가 `tsconfig.json`을 실행 설정으로 사용하지 않는다는 점입니다.
`paths`, `baseUrl`, `target`, `module` 같은 옵션을 Node.js가 읽어서 import 경로나 문법을 바꿔주지 않습니다.
이 옵션들은 TypeScript 컴파일러와 에디터, CI 타입 검사에 영향을 줍니다.

따라서 type stripping용 프로젝트에서는 `tsconfig.json`을 "Node.js 실행 설정"이 아니라 "타입 검사와 작성 규칙 검증"으로 이해해야 합니다.

```json
{
  "compilerOptions": {
    "noEmit": true,
    "target": "esnext",
    "module": "nodenext",
    "rewriteRelativeImportExtensions": true,
    "erasableSyntaxOnly": true,
    "verbatimModuleSyntax": true
  }
}
```

이 설정의 의도는 실행 전에 위험한 문법을 줄이는 것입니다.
`erasableSyntaxOnly`는 type stripping으로 지울 수 없는 TypeScript 문법을 피하게 돕고, `rewriteRelativeImportExtensions`는 TypeScript 작성 경험과 Node.js의 확장자 요구를 맞추는 데 유용합니다.
`noEmit`은 직접 `.ts` 파일을 실행하는 스크립트 저장소에서 산출물을 만들지 않겠다는 뜻을 분명히 합니다.

### 상대 import에는 확장자를 쓴다

Node.js ESM 실행에서는 상대 import에 확장자가 필요합니다.
TypeScript 파일도 예외가 아닙니다.

```ts
// 좋음
import { readConfig } from './config.ts';

// 피하기
import { readConfig } from './config';
```

CommonJS 파일에서도 `.cts` 또는 `.ts` 확장자를 명확히 쓰는 편이 좋습니다.
빌드 도구가 없는 실행 경로에서는 해석 규칙을 도구가 보정해 주지 않기 때문입니다.

처음에는 어색할 수 있지만, 직접 실행되는 TypeScript에서는 파일 확장자를 코드 계약으로 보는 편이 낫습니다.
확장자 없는 import를 허용하는 번들러 관성에 기대면 로컬 빌드와 Node.js 직접 실행 사이의 차이가 커집니다.

## 타입 검사와 실행을 분리하기

### node 실행은 타입 검사가 아니다

가장 흔한 오해는 `node file.ts`가 성공하면 타입도 안전하다고 보는 것입니다.
Node.js는 실행 가능한 JavaScript를 얻기 위해 타입 구문을 제거할 뿐, 타입 관계를 검증하지 않습니다.

```ts
type PortConfig = {
  port: number;
};

const config: PortConfig = {
  port: '3000'
};

console.log(config.port);
```

이 코드는 타입 검사에서는 실패해야 하지만, type stripping 실행만 놓고 보면 런타임 값은 그대로 남습니다.
문자열 포트가 실제 서버 코드에 들어가면 전혀 다른 오류로 터질 수 있습니다.

그래서 CI는 최소한 두 단계를 나누는 편이 좋습니다.

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "check:release": "node scripts/check-release.ts",
    "test": "node --test"
  }
}
```

`typecheck`는 타입 계약을 검증하고, `check:release`는 실제 스크립트 동작을 확인합니다.
둘 중 하나만 두면 문제의 종류가 섞입니다.

### 테스트에서도 런타임 버전을 고정한다

Type stripping 지원 범위는 Node.js 버전에 따라 달라질 수 있습니다.
팀이 여러 Node.js 버전을 쓰거나, CI와 배포 이미지가 다르면 테스트 결과가 흔들릴 수 있습니다.

운영 저장소라면 아래 항목을 함께 맞추세요.

- `package.json`의 `engines.node`
- `.nvmrc`, `.node-version`, Volta 설정 같은 로컬 버전 파일
- CI의 Node.js setup 버전
- Docker base image 태그
- 배포 플랫폼의 런타임 버전

버전 정책을 코드 가까이에 두면 새 팀원이 type stripping 실패를 "문법 오류"로 오해하기 전에 런타임 조건을 확인할 수 있습니다.
특히 `node_modules` 안의 TypeScript 파일은 Node.js가 직접 처리하지 않도록 제한되어 있으므로, 의존성은 여전히 패키지가 배포한 JavaScript 엔트리를 바라보는 구조가 안전합니다.

## 운영 적용 패턴

### 내부 스크립트는 직접 실행한다

가벼운 자동화에는 직접 실행이 가장 단순합니다.

```json
{
  "scripts": {
    "docs:links": "node scripts/check-doc-links.ts",
    "release:notes": "node scripts/render-release-notes.ts",
    "predeploy": "npm run typecheck && node scripts/predeploy-check.ts"
  }
}
```

이때 스크립트가 프로젝트 소스 코드를 많이 import한다면 주의해야 합니다.
서비스 코드가 path alias, decorator, enum, 빌드 타임 상수 치환에 의존한다면 type stripping 실행 경로와 맞지 않을 수 있습니다.
내부 스크립트는 가능한 한 표준 Node.js API와 작은 유틸리티에 의존하게 두는 편이 안정적입니다.

### 배포 엔트리포인트는 별도로 판단한다

운영 서버를 `node src/server.ts`로 바로 띄우고 싶다면 다음 질문에 답해야 합니다.

- 배포 런타임이 type stripping 안정화 버전 이상인가
- 모든 서버 코드가 erasable syntax만 쓰는가
- path alias 없이 상대 경로와 package imports로 해결되는가
- 장애 시 stack trace와 관측 도구가 충분히 읽기 쉬운가
- cold start, 메모리, 시작 시간이 기존 빌드 방식과 비교해 문제가 없는가

이 질문에 확신이 없다면 서버 본체는 기존 빌드 방식을 유지하고, 주변 스크립트부터 적용하는 것이 현실적입니다.
서비스 운영에서 중요한 것은 최신 기능 사용 자체가 아니라 장애 진단 가능한 실행 경로입니다.

### full TypeScript 지원이 필요하면 도구를 쓴다

아래 요구가 있으면 Node.js 기본 type stripping만으로는 부족할 가능성이 큽니다.

- decorator를 적극적으로 쓰는 프레임워크
- path alias가 많은 모노레포
- 낮은 JavaScript target으로 변환해야 하는 라이브러리
- enum, parameter property 등 변환 문법을 유지해야 하는 코드베이스
- 번들링, tree shaking, minification이 필요한 배포물

이 경우에는 `tsx`, `tsc`, `swc`, `esbuild` 같은 도구를 명시적으로 선택하세요.
Node.js 기본 기능과 외부 도구는 경쟁 관계라기보다 적용 범위가 다릅니다.
기본 type stripping은 작은 실행 경로를 단순하게 만들고, 전체 TypeScript 빌드는 변환과 배포 최적화를 담당합니다.

## 마이그레이션 체크리스트

### 기존 스크립트 하나부터 바꾼다

한 번에 모든 도구를 바꾸기보다 실패해도 영향이 작은 스크립트 하나를 고르세요.
예를 들어 Markdown 링크 검사나 릴리스 노트 생성처럼 외부 시스템을 변경하지 않는 작업이 좋습니다.

```bash
node --version
npm run typecheck
node scripts/check-doc-links.ts
```

성공 기준은 단순 실행 성공만이 아닙니다.
CI 로그가 읽기 쉬운지, 에러가 발생했을 때 원인 파일과 줄 번호가 충분히 드러나는지, 팀원이 로컬에서 같은 명령을 재현할 수 있는지도 확인해야 합니다.

### 금지 문법을 문서화한다

Type stripping 경로에서는 "쓸 수 없는 TypeScript 문법"을 팀 문서에 적어두는 편이 좋습니다.
특히 아래 항목은 코드 리뷰에서 자주 놓칩니다.

- runtime `enum`
- parameter property
- 값이 있는 `namespace`
- 확장자 없는 상대 import
- `type` 키워드 없는 타입 전용 import
- `tsconfig` paths alias에 의존하는 import

문서만으로 부족하면 ESLint와 `tsc --noEmit` 설정을 함께 사용하세요.
실행 전 정적 점검이 실패하도록 만들면 런타임에서 애매한 오류를 만날 확률이 줄어듭니다.

### 실패 메시지를 운영 친화적으로 만든다

내부 CLI라면 시작 시점에 Node.js 버전과 실행 조건을 확인하는 작은 가드를 둘 수 있습니다.

```ts
const [major, minor] = process.versions.node
  .split('.')
  .map(Number);

if (major < 24 || (major === 24 && minor < 12)) {
  throw new Error('This script requires Node.js 24.12.0 or newer for stable TypeScript type stripping.');
}
```

이런 가드는 모든 파일에 반복해서 넣을 필요는 없습니다.
배포 전 점검 스크립트나 팀 공용 CLI 엔트리포인트처럼 사람이 자주 실행하는 곳에 두면 충분합니다.
오류 메시지는 "문법이 틀렸다"보다 "요구 Node.js 버전이 맞지 않는다"를 바로 알려주는 쪽이 좋습니다.

## 발행 전 점검

### 적용 기준 요약

Node.js TypeScript type stripping은 이제 작은 `.ts` 실행 경로에서 꽤 실용적인 기본값이 되었습니다.
다만 실무에서는 아래 기준을 지키는 것이 중요합니다.

- type stripping은 타입 검사가 아니라 실행 보조 기능이다.
- `tsc --noEmit`은 CI에서 별도로 유지한다.
- erasable syntax만 쓰도록 `tsconfig`와 리뷰 규칙을 맞춘다.
- 상대 import에는 확장자를 명시한다.
- 타입 전용 import에는 `import type`을 쓴다.
- 서비스 본체보다 내부 스크립트부터 적용한다.
- 배포 Node.js 버전을 문서와 CI에서 고정한다.

이 선을 지키면 빌드 없는 TypeScript 실행은 꽤 편리한 도구가 됩니다.
반대로 이 선을 흐리면 "왜 Node.js가 TypeScript를 완전히 빌드해 주지 않지?"라는 혼란이 반복됩니다.
작은 자동화에는 가볍게, 서비스 본체에는 보수적으로 적용하는 것이 좋은 출발점입니다.

## 관련 글

- [Node.js TypeScript 타입 스트리핑 가이드](/development/blog/seo/2026/05/13/nodejs-typescript-type-stripping-runtime-guide.html)
- [Node.js stripTypeScriptTypes 가이드](/development/blog/seo/2026/07/22/nodejs-module-striptypescripttypes-runtime-transform-guide.html)
- [Node.js watch 모드 가이드](/development/blog/seo/2026/05/13/nodejs-watch-mode-auto-restart-guide.html)
- [Node.js module.registerHooks 가이드](/development/blog/seo/2026/06/20/nodejs-module-registerhooks-loader-customization-guide.html)

## 참고 자료

- [Node.js 공식 문서: Modules: TypeScript](https://nodejs.org/api/typescript.html)

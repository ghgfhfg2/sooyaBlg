---
layout: post
title: "Node.js module.findPackageJSON 가이드: 패키지 메타데이터 위치를 안전하게 찾는 법"
date: 2026-06-04 08:00:00 +0900
lang: ko
translation_key: nodejs-module-findpackagejson-package-metadata-guide
permalink: /development/blog/seo/2026/06/04/nodejs-module-findpackagejson-package-metadata-guide.html
alternates:
  ko: /development/blog/seo/2026/06/04/nodejs-module-findpackagejson-package-metadata-guide.html
  x_default: /development/blog/seo/2026/06/04/nodejs-module-findpackagejson-package-metadata-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, module, package-json, esm, commonjs, tooling]
description: "Node.js module.findPackageJSON으로 현재 파일이나 의존성의 package.json 위치를 찾는 방법을 정리했습니다. ESM과 CommonJS 사용법, import.meta.url 기준, 메타데이터 읽기, 버전 호환성과 실무 주의점을 예제로 설명합니다."
---

Node.js 도구를 만들다 보면 현재 실행 중인 파일이 어떤 패키지에 속하는지, 특정 의존성의 `package.json`이 어디 있는지 알아야 할 때가 있습니다.
예를 들어 CLI 버전을 출력하거나, 플러그인의 루트 경로를 찾거나, 패키지 메타데이터를 읽어 기능을 분기하는 경우입니다.

예전에는 `import.meta.url`, `require.resolve()`, `path.dirname()`, 상위 디렉터리 탐색을 직접 조합하는 코드가 많았습니다.
하지만 이런 코드는 ESM과 CommonJS가 섞인 프로젝트, monorepo, package exports가 있는 의존성에서 생각보다 쉽게 복잡해집니다.

Node.js의 `node:module`에는 `findPackageJSON()`이 있습니다.
이 API는 지정한 모듈 또는 경로 기준으로 연결되는 `package.json` 위치를 찾아 줍니다.
이 글에서는 `module.findPackageJSON()`의 기본 사용법, ESM과 CommonJS에서 기준 경로를 잡는 법, 메타데이터를 읽을 때의 안전장치, 실무 적용 체크리스트를 정리합니다.
ESM 경로 처리 자체가 먼저 헷갈린다면 [Node.js import.meta.dirname 가이드](/development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html)를 함께 보면 흐름을 잡기 쉽습니다.

## module.findPackageJSON이 필요한 상황

### H3. package.json 위치 찾기를 직접 구현하지 않아도 된다

`package.json`은 단순한 설정 파일처럼 보이지만, Node.js 프로젝트에서는 중요한 경계 파일입니다.
가까운 `package.json`의 `"type"` 값은 `.js` 파일의 모듈 형식을 바꾸고, 패키지 루트의 메타데이터는 CLI 이름, 버전, exports, dependencies 같은 정보를 담습니다.

문제는 이 파일의 위치를 직접 찾으려 할 때입니다.
현재 파일에서 상위 폴더를 계속 올라가며 `package.json`을 찾는 코드는 처음에는 간단하지만, 아래 요구사항이 붙으면 금방 지저분해집니다.

- ESM에서는 `__dirname`이 없고 `import.meta.url`을 써야 한다.
- CommonJS에서는 `import.meta.url`이 아니라 `__filename`이 자연스럽다.
- 상대 경로가 아니라 패키지 이름으로 의존성의 루트를 찾고 싶다.
- package exports 때문에 내부 파일 경로를 마음대로 추측하고 싶지 않다.
- 테스트에서 fixture 패키지를 기준으로 같은 로직을 재현해야 한다.

`findPackageJSON()`은 이런 작업을 Node.js 모듈 해석 문맥 안에서 더 명시적으로 표현하게 해 줍니다.
직접 디렉터리 탐색을 작성하는 대신 "이 specifier와 base 기준으로 연결되는 package.json을 찾아라"라고 말할 수 있습니다.

### H3. 도구 코드와 애플리케이션 코드를 분리해 생각한다

이 API는 일반적인 비즈니스 로직보다 도구 코드에서 더 자주 빛납니다.
예를 들어 빌드 도구, 코드 생성기, CLI, 플러그인 로더, 테스트 유틸리티처럼 프로젝트 구조를 읽어야 하는 코드입니다.

반대로 단순 웹 서버에서 매 요청마다 패키지 메타데이터를 읽는 용도로 쓰는 것은 적절하지 않을 수 있습니다.
패키지 메타데이터는 보통 프로세스 시작 시 한 번 읽고 캐시하면 충분합니다.
요청 처리 경로에서는 이미 로드된 값만 참조하는 편이 더 예측 가능합니다.

## 기본 사용법

### H3. ESM에서는 import.meta.url을 base로 넘긴다

ESM 파일에서는 현재 모듈의 위치를 `import.meta.url`로 표현합니다.
현재 파일이 속한 가장 가까운 `package.json`을 찾고 싶다면 상대 specifier와 `import.meta.url`을 함께 넘기면 됩니다.

```js
import { findPackageJSON } from 'node:module';

const packageJsonPath = findPackageJSON('.', import.meta.url);

console.log(packageJsonPath);
```

의존성 패키지의 `package.json` 위치를 찾고 싶다면 패키지 이름을 specifier로 넘깁니다.

```js
import { findPackageJSON } from 'node:module';

const reactPackageJson = findPackageJSON('react', import.meta.url);

if (reactPackageJson) {
  console.log(`react package.json: ${reactPackageJson}`);
}
```

여기서 중요한 점은 `base`가 "현재 파일 기준"이라는 것입니다.
도구가 실행되는 작업 디렉터리와 모듈이 위치한 디렉터리는 다를 수 있습니다.
그래서 `process.cwd()`를 무심코 기준으로 쓰기보다, 모듈 기준이 필요한지 작업 디렉터리 기준이 필요한지 먼저 구분해야 합니다.

### H3. CommonJS에서는 __filename을 base로 넘긴다

CommonJS에서는 `__filename`을 기준으로 쓰면 됩니다.
`__dirname`이 익숙하더라도, 이 API에서는 현재 파일의 위치를 나타내는 `__filename`이 더 정확한 기준입니다.

```js
const { findPackageJSON } = require('node:module');

const packageJsonPath = findPackageJSON('.', __filename);

console.log(packageJsonPath);
```

의존성을 찾는 코드도 같은 방식입니다.

```js
const { findPackageJSON } = require('node:module');

function findDependencyPackageJson(name) {
  return findPackageJSON(name, __filename);
}

console.log(findDependencyPackageJson('some-package'));
```

ESM과 CommonJS를 모두 지원하는 라이브러리라면 파일 형식별 entry를 나누는 편이 단순합니다.
한 파일 안에서 모든 런타임 차이를 억지로 흡수하려 하면 경로 처리 코드가 더 복잡해질 수 있습니다.

## package.json 메타데이터 읽기

### H3. 위치 찾기와 JSON 읽기를 분리한다

`findPackageJSON()`은 파일 위치를 찾아 주는 API입니다.
파일 내용을 읽고 파싱하는 작업은 별도로 처리해야 합니다.
이 둘을 분리하면 테스트와 에러 처리가 쉬워집니다.

```js
import { readFile } from 'node:fs/promises';
import { findPackageJSON } from 'node:module';

export async function readOwnPackageMetadata() {
  const packageJsonPath = findPackageJSON('.', import.meta.url);

  if (!packageJsonPath) {
    throw new Error('package.json not found');
  }

  const raw = await readFile(packageJsonPath, 'utf8');
  const metadata = JSON.parse(raw);

  return {
    name: metadata.name,
    version: metadata.version,
    type: metadata.type
  };
}
```

이 예제는 필요한 필드만 반환합니다.
운영 로그나 API 응답에 `package.json` 전체를 그대로 노출하는 것은 피하는 편이 좋습니다.
패키지 메타데이터에는 scripts, repository, private 설정, 내부 패키지 이름처럼 공개할 필요가 없는 정보가 들어 있을 수 있습니다.

### H3. 필드 검증을 작게라도 둔다

`package.json`은 JSON 파일이지만, 원하는 필드가 항상 문자열이라고 보장할 수는 없습니다.
특히 테스트 fixture나 내부 패키지를 읽는 도구라면 검증을 작게라도 두는 편이 좋습니다.

```js
function pickPackageIdentity(metadata) {
  const name = typeof metadata.name === 'string' ? metadata.name : null;
  const version = typeof metadata.version === 'string' ? metadata.version : null;

  if (!name || !version) {
    throw new Error('package metadata must include name and version');
  }

  return { name, version };
}
```

검증은 거창할 필요가 없습니다.
도구가 실제로 의존하는 필드만 확인해도, 잘못된 fixture나 깨진 패키지 때문에 애매한 오류가 나는 일을 줄일 수 있습니다.

## 실무 적용 패턴

### H3. CLI 버전 출력에 사용한다

CLI에서 `--version`을 구현할 때 하드코딩된 문자열을 두면 배포 과정에서 쉽게 어긋납니다.
패키지의 `package.json`에서 버전을 읽으면 릴리스 메타데이터와 출력값을 맞추기 쉽습니다.

```js
import { readFile } from 'node:fs/promises';
import { findPackageJSON } from 'node:module';

let cachedVersion;

export async function getCliVersion() {
  if (cachedVersion) {
    return cachedVersion;
  }

  const packageJsonPath = findPackageJSON('.', import.meta.url);

  if (!packageJsonPath) {
    return '0.0.0';
  }

  const metadata = JSON.parse(await readFile(packageJsonPath, 'utf8'));
  cachedVersion = typeof metadata.version === 'string' ? metadata.version : '0.0.0';

  return cachedVersion;
}
```

한 번 읽은 값은 캐시합니다.
CLI 실행 중 패키지 버전이 바뀌는 경우는 거의 없고, 같은 파일을 반복해서 읽을 이유도 적기 때문입니다.

### H3. 플러그인 루트 확인에 사용한다

플러그인 기반 도구에서는 사용자가 설치한 패키지의 루트 정보를 알아야 할 때가 있습니다.
이때 패키지 내부 파일 경로를 추측하지 말고, 패키지 specifier 기준으로 `package.json` 위치를 먼저 찾으면 루트 계산이 더 명확해집니다.

```js
import { dirname } from 'node:path';
import { findPackageJSON } from 'node:module';

export function findPluginRoot(packageName) {
  const packageJsonPath = findPackageJSON(packageName, import.meta.url);

  if (!packageJsonPath) {
    throw new Error(`plugin package not found: ${packageName}`);
  }

  return dirname(packageJsonPath);
}
```

이 코드는 "플러그인 패키지의 루트는 package.json이 있는 디렉터리"라는 기준을 분명히 드러냅니다.
내장 모듈이나 런타임 선택 의존성을 함께 다뤄야 한다면 [Node.js process.getBuiltinModule 가이드](/development/blog/seo/2026/05/04/nodejs-process-getbuiltinmodule-runtime-optional-dependency-guide.html)와 연결해 보면 좋습니다.

## 주의할 점

### H3. 런타임 버전과 안정성 표기를 확인한다

`module.findPackageJSON()`은 비교적 최근 Node.js에 추가된 API이고, 문서상 안정성 표기가 아직 높은 단계는 아닐 수 있습니다.
프로덕션 도구에 바로 넣기 전에는 팀이 사용하는 Node.js 버전에서 지원되는지 확인해야 합니다.

실무에서는 아래 기준을 문서화하는 편이 좋습니다.

- `package.json`의 `engines.node`에 최소 Node.js 버전을 적는다.
- CI의 Node.js 버전과 로컬 개발 버전을 맞춘다.
- API가 없는 런타임에서는 명확한 오류를 내거나 대체 경로를 둔다.
- 장기 지원 버전 전환 시 관련 도구 코드를 다시 점검한다.

Node.js 버전별 기능 도입을 다룰 때는 [Node.js TypeScript type stripping 가이드](/development/blog/seo/2026/05/13/nodejs-typescript-type-stripping-runtime-guide.html)처럼 실행 조건을 명확히 남기는 습관이 중요합니다.

### H3. package.json 전체를 신뢰하지 않는다

의존성 패키지의 `package.json`은 외부 입력에 가깝게 다루는 편이 안전합니다.
특정 필드가 있다고 가정하지 말고, 필요한 필드만 읽고 타입을 확인하세요.

또한 `scripts`, private registry 정보, 내부 경로 같은 값을 로그에 그대로 남기지 않는 것이 좋습니다.
디버깅 로그에는 패키지 이름, 버전, 찾은 여부 정도만 남겨도 대부분 충분합니다.

## 발행 전 체크리스트

### H3. 도입 전에 확인할 항목

- 현재 프로젝트의 최소 Node.js 버전에서 `module.findPackageJSON()`을 지원하는가?
- ESM에서는 `import.meta.url`, CommonJS에서는 `__filename`을 기준으로 쓰고 있는가?
- 작업 디렉터리 기준과 모듈 파일 기준을 혼동하지 않았는가?
- `package.json` 내용을 읽을 때 필요한 필드만 골라 검증하는가?
- 메타데이터 전체를 로그나 응답으로 노출하지 않는가?
- 반복 호출되는 경로에서는 결과를 캐시하는가?

## FAQ

### H3. findPackageJSON은 package.json 내용을 바로 반환하나요?

아닙니다.
`findPackageJSON()`은 연결되는 `package.json` 파일의 경로를 반환합니다.
JSON 내용을 읽으려면 `fs.readFile()`로 파일을 읽고 `JSON.parse()`를 직접 수행해야 합니다.

### H3. process.cwd()로 직접 package.json을 읽으면 안 되나요?

작업 디렉터리의 루트 `package.json`만 필요하다면 그렇게 해도 됩니다.
하지만 현재 모듈이 속한 패키지나 특정 의존성 패키지를 기준으로 찾아야 한다면 `process.cwd()`는 틀린 기준이 될 수 있습니다.
도구 코드에서는 "작업 디렉터리 기준"과 "모듈 위치 기준"을 분리하는 것이 중요합니다.

### H3. 모든 프로젝트에서 바로 써도 되나요?

아직은 Node.js 버전 확인이 필요합니다.
팀의 최소 지원 버전, CI 이미지, 배포 런타임이 모두 이 API를 지원하는지 먼저 확인하세요.
지원하지 않는 환경이 있다면 기존 경로 탐색 로직을 fallback으로 두거나, 최소 Node.js 버전을 올리는 결정을 명시해야 합니다.

## 정리

`module.findPackageJSON()`은 화려한 기능은 아니지만, Node.js 도구 코드에서 자주 반복되던 패키지 메타데이터 위치 찾기를 더 명확하게 만들어 줍니다.
ESM과 CommonJS의 기준 경로 차이를 인정하고, specifier와 base를 분명히 넘기면 직접 상위 디렉터리를 훑는 코드보다 의도가 잘 드러납니다.

다만 이 API는 파일 내용을 읽어 주는 도구가 아니고, 비교적 최근 API이므로 버전 확인이 필요합니다.
패키지 메타데이터를 읽을 때는 필요한 필드만 검증하고, 민감할 수 있는 내용을 로그에 그대로 남기지 않는 기준도 함께 두세요.

## 함께 읽기

- [Node.js import.meta.dirname 가이드: ESM에서 __dirname 없이 경로 다루는 법](/development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html)
- [Node.js process.getBuiltinModule 가이드: 런타임 선택 의존성을 더 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-process-getbuiltinmodule-runtime-optional-dependency-guide.html)
- [Node.js TypeScript type stripping 가이드: 빌드 없이 타입스크립트 실행하기](/development/blog/seo/2026/05/13/nodejs-typescript-type-stripping-runtime-guide.html)

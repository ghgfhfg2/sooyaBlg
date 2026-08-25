---
layout: post
title: "Node.js module.isBuiltin 가이드: 내장 모듈 specifier를 안전하게 판별하는 법"
date: 2026-08-25 20:00:00 +0900
lang: ko
translation_key: nodejs-module-isbuiltin-specifier-validation-guide
permalink: /development/blog/seo/2026/08/25/nodejs-module-isbuiltin-specifier-validation-guide.html
alternates:
  ko: /development/blog/seo/2026/08/25/nodejs-module-isbuiltin-specifier-validation-guide.html
  x_default: /development/blog/seo/2026/08/25/nodejs-module-isbuiltin-specifier-validation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, module, isbuiltin, builtin-modules, esm, commonjs, security, javascript]
description: "Node.js module.isBuiltin으로 내장 모듈 specifier를 판별하고 동적 import, 플러그인 로딩, allowlist 검증에서 안전하게 사용하는 방법을 정리합니다. node: 접두사, module.builtinModules, process.getBuiltinModule과의 차이까지 실무 예제로 설명합니다."
---

Node.js에서 모듈 이름을 문자열로 다루는 코드는 생각보다 많습니다.
플러그인 로더, 설정 기반 동적 import, 테스트 fixture, CLI 확장 기능, 번들러 보조 스크립트는 사용자가 넘긴 specifier가 실제로 어떤 모듈을 가리키는지 먼저 판단해야 합니다.
이때 "이 이름이 Node.js 내장 모듈인가?"를 확인하는 기본 도구가 `node:module`의 `isBuiltin()`입니다.

`module.isBuiltin()`은 전달한 모듈 이름이 Node.js 내장 모듈이면 `true`, 아니면 `false`를 반환합니다.
`fs`와 `node:fs`처럼 접두사가 있거나 없는 이름을 모두 다룰 수 있어, 단순 문자열 배열을 직접 관리하는 것보다 실수 여지가 적습니다.
다만 이 함수가 보안 정책 전체를 대신해 주지는 않습니다.
내장 모듈 여부 판별과 실제 허용 정책은 별도로 나누어야 합니다.

이 글에서는 `module.isBuiltin()`의 기본 사용법, `node:` 접두사 처리, 동적 import allowlist 설계, `module.builtinModules`와 `process.getBuiltinModule()`의 차이를 실무 기준으로 정리합니다.
런타임에서 내장 모듈을 조건부로 가져오는 흐름은 [Node.js process.getBuiltinModule 가이드](/development/blog/seo/2026/05/04/nodejs-process-getbuiltinmodule-runtime-optional-dependency-guide.html), 로더 훅에서 specifier를 다루는 기준은 [Node.js module.registerHooks 가이드](/development/blog/seo/2026/08/16/nodejs-module-registerhooks-loader-customization-guide.html)와 함께 보면 좋습니다.

## module.isBuiltin이 필요한 이유

### 문자열 specifier를 직접 비교하면 빠뜨리기 쉽다

모듈 specifier는 단순한 파일 경로가 아닙니다.
Node.js 내장 모듈은 `fs`, `node:fs`, `node:fs/promises`처럼 여러 형태로 나타날 수 있습니다.
코드에서 직접 배열을 만들어 비교하면 접두사, 하위 경로, Node.js 버전별 내장 모듈 목록을 놓치기 쉽습니다.

```js
const builtinNames = new Set(['fs', 'path', 'crypto']);

function isAllowedBuiltin(specifier) {
  return builtinNames.has(specifier);
}

console.log(isAllowedBuiltin('fs')); // true
console.log(isAllowedBuiltin('node:fs')); // false
console.log(isAllowedBuiltin('node:fs/promises')); // false
```

이 코드는 의도가 단순하지만 운영에서는 애매합니다.
내장 모듈을 막고 싶은 것인지, 특정 내장 모듈만 허용하고 싶은 것인지, `node:` 접두사를 표준으로 삼고 싶은 것인지가 섞여 있습니다.

`module.isBuiltin()`을 쓰면 1차 판별을 Node.js에 맡길 수 있습니다.

```js
import { isBuiltin } from 'node:module';

console.log(isBuiltin('fs')); // true
console.log(isBuiltin('node:fs')); // true
console.log(isBuiltin('node:fs/promises')); // true
console.log(isBuiltin('left-pad')); // false
```

여기서 중요한 점은 `isBuiltin()`이 "내장 모듈인지"만 말해 준다는 것입니다.
"이 모듈을 가져와도 되는지"는 별도 정책으로 판단해야 합니다.

### 동적 import 경계에서 입력을 분류한다

사용자 설정으로 모듈을 불러오는 코드는 먼저 입력을 분류해야 합니다.
내장 모듈인지, 패키지 의존성인지, 상대 파일인지, URL인지에 따라 위험과 처리 방식이 다르기 때문입니다.

```js
import { isBuiltin } from 'node:module';

function classifySpecifier(specifier) {
  if (isBuiltin(specifier)) {
    return 'builtin';
  }

  if (specifier.startsWith('./') || specifier.startsWith('../')) {
    return 'relative';
  }

  if (specifier.startsWith('file:')) {
    return 'file-url';
  }

  return 'package-or-other';
}
```

이런 분류는 플러그인 로더나 CLI 도구에서 특히 유용합니다.
예를 들어 내장 모듈은 일부만 허용하고, 상대 경로는 프로젝트의 특정 디렉터리 아래로 제한하며, 패키지 이름은 `package.json`의 명시 의존성에 있는지 확인할 수 있습니다.

## node: 접두사를 기준으로 정규화하기

### 내부 표현은 node: 접두사로 맞춘다

Node.js 새 코드에서는 내장 모듈을 `node:fs`, `node:path`, `node:crypto`처럼 `node:` 접두사로 import하는 편이 읽기 쉽습니다.
외부 패키지 이름과 Node.js 내장 모듈 이름이 섞이지 않고, 리뷰할 때도 의도가 분명합니다.

사용자 입력이나 오래된 설정에서는 여전히 `fs`처럼 접두사 없는 이름이 들어올 수 있습니다.
이때 `isBuiltin()`으로 내장 모듈 여부를 확인한 뒤 내부 표현을 `node:` 접두사 형태로 정규화하면 후속 코드가 단순해집니다.

```js
import { isBuiltin } from 'node:module';

function normalizeBuiltinSpecifier(specifier) {
  if (!isBuiltin(specifier)) {
    return null;
  }

  if (specifier.startsWith('node:')) {
    return specifier;
  }

  return `node:${specifier}`;
}

console.log(normalizeBuiltinSpecifier('fs/promises'));
// node:fs/promises

console.log(normalizeBuiltinSpecifier('node:path'));
// node:path
```

이 함수는 내장 모듈이 아닌 값에는 `null`을 반환합니다.
후속 로직은 `null`인지 확인한 뒤 패키지, 파일, URL 정책으로 넘기면 됩니다.

### 허용 목록도 같은 형태로 저장한다

내장 모듈 allowlist를 만든다면 비교 기준을 하나로 맞추는 편이 좋습니다.
입력은 `fs`일 수도 있고 `node:fs`일 수도 있지만, allowlist는 모두 `node:` 접두사 형태로 저장합니다.

```js
const allowedBuiltins = new Set([
  'node:crypto',
  'node:fs/promises',
  'node:path',
  'node:url'
]);

export function assertAllowedBuiltin(specifier) {
  const normalized = normalizeBuiltinSpecifier(specifier);

  if (!normalized) {
    throw new Error(`Not a Node.js builtin module: ${specifier}`);
  }

  if (!allowedBuiltins.has(normalized)) {
    throw new Error(`Builtin module is not allowed: ${normalized}`);
  }

  return normalized;
}
```

이 구조의 장점은 에러 메시지가 일관된다는 점입니다.
`fs`로 들어오든 `node:fs`로 들어오든 최종 로그에는 `node:fs`로 남습니다.
보안 리뷰나 설정 점검에서도 같은 이름으로 검색할 수 있습니다.

## 동적 import와 함께 쓰는 안전한 패턴

### import 전에 허용 정책을 적용한다

`isBuiltin()`을 가장 조심해서 써야 하는 곳은 동적 import입니다.
아래처럼 입력을 그대로 import하면 설정 파일 하나로 프로세스가 예상하지 못한 모듈을 읽게 됩니다.

```js
export async function loadAdapter(specifier) {
  return import(specifier);
}
```

개발 도구 안에서는 편해 보이지만, 운영 코드나 공유 CLI에서는 위험합니다.
어떤 specifier를 허용하는지 먼저 분리하세요.

```js
import { isBuiltin } from 'node:module';

const allowedBuiltinAdapters = new Set([
  'node:crypto',
  'node:url'
]);

const allowedLocalAdapters = new Map([
  ['json', new URL('./adapters/json.js', import.meta.url).href],
  ['csv', new URL('./adapters/csv.js', import.meta.url).href]
]);

function resolveAdapterSpecifier(name) {
  const builtin = normalizeBuiltinSpecifier(name);

  if (builtin) {
    if (!allowedBuiltinAdapters.has(builtin)) {
      throw new Error(`Builtin adapter is not allowed: ${builtin}`);
    }

    return builtin;
  }

  const local = allowedLocalAdapters.get(name);

  if (!local) {
    throw new Error(`Unknown adapter: ${name}`);
  }

  return local;
}

export async function loadAdapter(name) {
  const specifier = resolveAdapterSpecifier(name);
  return import(specifier);
}
```

이 예제는 사용자가 원하는 모든 경로를 허용하지 않습니다.
입력값은 "이름"으로 받고, 실제 import 대상은 코드 안의 allowlist에서 결정합니다.
내장 모듈도 무조건 허용하지 않고 필요한 것만 선택합니다.

### 내장 모듈 여부는 권한이 아니다

`isBuiltin('node:fs')`가 `true`라는 사실은 `node:fs`를 import해도 된다는 뜻이 아닙니다.
오히려 파일 시스템, 프로세스, 네트워크, 암호화 관련 내장 모듈은 영향 범위가 클 수 있습니다.

플러그인이나 사용자 스크립트 환경에서는 아래처럼 판단을 나누는 편이 안전합니다.

- 내장 모듈인지 판별한다.
- 서비스가 허용한 내장 모듈인지 확인한다.
- 하위 경로까지 허용할지 정한다.
- 실제 import 전에 감사 가능한 로그를 남긴다.
- 민감한 모듈은 wrapper를 통해 제한된 함수만 제공한다.

예를 들어 `node:fs/promises` 전체를 넘기는 대신, 필요한 디렉터리만 읽는 `readProjectFile()` 같은 wrapper를 제공하는 편이 안전할 수 있습니다.
이 기준은 [Node.js Permission Model 가이드](/development/blog/seo/2026/08/21/nodejs-permission-model-ci-build-script-hardening-guide.html)에서 다룬 최소 권한 설계와도 연결됩니다.

## module.builtinModules와의 차이

### 목록이 필요하면 builtinModules를 쓴다

`isBuiltin()`은 단일 specifier 판별에 맞는 함수입니다.
반대로 문서 출력, 진단 로그, 테스트에서 현재 런타임의 내장 모듈 목록을 보고 싶다면 `module.builtinModules`가 더 자연스럽습니다.

```js
import { builtinModules, isBuiltin } from 'node:module';

console.log(builtinModules.includes('fs'));
console.log(isBuiltin('node:fs'));
```

실무에서는 둘을 함께 쓸 수 있습니다.
예를 들어 CLI에서 "허용 가능한 내장 모듈 목록"을 출력할 때는 `builtinModules`를 참고하고, 실제 입력 검증은 `isBuiltin()`으로 처리합니다.

```js
import { builtinModules, isBuiltin } from 'node:module';

export function printBuiltinHelp() {
  const names = builtinModules
    .filter((name) => !name.startsWith('_'))
    .map((name) => name.startsWith('node:') ? name : `node:${name}`)
    .sort();

  console.log(names.join('\n'));
}

export function validateBuiltinInput(specifier) {
  if (!isBuiltin(specifier)) {
    throw new Error(`Unknown builtin module: ${specifier}`);
  }
}
```

목록을 직접 복사해 문서나 코드에 박아 두면 Node.js 버전이 바뀔 때 낡을 수 있습니다.
현재 런타임 기준 목록이 필요하다면 런타임에서 가져오는 편이 낫습니다.

### 가져오기까지 해야 한다면 getBuiltinModule과 역할이 다르다

`process.getBuiltinModule()`은 내장 모듈을 실제로 반환하는 함수입니다.
반면 `module.isBuiltin()`은 판별만 합니다.

```js
import { isBuiltin } from 'node:module';

const specifier = 'node:path';

if (isBuiltin(specifier)) {
  const path = process.getBuiltinModule?.(specifier);
  console.log(path?.basename('/tmp/report.json'));
}
```

두 함수를 섞어 쓸 때는 역할을 명확히 나누세요.

- `isBuiltin()`: 입력값이 내장 모듈인지 판별한다.
- `builtinModules`: 현재 런타임의 내장 모듈 목록을 확인한다.
- `process.getBuiltinModule()`: 내장 모듈을 런타임에서 조건부로 가져온다.
- `import 'node:fs'`: 코드가 항상 그 내장 모듈에 의존할 때 정적으로 가져온다.

대부분의 애플리케이션 코드는 정적 import가 가장 단순합니다.
`isBuiltin()`은 모듈 이름이 데이터로 들어오는 경계에서 빛납니다.

## 테스트와 운영 체크리스트

### 위험한 specifier를 테스트한다

입력 검증 코드는 정상 케이스보다 실패 케이스가 중요합니다.
내장 모듈, 패키지, 상대 경로, URL, 빈 문자열을 나누어 테스트하세요.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

test('normalizes allowed builtin specifiers', () => {
  assert.equal(assertAllowedBuiltin('path'), 'node:path');
  assert.equal(assertAllowedBuiltin('node:url'), 'node:url');
});

test('rejects builtin modules outside allowlist', () => {
  assert.throws(
    () => assertAllowedBuiltin('node:fs'),
    /not allowed/
  );
});

test('rejects non builtin specifiers', () => {
  assert.throws(
    () => assertAllowedBuiltin('./local.js'),
    /Not a Node.js builtin/
  );
});
```

이런 테스트는 작지만 효과가 큽니다.
나중에 allowlist가 넓어질 때 리뷰에서 의도를 확인할 수 있고, 사용자 입력이 import까지 직접 흘러가는 실수를 줄입니다.

### 발행 전 적용 기준

`module.isBuiltin()`을 실무 코드에 넣기 전에 아래 기준을 점검하세요.

- specifier가 외부 입력인지 내부 상수인지 구분했다.
- 내장 모듈 판별과 허용 정책을 분리했다.
- 내부 비교 기준을 `node:` 접두사 형태로 정규화했다.
- `node:fs`, `node:child_process`, `node:process`처럼 영향이 큰 모듈은 별도 검토했다.
- 동적 import 전에 allowlist를 통과하도록 만들었다.
- 실패 메시지에 원본 입력과 정규화된 이름을 과도하게 노출하지 않는다.
- 테스트에서 허용, 거절, 접두사 변형 케이스를 확인했다.

특히 마지막 항목이 중요합니다.
입력값 전체를 로그에 그대로 남기면 경로, 내부 패키지명, 사용자 입력 일부가 노출될 수 있습니다.
운영 로그에는 필요한 범위의 specifier만 남기고, 민감한 값은 마스킹하세요.

## 정리

`module.isBuiltin()`은 작은 API지만 모듈 로딩 경계에서는 꽤 유용합니다.
`fs`와 `node:fs`를 직접 비교하는 코드를 줄이고, 현재 Node.js 런타임이 아는 내장 모듈 판별을 표준 함수에 맡길 수 있습니다.

다만 내장 모듈이라는 사실은 허용 권한이 아닙니다.
동적 import, 플러그인 로더, 설정 기반 확장 기능에서는 `isBuiltin()`으로 1차 분류를 하고, 별도 allowlist와 wrapper로 실제 사용 범위를 좁히는 편이 안전합니다.
내부 표현은 `node:` 접두사로 정규화하고, 테스트에서는 허용되지 않은 내장 모듈이 확실히 거절되는지 확인하세요.

## 관련 글

- [Node.js process.getBuiltinModule 가이드](/development/blog/seo/2026/05/04/nodejs-process-getbuiltinmodule-runtime-optional-dependency-guide.html)
- [Node.js module.registerHooks 가이드](/development/blog/seo/2026/08/16/nodejs-module-registerhooks-loader-customization-guide.html)
- [Node.js syncBuiltinESMExports 가이드](/development/blog/seo/2026/08/10/nodejs-module-syncbuiltinesmexports-builtin-patching-test-guide.html)
- [Node.js Permission Model 가이드](/development/blog/seo/2026/08/21/nodejs-permission-model-ci-build-script-hardening-guide.html)

## 참고 자료

- [Node.js 공식 문서: module.isBuiltin](https://nodejs.org/api/module.html#moduleisbuiltinmodulename)

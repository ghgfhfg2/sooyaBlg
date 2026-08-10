---
layout: post
title: "Node.js syncBuiltinESMExports 가이드: 테스트에서 내장 모듈 패치를 안전하게 동기화하는 법"
date: 2026-08-10 20:00:00 +0900
lang: ko
translation_key: nodejs-module-syncbuiltinesmexports-builtin-patching-test-guide
permalink: /development/blog/seo/2026/08/10/nodejs-module-syncbuiltinesmexports-builtin-patching-test-guide.html
alternates:
  ko: /development/blog/seo/2026/08/10/nodejs-module-syncbuiltinesmexports-builtin-patching-test-guide.html
  x_default: /development/blog/seo/2026/08/10/nodejs-module-syncbuiltinesmexports-builtin-patching-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, module, syncbuiltinesmexports, esm, commonjs, testing, mock, javascript]
description: "Node.js module.syncBuiltinESMExports로 CommonJS에서 패치한 내장 모듈 변경을 ESM live binding에 동기화하는 방법을 정리합니다. 테스트 mock, 복구 순서, 격리 기준, 운영 코드 주의점까지 실무 예제로 설명합니다."
---

Node.js 프로젝트가 CommonJS와 ESM을 함께 쓰는 전환기에 있으면 테스트 코드에서 종종 애매한 상황을 만납니다.
예를 들어 `node:fs`를 임시로 패치했는데 CommonJS로 읽는 코드와 ESM으로 읽는 코드가 서로 다른 함수를 보고 있다면, mock이 적용됐는지 판단하기 어려워집니다.

`node:module`의 `syncBuiltinESMExports()`는 이런 특수한 경계에서 사용할 수 있는 API입니다.
CommonJS 쪽 내장 모듈 export 객체를 수정한 뒤, 그 변경을 내장 ESM 모듈의 기존 live binding에 반영합니다.
공식 문서 기준 이 함수는 기존 export 이름의 값을 동기화하지만, ESM에 새 이름을 추가하거나 기존 이름을 제거하지는 않습니다.

이 글에서는 `syncBuiltinESMExports()`를 테스트 mock과 진단 코드에서 안전하게 쓰는 기준을 정리합니다.
모듈 로딩 자체를 커스터마이징해야 한다면 [Node.js module.registerHooks 가이드](/development/blog/seo/2026/06/20/nodejs-module-registerhooks-loader-customization-guide.html)를 먼저 보고, 테스트 의존성 교체는 [Node.js test runner mock module 가이드](/development/blog/seo/2026/07/14/nodejs-test-runner-mock-module-dependency-guide.html)와 함께 설계하면 좋습니다.

## syncBuiltinESMExports가 필요한 상황

### H3. 내장 모듈을 CommonJS에서 패치한 뒤 ESM import와 맞춘다

Node.js 내장 모듈은 CommonJS와 ESM 양쪽에서 사용할 수 있습니다.
테스트에서 `require('node:fs')`로 가져온 객체를 패치했는데, 실제 코드가 `import { readFile } from 'node:fs'`처럼 named import를 쓰면 기대와 다른 결과가 나올 수 있습니다.

```js
const fs = require('node:fs');

fs.readFile = function fakeReadFile(path, callback) {
  callback(null, Buffer.from('fixture'));
};
```

위 패치는 CommonJS 객체에는 반영됩니다.
하지만 ESM named export가 이미 읽힌 뒤라면 그 binding이 새 값을 보게 만들지 따로 고민해야 합니다.
이때 `syncBuiltinESMExports()`를 호출하면 내장 ESM 모듈의 기존 export binding이 CommonJS export 객체의 현재 값과 동기화됩니다.

```js
const fs = require('node:fs');
const { syncBuiltinESMExports } = require('node:module');

const originalReadFile = fs.readFile;

fs.readFile = function fakeReadFile(path, callback) {
  callback(null, Buffer.from('fixture'));
};

syncBuiltinESMExports();

// 테스트 후에는 반드시 복구한다.
fs.readFile = originalReadFile;
syncBuiltinESMExports();
```

핵심은 패치와 복구를 한 묶음으로 다루는 것입니다.
내장 모듈은 프로세스 전체에서 공유되므로, 한 테스트의 패치가 다음 테스트로 새면 디버깅하기 어려운 실패를 만듭니다.

### H3. 일반적인 mock 도구가 먼저다

`syncBuiltinESMExports()`는 첫 번째 선택지가 아닙니다.
대부분의 애플리케이션 테스트에서는 의존성을 주입하거나, 테스트 대상 함수에 파일 시스템 adapter를 넘기거나, Node.js test runner의 mock API를 쓰는 쪽이 더 읽기 쉽습니다.

내장 모듈 자체를 패치하는 방식은 영향 범위가 큽니다.
다음 조건이 모두 맞을 때만 검토하는 편이 좋습니다.

- 테스트 대상 코드가 이미 내장 모듈을 직접 import하고 있다.
- 코드 구조를 크게 바꾸지 않고 레거시 동작을 검증해야 한다.
- CommonJS와 ESM import 경로가 섞여 있다.
- 패치 범위를 한 테스트 파일 안에서 엄격히 복구할 수 있다.

새 코드라면 내장 모듈을 직접 바꾸기보다 작은 wrapper를 두세요.
예를 들어 파일 읽기 함수 하나만 주입 가능하게 만들면 전역 패치 없이도 테스트할 수 있습니다.

```js
export async function loadConfig(path, fileSystem = fs.promises) {
  const source = await fileSystem.readFile(path, 'utf8');
  return JSON.parse(source);
}
```

이 구조에서는 테스트가 `fileSystem` 객체만 바꾸면 됩니다.
`syncBuiltinESMExports()`는 이런 리팩터링이 어렵거나, 런타임 경계 자체를 검증해야 할 때의 보조 도구로 두는 것이 안전합니다.

## 기본 사용법

### H3. 기존 export 값을 바꾸고 동기화한다

아래 예시는 `node:fs`의 `readFile`을 임시 함수로 바꾼 뒤 ESM import에서도 같은 값을 보게 하는 흐름입니다.

```js
const assert = require('node:assert/strict');
const fs = require('node:fs');
const { syncBuiltinESMExports } = require('node:module');

const originalReadFile = fs.readFile;

async function run() {
  function fakeReadFile(path, callback) {
    callback(null, Buffer.from('mocked'));
  }

  fs.readFile = fakeReadFile;
  syncBuiltinESMExports();

  const esmFs = await import('node:fs');
  assert.equal(esmFs.readFile, fakeReadFile);
}

run().finally(() => {
  fs.readFile = originalReadFile;
  syncBuiltinESMExports();
});
```

여기서 중요한 점은 두 번의 `syncBuiltinESMExports()`입니다.
첫 번째 호출은 패치 적용 후 동기화이고, 두 번째 호출은 원복 후 동기화입니다.
복구 동기화를 빼먹으면 다음 테스트가 원래 함수 대신 fake 함수를 보게 될 수 있습니다.

### H3. 새 export 이름을 추가하는 용도로 쓰지 않는다

`syncBuiltinESMExports()`는 CommonJS 객체에 새 속성을 추가했다고 해서 ESM named export 목록을 늘려 주지 않습니다.
이미 존재하는 export 이름의 값만 맞춥니다.

```js
const assert = require('node:assert/strict');
const fs = require('node:fs');
const { syncBuiltinESMExports } = require('node:module');

fs.readFixture = function readFixture() {
  return 'fixture';
};

syncBuiltinESMExports();

const esmFs = await import('node:fs');

assert.equal(esmFs.readFixture, undefined);

delete fs.readFixture;
```

따라서 이 API를 "내장 모듈에 새 기능을 주입하는 도구"로 쓰면 안 됩니다.
그런 요구가 있다면 별도 adapter 모듈을 만들고, 애플리케이션 코드가 그 adapter를 import하게 하는 편이 훨씬 명확합니다.

### H3. 삭제도 ESM export 목록을 제거하지 않는다

CommonJS 객체에서 기존 속성을 삭제해도 ESM named export 자체가 사라지지는 않습니다.
공식 예시처럼 기존 이름은 남고, 값 동기화 규칙만 적용됩니다.

실무 테스트에서는 삭제보다 명시적인 fake 함수 대체가 낫습니다.
삭제는 런타임마다 오류 모양이 달라질 수 있고, 테스트 실패 메시지도 읽기 어려워집니다.

```js
const fs = require('node:fs');
const { syncBuiltinESMExports } = require('node:module');

const originalReadFileSync = fs.readFileSync;

fs.readFileSync = function unavailableReadFileSync() {
  throw new Error('readFileSync is disabled in this test');
};

syncBuiltinESMExports();

// 테스트 실행

fs.readFileSync = originalReadFileSync;
syncBuiltinESMExports();
```

이렇게 하면 "동기 파일 읽기를 쓰면 실패해야 한다"는 테스트 의도가 코드에 드러납니다.
삭제보다 복구도 쉽습니다.

## 테스트 격리 패턴

### H3. t.after로 복구를 고정한다

Node.js test runner를 쓴다면 `t.after()`에 복구 로직을 넣어 테스트 실패 시에도 원래 상태로 되돌릴 수 있게 만드세요.

```js
import assert from 'node:assert/strict';
import fs from 'node:fs';
import { syncBuiltinESMExports } from 'node:module';
import test from 'node:test';

test('uses patched readFile in ESM code', async (t) => {
  const originalReadFile = fs.readFile;

  t.after(() => {
    fs.readFile = originalReadFile;
    syncBuiltinESMExports();
  });

  function fakeReadFile(path, callback) {
    callback(null, Buffer.from('test-data'));
  }

  fs.readFile = fakeReadFile;
  syncBuiltinESMExports();

  const { readFile } = await import('node:fs');

  assert.equal(readFile, fakeReadFile);
});
```

이 패턴은 최소한의 안전장치입니다.
패치 대상이 프로세스 전역에 가까운 공유 자원이라는 점은 변하지 않습니다.
가능하면 한 테스트 파일 안에서만 사용하고, 여러 테스트가 동시에 같은 내장 모듈을 패치하지 않게 하세요.

### H3. 동시 실행 테스트와 섞지 않는다

내장 모듈 패치는 전역 상태를 바꾸는 일입니다.
테스트 파일이나 테스트 케이스가 병렬로 실행되면 다른 테스트가 패치된 함수를 우연히 볼 수 있습니다.

안전한 기준은 다음과 같습니다.

- 같은 내장 모듈을 패치하는 테스트는 직렬로 실행한다.
- 패치 범위는 한 테스트 안으로 제한한다.
- `t.after()` 또는 `try/finally`로 반드시 복구한다.
- 패치 전후에 `syncBuiltinESMExports()`를 모두 호출한다.
- 병렬 테스트가 필요한 영역에서는 dependency injection이나 mock module을 우선한다.

병렬 실행 전략은 [Node.js test runner concurrency 가이드](/development/blog/seo/2026/07/13/nodejs-test-runner-concurrency-parallel-test-guide.html)를 함께 참고하면 좋습니다.
전역 상태가 섞이는 테스트는 빠른 실행보다 재현 가능성이 더 중요합니다.

### H3. 작은 헬퍼로 패치 수명주기를 묶는다

패치 코드가 여러 곳에 흩어지면 복구 누락이 생깁니다.
테스트 전용 헬퍼로 "바꾸기, 동기화, 복구 등록"을 한 함수에 모으세요.

```js
import { syncBuiltinESMExports } from 'node:module';

export function patchBuiltinExport(t, target, name, replacement) {
  const original = target[name];

  target[name] = replacement;
  syncBuiltinESMExports();

  t.after(() => {
    target[name] = original;
    syncBuiltinESMExports();
  });
}
```

사용하는 쪽은 패치 의도만 남깁니다.

```js
import fs from 'node:fs';
import test from 'node:test';
import { patchBuiltinExport } from './helpers/patch-builtin-export.js';

test('reads fixture through fs.readFile', async (t) => {
  patchBuiltinExport(t, fs, 'readFile', (path, callback) => {
    callback(null, Buffer.from('fixture'));
  });

  // 테스트 대상 실행
});
```

이 헬퍼는 테스트 전용 디렉터리에 두고 운영 코드에서 import하지 않게 관리하세요.
프로덕션에서 내장 모듈을 바꾸는 구조는 장애 원인을 숨기기 쉽습니다.

## 운영 코드에서의 주의점

### H3. 내장 모듈 monkey patch는 공개 계약으로 만들지 않는다

내장 모듈 패치는 강력하지만, 유지보수 비용이 큽니다.
애플리케이션 코드가 `fs.readFile`이 바뀌었다는 사실에 의존하기 시작하면 새 팀원이 동작을 추적하기 어려워집니다.

운영 코드에서는 다음 대안을 먼저 검토하세요.

- 파일 시스템, 네트워크, 시간 API를 adapter로 감싼다.
- 호출부에 의존성을 인자로 주입한다.
- 런타임 정책은 feature flag나 설정으로 제어한다.
- 모듈 해석 자체가 필요할 때만 loader hook을 검토한다.

`syncBuiltinESMExports()`는 내장 모듈 export 동기화 도구이지, 애플리케이션 확장 포인트가 아닙니다.
운영 경로에서 써야 한다면 왜 필요한지, 어느 모듈을 바꾸는지, 복구 조건이 무엇인지 문서화해야 합니다.

### H3. 보안 통제 수단으로 오해하지 않는다

`fs.readFileSync`를 fake 함수로 바꾼다고 해서 파일 접근이 안전하게 차단되는 것은 아닙니다.
다른 API를 통해 파일을 읽을 수도 있고, 이미 참조를 잡은 코드에는 기대한 정책이 적용되지 않을 수 있습니다.

파일 접근 제어가 목적이라면 Node.js Permission Model, 컨테이너 권한, 운영체제 파일 권한, 프로세스 격리를 검토해야 합니다.
경로 검증이 필요하다면 [Node.js path.resolve/normalize 가이드](/development/blog/seo/2026/08/06/nodejs-path-resolve-normalize-traversal-prevention-guide.html)를 함께 보세요.

테스트에서 "이 코드가 특정 API를 호출하지 않는다"를 확인하는 용도와, 운영에서 "이 API를 절대 사용할 수 없다"를 보장하는 용도는 다릅니다.
`syncBuiltinESMExports()`는 전자에 가까운 도구입니다.

## 적용 체크리스트

### H3. 테스트에 넣기 전 확인할 것

- `Node.js syncBuiltinESMExports` 핵심 키워드를 제목과 설명에 포함했다.
- 내장 모듈 패치가 정말 필요한 테스트인지 확인했다.
- 패치 대상 함수의 원본을 저장했다.
- 패치 직후와 복구 직후에 `syncBuiltinESMExports()`를 호출했다.
- `t.after()` 또는 `try/finally`로 복구 경로를 고정했다.
- 같은 내장 모듈을 패치하는 테스트를 병렬 실행하지 않는다.
- 운영 코드의 보안 통제 수단처럼 설명하지 않았다.
- 코드 예제에 실제 토큰, 내부 경로, 개인정보가 없다.

## FAQ

### H3. syncBuiltinESMExports는 일반 npm 패키지에도 적용되나요?

아닙니다.
이 함수는 Node.js 내장 모듈의 CommonJS export와 내장 ESM export binding을 맞추는 용도입니다.
일반 npm 패키지의 export를 동기화하는 도구가 아닙니다.

### H3. ESM import가 이미 끝난 뒤에도 값이 바뀌나요?

기존 ESM named export binding은 동기화 대상이 될 수 있습니다.
다만 새 export 이름을 추가하거나 기존 export 이름을 제거하는 용도는 아닙니다.
패치가 필요한 이름이 실제 내장 ESM export에 이미 있어야 합니다.

### H3. 테스트 mock을 위해 항상 써도 되나요?

권장하지 않습니다.
의존성 주입, wrapper 모듈, test runner mock API가 먼저입니다.
`syncBuiltinESMExports()`는 내장 모듈 패치가 불가피하고 CommonJS/ESM 경계를 맞춰야 할 때만 제한적으로 쓰는 편이 좋습니다.

## 마무리

`module.syncBuiltinESMExports()`는 Node.js의 CommonJS와 ESM 경계에서 내장 모듈 패치 값을 맞춰 주는 좁고 강한 API입니다.
테스트에서 레거시 코드의 내장 모듈 사용을 임시로 제어해야 할 때 유용하지만, 그만큼 전역 상태 변경의 위험도 큽니다.

안전하게 쓰려면 원본 저장, 패치 후 동기화, 테스트 후 복구, 복구 후 동기화를 하나의 수명주기로 묶어야 합니다.
그리고 새 코드에서는 가능한 한 내장 모듈 monkey patch보다 adapter와 의존성 주입을 우선하세요.
테스트는 런타임을 속이는 기술보다 의존성을 명확히 드러내는 구조에서 오래 버팁니다.

## 참고 자료

- [Node.js 공식 문서: module.syncBuiltinESMExports](https://nodejs.org/api/module.html#modulesyncbuiltinesmexports)
- [Node.js module.registerHooks 가이드: 모듈 로딩 커스터마이징을 안전하게 설계하기](/development/blog/seo/2026/06/20/nodejs-module-registerhooks-loader-customization-guide.html)
- [Node.js test runner mock module 가이드](/development/blog/seo/2026/07/14/nodejs-test-runner-mock-module-dependency-guide.html)
- [Node.js test runner concurrency 가이드](/development/blog/seo/2026/07/13/nodejs-test-runner-concurrency-parallel-test-guide.html)

---
layout: post
title: "Node.js module.registerHooks 가이드: 모듈 로딩 규칙을 동기 훅으로 제어하기"
date: 2026-08-16 20:00:00 +0900
lang: ko
translation_key: nodejs-module-registerhooks-loader-customization-guide
permalink: /development/blog/seo/2026/08/16/nodejs-module-registerhooks-loader-customization-guide.html
alternates:
  ko: /development/blog/seo/2026/08/16/nodejs-module-registerhooks-loader-customization-guide.html
  x_default: /development/blog/seo/2026/08/16/nodejs-module-registerhooks-loader-customization-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, module, registerhooks, loader-hooks, esm, commonjs, import, require, backend, javascript]
description: "Node.js module.registerHooks()로 ESM과 CommonJS 모듈 해석·로딩 규칙을 동기 훅으로 제어하는 방법을 정리합니다. resolve/load 훅, --import 등록, 훅 체이닝, shortCircuit, 테스트 격리까지 실무 예제로 설명합니다."
---

Node.js 애플리케이션이 커질수록 모듈을 "어디에서 어떻게 불러올 것인가"가 중요한 설계 문제가 됩니다.
개발 환경에서 별칭 import를 쓰고 싶거나, 테스트에서 특정 모듈을 가벼운 구현으로 바꾸고 싶거나, JSON이 아닌 설정 파일을 읽어 모듈처럼 다루고 싶은 순간이 생깁니다.
이때 빌드 도구만으로 해결할 수도 있지만, 런타임의 모듈 해석 단계에서 직접 규칙을 넣어야 하는 경우도 있습니다.

`module.registerHooks()`는 Node.js의 모듈 해석과 로딩 흐름에 동기 훅을 등록하는 API입니다.
공식 문서 기준으로 `resolve`와 `load` 훅을 직접 넘기며, 반환된 객체의 `deregister()` 또는 `[Symbol.dispose]`로 훅을 해제할 수 있습니다.
기존 `module.register()`가 별도 loader thread에서 비동기 훅을 실행하는 방식이라면, `registerHooks()`는 모듈이 로드되는 같은 thread에서 동기적으로 실행되는 쪽에 가깝습니다.

이 글에서는 `module.registerHooks()`를 언제 쓰면 좋은지, `resolve`와 `load` 훅을 어떻게 작게 유지할지, `--import`로 애플리케이션 코드보다 먼저 등록하는 패턴을 정리합니다.
모듈 시작 성능 최적화는 [Node.js module.enableCompileCache 가이드](/development/blog/seo/2026/08/16/nodejs-module-compile-cache-startup-performance-guide.html), 내장 모듈 export 동기화는 [Node.js module.syncBuiltinESMExports 가이드](/development/blog/seo/2026/08/10/nodejs-module-syncbuiltinesmexports-builtin-patching-test-guide.html)와 함께 보면 좋습니다.
런타임 플래그 관리 기준은 [Node.js process.execArgv 가이드](/development/blog/seo/2026/08/13/nodejs-process-execargv-runtime-flags-guide.html)도 이어서 참고할 수 있습니다.

## module.registerHooks가 필요한 순간

### H3. 런타임 import 별칭을 한곳에서 관리한다

프런트엔드 프로젝트에서는 `@/components/Button` 같은 별칭 import가 익숙합니다.
서버 코드에서도 비슷한 규칙을 쓰고 싶다면 bundler, TypeScript path mapping, 테스트 설정, 런타임 설정이 서로 어긋나지 않게 관리해야 합니다.
`registerHooks()`는 런타임 해석 단계에서 이 규칙을 명시할 수 있게 해 줍니다.

```js
// register-alias-hooks.js
import { registerHooks } from 'node:module';
import { pathToFileURL } from 'node:url';
import { resolve as resolvePath } from 'node:path';

const srcRoot = pathToFileURL(resolvePath(process.cwd(), 'src') + '/').href;

registerHooks({
  resolve(specifier, context, nextResolve) {
    if (specifier.startsWith('#src/')) {
      const url = new URL(specifier.slice('#src/'.length), srcRoot).href;
      return nextResolve(url, context);
    }

    return nextResolve(specifier, context);
  }
});
```

```bash
node --import ./register-alias-hooks.js ./src/server.js
```

이 예제는 `#src/`로 시작하는 specifier를 프로젝트의 `src` 디렉터리 아래 파일 URL로 바꿉니다.
중요한 점은 훅이 모듈 해석 규칙을 숨기는 장소가 되지 않게 하는 것입니다.
별칭은 가능한 한 적고 명확해야 하며, `package.json`의 `imports` 필드로 충분한 경우에는 표준 설정을 우선 검토하는 편이 좋습니다.

### H3. 테스트에서 제한된 모듈만 대체한다

테스트에서는 무거운 외부 SDK나 파일 시스템 접근을 가벼운 구현으로 바꾸고 싶을 때가 있습니다.
이때 모든 import를 가로채기보다, 명시적으로 허용한 specifier만 대체해야 합니다.

```js
// test/register-hooks.js
import { registerHooks } from 'node:module';

const replacements = new Map([
  ['payment-sdk', new URL('./fixtures/payment-sdk.js', import.meta.url).href]
]);

registerHooks({
  resolve(specifier, context, nextResolve) {
    if (replacements.has(specifier)) {
      return nextResolve(replacements.get(specifier), context);
    }

    return nextResolve(specifier, context);
  }
});
```

```bash
node --import ./test/register-hooks.js --test
```

이 방식은 테스트 전용 bootstrap에서만 훅을 등록합니다.
운영 실행 경로와 테스트 실행 경로가 섞이지 않으므로, 실제 서비스에서 의도치 않게 mock 구현이 로드되는 일을 줄일 수 있습니다.
또한 대체 목록이 코드에 드러나기 때문에 리뷰어가 "어떤 모듈이 바뀌는지"를 바로 확인할 수 있습니다.

## resolve 훅 설계 기준

### H3. nextResolve를 기본 경로로 둔다

`resolve` 훅은 import 또는 require specifier를 URL로 해석하는 단계에 끼어듭니다.
대부분의 specifier는 Node.js 기본 resolver가 처리해야 합니다.
따라서 커스텀 규칙에 해당하지 않는 경우에는 즉시 `nextResolve(specifier, context)`를 반환하는 구조가 안전합니다.

```js
import { registerHooks } from 'node:module';

registerHooks({
  resolve(specifier, context, nextResolve) {
    if (specifier === 'virtual:build-info') {
      return {
        url: 'data:text/javascript,export const version = "local";',
        shortCircuit: true
      };
    }

    return nextResolve(specifier, context);
  }
});
```

여기서 `shortCircuit: true`는 훅 체인을 의도적으로 여기서 끝낸다는 표시입니다.
훅이 `nextResolve()`를 호출하지 않으면서 `shortCircuit`도 반환하지 않으면 Node.js는 체인이 실수로 끊겼다고 판단해 예외를 던질 수 있습니다.
이 검사는 잘못된 loader 훅이 전체 모듈 그래프를 조용히 망가뜨리는 일을 줄여 줍니다.

### H3. 조건과 import attributes를 함부로 버리지 않는다

`resolve` 훅의 `context`에는 export conditions, import attributes, parent URL 같은 정보가 들어 있습니다.
커스텀 해석을 하더라도 이 정보를 무시하면 패키지의 조건부 export나 import attributes 기반 캐시 키가 어긋날 수 있습니다.

```js
registerHooks({
  resolve(specifier, context, nextResolve) {
    if (specifier === '#config') {
      return nextResolve(new URL('./config/default.js', context.parentURL).href, {
        ...context
      });
    }

    return nextResolve(specifier, context);
  }
});
```

작은 예제에서는 `context`를 그대로 넘기는 차이가 잘 보이지 않습니다.
하지만 실제 패키지에서는 `"import"`, `"require"`, `"node"` 같은 조건과 import attributes가 모듈 선택에 영향을 줄 수 있습니다.
훅은 필요한 부분만 바꾸고 나머지 context는 보존하는 편이 유지보수에 유리합니다.

## load 훅을 쓸 때의 주의점

### H3. 소스 변환은 좁은 범위에서만 수행한다

`load` 훅은 해석된 URL을 받아 실제 소스와 format을 반환하는 단계에 개입합니다.
예를 들어 작은 텍스트 파일을 JavaScript 모듈처럼 노출할 수 있습니다.

```js
import { readFileSync } from 'node:fs';
import { registerHooks } from 'node:module';

registerHooks({
  load(url, context, nextLoad) {
    if (url.endsWith('.txt')) {
      const text = readFileSync(new URL(url), 'utf8');

      return {
        format: 'module',
        source: `export default ${JSON.stringify(text)};`,
        shortCircuit: true
      };
    }

    return nextLoad(url, context);
  }
});
```

이런 변환은 편리하지만, 모든 파일 확장자에 넓게 적용하면 디버깅이 어려워집니다.
파일 경로, 확장자, 패키지 범위를 제한하고, 변환 결과가 어떤 JavaScript가 되는지 테스트로 고정해야 합니다.
특히 사용자 입력을 그대로 코드 문자열에 섞으면 코드 주입 위험이 생길 수 있으므로 `JSON.stringify()`처럼 안전한 직렬화 경로를 사용해야 합니다.

### H3. 빌드 도구와 책임을 나눈다

`load` 훅으로 TypeScript, YAML, Markdown 같은 파일을 런타임에서 변환할 수는 있습니다.
하지만 운영 서비스의 핵심 변환을 모두 runtime hook에 맡기면 시작 시간, 에러 위치, 캐시 전략이 복잡해질 수 있습니다.

실무에서는 아래처럼 책임을 나누는 편이 좋습니다.

- 개발 편의용 별칭: `resolve` 훅 또는 `package.json` `imports`
- 테스트 전용 대체: 테스트 bootstrap의 제한된 `resolve` 훅
- 운영 빌드 변환: bundler나 사전 빌드 단계
- 실험적 loader 검증: 별도 스크립트와 작은 fixture 테스트

이 기준을 두면 hook이 "모든 것을 해결하는 마법"이 되는 일을 막을 수 있습니다.
런타임 hook은 모듈 시스템의 낮은 단계에 들어가는 기능이므로, 적용 범위를 좁게 잡는 것이 장기적으로 안전합니다.

## 등록 순서와 해제 전략

### H3. 애플리케이션 코드보다 먼저 등록한다

`registerHooks()`는 등록 이후 로드되는 모듈에 영향을 줍니다.
ESM의 static import는 importer 코드가 실행되기 전에 먼저 평가되므로, 같은 파일에서 static import를 적어 놓고 그 아래에서 hook을 등록하면 이미 늦을 수 있습니다.

```js
// entrypoint.mjs
import { registerHooks } from 'node:module';

registerHooks({
  resolve(specifier, context, nextResolve) {
    return nextResolve(specifier, context);
  }
});

await import('./app.mjs');
```

더 명확한 방법은 별도 등록 파일을 만들고 `--import`로 먼저 실행하는 것입니다.
이 방식은 애플리케이션 entry point와 hook 등록 책임을 분리하고, 실행 명령만 봐도 loader 규칙이 적용된다는 사실을 알 수 있게 합니다.

```bash
node --import ./register-hooks.js ./app.mjs
```

테스트 runner, CLI, worker thread까지 같은 규칙을 적용해야 한다면 실행 스크립트에서 preload를 명시하는 편이 추적하기 쉽습니다.

### H3. 테스트에서는 deregister로 격리한다

`registerHooks()`가 반환하는 객체는 훅 해제 함수를 제공합니다.
테스트 파일 하나 안에서 특정 훅만 잠깐 켜야 한다면 `deregister()`를 사용해 다음 테스트로 영향이 새지 않게 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { registerHooks } from 'node:module';

test('virtual module resolves during this test only', async () => {
  const hook = registerHooks({
    resolve(specifier, context, nextResolve) {
      if (specifier === 'virtual:answer') {
        return {
          url: 'data:text/javascript,export default 42;',
          shortCircuit: true
        };
      }

      return nextResolve(specifier, context);
    }
  });

  try {
    const mod = await import('virtual:answer');
    assert.equal(mod.default, 42);
  } finally {
    hook.deregister();
  }
});
```

훅은 프로세스 수준의 모듈 로딩 흐름에 영향을 줍니다.
그래서 테스트 격리를 느슨하게 두면 파일 실행 순서에 따라 성공과 실패가 달라질 수 있습니다.
가능하면 테스트 전용 프로세스를 분리하거나, 훅을 등록한 테스트가 끝날 때 명시적으로 해제하는 습관이 필요합니다.

## 비동기 hook 대신 registerHooks를 우선 검토하는 이유

### H3. 같은 thread에서 동기적으로 판단한다

Node.js 문서는 현재 모듈 커스터마이징 훅을 크게 두 종류로 설명합니다.
`module.registerHooks()`는 동기 훅 함수를 직접 등록하고, `module.register()`는 훅을 내보내는 모듈을 별도로 등록해 비동기 훅을 별도 loader thread에서 실행합니다.
문서에서는 비동기 훅이 thread 간 통신 비용과 CommonJS 커스터마이징 관련 주의점을 갖기 때문에, 대부분의 경우 동기 훅을 더 단순한 선택지로 권장합니다.

동기 훅이 항상 정답이라는 뜻은 아닙니다.
훅 안에서 느린 파일 시스템 작업이나 네트워크 요청을 수행하면 모듈 로딩 자체가 지연됩니다.
따라서 `registerHooks()`를 쓸 때도 규칙은 메모리에 올려 두고, 훅 본문에서는 빠른 문자열 검사와 제한된 파일 읽기만 수행하는 편이 좋습니다.

### H3. CommonJS와 ESM 동작 차이를 테스트한다

모듈 훅은 ESM뿐 아니라 CommonJS의 `require()` 흐름에도 영향을 줄 수 있습니다.
하지만 프로젝트가 ESM과 CommonJS를 함께 쓰는 경우에는 각 경로에서 같은 결과가 나오는지 테스트해야 합니다.

```js
// smoke-test.mjs
import valueFromImport from '#src/example.js';
import { createRequire } from 'node:module';

const require = createRequire(import.meta.url);
const valueFromRequire = require('#src/example.cjs');

console.log({ valueFromImport, valueFromRequire });
```

별칭, format 반환, 조건부 export가 섞이면 한쪽에서는 정상이고 다른 쪽에서는 실패할 수 있습니다.
작은 smoke test를 두면 loader 규칙을 바꿀 때 런타임 전체가 깨지는 일을 빨리 잡을 수 있습니다.

## 운영 체크리스트

### H3. module.registerHooks 적용 전에 확인할 것

- `package.json`의 `imports`나 빌드 도구 설정으로 충분하지 않은가?
- 훅이 처리하는 specifier와 파일 범위가 명확한가?
- 기본 경로에서 `nextResolve()` 또는 `nextLoad()`를 호출하는가?
- 체인을 끝낼 때 `shortCircuit: true`를 명시하는가?
- `context`의 조건과 import attributes를 보존하는가?
- 운영 실행과 테스트 실행에서 훅 등록 파일이 분리되어 있는가?
- 훅 안에서 민감정보, 환경변수 전체, 사용자 입력 원문을 로그로 남기지 않는가?

`module.registerHooks()`는 모듈 시스템의 강력한 확장 지점입니다.
그래서 넓게 쓰기보다 작게 쓰는 편이 좋습니다.
별칭, 테스트 대체, 제한된 파일 변환처럼 목적이 분명한 규칙만 넣고, 기본 Node.js resolver와 빌드 도구가 잘하는 일은 그대로 맡기면 런타임 커스터마이징의 이점만 가져갈 수 있습니다.

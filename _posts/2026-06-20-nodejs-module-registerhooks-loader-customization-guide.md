---
layout: post
title: "Node.js module.registerHooks 가이드: 모듈 로딩 커스터마이징을 안전하게 설계하기"
date: 2026-06-20 20:00:00 +0900
lang: ko
translation_key: nodejs-module-registerhooks-loader-customization-guide
permalink: /development/blog/seo/2026/06/20/nodejs-module-registerhooks-loader-customization-guide.html
alternates:
  ko: /development/blog/seo/2026/06/20/nodejs-module-registerhooks-loader-customization-guide.html
  x_default: /development/blog/seo/2026/06/20/nodejs-module-registerhooks-loader-customization-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, module, registerhooks, loader, esm, commonjs, backend]
description: "Node.js module.registerHooks로 모듈 해석과 로딩을 커스터마이징할 때의 설계 기준을 정리합니다. resolve/load 훅, --import 등록, deregister, 보안 경계, 운영 적용 전 체크리스트까지 실무 예제로 설명합니다."
---

Node.js 애플리케이션이 커지면 모듈 로딩 단계에서 해결하고 싶은 문제가 생깁니다.
예를 들어 내부 패키지 별칭을 통일하거나, 특정 확장자를 실험적으로 처리하거나, 테스트 환경에서만 모듈 해석 규칙을 바꾸고 싶을 수 있습니다.
이때 모든 import 경로를 직접 고치거나 번들러 설정에만 의존하면 런타임과 개발 도구의 동작이 어긋나기 쉽습니다.

`node:module`의 `registerHooks()`는 Node.js 모듈 해석과 로딩 과정을 커스터마이징할 수 있는 API입니다.
[Node.js 공식 문서](https://nodejs.org/api/module.html#moduleregisterhooksoptions)에 따르면 `registerHooks()`는 `resolve`와 `load` 훅을 등록하고, 반환 객체의 `deregister()`로 훅을 제거할 수 있습니다.
이 글에서는 `module.registerHooks()`를 실무 프로젝트에 적용할 때 어디까지 맡기고, 어떤 위험을 피해야 하는지 정리합니다.

ESM 경로와 파일 위치 처리가 헷갈린다면 [Node.js import.meta.dirname·filename 가이드](/development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html)를 먼저 보면 좋습니다.
런타임 최적화와 함께 검토한다면 [Node.js module compile cache 가이드](/development/blog/seo/2026/05/10/nodejs-module-compile-cache-startup-performance-guide.html)를 참고하세요.
로딩 규칙 변경이 관측성 이벤트와 연결된다면 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/06/19/nodejs-diagnostics-channel-instrumentation-guide.html)도 함께 설계하는 편이 안전합니다.

## module.registerHooks가 해결하는 문제

### H3. import 경로 정책을 런타임에서 일관되게 적용한다

모노레포나 내부 플랫폼에서는 `@app/config`, `@app/logger`처럼 조직 안에서 약속한 경로 규칙을 쓰는 경우가 많습니다.
빌드 도구, 테스트 러너, 에디터 설정이 모두 같은 규칙을 이해하면 좋지만 실제로는 한 군데가 빠지기 쉽습니다.

`resolve` 훅은 모듈 specifier를 실제 URL로 바꾸는 단계에 개입합니다.
아래 예시는 `#internal/` 접두어를 프로젝트 내부 디렉터리로 매핑하는 단순한 형태입니다.

```js
import { registerHooks } from 'node:module';
import { pathToFileURL } from 'node:url';
import path from 'node:path';

const projectRoot = pathToFileURL(process.cwd() + path.sep).href;

registerHooks({
  resolve(specifier, context, nextResolve) {
    if (specifier.startsWith('#internal/')) {
      const relativePath = specifier.slice('#internal/'.length);
      return nextResolve(new URL(`./src/internal/${relativePath}.js`, projectRoot).href, context);
    }

    return nextResolve(specifier, context);
  }
});
```

핵심은 훅이 모든 경로를 새로 해석하려 하지 않는다는 점입니다.
내가 책임질 규칙만 처리하고, 나머지는 `nextResolve()`에 넘겨 Node.js 기본 동작과 다른 훅 체인을 유지합니다.

### H3. 로딩 변환은 작고 예측 가능한 범위로 제한한다

`load` 훅은 해석된 URL의 소스나 형식을 다루는 단계에 개입합니다.
이 기능은 강력하지만, 너무 많은 변환을 넣으면 디버깅이 어려워지고 보안 경계도 흐려질 수 있습니다.

```js
import { registerHooks } from 'node:module';

registerHooks({
  load(url, context, nextLoad) {
    if (!url.endsWith('.txt')) {
      return nextLoad(url, context);
    }

    const text = readSafeTextFile(url);

    return {
      format: 'module',
      source: `export default ${JSON.stringify(text)};`
    };
  }
});
```

실무에서는 임의 파일을 실행 가능한 코드로 바꾸는 변환을 특히 조심해야 합니다.
입력 파일의 위치, 확장자, 크기, 인코딩을 제한하고, 사용자 업로드 파일이나 외부에서 내려받은 파일을 바로 모듈로 변환하지 않습니다.

## 등록 시점과 실행 순서

### H3. 애플리케이션 코드보다 먼저 등록한다

모듈 로딩 훅은 등록 이후에 로드되는 모듈에 영향을 줍니다.
이미 정적으로 import된 모듈에는 늦게 등록한 훅이 기대한 방식으로 적용되지 않을 수 있습니다.

운영 진입점에서는 `--import`나 `--require`로 훅 등록 파일을 먼저 실행하는 방식이 관리하기 쉽습니다.

```bash
node --import ./register-module-hooks.js ./server.js
```

`register-module-hooks.js`는 작은 부트스트랩 파일로 유지합니다.
환경 변수 검증, 훅 등록, 필요한 최소 로깅 정도만 담고 실제 애플리케이션 코드는 별도 entrypoint에서 시작하는 편이 좋습니다.

### H3. ESM entrypoint는 동적 import로 시작한다

같은 파일 안에서 훅을 등록하고 곧바로 ESM 애플리케이션을 시작해야 한다면 정적 import보다 동적 import가 안전합니다.
ESM의 정적 import는 모듈 본문보다 먼저 평가되므로 훅 등록보다 애플리케이션 로딩이 앞설 수 있습니다.

```js
import { registerHooks } from 'node:module';

registerHooks({
  resolve(specifier, context, nextResolve) {
    return nextResolve(specifier, context);
  }
});

await import('./server.mjs');
```

이 패턴은 테스트 환경에서도 유용합니다.
테스트 전용 경로 매핑이나 fixture 로더를 등록한 뒤 테스트 대상 모듈을 동적으로 import하면 적용 범위를 더 명확히 볼 수 있습니다.

## 훅 설계 원칙

### H3. resolve 훅은 URL과 조건을 보존한다

`resolve` 훅에서 `context`를 무시하면 조건부 exports, import attributes, parent URL 같은 정보가 사라질 수 있습니다.
특별한 이유가 없다면 받은 `context`를 그대로 `nextResolve()`에 전달합니다.

```js
registerHooks({
  resolve(specifier, context, nextResolve) {
    if (specifier === '#config') {
      const target = process.env.NODE_ENV === 'test'
        ? './config/test.js'
        : './config/runtime.js';

      return nextResolve(new URL(target, context.parentURL).href, context);
    }

    return nextResolve(specifier, context);
  }
});
```

환경에 따라 다른 모듈을 연결할 때도 조건은 적게 유지하는 편이 좋습니다.
로딩 규칙이 너무 많아지면 `package.json`의 `exports`, 테스트 설정, 배포 스크립트가 서로 다른 진실을 갖게 됩니다.

### H3. load 훅은 소스맵과 오류 메시지를 생각하고 만든다

소스를 변환하는 훅은 장애가 났을 때 추적 가능해야 합니다.
변환된 코드에서 오류가 발생했는데 원본 파일과 줄 번호를 알 수 없다면 운영 디버깅 비용이 커집니다.

```js
function asJavaScriptModule(value, sourceUrl) {
  return [
    `export default ${JSON.stringify(value)};`,
    `//# sourceURL=${sourceUrl}`
  ].join('\n');
}
```

간단한 변환이라도 원본 URL을 보존하면 스택 트레이스와 디버깅 경험이 좋아집니다.
복잡한 변환이 필요하다면 이미 검증된 빌드 도구나 테스트 러너 확장 기능으로 옮기는 것이 더 나을 수 있습니다.

## 운영에서 조심해야 할 위험

### H3. 모듈 로딩 훅은 보안 경계가 아니다

`registerHooks()`로 특정 경로를 막거나 다른 모듈로 바꿀 수 있다고 해서 보안 통제 전체를 대체할 수는 없습니다.
공격자가 실행 인자, 환경 변수, 파일 시스템 권한을 바꿀 수 있다면 훅 파일 자체도 우회될 수 있습니다.

보안 목적의 제한은 OS 권한, 컨테이너 권한, 배포 정책, 시크릿 관리, Node.js Permission Model 같은 더 낮은 계층과 함께 설계해야 합니다.
모듈 훅은 개발자 경험과 런타임 정책을 정리하는 도구로 보고, 신뢰할 수 없는 코드를 안전하게 실행하는 샌드박스로 오해하지 않는 것이 중요합니다.

### H3. 로더 안에서 민감정보를 기록하지 않는다

로더 훅은 애플리케이션 초기화 초기에 실행되므로 디버그 로그를 많이 넣고 싶어집니다.
하지만 이 단계의 로그에는 파일 경로, 내부 패키지명, 환경별 설정 이름이 섞이기 쉽습니다.

```js
const DEBUG_MODULE_HOOKS = process.env.DEBUG_MODULE_HOOKS === '1';

function debugHook(event) {
  if (!DEBUG_MODULE_HOOKS) {
    return;
  }

  console.error('[module-hook]', {
    phase: event.phase,
    specifier: event.specifier,
    parentURL: event.parentURL ? new URL(event.parentURL).protocol : null
  });
}
```

디버그 로그를 켜더라도 토큰, 사용자 입력, 전체 절대 경로, 사내 네트워크 주소는 남기지 않습니다.
필요하면 hash나 요약값만 기록하고, 기본값은 로그 비활성화로 둡니다.

## 테스트와 해제 전략

### H3. deregister로 테스트 간 영향을 줄인다

`registerHooks()`가 반환하는 객체에는 `deregister()`가 있습니다.
테스트에서 훅을 등록했다면 테스트가 끝난 뒤 해제해 다음 테스트에 영향을 주지 않게 합니다.

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { registerHooks } from 'node:module';

test('maps #fixture to a local module', async () => {
  const hooks = registerHooks({
    resolve(specifier, context, nextResolve) {
      if (specifier === '#fixture') {
        return nextResolve(new URL('./fixtures/example.js', import.meta.url).href, context);
      }

      return nextResolve(specifier, context);
    }
  });

  try {
    const mod = await import('#fixture');
    assert.equal(mod.name, 'example');
  } finally {
    hooks.deregister();
  }
});
```

훅은 프로세스 전체의 모듈 로딩 흐름에 영향을 줄 수 있습니다.
따라서 테스트에서는 등록과 해제를 같은 범위에 두고, 병렬 테스트에서 같은 specifier를 서로 다르게 매핑하지 않도록 주의합니다.

### H3. 적용 전 체크리스트를 둔다

모듈 로딩 규칙은 한 번 들어가면 여러 팀과 도구가 의존하게 됩니다.
배포 전에 아래 항목을 확인하면 예상 밖의 장애를 줄일 수 있습니다.

- 훅 등록 파일이 애플리케이션 코드보다 먼저 실행되는가?
- 처리하지 않는 specifier는 항상 `nextResolve()` 또는 `nextLoad()`로 넘기는가?
- 변환 대상 파일의 위치, 확장자, 크기를 제한했는가?
- 민감정보가 로더 로그나 오류 메시지에 포함되지 않는가?
- 테스트에서 `deregister()`로 훅 영향을 정리하는가?
- 빌드 도구, 테스트 러너, 운영 실행 명령이 같은 경로 정책을 따르는가?

## 언제 registerHooks를 쓰지 않을까

### H3. 정적 빌드 설정으로 충분하면 그쪽이 단순하다

단순한 TypeScript path alias, 번들 타깃 변경, 테스트 mock 설정은 기존 도구 설정만으로 충분할 때가 많습니다.
이런 경우 런타임 훅을 추가하면 문제 해결보다 운영 복잡도가 더 커질 수 있습니다.

`registerHooks()`는 Node.js 런타임 자체의 모듈 해석과 로딩을 바꿔야 할 때 선택합니다.
개발 편의를 위한 경로 축약만 필요하다면 `package.json`의 `imports`, TypeScript 설정, 테스트 러너의 alias 기능을 먼저 검토하는 편이 좋습니다.

### H3. 고빈도 변환과 대형 컴파일 파이프라인에는 맞지 않을 수 있다

로더 훅에서 큰 파일을 매번 변환하거나 외부 프로세스를 호출하면 시작 시간이 불안정해집니다.
운영 서버에서는 시작 시간, 메모리 사용량, 오류 메시지의 예측 가능성이 중요합니다.

복잡한 변환은 빌드 단계에서 끝내고, 런타임 훅은 꼭 필요한 작은 규칙만 맡기는 쪽이 안전합니다.
모듈 로딩을 커스터마이징하더라도 목표는 마법을 늘리는 것이 아니라 팀이 합의한 런타임 경계를 명확히 만드는 것입니다.

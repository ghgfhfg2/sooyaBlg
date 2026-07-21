---
layout: post
title: "Node.js stripTypeScriptTypes 가이드: 런타임에서 TypeScript 코드 안전하게 변환하기"
date: 2026-07-22 08:00:00 +0900
lang: ko
translation_key: nodejs-module-striptypescripttypes-runtime-transform-guide
permalink: /development/blog/seo/2026/07/22/nodejs-module-striptypescripttypes-runtime-transform-guide.html
alternates:
  ko: /development/blog/seo/2026/07/22/nodejs-module-striptypescripttypes-runtime-transform-guide.html
  x_default: /development/blog/seo/2026/07/22/nodejs-module-striptypescripttypes-runtime-transform-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, typescript, striptypescripttypes, module, runtime, build, javascript, dx]
description: "Node.js node:module의 stripTypeScriptTypes()로 TypeScript 코드에서 타입을 제거하거나 변환하는 방법을 정리합니다. strip 모드와 transform 모드 차이, source map, 캐시, CI 점검, 보안 기준까지 실무 예제로 설명합니다."
---

작은 개발 도구나 내부 스크립트를 만들다 보면 TypeScript 파일을 꼭 별도 빌드 단계로 넘기지 않고 바로 다루고 싶을 때가 있습니다.
설정 파일을 읽어 검사하거나, 문서 예제의 TypeScript 코드를 JavaScript로 바꾸거나, 플러그인 입력을 런타임에서 검증해야 하는 경우입니다.
이때 무거운 번들러를 붙이면 작업은 가능하지만, 도구의 책임보다 빌드 체인이 더 커질 수 있습니다.

`node:module`의 `stripTypeScriptTypes()`는 문자열로 받은 TypeScript 코드에서 타입 구문을 제거하거나 일부 TypeScript 문법을 JavaScript로 변환하는 API입니다.
Node.js v24.14.0 기준으로 이 API는 experimental warning을 출력하므로, 운영 핵심 경로보다는 내부 도구, 테스트 보조 도구, 사전 점검 스크립트에서 조심스럽게 쓰는 편이 좋습니다.
이 글에서는 `strip` 모드와 `transform` 모드를 나누는 기준, source map을 붙이는 방법, 런타임 변환을 안전하게 제한하는 패턴을 정리합니다.
[Node.js TypeScript type stripping 가이드](/development/blog/seo/2026/05/13/nodejs-typescript-type-stripping-runtime-guide.html), [Node.js module.registerHooks 가이드](/development/blog/seo/2026/06/20/nodejs-module-registerhooks-loader-customization-guide.html), [Node.js source maps 가이드](/development/blog/seo/2026/05/22/nodejs-enable-source-maps-production-debugging-guide.html)와 함께 보면 Node.js에서 TypeScript 실행 경로를 더 단정하게 설계할 수 있습니다.

## stripTypeScriptTypes가 필요한 상황

### H3. 빌드가 아니라 코드 문자열 변환이 필요할 때

일반 애플리케이션이라면 `tsc`, `tsx`, `esbuild`, `swc` 같은 도구를 명확히 선택하는 편이 자연스럽습니다.
하지만 모든 상황이 애플리케이션 빌드는 아닙니다.
예를 들어 문서 사이트가 예제 코드를 미리 검사하거나, CLI가 사용자의 작은 TypeScript 설정 조각을 읽거나, 테스트가 fixture 안의 TypeScript 스니펫을 JavaScript로 바꿔 비교할 수 있습니다.

이런 경우에는 "프로젝트 전체를 빌드한다"보다 "이 문자열을 실행 가능한 JavaScript 형태로 바꾼다"가 목표입니다.
`stripTypeScriptTypes()`는 바로 이 좁은 요구에 맞습니다.

```js
import { stripTypeScriptTypes } from 'node:module';

const source = `
  type User = { id: string };

  export function label(user: User): string {
    return user.id;
  }
`;

const output = stripTypeScriptTypes(source);

console.log(output);
```

기본 모드는 `strip`입니다.
타입 annotation, type alias처럼 런타임에 남지 않는 타입 구문을 제거하는 데 초점이 있습니다.
코드 구조를 크게 바꾸지 않기 때문에 결과를 사람이 읽기도 비교적 쉽습니다.

### H3. 모든 TypeScript 문법을 처리하는 만능 컴파일러는 아니다

중요한 점은 `stripTypeScriptTypes()`를 TypeScript 컴파일러 전체로 생각하면 안 된다는 것입니다.
기본 `strip` 모드는 타입 제거만으로 JavaScript가 될 수 있는 코드를 대상으로 삼습니다.
`enum`처럼 런타임 코드 생성이 필요한 TypeScript 문법은 strip-only 모드에서 에러가 납니다.

```js
import { stripTypeScriptTypes } from 'node:module';

stripTypeScriptTypes(`
  enum Status {
    Ready
  }
`);
```

이 코드는 `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` 계열의 오류를 만날 수 있습니다.
이 동작은 오히려 장점이 될 수 있습니다.
내부 도구에서 "타입 제거만 허용한다"는 정책을 세우면, 런타임 변환이 예상보다 많은 의미를 바꾸는 일을 줄일 수 있습니다.

## strip 모드와 transform 모드 선택하기

### H3. strip 모드는 가장 보수적인 기본값이다

`strip` 모드는 타입 정보만 지우는 흐름에 가깝습니다.
도구가 TypeScript 설정 파일을 읽되, 실제 런타임 기능은 표준 JavaScript에 가깝게 제한하고 싶다면 좋은 기본값입니다.

```js
import { stripTypeScriptTypes } from 'node:module';

export function stripConfigSource(source) {
  return stripTypeScriptTypes(source, {
    mode: 'strip'
  });
}
```

이 함수는 변환 책임이 좁습니다.
팀원이 코드를 읽을 때도 "여기서는 타입만 없앤다"는 의도가 드러납니다.
`enum`, namespace, parameter property처럼 JavaScript 출력이 필요한 문법이 들어오면 실패하게 두고, 작성자에게 더 단순한 문법으로 바꾸라고 안내하는 편이 안전합니다.

### H3. transform 모드는 런타임 코드 생성이 필요할 때만 쓴다

`transform` 모드는 TypeScript 문법 중 일부를 JavaScript 코드로 바꿔야 할 때 사용합니다.
예를 들어 `enum`처럼 타입 제거만으로는 실행 코드가 되지 않는 문법이 필요하다면 transform 모드를 검토할 수 있습니다.

```js
import { stripTypeScriptTypes } from 'node:module';

const output = stripTypeScriptTypes(`
  enum Status {
    Ready,
    Failed
  }

  export const current = Status.Ready;
`, {
  mode: 'transform'
});
```

다만 transform 모드는 입력 코드와 출력 코드의 차이가 커질 수 있습니다.
디버깅할 때 원본 라인과 변환 라인이 어긋날 수 있고, 결과 코드의 크기도 늘어날 수 있습니다.
그래서 사용자 요청마다 즉석에서 무제한 변환하기보다, 사전 점검이 가능한 내부 코드나 신뢰된 저장소 안의 파일에 제한하는 편이 좋습니다.

## 안전한 변환 유틸 만들기

### H3. 입력 크기와 모드를 먼저 제한한다

런타임 변환 유틸은 작은 함수처럼 보여도 보안과 안정성 기준이 필요합니다.
외부 입력을 그대로 받아 큰 문자열을 변환하면 CPU와 메모리를 예측하기 어려워질 수 있습니다.
먼저 입력 크기, 허용 모드, source map 사용 여부를 명확히 제한하세요.

```js
import { stripTypeScriptTypes } from 'node:module';

const MAX_SOURCE_BYTES = 200_000;
const ALLOWED_MODES = new Set(['strip', 'transform']);

export function compileTypeScriptSnippet(source, options = {}) {
  if (Buffer.byteLength(source, 'utf8') > MAX_SOURCE_BYTES) {
    throw new Error('TypeScript snippet is too large');
  }

  const mode = options.mode ?? 'strip';

  if (!ALLOWED_MODES.has(mode)) {
    throw new Error(`Unsupported TypeScript transform mode: ${mode}`);
  }

  return stripTypeScriptTypes(source, {
    mode,
    sourceMap: options.sourceMap === true,
    sourceUrl: options.sourceUrl ?? ''
  });
}
```

이 예제는 입력 크기를 제한하고, 모드를 명시적인 허용 목록으로 좁힙니다.
실제 서비스에서는 여기에 파일 경로 허용 목록, 캐시, 실행 권한 제한을 더하는 것이 좋습니다.
특히 변환 결과를 곧바로 실행한다면 코드 실행 경계가 별도로 필요합니다.

### H3. 변환과 실행 책임을 분리한다

`stripTypeScriptTypes()`는 코드를 변환할 뿐입니다.
결과 JavaScript를 실행할지, 파일로 저장할지, 검사만 할지는 호출자가 결정합니다.
이 책임을 한 함수에 섞으면 위험한 기능이 되기 쉽습니다.

```js
export function prepareSnippetForReview(source) {
  const js = compileTypeScriptSnippet(source, {
    mode: 'strip'
  });

  return {
    language: 'javascript',
    code: js
  };
}
```

문서 미리보기나 코드 리뷰 도구라면 변환 결과를 보여 주기만 해도 충분할 수 있습니다.
반대로 실행이 필요하다면 별도 프로세스, 제한된 권한, 시간 제한, 임시 디렉터리 정리 기준을 함께 둬야 합니다.
변환 API를 샌드박스로 오해하면 안 됩니다.

## source map과 디버깅

### H3. transform 모드에서는 source map을 함께 검토한다

변환 결과를 테스트하거나 디버깅해야 한다면 `sourceMap` 옵션을 검토할 수 있습니다.
특히 transform 모드에서는 출력 코드가 원본과 달라질 수 있으므로, 오류 위치를 원본 TypeScript와 연결하는 정보가 도움이 됩니다.

```js
import { stripTypeScriptTypes } from 'node:module';

const output = stripTypeScriptTypes(source, {
  mode: 'transform',
  sourceMap: true,
  sourceUrl: 'file:///workspace/snippets/example.ts'
});
```

`sourceUrl`은 디버깅 출력에서 원본 위치를 식별하는 데 쓰기 좋습니다.
다만 실제 내부 경로나 사용자 이름이 그대로 드러나지 않도록 주의해야 합니다.
공개 로그, CI artifact, 블로그 예제에는 개인 경로나 내부 저장소 URL을 그대로 남기지 않는 편이 안전합니다.

### H3. 에러 메시지는 사용자 친화적으로 감싼다

입력 코드가 지원하지 않는 TypeScript 문법을 포함하면 변환 단계에서 예외가 발생할 수 있습니다.
이 예외를 그대로 CLI 사용자에게 보여 주기보다, 어떤 모드에서 실패했고 어떤 선택지가 있는지 설명하는 래퍼를 두면 좋습니다.

```js
export function safeStrip(source) {
  try {
    return compileTypeScriptSnippet(source, {
      mode: 'strip'
    });
  } catch (error) {
    if (error?.code === 'ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX') {
      throw new Error(
        'This snippet uses TypeScript syntax that requires transform mode. ' +
        'Prefer plain type annotations, or run the tool with --mode=transform.'
      );
    }

    throw error;
  }
}
```

이런 메시지는 사용자가 바로 다음 행동을 고를 수 있게 합니다.
반대로 내부 stack trace, 로컬 파일 경로, 임시 디렉터리 전체를 출력하면 진단보다 노출 위험이 커질 수 있습니다.

## 캐시와 CI 적용 기준

### H3. 반복 변환은 내용 해시로 캐시한다

문서 사이트나 코드 검사 도구에서 같은 스니펫을 여러 번 변환한다면 간단한 캐시를 둘 수 있습니다.
캐시 키는 파일 이름보다 코드 내용과 변환 옵션을 기준으로 만드는 편이 안전합니다.

```js
import { createHash } from 'node:crypto';

const cache = new Map();

export function compileWithCache(source, options = {}) {
  const key = createHash('sha256')
    .update(source)
    .update('\0')
    .update(JSON.stringify({
      mode: options.mode ?? 'strip',
      sourceMap: options.sourceMap === true
    }))
    .digest('hex');

  if (cache.has(key)) {
    return cache.get(key);
  }

  const output = compileTypeScriptSnippet(source, options);
  cache.set(key, output);

  return output;
}
```

캐시는 성능 최적화일 뿐, 권한 검사를 대체하지 않습니다.
허용되지 않은 파일이나 너무 큰 입력은 캐시 조회 전에 거부해야 합니다.

### H3. CI에서는 지원 문법을 테스트로 고정한다

API가 experimental인 동안에는 팀이 기대하는 동작을 작은 테스트로 고정해 두는 편이 좋습니다.
특히 `strip` 모드에서 허용할 문법과 거부할 문법을 명확히 해 두면 Node.js 업그레이드 때 변화를 빨리 발견할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { compileTypeScriptSnippet } from '../src/compile-ts-snippet.js';

test('strips type annotations in strip mode', () => {
  const output = compileTypeScriptSnippet('const count: number = 1;');

  assert.match(output, /const count\s*=\s*1/);
  assert.doesNotMatch(output, /number/);
});

test('rejects enum in strip mode', () => {
  assert.throws(() => {
    compileTypeScriptSnippet('enum Status { Ready }');
  }, /unsupported|TypeScript/i);
});
```

테스트 이름은 정책을 설명해야 합니다.
단순히 "works"라고 두기보다 "strip 모드는 타입 annotation만 허용한다"는 식으로 작성하면 CI 로그만 봐도 의도를 알 수 있습니다.

## 운영에서 피해야 할 패턴

### H3. 사용자 코드를 곧바로 실행하지 않는다

가장 위험한 패턴은 외부 사용자가 보낸 TypeScript 문자열을 변환한 뒤 같은 프로세스에서 바로 실행하는 것입니다.
`stripTypeScriptTypes()`는 타입을 제거하거나 변환할 뿐, 위험한 JavaScript 실행을 막아 주지 않습니다.

사용자 입력을 실행해야 하는 제품이라면 별도 격리 환경, 시간 제한, 파일 시스템 권한 제한, 네트워크 제한, 감사 로그가 필요합니다.
대부분의 블로그 예제나 내부 도구에서는 실행하지 않고 정적 검사, 미리보기, 변환 결과 저장까지만 허용하는 편이 훨씬 안전합니다.

### H3. 프로덕션 요청 경로에서 매번 변환하지 않는다

요청마다 TypeScript 코드를 변환하면 latency와 CPU 사용량이 예측하기 어려워집니다.
프로덕션에서 꼭 필요하다면 사전에 변환해 저장하거나, 시작 시점에 한 번만 변환하고 결과를 캐시하세요.
요청 경로에서는 이미 검증된 JavaScript 산출물을 읽는 구조가 더 안정적입니다.

```js
// 시작 시점에 한 번 준비한다.
const compiledTemplate = compileWithCache(templateSource, {
  mode: 'strip'
});

export function renderTemplateContext(context) {
  return {
    code: compiledTemplate,
    context
  };
}
```

이 구조는 변환 실패를 배포나 부팅 단계에서 드러나게 만듭니다.
사용자 요청 중간에 실험적 API의 경고나 문법 오류가 튀어나오는 상황을 줄일 수 있습니다.

## FAQ

### H3. stripTypeScriptTypes가 tsc를 대체할 수 있나요?

일반적으로는 대체하지 않는 편이 좋습니다.
`tsc`는 타입 검사, 프로젝트 설정, 선언 파일, 모듈 해석까지 넓게 다룹니다.
`stripTypeScriptTypes()`는 코드 문자열에서 TypeScript 구문을 제거하거나 변환하는 좁은 도구로 보는 것이 안전합니다.

### H3. strip 모드와 transform 모드 중 무엇을 기본값으로 둘까요?

내부 도구의 기본값은 `strip`이 좋습니다.
타입 제거만 허용하면 입력 문법이 단순해지고, 변환 결과가 예측하기 쉽습니다.
`enum` 같은 런타임 변환 문법이 실제로 필요하고 테스트로 고정할 수 있을 때만 `transform`을 명시적으로 켜세요.

### H3. API가 experimental이면 블로그 예제에 써도 되나요?

쓸 수는 있지만 조건을 밝혀야 합니다.
Node.js 버전, experimental warning, 운영 핵심 경로에 바로 넣지 말아야 한다는 점을 함께 적어야 독자가 위험을 오해하지 않습니다.
예제 코드는 작은 내부 도구나 CI 사전 점검처럼 실패 범위가 제한된 곳을 기준으로 설명하는 편이 좋습니다.

## 발행 전 점검

- 제목과 설명에 `Node.js stripTypeScriptTypes` 핵심 키워드를 포함했다.
- 본문은 H2와 H3 구조로 작성했고, `strip` 모드와 `transform` 모드 차이를 분리했다.
- 예제에는 실제 토큰, 내부 도메인, 개인정보, API 키를 넣지 않았다.
- 외부 사용자 코드 실행을 권장하지 않고, 변환과 실행 책임을 분리했다.
- 내부링크를 3개 포함해 TypeScript 실행, 모듈 훅, source map 글과 연결했다.

## 함께 보면 좋은 글

- [Node.js TypeScript type stripping 가이드: 빌드 없이 .ts 파일을 실행하는 법](/development/blog/seo/2026/05/13/nodejs-typescript-type-stripping-runtime-guide.html)
- [Node.js module.registerHooks 가이드: 모듈 로딩 커스터마이징을 안전하게 설계하기](/development/blog/seo/2026/06/20/nodejs-module-registerhooks-loader-customization-guide.html)
- [Node.js source maps 가이드: 운영 디버깅에서 스택 트레이스 읽기](/development/blog/seo/2026/05/22/nodejs-enable-source-maps-production-debugging-guide.html)

`stripTypeScriptTypes()`는 TypeScript 프로젝트 전체를 대신 빌드하는 도구가 아니라, 제한된 TypeScript 코드 조각을 JavaScript 형태로 다루기 위한 낮은 수준의 API입니다.
기본값은 보수적인 `strip` 모드로 두고, transform이 필요한 문법은 테스트와 캐시, source map, 실행 경계를 함께 설계하세요.
그렇게 하면 작은 내부 도구에서는 빌드 단계를 줄이면서도, 운영 코드의 예측 가능성과 보안 기준을 지킬 수 있습니다.

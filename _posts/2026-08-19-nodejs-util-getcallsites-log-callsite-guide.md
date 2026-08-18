---
layout: post
title: "Node.js util.getCallSites 가이드: 로그에 호출 위치를 안전하게 붙이는 방법"
date: 2026-08-19 08:00:00 +0900
lang: ko
translation_key: nodejs-util-getcallsites-log-callsite-guide
permalink: /development/blog/seo/2026/08/19/nodejs-util-getcallsites-log-callsite-guide.html
alternates:
  ko: /development/blog/seo/2026/08/19/nodejs-util-getcallsites-log-callsite-guide.html
  x_default: /development/blog/seo/2026/08/19/nodejs-util-getcallsites-log-callsite-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, util, getcallsites, logging, stack-trace, debugging, observability, javascript]
description: "Node.js util.getCallSites()로 Error.stack 문자열 파싱 없이 호출 위치를 구조화해 로그와 디버깅 정보에 붙이는 방법을 정리합니다. frameCount, sourceMap 옵션, 성능 주의점, 민감정보 점검까지 실무 예제로 설명합니다."
---

운영 로그에서 "어디서 이 로그가 찍혔는지"를 알고 싶을 때가 있습니다.
메시지는 같은데 호출 지점이 여러 곳이면, 로그만 보고 원인을 따라가기가 어렵습니다.
그래서 어떤 팀은 `new Error().stack`을 만들고 문자열을 파싱해 파일명과 라인 번호를 꺼냅니다.
문제는 stack 문자열 형식이 사람이 읽기 위한 출력에 가깝고, `Error.prepareStackTrace` 같은 전역 설정의 영향을 받을 수 있다는 점입니다.

Node.js의 `util.getCallSites()`는 호출 스택을 문자열이 아니라 구조화된 call site 객체 배열로 가져오는 API입니다.
공식 문서 기준으로 `frameCount`는 기본 10개 프레임을 캡처하며, 1개부터 200개까지 요청할 수 있습니다.
각 항목에는 함수 이름, 스크립트 이름, 스크립트 ID, 1부터 시작하는 라인 번호와 컬럼 번호가 포함됩니다.
또 `sourceMap` 옵션을 통해 source map 기준의 원본 위치를 재구성할 수 있습니다.

이 글에서는 `util.getCallSites()`를 로그 호출 위치 진단에 어떻게 적용할지, 어느 경로에서는 쓰지 않는 편이 좋은지, 민감정보와 성능 리스크를 어떻게 줄일지 정리합니다.
운영 에러 스택 매핑은 [Node.js source maps 가이드](/development/blog/seo/2026/05/22/nodejs-enable-source-maps-production-debugging-guide.html), 로그 컨텍스트 연결은 [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html), 이벤트 기반 관측성 분리는 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/08/07/nodejs-diagnostics-channel-observability-guide.html)도 함께 보면 좋습니다.

## util.getCallSites가 해결하는 문제

### Error.stack 문자열 파싱을 줄인다

호출 위치를 얻는 오래된 방식은 임시 `Error`를 만들고 `stack` 문자열을 정규식으로 파싱하는 것입니다.

```js
function getCallerFromStack() {
  const stack = new Error().stack ?? '';
  const line = stack.split('\n')[2] ?? '';

  return line.trim();
}
```

이 코드는 빨리 만들 수 있지만 오래 유지하기는 애매합니다.
런타임 버전, 번들러, source map, 테스트 환경에 따라 문자열 모양이 달라질 수 있습니다.
또 파싱 결과가 실패했을 때 빈 문자열을 남기면, 로그 품질이 조용히 나빠집니다.

`util.getCallSites()`는 같은 목적을 더 명시적으로 표현합니다.

```js
import { getCallSites } from 'node:util';

function getCallerLocation() {
  const callSites = getCallSites(3);
  const caller = callSites[2];

  if (!caller) {
    return null;
  }

  return {
    functionName: caller.functionName,
    file: caller.scriptName,
    line: caller.lineNumber,
    column: caller.columnNumber
  };
}
```

여기서 `getCallSites(3)`은 현재 함수와 그 위쪽 호출자를 포함해 필요한 만큼만 스택 프레임을 가져오려는 의도입니다.
로그 호출 위치만 필요하다면 10개 기본값을 그대로 쓰기보다 작은 `frameCount`를 명시하는 편이 읽기 쉽습니다.

### Error.prepareStackTrace 영향을 피한다

일부 테스트 도구, APM, 디버깅 도구는 `Error.prepareStackTrace`를 건드릴 수 있습니다.
이 설정은 `error.stack` 출력 형식에 영향을 주기 때문에 문자열 파싱 코드와 충돌할 수 있습니다.
반면 `util.getCallSites()`는 `error.stack` 문자열에 접근하는 방식이 아니므로, 호출 위치를 구조화해서 얻는 목적에 더 잘 맞습니다.

```js
import { getCallSites } from 'node:util';

export function captureLogSite() {
  const [, caller] = getCallSites(2);

  return caller
    ? `${caller.scriptName}:${caller.lineNumber}:${caller.columnNumber}`
    : 'unknown';
}
```

이 함수는 로그 포맷터 내부에서 "호출 위치" 필드를 만들 때 사용할 수 있습니다.
다만 모든 로그에 자동으로 붙이기 전에 성능과 노출 범위를 먼저 점검해야 합니다.

## 로그 호출 위치에 적용하기

### 래퍼 함수 안에서는 프레임을 한 단계 건너뛴다

대부분의 애플리케이션은 `logger.info()`를 직접 부르기보다 `logInfo()` 같은 래퍼를 둡니다.
이때 첫 번째 프레임은 위치 캡처 함수이고, 두 번째 프레임은 로그 래퍼일 수 있습니다.
실제로 알고 싶은 것은 그 래퍼를 호출한 업무 코드입니다.

```js
import { getCallSites } from 'node:util';

function getApplicationCaller() {
  const callSites = getCallSites(4);
  const caller = callSites[3];

  if (!caller) {
    return undefined;
  }

  return {
    file: shortenPath(caller.scriptName),
    line: caller.lineNumber,
    column: caller.columnNumber,
    functionName: caller.functionName || undefined
  };
}

export function logInfo(logger, message, fields = {}) {
  logger.info({
    ...fields,
    callSite: getApplicationCaller()
  }, message);
}

function shortenPath(scriptName) {
  return scriptName
    .replace(process.cwd(), '.')
    .replaceAll('\\', '/');
}
```

중요한 점은 인덱스를 코드베이스의 로깅 구조에 맞춰 테스트로 고정하는 것입니다.
래퍼가 한 겹 늘어나면 `callSites[3]`이 아니라 다른 프레임이 필요할 수 있습니다.
따라서 호출 위치 캡처는 작은 유틸 함수로 숨기되, 테스트에서는 실제 호출자 라인이 나오는지 확인해야 합니다.

### 개발과 진단 모드에서 먼저 켠다

호출 위치는 디버깅에 유용하지만 모든 운영 로그에 항상 필요한 필드는 아닐 수 있습니다.
스택 프레임을 캡처하는 작업은 일반 필드 조합보다 비용이 큽니다.
초당 수천 번 찍히는 access log, metric성 debug log, 반복 루프 내부 로그에 무조건 붙이면 지연과 로그 용량이 늘어납니다.

처음에는 환경 변수나 로그 레벨로 제한하는 편이 현실적입니다.

```js
const includeCallSite = process.env.LOG_CALL_SITE === '1';

export function logDebug(logger, message, fields = {}) {
  logger.debug({
    ...fields,
    ...(includeCallSite ? { callSite: getApplicationCaller() } : {})
  }, message);
}
```

이렇게 두면 장애 조사 중에만 호출 위치를 켜고, 평상시에는 로그 필드를 작게 유지할 수 있습니다.
운영에서 계속 켜야 한다면 샘플링, 특정 logger namespace, 특정 에러 등으로 범위를 줄이는 것이 좋습니다.

### 외부에 노출되는 응답에는 넣지 않는다

`scriptName`에는 서버의 파일 경로나 빌드 산출물 경로가 들어갈 수 있습니다.
이 값은 내부 로그에는 유용하지만 API 응답, 클라이언트 오류 메시지, 공개 페이지에 그대로 노출하면 불필요한 경로 정보가 새어 나갑니다.

```js
function toInternalLogField(callSite) {
  if (!callSite) {
    return undefined;
  }

  return {
    file: shortenPath(callSite.scriptName),
    line: callSite.lineNumber,
    column: callSite.columnNumber
  };
}

function toPublicErrorBody(error) {
  return {
    error: 'internal_error',
    message: '요청을 처리하지 못했습니다.',
    requestId: error.requestId
  };
}
```

내부 로그와 외부 응답의 필드 모델을 분리하면 실수 가능성이 줄어듭니다.
로그에 들어가는 경로도 절대 경로 전체보다 프로젝트 루트 기준 상대 경로로 줄이는 편이 안전합니다.

## sourceMap 옵션과 TypeScript 프로젝트

### 원본 위치가 필요하면 sourceMap을 명시한다

TypeScript나 번들러를 사용하는 프로젝트에서는 실제 실행 파일이 `dist` 아래 JavaScript일 수 있습니다.
이때 호출 위치가 원본 `.ts` 파일 기준이어야 한다면 source map 설정을 함께 봐야 합니다.

```js
import { getCallSites } from 'node:util';

export function captureSourceMappedCaller() {
  const callSites = getCallSites(4, {
    sourceMap: true
  });

  const caller = callSites[3];

  return caller
    ? {
        file: caller.scriptName,
        line: caller.lineNumber,
        column: caller.columnNumber
      }
    : undefined;
}
```

`sourceMap: true`를 설정해도 source map 파일이나 inline map 정보가 없으면 기대한 원본 위치가 나오지 않을 수 있습니다.
따라서 `tsconfig.json`, 번들러 설정, 배포 산출물 보존 정책까지 함께 점검해야 합니다.
Node.js를 `--enable-source-maps`로 실행하는 환경에서는 source map 사용이 기본 동작과 연결될 수 있으므로, 로컬과 운영 설정 차이도 확인합니다.

### 라인 번호는 테스트로 검증한다

호출 위치 기능은 눈으로 보기에는 맞아 보여도, 빌드 방식이 바뀌면 어긋날 수 있습니다.
특히 TypeScript, 번들러, minify, inline source map이 섞이면 환경별 결과가 달라질 수 있습니다.
간단한 테스트를 하나 두면 변경을 빨리 감지할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { captureSourceMappedCaller } from '../logger-callsite.js';

test('captures caller file and line', () => {
  const site = captureSourceMappedCaller();

  assert.ok(site);
  assert.match(site.file, /logger-callsite\.test\.(js|ts)$/);
  assert.equal(typeof site.line, 'number');
  assert.ok(site.line > 0);
});
```

정확한 라인 번호까지 고정하면 테스트가 너무 쉽게 깨질 수 있습니다.
대신 파일명, 양수 라인 번호, 상대 경로 변환 정도를 검증하는 편이 유지보수에 좋습니다.

## 운영 적용 체크리스트

### 성능과 로그량을 먼저 제한한다

`util.getCallSites()`는 진단 도구에 가깝습니다.
따라서 먼저 아래 기준을 정하고 적용하는 것이 좋습니다.

- hot path 로그에는 기본 비활성화한다.
- `frameCount`를 필요한 최소값으로 줄인다.
- debug, warn, error처럼 조사 가치가 큰 레벨부터 적용한다.
- 장애 조사 기간에만 켤 수 있는 환경 변수를 둔다.
- 로그 샘플링 정책과 함께 쓴다.

호출 위치가 붙은 로그는 문제를 빨리 좁히는 데 도움이 되지만, 로그 저장 비용과 처리 비용도 함께 늘립니다.
항상 켜는 기능이 아니라 "필요할 때 정확히 켤 수 있는 기능"으로 설계하는 편이 안전합니다.

### 경로와 함수명은 민감정보처럼 다룬다

파일 경로와 함수명은 비밀번호처럼 직접적인 민감정보는 아니지만, 시스템 구조를 드러낼 수 있습니다.
특히 사내 프로젝트명, 사용자 홈 디렉터리, 배포 경로, feature 이름이 파일명에 들어간 경우에는 외부 공유 로그에서 문제가 될 수 있습니다.

권장 기준은 아래와 같습니다.

- 절대 경로 대신 프로젝트 상대 경로를 남긴다.
- 공개 응답과 클라이언트 전송 이벤트에는 call site를 넣지 않는다.
- 로그 수집 도구에서 민감 경로 마스킹 규칙을 둔다.
- 샘플 로그를 문서에 넣을 때 실제 사용자명과 내부 경로를 바꾼다.
- source map을 공개 정적 경로에 두지 않는다.

이 기준은 `getCallSites()`에만 해당하지 않습니다.
스택 트레이스, source map, process report, 에러 리포팅 도구 전반에 같은 원칙을 적용해야 합니다.

### API 안정성 상태를 확인한다

공식 문서 기준으로 `util.getCallSites()`는 active development 안정성으로 표시됩니다.
또 과거에는 이름이 `util.getCallSite`에서 `util.getCallSites()`로 바뀐 이력이 있고, `column` 대신 `columnNumber` 사용이 권장되는 변화도 있었습니다.
따라서 라이브러리의 공개 API나 장기 지원 SDK의 핵심 경로에 바로 고정하기보다는, 내부 진단 유틸부터 점진적으로 쓰는 편이 좋습니다.

```js
export function tryGetCallSite() {
  if (typeof getCallSites !== 'function') {
    return undefined;
  }

  try {
    const [, caller] = getCallSites(2);
    return caller ? toInternalLogField(caller) : undefined;
  } catch {
    return undefined;
  }
}
```

로깅 보조 기능은 실패해도 업무 요청을 깨뜨리지 않아야 합니다.
호출 위치 캡처가 실패하면 필드를 생략하고, 핵심 로그 메시지와 요청 ID는 계속 남도록 설계합니다.

## 발행 전 점검

### SEO와 내부링크

- 제목과 설명에 `Node.js util.getCallSites` 핵심 키워드를 포함했다.
- 본문은 H2/H3 구조로 호출 위치, 로그 적용, source map, 운영 체크리스트를 나눴다.
- 내부링크를 3개 포함해 source maps, AsyncLocalStorage, diagnostics_channel 글과 연결했다.
- 코드 예제는 임의 토큰, 실제 경로, 개인정보를 포함하지 않는다.

### 안전성과 법적 리스크

- 불법 행위나 우회 방법을 안내하지 않는다.
- 실제 서버 경로, 사용자명, 토큰, API 키를 노출하지 않는다.
- `scriptName`과 source map 정보를 외부 응답에 넣지 말라고 명시했다.
- 공식 문서 기준 API 안정성 상태가 바뀔 수 있음을 전제로 내부 진단 유틸부터 적용하도록 안내했다.

## FAQ

### util.getCallSites를 모든 로그에 붙여도 되나요?

권장하지 않습니다.
스택 프레임 캡처는 일반 로그 필드 조합보다 비용이 크고, 파일 경로 정보도 늘어납니다.
먼저 debug, warn, error 레벨이나 장애 조사 모드에 제한해서 적용하는 편이 좋습니다.

### Error.stack을 완전히 대체할 수 있나요?

목적이 다릅니다.
`util.getCallSites()`는 호출 위치를 구조화해 얻는 데 좋고, `Error.stack`은 실제 에러가 발생한 흐름을 사람이 읽는 데 여전히 유용합니다.
로그 호출 위치 보강에는 `getCallSites()`가 적합하지만, 예외 원인 분석에는 에러 객체와 stack을 함께 남겨야 합니다.

### sourceMap 옵션만 켜면 TypeScript 원본 라인이 항상 나오나요?

아닙니다.
source map 파일 또는 inline map이 런타임에서 접근 가능해야 하고, 빌드 설정도 맞아야 합니다.
로컬, CI, 운영에서 같은 결과가 나오는지 작은 테스트와 샘플 로그로 확인하세요.

## 마무리

`util.getCallSites()`는 로그 호출 위치를 문자열 파싱이 아니라 구조화된 데이터로 다루게 해 주는 Node.js 진단 API입니다.
잘 쓰면 "이 로그가 어느 코드에서 찍혔는지"를 빠르게 좁힐 수 있고, source map과 결합하면 TypeScript 프로젝트의 원본 위치 추적에도 도움이 됩니다.

다만 모든 로그에 무조건 붙이는 기능은 아닙니다.
작은 `frameCount`, 환경 변수 기반 활성화, 상대 경로 변환, 외부 응답 분리, source map 보존 정책까지 함께 설계할 때 실무에서 오래 유지할 수 있습니다.

## 관련 글

- [Node.js source maps 가이드: 운영 에러 스택을 원본 TypeScript로 읽는 법](/development/blog/seo/2026/05/22/nodejs-enable-source-maps-production-debugging-guide.html)
- [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)
- [Node.js diagnostics_channel 가이드: 관측성 이벤트를 안전하게 분리하는 법](/development/blog/seo/2026/08/07/nodejs-diagnostics-channel-observability-guide.html)
- [Node.js 공식 문서: util.getCallSites](https://nodejs.org/api/util.html#utilgetcallsitesframecount-options)

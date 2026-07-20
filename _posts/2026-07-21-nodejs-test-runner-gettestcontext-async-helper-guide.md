---
layout: post
title: "Node.js getTestContext 가이드: 비동기 헬퍼에서 현재 테스트 정보 읽기"
date: 2026-07-21 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-gettestcontext-async-helper-guide
permalink: /development/blog/seo/2026/07/21/nodejs-test-runner-gettestcontext-async-helper-guide.html
alternates:
  ko: /development/blog/seo/2026/07/21/nodejs-test-runner-gettestcontext-async-helper-guide.html
  x_default: /development/blog/seo/2026/07/21/nodejs-test-runner-gettestcontext-async-helper-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, gettestcontext, test-context, async-helper, testing, diagnostics, ci]
description: "Node.js test runner의 getTestContext()로 비동기 헬퍼, fixture, 진단 함수에서 현재 테스트 이름과 파일 경로를 안전하게 읽는 방법을 정리합니다. context 전달 방식과 비교하고 CI 로그, 임시 리소스, 민감정보 없는 디버깅 예제를 설명합니다."
---

테스트 코드가 커질수록 공통 헬퍼가 늘어납니다.
임시 디렉터리를 만들고, 테스트용 서버를 띄우고, 실패했을 때 진단 로그를 남기고, CI artifact 이름을 정하는 함수들이 여기저기서 호출됩니다.
처음에는 `t`를 인자로 넘기면 충분하지만, 헬퍼가 여러 계층을 지나거나 비동기 콜백 안에서 실행되면 "현재 어떤 테스트 안에서 실행 중인지"를 알기 어려워집니다.

Node.js test runner의 `getTestContext()`는 현재 실행 중인 테스트 또는 suite의 context를 가져오는 API입니다.
공식 문서 기준으로 Node.js v26.1.0에 추가되었고, 테스트 함수 내부뿐 아니라 그 안에서 호출되는 비동기 작업에서도 현재 context를 읽을 수 있습니다.
이 글에서는 `getTestContext()`를 언제 쓰면 좋은지, 기존의 명시적 context 전달과 어떻게 나눠야 하는지, CI 로그와 fixture 헬퍼에서 안전하게 활용하는 방법을 정리합니다.
[Node.js test runner test context 가이드](/development/blog/seo/2026/07/11/nodejs-test-runner-test-context-guide.html), [Node.js test runner fixture helper 가이드](/development/blog/seo/2026/07/11/nodejs-test-runner-fixture-helper-temp-resource-guide.html), [Node.js test runner 이벤트 상관관계 가이드](/development/blog/seo/2026/07/20/nodejs-test-runner-event-correlation-custom-reporter-guide.html)와 함께 보면 테스트 헬퍼와 리포트 설계를 더 안정적으로 가져갈 수 있습니다.

## getTestContext가 해결하는 문제

### H3. 헬퍼가 현재 테스트 이름을 알아야 할 때가 있다

테스트 헬퍼는 보통 테스트 로직을 숨기기 위해 만듭니다.
하지만 운영하기 좋은 헬퍼는 단순히 기능만 수행하지 않고, 문제가 생겼을 때 어느 테스트에서 호출됐는지도 남깁니다.
예를 들어 임시 파일 이름, mock 서버 포트 로그, 실패 시 덤프 파일 이름에 테스트 이름이 들어가면 CI에서 원인을 찾기 쉽습니다.

기존 방식은 테스트 context를 헬퍼에 직접 넘기는 것입니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('creates invoice preview', async (t) => {
  const workspace = await createWorkspace(t);

  assert.ok(workspace.path.includes('creates-invoice-preview'));
});
```

이 방식은 명확하고 예측 가능합니다.
다만 헬퍼가 다시 내부 헬퍼를 부르고, 그 내부에서 로그 이름을 만들거나 cleanup을 등록해야 한다면 모든 계층에 `t`를 전달해야 합니다.
작은 코드에서는 괜찮지만, fixture 계층이 깊어질수록 함수 시그니처가 테스트 프레임워크에 강하게 묶입니다.

`getTestContext()`는 이런 경우 현재 테스트 context를 호출 지점에서 직접 가져오는 선택지를 제공합니다.
테스트 바깥에서 호출하면 `undefined`가 반환되므로, 일반 런타임 코드와 테스트 전용 헬퍼를 분리하는 기준도 함께 잡아야 합니다.

### H3. 비동기 콜백 안에서도 같은 테스트 맥락을 유지한다

테스트 헬퍼의 까다로운 부분은 비동기입니다.
`setTimeout()`, stream callback, server request handler, promise chain 안에서 진단 코드를 실행해야 할 때가 있습니다.
이때 전역 변수로 현재 테스트 이름을 저장하면 병렬 테스트에서 쉽게 꼬일 수 있습니다.

`getTestContext()`는 현재 실행 흐름에 연결된 테스트 또는 suite context를 반환합니다.
따라서 비동기 헬퍼 안에서 현재 테스트 이름, 전체 이름, 파일 경로 같은 정보를 읽어 로그를 안정적으로 꾸밀 수 있습니다.

```js
import { getTestContext } from 'node:test';

export function currentTestLabel() {
  const context = getTestContext();

  if (!context) {
    return 'outside-test';
  }

  return context.fullName ?? context.name;
}
```

이 함수는 테스트 안에서는 테스트 이름을, 테스트 바깥에서는 안전한 fallback을 반환합니다.
라이브러리 코드가 아니라 테스트 지원 코드에 두면 운영 코드가 Node.js test runner에 의존하는 문제도 줄일 수 있습니다.

## context 전달과 getTestContext 나누기

### H3. 테스트 동작을 바꾸는 헬퍼는 context를 명시적으로 받는다

모든 헬퍼를 `getTestContext()`로 바꾸는 것은 좋지 않습니다.
특히 테스트의 동작을 바꾸는 헬퍼는 context를 인자로 받는 편이 더 읽기 쉽습니다.

예를 들어 `t.mock`, `t.assert`, `t.test`, `t.after`처럼 context의 기능을 직접 쓰는 함수라면 인자를 명시하세요.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('uses payment gateway mock', async (t) => {
  const gateway = createGatewayMock(t);

  gateway.charge.mock.mockImplementationOnce(async () => ({
    status: 'approved'
  }));

  assert.equal((await gateway.charge()).status, 'approved');
});

function createGatewayMock(t) {
  const charge = t.mock.fn(async () => ({ status: 'pending' }));

  t.after(() => {
    charge.mock.restore();
  });

  return { charge };
}
```

이 코드는 헬퍼가 테스트 context를 사용한다는 사실이 호출부에 드러납니다.
나중에 테스트 runner를 바꾸거나 헬퍼를 일반 코드로 옮길 때도 의존성이 분명합니다.

### H3. 관찰용 헬퍼는 getTestContext를 쓰기 좋다

반대로 테스트 결과를 관찰하거나 이름을 붙이는 헬퍼는 `getTestContext()`와 잘 맞습니다.
대표적인 예는 다음과 같습니다.

- 실패 시 디버그 파일 이름 만들기
- 테스트별 임시 리소스 prefix 만들기
- 로그 라인에 테스트 이름 붙이기
- CI artifact 경로에 파일명과 테스트 이름 반영하기
- 테스트 바깥 실행에서는 조용히 fallback 처리하기

이런 헬퍼는 테스트의 동작을 바꾸기보다 주변 정보를 꾸밉니다.
따라서 호출부마다 `t`를 넘기지 않아도 코드 의도가 크게 흐려지지 않습니다.

```js
import { getTestContext } from 'node:test';
import { mkdir, writeFile } from 'node:fs/promises';
import { join } from 'node:path';

export async function writeDebugJson(name, value) {
  const context = getTestContext();
  const filePart = sanitizePathPart(context?.filePath ?? 'unknown-file');
  const testPart = sanitizePathPart(context?.fullName ?? context?.name ?? 'outside-test');
  const directory = join('reports', 'debug', filePart);

  await mkdir(directory, { recursive: true });

  const outputPath = join(directory, `${testPart}-${sanitizePathPart(name)}.json`);
  await writeFile(outputPath, `${JSON.stringify(redact(value), null, 2)}\n`);

  return outputPath;
}

function sanitizePathPart(value) {
  return String(value)
    .replace(process.cwd(), '')
    .replaceAll(/[^a-zA-Z0-9._-]+/g, '-')
    .replaceAll(/^-|-$/g, '')
    .slice(0, 120);
}

function redact(value) {
  return JSON.parse(
    JSON.stringify(value, (key, nestedValue) => {
      if (/token|secret|password|authorization/i.test(key)) {
        return '[REDACTED]';
      }

      return nestedValue;
    })
  );
}
```

이 예제는 테스트 context에서 파일 경로와 전체 테스트 이름을 가져와 debug artifact 경로를 만듭니다.
동시에 토큰, 비밀번호, authorization 같은 키는 저장 전에 마스킹합니다.
테스트 실패 로그는 CI에 오래 남을 수 있으므로, 편의 기능을 만들 때도 민감정보 제거를 기본값으로 두는 편이 안전합니다.

## fixture 헬퍼에서 활용하기

### H3. 테스트 이름 기반 임시 디렉터리를 만든다

임시 디렉터리를 테스트마다 분리하면 실패 후 파일을 확인하기 쉽습니다.
`getTestContext()`를 쓰면 헬퍼 내부에서 현재 테스트 이름을 읽어 prefix를 만들 수 있습니다.

```js
import { getTestContext } from 'node:test';
import { mkdtemp, rm } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

export async function createTempWorkspace(options = {}) {
  const context = getTestContext();
  const label = slugify(context?.fullName ?? context?.name ?? 'test');
  const directory = await mkdtemp(join(tmpdir(), `${label}-`));

  if (!options.keep) {
    context?.after?.(async () => {
      await rm(directory, { recursive: true, force: true });
    });
  }

  return directory;
}

function slugify(value) {
  return String(value)
    .toLowerCase()
    .replaceAll(/[^a-z0-9]+/g, '-')
    .replaceAll(/^-|-$/g, '')
    .slice(0, 80);
}
```

여기서 중요한 점은 `context`가 없을 수 있다는 사실을 처리하는 것입니다.
테스트 밖에서 헬퍼를 실수로 호출하더라도 즉시 알 수 있게 하려면 `undefined`일 때 예외를 던져도 됩니다.
반대로 CLI 스크립트와 테스트에서 같이 쓰는 헬퍼라면 위 예제처럼 fallback을 두는 편이 낫습니다.

### H3. cleanup 등록은 실패해도 실행되게 둔다

테스트에서 만든 리소스는 실패 여부와 무관하게 정리되어야 합니다.
현재 context를 가져올 수 있다면 `context.after()`에 cleanup을 등록할 수 있습니다.
다만 cleanup이 중요한 헬퍼라면 context가 없는 상황을 조용히 넘기지 않는 편이 안전합니다.

```js
import { getTestContext } from 'node:test';

export function registerCleanup(cleanup) {
  const context = getTestContext();

  if (!context) {
    throw new Error('registerCleanup() must be called inside a node:test test or suite');
  }

  context.after(cleanup);
}
```

이런 함수는 테스트 전용 유틸로 두는 것이 좋습니다.
일반 애플리케이션 코드에서 호출될 가능성이 있는 곳에 넣으면 Node.js test runner가 런타임 의존성으로 새어 나갑니다.

## CI 로그와 custom reporter에 연결하기

### H3. 테스트 안 로그에는 최소 맥락만 붙인다

CI 로그는 오래 보관되고 여러 사람이 볼 수 있습니다.
현재 테스트 이름을 붙이는 것은 좋지만, 요청 본문 전체나 사용자 이메일, bearer token 같은 값을 그대로 남기면 안 됩니다.

```js
import { getTestContext } from 'node:test';

export function testLog(message, fields = {}) {
  const context = getTestContext();
  const payload = {
    test: context?.fullName ?? context?.name ?? 'outside-test',
    message,
    ...safeFields(fields)
  };

  process.stderr.write(`${JSON.stringify(payload)}\n`);
}

function safeFields(fields) {
  const result = {};

  for (const [key, value] of Object.entries(fields)) {
    if (/token|secret|password|authorization|cookie/i.test(key)) {
      result[key] = '[REDACTED]';
    } else {
      result[key] = value;
    }
  }

  return result;
}
```

이 로그는 사람이 읽기에는 다소 건조하지만, 실패한 테스트와 관련된 이벤트를 검색하기 쉽습니다.
custom reporter가 `test:stderr` 이벤트를 수집한다면 이 JSON Lines를 후처리해 테스트별 진단 목록으로 묶을 수도 있습니다.

### H3. fullName과 filePath를 stable key처럼 쓰지는 않는다

`context.fullName`과 `context.filePath`는 디버깅에는 유용하지만 영구 식별자로 과신하면 안 됩니다.
테스트 이름을 바꾸거나 파일을 이동하면 값이 달라집니다.
대시보드에서 장기 추세를 추적하려면 테스트 이름과 파일 경로를 그대로 key로 쓰기보다 별도 test case id를 태그나 매핑 파일로 관리하는 편이 낫습니다.

일회성 CI artifact, 실패 로그, 임시 리소스 이름에는 `fullName`이 충분합니다.
반면 flaky test 히스토리, 품질 지표, 소유 팀 매핑처럼 장기적으로 누적되는 데이터에는 안정적인 키 설계를 따로 두세요.

## 도입 체크리스트

### H3. 작은 헬퍼부터 적용한다

`getTestContext()`는 테스트 구조를 크게 바꾸지 않아도 도입할 수 있습니다.
먼저 관찰용 헬퍼부터 적용하면 위험이 작습니다.

- 테스트별 debug artifact 이름 만들기
- fixture 임시 디렉터리 prefix 만들기
- 민감정보를 제거한 테스트 로그 남기기
- context가 없을 때 fallback 또는 명시적 에러 정책 정하기
- 테스트 동작을 바꾸는 헬퍼는 기존처럼 `t`를 명시적으로 받기

이 기준을 지키면 편의성과 명시성의 균형을 잡을 수 있습니다.
`getTestContext()`는 `t` 전달을 완전히 대체하는 API가 아니라, 깊은 비동기 헬퍼에서 현재 테스트 맥락을 읽는 보조 도구로 보는 편이 좋습니다.

## FAQ

### H3. getTestContext는 테스트 바깥에서 호출하면 어떻게 되나요?

현재 실행 중인 테스트나 suite가 없으면 `undefined`를 반환합니다.
테스트 전용 헬퍼라면 이 경우 예외를 던지고, 테스트와 CLI에서 함께 쓰는 헬퍼라면 안전한 fallback 값을 두는 방식이 좋습니다.

### H3. 모든 헬퍼에서 getTestContext를 쓰면 t 인자를 없앨 수 있나요?

권장하지 않습니다.
`t.mock`, `t.after`, `t.assert`처럼 테스트 동작을 직접 바꾸는 헬퍼는 context를 명시적으로 받는 편이 더 분명합니다.
`getTestContext()`는 로그, artifact 이름, fixture prefix처럼 관찰용 정보가 필요한 헬퍼에 먼저 쓰는 것이 좋습니다.

### H3. CI 로그에 테스트 context를 남길 때 가장 조심할 점은 무엇인가요?

민감정보 마스킹입니다.
테스트 이름과 파일 경로는 유용한 단서지만, 요청 헤더, 토큰, 쿠키, 이메일, 실제 사용자 데이터는 CI 로그에 남기지 않아야 합니다.
로그 헬퍼에서 key 기반 redaction을 기본값으로 두고, 필요한 값만 명시적으로 남기세요.

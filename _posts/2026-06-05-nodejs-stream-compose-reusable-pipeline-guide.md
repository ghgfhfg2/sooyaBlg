---
layout: post
title: "Node.js stream.compose 가이드: 재사용 가능한 스트림 파이프라인을 만드는 법"
date: 2026-06-05 20:00:00 +0900
lang: ko
translation_key: nodejs-stream-compose-reusable-pipeline-guide
permalink: /development/blog/seo/2026/06/05/nodejs-stream-compose-reusable-pipeline-guide.html
alternates:
  ko: /development/blog/seo/2026/06/05/nodejs-stream-compose-reusable-pipeline-guide.html
  x_default: /development/blog/seo/2026/06/05/nodejs-stream-compose-reusable-pipeline-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, stream, compose, pipeline, backpressure, backend]
description: "Node.js stream.compose로 여러 스트림 변환 단계를 하나의 재사용 가능한 Duplex 스트림으로 묶는 방법을 정리했습니다. pipeline과의 차이, 에러 처리, backpressure, 테스트 체크리스트를 예제로 설명합니다."
---

Node.js에서 스트림은 큰 파일, 로그, 압축 데이터, 네트워크 응답처럼 한 번에 메모리에 올리기 부담스러운 데이터를 다룰 때 강력합니다.
하지만 변환 단계가 늘어나면 `readable.pipe(a).pipe(b).pipe(writable)` 같은 코드가 여러 곳에 흩어지기 쉽습니다.
각 단계의 에러 처리, backpressure, 테스트 기준도 함께 흩어집니다.

`stream.compose()`는 여러 스트림이나 async iterable 변환을 하나의 `Duplex` 스트림처럼 묶어 주는 API입니다.
즉 "CSV 정리 파이프라인", "로그 마스킹 파이프라인", "압축 전처리 파이프라인"처럼 재사용 가능한 스트림 조합을 만들 수 있습니다.
이 글에서는 `stream.compose()`가 필요한 상황, `pipeline()`과의 차이, 실무에서 안전하게 쓰는 패턴과 테스트 포인트를 정리합니다.
스트림 기본 흐름과 브라우저 Web Stream 변환이 먼저 필요하다면 [Node.js Readable.fromWeb/toWeb 가이드](/development/blog/seo/2026/04/30/nodejs-readable-fromweb-toweb-stream-bridge-guide.html)를 함께 참고하세요.

## stream.compose가 필요한 이유

### H3. 스트림 변환 단계를 이름 붙여 재사용한다

스트림 코드는 처음에는 단순합니다.
파일을 읽고, 한두 번 변환하고, 결과를 씁니다.
문제는 같은 변환 조합이 여러 코드 경로에서 반복될 때 시작됩니다.

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { Transform } from 'node:stream';
import { pipeline } from 'node:stream/promises';

const trimLines = new Transform({
  transform(chunk, encoding, callback) {
    const text = String(chunk)
      .split('\n')
      .map((line) => line.trim())
      .join('\n');

    callback(null, text);
  }
});

await pipeline(
  createReadStream('input.log'),
  trimLines,
  createWriteStream('output.log')
);
```

이 코드가 한 곳에만 있다면 괜찮습니다.
하지만 trim, 마스킹, 필터링, 포맷 변환 같은 단계가 계속 붙고 여러 작업에서 재사용된다면 파이프라인 자체를 하나의 단위로 만들고 싶어집니다.
`stream.compose()`는 그 조합을 `Duplex` 스트림으로 돌려주기 때문에 다른 스트림에 다시 연결하기 쉽습니다.

### H3. pipeline은 실행, compose는 조합에 가깝다

`pipeline()`은 보통 입력 스트림부터 출력 스트림까지 연결해 작업을 끝까지 실행할 때 씁니다.
반면 `compose()`는 중간 변환 묶음을 하나의 스트림으로 만들어 둡니다.

```js
import { compose, Transform } from 'node:stream';

function replaceText(from, to) {
  return new Transform({
    transform(chunk, encoding, callback) {
      callback(null, String(chunk).replaceAll(from, to));
    }
  });
}

export function createLogSanitizer() {
  return compose(
    replaceText('token=', 'token=[redacted]'),
    replaceText('password=', 'password=[redacted]')
  );
}
```

이제 `createLogSanitizer()`는 여러 곳에서 같은 규칙으로 사용할 수 있습니다.
호출자는 내부 단계가 몇 개인지 몰라도 되고, 전체 조합을 하나의 스트림처럼 다룹니다.

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { createLogSanitizer } from './log-sanitizer.js';

await pipeline(
  createReadStream('server.log'),
  createLogSanitizer(),
  createWriteStream('server.safe.log')
);
```

민감정보를 다루는 파이프라인에서는 원문 토큰, 쿠키, 인증 헤더를 로그나 에러 메시지에 그대로 남기지 않는 기준이 중요합니다.
정규식 기반 마스킹이 필요하다면 [Node.js RegExp.escape 가이드](/development/blog/seo/2026/05/05/nodejs-regexp-escape-safe-dynamic-pattern-guide.html)도 함께 확인하면 좋습니다.

## 기본 사용법

### H3. Transform 스트림을 compose로 묶는다

`compose()`에 여러 스트림을 넘기면 첫 번째 스트림으로 쓰고 마지막 스트림에서 읽는 `Duplex` 스트림이 만들어집니다.
각 단계는 순서대로 연결됩니다.

```js
import { compose, Transform } from 'node:stream';

function normalizeNewlines() {
  return new Transform({
    transform(chunk, encoding, callback) {
      callback(null, String(chunk).replaceAll('\r\n', '\n'));
    }
  });
}

function removeBlankLines() {
  return new Transform({
    transform(chunk, encoding, callback) {
      const text = String(chunk)
        .split('\n')
        .filter((line) => line.trim() !== '')
        .join('\n');

      callback(null, text);
    }
  });
}

export function createTextCleanupStream() {
  return compose(
    normalizeNewlines(),
    removeBlankLines()
  );
}
```

이 함수의 장점은 의도가 분명하다는 점입니다.
호출자는 "텍스트 정리 스트림"이라는 이름만 보면 되고, 내부 단계는 테스트와 문서에서 따로 검증할 수 있습니다.
변환 단계를 추가하더라도 호출부를 크게 바꾸지 않아도 됩니다.

### H3. async generator 변환도 조합할 수 있다

스트림 변환을 꼭 `Transform` 클래스로만 작성할 필요는 없습니다.
입력을 순회하고 결과를 `yield`하는 async generator 형태가 더 읽기 쉬울 때도 있습니다.

```js
import { compose } from 'node:stream';

async function* upperCaseLines(source) {
  for await (const chunk of source) {
    yield String(chunk).toUpperCase();
  }
}

async function* prefixLines(source) {
  for await (const chunk of source) {
    const text = String(chunk)
      .split('\n')
      .map((line) => `[app] ${line}`)
      .join('\n');

    yield text;
  }
}

export function createDebugFormatter() {
  return compose(upperCaseLines, prefixLines);
}
```

async generator는 작은 변환 로직을 표현하기 좋습니다.
다만 chunk 경계가 줄 경계와 항상 일치하지 않는다는 점은 잊으면 안 됩니다.
정확한 line 단위 처리가 필요하면 버퍼를 유지하거나 `readline`, `filehandle.readLines()` 같은 API를 검토해야 합니다.
큰 로그 파일을 줄 단위로 처리하는 흐름은 [Node.js FileHandle.readLines 가이드](/development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html)와 같이 설계할 수 있습니다.

## 실무 적용 패턴

### H3. 변환 단계는 가능한 한 순수하게 둔다

재사용 가능한 파이프라인은 입력과 출력 계약이 단순해야 오래 갑니다.
각 단계가 외부 상태를 많이 읽거나, 전역 설정을 직접 바꾸거나, 네트워크 호출까지 수행하면 스트림 조합이 예측하기 어려워집니다.

좋은 변환 단계는 대체로 아래 기준을 지킵니다.

- 입력 chunk를 받아 출력 chunk를 만든다.
- 민감정보를 출력하지 않는다.
- 실패할 때는 명확한 에러를 던진다.
- 외부 설정은 생성자나 팩토리 인자로 받는다.
- 처리량이 큰 작업에서는 불필요한 전체 문자열 복사를 줄인다.

예를 들어 마스킹 규칙을 외부에서 주입하면 테스트하기 쉬운 스트림을 만들 수 있습니다.

```js
import { Transform } from 'node:stream';

export function createMaskingTransform(patterns) {
  return new Transform({
    transform(chunk, encoding, callback) {
      let text = String(chunk);

      for (const { pattern, replacement } of patterns) {
        text = text.replaceAll(pattern, replacement);
      }

      callback(null, text);
    }
  });
}
```

이렇게 만들면 운영 환경과 테스트 환경에서 같은 파이프라인 구조를 쓰되, 규칙만 바꿔 검증할 수 있습니다.

### H3. 에러 처리와 종료는 pipeline에 맡긴다

`compose()`가 조합을 만든다고 해서 최종 실행까지 대신 책임지는 것은 아닙니다.
파일 읽기부터 쓰기까지 실제 작업을 끝내야 한다면 `stream/promises`의 `pipeline()`으로 감싸는 편이 안전합니다.

```js
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createTextCleanupStream } from './cleanup.js';

export async function cleanupFile(inputPath, outputPath, { signal } = {}) {
  await pipeline(
    createReadStream(inputPath),
    createTextCleanupStream(),
    createWriteStream(outputPath),
    { signal }
  );
}
```

`pipeline()`은 중간 단계에서 에러가 나면 연결된 스트림들을 정리하는 실행 경로를 제공합니다.
취소 가능한 작업이라면 `AbortSignal`도 함께 넘겨 상위 요청 취소나 배치 종료 흐름과 맞출 수 있습니다.
취소 흐름은 [Node.js fetch AbortSignal timeout/retry 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)처럼 시작 전 체크와 실행 중 취소를 분리해 생각하면 좋습니다.

### H3. backpressure를 깨는 비동기 작업을 경계한다

스트림의 장점은 backpressure입니다.
쓰기 쪽이 느려지면 읽기 쪽도 자연스럽게 속도를 맞춥니다.
하지만 변환 단계 안에서 chunk를 배열에 계속 쌓거나, `Promise.all()`로 대량 비동기 작업을 한꺼번에 시작하면 이 장점이 사라질 수 있습니다.

나쁜 패턴은 아래처럼 생겼습니다.

```js
const pending = [];

async function* unsafeTransform(source) {
  for await (const chunk of source) {
    pending.push(expensiveAsyncWork(chunk));
  }

  yield* await Promise.all(pending);
}
```

이 코드는 입력 전체를 사실상 메모리에 모읍니다.
대용량 파일이나 긴 네트워크 스트림에서는 메모리 사용량이 예측하기 어려워집니다.
가능하면 chunk를 하나씩 처리하고 바로 `yield`하는 구조를 유지해야 합니다.

```js
async function* saferTransform(source) {
  for await (const chunk of source) {
    const result = await expensiveAsyncWork(chunk);
    yield result;
  }
}
```

동시성이 꼭 필요하다면 별도의 제한 큐나 작은 promise pool을 두고, 순서 보장 여부를 명확히 정해야 합니다.
무제한 동시성은 스트림을 쓰는 이유와 충돌할 수 있습니다.

## 테스트 체크리스트

### H3. 작은 입력으로 변환 계약을 검증한다

`compose()`로 만든 스트림은 단위 테스트하기 좋습니다.
작은 입력을 넣고 출력 문자열을 비교하면 파이프라인 전체 계약을 빠르게 확인할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { Readable } from 'node:stream';
import { text } from 'node:stream/consumers';
import test from 'node:test';
import { createTextCleanupStream } from './cleanup.js';

test('cleanup stream normalizes text', async () => {
  const stream = Readable
    .from(['a\r\n\r\nb\n'])
    .pipe(createTextCleanupStream());

  assert.equal(await text(stream), 'a\nb');
});
```

테스트 입력은 너무 크지 않아도 됩니다.
대신 빈 입력, 여러 chunk로 나뉜 입력, 에러가 나는 입력, 민감정보가 포함된 입력을 나눠 확인하는 편이 실무 리스크를 더 잘 줄입니다.
테스트 구조는 [Node.js test runner subtest 가이드](/development/blog/seo/2026/05/24/nodejs-test-runner-subtest-structure-guide.html)를 참고해 케이스별로 작게 나누면 유지보수가 쉽습니다.

### H3. 실패 케이스에서 모든 스트림이 종료되는지 본다

운영에서 더 중요한 것은 성공보다 실패입니다.
중간 변환 단계가 에러를 던졌을 때 입력 파일이 계속 읽히지 않는지, 출력 파일이 열린 채로 남지 않는지, 호출자가 reject를 받는지 확인해야 합니다.

```js
import { Transform } from 'node:stream';

function failOnKeyword(keyword) {
  return new Transform({
    transform(chunk, encoding, callback) {
      const text = String(chunk);

      if (text.includes(keyword)) {
        callback(new Error('blocked keyword found'));
        return;
      }

      callback(null, text);
    }
  });
}
```

에러 메시지에는 원문 데이터를 그대로 넣지 않는 편이 안전합니다.
특히 로그 마스킹 파이프라인에서 실패 메시지에 원문 chunk를 붙이면 마스킹하기 전의 민감정보가 새어 나갈 수 있습니다.

## 언제 쓰지 않는 편이 나은가

### H3. 단발성 작업에는 pipeline만으로 충분하다

한 파일을 한 번 읽고 쓰는 작은 스크립트라면 `compose()`까지 도입하지 않아도 됩니다.
재사용할 조합이 없고 호출부도 하나뿐이라면 `pipeline()`에 단계를 직접 나열하는 편이 더 단순합니다.

`compose()`는 파이프라인 자체가 도메인 개념이 될 때 가치가 큽니다.
예를 들어 "로그 정리", "CSV 표준화", "응답 압축 전처리", "사용자 입력 마스킹"처럼 이름 붙일 수 있는 조합이라면 별도 팩토리로 빼는 편이 좋습니다.

### H3. chunk 경계가 의미 있는 데이터에는 별도 파서를 고려한다

스트림 chunk는 임의의 경계로 나뉠 수 있습니다.
한 chunk가 한 줄, 한 JSON 객체, 한 CSV 레코드라는 보장은 없습니다.
따라서 레코드 경계가 중요한 데이터는 단순 문자열 replace보다 line parser, CSV parser, JSON streaming parser 같은 도구가 필요할 수 있습니다.

`compose()`는 이런 파서들을 연결하는 데도 쓸 수 있지만, 파싱 책임 자체를 없애 주지는 않습니다.
파이프라인을 만들 때는 "각 단계가 어떤 단위의 데이터를 입력으로 받는가"를 먼저 문서화해야 합니다.

## 마무리

`stream.compose()`는 스트림 변환 단계를 재사용 가능한 하나의 `Duplex` 스트림으로 묶는 데 유용합니다.
`pipeline()`이 실행과 정리를 책임지는 도구라면, `compose()`는 중간 변환 조합에 이름을 붙이는 도구에 가깝습니다.

실무에서는 세 가지 기준을 지키면 좋습니다.
첫째, 재사용할 만한 변환 묶음에만 `compose()`를 적용합니다.
둘째, 실제 파일·네트워크 작업은 `pipeline()`으로 실행해 에러와 종료를 관리합니다.
셋째, chunk 경계와 민감정보 노출을 테스트 케이스로 확인합니다.

이렇게 설계하면 스트림 파이프라인이 호출부마다 흩어지지 않고, 테스트 가능한 작은 모듈로 남습니다.
대용량 데이터를 다루는 백엔드 코드에서 이 차이는 유지보수성과 운영 안정성으로 이어집니다.

## 함께 보면 좋은 글

- [Node.js Readable.fromWeb/toWeb 가이드: Web Stream과 Node Stream을 연결하는 법](/development/blog/seo/2026/04/30/nodejs-readable-fromweb-toweb-stream-bridge-guide.html)
- [Node.js FileHandle.readLines 가이드: 큰 로그 파일을 줄 단위로 처리하는 법](/development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html)
- [Node.js test runner subtest 가이드: 테스트 구조를 읽기 쉽게 나누는 법](/development/blog/seo/2026/05/24/nodejs-test-runner-subtest-structure-guide.html)

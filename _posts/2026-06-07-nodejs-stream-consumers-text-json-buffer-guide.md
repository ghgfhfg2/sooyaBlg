---
layout: post
title: "Node.js stream/consumers 가이드: 스트림을 text, JSON, Buffer로 변환하는 법"
date: 2026-06-07 20:00:00 +0900
lang: ko
translation_key: nodejs-stream-consumers-text-json-buffer-guide
permalink: /development/blog/seo/2026/06/07/nodejs-stream-consumers-text-json-buffer-guide.html
alternates:
  ko: /development/blog/seo/2026/06/07/nodejs-stream-consumers-text-json-buffer-guide.html
  x_default: /development/blog/seo/2026/06/07/nodejs-stream-consumers-text-json-buffer-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, stream, consumers, text, json, buffer, backend]
description: "Node.js node:stream/consumers의 text, json, buffer, arrayBuffer, blob 함수로 스트림 전체를 간단히 읽는 방법을 정리했습니다. 메모리 위험, 크기 제한, 에러 처리와 테스트 기준을 실무 예제로 설명합니다."
---

Node.js에서 HTTP 응답 본문이나 파일 스트림을 한 번에 읽으려면 청크를 배열에 모으고 마지막에 합치는 코드를 자주 작성합니다.
문자열은 인코딩을 고려해야 하고, JSON은 문자열 변환 뒤 다시 파싱해야 하며, 에러와 종료 이벤트도 빠뜨리지 않아야 합니다.

`node:stream/consumers` 모듈은 이런 반복 작업을 줄여 줍니다.
`text()`, `json()`, `buffer()`, `arrayBuffer()`, `blob()` 함수에 스트림을 넘기면 전체 내용을 원하는 형식으로 소비한 결과를 Promise로 받을 수 있습니다.

다만 이 API는 스트림의 장점인 점진적 처리를 포기하고 **전체 데이터를 메모리에 올립니다**.
따라서 작은 설정 파일, 크기가 제한된 API 응답, 테스트 fixture처럼 입력 크기를 통제할 수 있는 곳에 사용하는 것이 중요합니다.

이 글에서는 `node:stream/consumers`의 기본 사용법, 형식별 선택 기준, 메모리 한도 설정, 에러 처리와 테스트 체크리스트를 정리합니다.
스트림 상태를 진단하는 방법이 필요하다면 [Node.js 스트림 상태 확인 가이드](/development/blog/seo/2026/06/07/nodejs-stream-status-isreadable-iswritable-iserrored-guide.html)를 함께 참고하세요.

## stream/consumers가 해결하는 문제

### H3. 청크를 직접 모으는 반복 코드를 줄인다

Readable 스트림 전체를 문자열로 만들기 위해 아래처럼 청크를 모을 수 있습니다.

```js
export async function readAll(readable) {
  const chunks = [];

  for await (const chunk of readable) {
    chunks.push(Buffer.from(chunk));
  }

  return Buffer.concat(chunks).toString('utf8');
}
```

이 코드는 동작하지만, 프로젝트마다 비슷한 함수를 다시 만들기 쉽습니다.
문자열이 아닌 JSON이나 `ArrayBuffer`가 필요하면 변환 단계도 추가해야 합니다.

`node:stream/consumers`의 `text()`를 사용하면 의도를 더 직접적으로 표현할 수 있습니다.

```js
import { text } from 'node:stream/consumers';

const content = await text(readable);
console.log(content);
```

함수는 스트림이 끝날 때까지 기다린 뒤 결과를 반환합니다.
스트림에서 에러가 발생하면 반환된 Promise도 거부되므로 호출부에서 일반 비동기 에러처럼 처리할 수 있습니다.

### H3. 작은 입력을 한 번에 다룰 때 적합하다

`stream/consumers`는 다음과 같이 전체 내용을 한 번에 사용해야 하는 작업에 잘 맞습니다.

- 크기가 제한된 JSON 설정 파일 읽기
- 내부 API의 작은 응답 본문 처리
- 테스트에서 Readable 스트림 결과 검증
- 작은 텍스트 템플릿이나 메타데이터 읽기
- Buffer 또는 Blob이 필요한 라이브러리와 연결

반대로 대용량 로그, 영상, 백업 파일, 크기를 신뢰할 수 없는 업로드에는 적합하지 않습니다.
이런 입력은 스트림을 청크 단위로 처리하거나 파일로 바로 기록해야 합니다.

## 형식별 기본 사용법

### H3. text로 UTF-8 문자열을 읽는다

`text()`는 스트림 전체를 UTF-8 문자열로 반환합니다.
작은 텍스트 파일이나 API 응답을 읽을 때 가장 단순한 선택입니다.

```js
import { createReadStream } from 'node:fs';
import { text } from 'node:stream/consumers';

const readable = createReadStream('./message.txt');
const message = await text(readable);

console.log(message);
```

텍스트를 읽은 뒤에는 원래 스트림을 다시 읽을 수 없다는 점에 주의해야 합니다.
consumer 함수가 스트림을 끝까지 소비했기 때문입니다.
같은 내용을 여러 곳에서 사용해야 한다면 반환된 결과를 전달하거나, 처음부터 데이터 흐름을 분기하는 설계를 검토해야 합니다.

### H3. json으로 파싱 결과를 바로 받는다

`json()`은 스트림을 모두 읽고 JSON으로 파싱한 값을 반환합니다.

```js
import { createReadStream } from 'node:fs';
import { json } from 'node:stream/consumers';

const readable = createReadStream('./config.json');
const config = await json(readable);

console.log(config.features);
```

문법이 잘못된 JSON이면 Promise가 거부됩니다.
하지만 JSON 파싱 성공은 애플리케이션에서 필요한 구조까지 보장하지 않습니다.
필수 속성, 값의 타입, 허용 범위는 별도 스키마나 검증 함수로 확인해야 합니다.

```js
function validateConfig(value) {
  if (!value || typeof value !== 'object') {
    throw new TypeError('config must be an object');
  }

  if (!Array.isArray(value.features)) {
    throw new TypeError('config.features must be an array');
  }

  return value;
}

const config = validateConfig(await json(readable));
```

외부 입력을 JSON으로 읽을 때는 파싱과 정책 검증을 분리하면 실패 원인을 더 명확하게 다룰 수 있습니다.

### H3. buffer와 arrayBuffer를 목적에 맞게 선택한다

Node.js 라이브러리와 파일 API를 주로 사용한다면 `buffer()`가 자연스럽습니다.

```js
import { createReadStream } from 'node:fs';
import { buffer } from 'node:stream/consumers';

const readable = createReadStream('./thumbnail.png');
const image = await buffer(readable);

console.log(image.length);
```

웹 표준 API나 브라우저 코드와 데이터를 주고받는다면 `arrayBuffer()`가 편리할 수 있습니다.

```js
import { createReadStream } from 'node:fs';
import { arrayBuffer } from 'node:stream/consumers';

const readable = createReadStream('./payload.bin');
const payload = await arrayBuffer(readable);

console.log(payload.byteLength);
```

두 함수 모두 전체 입력을 메모리에 보관한다는 점은 같습니다.
형식 선택보다 먼저 입력 크기가 충분히 작은지 확인해야 합니다.

### H3. blob으로 타입 정보를 함께 전달한다

`blob()`은 스트림 내용을 `Blob`으로 반환합니다.
`FormData`나 웹 표준 API와 연결할 때 활용할 수 있습니다.

```js
import { createReadStream } from 'node:fs';
import { blob } from 'node:stream/consumers';

const readable = createReadStream('./document.bin');
const data = await blob(readable);

console.log(data.size);
```

파일의 실제 형식과 MIME 타입은 별도 검증 대상입니다.
확장자나 사용자가 제공한 `Content-Type`만 믿고 안전한 파일이라고 판단하면 안 됩니다.

## 메모리 사용량을 제한하는 방법

### H3. 전체 소비 전에 최대 바이트 수를 정한다

consumer 함수는 스트림 끝까지 데이터를 모읍니다.
크기를 알 수 없는 입력을 그대로 넘기면 예상보다 큰 데이터가 프로세스 메모리를 차지할 수 있습니다.

아래 예제는 Transform 스트림에서 누적 바이트 수를 확인하고 한도를 넘으면 에러를 발생시킵니다.

```js
import { Transform } from 'node:stream';
import { text } from 'node:stream/consumers';

class ByteLimitTransform extends Transform {
  #received = 0;

  constructor(maxBytes) {
    super();
    this.maxBytes = maxBytes;
  }

  _transform(chunk, encoding, callback) {
    this.#received += chunk.length;

    if (this.#received > this.maxBytes) {
      callback(new Error('input exceeds the allowed size'));
      return;
    }

    callback(null, chunk);
  }
}

export async function readSmallText(readable, maxBytes = 1024 * 1024) {
  return text(readable.pipe(new ByteLimitTransform(maxBytes)));
}
```

이 예제는 일반적인 바이너리 또는 텍스트 Node.js 스트림을 대상으로 합니다.
object mode 스트림에서는 `chunk.length`가 바이트 크기를 뜻하지 않으므로 별도 정책이 필요합니다.

### H3. Content-Length만으로 안전을 보장하지 않는다

HTTP 응답의 `Content-Length` 헤더를 먼저 확인하면 너무 큰 입력을 일찍 거부하는 데 도움이 됩니다.
하지만 헤더가 없거나 실제 전송량과 다를 수 있으므로, 신뢰할 수 없는 입력에서는 실제로 읽은 바이트 수도 함께 제한해야 합니다.

크기 제한을 설계할 때는 다음 기준을 정해 두는 편이 좋습니다.

- 허용할 최대 바이트 수
- 한도 초과 시 사용할 에러 타입과 상태 코드
- 입력 스트림을 즉시 중단할지 여부
- 로그에 남길 필드와 제외할 민감정보
- 동시 요청 수를 고려한 전체 메모리 예산

개별 입력이 작아도 동시에 많은 요청이 들어오면 메모리 사용량이 커질 수 있습니다.
동시 실행 수를 제한하는 방법은 [Node.js Promise 동시성 제한 가이드](/development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)에서 확인할 수 있습니다.

## 에러와 취소 처리

### H3. 스트림 에러와 파싱 에러를 구분한다

`json()`을 사용할 때는 데이터를 읽는 과정과 JSON 파싱 과정 모두 실패할 수 있습니다.
호출부에서는 사용자에게 내부 에러 내용을 그대로 노출하지 않으면서, 운영 로그에는 원인을 구분할 정보를 남기는 것이 좋습니다.

```js
import { json } from 'node:stream/consumers';

export async function readJsonBody(readable) {
  try {
    return await json(readable);
  } catch (error) {
    console.error({
      event: 'json-body-read-failed',
      name: error.name,
      code: error.code
    });

    throw new Error('request body could not be processed');
  }
}
```

로그에 원문 본문 전체를 남기면 토큰, 개인정보, 사용자 데이터가 노출될 수 있습니다.
입력 내용 대신 에러 이름, 코드, 허용 한도, 실제 수신 크기처럼 진단에 필요한 최소 정보만 기록해야 합니다.

### H3. 취소는 원본 스트림의 생성 단계부터 연결한다

consumer 함수가 Promise를 반환한다고 해서 임의의 시점에 자동으로 취소할 수 있는 것은 아닙니다.
HTTP 요청, 파일 읽기, 상위 작업에서 `AbortSignal`을 지원한다면 스트림을 만드는 단계부터 취소 신호를 전달해야 합니다.

여러 종료 조건을 하나로 묶어야 한다면 [Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)를 참고할 수 있습니다.
핵심은 consumer만 감싸는 것이 아니라 데이터 생산자와 중간 처리 단계까지 같은 취소 정책을 적용하는 것입니다.

## 테스트에서 확인할 항목

### H3. 정상 결과와 실패 경로를 함께 검증한다

작은 `Readable.from()` 스트림을 사용하면 consumer 함수의 결과를 간단히 테스트할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { Readable } from 'node:stream';
import { json, text } from 'node:stream/consumers';

test('text joins all chunks', async () => {
  const readable = Readable.from(['hello', ' ', 'stream']);

  assert.equal(await text(readable), 'hello stream');
});

test('json parses a valid document', async () => {
  const readable = Readable.from(['{"enabled":', 'true}']);

  assert.deepEqual(await json(readable), { enabled: true });
});

test('json rejects invalid input', async () => {
  const readable = Readable.from(['{"enabled":']);

  await assert.rejects(() => json(readable), SyntaxError);
});
```

크기 제한을 추가했다면 한도 직전, 정확히 한도인 경우, 한도를 넘는 경우를 각각 테스트해야 합니다.
한글처럼 UTF-8에서 여러 바이트를 차지하는 문자열도 포함하면 문자 수와 바이트 수를 혼동하는 실수를 줄일 수 있습니다.

## 실무 선택 체크리스트

### H3. consumer 함수를 사용하기 전에 확인한다

- 입력 전체를 메모리에 올려도 되는가?
- 최대 크기를 코드에서 강제하고 있는가?
- 결과 형식이 `text`, `json`, `buffer`, `arrayBuffer`, `blob` 중 무엇인가?
- JSON 파싱 뒤 구조와 값을 별도로 검증하는가?
- 원본 스트림 에러와 파싱 에러를 처리하는가?
- 취소와 타임아웃이 데이터 생산 단계까지 연결되는가?
- 로그에서 본문과 민감정보를 제외했는가?
- 대용량 입력은 점진적 처리 방식으로 분리했는가?

## FAQ

### H3. stream/consumers는 대용량 파일에도 사용할 수 있나요?

기술적으로는 읽을 수 있지만 권장하지 않습니다.
전체 파일을 메모리에 모으기 때문에 파일 크기와 동시 처리량에 따라 프로세스 메모리가 급격히 늘어날 수 있습니다.
대용량 입력은 `pipeline()`과 Transform 스트림으로 청크 단위 처리하는 편이 안전합니다.

### H3. json은 데이터 스키마도 검증하나요?

아닙니다.
`json()`은 전체 내용을 JSON 문법에 따라 파싱할 뿐입니다.
필수 필드, 타입, 허용 값 같은 애플리케이션 규칙은 별도 검증이 필요합니다.

### H3. consumer 함수를 호출한 뒤 같은 스트림을 다시 읽을 수 있나요?

일반적으로 읽을 수 없습니다.
consumer 함수가 스트림을 끝까지 소비하기 때문입니다.
결과를 재사용하거나, 필요한 경우 데이터 흐름을 미리 분기해야 합니다.

## 마무리

`node:stream/consumers`는 작은 스트림 전체를 문자열, JSON, Buffer 같은 값으로 바꾸는 반복 코드를 간결하게 만들어 줍니다.
특히 테스트, 제한된 크기의 설정 파일, 작은 내부 API 응답을 처리할 때 의도가 잘 드러납니다.

하지만 편리함의 대가는 전체 입력을 메모리에 올린다는 점입니다.
입력 크기 제한, 동시 처리량, 에러와 취소 정책을 먼저 정하고, 대용량 데이터는 스트림답게 점진적으로 처리해야 합니다.

---
layout: post
title: "Node.js TransformStream 가이드: Web Streams로 변환 파이프라인과 backpressure 관리하기"
date: 2026-08-28 08:00:00 +0900
lang: ko
translation_key: nodejs-transformstream-backpressure-web-streams-guide
permalink: /development/blog/seo/2026/08/28/nodejs-transformstream-backpressure-web-streams-guide.html
alternates:
  ko: /development/blog/seo/2026/08/28/nodejs-transformstream-backpressure-web-streams-guide.html
  x_default: /development/blog/seo/2026/08/28/nodejs-transformstream-backpressure-web-streams-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, transformstream, web-streams, backpressure, stream, javascript, performance]
description: "Node.js TransformStream으로 Web Streams 기반 변환 파이프라인을 만드는 방법을 정리합니다. backpressure, TextEncoderStream, TextDecoderStream, pipeThrough, 에러 처리, 테스트 기준을 실무 예제로 설명합니다."
---

Node.js에서 스트림 코드를 작성하다 보면 두 가지 세계를 함께 만나게 됩니다.
오래전부터 쓰이던 `node:stream` 기반 스트림과, 브라우저 표준에 가까운 Web Streams API입니다.
최근 런타임에서는 `ReadableStream`, `WritableStream`, `TransformStream` 같은 Web Streams 타입을 서버 코드에서도 자연스럽게 사용할 수 있습니다.

특히 `TransformStream`은 입력을 받아 다른 형태로 바꾸는 중간 단계에 적합합니다.
JSON Lines를 객체로 바꾸거나, 텍스트를 줄 단위로 나누거나, 응답 본문을 압축 전에 정규화하는 식의 작업을 작은 변환 단위로 분리할 수 있습니다.
중요한 점은 변환 코드가 빨리 끝나는지만 보는 것이 아니라, 소비자가 느릴 때 생산 속도를 어떻게 늦출지까지 함께 생각해야 한다는 것입니다.

이 글에서는 Node.js에서 `TransformStream`을 사용해 변환 파이프라인을 만들고, `pipeThrough`, `TextDecoderStream`, `TextEncoderStream`, backpressure, 에러 처리를 실무 기준으로 정리합니다.
Node.js 스트림과 Web Streams를 연결하는 기준은 [Readable.fromWeb/toWeb 가이드](/development/blog/seo/2026/04/30/nodejs-readable-fromweb-toweb-stream-bridge-guide.html), 재사용 가능한 스트림 조립은 [stream.compose 가이드](/development/blog/seo/2026/06/05/nodejs-stream-compose-reusable-pipeline-guide.html), gzip/brotli 같은 압축 단계는 [CompressionStream 가이드](/development/blog/seo/2026/08/24/nodejs-compressionstream-decompressionstream-web-streams-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js Web Streams API 공식 문서](https://nodejs.org/api/webstreams.html)

## TransformStream을 쓰기 좋은 경우

### 데이터를 한 번에 메모리에 올리지 않을 때

작은 문자열이나 작은 JSON 파일은 `await response.text()`로 한 번에 읽어도 문제가 크지 않습니다.
하지만 로그 파일, 이벤트 스트림, 대용량 CSV, 외부 API의 긴 응답처럼 입력 크기가 커질 수 있는 데이터는 한 번에 모으는 방식이 위험합니다.

`TransformStream`은 데이터가 들어오는 대로 조각을 처리하고 다음 단계로 넘깁니다.
전체 입력을 다 읽은 뒤 변환하는 대신, 읽기와 변환과 쓰기를 겹쳐서 처리할 수 있습니다.

예를 들어 아래 작업은 변환 스트림으로 나누기 좋습니다.

- 바이트 스트림을 UTF-8 텍스트로 디코딩한다.
- 텍스트를 줄 단위로 나눈다.
- 각 줄을 JSON으로 파싱한다.
- 필요한 필드만 남기거나 민감한 필드를 마스킹한다.
- 다시 문자열이나 바이트 스트림으로 인코딩한다.

이런 단계를 함수 하나에 몰아넣으면 테스트하기 어렵고, 실패 지점도 흐려집니다.
반대로 작은 `TransformStream`을 이어 붙이면 각 단계의 입력과 출력이 명확해집니다.

### 브라우저와 서버의 스트림 모델을 맞추고 싶을 때

Web Streams API는 Fetch 응답 본문, `CompressionStream`, `TextDecoderStream` 같은 표준 API와 잘 맞습니다.
서버와 브라우저에서 비슷한 스트림 조합을 공유하고 싶다면 `TransformStream`이 좋은 공통 분모가 됩니다.

Node.js 전통 스트림을 이미 많이 쓰는 프로젝트라면 한 번에 모두 바꿀 필요는 없습니다.
경계에서는 `Readable.toWeb()`, `Readable.fromWeb()` 같은 bridge를 사용하고, 새로 작성하는 변환 로직만 Web Streams 스타일로 분리할 수 있습니다.

## 기본 구조

### transform 메서드에서 chunk를 처리한다

`TransformStream`은 보통 `transform(chunk, controller)` 메서드를 받습니다.
입력 chunk를 처리한 뒤 `controller.enqueue()`로 다음 단계에 넘길 값을 넣습니다.

```js
function createUppercaseStream() {
  return new TransformStream({
    transform(chunk, controller) {
      controller.enqueue(String(chunk).toUpperCase());
    },
  });
}

const stream = new ReadableStream({
  start(controller) {
    controller.enqueue('node');
    controller.enqueue(' streams');
    controller.close();
  },
});

const result = stream
  .pipeThrough(createUppercaseStream())
  .pipeThrough(new TextEncoderStream());
```

이 예제에서는 문자열을 대문자로 바꾼 뒤 바이트 스트림으로 인코딩합니다.
실제 코드에서는 입력 타입을 더 명확히 관리하는 편이 좋습니다.
바이트를 문자열로 다루려면 `TextDecoderStream`을 앞에 두고, 문자열을 바이트로 내보내려면 `TextEncoderStream`을 뒤에 둡니다.

### pipeThrough로 변환 단계를 연결한다

`pipeThrough()`는 읽기 가능한 스트림을 변환 스트림에 연결하고, 변환 결과의 readable 쪽을 돌려줍니다.
덕분에 여러 단계를 왼쪽에서 오른쪽으로 읽히는 형태로 연결할 수 있습니다.

```js
const transformed = response.body
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(createLineSplitStream())
  .pipeThrough(createJsonParseStream())
  .pipeThrough(createRedactionStream())
  .pipeThrough(new TextEncoderStream());
```

이 구조는 파이프라인의 의도를 잘 드러냅니다.
응답 본문을 텍스트로 디코딩하고, 줄 단위로 나눈 뒤, JSON을 파싱하고, 민감 필드를 제거하고, 다시 바이트로 인코딩한다는 흐름이 한눈에 보입니다.

## 줄 단위 변환 스트림 만들기

### flush에서 남은 버퍼를 처리한다

네트워크나 파일 스트림의 chunk 경계는 줄 경계와 일치하지 않습니다.
한 chunk 안에 여러 줄이 들어올 수도 있고, 한 줄이 여러 chunk로 나뉠 수도 있습니다.
따라서 줄 단위 변환을 만들 때는 남은 문자열 버퍼를 관리해야 합니다.

```js
function createLineSplitStream() {
  let buffer = '';

  return new TransformStream({
    transform(chunk, controller) {
      buffer += chunk;

      const lines = buffer.split('\n');
      buffer = lines.pop() ?? '';

      for (const line of lines) {
        controller.enqueue(line);
      }
    },

    flush(controller) {
      if (buffer.length > 0) {
        controller.enqueue(buffer);
      }
    },
  });
}
```

`transform`에서는 완성된 줄만 내보내고, 마지막 미완성 조각은 `buffer`에 남깁니다.
입력이 끝나면 `flush`가 호출되므로, 마지막 줄에 개행 문자가 없어도 누락되지 않습니다.

이 코드는 단순한 `\n` 기준 예제입니다.
윈도우 줄바꿈까지 정리해야 한다면 `line.replace(/\r$/, '')` 같은 정규화 단계를 추가할 수 있습니다.

### JSON Lines 파서는 실패 위치를 남긴다

JSON Lines를 처리할 때는 각 줄이 독립적인 JSON 문서입니다.
한 줄 파싱에 실패했을 때 전체 파일 이름이나 줄 번호를 함께 남기면 운영 로그에서 원인 파악이 쉬워집니다.

```js
function createJsonParseStream() {
  let lineNumber = 0;

  return new TransformStream({
    transform(line, controller) {
      lineNumber += 1;

      if (line.trim() === '') {
        return;
      }

      try {
        controller.enqueue(JSON.parse(line));
      } catch (error) {
        throw new Error(`Invalid JSON at line ${lineNumber}`, {
          cause: error,
        });
      }
    },
  });
}
```

여기서 빈 줄은 건너뜁니다.
팀 규칙에 따라 빈 줄을 오류로 처리할 수도 있습니다.
중요한 것은 파서의 허용 범위를 명확히 정하고 테스트로 고정하는 것입니다.

## backpressure를 의식한 설계

### 소비자가 느리면 생산도 느려져야 한다

스트림의 핵심 장점은 backpressure입니다.
다음 단계가 데이터를 빨리 소비하지 못하면 이전 단계도 무작정 데이터를 밀어 넣지 않아야 합니다.
그렇지 않으면 메모리에 대기 중인 chunk가 계속 쌓입니다.

`pipeThrough()`와 `pipeTo()`로 연결된 Web Streams 파이프라인은 기본적으로 backpressure를 전파합니다.
따라서 직접 배열에 모든 결과를 모으는 코드보다 안전합니다.

```js
await source
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(createLineSplitStream())
  .pipeThrough(createJsonParseStream())
  .pipeTo(createSlowDatabaseWriter());
```

이 예제에서 `createSlowDatabaseWriter()`가 느리게 처리하면 upstream도 그 속도에 맞춰 흐릅니다.
물론 변환 단계 안에서 별도 큐를 만들거나, `Promise.all()`로 무제한 병렬 처리를 시작하면 이 장점이 깨질 수 있습니다.

### transform 안의 비동기 작업은 제한을 둔다

`transform` 메서드는 async 함수가 될 수 있습니다.
예를 들어 각 이벤트를 외부 API로 보강해야 한다면 아래처럼 작성할 수 있습니다.

```js
function createEnrichmentStream(fetchUser) {
  return new TransformStream({
    async transform(event, controller) {
      const user = await fetchUser(event.userId);

      controller.enqueue({
        ...event,
        userTier: user.tier,
      });
    },
  });
}
```

이 방식은 이해하기 쉽고 backpressure도 다루기 쉽습니다.
다만 한 번에 하나씩 처리하므로 외부 호출이 많으면 느릴 수 있습니다.

병렬 처리가 필요하다면 무제한 병렬로 바꾸기보다 별도 제한을 둔 설계를 선택해야 합니다.
예를 들어 입력을 작은 batch로 묶고 batch 단위로만 병렬 처리하거나, downstream 쓰기 처리량에 맞춰 동시 실행 수를 제한하는 식입니다.
속도를 올리기 위해 backpressure를 우회하면 장애 상황에서 메모리 사용량과 외부 API 호출량이 동시에 튈 수 있습니다.

## 에러와 취소 처리

### throw하면 파이프라인이 실패한다

`transform` 안에서 예외를 던지면 파이프라인은 실패합니다.
데이터 정합성이 중요한 작업에서는 잘못된 입력을 조용히 버리는 것보다 빠르게 실패시키는 편이 안전합니다.

```js
function createRequiredFieldStream(fieldName) {
  return new TransformStream({
    transform(record, controller) {
      if (!(fieldName in record)) {
        throw new Error(`Missing required field: ${fieldName}`);
      }

      controller.enqueue(record);
    },
  });
}
```

이런 검증 스트림은 ETL 작업이나 로그 수집 파이프라인에서 유용합니다.
다만 운영 데이터가 일부 깨져도 계속 처리해야 하는 시스템이라면 오류를 별도 dead-letter 경로로 보내는 설계를 고려해야 합니다.

### AbortSignal로 긴 작업을 중단한다

파이프라인이 외부 요청, 파일 읽기, 업로드 같은 긴 작업과 연결되어 있다면 취소 경로를 준비해야 합니다.
`pipeTo()`에는 `signal` 옵션을 넘길 수 있습니다.

```js
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 30_000);

try {
  await source
    .pipeThrough(new TextDecoderStream())
    .pipeThrough(createLineSplitStream())
    .pipeThrough(createJsonParseStream())
    .pipeTo(destination, { signal: controller.signal });
} finally {
  clearTimeout(timeout);
}
```

시간 제한은 장애 전파를 줄이는 기본 장치입니다.
외부 시스템이 느려졌을 때 작업이 끝없이 대기하면 다음 배치나 다음 요청에도 영향을 줍니다.
취소가 발생했을 때 destination이 파일 핸들, 네트워크 연결, 임시 리소스를 정리하는지도 함께 테스트해야 합니다.

## 테스트 기준

### 작은 입력으로 정상 흐름을 고정한다

TransformStream은 입출력 타입이 명확해서 단위 테스트를 쓰기 좋습니다.
테스트에서는 작은 `ReadableStream`을 만들고 결과를 배열로 모으면 됩니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

async function collect(stream) {
  const values = [];

  for await (const value of stream) {
    values.push(value);
  }

  return values;
}

test('splits text into lines', async () => {
  const source = new ReadableStream({
    start(controller) {
      controller.enqueue('a\nb');
      controller.enqueue('\nc');
      controller.close();
    },
  });

  const lines = await collect(source.pipeThrough(createLineSplitStream()));

  assert.deepEqual(lines, ['a', 'b', 'c']);
});
```

테스트 데이터는 chunk 경계가 줄 경계와 어긋나도록 구성하는 것이 좋습니다.
그래야 실제 네트워크 입력에서 흔히 생기는 경계 문제를 잡을 수 있습니다.

### 실패 케이스도 스트림으로 검증한다

파싱 실패나 필수 필드 누락도 별도로 검증해야 합니다.
스트림 테스트에서 중요한 것은 오류가 발생한다는 사실뿐 아니라, 어디에서 발생했는지 알 수 있는 메시지를 남기는 것입니다.

```js
test('throws with line number for invalid JSON', async () => {
  const source = new ReadableStream({
    start(controller) {
      controller.enqueue('{"ok":true}\n');
      controller.enqueue('not-json\n');
      controller.close();
    },
  });

  await assert.rejects(
    collect(source.pipeThrough(createJsonParseStream())),
    /Invalid JSON at line 2/,
  );
});
```

운영 장애를 줄이는 테스트는 성공 경로보다 실패 경로에서 더 자주 나옵니다.
특히 스트림 파이프라인은 중간 단계가 많으므로, 어느 단계에서 어떤 오류가 났는지 추적할 수 있어야 합니다.

## 운영 적용 체크리스트

### 변환 단계를 작게 유지한다

`TransformStream` 하나가 너무 많은 일을 하면 재사용성과 테스트성이 떨어집니다.
디코딩, 줄 분리, 파싱, 검증, 마스킹, 인코딩을 각각 작은 단계로 나누면 변경 영향 범위를 줄일 수 있습니다.

실무에서는 아래 기준으로 나누면 무리가 적습니다.

- 입력 타입과 출력 타입이 달라지는 지점에서 분리한다.
- 실패 메시지가 달라지는 검증 단계는 별도 스트림으로 둔다.
- 보안 마스킹은 파싱 이후, 인코딩 이전에 수행한다.
- 외부 호출이 필요한 enrichment는 timeout과 재시도 정책을 따로 둔다.
- 최종 쓰기 단계는 `WritableStream`으로 분리해 backpressure가 전파되게 한다.

이 기준을 적용하면 파이프라인이 길어져도 각 단계의 책임은 작게 유지됩니다.

### 메모리 사용량을 로그로 확인한다

스트림을 쓴다고 해서 자동으로 메모리 문제가 사라지는 것은 아닙니다.
중간에서 결과를 배열에 계속 모으거나, batch 크기를 과하게 잡거나, 병렬 작업을 제한 없이 만들면 결국 메모리가 증가합니다.

대용량 입력을 처리하는 작업에는 최소한 다음 지표를 관찰하는 편이 좋습니다.

- 처리한 레코드 수
- 처리 시간과 초당 처리량
- 실패한 레코드 수
- batch 크기와 대기 중인 작업 수
- RSS와 heap used 변화

관찰 지표가 있어야 backpressure가 실제로 작동하는지 판단할 수 있습니다.
로컬 샘플 데이터에서는 괜찮아 보여도 운영 입력 크기에서는 병목이 전혀 다른 곳에서 나타날 수 있습니다.

## 마무리

`TransformStream`은 Node.js에서 Web Streams 기반 변환 파이프라인을 작고 읽기 쉬운 단위로 구성하게 해줍니다.
`pipeThrough()`로 단계를 연결하면 디코딩, 줄 분리, 파싱, 검증, 마스킹, 인코딩 같은 흐름을 코드 구조 그대로 드러낼 수 있습니다.

다만 스트림을 쓴다는 사실만으로 안정성이 보장되지는 않습니다.
chunk 경계, `flush` 처리, 실패 메시지, 취소 경로, 외부 호출 병렬도, 메모리 관찰까지 함께 설계해야 실제 운영에서 흔들리지 않습니다.

새 변환 파이프라인을 만들 때는 먼저 작은 `TransformStream`으로 책임을 나누고, 정상 입력과 실패 입력을 모두 테스트하세요.
그 다음 backpressure가 깨지는 지점이 없는지 확인하면, 대용량 데이터를 한 번에 메모리에 올리지 않고도 재현 가능한 처리 흐름을 만들 수 있습니다.

## 관련 글

- [Node.js Readable.fromWeb/toWeb 가이드: Web Streams와 Node 스트림 연결하기](/development/blog/seo/2026/04/30/nodejs-readable-fromweb-toweb-stream-bridge-guide.html)
- [Node.js stream.compose 가이드: 재사용 가능한 스트림 파이프라인 만들기](/development/blog/seo/2026/06/05/nodejs-stream-compose-reusable-pipeline-guide.html)
- [Node.js CompressionStream 가이드: Web Streams로 gzip과 brotli 압축 다루기](/development/blog/seo/2026/08/24/nodejs-compressionstream-decompressionstream-web-streams-guide.html)

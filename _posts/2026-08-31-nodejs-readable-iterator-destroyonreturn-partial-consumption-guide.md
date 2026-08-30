---
layout: post
title: "Node.js readable.iterator 가이드: 스트림을 중간까지만 읽고 안전하게 이어서 처리하는 법"
date: 2026-08-31 08:00:00 +0900
lang: ko
translation_key: nodejs-readable-iterator-destroyonreturn-partial-consumption-guide
permalink: /development/blog/seo/2026/08/31/nodejs-readable-iterator-destroyonreturn-partial-consumption-guide.html
alternates:
  ko: /development/blog/seo/2026/08/31/nodejs-readable-iterator-destroyonreturn-partial-consumption-guide.html
  x_default: /development/blog/seo/2026/08/31/nodejs-readable-iterator-destroyonreturn-partial-consumption-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, stream, readable, iterator, async-iterator, destroyonreturn, backpressure, log-processing]
description: "Node.js readable.iterator({ destroyOnReturn: false })로 Readable 스트림을 일부만 읽고도 스트림을 닫지 않는 방법을 정리합니다. Symbol.asyncIterator와의 차이, 로그 헤더 미리보기, 에러 처리, 민감정보 점검 기준까지 실무 예제로 설명합니다."
---

Node.js의 `Readable` 스트림은 `for await...of`로 읽을 수 있습니다.
파일, HTTP 응답, 압축 해제 결과처럼 순차적으로 들어오는 데이터를 다룰 때 이 방식은 코드가 짧고 자연스럽습니다.
하지만 스트림을 **조금만 읽고 나중에 이어서 처리해야 하는 경우**에는 기본 async iterator 동작을 정확히 이해해야 합니다.

공식 Node.js 문서 기준으로 `readable[Symbol.asyncIterator]()`를 사용한 반복문이 `break`, `return`, `throw`로 중단되면 스트림은 destroy됩니다.
반면 `readable.iterator({ destroyOnReturn: false })`는 반복문을 중간에 빠져나와도 스트림을 닫지 않도록 선택할 수 있으며, Node.js 24.0.0과 22.17.0에서 stable로 표시되었습니다.

이 글에서는 `readable.iterator()`가 필요한 상황, `destroyOnReturn`을 켜고 끄는 기준, 로그 파일 미리보기와 본문 처리 흐름을 어떻게 나누면 좋은지 정리합니다.

스트림 종료와 에러 감지는 [Node.js stream.finished 가이드](/development/blog/seo/2026/08/04/nodejs-stream-finished-completion-error-handling-guide.html), 스트림과 웹 스트림 변환은 [Node.js Readable.fromWeb/toWeb 가이드](/development/blog/seo/2026/04/30/nodejs-readable-fromweb-toweb-stream-bridge-guide.html), 전체 내용을 문자열이나 버퍼로 모으는 기준은 [Node.js stream/consumers 가이드](/development/blog/seo/2026/06/07/nodejs-stream-consumers-text-json-buffer-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js Stream 공식 문서](https://nodejs.org/api/stream.html), [readable.iterator 공식 문서](https://nodejs.org/api/stream.html#readableiteratoroptions), [Readable async iterator 공식 문서](https://nodejs.org/api/stream.html#readablesymbolasynciterator)

## readable.iterator가 필요한 이유

### 기본 async iterator는 중단 시 스트림을 닫는다

`Readable`은 기본적으로 async iterator를 제공합니다.
그래서 아래처럼 `for await...of`로 청크를 읽을 수 있습니다.

```js
import { createReadStream } from 'node:fs';

export async function readFirstChunk(path) {
  const stream = createReadStream(path, { encoding: 'utf8' });

  for await (const chunk of stream) {
    return chunk;
  }

  return '';
}
```

이 코드는 첫 청크만 필요할 때는 간단합니다.
하지만 `return`으로 반복문을 빠져나온 순간 스트림은 더 이상 이어서 읽을 수 없는 상태가 됩니다.
파일 앞부분을 살짝 확인한 뒤 같은 스트림을 다른 파서로 넘기려는 구조라면 문제가 됩니다.

이 동작은 실수를 줄이기 위한 기본값에 가깝습니다.
대부분의 소비자는 반복을 중단하면 스트림도 닫히는 편이 안전합니다.
네트워크 응답이나 파일 디스크립터를 잡은 채로 방치하는 것보다, 사용이 끝난 리소스를 빨리 정리하는 쪽이 일반적으로 맞습니다.

### destroyOnReturn false는 의도적인 부분 소비에 쓴다

`readable.iterator()`는 반복 중단 시 destroy 여부를 옵션으로 정할 수 있습니다.
`destroyOnReturn: false`를 주면 `break`나 `return`으로 빠져나와도 스트림을 유지합니다.

```js
import { Readable } from 'node:stream';

const readable = Readable.from(['header\n', 'body-1\n', 'body-2\n']);

for await (const chunk of readable.iterator({ destroyOnReturn: false })) {
  console.log('preview:', chunk);
  break;
}

console.log(readable.destroyed); // false

for await (const chunk of readable) {
  console.log('rest:', chunk);
}
```

이 옵션은 "조금 읽고 멈추는 것"이 비정상 종료가 아니라 설계된 흐름일 때만 사용해야 합니다.
예를 들어 로그 파일의 첫 줄에서 스키마 버전을 확인한 뒤, 같은 스트림을 버전별 파서에 넘기는 상황이 그렇습니다.

## 로그 파일 미리보기 예제

### 첫 줄만 읽어 처리 전략을 고른다

대용량 로그 파일을 처리할 때 전체 파일을 먼저 메모리에 올릴 필요는 없습니다.
첫 줄에 포맷 버전이나 컬럼 목록이 들어 있다면 그 줄만 확인하고 나머지는 스트림으로 계속 흘려보내면 됩니다.

```js
import { createReadStream } from 'node:fs';

export async function inspectFirstChunk(readable) {
  for await (const chunk of readable.iterator({ destroyOnReturn: false })) {
    return String(chunk);
  }

  return null;
}

export async function processLogFile(path) {
  const stream = createReadStream(path, { encoding: 'utf8' });
  const firstChunk = await inspectFirstChunk(stream);
  const header = firstChunk?.split('\n', 1)[0] ?? null;

  if (header?.startsWith('v2,')) {
    return processV2Log(stream, { header, firstChunk });
  }

  return processLegacyLog(stream, { header, firstChunk });
}
```

여기서 핵심은 미리보기 함수가 스트림의 소유권을 가져가지 않는다는 점입니다.
`inspectFirstChunk()`는 판단에 필요한 앞부분만 반환하고, 실제 소비는 `processV2Log()`나 `processLegacyLog()`가 맡습니다.

다만 이 방식은 첫 청크 전체를 이미 소비합니다.
정밀하게 "첫 바이트부터 그대로 이어 읽기"가 필요하다면 바이트 단위 파서를 두거나, 미리 읽은 청크를 후속 파서에 명시적으로 전달하는 편이 더 안전합니다.

### 헤더를 후속 처리에 명시적으로 넘긴다

부분 소비 패턴에서 가장 흔한 버그는 이미 읽은 데이터를 후속 처리에서 다시 기대하는 것입니다.
첫 줄을 읽었다면 나머지 스트림에는 첫 줄이 없을 수 있습니다.
그래서 후속 함수는 헤더를 별도 인자로 받아야 합니다.

```js
async function processV2Log(readable, { header, firstChunk }) {
  const columns = header.split(',');
  const required = ['timestamp', 'level', 'message'];

  for (const name of required) {
    if (!columns.includes(name)) {
      throw new Error(`Missing required column: ${name}`);
    }
  }

  if (firstChunk) {
    for (const line of firstChunk.split('\n').slice(1)) {
      if (!line) continue;
      const record = parseV2Line(line, columns);
      await writeRecord(record);
    }
  }

  for await (const line of readable) {
    const record = parseV2Line(line, columns);
    await writeRecord(record);
  }
}
```

이 구조는 함수 계약을 선명하게 만듭니다.
후속 파서는 헤더를 다시 읽으려 하지 않고, 남은 스트림만 순차적으로 처리합니다.

## destroyOnReturn 옵션 선택 기준

### 대부분의 업무 코드는 기본값이 더 안전하다

`destroyOnReturn: false`는 편리하지만 기본값으로 두기 좋은 옵션은 아닙니다.
반복문이 중간에 멈춘 뒤에도 파일 핸들, 소켓, 압축 스트림 같은 리소스가 계속 살아 있을 수 있기 때문입니다.

아래처럼 조건을 찾으면 끝나는 탐색 로직이라면 기본 async iterator가 더 자연스럽습니다.

```js
export async function findFirstErrorLine(readable) {
  for await (const line of readable) {
    if (line.includes('ERROR')) {
      return line;
    }
  }

  return null;
}
```

이 함수는 첫 번째 에러 줄을 찾으면 스트림을 더 읽을 필요가 없습니다.
따라서 중단과 리소스 정리가 함께 일어나는 기본 동작이 맞습니다.

### 이어 읽을 소유자가 명확할 때만 false를 쓴다

`destroyOnReturn: false`를 쓰려면 "이후에 누가 스트림을 끝까지 읽거나 닫는가"가 코드에서 보여야 합니다.
미리보기 함수만 있고 후속 소비자가 불분명하면 누수 위험이 커집니다.

```js
export async function routeStream(readable) {
  const firstChunk = await peekChunk(readable);

  try {
    if (looksLikeJson(firstChunk)) {
      return await consumeJson(readable, { firstChunk });
    }

    return await consumeText(readable, { firstChunk });
  } finally {
    if (!readable.destroyed) {
      readable.destroy();
    }
  }
}
```

`finally`에서 정리하는 방식은 특히 예외 경로에서 중요합니다.
후속 파서가 실패했을 때 스트림이 열린 채 남지 않도록, 소유권을 가진 함수가 마지막 정리를 책임져야 합니다.

## 에러 처리와 취소 신호

### AbortSignal로 부분 소비 흐름을 끊을 수 있게 한다

`readable.iterator()`는 `options`를 받으며, 스트림 소비 흐름에는 취소 신호를 함께 설계하는 편이 좋습니다.
긴 파일이나 네트워크 스트림을 다룰 때 호출자가 작업을 취소할 수 있어야 대기열이 불필요하게 쌓이지 않습니다.
`readable.iterator()` 자체는 중간 반환 시 destroy 여부를 다루는 도구로 보고, 취소는 호출부의 `AbortController`와 스트림 정리 책임으로 분리해 설계하는 편이 단순합니다.

```js
export async function previewChunks(readable, { limit = 5, signal } = {}) {
  const lines = [];

  if (signal?.aborted) {
    readable.destroy(signal.reason);
    return lines;
  }

  const abort = () => readable.destroy(signal.reason);
  signal?.addEventListener('abort', abort, { once: true });

  try {
    for await (const chunk of readable.iterator({ destroyOnReturn: false })) {
      lines.push(String(chunk));

      if (lines.length >= limit) {
        break;
      }
    }

    return lines;
  } finally {
    signal?.removeEventListener('abort', abort);
  }
}
```

취소가 발생하면 후속 소비자를 호출하지 않고 현재 작업을 정리해야 합니다.
부분 소비와 취소가 함께 있으면 성공 경로보다 실패 경로가 더 중요해집니다.

### 에러가 난 스트림은 재사용 대상으로 보지 않는다

스트림에서 에러가 발생했다면 이어 읽기를 시도하기보다 실패로 처리하는 편이 안전합니다.
압축 스트림, 네트워크 응답, 파일 읽기 스트림은 에러 이후 내부 상태를 신뢰하기 어렵습니다.

```js
export async function consumeWithPreview(readable) {
  try {
    const preview = await previewChunks(readable, { limit: 1 });
    return await consumeRemaining(readable, { preview });
  } catch (error) {
    readable.destroy(error);
    throw error;
  }
}
```

에러를 삼키고 다음 단계로 넘기면 데이터가 일부 누락된 상태로 처리될 수 있습니다.
개발 환경에서는 눈에 띄지 않다가 운영에서 통계, 정산, 검색 색인 같은 결과를 조용히 오염시키는 쪽이 더 위험합니다.

## 실무 체크리스트

### 부분 소비가 정말 필요한지 먼저 확인한다

`readable.iterator({ destroyOnReturn: false })`를 쓰기 전에 다음 질문을 먼저 확인합니다.

- 첫 부분을 읽은 뒤 같은 스트림을 반드시 이어서 읽어야 하는가?
- 이미 읽은 데이터가 후속 처리에 명시적으로 전달되는가?
- 후속 소비자가 실패했을 때 스트림을 닫는 책임자가 있는가?
- 테스트가 작은 파일, 큰 파일, 빈 파일, 중간 중단을 모두 다루는가?

대답이 흐릿하다면 스트림을 한 번만 소비하는 구조가 더 낫습니다.
미리보기 값을 별도 버퍼에 저장하고, 후속 처리에는 새 스트림을 열어 넘기는 방식도 충분히 실용적입니다.

### 민감정보 로그를 남기지 않는다

부분 소비 코드는 주로 헤더, 첫 줄, 샘플 레코드를 확인합니다.
이 값에는 이메일, 토큰, 세션 ID, 내부 URL 같은 민감정보가 들어 있을 수 있습니다.
디버깅 로그에 원본 라인을 그대로 남기지 말고 필요한 필드만 마스킹해서 기록해야 합니다.

```js
function redactPreview(value) {
  return String(value)
    .replace(/Bearer\s+[A-Za-z0-9._-]+/g, 'Bearer <redacted>')
    .replace(/[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}/gi, '<email>');
}

console.log('stream preview:', redactPreview(firstChunk));
```

특히 CI 로그와 배치 작업 로그는 오래 보관되거나 외부 시스템으로 전달될 수 있습니다.
스트림 첫 줄이라고 해서 안전한 데이터라고 가정하지 않는 것이 좋습니다.

## 마무리

`readable.iterator({ destroyOnReturn: false })`는 Node.js 스트림을 일부만 읽고도 이어서 처리해야 할 때 유용한 도구입니다.
하지만 이 옵션의 가치는 "스트림을 닫지 않는 것" 자체가 아니라, 부분 소비와 후속 소비의 책임을 코드에 명확히 드러내는 데 있습니다.

기본 `for await...of`는 중단 시 리소스를 정리하는 안전한 선택입니다.
정말로 이어 읽기가 필요한 흐름에서만 `destroyOnReturn: false`를 사용하고, 이미 읽은 데이터 전달, 실패 시 정리, 민감정보 마스킹까지 함께 설계하면 대용량 로그와 네트워크 스트림 처리 코드를 더 안정적으로 만들 수 있습니다.

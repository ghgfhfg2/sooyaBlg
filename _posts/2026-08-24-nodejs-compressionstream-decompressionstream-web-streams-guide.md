---
layout: post
title: "Node.js CompressionStream 가이드: Web Streams로 gzip·brotli 압축을 다루는 법"
date: 2026-08-24 20:00:00 +0900
lang: ko
translation_key: nodejs-compressionstream-decompressionstream-web-streams-guide
permalink: /development/blog/seo/2026/08/24/nodejs-compressionstream-decompressionstream-web-streams-guide.html
alternates:
  ko: /development/blog/seo/2026/08/24/nodejs-compressionstream-decompressionstream-web-streams-guide.html
  x_default: /development/blog/seo/2026/08/24/nodejs-compressionstream-decompressionstream-web-streams-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, compressionstream, decompressionstream, web-streams, gzip, brotli, backend, javascript]
description: "Node.js CompressionStream과 DecompressionStream으로 Web Streams 기반 gzip·brotli 압축 파이프라인을 만드는 방법을 정리합니다. node:zlib과의 차이, fetch 응답 처리, 메모리 관리, 운영 체크리스트까지 실무 예제로 설명합니다."
---

Node.js에서 압축을 다룬다고 하면 보통 `node:zlib`의 `createGzip()`, `createGunzip()`, `createBrotliCompress()`를 먼저 떠올립니다.
이 방식은 여전히 강력하고, 파일 스트림이나 HTTP 응답 압축처럼 Node.js 전통 스트림을 쓰는 코드에서는 가장 자연스럽습니다.
하지만 `fetch()`, `Response`, `ReadableStream`, 브라우저와 공유하는 유틸리티 코드가 늘어나면 Web Streams 기반 압축 API도 선택지가 됩니다.

`CompressionStream`과 `DecompressionStream`은 gzip, deflate, brotli 같은 압축 포맷을 Web Streams 파이프라인 안에서 다룰 수 있게 해주는 전역 클래스입니다.
Node.js 공식 문서 기준으로 이 API는 Node.js 18부터 제공되며, Node.js 22.15.0과 23.11.0에서 안정화되었고, brotli 포맷은 Node.js 22.20.0과 24.7.0부터 지원됩니다.
따라서 "모든 Node.js 18 이상에서 brotli까지 된다"가 아니라, 런타임 버전과 대상 포맷을 함께 확인해야 합니다.

이 글에서는 Node.js에서 `CompressionStream`과 `DecompressionStream`을 실무적으로 쓰는 기준을 정리합니다.
Node.js 전통 스트림과 Web Streams를 연결하는 방법은 [Node.js Readable.fromWeb/toWeb 스트림 브리지 가이드](/development/blog/seo/2026/04/30/nodejs-readable-fromweb-toweb-stream-bridge-guide.html), 압축 응답의 HTTP 협상은 [Node.js 응답 압축 실전 가이드](/development/blog/seo/2026/03/20/nodejs-response-compression-brotli-gzip-performance-guide.html), 스트림 완료와 에러 처리는 [Node.js stream.finished 완료 감지 가이드](/development/blog/seo/2026/08/04/nodejs-stream-finished-completion-error-handling-guide.html)와 함께 보면 좋습니다.

## CompressionStream을 검토할 때

### Web Streams 중심 코드에서 가장 자연스럽다

`CompressionStream`의 장점은 압축 자체보다 연결 방식에 있습니다.
입력과 출력이 모두 Web Streams이므로 `ReadableStream.prototype.pipeThrough()`와 잘 맞고, `fetch()` 응답 본문이나 `Response` 객체를 조합하는 코드에서 별도 변환 계층을 줄일 수 있습니다.

예를 들어 서버 내부에서 JSON 데이터를 gzip으로 압축한 `Response`를 만들고 싶다면 아래처럼 쓸 수 있습니다.

```js
function jsonStream(value) {
  return new Blob([JSON.stringify(value)]).stream();
}

export function createCompressedJsonResponse(value) {
  const compressedBody = jsonStream(value)
    .pipeThrough(new CompressionStream('gzip'));

  return new Response(compressedBody, {
    headers: {
      'content-type': 'application/json; charset=utf-8',
      'content-encoding': 'gzip',
      'vary': 'accept-encoding'
    }
  });
}
```

이 코드는 Node.js 전용 `node:zlib` 스트림을 직접 만들지 않습니다.
대신 Web Platform API에 가까운 형태로 압축 스트림을 조합합니다.
서비스가 이미 `fetch()`, `Request`, `Response`, `ReadableStream` 기반으로 작성되어 있다면 읽기 쉽고 테스트하기도 편합니다.

반대로 Express, Fastify의 Node.js 스트림 응답, 대용량 파일 파이프라인, 세밀한 zlib 옵션 튜닝이 중심이라면 `node:zlib`이 더 적합합니다.
도구를 바꾸는 기준은 "새 API가 더 최신인가"가 아니라 "현재 파이프라인의 스트림 모델과 맞는가"입니다.

### 지원 포맷을 런타임 기준으로 확인한다

Compression Streams 표준은 gzip, deflate, deflate-raw를 중심으로 다룹니다.
Node.js는 버전에 따라 brotli도 받을 수 있지만, 모든 배포 환경에서 같은 범위를 기대하면 안 됩니다.
특히 LTS 버전, 컨테이너 이미지, 서버리스 런타임이 섞여 있는 팀에서는 실제 런타임이 문서보다 늦게 따라오는 경우가 흔합니다.

운영 코드에서는 시작 시점에 지원 여부를 확인하는 편이 좋습니다.

```js
export function assertCompressionFormat(format) {
  try {
    new CompressionStream(format);
    new DecompressionStream(format);
  } catch (error) {
    throw new Error(`Compression format is not supported in this runtime: ${format}`, {
      cause: error
    });
  }
}

assertCompressionFormat('gzip');
```

라이브러리라면 이 검사를 모듈 로딩 시점에 바로 실행하기보다, 실제 기능을 켤 때 실행하는 편이 낫습니다.
그렇지 않으면 압축 기능을 쓰지 않는 사용자까지 특정 Node.js 버전에 묶이게 됩니다.

### node:zlib을 대체하는 만능 API는 아니다

`CompressionStream`은 간결하지만, `node:zlib`이 제공하는 모든 튜닝 표면을 그대로 노출하지는 않습니다.
압축 레벨, 메모리 레벨, dictionary, flush 동작, brotli 세부 파라미터를 직접 조절해야 한다면 `node:zlib` 쪽이 더 적합합니다.

실무적으로는 이렇게 나누면 됩니다.

- `CompressionStream`: Web Streams 기반 파이프라인, fetch 응답 조합, 브라우저와 공유 가능한 유틸리티
- `node:zlib`: Node.js 스트림 파일 처리, HTTP 서버 미들웨어, 압축 레벨 튜닝, 운영 성능 최적화
- `compression` 미들웨어: Express 계열 애플리케이션에서 빠르게 응답 압축을 붙이는 경우

압축은 보통 CPU, 메모리, 네트워크 비용을 서로 교환하는 작업입니다.
API가 짧아졌다고 해서 운영 비용이 사라지는 것은 아니므로, 큰 트래픽 구간에서는 반드시 측정해야 합니다.

## gzip 압축 파이프라인 만들기

### 문자열을 gzip Uint8Array로 변환한다

작은 문자열을 gzip으로 압축해 바이트 배열로 얻고 싶다면 `TextEncoderStream`, `CompressionStream`, `Response.arrayBuffer()`를 조합할 수 있습니다.

```js
export async function gzipText(text) {
  const source = new Blob([text]).stream();
  const compressed = source.pipeThrough(new CompressionStream('gzip'));
  const buffer = await new Response(compressed).arrayBuffer();

  return new Uint8Array(buffer);
}

const payload = await gzipText(JSON.stringify({ ok: true }));
console.log(payload.byteLength);
```

여기서 `Response`는 네트워크 요청을 보내기 위한 객체가 아니라, Web Streams를 편하게 수집하기 위한 컨테이너로 쓰였습니다.
테스트나 CLI 도구에서는 이런 형태가 꽤 실용적입니다.

다만 이 방식은 결과 전체를 메모리에 모읍니다.
작은 JSON, 캐시 키 재료, 테스트 fixture 압축에는 괜찮지만, 수백 MB 파일을 처리할 때는 파일 스트림과 backpressure를 끝까지 유지하는 구조를 써야 합니다.

### 다시 원문으로 복원한다

압축된 바이트를 원문 문자열로 되돌릴 때는 `DecompressionStream`을 씁니다.

```js
export async function gunzipText(bytes) {
  const source = new Blob([bytes]).stream();
  const decompressed = source.pipeThrough(new DecompressionStream('gzip'));

  return await new Response(decompressed).text();
}

const compressed = await gzipText('hello compression stream');
const restored = await gunzipText(compressed);

console.log(restored);
```

압축 해제는 입력을 신뢰할 수 없다는 전제에서 다뤄야 합니다.
외부에서 받은 압축 데이터는 작은 입력처럼 보여도 해제 후 크기가 매우 커질 수 있습니다.
사용자 업로드, 외부 webhook, 메시지 큐 payload를 풀 때는 최대 크기, 시간 제한, 실패 로그 정책을 함께 둬야 합니다.

### 파일 처리에는 Node.js 스트림 브리지를 쓴다

파일을 Web Streams 압축 API로 처리하고 싶다면 `Readable.toWeb()`과 `Writable.fromWeb()` 같은 브리지 API를 사용할 수 있습니다.
다만 이 경우에도 Node.js 파일 스트림과 Web Streams의 에러 전파를 함께 이해해야 합니다.

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { Readable, Writable } from 'node:stream';

export async function gzipFile(inputPath, outputPath) {
  const source = Readable.toWeb(createReadStream(inputPath));
  const destination = Writable.toWeb(createWriteStream(outputPath));

  await source
    .pipeThrough(new CompressionStream('gzip'))
    .pipeTo(destination);
}
```

이 패턴은 브라우저와 공유하는 스트림 처리 함수를 Node.js 파일에도 적용하고 싶을 때 유용합니다.
하지만 순수 Node.js 애플리케이션에서 파일 압축만 한다면 `node:zlib`과 `stream/promises`의 `pipeline()`이 더 익숙하고 진단하기 쉽습니다.

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';

export async function gzipFileWithZlib(inputPath, outputPath) {
  await pipeline(
    createReadStream(inputPath),
    createGzip(),
    createWriteStream(outputPath)
  );
}
```

팀 코드베이스가 이미 `pipeline()` 중심이라면 억지로 Web Streams로 바꿀 이유는 없습니다.
반대로 fetch, Request, Response 중심의 런타임을 함께 다룬다면 Web Streams 쪽으로 맞추는 편이 전체 코드가 단순해질 수 있습니다.

## fetch 응답과 함께 쓰기

### 압축된 응답 본문을 명시적으로 해제한다

Node.js의 `fetch()`는 일반적인 HTTP 응답 처리에서 압축 해제를 자동으로 다루는 경우가 많습니다.
하지만 파일, 오브젝트 스토리지, 메시지 큐, 테스트 fixture처럼 "이미 gzip으로 저장된 바이트"를 읽는 상황에서는 직접 `DecompressionStream`을 적용해야 합니다.

```js
export async function readGzipJson(url) {
  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`);
  }

  if (!response.body) {
    throw new Error('Response body is empty');
  }

  const text = await new Response(
    response.body.pipeThrough(new DecompressionStream('gzip'))
  ).text();

  return JSON.parse(text);
}
```

이 함수는 "HTTP가 자동으로 풀어주는 압축 응답"이 아니라 "gzip 파일을 바이트로 내려받아 직접 푸는 경우"에 맞습니다.
실제 HTTP 응답의 `Content-Encoding`과 저장 포맷의 압축 여부를 혼동하면 이중 압축 해제나 깨진 데이터가 생길 수 있습니다.

### Content-Encoding과 저장 포맷을 분리한다

압축에는 두 가지 층이 있습니다.

- 전송 압축: HTTP `Content-Encoding: gzip`처럼 네트워크 전송 중에 적용되는 압축
- 저장 압축: `.json.gz`, `.log.br`처럼 저장된 객체 자체가 압축된 포맷

전송 압축은 클라이언트와 서버가 헤더로 협상합니다.
저장 압축은 파일 이름, 메타데이터, 콘텐츠 타입, 애플리케이션 계약으로 판단합니다.

예를 들어 오브젝트 스토리지에 `events-2026-08-24.json.gz` 파일을 저장했다면, HTTP 응답이 자동으로 gzip 해제되는지와 별개로 애플리케이션은 그 파일이 gzip 포맷이라는 사실을 알아야 합니다.
이 구분이 흐려지면 로컬 테스트에서는 통과하지만 CDN, 프록시, SDK 버전 차이에서만 실패하는 문제가 생깁니다.

## brotli를 쓸 때의 기준

### 런타임 지원 버전을 먼저 고정한다

Node.js의 `CompressionStream('brotli')` 지원은 gzip보다 늦게 들어왔습니다.
따라서 brotli를 쓰려면 `package.json`의 `engines.node`, Docker 베이스 이미지, CI matrix, 서버리스 런타임 문서를 함께 맞춰야 합니다.

```js
export async function brotliText(text) {
  const stream = new Blob([text])
    .stream()
    .pipeThrough(new CompressionStream('brotli'));

  return new Uint8Array(await new Response(stream).arrayBuffer());
}
```

이 코드 자체는 짧지만, 운영 계약은 짧지 않습니다.
배포 대상 중 하나라도 brotli 포맷을 받지 못하면 시작 시점이나 첫 요청 시점에 실패합니다.
특히 여러 서비스가 같은 유틸리티 패키지를 공유한다면 gzip 기본값을 유지하고, brotli는 명시 옵션으로 켜는 구조가 더 안전합니다.

### 정적 자산과 동적 payload를 다르게 본다

Brotli는 텍스트 자산에서 좋은 압축률을 기대할 수 있지만, 실시간 동적 payload에 항상 최선은 아닙니다.
정적 HTML, CSS, JS, 문서 파일은 빌드 단계에서 미리 압축해 둘 수 있습니다.
반면 요청마다 달라지는 JSON 응답은 압축 비용이 요청 경로의 지연 시간에 직접 들어옵니다.

따라서 brotli를 적용할 때는 아래처럼 분리해 판단하세요.

- 빌드 산출물: 미리 압축하고 CDN 캐시에 올린다
- 반복 조회되는 API 응답: 압축 결과 캐시를 검토한다
- 사용자별 동적 응답: 응답 크기와 CPU 사용량을 함께 측정한다
- 작은 응답: 압축 헤더와 CPU 비용이 이득보다 클 수 있다

압축률만 보면 brotli가 매력적이지만, 실무에서는 p95 지연 시간, CPU 사용률, CDN 캐시 적중률, 모바일 네트워크 효과를 같이 봐야 합니다.

## 메모리와 오류 처리

### arrayBuffer()는 전체 결과를 메모리에 모은다

예제에서 `new Response(stream).arrayBuffer()`를 많이 썼지만, 이 방식은 결과 전체를 메모리에 적재합니다.
작은 payload를 다루는 샘플로는 좋지만, 대용량 파일 처리 기본 패턴으로 쓰면 위험합니다.

대용량 처리에서는 아래 기준을 지키는 편이 좋습니다.

- 입력과 출력을 스트림으로 끝까지 연결한다
- 중간에 전체 문자열이나 전체 Buffer로 만들지 않는다
- 최대 입력 크기와 최대 해제 후 크기를 제한한다
- 실패 시 부분 파일을 남길지 삭제할지 정책을 정한다
- 압축률, 처리 시간, 실패 건수를 지표로 남긴다

특히 압축 해제는 "작은 입력이 큰 출력으로 바뀔 수 있다"는 점 때문에 방어가 필요합니다.
외부 입력을 무제한으로 `text()`나 `arrayBuffer()`에 모으는 코드는 서비스 메모리 사용량을 급격히 키울 수 있습니다.

### pipeTo 실패를 호출자가 볼 수 있게 한다

Web Streams의 `pipeTo()`는 Promise를 반환합니다.
이 Promise를 `await`하지 않으면 압축 중 발생한 오류가 호출 흐름 밖으로 밀려나고, 테스트나 작업 큐에서 성공처럼 보일 수 있습니다.

```js
export async function compressToWritable(source, writable) {
  await source
    .pipeThrough(new CompressionStream('gzip'))
    .pipeTo(writable);
}
```

작업 큐나 배치 코드에서는 실패를 잡아 상태를 남기고 재시도 여부를 결정해야 합니다.

```js
export async function runCompressionJob(job) {
  try {
    await compressToWritable(job.source, job.destination);
    await job.markSucceeded();
  } catch (error) {
    await job.markFailed({
      reason: error instanceof Error ? error.message : 'unknown compression error'
    });
    throw error;
  }
}
```

여기서 로그에 원본 payload를 그대로 남기면 안 됩니다.
압축 대상에는 사용자 데이터, 토큰, 이메일, 내부 식별자가 섞일 수 있습니다.
운영 로그에는 파일 크기, 포맷, 작업 ID, 에러 타입처럼 진단에 필요한 최소 정보만 남기는 편이 안전합니다.

### AbortSignal과 시간 제한을 함께 설계한다

긴 압축 작업은 취소 가능해야 합니다.
Web Streams 파이프라인은 `pipeTo()` 옵션으로 `signal`을 받을 수 있습니다.

```js
export async function gzipWithTimeout(source, writable, timeoutMs) {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);

  try {
    await source
      .pipeThrough(new CompressionStream('gzip'))
      .pipeTo(writable, { signal: controller.signal });
  } finally {
    clearTimeout(timeout);
  }
}
```

시간 제한은 단순히 무한 대기를 막는 장치가 아닙니다.
배치 작업의 전체 처리량, 요청 타임아웃, 사용자 취소, 서버 종료 시 drain 정책과 연결됩니다.
압축 작업이 요청 처리 경로에 있다면 HTTP 요청 타임아웃보다 짧게 잡고, 백그라운드 작업이라면 재시도 횟수와 함께 설계하세요.

## 테스트 전략

### round trip 테스트를 기본으로 둔다

압축 코드는 결과 바이트가 매번 완전히 같다고 가정하기보다, 압축 후 해제했을 때 원문이 유지되는지를 먼저 검증하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('gzipText round trip', async () => {
  const input = JSON.stringify({
    id: 'demo',
    items: ['alpha', 'beta', 'gamma']
  });

  const compressed = await gzipText(input);
  const restored = await gunzipText(compressed);

  assert.equal(restored, input);
});
```

압축 포맷의 헤더나 런타임 구현 차이 때문에 바이트 단위 snapshot은 쉽게 깨질 수 있습니다.
정확한 바이너리 호환성을 검증해야 하는 라이브러리가 아니라면 round trip, 크기 범위, 에러 처리 테스트가 더 오래 유지됩니다.

### 잘못된 입력을 반드시 넣는다

압축 해제 함수에는 정상 gzip만 들어오지 않습니다.
빈 바이트, 일반 텍스트, 잘린 gzip, 다른 포맷의 압축 데이터도 테스트해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('gunzipText rejects plain text bytes', async () => {
  const bytes = new TextEncoder().encode('not a gzip payload');

  await assert.rejects(
    () => gunzipText(bytes),
    /decompression|incorrect|invalid|unexpected/i
  );
});
```

에러 메시지는 Node.js 버전이나 내부 구현에 따라 달라질 수 있으므로 너무 좁게 묶지 않는 것이 좋습니다.
중요한 것은 실패를 성공으로 삼키지 않고 호출자에게 전달하는지입니다.

## 운영 체크리스트

### 적용 전에 확인할 것

`CompressionStream`을 도입하기 전에 아래 항목을 확인하세요.

- 서비스가 Web Streams 기반 파이프라인을 이미 쓰고 있는가
- 대상 Node.js 버전에서 필요한 포맷이 지원되는가
- gzip, deflate, brotli 중 어떤 포맷을 기본값으로 둘 것인가
- 대용량 payload를 전체 메모리에 모으는 코드가 없는가
- 압축 실패가 호출자, 작업 상태, 로그에 드러나는가
- 외부 입력 압축 해제에 크기 제한과 시간 제한이 있는가
- HTTP 전송 압축과 저장 포맷 압축을 구분하고 있는가

작은 유틸리티라면 gzip round trip 테스트와 런타임 지원 검사만으로도 충분할 수 있습니다.
트래픽이 큰 API 경로라면 CPU 사용량, 응답 크기, p95 지연 시간, 캐시 적중률을 배포 전후로 비교해야 합니다.

### 도입하지 않는 편이 나은 경우

아래 상황에서는 `CompressionStream`보다 다른 선택이 낫습니다.

- 압축 레벨과 brotli 파라미터를 세밀하게 조절해야 한다
- 기존 코드가 Node.js `pipeline()`과 `node:zlib` 중심으로 안정화되어 있다
- Express 미들웨어 수준의 응답 압축만 필요하다
- 배포 런타임이 오래되어 Web Streams 압축 API 지원이 불확실하다
- 대용량 파일 처리에서 운영자가 Node.js 스트림 도구로 진단하는 데 익숙하다

새 API를 쓰는 목적은 코드를 짧게 만드는 것이 아니라 경계를 줄이는 것입니다.
이미 Web Streams 경계 안에 있다면 `CompressionStream`은 자연스럽고, Node.js 스트림 경계 안에 있다면 `node:zlib`이 자연스럽습니다.

## 마무리

`CompressionStream`과 `DecompressionStream`은 Node.js에서 압축을 다루는 또 하나의 표준적인 방법입니다.
특히 `fetch()`, `Response`, `ReadableStream`과 함께 쓰는 코드에서는 Node.js 전용 스트림으로 내려갔다가 다시 올라오는 변환을 줄일 수 있습니다.
하지만 포맷 지원 범위, 런타임 버전, 메모리 사용량, 압축 실패 처리까지 함께 봐야 운영에서 안정적으로 쓸 수 있습니다.

실무 기준은 단순합니다.
Web Streams 파이프라인에는 `CompressionStream`을 검토하고, Node.js 스트림 파이프라인과 세밀한 튜닝에는 `node:zlib`을 유지하세요.
그리고 어떤 방식을 쓰든 압축은 반드시 측정 대상입니다.
전송 바이트가 줄어든 만큼 CPU와 지연 시간이 어떻게 바뀌었는지 확인해야 좋은 최적화가 됩니다.

## 참고 자료

- [Node.js Global objects: CompressionStream, DecompressionStream](https://nodejs.org/api/globals.html#class-compressionstream)
- [Node.js Web Streams API](https://nodejs.org/api/webstreams.html)
- [Node.js Zlib 문서](https://nodejs.org/api/zlib.html)
- [WHATWG Compression Standard](https://compression.spec.whatwg.org/)
- [MDN Compression Streams API](https://developer.mozilla.org/en-US/docs/Web/API/Compression_Streams_API)


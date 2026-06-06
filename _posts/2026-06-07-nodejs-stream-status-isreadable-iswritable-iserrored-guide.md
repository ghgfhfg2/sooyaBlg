---
layout: post
title: "Node.js 스트림 상태 확인 가이드: isReadable, isWritable, isErrored 활용법"
date: 2026-06-07 08:00:00 +0900
lang: ko
translation_key: nodejs-stream-status-isreadable-iswritable-iserrored-guide
permalink: /development/blog/seo/2026/06/07/nodejs-stream-status-isreadable-iswritable-iserrored-guide.html
alternates:
  ko: /development/blog/seo/2026/06/07/nodejs-stream-status-isreadable-iswritable-iserrored-guide.html
  x_default: /development/blog/seo/2026/06/07/nodejs-stream-status-isreadable-iswritable-iserrored-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, stream, isreadable, iswritable, iserrored, diagnostics, backend]
description: "Node.js stream.isReadable, stream.isWritable, stream.isErrored로 스트림의 현재 상태를 점검하는 방법을 정리했습니다. 상태 플래그 직접 접근과의 차이, 운영 진단, 테스트, 종료 처리 기준을 예제로 설명합니다."
---

Node.js 스트림을 운영하다 보면 "지금 이 스트림에 더 쓸 수 있는가?", "읽기가 끝난 것인가?", "이미 에러가 난 상태인가?"를 확인해야 할 때가 있습니다.
업로드 처리, 파일 변환, 프록시 응답, 로그 수집처럼 스트림 수명이 긴 코드에서는 종료 직전의 상태를 진단하거나 테스트 결과를 검증하는 일이 특히 중요합니다.

스트림 객체에는 여러 상태 속성이 있지만, 내부 구현과 스트림 종류에 따라 직접 해석하기 까다로울 수 있습니다.
Node.js의 `node:stream` 모듈은 `isReadable()`, `isWritable()`, `isErrored()` 같은 상태 확인 함수를 제공합니다.
이 함수들은 스트림 객체를 받아 현재 읽기 가능 여부, 쓰기 가능 여부, 에러 발생 여부를 일관된 형태로 확인하게 해 줍니다.

이 글에서는 세 상태 함수의 의미, 안전하게 사용하는 패턴, 상태 확인만으로 흐름 제어를 대신하면 안 되는 이유, 테스트와 운영 진단 체크리스트를 정리합니다.
스트림 연결과 에러 전파가 먼저 필요하다면 [Node.js stream.compose 가이드](/development/blog/seo/2026/06/05/nodejs-stream-compose-reusable-pipeline-guide.html)를 함께 참고하세요.

## 스트림 상태 확인 함수가 필요한 이유

### H3. 상태 속성을 직접 조합하는 코드는 의도가 흐려진다

스트림이 끝났는지 확인하려고 여러 속성을 직접 읽기 시작하면 조건문이 빠르게 복잡해집니다.

```js
function canStillWrite(stream) {
  return !stream.destroyed &&
    !stream.writableEnded &&
    !stream.writableFinished;
}
```

이 코드는 언뜻 그럴듯하지만 "쓰기 가능한 상태"를 직접 정의하고 유지해야 합니다.
Duplex, Transform, 사용자 정의 스트림처럼 읽기와 쓰기 수명이 서로 다른 객체에서는 판단 기준이 더 복잡해질 수 있습니다.

`isReadable()`과 `isWritable()`을 사용하면 확인하려는 의도를 함수 이름으로 드러낼 수 있습니다.

```js
import { isReadable, isWritable } from 'node:stream';

console.log({
  readable: isReadable(stream),
  writable: isWritable(stream)
});
```

이 함수들은 상태를 변경하지 않습니다.
현재 상태를 관찰하는 진단 도구에 가깝기 때문에 로그, 테스트 assertion, 종료 과정 점검에 잘 맞습니다.

### H3. 읽기 상태와 쓰기 상태를 따로 봐야 한다

Duplex와 Transform 스트림은 읽기 측과 쓰기 측을 모두 가집니다.
한쪽이 끝났다고 다른 쪽도 항상 같은 시점에 끝나는 것은 아닙니다.

예를 들어 Transform 스트림은 입력 쓰기가 끝난 뒤에도 내부에 남은 결과를 읽기 측으로 내보낼 수 있습니다.
따라서 `isReadable()`과 `isWritable()`을 각각 확인하면 어느 방향의 수명이 끝났는지 더 명확하게 진단할 수 있습니다.

## 기본 사용법

### H3. PassThrough로 상태 변화를 확인한다

`PassThrough`는 입력 데이터를 그대로 출력하는 Transform 스트림입니다.
상태 함수의 변화를 확인하는 작은 예제에 적합합니다.

```js
import {
  PassThrough,
  isErrored,
  isReadable,
  isWritable
} from 'node:stream';

const stream = new PassThrough();

console.log({
  readable: isReadable(stream),
  writable: isWritable(stream),
  errored: isErrored(stream)
});

stream.end('done');

console.log({
  readable: isReadable(stream),
  writable: isWritable(stream),
  errored: isErrored(stream)
});
```

`end()`를 호출하면 쓰기 측 종료가 시작됩니다.
하지만 읽기 측에는 아직 소비되지 않은 데이터가 남아 있을 수 있으므로 두 상태가 동시에 같은 값으로 바뀐다고 가정하면 안 됩니다.

상태를 정확히 비교하려면 특정 메서드 호출 직후뿐 아니라 `'finish'`, `'end'`, `'close'` 같은 수명 주기 이벤트 이후도 관찰해야 합니다.

### H3. isErrored로 에러 상태를 구분한다

`isErrored()`는 스트림이 에러를 경험한 상태인지 확인할 때 사용합니다.
운영 로그나 테스트 실패 메시지에 상태를 붙이면 정상 종료와 에러 종료를 구분하기 쉬워집니다.

```js
import { PassThrough, isErrored } from 'node:stream';

const stream = new PassThrough();

stream.on('error', (error) => {
  console.error({
    message: error.message,
    errored: isErrored(stream)
  });
});

stream.destroy(new Error('conversion failed'));
```

중요한 점은 `isErrored()`가 에러 이벤트 처리기를 대신하지 않는다는 것입니다.
스트림의 `'error'` 이벤트를 처리하지 않으면 프로세스에 처리되지 않은 에러가 발생할 수 있습니다.
`isErrored()`는 에러를 처리하는 함수가 아니라 이미 발생한 상태를 확인하는 함수입니다.

## 실무에서 안전하게 사용하는 패턴

### H3. 상태 확인은 진단과 검증에 우선 사용한다

상태 확인 함수는 운영 진단 정보를 만드는 데 유용합니다.
예를 들어 파이프라인이 예상보다 오래 걸릴 때 각 스트림의 상태를 구조화된 로그로 남길 수 있습니다.

```js
import { isErrored, isReadable, isWritable } from 'node:stream';

export function describeStream(stream, name) {
  return {
    name,
    readable: isReadable(stream),
    writable: isWritable(stream),
    errored: isErrored(stream),
    destroyed: Boolean(stream?.destroyed)
  };
}
```

```js
console.info({
  event: 'pipeline-timeout',
  streams: [
    describeStream(source, 'source'),
    describeStream(transform, 'transform'),
    describeStream(destination, 'destination')
  ]
});
```

이런 정보는 어느 단계가 아직 열려 있는지 추정하는 데 도움을 줍니다.
다만 로그에 파일 경로, 사용자 입력, 인증 정보 같은 민감정보를 무심코 포함하지 않도록 상태 필드만 선별해야 합니다.

### H3. 확인 후 쓰는 패턴에는 경쟁 조건이 있다

아래처럼 쓰기 가능 여부를 확인한 뒤 `write()`를 호출하고 싶을 수 있습니다.

```js
if (isWritable(destination)) {
  destination.write(chunk);
}
```

하지만 상태를 확인한 직후 다른 비동기 흐름에서 스트림이 종료되거나 파괴될 수 있습니다.
즉 "확인 시점에는 쓰기 가능했다"는 사실이 다음 동작의 성공을 보장하지 않습니다.

실제 쓰기 흐름에서는 다음 원칙을 우선해야 합니다.

- `write()` 반환값과 `'drain'` 이벤트로 backpressure를 처리한다.
- `'error'`, `'finish'`, `'close'` 이벤트를 적절히 처리한다.
- 여러 스트림을 연결할 때는 `pipeline()`으로 에러와 종료를 함께 관리한다.
- `AbortSignal`이나 명시적인 종료 함수를 이용해 취소 경로를 한곳에 모은다.

메모리 사용량까지 안정적으로 관리하려면 [Node.js 스트림 backpressure 가이드](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)를 함께 확인하는 것이 좋습니다.

### H3. pipeline 종료 후 상태를 진단한다

`pipeline()`은 연결된 스트림의 에러 전파와 정리를 관리하는 기본 도구입니다.
상태 함수는 `pipeline()`을 대체하기보다, 성공 또는 실패 이후의 상태를 기록하는 보조 수단으로 사용하는 편이 안전합니다.

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { isErrored, isReadable, isWritable } from 'node:stream';
import { pipeline } from 'node:stream/promises';

const source = createReadStream('input.log');
const destination = createWriteStream('output.log');

try {
  await pipeline(source, destination);
} finally {
  console.info({
    sourceReadable: isReadable(source),
    sourceErrored: isErrored(source),
    destinationWritable: isWritable(destination),
    destinationErrored: isErrored(destination)
  });
}
```

이 로그는 파이프라인이 어떤 상태로 정리됐는지 확인하는 데 유용합니다.
실제 서비스에서는 에러 객체 전체나 입력 파일명을 그대로 기록하기보다 필요한 진단 필드만 남겨야 합니다.

## 테스트에서 상태 계약 검증하기

### H3. 정상 종료와 에러 종료를 나눠 테스트한다

상태 함수는 스트림 수명 주기 테스트의 실패 메시지를 명확하게 만들어 줍니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { PassThrough, isErrored, isWritable } from 'node:stream';
import { finished } from 'node:stream/promises';

test('stream finishes without an error', async () => {
  const stream = new PassThrough();
  stream.resume();
  stream.end('ok');

  await finished(stream);

  assert.equal(isWritable(stream), false);
  assert.equal(isErrored(stream), false);
});
```

에러 경로는 별도 테스트로 분리하는 편이 좋습니다.
에러 발생 여부뿐 아니라 호출자가 실패를 실제로 전달받는지도 함께 확인해야 합니다.

```js
test('destroyed stream reports an error', async () => {
  const stream = new PassThrough();
  const completion = finished(stream);

  stream.destroy(new Error('expected failure'));

  await assert.rejects(completion, /expected failure/);
  assert.equal(isErrored(stream), true);
});
```

단순히 `isErrored(stream) === true`만 확인하면 에러가 소비자에게 올바르게 전파됐는지 놓칠 수 있습니다.
상태와 외부 계약을 함께 검증하는 것이 중요합니다.

### H3. 구현 세부사항보다 관찰 가능한 결과를 검증한다

상태 함수가 편리하더라도 모든 내부 단계의 값을 테스트하면 리팩터링이 어려워집니다.
테스트에서는 다음처럼 사용자에게 영향을 주는 계약에 집중하는 편이 좋습니다.

- 정상 입력이 끝나면 출력이 완성되고 쓰기 측이 종료된다.
- 변환 실패는 호출자에게 reject되고 에러 상태로 남는다.
- 취소 시 연결된 스트림이 정리되고 작업이 더 진행되지 않는다.
- 대용량 입력에서도 backpressure가 유지된다.

브라우저 Web Stream과 Node.js 스트림을 연결하는 코드라면 [Node.js Readable.fromWeb/toWeb 가이드](/development/blog/seo/2026/04/30/nodejs-readable-fromweb-toweb-stream-bridge-guide.html)의 경계 처리도 함께 테스트해야 합니다.

## 자주 묻는 질문

### H3. isReadable이 false면 destroy를 호출해야 하나?

항상 그렇지는 않습니다.
`isReadable()`이 `false`라는 것은 현재 읽기 가능한 상태가 아니라는 뜻이며, 정상적으로 읽기가 끝난 경우도 포함할 수 있습니다.
정리 여부는 스트림의 수명 주기, `destroyed` 상태, 연결 방식, 에러 발생 여부를 함께 보고 결정해야 합니다.

### H3. isWritable이 true면 write가 반드시 성공하나?

아닙니다.
상태 확인과 실제 쓰기 사이에 스트림 상태가 바뀔 수 있고, `write()`가 `false`를 반환해 backpressure를 알릴 수도 있습니다.
쓰기 흐름은 `write()` 반환값, `'drain'`, `'error'`, 종료 이벤트를 기준으로 설계해야 합니다.

### H3. isErrored만 확인하면 error 이벤트 처리가 필요 없나?

아닙니다.
`isErrored()`는 상태를 관찰할 뿐 에러 이벤트를 처리하거나 전파하지 않습니다.
`pipeline()`, `finished()`, 이벤트 처리기를 이용해 에러 계약을 먼저 구성하고, `isErrored()`는 진단과 테스트에 사용하세요.

## 마무리

`stream.isReadable()`, `stream.isWritable()`, `stream.isErrored()`는 스트림의 현재 상태를 읽기 쉬운 형태로 점검하게 해 주는 도구입니다.
특히 Duplex와 Transform 스트림의 읽기·쓰기 수명을 따로 진단하거나, 정상 종료와 에러 종료를 테스트할 때 유용합니다.

다만 상태 확인은 흐름 제어 자체를 대신하지 않습니다.
실제 처리는 `pipeline()`, `finished()`, backpressure, 에러 이벤트, 취소 정책으로 구성하고 상태 함수는 로그와 검증을 보강하는 용도로 사용하는 것이 안전합니다.

## 함께 보면 좋은 글

- [Node.js stream.compose 가이드: 재사용 가능한 스트림 파이프라인을 만드는 법](/development/blog/seo/2026/06/05/nodejs-stream-compose-reusable-pipeline-guide.html)
- [Node.js 스트림 backpressure 가이드: 메모리 급증을 막는 법](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)
- [Node.js Readable.fromWeb/toWeb 가이드: Web Stream과 연결하는 법](/development/blog/seo/2026/04/30/nodejs-readable-fromweb-toweb-stream-bridge-guide.html)

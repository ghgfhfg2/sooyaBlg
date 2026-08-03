---
layout: post
title: "Node.js stream.finished 가이드: 스트림 종료와 오류를 안전하게 기다리는 법"
date: 2026-08-04 08:00:00 +0900
lang: ko
translation_key: nodejs-stream-finished-completion-error-handling-guide
permalink: /development/blog/seo/2026/08/04/nodejs-stream-finished-completion-error-handling-guide.html
alternates:
  ko: /development/blog/seo/2026/08/04/nodejs-stream-finished-completion-error-handling-guide.html
  x_default: /development/blog/seo/2026/08/04/nodejs-stream-finished-completion-error-handling-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, stream, finished, pipeline, error-handling, cleanup, filesystem, javascript]
description: "Node.js stream.finished로 파일 쓰기, 압축, 업로드 같은 스트림 작업의 완료와 오류를 안전하게 기다리는 방법을 정리합니다. pipeline과의 차이, cleanup, AbortSignal, 테스트 기준까지 실무 예제로 설명합니다."
---

Node.js 스트림은 큰 파일, 네트워크 응답, 압축, 로그 전송처럼 데이터를 조각으로 처리해야 할 때 유용합니다.
하지만 스트림 코드는 "함수를 호출했다"와 "작업이 끝났다"가 다릅니다.
`writeStream.end()`를 호출했다고 파일 쓰기가 디스크까지 끝난 것은 아니고, `readable.pipe(writable)`을 연결했다고 오류 전파와 정리가 자동으로 충분히 처리되는 것도 아닙니다.

이때 필요한 도구가 `stream.finished()`입니다.
스트림이 정상 종료됐는지, 중간에 오류가 났는지, 조기 종료됐는지를 Promise 또는 콜백으로 기다릴 수 있게 해 줍니다.
특히 기존 스트림 조합을 크게 바꾸지 않으면서 완료 시점만 명확히 잡고 싶을 때 좋습니다.

이 글에서는 Node.js `stream.finished()`를 실무 코드에 적용하는 기준을 정리합니다.
스트림 조합 전체를 안전하게 묶고 싶다면 [Node.js stream.pipeline AbortSignal 가이드](/development/blog/seo/2026/07/24/nodejs-stream-pipeline-abortsignal-cleanup-guide.html)를 함께 보면 좋습니다.
스트림 상태를 관찰해야 한다면 [Node.js stream 상태 가이드](/development/blog/seo/2026/06/07/nodejs-stream-status-isreadable-iswritable-iserrored-guide.html)가 도움이 됩니다.
메모리 급증을 줄이는 흐름은 [Node.js stream backpressure 가이드](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)와 연결해서 점검할 수 있습니다.

## stream.finished가 필요한 상황

### 파일 쓰기의 실제 완료를 기다린다

쓰기 스트림은 데이터를 버퍼에 넣고 비동기로 파일에 반영합니다.
따라서 `write()`나 `end()` 호출 직후에 다음 작업을 진행하면 아직 쓰기가 끝나지 않았을 수 있습니다.
예를 들어 임시 파일을 만든 뒤 업로드하거나 이름을 바꾸는 코드라면 완료 시점을 정확히 기다려야 합니다.

```js
import { createWriteStream } from 'node:fs';
import { finished } from 'node:stream/promises';

export async function writeReport(path, lines) {
  const output = createWriteStream(path, { encoding: 'utf8' });

  for (const line of lines) {
    output.write(`${line}\n`);
  }

  output.end();
  await finished(output);
}
```

`finished(output)`은 쓰기 스트림이 정상적으로 닫히면 resolve되고, 쓰기 오류가 나면 reject됩니다.
이 패턴을 쓰면 호출자는 `await writeReport()` 이후에 파일이 완성됐다고 판단할 수 있습니다.
파일을 만든 뒤 `rename()`으로 공개 경로에 반영하는 구조라면 이 완료 대기를 빼면 안 됩니다.

### pipe 연결 뒤 오류를 놓치지 않는다

`readable.pipe(writable)`은 간단하지만 오류 처리까지 완성해 주는 API는 아닙니다.
읽기 쪽 오류와 쓰기 쪽 오류를 각각 듣지 않으면 실패가 조용히 지나가거나, 일부만 쓰인 파일을 성공으로 착각할 수 있습니다.
`finished()`는 관찰 대상 스트림의 종료와 오류를 한 곳에서 기다리게 해 줍니다.

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { finished } from 'node:stream/promises';

export async function copyWithPipe(sourcePath, targetPath) {
  const source = createReadStream(sourcePath);
  const target = createWriteStream(targetPath);

  source.pipe(target);

  await Promise.all([
    finished(source),
    finished(target)
  ]);
}
```

이 예제는 `pipe()`를 유지하면서 양쪽 스트림의 실패를 모두 기다립니다.
다만 새 코드라면 대부분 `pipeline()`이 더 안전합니다.
`finished()`는 이미 만들어진 스트림을 관찰하거나, 스트림 하나의 수명만 따로 기다릴 때 빛을 발합니다.

## pipeline과 finished의 차이

### pipeline은 연결과 정리를 책임진다

`pipeline()`은 여러 스트림을 연결하고, 오류가 나면 관련 스트림을 정리하며, 전체 파이프라인이 끝날 때까지 기다립니다.
새로운 변환 흐름을 작성한다면 보통 `pipeline()`이 기본 선택입니다.

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';
import { pipeline } from 'node:stream/promises';

export async function gzipFile(sourcePath, targetPath) {
  await pipeline(
    createReadStream(sourcePath),
    createGzip(),
    createWriteStream(targetPath)
  );
}
```

이 코드는 읽기, 압축, 쓰기 중 어디에서 오류가 나도 Promise가 reject됩니다.
또한 연결된 스트림을 정리하는 경로도 `pipe()`를 직접 조합하는 것보다 명확합니다.
완전한 데이터 흐름을 새로 만들 수 있다면 `finished()`보다 `pipeline()`을 먼저 고려하세요.

### finished는 이미 존재하는 스트림을 관찰한다

반대로 어떤 라이브러리가 스트림을 이미 만들어 돌려주거나, HTTP 응답 스트림처럼 직접 소유하지 않는 스트림의 종료만 기다려야 할 때가 있습니다.
이때는 `pipeline()`으로 다시 감싸기보다 `finished()`가 더 간단합니다.

```js
import { finished } from 'node:stream/promises';

export async function waitUntilResponseClosed(response) {
  try {
    await finished(response);
  } catch (error) {
    if (error?.code === 'ERR_STREAM_PREMATURE_CLOSE') {
      return { closedEarly: true };
    }

    throw error;
  }

  return { closedEarly: false };
}
```

조기 종료를 오류로 볼지, 사용자가 연결을 끊은 정상적인 상황으로 볼지는 서비스 요구사항에 따라 다릅니다.
다운로드 서버라면 클라이언트가 취소한 요청을 장애로 집계하지 않을 수 있습니다.
반면 내부 배치 파일 쓰기에서 조기 종료가 발생했다면 실패로 다루는 편이 맞습니다.

## cleanup 옵션과 리스너 관리

### 기본 동작은 오류 리스너를 남길 수 있다

`finished()`는 스트림의 종료와 오류 이벤트를 관찰하기 위해 리스너를 붙입니다.
기본적으로는 일부 리스너가 남을 수 있는데, 이는 늦게 발생하는 오류로 인해 프로세스가 예기치 않게 크래시하는 상황을 피하기 위한 선택입니다.
하지만 짧은 수명 스트림을 매우 많이 만들고 관찰하는 코드라면 리스너 관리가 중요해집니다.

```js
import { finished } from 'node:stream/promises';

export async function waitForUpload(stream) {
  await finished(stream, { cleanup: true });
}
```

`cleanup: true`를 사용하면 `finished()`가 resolve 또는 reject된 뒤 자신이 등록한 리스너를 제거합니다.
반복 처리 작업, 테스트, 워커 프로세스처럼 같은 패턴이 많이 실행되는 코드에서는 이 옵션을 기본값처럼 검토할 만합니다.
단, 늦게 발생하는 스트림 오류까지 흡수해야 하는 상황이라면 제거 시점의 의미를 이해하고 써야 합니다.

### destroy와 close를 구분한다

스트림이 끝나는 경로는 하나가 아닙니다.
정상적으로 모든 데이터를 처리하고 `finish` 또는 `end`에 도달할 수도 있고, 오류 때문에 `destroy()`될 수도 있으며, 리소스가 먼저 닫혀 `close`가 발생할 수도 있습니다.
`finished()`를 쓰면 이런 이벤트 조합을 직접 모두 관리하지 않아도 됩니다.

```js
import { finished } from 'node:stream/promises';

export async function consumeSafely(stream, consume) {
  try {
    await consume(stream);
    await finished(stream, { cleanup: true });
  } finally {
    if (!stream.destroyed) {
      stream.destroy();
    }
  }
}
```

`finally`에서 `destroy()`를 호출하는 패턴은 소유권이 분명할 때만 사용해야 합니다.
내가 만든 파일 스트림이나 변환 스트림이라면 정리 책임을 갖는 것이 자연스럽습니다.
하지만 프레임워크가 관리하는 요청/응답 스트림을 임의로 destroy하면 다른 미들웨어나 로깅 흐름을 깨뜨릴 수 있습니다.

## AbortSignal과 함께 쓰기

### 대기 자체를 취소할 수 있게 만든다

긴 업로드나 외부 저장소 쓰기를 기다리는 코드는 취소 경로가 필요합니다.
`finished()`는 옵션으로 `signal`을 받을 수 있어, 호출자가 더 이상 기다리지 않아도 되는 시점에 대기를 끊을 수 있습니다.

```js
import { finished } from 'node:stream/promises';

export async function waitForStream(stream, { signal }) {
  await finished(stream, {
    signal,
    cleanup: true
  });
}
```

여기서 중요한 점은 `signal`이 `finished()`의 대기를 취소한다는 것입니다.
스트림 작업 자체를 중단해야 한다면 별도로 스트림을 destroy하거나, 처음부터 `pipeline()`에 `signal`을 넘기는 구조를 고려해야 합니다.
취소가 "그만 기다린다"인지 "작업을 중단한다"인지 함수 이름과 문서에 분명히 남겨야 합니다.

### 타임아웃을 오류 경계로 만든다

스트림이 끝나지 않는 상황은 운영에서 흔한 장애입니다.
원격 연결이 끊기지 않거나, 변환 스트림이 마지막 chunk를 내보내지 않거나, writable이 `finish`까지 가지 못하면 요청이나 배치가 계속 매달릴 수 있습니다.
이럴 때는 `AbortSignal.timeout()`으로 대기 시간을 제한할 수 있습니다.

```js
import { finished } from 'node:stream/promises';

export async function waitWithTimeout(stream, timeoutMs = 30_000) {
  await finished(stream, {
    signal: AbortSignal.timeout(timeoutMs),
    cleanup: true
  });
}
```

타임아웃 값은 데이터 크기와 네트워크 조건에 맞춰 정해야 합니다.
너무 짧으면 정상 작업을 실패로 만들고, 너무 길면 장애 감지가 늦어집니다.
운영 코드에서는 타임아웃이 발생했을 때 스트림을 어떻게 정리할지와 어떤 로그를 남길지도 함께 정해야 합니다.

## 테스트와 운영 점검 기준

### 성공과 실패를 모두 테스트한다

`finished()`를 쓰는 함수는 정상 종료뿐 아니라 오류 종료도 테스트해야 합니다.
스트림은 이벤트 기반이라 실패 테스트를 빼면 실제 운영에서 Promise가 풀리지 않거나 reject를 놓치는 문제가 늦게 발견됩니다.

```js
import assert from 'node:assert/strict';
import { PassThrough } from 'node:stream';
import test from 'node:test';
import { finished } from 'node:stream/promises';

test('finished rejects when stream is destroyed with error', async () => {
  const stream = new PassThrough();
  const done = finished(stream, { cleanup: true });

  stream.destroy(new Error('write failed'));

  await assert.rejects(done, /write failed/);
});
```

테스트에서는 실제 파일이나 네트워크보다 `PassThrough` 같은 메모리 스트림을 먼저 사용하면 빠르고 안정적입니다.
파일 시스템 동작까지 확인해야 하는 테스트는 임시 디렉터리와 cleanup을 함께 둡니다.
실패 메시지에는 토큰, 사용자 데이터, 전체 경로 같은 민감한 값을 넣지 않는 편이 좋습니다.

### 운영 로그는 결과와 원인을 나눈다

스트림 완료 로그에는 "완료됐다"는 결과와 "왜 실패했는가"를 구분해서 남기는 것이 좋습니다.
사용자가 업로드를 취소한 조기 종료, 디스크 공간 부족, 권한 오류, 네트워크 리셋은 대응 방식이 다릅니다.
오류 코드를 보존하면 알림 규칙과 재시도 정책을 만들기 쉬워집니다.

```js
export function classifyStreamError(error) {
  if (error?.name === 'AbortError') return 'timeout_or_cancelled';
  if (error?.code === 'ERR_STREAM_PREMATURE_CLOSE') return 'closed_early';
  if (error?.code === 'ENOSPC') return 'disk_full';
  if (error?.code === 'EACCES') return 'permission_denied';
  return 'stream_failed';
}
```

분류는 단순해도 효과가 큽니다.
재시도 가능한 오류와 즉시 조치가 필요한 오류를 나눌 수 있고, 대시보드에서도 같은 실패가 반복되는지 보기 쉬워집니다.
단, 로그에 원본 요청 body나 파일 내용을 그대로 남기면 개인정보와 민감정보 노출 위험이 생기므로 필요한 최소 정보만 기록하세요.

## 실무 체크리스트

### 완료 대기 기준을 명시한다

스트림 코드 리뷰에서는 다음 질문을 먼저 확인하면 좋습니다.
데이터 생산이 끝난 것인지, writable이 flush된 것인지, 전체 파이프라인이 끝난 것인지가 코드에 드러나야 합니다.

- 새 스트림 조합이라면 `pipeline()`을 먼저 고려했는가?
- 이미 존재하는 스트림을 기다리는 상황이라면 `finished()`를 사용했는가?
- 읽기와 쓰기 스트림을 따로 만들었다면 양쪽 오류를 모두 기다리는가?
- 반복 실행되는 코드에서 `cleanup: true`가 필요한가?
- 취소와 타임아웃이 스트림 작업 자체를 중단하는지, 대기만 중단하는지 명확한가?

### 부분 결과물을 성공으로 취급하지 않는다

파일 생성, 압축, 업로드처럼 결과물이 남는 작업은 특히 주의해야 합니다.
스트림이 중간에 실패했는데도 다음 단계가 실행되면 깨진 파일을 캐시에 올리거나, 불완전한 산출물을 배포할 수 있습니다.
`finished()`는 이런 경계를 Promise로 표현하게 해 줍니다.

```js
export async function publishAfterWrite(writeTask, publishTask) {
  await writeTask();
  await publishTask();
}
```

이런 상위 흐름에서 `writeTask()` 내부가 `finished()` 또는 `pipeline()`을 기다리고 있어야 합니다.
그렇지 않으면 함수 이름은 완료를 암시하지만 실제로는 백그라운드 쓰기가 남아 있을 수 있습니다.
완료 경계를 명확히 만들수록 배포, 캐시 갱신, 후속 알림이 더 예측 가능해집니다.

## 마무리

`stream.finished()`는 스트림을 더 화려하게 만드는 API가 아니라, 끝났다는 사실을 믿을 수 있게 만드는 API입니다.
쓰기 스트림의 실제 완료, 조기 종료, 오류 전파, 리스너 cleanup을 명확히 다루면 스트림 코드는 훨씬 운영 친화적으로 바뀝니다.

새로운 스트림 체인을 만든다면 `pipeline()`을 우선 검토하세요.
이미 존재하는 스트림의 수명을 기다리거나 단일 스트림의 완료 경계가 필요하다면 `finished()`가 적합합니다.
두 API의 역할을 나눠 쓰면 파일 처리와 네트워크 처리 모두에서 부분 성공을 줄이고, 실패를 더 빨리 발견할 수 있습니다.

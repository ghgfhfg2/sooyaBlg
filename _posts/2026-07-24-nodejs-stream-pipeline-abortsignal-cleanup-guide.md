---
layout: post
title: "Node.js stream pipeline AbortSignal 가이드: 중단 가능한 스트림과 리소스 정리 패턴"
date: 2026-07-24 20:00:00 +0900
lang: ko
translation_key: nodejs-stream-pipeline-abortsignal-cleanup-guide
permalink: /development/blog/seo/2026/07/24/nodejs-stream-pipeline-abortsignal-cleanup-guide.html
alternates:
  ko: /development/blog/seo/2026/07/24/nodejs-stream-pipeline-abortsignal-cleanup-guide.html
  x_default: /development/blog/seo/2026/07/24/nodejs-stream-pipeline-abortsignal-cleanup-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, stream, pipeline, abortsignal, cleanup, backend, performance, reliability]
description: "Node.js stream pipeline에서 AbortSignal을 적용해 긴 파일 처리와 네트워크 스트림을 안전하게 중단하고, 실패 후 리소스를 정리하는 실무 패턴을 정리합니다."
---

Node.js에서 큰 파일을 읽거나 외부 응답을 변환할 때 stream은 메모리 사용량을 낮추는 좋은 선택입니다.
하지만 스트림 처리는 한 번 시작하면 여러 리소스가 동시에 움직입니다.
파일 핸들, 네트워크 연결, 압축 스트림, 변환 스트림이 이어져 있기 때문에 중간에 요청이 취소됐을 때 정리 기준이 흐려지기 쉽습니다.

이 글에서는 Node.js `stream.pipeline()`에 `AbortSignal`을 연결해 중단 가능한 스트림 처리를 만드는 방법을 정리합니다.
핵심은 스트림을 단순히 연결하는 것이 아니라, **호출자의 취소 신호와 정리 책임을 pipeline 경계에 모으는 것**입니다.
[Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html), [Node.js fetch timeout 재시도 가이드](/development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html), [Node.js timeout budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와 함께 보면 외부 I/O와 내부 스트림 처리의 취소 흐름을 일관되게 잡을 수 있습니다.

## stream pipeline에 AbortSignal이 필요한 이유

### H3. 큰 데이터 처리는 성공보다 중단 경로가 더 어렵다

작은 JSON 응답은 실패해도 처리 범위가 좁습니다.
반대로 큰 CSV 파일, 이미지 변환, 로그 압축, 프록시 응답 같은 작업은 중간 상태가 많습니다.
읽는 쪽은 계속 데이터를 밀어 넣고, 변환 스트림은 버퍼를 들고 있으며, 쓰는 쪽은 디스크나 네트워크에 데이터를 보내고 있을 수 있습니다.

이때 사용자가 요청을 취소했는데 스트림이 계속 돌면 서버 리소스가 낭비됩니다.
더 나쁘게는 실패한 작업의 일부 결과가 파일로 남거나, 다음 작업이 같은 임시 파일을 잘못 읽을 수도 있습니다.
그래서 스트림 처리에서는 정상 완료 경로만큼 취소와 정리 경로가 중요합니다.

`pipeline()`은 여러 스트림을 하나의 작업으로 묶고, 실패가 발생했을 때 연결된 스트림을 함께 정리하는 데 도움을 줍니다.
여기에 `AbortSignal`을 붙이면 호출자가 작업 중단을 명시적으로 전달할 수 있습니다.

### H3. 이벤트를 직접 묶는 방식은 누락이 생기기 쉽다

스트림을 직접 `pipe()`로 연결하면 코드가 짧아 보입니다.
하지만 오류와 완료 이벤트를 모두 직접 관리해야 합니다.

```js
readable
  .pipe(transform)
  .pipe(writable);
```

이 코드는 예제에서는 충분해 보이지만 운영 코드에서는 부족합니다.
어느 스트림에서 오류가 났는지, 다른 스트림을 닫아야 하는지, 호출자가 취소했을 때 어떤 이벤트가 먼저 오는지 직접 다뤄야 합니다.

`pipeline()`은 이 책임을 하나의 Promise 경계로 모읍니다.
따라서 API 핸들러나 작업 큐에서는 `await pipeline(...)` 형태로 성공, 실패, 취소를 같은 흐름에서 처리할 수 있습니다.

## 중단 가능한 pipeline 기본형

### H3. promises API를 사용해 완료 경계를 만든다

Node.js에서는 `node:stream/promises`의 `pipeline`을 사용하면 스트림 완료를 Promise로 기다릴 수 있습니다.
파일을 읽어 gzip으로 압축한 뒤 다른 파일로 쓰는 코드는 다음처럼 구성할 수 있습니다.

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';

export async function compressFile(inputPath, outputPath, { signal } = {}) {
  await pipeline(
    createReadStream(inputPath),
    createGzip(),
    createWriteStream(outputPath),
    { signal }
  );
}
```

여기서 `signal`은 상위 호출자가 넘겨준 취소 신호입니다.
HTTP 요청이 끊겼거나 작업 큐에서 timeout이 났다면 같은 signal을 이 함수에 넘길 수 있습니다.
pipeline이 중단되면 Promise는 reject되고, 호출부는 그 오류를 기준으로 로그와 정리를 수행합니다.

### H3. timeout signal과 호출자 signal을 함께 묶는다

실무에서는 호출자 취소와 내부 timeout을 함께 적용해야 합니다.
예를 들어 사용자가 요청을 취소해도 멈춰야 하고, 사용자가 기다리고 있더라도 작업이 10초를 넘으면 멈춰야 합니다.

```js
function composePipelineSignal({ signal, timeoutMs }) {
  const timeoutSignal = AbortSignal.timeout(timeoutMs);

  if (!signal) {
    return timeoutSignal;
  }

  return AbortSignal.any([signal, timeoutSignal]);
}

export async function compressFileWithTimeout(inputPath, outputPath, {
  signal,
  timeoutMs = 10_000
} = {}) {
  await pipeline(
    createReadStream(inputPath),
    createGzip(),
    createWriteStream(outputPath),
    {
      signal: composePipelineSignal({ signal, timeoutMs })
    }
  );
}
```

이 방식은 상위 계층의 취소 정책을 하위 스트림 작업에 전달합니다.
API 서버에서는 클라이언트 연결 종료, 라우트별 timeout, 백그라운드 작업 timeout을 한 방향으로 모을 수 있습니다.

## 실패 후 리소스 정리 패턴

### H3. 임시 파일은 성공 후에만 최종 경로로 옮긴다

스트림 작업이 중간에 실패할 수 있다면 최종 파일에 바로 쓰는 방식은 위험합니다.
부분적으로 기록된 파일이 정상 결과처럼 노출될 수 있기 때문입니다.
더 안전한 방식은 임시 파일에 먼저 쓰고, pipeline이 성공한 뒤에만 rename하는 것입니다.

```js
import { rename, rm } from 'node:fs/promises';

export async function compressFileAtomically(inputPath, outputPath, options = {}) {
  const tempPath = `${outputPath}.tmp-${process.pid}-${Date.now()}`;

  try {
    await compressFileWithTimeout(inputPath, tempPath, options);
    await rename(tempPath, outputPath);
  } catch (error) {
    await rm(tempPath, { force: true });
    throw error;
  }
}
```

이 패턴은 배치 작업, 업로드 후 변환, 정적 파일 생성에 특히 유용합니다.
실패한 결과를 지우고 원래 오류를 다시 던지기 때문에 호출부는 실패 원인을 잃지 않습니다.

### H3. 취소 오류와 실제 장애를 로그에서 분리한다

AbortSignal로 중단된 작업은 모두 같은 심각도의 장애가 아닙니다.
사용자가 다운로드를 취소한 경우와 내부 timeout으로 작업을 끊은 경우는 운영 대응이 다릅니다.
따라서 로그에서는 취소 이유를 가능한 한 분리해야 합니다.

```js
function classifyPipelineError(error) {
  if (error?.name === 'AbortError') {
    return 'pipeline_aborted';
  }

  if (error?.name === 'TimeoutError') {
    return 'pipeline_timeout';
  }

  return 'pipeline_failed';
}
```

환경에 따라 timeout도 `AbortError`처럼 보일 수 있으므로, 호출부에서 어떤 signal을 만들었는지 함께 기록하는 편이 좋습니다.
예를 들어 `timeoutMs`, 작업 종류, 입력 파일 크기, 요청 ID를 구조화 로그에 남기면 원인 분석이 쉬워집니다.
단, 파일 경로나 사용자 식별자가 민감정보가 될 수 있다면 그대로 남기지 말고 필요한 범위만 마스킹해야 합니다.

## API 핸들러에서 적용하는 예시

### H3. 요청 종료를 stream 작업에 연결한다

Express나 Fastify 같은 서버에서는 클라이언트 연결이 끊겼을 때 작업을 멈추는 흐름을 만들 수 있습니다.
핵심은 요청 생명주기를 `AbortController`에 연결하고, 그 signal을 스트림 함수까지 전달하는 것입니다.

```js
function signalFromRequest(req) {
  const controller = new AbortController();

  req.on('close', () => {
    controller.abort();
  });

  return controller.signal;
}

app.post('/reports/:id/export', async (req, res, next) => {
  const signal = signalFromRequest(req);

  try {
    const outputPath = `/tmp/report-${req.params.id}.gz`;

    await compressFileAtomically('/data/report.csv', outputPath, {
      signal,
      timeoutMs: 15_000
    });

    res.download(outputPath);
  } catch (error) {
    if (signal.aborted) {
      return;
    }

    next(error);
  }
});
```

이 예시는 구조를 보여 주기 위한 코드입니다.
운영 환경에서는 사용자 입력으로 파일 경로를 직접 만들지 말고, 허용된 ID를 검증한 뒤 내부 저장소 경로로 매핑해야 합니다.
또한 `/tmp` 파일 정리 주기와 다운로드 완료 후 삭제 정책도 함께 정해야 합니다.

### H3. 백그라운드 작업에서는 재시도보다 멱등성이 먼저다

스트림 작업이 실패했다고 해서 항상 재시도하면 안 됩니다.
입력 데이터가 바뀌었거나, 출력 대상에 일부 결과가 남았거나, 같은 작업이 이미 다시 큐에 들어갔을 수 있습니다.
재시도 전에 작업 ID, 출력 경로, 임시 파일 정리, 최종 결과 교체 방식이 멱등적으로 설계되어 있어야 합니다.

좋은 기준은 다음과 같습니다.

- 같은 작업 ID는 같은 입력 스냅샷을 바라본다.
- 최종 결과는 성공 후 한 번에 교체한다.
- 실패한 임시 결과는 다음 실행 전에 제거한다.
- timeout과 사용자 취소는 별도 메트릭으로 기록한다.
- 재시도 횟수보다 전체 처리 시간을 우선 제한한다.

이 기준을 지키면 stream pipeline은 단순한 파일 처리 도구를 넘어 안정적인 데이터 처리 경계가 됩니다.

## 실무 체크리스트

### H3. pipeline을 도입할 때 확인할 것

`AbortSignal`을 붙인 pipeline은 코드 몇 줄로 시작할 수 있지만, 운영 품질은 주변 규칙에서 결정됩니다.
다음 항목을 배포 전에 확인하는 편이 좋습니다.

- `pipe()` 대신 `node:stream/promises`의 `pipeline()`을 사용했는가?
- 상위 호출자의 `signal`을 하위 스트림 작업까지 전달했는가?
- 내부 timeout을 두고 전체 요청 예산과 충돌하지 않게 조정했는가?
- 실패한 출력 파일이나 임시 파일을 정리하는가?
- 취소, timeout, 실제 장애를 로그와 메트릭에서 구분하는가?
- 사용자 입력이 파일 경로나 외부 URL로 직접 연결되지 않게 검증했는가?

스트림은 메모리 사용량을 줄여 주지만, 자동으로 안전한 작업 경계를 만들어 주지는 않습니다.
pipeline, AbortSignal, 임시 파일 정리를 함께 설계해야 긴 데이터 처리도 예측 가능한 운영 코드가 됩니다.

## FAQ

### H3. stream pipeline에 항상 timeout을 넣어야 하나요?

외부 입력, 네트워크, 큰 파일처럼 처리 시간이 흔들릴 수 있는 작업에는 timeout을 두는 편이 좋습니다.
다만 너무 짧은 timeout은 정상 작업을 실패로 만들 수 있으므로 파일 크기, 처리량, 사용자 경험 목표를 함께 보고 정해야 합니다.

### H3. AbortError는 오류로 기록해야 하나요?

기록은 하되 심각도를 나눠야 합니다.
사용자 취소로 생긴 AbortError는 정보성 로그로 충분할 수 있고, 내부 timeout으로 생긴 중단은 성능 문제나 의존성 장애의 신호일 수 있습니다.

### H3. pipeline 실패 후 재시도하면 안전한가요?

임시 파일에 쓰고 성공 후 rename하는 구조라면 비교적 안전합니다.
하지만 외부 시스템에 데이터를 쓰는 스트림이라면 중복 전송 가능성을 먼저 확인해야 합니다.
멱등성 없는 작업에는 자동 재시도를 좁게 적용해야 합니다.

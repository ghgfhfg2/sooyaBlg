---
layout: post
title: "Node.js scheduler.yield 가이드: 긴 작업에서 이벤트 루프를 안전하게 양보하는 법"
date: 2026-08-07 20:00:00 +0900
lang: ko
translation_key: nodejs-timers-promises-scheduler-yield-event-loop-guide
permalink: /development/blog/seo/2026/08/07/nodejs-timers-promises-scheduler-yield-event-loop-guide.html
alternates:
  ko: /development/blog/seo/2026/08/07/nodejs-timers-promises-scheduler-yield-event-loop-guide.html
  x_default: /development/blog/seo/2026/08/07/nodejs-timers-promises-scheduler-yield-event-loop-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, timers-promises, scheduler-yield, event-loop, performance, backend, javascript, batch-processing]
description: "Node.js scheduler.yield()로 대량 데이터 처리, JSON 변환, 배치 작업 중 이벤트 루프를 주기적으로 양보하는 방법을 정리합니다. setImmediate와의 차이, chunk 크기, AbortSignal, 테스트와 운영 지표까지 실무 예제로 설명합니다."
---

Node.js 서버에서 CPU 작업이 길어지면 증상이 애매하게 나타납니다.
API 자체는 죽지 않았는데 응답 시간이 출렁이고, health check는 간헐적으로 늦어지고, 로그에는 특별한 에러가 남지 않을 수 있습니다.
원인은 종종 긴 반복문, 큰 JSON 변환, 대량 레코드 정규화처럼 이벤트 루프를 오래 붙잡는 코드입니다.

이때 무조건 worker thread로 옮기는 것이 정답은 아닙니다.
작업이 아주 무겁거나 병렬 계산이 필요하면 worker가 맞지만, 짧은 동기 작업이 많이 이어지는 정도라면 먼저 **작업을 작은 단위로 나누고 이벤트 루프에 양보 지점을 넣는 방식**을 검토할 수 있습니다.
Node.js의 `node:timers/promises`가 제공하는 `scheduler.yield()`는 이런 상황에서 읽기 쉬운 비동기 양보 지점으로 쓰기 좋습니다.

이 글에서는 `scheduler.yield()`가 필요한 상황, `setImmediate`와의 차이, 배치 처리에서 chunk 크기를 잡는 기준, 취소와 테스트 방법을 실무 관점으로 정리합니다.
대기 시간을 명시적으로 제어하는 패턴은 [Node.js scheduler.wait 가이드](/development/blog/seo/2026/07/21/nodejs-timers-promises-scheduler-wait-cancellable-delay-guide.html)와 함께 보면 좋습니다.
CPU 바운드 작업의 분리 기준은 [Node.js worker_threads 가이드](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)를 참고하세요.
작업 시간이 실제로 줄었는지 확인하려면 [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)가 이어서 도움이 됩니다.

## scheduler.yield가 필요한 상황

### H3. 긴 반복문이 이벤트 루프를 계속 붙잡는다

Node.js는 JavaScript 실행이 끝나야 다음 I/O callback, timer, promise continuation을 처리할 수 있습니다.
아래처럼 대량 배열을 한 번에 순회하면 각 연산은 작아 보여도 전체 요청 경로에서는 이벤트 루프를 오래 점유할 수 있습니다.

```js
export function normalizeRows(rows) {
  return rows.map((row) => ({
    id: String(row.id),
    email: row.email.trim().toLowerCase(),
    score: Number(row.score ?? 0),
    active: row.deleted_at === null
  }));
}
```

1천 건에서는 문제가 없던 코드가 50만 건에서는 다른 요청까지 늦게 만들 수 있습니다.
특히 관리자 CSV 업로드, 검색 인덱스 재생성, 캐시 warm-up, 메시지 재처리처럼 백그라운드 성격의 작업이 웹 서버 프로세스 안에서 함께 실행될 때 문제가 커집니다.

`scheduler.yield()`는 이런 긴 작업 중간에 "여기까지 처리했으니 다른 대기 중인 작업도 한 번 진행하게 하자"는 지점을 만들 때 사용할 수 있습니다.

```js
import { scheduler } from 'node:timers/promises';

export async function normalizeRowsInChunks(rows, chunkSize = 1000) {
  const normalized = [];

  for (let index = 0; index < rows.length; index += 1) {
    const row = rows[index];

    normalized.push({
      id: String(row.id),
      email: row.email.trim().toLowerCase(),
      score: Number(row.score ?? 0),
      active: row.deleted_at === null
    });

    if ((index + 1) % chunkSize === 0) {
      await scheduler.yield();
    }
  }

  return normalized;
}
```

이 코드는 전체 작업량을 줄이지 않습니다.
대신 긴 작업을 잘게 끊어 이벤트 루프가 다른 callback을 처리할 기회를 얻도록 만듭니다.
사용자 요청을 처리하는 서버라면 전체 처리 시간보다 **동시 요청의 지연 폭**이 더 중요할 때가 많습니다.

### H3. yield는 성능 최적화가 아니라 응답성 최적화다

`scheduler.yield()`를 넣으면 단일 작업의 총 실행 시간이 약간 늘 수 있습니다.
중간에 양보하고 다시 이어서 실행해야 하기 때문입니다.
하지만 서버 전체 관점에서는 health check, 짧은 API 요청, timeout timer, 취소 signal이 더 늦지 않게 처리됩니다.

따라서 목표를 분명히 해야 합니다.

- 단일 batch를 가장 빨리 끝내고 싶다면 yield가 도움이 되지 않을 수 있습니다.
- 사용자 요청과 백그라운드 작업이 같은 프로세스에서 섞인다면 yield가 tail latency를 줄이는 데 도움이 될 수 있습니다.
- 계산 자체가 무겁고 오래 걸린다면 worker thread나 별도 job worker로 분리해야 합니다.
- 외부 API나 DB 호출이 병목이라면 yield보다 timeout, concurrency limit, backpressure가 먼저입니다.

이 기준을 혼동하면 yield를 넣고도 장애가 그대로일 수 있습니다.
이벤트 루프를 오래 막는 JavaScript 작업인지, 외부 의존성 대기인지 먼저 분리해서 봐야 합니다.

## setImmediate와 scheduler.yield의 차이

### H3. 목적은 비슷하지만 코드 의도가 다르게 보인다

예전에는 이벤트 루프에 양보하기 위해 `setImmediate`를 Promise로 감싸는 패턴을 자주 썼습니다.

```js
function yieldToEventLoop() {
  return new Promise((resolve) => {
    setImmediate(resolve);
  });
}
```

이 방식도 동작합니다.
다만 코드만 봐서는 "즉시 실행 timer를 쓰는 이유"를 매번 해석해야 합니다.
반면 `scheduler.yield()`는 이름 그대로 양보 의도를 드러냅니다.

```js
import { scheduler } from 'node:timers/promises';

await scheduler.yield();
```

팀 코드베이스에서는 이런 작은 차이가 중요합니다.
긴 반복문 중간에 `await scheduler.yield()`가 보이면, 이 줄은 지연을 만들기 위한 코드가 아니라 이벤트 루프 독점을 피하기 위한 코드라는 의도가 바로 드러납니다.

### H3. 무분별하게 모든 반복에 넣으면 안 된다

반복마다 yield를 넣으면 작업이 지나치게 느려질 수 있습니다.
아래 코드는 응답성은 좋아 보이지만, 실제 처리량은 크게 떨어질 가능성이 큽니다.

```js
import { scheduler } from 'node:timers/promises';

export async function badNormalizeRows(rows) {
  const normalized = [];

  for (const row of rows) {
    normalized.push(normalizeRow(row));
    await scheduler.yield();
  }

  return normalized;
}
```

대부분의 경우에는 일정 개수마다 한 번 양보하는 chunk 방식이 더 현실적입니다.
chunk 크기는 정답이 고정되어 있지 않습니다.
레코드 하나의 처리 비용, 서버의 동시 요청량, 허용 가능한 batch 완료 시간에 따라 달라집니다.

처음에는 500개, 1000개, 5000개 같은 단순한 값으로 시작하고 운영 지표를 보면서 조정하세요.
중요한 것은 "얼마나 자주 yield했는가"가 아니라 "event loop lag와 사용자 요청 p95, p99가 안정됐는가"입니다.

## 배치 처리에 적용하는 실무 패턴

### H3. chunk 단위 helper로 반복 구조를 고정한다

여러 곳에서 대량 처리를 한다면 yield 조건을 매번 직접 쓰기보다 작은 helper로 고정하는 편이 좋습니다.

```js
import { scheduler } from 'node:timers/promises';

export async function forEachWithYield(items, handler, options = {}) {
  const chunkSize = options.chunkSize ?? 1000;
  const signal = options.signal;

  for (let index = 0; index < items.length; index += 1) {
    if (signal?.aborted) {
      throw signal.reason ?? new Error('Operation aborted');
    }

    await handler(items[index], index);

    if ((index + 1) % chunkSize === 0) {
      await scheduler.yield();
    }
  }
}
```

이 helper는 동기 handler와 비동기 handler를 모두 받을 수 있습니다.
단, handler가 이미 느린 외부 I/O를 수행한다면 yield의 효과는 제한적입니다.
그 경우에는 concurrency limit, retry, timeout budget을 같이 설계해야 합니다.

사용 예시는 단순합니다.

```js
await forEachWithYield(users, async (user) => {
  await searchIndex.addDocument({
    id: user.id,
    title: user.name,
    emailDomain: user.email.split('@')[1]
  });
}, {
  chunkSize: 500,
  signal: abortController.signal
});
```

운영 코드에서는 `chunkSize`를 환경 변수나 작업 종류별 설정으로 빼도 됩니다.
다만 너무 많은 옵션을 노출하면 조정 기준이 흐려지므로, 처음에는 작업 성격별 기본값 2~3개 정도로 제한하는 편이 좋습니다.

### H3. AbortSignal과 함께 써야 종료가 빨라진다

긴 작업은 양보만큼 취소도 중요합니다.
배포 중 shutdown이 시작됐거나, 사용자가 업로드를 취소했거나, 상위 job deadline이 끝났다면 남은 레코드를 계속 처리하지 않아야 합니다.

```js
import { scheduler } from 'node:timers/promises';

export async function rebuildCache(records, { signal, chunkSize = 1000 } = {}) {
  const cacheEntries = new Map();

  for (let index = 0; index < records.length; index += 1) {
    if (signal?.aborted) {
      throw signal.reason ?? new Error('Cache rebuild aborted');
    }

    const record = records[index];
    cacheEntries.set(record.key, JSON.stringify(record.value));

    if ((index + 1) % chunkSize === 0) {
      await scheduler.yield();
    }
  }

  return cacheEntries;
}
```

`scheduler.yield()` 자체가 취소 signal을 받는 지연 API는 아닙니다.
그래서 loop 안에서 signal 상태를 직접 확인해야 합니다.
취소 가능한 일정 시간 대기가 필요하면 `scheduler.wait()`처럼 signal을 받을 수 있는 API와 목적을 나누어 사용하세요.

## 운영 지표와 테스트 기준

### H3. event loop lag와 작업 완료 시간을 같이 본다

yield를 넣은 뒤에는 두 가지를 함께 봐야 합니다.

- batch 전체 완료 시간
- event loop lag, API p95/p99, health check 지연

전체 완료 시간이 조금 늘어도 사용자 요청 지연이 크게 줄었다면 좋은 선택일 수 있습니다.
반대로 사용자 지연은 그대로인데 batch만 느려졌다면 chunk 크기가 너무 작거나 병목이 다른 곳에 있을 가능성이 큽니다.

간단한 측정은 `performance.now()`로 시작해도 됩니다.

```js
import { performance } from 'node:perf_hooks';

const startedAt = performance.now();

await normalizeRowsInChunks(rows, 1000);

logger.info({
  rowCount: rows.length,
  durationMs: Math.round(performance.now() - startedAt)
}, 'rows normalized');
```

운영 로그에는 원본 row 전체를 남기지 마세요.
row 수, chunk 크기, 소요 시간, 실패 코드 정도면 충분합니다.
개인정보나 토큰이 섞일 수 있는 payload를 그대로 기록하면 진단 코드가 보안 리스크가 됩니다.

### H3. 테스트에서는 양보 횟수보다 취소와 결과를 검증한다

`scheduler.yield()`가 정확히 몇 번 호출됐는지에 집착하면 테스트가 구현 세부사항에 묶입니다.
대신 결과가 유지되는지, 취소가 빠르게 반영되는지, chunk 옵션이 극단값에서도 깨지지 않는지를 보는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

test('normalizes rows without changing order', async () => {
  const rows = [
    { id: 1, email: ' A@EXAMPLE.COM ', score: '3', deleted_at: null },
    { id: 2, email: ' b@example.com ', score: null, deleted_at: '2026-01-01' }
  ];

  const result = await normalizeRowsInChunks(rows, 1);

  assert.deepEqual(result.map((row) => row.id), ['1', '2']);
  assert.equal(result[0].email, 'a@example.com');
  assert.equal(result[1].active, false);
});

test('stops when abort signal is already aborted', async () => {
  const controller = new AbortController();
  controller.abort(new Error('stop'));

  await assert.rejects(
    () => forEachWithYield([1, 2, 3], () => {}, { signal: controller.signal }),
    /stop/
  );
});
```

테스트에서 chunk 크기를 `1`로 낮추면 양보 경로를 쉽게 지나가게 할 수 있습니다.
운영 기본값과 테스트 기본값을 억지로 같게 만들 필요는 없습니다.

## scheduler.yield 사용 체크리스트

### H3. 먼저 병목이 이벤트 루프 점유인지 확인한다

`scheduler.yield()`는 좋은 도구지만, 모든 느린 작업의 해결책은 아닙니다.
적용 전에는 아래 질문을 확인하세요.

- 긴 동기 반복문이나 변환 로직이 있는가?
- 해당 작업이 웹 요청과 같은 Node.js 프로세스에서 실행되는가?
- 전체 처리량보다 동시 요청 응답성이 더 중요한가?
- chunk 처리 후에도 결과 순서와 오류 처리가 명확한가?
- 취소 signal, shutdown, deadline을 반영할 수 있는가?
- 민감정보를 로그나 metric payload에 남기지 않는가?

이 질문에 대부분 "예"라고 답할 수 있다면 `scheduler.yield()`를 도입할 만합니다.
반대로 작업이 CPU를 오래 태우는 압축, 이미지 처리, 암호화, 복잡한 계산이라면 yield보다 worker thread나 별도 job 시스템이 더 적합합니다.

### H3. 작은 양보 지점이 운영 안정성을 만든다

Node.js에서 성능 문제는 항상 "빠르게 끝내기"만의 문제가 아닙니다.
서버는 동시에 여러 일을 처리해야 하고, 짧은 요청이 긴 작업 뒤에 줄 서는 상황을 피해야 합니다.
`scheduler.yield()`는 긴 JavaScript 작업을 완전히 없애지는 않지만, 이벤트 루프가 숨 쉴 지점을 만들어 줍니다.

실무 적용 순서는 단순합니다.
먼저 긴 반복문을 찾고, chunk 단위로 나누고, `await scheduler.yield()`를 넣고, event loop lag와 p95/p99를 비교하세요.
그다음에도 지연이 크다면 worker thread, 별도 queue, adaptive concurrency limit 같은 더 강한 구조를 검토하면 됩니다.

## FAQ

### H3. scheduler.yield를 쓰면 CPU 사용량이 줄어드나요?

아닙니다.
총 작업량은 그대로입니다.
`scheduler.yield()`는 CPU 비용을 없애는 도구가 아니라 이벤트 루프에 다른 작업을 처리할 기회를 주는 도구입니다.

### H3. setTimeout 0 대신 scheduler.yield를 써도 되나요?

이벤트 루프에 양보하려는 의도라면 `scheduler.yield()`가 더 읽기 쉽습니다.
특정 시간만큼 기다리는 것이 목적이라면 `scheduler.wait()`나 `setTimeout` 계열을 쓰는 편이 맞습니다.

### H3. 모든 배치 작업에 yield를 넣어야 하나요?

아닙니다.
사용자 요청과 격리된 별도 worker 프로세스에서 실행되고 응답성 문제가 없다면 굳이 넣지 않아도 됩니다.
같은 프로세스에서 긴 JavaScript 작업이 다른 요청을 늦출 때 우선 검토하세요.

## 내부링크

- [Node.js scheduler.wait 가이드](/development/blog/seo/2026/07/21/nodejs-timers-promises-scheduler-wait-cancellable-delay-guide.html)
- [Node.js worker_threads 가이드](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)
- [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html)

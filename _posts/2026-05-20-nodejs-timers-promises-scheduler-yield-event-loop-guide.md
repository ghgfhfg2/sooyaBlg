---
layout: post
title: "Node.js scheduler.yield 가이드: 긴 작업에서 이벤트 루프를 안전하게 양보하는 법"
date: 2026-05-20 08:00:00 +0900
lang: ko
translation_key: nodejs-timers-promises-scheduler-yield-event-loop-guide
permalink: /development/blog/seo/2026/05/20/nodejs-timers-promises-scheduler-yield-event-loop-guide.html
alternates:
  ko: /development/blog/seo/2026/05/20/nodejs-timers-promises-scheduler-yield-event-loop-guide.html
  x_default: /development/blog/seo/2026/05/20/nodejs-timers-promises-scheduler-yield-event-loop-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, timers-promises, scheduler-yield, event-loop, performance, javascript]
description: "Node.js timers/promises의 scheduler.yield()로 긴 반복 작업 중 이벤트 루프를 양보하는 방법을 정리했습니다. setImmediate와의 차이, 배치 처리 기준, AbortSignal 취소 조합, 테스트와 성능 측정 포인트까지 실무 예제로 설명합니다."
---

Node.js 서버에서 CPU를 오래 붙잡는 작업은 응답 지연으로 바로 이어집니다.
대량 JSON 변환, 로그 파일 후처리, 캐시 재계산, 여러 레코드의 검증처럼 한 요청 안에서 많은 항목을 순회하는 코드는 특히 조심해야 합니다.
코드 자체는 비동기 함수 안에 있어도, 반복문 내부가 동기 계산으로 가득 차 있으면 이벤트 루프가 다른 콜백을 처리할 틈을 얻지 못합니다.

`node:timers/promises` 모듈의 `scheduler.yield()`는 이런 상황에서 의도적으로 실행권을 한 번 양보하게 해 줍니다.
작업을 완전히 다른 스레드로 보내는 도구는 아니지만, 긴 반복 작업을 작은 배치로 나누고 중간중간 이벤트 루프가 숨 쉴 시간을 주는 데 유용합니다.
이 글에서는 Node.js `scheduler.yield()` 기본 사용법, `setImmediate()`와의 차이, 배치 크기 결정 기준, `AbortSignal` 취소 조합, 테스트와 성능 측정 포인트를 정리합니다.
취소 신호 설계가 먼저 필요하다면 [Node.js AbortSignal.throwIfAborted 가이드: 취소 체크포인트를 안전하게 넣는 법](/development/blog/seo/2026/05/06/nodejs-abortsignal-throwifaborted-cancellation-checkpoint-guide.html)도 함께 보면 좋습니다.

## scheduler.yield가 필요한 이유

### H3. 비동기 함수라도 긴 동기 반복문은 이벤트 루프를 막는다

`async` 함수 안에 있다고 해서 모든 코드가 자동으로 나뉘어 실행되는 것은 아닙니다.
`await`를 만나기 전까지는 일반 동기 코드처럼 계속 실행됩니다.
아래처럼 큰 배열을 한 번에 처리하면 HTTP 요청 콜백, 타이머, 로그 flush 같은 다른 작업이 뒤로 밀릴 수 있습니다.

```js
export async function normalizeRows(rows) {
  const result = [];

  for (const row of rows) {
    result.push({
      id: String(row.id),
      email: String(row.email).trim().toLowerCase(),
      active: row.deletedAt == null,
    });
  }

  return result;
}
```

데이터가 적을 때는 문제가 보이지 않습니다.
하지만 수십만 건을 다루거나 여러 요청이 동시에 들어오면 한 번의 반복문이 프로세스 전체 응답성을 떨어뜨릴 수 있습니다.
이때 반복문을 배치 단위로 끊고 `await scheduler.yield()`를 넣으면 다른 대기 중인 작업이 실행될 기회를 얻습니다.

### H3. 워커 스레드로 보내기 전의 가벼운 완충 장치가 된다

CPU 사용량이 매우 큰 작업은 [Node.js worker_threads 환경 데이터 가이드: 워커에 설정값 안전하게 전달하는 법](/development/blog/seo/2026/05/19/nodejs-worker-threads-environment-data-guide.html)처럼 워커 스레드 분리를 검토하는 편이 맞습니다.
다만 모든 반복 작업을 워커로 옮기는 것은 비용이 있습니다.
직렬화, 워커 풀 관리, 에러 전달, 배포 복잡도가 늘어납니다.

`scheduler.yield()`는 작업을 병렬화하지 않습니다.
대신 하나의 스레드 안에서 긴 작업을 협력적으로 나누는 방식입니다.
작업 시간이 아주 길지는 않지만 이벤트 루프 지연을 줄이고 싶은 경우, 워커 도입 전 단계의 실용적인 선택지가 됩니다.

## 기본 사용법

### H3. node:timers/promises에서 scheduler를 가져온다

`scheduler.yield()`는 Promise를 반환하므로 `await`와 함께 사용합니다.
반복문에서 일정 개수마다 한 번씩 호출하면 처리량과 응답성 사이의 균형을 맞출 수 있습니다.

```js
import { scheduler } from 'node:timers/promises';

export async function normalizeRows(rows, batchSize = 1000) {
  const result = [];

  for (let index = 0; index < rows.length; index += 1) {
    const row = rows[index];

    result.push({
      id: String(row.id),
      email: String(row.email).trim().toLowerCase(),
      active: row.deletedAt == null,
    });

    if ((index + 1) % batchSize === 0) {
      await scheduler.yield();
    }
  }

  return result;
}
```

핵심은 매 항목마다 양보하지 않는 것입니다.
너무 자주 `yield`하면 컨텍스트 전환 비용이 커져 전체 처리 시간이 늘어납니다.
반대로 너무 드물게 호출하면 이벤트 루프 지연 개선 효과가 작습니다.
실무에서는 항목 수, 항목당 계산 비용, 허용 가능한 지연 시간을 함께 보고 배치 크기를 정해야 합니다.

### H3. 결과 순서를 유지하면서 처리할 수 있다

`scheduler.yield()`는 현재 함수의 상태를 잃어버리지 않습니다.
반복문은 다음 차례부터 이어지고, 배열에 쌓은 결과 순서도 그대로 유지됩니다.
따라서 순서가 중요한 변환 작업에도 비교적 안전하게 적용할 수 있습니다.

```js
import { scheduler } from 'node:timers/promises';

export async function buildSearchDocuments(products) {
  const documents = [];

  for (let index = 0; index < products.length; index += 1) {
    const product = products[index];

    documents.push({
      objectID: product.id,
      title: product.name,
      text: [product.name, product.brand, product.category].join(' '),
    });

    if (index > 0 && index % 500 === 0) {
      await scheduler.yield();
    }
  }

  return documents;
}
```

이 패턴은 검색 색인 문서 생성, CSV 행 변환, 캐시 엔트리 재계산처럼 입력 순서와 출력 순서가 맞아야 하는 작업에 잘 맞습니다.
다만 각 항목 처리가 비동기 I/O를 포함한다면 `scheduler.yield()`보다 동시성 제한 큐나 스트림 설계가 더 중요할 수 있습니다.

## setImmediate와 무엇이 다른가

### H3. 의도 표현이 더 명확하다

과거에는 이벤트 루프에 실행권을 돌려주기 위해 `setImmediate()`를 Promise로 감싸는 코드를 자주 썼습니다.
이 방식도 동작하지만, 코드의 의도가 타이머 예약인지 양보인지 한눈에 드러나지 않습니다.

```js
function yieldWithSetImmediate() {
  return new Promise((resolve) => {
    setImmediate(resolve);
  });
}

await yieldWithSetImmediate();
```

`scheduler.yield()`를 쓰면 “여기서 스케줄러에 한 번 양보한다”는 의도가 API 이름에 남습니다.
타이머를 직접 감싸는 보일러플레이트도 줄어듭니다.

```js
import { scheduler } from 'node:timers/promises';

await scheduler.yield();
```

새로운 팀원이 코드를 읽을 때도 차이가 큽니다.
`setImmediate` 래퍼는 왜 필요한지 주석을 요구하는 경우가 많지만, `scheduler.yield()`는 반복 작업의 체크포인트라는 의미가 비교적 분명합니다.

### H3. 지연 시간을 보장하는 API는 아니다

`scheduler.yield()`는 “몇 밀리초 기다린다”는 의미가 아닙니다.
정확한 시간 지연이 필요하다면 같은 모듈의 `setTimeout()` Promise API를 사용해야 합니다.

```js
import { setTimeout } from 'node:timers/promises';

await setTimeout(200);
```

반대로 `scheduler.yield()`는 시간을 늦추려는 목적보다 대기 중인 다른 작업에 차례를 넘기려는 목적에 가깝습니다.
그래서 레이트 리밋 대기, 재시도 backoff, 사용자에게 보이는 카운트다운에는 적합하지 않습니다.
실행 시간 자체를 측정하고 싶다면 [Node.js performance.now 가이드: 실행 시간을 정확하게 측정하는 법](/development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html)을 참고해 기준을 잡는 편이 좋습니다.

## 취소 가능한 배치 처리로 만들기

### H3. AbortSignal을 함께 받아 중단 지점을 만든다

긴 작업은 양보 지점뿐 아니라 취소 지점도 필요합니다.
요청이 종료됐거나 사용자가 작업을 취소했는데도 남은 항목을 계속 처리하면 리소스가 낭비됩니다.
`signal.throwIfAborted()`를 배치 경계에 배치하면 취소 시점을 명확하게 만들 수 있습니다.

```js
import { scheduler } from 'node:timers/promises';

export async function validateRecords(records, { signal, batchSize = 1000 } = {}) {
  const errors = [];

  for (let index = 0; index < records.length; index += 1) {
    signal?.throwIfAborted();

    const record = records[index];

    if (!record.email || !String(record.email).includes('@')) {
      errors.push({ index, field: 'email', message: 'invalid email' });
    }

    if ((index + 1) % batchSize === 0) {
      await scheduler.yield();
    }
  }

  return errors;
}
```

취소 체크를 매 항목마다 넣으면 가장 빠르게 멈출 수 있습니다.
항목당 계산이 가볍고 데이터가 매우 많다면 배치 경계에서만 확인해도 충분할 수 있습니다.
중요한 것은 취소 정책을 함수 인자로 드러내고, 호출자가 작업 수명주기를 제어할 수 있게 만드는 것입니다.

### H3. 타임아웃과 사용자 취소를 하나로 묶을 수 있다

서버 작업에서는 사용자 취소와 전체 시간 제한이 동시에 필요할 때가 많습니다.
이때 `AbortSignal.any()`로 여러 취소 조건을 하나의 신호로 묶을 수 있습니다.

```js
const controller = new AbortController();
const timeoutSignal = AbortSignal.timeout(3000);
const signal = AbortSignal.any([controller.signal, timeoutSignal]);

try {
  const errors = await validateRecords(records, {
    signal,
    batchSize: 2000,
  });

  console.log('validation errors:', errors.length);
} catch (error) {
  if (error.name === 'AbortError' || signal.aborted) {
    console.error('검증 작업이 취소되었습니다.');
  } else {
    throw error;
  }
}
```

취소 조건이 여러 개라면 [Node.js AbortSignal.any와 timeout 가이드: 여러 취소 조건을 하나로 묶는 법](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)처럼 상위 계층에서 신호를 조립하고, 실제 작업 함수는 하나의 `signal`만 받게 만드는 구조가 깔끔합니다.

## 배치 크기를 정하는 기준

### H3. 항목 수보다 항목당 계산 비용을 먼저 본다

`batchSize = 1000` 같은 숫자는 출발점일 뿐입니다.
간단한 문자열 정규화라면 5000개마다 양보해도 충분할 수 있고, 정규식 검사나 암호학적 계산처럼 무거운 작업이라면 100개마다 양보해야 할 수 있습니다.
배치 크기는 항목 수가 아니라 한 배치가 이벤트 루프를 붙잡는 시간을 기준으로 정하는 편이 좋습니다.

```js
import { performance } from 'node:perf_hooks';
import { scheduler } from 'node:timers/promises';

export async function processInBatches(items, handleItem, batchSize = 1000) {
  let batchStartedAt = performance.now();

  for (let index = 0; index < items.length; index += 1) {
    handleItem(items[index]);

    if ((index + 1) % batchSize === 0) {
      const elapsed = performance.now() - batchStartedAt;
      console.log('batch ms:', Math.round(elapsed));

      await scheduler.yield();
      batchStartedAt = performance.now();
    }
  }
}
```

운영 코드에서 모든 배치를 로그로 남길 필요는 없습니다.
초기 튜닝이나 부하 테스트에서만 측정하고, 적당한 기본값을 정한 뒤 설정으로 조정 가능하게 두면 됩니다.

### H3. p95 지연 시간과 전체 처리 시간을 함께 본다

배치 크기를 줄이면 이벤트 루프 지연은 줄어들 수 있지만 전체 처리 시간은 늘어날 수 있습니다.
반대로 배치 크기를 키우면 전체 처리량은 좋아져도 사용자 요청이 순간적으로 밀릴 수 있습니다.
따라서 한쪽 숫자만 보면 안 됩니다.

실무 점검 순서는 단순하게 잡을 수 있습니다.

- 작업 전체 소요 시간이 허용 범위 안에 있는지 확인합니다.
- 동시에 들어오는 요청의 p95 또는 p99 응답 시간이 악화되는지 확인합니다.
- 배치 크기를 바꿔 보며 처리량과 응답성의 균형점을 찾습니다.
- 작업이 계속 길어진다면 워커 스레드, 스트림, 큐 분리를 검토합니다.

테스트 자동화가 필요하다면 [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)처럼 내장 테스트 러너로 취소와 순서 보존을 검증할 수 있습니다.

## 실무 적용 패턴

### H3. 대량 데이터 변환 함수에 옵션으로 노출한다

라이브러리나 서비스 내부 함수라면 `batchSize`와 `signal`을 옵션으로 받는 형태가 유지보수에 좋습니다.
기본값은 안전하게 두고, 호출자가 트래픽 상황에 맞춰 조정할 수 있게 합니다.

```js
import { scheduler } from 'node:timers/promises';

export async function mapLargeArray(items, mapper, options = {}) {
  const { batchSize = 1000, signal } = options;
  const mapped = [];

  for (let index = 0; index < items.length; index += 1) {
    signal?.throwIfAborted();
    mapped.push(mapper(items[index], index));

    if ((index + 1) % batchSize === 0) {
      await scheduler.yield();
    }
  }

  return mapped;
}
```

이 함수는 일반 `Array.prototype.map()`보다 느릴 수 있습니다.
대신 매우 큰 배열을 처리할 때 이벤트 루프 독점을 줄일 수 있습니다.
작은 배열까지 모두 이 함수로 바꿀 필요는 없고, 실제로 문제가 되는 대량 처리 경로에만 적용하는 편이 낫습니다.

### H3. 스트림으로 바꿔야 할 작업과 구분한다

`scheduler.yield()`는 이미 메모리에 올라온 큰 배열을 다룰 때 특히 간단합니다.
하지만 파일을 처음부터 끝까지 한 번에 읽고, 그 결과를 다시 큰 배열로 만든 뒤 배치 처리한다면 병목은 다른 곳에 있을 수 있습니다.
큰 파일을 다룬다면 [Node.js fs.readFile AbortSignal 가이드: 파일 읽기를 안전하게 취소하는 법](/development/blog/seo/2026/05/19/nodejs-fs-readfile-abortsignal-cancel-file-read-guide.html)의 기준처럼 `readFile()`이 아니라 스트림 전환을 먼저 검토해야 합니다.

정리하면 다음처럼 나눌 수 있습니다.

- 이미 메모리에 있는 큰 배열을 변환한다면 `scheduler.yield()`가 간단합니다.
- 파일이나 네트워크 입력이 큰 경우에는 스트림과 backpressure가 우선입니다.
- CPU 계산이 무겁고 오래 걸리면 워커 스레드나 작업 큐를 검토합니다.
- 취소와 시간 제한은 어떤 방식이든 상위 작업 수명주기와 연결합니다.

## 자주 하는 실수

### H3. 매 반복마다 yield를 호출한다

가장 흔한 실수는 모든 항목 뒤에 `await scheduler.yield()`를 넣는 것입니다.
이러면 이벤트 루프에는 자주 양보하지만 전체 처리 시간이 불필요하게 길어집니다.
대부분의 경우에는 일정 개수마다 한 번씩 호출하는 배치 방식이 더 낫습니다.

```js
// 권장하지 않는 패턴
for (const item of items) {
  handleItem(item);
  await scheduler.yield();
}
```

양보는 비용이 없는 마법이 아닙니다.
응답성을 얻기 위해 적당한 비용을 지불하는 도구에 가깝습니다.
그래서 측정 없이 모든 반복문에 넣는 방식은 피해야 합니다.

### H3. CPU 병렬 처리 도구로 오해한다

`scheduler.yield()`는 CPU 작업을 다른 코어로 보내지 않습니다.
무거운 계산을 잘게 나눠도 총 계산량은 그대로이고, 같은 메인 스레드에서 실행됩니다.
이미 이벤트 루프 지연이 심각하고 작업 하나가 수백 밀리초 이상 CPU를 점유한다면 워커 스레드나 별도 프로세스가 더 적합할 수 있습니다.

이 도구의 역할은 명확합니다.
긴 동기 구간을 협력적으로 끊어서 Node.js 프로세스가 다른 이벤트도 처리할 수 있게 만드는 것입니다.
그 이상을 기대하면 병목이 숨겨지고 장애 대응이 늦어질 수 있습니다.

## FAQ

### H3. scheduler.yield는 setTimeout 0과 같은가요?

목적이 다릅니다.
`setTimeout(0)`은 타이머 큐를 통해 나중에 실행되도록 예약하는 방식이고, `scheduler.yield()`는 스케줄러에 실행권을 양보한다는 의도를 직접 표현합니다.
정확한 내부 단계보다 중요한 것은 코드 의미입니다.
시간 지연이 목적이면 `setTimeout`, 긴 작업 중 양보가 목적이면 `scheduler.yield()`를 우선 검토하면 됩니다.

### H3. 모든 대량 반복문에 넣어야 하나요?

아닙니다.
반복 횟수가 적거나 항목당 계산이 매우 가벼운 경우에는 오히려 불필요한 비용만 늘 수 있습니다.
응답 지연이 관측되는 경로, 큰 입력이 들어오는 경로, 관리자 배치 작업처럼 이벤트 루프를 오래 붙잡을 가능성이 있는 곳에 제한적으로 적용하는 것이 좋습니다.

### H3. scheduler.yield를 쓰면 서버가 빨라지나요?

전체 계산량을 줄이지는 않기 때문에 작업 자체가 빨라진다고 보기는 어렵습니다.
대신 한 작업이 이벤트 루프를 독점하는 시간을 줄여 다른 요청의 대기 시간을 낮출 수 있습니다.
즉 처리량 최적화 도구라기보다 응답성 관리 도구에 가깝습니다.

## 마무리

Node.js `scheduler.yield()`는 긴 반복 작업을 안전하게 쪼개는 작은 API입니다.
복잡한 큐나 워커 스레드를 바로 도입하기 전에, 이미 메모리에 있는 데이터를 배치로 처리하고 중간중간 이벤트 루프에 실행권을 돌려주는 데 잘 맞습니다.

실무에서는 세 가지를 함께 지키면 됩니다.
첫째, 매 항목이 아니라 배치 경계에서 양보합니다.
둘째, `AbortSignal`을 받아 취소 가능한 작업으로 만듭니다.
셋째, `performance.now()`나 부하 테스트로 배치 크기가 응답성과 전체 처리 시간에 어떤 영향을 주는지 확인합니다.
이 기준을 지키면 `scheduler.yield()`는 대량 처리 코드의 응답성을 높이는 안전한 완충 장치가 될 수 있습니다.

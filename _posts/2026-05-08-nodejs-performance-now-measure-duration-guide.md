---
layout: post
title: "Node.js performance.now 가이드: 실행 시간을 정확하게 측정하는 법"
date: 2026-05-08 20:00:00 +0900
lang: ko
translation_key: nodejs-performance-now-measure-duration-guide
permalink: /development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html
alternates:
  ko: /development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html
  x_default: /development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, performance, perf_hooks, monitoring, backend]
description: "Node.js performance.now로 실행 시간을 안정적으로 측정하는 방법을 정리했습니다. Date.now와의 차이, API 응답 시간 측정, 비동기 작업 계측, 운영 코드 주의사항까지 실무 예제로 설명합니다."
---

성능 개선을 시작할 때 가장 먼저 필요한 것은 추측이 아니라 측정입니다.
API가 느린지, 데이터베이스 호출이 느린지, 파일 처리나 직렬화가 병목인지 알아야 올바른 위치를 고칠 수 있습니다.

Node.js에서 짧은 실행 시간을 잴 때는 `Date.now()`보다 `performance.now()`를 우선 고려하는 편이 좋습니다.
핵심은 **벽시계 시간이 아니라 단조 증가하는 고해상도 시간 기준으로 duration을 계산할 수 있다**는 점입니다.

## performance.now가 필요한 이유

### H3. Date.now는 시각 확인용에 가깝다

`Date.now()`는 현재 Unix timestamp를 밀리초 단위로 반환합니다.
로그에 이벤트가 발생한 시각을 남기기에는 편하지만, 코드 실행 시간을 정밀하게 재는 용도로는 아쉬운 점이 있습니다.

```js
const startedAt = Date.now();

await doSomething();

const durationMs = Date.now() - startedAt;
console.log(durationMs);
```

이 방식은 단순하고 익숙하지만 시스템 시간이 조정되는 상황까지 고려하면 duration 측정 기준으로는 덜 적합합니다.
운영 환경에서는 NTP 동기화, 컨테이너 시간 보정, 서버 설정 변경처럼 벽시계 시간이 움직일 수 있는 요소가 있습니다.

반면 `performance.now()`는 특정 기준점 이후 흐른 시간을 반환합니다.
실제 날짜와 시각을 표현하려는 API가 아니라, 두 지점 사이의 경과 시간을 계산하기 위한 API입니다.

### H3. 짧은 작업일수록 측정 기준이 중요하다

API 전체 응답 시간이 2초라면 밀리초 단위 오차가 크게 문제 되지 않을 수 있습니다.
하지만 캐시 조회, JSON 직렬화, 짧은 유틸 함수, 미들웨어 한 단계처럼 작은 구간을 비교할 때는 더 안정적인 측정 기준이 필요합니다.

성능 측정은 숫자를 보는 일이지만, 숫자 자체보다 중요한 것은 일관성입니다.
동일한 기준으로 반복 측정해야 변경 전후를 비교할 수 있습니다.

## performance.now 기본 사용법

### H3. node:perf_hooks에서 가져오기

Node.js에서는 `node:perf_hooks` 모듈의 `performance` 객체를 사용할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

const start = performance.now();

await runTask();

const durationMs = performance.now() - start;
console.log(`duration=${durationMs.toFixed(2)}ms`);
```

반환값은 밀리초 단위 숫자입니다.
소수점 이하 값이 포함될 수 있으므로 화면 출력이나 로그 저장 시에는 필요한 자리수로 정리하는 편이 읽기 쉽습니다.

CommonJS 프로젝트라면 아래처럼 사용할 수 있습니다.

```js
const { performance } = require('node:perf_hooks');

const start = performance.now();
// ...
console.log(performance.now() - start);
```

### H3. 측정 헬퍼로 반복 코드를 줄이기

여러 곳에서 같은 방식으로 시간을 재야 한다면 작은 헬퍼를 만들어 둘 수 있습니다.

```js
import { performance } from 'node:perf_hooks';

export async function measureAsync(label, fn) {
  const start = performance.now();

  try {
    return await fn();
  } finally {
    const durationMs = performance.now() - start;
    console.log(`${label} took ${durationMs.toFixed(2)}ms`);
  }
}

await measureAsync('load-user-profile', async () => {
  return loadUserProfile('user_123');
});
```

`finally`에서 기록하면 작업이 성공하든 실패하든 duration을 남길 수 있습니다.
실패한 요청의 시간이 더 중요한 경우도 많기 때문에, 관측 코드는 정상 흐름에만 붙이지 않는 것이 좋습니다.

오류를 감싸서 원인을 보존하는 패턴은 [Node.js Error cause 가이드: 감싼 에러의 원인을 안전하게 남기는 법](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)과 함께 적용할 수 있습니다.

## 실무에서 자주 쓰는 측정 패턴

### H3. API 요청 처리 시간 기록하기

HTTP 서버에서는 요청이 들어온 시점과 응답이 끝난 시점의 차이를 기록하면 기본적인 latency 로그를 만들 수 있습니다.

```js
import http from 'node:http';
import { performance } from 'node:perf_hooks';

const server = http.createServer(async (req, res) => {
  const start = performance.now();

  res.on('finish', () => {
    const durationMs = performance.now() - start;
    console.log(JSON.stringify({
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      durationMs: Number(durationMs.toFixed(2)),
    }));
  });

  res.setHeader('content-type', 'application/json');
  res.end(JSON.stringify({ ok: true }));
});

server.listen(3000);
```

여기서 `finish` 이벤트를 쓰면 응답이 실제로 마무리된 시점에 가까운 값을 기록할 수 있습니다.
요청 시작 직후에 시간을 계산하면 비즈니스 로직만 일부 측정하고 네트워크 응답 흐름을 놓칠 수 있습니다.

요청 단위로 추적 ID를 함께 남기려면 [Node.js AsyncLocalStorage 가이드: 요청 컨텍스트 로깅을 안전하게 관리하는 법](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)과 연결해서 설계하면 좋습니다.

### H3. 외부 API 호출 구간만 따로 측정하기

전체 요청 시간이 느릴 때는 외부 API, 데이터베이스, 파일 시스템 같은 구간별 duration을 따로 봐야 합니다.

```js
import { performance } from 'node:perf_hooks';

async function fetchJson(url) {
  const start = performance.now();

  try {
    const response = await fetch(url);

    if (!response.ok) {
      throw new Error(`unexpected status: ${response.status}`);
    }

    return await response.json();
  } finally {
    const durationMs = performance.now() - start;
    console.log(`fetch ${url} ${durationMs.toFixed(2)}ms`);
  }
}
```

다만 운영 로그에 URL을 남길 때는 주의해야 합니다.
쿼리 문자열에 토큰, 이메일, 검색어, 내부 식별자처럼 민감한 값이 섞일 수 있기 때문입니다.
로그에는 전체 URL보다 호스트, 경로, 사전에 허용한 파라미터만 남기는 편이 안전합니다.

쿼리 문자열을 안전하게 조립하고 다루는 방법은 [Node.js URLSearchParams 가이드: 쿼리 문자열을 안전하게 조립하고 읽는 법](/development/blog/seo/2026/05/07/nodejs-urlsearchparams-query-string-handling-guide.html)에서 더 자세히 정리했습니다.

### H3. 작업 큐나 배치 처리 시간 측정하기

백그라운드 작업에서는 전체 처리 시간뿐 아니라 항목별 처리 시간을 함께 남기면 병목을 찾기 쉽습니다.

```js
import { performance } from 'node:perf_hooks';

async function processBatch(items) {
  const batchStart = performance.now();

  for (const item of items) {
    const itemStart = performance.now();

    await processItem(item);

    const itemDurationMs = performance.now() - itemStart;
    console.log(`item=${item.id} duration=${itemDurationMs.toFixed(2)}ms`);
  }

  const batchDurationMs = performance.now() - batchStart;
  console.log(`batch duration=${batchDurationMs.toFixed(2)}ms`);
}
```

이 방식은 느린 항목이 일부인지, 전체 처리 구조가 느린지 구분하는 데 도움이 됩니다.
다만 항목 수가 많다면 모든 항목을 로그로 남기기보다 샘플링하거나 느린 항목만 기록하는 전략이 필요합니다.

실패 작업을 따로 보관하고 재처리하는 흐름은 [Node.js Dead Letter Queue 가이드: 실패한 작업을 안전하게 복구하는 법](/development/blog/seo/2026/04/02/nodejs-dead-letter-queue-dlq-failure-recovery-guide.html)과 함께 보면 좋습니다.

## performance.now 사용 시 주의할 점

### H3. 한 번 측정한 값으로 결론 내리지 않는다

성능 숫자는 실행 환경에 영향을 많이 받습니다.
첫 실행에는 모듈 로딩, JIT 최적화, 캐시 상태, 네트워크 지연이 섞일 수 있습니다.
그래서 작은 코드 조각을 비교할 때는 여러 번 반복하고 중앙값이나 분위수 기준으로 보는 편이 안전합니다.

```js
import { performance } from 'node:perf_hooks';

function runMany(times, fn) {
  const durations = [];

  for (let i = 0; i < times; i += 1) {
    const start = performance.now();
    fn();
    durations.push(performance.now() - start);
  }

  return durations.sort((a, b) => a - b);
}
```

단, 직접 만든 간단한 벤치마크는 방향을 보는 용도에 가깝습니다.
정확한 마이크로 벤치마크가 필요하다면 전용 도구와 충분한 반복, 워밍업, 환경 고정이 필요합니다.

### H3. 측정 코드가 서비스 로직을 오염시키지 않게 한다

성능 측정을 위해 로그를 많이 추가하다 보면 비즈니스 로직이 지저분해질 수 있습니다.
처음에는 직접 `performance.now()`를 넣어도 되지만, 반복되는 지점은 미들웨어, 래퍼 함수, 공통 관측 모듈로 분리하는 편이 좋습니다.

관측 코드를 라이브러리처럼 분리해야 한다면 [Node.js diagnostics_channel 가이드: 관측 코드를 안전하게 분리하는 법](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)을 함께 참고할 수 있습니다.

### H3. duration과 deadline은 다르다

`performance.now()`는 시간이 얼마나 걸렸는지 측정하는 데 적합합니다.
하지만 작업을 언제 중단할지, 남은 시간 예산이 얼마인지 판단하는 로직은 별도 timeout/deadline 설계가 필요합니다.

예를 들어 외부 API 호출 시간이 900ms였다는 사실과, 전체 요청 예산 1초 안에서 더 이상 재시도하면 안 된다는 판단은 다른 문제입니다.
시간 예산을 전파하는 설계는 [Node.js Timeout Budget 가이드: 요청 시간 예산을 끝까지 지키는 법](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)과 연결해서 보는 것이 좋습니다.

## 적용 체크리스트

### H3. 운영 코드에 넣기 전에 확인할 것

1. 측정하려는 값이 시각인가, 경과 시간인가?
2. 요청 전체 시간과 하위 구간 시간을 구분했는가?
3. 실패한 작업도 duration을 남기는가?
4. 로그에 URL, 토큰, 개인정보가 그대로 들어가지 않는가?
5. 한 번의 측정값이 아니라 반복 측정과 추세로 판단하는가?

이 다섯 가지를 확인하면 단순한 타이머 로그를 더 믿을 수 있는 성능 관측 데이터로 바꿀 수 있습니다.

## 마무리

`performance.now()`는 Node.js에서 실행 시간을 안정적으로 측정하기 위한 기본 도구입니다.
`Date.now()`가 현재 시각을 남기는 데 유용하다면, `performance.now()`는 두 지점 사이의 duration을 계산하는 데 더 잘 맞습니다.

정리하면 이렇게 기억하면 됩니다.

- 실행 시간 측정에는 `performance.now()`를 우선 사용한다
- 요청 전체와 하위 구간을 나눠 병목을 찾는다
- 로그에는 duration뿐 아니라 상태 코드, 작업명, 추적 ID 같은 문맥을 함께 남긴다
- 민감한 URL과 입력값은 측정 로그에 그대로 남기지 않는다

## 함께 보면 좋은 글

- [Node.js AsyncLocalStorage 가이드: 요청 컨텍스트 로깅을 안전하게 관리하는 법](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)
- [Node.js diagnostics_channel 가이드: 관측 코드를 안전하게 분리하는 법](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)
- [Node.js Timeout Budget 가이드: 요청 시간 예산을 끝까지 지키는 법](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)

---
layout: post
title: "Node.js timers/promises setTimeout 가이드: sleep, backoff, 재시도를 깔끔하게 구현하는 법"
date: 2026-04-28 20:00:00 +0900
lang: ko
translation_key: nodejs-timers-promises-settimeout-delay-retry-guide
permalink: /development/blog/seo/2026/04/28/nodejs-timers-promises-settimeout-delay-retry-guide.html
alternates:
  ko: /development/blog/seo/2026/04/28/nodejs-timers-promises-settimeout-delay-retry-guide.html
  x_default: /development/blog/seo/2026/04/28/nodejs-timers-promises-settimeout-delay-retry-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, timers-promises, settimeout, sleep, retry, backoff, javascript, backend]
description: "Node.js timers/promises의 setTimeout으로 sleep, backoff, 재시도 로직을 더 읽기 좋게 구현하는 방법과 AbortSignal 기반 취소 패턴을 실무 예제로 정리했습니다."
---

Node.js에서 잠깐 기다리는 로직은 생각보다 자주 나옵니다.
외부 API 재시도, rate limit 회피, polling 간격 제어, 장애 복구 대기처럼 단순해 보이는 흐름이 실제 운영 안정성과 직결되기 때문입니다.

예전에는 `new Promise(resolve => setTimeout(resolve, ms))`를 매번 직접 감싸는 패턴이 흔했지만, 지금은 `node:timers/promises`의 `setTimeout()`으로 더 읽기 좋은 비동기 지연 코드를 만들 수 있습니다.
이 글에서는 **Node.js timers/promises setTimeout을 이용해 sleep, backoff, 재시도, 취소 처리까지 실무적으로 정리하는 방법**을 다룹니다.

## 왜 timers/promises setTimeout이 실무에서 유용한가

### H3. 지연 로직을 Promise 친화적으로 바로 표현할 수 있다

가장 큰 장점은 의도가 바로 드러난다는 점입니다.
기존 콜백 기반 `setTimeout()`은 Promise 흐름 안에서 쓸 때 매번 래핑 코드가 필요합니다.

```js
await new Promise((resolve) => setTimeout(resolve, 1000));
```

이 코드는 익숙하지만, 여러 파일에서 반복되면 잡음이 됩니다.
반면 `node:timers/promises`를 쓰면 아래처럼 바로 읽힙니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

await delay(1000);
```

이 차이는 작아 보여도 retry 루프, polling, 장애 복구 코드처럼 반복 구조가 많은 곳에서 유지보수성을 꽤 올려 줍니다.

### H3. value와 AbortSignal을 함께 다루기 좋다

`timers/promises`의 `setTimeout()`은 단순히 기다리는 것만이 아니라, resolve 값과 취소 신호도 함께 받을 수 있습니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

const result = await delay(500, 'done');
console.log(result); // done
```

여기에 `AbortSignal`을 붙이면 “필요 없어진 대기”를 안전하게 중단할 수 있습니다.
운영 코드에서는 성공 경로보다 취소·타임아웃·종료 신호 정리가 더 중요할 때가 많아서, 이 기능이 꽤 실용적입니다.

## 기본 사용법: sleep 유틸을 직접 만들지 않아도 된다

### H3. 가장 단순한 대기는 import alias만으로 충분하다

실무에서는 보통 `setTimeout` 이름이 기존 전역 함수와 겹치므로 alias를 붙입니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

async function main() {
  console.log('start');
  await delay(1500);
  console.log('1.5초 후 실행');
}

main();
```

이 패턴의 장점은 명확합니다.

- `delay()`라는 이름만 봐도 목적이 바로 보임
- 팀 내 공통 sleep 유틸을 따로 만들지 않아도 됨
- 래핑 보일러플레이트가 사라져 코드가 짧아짐

### H3. 지연 후 값을 넘기는 패턴은 retry 결과 처리에도 쓸 만하다

두 번째 인자로 resolve 값을 넘길 수 있어서 분기 처리도 단정해집니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

const status = await delay(300, 'retry');

if (status === 'retry') {
  console.log('다음 시도를 진행합니다.');
}
```

이 기능은 반드시 필요한 건 아니지만, 상태 값을 다시 조립하지 않아도 돼서 간단한 상태 머신이나 테스트 코드에서 특히 편합니다.

## 재시도와 backoff를 구현하는 실무 패턴

### H3. 고정 간격 재시도는 가장 먼저 읽기 좋게 정리할 수 있다

외부 API가 일시적으로 실패하는 상황에서는 아래처럼 고정 간격 재시도가 흔합니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

async function fetchWithRetry(fetcher, maxRetries = 3) {
  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt += 1) {
    try {
      return await fetcher();
    } catch (error) {
      lastError = error;

      if (attempt === maxRetries) {
        break;
      }

      await delay(1000);
    }
  }

  throw lastError;
}
```

여기서 중요한 건 retry 자체보다도 **실패 후 대기 의도가 한 줄로 분리되어 읽힌다**는 점입니다.
이런 구조는 에러 로그, 메트릭, 시도 횟수 제한을 추가할 때도 덜 엉킵니다.

### H3. 지수 backoff는 과도한 재시도를 줄이는 기본값이 된다

운영 환경에서는 모든 재시도를 1초 고정으로 두기보다 점점 간격을 늘리는 편이 안전합니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

async function fetchWithExponentialBackoff(fetcher, maxRetries = 5) {
  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt += 1) {
    try {
      return await fetcher();
    } catch (error) {
      lastError = error;

      if (attempt === maxRetries) {
        break;
      }

      const waitMs = Math.min(1000 * 2 ** (attempt - 1), 10000);
      await delay(waitMs);
    }
  }

  throw lastError;
}
```

이 패턴은 다음 상황에서 특히 유용합니다.

- 일시적인 네트워크 오류
- upstream 429 또는 503 응답
- 배포 직후 의존 서비스가 아직 덜 올라온 상태
- cron 작업이 외부 시스템 회복을 잠깐 기다려야 하는 상황

단, 재시도는 만능 복구가 아닙니다.
실패 원인이 영구 오류인데 무작정 delay만 넣으면 장애 인지만 늦어질 수 있습니다.
그래서 재시도 조건과 종료 조건을 함께 두는 게 중요합니다.

## AbortSignal과 함께 쓰면 더 안전하다

### H3. 더 이상 필요 없는 대기는 취소 가능해야 한다

프로세스 종료, 요청 취소, 상위 timeout이 걸린 상황에서도 sleep이 끝날 때까지 기다리면 shutdown이 지저분해질 수 있습니다.
이럴 때 `signal` 옵션을 함께 넘기면 대기 자체를 중단할 수 있습니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

const controller = new AbortController();

setTimeout(() => controller.abort(), 2000);

try {
  await delay(5000, null, { signal: controller.signal });
} catch (error) {
  if (error.name === 'AbortError') {
    console.log('대기가 취소되었습니다.');
  } else {
    throw error;
  }
}
```

이 패턴은 긴 polling 루프나 graceful shutdown 처리에서 꽤 중요합니다.
대기 중인 작업도 시스템의 종료 계약을 따라야 하기 때문입니다.

### H3. 상위 timeout과 결합하면 hanging 방지에 도움이 된다

재시도 루프를 만들 때는 각 시도 자체의 timeout과 시도 사이의 delay를 함께 설계해야 합니다.
이때 취소 신호를 한 방향으로 묶어 두면 구조가 훨씬 단정해집니다.

이 관점은 [Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)와 바로 연결됩니다.
요청 취소, 전체 작업 timeout, 운영자 중단 신호가 동시에 존재할 수 있기 때문에, delay도 예외 없이 취소 가능해야 합니다.

## polling과 배치 작업에서의 활용 포인트

### H3. polling은 성공 조건과 종료 조건을 같이 둬야 한다

주기적으로 상태를 확인하는 로직도 `delay()`와 잘 맞습니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

async function waitUntilReady(check, { intervalMs = 2000, maxAttempts = 10 } = {}) {
  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    const ready = await check();

    if (ready) {
      return true;
    }

    if (attempt < maxAttempts) {
      await delay(intervalMs);
    }
  }

  return false;
}
```

이때 꼭 점검할 건 세 가지입니다.

- 성공 조건이 명확한가
- 최대 시도 횟수나 총 대기 시간이 있는가
- 취소 신호를 상위에서 주입할 수 있는가

이 세 가지가 빠지면 polling은 금방 무한 대기 코드가 됩니다.

### H3. batch 간격 제어는 Promise.allSettled와 같이 보면 좋다

여러 작업을 한꺼번에 처리하되 일부 실패를 허용하는 배치에서는, 배치 사이 대기와 결과 수집을 함께 보는 편이 좋습니다.
예를 들어 한 묶음 실행 후 2초 쉬고 다음 묶음을 보내는 식입니다.

이런 설계는 [Node.js Promise.allSettled 가이드](/development/blog/seo/2026/04/24/nodejs-promise-allsettled-partial-failure-handling-guide.html)와 조합할 때 특히 실용적입니다.
실패를 버리지 않고 수집하면서도, upstream에 과한 부하를 주지 않는 리듬을 만들 수 있기 때문입니다.

## 사용할 때 주의할 점

### H3. sleep이 비즈니스 문제를 가리는 핑계가 되면 안 된다

지연 로직은 쉽게 넣을 수 있지만, 그렇다고 모든 문제를 “일단 1초 기다리자”로 덮으면 안 됩니다.
대표적으로 아래 경우는 원인 분석이 먼저입니다.

- race condition 때문에 우연히 기다리면 통과하는 경우
- 데이터 정합성 보장이 없어서 타이밍 운에 기대는 경우
- readiness가 없는데 임의 대기로 대신하는 경우

즉 `delay()`는 제어 도구이지, 구조적 문제를 숨기는 해결책은 아닙니다.

### H3. retry 간격은 고정값보다 정책으로 분리하는 편이 낫다

처음에는 `await delay(1000)` 한 줄로 충분하지만, 운영 코드가 길어지면 지연 정책을 함수로 분리하는 편이 좋습니다.
예를 들면 시도 횟수에 따른 backoff, 최대 대기 시간, jitter 적용 여부를 별도 함수로 관리하는 식입니다.

이렇게 해 두면 테스트도 쉬워지고, API별 특성에 맞게 조정하기도 편합니다.
`Promise.withResolvers()`처럼 비동기 흐름을 조립하는 도구와 함께 쓰면 더 읽기 좋은 제어 코드로 정리할 수 있습니다.
관련해서는 [Node.js Promise.withResolvers 가이드](/development/blog/seo/2026/04/27/nodejs-promise-withresolvers-deferred-pattern-guide.html)를 같이 보면 도움이 됩니다.

## 마무리

`node:timers/promises`의 `setTimeout()`은 작은 유틸처럼 보이지만, 실무에서는 retry, backoff, polling, graceful shutdown 같은 핵심 흐름을 더 읽기 좋게 정리하게 해 줍니다.
특히 **대기 로직을 Promise 흐름 안에서 자연스럽게 표현하고, AbortSignal로 취소까지 연결할 수 있다는 점**이 큰 장점입니다.

정리하면, Node.js에서 sleep 유틸을 반복 생성하고 있었다면 이제는 `timers/promises`를 기본 선택지로 두는 편이 낫습니다.
다만 delay는 어디까지나 제어 수단이므로, readiness·timeout·재시도 정책을 함께 설계해야 운영 코드가 안전해집니다.

관련 글:

- [Node.js AbortSignal.any 가이드: 여러 취소 신호를 한 번에 묶어 안전하게 다루는 법](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js Promise.allSettled 가이드: 일부 실패가 있어도 전체 작업을 안정적으로 처리하는 법](/development/blog/seo/2026/04/24/nodejs-promise-allsettled-partial-failure-handling-guide.html)
- [Node.js Promise.withResolvers 가이드: deferred 패턴을 더 깔끔하게 다루는 법](/development/blog/seo/2026/04/27/nodejs-promise-withresolvers-deferred-pattern-guide.html)

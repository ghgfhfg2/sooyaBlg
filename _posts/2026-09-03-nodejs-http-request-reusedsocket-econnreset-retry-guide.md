---
layout: post
title: "Node.js request.reusedSocket 가이드: keep-alive ECONNRESET을 안전하게 재시도하는 법"
date: 2026-09-03 08:00:00 +0900
lang: ko
translation_key: nodejs-http-request-reusedsocket-econnreset-retry-guide
permalink: /development/blog/seo/2026/09/03/nodejs-http-request-reusedsocket-econnreset-retry-guide.html
alternates:
  ko: /development/blog/seo/2026/09/03/nodejs-http-request-reusedsocket-econnreset-retry-guide.html
  x_default: /development/blog/seo/2026/09/03/nodejs-http-request-reusedsocket-econnreset-retry-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, http, keep-alive, reusedsocket, econnreset, retry, agent, backend]
description: "Node.js http.ClientRequest의 request.reusedSocket으로 재사용 연결의 ECONNRESET을 구분하고, 멱등성·재시도 횟수·백오프를 반영해 안전하게 복구하는 방법을 설명합니다."
---

HTTP keep-alive는 요청마다 새 TCP 연결을 만드는 비용을 줄여 줍니다.
하지만 클라이언트가 풀에 남아 있는 소켓을 재사용하는 순간 서버가 그 연결을 먼저 닫으면 `ECONNRESET`이 발생할 수 있습니다.
같은 오류 코드라도 새 연결 실패와 유휴 연결 재사용 경쟁은 대응 방식이 다릅니다.

Node.js `http.ClientRequest`의 `request.reusedSocket`은 현재 요청이 재사용된 소켓을 사용했는지 알려 줍니다.
이를 오류 코드와 함께 확인하면 keep-alive 경계에서 발생한 실패만 제한적으로 재시도할 수 있습니다.

이 글에서는 `request.reusedSocket`의 의미, 안전한 재시도 조건, 멱등성 제한, 백오프와 관측성 기준을 실무 코드로 정리합니다.
서버 쪽 keep-alive 설정은 [Node.js keepAliveTimeoutBuffer 가이드](/development/blog/seo/2026/08/17/nodejs-server-keepalivetimeoutbuffer-econnreset-guide.html), 일반적인 오류 분류는 [Node.js 시스템 오류 코드 가이드](/development/blog/seo/2026/08/05-nodejs-system-error-code-handling-guide.html), 외부 API 재시도 정책은 [Node.js fetch timeout 재시도 가이드](/development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js HTTP 공식 문서](https://nodejs.org/api/http.html#requestreusedsocket), [Node.js Errors 공식 문서](https://nodejs.org/api/errors.html#common-system-errors)

## request.reusedSocket이 필요한 이유

### keep-alive 소켓에는 작은 경쟁 구간이 있다

keep-alive를 사용하는 클라이언트와 서버는 연결을 영원히 유지하지 않습니다.
서버는 일정 시간 요청이 없는 연결을 닫고, 클라이언트 에이전트는 사용 가능한 소켓을 풀에 보관했다가 다음 요청에 재사용합니다.

경계 시점에는 다음 순서가 생길 수 있습니다.

1. 클라이언트 풀에는 소켓이 사용 가능한 것으로 남아 있습니다.
2. 서버는 keep-alive 제한에 따라 해당 연결을 닫습니다.
3. 클라이언트가 거의 동시에 새 요청을 그 소켓에 씁니다.
4. 요청은 `ECONNRESET`으로 실패합니다.

이 실패는 애플리케이션 응답이 500을 반환한 상황과 다릅니다.
또한 DNS 실패, 연결 거부, 새 소켓의 네트워크 단절과도 구분할 필요가 있습니다.

### reusedSocket은 재사용 여부를 알려 주는 신호다

`request.reusedSocket`은 해당 요청에 에이전트의 재사용 소켓이 할당되었다면 `true`입니다.
새 소켓을 사용했다면 `false`입니다.

```js
import http from 'node:http';

const agent = new http.Agent({ keepAlive: true });

const request = http.get(
  'http://127.0.0.1:3000/health',
  { agent },
  (response) => {
    console.log({
      statusCode: response.statusCode,
      reusedSocket: request.reusedSocket
    });

    response.resume();
  }
);

request.on('error', (error) => {
  console.error({
    code: error.code,
    reusedSocket: request.reusedSocket
  });
});
```

이 값은 원인을 완전히 증명하는 진단 결과가 아니라 재시도 판단에 쓸 수 있는 맥락입니다.
따라서 `reusedSocket === true`만 보고 재시도하지 말고 오류 코드, HTTP 메서드, 시도 횟수를 함께 확인해야 합니다.

## 가장 작은 안전한 재시도 구현

### 조건을 명시적으로 좁힌다

keep-alive 경쟁을 겨냥한 기본 조건은 다음처럼 잡을 수 있습니다.

- 오류 코드가 `ECONNRESET`이다.
- 실패한 요청이 재사용 소켓을 사용했다.
- 요청 메서드가 재실행해도 안전한 범위에 있다.
- 미리 정한 최대 시도 횟수를 넘지 않았다.

아래 예제는 응답 본문을 JSON으로 읽는 GET 요청을 한 번만 재시도합니다.

```js
import http from 'node:http';

const agent = new http.Agent({
  keepAlive: true,
  maxSockets: 50,
  maxFreeSockets: 10
});

function readJson(response) {
  return new Promise((resolve, reject) => {
    const chunks = [];

    response.on('data', (chunk) => chunks.push(chunk));
    response.on('end', () => {
      try {
        resolve(JSON.parse(Buffer.concat(chunks).toString('utf8')));
      } catch (error) {
        reject(error);
      }
    });
    response.on('error', reject);
  });
}

export function getJson(url, { maxRetries = 1 } = {}) {
  return new Promise((resolve, reject) => {
    let retryCount = 0;

    const run = () => {
      const request = http.get(url, { agent }, async (response) => {
        try {
          const body = await readJson(response);

          resolve({
            statusCode: response.statusCode,
            headers: response.headers,
            body
          });
        } catch (error) {
          reject(error);
        }
      });

      request.on('error', (error) => {
        const canRetry =
          error.code === 'ECONNRESET' &&
          request.reusedSocket === true &&
          retryCount < maxRetries;

        if (!canRetry) {
          reject(error);
          return;
        }

        retryCount += 1;
        run();
      });
    };

    run();
  });
}
```

핵심은 모든 `ECONNRESET`을 같은 방식으로 처리하지 않는 것입니다.
새 연결에서 반복되는 reset까지 무조건 재시도하면 실제 장애를 가리고 대상 서버에 부하를 더할 수 있습니다.

### 응답 스트림 오류는 별도로 생각한다

요청 객체의 `'error'` 이벤트뿐 아니라 응답을 읽는 중에도 오류가 발생할 수 있습니다.
이미 응답 헤더나 본문 일부를 받은 뒤의 실패는 단순한 유휴 소켓 재사용 실패와 동일하지 않습니다.

특히 스트리밍 응답을 자동으로 재시도하면 소비자가 데이터 일부를 두 번 처리할 수 있습니다.
따라서 다음 두 단계를 구분하는 편이 안전합니다.

- 응답을 받기 전 요청 연결 실패: 제한적 자동 재시도 후보
- 응답을 받거나 본문을 소비한 뒤 실패: 업무 로직에 맞춘 복구 필요

본문 처리 중 오류를 재시도하려면 기존 결과를 폐기할 수 있는지, 범위 요청이나 체크포인트가 있는지부터 확인해야 합니다.

## POST 요청을 자동 재시도하면 위험한 이유

### 네트워크 오류는 서버 미처리를 뜻하지 않는다

클라이언트가 `ECONNRESET`을 받았다고 해서 서버가 요청을 처리하지 않았다고 단정할 수 없습니다.
요청 바이트가 서버에 도착해 결제나 주문 생성이 끝났지만 응답만 돌아오지 못했을 수도 있습니다.

따라서 다음 요청은 기본적으로 자동 재시도 대상에서 제외하는 편이 안전합니다.

- 결제 승인 또는 취소
- 주문 생성
- 메시지 발송
- 포인트 차감
- 외부 시스템에 상태 변화를 만드는 명령

HTTP 메서드 이름만으로 안전성을 판단하는 것도 충분하지 않습니다.
GET 엔드포인트가 상태를 바꾸는 잘못된 설계일 수 있고, POST 엔드포인트가 멱등성 키를 지원할 수도 있기 때문입니다.

### 상태 변경에는 멱등성 키를 사용한다

업스트림 API가 멱등성 키를 지원한다면 각 논리 요청에 고정된 키를 부여해 중복 처리를 막을 수 있습니다.
재시도할 때 새 키를 만들면 중복 방지 효과가 사라지므로 같은 키를 유지해야 합니다.

```js
import { randomUUID } from 'node:crypto';

const operation = {
  idempotencyKey: randomUUID(),
  payload: JSON.stringify({ itemId: 'item-42', quantity: 1 })
};

// 같은 논리 작업을 재시도할 때는 operation.idempotencyKey를 유지합니다.
```

멱등성 키는 로그에 남겨도 되는 무작위 작업 식별자로 설계하고, 인증 토큰이나 개인정보를 키에 넣지 않아야 합니다.
서버 역시 키별 처리 결과를 원자적으로 저장하고 일정 기간 같은 결과를 반환해야 합니다.

## 재시도 정책을 운영 수준으로 확장하기

### 횟수 제한과 짧은 지연을 둔다

한 번의 재사용 소켓 경쟁은 즉시 재시도만으로 회복될 수 있습니다.
하지만 대상 서버가 불안정할 때 호출자가 동시에 반복 요청하면 재시도 폭주가 생깁니다.

재시도 횟수가 둘 이상이라면 지수 백오프와 작은 무작위 지연을 적용하는 편이 좋습니다.

```js
import { setTimeout as delay } from 'node:timers/promises';

function retryDelay(attempt) {
  const baseMs = Math.min(50 * (2 ** attempt), 500);
  const jitterMs = Math.floor(Math.random() * 30);

  return baseMs + jitterMs;
}

await delay(retryDelay(1));
```

다만 이 글의 좁은 문제인 재사용 소켓 `ECONNRESET`에는 많은 재시도가 필요하지 않은 경우가 대부분입니다.
우선 한 번으로 제한하고, 지표를 확인한 뒤 정책을 조정하는 방식이 운영하기 쉽습니다.

### 전체 시간 예산을 함께 관리한다

각 시도에만 timeout을 두면 전체 호출 시간은 `시도 횟수 × timeout`만큼 늘어날 수 있습니다.
호출자에게 1초 안에 답해야 하는데 1초짜리 요청을 세 번 실행하는 정책은 시간 예산을 지키지 못합니다.

재시도 루프에는 다음 값을 함께 두세요.

- 전체 deadline
- 시도별 연결·응답 timeout
- 최대 재시도 횟수
- 다음 시도를 시작할 최소 잔여 시간

남은 시간이 부족하면 새 시도를 시작하지 않고 상위 호출자에게 오류를 전달해야 합니다.

### 에이전트를 요청마다 만들지 않는다

`request.reusedSocket`을 활용하려면 실제로 연결을 재사용하는 에이전트가 있어야 합니다.
요청마다 `new http.Agent()`를 만들고 즉시 버리면 소켓 풀의 이점을 얻기 어렵습니다.

```js
const agent = new http.Agent({ keepAlive: true });

// 같은 대상과 정책을 공유하는 요청에서 agent를 재사용합니다.
```

반대로 서로 다른 신뢰 경계나 프록시 설정을 하나의 에이전트에 무조건 합칠 필요는 없습니다.
대상 서비스, 인증 방식, 연결 정책을 기준으로 에이전트 수명과 공유 범위를 정해야 합니다.

프로세스를 종료할 때 더 이상 새 요청을 받지 않는다면 `agent.destroy()`로 열린 소켓을 정리할 수 있습니다.

## 관측성과 로그에 남길 값

### 오류 코드와 재사용 여부를 함께 기록한다

운영 로그에는 원문 요청 전체보다 판단에 필요한 구조화 필드를 남기는 편이 안전합니다.

```js
logger.warn({
  event: 'upstream_request_failed',
  upstream: 'catalog-api',
  method: 'GET',
  errorCode: error.code,
  reusedSocket: request.reusedSocket,
  attempt: retryCount + 1,
  willRetry: canRetry
});
```

권장 지표는 다음과 같습니다.

- 전체 외부 요청 수
- `ECONNRESET` 발생 수와 비율
- `reusedSocket`이 참인 reset 수
- 재시도 성공·실패 수
- 시도 횟수를 포함한 전체 지연 시간
- 대상 서비스와 HTTP 메서드별 분포

URL 전체, 쿼리 문자열, `Authorization`, 쿠키, 요청 본문은 로그에 그대로 남기지 않습니다.
필요하면 사전에 정한 낮은 카디널리티의 라우트 이름만 기록합니다.

### 재시도는 원인을 숨기지 않아야 한다

재시도가 성공하더라도 첫 실패를 관측할 수 있어야 합니다.
성공 응답만 집계하면 keep-alive 설정 불일치나 서버의 과도한 연결 종료를 발견하기 어렵습니다.

다만 단발성 복구를 매번 오류 알림으로 올리면 경보 피로가 생깁니다.
개별 이벤트는 지표로 집계하고, 비율이나 연속 실패가 기준을 넘을 때 경보를 보내는 방식이 적합합니다.

## 테스트 전략

### 정책 함수는 순수 함수로 분리한다

실제 TCP 타이밍을 매번 재현하기보다 재시도 판단을 먼저 순수 함수로 분리하면 경계 조건을 빠르게 검증할 수 있습니다.

```js
export function shouldRetryReusedSocket({
  errorCode,
  reusedSocket,
  method,
  retryCount,
  maxRetries
}) {
  const retryableMethods = new Set(['GET', 'HEAD', 'OPTIONS']);

  return errorCode === 'ECONNRESET' &&
    reusedSocket === true &&
    retryableMethods.has(method) &&
    retryCount < maxRetries;
}
```

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { shouldRetryReusedSocket } from './retry-policy.js';

test('retries GET reset on a reused socket', () => {
  assert.equal(shouldRetryReusedSocket({
    errorCode: 'ECONNRESET',
    reusedSocket: true,
    method: 'GET',
    retryCount: 0,
    maxRetries: 1
  }), true);
});

test('does not retry POST by default', () => {
  assert.equal(shouldRetryReusedSocket({
    errorCode: 'ECONNRESET',
    reusedSocket: true,
    method: 'POST',
    retryCount: 0,
    maxRetries: 1
  }), false);
});

test('stops at the retry limit', () => {
  assert.equal(shouldRetryReusedSocket({
    errorCode: 'ECONNRESET',
    reusedSocket: true,
    method: 'GET',
    retryCount: 1,
    maxRetries: 1
  }), false);
});
```

### 통합 테스트에서는 결과보다 경계를 검증한다

통합 테스트에서는 짧은 keep-alive 설정의 로컬 서버를 만들 수 있지만, 정확한 reset 타이밍은 운영체제와 런타임 상태에 따라 흔들릴 수 있습니다.
CI의 핵심 테스트를 우연한 타이밍에 의존시키지 않는 것이 좋습니다.

다음 경계를 각각 검증하세요.

- 정책 함수가 reset과 다른 오류를 구분하는가?
- 새 소켓 실패는 자동 재시도하지 않는가?
- 재사용 소켓 reset은 최대 횟수까지만 재시도하는가?
- POST는 명시적 멱등성 정책 없이는 제외되는가?
- 재시도 후 최종 오류가 호출자에게 보존되는가?
- 로그에 인증 값이나 요청 본문이 포함되지 않는가?

## 도입 전 체크리스트

### 코드 체크

- keep-alive 에이전트를 적절한 범위에서 재사용하는가?
- `error.code === 'ECONNRESET'`과 `request.reusedSocket`을 함께 확인하는가?
- 재시도 가능한 메서드와 작업을 명시했는가?
- 최대 횟수와 전체 시간 예산이 있는가?
- 응답을 일부 소비한 실패를 별도로 처리하는가?
- 최종 오류를 삼키지 않고 호출자에게 전달하는가?

### 운영 체크

- 재시도 전후 지연 시간을 측정하는가?
- 대상별 reset 비율과 재시도 성공률을 볼 수 있는가?
- 서버와 프록시의 keep-alive 설정을 함께 확인했는가?
- URL 쿼리, 인증 헤더, 쿠키, 본문을 로그에서 제외했는가?
- 비멱등 작업은 멱등성 키나 상태 조회 수단이 있는가?

## FAQ

### reusedSocket이 true면 항상 재시도해도 되나요?

아닙니다.
재사용 소켓이라는 사실만으로 실패 원인이나 작업의 안전성이 결정되지는 않습니다.
`ECONNRESET` 같은 구체적인 오류, 멱등성, 시도 횟수, 전체 deadline을 함께 확인해야 합니다.

### ECONNRESET이면 서버가 요청을 처리하지 않은 건가요?

그렇게 단정할 수 없습니다.
연결이 끊긴 시점에 따라 서버가 요청을 받지 못했을 수도 있고, 처리를 끝낸 뒤 응답 전달만 실패했을 수도 있습니다.
그래서 상태 변경 요청에는 자동 재시도보다 멱등성 키와 처리 상태 조회가 중요합니다.

### fetch에서도 request.reusedSocket을 확인할 수 있나요?

`request.reusedSocket`은 `node:http`의 `ClientRequest` 속성입니다.
전역 `fetch()`는 같은 형태의 `ClientRequest` 객체를 노출하지 않으므로 이 속성을 직접 확인하는 패턴을 그대로 적용할 수 없습니다.
fetch를 쓴다면 dispatcher 정책과 오류 분류를 해당 API 수준에서 설계해야 합니다.

### keep-alive를 끄면 문제가 해결되나요?

재사용 소켓 경계 문제는 줄겠지만 요청마다 연결을 새로 만드는 비용이 생깁니다.
대부분은 keep-alive를 끄기보다 클라이언트·서버 timeout 관계를 점검하고, 좁은 재시도 정책과 관측성을 추가하는 편이 낫습니다.

## 마무리

`request.reusedSocket`은 Node.js HTTP 클라이언트에서 keep-alive 재사용 여부를 확인하는 작은 신호입니다.
이 값을 `ECONNRESET`과 함께 사용하면 모든 네트워크 오류를 무차별 재시도하지 않고, 유휴 연결 경계에서 생긴 실패를 더 좁게 복구할 수 있습니다.

안전한 정책의 핵심은 세 가지입니다.
재시도 대상을 멱등 요청으로 제한하고, 횟수와 전체 시간 예산을 정하며, 첫 실패와 최종 결과를 모두 관측해야 합니다.
상태 변경 요청에는 멱등성 키를 추가하고 서버·프록시의 keep-alive 설정도 함께 점검하면 일시적 복구와 근본 원인 개선을 동시에 진행할 수 있습니다.

## 함께 읽기

- [Node.js server.keepAliveTimeoutBuffer 가이드: ECONNRESET을 줄이는 HTTP keep-alive 설정법](/development/blog/seo/2026/08/17/nodejs-server-keepalivetimeoutbuffer-econnreset-guide.html)
- [Node.js 시스템 오류 코드 가이드: errno와 code로 실패 원인을 분류하는 법](/development/blog/seo/2026/08/05-nodejs-system-error-code-handling-guide.html)
- [Node.js fetch timeout 재시도 가이드: 실패 원인을 분류하고 안전하게 다시 요청하는 법](/development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html)

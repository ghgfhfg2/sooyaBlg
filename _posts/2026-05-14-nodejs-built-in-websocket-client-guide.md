---
layout: post
title: "Node.js WebSocket 내장 클라이언트 가이드: ws 없이 실시간 연결 테스트하기"
date: 2026-05-14 08:00:00 +0900
lang: ko
translation_key: nodejs-built-in-websocket-client-guide
permalink: /development/blog/seo/2026/05/14/nodejs-built-in-websocket-client-guide.html
alternates:
  ko: /development/blog/seo/2026/05/14/nodejs-built-in-websocket-client-guide.html
  x_default: /development/blog/seo/2026/05/14/nodejs-built-in-websocket-client-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, websocket, realtime, javascript, backend, testing]
description: "Node.js 내장 WebSocket 클라이언트로 ws 패키지 없이 실시간 연결을 테스트하는 방법을 정리했습니다. 연결·메시지·종료 처리, 재연결 전략, 테스트 스크립트 작성 팁까지 실무 예제로 설명합니다."
---

실시간 알림, 채팅, 대시보드, 주문 상태 업데이트처럼 서버가 먼저 이벤트를 밀어줘야 하는 기능에서는 WebSocket을 자주 사용합니다.
예전에는 Node.js에서 WebSocket 클라이언트 스크립트를 만들 때 `ws` 같은 패키지를 거의 기본처럼 설치했습니다.
하지만 최근 Node.js 런타임에서는 브라우저와 비슷한 **내장 WebSocket 클라이언트**를 사용할 수 있어, 간단한 연결 점검이나 운영 전 검증 스크립트는 의존성 없이 작성할 수 있습니다.

이 글에서는 Node.js 내장 `WebSocket`으로 실시간 서버에 연결하고, 메시지를 보내고, 응답을 검증하고, 안전하게 종료하는 흐름을 정리합니다.
운영 서버 구현 전체를 대체하는 글은 아닙니다.
목표는 “짧은 진단 스크립트와 테스트 클라이언트를 더 가볍게 만드는 법”입니다.
HTTP 요청의 타임아웃과 실패 처리를 함께 다뤄야 한다면 [HTTP 요청 타임아웃 가이드: 느린 외부 API 때문에 서버가 멈추지 않게 하기](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)도 같이 참고하면 좋습니다.

## Node.js 내장 WebSocket을 쓰면 좋은 상황

### H3. 의존성 없는 연결 점검 스크립트를 만들 수 있다

WebSocket 서버를 운영하다 보면 “지금 실제로 연결이 되는지”를 빠르게 확인해야 할 때가 있습니다.
브라우저를 열고 화면을 조작하기에는 무겁고, 별도 패키지를 설치한 테스트 프로젝트를 만들기에는 과한 상황입니다.
이때 Node.js 내장 WebSocket은 다음 용도에 잘 맞습니다.

- 배포 직후 WebSocket 엔드포인트 연결 확인
- staging 서버의 인증 토큰 흐름 점검
- ping, subscribe, unsubscribe 같은 기본 프로토콜 테스트
- 장애 대응 중 최소 재현 스크립트 작성
- CI에서 짧게 실행하는 smoke test

예를 들어 다음처럼 아주 작은 스크립트로 연결 여부를 확인할 수 있습니다.

```js
// scripts/check-websocket.js
const url = process.env.WS_URL ?? 'wss://example.com/realtime';

const socket = new WebSocket(url);

socket.addEventListener('open', () => {
  console.log('connected');
  socket.send(JSON.stringify({ type: 'ping' }));
});

socket.addEventListener('message', (event) => {
  console.log('received:', event.data);
  socket.close(1000, 'check complete');
});

socket.addEventListener('error', (event) => {
  console.error('websocket error:', event.message ?? event.type);
  process.exitCode = 1;
});

socket.addEventListener('close', (event) => {
  console.log('closed:', event.code, event.reason);
});
```

실행은 단순합니다.

```bash
WS_URL=wss://api.example.com/realtime node scripts/check-websocket.js
```

패키지 설치 없이 Node.js만으로 실행할 수 있다는 점이 장점입니다.
다만 팀원이 같은 결과를 얻으려면 Node.js 버전 조건을 문서화해야 합니다.
최근 런타임 기능을 활용하는 글과 마찬가지로, README나 `package.json`의 `engines`에 최소 버전을 명확히 남기는 편이 안전합니다.
Node.js 버전 조건을 다루는 방식은 [Node.js TypeScript 타입 스트리핑 가이드: 빌드 없이 .ts 파일을 실행하는 법](/development/blog/seo/2026/05/13/nodejs-typescript-type-stripping-runtime-guide.html)에서도 같은 원칙으로 설명했습니다.

### H3. 서버 구현과 클라이언트 검증을 분리할 수 있다

WebSocket 장애는 서버 코드만 보고는 원인을 찾기 어렵습니다.
연결은 되는데 메시지가 오지 않는지, 인증 직후 끊기는지, 특정 구독 메시지에서만 실패하는지, 프록시가 업그레이드 요청을 막는지 확인해야 합니다.

내장 WebSocket 클라이언트로 짧은 검증 스크립트를 두면 서버 구현과 별개로 다음을 확인할 수 있습니다.

```js
const socket = new WebSocket('wss://example.com/realtime');

socket.addEventListener('open', () => {
  socket.send(JSON.stringify({
    type: 'subscribe',
    channel: 'deployments'
  }));
});

socket.addEventListener('message', (event) => {
  const payload = JSON.parse(event.data);

  if (payload.type === 'subscribed') {
    console.log('subscription ok');
    socket.close(1000, 'done');
  }
});
```

이런 스크립트는 배포 후 점검, 온콜 대응, 문서 예제에 모두 유용합니다.
프론트엔드 앱 전체를 띄우지 않아도 프로토콜 단위로 문제를 좁힐 수 있기 때문입니다.

## 기본 사용법: 연결, 메시지, 종료 처리

### H3. open 이벤트 이후에 메시지를 보낸다

WebSocket은 생성 직후 바로 통신 가능한 상태가 아닙니다.
연결이 열리기 전에 `send()`를 호출하면 예외나 실패 흐름이 생길 수 있습니다.
따라서 메시지는 `open` 이벤트 이후에 보내는 습관을 들이는 것이 좋습니다.

```js
const socket = new WebSocket('wss://example.com/realtime');

socket.addEventListener('open', () => {
  socket.send(JSON.stringify({ type: 'hello' }));
});
```

실무에서는 전송할 메시지를 함수로 분리하면 테스트하기 쉽습니다.

```js
function sendJson(socket, payload) {
  socket.send(JSON.stringify(payload));
}

socket.addEventListener('open', () => {
  sendJson(socket, { type: 'ping', requestId: crypto.randomUUID() });
});
```

요청 ID가 필요하다면 `crypto.randomUUID()`를 사용할 수 있습니다.
고유 ID 생성 원칙은 [Node.js crypto.randomUUID 가이드: 안전한 요청 ID와 추적 ID 만들기](/development/blog/seo/2026/05/08/nodejs-crypto-randomuuid-safe-id-generation-guide.html)를 참고하세요.

### H3. message 이벤트에서는 입력 검증을 먼저 한다

WebSocket 메시지는 문자열, 바이너리 데이터 등 다양한 형태로 들어올 수 있습니다.
서버가 JSON을 보낸다고 약속했더라도 클라이언트 검증 스크립트에서는 파싱 실패를 명확히 처리해야 합니다.

```js
function parseJsonMessage(data) {
  if (typeof data !== 'string') {
    throw new Error('expected text message');
  }

  return JSON.parse(data);
}

socket.addEventListener('message', (event) => {
  try {
    const message = parseJsonMessage(event.data);
    console.log('message type:', message.type);
  } catch (error) {
    console.error('invalid message:', error.message);
    socket.close(1003, 'invalid message');
    process.exitCode = 1;
  }
});
```

여기서 중요한 점은 “테스트 스크립트니까 대충 파싱한다”가 아니라, 테스트 스크립트일수록 실패 이유를 또렷하게 남기는 것입니다.
JSON 파싱 실패, 예상하지 못한 메시지 타입, 인증 실패 응답을 구분해 두면 장애 대응 시간이 줄어듭니다.

입력값을 예외 없이 점검하는 습관은 URL 검증에서도 동일합니다.
WebSocket URL을 사용자 입력이나 환경 변수에서 받는다면 [Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)을 함께 적용할 수 있습니다.

### H3. close 이벤트에서 종료 코드를 확인한다

WebSocket 연결은 정상 종료와 비정상 종료를 구분해야 합니다.
테스트 스크립트에서 아무 메시지나 받은 뒤 `process.exit(0)`으로 끝내면, 실제로는 서버가 오류 코드로 끊었는데도 성공처럼 보일 수 있습니다.

```js
let completed = false;

socket.addEventListener('message', (event) => {
  const message = JSON.parse(event.data);

  if (message.type === 'pong') {
    completed = true;
    socket.close(1000, 'pong received');
  }
});

socket.addEventListener('close', (event) => {
  if (!completed || event.code !== 1000) {
    console.error('websocket closed unexpectedly:', event.code, event.reason);
    process.exitCode = 1;
    return;
  }

  console.log('websocket check passed');
});
```

종료 코드를 확인하면 CI smoke test에서도 실패를 명확히 잡아낼 수 있습니다.
운영 프로세스의 종료 처리까지 함께 설계해야 한다면 [Node.js graceful shutdown 가이드: Kubernetes 무중단 종료를 위한 SIGTERM 처리](/development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html)를 같이 보면 좋습니다.

## 타임아웃과 취소: 영원히 기다리지 않게 만들기

### H3. 연결 대기 시간에 제한을 둔다

가장 흔한 실수는 WebSocket 연결이 열리지 않을 때 스크립트가 계속 기다리는 것입니다.
진단 스크립트는 실패도 빠르게 알려줘야 합니다.
간단한 타임아웃은 `setTimeout()`으로 만들 수 있습니다.

```js
const socket = new WebSocket(url);

const connectTimer = setTimeout(() => {
  console.error('websocket connect timeout');
  socket.close(1000, 'connect timeout');
  process.exitCode = 1;
}, 5000);

socket.addEventListener('open', () => {
  clearTimeout(connectTimer);
  socket.send(JSON.stringify({ type: 'ping' }));
});
```

Node.js의 `timers/promises`를 쓰면 비동기 흐름으로도 표현할 수 있지만, WebSocket 이벤트 모델에서는 위처럼 명시적인 타이머가 더 읽기 쉬울 때가 많습니다.
지연과 재시도 흐름은 [Node.js timers/promises setTimeout 가이드: 재시도 지연을 깔끔하게 다루는 법](/development/blog/seo/2026/04/28/nodejs-timers-promises-settimeout-delay-retry-guide.html)에서 더 자세히 다뤘습니다.

### H3. 응답 대기 시간도 따로 제한한다

연결이 열렸다고 해서 프로토콜 검증이 끝난 것은 아닙니다.
`ping`을 보냈으면 `pong`이 일정 시간 안에 와야 합니다.
구독 요청을 보냈으면 `subscribed` 응답이 와야 합니다.

```js
function failAfter(ms, socket, message) {
  return setTimeout(() => {
    console.error(message);
    socket.close(1000, message);
    process.exitCode = 1;
  }, ms);
}

socket.addEventListener('open', () => {
  const responseTimer = failAfter(3000, socket, 'pong timeout');

  socket.addEventListener('message', (event) => {
    const payload = JSON.parse(event.data);

    if (payload.type === 'pong') {
      clearTimeout(responseTimer);
      socket.close(1000, 'ok');
    }
  });

  socket.send(JSON.stringify({ type: 'ping' }));
});
```

타임아웃을 연결 단계와 응답 단계로 나누면 원인 분석이 쉬워집니다.
“서버에 아예 연결하지 못함”과 “연결은 됐지만 프로토콜 응답이 없음”은 대응 방법이 다르기 때문입니다.

## 재연결 전략: 무조건 반복하지 않기

### H3. smoke test와 상시 클라이언트의 재시도 정책은 다르다

배포 후 smoke test는 실패를 빨리 알려주는 것이 목적입니다.
반대로 상시 실행되는 내부 모니터링 클라이언트는 일시적인 네트워크 흔들림을 흡수해야 할 수도 있습니다.
두 상황을 같은 재연결 정책으로 처리하면 문제가 생깁니다.

smoke test에서는 보통 다음 정책이 적절합니다.

- 연결 실패 시 1~2회만 재시도한다.
- 전체 실행 시간 상한을 둔다.
- 마지막 실패 원인을 로그에 남긴다.
- 실패를 숨기지 않고 non-zero exit code로 끝낸다.

간단한 재시도 흐름은 다음처럼 작성할 수 있습니다.

```js
async function wait(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function runWithRetry(task, maxAttempts = 3) {
  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    try {
      return await task(attempt);
    } catch (error) {
      lastError = error;
      console.error(`attempt ${attempt} failed:`, error.message);

      if (attempt < maxAttempts) {
        await wait(500 * attempt);
      }
    }
  }

  throw lastError;
}
```

실무 API 재시도에서는 지수 백오프와 지터를 함께 고려해야 합니다.
외부 시스템을 보호하는 재시도 전략은 [Node.js exponential backoff jitter 가이드: 재시도 폭주를 막는 안정적인 전략](/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html)를 참고하세요.

### H3. 인증 실패는 재시도로 숨기지 않는다

WebSocket 연결 실패가 모두 네트워크 문제는 아닙니다.
토큰 만료, 권한 부족, 잘못된 구독 채널, 프록시 설정 오류처럼 재시도해도 해결되지 않는 실패가 있습니다.

예를 들어 서버가 다음 메시지를 보낼 수 있습니다.

```json
{ "type": "error", "code": "UNAUTHORIZED", "message": "invalid token" }
```

이 경우 클라이언트는 계속 재연결하기보다 즉시 실패로 처리해야 합니다.

```js
socket.addEventListener('message', (event) => {
  const message = JSON.parse(event.data);

  if (message.type === 'error' && message.code === 'UNAUTHORIZED') {
    console.error('authentication failed');
    socket.close(1008, 'authentication failed');
    process.exitCode = 1;
    return;
  }
});
```

특히 로그에 토큰 원문을 남기지 않아야 합니다.
인증 헤더, 쿠키, 세션 ID, 개인 식별 정보는 반드시 마스킹해야 합니다.
블로그 예제와 운영 로그의 민감정보 처리 기준은 [CLI 출력 정리 가이드: 블로그 예제에서 토큰과 개인정보를 안전하게 마스킹하기](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)를 참고하면 좋습니다.

## 테스트 스크립트로 정리하기

### H3. 성공 조건을 코드로 명확히 표현한다

WebSocket 점검 스크립트는 “연결되면 성공”보다 더 구체적인 조건을 가져야 합니다.
예를 들어 실시간 알림 서버라면 다음을 성공 조건으로 둘 수 있습니다.

1. WebSocket 연결이 열린다.
2. `subscribe` 메시지를 보낸다.
3. `subscribed` 응답을 받는다.
4. 테스트 이벤트 또는 ping 응답을 받는다.
5. 정상 종료 코드로 닫힌다.

이를 함수로 만들면 CI와 로컬에서 함께 사용할 수 있습니다.

```js
export function checkRealtimeEndpoint(url) {
  return new Promise((resolve, reject) => {
    const socket = new WebSocket(url);
    const timer = setTimeout(() => {
      socket.close(1000, 'timeout');
      reject(new Error('websocket check timeout'));
    }, 8000);

    socket.addEventListener('open', () => {
      socket.send(JSON.stringify({ type: 'subscribe', channel: 'health' }));
    });

    socket.addEventListener('message', (event) => {
      const message = JSON.parse(event.data);

      if (message.type === 'subscribed' && message.channel === 'health') {
        clearTimeout(timer);
        socket.close(1000, 'check complete');
        resolve();
      }
    });

    socket.addEventListener('error', () => {
      clearTimeout(timer);
      reject(new Error('websocket connection error'));
    });
  });
}
```

테스트 러너와 연결하면 다음처럼 쓸 수 있습니다.

```js
import test from 'node:test';
import { checkRealtimeEndpoint } from './check-realtime-endpoint.js';

test('realtime websocket endpoint is healthy', async () => {
  await checkRealtimeEndpoint(process.env.WS_URL);
});
```

Node.js 내장 테스트 러너의 기본 흐름은 [Node.js test runner 가이드: Jest 없이 내장 테스트 시작하기](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)에 정리해 두었습니다.

### H3. 테스트 전용 채널을 분리한다

운영 WebSocket 서버를 테스트할 때 실제 사용자 채널을 구독하거나 이벤트를 발행하면 안 됩니다.
테스트 스크립트는 가능한 한 전용 채널을 사용해야 합니다.

권장 패턴은 다음과 같습니다.

- `health`, `debug`, `smoke-test` 같은 읽기 전용 채널을 둔다.
- 테스트 계정의 권한을 최소화한다.
- 테스트 이벤트는 실제 사용자 알림과 분리한다.
- 로그에는 토큰과 개인정보를 남기지 않는다.
- 실패 시 마지막 메시지 타입과 종료 코드만 요약한다.

이렇게 분리하면 smoke test를 자주 실행해도 사용자 경험에 영향을 주지 않습니다.
또한 운영 장애 대응 중에도 “테스트 때문에 만든 부작용”과 실제 장애를 구분하기 쉬워집니다.

## 도입 전 체크리스트

### H3. Node.js 내장 WebSocket 사용 기준

내장 WebSocket 클라이언트는 가벼운 검증과 자동화에 좋지만, 모든 상황에서 외부 패키지를 대체하는 것은 아닙니다.
도입 전에는 다음을 확인하세요.

- 팀의 Node.js 버전에서 `WebSocket`이 사용 가능한가?
- 필요한 인증 헤더나 서브프로토콜을 표현할 수 있는가?
- 바이너리 메시지 처리가 필요한가?
- 프록시, mTLS, 특수 TLS 설정이 필요한가?
- 서버 구현이 아니라 클라이언트 검증 목적이 맞는가?
- 실패 시 종료 코드와 로그가 명확한가?

복잡한 서버 구현, 고급 옵션, 오래된 Node.js 지원이 필요하다면 여전히 검증된 WebSocket 라이브러리가 더 나을 수 있습니다.
반대로 배포 후 연결 확인, 문서 예제, CI smoke test처럼 범위가 좁은 작업이라면 내장 WebSocket부터 검토해 볼 만합니다.

### H3. 문서와 package.json에 실행 조건을 남긴다

런타임 내장 기능은 “내 컴퓨터에서는 된다”로 끝내면 팀에서 문제가 생깁니다.
다음처럼 실행 조건을 문서화하세요.

```json
{
  "engines": {
    "node": ">=22"
  },
  "scripts": {
    "check:ws": "node scripts/check-websocket.js"
  }
}
```

README에는 환경 변수 예시를 함께 적어두면 좋습니다.

```bash
WS_URL=wss://staging.example.com/realtime npm run check:ws
```

환경 변수 예시를 공개 문서에 남길 때는 실제 토큰이나 내부 주소를 넣지 말아야 합니다.
필수 환경 변수와 예시 파일 관리 방식은 [환경 변수 예시 파일 가이드: .env.example로 배포 실수를 줄이는 법](/development/blog/seo/2026/03/08/env-example-and-required-variables-guide.html)를 참고하세요.

## FAQ

### H3. Node.js 내장 WebSocket은 서버도 만들 수 있나요?

이 글에서 다룬 범위는 클라이언트입니다.
Node.js 내장 `WebSocket`은 브라우저와 비슷한 클라이언트 API로 생각하는 것이 안전합니다.
WebSocket 서버를 운영하려면 프레임워크, 프록시, 인증, heartbeat, scale-out 구조까지 함께 설계해야 하므로 별도 라이브러리나 플랫폼 기능을 검토하는 편이 현실적입니다.

### H3. ws 패키지를 모두 제거해도 되나요?

간단한 연결 점검 스크립트라면 제거할 수 있는 경우가 많습니다.
하지만 서버 구현, 고급 클라이언트 옵션, 오래된 Node.js 지원, 특수한 바이너리 처리, 프록시 설정이 필요하다면 `ws` 같은 패키지가 여전히 적합할 수 있습니다.
먼저 smoke test나 내부 진단 스크립트부터 내장 WebSocket으로 바꿔보는 방식을 추천합니다.

### H3. WebSocket 테스트에서 가장 중요한 실패 조건은 무엇인가요?

연결 실패, 응답 타임아웃, 인증 실패, 비정상 종료 코드입니다.
이 네 가지를 구분해 로그에 남기면 대부분의 장애 원인 범위를 빠르게 좁힐 수 있습니다.
성공 조건도 “연결됨”이 아니라 “구독 응답을 받고 정상 종료됨”처럼 구체적으로 잡는 것이 좋습니다.

## 정리

Node.js 내장 WebSocket 클라이언트는 실시간 기능을 검증하는 작은 스크립트를 단순하게 만들어 줍니다.
별도 패키지 없이 연결, 메시지 송수신, 종료 코드 확인, smoke test를 구성할 수 있어 배포 후 점검이나 장애 대응에 특히 유용합니다.

핵심은 세 가지입니다.

- `open`, `message`, `close`, `error` 이벤트를 나눠 실패 원인을 명확히 기록한다.
- 연결 타임아웃과 응답 타임아웃을 따로 두어 무한 대기를 막는다.
- 인증 정보와 사용자 데이터는 로그와 예제에서 반드시 마스킹한다.

복잡한 서버 구현에는 여전히 전용 라이브러리가 필요할 수 있습니다.
하지만 가벼운 진단과 테스트 자동화부터 시작한다면, Node.js 내장 WebSocket은 의존성을 줄이면서도 실무적인 안정성을 높일 수 있는 좋은 선택지입니다.

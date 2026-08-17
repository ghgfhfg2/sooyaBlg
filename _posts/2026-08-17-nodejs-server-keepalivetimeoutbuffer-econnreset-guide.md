---
layout: post
title: "Node.js server.keepAliveTimeoutBuffer 가이드: ECONNRESET을 줄이는 HTTP keep-alive 설정법"
date: 2026-08-17 20:00:00 +0900
lang: ko
translation_key: nodejs-server-keepalivetimeoutbuffer-econnreset-guide
permalink: /development/blog/seo/2026/08/17/nodejs-server-keepalivetimeoutbuffer-econnreset-guide.html
alternates:
  ko: /development/blog/seo/2026/08/17/nodejs-server-keepalivetimeoutbuffer-econnreset-guide.html
  x_default: /development/blog/seo/2026/08/17/nodejs-server-keepalivetimeoutbuffer-econnreset-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, http, keep-alive, econnreset, timeout, backend, observability, javascript]
description: "Node.js server.keepAliveTimeoutBuffer로 HTTP keep-alive 재사용 중 발생하는 ECONNRESET을 줄이는 방법을 정리합니다. keepAliveTimeout, headersTimeout, 프록시 idle timeout과의 관계, 배포 전 점검 항목과 로그 기준까지 실무 예제로 설명합니다."
---

Node.js HTTP 서버에서 간헐적인 `ECONNRESET`, `socket hang up`, 502 오류가 보일 때 원인이 항상 애플리케이션 로직에 있는 것은 아닙니다.
요청은 정상적으로 처리됐는데 클라이언트나 프록시가 재사용하려던 keep-alive 소켓이 서버 쪽에서 거의 동시에 닫히면, 운영 로그에는 애매한 연결 오류로 남을 수 있습니다.

`server.keepAliveTimeoutBuffer`는 이런 경계 상황을 줄이기 위해 `server.keepAliveTimeout`에 내부 버퍼 시간을 더해 실제 소켓 종료 시점을 조금 늦추는 설정입니다.
Node.js 공식 문서 기준으로 이 옵션은 `v24.6.0`과 `v22.19.0`에 추가됐고, 기본값은 1초입니다.
핵심은 클라이언트에 알려지는 keep-alive 시간과 서버 내부 소켓 타임아웃 사이에 작은 여유를 두는 것입니다.

이 글에서는 `server.keepAliveTimeoutBuffer`가 왜 필요한지, 기존 `keepAliveTimeout`과 `headersTimeout` 설정과 어떻게 같이 봐야 하는지, 프록시 뒤에 있는 Node.js 서버에서 어떤 순서로 검증하면 좋은지 정리합니다.
기본 타임아웃의 역할은 [Node.js requestTimeout, timeout, headersTimeout 차이 가이드](/development/blog/seo/2026/04/16/nodejs-requesttimeout-timeout-headerstimeout-difference-guide.html), keep-alive와 프록시 간격은 [Node.js keepAliveTimeout, headersTimeout mismatch 가이드](/development/blog/seo/2026/04/14/nodejs-keepalive-timeout-headers-timeout-mismatch-guide.html), 연결 재활용 기준은 [Node.js maxRequestsPerSocket 가이드](/development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html)와 함께 보면 좋습니다.

## keepAliveTimeoutBuffer가 해결하는 문제

### keep-alive 소켓은 닫히는 순간이 중요하다

HTTP keep-alive는 요청마다 TCP 연결을 새로 만들지 않고 같은 연결을 재사용하게 해 줍니다.
짧은 API 호출이 많은 서비스에서는 지연 시간과 연결 비용을 줄이는 데 도움이 됩니다.
하지만 재사용 가능한 소켓은 언제까지 살아 있는지에 대한 약속이 필요합니다.

Node.js의 `server.keepAliveTimeout`은 마지막 응답을 보낸 뒤 추가 요청을 기다리는 시간입니다.
기본값은 5초입니다.
이 시간이 지나면 서버는 해당 keep-alive 소켓을 닫을 수 있습니다.
문제는 클라이언트나 프록시가 같은 시점 근처에서 그 소켓을 다시 쓰려고 할 때 생깁니다.

```js
import http from 'node:http';

const server = http.createServer((req, res) => {
  res.setHeader('content-type', 'application/json; charset=utf-8');
  res.end(JSON.stringify({ ok: true }));
});

server.keepAliveTimeout = 5000;
server.headersTimeout = 6000;

server.listen(3000);
```

이 설정은 단순하지만, 요청 간격이 keep-alive 종료 시점과 비슷하게 맞물리면 재사용 경계에서 연결 오류가 드물게 발생할 수 있습니다.
트래픽이 많을 때보다 오히려 낮은 QPS에서 간헐적으로 보이는 경우도 있습니다.
소켓이 오래 쉬다가 다시 선택되는 순간이 늘어나기 때문입니다.

### 버퍼는 외부 약속보다 내부 종료를 늦춘다

`server.keepAliveTimeoutBuffer`는 클라이언트에 광고되는 keep-alive 시간 자체를 늘리는 설정이 아닙니다.
서버 내부에서 소켓을 실제로 닫는 타임아웃을 `keepAliveTimeout + keepAliveTimeoutBuffer`로 계산하게 해 주는 여유 시간입니다.

```js
import http from 'node:http';

const server = http.createServer(async (req, res) => {
  res.end('ok');
});

server.keepAliveTimeout = 5000;
server.keepAliveTimeoutBuffer = 1000;
server.headersTimeout = 7000;

server.listen(3000);
```

위 예시에서는 외부적으로는 5초 keep-alive 정책을 유지하면서, 내부 소켓 종료에는 1초 여유를 둡니다.
그 결과 클라이언트가 서버의 keep-alive 힌트를 기준으로 소켓을 정리하기 전에 서버가 먼저 닫아 버릴 가능성을 낮출 수 있습니다.
버퍼는 오류를 완전히 없애는 마법 같은 설정이 아니라, 시간 경계에서 생기는 불필요한 리셋을 줄이는 안전장치에 가깝습니다.

## 타임아웃 설정을 함께 보는 법

### keepAliveTimeout은 프록시 idle timeout보다 짧게 둔다

Node.js 서버가 ALB, Nginx, CDN, API Gateway 뒤에 있다면 클라이언트는 대부분 프록시입니다.
이때 Node.js의 keep-alive 정책만 보고 설정하면 부족합니다.
프록시의 idle timeout, Node.js의 `keepAliveTimeout`, 내부 버퍼, `headersTimeout`을 함께 봐야 합니다.

운영 기준은 보통 다음 순서가 읽기 쉽습니다.

```txt
Node.js advertised keepAliveTimeout
< Node.js internal socket timeout
< headersTimeout
< reverse proxy idle timeout
```

예를 들어 프록시 idle timeout이 60초라면 Node.js 서버는 다음처럼 보수적으로 둘 수 있습니다.

```js
server.keepAliveTimeout = 55000;
server.keepAliveTimeoutBuffer = 1000;
server.headersTimeout = 57000;
```

이 값은 모든 서비스에 그대로 복사할 정답이 아닙니다.
중요한 것은 상대적인 순서입니다.
프록시가 먼저 소켓을 정리하는지, Node.js가 먼저 정리하는지, 헤더 수신 제한이 keep-alive 내부 종료보다 충분히 큰지 확인해야 합니다.

### headersTimeout은 버퍼까지 고려해 잡는다

`headersTimeout`은 요청 헤더를 다 받을 때까지 기다리는 시간입니다.
keep-alive 소켓의 내부 종료 시점보다 너무 짧으면 정상 요청도 경계 상황에서 잘릴 수 있습니다.
`keepAliveTimeoutBuffer`를 명시적으로 키웠다면 `headersTimeout`도 함께 검토해야 합니다.

```js
function configureHttpTimeouts(server, {
  keepAliveTimeoutMs = 55000,
  keepAliveTimeoutBufferMs = 1000,
  headersTimeoutMs = 57000,
  requestTimeoutMs = 30000
} = {}) {
  const internalKeepAliveMs = keepAliveTimeoutMs + keepAliveTimeoutBufferMs;

  if (headersTimeoutMs <= internalKeepAliveMs) {
    throw new Error('headersTimeout must be greater than keepAliveTimeout plus buffer');
  }

  server.keepAliveTimeout = keepAliveTimeoutMs;
  server.keepAliveTimeoutBuffer = keepAliveTimeoutBufferMs;
  server.headersTimeout = headersTimeoutMs;
  server.requestTimeout = requestTimeoutMs;
}
```

설정을 함수로 모으면 리뷰하기 쉬워집니다.
특히 여러 서비스가 같은 플랫폼 위에서 운영된다면 이 검증을 공통 부트스트랩에 넣어 두는 편이 좋습니다.
잘못된 숫자를 배포한 뒤 로그로 추적하는 것보다 시작 단계에서 멈추는 쪽이 훨씬 싸게 끝납니다.

### requestTimeout은 별도 예산으로 관리한다

`requestTimeout`은 keep-alive 소켓의 유휴 시간과 다른 문제를 다룹니다.
요청 전체를 수신하는 데 걸리는 시간을 제한하는 값입니다.
JSON API, 파일 업로드, 스트리밍 엔드포인트는 적절한 예산이 다를 수 있습니다.

```js
server.keepAliveTimeout = 55000;
server.keepAliveTimeoutBuffer = 1000;
server.headersTimeout = 57000;
server.requestTimeout = 30000;
```

`ECONNRESET`이 보인다고 `requestTimeout`만 늘리는 것은 원인과 맞지 않을 수 있습니다.
반대로 느린 업로드 때문에 408이 발생하는 상황에서 `keepAliveTimeoutBuffer`를 조정해도 효과가 없습니다.
오류 코드, 요청 단계, 프록시 로그, Node.js 서버 로그를 함께 보고 어떤 타임아웃이 개입했는지 먼저 분리해야 합니다.

## 프록시 뒤 Node.js 서버 적용 패턴

### 설정값을 환경별로 드러낸다

로컬 개발, 스테이징, 운영은 프록시 구성이 다를 수 있습니다.
따라서 HTTP 타임아웃 값을 코드에 흩뿌리기보다 환경 설정으로 모으고, 최종 적용값을 안전하게 로그에 남기는 편이 좋습니다.

```js
import http from 'node:http';

function readNumber(name, fallback) {
  const raw = process.env[name];
  if (raw == null || raw === '') return fallback;

  const value = Number(raw);
  if (!Number.isFinite(value) || value < 0) {
    throw new Error(`${name} must be a non-negative number`);
  }

  return value;
}

export function createServer(handler, logger = console) {
  const server = http.createServer(handler);

  const keepAliveTimeout = readNumber('HTTP_KEEP_ALIVE_TIMEOUT_MS', 55000);
  const keepAliveTimeoutBuffer = readNumber('HTTP_KEEP_ALIVE_TIMEOUT_BUFFER_MS', 1000);
  const headersTimeout = readNumber('HTTP_HEADERS_TIMEOUT_MS', 57000);

  configureHttpTimeouts(server, {
    keepAliveTimeoutMs: keepAliveTimeout,
    keepAliveTimeoutBufferMs: keepAliveTimeoutBuffer,
    headersTimeoutMs: headersTimeout,
    requestTimeoutMs: readNumber('HTTP_REQUEST_TIMEOUT_MS', 30000)
  });

  logger.info('http timeouts configured', {
    keepAliveTimeout,
    keepAliveTimeoutBuffer,
    headersTimeout,
    requestTimeout: server.requestTimeout
  });

  return server;
}
```

이 로그에는 토큰, 쿠키, 데이터베이스 주소 같은 민감정보가 들어가지 않습니다.
타임아웃 숫자는 운영 진단에 필요하지만, `process.env` 전체를 출력하는 방식은 피해야 합니다.
환경 설정을 검증하면서도 민감정보 노출 위험은 줄이는 것이 좋습니다.

### 프록시 로그와 같은 시간대로 맞춰 본다

연결 오류는 한쪽 로그만 보면 해석이 어렵습니다.
Node.js 애플리케이션은 요청 처리에 성공했다고 남겼지만, 프록시는 다음 재사용 요청에서 502를 기록할 수 있습니다.
반대로 Node.js 쪽에서는 클라이언트가 먼저 끊은 것으로만 보일 수도 있습니다.

확인할 항목은 단순합니다.

- 오류가 새 연결에서 발생했는지, 재사용 연결에서 발생했는지
- 프록시 idle timeout과 Node.js keep-alive 설정의 상대 순서가 맞는지
- 오류가 특정 경로가 아니라 낮은 빈도의 재사용 시점에 몰리는지
- 배포 직후 새 설정이 실제 프로세스에 반영됐는지

가능하다면 프록시 request ID와 애플리케이션 request ID를 연결해 둡니다.
관측성 필드를 일관되게 남겨야 시간 경계 문제와 애플리케이션 예외를 분리할 수 있습니다.

### 버퍼를 크게 잡아 문제를 덮지 않는다

`keepAliveTimeoutBuffer`는 보통 작은 값으로 충분합니다.
기본값 1초는 합리적인 출발점입니다.
오류가 보인다고 버퍼를 과하게 키우면 유휴 소켓이 더 오래 남아 리소스 사용량이 늘 수 있습니다.

버퍼를 조정할 때는 다음 순서로 보는 편이 안전합니다.

1. 프록시 idle timeout과 Node.js keep-alive 값의 순서를 확인한다.
2. `headersTimeout`이 내부 keep-alive 종료 시점보다 큰지 확인한다.
3. 낮은 QPS에서 재사용 소켓 오류가 줄어드는지 측정한다.
4. 열린 소켓 수, 메모리, 502/499/408 비율을 같이 본다.

버퍼는 증상 완화를 위한 설정이고, 전체 연결 정책의 일부입니다.
소켓 수가 급증하거나 프록시가 먼저 연결을 끊는 구조라면 숫자 하나보다 전체 타임아웃 표를 다시 정리하는 것이 먼저입니다.

## 배포 전 점검 체크리스트

### 설정 표를 문서화한다

HTTP 타임아웃은 한 번 맞춰 두면 잊히기 쉽습니다.
하지만 프록시 변경, Node.js 버전 업그레이드, 런타임 이미지 변경이 생기면 전제가 달라질 수 있습니다.
서비스 README나 운영 문서에 최소한 다음 값을 남깁니다.

```txt
reverse_proxy_idle_timeout_ms = 60000
node_keep_alive_timeout_ms = 55000
node_keep_alive_timeout_buffer_ms = 1000
node_headers_timeout_ms = 57000
node_request_timeout_ms = 30000
```

숫자만 적는 것보다 "왜 이 순서인지"를 함께 적어 두는 것이 더 중요합니다.
나중에 누군가 502를 줄이려고 값을 바꿀 때, 어떤 값을 먼저 움직여야 하는지 판단할 수 있기 때문입니다.

### 배포 후에는 오류 비율을 비교한다

설정을 바꾼 뒤에는 전체 에러 수보다 비율과 위치를 봐야 합니다.
트래픽이 늘면 에러 수는 자연스럽게 늘 수 있습니다.
대신 같은 시간대의 요청 수 대비 `ECONNRESET`, 502, 408 비율이 어떻게 바뀌었는지 확인합니다.

```js
server.on('clientError', (err, socket) => {
  logger.warn('http client error', {
    code: err.code,
    message: err.message,
    bytesParsed: err.bytesParsed
  });

  if (socket.writable) {
    socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
  }
});
```

`clientError` 로그는 원인 파악에 도움이 되지만, 요청 헤더나 원문 패킷을 그대로 남기지 않는 편이 안전합니다.
인증 헤더, 쿠키, 사용자 입력이 섞일 수 있기 때문입니다.
오류 코드와 카운터 중심으로 수집하고, 필요할 때만 제한된 샘플을 별도 보호된 위치에서 확인하는 흐름이 좋습니다.

### 구버전 런타임을 고려한다

`server.keepAliveTimeoutBuffer`가 없는 Node.js 버전에서도 같은 코드가 실행될 수 있다면 버전 차이를 고려해야 합니다.
가장 좋은 방법은 지원 런타임을 명확히 고정하는 것입니다.
그래도 라이브러리나 공통 부트스트랩이 여러 버전에서 실행되어야 한다면 속성 존재 여부를 확인할 수 있습니다.

```js
if ('keepAliveTimeoutBuffer' in server) {
  server.keepAliveTimeoutBuffer = 1000;
}
```

다만 이 방식은 임시 호환 장치입니다.
운영에서는 실제 Node.js 버전, 컨테이너 이미지, 배포된 프로세스의 런타임을 함께 확인해야 합니다.
문서에 있는 API라고 해서 현재 실행 중인 환경에 반드시 있는 것은 아닙니다.

## FAQ

### keepAliveTimeoutBuffer를 0으로 두면 안 되나요?

둘 수는 있습니다.
하지만 keep-alive 재사용 경계에서 서버가 먼저 소켓을 닫아 `ECONNRESET`이 늘어나는 상황이라면 버퍼를 두는 편이 안전합니다.
기본값인 1초를 출발점으로 삼고, 프록시 idle timeout과 전체 소켓 수를 함께 보며 조정하는 것이 좋습니다.

### keepAliveTimeoutBuffer만 설정하면 502가 사라지나요?

항상 그렇지는 않습니다.
502는 애플리케이션 예외, 프록시 연결 실패, upstream timeout, 배포 중 연결 종료 등 여러 원인으로 발생합니다.
`keepAliveTimeoutBuffer`는 keep-alive 소켓 종료 경계에서 발생하는 연결 리셋을 줄이는 설정입니다.
오류 로그가 재사용 소켓과 유휴 시간 근처에 몰릴 때 특히 확인할 가치가 있습니다.

### Express나 Fastify에서도 적용할 수 있나요?

적용할 수 있습니다.
Express와 Fastify도 결국 Node.js HTTP 서버 위에서 동작합니다.
중요한 것은 프레임워크 객체가 아니라 실제 `http.Server` 인스턴스에 값을 설정하는 것입니다.
프레임워크가 서버를 직접 만들거나 플러그인에서 감싸는 구조라면 서버 인스턴스를 얻는 위치를 먼저 확인해야 합니다.

## 마무리

`server.keepAliveTimeoutBuffer`는 작은 옵션이지만, Node.js HTTP 서버를 프록시 뒤에서 운영할 때 꽤 실용적인 안전장치입니다.
클라이언트에 알려지는 keep-alive 시간보다 서버 내부 소켓 종료를 조금 늦춰 재사용 경계의 `ECONNRESET` 가능성을 줄입니다.

적용할 때는 `keepAliveTimeout`, `keepAliveTimeoutBuffer`, `headersTimeout`, 프록시 idle timeout을 하나의 표로 관리하세요.
그리고 배포 후에는 502와 `ECONNRESET` 비율, 열린 소켓 수, 408 응답을 함께 비교해야 합니다.
타임아웃 설정은 숫자 하나의 문제가 아니라 연결 수명 전체를 설계하는 문제입니다.

### 함께 읽기

- [Node.js requestTimeout, timeout, headersTimeout 차이 가이드](/development/blog/seo/2026/04/16/nodejs-requesttimeout-timeout-headerstimeout-difference-guide.html)
- [Node.js keepAliveTimeout, headersTimeout mismatch 가이드](/development/blog/seo/2026/04/14/nodejs-keepalive-timeout-headers-timeout-mismatch-guide.html)
- [Node.js maxRequestsPerSocket 가이드](/development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html)

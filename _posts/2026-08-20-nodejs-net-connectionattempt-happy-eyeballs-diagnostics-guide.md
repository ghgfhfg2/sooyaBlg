---
layout: post
title: "Node.js net.Socket connectionAttempt 가이드: IPv4·IPv6 연결 시도를 로그로 추적하는 법"
date: 2026-08-20 20:00:00 +0900
lang: ko
translation_key: nodejs-net-connectionattempt-happy-eyeballs-diagnostics-guide
permalink: /development/blog/seo/2026/08/20/nodejs-net-connectionattempt-happy-eyeballs-diagnostics-guide.html
alternates:
  ko: /development/blog/seo/2026/08/20/nodejs-net-connectionattempt-happy-eyeballs-diagnostics-guide.html
  x_default: /development/blog/seo/2026/08/20/nodejs-net-connectionattempt-happy-eyeballs-diagnostics-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, net, socket, connectionattempt, ipv4, ipv6, happy-eyeballs, networking, observability, backend, javascript]
description: "Node.js net.Socket의 connectionAttempt, connectionAttemptFailed, connectionAttemptTimeout 이벤트로 IPv4·IPv6 연결 시도를 추적하는 방법을 정리합니다. autoSelectFamily, attempted addresses, 민감정보 없는 네트워크 로그 설계까지 실무 예제로 설명합니다."
---

외부 데이터베이스, 캐시, 사내 API, 프록시처럼 TCP 연결을 직접 여는 코드에서는 "왜 연결이 느렸는지"를 나중에 설명하기 어렵습니다.
최종 에러는 `ECONNREFUSED`나 `ETIMEDOUT` 하나로 보이지만, 실제로는 IPv6 주소를 먼저 시도했다가 실패하고 IPv4로 넘어갔거나, 여러 주소 중 일부만 느렸을 수 있습니다.
이 차이를 로그로 남기지 않으면 DNS, 방화벽, 네트워크 경로, 원격 서비스 상태 중 어디를 봐야 하는지 판단이 늦어집니다.

Node.js `net.Socket`에는 이런 진단을 위해 `connectionAttempt`, `connectionAttemptFailed`, `connectionAttemptTimeout` 이벤트가 있습니다.
Node.js 공식 문서 기준으로 이 이벤트들은 `v21.6.0`과 `v20.12.0`에 추가됐고, `socket.connect(options)`에서 family autoselection 알고리즘이 활성화되면 여러 번 발생할 수 있습니다.
또 `socket.autoSelectFamilyAttemptedAddresses`를 보면 실제로 시도된 주소 목록을 확인할 수 있습니다.

이 글에서는 Node.js `connectionAttempt` 이벤트를 어디에 붙이면 좋은지, `autoSelectFamily`와 함께 어떤 로그를 남겨야 하는지, 운영 로그에 민감정보를 섞지 않으려면 무엇을 제한해야 하는지 정리합니다.
IP 허용·차단 규칙은 [Node.js net.BlockList 가이드](/development/blog/seo/2026/06/01/nodejs-net-blocklist-ip-filtering-guide.html), 주소 문자열 검증은 [Node.js net.SocketAddress.parse 가이드](/development/blog/seo/2026/06/06/nodejs-net-socketaddress-parse-input-validation-guide.html), 외부 요청 timeout 기준은 [Node.js fetch timeout 재시도 가이드](/development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html)와 함께 보면 좋습니다.

## connectionAttempt 이벤트가 필요한 이유

### 최종 에러만으로는 연결 지연을 설명하기 어렵다

TCP 연결 실패는 단순한 성공·실패 문제가 아닙니다.
호스트 이름 하나가 IPv6와 IPv4 주소를 모두 가질 수 있고, Node.js는 `autoSelectFamily`가 켜진 상황에서 여러 주소를 순서대로 시도할 수 있습니다.
사용자 입장에서는 "접속이 1초 늦었다"로 보이지만 운영자는 다음 질문에 답해야 합니다.

- IPv6 주소를 먼저 시도했는가?
- 첫 번째 주소가 timeout 난 뒤 다른 주소로 넘어갔는가?
- 최종 연결은 IPv4로 성공했는가?
- 모든 주소가 실패했는가, 아니면 일부만 실패했는가?

`error` 이벤트만 보면 마지막 실패나 집계된 실패만 보이는 경우가 많습니다.
반면 `connectionAttempt` 계열 이벤트를 남기면 각 주소 시도 단위로 흐름을 따라갈 수 있습니다.

```js
import net from 'node:net';

const socket = net.createConnection({
  host: 'example.internal',
  port: 5432,
  autoSelectFamily: true
});

socket.on('connectionAttempt', (ip, port, family) => {
  console.info('connection attempt started', { ip, port, family });
});

socket.on('connectionAttemptFailed', (ip, port, family, error) => {
  console.warn('connection attempt failed', {
    ip,
    port,
    family,
    code: error.code
  });
});

socket.on('connectionAttemptTimeout', (ip, port, family) => {
  console.warn('connection attempt timed out', { ip, port, family });
});
```

이 로그는 "연결 실패"라는 한 줄보다 훨씬 실용적입니다.
특히 IPv6 라우팅이 일부 환경에서만 불안정하거나, DNS가 여러 주소를 반환하는 서비스에서 원인 범위를 빠르게 줄일 수 있습니다.

### autoSelectFamily 흐름을 관찰한다

Node.js의 `autoSelectFamily` 옵션은 IPv6와 IPv4 주소를 순차적으로 시도해 연결 성공 가능성을 높이는 기능입니다.
공식 문서에 따르면 `autoSelectFamily`가 활성화되면 lookup의 `all` 옵션을 `true`로 두고, 반환된 IPv6와 IPv4 주소를 교차 순서로 시도합니다.
각 시도는 `autoSelectFamilyAttemptTimeout` 값만큼 기다린 뒤 다음 주소로 넘어갈 수 있습니다.

```js
import net from 'node:net';

const socket = net.createConnection({
  host: 'api.example.com',
  port: 443,
  autoSelectFamily: true,
  autoSelectFamilyAttemptTimeout: 250
});
```

`autoSelectFamilyAttemptTimeout`은 전체 연결 timeout이 아닙니다.
주소 하나를 시도하다가 다음 주소로 넘어가기 전까지의 대기 기준입니다.
전체 요청이나 작업 상한은 별도의 timeout 예산으로 관리해야 합니다.

## 연결 시도 로그 설계하기

### connect 함수 안에 관측 이벤트를 묶는다

서비스 코드 곳곳에서 `net.createConnection()`을 직접 호출하면 로그 형식이 흔들리기 쉽습니다.
운영에서 조사 가능한 로그를 만들려면 작은 연결 헬퍼를 두고, 그 안에서 이벤트를 일관되게 붙이는 편이 좋습니다.

```js
import net from 'node:net';

export function createObservedSocket({
  host,
  port,
  serviceName,
  logger,
  signal
}) {
  const startedAt = performance.now();
  const socket = net.createConnection({
    host,
    port,
    autoSelectFamily: true,
    autoSelectFamilyAttemptTimeout: 250,
    signal
  });

  socket.on('connectionAttempt', (ip, attemptedPort, family) => {
    logger.info({
      event: 'tcp_connection_attempt',
      serviceName,
      ipFamily: family,
      port: attemptedPort,
      ipHash: hashIpForLogs(ip)
    });
  });

  socket.on('connectionAttemptFailed', (ip, attemptedPort, family, error) => {
    logger.warn({
      event: 'tcp_connection_attempt_failed',
      serviceName,
      ipFamily: family,
      port: attemptedPort,
      ipHash: hashIpForLogs(ip),
      errorCode: error.code
    });
  });

  socket.on('connectionAttemptTimeout', (ip, attemptedPort, family) => {
    logger.warn({
      event: 'tcp_connection_attempt_timeout',
      serviceName,
      ipFamily: family,
      port: attemptedPort,
      ipHash: hashIpForLogs(ip)
    });
  });

  socket.once('connect', () => {
    logger.info({
      event: 'tcp_connection_connected',
      serviceName,
      durationMs: Math.round(performance.now() - startedAt),
      remoteFamily: socket.remoteFamily
    });
  });

  return socket;
}

function hashIpForLogs(ip) {
  return `ip:${Buffer.from(ip).toString('base64url').slice(0, 10)}`;
}
```

예제의 `hashIpForLogs()`는 운영용 해시 함수라기보다 로그 필드 설계를 설명하기 위한 자리입니다.
실제 서비스에서는 일관된 salt를 쓰는 해시, 내부망 여부만 남기는 분류값, 또는 보안팀 기준에 맞춘 마스킹 함수를 사용해야 합니다.
중요한 점은 원격 IP를 그대로 남기는 것이 항상 필요한 것은 아니라는 사실입니다.

### attempted addresses는 성공·실패 요약에만 남긴다

`socket.autoSelectFamilyAttemptedAddresses`는 family autoselection이 켜진 상황에서 시도된 주소 목록을 담습니다.
각 항목은 `$IP:$PORT` 형태의 문자열입니다.
연결이 성공했다면 마지막 주소가 현재 연결된 주소입니다.

```js
socket.once('close', (hadError) => {
  logger.info({
    event: 'tcp_connection_closed',
    serviceName,
    hadError,
    attemptedAddressCount: socket.autoSelectFamilyAttemptedAddresses?.length ?? 0
  });
});
```

주소 전체 목록을 매번 로그에 남기면 유용해 보일 수 있습니다.
하지만 운영 로그가 외부 시스템으로 전송된다면 IP 주소도 민감한 인프라 정보가 될 수 있습니다.
기본 로그에는 개수와 family, 성공 여부를 남기고, 상세 주소는 디버그 모드나 제한된 진단 환경에서만 확인하는 편이 안전합니다.

## timeout과 AbortSignal 기준 나누기

### autoSelectFamilyAttemptTimeout은 주소별 대기 시간이다

`autoSelectFamilyAttemptTimeout`은 여러 주소를 시도할 때 각 연결 시도에 얼마만큼 시간을 줄지 결정합니다.
값을 너무 길게 잡으면 느린 주소 하나가 전체 연결 지연을 키울 수 있습니다.
값을 너무 짧게 잡으면 정상적으로 조금 늦게 응답하는 네트워크까지 실패로 판단할 수 있습니다.

```js
const socket = net.createConnection({
  host,
  port,
  autoSelectFamily: true,
  autoSelectFamilyAttemptTimeout: 300
});
```

처음에는 Node.js 기본값이나 보수적인 값으로 시작하고, 실제 로그에서 timeout 빈도와 최종 성공 family를 확인한 뒤 조정하는 방식이 좋습니다.
네트워크 옵션은 감으로 정하기보다 운영 로그로 확인하며 좁혀야 합니다.

### 전체 작업 timeout은 signal로 관리한다

주소별 시도 시간과 전체 작업 시간은 분리해야 합니다.
예를 들어 주소별 시도는 250ms로 넘기더라도, 전체 연결 작업은 2초 안에 끝나야 할 수 있습니다.
이때 `signal` 옵션을 받아 상위 호출자가 전체 timeout을 관리하게 만들면 흐름이 명확해집니다.

```js
import net from 'node:net';

export function connectWithBudget({ host, port, timeoutMs }) {
  const signal = AbortSignal.timeout(timeoutMs);

  return net.createConnection({
    host,
    port,
    autoSelectFamily: true,
    autoSelectFamilyAttemptTimeout: 250,
    signal
  });
}
```

상위 요청 취소와 timeout을 함께 다뤄야 한다면 `AbortSignal.any()`로 합친 신호를 넘길 수 있습니다.
연결 헬퍼는 하나의 `signal`만 받게 만들고, 조합 책임은 호출자 쪽에 두는 구조가 테스트와 운영 모두에서 단순합니다.

## 에러 분류와 알림 기준

### 실패 이벤트는 시도 단위, error 이벤트는 연결 단위로 본다

`connectionAttemptFailed`는 개별 주소 시도가 실패할 때 발생할 수 있습니다.
하지만 그 뒤 다른 주소 연결이 성공하면 전체 연결은 성공입니다.
따라서 이 이벤트만으로 장애 알림을 보내면 불필요한 알림이 많아질 수 있습니다.

```js
const attemptStats = {
  failed: 0,
  timedOut: 0
};

socket.on('connectionAttemptFailed', () => {
  attemptStats.failed += 1;
});

socket.on('connectionAttemptTimeout', () => {
  attemptStats.timedOut += 1;
});

socket.once('connect', () => {
  logger.info({
    event: 'tcp_connection_success_summary',
    failedAttempts: attemptStats.failed,
    timedOutAttempts: attemptStats.timedOut
  });
});

socket.once('error', (error) => {
  logger.error({
    event: 'tcp_connection_failed',
    failedAttempts: attemptStats.failed,
    timedOutAttempts: attemptStats.timedOut,
    errorCode: error.code
  });
});
```

알림은 최종 `error`나 일정 비율 이상의 timeout 증가에 두는 편이 좋습니다.
개별 시도 실패는 진단 신호로 남기고, 서비스 영향이 확인될 때만 알림으로 승격합니다.

### IPv6 실패율은 별도 지표로 본다

IPv6 연결 시도가 반복적으로 timeout 난 뒤 IPv4로 성공하는 패턴은 사용자에게 지연으로 나타날 수 있습니다.
최종 성공률만 보면 장애가 없어 보이지만, 연결 시간이 길어지는 원인이 될 수 있습니다.

```js
socket.on('connectionAttemptTimeout', (ip, port, family) => {
  metrics.increment('tcp.connection_attempt_timeout', {
    service: serviceName,
    family: family === 6 ? 'ipv6' : 'ipv4'
  });
});
```

지표 태그에는 원격 IP 전체를 넣지 않는 편이 좋습니다.
카디널리티가 커지고, 내부 네트워크 정보가 관측 시스템에 퍼질 수 있습니다.
서비스명, family, 환경, 리전처럼 운영 판단에 필요한 낮은 카디널리티 필드로 시작하세요.

## 운영 적용 체크리스트

### 먼저 관찰하고, 그다음 옵션을 조정한다

네트워크 옵션은 애플리케이션 코드만 보고 정답을 고르기 어렵습니다.
먼저 `connectionAttempt` 계열 이벤트로 현재 패턴을 관찰하고, 그 결과를 바탕으로 `autoSelectFamilyAttemptTimeout`, DNS 설정, IPv6 라우팅, 방화벽 정책을 조정해야 합니다.

- 연결 대상별로 `serviceName`을 명확히 남긴다.
- `family`와 실패 코드, timeout 여부를 구분한다.
- IP 원문은 기본 로그에 남기지 않거나 마스킹한다.
- 개별 시도 실패와 최종 연결 실패를 다른 이벤트로 다룬다.
- 전체 작업 timeout은 `AbortSignal`로 따로 제한한다.

이 기준을 정해 두면 "가끔 느린 연결"을 감으로 추적하지 않고, 실제 시도 순서와 실패 유형으로 설명할 수 있습니다.

### 보안 점검은 로그 필드에서 시작한다

연결 진단 로그는 운영에 유용하지만, 과하게 자세하면 인프라 구조를 노출합니다.
호스트 이름, IP 주소, 포트, 내부 서비스명은 모두 조직에 따라 민감정보가 될 수 있습니다.
따라서 기본 로그는 문제 해결에 필요한 최소 필드로 시작하고, 상세 진단은 접근이 제한된 환경에서만 켜는 방식이 좋습니다.

## 마무리

Node.js `net.Socket`의 `connectionAttempt`, `connectionAttemptFailed`, `connectionAttemptTimeout` 이벤트는 TCP 연결을 주소 시도 단위로 볼 수 있게 해 줍니다.
특히 IPv4와 IPv6가 함께 있는 환경에서 연결 지연을 추적할 때 유용합니다.

실무에서는 이 이벤트를 단순 디버그 로그로 흘려보내기보다 연결 헬퍼 안에 표준화하고, `autoSelectFamilyAttemptTimeout`, 전체 `AbortSignal` timeout, 마스킹된 로그 필드를 함께 설계하는 편이 좋습니다.
최종 에러만 보는 단계에서 벗어나 각 주소 시도의 흐름을 남기면, 네트워크 문제를 훨씬 빠르고 안전하게 좁힐 수 있습니다.

## 함께 읽기

- [Node.js net.BlockList 가이드: IP 허용·차단 규칙을 코드에서 안전하게 관리하는 법](/development/blog/seo/2026/06/01/nodejs-net-blocklist-ip-filtering-guide.html)
- [Node.js net.SocketAddress.parse 가이드: 사용자 입력 주소를 안전하게 검증하는 법](/development/blog/seo/2026/06/06/nodejs-net-socketaddress-parse-input-validation-guide.html)
- [Node.js fetch timeout 재시도 가이드: 외부 API 실패를 분류하고 복구하는 법](/development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html)

---
layout: post
title: "Node.js net.SocketAddress.parse 가이드: IP와 포트 입력을 안전하게 파싱하는 법"
date: 2026-06-06 08:00:00 +0900
lang: ko
translation_key: nodejs-net-socketaddress-parse-input-validation-guide
permalink: /development/blog/seo/2026/06/06/nodejs-net-socketaddress-parse-input-validation-guide.html
alternates:
  ko: /development/blog/seo/2026/06/06/nodejs-net-socketaddress-parse-input-validation-guide.html
  x_default: /development/blog/seo/2026/06/06/nodejs-net-socketaddress-parse-input-validation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, net, socketaddress, ip, port, validation, backend]
description: "Node.js net.SocketAddress.parse로 IP 주소와 포트 문자열을 예외 없이 파싱하는 방법을 정리했습니다. IPv4·IPv6 입력 처리, URL 검증과의 차이, net.BlockList 연결, 설정 파일과 운영 로그에서의 안전한 사용 기준을 예제로 설명합니다."
---

환경 변수, 관리자 설정, 프록시 설정 파일을 다루다 보면 `203.0.113.10:443`처럼 IP 주소와 포트가 한 문자열로 들어오는 경우가 있습니다.
처음에는 `split(':')`로 나누면 될 것 같지만, IPv6 주소가 들어오는 순간 이 방식은 바로 깨집니다.
`[2001:db8::1]:443`처럼 콜론이 주소 자체에도 들어가기 때문입니다.

Node.js의 `node:net` 모듈에는 `SocketAddress.parse()`가 있습니다.
이 API는 IP 주소와 선택적 포트를 담은 문자열을 `SocketAddress` 객체로 파싱하고, 실패하면 예외 대신 `undefined`를 반환합니다.
즉 입력 검증 단계에서 `try/catch`를 크게 두르지 않고도 "파싱 가능 여부"와 "정책 검증"을 분리할 수 있습니다.

이 글에서는 Node.js `net.SocketAddress.parse()`의 기본 사용법, IPv4와 IPv6 입력 처리, URL 검증과의 차이, `net.BlockList`와 함께 쓰는 접근 제어 패턴, 운영 로그에서 조심할 점을 정리합니다.
IP 허용·차단 규칙 자체가 필요하다면 [Node.js net.BlockList 가이드](/development/blog/seo/2026/06/01/nodejs-net-blocklist-ip-filtering-guide.html)를 함께 참고하세요.

## SocketAddress.parse가 필요한 상황

### H3. host:port 문자열을 직접 split하면 IPv6에서 깨진다

가장 흔한 실수는 아래처럼 콜론 기준으로 문자열을 나누는 방식입니다.

```js
function parseEndpoint(input) {
  const [host, port] = input.split(':');

  return {
    host,
    port: Number(port)
  };
}

console.log(parseEndpoint('203.0.113.10:443'));
```

IPv4 예제에서는 동작해 보입니다.
하지만 IPv6 주소는 콜론을 여러 개 포함합니다.

```js
console.log(parseEndpoint('[2001:db8::1]:443'));
```

이 코드는 주소와 포트를 정확히 나누지 못합니다.
정규식으로 직접 보완할 수도 있지만, 네트워크 주소 문법을 문자열 처리로 계속 확장하는 것은 유지보수에 좋지 않습니다.

`SocketAddress.parse()`는 이 부분을 표준 API로 표현하게 해 줍니다.
입력 문자열이 유효한 소켓 주소 형식이면 객체를 돌려주고, 아니면 `undefined`를 돌려줍니다.

### H3. 검증 실패를 예외가 아니라 조건문으로 다룬다

설정 파일이나 사용자 입력은 실패가 자주 발생하는 경로입니다.
이런 곳에서는 잘못된 값이 들어올 때마다 예외를 던지고 잡는 구조보다, 파싱 가능 여부를 조건문으로 다루는 편이 읽기 쉽습니다.

```js
import { SocketAddress } from 'node:net';

function parseListenAddress(input) {
  const address = SocketAddress.parse(input);

  if (!address) {
    return null;
  }

  return {
    address: address.address,
    port: address.port,
    family: address.family
  };
}

console.log(parseListenAddress('203.0.113.10:443'));
console.log(parseListenAddress('[2001:db8::1]:443'));
console.log(parseListenAddress('not-an-address'));
```

이 패턴은 [Node.js URL.canParse 가이드](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)에서 다룬 흐름과 비슷합니다.
먼저 문법적으로 파싱 가능한지 확인하고, 그다음 서비스 정책에 맞는지 별도로 검사합니다.

## 기본 사용법

### H3. node:net에서 SocketAddress를 가져온다

`SocketAddress.parse()`는 `node:net` 모듈의 `SocketAddress` 클래스에 있는 정적 메서드입니다.
성공하면 `SocketAddress` 인스턴스를 반환합니다.

```js
import { SocketAddress } from 'node:net';

const endpoint = SocketAddress.parse('203.0.113.10:8080');

if (endpoint) {
  console.log(endpoint.address);
  console.log(endpoint.port);
  console.log(endpoint.family);
}
```

`address`는 IP 주소 문자열, `port`는 숫자 포트, `family`는 주소 계열을 나타냅니다.
운영 코드에서는 이 값을 바로 로그에 길게 남기기보다, 필요한 필드만 구조화해서 남기는 편이 좋습니다.

```js
function endpointSummary(endpoint) {
  return {
    family: endpoint.family,
    port: endpoint.port,
    address: endpoint.address
  };
}
```

예제에는 문서용 IP 대역을 사용했습니다.
실제 운영 로그나 블로그 예제에는 사내 IP, 고객 IP, 내부 호스트명을 그대로 싣지 않는 것이 좋습니다.

### H3. IPv6 주소는 대괄호 형식을 사용한다

포트가 붙은 IPv6 주소는 대괄호로 주소 부분을 감싸는 형식이 일반적입니다.
이렇게 해야 주소 안의 콜론과 포트 앞의 콜론을 구분할 수 있습니다.

```js
import { SocketAddress } from 'node:net';

const endpoint = SocketAddress.parse('[2001:db8::1]:443');

if (endpoint) {
  console.log(endpoint.address);
  console.log(endpoint.port);
  console.log(endpoint.family);
}
```

반대로 사람이 적은 설정 값이 `2001:db8::1:443`처럼 모호하다면, 코드가 추측해서 고치기보다 잘못된 설정으로 거부하는 편이 안전합니다.
네트워크 주소는 "대충 맞는 값"을 받아들이면 나중에 접근 제어, 연결 대상, 로그 분석에서 더 큰 혼란을 만듭니다.

### H3. 포트가 없는 입력도 정책으로 분리한다

`SocketAddress.parse()`는 IP 주소와 선택적 포트를 담은 입력을 파싱합니다.
하지만 서비스에서는 포트가 반드시 필요할 수도 있고, 기본 포트를 채워도 되는 경우도 있습니다.
이 기준은 API의 책임이 아니라 애플리케이션 정책입니다.

```js
import { SocketAddress } from 'node:net';

function parseRequiredPort(input) {
  const endpoint = SocketAddress.parse(input);

  if (!endpoint || endpoint.port === 0) {
    return null;
  }

  return endpoint;
}
```

위 예제처럼 포트가 필요한 설정이라면 파싱 결과에 한 번 더 정책 검증을 붙입니다.
반대로 포트가 없을 때 기본값을 채우는 정책이라면 그 의도를 함수 이름에 드러내는 편이 좋습니다.

```js
function parseEndpointWithDefaultPort(input, defaultPort) {
  const endpoint = SocketAddress.parse(input);

  if (!endpoint) {
    return null;
  }

  return {
    address: endpoint.address,
    family: endpoint.family,
    port: endpoint.port || defaultPort
  };
}
```

중요한 점은 파싱과 정책을 같은 코드에 뒤섞지 않는 것입니다.
파싱 함수는 "주소로 읽을 수 있는가"를 확인하고, 정책 함수는 "우리 서비스에서 허용할 값인가"를 판단하게 두면 테스트가 쉬워집니다.

## URL 검증과 SocketAddress 검증의 차이

### H3. URL은 리소스 위치, SocketAddress는 네트워크 주소다

`https://example.com:443/path` 같은 값은 URL입니다.
반면 `203.0.113.10:443`이나 `[2001:db8::1]:443`은 네트워크 소켓 주소에 가깝습니다.
두 값은 비슷해 보일 수 있지만 검증 기준이 다릅니다.

```js
const url = new URL('https://example.com:443/api');

console.log(url.hostname);
console.log(url.port);
console.log(url.pathname);
```

URL에는 프로토콜, 호스트, 포트, 경로, 쿼리, 해시가 들어갈 수 있습니다.
`SocketAddress.parse()`는 이런 URL 전체를 처리하려는 도구가 아닙니다.
IP 주소와 포트 중심의 낮은 수준 입력을 다룰 때 쓰는 편이 자연스럽습니다.

외부 콜백 URL, 리다이렉트 URL, 웹훅 URL을 검증한다면 `URL.canParse()`와 `new URL()` 조합이 더 적합합니다.
반대로 TCP 연결 대상, 허용 IP 설정, 프록시 바인딩 주소처럼 IP와 포트만 다룬다면 `SocketAddress.parse()`가 더 직접적입니다.

### H3. 호스트명 허용 여부를 명확히 정한다

운영 설정에서는 `api.example.com:443` 같은 호스트명을 받고 싶은 경우도 있습니다.
하지만 `SocketAddress.parse()`는 IP 주소 기반 소켓 주소를 다루는 API입니다.
호스트명을 허용하려면 DNS 해석, 캐시, 실패 처리, 재시도, 보안 정책이 추가로 필요합니다.

그래서 설정을 설계할 때는 아래처럼 경계를 명확히 나누는 편이 좋습니다.

- IP 주소만 허용한다면 `SocketAddress.parse()`로 검증한다.
- URL을 허용한다면 `URL.canParse()`와 URL 객체로 검증한다.
- 호스트명을 허용한다면 DNS 해석과 접근 정책을 별도 단계로 둔다.

이 구분을 문서화해 두면 운영 중 "왜 이 값은 안 들어가나요?" 같은 질문이 줄어듭니다.
특히 방화벽, 프록시, 웹훅 수신 서버처럼 네트워크 경계에 가까운 설정일수록 모호한 입력을 줄이는 것이 좋습니다.

## net.BlockList와 함께 쓰기

### H3. 파싱한 주소를 접근 규칙에 넘긴다

`SocketAddress.parse()`는 입력을 구조화하고, `net.BlockList`는 주소가 규칙에 포함되는지 확인합니다.
두 API를 함께 쓰면 설정 문자열에서 주소를 안전하게 뽑아 접근 제어 규칙과 연결할 수 있습니다.

```js
import { BlockList, SocketAddress } from 'node:net';

const allowed = new BlockList();
allowed.addSubnet('203.0.113.0', 24);

export function isAllowedEndpoint(input) {
  const endpoint = SocketAddress.parse(input);

  if (!endpoint) {
    return false;
  }

  return allowed.check(endpoint.address, endpoint.family);
}

console.log(isAllowedEndpoint('203.0.113.10:443'));
console.log(isAllowedEndpoint('198.51.100.10:443'));
```

이 코드는 입력 파싱 실패와 접근 거부를 모두 `false`로 다룹니다.
API 응답이나 CLI 메시지에서는 두 경우를 구분해도 되지만, 내부 정책 함수에서는 단순한 불리언이 더 다루기 쉬울 수 있습니다.

### H3. 인증과 권한 검사를 대체하지 않는다

IP 주소 검증은 보안의 한 층일 뿐입니다.
`SocketAddress.parse()`는 문자열을 파싱하고, `BlockList`는 주소 규칙을 검사합니다.
이 둘을 붙였다고 해서 인증, 권한, 요청 서명, rate limit, 감사 로그가 필요 없어지는 것은 아닙니다.

예를 들어 관리자 API라면 보통 아래 계층을 함께 봅니다.

- 프록시나 방화벽의 네트워크 제한
- 애플리케이션의 IP 허용 규칙
- 사용자 인증과 권한 검사
- 요청 단위 감사 로그
- 비정상 접근 시 알림 기준

특히 `X-Forwarded-For` 같은 헤더를 신뢰할 때는 프록시 체인을 먼저 검증해야 합니다.
클라이언트가 임의로 보낸 헤더를 그대로 파싱해 허용 판단에 쓰면 접근 제어가 쉽게 무너집니다.
프록시 환경의 IP 판단 기준은 [Node.js net.BlockList 가이드](/development/blog/seo/2026/06/01/nodejs-net-blocklist-ip-filtering-guide.html)에서 다룬 흐름과 함께 보세요.

## 설정 파일에서 쓰는 패턴

### H3. 시작 시점에 설정을 검증한다

네트워크 설정은 애플리케이션 시작 시점에 검증하는 편이 좋습니다.
잘못된 주소가 들어왔는데 서버가 일단 떠 버리면, 실제 요청이 들어올 때까지 문제를 모를 수 있습니다.

```js
import { SocketAddress } from 'node:net';

export function loadAdminEndpoints(values) {
  const endpoints = [];

  for (const value of values) {
    const endpoint = SocketAddress.parse(value);

    if (!endpoint || endpoint.port === 0) {
      throw new Error(`invalid admin endpoint: ${value}`);
    }

    endpoints.push(endpoint);
  }

  return endpoints;
}
```

다만 에러 메시지에는 실제 내부 주소를 그대로 남길지 신중히 결정해야 합니다.
운영 로그가 외부 수집기로 전송되거나 여러 팀에 공유된다면 민감한 주소가 노출될 수 있습니다.
공개 문서나 블로그 예제에는 문서용 주소를 사용하고, 실제 서비스 로그에서는 필요한 경우 마스킹 규칙을 적용하세요.

### H3. 테스트에서는 정상 입력과 실패 입력을 같이 둔다

입력 검증 코드는 정상 케이스만 테스트하면 부족합니다.
실패 입력을 함께 넣어야 나중에 누군가 직접 split 방식으로 되돌리는 일을 막을 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { SocketAddress } from 'node:net';

test('parse socket address inputs', () => {
  const ipv4 = SocketAddress.parse('203.0.113.10:443');
  assert.equal(ipv4?.address, '203.0.113.10');
  assert.equal(ipv4?.port, 443);

  const ipv6 = SocketAddress.parse('[2001:db8::1]:443');
  assert.equal(ipv6?.port, 443);

  assert.equal(SocketAddress.parse('not-an-address'), undefined);
});
```

테스트 이름에는 "무엇을 보장하는지"를 넣는 편이 좋습니다.
단순히 `parse works`보다 `parse socket address inputs`처럼 입력 종류가 드러나는 이름이 유지보수에 유리합니다.
Node.js 내장 테스트 러너의 기본 흐름은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)에서 이어서 볼 수 있습니다.

## 운영 체크리스트

### H3. SocketAddress.parse 도입 전 확인할 것

`SocketAddress.parse()`를 운영 코드에 넣기 전에는 아래 기준을 먼저 정리하세요.

- 입력이 URL인지, IP:port인지, 호스트명인지 구분했는가?
- IPv6 입력 형식을 문서화했는가?
- 포트가 필수인지, 기본값을 채울지 정했는가?
- 파싱 실패와 정책 거부를 사용자 메시지에서 구분할지 정했는가?
- 실제 내부 IP나 고객 IP를 로그에 그대로 남기지 않는가?
- 프록시 헤더를 신뢰하기 전에 신뢰 가능한 프록시 목록을 검증하는가?
- IP 검증 외에 인증, 권한, rate limit, 감사 로그를 별도로 유지하는가?

이 체크리스트는 코드보다 운영 경계를 분명히 하는 데 목적이 있습니다.
주소 파싱 API 하나로 모든 보안 요구사항을 해결하려 하기보다, 입력 형식과 접근 정책을 작게 나눠 검증하는 것이 좋습니다.

### H3. 작게 쓰되 경계에서는 엄격하게 쓴다

`SocketAddress.parse()`는 거대한 네트워크 프레임워크가 아닙니다.
하지만 IP 주소와 포트 문자열을 직접 쪼개던 코드를 줄이고, IPv4와 IPv6 입력을 같은 흐름에서 다루게 해 주는 실용적인 도구입니다.

특히 아래 상황에서는 도입 효과가 큽니다.

- 환경 변수로 바인딩 주소를 받는다.
- 관리자 도구에서 허용 엔드포인트 목록을 관리한다.
- TCP 클라이언트 설정을 JSON으로 읽는다.
- IP 기반 접근 규칙과 포트 설정을 함께 검증한다.
- 테스트에서 네트워크 입력 실패 케이스를 재현하고 싶다.

정리하면, Node.js에서 IP와 포트 입력을 다룰 때는 먼저 `SocketAddress.parse()`로 구조화하고, 그다음 서비스 정책을 별도 함수로 검증하는 흐름이 가장 깔끔합니다.
URL 검증은 `URL.canParse()`, IP 규칙 검사는 `net.BlockList`, 테스트는 `node:test`로 나누면 네트워크 입력 경계가 훨씬 읽기 쉬워집니다.

## 함께 읽기

- [Node.js net.BlockList 가이드: IP 허용·차단 규칙을 코드에서 안전하게 관리하는 법](/development/blog/seo/2026/06/01/nodejs-net-blocklist-ip-filtering-guide.html)
- [Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)
- [Node.js test runner 가이드: 내장 테스트 러너로 단위 테스트 시작하기](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)

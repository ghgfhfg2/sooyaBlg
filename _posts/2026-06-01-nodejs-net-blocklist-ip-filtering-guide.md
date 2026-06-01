---
layout: post
title: "Node.js net.BlockList 가이드: IP 허용·차단 규칙을 코드에서 안전하게 관리하는 법"
date: 2026-06-01 20:00:00 +0900
lang: ko
translation_key: nodejs-net-blocklist-ip-filtering-guide
permalink: /development/blog/seo/2026/06/01/nodejs-net-blocklist-ip-filtering-guide.html
alternates:
  ko: /development/blog/seo/2026/06/01/nodejs-net-blocklist-ip-filtering-guide.html
  x_default: /development/blog/seo/2026/06/01/nodejs-net-blocklist-ip-filtering-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, net, blocklist, ip-filtering, security, backend]
description: "Node.js net.BlockList로 IP 주소, 범위, 서브넷 기반 허용·차단 규칙을 코드에서 관리하는 방법을 정리했습니다. 기본 사용법, HTTP 요청 필터링, 프록시 환경 주의점, 테스트와 운영 체크리스트까지 예제로 설명합니다."
---

관리자 API, 내부 도구, 웹훅 수신 서버를 만들다 보면 특정 IP 대역만 허용하거나 위험한 주소를 빠르게 차단해야 하는 상황이 생깁니다.
이때 모든 판단을 문자열 비교나 정규식으로 처리하면 IPv4, IPv6, 범위, 서브넷을 다루는 과정에서 실수하기 쉽습니다.

Node.js에는 `node:net` 모듈의 `BlockList`가 있습니다.
`BlockList`는 주소 하나, 주소 범위, 서브넷을 규칙으로 등록하고 주어진 IP가 그 규칙에 포함되는지 확인하는 도구입니다.

이 글에서는 `net.BlockList`의 기본 사용법, HTTP 서버에서 요청 IP를 검사하는 패턴, 프록시 환경에서 조심할 점, 운영 중 규칙을 관리하는 기준을 정리합니다.
경로 기반 라우팅이나 URL 검증이 함께 필요하다면 [Node.js URLPattern 가이드](/development/blog/seo/2026/05/03/nodejs-urlpattern-routing-validation-guide.html)와 연결해서 보면 접근 제어 경계를 나누기 쉽습니다.

## net.BlockList가 필요한 상황

### H3. IP 규칙은 문자열 비교보다 구조화된 API가 낫다

IP 주소는 겉으로는 단순한 문자열처럼 보이지만 실제로는 비교 규칙이 까다롭습니다.
주소 하나만 비교할 때는 `remoteAddress === '203.0.113.10'` 같은 코드가 동작할 수 있습니다.
하지만 아래 요구사항이 생기면 문자열 비교는 금방 불안해집니다.

- 특정 IPv4 주소 하나만 차단한다.
- 사내 VPN 대역만 관리자 API에 접근하게 한다.
- 여러 주소 범위를 임시 차단한다.
- IPv6 주소도 같은 코드 경로로 처리한다.
- 규칙 목록을 로그나 테스트에서 확인한다.

`BlockList`는 이런 규칙을 명시적으로 표현합니다.
주소는 주소로, 범위는 범위로, 서브넷은 서브넷으로 등록하므로 코드의 의도가 더 잘 드러납니다.

### H3. 애플리케이션 방어의 한 층으로 사용한다

IP 필터링은 보안의 전부가 아닙니다.
인증, 권한, 요청 서명, rate limit, 방화벽, WAF 같은 다른 장치와 함께 써야 합니다.

그래도 애플리케이션 코드 안에서 IP 규칙을 한 번 더 확인하면 아래 상황에서 도움이 됩니다.

- 관리자 엔드포인트를 특정 네트워크에서만 열고 싶다.
- 웹훅 수신 서버에 임시 차단 규칙을 추가하고 싶다.
- 사내 도구가 실수로 외부에 노출됐을 때 방어선을 하나 더 두고 싶다.
- 테스트 환경에서 접근 제어 규칙을 재현하고 싶다.

다만 규칙 변경이 잦거나 대규모 차단 목록을 운영한다면 애플리케이션 코드보다 프록시, 로드밸런서, 방화벽 계층에서 처리하는 편이 더 적합할 수 있습니다.
`BlockList`는 작고 명확한 규칙을 코드와 테스트로 관리할 때 특히 실용적입니다.

## 기본 사용법

### H3. 주소 하나를 등록하고 확인한다

가장 단순한 형태는 `addAddress()`로 주소를 등록하고 `check()`로 포함 여부를 확인하는 것입니다.

```js
import { BlockList } from 'node:net';

const blocked = new BlockList();

blocked.addAddress('203.0.113.10');

console.log(blocked.check('203.0.113.10')); // true
console.log(blocked.check('203.0.113.11')); // false
```

차단 목록으로 쓰면 `check()`가 `true`일 때 요청을 거부하면 됩니다.
반대로 허용 목록으로 쓰고 싶다면 `check()`가 `false`일 때 거부하는 방식으로 정책을 바꾸면 됩니다.

```js
import { BlockList } from 'node:net';

const allowed = new BlockList();

allowed.addAddress('203.0.113.10');

function assertAllowedIp(ip) {
  if (!allowed.check(ip)) {
    throw new Error('ip_not_allowed');
  }
}
```

같은 자료구조를 차단 목록과 허용 목록에 모두 쓸 수 있지만, 변수명은 정책을 분명히 드러내야 합니다.
`list`, `rules`처럼 중립적인 이름만 쓰면 `true`가 허용인지 차단인지 읽을 때마다 확인해야 합니다.

### H3. 범위와 서브넷을 등록한다

주소 하나가 아니라 대역을 다룰 때는 `addRange()`와 `addSubnet()`을 사용할 수 있습니다.

```js
import { BlockList } from 'node:net';

const officeNetwork = new BlockList();

officeNetwork.addRange('203.0.113.10', '203.0.113.20');
officeNetwork.addSubnet('198.51.100.0', 24);

console.log(officeNetwork.check('203.0.113.15')); // true
console.log(officeNetwork.check('198.51.100.42')); // true
console.log(officeNetwork.check('192.0.2.10')); // false
```

범위는 시작 주소와 끝 주소를 기준으로 판단하고, 서브넷은 주소와 prefix 길이를 기준으로 판단합니다.
운영 규칙을 문서화할 때도 같은 단어를 쓰면 좋습니다.
예를 들어 "203.0.113.10부터 203.0.113.20까지"와 "198.51.100.0/24"는 서로 다른 종류의 규칙입니다.

### H3. IPv4와 IPv6 타입을 명시할 수 있다

규칙을 추가하거나 검사할 때 주소 타입을 명시할 수 있습니다.
대부분의 단순 IPv4 예제에서는 생략해도 되지만, IPv6를 함께 다루는 서비스라면 타입을 명시하는 편이 읽기 좋습니다.

```js
import { BlockList } from 'node:net';

const blocked = new BlockList();

blocked.addAddress('2001:db8::1', 'ipv6');
blocked.addSubnet('203.0.113.0', 24, 'ipv4');

console.log(blocked.check('2001:db8::1', 'ipv6'));
console.log(blocked.check('203.0.113.25', 'ipv4'));
```

운영 로그에도 IP 문자열만 남기지 말고 가능하면 주소 타입과 정책 이름을 함께 남기세요.
문제가 생겼을 때 "어떤 규칙 때문에 거부됐는지"를 추적하기 쉬워집니다.

## HTTP 서버에서 요청 IP 검사하기

### H3. remoteAddress를 먼저 확인한다

Node.js 기본 HTTP 서버에서는 `req.socket.remoteAddress`로 연결한 클라이언트 주소를 확인할 수 있습니다.
프록시가 없는 단순 서버라면 이 값을 기준으로 필터링할 수 있습니다.

```js
import http from 'node:http';
import { BlockList } from 'node:net';

const blockedIps = new BlockList();
blockedIps.addSubnet('203.0.113.0', 24);

const server = http.createServer((req, res) => {
  const remoteAddress = req.socket.remoteAddress;

  if (remoteAddress && blockedIps.check(remoteAddress)) {
    res.writeHead(403, { 'content-type': 'application/json' });
    res.end(JSON.stringify({ error: 'forbidden' }));
    return;
  }

  res.writeHead(200, { 'content-type': 'text/plain; charset=utf-8' });
  res.end('ok');
});

server.listen(3000);
```

이 코드는 구조는 단순하지만 정책이 분명합니다.
차단 목록에 포함된 주소는 애플리케이션 로직에 들어가기 전에 403으로 종료합니다.

### H3. 관리자 API는 허용 목록 방식이 더 안전할 때가 많다

공개 API에서는 차단 목록이 현실적일 수 있지만, 관리자 API나 내부 도구는 허용 목록이 더 안전한 경우가 많습니다.
허용된 네트워크가 아니면 기본적으로 거부하는 방식입니다.

```js
import { BlockList } from 'node:net';

const adminAllowedIps = new BlockList();

adminAllowedIps.addSubnet('198.51.100.0', 24);
adminAllowedIps.addAddress('203.0.113.10');

export function isAdminIpAllowed(ip) {
  return Boolean(ip && adminAllowedIps.check(ip));
}

export function guardAdminRequest(req, res) {
  if (!isAdminIpAllowed(req.socket.remoteAddress)) {
    res.writeHead(403, { 'content-type': 'application/json' });
    res.end(JSON.stringify({ error: 'admin_ip_not_allowed' }));
    return false;
  }

  return true;
}
```

IP 허용 목록은 인증을 대체하지 않습니다.
관리자 기능에는 여전히 로그인, 권한 확인, 감사 로그가 필요합니다.
IP 규칙은 "어디서 접속했는가"를 보는 추가 조건으로 두는 편이 안전합니다.

## 프록시 환경에서 조심할 점

### H3. X-Forwarded-For를 무조건 믿으면 안 된다

로드밸런서나 리버스 프록시 뒤에 Node.js 서버가 있다면 `req.socket.remoteAddress`는 실제 사용자 IP가 아니라 프록시 IP일 수 있습니다.
이때 `X-Forwarded-For` 같은 헤더를 참고해야 할 수 있지만, 이 헤더는 클라이언트가 직접 보낼 수도 있습니다.

따라서 아래 순서를 지키는 편이 안전합니다.

1. 신뢰할 수 있는 프록시에서 온 요청인지 먼저 확인한다.
2. 신뢰된 프록시 요청일 때만 전달 헤더를 읽는다.
3. 헤더 안의 여러 주소 중 어떤 위치를 사용할지 운영 환경 기준으로 고정한다.
4. 결정된 클라이언트 IP를 `BlockList`로 검사한다.

예시는 아래처럼 구성할 수 있습니다.

```js
import { BlockList } from 'node:net';

const trustedProxies = new BlockList();
trustedProxies.addSubnet('10.0.0.0', 8);

function firstForwardedFor(value) {
  if (!value) {
    return null;
  }

  return value.split(',')[0]?.trim() || null;
}

export function getClientIp(req) {
  const peerIp = req.socket.remoteAddress;

  if (peerIp && trustedProxies.check(peerIp)) {
    return firstForwardedFor(req.headers['x-forwarded-for']) || peerIp;
  }

  return peerIp;
}
```

실제 서비스에서는 프록시 체인, IPv6 표기, 사설망 주소 정책을 더 엄격히 정해야 합니다.
중요한 것은 전달 헤더를 읽기 전에 "이 요청이 신뢰된 프록시에서 왔는가"를 먼저 확인하는 것입니다.

### H3. 접근 거부 로그에는 민감정보를 남기지 않는다

IP 필터링 로그는 보안 분석에 유용하지만, 요청 본문이나 인증 토큰을 함께 남기면 위험합니다.
거부 로그에는 필요한 최소 정보만 남기세요.

```js
function logRejectedRequest({ ip, path, policy }) {
  console.warn(JSON.stringify({
    event: 'request_rejected_by_ip_policy',
    ip,
    path,
    policy
  }));
}
```

웹훅 검증처럼 요청 서명까지 함께 다룬다면 [Node.js Web Crypto HMAC 웹훅 서명 검증 가이드](/development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html)를 함께 참고할 수 있습니다.
IP 제한은 네트워크 조건이고, 서명 검증은 메시지 무결성 조건입니다.
둘은 서로 보완 관계로 보는 편이 좋습니다.

## 규칙을 설정 파일로 분리하기

### H3. 운영 규칙은 코드와 데이터의 경계를 나눈다

IP 규칙이 자주 바뀐다면 소스 코드에 직접 박아 두는 방식은 불편합니다.
JSON 같은 설정 파일로 분리한 뒤 시작 시점에 읽어 `BlockList`를 만드는 구조가 더 운영하기 쉽습니다.

```js
import { readFile } from 'node:fs/promises';
import { BlockList } from 'node:net';

export async function loadBlockList(path) {
  const raw = JSON.parse(await readFile(path, 'utf8'));
  const blockList = new BlockList();

  for (const address of raw.addresses ?? []) {
    blockList.addAddress(address.value, address.type);
  }

  for (const range of raw.ranges ?? []) {
    blockList.addRange(range.start, range.end, range.type);
  }

  for (const subnet of raw.subnets ?? []) {
    blockList.addSubnet(subnet.net, subnet.prefix, subnet.type);
  }

  return blockList;
}
```

설정 파일 예시는 아래처럼 단순하게 둘 수 있습니다.

```json
{
  "addresses": [
    { "value": "203.0.113.10", "type": "ipv4" }
  ],
  "ranges": [
    { "start": "203.0.113.20", "end": "203.0.113.30", "type": "ipv4" }
  ],
  "subnets": [
    { "net": "198.51.100.0", "prefix": 24, "type": "ipv4" }
  ]
}
```

여기서도 검증이 중요합니다.
운영에서는 파일을 읽은 뒤 새 `BlockList`를 완성하고, 모든 규칙이 정상 등록된 경우에만 교체하는 방식이 안전합니다.
설정 리로드 구조는 [Node.js SIGHUP 설정 리로드 가이드](/development/blog/seo/2026/06/01/nodejs-sighup-config-reload-guide.html)의 패턴을 그대로 가져올 수 있습니다.

### H3. 규칙 이름은 별도로 관리한다

`BlockList`는 어떤 주소가 규칙에 포함되는지 확인하는 데 집중합니다.
"왜 이 주소가 막혔는가"까지 설명하려면 정책 이름이나 출처를 별도 데이터로 관리하는 편이 좋습니다.

예를 들어 작은 규모에서는 아래처럼 정책 단위를 나눌 수 있습니다.

```js
const policies = [
  {
    name: 'admin-office-network',
    mode: 'allow',
    list: adminAllowedIps
  },
  {
    name: 'temporary-abuse-block',
    mode: 'deny',
    list: temporaryBlockedIps
  }
];
```

이렇게 하면 로그에 `policy.name`을 남길 수 있고, 테스트에서도 어떤 정책이 적용됐는지 확인하기 쉽습니다.
규칙이 커지면 애플리케이션 내부 목록보다 네트워크 계층의 정책 관리 도구로 옮기는 것이 좋습니다.

## 테스트와 운영 체크리스트

### H3. 경계값을 테스트한다

IP 범위와 서브넷은 경계값 실수가 자주 생깁니다.
테스트에는 범위 안쪽 주소뿐 아니라 시작 주소, 끝 주소, 바로 바깥 주소를 넣어야 합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { BlockList } from 'node:net';

test('office subnet allows only expected range', () => {
  const list = new BlockList();
  list.addSubnet('198.51.100.0', 24);

  assert.equal(list.check('198.51.100.1'), true);
  assert.equal(list.check('198.51.100.255'), true);
  assert.equal(list.check('198.51.101.1'), false);
});
```

내장 테스트 러너 구조가 필요하다면 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 보면 충분합니다.
IP 정책 테스트는 외부 네트워크를 실제로 호출하지 않아도 되므로 빠르고 결정적으로 만들기 좋습니다.

### H3. 배포 전 확인할 항목

`net.BlockList`를 운영 코드에 넣기 전에는 아래 항목을 확인하세요.

- 차단 목록인지 허용 목록인지 변수명과 함수명에서 드러나는가?
- 프록시 뒤에서 실행된다면 신뢰할 프록시 기준이 명확한가?
- `X-Forwarded-For` 같은 전달 헤더를 무조건 신뢰하지 않는가?
- IPv4와 IPv6를 모두 다루는 서비스라면 타입 기준을 정했는가?
- 거부 로그에 토큰, 쿠키, 요청 본문 같은 민감정보가 남지 않는가?
- 중요한 관리자 기능은 IP 필터링 외에 인증과 권한 확인도 거치는가?
- 규칙 변경 절차와 롤백 방법이 문서화돼 있는가?

작게 시작한다면 관리자 API 허용 목록처럼 범위가 분명한 곳부터 적용하는 편이 좋습니다.
공개 트래픽 전체에 큰 차단 규칙을 넣는 작업은 영향 범위가 넓으므로 프록시나 방화벽 정책과 함께 검토해야 합니다.

### H3. 내부 링크로 이어서 볼 글

- [Node.js URLPattern 가이드: 라우팅과 URL 검증을 더 단순하게 만드는 법](/development/blog/seo/2026/05/03/nodejs-urlpattern-routing-validation-guide.html)
- [Node.js Web Crypto HMAC 웹훅 서명 검증 가이드](/development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html)
- [Node.js SIGHUP 설정 리로드 가이드: 서버 재시작 없이 안전하게 설정을 반영하는 법](/development/blog/seo/2026/06/01/nodejs-sighup-config-reload-guide.html)

## 마무리

`net.BlockList`는 화려한 보안 제품이 아니라, Node.js 코드 안에서 IP 규칙을 명확하게 표현하게 해 주는 작은 기본 도구입니다.
주소, 범위, 서브넷을 직접 문자열로 비교하는 대신 구조화된 API로 다루면 실수할 여지가 줄어듭니다.

운영에서 중요한 것은 이 기능을 과신하지 않는 것입니다.
IP 필터링은 인증과 권한, 프록시 신뢰 경계, 요청 서명, 로그 정책과 함께 설계해야 효과가 있습니다.
작고 명확한 규칙부터 `BlockList`로 테스트 가능하게 만들면 Node.js 애플리케이션의 접근 제어가 훨씬 읽기 쉬워집니다.

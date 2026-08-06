---
layout: post
title: "Node.js dns.promises.Resolver 가이드: 외부 API별 DNS 타임아웃을 분리하는 법"
date: 2026-08-06 20:00:00 +0900
lang: ko
translation_key: nodejs-dns-promises-resolver-timeout-guide
permalink: /development/blog/seo/2026/08/06/nodejs-dns-promises-resolver-timeout-guide.html
alternates:
  ko: /development/blog/seo/2026/08/06/nodejs-dns-promises-resolver-timeout-guide.html
  x_default: /development/blog/seo/2026/08/06/nodejs-dns-promises-resolver-timeout-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, dns, resolver, timeout, external-api, reliability, backend, javascript]
description: "Node.js dns.promises.Resolver로 외부 API별 DNS 서버, timeout, tries 정책을 분리하는 방법을 정리합니다. dns.lookup과 resolve의 차이, cancel 처리, 오류 코드 분류, 운영 로그와 테스트 체크리스트까지 실무 예제로 설명합니다."
---

외부 API 호출이 느려졌을 때 HTTP client의 timeout만 늘리면 원인을 가리기 쉽습니다.
실제로는 TCP 연결 전에 DNS 질의가 지연되거나, 특정 이름 서버만 불안정하거나, 사내 네트워크와 퍼블릭 DNS 정책이 섞여 문제가 커지는 경우가 있습니다.
이럴 때 Node.js의 `dns.promises.Resolver`를 쓰면 전역 DNS 설정을 건드리지 않고 호출 대상별 조회 정책을 분리할 수 있습니다.

핵심은 **서비스 전체의 DNS 동작을 바꾸기보다, 위험도가 높은 외부 경계에만 별도 Resolver와 관측 지점을 두는 것**입니다.
공식 문서 기준 `Resolver`는 `timeout`, `tries`, `maxTimeout` 옵션을 받을 수 있고, 인스턴스별로 `setServers()`를 호출해 전역 설정과 독립적인 DNS 서버 목록을 사용할 수 있습니다.
이 글에서는 `dns.lookup()`과 `resolve4()`의 차이, 외부 API별 DNS timeout 설계, 장애 로그와 테스트 기준을 함께 정리합니다.

DNS 지연 자체를 먼저 진단해야 한다면 [Node.js DNS Lookup 지연 가이드](/development/blog/seo/2026/04/22/nodejs-dns-lookup-latency-caching-guide.html)를 참고하세요.
외부 API 연결 풀까지 같이 튜닝한다면 [Node.js HTTP Agent maxSockets 가이드](/development/blog/seo/2026/04/23/nodejs-http-agent-maxsockets-maxtotalsockets-guide.html)가 이어서 도움이 됩니다.
호출 전체의 deadline 전파는 [Node.js timeout budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와 함께 설계하는 편이 좋습니다.

## dns.promises.Resolver가 필요한 상황

### H3. 전역 DNS 설정을 바꾸면 영향 범위가 너무 넓다

`dns.setServers()`는 프로세스의 DNS resolver 설정을 바꿉니다.
작은 스크립트에서는 단순할 수 있지만, 여러 외부 API와 내부 서비스 호출이 함께 있는 서버에서는 영향 범위가 큽니다.
한 API의 장애 대응을 위해 DNS 서버를 바꿨다가 다른 호출 경로의 동작까지 달라질 수 있습니다.

`Resolver` 인스턴스를 만들면 이 범위를 좁힐 수 있습니다.

```js
import { Resolver } from 'node:dns/promises';

const paymentDns = new Resolver({
  timeout: 800,
  tries: 2,
  maxTimeout: 1200
});

paymentDns.setServers(['1.1.1.1', '8.8.8.8']);

export async function resolvePaymentHost(hostname) {
  return paymentDns.resolve4(hostname, { ttl: true });
}
```

이 코드는 결제 API hostname을 조회할 때만 별도 DNS 서버와 timeout 정책을 씁니다.
다른 DB 연결, 내부 서비스 discovery, 기본 `fetch()` 동작에는 직접 영향을 주지 않습니다.

### H3. lookup과 resolve는 같은 문제가 아니다

Node.js에서 `dns.lookup()`은 운영체제의 이름 해석 흐름을 사용합니다.
`/etc/hosts`, OS resolver, 로컬 네트워크 정책과 연결될 수 있어서 일반적인 socket 연결과 가까운 동작을 합니다.
반면 `resolve4()`, `resolve6()`, `resolveTxt()` 같은 메서드는 DNS 프로토콜로 레코드를 조회합니다.

그래서 목적을 나눠야 합니다.

- HTTP client가 실제로 어떤 주소로 연결되는지 확인하려면 `lookup()` 관점이 중요합니다.
- 특정 DNS 서버에서 A, AAAA, TXT, SRV 레코드가 어떻게 보이는지 점검하려면 `Resolver`와 `resolve*()`가 더 적합합니다.
- 외부 API별 DNS 장애를 관측하려면 `resolve4(host, { ttl: true })`처럼 TTL까지 남기는 방식이 좋습니다.

서비스 호출 자체를 강제로 특정 IP로 보내려는 용도로 `resolve4()` 결과를 무심코 HTTP client에 주입하면 TLS 인증서, SNI, 프록시 정책이 꼬일 수 있습니다.
실제 요청 경로를 바꾸기 전에는 DNS 관측, fallback, 연결 정책을 분리해서 검토해야 합니다.

## 외부 API별 DNS timeout 설계

### H3. Resolver를 의존성으로 만들고 호출부는 정책 이름만 본다

운영 코드에서는 DNS 서버 주소와 timeout 숫자가 handler 곳곳에 흩어지면 유지보수가 어려워집니다.
외부 경계별 resolver를 만들고, 호출부에는 의도를 드러내는 함수만 노출하는 편이 안전합니다.

```js
import { Resolver } from 'node:dns/promises';

function createExternalResolver({ servers, timeoutMs, tries }) {
  const resolver = new Resolver({
    timeout: timeoutMs,
    tries,
    maxTimeout: timeoutMs * 2
  });

  resolver.setServers(servers);
  return resolver;
}

const resolvers = {
  payment: createExternalResolver({
    servers: ['1.1.1.1', '8.8.8.8'],
    timeoutMs: 700,
    tries: 2
  }),
  analytics: createExternalResolver({
    servers: ['8.8.8.8'],
    timeoutMs: 500,
    tries: 1
  })
};

export async function resolveExternalApi(kind, hostname) {
  const resolver = resolvers[kind];

  if (!resolver) {
    throw new Error(`Unknown resolver policy: ${kind}`);
  }

  return resolver.resolve4(hostname, { ttl: true });
}
```

이 구조는 장애 대응 때 특히 유리합니다.
결제 API DNS timeout은 길게 두고, 분석 이벤트 API는 짧게 실패시키는 식으로 정책을 다르게 가져갈 수 있습니다.
또 로그에서 `kind`를 기준으로 어느 외부 경계가 흔들리는지 바로 볼 수 있습니다.

### H3. timeout은 HTTP timeout과 별도로 측정한다

DNS 조회 시간과 HTTP 요청 시간을 합쳐서 하나의 "외부 호출 지연"으로만 보면 대응이 늦어집니다.
DNS가 느린지, TCP 연결이 느린지, 응답 본문이 늦는지 분리해서 기록해야 합니다.

```js
import { performance } from 'node:perf_hooks';

export async function resolveWithTiming({ kind, hostname }) {
  const startedAt = performance.now();

  try {
    const records = await resolveExternalApi(kind, hostname);

    return {
      ok: true,
      hostname,
      records,
      durationMs: Math.round(performance.now() - startedAt)
    };
  } catch (error) {
    return {
      ok: false,
      hostname,
      code: error.code ?? 'UNKNOWN_DNS_ERROR',
      durationMs: Math.round(performance.now() - startedAt)
    };
  }
}
```

운영 로그에는 전체 record를 과하게 남기기보다 record 수, 최소 TTL, 오류 코드, 소요 시간을 남기는 편이 좋습니다.
도메인 자체가 민감할 수 있는 내부 시스템이라면 hostname도 allowlist 이름이나 해시로 줄여 기록하세요.

## 장애 처리와 cancel 기준

### H3. DNS 오류 코드는 사용자 메시지보다 운영 분류에 가깝다

DNS 조회 실패는 `ENOTFOUND`, `ETIMEOUT`, `ECANCELLED`, `ESERVFAIL`처럼 코드로 분류할 수 있습니다.
하지만 이 코드를 사용자 응답에 그대로 노출할 필요는 없습니다.
사용자에게는 "일시적으로 외부 서비스 연결을 확인할 수 없습니다" 정도로 표현하고, 내부 로그에서만 자세히 구분하는 편이 안전합니다.

```js
export function classifyDnsError(error) {
  switch (error.code) {
    case 'ENOTFOUND':
      return 'hostname_not_found';
    case 'ETIMEOUT':
      return 'dns_timeout';
    case 'ECANCELLED':
      return 'dns_cancelled';
    case 'ESERVFAIL':
      return 'dns_server_failure';
    default:
      return 'dns_unknown_failure';
  }
}
```

`ENOTFOUND`는 hostname이 없을 때뿐 아니라 다른 조회 실패 상황에서도 나타날 수 있습니다.
따라서 한 번의 코드만 보고 "도메인이 삭제됐다"고 단정하지 말고, 최근 배포, 네트워크 변경, resolver 서버 상태를 같이 확인해야 합니다.

### H3. cancel은 요청 deadline과 연결한다

사용자 요청이 이미 취소됐거나 전체 deadline을 넘었다면 DNS 조회를 계속 붙잡고 있을 이유가 없습니다.
`resolver.cancel()`은 해당 resolver의 진행 중인 질의를 취소하고 promise를 `ECANCELLED` 오류로 거절하게 만들 수 있습니다.
단, 같은 resolver 인스턴스를 여러 요청이 공유하고 있다면 한 요청 때문에 다른 요청의 DNS 질의까지 취소될 수 있습니다.

요청 단위 취소가 중요하면 짧은 생명주기의 resolver를 만들거나, DNS 조회를 별도 timeout wrapper로 감싸는 편이 더 안전합니다.

```js
import { Resolver } from 'node:dns/promises';

export async function resolveOnceWithDeadline(hostname, deadlineMs) {
  const resolver = new Resolver({ timeout: deadlineMs, tries: 1 });
  const timer = setTimeout(() => resolver.cancel(), deadlineMs);

  try {
    return await resolver.resolve4(hostname, { ttl: true });
  } finally {
    clearTimeout(timer);
  }
}
```

공유 resolver는 정책 재사용에는 좋지만 cancel 범위가 넓습니다.
반대로 요청별 resolver는 격리는 쉽지만 객체 생성과 설정 비용이 늘어납니다.
트래픽이 높은 서버라면 먼저 관측용 resolver를 공유하고, deadline이 엄격한 일부 경계에만 요청별 resolver를 적용하는 방식이 현실적입니다.

## 테스트와 운영 체크리스트

### H3. DNS 조회 코드는 네트워크 없이도 테스트 가능해야 한다

단위 테스트가 실제 DNS 서버에 의존하면 flaky test가 되기 쉽습니다.
조회 함수에 resolver를 주입할 수 있게 만들면 네트워크 없이 성공, timeout, 취소, 빈 결과를 검증할 수 있습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

async function resolveForHealthCheck(resolver, hostname) {
  const records = await resolver.resolve4(hostname, { ttl: true });
  return records.map((record) => record.address);
}

test('resolveForHealthCheck returns IPv4 addresses', async () => {
  const fakeResolver = {
    async resolve4(hostname) {
      assert.equal(hostname, 'api.example.test');
      return [{ address: '203.0.113.10', ttl: 60 }];
    }
  };

  assert.deepEqual(
    await resolveForHealthCheck(fakeResolver, 'api.example.test'),
    ['203.0.113.10']
  );
});
```

통합 테스트에서만 실제 DNS를 짧게 확인하고, CI 기본 경로에서는 fake resolver를 쓰는 편이 안정적입니다.
외부 네트워크가 막힌 환경에서도 테스트가 통과해야 배포 파이프라인을 믿을 수 있습니다.

### H3. 배포 전 확인할 항목

- 외부 API별 DNS 정책 이름이 있는가?
- 전역 `dns.setServers()` 호출을 남용하지 않는가?
- `timeout`, `tries`, `maxTimeout` 값이 HTTP timeout budget보다 작게 잡혀 있는가?
- `resolve4()`와 `lookup()`의 차이를 호출 목적에 맞게 구분했는가?
- DNS 오류 코드, duration, resolver policy를 구조화 로그로 남기는가?
- 내부 hostname, 사설 IP, 고객 식별자가 로그에 그대로 남지 않는가?
- 단위 테스트가 실제 DNS 네트워크에 의존하지 않는가?

## 마무리

`dns.promises.Resolver`는 외부 API 장애를 마법처럼 해결하는 도구가 아닙니다.
대신 DNS 조회 정책을 전역 설정에서 떼어내고, timeout과 서버 목록, 오류 분류를 호출 경계별로 관리할 수 있게 해 줍니다.

외부 호출 장애를 줄이려면 DNS, 연결 풀, HTTP timeout, retry budget을 한 번에 봐야 합니다.
그중 DNS Resolver 분리는 "어느 외부 경계에서 이름 해석이 흔들리는지"를 빠르게 좁히는 실용적인 출발점입니다.

## 함께 보면 좋은 글

- [Node.js DNS Lookup 지연 가이드](/development/blog/seo/2026/04/22/nodejs-dns-lookup-latency-caching-guide.html)
- [Node.js HTTP Agent maxSockets 가이드](/development/blog/seo/2026/04/23/nodejs-http-agent-maxsockets-maxtotalsockets-guide.html)
- [Node.js timeout budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)

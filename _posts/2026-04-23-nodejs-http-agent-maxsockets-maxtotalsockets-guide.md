---
layout: post
title: "Node.js http.Agent maxSockets, maxTotalSockets 가이드: 외부 호출 폭주를 연결 단계에서 제어하는 법"
date: 2026-04-23 08:00:00 +0900
lang: ko
translation_key: nodejs-http-agent-maxsockets-maxtotalsockets-guide
permalink: /development/blog/seo/2026/04/23/nodejs-http-agent-maxsockets-maxtotalsockets-guide.html
alternates:
  ko: /development/blog/seo/2026/04/23/nodejs-http-agent-maxsockets-maxtotalsockets-guide.html
  x_default: /development/blog/seo/2026/04/23/nodejs-http-agent-maxsockets-maxtotalsockets-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, http-agent, maxsockets, maxtotalsockets, keepalive, connection-pool, backend, reliability, performance]
description: "Node.js 외부 API 호출이 몰릴 때 http.Agent의 maxSockets, maxTotalSockets를 어떻게 잡아야 하는지, keep-alive·큐 적체·timeout과 함께 실무 기준으로 정리했습니다."
---

Node.js 서비스에서 외부 API 호출이 갑자기 몰릴 때, 애플리케이션 코드만 보고 있으면 원인을 반쯤 놓치기 쉽습니다.
실제로는 요청 로직보다 먼저 **연결을 몇 개까지 동시에 열어둘지**가 병목과 장애 반경을 크게 좌우하는 경우가 많습니다.

이때 자주 만나는 설정이 `http.Agent`의 `maxSockets`, `maxTotalSockets`입니다.
이 둘을 모르고 기본값에만 기대면 순간 트래픽 때 연결이 과하게 늘거나, 반대로 필요한 처리량을 못 내는 일이 생길 수 있습니다.

결론부터 말하면, `maxSockets`는 **호스트별 동시 연결 상한**, `maxTotalSockets`는 **에이전트 전체 동시 연결 상한**으로 이해하면 좋습니다.
실무에서는 둘을 keep-alive, timeout budget, retry 정책과 함께 봐야 안전합니다.

## 왜 http.Agent 연결 상한이 중요한가

### H3. 외부 호출 폭주는 애플리케이션보다 먼저 연결 계층에서 번진다

외부 API를 많이 호출하는 서비스는 CPU나 이벤트 루프보다 먼저 연결 수가 불어나면서 불안정해지는 경우가 있습니다.
특히 같은 시점에 여러 upstream으로 요청이 몰리면, 처리 로직보다 소켓 경쟁과 큐 적체가 먼저 눈에 띕니다.

대표적으로 아래 상황에서 잘 드러납니다.

- 같은 프로세스가 여러 외부 API를 동시에 호출하는 경우
- 배치와 사용자 요청이 같은 연결 예산을 공유하는 경우
- timeout이 길고 retry가 붙어 있어 연결 점유 시간이 늘어나는 경우
- keep-alive는 켰지만 상한값을 따로 관리하지 않은 경우

이때 연결 상한이 없으면 순간적으로 많은 연결을 열면서 downstream과 우리 프로세스 둘 다 압박할 수 있습니다.
반대로 너무 보수적으로 잡으면 큐 대기가 길어져 tail latency가 커집니다.

### H3. connection pool exhaustion은 DB에서만 생기는 문제가 아니다

풀 고갈이라고 하면 DB 커넥션만 떠올리기 쉽지만, 외부 HTTP 연결도 비슷한 문제를 일으킵니다.
응답이 느린 upstream으로 요청이 몰리면 소켓이 오래 점유되고, 그 사이 신규 요청은 대기열에 쌓입니다.

이 감각은 [Node.js Connection Pool Exhaustion 가이드](/development/blog/seo/2026/04/11/nodejs-connection-pool-exhaustion-prevention-guide.html)와 비슷합니다.
차이는 대상이 DB 풀인지 HTTP agent인지일 뿐, 핵심은 **제한 없는 동시성이 결국 전체 지연으로 번진다**는 점입니다.

## maxSockets와 maxTotalSockets는 무엇이 다를까

### H3. maxSockets는 호스트별 상한에 가깝다

`maxSockets`는 특정 호스트 기준으로 동시에 열 수 있는 소켓 수를 제한할 때 이해하기 쉽습니다.
즉 같은 upstream 하나로 요청이 몰릴 때, 그 호스트에 대해 몇 개까지 병렬 연결을 허용할지를 정하는 값에 가깝습니다.

예를 들어 결제 API 하나에 트래픽이 쏠린다면, `maxSockets`가 너무 크면 그 upstream을 향한 연결이 과하게 불어날 수 있습니다.
반대로 너무 작으면 해당 호스트 호출만 유난히 대기 시간이 길어질 수 있습니다.

핵심은 `maxSockets`가 **개별 upstream 보호와 호출 병렬성 사이의 균형점**이라는 것입니다.
한 호스트가 전체 연결 예산을 다 먹지 못하게 막는 첫 번째 장치로 보면 됩니다.

### H3. maxTotalSockets는 전체 에이전트 예산을 묶는 장치다

`maxTotalSockets`는 여러 호스트로 흩어지는 요청을 포함해 에이전트 전체에서 동시에 열 수 있는 소켓 총량을 제한할 때 유용합니다.
즉 특정 호스트 하나만이 아니라, 프로세스 전체 외부 호출의 동시 연결 예산을 묶어두는 개념입니다.

이 값이 없거나 너무 크면 이런 문제가 생길 수 있습니다.

- 부가 기능용 외부 API가 핵심 경로 연결 예산까지 잠식함
- 배치 작업이 낮 시간대 사용자 요청과 경쟁함
- retry, timeout 증가 구간에서 전체 연결 수가 빠르게 튐
- keep-alive 소켓이 누적되며 운영 지표가 흔들림

이런 이유로 `maxTotalSockets`는 [Node.js Bulkhead Pattern 가이드](/development/blog/seo/2026/03/29/nodejs-bulkhead-pattern-resource-isolation-guide.html)에서 다룬 자원 경계 설계와도 맞닿아 있습니다.
기능별 agent를 분리하지 못하더라도, 최소한 전체 상한을 두는 편이 훨씬 낫습니다.

## 어떤 기준으로 값을 잡아야 할까

### H3. downstream 처리 용량과 우리 timeout 예산을 먼저 본다

연결 상한은 큰 숫자를 넣는다고 성능이 좋아지는 설정이 아닙니다.
오히려 downstream이 감당 가능한 동시성, 평균 응답 시간, timeout budget을 같이 봐야 합니다.

실무에서는 보통 아래 질문부터 정리합니다.

1. 이 upstream이 동시에 몇 개 요청까지 안정적으로 받는가?
2. 평균이 아니라 p95, p99 지연은 어느 정도인가?
3. timeout이 길어질 때 소켓 점유 시간이 얼마나 늘어나는가?
4. 재시도까지 포함하면 실제 요청 수가 몇 배가 되는가?

즉 `maxSockets=무조건 크게`가 아니라, **느린 upstream에도 전체 시스템이 같이 잠기지 않게 하는 숫자**를 찾는 것이 목적입니다.
이 지점은 [Node.js Timeout Budget, Deadline Propagation 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와 같이 봐야 더 정확합니다.

### H3. 사용자 요청과 배치 작업은 같은 agent를 공유하지 않는 편이 좋다

같은 프로세스 안에서도 트래픽 성격은 다릅니다.
실시간 사용자 요청과 느린 배치 작업이 같은 agent를 공유하면, 한쪽의 연결 점유가 다른 쪽 응답성을 망칠 수 있습니다.

그래서 실무에서는 아래처럼 나누는 편이 안전합니다.

- 핵심 사용자 요청용 agent
- 부가 기능 호출용 agent
- 배치/동기화 작업용 agent

이렇게 나누면 `maxSockets`, `maxTotalSockets`도 성격에 맞게 따로 조정할 수 있습니다.
같은 맥락에서 [Node.js Semaphore Pattern 가이드](/development/blog/seo/2026/04/08/nodejs-semaphore-pattern-shared-resource-concurrency-guide.html)를 함께 보면, 연결 상한과 애플리케이션 동시성 제한을 어떻게 분리할지 감이 더 잘 옵니다.

## keep-alive와 함께 볼 때 주의할 점

### H3. keep-alive는 좋지만, 제한 없는 재사용은 아니다

keep-alive는 매 요청마다 새 연결을 만들지 않게 해 latency와 비용을 줄이는 데 유리합니다.
하지만 keep-alive를 켰다고 해서 연결 관리 문제가 사라지는 것은 아닙니다.

오히려 keep-alive가 켜져 있으면 유휴 소켓과 활성 소켓을 어떻게 유지할지 더 분명히 봐야 합니다.
이때 `maxSockets`, `maxTotalSockets`, `maxFreeSockets` 같은 값이 함께 의미를 가집니다.

특히 아래처럼 이해하면 덜 헷갈립니다.

- keep-alive: 연결 재사용 전략
- `maxSockets`: 활성 연결 상한
- `maxTotalSockets`: 전체 연결 총량 상한
- `maxFreeSockets`: 놀고 있는 keep-alive 소켓 보관 상한

관련해서 연결 재활용 자체는 [Node.js maxRequestsPerSocket, keep-alive connection recycling 가이드](/development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html)와 이어집니다.

### H3. DNS 병목이 있으면 연결 상한만으로는 부족할 수 있다

외부 호출이 느릴 때 연결 수만 줄인다고 끝나지 않는 경우도 있습니다.
이름 해석이 느리면 연결 전 단계에서 이미 시간이 새고 있을 수 있기 때문입니다.

즉 이런 조합으로 봐야 합니다.

- 연결 수가 너무 많은가
- 연결 재사용이 충분한가
- DNS lookup이 병목인가
- timeout과 retry가 병목을 확대하고 있지는 않은가

이 부분은 어제 글인 [Node.js DNS Lookup 지연 가이드](/development/blog/seo/2026/04/22/nodejs-dns-lookup-latency-caching-guide.html)와 직접 연결됩니다.
연결 관리와 이름 해석은 따로 보이지만, 실제 체감 latency에서는 거의 붙어서 움직입니다.

## Node.js에서 간단히 적용하는 예시

### H3. upstream 성격별로 agent를 분리하는 편이 운영이 쉽다

아래 예시는 핵심 API와 부가 기능 API에 서로 다른 연결 예산을 주는 단순한 패턴입니다.

```js
import https from 'node:https';

export const coreApiAgent = new https.Agent({
  keepAlive: true,
  maxSockets: 40,
  maxTotalSockets: 80,
  maxFreeSockets: 10,
});

export const optionalApiAgent = new https.Agent({
  keepAlive: true,
  maxSockets: 10,
  maxTotalSockets: 20,
  maxFreeSockets: 5,
});
```

핵심은 숫자 자체보다 **중요도가 다른 호출이 같은 연결 예산을 공유하지 않게 만드는 것**입니다.
부가 기능이 느려져도 핵심 경로 소켓까지 다 먹지 못하게 하는 편이 좋습니다.

### H3. agent 상한만 두고 timeout이 없으면 적체는 계속 남는다

연결 상한은 동시에 얼마나 열지 제한할 뿐, 느린 요청을 언제 정리할지는 말해주지 않습니다.
그래서 아래처럼 timeout budget과 함께 써야 의미가 생깁니다.

```js
import https from 'node:https';

const req = https.get('https://api.example.com/data', {
  agent: coreApiAgent,
  timeout: 1500,
}, (res) => {
  res.resume();
});

req.on('timeout', () => {
  req.destroy(new Error('upstream timeout'));
});
```

실무에서는 라이브러리별 인터페이스가 조금 다를 수 있지만, 원칙은 같습니다.
**연결 수 제한 + 대기 시간 제한 + 제한된 재시도**가 한 세트여야 합니다.
연결 상한만 두면 큐는 짧아져도 느린 요청이 오래 자리를 차지할 수 있습니다.

## 운영에서는 어떤 지표를 봐야 할까

### H3. 평균 latency보다 queueing과 socket saturation을 먼저 본다

`maxSockets`와 `maxTotalSockets`를 조정할 때 평균 응답 시간만 보면 판단이 늦습니다.
실제로는 아래 지표가 더 직접적입니다.

```txt
[권장 점검 항목]
- 호스트별 active socket 수
- agent 전체 active socket 수
- free socket 수와 재사용 비율
- 요청 대기 시간(queueing) 분포
- timeout 발생 비율
- retry 비율과 재시도 후 성공률
```

이 지표를 같이 보면, 상한이 너무 낮아서 대기열이 길어진 것인지, 반대로 너무 높아서 downstream을 압박하는 것인지 구분하기 쉬워집니다.

### H3. 값 하나보다 기능별 분리 여부가 더 중요할 때가 많다

운영에서 자주 보는 실수는 숫자 튜닝만 반복하는 것입니다.
하지만 정말 중요한 것은 “누가 누구의 연결 예산을 뺏고 있는가”입니다.

그래서 아래 질문이 더 중요합니다.

- 핵심 기능과 부가 기능이 같은 agent를 쓰는가?
- 느린 upstream 하나가 다른 호출까지 막고 있지 않은가?
- 배치 시간이 되면 사용자 요청 latency가 같이 튀는가?
- retry가 연결 슬롯을 추가로 오래 점유하지 않는가?

숫자 미세 조정보다 이런 경계 분리가 먼저 잡히면 훨씬 안정적입니다.

## 빠르게 판단할 때 쓰는 체크리스트

### H3. 아래 항목이 맞지 않으면 연결 상한부터 점검할 만하다

- 특정 upstream 호출만 몰릴 때 전체 latency가 같이 튀는가?
- keep-alive는 켰지만 연결 총량 상한은 없는가?
- 사용자 요청과 배치 작업이 같은 agent를 공유하는가?
- timeout 없이 연결 상한만으로 버티려 하고 있는가?
- retry budget 없이 실패 구간에서 재시도가 같이 늘어나는가?

## 마무리

Node.js의 `http.Agent`에서 `maxSockets`, `maxTotalSockets`는 단순 성능 옵션이 아닙니다.
이 값들은 외부 호출이 몰릴 때 **어디까지 버티고, 어디서부터 보호할지**를 정하는 운영 설정에 가깝습니다.

실무에서는 keep-alive만 켜고 끝내지 말고, 호스트별 상한과 전체 상한을 함께 설계하세요.
거기에 timeout budget, retry budget, 기능별 agent 분리까지 더하면 외부 의존성이 느려질 때도 전체 서비스가 같이 무너지지 않을 가능성이 훨씬 높아집니다.

관련해서 함께 보면 좋은 글은 아래입니다.

- [Node.js Connection Pool Exhaustion 가이드](/development/blog/seo/2026/04/11/nodejs-connection-pool-exhaustion-prevention-guide.html)
- [Node.js Timeout Budget, Deadline Propagation 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)
- [Node.js maxRequestsPerSocket, keep-alive connection recycling 가이드](/development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html)
- [Node.js DNS Lookup 지연 가이드](/development/blog/seo/2026/04/22/nodejs-dns-lookup-latency-caching-guide.html)

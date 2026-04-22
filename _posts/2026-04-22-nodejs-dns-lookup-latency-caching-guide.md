---
layout: post
title: "Node.js DNS Lookup 지연 가이드: 외부 API 호출이 느린데 서버는 멀쩡할 때"
date: 2026-04-22 20:00:00 +0900
lang: ko
translation_key: nodejs-dns-lookup-latency-caching-guide
permalink: /development/blog/seo/2026/04/22/nodejs-dns-lookup-latency-caching-guide.html
alternates:
  ko: /development/blog/seo/2026/04/22/nodejs-dns-lookup-latency-caching-guide.html
  x_default: /development/blog/seo/2026/04/22/nodejs-dns-lookup-latency-caching-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, dns, dnscache, lookup, latency, httpagent, undici, backend, performance, reliability]
description: "Node.js 서비스에서 외부 API 호출만 유독 느릴 때 놓치기 쉬운 DNS lookup 병목을 점검하는 방법과 DNS 캐싱 전략, 운영 체크포인트를 실무 기준으로 정리했습니다."
---

Node.js 서비스가 평소에는 멀쩡한데 외부 API 호출만 간헐적으로 느려질 때가 있습니다.
이때 애플리케이션 코드나 DB만 보고 있으면 원인을 놓치기 쉬운데, 의외로 자주 숨어 있는 병목이 `DNS lookup`입니다.

특히 upstream 응답 시간만 보고 있으면 “상대 서버가 느리다”로 오해하기 쉽습니다.
실제로는 연결 전에 일어나는 이름 해석이 지연돼 전체 응답 시간이 늘어난 경우도 많습니다.

결론부터 말하면, Node.js 외부 호출 지연은 **애플리케이션 처리 시간**, **네트워크 연결 시간**, **DNS 조회 시간**을 분리해서 봐야 합니다.
그리고 DNS가 병목이라면 무작정 재시도부터 늘리기보다, 캐시 전략과 관측 지표를 먼저 정리하는 편이 안전합니다.

## 왜 DNS lookup 병목을 먼저 의심해야 할까

### H3. 애플리케이션은 한가한데 외부 호출만 들쭉날쭉할 수 있다

CPU 사용률, 이벤트 루프 지연, DB 응답 시간이 안정적인데 특정 외부 API 호출만 간헐적으로 튀면 DNS 구간을 의심해볼 만합니다.
이름 해석은 요청 본문 처리보다 먼저 일어나기 때문에, 여기서 밀리면 애플리케이션 로그에는 그냥 “요청이 늦었다” 정도로만 보일 수 있습니다.

특히 아래 상황에서 더 자주 드러납니다.

- 짧은 주기의 외부 API 호출이 많은 서비스
- 같은 호스트로 반복 호출하지만 연결 재사용이 낮은 서비스
- 컨테이너 환경에서 DNS 리졸버를 여러 단계 거치는 경우
- 장애 시 재시도가 늘면서 lookup 부하까지 함께 커지는 경우

이런 경우는 단순 timeout 상향보다, 병목 구간 분해가 먼저입니다.
[Node.js Timeout Budget, Deadline Propagation 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)에서 다룬 것처럼 시간 예산을 구간별로 나눠 봐야 원인을 잡을 수 있습니다.

### H3. lookup 지연은 상대 서버 문제처럼 보이기 쉽다

운영에서 흔한 착시는 “외부 API가 느리다”는 결론을 너무 빨리 내리는 것입니다.
하지만 실제 총 지연 시간에는 아래가 섞여 있습니다.

1. DNS 이름 해석
2. TCP 연결 수립
3. TLS 핸드셰이크
4. 요청 전송과 응답 대기

즉 외부 호출이 800ms 걸렸다고 해서, 상대 애플리케이션이 800ms 느린 것은 아닙니다.
DNS만 250ms 튀어도 체감은 꽤 커집니다.

## Node.js에서 DNS lookup이 특히 눈에 띄는 이유

### H3. 매 요청마다 새 연결이 많으면 lookup 비용이 반복된다

외부 호출에서 keep-alive나 connection reuse가 충분하지 않으면, 호스트 이름 해석과 연결 수립 비용이 반복됩니다.
이 경우 애플리케이션 자체는 빠른데도 호출 수가 많아질수록 lookup 비용이 누적될 수 있습니다.

그래서 DNS 문제는 단독 이슈라기보다 연결 재사용 정책과 같이 보는 편이 좋습니다.
같은 맥락에서 [Node.js maxRequestsPerSocket, keep-alive connection recycling 가이드](/development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html)를 함께 보면 구조가 더 잘 보입니다.

### H3. Node.js에 애플리케이션 레벨 DNS 캐시가 자동으로 생기지는 않는다

많이 헷갈리는 부분인데, 운영체제나 네트워크 레이어에 캐시가 있다고 해서 애플리케이션 관점의 지연이 항상 충분히 줄어드는 것은 아닙니다.
Node.js에서 외부 호스트 호출을 많이 한다면, 리졸버 경로와 연결 재사용 정책에 따라 lookup 비용이 계속 체감될 수 있습니다.

즉 “DNS 캐시는 OS가 알아서 하겠지”라고 단정하면 놓치는 경우가 있습니다.
실무에서는 실제 지연 분포를 측정한 뒤, 캐시 계층을 어디에 둘지 결정하는 편이 낫습니다.

## 어떤 증상이 나오면 DNS 병목일 가능성이 높을까

### H3. 같은 upstream인데 지연 편차가 유난히 크다

같은 외부 API를 호출하는데 어떤 요청은 80ms, 어떤 요청은 700ms라면 단순 애플리케이션 처리 시간보다 이름 해석이나 연결 수립 구간 변동을 의심해볼 수 있습니다.

특히 이런 징후가 있으면 더 그렇습니다.

- 특정 호스트 호출만 유독 편차가 큼
- 재시도 후 성공하지만 첫 요청이 느림
- 새 프로세스 기동 직후 첫 호출이 느림
- 피크 시간대에만 lookup 관련 에러나 timeout이 증가함

이럴 때는 무조건 재시도 횟수부터 늘리기보다, 왜 첫 시도 자체가 느린지 먼저 보는 편이 맞습니다.
재시도는 병목을 가릴 수도 있고, 오히려 부하를 키울 수도 있습니다.

### H3. circuit breaker나 timeout이 자주 열리는데 근본 원인이 다를 수 있다

외부 API 장애처럼 보여서 circuit breaker를 강화했는데도 체감이 나아지지 않는다면, 실제 병목이 upstream 애플리케이션이 아니라 lookup 단계일 수 있습니다.
이 경우 보호 장치는 필요하지만, 원인 제거는 별도입니다.

장애 격리 관점은 [Node.js circuit breaker failure isolation 가이드](/development/blog/seo/2026/04/04/nodejs-circuit-breaker-failure-isolation-guide.html)와 연결되고, tail latency 완화는 [Node.js hedged requests tail latency reduction 가이드](/development/blog/seo/2026/04/08/nodejs-hedged-requests-tail-latency-reduction-guide.html)와 이어집니다.
다만 DNS 병목을 놔둔 채 우회 전략만 늘리면 비용이 커질 수 있습니다.

## DNS lookup 지연은 어떻게 측정해야 할까

### H3. 총 요청 시간만 보면 원인을 분리하기 어렵다

가장 먼저 해야 할 일은 총 요청 시간 하나만 기록하는 습관에서 벗어나는 것입니다.
실무에서는 아래처럼 최소한 구간별 시간을 나눠 보는 편이 좋습니다.

- DNS lookup 시간
- TCP connect 시간
- TLS handshake 시간
- upstream first byte 시간
- 전체 응답 완료 시간

이렇게 분해해두면 “느리다”가 아니라 “lookup이 튄다”처럼 말할 수 있게 됩니다.
원인이 분리돼야 대응도 정확해집니다.

### H3. 호스트별 p95, p99를 따로 보는 것이 중요하다

평균값만 보면 DNS 문제를 놓치기 쉽습니다.
lookup 지연은 항상 느린 것이 아니라 간헐적으로 튀는 경우가 많아서, p95나 p99가 더 중요합니다.

특히 아래 기준으로 보면 도움이 됩니다.

- 호스트별 lookup p95, p99
- 특정 배포 이후 lookup 악화 여부
- 재시도 증가와 lookup 악화의 상관관계
- 프로세스 재기동 직후와 안정화 후 차이

## DNS 캐싱 전략은 어떻게 잡아야 할까

### H3. TTL을 무시한 장기 캐시는 오히려 위험하다

DNS 병목이 보인다고 무조건 오래 캐시하면 안 됩니다.
호스트 IP가 바뀌는 환경에서 TTL을 무시한 캐시는 장애를 길게 끌 수 있습니다.

그래서 DNS 캐시는 보통 아래 원칙으로 접근합니다.

- TTL을 존중한다
- 실패한 주소를 과하게 오래 붙잡지 않는다
- 서비스 디스커버리 변경 주기를 이해한다
- 캐시 도입 전후 지연과 오류율을 같이 본다

즉 캐시는 lookup 비용을 줄이기 위한 수단이지, 고정 IP를 영구 보관하는 장치가 아닙니다.

### H3. 애플리케이션 캐시보다 먼저 연결 재사용을 점검한다

많은 경우 더 먼저 효과를 보는 것은 DNS 캐시 라이브러리 추가보다 연결 재사용 개선입니다.
같은 호스트를 자주 호출한다면 keep-alive와 socket reuse만으로도 lookup 반복 비용을 크게 줄일 수 있습니다.

점검 순서는 보통 아래가 현실적입니다.

1. 새 연결이 너무 많지 않은지 확인한다.
2. keep-alive와 agent 설정이 호출 패턴에 맞는지 본다.
3. 그래도 lookup이 병목이면 DNS 캐시 전략을 검토한다.
4. 캐시 TTL과 장애 전파 리스크를 함께 점검한다.

## 운영에서는 무엇을 함께 봐야 할까

### H3. timeout, retry, connection reuse를 분리해서 튜닝한다

DNS 병목이 있을 때 timeout만 키우면 느린 요청을 더 오래 붙잡게 될 수 있습니다.
반대로 timeout을 너무 공격적으로 줄이면 일시적인 lookup 변동에도 실패율이 커질 수 있습니다.

그래서 아래 항목을 분리해서 봐야 합니다.

- lookup 지연 자체를 줄일 수 있는가
- timeout budget이 현실적인가
- retry가 DNS 부하를 더 키우고 있지 않은가
- connection reuse가 충분한가

이 네 가지를 분리하지 않으면 “timeout 올림 → 재시도 증가 → 더 느려짐” 같은 악순환이 생길 수 있습니다.

### H3. 관측 지표가 없으면 DNS는 늘 마지막에 의심된다

DNS 문제는 흔하지만, 관측이 약하면 거의 항상 뒤늦게 발견됩니다.
그래서 외부 API 호출이 중요한 서비스라면 최소한 아래는 남겨두는 편이 좋습니다.

```txt
[권장 점검 항목]
- 호스트별 lookup latency p95/p99
- 새 연결 비율과 keep-alive reuse 비율
- timeout 발생 구간(lookup/connect/read)
- retry 횟수와 성공/실패 전환율
- 특정 리졸버 또는 특정 호스트 편중 여부
```

이 정도만 있어도 “상대 서버가 느린가?”와 “우리 lookup 경로가 느린가?”를 구분하기가 훨씬 쉬워집니다.

## 빠르게 판단할 때 쓰는 체크리스트

### H3. 아래 순서로 보면 헛다리 짚을 가능성이 줄어든다

- 외부 API 총 지연을 lookup, connect, TLS, read로 분해했는가?
- 같은 호스트로 새 연결이 과도하게 많이 생기지 않는가?
- keep-alive 재사용이 충분한가?
- retry가 병목을 가리는 대신 확대하고 있지 않은가?
- DNS 캐시를 넣더라도 TTL과 장애 전파 리스크를 같이 봤는가?

## 마무리

Node.js 외부 호출이 느릴 때 DNS lookup은 생각보다 자주 놓치는 병목입니다.
애플리케이션 코드가 빠르다고 해서, 연결 전 단계까지 자동으로 건강한 것은 아닙니다.

실무에서는 먼저 시간을 구간별로 나누고, 그다음 연결 재사용과 DNS 캐싱 전략을 검토하세요.
이 순서로 보면 불필요한 재시도나 과한 timeout 상향 없이도 원인을 훨씬 빨리 좁힐 수 있습니다.

관련해서 함께 보면 좋은 글은 아래입니다.

- [Node.js Timeout Budget, Deadline Propagation 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)
- [Node.js maxRequestsPerSocket, keep-alive connection recycling 가이드](/development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html)
- [Node.js circuit breaker failure isolation 가이드](/development/blog/seo/2026/04/04/nodejs-circuit-breaker-failure-isolation-guide.html)

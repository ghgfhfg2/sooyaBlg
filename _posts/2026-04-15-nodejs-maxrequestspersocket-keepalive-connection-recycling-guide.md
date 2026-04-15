---
layout: post
title: "Node.js maxRequestsPerSocket 가이드: keep-alive 연결 재사용 편차를 줄이는 법"
date: 2026-04-15 20:00:00 +0900
lang: ko
translation_key: nodejs-maxrequestspersocket-keepalive-connection-recycling-guide
permalink: /development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html
alternates:
  ko: /development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html
  x_default: /development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, maxrequestspersocket, keepalive, http, connectionpooling, backend, performance, reliability, production]
description: "Node.js 서버에서 maxRequestsPerSocket을 활용하면 일부 keep-alive 연결에 요청이 과도하게 몰리는 현상과 장수 커넥션 편차를 줄일 수 있습니다. 설정 기준, 주의점, 운영 체크리스트를 실무 중심으로 정리했습니다."
---

Node.js 서비스를 운영하다 보면 CPU나 DB는 여유가 있는데도 특정 인스턴스나 일부 소켓에만 요청이 유난히 오래 붙는 느낌을 받을 때가 있습니다.
로그를 자세히 보면 같은 keep-alive 연결이 지나치게 오래 재사용되거나, 연결별 요청 수 편차가 커져서 디버깅이 까다로워지는 경우가 있습니다.

이럴 때 무작정 timeout만 바꾸기보다 `maxRequestsPerSocket`을 검토해볼 만합니다.
이 옵션은 **하나의 소켓이 처리할 수 있는 요청 수에 상한을 두어, 연결을 주기적으로 교체하는 기준**을 만들어 줍니다.
이 글에서는 언제 유용한지, 어떤 부작용이 있는지, 운영에서 어떤 값으로 시작하면 좋은지 정리합니다.

## maxRequestsPerSocket은 무엇을 제어할까

### H3. keep-alive 연결의 수명을 시간 대신 요청 수로 제한한다

`server.maxRequestsPerSocket`은 하나의 keep-alive 소켓이 몇 개의 요청을 처리한 뒤 더 이상 재사용되지 않게 할지 결정합니다.
시간 기반 설정인 `keepAliveTimeout`이 "얼마나 오래 idle 상태로 둘 것인가"를 다룬다면, `maxRequestsPerSocket`은 "한 연결을 얼마나 오래 현역으로 쓸 것인가"를 다룹니다.

예시는 아래처럼 둘 수 있습니다.

```js
const http = require('http');
const app = require('./app');

const server = http.createServer(app);

server.keepAliveTimeout = 65000;
server.headersTimeout = 66000;
server.maxRequestsPerSocket = 1000;

server.listen(3000);
```

핵심은 성능 최적화용 숫자를 찾는 것보다, **장수 연결이 만드는 편차를 제어 가능한 범위로 줄이는 것**입니다.
특히 프록시나 로드밸런서가 연결을 오래 붙잡는 환경에서는 이 옵션이 생각보다 유용합니다.

### H3. 오래 살아남은 소켓에 요청이 몰리는 현상을 완화할 수 있다

HTTP keep-alive는 원래 효율을 높이는 기능이지만, 모든 연결이 균등하게 재사용되지는 않습니다.
트래픽 패턴이나 프록시 동작에 따라 일부 연결만 유난히 오래 살아남을 수 있습니다.

그 결과 아래 같은 문제가 생길 수 있습니다.

- 특정 인스턴스의 일부 연결에 요청이 치우쳐 보임
- 커넥션 단위 로그를 보면 편차가 커서 분석이 어려움
- 배포 직후에는 멀쩡하지만 시간이 지나며 분산이 흐트러짐
- 장수 연결에만 이상한 에러 패턴이 반복됨

이때 요청 수 기준으로 연결을 순환시키면, 연결 재사용 편차를 완전히 없앨 수는 없어도 더 예측 가능한 상태로 만들 수 있습니다.

## 어떤 상황에서 특히 효과가 좋을까

### H3. 프록시 뒤에서 keep-alive 연결이 길게 유지되는 환경

ALB, Nginx, Ingress 뒤에 있는 Node.js 서비스는 애플리케이션과 직접 붙는 클라이언트보다 **프록시가 관리하는 재사용 연결**의 영향을 더 크게 받습니다.
이 경우 timeout 정렬도 중요하지만, 연결 하나가 너무 오래 살아남지 않게 하는 기준도 같이 있으면 운영이 편해집니다.

timeout 자체를 맞추는 기준은 [Node.js keepAliveTimeout, headersTimeout mismatch 가이드](/development/blog/seo/2026/04/14/nodejs-keepalive-timeout-headers-timeout-mismatch-guide.html)에서 정리한 내용과 함께 보는 편이 좋습니다.
`maxRequestsPerSocket`은 timeout을 대체하는 옵션이 아니라, **재사용 수명 관리용 보조 장치**에 가깝습니다.

### H3. 인스턴스 교체나 롤링 배포 때 연결 잔존 편차를 줄이고 싶을 때

롤링 배포 중에는 readiness를 내려도 기존 keep-alive 연결이 한동안 남아 있을 수 있습니다.
이 자체는 정상 동작이지만, 연결이 지나치게 오래 유지되면 새 인스턴스로의 트래픽 재분배가 예상보다 느려질 수 있습니다.

이럴 때 요청 수 상한이 있으면 오래된 연결이 영구히 남아 있는 느낌을 줄일 수 있습니다.
배포 시 트래픽 차단 순서와 드레이닝 자체는 [Node.js Graceful Shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)와 [Node.js Readiness, Liveness, Startup Probe 가이드](/development/blog/seo/2026/04/14/nodejs-readiness-liveness-startup-probe-guide.html)를 함께 참고하는 것이 좋습니다.

## 설정값은 어떻게 시작하는 게 좋을까

### H3. 처음부터 너무 작은 값은 피하는 편이 안전하다

`maxRequestsPerSocket`을 너무 작게 잡으면 연결이 지나치게 자주 교체됩니다.
그러면 keep-alive의 장점이 줄어들고, TLS 핸드셰이크나 연결 설정 비용이 다시 커질 수 있습니다.

처음부터 10, 20처럼 과하게 작은 값으로 가기보다 아래처럼 접근하는 편이 현실적입니다.

- 내부 프록시 환경: 수백 ~ 수천 단위에서 시작
- 요청이 매우 짧고 빈번한 API: 너무 작은 값은 피함
- 장수 연결 편차가 실제 문제일 때만 도입
- 변경 전후에 502, reset, handshake 비용, 지연 분포를 함께 확인

중요한 것은 절대 정답 숫자가 아니라, **현재 장애 패턴을 줄이는 방향인지**입니다.
그냥 신기한 옵션이라서 넣는 것은 별로 추천하지 않습니다.

### H3. keepAliveTimeout, headersTimeout과 같이 본다

이 옵션은 독립적으로 보면 오해하기 쉽습니다.
`maxRequestsPerSocket`만 추가한다고 timeout mismatch가 해결되지는 않습니다.
반대로 timeout만 정렬해도 장수 연결 편차가 남을 수 있습니다.

실무에서는 보통 아래 순서로 봅니다.

1. 프록시와 애플리케이션의 timeout 관계를 먼저 정리한다.
2. 배포 드레이닝과 readiness 전환이 정상인지 확인한다.
3. 그래도 장수 연결 편차가 크면 `maxRequestsPerSocket`을 도입한다.
4. 변경 후 연결 생성률과 에러 패턴이 악화되지 않았는지 본다.

즉 이 옵션은 만능 해법이 아니라, **연결 생애주기 설계의 마지막 미세조정 축**에 더 가깝습니다.

## 도입할 때 주의해야 할 부작용

### H3. 연결 재생성이 늘면서 오히려 비용이 증가할 수 있다

연결을 더 자주 닫고 다시 만들면 당연히 비용도 늘어납니다.
특히 TLS 종료 지점, 프록시 홉 수, 네트워크 지연이 큰 환경에서는 그 차이가 더 크게 보일 수 있습니다.

그래서 아래 지표를 같이 봐야 합니다.

- 신규 TCP 연결 수
- TLS handshake 횟수와 시간
- upstream reset, 502, ECONNRESET 변화
- p95, p99 지연 시간 변화
- 인스턴스별 요청 분포 편차

한쪽 편차를 줄이려다가 다른 쪽 비용을 키우면 의미가 없습니다.
운영에서는 늘 "덜 나쁜 쪽"을 택해야 합니다.

### H3. HTTP/2, gRPC 환경이라면 별도 검토가 필요하다

`maxRequestsPerSocket`은 주로 Node.js의 HTTP/1.1 keep-alive 연결 재사용 관점에서 이해하는 것이 자연스럽습니다.
HTTP/2나 gRPC처럼 하나의 연결에서 다중 스트림을 처리하는 환경은 연결 생명주기와 병목 지점이 다릅니다.

그래서 서비스 경로가 아래 중 어디인지 먼저 구분해야 합니다.

- 클라이언트 ↔ ALB/Nginx: HTTP/2
- 프록시 ↔ Node.js 앱: HTTP/1.1
- 전체 경로가 gRPC

실제 병목이 HTTP/1.1 upstream keep-alive에 있는지 확인하지 않으면, 엉뚱한 레이어만 만지게 됩니다.

## 운영에서 이렇게 점검하면 덜 헤맨다

### H3. 연결 단위 로그나 메트릭을 남겨 변화량을 본다

이 옵션은 체감만으로 평가하면 위험합니다.
도입 전후를 비교할 수 있어야 합니다.

가능하면 아래 정보를 남기는 편이 좋습니다.

- 인스턴스별 활성 연결 수
- 소켓당 처리 요청 수 분포
- 연결 생성/종료 비율
- 배포 전후의 잔존 연결 시간
- 5xx와 reset 계열 오류 발생 시점

요청 컨텍스트 추적이 필요하다면 [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)처럼 요청 ID와 배포 버전을 함께 남겨두면 분석이 훨씬 쉬워집니다.

### H3. 장애를 줄이는 목적이 맞는지 끝까지 확인한다

운영 설정은 종종 "정교해 보이는 것"과 "실제로 효과 있는 것"이 다릅니다.
`maxRequestsPerSocket`도 마찬가지입니다.

아래 질문에 명확히 답할 수 있어야 합니다.

- 지금 문제는 timeout mismatch인가, 연결 편차인가?
- 일부 장수 연결이 실제로 운영 불안정에 기여하는가?
- 값을 낮춘 뒤 연결 생성 비용이 감당 가능한가?
- 롤백 기준은 무엇인가?

이 질문 없이 설정만 추가하면, 나중에 장애 회고에서 "왜 넣었는지 기억이 안 나는 옵션"이 되기 쉽습니다.

## 실무 적용 체크리스트

### H3. 적용 전 체크

- 프록시 idle timeout과 Node.js `keepAliveTimeout`, `headersTimeout` 관계를 문서화했는가
- 현재 502, `ECONNRESET`, socket hang up 패턴을 수집했는가
- 연결 재사용 편차를 확인할 메트릭이 있는가
- HTTP/1.1 upstream 구간이 실제 병목인지 확인했는가

### H3. 적용 후 체크

- 연결 생성률이 비정상적으로 증가하지 않았는가
- p95, p99 지연 시간이 악화되지 않았는가
- 인스턴스별 요청 편차가 줄었는가
- 배포 후 잔존 연결이 예상보다 길게 남지 않는가
- 롤백 기준을 충족하면 빠르게 되돌릴 수 있는가

## 마무리

`maxRequestsPerSocket`은 자주 만지는 옵션은 아니지만, keep-alive 연결이 너무 오래 살아남아 운영 편차를 키우는 환경에서는 꽤 실용적입니다.
중요한 것은 이 옵션 하나로 모든 네트워크 문제를 해결하려 하지 않는 것입니다.

먼저 timeout 정렬, readiness 전환, graceful shutdown 같은 기본기를 맞추고, 그다음에 연결 재사용 수명까지 다듬는 순서가 좋습니다.
그렇게 접근하면 설정이 늘어나는 대신, 시스템은 오히려 더 단순하게 설명됩니다.

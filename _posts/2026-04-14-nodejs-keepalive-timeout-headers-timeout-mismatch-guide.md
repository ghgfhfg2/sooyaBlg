---
layout: post
title: "Node.js keepAliveTimeout, headersTimeout mismatch 가이드: ALB·Nginx 뒤에서 간헐적 502를 줄이는 법"
date: 2026-04-14 20:00:00 +0900
lang: ko
translation_key: nodejs-keepalive-timeout-headers-timeout-mismatch-guide
permalink: /development/blog/seo/2026/04/14/nodejs-keepalive-timeout-headers-timeout-mismatch-guide.html
alternates:
  ko: /development/blog/seo/2026/04/14/nodejs-keepalive-timeout-headers-timeout-mismatch-guide.html
  x_default: /development/blog/seo/2026/04/14/nodejs-keepalive-timeout-headers-timeout-mismatch-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, keepalivetimeout, headerstimeout, http, nginx, alb, proxy, backend, reliability]
description: "Node.js 서버의 keepAliveTimeout과 headersTimeout 설정이 프록시보다 짧거나 어긋나면 간헐적 502, socket hang up, ECONNRESET이 늘어날 수 있습니다. 실무에서 점검하는 기준과 설정 예시를 정리했습니다."
---

Node.js 서비스를 ALB, Nginx, Ingress 뒤에 두고 운영하다 보면 평소에는 멀쩡한데도 간헐적으로 502, socket hang up, ECONNRESET 같은 오류가 튀는 경우가 있습니다.
애플리케이션 코드나 데이터베이스만 의심하다가 시간을 많이 쓰는데, 실제로는 **HTTP keep-alive timeout 정렬이 어긋난 문제**인 경우가 꽤 많습니다.

특히 Node.js 서버의 `keepAliveTimeout`, `headersTimeout`, 프록시의 idle timeout이 서로 다르게 잡혀 있으면 재사용된 연결에서 애매한 끊김이 생길 수 있습니다.
이 글에서는 왜 이런 문제가 생기는지, 어떤 순서로 점검해야 하는지, 그리고 실무에서 어떻게 안전한 기본값을 잡는지 정리합니다.

## keepAliveTimeout mismatch가 왜 502와 ECONNRESET을 만들까

### H3. 재사용된 연결이 이미 반쯤 죽어 있으면 프록시가 먼저 맞는다

HTTP keep-alive는 매 요청마다 새 TCP 연결을 만들지 않고 기존 연결을 재사용해서 지연과 비용을 줄입니다.
문제는 프록시와 Node.js가 **연결을 언제 닫을지 서로 다르게 이해할 때** 발생합니다.

예를 들어 이런 순서가 생길 수 있습니다.

1. ALB나 Nginx가 백엔드 연결을 재사용하려고 유지합니다.
2. Node.js는 자신의 `keepAliveTimeout` 기준으로 먼저 소켓을 닫습니다.
3. 프록시는 그 소켓이 아직 살아 있다고 믿고 다음 요청을 보냅니다.
4. 결과적으로 upstream reset, 502, `ECONNRESET`, `socket hang up`이 터집니다.

이 문제는 항상 재현되지 않아서 더 까다롭습니다.
트래픽 패턴, 커넥션 재사용 시점, 프록시 idle timeout에 따라 드문 간헐 장애처럼 보이기 쉽습니다.

### H3. timeout 문제는 애플리케이션 버그처럼 보이기 쉽다

로그만 보면 아래처럼 찍히는 경우가 많습니다.

- upstream prematurely closed connection
- read ECONNRESET
- socket hang up
- 502 Bad Gateway

이런 메시지는 겉으로는 네트워크 불안정, 프록시 버그, 일시 과부하처럼 보입니다.
하지만 실제로는 **정상적인 idle connection 정리 타이밍이 어긋난 것**일 수 있습니다.

비슷하게 타임아웃 경계를 설계하는 관점은 [Node.js Deadline Exceeded 에러 처리 가이드](/development/blog/seo/2026/04/12/nodejs-deadline-exceeded-error-handling-guide.html)와도 연결됩니다.
중요한 것은 각 계층의 timeout이 독립적으로 보이더라도, 운영에서는 결국 하나의 요청 경로로 이어진다는 점입니다.

## Node.js에서 먼저 봐야 할 timeout 설정

### H3. keepAliveTimeout은 idle keep-alive 소켓을 얼마나 유지할지 결정한다

Node.js HTTP 서버에서 `keepAliveTimeout`은 응답을 마친 뒤 다음 요청을 기다리며 연결을 유지하는 시간을 뜻합니다.
이 값이 너무 짧으면 프록시가 재사용하려는 연결을 Node.js가 먼저 닫아버릴 수 있습니다.

기본 예시는 아래처럼 둘 수 있습니다.

```js
const http = require('http');
const app = require('./app');

const server = http.createServer(app);

server.keepAliveTimeout = 65000;
server.headersTimeout = 66000;
server.requestTimeout = 30000;

server.listen(3000);
```

핵심은 절대값보다 **프록시보다 누가 먼저 연결을 닫을지**를 명확히 정하는 것입니다.
실무에서는 보통 프록시 idle timeout과 서버 keep-alive 관계를 의도적으로 맞춰 둡니다.

### H3. headersTimeout은 keepAliveTimeout보다 약간 크게 둔다

`headersTimeout`은 클라이언트가 요청 헤더를 보내는 데 허용하는 최대 시간입니다.
이 값이 `keepAliveTimeout`보다 같거나 더 작으면, keep-alive 연결 처리 중 애매한 타이밍 문제가 생길 수 있습니다.

그래서 일반적으로는 아래 원칙을 많이 씁니다.

- `headersTimeout`은 `keepAliveTimeout`보다 조금 크게 설정
- 차이는 보통 1초 이상 여유를 둠
- `requestTimeout`은 실제 요청 처리 SLA에 맞게 별도로 결정

즉 `keepAliveTimeout = 65s`, `headersTimeout = 66s`처럼 **headersTimeout이 약간 더 큰 구조**가 해석하기 쉽고 운영도 안정적입니다.

## ALB, Nginx, Ingress와 같이 볼 때 점검 순서

### H3. 프록시 idle timeout과 Node.js keepAliveTimeout의 선후관계를 먼저 정한다

가장 먼저 확인할 것은 프록시의 idle timeout입니다.
대표적으로 아래 값들을 봅니다.

- AWS ALB idle timeout
- Nginx `keepalive_timeout`
- Ingress controller 기본 upstream keep-alive 설정
- 서비스 메시나 API gateway의 backend connection timeout

여기서 중요한 것은 누가 먼저 연결을 정리할지 **운영 기준을 하나로 정하는 것**입니다.
보통은 프록시와 애플리케이션이 서로 엇갈리게 닫지 않도록 다음 중 하나를 택합니다.

1. 프록시가 더 길게 유지하고, Node.js도 그보다 충분히 길게 유지한다.
2. 프록시가 먼저 정리하게 하고, Node.js는 그보다 짧거나 의미 있게 다르게 두지 않는다.

실무에서는 팀마다 기준이 다를 수 있지만, 제일 나쁜 것은 **기준 없이 기본값이 섞여 있는 상태**입니다.

### H3. 배포 직후보다 트래픽이 애매하게 줄었을 때 더 잘 터질 수 있다

이 문제는 피크 시간보다 오히려 요청 간 간격이 애매하게 벌어질 때 더 잘 보이기도 합니다.
연결 재사용은 일어나지만 idle 시간이 timeout 경계 근처에 걸리기 쉽기 때문입니다.

그래서 아래 상황에서 특히 의심해볼 만합니다.

- 배포 후 일부 pod에서만 간헐적 502가 남
- 저녁이나 새벽처럼 QPS가 애매하게 줄어든 시간대에 발생
- 특정 API만 아니라 여러 엔드포인트에서 드물게 발생
- 앱 CPU, 메모리, DB는 정상인데 프록시 에러 비율만 올라감

이럴 때 readiness/liveness 문제와 혼동하기도 쉬운데, 헬스체크 경계는 [Node.js Readiness, Liveness, Startup Probe 가이드](/development/blog/seo/2026/04/14/nodejs-readiness-liveness-startup-probe-guide.html)에서 따로 정리한 내용과 구분해서 봐야 합니다.
프로브는 pod 생명주기 판단이고, keep-alive timeout은 **재사용 연결의 생명주기 판단**에 가깝습니다.

## 설정 예시와 안전한 기본 접근

### H3. 먼저 요청 시간 예산과 idle 연결 예산을 분리해서 생각한다

timeout을 하나의 숫자로 다루면 설정이 꼬입니다.
실무에서는 아래처럼 층위를 나누는 편이 좋습니다.

- `requestTimeout`: 개별 요청 처리 제한 시간
- `keepAliveTimeout`: idle 상태의 연결 유지 시간
- `headersTimeout`: 헤더 수신 허용 시간
- 프록시 idle timeout: upstream/back-end 연결 유지 시간

이 값을 섞어버리면 문제 원인이 불명확해집니다.
예를 들어 느린 요청이 많다고 해서 `keepAliveTimeout`만 늘리면 해결되지 않습니다.
반대로 idle 소켓 재사용 문제인데 `requestTimeout`만 만져도 효과가 없습니다.

### H3. 장애를 줄이는 시작점 예시

아래는 Node.js 뒤에 프록시가 있는 일반적인 API 서버에서 시작점으로 검토할 만한 예시입니다.

```js
server.requestTimeout = 30000;
server.keepAliveTimeout = 65000;
server.headersTimeout = 66000;
```

그리고 프록시 측에서도 아래를 같이 확인합니다.

- ALB idle timeout이 너무 짧지 않은지
- Nginx upstream keep-alive 관련 설정이 기본값에 묶여 있지 않은지
- Ingress annotation이나 controller 기본 timeout이 앱 의도와 맞는지
- 여러 환경(dev, stage, prod)에서 값이 다르게 드리프트하지 않았는지

이런 식의 일관성 점검은 장애 전파를 줄이는 관점에서 [Node.js Graceful Shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)와도 잘 맞습니다.
연결 종료는 요청 처리보다 덜 눈에 띄지만, 잘못 설계되면 사용자 체감 에러로 바로 이어집니다.

## 실무에서 자주 하는 실수

### H3. Node.js 값만 바꾸고 프록시는 안 본다

애플리케이션 개발자가 Node.js 서버 옵션만 수정하고 끝내는 경우가 많습니다.
하지만 실제 장애는 프록시, ingress, 로드밸런서와의 조합에서 발생합니다.

특히 아래 실수는 자주 나옵니다.

- Node.js 버전 업 후 timeout 기본값 변화 확인 누락
- ALB, Nginx, CDN 계층이 동시에 있는 구조를 하나만 보고 판단
- pod 내부 curl 테스트만 하고 실제 프록시 경로는 재현하지 않음
- stage와 prod의 load balancer 설정 차이를 놓침

### H3. 모든 502를 서버 과부하로 단정한다

502가 난다고 해서 항상 CPU, 메모리, DB 병목은 아닙니다.
물론 과부하도 흔한 원인이지만, idle socket 재사용 문제는 리소스 메트릭이 깨끗해도 발생할 수 있습니다.

그래서 아래를 함께 보면 좋습니다.

- 프록시 에러 로그의 reset 패턴
- upstream connection 재사용 여부
- 특정 시간대의 idle gap 분포
- Node.js 서버 timeout 설정과 프록시 timeout 설정 비교표

즉 502 분석은 서버 자원만 보는 문제가 아니라, **연결 수명주기 설계**를 보는 문제이기도 합니다.

## 점검 체크리스트

### H3. 배포 전에 아래 항목을 한 번에 확인한다

- Node.js `keepAliveTimeout` 값을 명시적으로 설정했는가
- `headersTimeout`이 `keepAliveTimeout`보다 약간 크게 설정돼 있는가
- `requestTimeout`을 별도 예산으로 관리하고 있는가
- ALB, Nginx, Ingress의 idle timeout을 문서화했는가
- 환경별 timeout 값 차이가 없는가
- 502, ECONNRESET, socket hang up 로그를 연결 재사용 관점에서 확인했는가

이 체크리스트만 잡아도 간헐 장애 디버깅 시간을 많이 줄일 수 있습니다.

## 마무리

Node.js keepAliveTimeout, headersTimeout mismatch 문제는 작은 설정처럼 보이지만 운영에서는 꽤 비싼 장애로 번집니다.
특히 프록시 뒤에 있는 서비스라면 502, `ECONNRESET`, `socket hang up`이 보일 때 애플리케이션 코드만 보지 말고 **연결이 언제 닫히는지**부터 확인하는 편이 빠릅니다.

정리하면 핵심은 세 가지입니다.

- keep-alive 연결의 종료 시점을 계층별로 맞춘다.
- `headersTimeout`은 `keepAliveTimeout`보다 약간 크게 둔다.
- 프록시와 Node.js를 따로 보지 말고 하나의 요청 경로로 점검한다.

간헐적 502가 계속 남는다면, 이번에는 에러 로그보다 먼저 timeout 표를 나란히 적어보는 걸 권합니다.
생각보다 거기서 바로 답이 나오는 경우가 많습니다.

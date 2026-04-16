---
layout: post
title: "Node.js requestTimeout, timeout, headersTimeout 차이 가이드: 서버 타임아웃을 헷갈리지 않는 법"
date: 2026-04-16 20:00:00 +0900
lang: ko
translation_key: nodejs-requesttimeout-timeout-headerstimeout-difference-guide
permalink: /development/blog/seo/2026/04/16/nodejs-requesttimeout-timeout-headerstimeout-difference-guide.html
alternates:
  ko: /development/blog/seo/2026/04/16/nodejs-requesttimeout-timeout-headerstimeout-difference-guide.html
  x_default: /development/blog/seo/2026/04/16/nodejs-requesttimeout-timeout-headerstimeout-difference-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, requesttimeout, timeout, headerstimeout, keepalivetimeout, httpserver, backend, reliability, production]
description: "Node.js HTTP 서버의 requestTimeout, timeout, headersTimeout이 각각 무엇을 제한하는지, 어떤 값으로 맞춰야 운영 사고를 줄일 수 있는지 실무 기준으로 정리했습니다."
---

Node.js HTTP 서버를 운영하다 보면 `requestTimeout`, `timeout`, `headersTimeout` 이름이 너무 비슷해서 설정을 잘못 넣기 쉽습니다.
문제는 이 셋을 헷갈리면 느린 클라이언트 방어가 어긋나거나, 반대로 정상 요청까지 너무 빨리 끊어버릴 수 있다는 점입니다.

결론부터 말하면, `requestTimeout`은 **요청 전체 수신 시간의 상한**, `headersTimeout`은 **헤더를 얼마나 오래 받을지에 대한 상한**, `timeout`은 **소켓 비활성 상태 감시**에 가깝습니다.
실무에서는 셋의 역할을 분리해서 생각해야 운영이 안정적입니다.

## 왜 requestTimeout, timeout, headersTimeout을 따로 이해해야 할까

### H3. 이름은 비슷하지만 막는 대상이 다르다

이 세 설정은 모두 "느린 연결을 언제 정리할 것인가"와 관련이 있지만, 관찰하는 구간이 다릅니다.

- `headersTimeout`: 요청 헤더를 다 받을 때까지의 시간
- `requestTimeout`: 요청 전체를 다 받을 때까지의 시간
- `timeout`: 소켓이 비활성 상태로 머무는 시간

그래서 하나만 줄인다고 전체 문제가 해결되지는 않습니다.
예를 들어 헤더는 빨리 오지만 바디 업로드가 지나치게 느린 요청은 `headersTimeout`보다 `requestTimeout`이 더 직접적으로 막습니다.
반대로 헤더 자체를 질질 끄는 slowloris류 문제는 `headersTimeout`이 더 중요합니다.

### H3. 타임아웃은 성능 튜닝보다 방어선 설계에 가깝다

많은 팀이 타임아웃을 응답 속도 튜닝 관점에서만 보지만, 서버 타임아웃은 사실상 **리소스 보호 정책**입니다.
느린 연결이 워커와 소켓을 오래 점유하면 결국 정상 사용자 요청까지 영향을 받습니다.

이 점은 과부하 상황에서 요청을 어떻게 제한할지 다룬 [Node.js Load Shedding 가이드](/development/blog/seo/2026/04/07/nodejs-load-shedding-overload-protection-guide.html)와도 연결됩니다.
타임아웃은 단순 편의 기능이 아니라, 서버가 나쁜 트래픽 패턴에 끌려가지 않게 하는 기본 방어막입니다.

## requestTimeout은 무엇을 제한할까

### H3. 요청 바디까지 포함한 전체 수신 시간을 본다

`server.requestTimeout`은 서버가 **완전한 요청을 수신하는 데 허용할 최대 시간**에 가깝습니다.
GET처럼 바디가 거의 없는 요청에서는 체감이 적지만, 파일 업로드나 큰 JSON 바디에서는 의미가 커집니다.

즉 이런 상황에서 중요합니다.

- 대용량 업로드 API가 있는 경우
- 모바일 네트워크처럼 업로드가 불안정한 클라이언트가 많은 경우
- 프록시 뒤에서 요청 바디 전달이 느려질 수 있는 경우
- 일부 비정상 클라이언트가 바디를 아주 천천히 보내는 경우

핵심은 `requestTimeout`이 너무 짧으면 정상 업로드까지 실패하고, 너무 길면 느린 요청이 소켓을 오래 점유한다는 점입니다.
그래서 업로드 API와 일반 API를 같은 기준으로 보지 않는 편이 좋습니다.

### H3. 업로드 경로가 있으면 엔드포인트별 전략이 필요하다

모든 요청에 같은 수치를 기계적으로 넣으면 부작용이 생깁니다.
예를 들어 일반 JSON API는 짧은 `requestTimeout`이 적절하지만, 파일 업로드는 더 긴 시간이 필요할 수 있습니다.

이럴 때는 아래를 먼저 정리하세요.

- 정말 큰 업로드를 애플리케이션 서버가 직접 받을지
- CDN, 오브젝트 스토리지 presigned URL로 우회할지
- 긴 업로드 요청을 별도 경로로 분리할지
- 프록시와 앱의 제한값을 어디에 둘지

서버 전체에 무작정 큰 `requestTimeout`을 주는 것보다, 긴 업로드 자체를 분리하는 편이 운영이 깔끔합니다.

## headersTimeout은 언제 중요할까

### H3. 헤더를 끝까지 안 보내는 느린 연결 방어에 가깝다

`server.headersTimeout`은 요청 헤더를 다 받을 때까지 기다리는 시간을 제한합니다.
클라이언트가 연결은 맺었지만 헤더를 아주 천천히 흘려보내는 경우, 이 값이 너무 크면 소켓이 오래 붙잡힐 수 있습니다.

특히 아래 같은 환경에서 의미가 큽니다.

- 공개 인터넷에서 직접 트래픽을 받는 경우
- 봇이나 비정상 스캐너가 자주 들어오는 경우
- 프록시가 있더라도 앱 서버까지 느린 연결이 전달될 수 있는 경우

이 값은 [Node.js keepAliveTimeout, headersTimeout mismatch 가이드](/development/blog/seo/2026/04/14/nodejs-keepalive-timeout-headers-timeout-mismatch-guide.html)에서 다룬 것처럼 다른 timeout들과 관계를 같이 봐야 합니다.
단독 숫자보다 **주변 timeout과의 정렬**이 더 중요합니다.

### H3. 너무 공격적으로 줄이면 정상 프록시 환경에서도 문제가 생길 수 있다

헤더는 보통 바디보다 작아서 빨리 들어오지만, 프록시 체인이나 TLS 종료 환경에 따라 체감이 달라질 수 있습니다.
그래서 `headersTimeout`을 지나치게 짧게 두면 드물게 정상 요청도 잘릴 수 있습니다.

실무에서는 아래 질문을 같이 보세요.

- 앱 서버가 직접 인터넷을 받는가, 아니면 ALB/Nginx 뒤에 있는가
- 프록시 레이어에서 이미 느린 헤더를 어느 정도 걸러주는가
- 헤더가 큰 요청이 들어오는 API가 있는가
- 장애 시 어떤 로그와 메트릭으로 잘린 원인을 확인할 수 있는가

## timeout은 왜 더 헷갈릴까

### H3. timeout은 요청 전체 제한이라기보다 소켓 idle 감시에 가깝다

`server.timeout`은 이름만 보면 모든 요청의 최대 처리 시간처럼 느껴지지만, 실제로는 **소켓 비활성 시간** 개념으로 이해하는 편이 안전합니다.
즉 요청이나 응답 흐름 중 일정 시간 동안 아무 활동이 없을 때 연결을 어떻게 볼지와 관련이 있습니다.

그래서 `requestTimeout`과 `timeout`을 같은 것으로 보면 안 됩니다.

- `requestTimeout`: 요청을 다 받는 데 너무 오래 걸리는가
- `timeout`: 소켓이 너무 오래 놀고 있는가

이 차이를 놓치면 API 응답 SLA를 제어하려다 엉뚱하게 연결 idle 정책만 바꾸는 일이 생깁니다.
응답 처리 상한은 보통 애플리케이션 레벨의 deadline, upstream timeout, DB timeout과 같이 설계해야 합니다.
이 부분은 [Timeout Budget, Deadline Propagation 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와 함께 보면 더 선명합니다.

### H3. 애플리케이션 실행 시간 제어는 다른 계층과 나눠서 봐야 한다

서버 소켓 타임아웃만으로 "이 API는 2초 안에 끝나야 한다"를 구현하려 하면 설계가 꼬입니다.
보통은 아래처럼 분리하는 편이 낫습니다.

- 클라이언트 deadline
- API 게이트웨이 또는 프록시 timeout
- Node.js 서버 소켓 관련 timeout
- DB statement timeout
- 외부 API 호출 timeout

각 계층이 같은 문제를 보지 않기 때문입니다.
하나의 숫자로 전부 해결하려 하면 오히려 원인 파악이 더 어려워집니다.

## 실무에서는 어떤 기준으로 값을 잡아야 할까

### H3. 일반 API와 업로드 API를 분리해서 생각한다

가장 단순하면서 효과적인 기준은 "일반 API"와 "긴 업로드/스트리밍"을 분리하는 것입니다.

일반 API 중심 서비스라면 아래 방향이 무난합니다.

1. `headersTimeout`은 느린 헤더 방어가 가능하도록 충분히 보수적으로 둔다.
2. `requestTimeout`은 일반 요청 바디 수신 시간을 기준으로 둔다.
3. `timeout`은 비활성 소켓이 너무 오래 남지 않게 관리한다.
4. keep-alive 관련 값과 함께 정렬한다.

반대로 업로드가 핵심인 서비스라면, 긴 요청을 허용할 별도 경로와 프록시 정책을 먼저 설계한 뒤 서버 값을 맞추는 편이 낫습니다.

### H3. 프록시, 로드밸런서, 앱 서버 값을 따로 문서화한다

운영 사고는 대개 한 값이 틀려서보다, 여러 레이어 값이 서로 어긋나서 납니다.
아래 표처럼 내부 문서에 정리해 두면 좋습니다.

```txt
[권장 점검 항목]
- CDN/ALB idle timeout
- Nginx proxy_read_timeout / client_body_timeout
- Node.js headersTimeout
- Node.js requestTimeout
- Node.js timeout
- keepAliveTimeout
- 업로드 전용 경로의 제한값
```

이렇게 해두면 배포 후 "왜 여기서만 408이 늘었지?" 같은 질문에 더 빨리 답할 수 있습니다.

## 운영에서 같이 봐야 할 증상은 무엇일까

### H3. 408 증가, ECONNRESET, 긴 업로드 실패를 함께 본다

서버 타임아웃 변경은 숫자 하나 바꾸는 작업처럼 보여도, 실제로는 트래픽 분포를 바꿉니다.
그래서 아래 증상을 함께 봐야 합니다.

- 408 응답 비율 변화
- `ECONNRESET`, socket hang up 증가 여부
- 파일 업로드 실패율 변화
- 특정 프록시 구간 이후만 실패하는지 여부
- 워커 점유 시간과 동시 연결 수 변화

특히 업로드 실패율은 바로 티가 안 날 수 있어서, 에러 로그뿐 아니라 비즈니스 지표도 같이 보는 편이 좋습니다.

### H3. 재현 테스트를 만들어 두면 설정 변경이 쉬워진다

타임아웃은 눈으로만 검토하면 감이 안 옵니다.
느린 헤더, 느린 바디, idle socket 상황을 재현할 스크립트를 간단히라도 두면 운영이 쉬워집니다.

예를 들어 테스트 목적이라면 이런 항목을 준비할 수 있습니다.

- 헤더를 천천히 보내는 클라이언트
- 바디를 일정 간격으로 잘게 보내는 업로드 클라이언트
- 연결만 유지하고 유휴 상태를 만드는 케이스
- 프록시 유무에 따른 차이 비교

이런 재현성은 기술 글뿐 아니라 실제 변경 안정성에도 직결됩니다.

## 빠르게 결정해야 할 때 쓰는 체크리스트

### H3. 이렇게 정리하면 덜 헷갈린다

- 느린 헤더를 막고 싶은가? `headersTimeout`
- 요청 바디까지 포함한 전체 수신 시간을 제한하고 싶은가? `requestTimeout`
- 비활성 소켓을 오래 두고 싶지 않은가? `timeout`
- keep-alive 잔류 연결까지 같이 보고 있는가? `keepAliveTimeout`
- 응답 처리 상한은 앱 deadline과 upstream timeout으로 별도 설계했는가?

## 마무리

`requestTimeout`, `timeout`, `headersTimeout`은 비슷해 보여도 같은 문제가 아닙니다.
이 셋을 한 덩어리로 이해하면 숫자는 맞췄는데 운영은 더 불안해질 수 있습니다.

실무에서는 "어떤 구간을 제한하는 값인가"부터 분리해서 보세요.
그다음 프록시, keep-alive, 업로드 경로, 애플리케이션 deadline까지 같이 정렬하면 타임아웃 설정이 훨씬 덜 헷갈립니다.

관련해서 함께 보면 좋은 글은 아래입니다.

- [Node.js keepAliveTimeout, headersTimeout mismatch 가이드](/development/blog/seo/2026/04/14/nodejs-keepalive-timeout-headers-timeout-mismatch-guide.html)
- [Node.js Timeout Budget, Deadline Propagation 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)
- [Node.js Load Shedding 가이드](/development/blog/seo/2026/04/07/nodejs-load-shedding-overload-protection-guide.html)

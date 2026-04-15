---
layout: post
title: "Node.js closeIdleConnections, closeAllConnections 가이드: 배포 종료 시 keep-alive 연결 정리하는 법"
date: 2026-04-16 08:00:00 +0900
lang: ko
translation_key: nodejs-closeidleconnections-closeallconnections-graceful-drain-guide
permalink: /development/blog/seo/2026/04/16/nodejs-closeidleconnections-closeallconnections-graceful-drain-guide.html
alternates:
  ko: /development/blog/seo/2026/04/16/nodejs-closeidleconnections-closeallconnections-graceful-drain-guide.html
  x_default: /development/blog/seo/2026/04/16/nodejs-closeidleconnections-closeallconnections-graceful-drain-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, closeidleconnections, closeallconnections, gracefulshutdown, keepalive, backend, reliability, production]
description: "Node.js 서버 종료나 롤링 배포 시 closeIdleConnections와 closeAllConnections를 언제 어떻게 써야 하는지, keep-alive 잔류 연결을 안전하게 드레이닝하는 실무 기준으로 정리했습니다."
---

Node.js 서비스를 롤링 배포할 때 readiness를 내렸는데도 종료가 생각보다 오래 걸리거나, 드물게는 오래 붙어 있던 keep-alive 연결 때문에 인스턴스가 깔끔하게 빠지지 않는 경우가 있습니다.
이럴 때 `server.close()`만 믿고 있으면 충분한지, `closeIdleConnections()`나 `closeAllConnections()`까지 써야 하는지 헷갈리기 쉽습니다.

결론부터 말하면, 기본값은 **`server.close()` 중심의 graceful drain**이고, `closeIdleConnections()`는 keep-alive 잔류 연결을 더 빨리 정리하고 싶을 때 검토할 수 있습니다.
반면 `closeAllConnections()`는 강한 도구라서, 무조건 넣기보다 **종료 예산을 초과했을 때의 마지막 단계**로 두는 편이 안전합니다.

## closeIdleConnections와 closeAllConnections는 무엇이 다를까

### H3. `server.close()`는 새 연결을 막고 기존 작업이 끝나길 기다린다

Node.js에서 graceful shutdown의 출발점은 보통 `server.close()`입니다.
이 메서드는 새 연결 수락을 멈추고, 이미 들어온 연결이 자연스럽게 끝나기를 기다립니다.

즉 핵심 목적은 "지금 처리 중인 요청을 최대한 덜 깨뜨리면서 인스턴스를 빼는 것"입니다.
이 기본 흐름은 [Node.js Graceful Shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)에서 다룬 원칙과 같습니다.
먼저 트래픽 유입을 멈추고, 그다음에 남은 요청을 정리해야 합니다.

### H3. `closeIdleConnections()`는 놀고 있는 keep-alive 연결만 정리한다

`server.closeIdleConnections()`는 현재 요청을 처리 중이지 않은 idle 연결을 닫는 데 초점이 있습니다.
이미 응답은 끝났지만 keep-alive로 소켓이 열려 있는 상태를 더 빨리 걷어내고 싶을 때 유용합니다.

특히 아래 같은 상황에서 생각해볼 수 있습니다.

- 롤링 배포 중 일부 keep-alive 연결이 예상보다 오래 남는 경우
- 프록시 뒤에서 idle 연결 정리가 느려 종료 시간이 흔들리는 경우
- 새 요청은 막았는데 종료 완료 콜백이 늦게 오는 경우

이 메서드는 진행 중 요청까지 강제로 자르지는 않는다는 점에서 상대적으로 보수적인 편입니다.
그래도 프록시 timeout과 애플리케이션 timeout 정렬이 먼저라는 점은 변하지 않습니다.
이 부분은 [Node.js keepAliveTimeout, headersTimeout mismatch 가이드](/development/blog/seo/2026/04/14/nodejs-keepalive-timeout-headers-timeout-mismatch-guide.html)와 함께 보는 편이 좋습니다.

### H3. `closeAllConnections()`는 진행 중 연결까지 넓게 정리할 수 있다

`server.closeAllConnections()`는 이름 그대로 훨씬 공격적입니다.
idle 연결만이 아니라 현재 살아 있는 연결 전체를 강하게 정리하는 용도로 이해하는 편이 맞습니다.

문제는 이 방식이 종료를 빠르게 만들 수는 있어도, 잘못 쓰면 아직 처리 중인 요청까지 사용자 입장에서 끊긴 것처럼 보이게 만들 수 있다는 점입니다.
그래서 이 메서드는 기본 graceful shutdown 단계가 아니라, 보통 아래처럼 다룹니다.

- 종료 예산이 충분할 때는 사용하지 않음
- `server.close()`로 drain 시작
- 필요하면 `closeIdleConnections()`로 idle 연결 정리
- 그래도 제한 시간 내 종료되지 않으면 마지막 수단으로 검토

## 어떤 상황에서 closeIdleConnections가 특히 실용적일까

### H3. readiness를 내렸는데 keep-alive 잔류 때문에 인스턴스가 늦게 빠질 때

쿠버네티스나 ALB 뒤에서 운영하면 readiness를 내린 직후에도 기존 keep-alive 연결은 잠시 남을 수 있습니다.
이 자체는 정상입니다.
하지만 idle 소켓이 기대보다 길게 남아서 종료 순서가 지저분해지면 운영자가 보기에 꽤 찜찜합니다.

이럴 때 `closeIdleConnections()`는 유용한 보조 장치가 됩니다.
트래픽 차단 순서 자체는 [Node.js Readiness, Liveness, Startup Probe 가이드](/development/blog/seo/2026/04/14/nodejs-readiness-liveness-startup-probe-guide.html)처럼 가져가고, idle 연결만 조금 더 적극적으로 정리하는 식입니다.

### H3. 일부 장수 연결이 배포 드레이닝을 느리게 만들 때

오래 살아남은 keep-alive 연결이 분산을 흐트러뜨리는 문제는 [Node.js maxRequestsPerSocket 가이드](/development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html)에서도 살펴본 주제입니다.
그 글이 "연결을 너무 오래 쓰지 않게 순환시키는 방법"에 가깝다면, 여기서 다루는 메서드들은 **종료 시점에 남은 연결을 어떻게 정리할지**에 가깝습니다.

즉 평소에는 `maxRequestsPerSocket`으로 연결 생애주기 편차를 줄이고, 종료 순간에는 `closeIdleConnections()`로 잔류 idle 연결을 정리하는 식으로 역할이 다릅니다.

## 실무에서는 어떤 순서로 적용하는 게 좋을까

### H3. 기본 권장 순서는 readiness 차단, `server.close()`, idle 정리, 강제 종료다

실무에서는 아래 순서가 가장 설명하기 쉽습니다.

1. 종료 신호를 받으면 readiness를 실패로 전환한다.
2. 큐 소비나 스케줄러처럼 새 작업 유입도 함께 멈춘다.
3. `server.close()`로 새 연결 수락을 중단한다.
4. 필요하면 `closeIdleConnections()`로 idle keep-alive 연결을 정리한다.
5. 종료 예산 안에 끝나지 않으면 마지막 단계에서 강제 종료를 검토한다.

예시는 아래처럼 잡을 수 있습니다.

```js
const http = require('http');
const app = require('./app');

const server = http.createServer(app);
let shuttingDown = false;
let ready = true;

app.get('/readyz', (req, res) => {
  if (!ready) return res.status(503).send('shutting down');
  return res.status(200).send('ok');
});

function shutdown(signal) {
  if (shuttingDown) return;
  shuttingDown = true;
  ready = false;

  console.log({ signal }, 'shutdown started');

  server.close(() => {
    console.log('server closed gracefully');
    process.exit(0);
  });

  if (typeof server.closeIdleConnections === 'function') {
    setTimeout(() => {
      server.closeIdleConnections();
    }, 1000).unref();
  }

  setTimeout(() => {
    console.error('shutdown deadline exceeded');

    if (typeof server.closeAllConnections === 'function') {
      server.closeAllConnections();
    }

    process.exit(1);
  }, 10000).unref();
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));

server.listen(3000);
```

핵심은 `closeAllConnections()`를 먼저 쓰지 않는 것입니다.
서비스가 정상적으로 drain될 수 있는 시간을 먼저 주고, 마지막 예산 초과 지점에서만 더 강한 수단을 꺼내는 편이 운영 사고를 줄입니다.

### H3. 종료 예산은 오케스트레이터 설정과 같이 봐야 한다

애플리케이션 코드만 잘 짜도 끝나는 문제는 아닙니다.
쿠버네티스의 `terminationGracePeriodSeconds`, 프록시의 드레이닝 시간, 로드밸런서의 deregistration delay가 서로 어긋나면 종료 동작이 이상해집니다.

즉 아래 질문에 답할 수 있어야 합니다.

- readiness를 내린 뒤 로드밸런서가 실제로 언제 트래픽을 멈추는가
- 애플리케이션이 진행 중 요청을 정리하는 데 평균 얼마나 걸리는가
- 종료 예산을 초과할 때 사용자 영향은 어느 정도인가
- 마지막 강제 종료를 어디서 실행할 것인가

## closeAllConnections를 기본값으로 두면 왜 위험할까

### H3. 진행 중 요청이 잘리면 장애는 짧아져도 사용자 경험은 나빠질 수 있다

`closeAllConnections()`는 종료 시간을 짧게 보이게 만들 수 있습니다.
하지만 처리 중인 업로드, 스트리밍 응답, 긴 DB 작업을 기다리는 요청이 있었다면, 사용자 입장에서는 갑작스러운 실패로 보일 수 있습니다.

특히 아래 상황에서는 더 조심해야 합니다.

- 응답 시간이 긴 API가 있는 경우
- 파일 업로드나 다운로드가 있는 경우
- SSE, long polling, 스트리밍 응답이 있는 경우
- 외부 API 응답을 기다리느라 인플라이트 시간이 긴 경우

이럴수록 강제 정리 전의 유예 시간이 중요합니다.
단순히 빨리 꺼지는 것이 아니라, **덜 깨지게 꺼지는 것**이 목표여야 합니다.

### H3. 치명 오류 대응과 정상 배포 종료는 분리해서 생각하는 편이 낫다

`uncaughtException`이나 `unhandledRejection` 같은 치명 오류 상황은 정상 배포 종료와 다릅니다.
프로세스 상태 신뢰성이 이미 떨어졌다면 더 공격적으로 종료하는 판단이 나올 수 있습니다.

이 기준은 [Node.js unhandledRejection, uncaughtException 대응 가이드](/development/blog/seo/2026/04/15/nodejs-unhandledrejection-uncaughtexception-shutdown-guide.html)와 이어집니다.
정상 배포에서는 drain 우선, 치명 오류에서는 기록 후 제한 시간 내 종료 우선으로 가져가면 혼선이 적습니다.

## 운영에서 같이 봐야 할 메트릭은 무엇일까

### H3. 종료 시간만 보지 말고 끊긴 요청과 reset도 같이 본다

이 주제는 종료 성공 여부만 보면 오판하기 쉽습니다.
예를 들어 종료는 빨라졌는데 사용자 에러가 늘었다면 좋은 변경이 아닙니다.

가능하면 아래 지표를 함께 보세요.

- 종료 시작부터 프로세스 종료까지 걸린 시간
- 종료 중 5xx, `ECONNRESET`, socket hang up 증가 여부
- 인플라이트 요청 수 감소 곡선
- keep-alive idle 연결 수 감소 속도
- 배포 직후 인스턴스별 요청 분포

### H3. 요청 컨텍스트 로그를 남기면 종료 품질을 더 쉽게 해석할 수 있다

종료 중 실패한 요청이 어떤 경로였는지 모르면 개선도 어렵습니다.
가능하면 요청 ID, 배포 버전, 종료 시작 시각을 같이 남겨두는 편이 좋습니다.
이런 분석은 [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)와 조합할 때 특히 효과적입니다.

## 실무 체크리스트

### H3. 적용 전 체크

- readiness 전환이 실제 트래픽 차단으로 이어지는지 확인했는가
- `server.close()`만으로 충분한지 먼저 검증했는가
- keep-alive idle 연결이 실제 종료 지연 원인인지 메트릭으로 확인했는가
- 종료 예산과 강제 종료 시점을 문서화했는가

### H3. 적용 후 체크

- 배포 종료 시간이 안정적으로 줄었는가
- 종료 중 사용자 에러가 늘지 않았는가
- 스트리밍, 업로드, 긴 요청 경로가 의도치 않게 잘리지 않았는가
- `closeAllConnections()`가 기본 경로가 아니라 예외 경로로만 동작하는가

## 마무리

`closeIdleConnections()`와 `closeAllConnections()`는 둘 다 유용하지만 성격이 다릅니다.
`closeIdleConnections()`는 graceful shutdown을 보완하는 도구이고, `closeAllConnections()`는 종료 예산을 넘겼을 때 꺼낼 수 있는 더 강한 수단에 가깝습니다.

운영에서는 멋진 메서드 이름보다 종료 순서가 더 중요합니다.
먼저 readiness를 내리고, `server.close()`로 drain을 시작하고, 필요할 때만 idle 연결 정리와 강제 종료를 단계적으로 추가하세요.
그렇게 해야 배포는 빨라지면서도, 사용자 경험은 덜 망가집니다.

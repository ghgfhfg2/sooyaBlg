---
layout: post
title: "Node.js unhandledRejection, uncaughtException 대응 가이드: 프로세스를 언제 종료해야 할까"
date: 2026-04-15 08:00:00 +0900
lang: ko
translation_key: nodejs-unhandledrejection-uncaughtexception-shutdown-guide
permalink: /development/blog/seo/2026/04/15/nodejs-unhandledrejection-uncaughtexception-shutdown-guide.html
alternates:
  ko: /development/blog/seo/2026/04/15/nodejs-unhandledrejection-uncaughtexception-shutdown-guide.html
  x_default: /development/blog/seo/2026/04/15/nodejs-unhandledrejection-uncaughtexception-shutdown-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, unhandledrejection, uncaughtexception, gracefulshutdown, errorhandling, reliability, backend, production]
description: "Node.js에서 unhandledRejection, uncaughtException이 발생했을 때 로그만 남기고 계속 실행해도 되는지, 언제 종료하고 어떻게 graceful shutdown으로 복구해야 하는지 실무 기준으로 정리했습니다."
---

Node.js 운영 중 `unhandledRejection`이나 `uncaughtException` 로그를 보면 일단 잡아서 무시하고 넘어가고 싶을 때가 많습니다.
서비스가 살아 있기만 하면 괜찮아 보이기 때문입니다.
그런데 이 두 이벤트는 단순 에러 로그가 아니라, **프로세스 상태가 이미 신뢰하기 어려워졌다는 신호**일 수 있습니다.

이 글에서는 `unhandledRejection`, `uncaughtException`의 차이, 언제 프로세스를 종료해야 하는지, 그리고 종료하더라도 사용자 영향과 장애 전파를 줄이는 운영 패턴을 정리합니다.

## unhandledRejection과 uncaughtException은 왜 다르게 봐야 할까

### H3. unhandledRejection은 비동기 흐름에서 놓친 실패다

`unhandledRejection`은 Promise가 reject됐는데 적절한 `catch`가 연결되지 않았을 때 발생합니다.
대표적으로 아래 상황에서 자주 나옵니다.

- `await` 없이 Promise를 날리고 에러 처리를 빠뜨린 경우
- 배치 작업이나 이벤트 핸들러에서 반환된 Promise를 호출자가 기다리지 않은 경우
- 라이브러리 래퍼를 만들면서 에러를 다시 던지는 흐름이 끊긴 경우

예를 들어 이런 코드는 위험합니다.

```js
app.post('/users', async (req, res) => {
  createUser(req.body); // await 누락
  res.status(202).send({ ok: true });
});
```

`createUser()` 내부에서 실패해도 현재 요청 흐름에서는 그 에러를 관찰하지 못할 수 있습니다.
이런 문제는 요청은 성공처럼 보이는데 실제 작업은 실패하는 형태라서 더 위험합니다.

### H3. uncaughtException은 동기 예외가 프로세스 최상단까지 올라온 상태다

`uncaughtException`은 try/catch로 처리되지 않은 예외가 이벤트 루프 최상단까지 도달했을 때 발생합니다.
즉 현재 실행 문맥에서 **복구 책임이 사라진 상태**라고 보는 편이 맞습니다.

아래처럼 단순 예시로도 재현됩니다.

```js
setTimeout(() => {
  throw new Error('unexpected crash');
}, 10);
```

이 시점에는 일부 리소스가 반쯤 정리됐을 수도 있고, 요청 컨텍스트가 깨졌을 수도 있습니다.
그래서 `uncaughtException`을 잡더라도 "계속 서비스"보다 "빠르게 기록하고 종료"가 기본 전략이 되는 경우가 많습니다.

## 프로세스를 계속 살려도 되는가

### H3. 로그만 남기고 계속 실행하는 전략은 대체로 위험하다

실무에서 가장 흔한 안티패턴은 아래와 같습니다.

```js
process.on('unhandledRejection', (reason) => {
  console.error('unhandledRejection', reason);
});

process.on('uncaughtException', (err) => {
  console.error('uncaughtException', err);
});
```

겉으로는 장애를 막은 것처럼 보이지만, 실제로는 문제를 늦게 더 크게 터뜨릴 수 있습니다.
이유는 간단합니다.

- 일부 요청 상태가 이미 불일치할 수 있음
- DB 트랜잭션, 메시지 ack, 캐시 갱신이 절반만 끝났을 수 있음
- 메모리 손상이나 잘못된 전역 상태가 다음 요청에도 영향을 줄 수 있음
- 장애 원인이 묻혀서 재현과 분석이 더 어려워짐

특히 운영 안정성을 중요하게 본다면 [Node.js Graceful Shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)에서 다룬 것처럼, 비정상 상태를 길게 끌기보다 **짧게 배수진을 치고 정상 인스턴스로 교체**하는 쪽이 낫습니다.

### H3. 예외를 수집하는 것과 복구하는 것은 다른 문제다

이벤트 핸들러를 등록하는 목적은 "완전 복구"가 아니라 보통 아래 두 가지입니다.

1. 원인 분석에 필요한 로그와 메트릭 남기기
2. 종료 전에 새 요청 유입을 막고 진행 중인 작업을 정리하기

즉 `process.on(...)`은 생존 장치라기보다 **통제된 종료 장치**에 가깝습니다.

## 실무 권장 패턴은 기록 후 graceful shutdown이다

### H3. 새 요청부터 차단하고 종료 예산 안에서 정리한다

아래 흐름으로 설계하면 운영이 한결 단순해집니다.

1. 치명 이벤트 감지
2. 구조화 로그, 알림, 에러 컨텍스트 기록
3. readiness 실패 처리 또는 로드밸런서 대상 제외
4. 새 요청 유입 차단
5. 진행 중 요청, 큐 작업, 연결 정리
6. 제한 시간 안에 종료
7. 프로세스 매니저나 오케스트레이터가 재기동

예시 코드는 아래처럼 잡을 수 있습니다.

```js
const http = require('http');
const app = require('./app');

const server = http.createServer(app);
let shuttingDown = false;

function fatal(event, error) {
  if (shuttingDown) return;
  shuttingDown = true;

  console.error({ event, error }, 'fatal process event');

  server.close(() => {
    process.exit(1);
  });

  setTimeout(() => {
    process.exit(1);
  }, 10000).unref();
}

process.on('unhandledRejection', (reason) => {
  fatal('unhandledRejection', reason);
});

process.on('uncaughtException', (error) => {
  fatal('uncaughtException', error);
});

server.listen(3000);
```

핵심은 에러를 무시하지 않고, 종료를 무한정 미루지도 않는 것입니다.
종료 예산이 없으면 hung 상태가 길어지고, 너무 짧으면 진행 중 요청이 불필요하게 잘립니다.

### H3. 종료 전에는 헬스체크 상태도 같이 바꿔야 한다

쿠버네티스나 ALB 뒤에서 운영한다면 프로세스 종료만으로는 부족합니다.
종료 시작과 함께 readiness를 실패시키지 않으면, 드레이닝 중인 인스턴스로 요청이 계속 들어올 수 있습니다.

이 부분은 [Node.js Readiness, Liveness, Startup Probe 가이드](/development/blog/seo/2026/04/14/nodejs-readiness-liveness-startup-probe-guide.html)와 같이 보는 편이 좋습니다.
치명 오류 대응은 에러 핸들링 문제이면서 동시에 **트래픽 차단 순서 문제**이기도 합니다.

## unhandledRejection은 무조건 종료해야 할까

### H3. 핵심은 원인을 확실히 알고 국소적으로 격리 가능한지다

모든 `unhandledRejection`이 반드시 동일한 강도로 위험한 것은 아닙니다.
예를 들어 백그라운드성 보조 작업에서 실패했고, 해당 실패가 요청 상태나 전역 상태를 오염시키지 않는다고 강하게 확신할 수도 있습니다.

하지만 실무에서는 이 확신이 과장되는 경우가 많습니다.
특히 아래 조건이 하나라도 있으면 종료 쪽으로 기우는 것이 안전합니다.

- 어떤 요청 또는 작업이 영향을 받았는지 추적이 어려움
- 전역 싱글톤 상태를 건드리는 로직이 포함됨
- DB 쓰기, 메시지 발행, 외부 API 호출이 섞여 있음
- 같은 코드 경로에서 재발 빈도가 높음

반대로 완전히 격리된 작업 러너, 일회성 배치, 실패 재처리가 명확한 잡 시스템이라면 프로세스 전체 종료 대신 작업 단위 실패로 수습할 수도 있습니다.
다만 이 경우에도 **왜 unhandled 상태까지 갔는지**는 반드시 고쳐야 합니다.

### H3. Node.js 버전과 런타임 옵션 차이도 확인해야 한다

`unhandledRejection`의 기본 동작은 Node.js 버전이나 실행 옵션에 따라 체감이 달라질 수 있습니다.
그래서 팀 내에서는 "우리 서비스가 reject를 어떤 정책으로 다루는지"를 문서로 고정해두는 것이 좋습니다.

운영에서 중요한 것은 세부 기본값 암기보다 아래입니다.

- 프로세스 이벤트를 관찰하고 있는가
- 치명 오류 시 종료 정책이 명시돼 있는가
- 재기동 주체(systemd, PM2, Kubernetes)가 준비돼 있는가
- 에러 발생 시 중복 작업을 막는 장치가 있는가

중복 실행 방지는 [Node.js Idempotency Key 가이드](/development/blog/seo/2026/04/10/nodejs-idempotency-key-duplicate-request-prevention-guide.html)와도 연결됩니다.
치명 오류 이후 재시도나 재전송이 일어날 수 있기 때문입니다.

## 운영 환경에서 같이 챙겨야 할 것들

### H3. 구조화 로그와 요청 컨텍스트를 남긴다

치명 오류가 발생했을 때 텍스트 로그 한 줄만 남으면 원인 추적이 어렵습니다.
가능하면 아래 정보를 함께 남기는 편이 좋습니다.

- 에러 이름, 메시지, stack
- 요청 ID, 사용자 영향 범위, 배포 버전
- 현재 처리 중이던 큐 작업 또는 크론 이름
- 프로세스 메모리, 이벤트 루프 지연, 재시작 횟수

이때 요청 단위 추적은 [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/04/13/nodejs-async-local-storage-request-context-logging-guide.html)와 조합하면 훨씬 유용합니다.
에러는 결국 "어느 요청에서, 어떤 컨텍스트로" 발생했는지가 중요합니다.

### H3. 종료 중에는 타임아웃과 백프레셔를 같이 관리한다

graceful shutdown이 있어도 정리해야 할 작업이 너무 많으면 종료가 늘어질 수 있습니다.
그래서 아래를 함께 보면 좋습니다.

- 큐 소비 중단
- 새 DB 커넥션 생성 차단
- 외부 API 재시도 중지
- 요청별 deadline 초과 작업 빠르게 중단
- 종료 예산 초과 시 강제 종료

타임아웃 경계 설계는 [Node.js Deadline Exceeded 에러 처리 가이드](/development/blog/seo/2026/04/12/nodejs-deadline-exceeded-error-handling-guide.html)와도 맞물립니다.
치명 오류 상황에서는 평소보다 더 공격적으로 정리 기준을 적용해야 할 때가 많습니다.

## 점검 체크리스트

### H3. 발행 전에 팀 문서로 남겨두면 좋은 항목

- `unhandledRejection`, `uncaughtException` 핸들러를 등록했는가
- 핸들러 목적이 "로그 후 계속 실행"이 아니라 "로그 후 통제된 종료"인가
- readiness 실패 전환과 `server.close()` 순서가 정의돼 있는가
- 종료 유예 시간과 강제 종료 시간이 명시돼 있는가
- 프로세스 재기동 주체가 준비돼 있는가
- 재시도 시 중복 처리 방지 장치가 있는가
- 치명 오류 발생 시 알림과 관측성 데이터가 남는가

## 마무리

Node.js의 `unhandledRejection`, `uncaughtException`은 단순히 콘솔에 찍고 넘어갈 로그가 아닙니다.
많은 경우 이것은 "이 프로세스를 더 신뢰해도 되는가"라는 운영 질문입니다.

정리하면 실무 기준은 아래에 가깝습니다.

- `uncaughtException`은 대체로 치명 오류로 보고 종료를 기본값으로 둔다.
- `unhandledRejection`도 원인과 영향 범위가 확실하지 않다면 종료 쪽이 안전하다.
- 종료할 때는 그냥 죽지 말고, 로그, 드레이닝, 재기동까지 한 흐름으로 설계한다.

서비스를 오래 안정적으로 운영하려면, 에러를 안 보이게 숨기는 것보다 **망가진 프로세스를 빨리 정상 교체하는 시스템**을 만드는 쪽이 훨씬 낫습니다.

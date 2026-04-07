---
layout: post
title: "Node.js Graceful Shutdown 가이드: In-Flight 요청을 안전하게 Drain하고 배포 중 에러를 줄이는 법"
date: 2026-04-07 20:00:00 +0900
lang: ko
translation_key: nodejs-graceful-shutdown-inflight-request-draining-guide
permalink: /development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html
alternates:
  ko: /development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html
  x_default: /development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, graceful-shutdown, inflight-requests, deployment, reliability, zero-downtime, backend, kubernetes]
description: "Node.js 서비스에서 graceful shutdown을 구현해 새 요청은 막고 in-flight 요청은 안전하게 drain하며 배포 중 5xx와 연결 끊김을 줄이는 실무 패턴을 정리했습니다."
---

배포 자체는 성공했는데 직후에 502, 503, socket hang up, ECONNRESET 같은 오류가 잠깐씩 튀는 서비스가 있습니다.
애플리케이션은 살아 있고 헬스체크도 통과하는데, 롤링 배포나 재시작 순간에만 사용자 요청이 어색하게 끊기는 경우입니다.
이때 흔히 놓치는 것이 **graceful shutdown**입니다.

graceful shutdown의 핵심은 단순히 프로세스를 늦게 종료하는 것이 아닙니다.
**새 요청은 더 이상 받지 않고, 이미 처리 중인 in-flight 요청은 가능한 한 안전하게 마무리한 뒤 종료하는 것**입니다.
이 글에서는 Node.js 환경에서 graceful shutdown이 왜 중요한지, SIGTERM을 받았을 때 어떤 순서로 종료해야 하는지, 그리고 load balancer·Kubernetes·timeout budget과 어떻게 맞물려야 하는지 실무 관점에서 정리합니다.

## Node.js Graceful Shutdown이 왜 중요한가

### H3. 종료 순간의 짧은 공백이 사용자 체감 오류로 바로 이어질 수 있다

많은 팀이 평상시 latency와 에러율은 열심히 보지만, 종료 순간은 상대적으로 덜 봅니다.
하지만 실서비스에서는 배포, 오토스케일링 축소, 인스턴스 교체, 장애 복구 재시작이 반복적으로 일어납니다.
이때 종료 절차가 거칠면 아래 문제가 자주 생깁니다.

- 처리 중이던 HTTP 요청이 중간에 끊긴다
- keep-alive 연결이 갑자기 닫혀 클라이언트가 재시도한다
- 메시지 소비 중이던 작업이 애매한 상태로 남는다
- 종료 직전까지 새 요청을 받아 tail latency가 급증한다
- readiness는 내려갔지만 이미 들어온 요청 정리가 끝나기 전에 프로세스가 죽는다

즉 graceful shutdown은 보기 좋은 마무리가 아니라, **배포 중 품질 저하를 줄이는 보호 장치**입니다.
특히 트래픽이 있는 서비스에서는 종료 설계가 곧 가용성 설계와 연결됩니다.

### H3. 단순 process.exit는 빠르지만 가장 비싼 종료 방식일 수 있다

Node.js에서 종료를 서두를 때 흔히 보이는 실수는 signal을 받자마자 `process.exit(0)`을 호출하는 것입니다.
이 방식은 겉보기엔 깔끔하지만 실제로는 가장 거친 종료에 가깝습니다.

이렇게 되면 아래 문제가 생기기 쉽습니다.

- 아직 응답을 쓰는 중인 요청이 중단된다
- DB 쓰기나 외부 API 호출 결과가 불명확해진다
- 로그 flush, metric export, trace 전송이 누락될 수 있다
- 워커가 소비 중이던 메시지 ack 타이밍이 꼬일 수 있다

종료는 “빨리 죽는 것”보다 **안전하게 비우는 것**이 더 중요합니다.
이 점은 [Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와도 이어집니다.
시스템은 요청 처리뿐 아니라 종료에도 시간 예산이 필요합니다.

## SIGTERM을 받으면 어떤 순서로 종료해야 할까

### H3. 첫 단계는 새 요청을 막고 종료 상태를 명확하게 알리는 것이다

가장 먼저 해야 할 일은 “이 인스턴스는 이제 내려갈 예정”이라는 사실을 시스템 전체에 반영하는 것입니다.
보통은 아래 순서가 안정적입니다.

1. 종료 플래그를 켠다
2. readiness probe를 실패시키거나 로드밸런서에서 제외되게 만든다
3. 새 요청 수락을 중단한다
4. 기존 in-flight 요청이 끝날 시간을 준다
5. 남은 연결과 리소스를 정리한 뒤 종료한다

여기서 핵심은 **새 요청 차단이 기존 요청 drain보다 먼저 와야 한다**는 점입니다.
새 요청을 계속 받으면서 drain을 기대하면 종료는 끝나지 않거나, 결국 강제 종료로 끝날 가능성이 큽니다.

Kubernetes를 쓴다면 이 단계는 [Readiness/Liveness Probe 가이드](/development/blog/seo/2026/03/10/readiness-liveness-probe-deploy-reliability-guide.md)와 직접 연결됩니다.
readiness를 먼저 내리지 않으면 트래픽이 계속 들어오기 때문입니다.

### H3. 그다음은 in-flight 요청 수와 종료 deadline을 함께 관리해야 한다

종료를 무한정 기다릴 수는 없습니다.
느린 외부 의존성이나 hanging request가 하나라도 있으면 프로세스가 끝나지 않을 수 있기 때문입니다.
그래서 graceful shutdown에는 항상 **종료 deadline**이 필요합니다.

보통은 아래처럼 생각합니다.

- 0초: SIGTERM 수신, 종료 모드 진입
- 0~2초: readiness 제외 전파, 새 요청 차단
- 2~20초: in-flight 요청 drain 대기
- 20초 이후: 남은 연결 강제 종료, 프로세스 종료

정확한 숫자는 서비스마다 다르지만, 원칙은 같습니다.
**기다릴 시간은 주되 끝은 있어야 한다**는 것입니다.

## Node.js에서 Graceful Shutdown을 구현하는 기본 패턴

### H3. HTTP 서버 close만 호출해도 절반은 해결되지만, 절반은 여전히 남아 있다

Node.js의 `server.close()`는 새 연결 수락을 멈추고 기존 연결이 닫히기를 기다리는 데 도움이 됩니다.
그래서 graceful shutdown의 출발점으로는 좋습니다.
하지만 이것만으로는 완전하지 않습니다.

이유는 아래와 같습니다.

- keep-alive 연결이 오래 남을 수 있다
- 장시간 요청이나 스트리밍 응답은 자연 종료가 늦을 수 있다
- 애플리케이션이 따로 관리하는 DB, Redis, queue consumer는 자동 정리되지 않는다
- 종료 중 들어온 요청에 대해 일관된 응답 정책이 필요하다

즉 `server.close()`는 필요하지만, **애플리케이션 레벨 drain 정책**이 함께 있어야 실무에서 안정적입니다.

### H3. 종료 상태와 in-flight 카운터를 두면 동작이 훨씬 예측 가능해진다

아래 예시는 HTTP 요청 수를 추적하면서 SIGTERM 수신 시 새 요청을 막고, 일정 시간 동안 in-flight 요청을 기다렸다가 종료하는 단순한 패턴입니다.

```js
import http from 'node:http';

let isShuttingDown = false;
let inFlight = 0;

const server = http.createServer(async (req, res) => {
  if (isShuttingDown) {
    res.setHeader('Connection', 'close');
    res.statusCode = 503;
    res.end('server is shutting down');
    return;
  }

  inFlight += 1;

  try {
    await handleRequest(req, res);
  } finally {
    inFlight -= 1;
  }
});

server.listen(3000, () => {
  console.log('server listening on 3000');
});

process.on('SIGTERM', () => {
  gracefulShutdown().catch((err) => {
    console.error('graceful shutdown failed', err);
    process.exit(1);
  });
});

async function gracefulShutdown() {
  if (isShuttingDown) return;
  isShuttingDown = true;

  const deadlineMs = 20_000;
  const startedAt = Date.now();

  await new Promise((resolve, reject) => {
    server.close((err) => {
      if (err) reject(err);
      else resolve();
    });
  });

  while (inFlight > 0 && Date.now() - startedAt < deadlineMs) {
    await new Promise((r) => setTimeout(r, 200));
  }

  process.exit(0);
}

async function handleRequest(req, res) {
  await new Promise((r) => setTimeout(r, 150));
  res.statusCode = 200;
  res.end('ok');
}
```

이 예시의 포인트는 세 가지입니다.

- 종료 상태를 명시적으로 관리한다
- 새 요청에는 빠르게 503을 돌려 더 깊이 들어오지 않게 한다
- 기존 요청은 deadline 안에서만 기다린다

프로덕션에서는 여기에 DB pool 종료, queue consumer 중지, trace flush, keep-alive timeout 조정 등을 더해야 합니다.

## Kubernetes와 Load Balancer 환경에서는 무엇을 추가로 봐야 할까

### H3. readiness를 내리는 타이밍과 preStop 동작이 어긋나면 배포 중 오류가 남는다

Kubernetes에서는 애플리케이션 코드만 잘 짜도 끝나지 않습니다.
오케스트레이터와 종료 순서를 맞춰야 합니다.
가장 흔한 문제는 아래 조합입니다.

- SIGTERM은 받았는데 readiness가 아직 정상
- readiness는 내려갔지만 외부 load balancer 반영이 늦음
- `terminationGracePeriodSeconds`가 너무 짧아 drain 전에 강제 종료됨

실무에서는 보통 아래를 같이 점검합니다.

- 종료 시 즉시 readiness 실패 전환
- 필요하면 `preStop`에서 짧은 sleep으로 제외 전파 시간 확보
- `terminationGracePeriodSeconds`를 실제 longest in-flight 요청보다 넉넉히 설정
- 장시간 스트리밍이나 파일 다운로드가 있으면 별도 정책 적용

즉 graceful shutdown은 앱 코드만의 문제가 아니라, **플랫폼과의 계약**입니다.
앱은 준비됐는데 플랫폼 시간이 부족하면 결국 끊깁니다.

### H3. keep-alive와 프록시 timeout이 어긋나면 종료가 예상보다 오래 끌 수 있다

종료가 잘 안 되는 이유 중 하나는 연결이 생각보다 오래 살아 있기 때문입니다.
특히 keep-alive가 긴 환경에서는 새 비즈니스 요청은 끝났어도 TCP 연결이 남아 종료를 지연시킬 수 있습니다.

점검할 항목은 아래와 같습니다.

- Node.js `keepAliveTimeout`
- reverse proxy의 idle timeout
- ingress/controller의 drain 정책
- 클라이언트 재시도 및 connection reuse 방식

이 값들이 서로 크게 어긋나면 애플리케이션은 종료됐다고 생각하는데 프록시 관점에서는 여전히 연결이 살아 있을 수 있습니다.
배포 직후 간헐적 연결 오류가 난다면 코드뿐 아니라 **네트워크 계층 timeout 정렬**도 같이 봐야 합니다.

## Graceful Shutdown과 Load Shedding은 어떻게 다를까

### H3. graceful shutdown은 종료 상황용이고, load shedding은 과부하 상황용이다

둘 다 요청을 거절할 수 있어서 비슷해 보이지만 목적은 다릅니다.

- graceful shutdown: 인스턴스가 내려가는 동안 새 요청을 받지 않기 위한 보호
- load shedding: 인스턴스는 살아 있지만 과부하 상태에서 일부 요청을 버려 전체를 보호하기 위한 전략

운영에서는 둘을 같이 쓰는 경우가 많습니다.
예를 들어 종료 중인 인스턴스는 모든 새 요청을 503으로 거절하고, 살아 있는 인스턴스는 부하 수준에 따라 낮은 우선순위 요청만 shed할 수 있습니다.
이 차이는 [Node.js Load Shedding 가이드](/development/blog/seo/2026/04/07/nodejs-load-shedding-overload-protection-guide.html)와 이어서 보면 더 명확합니다.

### H3. 종료 중 503 응답은 실패라기보다 더 큰 실패를 막기 위한 신호다

종료 중인 인스턴스가 요청을 억지로 받다가 중간에 끊는 것보다, 초기에 503으로 명확히 거절하는 편이 더 낫습니다.
클라이언트나 프록시가 재시도하기도 쉬워지고, 어떤 인스턴스가 이미 drain 중인지 운영 관측도 쉬워집니다.

핵심은 “무조건 성공 처리”가 아니라 **애매한 실패를 줄이는 것**입니다.
graceful shutdown은 그 관점에서 매우 실용적인 안정성 패턴입니다.

## 실무에서 자주 놓치는 함정

### H3. HTTP만 닫고 백그라운드 워커는 계속 돌게 두는 경우

API 서버가 동시에 queue consumer, cron worker, WebSocket 브로드캐스터 역할도 수행하는 구조에서는 HTTP 종료만 처리하면 반쪽짜리가 됩니다.
아래 리소스도 함께 정리해야 합니다.

- DB connection pool
- Redis 연결
- Kafka, RabbitMQ, SQS consumer
- BullMQ 같은 job worker
- scheduler, interval, background timer

특히 메시지 소비는 ack 시점이 중요해서, 종료 중에 새 메시지를 계속 가져오지 않도록 **consumer pause → in-progress 처리 마무리 → connection close** 순서를 명확히 잡는 편이 안전합니다.

### H3. 종료 deadline 없이 무작정 기다리면 롤링 배포가 오히려 막힌다

graceful shutdown을 넣었는데 배포 시간이 이상하게 길어지는 팀도 많습니다.
대개는 종료 대기만 있고 마감 시간이 없을 때 생깁니다.
그러면 한두 개의 느린 요청 때문에 롤링 배포 전체가 늘어질 수 있습니다.

이 문제를 줄이려면 아래 원칙이 필요합니다.

- longest normal request time을 기준으로 종료 예산을 정한다
- 그보다 긴 요청은 별도 경로로 분리하거나 timeout을 명시한다
- 종료 중에는 신규 long-running 작업을 받지 않는다
- 마감 시간이 지나면 연결을 강제 정리한다

즉 graceful shutdown은 무한한 배려가 아니라 **통제된 배려**여야 합니다.

## 운영 지표와 점검 체크리스트

### H3. 발행 전이나 배포 전이 아니라, 실제 종료 이벤트 때 지표를 봐야 한다

graceful shutdown은 평상시보다 배포 직전과 직후에 진짜 품질이 드러납니다.
아래 지표를 같이 보면 좋습니다.

- 배포 직후 5xx 비율 변화
- SIGTERM 이후 종료 완료까지 걸린 시간
- 종료 중 in-flight 요청 최대 개수
- 종료 중 강제 종료된 연결 수
- readiness 제외 후에도 들어온 요청 수
- 종료 직전/직후 retry 증가율

특히 “배포는 성공했지만 직후 몇 분간 에러가 튄다”는 패턴이 있다면 graceful shutdown과 readiness 전파 타이밍부터 의심해 보는 편이 좋습니다.

### H3. 적용 전 체크리스트

- SIGTERM 수신 시 종료 모드로 전환하는가?
- readiness 또는 트래픽 제외 신호가 먼저 반영되는가?
- 새 요청 수락 중단과 in-flight drain 순서가 맞는가?
- 종료 deadline과 강제 종료 기준이 있는가?
- HTTP 외의 worker, consumer, DB pool도 함께 정리되는가?

### H3. 적용 후 체크리스트

- 배포 중 502, 503, ECONNRESET이 줄었는가?
- 종료 시간이 `terminationGracePeriodSeconds` 안에 안정적으로 들어오는가?
- 장시간 요청이 drain을 방해하고 있지 않은가?
- 종료 중 503이 의도한 범위에서만 발생하는가?
- 프록시와 애플리케이션 timeout이 서로 어긋나지 않는가?

## 마무리

Node.js에서 graceful shutdown은 옵션이 아니라, 운영하는 서비스라면 거의 필수에 가깝습니다.
특히 롤링 배포, 오토스케일링, 장애 복구가 일상인 환경에서는 **어떻게 시작하느냐만큼 어떻게 끝내느냐도 중요**합니다.

핵심은 복잡하지 않습니다.
새 요청은 빨리 막고, 이미 처리 중인 요청은 제한된 시간 안에서 안전하게 마무리하고, 남은 리소스를 정리한 뒤 종료하면 됩니다.
이 흐름이 정리돼 있으면 배포 중의 짧은 불안정 구간을 꽤 많이 줄일 수 있습니다.

지금 서비스가 배포 직후에만 유난히 흔들린다면, 다음에 봐야 할 것은 CPU나 replica 수가 아니라 **종료 절차가 정말 graceful한가**일 수 있습니다.

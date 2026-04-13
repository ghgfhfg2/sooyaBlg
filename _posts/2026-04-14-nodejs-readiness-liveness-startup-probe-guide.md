---
layout: post
title: "Node.js Readiness, Liveness, Startup Probe 가이드: 쿠버네티스 헬스체크를 장애 없이 설계하는 법"
date: 2026-04-14 08:00:00 +0900
lang: ko
translation_key: nodejs-readiness-liveness-startup-probe-guide
permalink: /development/blog/seo/2026/04/14/nodejs-readiness-liveness-startup-probe-guide.html
alternates:
  ko: /development/blog/seo/2026/04/14/nodejs-readiness-liveness-startup-probe-guide.html
  x_default: /development/blog/seo/2026/04/14/nodejs-readiness-liveness-startup-probe-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, kubernetes, readiness-probe, liveness-probe, startup-probe, healthcheck, backend, reliability]
description: "Node.js 서비스에서 readiness probe, liveness probe, startup probe를 어떻게 나누고 어떤 체크를 넣어야 불필요한 재시작과 장애 전파를 줄일 수 있는지 실무 기준으로 정리했습니다."
---

쿠버네티스에 올린 Node.js 서비스가 간헐적으로 재시작되거나, 아직 준비되지 않은 pod로 트래픽이 들어가 장애가 커지는 경우가 있습니다.
이럴 때 원인을 따라가 보면 애플리케이션 코드 자체보다 **헬스체크 설계가 모호한 경우**가 꽤 많습니다.
readiness probe, liveness probe, startup probe를 제대로 나누지 않으면 정상 pod도 죽이고, 반대로 문제 pod를 너무 오래 살려둘 수도 있습니다.

이 글에서는 Node.js 서비스 기준으로 각 probe의 역할을 어떻게 구분해야 하는지, 어떤 체크를 넣어야 하고 무엇은 넣지 말아야 하는지, 그리고 운영에서 흔히 생기는 실수를 어떻게 피할지 정리합니다.

## Readiness, Liveness, Startup Probe의 차이부터 명확히 구분하기

### H3. readiness probe는 트래픽을 받아도 되는 상태인지 본다

readiness probe는 컨테이너가 살아 있는지가 아니라 **현재 요청을 받아도 되는지**를 판단합니다.
이 값이 실패하면 pod는 죽지 않지만, service endpoint에서 빠져 새 요청을 받지 않게 됩니다.

Node.js에서는 아래 상황을 readiness 실패로 보는 편이 실무적입니다.

- 앱 부팅은 끝났지만 필수 의존성 연결이 아직 준비되지 않음
- DB pool, Redis, 메시지 브로커 등 핵심 dependency가 초기화 중임
- 배포 직후 캐시 워밍이나 마이그레이션 확인이 끝나지 않음
- graceful shutdown 중이라 새 요청을 더 받으면 안 됨

즉 readiness는 "프로세스가 떠 있나"가 아니라 "**지금 이 인스턴스에 트래픽을 붙여도 안전한가**"를 묻는 체크입니다.

### H3. liveness probe는 프로세스가 회복 불가능한 상태인지 본다

liveness probe는 애플리케이션이 스스로 회복할 수 없을 정도로 망가졌는지 판단해 재시작을 유도합니다.
그래서 지나치게 민감하면 정상적인 순간적 지연도 장애로 오인하고 pod를 계속 재시작하게 만듭니다.

Node.js liveness는 보통 아래 질문에 답해야 합니다.

- 이벤트 루프가 사실상 멈췄는가
- 프로세스가 deadlock 비슷한 상태에 빠졌는가
- 내부적으로 치명적 오류 상태를 감지했는가

반대로 아래는 liveness에 바로 넣지 않는 편이 좋습니다.

- 외부 DB 일시 장애
- Redis 순간 timeout
- downstream API 응답 지연
- 일시적인 트래픽 급증

이런 문제는 재시작한다고 자동 해결되지 않는 경우가 많습니다.
오히려 liveness가 과민하면 장애 중에 pod churn만 키웁니다.
이런 과부하 전파를 줄이는 관점은 [Node.js Load Shedding 가이드](/development/blog/seo/2026/04/07/nodejs-load-shedding-overload-protection-guide.html)와도 연결됩니다.

### H3. startup probe는 느린 부팅을 정상으로 인정하는 장치다

startup probe는 앱 부팅이 오래 걸리는 동안 liveness가 섣불리 개입하지 못하게 막아줍니다.
Node.js 앱은 상대적으로 빨리 뜨는 편이지만, 실제 운영에서는 아래 요인 때문에 시작 시간이 길어질 수 있습니다.

- 대형 설정 로딩
- 초기 schema 검증
- secret fetch
- warm-up 작업
- 초기 외부 연결 설정

startup probe가 없으면 애플리케이션이 아직 부팅 중인데 liveness가 실패를 누적해 재시작시키는 악순환이 생길 수 있습니다.
부팅이 긴 서비스일수록 startup probe를 따로 두는 편이 훨씬 안전합니다.

## Node.js 서비스에서 probe를 어떻게 설계해야 하나

### H3. 헬스체크 엔드포인트는 역할별로 분리한다

가장 흔한 실수는 `/health` 하나에 모든 판단을 몰아넣는 것입니다.
그러면 체크 목적이 섞여서 운영 해석이 어려워집니다.
실무에서는 아래처럼 분리하는 편이 낫습니다.

- `/live`: 프로세스가 재시작이 필요한 상태인지 확인
- `/ready`: 트래픽 수신 가능 여부 확인
- `/startup`: 초기 부팅 완료 여부 확인

Express 예시는 아래처럼 단순하게 시작할 수 있습니다.

```js
const express = require('express');
const app = express();

let isShuttingDown = false;
let startupComplete = false;
let dependenciesReady = false;

app.get('/live', (req, res) => {
  res.status(200).json({ ok: true });
});

app.get('/ready', (req, res) => {
  if (!startupComplete || !dependenciesReady || isShuttingDown) {
    return res.status(503).json({ ok: false });
  }

  return res.status(200).json({ ok: true });
});

app.get('/startup', (req, res) => {
  if (!startupComplete) {
    return res.status(503).json({ ok: false });
  }

  return res.status(200).json({ ok: true });
});
```

핵심은 엔드포인트를 복잡하게 만드는 것이 아니라, **판단 기준을 분리해서 운영 의도를 코드에 드러내는 것**입니다.

### H3. readiness에는 "필수 의존성"만 넣는다

readiness에 모든 외부 시스템 상태를 다 넣기 시작하면 작은 흔들림에도 pod가 서비스에서 빠졌다 들어오는 flap 현상이 심해집니다.
실무에서는 "이게 없으면 요청을 처리할 수 없는가"를 기준으로 최소한만 넣는 편이 좋습니다.

예를 들면 아래처럼 나눌 수 있습니다.

**넣을 가능성이 높은 것**
- 주 데이터베이스 연결 가능 여부
- 필수 설정 로딩 완료 여부
- graceful shutdown 상태 여부
- 꼭 필요한 메시지 브로커 초기화 여부

**신중해야 하는 것**
- 선택적 캐시 시스템
- 분석용 이벤트 파이프라인
- 일시 실패를 허용하는 외부 API
- 백그라운드 배치 시스템

readiness는 완벽성 체크가 아니라 **트래픽 라우팅 판단**입니다.
너무 많이 넣으면 오히려 가용성이 떨어집니다.
DB 연결 풀 고갈처럼 요청을 실제로 못 받는 상황을 구분하는 기준은 [Node.js Connection Pool Exhaustion 방지 가이드](/development/blog/seo/2026/04/11/nodejs-connection-pool-exhaustion-prevention-guide.html)도 함께 참고할 만합니다.

## liveness에는 무엇을 넣고, 무엇을 빼야 하나

### H3. liveness는 외부 의존성보다 프로세스 자가 건강에 집중한다

liveness는 "재시작하면 좋아질 가능성이 큰가"를 기준으로 설계해야 합니다.
그래서 외부 시스템 상태를 그대로 매달기보다, 프로세스 내부 이상 징후에 집중하는 편이 맞습니다.

Node.js에서는 아래 같은 기준이 비교적 현실적입니다.

- 최근 일정 시간 동안 이벤트 루프 지연이 비정상적으로 높게 지속됨
- 치명적 내부 상태 플래그가 세워짐
- 메인 루프가 요청을 거의 처리하지 못하는 상태가 감지됨

예를 들어 event loop lag를 메트릭으로 수집하고, 일정 기준 이상이 지속될 때만 liveness 실패로 연결하는 방식이 가능합니다.
다만 단발성 spike에 바로 반응하면 안 됩니다.
이 부분은 [Node.js Event Loop Lag 모니터링 가이드](/development/blog/seo/2026/04/13/nodejs-event-loop-lag-monitoring-guide.html)와 함께 보는 것이 좋습니다.

### H3. 단순 timeout 하나로 liveness를 실패시키면 위험하다

헬스체크 요청이 잠깐 느렸다는 이유만으로 liveness를 실패시키면, 트래픽 피크 때 정상 pod도 재시작 루프에 들어갈 수 있습니다.
특히 GC, CPU spike, 일시적 스케줄링 지연은 짧게 지나가는 경우가 많습니다.

그래서 보통은 아래처럼 완충 장치를 둡니다.

- `failureThreshold`를 너무 낮게 잡지 않기
- 짧은 spike보다 연속 실패를 기준으로 보기
- 헬스체크 핸들러 자체는 매우 가볍게 유지하기
- 상세 진단은 별도 메트릭과 로그에서 보기

즉 liveness는 정밀 진단 도구가 아니라, **정말 재시작이 필요한 상태를 선별하는 안전장치**에 가깝습니다.

## Startup Probe를 꼭 써야 하는 경우

### H3. 초기화가 긴 서비스는 startup probe 없으면 억울하게 죽는다

Node.js 앱이더라도 초기화 단계에서 ORM 연결, 설정 검증, 캐시 준비, 워커 등록 등을 하다 보면 시작 시간이 길어질 수 있습니다.
이때 liveness만 두면 아직 정상 부팅 중인데 kubelet이 비정상으로 판단할 수 있습니다.

특히 아래 상황에서는 startup probe가 거의 필수에 가깝습니다.

- cold start 편차가 큰 서비스
- 멀티테넌트 설정을 시작 시 로딩하는 서비스
- 부팅 시 schema sync 또는 상태 확인이 필요한 서비스
- secret manager 또는 외부 설정 서버 의존도가 높은 서비스

startup probe는 부팅 시간을 숨기는 도구가 아니라, **느린 시작을 정상 플로우로 분리하는 장치**입니다.

### H3. startup probe가 끝난 뒤 readiness와 liveness가 이어지게 만든다

운영에서 자주 놓치는 부분은 probe 간의 연결입니다.
startup probe가 성공한 뒤에는 readiness와 liveness가 각자 역할을 이어받아야 합니다.

흐름을 단순화하면 아래와 같습니다.

1. startup probe가 부팅 완료 전까지 보호한다.
2. startup probe 성공 후 readiness가 트래픽 수신 여부를 판단한다.
3. 운영 중 회복 불가능 상태일 때만 liveness가 재시작을 유도한다.

이 순서를 의식하면 probe 설계가 훨씬 덜 꼬입니다.

## 실무에서 자주 나오는 실수

### H3. readiness 실패와 liveness 실패를 같은 의미로 취급한다

이 둘을 같은 의미로 쓰면 장애 처리 전략이 꼬입니다.
readiness 실패는 보통 "지금은 트래픽 받지 마"이고, liveness 실패는 "이 프로세스는 다시 띄우는 게 낫다"에 가깝습니다.

예를 들어 DB가 잠깐 불안정할 때는 readiness를 실패시켜 새 요청 유입을 줄일 수 있습니다.
하지만 이걸 liveness까지 같이 실패시키면 모든 pod가 재시작되면서 오히려 복구가 더 늦어질 수 있습니다.

### H3. graceful shutdown과 readiness를 연결하지 않는다

롤링 배포나 scale down 시 종료 신호를 받은 pod가 여전히 readiness 200을 반환하면, 새 요청이 계속 유입될 수 있습니다.
그러면 inflight request가 깔끔하게 빠지지 않고 에러가 늘어납니다.

보통은 `SIGTERM`을 받으면 아래 순서로 처리합니다.

```js
process.on('SIGTERM', async () => {
  isShuttingDown = true;

  server.close(async () => {
    await closeDatabase();
    process.exit(0);
  });
});
```

이렇게 하면 종료 시작과 동시에 readiness를 실패로 돌려 새 트래픽 유입을 멈출 수 있습니다.
이 패턴은 [Node.js Graceful Shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)와도 직접 이어집니다.

## 쿠버네티스 설정 예시

### H3. probe 주기와 임계치는 애플리케이션 성격에 맞춰 조정한다

아래는 시작점으로 쓸 만한 예시입니다.

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 3000
  periodSeconds: 5
  failureThreshold: 24

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  periodSeconds: 5
  timeoutSeconds: 1
  failureThreshold: 2
  successThreshold: 1

livenessProbe:
  httpGet:
    path: /live
    port: 3000
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 3
```

이 값은 정답이 아닙니다.
중요한 것은 아래 기준입니다.

- startup probe는 가장 느린 정상 부팅 시간보다 여유 있게 잡기
- readiness는 비교적 민감하게, liveness는 보수적으로 잡기
- timeout을 너무 짧게 잡아 정상 지연까지 장애로 만들지 않기
- pod 종료 시간과 배포 전략까지 함께 고려하기

## 운영 체크리스트

### H3. 발행 전에 점검하듯 운영 기준도 체크리스트로 고정한다

Node.js 서비스의 probe 설계를 점검할 때는 아래 항목을 확인하면 좋습니다.

- readiness와 liveness의 목적이 코드와 문서에서 분리돼 있는가
- readiness가 필수 의존성만 확인하도록 제한돼 있는가
- startup probe가 긴 부팅 시간을 정상으로 보호하는가
- SIGTERM 수신 시 readiness가 즉시 실패로 전환되는가
- liveness가 외부 의존성 일시 장애에 과민 반응하지 않는가
- 헬스체크 실패 원인을 메트릭과 로그에서 함께 추적할 수 있는가

probe는 작아 보여도 운영 안정성에 미치는 영향이 큽니다.
헬스체크를 "있으니까 넣는 설정"으로 다루면 장애를 더 만들고, 역할을 나눠 설계하면 장애 전파를 줄이는 안전장치가 됩니다.

## 마무리

Node.js에서 readiness probe, liveness probe, startup probe를 잘 나누는 핵심은 단순합니다.
**트래픽 수신 판단, 재시작 판단, 부팅 보호를 서로 다른 문제로 취급하는 것**입니다.

readiness는 트래픽을 받아도 되는지, liveness는 재시작이 필요한지, startup probe는 아직 부팅 중인지 봅니다.
이 경계를 명확히 해두면 불필요한 재시작을 줄이고, 롤링 배포와 장애 대응도 훨씬 안정적으로 가져갈 수 있습니다.

헬스체크가 자꾸 운영 이슈로 번졌다면, 엔드포인트를 더 늘리기보다 먼저 각 probe가 어떤 결정을 내리는지부터 다시 정의해보는 편이 좋습니다.

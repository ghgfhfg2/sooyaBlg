---
layout: post
title: "Graceful Shutdown으로 Kubernetes 무중단 배포 안정성 높이는 방법 (SIGTERM, preStop 실전 가이드)"
date: 2026-03-11 08:00:00 +0900
lang: ko
translation_key: graceful-shutdown-kubernetes-zero-downtime-guide
permalink: /development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html
alternates:
  ko: /development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html
  x_default: /development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, kubernetes, graceful-shutdown, sigterm, 무중단배포]
description: "Kubernetes 배포 중 연결 끊김과 502/503 오류를 줄이려면 SIGTERM 처리, preStop 훅, terminationGracePeriodSeconds를 함께 설계해야 합니다. Graceful Shutdown의 핵심 원칙과 재현 가능한 설정 예시를 정리합니다."
---

## 문제: 배포는 정상인데 사용자 요청이 중간에 끊기는 이유

Kubernetes 롤링 배포에서 자주 생기는 장애가 있습니다.
파드 교체는 완료됐지만 배포 직후 502/503, 소켓 끊김, 결제 요청 중복 같은 문제가 발생하는 패턴입니다.

대부분 원인은 단순합니다.
**종료 신호(SIGTERM)를 받은 애플리케이션이 현재 처리 중인 요청을 정리하기 전에 강제 종료**되기 때문입니다.

즉, 무중단 배포는 "새 파드를 빨리 띄우는 문제"가 아니라,
"기존 파드를 안전하게 내려주는 문제"까지 포함해야 완성됩니다.

## 원칙: 트래픽 차단 → 처리 중 요청 정리 → 프로세스 종료 순서 지키기

### 왜 SIGTERM 핸들링이 핵심인가

Kubernetes는 파드를 종료할 때 먼저 SIGTERM을 보냅니다.
이 시점에 앱이 해야 할 일은 아래 3가지입니다.

1. 새 요청 수신 중단(ready=false 전환)
2. 진행 중인 요청/작업 정상 마무리
3. 연결/리소스 정리 후 프로세스 종료

이 순서가 깨지면 사용자 입장에서는 "가끔 실패하는 서비스"가 됩니다.

### preStop과 terminationGracePeriodSeconds 역할 분리

- `preStop`: 종료 직전 짧은 유예를 주어 로드밸런서 라우팅에서 빠질 시간을 확보
- `terminationGracePeriodSeconds`: 앱이 실제 정리 작업을 끝낼 수 있는 최대 시간

두 값을 같이 조정해야 효과가 납니다.
`preStop`만 길고 앱 SIGTERM 처리가 없으면 여전히 중간 끊김이 발생합니다.

## 실전 설정 예시: 최소 재현 가능한 무중단 종료 구성

### Kubernetes Deployment 예시

```yaml
spec:
  terminationGracePeriodSeconds: 40
  containers:
    - name: api
      image: your-api:latest
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 10"]
```

운영 시작점으로는 아래처럼 잡는 경우가 많습니다.

- `preStop: 5~15초`
- `terminationGracePeriodSeconds: 30~60초`

정답은 고정값이 아니라 **요청 최대 처리 시간(P95/P99)** 기준으로 맞춰야 합니다.

### Node.js SIGTERM 처리 예시

```js
const server = app.listen(8080);
let isShuttingDown = false;

app.get('/health/ready', (_req, res) => {
  if (isShuttingDown) return res.status(503).json({ status: 'shutting_down' });
  return res.status(200).json({ status: 'ready' });
});

process.on('SIGTERM', () => {
  isShuttingDown = true;

  server.close(() => {
    // DB pool, queue consumer 등 정리
    process.exit(0);
  });

  setTimeout(() => process.exit(1), 35000); // 안전 타임아웃
});
```

핵심은 SIGTERM 수신 즉시 readiness를 내려 새 요청 유입을 막고,
`server.close()`로 기존 연결 정리를 기다리는 것입니다.

## 운영 체크리스트: 배포 전후에 꼭 확인할 지표

### 배포 직후 필수 모니터링

- 5xx 비율(특히 롤링 교체 구간)
- in-flight request 처리 시간 분포
- 파드 종료 사유(Graceful vs SIGKILL)

### 흔한 안티패턴

- SIGTERM 핸들러 없이 프로세스를 즉시 종료
- `terminationGracePeriodSeconds`를 5~10초로 과도하게 짧게 설정
- readiness는 정상인데 종료 중에도 요청을 계속 받는 구조

## 요약

Graceful Shutdown은 선택 옵션이 아니라,
Kubernetes 무중단 배포에서 사용자 오류를 줄이는 핵심 안전장치입니다.

- SIGTERM 수신 즉시 새 요청 유입 차단
- 진행 중 요청을 정리할 충분한 grace period 확보
- preStop + readiness + 앱 종료 로직을 하나의 흐름으로 설계

이 3가지만 지켜도 배포 품질, 장애 재현성, 문서 신뢰도를 함께 끌어올릴 수 있습니다.

## 내부 링크

- [/development/blog/seo/2026/03/10/readiness-liveness-probe-deploy-reliability-guide.html](/development/blog/seo/2026/03/10/readiness-liveness-probe-deploy-reliability-guide.html)
- [/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)
- [/development/blog/seo/2026/03/08/idempotency-key-api-retry-consistency-guide.html](/development/blog/seo/2026/03/08/idempotency-key-api-retry-consistency-guide.html)

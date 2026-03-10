---
layout: post
title: "Readiness/Liveness Probe를 분리하면 배포 장애를 줄이는 이유 (Kubernetes 운영 가이드)"
date: 2026-03-10 20:00:00 +0900
lang: ko
translation_key: readiness-liveness-probe-deploy-reliability-guide
permalink: /development/blog/seo/2026/03/10/readiness-liveness-probe-deploy-reliability-guide.html
alternates:
  ko: /development/blog/seo/2026/03/10/readiness-liveness-probe-deploy-reliability-guide.html
  x_default: /development/blog/seo/2026/03/10/readiness-liveness-probe-deploy-reliability-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, kubernetes, readiness, liveness, 배포안정성]
description: "Kubernetes에서 Readiness Probe와 Liveness Probe를 분리해 설정하면 롤링 배포 중 트래픽 오류를 줄이고, 장애 복구 판단 기준을 명확히 해 운영 신뢰도를 높일 수 있습니다."
---

## 문제: 배포는 성공했는데 사용자 오류가 늘어나는 이유

Kubernetes 기반 서비스에서 자주 보는 장애 패턴이 있습니다.
파드가 `Running` 상태로 보이는데도 배포 직후 5xx가 급증하는 상황입니다.

원인은 대부분 **트래픽 수신 가능 상태(readiness)** 와
**프로세스 생존 상태(liveness)** 를 같은 기준으로 처리했기 때문입니다.

- 앱 프로세스는 살아 있지만 DB 마이그레이션이 끝나지 않았을 수 있습니다.
- 캐시 워밍업이 안 끝났는데 서비스 엔드포인트에 먼저 포함될 수 있습니다.
- 일시적 의존성 장애를 liveness 실패로 오판해 재시작 루프가 생길 수 있습니다.

결국 "앱이 살아있다"와 "지금 요청을 받아도 된다"를 분리하지 않으면,
배포 품질과 문서 신뢰도가 함께 떨어집니다.

## 원칙: Readiness와 Liveness를 역할별로 분리하기

### Readiness Probe가 담당해야 할 것

Readiness는 **트래픽을 받아도 안전한지**를 판단합니다.
즉, 실패 시 파드를 죽이는 게 아니라 서비스 라우팅 대상에서 제외해야 합니다.

핵심 기준:

- 필수 의존성(예: DB, 메시지 브로커) 연결 가능 여부
- 애플리케이션 초기화 완료 여부
- 실제 요청 처리에 필요한 최소 준비 상태

### Liveness Probe가 담당해야 할 것

Liveness는 **프로세스가 회복 불가능한 상태인지**를 판단합니다.
실패가 누적되면 재시작으로 복구를 시도합니다.

핵심 기준:

- 이벤트 루프 정지, 데드락, 내부 워커 정지 같은 치명 상태
- 짧은 외부 장애(일시적 DB 끊김)에는 과민하게 반응하지 않도록 설계

### 흔한 안티패턴

- `/health` 하나로 readiness/liveness를 동시에 처리
- 외부 의존성 장애를 liveness 실패로 바로 연결
- 초기 구동 시간이 긴데 `initialDelaySeconds`를 너무 짧게 설정

## 예시: 최소 재현 가능한 Probe 설정

아래는 운영에서 자주 쓰는 안전한 기본 형태입니다.

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

### 엔드포인트 구현 시 체크리스트

1. `/health/ready`는 의존성 준비 여부를 반영한다.
2. `/health/live`는 프로세스 자체 이상 여부만 판단한다.
3. 오류 응답에는 내부 비밀값(DSN, 토큰, 계정 정보)을 포함하지 않는다.

Node.js 의사코드 예시:

```js
app.get('/health/live', (_req, res) => {
  res.status(200).json({ status: 'ok' });
});

app.get('/health/ready', async (_req, res) => {
  const dbOk = await pingDb();
  if (!dbOk) return res.status(503).json({ status: 'not_ready' });
  return res.status(200).json({ status: 'ready' });
});
```

## 운영 팁: 배포 파이프라인에서 함께 고정할 것

### HPA/오토스케일과 Probe 간섭 줄이기

CPU 스파이크 구간에서 readiness 실패가 잦다면,
애플리케이션 처리량보다 probe 타임아웃이 과도하게 짧은지 먼저 확인합니다.

### 관측 지표 연결

아래 지표를 같이 보면 원인 파악이 빨라집니다.

- readiness 실패 횟수
- liveness 재시작 횟수
- 롤링 배포 구간의 5xx 비율

이 3가지를 릴리스 노트에 함께 남기면
"배포 성공"을 단순 완료가 아닌 **사용자 영향 기준**으로 평가할 수 있습니다.

## 요약

Readiness/Liveness Probe 분리는 Kubernetes 운영의 기본이지만,
실제로는 장애 예방 효과가 큰 품질 레버입니다.

- readiness: 트래픽 수신 가능 여부
- liveness: 프로세스 복구 필요 여부

두 기준을 분리하면 롤링 배포 중 오류를 줄이고,
검색 유입 독자도 재현 가능한 방식으로 운영 기준을 적용할 수 있습니다.
이런 문서 구조가 기술 블로그의 SEO 신뢰도를 안정적으로 높입니다.

## 내부 링크

- [/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)
- [/development/blog/seo/2026/03/08/idempotency-key-api-retry-consistency-guide.html](/development/blog/seo/2026/03/08/idempotency-key-api-retry-consistency-guide.html)
- [/development/blog/seo/2026/03/08/env-example-and-required-variables-guide.html](/development/blog/seo/2026/03/08/env-example-and-required-variables-guide.html)

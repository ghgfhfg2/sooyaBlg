---
layout: post
title: "SLI/SLO 에러 버짓 실무 가이드: Node.js 온콜 운영을 숫자로 안정화하는 방법"
date: 2026-03-13 20:00:00 +0900
lang: ko
translation_key: sli-slo-error-budget-nodejs-oncall-reliability-guide
permalink: /development/blog/seo/2026/03/13/sli-slo-error-budget-nodejs-oncall-reliability-guide.html
alternates:
  ko: /development/blog/seo/2026/03/13/sli-slo-error-budget-nodejs-oncall-reliability-guide.html
  x_default: /development/blog/seo/2026/03/13/sli-slo-error-budget-nodejs-oncall-reliability-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, sli, slo, error-budget, oncall]
description: "Node.js 서비스에서 SLI/SLO와 에러 버짓을 설계해 온콜 피로를 줄이고 배포 의사결정을 일관되게 만드는 실무 방법을 정리합니다. 지표 정의, 알림 임계치, 운영 체크리스트를 예시와 함께 설명합니다."
---

장애 대응이 힘든 이유는 종종 “기준의 부재”입니다.
Node.js 서비스를 운영할 때 SLI/SLO와 에러 버짓을 먼저 정의해 두면,
배포를 멈출지 계속할지, 알림을 울릴지 말지 같은 판단을 감정이 아니라 **숫자**로 할 수 있습니다.

## SLI/SLO/에러 버짓을 왜 같이 봐야 할까

### SLI는 사용자 경험을 측정하는 지표다

SLI(Service Level Indicator)는 “사용자가 실제로 느끼는 품질”을 수치화한 지표입니다.
예를 들어 API 성공률, p95 지연시간, 결제 성공률이 SLI가 됩니다.

### SLO는 팀의 약속이다

SLO(Service Level Objective)는 “얼마나 잘해야 하는가”에 대한 목표치입니다.
예: `월간 API 성공률 99.9% 이상`.

### 에러 버짓은 변화 속도의 안전장치다

SLO를 99.9%로 잡으면 월간 허용 실패율은 0.1%입니다.
이 허용 범위가 에러 버짓이며, 버짓을 과도하게 소모하면 신규 배포보다 안정화 작업을 우선해야 합니다.

## Node.js 서비스에서 지표를 정의하는 순서

### 1) 유저 저니 기준으로 핵심 경로를 먼저 고른다

지표는 시스템 내부가 아니라 유저 행동 기준으로 고르는 게 좋습니다.
예시:

- 로그인 API 성공률
- 결제 승인 API 성공률
- 추천 API p95 응답시간

### 2) 측정 구간과 분모/분자를 명확히 적는다

애매한 지표는 논쟁만 만들고 행동을 바꾸지 못합니다.
예를 들어 성공률 SLI는 아래처럼 정의를 문서화합니다.

- 분모: 전체 요청 수(4xx 중 클라이언트 실수 제외 여부 명시)
- 분자: 정상 응답(2xx/3xx)
- 집계 단위: 1분/5분/1시간

### 3) 온콜 알림 임계치는 SLO보다 더 보수적으로 둔다

SLO 위반이 발생한 뒤 알림이 울리면 늦습니다.
보통은 아래처럼 2단계 임계치를 둡니다.

- 경고(Warning): 버짓 소모 속도가 평시 대비 급증
- 심각(Critical): 예측 소진 시간이 24시간 이내

## 구현 예시: Express + Prometheus 스타일 메트릭

### 요청 성공/실패를 라우트 단위로 기록

```js
import client from 'prom-client';

const register = new client.Registry();
client.collectDefaultMetrics({ register });

const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['route', 'method', 'status_class'],
  registers: [register],
});

export function metricsMiddleware(req, res, next) {
  res.on('finish', () => {
    const statusClass = `${Math.floor(res.statusCode / 100)}xx`;
    const route = req.route?.path || req.path || 'unknown';
    httpRequestsTotal.inc({ route, method: req.method, status_class: statusClass });
  });
  next();
}
```

핵심은 “어디서 실패가 늘었는지”를 라우트 기준으로 빠르게 찾는 것입니다.

### 버짓 소진 속도를 대시보드에 함께 보여준다

성공률만 보면 안정적으로 보이는데, 짧은 시간에 급격히 악화되는 패턴을 놓치기 쉽습니다.
그래서 최근 1시간/6시간 버짓 소모율을 같이 시각화하면 온콜 판단이 빨라집니다.

## 운영 의사결정 룰(실무 템플릿)

### 배포 계속/중단 기준

- 버짓 잔여 70% 이상: 계획된 배포 진행
- 버짓 잔여 30~70%: 저위험 변경만 배포
- 버짓 잔여 30% 미만: 기능 배포 중단, 안정화 우선

### 온콜 핸드오프 기준

핸드오프 문서에 아래 3가지는 반드시 남깁니다.

- 현재 버짓 잔여율
- 지난 2시간 주요 에러 원인 Top 3
- 이미 적용한 완화 조치와 다음 액션

## 자주 하는 실수와 보완 포인트

### 내부 지표만 보고 사용자 지표를 놓치는 경우

CPU/메모리만 봐서는 결제 실패 증가를 설명하지 못할 때가 많습니다.
항상 사용자 경로 SLI를 우선 지표로 유지하세요.

### SLO는 높은데 액션 플랜이 없는 경우

`99.99%` 같은 숫자만 높이면 팀이 지칩니다.
SLO 수치와 함께 “위반 시 무슨 행동을 할지”를 룰로 묶어야 운영이 안정됩니다.

## 요약

SLI/SLO는 보고서용 지표가 아니라 **배포와 온콜 의사결정을 표준화하는 도구**입니다.
Node.js 서비스에서는 라우트 단위 계측, 버짓 소모율 모니터링, 배포 중단 기준까지 한 세트로 운영할 때 효과가 큽니다.

## 내부 링크

- [/development/blog/seo/2026/03/13/feature-flag-rollout-strategy-nodejs-safe-release-guide.html](/development/blog/seo/2026/03/13/feature-flag-rollout-strategy-nodejs-safe-release-guide.html)
- [/development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html](/development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html)
- [/development/blog/seo/2026/03/12/rate-limiting-nodejs-express-redis-abuse-protection-guide.html](/development/blog/seo/2026/03/12/rate-limiting-nodejs-express-redis-abuse-protection-guide.html)

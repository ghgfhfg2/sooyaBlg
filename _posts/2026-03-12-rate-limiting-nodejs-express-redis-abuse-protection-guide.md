---
layout: post
title: "Node.js Rate Limiting 실전 가이드: Express + Redis로 트래픽 급증과 봇 남용 막기"
date: 2026-03-12 20:00:00 +0900
lang: ko
translation_key: rate-limiting-nodejs-express-redis-abuse-protection-guide
permalink: /development/blog/seo/2026/03/12/rate-limiting-nodejs-express-redis-abuse-protection-guide.html
alternates:
  ko: /development/blog/seo/2026/03/12/rate-limiting-nodejs-express-redis-abuse-protection-guide.html
  x_default: /development/blog/seo/2026/03/12/rate-limiting-nodejs-express-redis-abuse-protection-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, express, redis, rate-limiting, security]
description: "Node.js 서비스에서 트래픽 급증·봇 남용을 완화하는 Rate Limiting 설계를 정리합니다. Express + Redis 기준으로 고정 윈도우와 슬라이딩 윈도우, 429 응답 설계, 프록시 환경 주의사항까지 재현 가능한 예시로 설명합니다."
---

## 문제: 왜 지금도 Rate Limiting이 자주 실패할까

대부분의 서비스는 Rate Limiting을 "붙이긴" 합니다.
그런데 실제 장애 상황에서는 다음 이유로 금방 무력화됩니다.

- 인메모리 카운터만 사용해서 멀티 인스턴스에서 제한이 일관되지 않음
- 프록시/CDN 뒤에서 클라이언트 IP 추출이 잘못되어 정상 사용자가 차단됨
- 429 응답은 보내지만 `Retry-After` 같은 복구 힌트를 주지 않음

핵심은 단순 차단이 아니라 **공정하게 제한하고, 정상 사용자가 복구 가능한 UX를 주는 것**입니다.

## 설계 원칙: “정책 분리 + 공유 저장소 + 관측성”

### 정책은 엔드포인트별로 분리한다

로그인/회원가입/비밀번호 재설정은 공격 표면이 다릅니다.
같은 제한값을 전 구간에 적용하면 오탐이 늘고 방어력은 떨어집니다.

예시 정책:

- `/auth/login`: IP 기준 분당 10회
- `/auth/password-reset`: 계정 + IP 기준 시간당 5회
- `/api/search`: 사용자 토큰 기준 초당 20회(버스트 허용)

### 카운터는 Redis 같은 공유 저장소를 쓴다

오토스케일 환경에서는 인메모리 제한이 사실상 무용지물입니다.
인스턴스가 늘어날수록 공격자가 더 쉽게 우회합니다.

### 관측 지표를 반드시 남긴다

아래 지표를 대시보드에 분리해 두면 운영 판단이 빨라집니다.

- 429 응답률(엔드포인트별)
- 상위 차단 키(IP/user/apiKey)
- 차단 후 5분 내 정상 재시도 성공률

## Express + Redis 구현 예시

### 1) 프록시 신뢰 설정 먼저 고정

```js
import express from 'express';

const app = express();
app.set('trust proxy', 1); // Nginx/ALB 뒤라면 환경에 맞게 조정
```

프록시 홉 수를 잘못 설정하면 실제 IP 대신 내부 IP를 읽어 차단 정확도가 무너집니다.

### 2) 로그인 엔드포인트 고정 윈도우 제한

```js
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

const loginLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 10,
  standardHeaders: true,
  legacyHeaders: false,
  message: {
    code: 'RATE_LIMITED',
    message: '요청이 너무 많습니다. 잠시 후 다시 시도해 주세요.'
  },
  store: new RedisStore({
    sendCommand: (...args) => redis.sendCommand(args)
  })
});

app.post('/auth/login', loginLimiter, async (req, res) => {
  // 로그인 로직
  res.status(200).json({ ok: true });
});
```

### 3) 429 응답에 복구 힌트 넣기

```js
app.use((err, req, res, next) => {
  if (err?.status !== 429) return next(err);

  const retryAfter = Number(res.getHeader('Retry-After') || 60);
  return res.status(429).json({
    code: 'TOO_MANY_REQUESTS',
    retry_after_seconds: retryAfter,
    message: '요청 한도를 초과했습니다. 잠시 후 재시도해 주세요.'
  });
});
```

사용자는 기다려야 할 시간을 알 수 있고, 클라이언트는 백오프 로직을 안정적으로 구현할 수 있습니다.

## 알고리즘 선택: 고정 윈도우 vs 슬라이딩 윈도우

### 고정 윈도우(Fixed Window)

- 장점: 구현 단순, 비용 낮음
- 단점: 경계 시점에서 버스트 허용량이 커질 수 있음

관리자 API처럼 예측 가능한 트래픽에는 충분히 실용적입니다.

### 슬라이딩 윈도우(Sliding Window)

- 장점: 공정한 제한, 경계 버스트 완화
- 단점: 구현 복잡도와 저장 비용 증가

로그인/인증처럼 공격 표면이 큰 구간에는 슬라이딩 윈도우가 더 안전합니다.

## 운영 체크리스트

### 릴리스 전

- `trust proxy`가 실제 네트워크 토폴로지와 일치하는가
- Redis 장애 시 fail-open/fail-closed 정책을 정했는가
- 429 응답 본문과 `Retry-After`를 API 문서에 명시했는가

### 릴리스 후

- 정상 사용자 차단률이 급증하지 않는가
- 특정 ASN/국가/UA에서 차단이 집중되는가
- 차단 정책 변경 후 인증 성공률이 회복되는가

## 요약

Rate Limiting은 보안 기능이면서 동시에 사용자 경험 기능입니다.
정확한 키 추출, 공유 저장소, 복구 가능한 429 응답을 함께 설계해야 실제 운영에서 효과가 납니다.

처음에는 단순한 고정 윈도우로 시작하되,
인증·결제 같은 민감 구간은 슬라이딩 윈도우와 더 강한 정책으로 분리해 운영하세요.

## 내부 링크

- [/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)
- [/development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html](/development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html)
- [/development/blog/seo/2026/03/10/readiness-liveness-probe-deploy-reliability-guide.html](/development/blog/seo/2026/03/10/readiness-liveness-probe-deploy-reliability-guide.html)

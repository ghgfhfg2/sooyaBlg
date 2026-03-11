---
layout: post
title: "Circuit Breaker 패턴으로 Node.js 외부 API 장애 전파 막는 방법 (opossum 실전 가이드)"
date: 2026-03-12 08:00:00 +0900
lang: ko
translation_key: circuit-breaker-pattern-nodejs-external-api-stability-guide
permalink: /development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html
alternates:
  ko: /development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html
  x_default: /development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, circuit-breaker, resilience, api]
description: "외부 API가 느려지거나 실패할 때 Node.js 서비스 전체가 함께 무너지는 문제를 Circuit Breaker로 막을 수 있습니다. 실패 임계치, 타임아웃, half-open 복구 전략과 opossum 예시를 통해 재현 가능한 안정화 방법을 정리합니다."
---

## 문제: 외부 API 한 곳이 느려지면 왜 전체 응답성이 무너질까

결제, 메시지, 지도, AI 추론처럼 외부 API 의존이 큰 서비스는 작은 장애가 빠르게 전파됩니다.
특히 타임아웃 없이 재시도만 늘리면 요청 대기열이 쌓이고, 결국 애플리케이션 전체 응답성이 떨어집니다.

핵심 문제는 간단합니다.
**실패를 빠르게 감지하고, 잠깐 차단하고, 회복 가능성을 확인하는 제어 장치가 없기 때문**입니다.

## 원칙: Timeout + Circuit Breaker + Fallback을 한 세트로 설계하기

### Circuit Breaker가 필요한 이유

Circuit Breaker는 연속 실패가 임계치를 넘으면 회로를 `open` 상태로 바꿔
일정 시간 동안 외부 호출을 즉시 차단합니다.

이렇게 하면 느린 의존성 때문에 워커/스레드/커넥션 풀이 잠식되는 상황을 막을 수 있습니다.

### 상태 전이 이해하기 (closed → open → half-open)

- `closed`: 정상 호출
- `open`: 실패율이 높아 즉시 차단
- `half-open`: 일부 트래픽만 허용해 복구 여부 확인

운영에서 중요한 포인트는, `half-open`에서 성공/실패 판단 기준을 명확히 두는 것입니다.

## Node.js 실전 예시: opossum으로 외부 API 보호하기

### 1) 보호 대상 함수 정의

```js
import CircuitBreaker from 'opossum';

async function fetchPartnerProfile(userId) {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 1200);

  try {
    const res = await fetch(`https://partner.example.com/v1/users/${userId}`, {
      signal: controller.signal,
      headers: { 'x-request-id': crypto.randomUUID() }
    });

    if (!res.ok) throw new Error(`partner_api_${res.status}`);
    return await res.json();
  } finally {
    clearTimeout(timeout);
  }
}
```

### 2) Circuit Breaker 설정

```js
const breaker = new CircuitBreaker(fetchPartnerProfile, {
  timeout: 1500, // 호출 1.5초 초과 시 실패 처리
  errorThresholdPercentage: 50, // 실패율 50% 이상이면 open
  volumeThreshold: 20, // 최소 20건 관측 후 판단
  resetTimeout: 10000 // 10초 뒤 half-open 시도
});

breaker.fallback((_userId) => ({
  source: 'fallback',
  profile: null,
  message: '파트너 응답 지연으로 기본 정보를 반환합니다.'
}));
```

### 3) API 핸들러에서 사용

```js
app.get('/profiles/:id', async (req, res) => {
  try {
    const data = await breaker.fire(req.params.id);
    return res.status(200).json(data);
  } catch (err) {
    return res.status(503).json({
      code: 'PARTNER_TEMP_UNAVAILABLE',
      detail: err.message
    });
  }
});
```

## 튜닝 체크리스트: 숫자를 어떻게 잡아야 하나

### timeout은 P95보다 짧게, resetTimeout은 복구 시간을 반영

- `timeout`: 파트너 API P95 응답 시간보다 약간 짧게
- `volumeThreshold`: 트래픽이 적은 서비스는 과도하게 높이지 않기
- `resetTimeout`: 장애 평균 복구 시간(MTTR) 기준으로 시작

### 메트릭 없이 운영하면 실패한다

반드시 아래 지표를 함께 수집하세요.

- breaker 상태 전이 횟수(open/half-open/closed)
- fallback 응답 비율
- 외부 API 에러 코드 분포(5xx, timeout, DNS)

## 흔한 실수와 예방 팁

### 실수 1) 재시도와 Circuit Breaker를 무제한으로 함께 쓰기

재시도는 총 예산(횟수/시간)을 제한해야 합니다.
무제한 재시도는 breaker 효과를 상쇄합니다.

### 실수 2) fallback을 "성공"으로만 기록하기

fallback은 사용자 보호 전략이지 정상 처리 자체가 아닙니다.
모니터링에서 분리 집계해야 실제 장애 강도를 볼 수 있습니다.

## 요약

외부 API 장애를 100% 막을 수는 없지만,
장애가 우리 시스템 전체로 번지는 속도와 범위는 제어할 수 있습니다.

- 호출 타임아웃을 먼저 설정하고
- Circuit Breaker로 연쇄 대기를 차단하고
- fallback + 관측 지표로 복구 전략을 완성하세요.

이 구조를 적용하면 장애 시에도 "느리게 무너지는 서비스"가 아니라
"부분 저하로 버티는 서비스"에 가까워집니다.

## 내부 링크

- [/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)
- [/development/blog/seo/2026/03/11/nodejs-memory-leak-heapdump-clinicjs-guide.html](/development/blog/seo/2026/03/11/nodejs-memory-leak-heapdump-clinicjs-guide.html)
- [/development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html](/development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html)

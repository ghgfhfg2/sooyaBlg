---
layout: post
title: "Node.js AsyncLocalStorage 실전 가이드: 요청 컨텍스트·로그 추적으로 디버깅 시간 줄이기"
date: 2026-03-19 20:00:00 +0900
lang: ko
translation_key: nodejs-asynclocalstorage-request-context-logging-guide
permalink: /development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html
alternates:
  ko: /development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html
  x_default: /development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, asynclocalstorage, logging, observability, debugging, backend]
description: "Node.js AsyncLocalStorage로 요청 ID, 사용자 ID, trace 정보를 비동기 흐름 전체에 전파해 로그 검색성과 장애 분석 속도를 높이는 방법을 실무 예시와 함께 정리했습니다."
---

장애가 났을 때 가장 답답한 순간은 에러 자체보다 **로그가 서로 이어지지 않을 때**입니다.
같은 요청에서 나온 로그인데도 요청 ID가 중간에 비거나, 서비스 계층을 몇 번 타고 내려가면 누가 남긴 로그인지 흐려지는 경우가 많습니다.
이 글에서는 Node.js의 `AsyncLocalStorage`를 이용해 요청 컨텍스트를 안정적으로 전파하고, 로그·메트릭·추적 데이터를 더 읽기 쉽게 만드는 방법을 정리합니다.

## 왜 AsyncLocalStorage가 필요한가

### H3. 파라미터로 requestId를 계속 넘기는 방식은 금방 무너진다

작은 프로젝트에서는 `requestId`나 `userId`를 함수 인자로 계속 전달해도 버틸 수 있습니다.
하지만 API 핸들러, 서비스 레이어, 저장소 레이어, 외부 API 클라이언트가 늘어나면 금세 문제가 생깁니다.

- 함수 시그니처가 컨텍스트 전달용 인자로 계속 비대해진다
- 중간 함수 하나라도 빠뜨리면 로그 연결이 끊긴다
- 팀원이 새 코드를 추가할 때 규칙이 쉽게 누락된다

결국 "컨텍스트를 어디서나 읽을 수 있는 공통 저장소"가 필요해집니다.

### H3. 비동기 경계가 많을수록 운영 가치는 더 커진다

Node.js 서비스는 `await`, 이벤트 리스너, 큐 소비자, DB 호출, 외부 API 호출처럼 비동기 경계가 매우 많습니다.
이 구조에서 요청별 문맥이 유지되지 않으면 아래 문제가 반복됩니다.

- 장애 로그를 요청 단위로 묶어 보기 어렵다
- 특정 사용자 요청의 실행 흐름을 빠르게 따라가기 힘들다
- 에러 재현 전까지 원인 추적 시간이 길어진다

`AsyncLocalStorage`는 이런 비동기 흐름 속에서도 요청 단위 문맥을 유지하도록 도와줍니다.

## 기본 구현 패턴

### H3. 요청 시작 시 컨텍스트를 초기화한다

가장 먼저 할 일은 HTTP 요청 진입점에서 공통 컨텍스트를 만드는 것입니다.
아래 예시는 Express 스타일이지만 Fastify나 NestJS에도 같은 원리로 적용할 수 있습니다.

```ts
import { AsyncLocalStorage } from 'node:async_hooks';
import crypto from 'node:crypto';

type RequestContext = {
  requestId: string;
  userId?: string;
  path: string;
  startedAt: number;
};

export const requestContext = new AsyncLocalStorage<RequestContext>();

export function contextMiddleware(req, res, next) {
  const store: RequestContext = {
    requestId: req.headers['x-request-id']?.toString() ?? crypto.randomUUID(),
    userId: req.headers['x-user-id']?.toString(),
    path: req.path,
    startedAt: Date.now(),
  };

  requestContext.run(store, () => next());
}
```

핵심은 요청이 시작되는 지점에서 `run()`으로 스토어를 열고,
그 이후 비동기 작업 전체가 같은 문맥을 공유하도록 만드는 것입니다.

### H3. 로거는 컨텍스트를 직접 읽도록 만든다

컨텍스트를 넣어두고도 매번 수동으로 꺼내 조합하면 다시 누락이 생깁니다.
가장 실용적인 방법은 **로거가 스스로 컨텍스트를 읽게 만드는 것**입니다.

```ts
import pino from 'pino';
import { requestContext } from './request-context';

const logger = pino();

export function logInfo(message: string, extra: Record<string, unknown> = {}) {
  const ctx = requestContext.getStore();

  logger.info({
    requestId: ctx?.requestId,
    userId: ctx?.userId,
    path: ctx?.path,
    ...extra,
  }, message);
}
```

이렇게 하면 서비스 함수는 `requestId`를 몰라도 되고,
로그를 남길 때도 호출 코드가 훨씬 단순해집니다.

## 실무에서 많이 쓰는 확장 포인트

### H3. 외부 API 호출과 trace 정보를 함께 기록한다

장애 분석에서 자주 필요한 정보는 "어느 외부 호출이 느렸는가"입니다.
컨텍스트에 `requestId`만 넣지 말고 trace/span 식별자나 상위 작업 이름도 같이 넣어두면 좋습니다.

예를 들어 아래처럼 기록할 수 있습니다.

```ts
export async function fetchBillingProfile(accountId: string) {
  const ctx = requestContext.getStore();
  const startedAt = Date.now();

  const res = await fetch(`https://billing.example.com/accounts/${accountId}`);

  logInfo('billing profile fetched', {
    upstream: 'billing-api',
    status: res.status,
    durationMs: Date.now() - startedAt,
    requestId: ctx?.requestId,
  });

  return res.json();
}
```

이 패턴이 쌓이면 "느린 요청"뿐 아니라 "특정 업스트림 때문에 지연된 요청"까지 빠르게 좁혀갈 수 있습니다.

### H3. 큐 워커와 배치에도 같은 규약을 재사용한다

`AsyncLocalStorage`는 HTTP 서버 전용 도구가 아닙니다.
BullMQ 같은 워커나 배치 작업에서도 잡 단위 컨텍스트를 만들면 로그 품질이 눈에 띄게 올라갑니다.

- `jobId`
- `queueName`
- `attempt`
- `triggeredBy`

이런 필드를 컨텍스트에 넣어두면 재시도 로그와 실패 로그를 서로 쉽게 연결할 수 있습니다.
특히 소비자 중복 처리나 DLQ 분석처럼 흐름을 따라가야 하는 작업에서 효과가 큽니다.

## 도입할 때 주의할 점

### H3. 컨텍스트를 "전역 상태"처럼 오용하면 안 된다

`AsyncLocalStorage`는 요청 문맥 저장소이지, 아무 데이터나 넣는 캐시가 아닙니다.
아래 같은 사용은 피하는 편이 좋습니다.

- 큰 객체 전체를 저장하는 것
- 비밀번호, 토큰, 주민번호 같은 민감정보를 저장하는 것
- 비즈니스 로직의 진짜 입력값을 컨텍스트에 숨기는 것

컨텍스트에는 **로그·관측에 필요한 최소 식별 정보**만 넣는 게 안전합니다.

### H3. 라이브러리 경계에서 문맥 유지 여부를 검증해야 한다

대부분의 최신 Node.js 런타임에서는 잘 동작하지만,
일부 라이브러리나 오래된 래퍼 코드에서는 비동기 경계에서 문맥이 끊기는 경우가 있습니다.
그래서 도입 후에는 꼭 아래를 확인해야 합니다.

1. HTTP 진입 → 서비스 레이어 → DB/외부 API 호출까지 requestId가 유지되는가
2. 큐 워커 재시도 시도별로 새 컨텍스트가 정확히 생성되는가
3. 에러 로깅 시 requestId와 작업 이름이 빠지지 않는가

"설치했다"보다 중요한 건 **실제 운영 경로에서 문맥이 끝까지 살아 있는지** 확인하는 것입니다.

## 배포 전 체크리스트

### H3. 코드와 운영 기준 점검

- 요청 시작 지점마다 `AsyncLocalStorage.run()`이 일관되게 호출되는가
- 공통 로거가 컨텍스트 값을 자동 주입하는가
- 민감정보 없이 requestId, userId, upstream, duration 정도만 기록하는가
- 샘플링 정책이나 로그 적재 비용까지 고려했는가

### H3. SEO·품질 점검

- 제목에 `Node.js AsyncLocalStorage`와 실무 의도가 자연스럽게 포함됐는가
- 첫 단락에서 문제와 해결 가치를 바로 설명했는가
- H2/H3 구조가 명확하고 내부링크가 연결돼 있는가
- 과장 표현 없이 재현 가능한 코드 예시를 제공했는가

## 요약

`AsyncLocalStorage`의 핵심 가치는 "코드를 멋지게 보이게 하는 것"이 아니라,
**요청 단위 문맥을 잃지 않고 장애를 더 빨리 읽게 만드는 것**에 있습니다.
파라미터 전달에 의존하던 로그 체계를 공통 컨텍스트 기반으로 바꾸면,
운영 중에는 로그 검색 시간이 줄고 개발 중에는 디버깅 흐름이 훨씬 단순해집니다.

## 내부 링크

- [Node.js AbortController 실전 가이드: 타임아웃·취소 표준화로 장애 반경 줄이기](/development/blog/seo/2026/03/19/nodejs-abortcontroller-timeout-cancellation-pattern-guide.html)
- [Node.js 스트림 백프레셔 실전 가이드: 메모리 급증과 처리 지연 줄이기](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)
- [Node.js Kafka 중복 메시지 처리 가이드: Idempotent Consumer로 재처리 안전성 높이기](/development/blog/seo/2026/03/17/nodejs-idempotent-consumer-kafka-duplicate-message-handling-guide.html)

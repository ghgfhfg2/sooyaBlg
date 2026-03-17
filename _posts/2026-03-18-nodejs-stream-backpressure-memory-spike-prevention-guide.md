---
layout: post
title: "Node.js 스트림 백프레셔 실전 가이드: 메모리 급증 없이 대용량 처리 안정화하기"
date: 2026-03-18 08:00:00 +0900
lang: ko
translation_key: nodejs-stream-backpressure-memory-spike-prevention-guide
permalink: /development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html
alternates:
  ko: /development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html
  x_default: /development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, stream, backpressure, memory, performance, backend]
description: "Node.js에서 대용량 파일/이벤트 처리 시 발생하는 메모리 급증 문제를 백프레셔로 해결하는 방법을 정리했습니다. stream.pipeline, highWaterMark 튜닝, 모니터링 체크리스트까지 실무 기준으로 설명합니다."
---

대용량 로그, CSV, 이벤트를 처리할 때 Node.js 서비스가 갑자기 느려지거나 메모리를 과도하게 쓰는 경우가 있습니다.
대부분은 "처리 속도" 문제가 아니라 **생산자(읽기)가 소비자(쓰기)보다 빠른 상태를 방치**해서 생깁니다.
이 글에서는 백프레셔(Backpressure)를 기준으로 Node.js 스트림 파이프라인을 안정화하는 실전 방법을 정리합니다.

## 백프레셔를 이해해야 메모리 급증을 막을 수 있다

### H3. 백프레셔란 무엇인가

백프레셔는 "지금은 더 못 받는다"는 신호를 upstream으로 전달해
버퍼가 무한정 쌓이지 않게 제어하는 메커니즘입니다.

- 읽기 속도 > 처리 속도 > 쓰기 속도 순으로 불균형이 생기면 메모리 사용량 급증
- 백프레셔가 정상 작동하면 생산자 속도가 자동으로 완화
- 결과적으로 GC 부담, 지연, OOM 위험이 감소

### H3. 흔한 안티패턴

아래처럼 데이터를 배열에 계속 쌓아두고 한 번에 처리하면,
입력량이 커질수록 힙 메모리가 빠르게 증가합니다.

```ts
// 안티패턴 예시
const chunks: Buffer[] = [];
readable.on('data', (chunk) => {
  chunks.push(chunk); // 입력이 빠르면 메모리 누적
});

readable.on('end', async () => {
  await processAll(Buffer.concat(chunks));
});
```

핵심은 "나중에 한 번에"가 아니라 "들어오는 즉시, 처리 가능한 속도로" 흘려보내는 구조입니다.

## Node.js에서 안전한 스트림 파이프라인 구성

### H3. stream.pipeline을 기본값으로 사용하기

`pipeline`은 스트림 연결, 에러 전파, 정리를 한 번에 처리해 실수 여지를 줄입니다.

```ts
import { createReadStream, createWriteStream } from 'node:fs';
import { Transform } from 'node:stream';
import { pipeline } from 'node:stream/promises';

const transform = new Transform({
  objectMode: false,
  transform(chunk, _enc, cb) {
    // CPU 비용이 큰 작업은 워커/큐 분리 고려
    const out = chunk.toString('utf8').replaceAll('ERROR', 'WARN');
    cb(null, Buffer.from(out));
  },
});

await pipeline(
  createReadStream('app.log', { highWaterMark: 64 * 1024 }),
  transform,
  createWriteStream('app.sanitized.log')
);
```

이 패턴은 소비자가 느려지면 upstream도 자연스럽게 속도를 조절합니다.

### H3. highWaterMark 튜닝 기준

`highWaterMark`를 무작정 키우면 처리량이 늘기보다 메모리만 늘어날 수 있습니다.
실무에서는 다음 순서로 조정하는 편이 안전합니다.

1. 기본값으로 시작해 처리량/메모리 기준선 측정
2. 병목 구간(디스크, 네트워크, CPU)을 먼저 식별
3. 작은 단위(예: 64KB → 128KB)로 점진 조정
4. 조정마다 p95 지연, RSS/Heap, GC pause 비교

## 운영 환경에서 자주 터지는 문제와 대응

### H3. 문제 1) 외부 API/DB가 느려져 버퍼 적체

스트림 중간에서 비동기 I/O를 수행할 때 응답 지연이 늘어나면,
앞단 입력이 계속 들어오며 적체가 커집니다.

대응:
- 동시성 제한(예: p-limit, 큐) 적용
- 단계별 타임아웃 설정
- 실패 빠른 전파(fail-fast)로 대기열 장기 점유 방지

### H3. 문제 2) 장애 시 파이프라인 일부만 종료

한 스트림에서 에러가 나도 다른 구간이 열린 채 남으면 핸들/메모리 누수가 발생합니다.

대응:
- `pipeline`으로 통합 에러 처리
- `AbortController`로 취소 경로 표준화
- 종료 시 열린 리소스(파일/소켓) 명시적으로 정리

### H3. 문제 3) 관측 없이 감으로 튜닝

"메모리가 좀 높아 보인다" 수준으로는 재발 방지가 어렵습니다.

대응 지표:
- 프로세스 메모리(RSS, Heap Used)
- 스트림 처리량(초당 레코드/바이트)
- p95 처리 지연, 타임아웃 비율
- OOM 직전 로그 패턴/GC pause

## 배포 전 체크리스트

### H3. 코드/설계

- `pipeline` 기반으로 연결했는가
- Transform 단계에서 무제한 버퍼링을 하지 않는가
- 동시성 제한과 타임아웃이 있는가

### H3. 보안/안전

- 로그/예시에 토큰, API 키, 개인정보가 없는가
- 실패 로그에 민감 값이 그대로 노출되지 않는가
- 과장된 성능 수치(검증 불가 주장)를 쓰지 않았는가

## 요약

Node.js에서 대용량 처리 안정성은 "빠른 코드"보다 **흐름 제어**에서 갈립니다.
백프레셔를 기준으로 `stream.pipeline`, 점진적 `highWaterMark` 튜닝, 운영 지표 관측을 결합하면
메모리 급증과 장애 반경을 크게 줄일 수 있습니다.

## 내부 링크

- [/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)
- [/development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html](/development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html)
- [/development/blog/seo/2026/03/12/rate-limiting-nodejs-express-redis-abuse-protection-guide.html](/development/blog/seo/2026/03/12/rate-limiting-nodejs-express-redis-abuse-protection-guide.html)

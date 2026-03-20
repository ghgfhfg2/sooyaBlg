---
layout: post
title: "Node.js 응답 압축 실전 가이드: Brotli·Gzip으로 API 성능과 Core Web Vitals 함께 개선하기"
date: 2026-03-20 20:00:00 +0900
lang: ko
translation_key: nodejs-response-compression-brotli-gzip-performance-guide
permalink: /development/blog/seo/2026/03/20/nodejs-response-compression-brotli-gzip-performance-guide.html
alternates:
  ko: /development/blog/seo/2026/03/20/nodejs-response-compression-brotli-gzip-performance-guide.html
  x_default: /development/blog/seo/2026/03/20/nodejs-response-compression-brotli-gzip-performance-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, compression, brotli, gzip, performance, backend]
description: "Node.js에서 Brotli와 Gzip 응답 압축을 어떻게 적용하고, 어떤 콘텐츠에 선택적으로 써야 하며, CPU 비용과 캐시 전략은 어떻게 함께 봐야 하는지 실무 기준으로 정리했습니다."
---

API가 느린 이유가 항상 DB나 외부 API 때문만은 아닙니다.
응답 본문이 크고 네트워크 구간이 길다면, **전송 바이트 자체를 줄이는 것**만으로도 체감 성능이 꽤 달라집니다.
이 글에서는 Node.js 서비스에서 `Brotli`와 `Gzip` 응답 압축을 언제 쓰는지, 무엇을 압축해야 하고 무엇은 건드리지 말아야 하는지, 그리고 CPU 비용·캐시·관측 지표를 어떻게 같이 봐야 하는지 실무 기준으로 정리합니다.

## 왜 응답 압축이 아직도 중요한가

### H3. JSON API도 압축 효과가 꽤 크다

응답 압축이라고 하면 정적 HTML이나 JS 번들만 떠올리기 쉽지만, 실제 운영에서는 JSON API도 효과가 큽니다.
특히 아래처럼 텍스트 비중이 높은 응답은 압축 효율이 좋습니다.

- 목록 조회 API
- 검색 결과 API
- CMS·설정 데이터 응답
- 서버 사이드 렌더링 HTML

예를 들어 120KB짜리 JSON 응답이 20~35KB 수준으로 줄어들면, 모바일 네트워크나 해외 구간에서 체감 차이가 분명하게 납니다.
백엔드 입장에서는 같은 기능을 유지한 채 **대역폭 비용과 응답 전송 시간을 동시에 줄이는** 셈입니다.

### H3. 다만 CPU를 공짜로 바꾸는 기술은 아니다

압축은 네트워크를 아끼는 대신 CPU를 씁니다.
그래서 무조건 최고 압축률만 추구하면 오히려 서버 사용량이 증가하고, 피크 타임에는 응답 지연이 커질 수 있습니다.
핵심은 단순합니다.

- 텍스트 기반 콘텐츠는 압축 후보로 본다
- 이미 압축된 바이너리는 보통 제외한다
- 압축 레벨은 품질보다 **응답 지연과 CPU 예산** 기준으로 잡는다

즉, 응답 압축은 켜기만 하면 끝나는 옵션이 아니라 **트래픽 형태에 맞춘 튜닝 항목**입니다.

## Brotli와 Gzip은 어떻게 다를까

### H3. Brotli는 더 잘 줄어들지만 비용이 더 클 수 있다

일반적으로 Brotli는 Gzip보다 압축률이 더 좋습니다.
특히 텍스트 자산에서 효과가 좋고, 프론트엔드 정적 파일 배포에서는 Brotli가 많이 쓰입니다.
다만 동적 응답까지 실시간 Brotli 최고 레벨로 처리하면 CPU 부담이 커질 수 있습니다.

실무적으로는 이렇게 생각하면 편합니다.

- **정적 자산**: 미리 Brotli 압축해 두면 효율이 좋다
- **동적 API 응답**: 너무 높은 Brotli 레벨은 신중히 쓴다
- **광범위 호환성**: Gzip은 여전히 무난한 기본값이다

### H3. Accept-Encoding 협상과 Vary 헤더를 같이 봐야 한다

클라이언트가 `Accept-Encoding: br, gzip`를 보내면 서버는 지원 가능한 포맷을 골라 응답할 수 있습니다.
이때 중요한 것은 `Vary: Accept-Encoding` 헤더입니다.
이 헤더가 빠지면 CDN이나 프록시 캐시가 압축 버전을 잘못 섞어 전달할 수 있습니다.

즉, 압축은 단순히 본문만 바꾸는 기능이 아니라, **캐시 키 설계와 연결된 HTTP 동작**입니다.

## Node.js에서 기본 적용 패턴

### H3. Express라면 compression 미들웨어로 시작할 수 있다

가장 간단한 출발점은 `compression` 미들웨어입니다.
기본 설정만으로도 Gzip/Deflate 압축을 쉽게 붙일 수 있습니다.

```ts
import express from 'express';
import compression from 'compression';

const app = express();

app.use(compression({
  threshold: 1024,
  level: 6,
}));
```

여기서 `threshold`는 너무 작은 응답까지 압축하지 않도록 막는 역할을 합니다.
작은 응답은 헤더와 CPU 비용을 감안하면 이득이 거의 없기 때문입니다.

### H3. 필터 함수로 압축 대상을 명확히 제한하는 편이 안전하다

실무에서는 "전부 압축"보다 "압축할 것만 압축"이 더 예측 가능할 때가 많습니다.
예를 들면 아래처럼 필터를 둘 수 있습니다.

```ts
import compression from 'compression';

function shouldCompress(req, res) {
  const contentType = res.getHeader('Content-Type')?.toString() ?? '';

  if (req.headers['x-no-compression']) return false;
  if (contentType.startsWith('image/')) return false;
  if (contentType.startsWith('video/')) return false;
  if (contentType.includes('application/zip')) return false;

  return compression.filter(req, res);
}

app.use(compression({
  threshold: 1024,
  level: 5,
  filter: shouldCompress,
}));
```

이렇게 하면 이미 압축된 자산이나 압축 이득이 낮은 바이너리를 굳이 다시 처리하지 않아도 됩니다.
운영 중 CPU 예산을 지키는 데도 도움이 됩니다.

## 어떤 응답은 압축하지 않는 게 낫다

### H3. 이미지·동영상·zip 파일은 대부분 제외 대상이다

JPEG, PNG, WebP, MP4, ZIP처럼 이미 압축된 포맷은 재압축 효과가 거의 없거나 오히려 손해일 수 있습니다.
이런 응답까지 무조건 압축하면 CPU만 더 쓰고 지연도 늘어납니다.

압축 우선순위는 보통 아래 순서로 잡는 편이 낫습니다.

1. HTML
2. CSS/JS
3. JSON
4. SVG
5. 텍스트 로그·리포트

반대로 바이너리 파일 다운로드 API는 먼저 제외 후보로 보는 편이 현실적입니다.

### H3. 실시간 스트리밍과 SSE는 별도 검토가 필요하다

Server-Sent Events(SSE), 장시간 스트리밍 응답, chunked 전송은 압축과 궁합이 애매할 수 있습니다.
버퍼링 특성 때문에 이벤트가 늦게 보이거나, flush 타이밍이 기대와 달라질 수 있기 때문입니다.

따라서 아래처럼 생각하는 편이 안전합니다.

- SSE: 기본적으로 비활성화 검토
- 대용량 다운로드: 원본 그대로 전달 검토
- 일반 JSON/HTML: 적극 적용

## Brotli·Gzip 운영 기준은 어떻게 잡을까

### H3. 동적 응답은 중간 압축 레벨부터 시작하는 편이 낫다

압축률 1~2% 더 얻으려고 CPU를 과하게 쓰는 경우가 흔합니다.
실전에서는 극단적 최고 레벨보다 **중간 레벨에서 전체 지연이 안정적인지**가 더 중요합니다.

권장 접근은 단순합니다.

- Gzip: `level 4~6`부터 시작
- Brotli: 동적 응답은 너무 높은 레벨을 피함
- CDN/리버스 프록시가 대신 압축할 수 있으면 역할 분담 검토

애플리케이션 서버가 이미 DB/비즈니스 로직으로 바쁜데 압축까지 무겁게 떠안으면, 피크 타임에 병목이 생기기 쉽습니다.

### H3. CDN과 앱 서버 중 어디서 압축할지도 정해야 한다

Cloudflare, Nginx, CDN 계층에서 압축을 맡길 수도 있습니다.
이 경우 앱 서버는 비즈니스 로직에 집중하고, 엣지에서 클라이언트 맞춤 압축을 처리할 수 있습니다.
다만 아래를 함께 봐야 합니다.

- 원본 응답이 캐시 가능한가
- 사용자별로 달라지는 private 응답이 많은가
- 앱 서버와 CDN 중 어느 쪽 CPU 예산이 더 여유로운가

정적 파일은 사전 압축 + CDN 캐시 조합이 가장 예측 가능하고, 동적 API는 앱 서버 압축 또는 프록시 압축 중 하나를 명확히 책임지게 하는 편이 좋습니다.

## 캐시와 관측 지표를 함께 봐야 하는 이유

### H3. 압축 응답은 캐시 키와 함께 설계해야 한다

압축 포맷이 달라지면 응답 바이트가 달라지므로 캐시도 영향을 받습니다.
따라서 CDN, 리버스 프록시, 브라우저 캐시가 어떤 단위로 응답을 저장하는지 확인해야 합니다.

최소한 아래는 같이 점검하는 편이 좋습니다.

- `Vary: Accept-Encoding` 설정 여부
- 캐시 히트율이 압축 도입 전후로 어떻게 변했는가
- 원본 서버 트래픽과 전송 바이트가 얼마나 줄었는가

압축은 네트워크 최적화이지만, 실제 효과는 **캐시 전략과 합쳐졌을 때** 더 크게 나타납니다.

### H3. 평균 응답시간만 보지 말고 CPU와 p95도 같이 본다

압축 도입 후 평균 전송 시간은 줄어도, CPU가 치솟아 p95나 p99가 나빠질 수 있습니다.
그래서 아래 지표를 같이 보는 게 좋습니다.

- compressed/uncompressed 응답 크기
- route별 p50, p95, p99 응답시간
- 인스턴스 CPU 사용률
- 네트워크 egress 감소량
- 에러율과 타임아웃 변화

핵심 질문은 이것입니다.
**"바이트는 줄었는데 전체 사용자 경험도 실제로 좋아졌는가?"**

## 실전 체크리스트

### H3. 적용 전 점검

- 텍스트 기반 응답과 이미 압축된 바이너리를 분리했는가
- `Accept-Encoding`과 `Vary` 헤더 동작을 이해하고 있는가
- SSE, 다운로드, 프록시 구간 같은 예외 경로를 확인했는가
- CDN과 앱 서버 중 압축 책임 주체를 정했는가

### H3. 배포 후 점검

- 압축 적용 후 CPU와 p95 응답시간이 안정적인가
- 응답 크기 감소가 실제 egress 절감으로 이어졌는가
- 캐시 계층에서 인코딩별 응답이 올바르게 분리되는가
- 코드 예시와 설정값에 민감정보, 내부 도메인, 토큰이 없는가

## 요약

Node.js 응답 압축은 생각보다 단순한 최적화이면서도, 잘 적용하면 **전송 시간·대역폭·체감 성능**을 동시에 개선할 수 있습니다.
다만 모든 응답을 기계적으로 압축하기보다, 텍스트 응답 중심으로 적용하고 `Brotli`와 `Gzip`의 비용 차이, `Vary` 헤더, CDN 역할 분담, CPU 지표를 함께 보는 편이 훨씬 안전합니다.
결국 중요한 것은 "압축을 켰다"가 아니라, **사용자 경험과 운영 비용이 같이 좋아졌는지** 확인하는 것입니다.

## 내부 링크

- [Node.js 스트림 백프레셔 실전 가이드: 메모리 급증과 처리 지연 줄이기](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)
- [Node.js AbortController 실전 가이드: 타임아웃·취소 표준화로 장애 반경 줄이기](/development/blog/seo/2026/03/19/nodejs-abortcontroller-timeout-cancellation-pattern-guide.html)
- [Node.js AsyncLocalStorage 실전 가이드: 요청 컨텍스트·로그 추적으로 디버깅 시간 줄이기](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)

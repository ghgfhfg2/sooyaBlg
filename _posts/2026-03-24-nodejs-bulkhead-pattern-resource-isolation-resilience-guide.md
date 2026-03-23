---
layout: post
title: "Node.js Bulkhead Pattern 가이드: 리소스 격리로 느린 의존성 장애가 전체 서비스로 번지는 것 막기"
date: 2026-03-24 08:00:00 +0900
lang: ko
translation_key: nodejs-bulkhead-pattern-resource-isolation-resilience-guide
permalink: /development/blog/seo/2026/03/24/nodejs-bulkhead-pattern-resource-isolation-resilience-guide.html
alternates:
  ko: /development/blog/seo/2026/03/24/nodejs-bulkhead-pattern-resource-isolation-resilience-guide.html
  x_default: /development/blog/seo/2026/03/24/nodejs-bulkhead-pattern-resource-isolation-resilience-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, bulkhead, resilience, backend, concurrency, reliability]
description: "Node.js 서비스에서 bulkhead pattern을 적용해 외부 API 지연, 큐 적체, 커넥션 고갈이 전체 장애로 번지는 것을 막는 방법을 실무 관점으로 정리했습니다."
---

Node.js 서비스를 운영하다 보면 문제는 항상 "완전히 죽는 것"보다 **일부가 느려지기 시작하는 것**에서 출발하는 경우가 많습니다.
예를 들어 결제 API 하나가 지연되기 시작했는데 같은 프로세스 안에서 같은 커넥션 풀·같은 워커·같은 이벤트 루프 여유분을 함께 갉아먹으면, 원래 멀쩡해야 할 다른 기능까지 같이 느려집니다.
이 글에서는 **Node.js bulkhead pattern** 을 통해 외부 의존성 장애가 전체 서비스로 번지는 것을 어떻게 막을지, 그리고 타임아웃·서킷 브레이커·큐 분리와 어떻게 함께 써야 하는지 실무 기준으로 정리합니다.

## 왜 bulkhead pattern이 Node.js 운영에서 중요한가

### H3. 장애는 한 지점에서 시작해도 자원은 전체에서 고갈된다

bulkhead pattern은 원래 배의 격벽 개념에서 온 말입니다.
한 구획에 물이 차도 배 전체가 가라앉지 않도록 칸을 나누는 것처럼, 소프트웨어에서도 **느린 작업과 중요한 작업이 같은 자원을 무제한 공유하지 않게 분리**하는 사고방식입니다.

Node.js에서는 특히 아래 상황에서 bulkhead가 필요합니다.

- 외부 API 호출이 갑자기 3초, 5초씩 지연된다
- 특정 큐 소비자가 실패 재시도로 워커를 오래 점유한다
- 이미지 처리나 PDF 생성 같은 무거운 작업이 동시에 몰린다
- 한 종류의 요청이 DB 커넥션 풀을 과하게 점유한다
- 백오피스·배치성 요청이 사용자-facing API 응답성을 잠식한다

이런 문제는 코드 버그가 없어도 생깁니다.
핵심은 "요청이 실패했는가"보다 **느린 요청이 어떤 공유 자원을 얼마나 오래 붙잡고 있느냐**입니다.

### H3. 이벤트 루프 하나만 본다고 해결되지 않는다

Node.js를 쓰면 흔히 이벤트 루프 블로킹만 피하면 된다고 생각하기 쉽습니다.
하지만 실제 운영에서 번지는 장애는 이벤트 루프 자체보다 **공유된 실행 슬롯, DB 풀, HTTP 에이전트 소켓, 큐 워커 개수** 때문에 더 자주 발생합니다.

예를 들어 아래처럼 벌어질 수 있습니다.

1. 외부 추천 API가 느려진다
2. 추천 API를 기다리는 Promise가 동시에 수백 개 쌓인다
3. 애플리케이션 메모리와 커넥션 점유 시간이 늘어난다
4. 원래 빠르게 끝나야 할 프로필 조회나 주문 조회까지 대기열에 갇힌다
5. 결국 전체 서비스가 느려진 것처럼 보인다

즉 bulkhead는 "성능 최적화 기법"이라기보다 **장애 전파 범위를 제한하는 구조적 안전장치**에 가깝습니다.

## Node.js bulkhead pattern의 핵심은 무엇일까

### H3. 작업 종류마다 동시성 예산을 따로 둔다

bulkhead pattern의 핵심은 단순합니다.
모든 작업을 하나의 큰 풀에서 처리하지 말고, **업무 중요도와 비용 특성에 따라 동시성 예산을 분리**하는 것입니다.

예를 들면 아래처럼 나눌 수 있습니다.

- 결제 승인 API: 동시성 20
- 추천 API 호출: 동시성 10
- 이미지 썸네일 생성: 동시성 4
- 관리자용 CSV 다운로드: 동시성 2
- 웹훅 후처리 백그라운드 작업: 별도 워커 풀

이렇게 두면 추천 API가 느려져도 결제 승인 슬롯을 전부 먹어치우지 못합니다.
반대로 관리자 기능이 무거워도 일반 사용자 요청을 함께 질식시키는 일을 줄일 수 있습니다.

### H3. 격리는 프로세스 분리만 뜻하지 않는다

bulkhead를 들으면 마이크로서비스나 별도 프로세스 분리만 떠올리기 쉽습니다.
물론 가장 강한 격리는 프로세스·컨테이너·서비스 분리입니다.
하지만 Node.js 실무에서는 그보다 먼저 적용할 수 있는 계층이 많습니다.

- 기능별 동시 요청 수 제한
- 외부 API별 HTTP agent/socket 분리
- DB 읽기/쓰기 풀 또는 워커 큐 분리
- 사용자 요청용 워커와 배치용 워커 분리
- CPU 집약 작업의 worker_threads 분리
- 중요 API와 부가 기능의 별도 큐/토픽 운영

즉 "같은 서버니까 어쩔 수 없다"가 아니라, **같은 프로세스 안에서도 공유 자원을 어디까지 같이 쓰게 둘지 결정**하는 것이 중요합니다.

## 어디서 먼저 격리해야 할까

### H3. 외부 API 호출부부터 보는 편이 효과가 크다

대부분의 장애 전파는 내가 통제하지 못하는 외부 의존성에서 시작합니다.
그래서 첫 bulkhead 적용 지점으로는 외부 API 호출부가 가장 실용적입니다.

특히 아래 조건이면 우선순위가 높습니다.

- 응답 시간이 들쭉날쭉한 API
- 공급자 장애 이력이 있는 API
- 호출당 비용이 큰 API
- 사용자 요청 흐름에서 선택적(optional)인 기능
- 재시도 로직까지 붙어 있어 증폭 가능성이 큰 API

이 지점은 <a href="/development/blog/seo/2026/03/19/nodejs-abortcontroller-timeout-cancellation-pattern-guide.html">Node.js AbortController timeout 가이드</a>와 <a href="/development/blog/seo/2026/03/12-circuit-breaker-pattern-nodejs-external-api-stability-guide.html">Node.js circuit breaker pattern 가이드</a>에서 다룬 타임아웃·빠른 실패 전략과도 연결됩니다.
타임아웃이 "얼마나 오래 기다릴지"를 정한다면, bulkhead는 **동시에 몇 개까지 기다리게 둘지**를 정합니다.

### H3. 큐와 백그라운드 워커도 격리하지 않으면 API까지 흔들린다

백그라운드 작업은 사용자 응답 경로 밖에 있으니 안전하다고 생각하기 쉽습니다.
하지만 같은 Redis, 같은 DB, 같은 외부 API 한도를 공유하면 큐 적체가 결국 API 품질까지 끌어내릴 수 있습니다.

예를 들어 아래와 같은 실수가 흔합니다.

- 이메일 발송 재시도가 급증하며 Redis/DB 부하를 키운다
- 대량 리포트 생성 작업이 워커 CPU를 오래 점유한다
- 실패한 작업이 DLQ로 가지 않고 계속 hot loop 재처리된다
- 배치 작업이 평일 업무 시간에도 동일 우선순위로 돈다

이런 경우에는 API 서버와 별개로 **작업 종류별 큐 분리, 워커 수 상한, 재시도 간격, DLQ 정책**을 같이 봐야 합니다.
관련해서는 <a href="/development/blog/seo/2026/03/14/dead-letter-queue-bullmq-nodejs-failed-job-recovery-guide.html">BullMQ dead letter queue 가이드</a>도 함께 보면 설계가 훨씬 선명해집니다.

## Node.js에서 간단히 적용하는 bulkhead 예시

### H3. p-limit 같은 동시성 제한기로 기능별 슬롯을 분리한다

거창한 프레임워크가 없어도 시작은 단순합니다.
예를 들어 외부 추천 API와 결제 API의 동시 호출 상한을 다르게 둘 수 있습니다.

```ts
import pLimit from 'p-limit';

const recommendationLimit = pLimit(10);
const paymentLimit = pLimit(20);

async function fetchRecommendations(userId: string) {
  return recommendationLimit(async () => {
    return callRecommendationApi(userId);
  });
}

async function approvePayment(payload: PaymentPayload) {
  return paymentLimit(async () => {
    return callPaymentGateway(payload);
  });
}
```

이 예시의 포인트는 라이브러리 자체보다 **기능별로 동시성 버짓을 분리했다**는 점입니다.
추천 API가 밀려도 결제 승인 슬롯을 침범하지 못하게 만드는 것이 목적입니다.

### H3. 타임아웃과 함께 써야 대기열이 무한정 길어지지 않는다

bulkhead만 두고 타임아웃이 없으면 대기열이 줄지 않을 수 있습니다.
느린 작업이 슬롯을 너무 오래 점유하면 결국 새 요청은 계속 쌓이기 때문입니다.

그래서 보통 아래 조합이 좋습니다.

- 동시성 상한: 동시에 몇 개까지 허용할지 제한
- 요청 타임아웃: 오래 걸리는 작업은 중단
- 대기열 상한: 너무 많이 쌓이면 즉시 거절
- fallback: 선택 기능이면 기본 응답으로 대체

예를 들어 추천 영역이라면 전체 페이지를 실패시키기보다 "추천 없음"으로 응답하는 편이 낫습니다.
반면 결제 승인이라면 fallback보다 명확한 실패와 재시도 정책이 더 중요할 수 있습니다.

## HTTP 클라이언트와 커넥션 풀도 격리해야 할까

### H3. 같은 keep-alive 소켓 풀을 공유하면 병목이 번질 수 있다

Node.js에서 `fetch`, `axios`, `undici`, DB 드라이버를 쓸 때는 결국 커넥션 풀이나 소켓 풀의 영향을 받습니다.
만약 서로 성격이 다른 외부 API가 같은 풀 설정을 공유하면, 한쪽 지연이 다른 요청의 대기시간에 영향을 줄 수 있습니다.

실무에서는 아래를 점검해볼 만합니다.

- 외부 API별 별도 client/agent 사용 여부
- max sockets / connections 설정값
- keep-alive 유지 정책
- 큐잉된 요청 수 관찰 가능 여부
- 공급자별 timeout 값 분리 여부

중요한 점은 "클라이언트 인스턴스를 하나로 통일하면 깔끔하다"보다 **느린 의존성이 다른 의존성의 연결 자원까지 삼키지 못하게 하는 것**입니다.

### H3. DB도 모든 요청이 같은 풀을 무한정 공유하게 두면 위험하다

DB는 더 민감합니다.
특정 기능이 느린 쿼리를 만들거나 락 경쟁을 유발하면, 같은 풀을 쓰는 다른 요청까지 줄줄이 대기할 수 있습니다.

그래서 아래 같은 질문이 필요합니다.

- 읽기와 쓰기 트래픽을 분리할 수 있는가
- 배치성 쿼리를 업무 시간대에 별도 제한할 수 있는가
- 관리자 검색, 통계 집계, 일반 사용자 조회를 같은 경로로 둘 것인가
- connection pool saturation 지표를 보고 있는가

bulkhead는 꼭 DB를 두 개 만든다는 뜻이 아니라, **무거운 작업이 같은 연결 예산을 독점하지 못하게 설계하는 것**입니다.

## bulkhead pattern만으로 충분하지 않은 이유

### H3. 서킷 브레이커, 재시도, 백오프와 함께 봐야 한다

bulkhead는 장애 확산을 줄이지만 장애 원인을 없애지는 않습니다.
그래서 보통 아래 패턴과 함께 동작해야 효과가 납니다.

- timeout: 느린 작업을 빨리 끊기
- retry with jitter: 필요한 재시도만 분산해서 수행
- circuit breaker: 실패가 이어질 때 빠르게 차단
- fallback: 선택 기능은 기본값으로 대체
- observability: 대기열 길이·슬롯 사용률·실패율 추적

특히 재시도는 bulkhead 없이 쓰면 역효과가 날 수 있습니다.
실패한 요청을 다시 보내는 동안 같은 슬롯을 더 많이 잡아먹을 수 있기 때문입니다.
이 부분은 <a href="/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html">Node.js exponential backoff with jitter 가이드</a>에서 설명한 증폭 문제와 정확히 맞물립니다.

### H3. 관측 지표가 없으면 격리 설계가 맞는지 판단하기 어렵다

bulkhead를 걸었더라도 운영에서 아래 지표를 못 보면 "잘 막고 있는지" 알 수 없습니다.

- 기능별 동시 실행 수
- 대기열 길이
- 슬롯 획득 대기 시간
- timeout 발생률
- fallback 비율
- 외부 API별 p95/p99 지연 시간
- DB pool saturation

즉 bulkhead는 코드 한두 줄로 끝나는 문제가 아니라 **자원 예산을 측정하고 유지하는 운영 방식**에 가깝습니다.

## 실무 체크리스트: 어디까지 나누면 좋을까

### H3. 사용자 핵심 경로와 부가 기능을 먼저 분리한다

처음부터 모든 기능에 복잡한 풀을 만들 필요는 없습니다.
대신 아래 우선순위로 보는 편이 좋습니다.

1. 결제·인증·주문 같은 핵심 경로
2. 선택적 기능인 추천·알림·통계·리포트
3. 백오피스와 배치 작업
4. CPU 집약 처리와 대용량 다운로드
5. 외부 공급자별 API 호출부

이 순서대로만 정리해도 장애 전파가 크게 줄어드는 경우가 많습니다.

### H3. "다 허용 후 모니터링"보다 "작게 제한 후 조정"이 안전하다

bulkhead 설정값은 처음부터 완벽할 수 없습니다.
그래도 무제한으로 열어둔 뒤 사고를 보는 것보다, 보수적으로 시작해 지표를 보며 조정하는 편이 안전합니다.

예를 들면 이런 식입니다.

- 추천 API 동시성 10으로 시작
- 429 또는 fallback 비율 관찰
- 정상 트래픽에도 거절이 많으면 12~15로 소폭 증가
- 장애 시 핵심 API latency가 방어되는지 함께 확인

중요한 건 최대 처리량만 보는 것이 아니라, **중요 경로를 지켜냈는가**를 기준으로 판단하는 것입니다.

## 마무리

Node.js bulkhead pattern은 "느린 의존성이 생겨도 전체 서비스가 함께 가라앉지 않게 하는 설계"입니다.
타임아웃이 늦은 요청 하나를 끊는 도구라면, bulkhead는 그런 요청이 동시에 몰려도 **핵심 기능까지 잠식하지 못하게 막는 구조**입니다.

운영에서 가장 위험한 장애는 올오어낫싱보다는 점진적 전염 형태로 옵니다.
그래서 외부 API, 큐, 워커, DB 풀, 백오피스 기능이 어떤 자원을 공유하는지 한 번만 제대로 그려봐도 개선 포인트가 꽤 많이 보입니다.
오늘 바로 할 수 있는 첫 단계는 단순합니다.
**중요한 요청과 덜 중요한 요청을 같은 동시성 풀에 무제한으로 태우고 있지는 않은지 확인하는 것**부터 시작하면 됩니다.

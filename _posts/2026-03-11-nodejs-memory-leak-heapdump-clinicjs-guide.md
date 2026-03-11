---
layout: post
title: "Node.js 메모리 누수 추적 실전: heap snapshot·clinic.js로 OOM 재현하고 해결하기"
date: 2026-03-11 20:00:00 +0900
lang: ko
translation_key: nodejs-memory-leak-heapdump-clinicjs-guide
permalink: /development/blog/seo/2026/03/11/nodejs-memory-leak-heapdump-clinicjs-guide.html
alternates:
  ko: /development/blog/seo/2026/03/11/nodejs-memory-leak-heapdump-clinicjs-guide.html
  x_default: /development/blog/seo/2026/03/11/nodejs-memory-leak-heapdump-clinicjs-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, memory-leak, heap-snapshot, clinicjs, observability]
description: "Node.js 서비스에서 반복적으로 OOM이 발생할 때, heap snapshot과 clinic.js를 이용해 메모리 누수를 재현·진단·수정하는 실전 절차를 정리합니다. 프로덕션에서 안전하게 적용할 체크리스트도 함께 제공합니다."
---

## 문제: CPU는 안정적인데 왜 Node.js만 주기적으로 OOM이 날까

실서비스에서 자주 보는 장애 패턴이 있습니다.
트래픽은 평소와 비슷한데, 몇 시간~며칠 주기로 컨테이너가 OOMKilled 되는 경우입니다.

이때 핵심은 "순간 사용량"보다 **메모리가 회수되지 않고 누적되는 흐름**을 확인하는 것입니다.
Node.js는 GC가 자동으로 동작하지만, 참조가 남아 있으면 객체는 계속 힙에 남습니다.

즉, OOM 대응은 메모리 제한을 키우는 문제가 아니라,
**어떤 객체가 왜 살아남는지**를 증거 기반으로 추적하는 작업입니다.

## 진단 원칙: 재현 가능한 증거를 먼저 확보한다

### 1) 누수 의심 지표를 먼저 고정

아래 지표가 동시에 보이면 누수 가능성이 큽니다.

- `heapUsed`가 트래픽 감소 후에도 이전 수준으로 내려오지 않음
- Full GC 이후에도 힙 기준선(baseline)이 계단형으로 상승
- RSS 증가 속도가 요청 수 증가보다 빠름

먼저 관측 지표를 고정해야 "느낌"이 아니라 "재현"으로 진단할 수 있습니다.

### 2) 동일 조건으로 heap snapshot 2~3개 수집

스냅샷은 보통 아래 순서로 수집합니다.

1. 워밍업 직후(기준점)
2. 부하 15~30분 후
3. OOM 직전 또는 직전 시점과 유사한 구간

비교 시에는 객체 개수보다 **Retained Size 증가 상위 타입**을 우선 확인하세요.
증가량이 큰 타입의 참조 체인을 따라가면 원인 코드에 빠르게 도달합니다.

## 실전 절차: heap snapshot + clinic.js 조합

### Heap snapshot 수집 예시

```bash
# 1) 인스턴스 PID 확인
ps -ef | grep node

# 2) inspector 활성화(환경에 맞게 포트/보안 제한 필수)
node --inspect=0.0.0.0:9229 server.js

# 3) Chrome DevTools 또는 node --heapsnapshot-signal 사용
node --heapsnapshot-signal=SIGUSR2 server.js
kill -USR2 <PID>
```

운영 환경에서는 스냅샷 파일 크기와 I/O 부담이 크므로,
트래픽 저점 시간대에 샘플링하고 보관 주기를 짧게 가져가는 것이 안전합니다.

### clinic.js로 누수 패턴 빠르게 확인

```bash
npx clinic doctor -- node server.js
# 부하 생성 후 종료하면 리포트 생성
```

clinic.js는 이벤트 루프 지연, CPU, 메모리 흐름을 함께 보여줘서
"CPU 병목"과 "메모리 누수"를 분리하는 데 유용합니다.

## 자주 나오는 원인 3가지

### 전역 캐시 무한 성장

- TTL 없는 Map/Object 캐시
- 키 정리 정책(LRU/만료) 부재

해결: 최대 크기 + TTL + 제거 메트릭(히트율/eviction) 함께 적용

### 이벤트 리스너 해제 누락

- 요청 단위 객체에 `on()`만 등록하고 `off()` 누락
- 재연결 로직에서 중복 리스너 누적

해결: 등록/해제 쌍을 코드리뷰 체크리스트에 포함

### 클로저가 큰 객체를 오래 붙잡는 구조

- 비동기 큐/타이머 콜백이 불필요한 상위 스코프 참조

해결: 필요한 필드만 복사하고, 작업 종료 시 참조를 명시적으로 끊기

## 운영 체크리스트: 수정 후 검증까지 완료해야 끝난다

### 배포 전

- 누수 재현 시나리오(요청 패턴/시간) 문서화
- 기준 메트릭(`heapUsed`, RSS, GC pause) 대시보드 고정

### 배포 후

- 동일 부하에서 힙 기준선 상승이 멈췄는지 확인
- OOMKilled/재시작 횟수, P95 지연시간 동시 관찰
- 24~72시간 추세로 회귀(regression) 여부 점검

## 요약

Node.js 메모리 누수 대응의 핵심은 다음 3단계입니다.

1. 누수 의심 지표를 먼저 정의하고
2. 동일 조건 스냅샷으로 증가 객체를 특정한 뒤
3. 수정 후 장시간 추세까지 검증한다

이 과정을 표준화하면 OOM 대응 속도뿐 아니라,
개발 블로그 문서의 신뢰도와 재현성도 함께 높일 수 있습니다.

## 내부 링크

- [/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html](/development/blog/seo/2026/03/05/http-request-timeout-and-fail-fast-guide.html)
- [/development/blog/seo/2026/03/06/npm-ci-lockfile-reproducible-build-guide.html](/development/blog/seo/2026/03/06/npm-ci-lockfile-reproducible-build-guide.html)
- [/development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html](/development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html)

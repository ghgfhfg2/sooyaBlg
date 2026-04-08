---
layout: post
title: "Node.js Semaphore 패턴 가이드: 공유 리소스 동시성 폭주를 안전하게 막는 법"
date: 2026-04-08 20:00:00 +0900
lang: ko
translation_key: nodejs-semaphore-pattern-shared-resource-concurrency-guide
permalink: /development/blog/seo/2026/04/08/nodejs-semaphore-pattern-shared-resource-concurrency-guide.html
alternates:
  ko: /development/blog/seo/2026/04/08/nodejs-semaphore-pattern-shared-resource-concurrency-guide.html
  x_default: /development/blog/seo/2026/04/08/nodejs-semaphore-pattern-shared-resource-concurrency-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, semaphore, concurrency-control, backpressure, resource-protection, backend, performance, resilience]
description: "Node.js에서 semaphore 패턴으로 DB 연결, 외부 API, 파일 처리 같은 공유 리소스의 동시성 폭주를 막는 방법과 timeout·queue·fallback 설계 포인트를 정리했습니다."
---

Node.js 서비스가 느려질 때 원인이 항상 CPU 부족인 것은 아닙니다.
오히려 실무에서는 **한정된 공유 리소스에 너무 많은 요청이 한꺼번에 몰리는 문제**가 더 자주 터집니다.
예를 들어 DB connection pool은 20개인데 동시에 300개의 쿼리가 몰리거나, 외부 API는 초당 처리량이 낮은데 애플리케이션이 무제한 병렬 호출을 보내는 식입니다.
이때 단순히 `Promise.all()`로 병렬성을 높이면 순간 처리량은 좋아 보여도, 실제로는 대기열 폭증·timeout 증가·tail latency 악화로 이어지기 쉽습니다.

이런 상황에서 유용한 기본 도구가 **semaphore 패턴**입니다.
핵심은 동시에 실행할 수 있는 작업 수를 정해 놓고, 그 수를 넘는 요청은 기다리게 하거나 아예 빠르게 거절하는 것입니다.
이 글에서는 Node.js에서 semaphore가 왜 필요한지, 어떤 리소스에 적용해야 하는지, queue timeout·fallback·관측까지 포함해 실무적으로 어떻게 설계하면 좋은지 정리합니다.

## Node.js Semaphore 패턴이 필요한 이유

### H3. 병목은 서버 전체가 아니라 특정 공유 리소스에서 먼저 발생하는 경우가 많다

Node.js 프로세스 하나가 멀쩡해 보여도, 내부적으로는 더 작은 병목이 먼저 한계에 도달할 수 있습니다.
대표적인 예시는 아래와 같습니다.

- DB connection pool 개수보다 많은 동시 쿼리
- 외부 결제·인증 API에 대한 과도한 병렬 호출
- S3, 파일 시스템, 이미지 처리 라이브러리 같은 I/O 집중 작업
- CPU 연산이 필요한 worker thread 풀의 제한된 슬롯

문제는 이런 리소스가 포화되면 단순히 "조금 느려지는" 수준에서 끝나지 않는다는 점입니다.
대기열이 길어질수록 응답 시간이 늘고, timeout이 증가하고, 재시도가 붙으면서 부하가 다시 증폭됩니다.
결국 작은 병목이 전체 서비스 장애로 번질 수 있습니다.

### H3. 무제한 병렬 실행은 짧게는 빨라 보여도 길게는 시스템을 더 불안정하게 만든다

개발 초기에 흔히 하는 실수가 있습니다.
처리량이 부족해 보이면 병렬 호출 수를 늘리는 것입니다.
하지만 병목 리소스가 고정돼 있다면 병렬 수를 늘릴수록 오히려 아래 문제가 생깁니다.

- 대기 시간 증가
- queue memory 사용량 증가
- downstream timeout 증가
- retry 폭증
- tail latency 악화

즉 semaphore는 성능을 일부러 늦추는 장치가 아니라, **시스템이 감당할 수 있는 수준 안에서만 일을 받도록 만드는 안전장치**에 가깝습니다.
이 점은 [Node.js Concurrency Limit 가이드](/development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)와도 닿아 있습니다.
다만 semaphore는 일반적인 promise pool보다 **특정 공유 리소스 슬롯을 보호한다**는 관점이 더 강합니다.

## 어떤 곳에 Semaphore를 적용하면 효과가 좋을까

### H3. DB, 외부 API, 파일 처리처럼 "병목 개수가 명확한 리소스"에 특히 잘 맞는다

semaphore는 한정된 슬롯이 비교적 분명한 곳에서 효과가 좋습니다.
예를 들면 아래와 같습니다.

- PostgreSQL/MySQL connection pool 앞단
- Redis를 포함한 특정 네트워크 의존성 호출 그룹
- 외부 SaaS API 호출 래퍼
- Sharp 같은 이미지 변환 작업
- worker threads로 넘기는 CPU 집약 작업

이런 리소스는 "한 번에 몇 개까지 처리할 수 있는가"를 대략 숫자로 정할 수 있습니다.
따라서 semaphore limit도 현실적인 기준으로 잡을 수 있습니다.

### H3. 모든 요청에 하나의 전역 semaphore를 두기보다, 리소스별로 분리하는 편이 안전하다

실무에서 중요한 포인트는 semaphore를 하나만 두지 않는 것입니다.
예를 들어 결제 API와 이미지 처리 작업이 같은 semaphore를 공유하면, 이미지 트래픽 급증이 결제까지 막아버릴 수 있습니다.
그래서 보통은 아래처럼 나눕니다.

- DB 조회용 semaphore
- 외부 파트너 API용 semaphore
- CPU 작업용 semaphore
- 비핵심 백그라운드 작업용 semaphore

이 분리는 [Node.js Bulkhead Pattern 가이드](/development/blog/seo/2026/03/29/nodejs-bulkhead-pattern-resource-isolation-guide.html)와 연결됩니다.
semaphore가 개별 슬롯 수를 통제한다면, bulkhead는 아예 **실패와 적체가 전파되지 않도록 경계를 나누는 설계**라고 볼 수 있습니다.

## Node.js에서 Semaphore를 구현하는 기본 방식

### H3. 슬롯을 획득한 작업만 실행하고, 끝나면 반드시 release 해야 한다

가장 단순한 semaphore의 규칙은 아래 3가지입니다.

1. 사용 가능한 슬롯 수를 정한다
2. 작업 시작 전 슬롯을 획득한다
3. 작업 종료 시 성공/실패와 무관하게 슬롯을 반환한다

직접 구현할 수도 있지만, 개념 이해를 위해 간단한 예시를 보면 구조가 명확합니다.

```js
class Semaphore {
  constructor(limit) {
    this.limit = limit;
    this.inUse = 0;
    this.waiters = [];
  }

  async acquire() {
    if (this.inUse < this.limit) {
      this.inUse += 1;
      return this._createRelease();
    }

    return new Promise((resolve) => {
      this.waiters.push(resolve);
    }).then(() => {
      this.inUse += 1;
      return this._createRelease();
    });
  }

  _createRelease() {
    let released = false;
    return () => {
      if (released) return;
      released = true;
      this.inUse -= 1;

      const next = this.waiters.shift();
      if (next) next();
    };
  }
}

const dbSemaphore = new Semaphore(20);

async function runProtectedQuery(task) {
  const release = await dbSemaphore.acquire();
  try {
    return await task();
  } finally {
    release();
  }
}
```

핵심은 `finally`에서 `release()`를 보장하는 것입니다.
이 부분이 빠지면 semaphore는 시간이 갈수록 슬롯을 잃어버리는 형태로 망가집니다.

### H3. semaphore 자체보다 "대기 정책"을 어떻게 둘지가 실무 품질을 가른다

단순 acquire/release만으로는 충분하지 않습니다.
중요한 건 기다리는 요청을 어떻게 다룰지입니다.
보통 선택지는 아래 세 가지입니다.

- 무한 대기
- 일정 시간까지만 대기 후 timeout
- 대기열이 길면 즉시 거절

실무에서는 첫 번째가 가장 위험합니다.
무한 대기는 순간적으로는 친절해 보여도, 장애 상황에서 요청을 끝없이 쌓아두며 메모리와 latency를 함께 악화시킵니다.
그래서 대부분은 **queue timeout** 또는 **max queue length**를 같이 둬야 합니다.
이 부분은 [Node.js Queue Timeout 가이드](/development/blog/seo/2026/04/06/nodejs-queue-timeout-max-wait-overload-control-guide.html)와 [Node.js Bounded Queue 가이드](/development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html)와 함께 설계하는 편이 좋습니다.

## Queue Timeout과 함께 써야 하는 이유

### H3. 기다리는 시간이 작업 실행 시간보다 길어지면, 이미 실패한 요청이나 다름없다

예를 들어 외부 API 실제 호출 시간은 150ms인데, semaphore 대기열에서 2초를 기다린 뒤 실행된다면 사용자는 이미 느리다고 느낍니다.
더 심한 경우 상위 요청 deadline이 먼저 끝나서 downstream 호출은 의미 없는 일이 됩니다.

그래서 semaphore를 도입할 때는 아래를 같이 정해야 합니다.

- 최대 대기 시간
- 대기열 최대 길이
- 상위 요청 deadline을 초과하면 취소할지
- 거절 시 어떤 fallback을 줄지

대기 시간이 너무 길어지는 순간 semaphore는 보호 장치가 아니라 **지연 증폭기**가 됩니다.

### H3. deadline-aware acquire가 있으면 불필요한 작업을 줄일 수 있다

좋은 구현은 작업이 슬롯을 기다리는 동안에도 상위 요청의 남은 시간을 확인합니다.
예를 들어 HTTP 요청 전체 deadline이 300ms인데 이미 250ms를 대기했다면, 이제 슬롯을 받아도 성공 확률이 거의 없습니다.
이때는 실행하지 않고 빨리 실패시키는 편이 낫습니다.

이 개념은 [Node.js Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와 자연스럽게 이어집니다.
semaphore는 단순 동시성 제한 장치가 아니라, **남은 시간 안에 처리 가능한 요청만 통과시키는 필터**여야 더 효과적입니다.

## 실패 처리와 Fallback을 같이 설계해야 한다

### H3. 핵심 요청과 비핵심 요청을 같은 방식으로 막으면 사용자 경험이 오히려 나빠질 수 있다

모든 요청을 동일하게 timeout 처리하면 시스템은 안전해질 수 있어도 사용자 경험은 거칠어집니다.
예를 들어 아래처럼 차등 전략을 둘 수 있습니다.

- 결제 승인: 짧게 대기 후 명확한 오류 반환
- 추천 영역 조회: 오래 기다리지 않고 빈 목록 또는 캐시 반환
- 이미지 썸네일 생성: 나중에 재시도 가능한 작업으로 전환
- 통계성 로그 적재: 과부하 시 드롭

즉 semaphore는 "무조건 막는다"가 아니라, **무엇을 우선 처리하고 무엇을 포기할지 결정하는 정책 레이어**와 함께 써야 합니다.
이 점은 [Node.js Load Shedding 가이드](/development/blog/seo/2026/04/07/nodejs-load-shedding-overload-protection-guide.html)와 [Node.js Graceful Degradation 가이드](/development/blog/seo/2026/03/24/nodejs-graceful-degradation-fallback-pattern-guide.html)와도 연결됩니다.

### H3. retry를 붙일수록 semaphore 바깥에서 다시 혼잡이 생길 수 있다

semaphore 하나만 있다고 해서 혼잡 문제가 사라지지는 않습니다.
상위 레이어가 timeout 후 재시도를 남발하면, semaphore 바깥 대기열이 계속 커질 수 있습니다.
특히 사용자 요청 1건이 아래처럼 증폭되기 쉽습니다.

- semaphore 대기 1회
- 실행 timeout 1회
- 재시도 1회
- 재시도도 다시 semaphore 대기

이런 구조에서는 semaphore limit를 낮춰도 전체 시스템은 계속 답답합니다.
그래서 retry는 [Node.js Retry Budget 가이드](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html)처럼 별도 예산으로 통제해야 합니다.

## 관측 없이 Semaphore를 튜닝하면 오히려 역효과가 날 수 있다

### H3. limit 숫자 하나보다 queue wait, rejection rate, downstream latency를 함께 봐야 한다

많이 하는 실수가 "현재 limit 20이니까 40으로 늘리면 더 빨라지겠지"라고 생각하는 것입니다.
하지만 실제로는 아래 지표를 같이 봐야 합니다.

- semaphore current in-use
- queue wait time p50/p95/p99
- queue timeout 비율
- immediate rejection 비율
- downstream 처리 시간 변화
- 전체 요청 성공률과 tail latency

만약 limit를 올린 뒤 downstream latency가 악화되고 queue wait가 비슷하다면, 병목을 완화한 것이 아니라 **병목 뒤에 더 많은 일을 밀어 넣은 것**일 수 있습니다.

### H3. 리소스 특성에 따라 고정 limit보다 동적 조정이 유리할 때도 있다

항상 고정 limit만이 정답은 아닙니다.
외부 API 상태가 시간대별로 달라지거나, DB 부하가 배치 작업 시간에만 급증한다면 동적 조정이 더 적합할 수 있습니다.
다만 처음부터 복잡한 adaptive limit로 가기보다 아래 순서가 현실적입니다.

1. 보수적인 고정 limit 설정
2. queue wait와 error rate 관측
3. 특정 구간에서만 병목이 심한지 확인
4. 필요 시 adaptive concurrency나 admission control 검토

즉 semaphore는 시작점으로 좋고, 이후에는 [Node.js Admission Control 가이드](/development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html)나 adaptive concurrency 같은 상위 전략으로 확장할 수 있습니다.

## Node.js Semaphore 도입 체크리스트

### H3. 적용 전 체크리스트

- 보호하려는 리소스가 무엇인지 명확한가
- 그 리소스의 실제 처리 한계가 대략 추정되는가
- 최대 대기 시간과 최대 queue 길이가 정해져 있는가
- 실패 시 fallback 또는 명확한 오류 응답이 있는가
- retry와 함께 요청 증폭이 일어나지 않는가

### H3. 적용 후 체크리스트

- queue wait p95가 허용 범위 안에 있는가
- timeout보다 queue 대기가 더 큰 문제가 아닌가
- rejection rate가 비즈니스적으로 감당 가능한가
- 비핵심 작업이 핵심 요청을 막고 있지 않은가
- 슬롯 수 변경 전후의 tail latency가 실제로 개선됐는가

## 마무리

Node.js에서 semaphore는 화려한 패턴은 아니지만, 공유 리소스 보호에는 아주 강력한 기본기입니다.
핵심은 단순히 동시 실행 수를 줄이는 것이 아니라, **감당 가능한 양만 처리하고 나머지는 기다리게 하거나 포기하게 만드는 기준을 명확히 두는 것**입니다.

DB, 외부 API, 파일 처리, worker thread 같은 병목 지점이 있다면 semaphore 하나만으로도 장애 확산을 꽤 잘 막을 수 있습니다.
다만 실무에서는 semaphore만 두고 끝내지 말고, queue timeout·bounded queue·retry budget·fallback까지 같이 묶어서 설계해야 진짜 효과가 납니다.

관련해서 함께 보면 좋은 글:

- [Node.js Concurrency Limit 가이드: Promise Pool로 과도한 병렬 실행을 막는 법](/development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)
- [Node.js Queue Timeout 가이드: 오래 기다리게 하지 않는 Max Wait 설계법](/development/blog/seo/2026/04/06/nodejs-queue-timeout-max-wait-overload-control-guide.html)
- [Node.js Bounded Queue 가이드: 대기열 길이 제한으로 과부하를 늦기 전에 차단하는 법](/development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html)

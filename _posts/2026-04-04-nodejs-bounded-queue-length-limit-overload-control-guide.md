---
layout: post
title: "Node.js Bounded Queue 가이드: 무한 대기열 대신 길이를 제한해 과부하를 막는 방법"
date: 2026-04-04 08:00:00 +0900
lang: ko
translation_key: nodejs-bounded-queue-length-limit-overload-control-guide
permalink: /development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html
alternates:
  ko: /development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html
  x_default: /development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, bounded-queue, queue, overload-control, backpressure, concurrency, resilience, performance]
description: "Node.js 서비스에서 bounded queue를 적용해 무한 대기열로 인한 메모리 증가와 지연 확산을 막고, 감당 가능한 수준으로 작업량을 제어하는 실무 방법을 정리했습니다."
---

트래픽이 늘어날 때 많은 서비스는 처음에는 버팁니다.
문제는 버티는 방식이 종종 건강하지 않다는 점입니다.
요청을 바로 실패시키지 않는 대신, 내부 큐에 계속 쌓아 두면서 **느리게 무너지는 상태**로 들어가기 쉽습니다.
이 단계에서는 에러율이 아직 낮아 보여도, 응답시간은 길어지고 메모리는 늘어나며, 결국 timeout과 retry가 겹쳐 더 큰 장애로 이어집니다.

이럴 때 중요한 개념이 **bounded queue**입니다.
큐를 아예 없애자는 뜻이 아니라, **큐 길이에 상한을 두고 시스템이 감당 가능한 범위만 대기시키자**는 접근입니다.
이 글에서는 Node.js 환경에서 bounded queue가 왜 필요한지, priority queue나 concurrency limit와 무엇이 다른지, 그리고 운영에서 어떤 기준으로 큐 길이를 제한해야 하는지 실무 관점에서 정리합니다.

## Node.js Bounded Queue가 왜 필요한가

### H3. 무한 대기열은 장애를 숨기고 늦게 폭발시키는 경우가 많다

시스템이 과부하를 받을 때 가장 위험한 상태 중 하나는 “아직 받고는 있다”는 착시입니다.
입구에서 거절하지 않고 모든 작업을 큐에 계속 쌓아 두면 처음에는 안정적으로 보일 수 있습니다.
하지만 실제로는 아래 문제가 누적됩니다.

- 요청 대기 시간이 계속 길어짐
- 메모리 사용량이 증가함
- 이미 가치가 없어진 작업도 계속 남아 있음
- timeout 이후에도 서버 안에서는 낡은 작업이 계속 실행됨
- retry가 새로운 작업을 더 밀어 넣어 큐를 악화시킴

즉 unbounded queue는 문제를 해결하는 장치가 아니라, 종종 **문제를 안 보이게 잠깐 덮는 장치**가 됩니다.
나중에 한꺼번에 터질 뿐입니다.
이 점은 [Node.js Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)에서 설명한 deadline 개념과도 연결됩니다.
이미 타임아웃이 지난 요청을 계속 큐에 붙잡아 두는 것은 시스템 입장에서도 손해입니다.

### H3. 처리율보다 유입 속도가 빠르면 큐는 결국 무한히 늘어난다

큐가 안전해 보이는 이유는 작업을 당장 실행하지 않기 때문입니다.
하지만 처리율보다 유입 속도가 높은 상태가 지속되면 결과는 단순합니다.
큐 길이가 계속 늘어납니다.

예를 들어 워커가 초당 50개 작업을 안정적으로 처리할 수 있는데, 초당 80개가 들어오면 매초 30개씩 밀립니다.
처음 1분은 버틸 수 있어도, 10분이 지나면 대기열은 이미 운영자가 감당하기 어려운 크기로 부풀어 있을 수 있습니다.
이때 필요한 것은 “더 많이 쌓아 두기”가 아니라 **어디까지 받을지 정하는 정책**입니다.

bounded queue는 바로 이 지점에서 의미가 있습니다.
시스템이 감당 가능한 길이까지만 대기열을 허용하고, 그 이상은 거절하거나 낮은 우선순위 작업부터 탈락시키는 방식입니다.

## Bounded Queue와 Concurrency Limit, Priority Queue는 무엇이 다를까

### H3. concurrency limit는 동시에 실행할 개수를 제한하고, bounded queue는 기다릴 수 있는 양을 제한한다

이 둘은 비슷해 보여도 제어하는 대상이 다릅니다.

- concurrency limit: 지금 당장 실행 중인 작업 수를 제한
- bounded queue: 아직 실행되지 않았지만 대기 중인 작업 수를 제한

예를 들어 동시에 10개만 실행하도록 제한해도, 대기열이 무한이면 10만 개 작업이 뒤에서 줄을 설 수 있습니다.
그러면 실행 수는 안정적이어도, 지연과 메모리 문제는 여전히 커질 수 있습니다.
그래서 실무에서는 concurrency limit만으로 충분하지 않은 경우가 많습니다.
[Node.js Concurrency Limit 가이드](/development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)에서 실행 폭을 줄였다면, bounded queue는 그 뒤쪽 대기열 길이까지 함께 관리하는 보완 장치라고 보면 이해가 쉽습니다.

### H3. priority queue는 순서를 정하고, bounded queue는 총량을 제한한다

priority queue는 중요한 작업을 먼저 처리하는 데 유리합니다.
예를 들어 결제 승인 작업과 통계 집계 작업이 함께 들어온다면, 결제를 앞쪽에 두는 식입니다.
하지만 priority queue만 있다고 해서 대기열이 안전해지는 것은 아닙니다.
전체 큐가 끝없이 길어지면 우선순위가 높은 작업도 결국 오래 기다리게 됩니다.

즉 아래처럼 보는 편이 정확합니다.

- priority queue: 어떤 작업을 먼저 할지 결정
- bounded queue: 전체적으로 얼마나 기다리게 할지 결정

실무에서는 둘을 함께 쓰는 경우가 많습니다.
이 구조는 [Node.js Priority Queue 가이드](/development/blog/seo/2026/04/02/nodejs-priority-queue-job-scheduling-backpressure-guide.html)와도 자연스럽게 이어집니다.
중요한 작업을 우선 배치하되, 큐 전체 길이에는 분명한 상한을 두어야 합니다.

## Node.js에서 Bounded Queue를 어디에 적용하면 좋을까

### H3. 배치 작업, 이미지 처리, 외부 API fan-out, 메시지 소비에서 특히 효과가 크다

bounded queue는 요청-응답 API뿐 아니라 비동기 작업 파이프라인에서도 중요합니다.
특히 아래 같은 작업에서 효과가 큽니다.

- 업로드 후 썸네일 생성, 인코딩, OCR 같은 파일 처리
- 외부 API를 대량 호출하는 fan-out 워커
- 이메일, 알림, webhook 발송 잡
- queue consumer가 메시지를 과도하게 가져오는 경우
- 관리자용 일괄 처리 배치

이런 작업은 보통 “나중에 하면 되니까 일단 받아 두자”는 유혹이 큽니다.
하지만 downstream가 느려지면 큐만 길어지고, 결국 오래된 작업이 쓸모를 잃거나 장애 반경이 커질 수 있습니다.
이때 bounded queue는 **유실보다 더 나쁜 무의미한 대기를 줄이는 전략**이 됩니다.

### H3. 사용자 요청 경로에서도 내부 비동기 큐는 반드시 길이를 관리해야 한다

HTTP 요청은 빠르게 202 Accepted를 주고, 내부에서 작업을 처리하는 아키텍처도 흔합니다.
이 방식 자체는 나쁘지 않습니다.
문제는 내부 작업 큐 길이를 제한하지 않으면, 겉으로는 “빠른 API”처럼 보여도 실제로는 처리 지연이 계속 늘어나는 상태가 된다는 점입니다.

예를 들면 아래 같은 징후가 나타납니다.

- 사용자는 요청이 접수됐다고 보지만 실제 완료는 수십 분 뒤에 일어남
- 재시도 요청이 중복 접수를 늘림
- 운영자는 워커가 바쁘기만 한데 처리 완료 수는 늘지 않는 상황을 보게 됨
- 오래된 작업이 최신 작업보다 더 먼저 자원을 차지함

이런 구조에서는 admission control과 bounded queue를 같이 봐야 합니다.
입구에서 너무 많이 받지 않는 기준은 [Node.js Admission Control 가이드](/development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html)에서 설명한 것처럼 중요하고, 그 뒤에서 **기다리게 둘 수 있는 양도 제한**해야 시스템이 예측 가능해집니다.

## Node.js에서 Bounded Queue를 어떻게 구현할까

### H3. 가장 단순한 시작점은 최대 대기 개수를 명시한 in-memory queue다

작은 서비스라면 먼저 애플리케이션 내부 큐에 상한을 두는 것만으로도 큰 개선이 됩니다.
핵심은 간단합니다.
실행 슬롯이 모두 찼을 때 아무 작업이나 계속 받지 말고, 큐 길이가 기준을 넘으면 즉시 거절하는 것입니다.

아래는 개념을 설명하기 위한 단순 예시입니다.

```js
class BoundedQueue {
  constructor({ concurrency, maxQueue }) {
    this.concurrency = concurrency;
    this.maxQueue = maxQueue;
    this.running = 0;
    this.queue = [];
  }

  async push(task) {
    if (this.running < this.concurrency) {
      return this.run(task);
    }

    if (this.queue.length >= this.maxQueue) {
      const error = new Error('Queue is full');
      error.code = 'QUEUE_FULL';
      throw error;
    }

    return new Promise((resolve, reject) => {
      this.queue.push({ task, resolve, reject, enqueuedAt: Date.now() });
    });
  }

  async run(task) {
    this.running += 1;
    try {
      return await task();
    } finally {
      this.running -= 1;
      this.drain();
    }
  }

  drain() {
    if (this.running >= this.concurrency) return;
    const next = this.queue.shift();
    if (!next) return;

    this.run(next.task).then(next.resolve).catch(next.reject);
  }
}
```

이 예시는 단순하지만 중요한 원칙을 보여 줍니다.
**실행 수와 대기 수를 분리해서 관리하고, 대기 수가 기준을 넘으면 즉시 실패시킨다**는 점입니다.

### H3. production에서는 큐 길이뿐 아니라 대기 시간 제한도 함께 둬야 한다

큐 길이 상한만 있어도 도움이 되지만, 그것만으로 충분하지 않은 경우가 많습니다.
왜냐하면 큐에 남아 있는 동안 작업 가치가 빠르게 떨어질 수 있기 때문입니다.
예를 들어 2초 안에 끝나야 의미 있는 추천 요청이 30초 뒤에 실행된다면, 기술적으로는 성공이어도 사용자 입장에서는 실패에 가깝습니다.

그래서 실무에서는 아래 기준을 함께 둡니다.

- 최대 대기 개수
- 최대 대기 시간
- 작업별 deadline 또는 expiresAt
- 오래된 작업 우선 탈락 여부
- low priority 작업 선제 제거 여부

예를 들면 큐에 들어간 지 3초가 넘은 작업은 실행 전에 폐기할 수 있습니다.
이 방식은 timeout budget과도 잘 맞습니다.
**이미 유효시간이 끝난 작업은 실행하지 않는 것**이 downstream 보호에 더 유리하기 때문입니다.

### H3. 분산 환경에서는 중앙 큐의 길이와 소비 속도를 함께 관찰해야 한다

워커가 여러 대인 환경에서는 단순 in-memory queue만으로 전체 상태를 판단하기 어렵습니다.
이 경우 Redis, SQS, Kafka 같은 중앙 큐를 쓸 수 있는데, 여기서도 bounded queue 사고방식은 그대로 유지됩니다.
중요한 것은 큐가 있다는 사실이 아니라, **어디까지 적체를 허용할지 운영 기준이 있느냐**입니다.

운영 관점에서는 아래 지표를 같이 봐야 합니다.

- queue depth
- oldest message age
- enqueue rate / dequeue rate
- 처리 성공률과 실패율
- 재시도 횟수
- 대기 후 폐기된 작업 수

큐 길이만 보고 안전하다고 판단하면 오해하기 쉽습니다.
메시지 수는 적어도 오래된 작업이 쌓여 있을 수 있고, 반대로 길이는 길어도 소비 속도가 충분하면 일시적 피크일 수 있습니다.
따라서 bounded queue 정책은 숫자 하나가 아니라 **길이, 시간, 처리율을 함께 보는 운영 규칙**으로 가져가는 편이 낫습니다.

## Bounded Queue 크기는 어떻게 정해야 할까

### H3. 메모리 여유와 허용 가능한 대기 시간을 함께 기준으로 잡아야 한다

maxQueue 값을 감으로 정하면 나중에 다시 장애로 돌아오기 쉽습니다.
보통은 아래 두 질문으로 시작하는 편이 좋습니다.

1. 이 작업은 최대 몇 초까지 기다려도 의미가 있는가
2. 작업 하나가 평균적으로 얼마나 많은 메모리와 downstream 자원을 소비하는가

예를 들어 워커 처리율이 초당 20개이고, 작업이 5초 이상 기다리면 무의미하다면 대기열 상한은 대략 100개 안팎에서 시작해 볼 수 있습니다.
물론 정확한 값은 작업 크기와 변동성에 따라 달라집니다.
중요한 것은 “많을수록 안전하다”가 아니라, **유효한 대기 범위 안에 머무는가**입니다.

### H3. 정상 상태가 아니라 장애 전조 구간에서 어떻게 보이는지 봐야 한다

튜닝은 평상시 평균값만 보고 하면 실패하기 쉽습니다.
bounded queue는 특히 피크 시간, downstream 지연, 재시도 폭증 구간에서 어떻게 동작하는지가 중요합니다.
아래 상황을 일부러 테스트해 보는 편이 좋습니다.

- 외부 API 응답이 평소보다 3배 느려질 때
- DB latency가 튀어 워커 처리율이 절반으로 줄 때
- retry가 급증해 enqueue rate가 갑자기 높아질 때
- low priority 작업이 동시에 몰릴 때

이때 확인할 질문은 단순합니다.
큐가 길어질 때 시스템이 천천히 질식하는가, 아니면 **예상 가능한 방식으로 제한하고 회복하는가**입니다.
후자가 되려면 bounded queue 기준과 alert 기준이 함께 있어야 합니다.

## Bounded Queue를 도입할 때 자주 하는 실수

### H3. 거절은 하는데 이유와 재시도 전략을 클라이언트에 설명하지 않는 경우

큐가 가득 찼을 때 무조건 500으로 응답하면 운영과 클라이언트 모두 혼란스러워집니다.
가능하면 429, 503 같은 의미 있는 상태코드와 함께 재시도 여부를 분명히 전달하는 편이 낫습니다.
또한 모든 작업이 자동 재시도 대상은 아니라는 점도 중요합니다.
무분별한 retry는 큐를 더 빠르게 다시 채웁니다.

### H3. high priority 작업과 best-effort 작업을 같은 큐에 섞는 경우

결제 승인, 로그인, 관리자 리포트, 썸네일 재생성 같은 작업이 같은 큐를 공유하면 우선순위가 낮은 작업이 중요한 작업의 지연을 키울 수 있습니다.
이 문제는 bulkhead나 priority queue와 함께 풀어야 합니다.
bounded queue는 총량 제어 장치이지, 모든 종류의 격리를 자동으로 해결해 주는 장치는 아닙니다.

### H3. 큐가 짧아졌다는 이유만으로 과부하가 해결됐다고 착각하는 경우

큐 상한을 낮추면 queue depth 그래프는 보기 좋아질 수 있습니다.
하지만 그것만으로 시스템이 건강해졌다고 볼 수는 없습니다.
실제로는 거절률이 올라갔을 수 있고, 중요한 요청까지 너무 공격적으로 탈락시켰을 수 있습니다.
그래서 아래 지표를 함께 봐야 합니다.

- queue full rejection rate
- 요청 유형별 성공률
- p95, p99 latency
- retry 유입량
- 사용자 체감 완료 시간

즉 bounded queue는 **숫자를 작게 만드는 장치가 아니라, 실패를 통제 가능한 형태로 바꾸는 장치**입니다.

## 실무 체크리스트

### H3. 도입 전에 먼저 정해야 할 질문

bounded queue를 실제 서비스에 넣기 전에 아래 질문부터 정리하면 시행착오를 많이 줄일 수 있습니다.

- 이 작업은 최대 몇 초까지 기다릴 수 있는가
- 큐가 가득 차면 거절할지, 오래된 작업을 버릴지, 저우선순위를 먼저 버릴지
- 클라이언트는 어떤 상태코드와 메시지를 받게 할지
- 재시도는 어디서, 몇 번까지 허용할지
- 어떤 지표를 alert 기준으로 삼을지

이 다섯 가지가 모호하면, 코드로 큐 상한을 넣어도 운영은 여전히 흔들립니다.

## 마무리

Node.js에서 bounded queue는 화려한 최적화 기법이라기보다, 시스템이 감당 가능한 현실을 코드와 운영 정책에 반영하는 방법에 가깝습니다.
무한 대기열은 친절해 보이지만, 실제로는 늦고 비싼 실패를 만들어 내는 경우가 많습니다.
반대로 bounded queue는 일부 요청을 더 일찍 거절하더라도, 전체 시스템을 더 예측 가능하고 회복 가능한 상태로 유지하게 도와줍니다.

이미 concurrency limit나 admission control을 적용했는데도 피크 시간 지연이 길게 남아 있다면, 이제는 **실행 수뿐 아니라 대기열 길이도 제한할 시점**일 가능성이 큽니다.
특히 오래 기다린 작업의 가치가 빠르게 떨어지는 서비스일수록 bounded queue의 효과는 생각보다 큽니다.

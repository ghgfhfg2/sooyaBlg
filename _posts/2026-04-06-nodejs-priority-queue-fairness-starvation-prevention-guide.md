---
layout: post
title: "Node.js Priority Queue 가이드: 핵심 요청을 먼저 처리하고 Starvation을 막는 설계법"
date: 2026-04-06 20:00:00 +0900
lang: ko
translation_key: nodejs-priority-queue-fairness-starvation-prevention-guide
permalink: /development/blog/seo/2026/04/06/nodejs-priority-queue-fairness-starvation-prevention-guide.html
alternates:
  ko: /development/blog/seo/2026/04/06/nodejs-priority-queue-fairness-starvation-prevention-guide.html
  x_default: /development/blog/seo/2026/04/06/nodejs-priority-queue-fairness-starvation-prevention-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, priority-queue, fairness, starvation, overload-control, queueing, resilience, backend]
description: "Node.js 서비스에서 priority queue를 적용해 핵심 요청을 먼저 처리하고 starvation을 막는 방법, fairness 설계, aging 전략, 실무 운영 지표를 정리했습니다."
---

트래픽이 몰릴 때 모든 요청을 똑같이 취급하면 생각보다 빨리 문제가 커집니다.
결제 승인, 로그인, 웹훅 처리처럼 **지금 바로 처리돼야 하는 요청**과 배치성 집계, 백오피스 조회, 낮은 중요도의 추천 API가 같은 대기열에서 경쟁하면 결국 중요한 요청까지 함께 느려집니다.
이때 필요한 것이 **priority queue**입니다.

priority queue는 단순히 “급한 것 먼저”라는 감각적 규칙이 아닙니다.
어떤 요청에 더 많은 시스템 자원을 우선 배정할지, 그리고 그 과정에서 낮은 우선순위 요청이 영원히 굶지 않도록 어떻게 공정성을 유지할지 정하는 운영 설계입니다.
이 글에서는 Node.js 환경에서 priority queue가 왜 필요한지, starvation을 막기 위한 fairness 전략은 무엇인지, 그리고 concurrency limit·queue timeout·admission control과 어떻게 함께 써야 하는지 실무 관점에서 정리합니다.

## Node.js Priority Queue가 왜 필요한가

### H3. 모든 요청을 동일하게 처리하면 중요한 요청의 SLA가 쉽게 무너진다

서비스 입장에서는 모든 요청이 “들어온 일”이지만, 비즈니스 입장에서는 중요도가 다릅니다.
예를 들어 아래 요청이 한 워커 풀을 공유한다고 가정해 보겠습니다.

- 결제 승인 API
- 로그인 세션 갱신 API
- 관리자용 CSV 내보내기
- 홈 화면 추천 목록 재계산

이 네 가지를 FIFO 하나로 처리하면, 먼저 들어온 무거운 작업이 뒤에 온 핵심 요청까지 막아설 수 있습니다.
특히 피크 시간대에는 이런 현상이 더 심해집니다.
결과적으로 전체 처리량은 비슷해 보여도, **정작 지켜야 할 SLA는 가장 먼저 무너질 수 있습니다.**

그래서 priority queue의 목적은 단순한 편의가 아니라, **한정된 처리 자원을 중요한 경로에 먼저 배정하는 것**입니다.
이는 [Node.js Admission Control 가이드](/development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html)에서 다룬 “무엇을 받을지 결정하는 정책”과도 이어집니다.
받아들인 요청 안에서도 다시 **무엇을 먼저 처리할지**를 정해야 하기 때문입니다.

### H3. 과부하 상황에서는 처리 순서 자체가 보호 장치가 된다

보통 과부하 대응이라고 하면 동시성 제한이나 rate limit부터 떠올립니다.
물론 중요합니다.
하지만 이미 큐에 들어온 요청들 사이에서도 우선순위 구분이 없으면, 시스템은 여전히 중요한 트래픽을 충분히 보호하지 못할 수 있습니다.

예를 들어 같은 큐 길이 100이라도 구성은 완전히 다를 수 있습니다.

- 경우 A: 결제 5개, 로그인 10개, 백오피스 85개
- 경우 B: 결제 80개, 로그인 20개

표면적으로는 둘 다 큐 길이 100이지만, 실제 운영상 위험은 다릅니다.
priority queue가 없으면 경우 A에서도 백오피스 요청이 핵심 트래픽 앞을 막을 수 있습니다.
반대로 우선순위가 잘 설계돼 있으면, 낮은 중요도 작업 일부가 늦어지더라도 핵심 경로의 체감 품질은 지킬 가능성이 커집니다.

## Priority Queue와 단순 FIFO 큐는 무엇이 다른가

### H3. FIFO는 단순하지만, 비즈니스 중요도까지 반영하지는 못한다

FIFO는 먼저 들어온 작업을 먼저 처리하므로 구현이 간단하고 예측 가능성이 높습니다.
하지만 “먼저 왔다”는 기준만 있을 뿐, **어떤 요청이 더 중요하냐**는 질문에는 답하지 못합니다.
그래서 모든 요청 가치가 비슷한 내부 배치에는 적합할 수 있어도, 사용자가 직접 체감하는 실시간 API에는 한계가 있습니다.

priority queue는 보통 아래처럼 기준을 추가합니다.

- 사용자 영향도
- 매출/핵심 기능 연관성
- deadline 민감도
- 외부 시스템 의존성
- 작업 비용 대비 가치

즉 우선순위 큐는 자료구조 선택이 아니라, **서비스 정책을 실행 순서에 반영하는 방법**에 가깝습니다.

### H3. priority만 높이면 끝나는 게 아니라 fairness를 같이 설계해야 한다

많이 놓치는 부분이 있습니다.
priority queue를 넣었다고 자동으로 좋은 시스템이 되지는 않습니다.
높은 우선순위 요청이 계속 들어오면 낮은 우선순위 요청은 계속 뒤로 밀릴 수 있기 때문입니다.
이 현상을 **starvation**이라고 부릅니다.

예를 들어 아래처럼 구성하면 문제가 생깁니다.

- high: 결제, 로그인, 알림 전송
- medium: 일반 조회 API
- low: 관리자 리포트, 배치성 재계산

피크 시간대에 high 요청이 끊임없이 들어오면 low 큐는 사실상 처리되지 않을 수 있습니다.
따라서 priority queue를 도입할 때 핵심 질문은 두 가지입니다.

1. 무엇을 먼저 처리할 것인가?
2. 무엇이 영원히 밀리지 않게 만들 것인가?

즉 priority 설계와 fairness 설계는 반드시 같이 가야 합니다.

## Node.js에서 Priority Queue를 어떻게 설계할까

### H3. 우선순위는 2~4단계 정도로 단순하게 시작하는 편이 낫다

실무에서는 우선순위를 너무 세밀하게 나누면 오히려 운영이 어려워집니다.
팀마다 분류 기준이 흔들리고, 요청 경계가 모호해지고, 디버깅도 복잡해집니다.
그래서 처음에는 아래처럼 단순한 구조가 보통 더 낫습니다.

- high: 결제, 로그인, 사용자 액션 직결 API
- medium: 일반 사용자 조회 API
- low: 백오피스, 비동기성 리프레시, 무거운 집계

이 정도만으로도 대부분의 서비스는 효과를 봅니다.
중요한 것은 단계 수가 아니라, **분류 기준이 일관되게 적용되는지**입니다.
우선순위를 마음대로 올릴 수 있게 두면 결국 모든 팀이 자기 요청을 high라고 주장하게 되고, 시스템은 다시 평평해집니다.

### H3. priority와 concurrency limit을 함께 써야 효과가 크다

priority queue만 있고 실행 슬롯 제한이 없으면, 무거운 high 작업이 워커를 모두 점유해버릴 수 있습니다.
반대로 concurrency limit만 있고 priority가 없으면 중요한 요청이 뒤에서 기다릴 수 있습니다.
그래서 실무에서는 두 가지를 조합하는 편이 좋습니다.

예를 들면 이런 식입니다.

- 전체 concurrency 16
- high queue 전용 최소 슬롯 8 보장
- medium/low는 남는 슬롯 공유
- low는 큐 길이와 max wait를 더 엄격하게 제한

이 구조는 [Node.js Concurrency Limit 가이드](/development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)와 [Node.js Queue Timeout 가이드](/development/blog/seo/2026/04/06/nodejs-queue-timeout-max-wait-overload-control-guide.html)를 함께 적용하는 방식으로 이해하면 쉽습니다.
핵심은 단순합니다.
**무엇을 먼저 실행할지 정하고, 동시에 얼마나 실행할지도 제한해야 한다**는 것입니다.

## Node.js Priority Queue 구현 예시

### H3. 가장 단순한 형태는 우선순위별 큐를 두고 높은 큐부터 drain하는 방식이다

아래 예시는 high, medium, low 세 단계 큐를 두고, 실행 슬롯이 비면 높은 우선순위부터 꺼내는 단순한 예시입니다.
개념 설명용이므로 프로덕션에서는 취소 전파, 메트릭, deadline 확인, 테넌트 격리 등을 추가해야 합니다.

```js
class PriorityExecutor {
  constructor({ concurrency = 8, maxQueueByPriority = {} }) {
    this.concurrency = concurrency;
    this.maxQueueByPriority = {
      high: maxQueueByPriority.high ?? 100,
      medium: maxQueueByPriority.medium ?? 200,
      low: maxQueueByPriority.low ?? 100,
    };

    this.active = 0;
    this.queues = {
      high: [],
      medium: [],
      low: [],
    };
  }

  async run(task, priority = 'medium') {
    if (!this.queues[priority]) {
      throw new Error(`unknown priority: ${priority}`);
    }

    if (this.active < this.concurrency && this.isHigherQueueEmpty(priority)) {
      return this.execute(task);
    }

    if (this.queues[priority].length >= this.maxQueueByPriority[priority]) {
      throw new Error(`${priority} queue capacity exceeded`);
    }

    return new Promise((resolve, reject) => {
      this.queues[priority].push({ task, resolve, reject, queuedAt: Date.now() });
    });
  }

  isHigherQueueEmpty(priority) {
    if (priority === 'high') return true;
    if (priority === 'medium') return this.queues.high.length === 0;
    return this.queues.high.length === 0 && this.queues.medium.length === 0;
  }

  nextTask() {
    return this.queues.high.shift()
      ?? this.queues.medium.shift()
      ?? this.queues.low.shift();
  }

  async execute(task) {
    this.active += 1;
    try {
      return await task();
    } finally {
      this.active -= 1;
      this.drain();
    }
  }

  drain() {
    while (this.active < this.concurrency) {
      const next = this.nextTask();
      if (!next) break;

      this.execute(next.task)
        .then(next.resolve)
        .catch(next.reject);
    }
  }
}
```

이 방식의 장점은 구조가 명확하다는 점입니다.

- 높은 우선순위 요청을 먼저 실행할 수 있다
- 우선순위별 큐 길이 제한을 따로 둘 수 있다
- 정책 변경 지점이 비교적 분명하다

하지만 이 상태만으로는 starvation 문제가 남아 있습니다.
low 큐는 high 요청이 계속 들어오면 영원히 밀릴 수 있기 때문입니다.

### H3. aging이나 weighted fair scheduling을 넣어 starvation을 줄여야 한다

가장 실용적인 보완책 중 하나는 **aging**입니다.
대기 시간이 길어진 요청의 실질 우선순위를 조금씩 올려 주는 방식입니다.
즉 처음엔 low였더라도 너무 오래 기다리면 medium처럼 취급하게 만듭니다.

예를 들어 아래처럼 규칙을 둘 수 있습니다.

- low에서 3초 이상 기다리면 medium으로 승격
- medium에서 2초 이상 기다리면 high에 준하는 가중치 부여
- 또는 5번에 1번은 low 큐에서 반드시 하나 실행

이런 방식은 “무조건 high 먼저”보다 실무적으로 더 안정적입니다.
특히 운영자용 기능이나 배치성 작업도 완전히 버릴 수는 없는 환경에서 유용합니다.

간단한 weighted fair scheduling 예시는 아래처럼 생각할 수 있습니다.

```js
const schedule = ['high', 'high', 'high', 'medium', 'medium', 'low'];
let cursor = 0;

function pickNextQueue(queues) {
  for (let i = 0; i < schedule.length; i += 1) {
    const priority = schedule[cursor % schedule.length];
    cursor += 1;

    if (queues[priority].length > 0) {
      return priority;
    }
  }

  return null;
}
```

이 방식은 high를 더 자주 처리하되, medium과 low에도 주기적으로 기회를 줍니다.
절대적으로 완벽한 공정성은 아니지만, 많은 서비스에는 충분히 실용적입니다.

## 어떤 요청을 high priority로 둘 것인가

### H3. 매출과 사용자 세션 유지에 직결되는 경로부터 시작하는 편이 안전하다

priority queue를 도입할 때 가장 어려운 일은 분류입니다.
처음부터 복잡한 철학을 세우기보다, 아래처럼 **비즈니스 영향이 분명한 경로**부터 시작하는 편이 낫습니다.

high로 고려할 만한 예시는 다음과 같습니다.

- 결제 승인, 주문 확정
- 로그인, 토큰 갱신
- 웹훅 수신 후 짧은 ACK 응답
- 사용자 입력 직후 즉시 체감되는 API

반대로 low로 두기 쉬운 경로는 아래와 같습니다.

- 무거운 내보내기 작업
- 백오피스 통계 조회
- 재계산성 추천/집계 API
- 지연돼도 큰 문제 없는 동기화 작업

중요한 것은 “중요해 보인다”가 아니라, **늦었을 때 실제 손실이 큰가**입니다.
이 기준이 있어야 우선순위 인플레이션을 막을 수 있습니다.

### H3. 사용자 체감 latency와 작업 비용을 같이 봐야 한다

우선순위를 정할 때 흔한 실수는 사용자 체감만 보고 판단하는 것입니다.
하지만 아주 중요한 요청이라도 지나치게 무겁다면, high 큐를 독점해 전체 시스템을 해칠 수 있습니다.
그래서 아래 두 축을 함께 보는 편이 좋습니다.

- 늦었을 때의 비즈니스 손실
- 처리에 드는 평균 비용과 tail latency

필요하다면 high 안에서도 다시 분리할 수 있습니다.
예를 들어 “짧고 중요한 요청”과 “중요하지만 무거운 요청”을 같은 워커 풀에 섞지 않는 식입니다.
이 지점은 priority queue 하나만으로 해결되지 않을 수 있으므로, 전용 워커 풀이나 bulkhead 패턴을 함께 검토해야 합니다.

## 운영에서 꼭 봐야 할 지표

### H3. 큐 길이보다 priority별 wait time이 더 중요할 때가 많다

priority queue를 넣고도 효과가 없는 팀들은 종종 전체 큐 길이만 봅니다.
하지만 운영에서 더 중요한 것은 **각 우선순위가 실제로 얼마나 기다렸는가**입니다.

아래 지표를 함께 보는 편이 좋습니다.

- priority별 queue wait p50, p95, p99
- priority별 drop/timeout 비율
- high 요청의 SLA 달성률
- low 요청 starvation 비율 또는 최대 대기 시간
- priority별 실행 시간 분포와 active slot 점유율

특히 high SLA가 좋아졌더라도 low가 완전히 얼어붙었다면, 현재 정책은 보호는 했지만 공정성은 잃은 상태일 수 있습니다.

### H3. high queue가 항상 가득 차 있다면 분류 자체가 잘못됐을 수 있다

priority queue 도입 후 high 비중이 지나치게 커지면 시스템은 다시 평평해집니다.
이 경우 보통 아래 둘 중 하나입니다.

- high 기준이 너무 느슨하다
- high 요청 자체가 지나치게 많아 별도 격리가 필요하다

즉 high 큐가 만성적으로 포화라면 스케줄링 튜닝 이전에 **분류 정책을 다시 봐야 할 가능성**이 큽니다.
priority는 희소해야 의미가 있습니다.

## 실무 적용 체크리스트

### H3. 적용 전 확인할 것

- 정말 같은 워커 풀에서 서로 다른 중요도의 요청이 경쟁하고 있는가?
- high, medium, low 기준이 문서화돼 있는가?
- priority별 queue length, max wait, timeout 정책이 따로 있는가?
- starvation을 막기 위한 aging 또는 weighted fairness가 있는가?
- retry 정책이 high 큐를 다시 과부하시키지 않는가?

### H3. 적용 후 확인할 것

- high 요청 SLA가 실제로 개선됐는가?
- low 요청의 최대 대기 시간이 통제 가능한 수준인가?
- 특정 테넌트나 엔드포인트가 high를 독점하고 있지 않은가?
- high 큐가 상시 포화 상태라면 우선순위 분류가 과도하지 않은가?
- 큐 분리보다 워커 풀 분리가 더 필요한 구간은 없는가?

## 마무리

Node.js에서 priority queue는 단순한 자료구조 선택이 아닙니다.
**한정된 처리 자원을 어떤 요청에 먼저 줄 것인지 결정하는 운영 정책**입니다.
특히 과부하 상황에서는 평균 처리량보다 “무엇을 먼저 살릴 것인가”가 더 중요해질 때가 많습니다.

다만 priority만 높인다고 문제는 끝나지 않습니다.
starvation을 막기 위한 fairness, 우선순위별 queue timeout, concurrency limit, 필요하다면 워커 풀 분리까지 함께 설계해야 실제로 안정적인 시스템이 됩니다.

지금 운영 중인 서비스에서 중요한 요청과 덜 중요한 요청이 같은 줄에 서 있다면, 다음 최적화 포인트는 단순한 성능 튜닝이 아니라 **처리 순서의 재설계**일 가능성이 큽니다.
priority queue는 그 출발점이 될 수 있습니다.

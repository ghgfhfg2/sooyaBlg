---
layout: post
title: "Node.js Load Shedding 가이드: 과부하 상황에서 중요한 요청을 살리는 보호 설계"
date: 2026-03-31 20:00:00 +0900
lang: ko
translation_key: nodejs-load-shedding-overload-protection-guide
permalink: /development/blog/seo/2026/03/31/nodejs-load-shedding-overload-protection-guide.html
alternates:
  ko: /development/blog/seo/2026/03/31/nodejs-load-shedding-overload-protection-guide.html
  x_default: /development/blog/seo/2026/03/31/nodejs-load-shedding-overload-protection-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, load-shedding, overload-protection, backpressure, resilience, backend, availability]
description: "Node.js 서비스에서 load shedding을 적용해 과부하 상황에서도 핵심 요청을 보호하고 전체 장애 확산을 줄이는 실무 설계 방법을 정리했습니다."
---

트래픽이 몰릴 때 많은 팀이 먼저 하는 일은 서버를 더 늘리거나 timeout 숫자를 조정하는 것입니다.
물론 그것도 필요하지만, 실제 장애 구간에서는 **모든 요청을 끝까지 받아주려는 태도 자체가 시스템을 더 빨리 무너뜨리는 경우**가 많습니다.
CPU, worker, DB connection, 외부 API quota가 이미 한계에 가까운데도 모든 요청을 동일하게 처리하려 하면 결국 중요한 요청까지 같이 느려지고, tail latency가 길어지고, 재시도까지 겹치면서 전체 서비스가 흔들립니다.
이때 필요한 개념이 **load shedding**입니다.
이 글에서는 Node.js 서비스에서 load shedding이 왜 필요한지, 어떤 기준으로 요청을 버려야 하는지, backpressure·timeout budget·bulkhead와는 어떻게 연결되는지 실무 관점에서 정리합니다.

## Node.js load shedding이 왜 필요한가

### H3. 과부하 상황에서는 모든 요청을 동일하게 처리하려는 설계가 가장 위험하다

정상 상태에서는 들어온 요청을 최대한 많이 처리하는 것이 좋은 전략처럼 보입니다.
하지만 시스템이 한계에 가까워지면 이야기가 달라집니다.
그 시점부터는 요청을 더 받는 것이 처리량 증가로 이어지지 않고, 오히려 아래 같은 부작용으로 바뀝니다.

- event loop 지연 증가
- in-flight request 수 급증
- DB connection pool 고갈
- 외부 API timeout 증가
- 큐 적체와 재시도 폭증

즉 문제는 "요청이 많다"가 아니라 **감당 불가능한 요청까지 계속 받아서 시스템 전체 품질을 무너뜨린다**는 데 있습니다.
load shedding은 이 구간에서 일부 요청을 의도적으로 거절해 핵심 경로를 지키는 전략입니다.

### H3. 실패를 늦게 만드는 것보다 빨리 결정하는 편이 전체 가용성에 유리하다

과부하 상황에서 가장 나쁜 패턴은 느리게 실패하는 것입니다.
사용자는 결국 실패를 받는데, 그 전까지 서버는 메모리와 연결, worker 시간을 계속 소모합니다.
이런 구조는 단일 요청 하나보다 **동시에 대기 중인 수많은 요청** 때문에 더 위험해집니다.

그래서 load shedding의 핵심은 성공률 100%를 고집하는 것이 아니라, **지금 반드시 살려야 하는 요청의 성공 확률을 높이는 것**입니다.
로그인, 결제, 주문 확인 같은 핵심 요청을 지키기 위해 추천, 집계, 부가 위젯, 저우선순위 API를 빠르게 제한하는 판단이 필요합니다.

## load shedding은 무엇인가

### H3. 시스템 한계를 넘기기 전에 일부 요청을 의도적으로 거절하는 운영 전략이다

load shedding은 시스템이 감당 가능한 용량을 넘기기 시작할 때 일부 요청을 선택적으로 드롭하거나 빠르게 실패시키는 방식입니다.
중요한 점은 이것이 단순한 에러 처리 기법이 아니라는 점입니다.
load shedding은 **과부하를 전체 장애로 키우지 않기 위한 보호 장치**입니다.

예를 들어 아래 같은 조건에서 작동할 수 있습니다.

- 현재 in-flight request 수가 임계치를 초과함
- event loop lag가 기준 이상으로 증가함
- DB pool 대기 시간이 급증함
- downstream timeout 비율이 급격히 높아짐
- 남은 deadline budget이 거의 없음

이런 상황에서 모든 요청을 계속 처리하려 하기보다, 저우선순위 요청을 먼저 거절하거나 축소하는 편이 전체 시스템 안정성에 더 유리합니다.

### H3. 무작정 차단하는 것이 아니라 우선순위와 비용을 기준으로 버려야 한다

load shedding은 "트래픽 많으면 503" 같은 단순 규칙으로 끝나지 않습니다.
중요한 것은 **어떤 요청을 버려도 되는가**를 미리 정하는 것입니다.

실무에서는 보통 아래 기준을 함께 봅니다.

- 비즈니스 중요도: 결제/인증/주문 > 추천/배너/통계
- 비용: DB heavy query, fan-out API, 외부 API 호출이 큰 요청 우선 제한
- 복구 가능성: 나중에 재시도 가능한 작업인지 여부
- 사용자 체감: 부분 응답으로도 괜찮은지 여부

즉 load shedding은 시스템 레벨 기술이면서 동시에 제품 우선순위 정책이기도 합니다.

## Node.js에서 어디에 먼저 적용해야 할까

### H3. fan-out API와 외부 의존성이 큰 경로가 1차 대상이다

Node.js 서비스에서 load shedding 효과가 가장 큰 구간은 아래와 같습니다.

- 여러 downstream을 동시에 호출하는 aggregator API
- 추천, 랭킹, 통계, 검색 자동완성 같은 부가 기능 API
- 외부 결제/메시징/지도/LLM API를 호출하는 경로
- 관리자용 대량 조회나 export API

이런 경로는 요청 1건당 비용이 크고, tail latency가 길어질수록 다른 요청까지 함께 끌어내리기 쉽습니다.
그래서 핵심 트랜잭션과 분리해 먼저 제한하거나, 부분 응답으로 낮출 수 있어야 합니다.

### H3. 로그인·결제 같은 핵심 트랜잭션은 보호 대상으로 따로 다뤄야 한다

모든 API를 같은 큐, 같은 connection pool, 같은 concurrency limit 아래 두면 과부하 시 핵심 요청도 같이 죽습니다.
이때 필요한 관점이 "모든 요청을 최대한 처리"가 아니라 **핵심 요청을 끝까지 살리는 보호 설계**입니다.

예를 들어 아래처럼 구분할 수 있습니다.

- 인증/주문/결제 승인: 보호 대상, 높은 우선순위
- 마이페이지 집계/추천 위젯: 축소 가능
- 관리자 export/배치성 조회: 혼잡 시 지연 또는 차단
- 백오피스 리포트: 별도 경로 또는 시간대 분리

이 분리가 없으면 load shedding을 도입해도 비즈니스 가치가 낮은 요청 때문에 중요한 요청이 여전히 영향을 받습니다.

## Node.js load shedding 구현 아이디어

### H3. in-flight request 수 기준으로 빠르게 거절하는 기본 게이트를 둘 수 있다

가장 단순한 시작점은 현재 처리 중인 요청 수를 기준으로 제한하는 것입니다.
물론 이것만으로 충분하지는 않지만, 과부하 완화의 첫 번째 안전장치로는 꽤 실용적입니다.

```js
let inFlight = 0;
const MAX_IN_FLIGHT = 300;

function loadSheddingMiddleware(req, res, next) {
  if (inFlight >= MAX_IN_FLIGHT) {
    return res.status(503).json({
      message: 'server is busy, please retry later'
    });
  }

  inFlight += 1;

  res.on('finish', () => {
    inFlight -= 1;
  });

  res.on('close', () => {
    inFlight -= 1;
  });

  next();
}
```

실전에서는 `finish`와 `close`에서 중복 차감이 나지 않도록 보호 로직을 더 두는 편이 안전합니다.
그럼에도 핵심 아이디어는 단순합니다.
**이미 감당 가능한 동시 처리량을 넘겼다면, 그 뒤 요청은 늦게 죽이기보다 입구에서 빠르게 거절한다**는 것입니다.

### H3. 우선순위별로 다른 한도를 두면 중요한 요청을 더 잘 지킬 수 있다

load shedding은 전역 한도 하나보다, 우선순위별 한도를 나누는 편이 더 효과적입니다.
예를 들어 `/api/payments`와 `/api/recommendations`를 같은 규칙으로 처리하면 과부하 시 결제 요청까지 같이 밀릴 수 있습니다.

```js
const limits = {
  critical: { current: 0, max: 120 },
  normal: { current: 0, max: 150 },
  low: { current: 0, max: 40 }
};

function classifyRequest(req) {
  if (req.path.startsWith('/api/payments') || req.path.startsWith('/api/auth')) {
    return 'critical';
  }

  if (req.path.startsWith('/api/recommendations') || req.path.startsWith('/api/widgets')) {
    return 'low';
  }

  return 'normal';
}
```

이 접근은 단순하지만 강력합니다.
과부하가 왔을 때 가장 먼저 포기할 요청을 명확히 정할 수 있기 때문입니다.

### H3. event loop lag와 downstream 상태를 같이 보면 더 현실적인 보호가 가능하다

동시 요청 수만으로는 실제 시스템 압박을 충분히 설명하지 못할 때가 많습니다.
예를 들어 요청 수는 많지 않아도 특정 external API가 느려져 전체 worker가 오래 묶일 수 있습니다.
그래서 가능하면 아래 신호를 함께 보는 편이 좋습니다.

- event loop lag
- 평균/상위 percentile 응답 시간
- DB pool wait time
- downstream error rate
- timeout 증가율

즉 load shedding은 고정 숫자 하나보다 **실시간 건강 상태를 보고 보호 수준을 조절하는 운영 메커니즘**에 가깝습니다.

## backpressure, timeout budget, bulkhead와는 어떻게 연결될까

### H3. backpressure는 속도를 늦추고, load shedding은 넘친 요청을 버린다

둘은 비슷해 보이지만 역할이 다릅니다.
backpressure는 생산 속도를 소비 속도에 맞추도록 조절해 밀어넣는 양을 줄이는 전략입니다.
반면 load shedding은 이미 감당 가능한 수준을 넘긴 요청을 **받지 않거나 빨리 포기하는 전략**입니다.

그래서 실무에서는 둘을 함께 써야 합니다.
입력 제어 관점은 [Node.js Backpressure 가이드](/development/blog/seo/2026/03/30/nodejs-backpressure-overload-control-guide.html)에서 더 자세히 볼 수 있습니다.

### H3. timeout budget이 없으면 늦게 실패하는 요청이 load shedding 효과를 갉아먹는다

입구에서 일부 요청을 거절하더라도, 이미 들어온 요청이 끝까지 오래 붙잡고 있으면 효과가 제한됩니다.
특히 상위 요청이 사실상 끝난 뒤에도 하위 호출이 계속 돌면 worker와 connection은 계속 묶여 있습니다.
이런 문제는 deadline-aware 하지 않은 서비스에서 자주 나타납니다.

요청 전체 시간 예산을 관리하는 방법은 [Node.js Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와 함께 보면 연결이 잘 됩니다.
load shedding이 입구 보호라면, timeout budget은 **이미 들어온 요청이 자원을 오래 점유하지 않도록 만드는 내부 규율**에 가깝습니다.

### H3. bulkhead가 없으면 한 종류의 과부하가 전체 시스템으로 번지기 쉽다

load shedding만 있고 bulkhead가 없으면 특정 기능의 폭주가 전체 pool을 함께 잠식할 수 있습니다.
예를 들어 추천 API가 느려졌는데도 결제 API와 같은 worker, 같은 downstream pool을 공유하면 보호 효과가 반감됩니다.

리소스 격리 관점은 [Node.js Bulkhead Pattern 가이드](/development/blog/seo/2026/03/29/nodejs-bulkhead-pattern-resource-isolation-guide.html)에서 이어서 볼 수 있습니다.
핵심은 **버리는 판단(load shedding)** 과 **섞이지 않게 나누는 설계(bulkhead)** 가 같이 가야 한다는 점입니다.

## 운영할 때 꼭 봐야 할 지표

### H3. 성공률만 보면 늦고, 거절 품질과 핵심 경로 보호 정도를 함께 봐야 한다

load shedding은 일부 요청을 의도적으로 실패시키는 전략이기 때문에, 단순 성공률만 보면 오히려 나빠 보일 수 있습니다.
하지만 중요한 것은 전체 숫자보다 **무엇을 지켰는가**입니다.

실무에서는 아래 지표를 같이 보는 편이 좋습니다.

- 우선순위별 요청 성공률
- shed된 요청 비율과 대상 endpoint
- event loop lag 변화
- in-flight request 상한 도달 빈도
- 핵심 API와 비핵심 API의 latency 분리 추적
- 503 이후 클라이언트 retry 패턴

이 데이터가 있어야 "조금 버려서 전체를 지킨 것인지" 아니면 "아무 기준 없이 막고 있는 것인지"를 구분할 수 있습니다.

### H3. 클라이언트 재시도 정책까지 함께 맞추지 않으면 효과가 줄어든다

서버가 503으로 빨리 거절해도, 클라이언트가 즉시 무한 재시도하면 다시 같은 과부하가 반복됩니다.
그래서 load shedding은 서버 단독 기술이 아닙니다.
서버와 클라이언트가 아래 원칙을 함께 맞추는 것이 중요합니다.

- 재시도는 exponential backoff + jitter 적용
- 멱등성이 없는 쓰기 요청은 신중하게 재시도
- 저우선순위 요청은 사용자 상호작용 시점에만 재시도
- 가능하면 `Retry-After` 또는 명확한 에러 의미 제공

즉 빠르게 거절하는 것과 **무의미한 재시도를 줄이는 것**이 함께 설계돼야 진짜 보호 효과가 납니다.

## Node.js load shedding 적용 체크리스트

### H3. 아래 항목에 해당하면 과부하 보호 설계를 먼저 검토할 시점이다

다음 항목 중 여러 개가 반복된다면 load shedding을 우선순위에 올릴 만합니다.

- 트래픽 급증 시 전체 API latency가 같이 무너진다
- 비핵심 기능 지연이 로그인·결제 같은 핵심 기능까지 전파된다
- 503보다 timeout이 훨씬 많이 발생한다
- worker, connection pool, 외부 API quota가 동시에 바닥난다
- 과부하 구간에서 재시도 때문에 상황이 더 악화된다
- 일부 기능을 버려도 되는데 현재는 전부 같은 우선순위로 처리한다

load shedding은 장애를 완전히 없애는 기술이 아닙니다.
대신 **망가질 때 어디까지 지킬 것인가를 미리 결정하게 해주는 기술**입니다.

## 자주 묻는 질문

### H3. load shedding과 rate limiting은 같은가요?

비슷하지만 목적이 다릅니다.
rate limiting은 보통 사용자나 client 단위로 사용량을 제한하는 정책이고, load shedding은 **시스템 전체 건강 상태를 지키기 위해 과부하 시점에 요청을 거절하는 전략**에 가깝습니다.
둘은 함께 쓸 수 있습니다.

### H3. 모든 503이 load shedding이라고 봐도 되나요?

그렇지 않습니다.
단순 오류로 503이 나는 경우도 많습니다.
load shedding이라면 어떤 조건에서, 어떤 우선순위 요청을, 왜 거절했는지가 운영 정책으로 정의돼 있어야 합니다.

### H3. 핵심 요청도 결국 버려야 하는 순간이 있지 않나요?

있습니다.
다만 그 전 단계에서 저우선순위 요청을 먼저 줄이고, bulkhead로 격리하고, timeout budget으로 늦은 작업을 정리하면 핵심 요청이 버려지는 시점을 더 늦출 수 있습니다.
load shedding의 목표는 모든 핵심 요청을 영원히 보장하는 것이 아니라, **전체 붕괴 전에 중요한 기능을 최대한 오래 지키는 것**입니다.

## 마무리

Node.js 서비스에서 load shedding은 "트래픽이 많으면 어쩔 수 없이 일부 실패한다"는 체념이 아닙니다.
오히려 과부하가 왔을 때도 시스템이 무엇을 우선적으로 지켜야 하는지 명확히 드러내는 설계입니다.

모든 요청을 동일하게 끝까지 처리하려는 시스템은 평소에는 친절해 보여도, 장애 순간에는 가장 중요한 요청까지 같이 잃기 쉽습니다.
반대로 load shedding이 잘 설계된 시스템은 일부 요청을 포기하더라도 전체 서비스의 중심을 지켜냅니다.

트래픽이 늘수록 필요한 것은 무조건 더 많이 처리하는 힘만이 아닙니다.
**지금 무엇을 받아들이고 무엇을 버릴지 결정하는 힘**도 함께 필요합니다.
그 기준이 분명해질수록 과부하 상황의 서비스는 훨씬 덜 무너집니다.

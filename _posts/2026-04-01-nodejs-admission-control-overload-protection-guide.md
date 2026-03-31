---
layout: post
title: "Node.js Admission Control 가이드: 과부하 전에 요청을 선별해 시스템을 지키는 설계법"
date: 2026-04-01 08:00:00 +0900
lang: ko
translation_key: nodejs-admission-control-overload-protection-guide
permalink: /development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html
alternates:
  ko: /development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html
  x_default: /development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, admission-control, overload-protection, concurrency, resilience, backend, availability]
description: "Node.js 서비스에서 admission control을 적용해 감당 가능한 요청만 받아들이고, 과부하가 전체 장애로 번지기 전에 차단하는 실무 설계 방법을 정리했습니다."
---

트래픽이 급증할 때 많은 팀은 먼저 autoscaling, 캐시, 쿼리 최적화를 떠올립니다.
물론 다 중요합니다.
그런데 실제 장애 구간에서는 성능 최적화보다 먼저 필요한 것이 **지금 이 요청을 받아도 되는가를 결정하는 기준**인 경우가 많습니다.
이미 event loop가 밀리고, DB pool 대기가 길어지고, 외부 API latency가 튀기 시작했는데도 모든 요청을 계속 받아들이면 시스템은 처리 능력을 회복하기보다 더 깊게 잠깁니다.
이때 필요한 개념이 **admission control**입니다.
이 글에서는 Node.js 서비스에서 admission control이 왜 중요한지, rate limiting·load shedding과 무엇이 다른지, 어떤 지표를 기준으로 요청을 받거나 거절해야 하는지 실무 관점에서 정리합니다.

## Node.js admission control이 왜 중요한가

### H3. 과부하 문제는 처리 속도보다 입구 통제가 없을 때 더 크게 터진다

서비스 장애를 겪고 나면 흔히 "서버가 느렸다"고 정리합니다.
하지만 실제로는 느린 서버 자체보다, **이미 느려진 서버에 요청이 계속 쌓이도록 놔둔 구조**가 더 큰 문제인 경우가 많습니다.
특히 Node.js는 적은 프로세스로도 높은 처리량을 낼 수 있지만, 반대로 대기 중인 작업이 빠르게 불어나면 event loop 지연과 downstream 대기가 한꺼번에 악화되기 쉽습니다.

아래 같은 상황이 반복된다면 admission control을 검토할 시점입니다.

- latency가 오르는 순간 in-flight request 수가 급격히 증가함
- DB connection pool wait time이 급증함
- 외부 API timeout이 늘수록 내부 queue도 같이 길어짐
- 재시도 트래픽이 원래 트래픽 위에 겹쳐짐
- 핵심 요청과 비핵심 요청이 같은 입구로 무제한 유입됨

즉 admission control은 단순 최적화가 아니라, **처리 불가능한 부하를 입구에서 제한하는 운영 안전장치**입니다.

### H3. 받아놓고 늦게 실패하는 구조가 가장 비싸다

과부하 상황에서 제일 손해가 큰 패턴은 "일단 받는다"입니다.
받아놓은 요청이 결국 timeout으로 끝나더라도, 그 사이 CPU, 메모리, worker 시간, DB connection, 외부 API quota는 계속 소모됩니다.
그 결과는 보통 아래 순서로 나타납니다.

- 평균 latency보다 p95, p99 latency가 먼저 크게 튐
- 클라이언트 retry가 붙으면서 유입량이 더 증가함
- 이미 늦은 요청 때문에 새 요청도 같이 밀림
- 결국 중요한 요청까지 성공률이 무너짐

그래서 admission control의 핵심은 성공률 100%를 고집하는 것이 아니라, **받을 수 있는 양만 받아서 시스템 전체 품질을 지키는 것**입니다.

## admission control은 무엇인가

### H3. 시스템 건강 상태를 기준으로 요청 수락 여부를 결정하는 방식이다

admission control은 사용자별 사용량 제한만 다루는 rate limiting과 다릅니다.
핵심은 **현재 시스템이 이 요청을 추가로 감당할 수 있는가**를 보고 수락 여부를 결정하는 데 있습니다.

예를 들어 아래 신호를 함께 볼 수 있습니다.

- 현재 in-flight request 수
- endpoint별 concurrency 사용량
- event loop lag
- DB pool 대기 시간
- downstream error rate 또는 timeout 비율
- 요청의 우선순위와 예상 비용

즉 admission control은 "누가 얼마나 많이 보냈는가"보다, **지금 받아들이면 시스템 전체가 더 위험해지는가**를 판단하는 로직에 가깝습니다.

### H3. rate limiting과 load shedding 사이를 연결하는 실전 장치로 볼 수 있다

실무에서는 admission control, rate limiting, load shedding이 자주 함께 언급됩니다.
하지만 역할은 조금 다릅니다.

- rate limiting: 사용자·토큰·IP 단위로 사용량을 제한함
- admission control: 시스템 상태를 보고 요청을 받을지 결정함
- load shedding: 이미 과부하가 시작된 상태에서 일부 요청을 의도적으로 버림

admission control은 이 셋 사이에서 꽤 중요한 위치를 차지합니다.
과부하가 완전히 터진 뒤 버리는 것보다, **들어오는 시점에서 감당 가능한 요청만 선별하는 편이 훨씬 싸고 안정적**이기 때문입니다.

## Node.js에서 어디에 먼저 적용해야 할까

### H3. fan-out API와 고비용 endpoint가 1차 대상이다

admission control은 모든 API에 한 번에 넣기보다, 요청 1건당 비용이 큰 경로부터 적용하는 편이 현실적입니다.
대표적으로 아래 경로가 우선순위가 높습니다.

- 여러 downstream을 동시에 호출하는 aggregator API
- 검색, 추천, 랭킹처럼 fan-out 비용이 큰 API
- 외부 결제, 메시징, LLM API를 붙인 endpoint
- 대량 조회, export, 리포트 생성 API

이런 경로는 요청 수 자체보다 **요청 한 건이 오래 붙잡고 있는 자원량**이 더 문제입니다.
그래서 admission control을 붙였을 때 체감 효과가 큽니다.

### H3. 핵심 요청과 비핵심 요청을 같은 입구로 받지 않는 것이 중요하다

로그인, 결제 승인, 주문 생성 같은 핵심 트랜잭션이 추천 위젯, 통계 패널, 관리자 export와 같은 조건으로 들어오면 과부하 순간에 가장 먼저 같이 다칩니다.
실무에서는 아래처럼 우선순위를 먼저 나누는 편이 좋습니다.

- critical: 인증, 결제, 주문, 쓰기 트랜잭션
- normal: 일반 조회, 사용자 상호작용 API
- low: 추천, 배너, 부가 위젯, 백오피스 조회

이 구분이 있어야 admission control이 단순 차단이 아니라 **비즈니스 가치에 맞춘 보호 전략**이 됩니다.

## Node.js admission control 구현 아이디어

### H3. 가장 단순한 시작점은 전역 in-flight 제한이다

가장 빠르게 도입할 수 있는 방식은 현재 처리 중인 요청 수를 세고, 임계치를 넘으면 새 요청을 거절하는 것입니다.
완벽한 방법은 아니지만 입구 보호 장치로는 꽤 실용적입니다.

```js
let inFlight = 0;
const MAX_IN_FLIGHT = 250;

function admissionControl(req, res, next) {
  if (inFlight >= MAX_IN_FLIGHT) {
    return res.status(503).json({
      message: 'server is busy, please retry later'
    });
  }

  inFlight += 1;

  let released = false;
  const release = () => {
    if (released) return;
    released = true;
    inFlight -= 1;
  };

  res.on('finish', release);
  res.on('close', release);
  next();
}
```

핵심은 단순합니다.
**이미 처리 가능한 동시 작업 수를 넘겼다면, 늦게 실패시키지 말고 입구에서 빨리 결정한다**는 것입니다.

### H3. 우선순위별 슬롯을 분리하면 중요한 요청을 더 오래 지킬 수 있다

전역 한도 하나만 두면 저우선순위 요청이 임계치를 먼저 잠식해 핵심 요청까지 막을 수 있습니다.
그래서 실전에서는 우선순위별 슬롯을 따로 나누는 편이 훨씬 효과적입니다.

```js
const pools = {
  critical: { current: 0, max: 120 },
  normal: { current: 0, max: 100 },
  low: { current: 0, max: 30 }
};

function classify(req) {
  if (req.path.startsWith('/api/payments') || req.path.startsWith('/api/orders')) {
    return 'critical';
  }

  if (req.path.startsWith('/api/recommendations') || req.path.startsWith('/api/widgets')) {
    return 'low';
  }

  return 'normal';
}
```

이렇게 해두면 과부하 시점에도 critical 요청이 low 요청 때문에 함께 밀리는 상황을 줄일 수 있습니다.
입구에서의 분리는 작아 보여도 운영 안정성에는 꽤 큰 차이를 만듭니다.

### H3. event loop lag와 downstream 상태까지 보면 더 현실적인 제어가 가능하다

요청 수만으로는 시스템 압박을 정확히 설명하기 어려운 경우가 많습니다.
예를 들어 in-flight 수는 비슷한데, 특정 외부 API가 느려지면서 worker가 오래 묶이면 실제 체감은 훨씬 나빠질 수 있습니다.
그래서 가능하면 아래 지표를 admission control 조건에 함께 반영하는 편이 좋습니다.

- event loop lag가 특정 기준 이상이면 low priority 요청 거절
- DB pool wait time이 급증하면 read-heavy endpoint 제한
- 외부 API timeout 비율이 높아지면 fan-out 호출 축소
- 남은 deadline budget이 작으면 부가 호출 생략

결국 중요한 것은 요청 개수보다 **추가 요청 1건이 지금 얼마나 위험한가**를 보는 것입니다.

## rate limiting, load shedding, timeout budget과는 어떻게 연결될까

### H3. rate limiting은 남용을 줄이고 admission control은 붕괴를 막는다

rate limiting은 특정 사용자나 클라이언트가 너무 많은 요청을 보내는 상황에 효과적입니다.
하지만 모든 사용자가 정상적으로 조금씩만 요청해도, 시스템 자체가 이미 느려진 상태라면 rate limiting만으로는 부족할 수 있습니다.
그때 필요한 것이 admission control입니다.

API 보호 정책 자체는 [Node.js Rate Limiting 가이드](/development/blog/seo/2026/03/23/nodejs-rate-limiting-token-bucket-redis-api-protection-guide.html)와 함께 보면 더 잘 연결됩니다.
admission control은 abuse 방지보다 **시스템 생존성 확보**에 더 가깝습니다.

### H3. load shedding은 이미 넘친 요청을 버리고 admission control은 덜 넘치게 만든다

load shedding과 admission control은 비슷해 보이지만 타이밍이 다릅니다.
load shedding은 과부하가 이미 시작된 상태에서 일부 요청을 포기하는 전략이고, admission control은 그 전에 **입구에서 수락량 자체를 더 보수적으로 조절하는 전략**입니다.

과부하 시 버리는 기준이 궁금하면 [Node.js Load Shedding 가이드](/development/blog/seo/2026/03/31/nodejs-load-shedding-overload-protection-guide.html)도 같이 볼 만합니다.
실무에서는 둘 중 하나만 고르기보다, admission control로 먼저 완충하고 load shedding으로 마지막 방어선을 두는 편이 좋습니다.

### H3. timeout budget이 없으면 받아들인 요청이 너무 오래 자원을 붙잡는다

입구를 잘 통제해도, 이미 받아들인 요청이 끝없이 downstream을 기다리면 admission control 효과는 금방 약해집니다.
그래서 요청 수락 정책은 timeout budget과 함께 설계하는 편이 안전합니다.

요청 전체 시간 예산을 관리하는 방식은 [Node.js Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)에서 따로 정리했지만, 요지는 단순합니다.
**받아들인 요청도 남은 시간이 없으면 빨리 끝내야 다음 요청을 살릴 수 있다**는 것입니다.

## 운영에서 꼭 봐야 할 지표

### H3. 단순 503 개수보다 어떤 요청을 왜 막았는지가 중요하다

admission control을 도입하면 일부 요청은 의도적으로 거절됩니다.
그래서 "503이 늘었다"만 보면 실패처럼 보일 수 있습니다.
하지만 실제로 봐야 할 것은 전체 숫자보다 아래 항목입니다.

- 우선순위별 요청 성공률 변화
- critical endpoint의 p95, p99 latency 변화
- admission control로 거절된 요청 비율
- 어떤 사유로 거절됐는지(event loop lag, in-flight, DB wait 등)
- retry 이후 재유입 비율
- 거절 이후 시스템 회복 속도

즉 admission control의 평가는 거절 수 자체가 아니라, **무엇을 막아서 무엇을 지켰는가**로 해야 합니다.

### H3. 클라이언트 재시도 정책까지 같이 설계해야 효과가 커진다

서버가 admission control로 503을 내보내도, 클라이언트가 즉시 재시도하면 입구 통제가 쉽게 무력화됩니다.
그래서 서버와 클라이언트는 함께 아래 원칙을 맞추는 편이 좋습니다.

- retry에는 exponential backoff와 jitter 적용
- low priority 조회는 자동 재시도 횟수 축소
- 멱등성이 없는 쓰기 요청은 신중하게 재시도
- 가능하면 `Retry-After` 또는 명확한 에러 메시지 제공

입구에서 잘 막는 것만큼, **막힌 뒤 다시 어떻게 들어오는가**를 제어하는 것도 중요합니다.

## Node.js admission control 적용 체크리스트

### H3. 아래 항목이 자주 보이면 입구 제어를 설계할 시점이다

다음 항목 중 여러 개가 반복되면 admission control이 꽤 높은 우선순위가 됩니다.

- 트래픽 급증 시 timeout보다 대기 시간이 먼저 늘어난다
- 핵심 요청과 비핵심 요청이 같은 concurrency pool을 쓴다
- 평균 latency는 괜찮은데 p99 latency가 자주 튄다
- DB pool, 외부 API, worker 슬롯이 동시에 포화된다
- 클라이언트 retry가 원래 부하를 더 증폭시킨다
- 일부 요청은 과감히 버려도 되는데 지금은 전부 동일하게 받는다

admission control은 화려한 최적화처럼 보이지 않을 수 있습니다.
하지만 운영 관점에서는 **안 받아야 할 요청을 안 받는 힘**이 시스템을 꽤 오래 버티게 만듭니다.

## 자주 묻는 질문

### H3. admission control과 rate limiting은 같은 의미인가요?

같지 않습니다.
rate limiting은 보통 사용자나 클라이언트 단위의 사용량 제한이고, admission control은 **현재 시스템 건강 상태를 기준으로 요청 수락 여부를 정하는 방식**입니다.
둘은 함께 쓰는 편이 일반적입니다.

### H3. admission control이 있으면 autoscaling은 필요 없나요?

여전히 필요합니다.
autoscaling은 용량을 늘리는 전략이고, admission control은 **용량 한계를 넘는 순간 전체 붕괴를 막는 전략**입니다.
서로 대체 관계라기보다 보완 관계에 가깝습니다.

### H3. 모든 503을 admission control로 처리해도 괜찮을까요?

권장하지 않습니다.
어떤 조건에서 어떤 우선순위 요청을 거절할지, 그리고 클라이언트가 어떻게 대응해야 할지를 함께 정의해야 합니다.
아무 기준 없는 503 남발은 보호 장치가 아니라 혼란만 키울 수 있습니다.

## 마무리

Node.js 서비스에서 admission control은 성능 튜닝의 마지막 디테일이 아닙니다.
오히려 과부하 상황에서 시스템이 스스로를 지키기 위해 가장 먼저 가져야 할 태도에 가깝습니다.

문제가 생길 때마다 서버를 더 늘리거나 timeout 숫자만 조정하면 일시적으로는 버틸 수 있습니다.
하지만 부하가 더 커지면 결국 다시 같은 문제를 만납니다.
그때 필요한 것은 "최대한 다 받자"가 아니라, **지금 받아야 할 요청만 정확히 받자**는 기준입니다.

입구에서의 한 번의 보수적인 결정이, 뒤쪽에서 수백 개의 느린 실패를 막는 경우가 꽤 많습니다.
그래서 admission control은 차갑게 보이지만, 결과적으로는 가장 중요한 요청을 살리는 친절한 설계이기도 합니다.

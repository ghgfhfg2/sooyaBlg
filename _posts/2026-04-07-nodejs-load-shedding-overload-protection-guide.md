---
layout: post
title: "Node.js Load Shedding 가이드: 과부하 순간 일부 요청을 버려 전체 서비스를 지키는 법"
date: 2026-04-07 08:00:00 +0900
lang: ko
translation_key: nodejs-load-shedding-overload-protection-guide
permalink: /development/blog/seo/2026/04/07/nodejs-load-shedding-overload-protection-guide.html
alternates:
  ko: /development/blog/seo/2026/04/07/nodejs-load-shedding-overload-protection-guide.html
  x_default: /development/blog/seo/2026/04/07/nodejs-load-shedding-overload-protection-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, load-shedding, overload-control, resilience, backpressure, backend, admission-control, latency]
description: "Node.js 서비스에서 load shedding을 적용해 과부하 순간 일부 요청을 빠르게 거절하고 핵심 기능의 안정성을 지키는 설계 방법과 운영 지표를 정리했습니다."
---

트래픽이 갑자기 몰릴 때 많은 서비스는 끝까지 버텨 보려 합니다.
들어오는 요청을 최대한 다 받아서 처리하려고 하고, 큐를 늘리고, timeout을 조금 더 길게 잡고, 워커를 더 바쁘게 돌립니다.
그런데 과부하 상황에서는 이런 태도가 오히려 전체 서비스를 더 빨리 무너뜨릴 수 있습니다.

이때 필요한 개념이 **load shedding**입니다.
쉽게 말하면 시스템이 감당할 수 없는 순간, 일부 요청을 빠르게 거절해 **전체 서비스의 생존성과 핵심 경로의 품질을 지키는 전략**입니다.
이 글에서는 Node.js 환경에서 load shedding이 왜 필요한지, rate limiting·admission control·queue timeout과 무엇이 다른지, 그리고 어떤 기준으로 버릴 요청을 정해야 하는지 실무 관점에서 정리합니다.

## Node.js Load Shedding이 왜 필요한가

### H3. 과부하에서는 모든 요청을 살리려는 시도가 전체 실패로 이어질 수 있다

평상시에는 요청을 많이 받는 것이 좋은 일처럼 보입니다.
하지만 시스템 용량을 넘는 순간부터는 이야기가 달라집니다.
처리할 수 없는 요청까지 계속 붙잡고 있으면 아래 문제가 동시에 커집니다.

- 응답 시간이 급격히 늘어난다
- 큐 대기 시간이 길어지며 tail latency가 폭증한다
- 메모리, 커넥션, CPU 같은 자원이 무의미하게 소모된다
- 이미 느린 요청이 추가 retry를 유발해 부하를 다시 키운다
- 핵심 요청과 비핵심 요청이 함께 무너진다

즉 과부하에서는 “더 많이 받는 것”이 아니라 **무엇을 버릴지 빨리 결정하는 것**이 더 중요할 수 있습니다.
이 지점에서 load shedding은 단순한 에러 처리 기법이 아니라, 시스템 전체를 보호하는 운영 규칙이 됩니다.

### H3. 핵심 목표는 평균 성공률이 아니라 중요한 기능의 생존이다

load shedding을 처음 들으면 “일부러 요청을 버린다”는 점 때문에 거부감이 생길 수 있습니다.
하지만 실제 운영에서는 모든 요청을 어설프게 살리려다 로그인, 결제, 핵심 조회까지 함께 망가지는 쪽이 훨씬 더 위험합니다.

예를 들어 아래 두 상황을 비교해 볼 수 있습니다.

- 모든 요청을 다 받다가 전체 응답이 10초 이상 느려지고 대부분 timeout으로 끝남
- 중요하지 않은 요청 일부는 즉시 503 또는 fallback으로 돌리고, 핵심 API는 정상 SLA를 유지함

대부분의 서비스는 두 번째가 훨씬 낫습니다.
그래서 load shedding의 본질은 드롭 자체가 아니라, **제한된 자원을 어디에 남겨 둘지 선택하는 것**입니다.

## Load Shedding과 Rate Limiting, Admission Control은 무엇이 다를까

### H3. rate limiting은 유입 속도를 제한하고, load shedding은 현재 상태를 보고 즉시 버린다

세 개념은 비슷해 보이지만 초점이 다릅니다.

- rate limiting: 일정 시간 동안 허용할 요청 수를 제한
- admission control: 지금 시스템이 받아도 되는지 여부를 입구에서 판단
- load shedding: 이미 과부하 징후가 보이면 일부 요청을 빠르게 탈락시킴

실무에서는 이 셋을 함께 쓰는 경우가 많습니다.
예를 들어 평상시에는 rate limit으로 과도한 호출을 막고, 순간적인 피크에서는 admission control로 큐 진입 자체를 제한하고, 임계치를 넘는 시점에는 load shedding으로 우선순위 낮은 요청을 즉시 거절하는 식입니다.

이 관점은 [Node.js Admission Control 가이드](/development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html)와 자연스럽게 이어집니다.
admission control이 “받을지 말지”를 정한다면, load shedding은 **지금 버려야 전체가 산다**는 판단을 더 공격적으로 적용하는 단계라고 볼 수 있습니다.

### H3. queue timeout은 오래 기다린 뒤 포기하고, load shedding은 애초에 늦을 요청을 빨리 막는다

queue timeout은 요청이 대기열에서 너무 오래 기다리면 실패시키는 방식입니다.
반면 load shedding은 **기다리게 해 봐야 의미가 없을 요청을 초반에 바로 걸러내는 전략**에 가깝습니다.

예를 들면 이런 차이가 있습니다.

- queue timeout: 200ms 기다렸는데도 슬롯이 없으면 실패
- load shedding: 현재 active worker, queue depth, CPU, event loop lag를 보고 지금은 받지 않음

둘은 경쟁 관계가 아니라 보완 관계입니다.
[Node.js Queue Timeout 가이드](/development/blog/seo/2026/04/06/nodejs-queue-timeout-max-wait-overload-control-guide.html)에서 다룬 것처럼 대기 시간을 제한하는 것도 중요하지만, 더 앞단에서 **가치가 낮거나 성공 가능성이 낮은 요청을 미리 잘라내는 것**이 전체 안정성에는 더 큰 도움이 될 수 있습니다.

## 어떤 요청을 버리고 어떤 요청을 살릴 것인가

### H3. 사용자 가치와 비즈니스 중요도로 먼저 나누는 편이 안전하다

load shedding을 적용할 때 가장 중요한 질문은 “무엇을 버릴 수 있는가”입니다.
여기서 감으로 판단하면 운영이 쉽게 꼬입니다.
실무에서는 아래 기준으로 먼저 나눠 보는 편이 좋습니다.

우선 보호할 가능성이 높은 요청:

- 로그인, 결제, 주문 확정 같은 핵심 경로
- 사용자 입력 직후 즉시 체감되는 액션
- 외부 시스템에 빠른 ACK가 필요한 웹훅 수신
- 실패 시 복구 비용이 큰 쓰기 요청

먼저 버리거나 축소할 수 있는 요청:

- 추천 목록, 랭킹, 부가 통계 조회
- 관리자용 무거운 리포트 API
- 실시간성이 약한 새로고침성 요청
- 캐시된 대체 응답이 가능한 읽기 API

핵심은 기술 관점만이 아니라 **늦거나 실패했을 때 비즈니스 손실이 얼마나 큰가**를 기준으로 정하는 것입니다.
이 기준이 있어야 모든 팀이 자기 요청을 “중요함”으로 분류하는 상황을 막을 수 있습니다.

### H3. 같은 high priority라도 비용이 너무 크면 별도 보호가 필요하다

중요한 요청이라고 해서 무조건 다 살릴 수 있는 것은 아닙니다.
매우 무거운 high priority 요청이 워커를 오래 점유하면, 오히려 짧고 중요한 요청을 막을 수 있습니다.
그래서 아래처럼 다시 나누는 편이 좋습니다.

- 짧고 중요한 요청
- 중요하지만 무거운 요청
- 늦어도 되는 요청
- 바로 버려도 되는 요청

이런 구분은 [Node.js Priority Queue 가이드](/development/blog/seo/2026/04/06/nodejs-priority-queue-fairness-starvation-prevention-guide.html)와도 연결됩니다.
즉 load shedding은 단순히 low를 버리는 게 아니라, **가치와 비용을 같이 보고 자원을 남기는 작업**입니다.

## Node.js에서 Load Shedding을 어떻게 설계할까

### H3. 한 가지 지표가 아니라 여러 신호를 함께 보는 편이 안정적이다

CPU 사용률 하나만 보고 요청을 버리기 시작하면 너무 늦거나 너무 이른 판단을 할 수 있습니다.
실무에서는 보통 여러 신호를 함께 봅니다.

대표적인 신호는 아래와 같습니다.

- active concurrency가 상한에 가까운가
- queue depth가 임계치를 넘었는가
- queue wait p95가 SLA를 잠식하는가
- event loop lag가 급격히 증가했는가
- 다운스트림 에러율과 timeout 비율이 함께 올라가는가

즉 load shedding은 “CPU 80%면 드롭”처럼 한 줄 규칙으로 끝내기보다, **지금 시스템이 정상 회복 가능한 상태인지**를 종합적으로 보는 편이 좋습니다.

### H3. 전면 차단보다 단계적 shedding이 운영에 더 잘 맞는다

처음부터 모든 비핵심 요청을 막기보다, 부하 수준에 따라 단계적으로 정책을 올리는 방식이 더 실용적입니다.
예를 들면 다음처럼 설계할 수 있습니다.

1. 정상 구간: 모든 요청 허용
2. 주의 구간: 저우선순위 요청에 더 짧은 queue timeout 적용
3. 위험 구간: 추천, 랭킹, 백오피스 조회를 즉시 503 또는 fallback 처리
4. 심각 구간: 핵심 경로만 허용하고 나머지는 차단

이렇게 하면 급격한 정책 변경으로 인한 혼란을 줄이고, 관측 가능한 임계치에 맞춰 운영하기가 쉬워집니다.
또한 fallback 캐시나 간소화 응답을 붙이기에도 좋습니다.

## Node.js Load Shedding 구현 예시

### H3. 입구에서 현재 부하 상태를 보고 빠르게 거절하는 단순한 형태부터 시작할 수 있다

아래 예시는 active request 수, queue depth, event loop lag를 기준으로 특정 요청을 거절하는 개념 예시입니다.
프로덕션에서는 메트릭 수집, 히스테리시스, priority별 정책, tenant별 격리 등을 추가로 고려해야 합니다.

```js
import { monitorEventLoopDelay } from 'node:perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

const state = {
  activeRequests: 0,
  queuedRequests: 0,
};

function shouldShedLoad({ priority = 'medium' }) {
  const eventLoopLagMs = histogram.mean / 1e6;

  const overload =
    state.activeRequests > 120 ||
    state.queuedRequests > 300 ||
    eventLoopLagMs > 80;

  if (!overload) {
    return false;
  }

  if (priority === 'high') {
    return state.activeRequests > 160 && eventLoopLagMs > 120;
  }

  return true;
}

async function requestHandler(req, res) {
  const priority = classifyPriority(req);

  if (shouldShedLoad({ priority })) {
    return res.status(503).json({
      message: 'server is temporarily busy, please retry shortly'
    });
  }

  state.activeRequests += 1;

  try {
    const data = await handleBusinessLogic(req);
    res.json(data);
  } finally {
    state.activeRequests -= 1;
  }
}

function classifyPriority(req) {
  if (req.path.startsWith('/payments')) return 'high';
  if (req.path.startsWith('/auth')) return 'high';
  if (req.path.startsWith('/reports')) return 'low';
  return 'medium';
}
```

이 예시의 핵심은 두 가지입니다.

- 부하가 높을 때 모든 요청을 동일하게 받지 않는다
- 요청 중요도에 따라 다른 shedding 기준을 적용한다

즉 load shedding은 “서비스가 바쁘니 모두 느리게 처리하자”가 아니라, **지금 받으면 더 큰 피해가 나는 요청을 빨리 정리하는 전략**입니다.

### H3. fallback 응답이 가능하면 단순 503보다 사용자 경험이 더 좋아질 수 있다

모든 shedding이 곧바로 에러 응답이어야 하는 것은 아닙니다.
특히 읽기 API는 degraded response가 더 나은 경우가 많습니다.

예를 들면 아래처럼 대응할 수 있습니다.

- 추천 API는 최신 계산 대신 마지막 캐시 결과 반환
- 대시보드 집계는 일부 항목을 생략한 간소화 응답 반환
- 검색 자동완성은 결과 수를 줄여 빠르게 응답
- 랭킹 API는 실시간 대신 5분 전 스냅샷 사용

이 방식은 사용자가 “완전히 실패했다”는 느낌을 덜 받게 해 줍니다.
다만 fallback 데이터의 신선도와 정확도 한계를 분명히 알고 있어야 하고, 핵심 쓰기 경로에는 함부로 적용하면 안 됩니다.

## 운영하면서 꼭 봐야 할 지표

### H3. shedding 비율만 보지 말고 무엇을 보호했는지도 같이 봐야 한다

load shedding을 적용하면 드롭 비율만 먼저 보게 됩니다.
하지만 더 중요한 것은 **그 대가로 무엇을 지켰는가**입니다.
아래 지표를 함께 봐야 판단이 정확해집니다.

- priority별 성공률과 latency p95, p99
- shedding 발생 비율과 시간대별 분포
- shedding 이후 retry 증가율
- 핵심 경로 SLA 유지 여부
- fallback 응답 사용 비율과 사용자 이탈 변화

예를 들어 shedding 비율이 조금 올랐더라도 결제와 로그인 SLA가 안정화됐다면 정책은 성공적일 수 있습니다.
반대로 드롭은 적은데 핵심 경로까지 계속 느리다면 현재 기준은 너무 약한 것입니다.

### H3. retry 폭증이 보이면 shedding 정책과 클라이언트 전략을 함께 봐야 한다

load shedding은 빠른 실패를 주기 때문에, 잘못하면 클라이언트가 이를 즉시 재시도하며 부하를 더 키울 수 있습니다.
그래서 아래를 함께 점검해야 합니다.

- 503 이후 재시도 간격에 jitter가 있는가
- 모바일 앱이나 프런트엔드가 자동 재시도를 과하게 하지 않는가
- 서버가 Retry-After 같은 힌트를 줄 필요가 있는가
- shedding이 특정 사용자나 특정 테넌트에 편향돼 있지 않은가

이 지점은 [Node.js Retry Budget 가이드](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html)와도 직접 연결됩니다.
load shedding은 빠른 실패 전략이고, retry budget은 **그 실패가 다시 폭주로 이어지지 않게 하는 제어 장치**입니다.

## 실무 적용 체크리스트

### H3. 적용 전 확인할 것

- 어떤 요청이 핵심 기능이고 어떤 요청이 비핵심인지 문서화돼 있는가?
- 과부하를 판단할 기준이 한 가지 지표에만 의존하지 않는가?
- 503 외에 fallback 응답으로 전환할 수 있는 경로가 있는가?
- shedding 이후 클라이언트 재시도 정책이 통제 가능한가?
- priority queue, queue timeout, admission control과 역할이 겹치지 않게 설계돼 있는가?

### H3. 적용 후 확인할 것

- shedding이 실제로 핵심 경로 SLA를 보호했는가?
- 너무 빨리 작동해 불필요한 실패를 만들고 있지는 않은가?
- 너무 늦게 작동해 이미 시스템이 질식한 뒤에야 반응하고 있지는 않은가?
- 특정 API 또는 특정 고객군만 과도하게 희생되고 있지 않은가?
- 부하가 내려간 뒤 정책이 자연스럽게 해제되고 있는가?

## 마무리

Node.js 서비스에서 과부하를 다루는 일은 단순히 서버를 더 열심히 일하게 만드는 문제가 아닙니다.
어떤 순간에는 더 많이 처리하려는 시도보다, **덜 중요한 요청을 과감히 버려 전체 시스템을 살리는 선택**이 훨씬 현명합니다.

load shedding은 실패를 만드는 기술이 아니라, 더 큰 실패를 막기 위한 보호 장치입니다.
특히 admission control, priority queue, queue timeout, retry budget과 함께 설계하면 과부하 순간에도 핵심 기능의 생존 가능성을 훨씬 높일 수 있습니다.

지금 운영 중인 서비스가 트래픽 피크에서 한꺼번에 느려진다면, 다음에 점검할 것은 단순한 스케일 업만이 아닐 수 있습니다.
어쩌면 정말 필요한 질문은 이것일 겁니다.
**이 순간 모든 요청을 끝까지 붙잡아야 하는가, 아니면 일부를 빨리 놓아줘야 전체가 사는가?**

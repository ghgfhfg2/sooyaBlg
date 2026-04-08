---
layout: post
title: "Node.js Leaky Bucket 가이드: Burst 트래픽을 평탄화해 다운스트림을 안정적으로 보호하는 법"
date: 2026-04-09 08:00:00 +0900
lang: ko
translation_key: nodejs-leaky-bucket-traffic-smoothing-rate-control-guide
permalink: /development/blog/seo/2026/04/09/nodejs-leaky-bucket-traffic-smoothing-rate-control-guide.html
alternates:
  ko: /development/blog/seo/2026/04/09/nodejs-leaky-bucket-traffic-smoothing-rate-control-guide.html
  x_default: /development/blog/seo/2026/04/09/nodejs-leaky-bucket-traffic-smoothing-rate-control-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, leaky-bucket, rate-limiting, traffic-shaping, backpressure, overload-control, backend, resilience]
description: "Node.js에서 leaky bucket으로 burst 트래픽을 평탄화해 다운스트림을 보호하는 방법과 queue 길이·drain rate·drop 정책을 함께 설계하는 실무 포인트를 정리했습니다."
---

트래픽 제어를 이야기할 때 많은 팀이 먼저 떠올리는 것은 rate limiting입니다.
하지만 실무에서는 단순히 "초당 몇 건까지 허용할까"만으로는 부족한 경우가 많습니다.
문제는 요청 총량보다도 **짧은 순간에 몰리는 burst**인 경우가 많기 때문입니다.
평균 QPS는 감당 가능한데도, 1~2초 사이에 요청이 한꺼번에 튀면서 DB, 외부 API, 작업 큐가 순간적으로 포화되는 식입니다.

이럴 때 검토할 수 있는 패턴이 **leaky bucket**입니다.
핵심은 들어오는 요청을 버킷에 모아 두고, 정해진 속도로만 흘려보내는 것입니다.
즉 "얼마나 많이 받느냐"보다 **얼마나 일정하게 내보내느냐**에 초점을 맞춘 제어 방식입니다.
이 글에서는 Node.js에서 leaky bucket이 왜 필요한지, token bucket과 무엇이 다른지, queue 길이·drain rate·drop 정책을 어떻게 잡아야 하는지 실무 관점에서 정리합니다.

## Node.js Leaky Bucket이 필요한 이유

### H3. 평균 트래픽보다 짧은 burst가 다운스트림을 먼저 무너뜨리는 경우가 많다

예를 들어 평소 초당 100건 수준의 API를 처리하는 서비스가 있다고 가정해 보겠습니다.
평균적으로는 충분히 감당 가능하지만, 앱 푸시 발송 직후나 배치 작업 종료 직후 1초 안에 800건이 몰리면 상황이 달라집니다.
애플리케이션 프로세스는 살아 있어도 아래 같은 병목이 먼저 터질 수 있습니다.

- DB connection pool 고갈
- 외부 API 429 증가
- queue 적체로 인한 latency 급증
- retry 폭증으로 인한 2차 부하

이런 상황에서는 단순 허용/차단만 하는 rate limiter보다, **들어온 burst를 조금 느리더라도 고르게 흘려보내는 장치**가 더 유용할 수 있습니다.
leaky bucket은 바로 이 지점을 겨냥합니다.

### H3. 처리량 제한이 아니라 트래픽 평탄화가 목적일 때 특히 잘 맞는다

token bucket은 "일정량의 burst를 허용하되 평균 속도를 제한"하는 데 강합니다.
반면 leaky bucket은 "들어온 요청을 일정한 속도로 배출"하는 데 더 가깝습니다.
그래서 아래 같은 상황에서 자주 검토합니다.

- 외부 파트너 API가 순간 burst에 취약한 경우
- 내부 작업 큐에 요청을 일정한 속도로만 넣고 싶은 경우
- DB 쓰기 작업을 갑자기 몰아넣지 않고 평탄화하고 싶은 경우
- webhook 처리량을 완전히 막지 않되 스파이크를 줄이고 싶은 경우

즉 leaky bucket은 차단기라기보다 **traffic shaper**에 더 가깝습니다.
이 점은 [Node.js Token Bucket 가이드](/development/blog/seo/2026/04/03/nodejs-token-bucket-rate-limiter-traffic-shaping-guide.html)와 비교해서 이해하면 더 명확합니다.

## Leaky Bucket과 Token Bucket은 무엇이 다를까

### H3. token bucket은 burst 허용에 강하고, leaky bucket은 출력 속도 고정에 강하다

두 패턴은 이름이 비슷해서 자주 헷갈리지만 목적이 다릅니다.

- **token bucket**: 토큰이 남아 있으면 burst를 허용하고, 평균적으로만 속도를 제한
- **leaky bucket**: 입력 burst와 무관하게 출력 속도를 비교적 일정하게 유지

예를 들어 초당 50건 처리 가능한 시스템이 있을 때, token bucket은 토큰이 쌓여 있다면 잠깐 150건을 허용할 수 있습니다.
반면 leaky bucket은 입력이 150건이든 500건이든, **배출 속도는 초당 50건 수준으로 유지**하려고 합니다.

그래서 사용자 경험 관점에서는 token bucket이 더 유연할 수 있지만, 다운스트림 보호 관점에서는 leaky bucket이 더 예측 가능한 경우가 많습니다.

### H3. burst를 허용해야 하는지, burst를 흡수해 평탄화해야 하는지 먼저 구분해야 한다

실무에서 중요한 질문은 이것입니다.
"우리 시스템은 순간 burst를 받아도 되는가, 아니면 반드시 평탄화해야 하는가?"

아래처럼 구분해 볼 수 있습니다.

- 로그인 API처럼 사용자 응답성이 더 중요하면 token bucket 쪽이 유리할 수 있음
- 결제 정산, 외부 파트너 동기화, 대량 webhook 후처리처럼 다운스트림 안정성이 중요하면 leaky bucket이 유리할 수 있음

즉 둘 중 무엇이 더 우월하냐가 아니라, **무엇을 보호하려는가**가 먼저입니다.
이 판단은 [Node.js Admission Control 가이드](/development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html)와도 연결됩니다.
모든 요청을 다 받는 것이 능사가 아니라, 어떤 요청을 어떤 속도로 통과시킬지 정책이 필요하기 때문입니다.

## Node.js에서 Leaky Bucket을 구현하는 기본 흐름

### H3. 입력은 queue에 쌓고, 배출은 고정된 drain rate로 처리한다

leaky bucket의 가장 단순한 구조는 아래와 같습니다.

1. 요청이나 작업을 bucket queue에 넣는다
2. 정해진 주기마다 일정 개수만 꺼낸다
3. queue가 가득 차면 대기시키거나 거절한다
4. 너무 오래 기다린 작업은 timeout 또는 drop 처리한다

개념 설명을 위한 단순 예시는 아래와 같습니다.

```js
class LeakyBucket {
  constructor({ capacity, drainPerTick, tickMs }) {
    this.capacity = capacity;
    this.drainPerTick = drainPerTick;
    this.tickMs = tickMs;
    this.queue = [];

    this.timer = setInterval(() => {
      for (let i = 0; i < this.drainPerTick; i += 1) {
        const item = this.queue.shift();
        if (!item) break;
        item.resolve(item.task());
      }
    }, this.tickMs);
  }

  submit(task) {
    if (this.queue.length >= this.capacity) {
      throw new Error('leaky bucket overflow');
    }

    return new Promise((resolve, reject) => {
      this.queue.push({
        task: async () => {
          try {
            return await task();
          } catch (error) {
            reject(error);
          }
        },
        resolve,
      });
    });
  }

  stop() {
    clearInterval(this.timer);
  }
}
```

실서비스에서는 이보다 더 많은 보호 장치가 필요합니다.
예를 들면 작업별 timeout, 상위 요청 취소 연동, priority 분리, queue 길이 관측 같은 것들입니다.
그래도 핵심은 단순합니다.
**입력은 순간적으로 늘어날 수 있지만, 출력은 의도적으로 일정하게 만든다**는 점입니다.

### H3. drain rate보다 queue 정책이 더 중요할 때가 많다

많은 팀이 leaky bucket을 도입하면서 drain rate 숫자만 조정합니다.
하지만 실제 장애는 대개 queue 정책에서 시작됩니다.
예를 들어 아래 조건을 정하지 않으면 시스템이 금방 지저분해집니다.

- 최대 queue 길이
- 최대 대기 시간
- overflow 시 즉시 거절할지 여부
- 같은 bucket에 넣을 작업 종류

queue가 무한히 늘어나면 leaky bucket은 보호 장치가 아니라 **지연 저장소**가 됩니다.
이 문제는 [Node.js Bounded Queue 가이드](/development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html)와 [Node.js Queue Timeout 가이드](/development/blog/seo/2026/04/06/nodejs-queue-timeout-max-wait-overload-control-guide.html)에서 다룬 내용과 그대로 이어집니다.

## 어떤 작업을 Leaky Bucket에 넣어야 할까

### H3. 외부 API, DB 쓰기, 메시지 발행처럼 순간 스파이크가 위험한 작업에 잘 맞는다

아래 같은 작업은 leaky bucket의 효과를 보기 좋습니다.

- 외부 SaaS API 호출
- 대량 webhook 후처리
- 검색 인덱스 업데이트 요청
- 쓰기 집중형 DB 작업
- 메시지 브로커 발행 전단의 평탄화 레이어

이런 작업은 응답을 몇 ms 더 늦추더라도, 순간 폭주를 줄이는 편이 전체 안정성에 유리한 경우가 많습니다.
특히 downstream이 자체 autoscaling 없이 고정 용량으로 운영될 때 효과가 큽니다.

### H3. 사용자 직접 응답 경로에는 무조건 넣기보다 비동기화 여부를 먼저 검토해야 한다

반대로 사용자가 클릭하고 바로 결과를 기다리는 경로에서는 leaky bucket이 답답하게 느껴질 수 있습니다.
예를 들어 결제 승인, 로그인, 핵심 검색 응답을 bucket 뒤에 오래 세워두면 사용자 경험이 급격히 나빠집니다.

이런 경우에는 아래 순서를 먼저 검토하는 편이 낫습니다.

1. 정말 동기 처리여야 하는가
2. 비동기 작업으로 분리할 수 있는가
3. 동기 경로라면 leaky bucket보다 semaphore나 admission control이 더 맞는가

즉 leaky bucket은 모든 곳에 넣는 범용 미들웨어가 아니라, **평탄화가 가치가 있는 경로에 선별 적용하는 장치**로 보는 편이 안전합니다.

## Queue 길이와 Drop 정책을 같이 설계해야 하는 이유

### H3. bucket이 가득 찼을 때의 정책이 없으면 지연만 늘고 보호 효과는 약해진다

leaky bucket이 있다고 해서 무한정 요청을 쌓아둘 수 있는 것은 아닙니다.
capacity를 넘는 순간 무엇을 할지 먼저 정해야 합니다.
보통 선택지는 아래와 같습니다.

- 즉시 429 또는 503 반환
- 비핵심 작업은 드롭
- 캐시된 응답이나 축소된 응답으로 fallback
- 비동기 큐로 넘기고 나중 처리

이 정책이 없으면 서비스는 실패하지 않는 대신, 모든 요청을 너무 늦게 처리하게 됩니다.
그건 사용자 입장에서는 사실상 실패와 비슷합니다.

### H3. 핵심 요청과 비핵심 요청을 같은 bucket에 섞지 않는 편이 좋다

가장 흔한 실수 중 하나는 모든 작업을 하나의 leaky bucket에 몰아넣는 것입니다.
그러면 비핵심 작업 burst가 핵심 작업의 대기 시간을 밀어 올립니다.
그래서 보통은 아래처럼 분리합니다.

- 결제/주문 후처리 bucket
- 추천/알림 bucket
- 백오피스 배치 bucket
- 외부 파트너별 bucket

이 구조는 [Node.js Priority Queue 가이드](/development/blog/seo/2026/04/06/nodejs-priority-queue-fairness-starvation-prevention-guide.html)와 [Node.js Bulkhead Pattern 가이드](/development/blog/seo/2026/03/29/nodejs-bulkhead-pattern-resource-isolation-guide.html)와도 맞물립니다.
핵심은 평탄화와 격리를 함께 설계하는 것입니다.

## 관측 없이 Leaky Bucket을 튜닝하면 안 되는 이유

### H3. drain rate 하나만 보지 말고 queue wait, overflow, downstream latency를 함께 봐야 한다

leaky bucket 도입 후에는 단순 처리량만 보면 안 됩니다.
최소한 아래 지표를 함께 봐야 합니다.

- current queue length
- queue wait time p50, p95, p99
- overflow 또는 drop 비율
- 실제 drain throughput
- downstream error rate와 latency 변화
- 사용자 요청 성공률 변화

예를 들어 overflow는 줄었는데 queue wait p99가 급증했다면, 보호는 되었지만 사용자 경험이 나빠졌을 수 있습니다.
반대로 queue wait는 안정적인데 downstream 429가 줄었다면, 평탄화가 제대로 작동한 것입니다.

### H3. drain rate는 다운스트림 실제 처리 능력보다 약간 보수적으로 잡는 편이 안전하다

drain rate를 너무 공격적으로 잡으면 leaky bucket을 둔 의미가 줄어듭니다.
반대로 너무 낮게 잡으면 불필요한 지연이 커집니다.
실무에서는 아래 순서가 현실적입니다.

1. downstream의 안정적인 처리량 기준을 잡는다
2. 그보다 약간 낮은 drain rate로 시작한다
3. queue wait와 overflow를 관측한다
4. 점진적으로 조정한다

처음부터 한 번에 최적값을 맞추려 하기보다, **보수적으로 시작하고 데이터로 다듬는 방식**이 훨씬 안전합니다.

## Node.js Leaky Bucket 도입 체크리스트

### H3. 적용 전 체크리스트

- 이 경로에서 진짜 필요한 것이 burst 허용인지, burst 평탄화인지 구분했는가
- 사용자 직접 응답 경로인지, 비동기 후처리 경로인지 구분했는가
- bucket capacity와 최대 대기 시간이 정의돼 있는가
- overflow 시 drop, reject, fallback 정책이 있는가
- 핵심 요청과 비핵심 요청이 분리돼 있는가

### H3. 적용 후 체크리스트

- downstream 429, timeout, saturation이 실제로 줄었는가
- queue wait p95, p99가 허용 범위 안에 있는가
- overflow 비율이 비즈니스적으로 감당 가능한가
- drain rate 조정 근거가 관측 데이터로 남아 있는가
- token bucket, semaphore, admission control과 역할이 겹치지 않는가

## 마무리

leaky bucket은 화려한 패턴은 아니지만, burst를 그대로 받아서 다운스트림에 쏟아붓지 않게 만드는 데 꽤 강력합니다.
핵심은 요청을 덜 받는 것이 아니라, **감당 가능한 속도로만 흘려보내는 것**입니다.
그래서 평균 트래픽은 괜찮은데 순간 스파이크가 문제인 시스템에서 특히 효과가 좋습니다.

다만 leaky bucket도 만능은 아닙니다.
queue 길이, 대기 시간, drop 정책 없이 도입하면 단지 느린 병목 저장소가 될 뿐입니다.
Node.js 서비스에서 이 패턴을 검토한다면, token bucket·bounded queue·priority queue·admission control과 함께 역할을 분명히 나눠 설계하는 쪽이 훨씬 안전합니다.

관련해서 함께 보면 좋은 글:

- [Node.js Token Bucket 가이드: Rate Limiter를 넘어서 Traffic Shaping까지 설계하는 법](/development/blog/seo/2026/04/03/nodejs-token-bucket-rate-limiter-traffic-shaping-guide.html)
- [Node.js Bounded Queue 가이드: 대기열 길이 제한으로 과부하를 늦기 전에 차단하는 법](/development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html)
- [Node.js Admission Control 가이드: 과부하 전에 요청을 받지 않는 것이 왜 더 안전한가](/development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html)

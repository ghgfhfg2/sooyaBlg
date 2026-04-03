---
layout: post
title: "Node.js Concurrency Limit 가이드: Promise Pool로 과부하를 막고 처리량을 안정화하는 방법"
date: 2026-04-03 20:00:00 +0900
lang: ko
translation_key: nodejs-concurrency-limit-promise-pool-overload-control-guide
permalink: /development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html
alternates:
  ko: /development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html
  x_default: /development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, concurrency-limit, promise-pool, overload-control, backpressure, queue, performance, resilience]
description: "Node.js에서 Promise Pool과 concurrency limit를 적용해 한꺼번에 몰리는 비동기 작업을 안정적으로 제어하고, 처리량은 유지하면서 과부하를 줄이는 실무 방법을 정리했습니다."
---

Node.js 서비스를 운영하다 보면 CPU보다 먼저 무너지는 것은 종종 **동시 실행되는 비동기 작업의 개수**입니다.
코드만 보면 `Promise.all()` 한 줄이라 깔끔한데, 실제 운영에서는 그 한 줄이 DB 커넥션을 고갈시키고, 외부 API quota를 태우고, 이벤트 루프 뒤쪽에서 대기열을 길게 만들어 응답 시간을 흔들어 버리는 경우가 적지 않습니다.
특히 배치 작업, fan-out API 호출, 파일 처리, 메시지 소비 로직에서는 “빠르게 병렬화했다”와 “안전하게 처리량을 높였다”가 전혀 같은 말이 아닙니다.

이럴 때 필요한 것이 **concurrency limit**입니다.
한 번에 실행할 작업 수를 의도적으로 제한해 시스템이 감당 가능한 범위 안에서만 병렬성을 사용하도록 만드는 방식입니다.
실무에서는 이를 Promise Pool, worker pool, semaphore 같은 형태로 구현합니다.
이 글에서는 Node.js에서 concurrency limit가 왜 중요한지, `Promise.all()`과 무엇이 다른지, 그리고 어떤 기준으로 제한값을 정해야 하는지를 실무 관점에서 정리합니다.

## Node.js Concurrency Limit가 필요한 이유

### H3. 비동기 함수가 많다고 무한 병렬이 공짜는 아니다

Node.js는 비동기 I/O에 강합니다.
그래서 많은 팀이 "어차피 await 안 막히니까 많이 돌릴수록 빠르다"는 감각으로 접근합니다.
하지만 실제로는 각 작업이 아래 자원을 함께 소비합니다.

- DB connection pool
- 외부 API 동시 요청 한도
- Redis 연결과 명령 처리량
- 파일 디스크립터
- 메모리 버퍼
- downstream 서비스의 처리 여유

즉 JavaScript 스레드가 하나라는 사실과 별개로, **뒤쪽 자원은 동시에 무한정 감당하지 못합니다**.
`Promise.all(items.map(doWork))`가 위험한 이유는 코드가 짧아서가 아니라, 실행 순간에 시스템에 “이 작업을 전부 지금 시작해도 된다”는 신호를 보내기 때문입니다.

### H3. 처리량을 높이려다 오히려 tail latency가 나빠지는 경우가 많다

동시성을 올리면 초반에는 처리량이 좋아 보일 수 있습니다.
하지만 어느 임계점을 넘기면 상황이 뒤집힙니다.

- DB 대기가 길어짐
- 외부 API timeout이 증가함
- 재시도 로직이 다시 부하를 키움
- 메모리 사용량이 치솟음
- 느린 작업이 전체 완료 시간을 끌어올림

이 구간부터는 더 많이 동시에 돌리는 것이 최적화가 아니라 **과부하 유도**가 됩니다.
이 문제는 [Node.js Backpressure 가이드](/development/blog/seo/2026/03/30/nodejs-backpressure-overload-control-guide.html)에서 다룬 흐름과도 연결됩니다.
입구에서 너무 많은 작업을 한꺼번에 밀어 넣으면, 내부에서는 결국 압력이 쌓입니다.
concurrency limit는 그 압력을 제어하는 가장 단순하고 효과적인 방법 중 하나입니다.

## Promise.all과 Promise Pool은 무엇이 다를까

### H3. Promise.all은 모든 작업을 즉시 시작하고, Promise Pool은 시작 속도를 조절한다

이 둘의 차이는 결과를 모으는 방식보다 **언제 작업을 시작하느냐**에 있습니다.

`Promise.all()`은 배열에 담긴 작업을 거의 동시에 시작합니다.
반면 Promise Pool은 예를 들어 최대 5개만 먼저 시작하고, 그중 하나가 끝나면 다음 작업을 시작합니다.
즉 아래처럼 사고할 수 있습니다.

- `Promise.all()`: 전체 작업을 한 번에 발사
- Promise Pool: 정해진 슬롯 수만큼만 실행
- 작업 완료 시 빈 슬롯에 다음 작업 투입

이 차이는 특히 작업 수가 수십 개가 아니라 수백, 수천 개가 될 때 크게 드러납니다.
작업을 모두 미리 시작하면 메모리와 연결 자원을 먼저 잠식합니다.
반대로 풀 기반 제어를 두면 **처리 속도는 유지하면서 시스템이 감당 가능한 폭 안에서만 병렬화**할 수 있습니다.

### H3. 순차 처리와 무제한 병렬 처리 사이의 현실적인 타협점이다

concurrency limit는 성능을 포기하는 전략이 아닙니다.
오히려 무제한 병렬 처리와 순차 처리 사이에서 **가장 효율적인 균형점**을 찾는 전략에 가깝습니다.

예를 들어 외부 API 호출 100건이 있다고 가정해 보겠습니다.

- 순차 처리: 가장 안전하지만 너무 느림
- 무제한 병렬 처리: 빠를 수 있지만 timeout과 429가 급증할 수 있음
- concurrency 5~20: 안정성과 속도의 균형 가능

실무에서는 세 번째 선택지가 가장 자주 살아남습니다.
전체 완료 시간이 조금 느려질 수는 있어도, 실패율과 변동성이 크게 줄어드는 경우가 많기 때문입니다.

## Node.js에서 어떤 작업에 concurrency limit를 걸어야 할까

### H3. fan-out API 호출, 배치, 메시지 소비, 파일 처리에서 특히 중요하다

아래 같은 작업은 concurrency limit 적용 효과가 큰 편입니다.

- 하나의 요청 안에서 외부 API를 여러 번 병렬 호출하는 로직
- CSV, 엑셀, 이미지 등 대량 파일 처리 파이프라인
- queue consumer가 메시지를 대량으로 가져와 처리하는 작업
- 관리자 배치에서 사용자별 후속 작업을 순회하는 코드
- 검색 색인, 썸네일 생성, 알림 발송 같은 백그라운드 잡

이런 작업은 대부분 “작업 수가 늘수록 빨라지겠지”라는 유혹이 있습니다.
하지만 실제 병목은 JavaScript가 아니라 DB, Redis, 네트워크, 외부 vendor 쪽에 있는 경우가 많습니다.
그래서 concurrency limit는 애플리케이션 내부 최적화이면서 동시에 **외부 의존성 보호 장치**이기도 합니다.

### H3. 한 요청 안의 병렬화와 워커 전체 병렬화를 구분해야 한다

실무에서 자주 놓치는 지점이 이것입니다.
예를 들어 요청 하나가 내부적으로 외부 API 10개를 병렬 호출한다고 해보겠습니다.
그리고 서버에는 동시에 50개의 요청이 들어옵니다.
그러면 이론상 외부 API에는 최대 500개의 동시 호출이 날아갈 수 있습니다.

즉 아래 두 층위를 따로 봐야 합니다.

- 요청 내부 concurrency
- 프로세스 전체 concurrency

요청 단위 제한만 걸어도 전체적으로는 여전히 과부하일 수 있습니다.
이 문제는 [Node.js Admission Control 가이드](/development/blog/seo/2026/04/01/nodejs-admission-control-overload-protection-guide.html)와 함께 봐야 이해가 쉽습니다.
입구에서 수용량을 조절하고, 내부에서는 각 작업군의 동시성을 제한해야 진짜 효과가 납니다.

## Node.js Concurrency Limit 구현 패턴

### H3. 가장 단순한 시작점은 직접 Promise Pool을 만드는 것이다

작은 서비스나 특정 배치 작업이라면 직접 구현한 Promise Pool만으로도 충분할 수 있습니다.
아래는 개념을 설명하기 위한 단순한 예시입니다.

```js
async function runWithConcurrency(items, limit, worker) {
  const results = new Array(items.length);
  let nextIndex = 0;

  async function runner() {
    while (true) {
      const currentIndex = nextIndex;
      nextIndex += 1;

      if (currentIndex >= items.length) {
        return;
      }

      results[currentIndex] = await worker(items[currentIndex], currentIndex);
    }
  }

  const runners = Array.from(
    { length: Math.min(limit, items.length) },
    () => runner()
  );

  await Promise.all(runners);
  return results;
}
```

사용 방식은 아래처럼 단순합니다.

```js
const users = await runWithConcurrency(userIds, 10, async (userId) => {
  return fetchUserProfile(userId);
});
```

핵심은 100개의 작업이 있어도 처음부터 100개를 다 시작하지 않는다는 점입니다.
한 번에 10개만 돌리고, 끝난 슬롯에 다음 작업을 넣습니다.

### H3. production에서는 실패 처리와 타임아웃 정책도 함께 설계해야 한다

실무에서는 단순히 limit만 두는 것으로 끝나지 않습니다.
아래 질문에 답할 수 있어야 합니다.

- 하나의 작업 실패가 전체 배치를 중단해야 하는가?
- 부분 성공을 허용할 것인가?
- 느린 작업 하나가 슬롯을 오래 점유하면 어떻게 할 것인가?
- timeout 후 재시도는 몇 번까지 허용할 것인가?
- 결과 순서를 보장해야 하는가?

예를 들어 외부 API fan-out에서는 실패한 항목만 별도로 모아 재시도 큐로 보내는 편이 더 낫습니다.
반면 정합성이 중요한 배치에서는 중간 실패 시 전체를 멈추고 원인을 먼저 보는 편이 맞을 수도 있습니다.
concurrency limit는 단독 기능이 아니라 **timeout, retry, idempotency, queue 설계와 묶여서 봐야 하는 운영 정책**입니다.

### H3. 검증된 라이브러리를 쓰면 운영 리스크를 줄이기 쉽다

직접 구현이 가능하더라도 production에서는 검증된 라이브러리를 선택하는 편이 안전한 경우가 많습니다.
예를 들면 다음과 같은 도구를 고려할 수 있습니다.

- `p-limit`
- `p-map`
- `Bottleneck`
- queue 시스템의 built-in concurrency 옵션

중요한 것은 라이브러리 이름보다 정책입니다.
무슨 도구를 쓰든 아래 정보가 명확해야 합니다.

- 동시 실행 수
- timeout 기준
- retry/backoff 여부
- 실패 시 중단 또는 계속 진행 정책
- metrics 수집 방식

## Concurrency Limit 값을 어떻게 정해야 할까

### H3. 감으로 정하지 말고 가장 약한 downstream 기준으로 시작하는 편이 낫다

limit 값을 정할 때 CPU 코어 수만 보는 것은 대개 충분하지 않습니다.
오히려 아래 자원이 더 직접적인 상한이 됩니다.

- DB connection pool 크기
- 외부 API vendor의 rate/concurrency 제한
- Redis 처리량과 네트워크 지연
- 파일 저장소 업로드 동시성 한도
- 워커 프로세스당 메모리 여유

예를 들어 DB connection pool이 20인데, 고비용 쿼리를 동반한 작업을 프로세스 하나에서 50개씩 병렬 처리하면 좋은 결과가 나오기 어렵습니다.
실무에서는 보통 **보수적인 값으로 시작한 뒤 모니터링을 통해 올리는 방식**이 덜 위험합니다.

### H3. 평균 응답시간보다 p95, p99와 실패율을 함께 봐야 한다

concurrency를 5에서 20으로 늘렸을 때 평균 처리시간은 좋아 보일 수 있습니다.
하지만 p95, p99가 급격히 나빠지고 timeout이 늘어나면 사용자 체감은 오히려 악화됩니다.
그래서 아래 지표를 함께 보는 편이 좋습니다.

- 작업 성공률
- timeout 비율
- p95, p99 지연 시간
- DB 대기 시간
- 외부 API 429 또는 5xx 비율
- 프로세스 메모리 사용량
- queue 적체 길이

좋은 concurrency limit는 순간 최고 속도를 만드는 값이 아니라, **오래 돌려도 흔들리지 않는 값**입니다.
이 관점은 [Node.js Priority Queue 가이드](/development/blog/seo/2026/04/02/nodejs-priority-queue-job-scheduling-backpressure-guide.html)와도 맞닿아 있습니다.
중요한 작업을 먼저 처리하더라도, 동시에 너무 많이 실행하면 결국 전체 품질이 같이 무너질 수 있기 때문입니다.

## Backpressure, Rate Limit, Queue와는 어떻게 다를까

### H3. concurrency limit는 이미 받아들인 작업의 실행 폭을 제어한다

비슷해 보이지만 역할은 조금씩 다릅니다.

- rate limit: 일정 시간 동안 들어오는 요청 수를 제한
- admission control: 시스템 상태에 따라 요청 수용 자체를 더 보수적으로 판단
- queue: 지금 당장 못 하는 작업을 뒤로 미룸
- concurrency limit: 현재 동시에 실행할 작업 수를 제한

즉 concurrency limit는 보통 **실행 단계의 안전장치**입니다.
요청을 받았더라도, 작업을 바로 다 시작하지 않고 감당 가능한 수만 실행합니다.
이 장치가 없으면 queue가 있어도 worker가 꺼내는 속도가 너무 빨라 다시 과부하가 생길 수 있습니다.

### H3. queue가 있다고 concurrency 문제가 자동으로 해결되지는 않는다

메시지 큐를 도입하면 안전해졌다고 느끼기 쉽습니다.
하지만 consumer가 한 번에 너무 많은 메시지를 꺼내 처리하면 문제는 그대로 반복됩니다.

예를 들어 아래 상황은 흔합니다.

- 큐 적체가 쌓임
- 처리 속도를 올리려고 consumer concurrency를 급격히 증가시킴
- DB와 외부 API가 먼저 포화됨
- 실패와 재시도가 늘어 큐가 다시 불어남

이런 악순환을 끊으려면 queue depth만 볼 것이 아니라 **실행 중 작업 수 자체를 제한**해야 합니다.
그래서 많은 시스템에서 queue와 concurrency limit는 세트로 다뤄집니다.

## 운영에서 꼭 챙겨야 할 체크리스트

### H3. limit 값보다 더 중요한 것은 관측 가능성이다

아무리 좋은 값을 설정해도 관측이 안 되면 금방 감으로 운영하게 됩니다.
아래 지표는 최소한 남겨 두는 편이 좋습니다.

- 현재 실행 중인 작업 수
- 대기 중인 작업 수
- 작업별 평균/상위 지연 시간
- timeout과 retry 횟수
- downstream별 실패율
- 작업 취소 또는 드롭 건수

가능하면 로그에도 아래 맥락을 함께 남기는 편이 좋습니다.

- 작업 종류
- limit 값
- 실제 동시 실행 수
- 작업 시작/종료 시각
- 실패 원인 분류

이 정보가 있어야 "왜 느려졌는가"를 감으로 추측하지 않고 확인할 수 있습니다.

### H3. limit는 고정 숫자가 아니라 환경에 따라 다르게 가져갈 수 있다

모든 환경에서 같은 값을 쓸 필요는 없습니다.
예를 들어 아래처럼 다르게 가져갈 수 있습니다.

- 개발 환경: 작은 값으로 안전하게 테스트
- 배치 전용 워커: 작업 특성에 맞는 별도 값 사용
- 고비용 엔드포인트: 더 낮은 limit 적용
- 읽기 중심 작업: 상대적으로 높은 limit 허용

또한 feature flag를 이용해 limit 값을 점진적으로 조정하면 장애 위험을 줄일 수 있습니다.
이 방식은 [Node.js Brownout Pattern 가이드](/development/blog/seo/2026/04/01/nodejs-brownout-pattern-feature-toggle-overload-resilience-guide.html)처럼 과부하 시 일부 기능을 완화하는 운영 전략과도 잘 어울립니다.

## Node.js Concurrency Limit 적용 체크리스트

### H3. 아래 질문에 답할 수 있으면 운영 적용 준비가 어느 정도 된 상태다

- 지금 병목은 CPU인가, DB인가, 외부 API인가?
- 무제한 병렬 처리 구간이 코드 어디에 있는가?
- limit는 요청 내부와 프로세스 전체 중 어느 층위에 필요한가?
- timeout, retry, partial failure 정책은 정의돼 있는가?
- 실행 중 작업 수와 대기열 길이를 metrics로 볼 수 있는가?
- 큐 consumer와 API 서버의 동시성 정책이 충돌하지 않는가?

concurrency limit의 핵심은 느리게 만드는 것이 아닙니다.
**감당 가능한 폭 안에서 안정적으로 빠르게 만드는 것**입니다.
무제한 병렬화가 대담해 보일 수는 있어도, 운영에서는 대부분 오래 버티지 못합니다.

## 자주 묻는 질문

### H3. Promise.all을 쓰면 항상 안 좋은가요?

그렇지는 않습니다.
작업 수가 매우 적고 downstream 비용이 낮으며, 동시에 실행해도 자원 압박이 거의 없다면 충분히 실용적입니다.
문제는 작업 수가 커지거나 외부 의존성이 비싸질 때입니다.
그때는 `Promise.all()`이 단순함의 장점보다 운영 리스크를 더 크게 만들 수 있습니다.

### H3. concurrency limit 값은 보통 몇으로 시작하나요?

정답은 없습니다.
다만 많은 경우 5, 10, 20처럼 비교적 작은 값으로 시작해서 지표를 보며 늘리는 편이 안전합니다.
처음부터 큰 값으로 올렸다가 장애를 맞는 것보다, 보수적으로 시작해 처리량을 확인하는 쪽이 운영 난이도가 낮습니다.

### H3. rate limit만 잘 걸면 concurrency limit는 필요 없나요?

아닌 경우가 많습니다.
rate limit는 들어오는 요청 수를 다루고, concurrency limit는 이미 받아들인 작업의 실행 폭을 다룹니다.
예를 들어 요청 수는 적어도 요청 하나가 내부적으로 외부 API를 수십 번 호출한다면 concurrency 문제가 여전히 발생할 수 있습니다.

## 마무리

Node.js에서 concurrency limit는 사소한 최적화 옵션이 아닙니다.
서비스가 안정적으로 버틸 수 있는 병렬성의 경계를 코드로 표현하는 운영 장치에 가깝습니다.

많은 장애는 “서버가 바빠서”라기보다, **한 번에 너무 많은 일을 동시에 시작해서** 생깁니다.
그런 점에서 Promise Pool은 속도를 늦추는 도구가 아니라, 시스템이 무너지지 않으면서 처리량을 유지하도록 돕는 안전장치입니다.
특히 배치, 큐 consumer, 외부 API fan-out 로직이 많은 Node.js 서비스라면 concurrency limit는 나중에 붙이는 장식이 아니라 초기에 설계해 둘수록 값이 큽니다.

결국 중요한 질문은 이것입니다.
**이 작업을 더 빨리 시작할 수 있는가가 아니라, 지금 이만큼 동시에 시작해도 시스템이 건강한가**.
그 질문에 정직하게 답하도록 도와주는 것이 concurrency limit의 진짜 역할입니다.

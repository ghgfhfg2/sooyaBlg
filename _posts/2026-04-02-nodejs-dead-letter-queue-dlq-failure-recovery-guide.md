---
layout: post
title: "Node.js Dead Letter Queue 가이드: 실패한 메시지를 버리지 않고 안전하게 복구하는 방법"
date: 2026-04-02 20:00:00 +0900
lang: ko
translation_key: nodejs-dead-letter-queue-dlq-failure-recovery-guide
permalink: /development/blog/seo/2026/04/02/nodejs-dead-letter-queue-dlq-failure-recovery-guide.html
alternates:
  ko: /development/blog/seo/2026/04/02/nodejs-dead-letter-queue-dlq-failure-recovery-guide.html
  x_default: /development/blog/seo/2026/04/02/nodejs-dead-letter-queue-dlq-failure-recovery-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, dead-letter-queue, dlq, message-queue, bullmq, retry, idempotency, resilience]
description: "Node.js 서비스에서 Dead Letter Queue(DLQ)를 설계해 반복 실패 작업을 안전하게 격리하고, 재처리와 장애 복구 흐름을 만드는 실무 방법을 정리했습니다."
---

메시지 큐를 운영하다 보면 언젠가는 **계속 실패하는 작업**을 만나게 됩니다.
외부 API 스키마가 바뀌었거나, 특정 데이터가 이미 깨져 있거나, 코드 버그 때문에 어떤 잡만 반복해서 실패하는 경우가 대표적입니다.
이때 가장 위험한 선택은 실패한 작업을 무한 재시도하게 두는 것입니다.
큐는 점점 막히고, 같은 에러 로그만 쌓이고, 정작 정상 작업까지 밀리기 시작합니다.

이럴 때 필요한 것이 **Dead Letter Queue(DLQ)** 입니다.
DLQ는 실패한 작업을 그냥 버리는 곳이 아니라, **정상 흐름에서 분리해 원인을 분석하고 안전하게 재처리하기 위한 격리 구역**에 가깝습니다.
이 글에서는 Node.js 환경에서 DLQ가 왜 필요한지, 어떤 실패를 보내야 하는지, 재처리와 관측은 어떻게 설계해야 하는지 실무 관점에서 정리합니다.

## Node.js Dead Letter Queue가 필요한 이유

### H3. 반복 실패 작업은 처리량보다 운영 피로도를 먼저 망가뜨린다

큐 시스템 장애는 항상 대규모 장애로만 오지 않습니다.
실제로는 전체 작업의 일부만 계속 실패하는 상태가 더 흔합니다.
문제는 이런 실패가 조용히 시스템을 갉아먹는다는 점입니다.

예를 들어 아래 같은 상황을 떠올려볼 수 있습니다.

- 외부 결제사 웹훅 payload 형식이 일부 변경됨
- 특정 고객 데이터가 누락돼 후처리 job이 항상 예외를 던짐
- 이미 삭제된 리소스를 다시 동기화하려다 404가 반복됨
- 오래된 이벤트 스키마가 신규 consumer 코드와 맞지 않음

이 작업들을 본 큐에 그대로 두고 무한 재시도하면, worker는 가치 없는 실패를 계속 반복합니다.
결국 시스템은 바쁘게 일하는 것처럼 보이지만 실제로는 **아무것도 회복하지 못한 채 자원만 태우는 상태**가 됩니다.
DLQ는 이 지점에서 중요합니다.
실패를 숨기지 않고, 정상 작업 흐름과 분리해서 다룰 수 있게 해주기 때문입니다.

### H3. 실패를 격리해야 정상 작업의 지연과 전파를 줄일 수 있다

반복 실패 작업이 위험한 이유는 실패 그 자체보다 **전파 효과**에 있습니다.
재시도가 공격적으로 설정돼 있으면 아래 문제가 연쇄적으로 생깁니다.

- 같은 job이 worker 슬롯을 반복 점유함
- 외부 시스템에 불필요한 요청이 계속 나감
- queue wait time이 증가해 정상 작업도 늦어짐
- 에러 알림이 폭증해 진짜 중요한 이상 징후를 놓치게 됨

이런 상황에서는 단순 retry tuning만으로 부족한 경우가 많습니다.
어느 시점 이후에는 "이 작업은 지금 정상 경로에서 빼야 한다"는 판단이 필요합니다.
DLQ는 바로 그 판단을 시스템에 반영하는 방법입니다.

## Dead Letter Queue는 무엇인가

### H3. 일정 횟수 이상 실패하거나 처리 불가능하다고 판단된 작업을 보내는 별도 큐다

Dead Letter Queue는 말 그대로 실패한 메시지를 모아두는 큐입니다.
하지만 실무에서는 단순 보관함보다는 아래 역할을 동시에 수행합니다.

- 반복 실패 작업 격리
- 원인 분석용 증거 보존
- 수동 또는 자동 재처리 출발점 제공
- 장애 범위 축소

즉 DLQ의 핵심은 "실패를 따로 모은다"보다, **정상 처리 파이프라인을 오염시키지 않으면서 복구 가능성을 남긴다**에 있습니다.

보통 아래 기준 중 하나에 걸리면 DLQ로 보냅니다.

- 최대 재시도 횟수 초과
- 역직렬화 실패로 즉시 처리 불가
- 비즈니스 검증 오류로 현재 상태에서는 처리 불가
- 외부 의존성 변경으로 코드 수정 전까지 재처리 의미 없음

### H3. 모든 실패를 DLQ로 보내는 것이 아니라, 본 큐에서 더 처리할 가치가 없는 실패를 분리하는 개념이다

DLQ를 처음 도입할 때 흔한 오해는 실패한 작업을 전부 DLQ로 보내려는 것입니다.
하지만 그렇게 하면 transient failure까지 너무 빨리 격리돼 복구 효율이 떨어집니다.

실무에서는 보통 이렇게 나눕니다.

- transient failure: 일시적 네트워크 오류, 짧은 timeout, 순간적인 rate limit
- persistent failure: 잘못된 payload, 코드 버그, 스키마 불일치, 누락 데이터

첫 번째는 retry와 backoff로 회복될 가능성이 큽니다.
두 번째는 재시도만으로는 거의 해결되지 않습니다.
DLQ는 주로 두 번째 범주를 다루기 위한 장치입니다.

## Node.js에서 어떤 작업을 DLQ로 보내야 할까

### H3. 재시도로 회복될 가능성이 낮은 실패부터 분류해야 한다

DLQ 설계에서 가장 중요한 것은 "몇 번 실패했는가"보다 **왜 실패했는가**입니다.
예를 들어 외부 API가 잠깐 느린 상황이라면 3번 실패 후에도 4번째에 성공할 수 있습니다.
반대로 필수 필드가 빠진 payload는 100번 재시도해도 성공하지 않습니다.

아래는 DLQ 후보로 보기 좋은 실패들입니다.

- 필수 데이터 누락
- 예상하지 못한 이벤트 버전
- 멱등성 키 불일치로 인한 정합성 오류
- 코드 버그로 특정 입력에서 항상 발생하는 예외
- 더 이상 존재하지 않는 리소스를 참조하는 작업

반대로 아래는 먼저 retry 정책을 점검하는 편이 낫습니다.

- 순간적 네트워크 불안정
- 외부 API의 일시적 429/503 응답
- DB failover 순간 timeout
- 짧은 락 경합으로 인한 재시도 가능 오류

중요한 것은 실패를 개수 기준으로만 보지 않고, **회복 가능성과 비즈니스 위험 기준으로 분류하는 것**입니다.

### H3. 정합성 관련 작업은 DLQ 진입 시점에 맥락 정보를 꼭 남겨야 한다

결제, 주문, 정산, 데이터 동기화처럼 정합성이 중요한 작업은 DLQ로 보낼 때 메타데이터를 충분히 남겨야 합니다.
나중에 원인을 추적하고 재처리할 때 문맥이 없으면 복구가 더 어려워집니다.

최소한 아래 정보는 남기는 편이 좋습니다.

- job type
- 원본 payload 또는 복구 가능한 축약본
- 실패 시각과 시도 횟수
- 마지막 에러 메시지와 스택 일부
- 관련 리소스 식별자(orderId, userId 등)
- producer 버전 또는 이벤트 버전

다만 여기서도 개인정보나 민감정보는 그대로 저장하면 안 됩니다.
이메일, 전화번호, 토큰, 카드 관련 값은 마스킹하거나 아예 제외해야 합니다.
DLQ는 디버깅에 유용하지만, **디버깅 편의가 데이터 보호 원칙보다 앞설 수는 없습니다**.

## Node.js Dead Letter Queue 구현 방식

### H3. 가장 단순한 시작은 본 큐와 별도의 DLQ를 분리하는 것이다

BullMQ 같은 큐를 쓴다면 애플리케이션 레벨에서 DLQ를 별도 큐로 두는 방식이 이해하기 쉽습니다.

```js
import { Queue, Worker } from 'bullmq';

const connection = {
  host: process.env.REDIS_HOST,
  port: Number(process.env.REDIS_PORT || 6379)
};

const jobsQueue = new Queue('jobs', { connection });
const dlqQueue = new Queue('jobs-dlq', { connection });

const worker = new Worker(
  'jobs',
  async job => {
    await processJob(job.name, job.data);
  },
  { connection }
);

worker.on('failed', async (job, err) => {
  if (!job) return;

  const attemptsMade = job.attemptsMade ?? 0;
  const maxAttempts = job.opts.attempts ?? 1;

  if (attemptsMade >= maxAttempts) {
    await dlqQueue.add('failed-job', {
      originalJobName: job.name,
      originalData: sanitizeForDlq(job.data),
      attemptsMade,
      failedAt: new Date().toISOString(),
      errorMessage: err.message
    });
  }
});
```

이 구조의 장점은 명확합니다.

- 정상 작업 큐와 실패 격리 큐의 역할이 분리됨
- DLQ backlog를 별도 메트릭으로 추적 가능
- 재처리 워크플로를 독립적으로 설계 가능

중요한 점은 DLQ enqueue 전에 `sanitizeForDlq` 같은 단계로 민감 데이터를 정리하는 것입니다.
원본 payload 전체를 습관적으로 복사하면 나중에 운영 리스크가 커집니다.

### H3. 에러 유형별로 retry와 DLQ 진입 조건을 다르게 두는 편이 낫다

모든 예외를 같은 방식으로 다루면 DLQ가 너무 빨리 차거나, 반대로 무의미한 재시도가 오래 지속될 수 있습니다.
실무에서는 에러 클래스를 나눠 처리하는 편이 훨씬 안정적입니다.

```js
class RetryableError extends Error {}
class NonRetryableError extends Error {}

async function processJob(name, data) {
  const payload = validatePayload(data);

  if (!payload.userId) {
    throw new NonRetryableError('userId is required');
  }

  const result = await callExternalApi(payload);

  if (result.status === 503) {
    throw new RetryableError('temporary downstream failure');
  }
}
```

이 접근을 쓰면 아래처럼 정책이 쉬워집니다.

- `RetryableError`: exponential backoff 후 재시도
- `NonRetryableError`: 빠르게 DLQ로 이동
- 알 수 없는 예외: 제한된 횟수 재시도 후 DLQ 이동

재시도 전략은 [Node.js Exponential Backoff 가이드](/development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html)와 함께 설계하면 더 안정적입니다.
핵심은 같은 실패를 같은 방식으로 때리는 것이 아니라, **실패 성격에 맞는 복구 경로를 주는 것**입니다.

### H3. 재처리 파이프라인은 원본 큐와 분리해 천천히 운영해야 한다

DLQ를 만들고 나서 가장 많이 생기는 실수는 실패 작업을 한 번에 본 큐로 되돌리는 것입니다.
하지만 코드 버그나 데이터 문제를 고치지 않은 상태에서 대량 재주입하면 장애를 다시 재현할 뿐입니다.

안전한 재처리 흐름은 보통 아래와 같습니다.

1. DLQ 적재 원인 분류
2. 코드 수정 또는 데이터 보정
3. 샘플 단위 재처리
4. 제한된 배치 재처리
5. 정상 지표 확인 후 점진 확대

이 과정은 [Node.js Idempotency Key 가이드](/development/blog/seo/2026/03/27/nodejs-idempotency-key-api-duplicate-request-guide.html)와도 밀접합니다.
실패 작업을 다시 넣을 때는 **중복 처리돼도 안전한지**가 반드시 보장돼야 합니다.
재처리는 복구 수단이지만, 멱등성이 없으면 새로운 장애 원인이 될 수 있습니다.

## priority queue, backpressure와는 어떻게 연결될까

### H3. DLQ는 실패 작업을 격리하고, priority queue는 중요한 작업을 먼저 살린다

DLQ와 priority queue는 서로 대체 관계가 아닙니다.
둘은 보는 문제가 다릅니다.

- priority queue: 이미 들어온 작업 중 무엇을 먼저 처리할 것인가
- DLQ: 더 이상 본 흐름에서 처리할 가치가 없는 실패를 어디로 뺄 것인가

예를 들어 인증 메일 발송 job이 중요해서 high priority라고 해도, payload 자체가 깨져 있다면 high priority로 오래 재시도하는 것은 오히려 손해입니다.
이럴 때는 중요 작업일수록 더 빨리 원인을 드러내고 격리하는 DLQ 전략이 필요합니다.

관련 흐름은 오늘 아침 글인 [Node.js Priority Queue 가이드](/development/blog/seo/2026/04/02/nodejs-priority-queue-job-scheduling-backpressure-guide.html)와 함께 보면 더 잘 연결됩니다.
중요한 작업을 먼저 처리하는 것과, **처리 불가능한 작업을 빨리 분리하는 것**은 같이 있어야 운영 효율이 올라갑니다.

### H3. backpressure가 없다면 DLQ만으로는 실패 폭주를 막기 어렵다

DLQ는 실패 작업을 격리해 주지만, producer가 계속 같은 잘못된 작업을 밀어 넣으면 결국 또 다른 backlog가 생깁니다.
그래서 DLQ는 backpressure나 admission control과 함께 봐야 합니다.

예를 들어 이런 식입니다.

- DLQ 증가율이 급증하면 신규 특정 job type 생성 속도 제한
- 특정 이벤트 버전의 실패가 많으면 producer 쪽 publish 일시 중단
- downstream 장애 시 low priority 후처리 enqueue 자체를 축소

즉 DLQ는 결과를 처리하는 장치이고, backpressure는 원인을 줄이는 장치에 가깝습니다.
둘을 분리해서 보면 대응이 늦어지기 쉽습니다.

## 운영에서 꼭 봐야 할 DLQ 지표

### H3. DLQ 건수보다 실패 유형과 재처리 성공률이 더 중요하다

DLQ 운영에서 단순 적재 건수만 보면 실체를 놓치기 쉽습니다.
같은 100건이라도 단일 버그 때문에 몰린 것과, 여러 시스템 문제가 섞인 것은 의미가 다릅니다.

실무에서 특히 볼 만한 지표는 아래와 같습니다.

- job type별 DLQ 유입 건수
- 에러 유형별 분포
- 첫 실패부터 DLQ 진입까지 걸린 시간
- DLQ 적재 후 재처리 성공률
- 동일 원인 반복 발생 비율
- DLQ oldest message age

이 지표를 보면 단순히 "실패가 많다"가 아니라, **무엇이 얼마나 오래 복구되지 않고 있는가**를 파악할 수 있습니다.

### H3. 알림은 DLQ 누적량보다 급증 패턴을 먼저 잡는 편이 실용적이다

DLQ는 어느 정도 존재하는 것이 이상하지 않습니다.
그래서 절대 개수만으로 알림을 걸면 노이즈가 많아질 수 있습니다.
오히려 아래 같은 신호가 더 유용합니다.

- 10분 내 특정 job type DLQ 급증
- 새 배포 직후 동일 에러 메시지 반복 증가
- 재처리 성공률 급락
- critical job이 DLQ로 유입되기 시작함

알림의 목적은 운영자를 괴롭히는 것이 아니라, **복구가 필요한 패턴 변화를 빨리 잡는 것**입니다.

## Node.js Dead Letter Queue 적용 체크리스트

### H3. 아래 질문에 답할 수 있으면 DLQ 설계가 꽤 준비된 상태다

다음 질문에 명확히 답할 수 있으면 DLQ를 도입해도 운영 혼란이 적습니다.

- 어떤 실패가 retry 대상이고 어떤 실패가 DLQ 대상인가?
- DLQ에 남길 메타데이터는 무엇이고, 무엇은 마스킹해야 하는가?
- 재처리는 누가 어떤 절차로 승인하고 실행하는가?
- 재처리 시 멱등성은 어떻게 보장하는가?
- DLQ 급증 시 producer 또는 enqueue 정책을 어떻게 조정할 것인가?
- critical job의 DLQ 유입은 어떤 임계치에서 경고할 것인가?

DLQ의 목적은 실패를 보기 좋게 모으는 데 있지 않습니다.
**반복 실패가 정상 작업을 망치지 못하게 격리하고, 복구 가능한 형태로 남겨두는 것**이 핵심입니다.

## 자주 묻는 질문

### H3. Dead Letter Queue가 있으면 실패한 작업을 절대 잃지 않나요?

반드시 그렇지는 않습니다.
DLQ도 결국 별도 저장소이기 때문에 보존 기간, 적재 실패, 직렬화 실패 같은 운영 문제가 있을 수 있습니다.
중요한 것은 DLQ를 만들었다는 사실보다, 저장·관측·재처리 절차를 함께 설계하는 것입니다.

### H3. 모든 큐 시스템에 DLQ가 필요한가요?

항상 그렇지는 않습니다.
작업 수가 작고 실패가 거의 없으며, 수동 복구 비용도 낮다면 단순 로그와 제한된 retry만으로 충분할 수 있습니다.
하지만 외부 의존성, 데이터 정합성, 비동기 후처리가 늘어날수록 DLQ의 가치가 커집니다.

### H3. DLQ에 쌓인 작업은 자동 재처리하면 안 되나요?

가능은 하지만 매우 신중해야 합니다.
원인이 해결되지 않은 상태에서 자동 재처리하면 같은 실패를 반복하거나 중복 처리 위험이 생길 수 있습니다.
특히 주문·결제·동기화 작업은 자동 재처리 전에 멱등성과 데이터 상태를 먼저 확인하는 편이 안전합니다.

## 마무리

Node.js 서비스에서 Dead Letter Queue는 실패를 포기하기 위한 장치가 아닙니다.
오히려 실패를 통제 가능한 형태로 바꾸기 위한 운영 장치에 가깝습니다.

실패한 작업을 본 큐에 계속 남겨두면 시스템은 바쁜데 회복은 되지 않는 상태에 빠지기 쉽습니다.
반복 실패를 정상 흐름에서 분리하고, 필요한 문맥을 보존한 채 재처리 경로를 따로 두면 장애 대응은 훨씬 차분해집니다.
특히 외부 API 연동, 이벤트 기반 처리, 배치 후처리가 많은 Node.js 시스템일수록 DLQ는 선택이 아니라 운영 안전장치에 가깝습니다.

결국 중요한 질문은 이것입니다.
**이 실패를 지금 다시 시도하는 것이 정말 회복에 도움이 되는가, 아니면 격리해서 원인을 먼저 봐야 하는가**.
그 질문에 체계적으로 답하게 만들어 주는 것이 DLQ의 진짜 가치입니다.

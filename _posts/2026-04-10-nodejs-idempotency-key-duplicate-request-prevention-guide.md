---
layout: post
title: "Node.js Idempotency Key 가이드: 중복 요청과 재시도를 안전하게 처리하는 법"
date: 2026-04-10 20:00:00 +0900
lang: ko
translation_key: nodejs-idempotency-key-duplicate-request-prevention-guide
permalink: /development/blog/seo/2026/04/10/nodejs-idempotency-key-duplicate-request-prevention-guide.html
alternates:
  ko: /development/blog/seo/2026/04/10/nodejs-idempotency-key-duplicate-request-prevention-guide.html
  x_default: /development/blog/seo/2026/04/10/nodejs-idempotency-key-duplicate-request-prevention-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, idempotency, retry, duplicate-request, api-design, payment, backend, resilience]
description: "Node.js에서 idempotency key로 중복 요청과 재시도를 안전하게 처리하는 방법, 저장 전략, TTL, 충돌 처리, 운영 관측 포인트를 실무 관점에서 정리했습니다."
---

분산 시스템에서는 실패보다도 **애매한 성공**이 더 까다로운 경우가 많습니다.
클라이언트는 timeout을 봤고 다시 요청을 보냈는데, 서버에서는 첫 번째 요청이 이미 처리됐을 수도 있기 때문입니다.
특히 결제, 주문 생성, 쿠폰 발급, 외부 API 연동처럼 한 번만 실행돼야 하는 작업에서는 이 문제가 바로 장애와 데이터 불일치로 이어집니다.

이럴 때 핵심 도구가 **idempotency key**입니다.
같은 의도를 가진 재시도 요청에는 같은 키를 붙이고, 서버는 그 키를 기준으로 이미 처리한 결과인지 확인합니다.
이 글에서는 Node.js에서 idempotency key가 왜 필요한지, 어디에 저장해야 하는지, TTL과 충돌 검증을 어떻게 설계해야 하는지 실무 관점에서 정리합니다.

## Node.js Idempotency Key가 필요한 이유

### H3. timeout 이후 재시도는 정상 동작이지만, 서버 입장에서는 중복 실행이 될 수 있다

실무에서 클라이언트 재시도는 예외적인 상황이 아닙니다.
모바일 네트워크가 불안정하거나, API gateway timeout이 짧거나, 외부 결제사 응답이 늦으면 같은 요청이 몇 초 안에 다시 들어올 수 있습니다.
문제는 서버가 첫 번째 요청을 처리 중이거나 이미 완료했는데, 두 번째 요청도 새 작업으로 받아들이는 경우입니다.

이때 생길 수 있는 문제는 아래와 같습니다.

- 주문이 두 번 생성됨
- 결제가 중복 승인됨
- 포인트나 쿠폰이 두 번 지급됨
- 동일 메시지가 여러 번 발행됨
- 외부 시스템과 내부 상태가 어긋남

즉 재시도 자체를 막는 것이 아니라, **재시도를 안전하게 받아줄 수 있게 만드는 설계**가 필요합니다.
이 점은 [Node.js Retry Budget 가이드](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html)와도 이어집니다.
재시도를 줄이는 정책도 중요하지만, 반드시 발생하는 재시도에 대해 중복 실행 방지 장치가 있어야 합니다.

### H3. “중복 요청 방지”가 아니라 “같은 의도인지 판별”하는 것이 본질이다

idempotency key의 핵심은 단순히 같은 요청을 버리는 것이 아닙니다.
**같은 사용자 의도에서 나온 재시도인지 식별하는 것**이 더 중요합니다.
예를 들어 사용자가 같은 상품을 정말 두 번 결제하고 싶을 수도 있습니다.
이 경우에는 서로 다른 키를 써야 합니다.

그래서 idempotency key는 보통 아래처럼 다룹니다.

- 동일 작업의 재시도면 같은 키 사용
- 새로운 작업이면 새로운 키 사용
- 서버는 키와 요청 payload가 일치하는지도 함께 검증

이 원칙이 없으면 정상 재주문까지 중복 요청으로 오판할 수 있습니다.

## Idempotency Key를 어디에 적용하면 좋을까

### H3. 결제, 주문 생성, 계정 변경처럼 “한 번만 반영돼야 하는 쓰기 작업”에 우선 적용한다

idempotency key는 모든 API에 무조건 붙이는 장치는 아닙니다.
대개 아래 같은 **부작용이 있는 write API**에 우선 적용합니다.

- 결제 승인 요청
- 주문 생성 API
- 환불 요청
- 포인트 적립/차감
- 이메일, SMS, 푸시 발송 트리거
- 외부 파트너 시스템으로 전송하는 동기화 요청

반대로 단순 조회 API는 캐시나 request coalescing으로 풀 수 있는 경우가 많습니다.
조회 쪽 중복 완화는 [Node.js Request Coalescing 가이드](/development/blog/seo/2026/04/05/nodejs-request-coalescing-cache-stampede-prevention-guide.html)를 함께 보는 편이 좋습니다.

### H3. 비동기 이벤트 발행 전에도 중복 실행 방지 계층이 필요할 수 있다

HTTP 요청 단에서만 막으면 끝나는 것도 아닙니다.
애플리케이션이 요청을 한 번만 받았더라도, 내부 재시도나 메시지 브로커 재전송으로 이벤트가 중복 발행될 수 있습니다.
그래서 아래처럼 경계를 나눠 생각하는 편이 안전합니다.

- API 경계: 동일 HTTP 요청 재시도 방지
- 작업 큐 경계: 동일 작업 enqueue 중복 방지
- 소비자 경계: 동일 이벤트 소비 중복 방지

즉 idempotency는 API 옵션이 아니라 **시스템 전체의 중복 처리 전략**에 가깝습니다.

## Node.js에서 Idempotency Key를 구현하는 기본 흐름

### H3. 요청 시작 시 키를 저장하고, 처리 결과를 같은 키에 묶어 재사용한다

가장 단순한 흐름은 아래와 같습니다.

1. 클라이언트가 `Idempotency-Key` 헤더를 보낸다
2. 서버는 키가 이미 존재하는지 확인한다
3. 없으면 처리 중 상태로 먼저 기록한다
4. 실제 비즈니스 로직을 수행한다
5. 성공 또는 실패 결과를 키에 연결해 저장한다
6. 같은 키로 재요청이 오면 저장된 결과를 그대로 반환한다

개념 설명용 Express 예시는 아래와 같습니다.

```js
import crypto from 'node:crypto';
import express from 'express';

const app = express();
app.use(express.json());

const store = new Map();

function hashPayload(body) {
  return crypto
    .createHash('sha256')
    .update(JSON.stringify(body))
    .digest('hex');
}

app.post('/payments', async (req, res) => {
  const key = req.header('Idempotency-Key');
  if (!key) {
    return res.status(400).json({ message: 'Idempotency-Key header is required' });
  }

  const payloadHash = hashPayload(req.body);
  const existing = store.get(key);

  if (existing) {
    if (existing.payloadHash !== payloadHash) {
      return res.status(409).json({
        message: 'Same idempotency key was used with a different payload',
      });
    }

    if (existing.status === 'completed') {
      return res.status(existing.responseStatus).json(existing.responseBody);
    }

    return res.status(202).json({ message: 'Request is already being processed' });
  }

  store.set(key, {
    status: 'processing',
    payloadHash,
    createdAt: Date.now(),
  });

  try {
    const payment = await createPayment(req.body);

    store.set(key, {
      status: 'completed',
      payloadHash,
      responseStatus: 201,
      responseBody: payment,
      createdAt: Date.now(),
    });

    return res.status(201).json(payment);
  } catch (error) {
    store.delete(key);
    throw error;
  }
});
```

핵심은 **비즈니스 로직 실행 전에 먼저 키를 기록**하는 것입니다.
처리 후에 저장하면 동시에 들어온 두 요청이 모두 실행될 수 있습니다.

### H3. 메모리 Map 예시는 개념용일 뿐, 실서비스에서는 공유 저장소가 필요하다

프로세스 하나에서만 돌고 재시작이 없다는 보장이 없다면 in-memory 저장은 금방 한계를 드러냅니다.
실서비스에서는 보통 아래 저장소를 검토합니다.

- Redis: 빠르고 TTL 관리가 쉬워서 가장 흔함
- RDB: 주문/결제 레코드와 함께 강한 일관성으로 처리 가능
- DynamoDB 같은 KV 저장소: 고유 키 제약과 만료 정책 활용 가능

멀티 인스턴스 환경에서는 모든 애플리케이션 노드가 같은 키 상태를 볼 수 있어야 합니다.
그렇지 않으면 인스턴스 A와 B가 같은 키를 각각 처음 보는 것으로 판단해 중복 실행이 발생합니다.

## 충돌과 경쟁 상태를 막으려면 무엇을 같이 설계해야 할까

### H3. “키 존재 확인 후 저장”을 분리하면 race condition이 생길 수 있다

가장 흔한 실수는 아래 순서입니다.

1. 키 조회
2. 없다고 판단
3. 비즈니스 로직 실행
4. 나중에 결과 저장

이 구조에서는 같은 키 요청 두 개가 거의 동시에 들어오면 둘 다 2번에서 통과할 수 있습니다.
그래서 저장소는 가능한 한 **원자적(atomic)으로 선점**할 수 있어야 합니다.

예를 들어 Redis라면 `SET key value NX EX ttl`처럼, 없을 때만 저장하는 연산을 먼저 사용합니다.
RDB라면 unique key 제약과 트랜잭션을 조합해 구현할 수 있습니다.

### H3. 같은 키에 다른 payload가 오면 409로 거절하는 편이 안전하다

idempotency key는 재시도 식별자이지, 임의의 요청 그룹 이름이 아닙니다.
같은 키로 다른 payload를 보내는 순간 의미가 모호해집니다.
그래서 서버는 키만 볼 것이 아니라 **payload fingerprint**도 함께 저장하는 편이 좋습니다.

검증 항목 예시는 아래와 같습니다.

- 사용자 ID 또는 테넌트 ID
- HTTP method + path
- 요청 본문 해시
- 주요 비즈니스 필드(예: orderId, amount, currency)

이렇게 해 두면 클라이언트 버그로 키를 재사용했을 때 조용히 잘못된 결과를 돌려주는 일을 줄일 수 있습니다.

## TTL과 상태 관리는 어떻게 잡아야 할까

### H3. TTL은 재시도 윈도우와 비즈니스 위험도를 기준으로 정한다

idempotency record를 영원히 보관할 필요는 없는 경우가 많습니다.
대신 너무 빨리 지우면 늦게 도착한 재시도를 새 요청으로 처리하게 됩니다.
그래서 TTL은 아래를 기준으로 잡습니다.

- 클라이언트가 재시도할 수 있는 최대 시간
- 네트워크 지연이나 큐 적체 가능성
- 결제/주문처럼 중복 비용이 큰 작업인지 여부
- 저장소 비용과 키 개수 증가 속도

보통은 몇 분에서 24시간 사이에서 시작해 조정합니다.
중복 비용이 큰 작업일수록 보수적으로 길게 잡는 편이 낫습니다.

### H3. 실패 결과를 저장할지, 삭제하고 재시도하게 할지도 정책이 필요하다

모든 실패를 똑같이 다뤄서는 안 됩니다.
예를 들어 validation error처럼 **같은 요청을 다시 보내도 반드시 실패하는 경우**는 실패 응답도 저장하는 편이 낫습니다.
반면 일시적 DB 오류나 외부 API timeout처럼 재시도가 의미 있는 실패는 키를 삭제하거나, retriable 상태로 남기는 편이 더 적절할 수 있습니다.

실무에서는 대개 아래처럼 나눕니다.

- 4xx 확정 실패: 결과 캐시 또는 재사용
- 5xx 일시 실패: 제한적으로 재시도 허용
- 처리 중 상태: 짧은 TTL로 보호 후 상태 조회 유도

이 판단은 [Node.js Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와 함께 보면 더 명확합니다.
어디까지 기다리고, 어디서 실패로 간주하며, 어떤 실패를 재시도할지 기준이 맞물리기 때문입니다.

## 운영에서 꼭 봐야 할 관측 지표

### H3. hit rate만 보지 말고 충돌률, processing 체류 시간, 만료 후 재시도도 함께 본다

idempotency를 도입한 뒤에는 단순 요청 수보다 아래 지표를 보는 편이 좋습니다.

- idempotency hit 비율
- 같은 키, 다른 payload 충돌 건수
- processing 상태 평균 체류 시간
- TTL 만료 후 동일 의도 요청 재발생 비율
- 저장소 오류로 인해 보호를 건너뛴 횟수

특히 processing 상태가 오래 남는다면, 락만 잡고 실제 작업이 끝나지 않는 문제가 숨어 있을 수 있습니다.
이때는 timeout, 보상 작업, 상태 조회 API를 함께 검토해야 합니다.

### H3. 로그에는 키 원문보다 해시나 일부 마스킹 값을 남기는 편이 안전하다

idempotency key 자체가 민감정보는 아닐 수 있지만, 로그에 그대로 남기면 추적 식별자로 악용될 여지가 있습니다.
그래서 운영 로그에는 아래처럼 제한적으로 남기는 편이 안전합니다.

- key 원문 대신 해시
- 앞 6자리와 뒤 4자리만 마스킹 표시
- 사용자 ID와 함께 저장하되 개인정보는 최소화

민감정보를 남기지 않는 기본 원칙은 공통 운영 가이드와도 같습니다.
실무에서는 편의보다 **로그 최소 수집**이 오래 갑니다.

## Node.js에서 Idempotency Key를 도입할 때 자주 하는 실수

### H3. HTTP 계층만 막고, 실제 side effect 중복까지는 막지 못하는 경우

API 서버에서 중복 응답만 잘 처리해도, 내부적으로는 외부 결제 호출이나 메시지 발행이 두 번 일어날 수 있습니다.
그래서 중요한 작업은 아래처럼 끝까지 확인해야 합니다.

- DB 쓰기가 한 번만 반영되는가
- 외부 API 호출이 재진입에 안전한가
- 메시지 발행이 중복 소비에 안전한가
- 보상 트랜잭션이 필요한가

즉 “응답이 같았다”와 “부작용이 한 번만 발생했다”는 다른 문제입니다.

### H3. 키를 너무 넓게 재사용하거나, 반대로 너무 짧게 유지하는 경우

키를 사용자 단위로 너무 넓게 잡으면 정상 요청까지 묶여버립니다.
반대로 요청마다 무조건 새로운 키를 만들면 재시도 보호 효과가 사라집니다.
그래서 보통은 **하나의 사용자 의도당 하나의 키**라는 규칙을 API 문서에 명확히 적어두는 편이 좋습니다.

클라이언트 SDK나 프론트엔드에서 키 생성 규칙을 통일하지 않으면 서버 정책이 좋아도 운영이 흔들립니다.

## 실무 적용 체크리스트

### H3. 아래 다섯 가지가 정리돼야 운영에서 덜 흔들린다

도입 전에 최소한 아래 항목은 문서로 남겨두는 편이 좋습니다.

- 어떤 API에 idempotency key를 요구할지
- 키와 payload 충돌을 어떻게 판정할지
- 저장소를 무엇으로 할지, 선점 연산은 원자적인지
- 성공/실패/processing 상태별 TTL 정책이 무엇인지
- 만료 후 재시도, 장애 복구, 관측 지표를 어떻게 볼지

이 다섯 가지가 정리되면, idempotency는 단순 방어 코드가 아니라 **재시도 친화적인 API 계약**이 됩니다.

## 마무리

Node.js에서 idempotency key는 단순히 중복 요청을 막는 기능이 아닙니다.
불안정한 네트워크와 재시도를 전제로 한 현실적인 백엔드 설계의 일부입니다.
특히 결제, 주문, 포인트처럼 한 번만 반영돼야 하는 작업에서는 “재시도를 막는 것”보다 **재시도가 와도 같은 결과를 안전하게 돌려주는 것**이 더 중요합니다.

정리하면 핵심은 세 가지입니다.

- 비즈니스 로직 전에 키를 원자적으로 선점할 것
- 같은 키에 다른 payload가 오면 충돌로 처리할 것
- TTL, 실패 정책, 관측 지표까지 함께 설계할 것

이 원칙만 지켜도 중복 실행으로 생기는 많은 장애를 꽤 현실적으로 줄일 수 있습니다.

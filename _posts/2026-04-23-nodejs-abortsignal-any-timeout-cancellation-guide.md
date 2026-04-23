---
layout: post
title: "Node.js AbortSignal.any 가이드: timeout과 사용자 취소를 함께 처리하는 법"
date: 2026-04-23 20:00:00 +0900
lang: ko
translation_key: nodejs-abortsignal-any-timeout-cancellation-guide
permalink: /development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html
alternates:
  ko: /development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html
  x_default: /development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, abortsignal, abortcontroller, timeout, cancellation, fetch, backend, reliability, performance]
description: "Node.js에서 AbortSignal.any를 사용해 timeout, 사용자 이탈, 상위 요청 취소를 한 번에 묶는 방법과 실무에서 자주 놓치는 에러 처리 포인트를 정리했습니다."
---

Node.js에서 외부 API 호출이나 긴 비동기 작업을 다루다 보면 `timeout`만으로는 부족한 순간이 자주 나옵니다.
브라우저 탭이 닫혔거나, 상위 요청이 이미 취소됐거나, 더 빠른 대체 응답이 도착했는데도 하위 작업이 계속 돌고 있으면 쓸데없이 자원을 붙잡게 됩니다.

이럴 때 유용한 도구가 `AbortSignal.any()`입니다.
결론부터 말하면 `AbortSignal.any()`는 **여러 취소 조건을 하나의 signal로 합치는 방법**이고, Node.js에서는 `timeout`, 사용자 취소, 상위 요청 종료를 함께 다뤄야 할 때 특히 실용적입니다.

## 왜 timeout 하나만으로는 부족할까

### H3. 느린 작업을 끊는 것과 더 이상 필요 없는 작업을 멈추는 것은 다르다

많은 코드가 취소를 사실상 timeout 하나로만 처리합니다.
하지만 실무에서는 아래 상황이 더 자주 문제를 만듭니다.

- 클라이언트가 이미 연결을 끊었는데 서버 내부 fetch는 계속 실행되는 경우
- 상위 요청 deadline이 지났는데 하위 작업이 별도 timeout만 믿고 계속 도는 경우
- hedged request나 fallback 응답이 먼저 끝났는데 느린 원본 호출이 남아 있는 경우
- 배치 중단, 프로세스 종료, 사용자 취소를 같은 코드 경로에서 처리해야 하는 경우

즉 timeout은 **너무 오래 걸리는 작업을 끊는 장치**이고, cancellation은 **더 이상 의미 없는 작업을 멈추는 장치**입니다.
둘은 겹치지만 완전히 같지 않습니다.

### H3. 취소가 전파되지 않으면 tail latency와 자원 낭비가 함께 커진다

상위 요청이 이미 실패했는데 하위 작업이 남아 있으면 socket, DB connection, CPU 시간을 계속 점유할 수 있습니다.
이런 패턴은 장애 구간에서 더 위험합니다.
재시도까지 붙으면 필요 없는 작업이 시스템 안에 오래 남아 전체 지연을 늘릴 수 있기 때문입니다.

이 감각은 [Node.js Timeout Budget, Deadline Propagation 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와 이어집니다.
시간 예산을 나누는 것만큼 중요한 것이 **취소 신호를 하위 작업까지 전달하는 것**입니다.

## AbortSignal.any는 무엇을 해결하나

### H3. 여러 종료 조건을 하나의 signal로 합친다

`AbortSignal.any()`는 여러 개의 `AbortSignal` 중 하나라도 abort되면, 결과 signal도 즉시 abort되게 해줍니다.
즉 아래 조건을 각각 따로 처리하던 코드를 한데 묶을 수 있습니다.

- 1.5초 timeout
- 상위 HTTP 요청 종료
- 운영자가 수동으로 취소한 작업

이렇게 합쳐 두면 하위 함수는 단일 `signal`만 받아도 됩니다.
API 설계가 단순해지고, 취소 조건이 늘어나도 호출부에서 조합하기 쉬워집니다.

### H3. cancellation 로직이 함수 시그니처를 덜 복잡하게 만든다

실무에서 흔한 안 좋은 패턴은 함수마다 `timeoutMs`, `isCancelled`, `requestClosed`, `shutdownRequested` 같은 인자를 따로 넘기는 것입니다.
이 방식은 조건이 늘수록 코드가 빠르게 지저분해집니다.

반면 `AbortSignal.any()`를 쓰면 함수는 아래처럼 단순해집니다.

```js
async function fetchUserProfile(userId, { signal }) {
  const res = await fetch(`https://api.example.com/users/${userId}`, {
    signal,
  });

  if (!res.ok) {
    throw new Error(`upstream error: ${res.status}`);
  }

  return res.json();
}
```

핵심은 하위 함수가 취소 사유를 일일이 몰라도 된다는 점입니다.
취소 조건의 조합은 상위 orchestration 레이어에서 처리하는 편이 더 깔끔합니다.

## Node.js에서 어떻게 쓰면 좋을까

### H3. timeout signal과 상위 요청 signal을 합치는 기본 패턴

아래 예시는 상위 요청에서 내려온 `requestSignal`과 1.5초 timeout을 하나로 합치는 방식입니다.

```js
async function fetchInventory(itemId, { requestSignal }) {
  const timeoutSignal = AbortSignal.timeout(1500);
  const combinedSignal = AbortSignal.any([
    requestSignal,
    timeoutSignal,
  ]);

  const res = await fetch(`https://inventory.internal/items/${itemId}`, {
    signal: combinedSignal,
  });

  if (!res.ok) {
    throw new Error(`inventory error: ${res.status}`);
  }

  return res.json();
}
```

이 패턴의 장점은 분명합니다.
클라이언트가 먼저 떠나도 취소되고, 서버가 정한 deadline을 넘겨도 취소됩니다.
둘 중 어느 쪽이 먼저 오든 하위 fetch는 계속 남아 있지 않습니다.

### H3. Express나 Fastify에서 요청 종료를 signal로 바꾸는 편이 좋다

프레임워크에 따라 구현은 조금 다르지만, 요청 종료 이벤트를 `AbortController`로 감싸 두면 재사용하기 쉽습니다.

```js
function createRequestSignal(req) {
  const controller = new AbortController();

  req.on('close', () => {
    controller.abort(new Error('client disconnected'));
  });

  return controller.signal;
}

app.get('/orders/:id', async (req, res) => {
  const requestSignal = createRequestSignal(req);
  const timeoutSignal = AbortSignal.timeout(2000);
  const signal = AbortSignal.any([requestSignal, timeoutSignal]);

  try {
    const response = await fetch(`https://orders.internal/${req.params.id}`, {
      signal,
    });

    const data = await response.json();
    res.json(data);
  } catch (error) {
    if (signal.aborted) {
      return;
    }

    res.status(502).json({ message: 'upstream failure' });
  }
});
```

이 구조를 잡아두면 같은 request signal을 DB 작업, 큐 대기, 외부 API 호출 등에도 일관되게 전달할 수 있습니다.
관련 기본기는 [Node.js AbortController timeout/cancellation pattern 가이드](/development/blog/seo/2026/03/19/nodejs-abortcontroller-timeout-cancellation-pattern-guide.html)와 함께 보면 더 이해가 쉽습니다.

## 실무에서 자주 놓치는 포인트

### H3. abort 자체를 일반 에러처럼 취급하면 로그가 시끄러워진다

정상적인 취소까지 모두 에러로 집계하면 운영 지표가 왜곡됩니다.
특히 사용자가 이미 페이지를 떠났는데 `AbortError`를 5xx처럼 세면 장애처럼 보일 수 있습니다.

그래서 보통 아래처럼 구분합니다.

- abort: 예상 가능한 종료
- timeout: budget 초과로 인한 제어된 종료
- 진짜 오류: DNS, connect, TLS, 5xx, parsing 실패 등

취소는 실패이긴 하지만, **항상 장애는 아닙니다**.
로그 레벨과 알림 기준을 분리해 두는 편이 좋습니다.

### H3. 모든 라이브러리가 signal을 같은 수준으로 지원하는 것은 아니다

Node.js 내장 `fetch`처럼 `signal` 지원이 좋은 API도 있지만, 일부 라이브러리는 중간 단계에서 취소가 깔끔하게 전파되지 않을 수 있습니다.
이 경우 호출부만 취소해도 실제 소켓이나 내부 작업은 늦게 정리될 수 있습니다.

그래서 적용 전에는 아래를 확인하는 편이 안전합니다.

- 사용하는 HTTP 클라이언트가 `signal`을 제대로 지원하는가
- 스트림/파일/DB 작업에도 취소가 실제 반영되는가
- abort 시 자원 정리가 누락되지 않는가
- timeout과 cancellation이 중복 처리되지 않는가

즉 `AbortSignal.any()`는 조합 도구이지, 모든 하위 라이브러리의 취소 품질까지 자동 보장해주지는 않습니다.

### H3. deadline이 있는데 하위 단계별 timeout이 없으면 원인 분리가 어려워진다

상위 요청 signal 하나만 전달하면 cancellation 전파는 좋아집니다.
하지만 모든 하위 단계가 같은 큰 deadline만 공유하면 어디서 시간이 샜는지 보기가 어려울 수 있습니다.

실무에서는 보통 이렇게 가져갑니다.

- 상위 deadline signal은 전체 요청 생명주기를 묶는다
- 하위 작업마다 더 짧은 local timeout을 둔다
- 둘을 `AbortSignal.any()`로 합친다

이렇게 하면 전체 budget을 넘기지 않으면서도, 어떤 단계가 유난히 늦은지 구분하기 쉬워집니다.
이 관점은 [Node.js Deadline Exceeded 에러 처리 가이드](/development/blog/seo/2026/04/12/nodejs-deadline-exceeded-error-handling-guide.html)와도 자연스럽게 이어집니다.

## 어떤 상황에서 특히 효과가 큰가

### H3. hedged request나 fallback 전략에서 남는 작업을 빨리 정리할 수 있다

빠른 응답을 위해 동일 요청을 두 경로로 보내는 hedged request 패턴에서는 먼저 성공한 쪽 외 나머지 작업을 빨리 취소하는 것이 중요합니다.
그렇지 않으면 지연은 줄어도 비용과 부하는 그대로 남습니다.

예를 들어 아래처럼 쓸 수 있습니다.

- 사용자 요청 취소 signal
- 전체 timeout signal
- 우승 응답이 나오면 취소하는 내부 controller signal

이 셋을 합쳐 놓으면 먼저 끝난 결과만 쓰고 나머지 호출은 빠르게 정리할 수 있습니다.
관련 내용은 [Node.js Hedged Requests 가이드](/development/blog/seo/2026/04/08/nodejs-hedged-requests-tail-latency-reduction-guide.html)도 같이 보면 좋습니다.

### H3. 서버 종료 중 inflight 작업 정리에도 응용할 수 있다

배포나 스케일 인 때는 프로세스 종료 signal과 요청별 timeout을 함께 봐야 합니다.
shutdown signal이 왔는데도 하위 외부 호출이 오래 남아 있으면 drain 시간이 길어지고 종료가 지저분해질 수 있습니다.

이때도 요청 signal, shutdown signal, local timeout signal을 합치면 구조가 단순해집니다.
배포 안정성 측면에서는 [Node.js Graceful Shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)와 연결해서 보는 편이 좋습니다.

## 빠르게 적용할 때 쓰는 체크리스트

### H3. 아래 네 가지만 먼저 맞춰도 실수가 크게 줄어든다

- 취소 조건이 2개 이상이면 `AbortSignal.any()`로 합칠 수 있는지 본다.
- 하위 함수는 `signal` 하나만 받게 설계한다.
- `AbortError`와 실제 장애 로그를 분리한다.
- 사용하는 라이브러리가 `signal`을 진짜로 지원하는지 검증한다.

## 마무리

Node.js에서 `AbortSignal.any()`는 작은 문법처럼 보이지만, 실제로는 timeout과 cancellation을 하나의 모델로 정리하게 해주는 도구입니다.
특히 외부 API 호출, 요청 전파, hedged request, graceful shutdown 같은 상황에서는 코드 복잡도를 낮추면서도 자원 낭비를 줄이는 데 도움이 큽니다.

실무에서는 timeout 숫자만 만지기보다 **어떤 순간에 이 작업이 더 이상 필요 없어지는지**부터 정의해보세요.
그다음 그 조건들을 `AbortSignal.any()`로 묶으면, cancellation 설계가 훨씬 명확해집니다.

관련해서 함께 보면 좋은 글은 아래입니다.

- [Node.js AbortController timeout/cancellation pattern 가이드](/development/blog/seo/2026/03/19/nodejs-abortcontroller-timeout-cancellation-pattern-guide.html)
- [Node.js Timeout Budget, Deadline Propagation 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)
- [Node.js Deadline Exceeded 에러 처리 가이드](/development/blog/seo/2026/04/12/nodejs-deadline-exceeded-error-handling-guide.html)
- [Node.js Hedged Requests tail latency reduction 가이드](/development/blog/seo/2026/04/08/nodejs-hedged-requests-tail-latency-reduction-guide.html)

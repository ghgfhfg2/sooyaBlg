---
layout: post
title: "Node.js EventSource 가이드: 서버 전송 이벤트(SSE) 클라이언트를 안전하게 붙이는 법"
date: 2026-08-23 08:00:00 +0900
lang: ko
translation_key: nodejs-eventsource-sse-client-guide
permalink: /development/blog/seo/2026/08/23/nodejs-eventsource-sse-client-guide.html
alternates:
  ko: /development/blog/seo/2026/08/23/nodejs-eventsource-sse-client-guide.html
  x_default: /development/blog/seo/2026/08/23/nodejs-eventsource-sse-client-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, eventsource, sse, server-sent-events, streaming, backend, realtime, javascript]
description: "Node.js EventSource로 서버 전송 이벤트(SSE) 클라이언트를 구성하는 방법을 정리합니다. 실험적 API 플래그, 메시지 처리, 재연결, 타임아웃, WebSocket과의 선택 기준까지 실무 예제로 설명합니다."
---

실시간 기능을 만들 때 항상 WebSocket이 정답은 아닙니다.
서버가 상태 변경, 알림, 진행률, 로그 스트림처럼 **단방향 이벤트**를 계속 보내면 클라이언트가 받기만 해도 충분한 경우가 많습니다.
이럴 때 HTTP 기반의 서버 전송 이벤트(Server-Sent Events, SSE)를 검토할 수 있습니다.

Node.js에는 브라우저 호환 형태의 `EventSource` 전역 객체가 제공됩니다.
다만 현재 Node.js 공식 문서 기준으로 `EventSource`는 실험적 API이며, 실행 시 `--experimental-eventsource` 플래그가 필요합니다.
따라서 운영 코드에 바로 고정하기보다 버전 정책, 장애 처리, 대체 구현을 함께 검토하는 것이 안전합니다.

이 글에서는 Node.js에서 `EventSource`로 SSE 클라이언트를 붙이는 기본 구조와 실무에서 놓치기 쉬운 재연결, 타임아웃, 메시지 검증 기준을 정리합니다.
양방향 통신이 필요하다면 [Node.js 내장 WebSocket 클라이언트 가이드](/development/blog/seo/2026/05/14/nodejs-built-in-websocket-client-guide.html), 일반 HTTP 요청의 취소와 재시도는 [Node.js fetch AbortSignal 타임아웃 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html), 여러 취소 신호 조합은 [Node.js AbortSignal.any 타임아웃 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)와 함께 보면 좋습니다.

## EventSource와 SSE의 역할

### 서버가 계속 보내고 클라이언트는 듣는다

SSE는 클라이언트가 HTTP 요청을 열어두면 서버가 `text/event-stream` 형식으로 이벤트를 계속 보내는 방식입니다.
브라우저의 `EventSource` API와 같은 모델이라서, 서버에서 이벤트 이름과 데이터 조각을 흘려보내고 클라이언트는 메시지 이벤트를 받아 처리합니다.

대표적인 사용처는 아래처럼 서버에서 클라이언트로 흐르는 정보입니다.

- 배치 작업 진행률
- 주문, 결제, 배송 상태 변경 알림
- 운영 대시보드의 상태 이벤트
- AI 응답 생성 진행 로그
- 긴 작업의 단계별 결과 스트림

반대로 클라이언트가 서버로도 자주 메시지를 보내야 한다면 SSE보다 WebSocket이 자연스럽습니다.
SSE는 단방향 스트림에 강하고, WebSocket은 양방향 세션에 강합니다.

### Node.js에서는 플래그와 버전 정책을 먼저 확인한다

Node.js의 `EventSource`는 전역 객체로 노출되지만 실험적 API입니다.
실행할 때 아래처럼 플래그를 켜야 합니다.

```bash
node --experimental-eventsource sse-client.js
```

이 점은 배포 스크립트와 런타임 문서에 명확히 남겨야 합니다.
로컬에서는 잘 동작했는데 프로덕션 프로세스 매니저에는 플래그가 빠져 `ReferenceError: EventSource is not defined`가 나는 식의 장애가 생길 수 있습니다.

운영 안정성이 더 중요하다면 현재 Node.js 버전에서의 내장 API 상태를 확인하고, 필요하면 검증된 서드파티 패키지나 `fetch()` 기반 스트리밍 구현을 대안으로 두는 편이 좋습니다.
핵심은 "내장 API를 쓴다"보다 "현재 배포 환경에서 어떤 계약으로 동작하는지 고정한다"입니다.

## 기본 클라이언트 구현

### 메시지와 에러를 분리해서 처리한다

가장 단순한 SSE 클라이언트는 `message`와 `error` 이벤트를 구독합니다.
서버가 JSON 문자열을 보낸다고 가정하면, 파싱 실패도 정상적인 장애 경로로 다뤄야 합니다.

```js
const url = new URL('https://example.com/events');
url.searchParams.set('topic', 'build-status');

const source = new EventSource(url.href);

source.addEventListener('message', (event) => {
  try {
    const payload = JSON.parse(event.data);

    console.log('event received', {
      id: event.lastEventId,
      type: payload.type,
      status: payload.status
    });
  } catch (error) {
    console.warn('invalid sse payload', {
      id: event.lastEventId,
      reason: error instanceof Error ? error.message : 'unknown'
    });
  }
});

source.addEventListener('error', () => {
  console.warn('sse connection error', {
    readyState: source.readyState
  });
});
```

SSE 메시지는 외부 입력입니다.
내부 서비스에서 보낸 이벤트라도 JSON 파싱, 필수 필드, 타입, 값 범위는 검사하는 편이 좋습니다.
특히 이벤트 데이터를 그대로 SQL, 쉘 명령, HTML, 로그 원문에 넣지 않도록 경계를 둬야 합니다.

### 커스텀 이벤트 이름을 구독한다

SSE 서버는 기본 `message` 외에도 이벤트 이름을 지정할 수 있습니다.
예를 들어 `progress`, `done`, `failed`처럼 이벤트를 나누면 클라이언트 코드가 더 읽기 쉬워집니다.

```js
const source = new EventSource('https://example.com/jobs/123/events');

source.addEventListener('progress', (event) => {
  const payload = JSON.parse(event.data);
  console.log(`progress: ${payload.percent}%`);
});

source.addEventListener('done', (event) => {
  const payload = JSON.parse(event.data);
  console.log('job done', payload.resultId);
  source.close();
});

source.addEventListener('failed', (event) => {
  const payload = JSON.parse(event.data);
  console.error('job failed', payload.code);
  source.close();
});
```

종료 조건이 있는 스트림이라면 `done`이나 `failed` 이벤트에서 `source.close()`를 호출하세요.
닫지 않은 연결은 프로세스가 살아 있는 동안 계속 유지될 수 있고, 테스트나 CLI 작업에서는 종료되지 않는 프로세스의 원인이 됩니다.

## 운영 코드에서 보강할 부분

### 애플리케이션 수준 타임아웃을 둔다

`EventSource`는 연결 유지와 재연결을 API가 처리하는 모델입니다.
하지만 업무 요구사항상 "이 작업은 2분 안에 끝나야 한다" 같은 상한은 별도로 둬야 합니다.

```js
export function waitForJobDone(jobId, { timeoutMs = 120_000 } = {}) {
  return new Promise((resolve, reject) => {
    const source = new EventSource(`https://example.com/jobs/${jobId}/events`);

    const timeout = setTimeout(() => {
      source.close();
      reject(new Error(`Timed out waiting for job ${jobId}`));
    }, timeoutMs);

    source.addEventListener('done', (event) => {
      clearTimeout(timeout);
      source.close();
      resolve(JSON.parse(event.data));
    });

    source.addEventListener('failed', (event) => {
      clearTimeout(timeout);
      source.close();
      reject(new Error(`Job failed: ${event.data}`));
    });

    source.addEventListener('error', () => {
      if (source.readyState === EventSource.CLOSED) {
        clearTimeout(timeout);
        reject(new Error(`SSE closed before job ${jobId} completed`));
      }
    });
  });
}
```

여기서 타임아웃은 네트워크 연결 하나의 타임아웃이 아니라 비즈니스 작업의 전체 대기 시간입니다.
HTTP 스트림이 계속 재연결되더라도 사용자가 기다릴 수 있는 상한은 별도로 관리해야 합니다.

### Last-Event-ID와 멱등 처리를 고려한다

SSE는 재연결을 전제로 합니다.
서버가 이벤트 `id`를 보내면 클라이언트는 이후 재연결 시 마지막 이벤트 ID를 기준으로 이어 받을 수 있습니다.
하지만 이 기능이 실제로 의미 있으려면 서버도 이벤트 로그를 일정 시간 보관하고, 같은 이벤트가 다시 와도 클라이언트 처리가 망가지지 않아야 합니다.

실무에서는 아래 기준을 권장합니다.

- 이벤트마다 안정적인 `id`를 부여한다
- 클라이언트는 마지막 처리 ID를 메모리나 저장소에 기록한다
- 같은 이벤트 ID가 다시 와도 한 번만 반영한다
- 서버는 재전송 가능한 보관 기간을 문서화한다

진행률 표시처럼 덮어쓰기 가능한 이벤트는 중복에 강합니다.
반면 결제 완료, 포인트 적립, 외부 API 호출처럼 부작용이 있는 작업은 이벤트 소비 쪽에 멱등성을 반드시 넣어야 합니다.

### 인증 정보는 URL보다 헤더와 세션 정책으로 관리한다

브라우저 API와의 호환성 때문에 `EventSource`는 일반적인 `fetch()`처럼 요청 헤더를 자유롭게 넣는 모델이 아닙니다.
그래서 토큰을 쿼리 문자열에 붙이고 싶어질 수 있습니다.
하지만 URL은 로그, 리퍼러, 모니터링 도구에 남기 쉬워 민감한 토큰을 넣기에 좋지 않습니다.

가능하면 아래 순서로 검토하세요.

1. 같은 출처 세션 쿠키 기반 인증
2. 짧은 수명의 스트림 전용 토큰
3. 토큰 원문을 남기지 않는 서버 로그 마스킹
4. 연결별 권한과 만료 시간 검증

내부 시스템끼리 연결하더라도 URL에 장기 토큰을 넣는 방식은 피하는 편이 안전합니다.
SSE 스트림은 오래 유지되므로 인증 만료, 권한 변경, 강제 종료 정책까지 같이 설계해야 합니다.

## WebSocket, fetch 스트림과 선택 기준

### 단방향 알림이면 EventSource가 단순하다

SSE의 장점은 HTTP 기반이라 운영 모델이 비교적 단순하다는 점입니다.
서버가 클라이언트에게 이벤트만 보내면 되는 경우에는 WebSocket보다 구현과 디버깅이 가벼울 수 있습니다.
프록시와 로드밸런서 환경에서도 HTTP 스트리밍 정책만 맞추면 흐름을 이해하기 쉽습니다.

다만 연결 수가 많아질수록 서버 리소스, idle timeout, 프록시 버퍼링 정책이 중요해집니다.
프록시가 `text/event-stream` 응답을 버퍼링하면 이벤트가 즉시 도착하지 않을 수 있으므로, 배포 환경에서 실제 스트림 지연을 측정해야 합니다.

### 양방향 상호작용이면 WebSocket을 검토한다

채팅, 실시간 협업 편집, 게임 상태 동기화처럼 클라이언트와 서버가 모두 자주 메시지를 보내야 한다면 WebSocket이 더 자연스럽습니다.
SSE로도 클라이언트 발신은 별도 `fetch()` 요청을 섞어 구현할 수 있지만, 상호작용이 잦아질수록 흐름이 복잡해집니다.

선택 기준은 간단합니다.
서버에서 클라이언트로만 지속 이벤트가 흐르면 SSE를 먼저 검토하고, 양방향 세션 자체가 도메인 모델이면 WebSocket을 검토하세요.
둘 다 과하면 일반 `fetch()` 폴링에 `ETag`, `If-None-Match`, 백오프를 붙이는 방식이 더 운영하기 쉬울 수 있습니다.

## 배포 전 체크리스트

### 런타임과 장애 경로를 확인한다

배포 전에는 기능 동작보다 운영 조건을 먼저 고정해야 합니다.

- Node.js 실행 옵션에 `--experimental-eventsource`가 포함됐는가?
- 실험적 API 사용 사실과 대체 경로를 문서화했는가?
- 연결 종료 조건에서 `source.close()`를 호출하는가?
- 전체 대기 시간 타임아웃이 있는가?
- 메시지 JSON 파싱 실패와 필드 검증 실패를 구분하는가?
- 이벤트 중복 수신 시 부작용이 한 번만 발생하는가?
- URL, 로그, 에러 메시지에 민감한 토큰이 남지 않는가?
- 프록시와 로드밸런서가 `text/event-stream`을 버퍼링하지 않는가?

SSE는 코드 몇 줄로 시작할 수 있지만, 운영에서는 오래 열린 연결을 다루는 기술입니다.
연결 수, 재연결, 인증 만료, 중복 이벤트, 프록시 정책을 함께 확인하면 단방향 실시간 기능을 훨씬 안정적으로 붙일 수 있습니다.

## FAQ

### Node.js EventSource는 운영에서 바로 써도 되나요?

현재 공식 문서 기준으로 실험적 API이므로 조직의 Node.js 버전 정책과 장애 대응 기준에 맞춰 판단해야 합니다.
운영에 쓰려면 플래그, 대체 구현, 회귀 테스트, 배포 문서까지 같이 준비하는 편이 안전합니다.

### SSE와 WebSocket 중 무엇을 먼저 선택해야 하나요?

서버가 클라이언트에게 이벤트를 보내기만 하면 SSE가 단순합니다.
클라이언트와 서버가 모두 자주 메시지를 주고받는다면 WebSocket이 더 자연스럽습니다.

### SSE 메시지는 신뢰해도 되나요?

그렇지 않습니다.
내부 서비스에서 온 메시지라도 외부 입력처럼 파싱하고 검증해야 합니다.
특히 토큰, 개인정보, 원문 payload를 로그에 그대로 남기지 않는 것이 중요합니다.

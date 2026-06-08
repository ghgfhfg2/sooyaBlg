---
layout: post
title: "Node.js process.getActiveResourcesInfo 가이드: 프로세스 종료 지연 원인 찾기"
date: 2026-06-09 08:00:00 +0900
lang: ko
translation_key: nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide
permalink: /development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html
alternates:
  ko: /development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html
  x_default: /development/blog/seo/2026/06/09/nodejs-process-getactiveresourcesinfo-exit-delay-diagnostics-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, process, getactiveresourcesinfo, diagnostics, gracefulshutdown, backend]
description: "Node.js process.getActiveResourcesInfo()로 이벤트 루프를 붙잡는 활성 리소스 유형을 확인하고, 프로세스 종료 지연과 테스트 멈춤 원인을 진단하는 방법을 예제로 정리합니다."
---

Node.js 서버를 종료했는데 프로세스가 계속 남아 있거나, 테스트가 모두 통과한 뒤 명령이 끝나지 않는 문제가 종종 발생합니다.
이때 원인은 대개 아직 정리되지 않은 타이머, 서버 소켓, 네트워크 연결, 파일 감시자처럼 이벤트 루프를 활성 상태로 유지하는 리소스입니다.

`process.getActiveResourcesInfo()`는 현재 Node.js 이벤트 루프를 유지하고 있는 활성 리소스의 **유형 이름**을 배열로 반환합니다.
개별 객체나 민감한 연결 정보를 노출하지 않으면서도 어떤 종류의 리소스가 프로세스 종료를 막는지 빠르게 좁힐 수 있습니다.

이 글에서는 `process.getActiveResourcesInfo()`의 기본 사용법, 종료 지연을 재현하고 진단하는 방법, 결과를 해석할 때 주의할 점, graceful shutdown과 테스트에 적용하는 패턴을 정리합니다.
종료 신호를 받은 뒤 요청을 안전하게 비우는 전체 흐름이 필요하다면 [Node.js graceful shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)를 함께 참고하세요.

## process.getActiveResourcesInfo가 필요한 이유

### H3. 정상 종료는 남아 있는 작업이 없어야 가능하다

Node.js는 실행할 JavaScript 코드가 끝났더라도 이벤트 루프를 유지하는 리소스가 남아 있으면 프로세스를 바로 종료하지 않습니다.
대표적인 예는 다음과 같습니다.

- 종료하지 않은 HTTP 서버
- 해제하지 않은 `setInterval()` 타이머
- 닫지 않은 소켓과 메시지 포트
- 중단하지 않은 파일 시스템 감시자
- 아직 진행 중인 비동기 작업

문제는 애플리케이션이 커질수록 어떤 모듈이 리소스를 만들었고 정리하지 않았는지 찾기 어렵다는 점입니다.
`getActiveResourcesInfo()`를 사용하면 먼저 리소스 유형을 확인해 조사 범위를 줄일 수 있습니다.

```js
console.log(process.getActiveResourcesInfo());
```

출력은 실행 시점과 환경에 따라 달라질 수 있습니다.
예를 들어 타이머가 남아 있다면 `Timeout`, 서버가 열려 있다면 서버나 TCP 관련 유형이 포함될 수 있습니다.

### H3. 개별 리소스가 아니라 유형 목록을 제공한다

이 API는 활성 리소스 객체 자체가 아니라 문자열 유형 목록을 반환합니다.
따라서 어떤 타이머 콜백이 문제인지, 어느 코드 줄에서 소켓을 열었는지까지 직접 알려 주지는 않습니다.

대신 다음 질문에 빠르게 답할 수 있습니다.

- 종료 직전에 타이머가 여전히 남아 있는가?
- 서버를 닫은 뒤 TCP 관련 리소스가 줄었는가?
- 테스트 전후에 특정 리소스 유형의 개수가 증가했는가?
- 종료 처리 단계 중 어느 시점부터 리소스가 남는가?

즉, `getActiveResourcesInfo()`는 원인을 확정하는 도구라기보다 다음 조사 방향을 정하는 진단 도구입니다.

## 기본 사용법

### H3. 정리되지 않은 interval을 확인한다

아래 예제는 `setInterval()`을 만들고 활성 리소스 정보를 확인합니다.

```js
const interval = setInterval(() => {
  // 주기 작업
}, 60_000);

console.log(process.getActiveResourcesInfo());

clearInterval(interval);
```

`clearInterval()`을 호출하지 않으면 주기 작업이 이벤트 루프를 계속 유지해 프로세스가 종료되지 않을 수 있습니다.
실제 서비스에서는 캐시 정리, 통계 집계, 상태 확인 같은 주기 작업을 시작할 때 종료 함수도 함께 제공하는 편이 좋습니다.

```js
export function startMaintenanceJob() {
  const interval = setInterval(runMaintenance, 60_000);

  return function stopMaintenanceJob() {
    clearInterval(interval);
  };
}
```

리소스를 만드는 코드와 정리하는 코드를 한 모듈 안에 두면 shutdown 과정에서 누락될 가능성을 줄일 수 있습니다.

### H3. 유형별 개수를 요약한다

원본 배열은 같은 유형 이름을 여러 번 포함할 수 있습니다.
운영 로그에서는 유형별 개수로 요약하면 전후 변화를 비교하기 쉽습니다.

```js
export function summarizeActiveResources() {
  const summary = {};

  for (const type of process.getActiveResourcesInfo()) {
    summary[type] = (summary[type] ?? 0) + 1;
  }

  return summary;
}

console.info({
  event: 'active-resources-snapshot',
  resources: summarizeActiveResources()
});
```

이 방식은 리소스 객체, 요청 데이터, 연결 대상 주소를 기록하지 않습니다.
진단에 필요한 유형과 개수만 남기므로 로그에 민감정보가 섞일 위험도 낮출 수 있습니다.

## 종료 지연 진단에 적용하기

### H3. shutdown 단계별 스냅샷을 비교한다

프로세스 종료가 늦다면 종료 처리 전, 서버 종료 후, 백그라운드 작업 정리 후처럼 단계별로 활성 리소스를 기록합니다.

```js
import http from 'node:http';

const server = http.createServer((request, response) => {
  response.end('ok');
});

server.listen(3000);

function closeServer() {
  return new Promise((resolve, reject) => {
    server.close((error) => {
      if (error) reject(error);
      else resolve();
    });
  });
}

process.once('SIGTERM', async () => {
  console.info({
    event: 'shutdown-start',
    resources: summarizeActiveResources()
  });

  await closeServer();

  console.info({
    event: 'server-closed',
    resources: summarizeActiveResources()
  });
});
```

첫 번째 로그와 두 번째 로그를 비교하면 서버 종료로 어떤 리소스 유형이 줄었는지 확인할 수 있습니다.
그래도 프로세스가 남는다면 데이터베이스 풀, 메시지 소비자, 주기 작업, 워커 같은 다른 종료 대상을 차례로 점검합니다.

### H3. 종료 시간 제한과 함께 사용한다

운영 환경에서는 종료가 무기한 기다리지 않도록 전체 시간 제한을 두는 경우가 많습니다.
시간 제한 직전에 활성 리소스 요약을 남기면 강제 종료가 발생한 이유를 사후 분석하기 쉽습니다.

```js
const shutdownTimeout = setTimeout(() => {
  console.error({
    event: 'shutdown-timeout',
    resources: summarizeActiveResources()
  });

  process.exit(1);
}, 10_000);

shutdownTimeout.unref();
```

`unref()`를 호출하면 이 진단용 타이머 자체가 프로세스 종료를 붙잡지 않습니다.
다만 `process.exit()`로 즉시 종료하면 아직 처리 중인 로그와 정리 작업까지 끊길 수 있으므로, 강제 종료 정책은 실행 환경의 종료 유예 시간과 함께 설계해야 합니다.

HTTP 연결 정리 기준이 필요하다면 [Node.js closeIdleConnections와 closeAllConnections 가이드](/development/blog/seo/2026/04/16/nodejs-closeidleconnections-closeallconnections-graceful-drain-guide.html)를 참고할 수 있습니다.

## 테스트가 끝나지 않을 때 활용하기

### H3. 테스트 전후 리소스 차이를 확인한다

테스트가 통과했지만 프로세스가 끝나지 않는다면 테스트가 리소스를 새로 만들고 정리하지 않았을 가능성이 있습니다.
테스트 전후의 유형별 개수를 비교하면 누수 후보를 찾는 데 도움이 됩니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

function countResources(type) {
  return process.getActiveResourcesInfo()
    .filter((resourceType) => resourceType === type)
    .length;
}

test('주기 작업을 종료하면 Timeout 수가 원래 수준으로 돌아온다', () => {
  const before = countResources('Timeout');
  const stop = startMaintenanceJob();

  stop();

  const after = countResources('Timeout');
  assert.equal(after, before);
});
```

이 예제는 작은 진단 패턴을 보여 주기 위한 것입니다.
테스트 러너 자체나 병렬 테스트가 리소스를 사용할 수 있으므로, 모든 환경에서 특정 유형의 절대 개수를 고정값으로 단언하면 테스트가 불안정해질 수 있습니다.
가능하면 동일한 테스트 안에서 전후 차이를 비교하고, 자신이 만든 리소스를 명시적으로 정리했는지도 별도로 검증해야 합니다.

### H3. teardown에서 생성한 리소스를 직접 닫는다

활성 리소스 목록을 확인하는 것보다 더 중요한 것은 테스트가 만든 대상을 명시적으로 정리하는 것입니다.

```js
import test from 'node:test';

test('서버 응답 테스트', async (t) => {
  const server = createTestServer();

  await new Promise((resolve) => server.listen(0, resolve));

  t.after(() => {
    return new Promise((resolve, reject) => {
      server.close((error) => {
        if (error) reject(error);
        else resolve();
      });
    });
  });

  // 테스트 요청과 검증
});
```

`getActiveResourcesInfo()`는 누락된 teardown을 찾는 보조 도구로 사용하고, 실제 해결은 서버·타이머·소켓·워커의 공식 종료 API를 호출하는 방식으로 해야 합니다.

## 결과를 해석할 때 주의할 점

### H3. 유형 이름은 업무 원인을 직접 설명하지 않는다

같은 `Timeout` 유형이라도 원인은 재시도 타이머, 주기 작업, 라이브러리 내부 타이머 등 여러 가지일 수 있습니다.
TCP 관련 유형도 HTTP 서버, 외부 API 연결, 데이터베이스 드라이버에서 만들어질 수 있습니다.

유형을 확인한 다음에는 다음 순서로 범위를 좁히는 것이 좋습니다.

1. 최근 추가되거나 변경된 장기 실행 리소스를 확인한다.
2. shutdown 단계마다 스냅샷을 남겨 어느 정리 함수 이후 변화하는지 비교한다.
3. 모듈별 종료 함수를 하나씩 실행하며 남은 유형을 확인한다.
4. 필요하면 프로세스 진단 보고서와 애플리케이션 로그를 함께 분석한다.

더 깊은 런타임 상태가 필요하다면 [Node.js process report 운영 진단 가이드](/development/blog/seo/2026/05/29/nodejs-process-report-production-diagnostics-guide.html)를 함께 사용할 수 있습니다.

### H3. 정상적인 활성 리소스도 포함될 수 있다

실행 중인 서버에서 활성 리소스가 존재하는 것은 정상입니다.
문제는 리소스가 있다는 사실 자체가 아니라, 종료 단계가 끝난 뒤에도 예상하지 않은 리소스가 남아 프로세스를 유지하는 상황입니다.

따라서 운영 중 주기적으로 결과를 수집해 "0이 아니면 장애"로 판단하면 안 됩니다.
다음처럼 상황에 맞는 기준을 사용해야 합니다.

- 정상 실행 중에는 기준선과 급격한 증가 여부를 본다.
- 종료 중에는 정리 단계별 감소 여부를 본다.
- 테스트에서는 자신이 만든 리소스의 전후 차이를 본다.
- 장애 분석에서는 다른 로그와 지표를 함께 본다.

## 실무 적용 체크리스트

### H3. 종료 가능한 리소스의 소유권을 명확히 한다

- 타이머, 서버, 워커, 감시자를 만든 모듈이 종료 함수도 제공한다.
- shutdown 진입 후 새 작업과 새 연결을 받지 않는다.
- 종료 함수는 여러 번 호출돼도 안전하도록 설계한다.
- 전체 종료 시간 제한과 실행 환경의 유예 시간을 맞춘다.
- 시간 초과 직전에는 활성 리소스 유형 요약을 기록한다.

### H3. 진단 로그는 최소 정보만 남긴다

- 리소스 유형과 개수처럼 조사에 필요한 정보만 기록한다.
- 요청 본문, 인증 토큰, 사용자 식별자, 연결 문자열은 기록하지 않는다.
- 정상 운영과 종료 과정의 로그 이벤트 이름을 구분한다.
- 절대 개수보다 기준선과 단계별 변화를 함께 본다.

## FAQ

### H3. getActiveResourcesInfo 결과만으로 누수 위치를 찾을 수 있나요?

아닙니다.
결과는 활성 리소스의 유형을 알려 주지만 생성 위치나 개별 객체는 제공하지 않습니다.
유형을 단서로 삼아 shutdown 단계 로그, 테스트 teardown, 프로세스 보고서 등을 함께 확인해야 합니다.

### H3. 활성 리소스가 하나라도 있으면 문제인가요?

아닙니다.
서버가 정상 실행 중이라면 소켓과 타이머 같은 활성 리소스가 존재할 수 있습니다.
종료가 완료돼야 하는 시점에 예상하지 않은 리소스가 남아 있는지가 중요합니다.

### H3. process.exit()를 호출하면 종료 지연을 해결할 수 있나요?

즉시 프로세스를 끝낼 수는 있지만 근본 원인을 해결하지는 못합니다.
처리 중인 요청, 로그, 데이터 쓰기가 끊길 수 있으므로 먼저 리소스를 정상적으로 정리하고, 제한 시간을 넘긴 예외 상황에서만 강제 종료 정책을 적용해야 합니다.

## 마무리

`process.getActiveResourcesInfo()`는 Node.js 프로세스가 왜 아직 살아 있는지 조사할 때 첫 단서를 제공하는 가벼운 진단 API입니다.
유형별 개수를 요약하고 shutdown 단계별로 비교하면 정리되지 않은 타이머, 서버, 소켓 같은 후보를 빠르게 좁힐 수 있습니다.

다만 이 API는 개별 원인을 자동으로 찾아 주지 않습니다.
리소스를 만든 모듈이 명시적인 종료 함수를 제공하고, 테스트 teardown과 graceful shutdown에서 그 함수를 빠짐없이 호출하는 구조가 근본적인 해결책입니다.

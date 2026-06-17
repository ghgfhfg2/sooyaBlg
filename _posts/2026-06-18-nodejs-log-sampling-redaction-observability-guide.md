---
layout: post
title: "Node.js 로그 샘플링 가이드: 운영 비용과 민감정보 리스크 줄이기"
date: 2026-06-18 08:00:00 +0900
lang: ko
translation_key: nodejs-log-sampling-redaction-observability-guide
permalink: /development/blog/seo/2026/06/18/nodejs-log-sampling-redaction-observability-guide.html
alternates:
  ko: /development/blog/seo/2026/06/18/nodejs-log-sampling-redaction-observability-guide.html
  x_default: /development/blog/seo/2026/06/18/nodejs-log-sampling-redaction-observability-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, logging, observability, sampling, redaction, monitoring, backend]
description: "Node.js 서비스에서 로그 샘플링과 필드 마스킹을 함께 설계해 운영 비용, 노이즈, 민감정보 노출 위험을 줄이는 방법을 정리합니다. 요청 로그, 성능 진단 로그, 오류 로그를 나누어 실무 예제로 설명합니다."
---

Node.js 서비스를 운영하다 보면 로그를 더 많이 남기고 싶은 순간이 자주 옵니다.
장애를 빨리 파악하려면 요청 경로, 상태 코드, 지연 시간, 리소스 사용량, 에러 원인 같은 정보가 필요합니다.
하지만 모든 요청의 모든 정보를 그대로 남기면 로그 비용이 빠르게 늘고, 중요한 신호는 노이즈에 묻히며, 사용자 입력이나 토큰 같은 민감정보가 섞일 위험도 커집니다.

로그 샘플링은 "대충 일부만 버리는 기법"이 아닙니다.
어떤 로그는 항상 남기고, 어떤 로그는 비율로 남기며, 어떤 로그는 임계값을 넘을 때만 남기는 운영 정책입니다.
여기에 필드 마스킹과 허용 목록 기반 구조화를 함께 적용해야 비용과 안전을 동시에 잡을 수 있습니다.

이 글에서는 Node.js 서비스에서 요청 로그, 성능 진단 로그, 오류 로그를 어떻게 나누고, 샘플링과 마스킹을 어떤 순서로 적용할지 정리합니다.
요청 단위 컨텍스트를 먼저 정리하고 싶다면 [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)를 함께 참고하세요.

## 로그 샘플링이 필요한 이유

### H3. 로그를 많이 남기는 것과 관측성이 좋은 것은 다르다

운영 초기에 흔한 실수는 "나중에 필요할 수 있으니 전부 남기자"입니다.
처음에는 편해 보이지만 트래픽이 늘면 로그 저장 비용, 검색 비용, 대시보드 지연, 알림 피로가 같이 늘어납니다.

좋은 로그는 양이 많은 로그가 아니라, 문제를 좁히는 데 필요한 필드를 안정적으로 담은 로그입니다.
예를 들어 정상 요청은 낮은 비율로 샘플링하고, 실패 요청과 느린 요청은 항상 남기면 비용을 줄이면서도 장애 분석에 필요한 단서를 확보할 수 있습니다.

```js
function shouldLogRequest({ statusCode, elapsedMs, sampleRate }) {
  if (statusCode >= 500) return true;
  if (statusCode >= 400) return true;
  if (elapsedMs >= 1000) return true;

  return Math.random() < sampleRate;
}
```

이 함수의 핵심은 정상 요청과 비정상 요청을 같은 비율로 다루지 않는 것입니다.
장애 단서는 대부분 실패, 지연, 재시도, 리소스 급증 같은 이벤트에 몰려 있기 때문에 그런 로그는 샘플링에서 제외하는 편이 실무적입니다.

### H3. 샘플링 전에 필드 정책부터 정해야 한다

샘플링은 저장량을 줄여 주지만 민감정보 문제를 자동으로 해결하지 않습니다.
10%만 저장해도 그 10% 안에 토큰, 세션 쿠키, 이메일, 전화번호, 원문 요청 본문이 들어가면 여전히 위험합니다.

그래서 로그 정책은 다음 순서로 잡는 편이 좋습니다.

1. 로그에 허용할 필드 목록을 정한다.
2. 민감할 수 있는 필드는 마스킹하거나 해시한다.
3. 정상 로그와 비정상 로그의 샘플링 기준을 나눈다.
4. 보관 기간과 접근 권한을 별도로 제한한다.

```js
const ALLOWED_REQUEST_FIELDS = [
  'requestId',
  'method',
  'route',
  'statusCode',
  'elapsedMs',
  'userAgentFamily',
  'sampled'
];

function pickAllowedFields(source, allowedFields) {
  return Object.fromEntries(
    allowedFields
      .filter((field) => Object.hasOwn(source, field))
      .map((field) => [field, source[field]])
  );
}
```

허용 목록 방식은 처음에는 번거롭지만, 장기적으로는 가장 관리하기 쉽습니다.
새 필드를 추가할 때마다 "이 값을 운영 로그에 남겨도 되는가"를 코드 리뷰에서 확인할 수 있기 때문입니다.

## 요청 로그 샘플링 설계

### H3. 정상 요청은 낮은 비율로 남긴다

정상 2xx, 3xx 요청은 전체 트래픽 흐름과 대표 샘플을 보는 용도로 충분한 경우가 많습니다.
서비스 규모에 따라 다르지만, 초고QPS 경로에서는 1% 이하 샘플링도 의미 있는 분포를 보여 줄 수 있습니다.

```js
function getRouteSampleRate(route) {
  if (route === 'GET /health') return 0;
  if (route.startsWith('GET /assets/')) return 0.001;
  if (route === 'POST /checkout') return 0.1;

  return 0.02;
}
```

헬스체크나 정적 자산처럼 반복 호출이 많은 경로는 별도 정책을 두는 편이 좋습니다.
반대로 결제, 가입, 배치 트리거처럼 비즈니스 영향이 큰 경로는 정상 요청도 더 높은 비율로 남길 수 있습니다.

### H3. 실패와 지연은 샘플링하지 않는다

오류와 지연은 운영 분석의 핵심 신호입니다.
특히 5xx 응답, 다운스트림 타임아웃, 큐 처리 실패, p95 이상의 느린 요청은 샘플링하지 않고 남기는 편이 안전합니다.

```js
function classifyRequestLog({ statusCode, elapsedMs }) {
  if (statusCode >= 500) return 'error';
  if (statusCode >= 400) return 'warn';
  if (elapsedMs >= 1000) return 'slow';
  return 'normal';
}

function shouldKeepRequestLog(entry) {
  const level = classifyRequestLog(entry);

  if (level !== 'normal') return true;

  return Math.random() < getRouteSampleRate(entry.route);
}
```

이렇게 분리하면 알림과 대시보드도 단순해집니다.
`error`, `warn`, `slow`, `normal` 같은 분류가 있으면 검색 쿼리와 집계 기준을 팀이 공유하기 쉬워집니다.

### H3. 샘플링 여부를 로그에 남긴다

샘플링된 로그에는 `sampled: true` 같은 필드를 넣어 두는 편이 좋습니다.
그래야 나중에 집계할 때 "이 숫자가 전체 건수인지, 샘플인지"를 혼동하지 않습니다.

```js
function buildRequestLog(input) {
  const level = classifyRequestLog(input);
  const sampled = level === 'normal';

  return pickAllowedFields(
    {
      requestId: input.requestId,
      method: input.method,
      route: input.route,
      statusCode: input.statusCode,
      elapsedMs: Math.round(input.elapsedMs),
      userAgentFamily: normalizeUserAgent(input.userAgent),
      sampled
    },
    ALLOWED_REQUEST_FIELDS
  );
}
```

샘플링된 로그로 전체 요청 수를 추정할 수는 있지만, 과금이나 법적 감사처럼 정확한 카운트가 필요한 영역에는 쓰면 안 됩니다.
정확한 수치는 메트릭 카운터나 이벤트 저장소처럼 별도의 신뢰 경로로 관리해야 합니다.

## 민감정보를 줄이는 필드 설계

### H3. 원문 body와 header는 기본적으로 남기지 않는다

장애 분석을 위해 요청 본문이나 헤더 전체를 남기고 싶은 유혹이 있습니다.
하지만 이 영역에는 비밀번호, 토큰, 쿠키, 이메일, 결제 관련 식별자, 내부 테스트 값이 섞이기 쉽습니다.

운영 로그는 원문을 저장하는 장소가 아니라 진단에 필요한 요약을 저장하는 장소로 봐야 합니다.
본문 전체 대신 스키마 이름, 검증 실패 필드명, payload 크기, 파일 개수처럼 안전한 요약값을 남깁니다.

```js
function summarizeValidationError(error) {
  return {
    errorName: error.name,
    invalidFields: error.issues.map((issue) => issue.path.join('.')),
    issueCount: error.issues.length
  };
}
```

여기서도 입력값 자체는 남기지 않습니다.
`invalidFields`는 어떤 필드가 잘못됐는지만 보여 주고, 실제 사용자가 입력한 값은 제외합니다.

### H3. 식별자는 목적에 따라 마스킹하거나 해시한다

사용자 식별자가 필요할 때도 원문을 그대로 남길 필요는 많지 않습니다.
문제 재현을 위해 같은 사용자의 요청을 묶어야 한다면 salt를 둔 해시를 사용할 수 있고, 운영자가 사람이 읽어야 한다면 일부 마스킹을 적용할 수 있습니다.

```js
import { createHash } from 'node:crypto';

function hashForLog(value, salt) {
  return createHash('sha256')
    .update(`${salt}:${value}`)
    .digest('hex')
    .slice(0, 16);
}

function maskEmail(email) {
  const [name, domain] = email.split('@');
  if (!name || !domain) return '[invalid-email]';

  return `${name.slice(0, 2)}***@${domain}`;
}
```

해시는 복호화가 목적이 아닙니다.
로그 안에서 같은 주체의 이벤트를 묶는 최소한의 상관관계만 제공하는 용도로 제한하는 편이 좋습니다.
salt는 코드 저장소에 하드코딩하지 말고 런타임 설정으로 관리해야 합니다.

### H3. 에러 객체도 그대로 직렬화하지 않는다

에러 객체에는 메시지, 스택, cause, 외부 SDK가 붙인 메타데이터가 들어갈 수 있습니다.
특히 외부 API 오류는 요청 URL, 쿼리 문자열, 응답 일부를 에러 메시지에 포함하는 경우가 있습니다.

```js
function serializeErrorForLog(error) {
  return {
    name: error.name,
    code: error.code,
    message: sanitizeErrorMessage(error.message),
    stack: sanitizeStack(error.stack)
  };
}

function sanitizeErrorMessage(message = '') {
  return message
    .replace(/Bearer\s+[A-Za-z0-9._-]+/g, 'Bearer [redacted]')
    .replace(/token=[^&\s]+/g, 'token=[redacted]');
}

function sanitizeStack(stack = '') {
  return stack
    .split('\n')
    .slice(0, 5)
    .join('\n');
}
```

스택은 개발자에게 유용하지만 내부 경로를 포함할 수 있습니다.
외부로 공유될 가능성이 있는 로그나 리포트에는 스택 전체를 그대로 노출하지 않는 정책이 필요합니다.

## 성능 진단 로그와 샘플링

### H3. CPU와 리소스 지표는 임계값 기반으로 남긴다

성능 진단 로그는 매 요청마다 남기면 금방 부담이 됩니다.
대신 느린 작업, CPU 사용량이 높은 작업, 파일 시스템 I/O가 많은 작업만 자세히 남기는 방식이 좋습니다.

```js
function shouldKeepPerformanceLog({ elapsedMs, cpuMs, fsRead, fsWrite }) {
  if (elapsedMs >= 1000) return true;
  if (cpuMs >= 300) return true;
  if (fsRead + fsWrite >= 100) return true;

  return Math.random() < 0.01;
}
```

CPU 시간 측정은 [Node.js process.cpuUsage 가이드](/development/blog/seo/2026/06/17/nodejs-process-cpuusage-cpu-time-monitoring-guide.html)와 연결하면 좋습니다.
프로세스 전체 리소스 스냅샷은 [Node.js process.resourceUsage 가이드](/development/blog/seo/2026/06/17/nodejs-process-resourceusage-runtime-resource-diagnostics-guide.html)를 함께 보면 기준을 세우기 쉽습니다.

### H3. 성능 로그에는 입력 원문보다 작업 이름을 남긴다

성능 로그의 목적은 "무슨 데이터였는가"보다 "어떤 작업이 얼마나 비쌌는가"를 확인하는 것입니다.
그래서 작업 이름, route, jobType, tenant 등 제한된 분류값과 숫자 지표를 중심으로 남겨야 합니다.

```js
function buildPerformanceLog(input) {
  return {
    event: 'operation.performance',
    operation: input.operation,
    route: input.route,
    elapsedMs: Math.round(input.elapsedMs),
    cpuMs: Math.round(input.cpuMs),
    maxRssMb: Math.round(input.maxRssMb),
    sampled: input.sampled
  };
}
```

`operation`과 `route`는 정해진 값에서만 나오도록 관리하는 편이 좋습니다.
사용자 입력을 그대로 operation 이름에 붙이면 메트릭 cardinality가 폭발하고 민감정보가 섞일 수 있습니다.

### H3. 샘플링은 메트릭을 대체하지 않는다

로그 샘플링은 상세 사례를 줄이는 방법입니다.
전체 성공률, 에러율, p95 지연 시간, 큐 대기 시간, 메모리 사용량 같은 운영 지표는 별도의 메트릭으로 수집해야 합니다.

로그는 "왜 그랬는지"를 추적하는 데 강하고, 메트릭은 "얼마나 자주, 얼마나 크게 발생했는지"를 보는 데 강합니다.
둘을 섞으면 장애 대응 중 숫자를 잘못 해석하기 쉽습니다.

## 오류 로그 정책

### H3. 5xx와 처리 실패는 항상 남긴다

서버 오류, 큐 작업 실패, 배치 실패, 다운스트림 호출 실패는 샘플링하지 않는 편이 안전합니다.
대신 같은 오류가 폭주할 수 있으므로 알림은 rate limit을 두고, 로그는 구조화된 필드로 남깁니다.

```js
function logHandledError(logger, error, context) {
  logger.error({
    event: 'operation.failed',
    operation: context.operation,
    route: context.route,
    requestId: context.requestId,
    retryable: Boolean(context.retryable),
    err: serializeErrorForLog(error)
  });
}
```

여기서 `context`도 허용된 필드만 넣어야 합니다.
요청 본문, 인증 헤더, 외부 API 응답 원문은 오류 로그에도 넣지 않는 원칙을 유지합니다.

### H3. 중복 오류는 집계 키를 둔다

같은 원인의 오류가 초당 수천 번 발생하면 로그도 알림도 무력해집니다.
오류 메시지 전체 대신 오류 이름, 코드, 작업 이름을 묶어 집계 키를 만들면 중복을 줄일 수 있습니다.

```js
function buildErrorFingerprint(error, context) {
  return [
    context.operation,
    error.name,
    error.code || 'NO_CODE'
  ].join(':');
}
```

이 fingerprint는 원인 그룹을 나누기 위한 값입니다.
사용자 ID나 요청 URL 전체처럼 cardinality가 높은 값은 fingerprint에 넣지 않는 편이 좋습니다.

## 배포 전 체크리스트

### H3. 로그 정책은 코드와 문서에 같이 남긴다

로그 샘플링은 팀 전체가 공유해야 하는 운영 계약입니다.
코드에만 숨어 있으면 장애 대응 중 "왜 이 요청은 로그가 없지" 같은 혼란이 생깁니다.

배포 전에는 다음 항목을 확인합니다.

1. 정상 요청, 느린 요청, 실패 요청의 샘플링 기준이 분리되어 있는가?
2. 원문 body, 원문 header, 토큰, 쿠키를 저장하지 않는가?
3. 사용자 식별자는 마스킹 또는 해시 정책을 거치는가?
4. `sampled` 같은 필드로 샘플 로그임을 알 수 있는가?
5. 정확한 카운트가 필요한 값은 메트릭으로 따로 수집하는가?
6. 로그 보관 기간과 접근 권한이 서비스 위험도에 맞게 설정되어 있는가?

### H3. 테스트는 금지 필드와 필수 필드를 함께 검증한다

로그 유틸리티는 작은 함수처럼 보여도 보안과 운영 비용에 직접 영향을 줍니다.
그래서 단위 테스트로 필수 필드가 들어가는지, 금지 필드가 빠지는지 확인하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

test('request log keeps only allowed fields', () => {
  const log = buildRequestLog({
    requestId: 'req_123',
    method: 'POST',
    route: 'POST /login',
    statusCode: 401,
    elapsedMs: 45,
    userAgent: 'Example Browser',
    authorization: 'Bearer [example-token]',
    body: { password: '[example-password]' }
  });

  assert.equal(log.route, 'POST /login');
  assert.equal(log.authorization, undefined);
  assert.equal(log.body, undefined);
});
```

이런 테스트는 화려하지 않지만 회귀를 잘 막습니다.
나중에 누군가 디버깅 목적으로 필드를 추가할 때도 안전 기준을 자동으로 확인할 수 있습니다.

## 마무리

Node.js 로그 샘플링은 저장량을 줄이는 최적화가 아니라 운영 신호를 선명하게 만드는 설계입니다.
정상 요청은 낮은 비율로 남기고, 실패와 지연은 항상 남기며, 성능 진단 로그는 임계값 기반으로 제한하는 방식이 실무적으로 균형이 좋습니다.

다만 샘플링만으로는 민감정보 리스크를 줄일 수 없습니다.
허용 목록 기반 필드 선택, 원문 body와 header 제외, 식별자 마스킹 또는 해시, 에러 메시지 정제를 함께 적용해야 합니다.

다음 글을 함께 보면 Node.js 관측성 설계를 더 체계적으로 연결할 수 있습니다.

- [Node.js AsyncLocalStorage 요청 컨텍스트 로깅 가이드](/development/blog/seo/2026/03/19/nodejs-asynclocalstorage-request-context-logging-guide.html)
- [Node.js process.cpuUsage 가이드: CPU 시간으로 병목 구분하기](/development/blog/seo/2026/06/17/nodejs-process-cpuusage-cpu-time-monitoring-guide.html)
- [Node.js process.resourceUsage 가이드: 런타임 리소스 사용량 진단하기](/development/blog/seo/2026/06/17/nodejs-process-resourceusage-runtime-resource-diagnostics-guide.html)
- [Node.js OpenTelemetry 분산 트레이싱 실무 가이드](/development/blog/seo/2026/03/14/opentelemetry-nodejs-distributed-tracing-practical-guide.html)

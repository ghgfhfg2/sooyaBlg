---
layout: post
title: "Node.js SIGHUP 설정 리로드 가이드: 서버 재시작 없이 안전하게 설정을 반영하는 법"
date: 2026-06-01 08:00:00 +0900
lang: ko
translation_key: nodejs-sighup-config-reload-guide
permalink: /development/blog/seo/2026/06/01/nodejs-sighup-config-reload-guide.html
alternates:
  ko: /development/blog/seo/2026/06/01/nodejs-sighup-config-reload-guide.html
  x_default: /development/blog/seo/2026/06/01/nodejs-sighup-config-reload-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, sighup, config, operations, reliability, graceful-reload]
description: "Node.js에서 SIGHUP 신호로 설정 파일을 다시 읽고 서버 재시작 없이 안전하게 반영하는 방법을 정리했습니다. 검증, 원자적 교체, 실패 시 이전 설정 유지, 로그와 테스트 기준까지 실무 예제로 설명합니다."
---

운영 중인 Node.js 서버에서 설정을 바꿔야 할 때마다 프로세스를 재시작하면 간단해 보이지만, 모든 상황에서 좋은 선택은 아닙니다.
로그 레벨, 기능 플래그, 외부 API 타임아웃, 캐시 TTL처럼 런타임에 바꿔도 되는 값은 재시작 없이 반영할 수 있으면 운영 부담이 줄어듭니다.

이때 자주 쓰는 방식이 `SIGHUP` 신호를 받아 설정 파일을 다시 읽는 패턴입니다.
중요한 점은 신호를 받자마자 무작정 전역 객체를 바꾸는 것이 아니라, 새 설정을 검증한 뒤 한 번에 교체하고 실패하면 이전 설정을 유지하는 것입니다.

이 글에서는 Node.js에서 `SIGHUP`으로 설정을 리로드하는 기본 구조, 실패 시 안전장치, 요청 처리 코드와 연결하는 방법, 테스트와 운영 체크리스트를 정리합니다.
환경변수 로딩 기준이 먼저 필요하다면 [Node.js loadEnvFile 가이드](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)를 함께 보면 흐름을 잡기 쉽습니다.

## SIGHUP 설정 리로드가 맞는 상황

### 재시작 비용이 큰 값만 대상으로 삼는다

설정 리로드는 모든 값을 실시간으로 바꾸기 위한 만능 기능이 아닙니다.
DB 연결 문자열, 포트, 인증 키처럼 프로세스 초기화와 강하게 묶인 값은 재시작 배포가 더 명확할 수 있습니다.

반대로 아래 값들은 리로드 대상으로 검토하기 좋습니다.

- 로그 레벨
- 외부 API 호출 타임아웃
- 기능 플래그
- 샘플링 비율
- 캐시 TTL
- 서킷 브레이커 임계값

핵심 기준은 **이미 실행 중인 요청을 깨지 않고 다음 요청부터 반영할 수 있는가**입니다.
요청 중간에 정책이 바뀌면 디버깅이 어려워지므로, 보통은 요청 시작 시점에 현재 설정 스냅샷을 읽고 그 요청 안에서는 같은 값을 유지하는 편이 안전합니다.

### 배포와 운영 조작의 경계를 분리한다

코드 변경은 배포 파이프라인을 타야 합니다.
하지만 로그 레벨을 잠시 올리거나 외부 API 타임아웃을 보수적으로 늘리는 조작까지 매번 새 이미지를 빌드하게 만들면 대응이 느려집니다.

`SIGHUP` 리로드는 이런 운영 조작을 코드 배포와 분리합니다.
다만 권한 관리와 감사 로그가 빠지면 위험해질 수 있으므로, 누가 어떤 설정을 바꿨는지 남기는 운영 절차도 함께 있어야 합니다.
민감한 값은 파일 예제나 로그에 직접 남기지 말고 마스킹하는 규칙을 적용하세요.

## 기본 구현 구조

### 설정 파일을 읽고 검증한 뒤 반환한다

먼저 설정 로딩 함수를 순수하게 분리합니다.
파일을 읽고, JSON을 파싱하고, 필수 필드와 범위를 검증한 뒤 정상 설정 객체를 반환합니다.

```js
import { readFile } from 'node:fs/promises';

const DEFAULT_CONFIG_PATH = new URL('./config/runtime.json', import.meta.url);

function assertNumberInRange(name, value, min, max) {
  if (!Number.isFinite(value) || value < min || value > max) {
    throw new Error(`${name} must be a number between ${min} and ${max}`);
  }
}

function validateConfig(raw) {
  const config = {
    logLevel: raw.logLevel ?? 'info',
    upstreamTimeoutMs: raw.upstreamTimeoutMs ?? 2000,
    cacheTtlSeconds: raw.cacheTtlSeconds ?? 60
  };

  if (!['debug', 'info', 'warn', 'error'].includes(config.logLevel)) {
    throw new Error('logLevel must be one of debug, info, warn, error');
  }

  assertNumberInRange('upstreamTimeoutMs', config.upstreamTimeoutMs, 100, 30000);
  assertNumberInRange('cacheTtlSeconds', config.cacheTtlSeconds, 1, 3600);

  return Object.freeze(config);
}

export async function loadRuntimeConfig(path = DEFAULT_CONFIG_PATH) {
  const content = await readFile(path, 'utf8');
  return validateConfig(JSON.parse(content));
}
```

`Object.freeze()`는 깊은 불변성을 보장하지는 않지만, 단순 설정 객체에서는 실수로 값을 바꾸는 일을 줄여 줍니다.
중첩 객체가 많다면 검증 단계에서 새 객체를 만들어 반환하고, 필요한 범위까지 명시적으로 고정하는 편이 좋습니다.

### 현재 설정은 한 번에 교체한다

다음은 현재 설정을 보관하는 작은 모듈입니다.
새 설정이 정상적으로 로드된 뒤에만 참조를 교체합니다.
로드나 검증이 실패하면 이전 설정을 그대로 둡니다.

```js
import { loadRuntimeConfig } from './runtime-config.js';

let currentConfig = await loadRuntimeConfig();

export function getRuntimeConfig() {
  return currentConfig;
}

export async function reloadRuntimeConfig() {
  const nextConfig = await loadRuntimeConfig();
  currentConfig = nextConfig;
  return nextConfig;
}
```

이 구조에서는 요청 처리 코드가 `getRuntimeConfig()`를 호출할 때 항상 완성된 설정 객체만 받습니다.
부분적으로 갱신된 상태가 노출되지 않는다는 점이 중요합니다.

## SIGHUP 신호 처리하기

### process.on으로 리로드를 연결한다

Node.js에서는 `process.on('SIGHUP', handler)`로 신호를 받을 수 있습니다.
핸들러 안에서 비동기 리로드를 실행하되, 실패를 반드시 잡아 로그로 남겨야 합니다.

```js
import { reloadRuntimeConfig } from './config-store.js';

let reloadInProgress = false;

process.on('SIGHUP', () => {
  if (reloadInProgress) {
    console.warn('config reload skipped because another reload is running');
    return;
  }

  reloadInProgress = true;

  reloadRuntimeConfig()
    .then((config) => {
      console.info('config reloaded', {
        logLevel: config.logLevel,
        upstreamTimeoutMs: config.upstreamTimeoutMs,
        cacheTtlSeconds: config.cacheTtlSeconds
      });
    })
    .catch((error) => {
      console.error('config reload failed; keeping previous config', {
        message: error.message
      });
    })
    .finally(() => {
      reloadInProgress = false;
    });
});
```

여기서는 리로드가 겹치지 않도록 `reloadInProgress`를 두었습니다.
짧은 시간에 신호가 여러 번 들어오면 마지막 요청만 처리하는 방식도 가능하지만, 대부분의 운영 설정 리로드에서는 동시에 여러 번 처리하지 않는 단순한 정책이 더 읽기 쉽습니다.

### 종료 신호와 역할을 섞지 않는다

`SIGHUP`은 설정 리로드용으로 쓰고, `SIGTERM`이나 `SIGINT`는 종료 절차로 분리하는 편이 좋습니다.
종료와 리로드가 같은 함수 안에 섞이면 운영 중 어떤 신호가 어떤 동작을 만드는지 추적하기 어려워집니다.

종료 처리 구조는 [Node.js graceful shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)처럼 별도 경로로 정리해 두는 것이 안전합니다.
설정 리로드는 서버를 살려 둔 채 다음 요청부터 정책을 바꾸는 동작이고, graceful shutdown은 새 요청을 막고 기존 요청을 마무리하는 동작입니다.

## 요청 처리 코드에 연결하기

### 요청 시작 시점에 설정 스냅샷을 잡는다

HTTP 핸들러 안에서는 요청 시작 시점에 현재 설정을 한 번 읽고, 그 요청에서는 같은 객체를 사용합니다.
이렇게 하면 요청 처리 중간에 `SIGHUP`이 들어와도 하나의 요청 안에서 타임아웃이나 로깅 정책이 갑자기 바뀌지 않습니다.

```js
import http from 'node:http';
import { getRuntimeConfig } from './config-store.js';

const server = http.createServer(async (req, res) => {
  const config = getRuntimeConfig();

  try {
    const data = await fetchUpstream({
      timeoutMs: config.upstreamTimeoutMs
    });

    res.writeHead(200, { 'content-type': 'application/json' });
    res.end(JSON.stringify({ data }));
  } catch (error) {
    res.writeHead(502, { 'content-type': 'application/json' });
    res.end(JSON.stringify({ error: 'upstream_failed' }));
  }
});

server.listen(3000);
```

이 패턴은 단순하지만 효과가 큽니다.
설정은 전역으로 보관하되, 요청 단위에서는 명시적인 스냅샷처럼 다루기 때문입니다.

### 타임아웃과 취소 신호를 함께 다룬다

외부 API 타임아웃을 리로드 대상으로 삼는다면, 실제 호출 코드는 `AbortSignal.timeout()`이나 별도 `AbortController`와 연결해야 합니다.

```js
async function fetchUpstream({ timeoutMs }) {
  const response = await fetch('https://api.example.internal/status', {
    signal: AbortSignal.timeout(timeoutMs)
  });

  if (!response.ok) {
    throw new Error(`upstream responded with ${response.status}`);
  }

  return response.json();
}
```

타임아웃 자체의 설계는 [Node.js fetch AbortSignal timeout 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)와 이어집니다.
설정 리로드는 값을 바꾸는 통로이고, 실제 안정성은 호출 코드가 그 값을 일관되게 적용할 때 만들어집니다.

## 실패해도 이전 설정을 유지하기

### 잘못된 파일은 반영하지 않는다

운영에서 가장 중요한 규칙은 간단합니다.
새 설정이 깨졌다면 현재 정상 설정을 그대로 써야 합니다.

예를 들어 아래 파일은 `upstreamTimeoutMs` 범위를 벗어나므로 반영되면 안 됩니다.

```json
{
  "logLevel": "debug",
  "upstreamTimeoutMs": 0,
  "cacheTtlSeconds": 30
}
```

앞의 구현에서는 `validateConfig()`가 예외를 던지고, `reloadRuntimeConfig()`는 `currentConfig`를 교체하지 못한 채 실패합니다.
신호 핸들러는 실패 로그만 남기고 서버는 이전 설정으로 계속 동작합니다.

### 로그에는 값보다 결과를 남긴다

설정 리로드 로그는 너무 자세하면 민감정보 노출 위험이 생깁니다.
특히 API 키, 토큰, 사내 호스트명, 사용자 식별자 같은 값이 설정에 포함될 수 있다면 전체 설정 객체를 그대로 출력하지 않아야 합니다.

권장하는 로그는 아래 정도입니다.

- 리로드 성공 여부
- 변경 가능한 비민감 요약값
- 실패한 검증 항목의 이름
- 적용 시각
- 요청자 또는 실행 주체를 알 수 있는 운영 감사 로그

공개 문서나 블로그 예제를 만들 때도 실제 운영 값을 쓰지 않는 것이 좋습니다.
로그 예제 정리 기준은 [로그 예제 민감정보 정리 가이드](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)를 기준으로 삼을 수 있습니다.

## 테스트 기준

### 로딩 함수는 파일 단위로 테스트한다

설정 리로드의 핵심은 신호 처리보다 로딩과 검증입니다.
테스트에서는 정상 파일, 누락 필드, 범위 초과 값, JSON 파싱 실패를 각각 확인합니다.

```js
import assert from 'node:assert/strict';
import { mkdtemp, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import test from 'node:test';
import { loadRuntimeConfig } from '../src/runtime-config.js';

test('loads valid runtime config', async () => {
  const dir = await mkdtemp(join(tmpdir(), 'runtime-config-'));
  const path = join(dir, 'runtime.json');

  await writeFile(path, JSON.stringify({
    logLevel: 'debug',
    upstreamTimeoutMs: 1500,
    cacheTtlSeconds: 120
  }));

  const config = await loadRuntimeConfig(path);

  assert.equal(config.logLevel, 'debug');
  assert.equal(config.upstreamTimeoutMs, 1500);
  assert.equal(config.cacheTtlSeconds, 120);
});
```

내장 테스트 러너로 파일 입출력 테스트를 구성한다면 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)의 기본 구조와 함께 가져가면 충분합니다.

### 실패 시 기존 설정이 유지되는지 확인한다

리로드 스토어는 실패 시 이전 값을 보존해야 합니다.
이 동작은 운영 안정성과 직접 연결되므로 별도 테스트로 고정하는 편이 좋습니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';

test('keeps previous config when reload fails', async () => {
  let currentConfig = Object.freeze({ logLevel: 'info' });

  async function reloadWith(loader) {
    const nextConfig = await loader();
    currentConfig = nextConfig;
  }

  await assert.rejects(
    reloadWith(async () => {
      throw new Error('invalid config');
    }),
    /invalid config/
  );

  assert.equal(currentConfig.logLevel, 'info');
});
```

실제 코드에서는 모듈 상태를 직접 테스트하기보다, 설정 저장소를 작은 클래스나 팩토리 함수로 분리하면 더 검증하기 쉽습니다.
다만 과한 추상화보다 중요한 것은 “성공 전에는 교체하지 않는다”는 규칙을 테스트로 남기는 것입니다.

## 운영 체크리스트

### 리로드 가능한 값과 불가능한 값을 문서화한다

설정 리로드를 도입하면 팀 안에서 기대가 커질 수 있습니다.
그래서 어떤 값은 `SIGHUP`으로 반영되고, 어떤 값은 재배포가 필요한지 명확히 적어 두어야 합니다.

- 런타임 리로드 가능: 로그 레벨, 타임아웃, 캐시 TTL, 기능 플래그
- 재시작 필요: 포트, 프로세스 권한, 필수 인증 정보, DB 연결 구조
- 배포 필요: 코드 경로, 데이터 스키마 변경, 의존성 변경

이 경계를 문서화하지 않으면 운영자는 리로드가 된 줄 알았는데 실제로는 초기화 시점에만 읽히는 값이 남아 혼란이 생길 수 있습니다.

### 적용 전후 관측 지표를 확인한다

설정을 바꾸는 목적은 보통 운영 상태를 개선하는 것입니다.
리로드 성공 로그만 보고 끝내지 말고, 바꾼 값이 의도한 지표에 영향을 줬는지 확인해야 합니다.

- 타임아웃을 늘렸다면 upstream timeout 비율이 줄었는가?
- 로그 레벨을 올렸다면 필요한 진단 로그가 보이는가?
- 캐시 TTL을 바꿨다면 hit ratio와 응답 지연이 예상대로 움직이는가?
- 기능 플래그를 바꿨다면 에러율과 지연 시간이 안정적인가?

운영 중 정책 변경이 잦은 서비스라면 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)처럼 계측 이벤트를 낮은 결합도로 남기는 구조도 도움이 됩니다.

## 함께 보면 좋은 글

- [Node.js loadEnvFile 가이드: dotenv 없이 환경변수 관리하기](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)
- [Node.js graceful shutdown 가이드: 진행 중 요청을 안전하게 마무리하는 법](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)
- [Node.js fetch AbortSignal timeout 가이드: 외부 호출 타임아웃과 재시도 다루기](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html)

`SIGHUP` 설정 리로드는 재시작을 줄이는 편의 기능이 아니라, 운영 중 바꿔도 되는 정책을 안전하게 반영하는 작은 계약입니다.
새 설정을 먼저 검증하고, 성공했을 때만 한 번에 교체하고, 실패하면 이전 설정을 유지하세요.
그 세 가지 규칙만 지켜도 설정 리로드는 훨씬 예측 가능한 운영 도구가 됩니다.

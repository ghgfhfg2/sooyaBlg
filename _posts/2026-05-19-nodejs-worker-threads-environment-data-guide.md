---
layout: post
title: "Node.js worker_threads 환경 데이터 가이드: 워커에 설정값 안전하게 전달하는 법"
date: 2026-05-19 20:00:00 +0900
lang: ko
translation_key: nodejs-worker-threads-environment-data-guide
permalink: /development/blog/seo/2026/05/19/nodejs-worker-threads-environment-data-guide.html
alternates:
  ko: /development/blog/seo/2026/05/19/nodejs-worker-threads-environment-data-guide.html
  x_default: /development/blog/seo/2026/05/19/nodejs-worker-threads-environment-data-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, worker_threads, worker, environment-data, backend, javascript]
description: "Node.js worker_threads의 setEnvironmentData와 getEnvironmentData로 워커 스레드에 공통 설정값을 안전하게 전달하는 방법을 정리했습니다. workerData와의 차이, 구조화 복제 주의점, 설정 스냅샷, 에러 처리와 실무 적용 기준까지 예제로 설명합니다."
---

Node.js에서 CPU 사용량이 큰 작업을 분리하려면 `worker_threads`를 자주 검토하게 됩니다.
이미지 리사이징, 압축, 대량 JSON 변환, 암호학적 해시 계산, 리포트 생성처럼 이벤트 루프를 오래 붙잡는 일을 워커로 넘기면 메인 서버의 응답성을 지키기 쉽습니다.
그런데 워커가 늘어나면 또 다른 문제가 생깁니다.
모든 워커에 공통으로 필요한 설정값, 예를 들어 기능 플래그, 처리 한도, 로깅 레벨, 경로 정책을 어떻게 전달할지 정해야 합니다.

Node.js의 `worker_threads` 모듈에는 `setEnvironmentData()`와 `getEnvironmentData()`가 있습니다.
메인 스레드에서 환경 데이터를 등록하면 이후 생성되는 워커가 같은 키로 값을 읽을 수 있습니다.
이 글에서는 Node.js worker_threads 환경 데이터 기본 사용법, `workerData`와의 차이, 구조화 복제 특성, 설정 변경 시점, 실무에서 안전하게 쓰는 기준을 정리합니다.
워커에서 넘기는 값이 객체 복사와 관련 있다는 점이 헷갈린다면 [Node.js structuredClone 가이드: JSON 없이 안전하게 깊은 복사하는 법](/development/blog/seo/2026/05/15/nodejs-structuredclone-deep-copy-guide.html)도 함께 보면 좋습니다.

## worker_threads 환경 데이터가 필요한 이유

### H3. 워커마다 반복되는 설정 전달 코드를 줄인다

`Worker`를 만들 때 `workerData` 옵션으로 값을 넘길 수 있습니다.
작업 하나에만 필요한 입력값이라면 이 방식이 가장 명확합니다.
하지만 모든 워커가 공통으로 읽어야 하는 설정을 매번 `workerData`에 섞어 넣으면 코드가 금방 흐려집니다.

```js
import { Worker } from 'node:worker_threads';

const worker = new Worker(new URL('./image-worker.js', import.meta.url), {
  workerData: {
    jobId: 'job-101',
    inputPath: './uploads/photo.png',
    maxPixels: 12_000_000,
    logLevel: 'info',
    featureFlags: { webpOutput: true },
  },
});
```

위 코드에서 `jobId`와 `inputPath`는 개별 작업 입력입니다.
반면 `maxPixels`, `logLevel`, `featureFlags`는 모든 워커가 공유할 수 있는 실행 정책에 가깝습니다.
이 둘을 계속 같은 객체에 넣으면 작업 입력과 런타임 정책의 경계가 흐려집니다.
환경 데이터는 이런 공통 설정을 별도 채널로 분리할 때 유용합니다.

### H3. 워커 파일 안에서 공통 정책을 직접 읽게 만든다

`setEnvironmentData()`로 값을 등록하면 이후 만들어지는 워커에서 `getEnvironmentData()`로 값을 읽을 수 있습니다.
워커 생성 코드가 복잡한 설정 객체를 매번 조립하지 않아도 되고, 워커 파일은 자신에게 필요한 정책만 명시적으로 가져올 수 있습니다.

```js
// main.js
import {
  Worker,
  setEnvironmentData,
} from 'node:worker_threads';

setEnvironmentData('app:limits', {
  maxPixels: 12_000_000,
  maxOutputBytes: 8 * 1024 * 1024,
});

setEnvironmentData('app:logLevel', 'info');

new Worker(new URL('./image-worker.js', import.meta.url), {
  workerData: {
    jobId: 'job-101',
    inputPath: './uploads/photo.png',
  },
});
```

```js
// image-worker.js
import {
  getEnvironmentData,
  workerData,
} from 'node:worker_threads';

const limits = getEnvironmentData('app:limits');
const logLevel = getEnvironmentData('app:logLevel');

console.log('job:', workerData.jobId);
console.log('max pixels:', limits.maxPixels);
console.log('log level:', logLevel);
```

이렇게 나누면 `workerData`는 작업별 입력, 환경 데이터는 워커 공통 설정이라는 역할이 선명해집니다.
운영 중 워커가 많아질수록 이 구분은 디버깅과 테스트에서 차이를 만듭니다.

## 기본 사용법

### H3. setEnvironmentData는 워커 생성 전에 호출한다

환경 데이터는 앞으로 생성될 워커가 읽을 기본값을 정하는 방식으로 이해하는 편이 안전합니다.
이미 실행 중인 워커의 값을 실시간으로 바꾸는 통로가 아닙니다.
따라서 서버 부팅 단계나 워커 풀 초기화 직전에 등록하는 패턴이 좋습니다.

```js
// bootstrap-workers.js
import {
  Worker,
  setEnvironmentData,
} from 'node:worker_threads';

export function createReportWorkerPool(size) {
  setEnvironmentData('report:config', {
    templateDir: './templates',
    timeoutMs: 3000,
    currency: 'KRW',
  });

  return Array.from({ length: size }, () => {
    return new Worker(new URL('./report-worker.js', import.meta.url));
  });
}
```

이 예시에서는 워커 풀을 만들기 전에 리포트 생성 정책을 등록합니다.
워커 파일은 이 설정을 읽고 템플릿 경로나 시간 제한 정책을 적용할 수 있습니다.
설정값을 여러 모듈에서 흩어져 등록하면 추적이 어려우므로, 가능한 한 부트스트랩 함수 한곳에서 관리하는 것이 좋습니다.

### H3. getEnvironmentData는 워커와 메인 스레드 모두에서 읽을 수 있다

`getEnvironmentData()`는 워커 안에서만 쓸 수 있는 함수가 아닙니다.
메인 스레드에서도 같은 키로 값을 읽을 수 있습니다.
그래서 테스트나 초기화 검증 코드에서 설정이 제대로 등록됐는지 확인하기 쉽습니다.

```js
import {
  getEnvironmentData,
  setEnvironmentData,
} from 'node:worker_threads';

setEnvironmentData('app:mode', 'production');

const mode = getEnvironmentData('app:mode');

if (mode !== 'production') {
  throw new Error('worker environment mode is invalid');
}
```

단, 이 함수가 전역 설정 저장소처럼 남용되어서는 안 됩니다.
애플리케이션 전체 설정은 일반 설정 모듈에서 관리하고, 워커가 생성될 때 함께 복제되어야 하는 값만 환경 데이터로 넘기는 편이 명확합니다.
메인 스레드의 요청 단위 상태를 보존하고 싶다면 환경 데이터보다 [Node.js AsyncLocalStorage bind 가이드: 콜백에서도 요청 컨텍스트 유지하기](/development/blog/seo/2026/05/09/nodejs-asynclocalstorage-bind-callback-context-guide.html) 같은 요청 컨텍스트 도구가 더 적합합니다.

## workerData와 환경 데이터의 차이

### H3. workerData는 작업 입력, 환경 데이터는 공통 설정에 둔다

두 기능은 모두 워커에 값을 전달하지만 의도가 다릅니다.
`workerData`는 워커 인스턴스 하나를 만들 때 넘기는 개별 데이터입니다.
반면 환경 데이터는 같은 프로세스에서 만들어지는 워커들이 공통으로 읽을 수 있는 키-값 설정에 가깝습니다.

```js
import {
  Worker,
  setEnvironmentData,
} from 'node:worker_threads';

setEnvironmentData('thumbnail:policy', {
  maxWidth: 1280,
  format: 'webp',
  quality: 82,
});

function runThumbnailJob(filePath) {
  return new Worker(new URL('./thumbnail-worker.js', import.meta.url), {
    workerData: {
      filePath,
      requestedAt: Date.now(),
    },
  });
}

runThumbnailJob('./uploads/a.png');
runThumbnailJob('./uploads/b.png');
```

위 코드에서 `filePath`는 작업마다 달라집니다.
반면 썸네일 폭, 출력 포맷, 품질은 여러 작업에 같은 정책으로 적용됩니다.
이처럼 변경 주기와 책임이 다른 값을 분리하면 워커 코드의 입력 검증도 단순해집니다.

### H3. 실시간 업데이트가 필요하면 메시지 채널을 쓴다

환경 데이터는 워커 생성 시점의 설정 스냅샷으로 다루는 것이 좋습니다.
이미 떠 있는 워커에 새로운 정책을 즉시 반영해야 한다면 `worker.postMessage()` 같은 메시지 전달을 사용해야 합니다.

```js
// main.js
import { Worker } from 'node:worker_threads';

const worker = new Worker(new URL('./policy-worker.js', import.meta.url));

worker.postMessage({
  type: 'policy:update',
  payload: {
    maxItems: 500,
    logLevel: 'debug',
  },
});
```

```js
// policy-worker.js
import { parentPort } from 'node:worker_threads';

let policy = {
  maxItems: 100,
  logLevel: 'info',
};

parentPort.on('message', (message) => {
  if (message.type !== 'policy:update') {
    return;
  }

  policy = {
    ...policy,
    ...message.payload,
  };
});
```

실시간 정책 변경은 강력하지만 동시성 문제가 생길 수 있습니다.
작업 중간에 정책이 바뀌어도 되는지, 새 작업부터만 반영할지, 이전 작업은 어떤 기준으로 기록할지 먼저 정해야 합니다.
정책 변경이 드물다면 워커 풀을 재시작해 새 환경 데이터로 다시 띄우는 방식이 더 단순할 수 있습니다.

## 구조화 복제와 값 설계

### H3. 환경 데이터는 복제 가능한 값으로 제한한다

환경 데이터에 넣는 값은 워커로 전달될 수 있어야 합니다.
일반 객체, 배열, 문자열, 숫자, 불리언처럼 구조화 복제 가능한 값은 적합합니다.
반대로 함수, 클래스 인스턴스에 강하게 의존하는 객체, 열린 파일 핸들, 데이터베이스 연결 객체 같은 값은 설정 데이터로 넘기면 안 됩니다.

```js
import { setEnvironmentData } from 'node:worker_threads';

setEnvironmentData('safe:config', {
  retry: 2,
  batchSize: 100,
  allowedExtensions: ['.jpg', '.png', '.webp'],
});
```

좋은 환경 데이터는 직렬화된 설정처럼 읽혀야 합니다.
워커가 특정 서비스 객체나 열린 리소스를 공유해야 한다면 환경 데이터가 아니라 별도의 초기화 코드, 메시지 프로토콜, 외부 저장소 연결 전략을 사용해야 합니다.
성능 측정이 필요한 작업이라면 [Node.js performance.now 가이드: 실행 시간 정확하게 측정하는 법](/development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html)을 참고해 워커 내부 처리 시간과 메인 스레드 대기 시간을 나눠 보는 것도 좋습니다.

### H3. 큰 객체를 무심코 복제하지 않는다

환경 데이터가 편하다고 해서 대형 룩업 테이블이나 수십 MB 설정을 넣으면 워커 생성 비용이 커질 수 있습니다.
워커는 별도 스레드에서 실행되지만, 값을 넘기는 과정의 복제 비용은 사라지지 않습니다.
특히 워커를 자주 만들고 버리는 구조에서는 큰 환경 데이터가 시작 지연으로 이어질 수 있습니다.

```js
import { setEnvironmentData } from 'node:worker_threads';

setEnvironmentData('parser:rules', {
  version: '2026-05-19',
  rulesPath: './config/parser-rules.json',
  checksum: 'sha256:example-checksum',
});
```

큰 데이터 자체를 넣기보다 경로, 버전, 체크섬처럼 워커가 필요한 리소스를 찾을 수 있는 작은 메타데이터만 넣는 방식이 더 낫습니다.
워커가 시작될 때 파일을 읽어 캐시하면 메모리 사용량과 시작 시간을 더 예측하기 쉽습니다.
다만 파일 읽기에도 취소와 시간 제한이 필요하다면 [Node.js fs.readFile AbortSignal 가이드: 파일 읽기를 안전하게 취소하는 법](/development/blog/seo/2026/05/19/nodejs-fs-readfile-abortsignal-cancel-file-read-guide.html)을 함께 적용하세요.

## 실무 패턴

### H3. 키 이름에 네임스페이스를 붙인다

환경 데이터 키는 문자열이나 심볼을 사용할 수 있습니다.
여러 모듈이 같은 프로세스에서 값을 등록한다면 충돌을 피해야 합니다.
가장 단순한 방법은 `app:feature`, `report:config`, `thumbnail:policy`처럼 도메인 접두사를 붙이는 것입니다.

```js
import { setEnvironmentData } from 'node:worker_threads';

setEnvironmentData('search:indexer:limits', {
  maxDocuments: 10_000,
  chunkSize: 250,
});

setEnvironmentData('search:indexer:logging', {
  level: 'warn',
  sampleRate: 0.1,
});
```

키를 상수로 분리해 두면 오타도 줄일 수 있습니다.
워커와 메인 스레드가 같은 상수 모듈을 가져오게 하면 문자열 불일치로 `undefined`가 나오는 문제를 막기 쉽습니다.

```js
// worker-env-keys.js
export const WORKER_ENV_KEYS = Object.freeze({
  indexerLimits: 'search:indexer:limits',
  indexerLogging: 'search:indexer:logging',
});
```

### H3. 워커 시작 시 필수 설정을 검증한다

환경 데이터는 키가 없으면 `undefined`를 반환할 수 있습니다.
워커가 필수 설정 없이 조용히 실행되면 나중에 더 이상한 오류로 이어집니다.
워커 시작 단계에서 필요한 설정을 확인하고 빠르게 실패시키는 편이 안전합니다.

```js
import { getEnvironmentData } from 'node:worker_threads';

function readRequiredEnvironmentData(key) {
  const value = getEnvironmentData(key);

  if (value === undefined) {
    throw new Error(`missing worker environment data: ${key}`);
  }

  return value;
}

const limits = readRequiredEnvironmentData('search:indexer:limits');

if (!Number.isInteger(limits.chunkSize) || limits.chunkSize <= 0) {
  throw new Error('invalid indexer chunk size');
}
```

검증은 복잡할 필요가 없습니다.
필수 키 존재 여부, 숫자 범위, 배열 여부, 허용된 문자열 정도만 확인해도 운영 중 실수를 빠르게 잡을 수 있습니다.
타입스크립트를 사용한다면 런타임 검증과 타입 선언을 함께 두면 더 안정적입니다.

### H3. 민감정보는 환경 데이터로 넘기지 않는다

환경 데이터는 워커 코드에서 쉽게 읽을 수 있는 설정 채널입니다.
API 키, 접근 토큰, 개인 식별 정보, 비밀번호 같은 민감정보를 넣는 저장소로 쓰면 안 됩니다.
워커가 꼭 외부 서비스에 접근해야 한다면 최소 권한 토큰을 별도 비밀 관리 체계에서 읽게 하거나, 메인 스레드가 필요한 작업만 대리 수행하는 구조를 고려하세요.

```js
import { setEnvironmentData } from 'node:worker_threads';

setEnvironmentData('safe:public-policy', {
  region: 'ap-northeast-2',
  requestTimeoutMs: 1500,
});
```

환경 데이터에 들어가도 되는 값은 로그에 남아도 큰 문제가 없는 운영 정책 수준이어야 합니다.
민감정보를 피하면 디버깅 로그나 프로세스 리포트에서도 노출 위험을 줄일 수 있습니다.
장애 분석용 리포트를 남길 때는 [Node.js process.report 가이드: 운영 장애 진단 리포트 남기는 법](/development/blog/seo/2026/05/14/nodejs-process-report-production-diagnostics-guide.html)의 민감정보 점검 기준도 같이 적용하면 좋습니다.

## 테스트와 디버깅 기준

### H3. 워커 생성 전후를 나눠 테스트한다

환경 데이터 관련 버그는 대부분 “등록 시점”에서 나옵니다.
워커를 만들기 전에 값이 등록됐는지, 워커 안에서 같은 키로 읽는지, 필수 설정이 없을 때 빠르게 실패하는지 테스트하면 충분한 경우가 많습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import {
  getEnvironmentData,
  setEnvironmentData,
} from 'node:worker_threads';

test('worker environment data can be registered', () => {
  setEnvironmentData('test:limits', {
    batchSize: 50,
  });

  assert.deepEqual(getEnvironmentData('test:limits'), {
    batchSize: 50,
  });
});
```

실제 워커까지 띄우는 통합 테스트는 비용이 더 크지만, 설정 누락을 잡는 데 효과적입니다.
Node.js 기본 테스트 러너를 사용한다면 [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하기](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 참고해 작은 단위부터 자동화할 수 있습니다.

### H3. 워커 로그에는 설정 버전만 남긴다

디버깅을 위해 워커 시작 시 설정 전체를 로그로 남기고 싶을 때가 있습니다.
하지만 설정 객체가 커지거나 민감한 값이 섞이면 로그가 위험해집니다.
대신 설정 버전, 정책 이름, 체크섬처럼 안전한 식별자만 남기는 편이 좋습니다.

```js
import { getEnvironmentData } from 'node:worker_threads';

const policy = getEnvironmentData('parser:policy');

console.info('parser worker started', {
  policyVersion: policy?.version,
  rulesChecksum: policy?.checksum,
});
```

로그는 장애 대응의 출발점이지만, 모든 값을 담을 필요는 없습니다.
워커가 어떤 정책 세트로 시작했는지 추적할 수 있을 정도면 충분합니다.
필요하면 정책 파일 자체는 배포 산출물이나 설정 저장소에서 별도로 조회하도록 설계하세요.

## 언제 쓰지 않는 게 나을까

### H3. 요청별 데이터 전달에는 적합하지 않다

사용자 ID, 요청 ID, 업로드 파일 경로, 즉석 계산 입력처럼 요청마다 달라지는 값은 환경 데이터에 넣지 마세요.
이런 값은 `workerData`, `postMessage()`, 작업 큐 메시지처럼 작업 단위 채널로 전달해야 합니다.
환경 데이터에 요청별 값을 넣기 시작하면 워커 생성 순서와 값 갱신 시점에 의존하는 버그가 생깁니다.

### H3. 프로세스 전체 설정 저장소로 남용하지 않는다

환경 데이터는 “워커가 시작될 때 필요한 공통 설정”에 초점을 맞춘 도구입니다.
일반 애플리케이션 설정, 사용자 세션 상태, 런타임 캐시, 서비스 객체 보관소를 대체하지 않습니다.
값의 수명이 워커 생성 시점과 맞지 않는다면 다른 도구를 쓰는 편이 더 명확합니다.

## FAQ

### H3. setEnvironmentData를 나중에 다시 호출하면 기존 워커 값도 바뀌나요?

이미 실행 중인 워커가 읽은 값이 실시간으로 바뀐다고 기대하면 안 됩니다.
환경 데이터는 이후 생성되는 워커에 전달할 공통 설정으로 다루는 것이 안전합니다.
기존 워커에 정책을 반영해야 한다면 메시지로 업데이트하거나 워커를 재시작하세요.

### H3. workerData 대신 항상 환경 데이터를 쓰면 되나요?

아닙니다.
작업마다 다른 입력은 `workerData`나 메시지로 넘기고, 여러 워커가 공유하는 설정만 환경 데이터로 분리하는 편이 좋습니다.
두 기능은 대체 관계라기보다 책임이 다른 전달 경로입니다.

### H3. 환경 데이터에 비밀 키를 넣어도 되나요?

권장하지 않습니다.
환경 데이터는 워커 코드에서 쉽게 읽히는 설정값이며, 디버깅 과정에서 노출될 가능성도 있습니다.
비밀 키는 별도 비밀 관리 체계와 최소 권한 원칙에 맞춰 다루는 편이 안전합니다.

### H3. 큰 설정 객체를 넣어도 성능 문제가 없나요?

작은 객체라면 보통 문제가 되지 않습니다.
하지만 큰 룩업 테이블이나 대용량 설정을 넣으면 워커 생성 비용과 메모리 사용량이 커질 수 있습니다.
가능하면 경로, 버전, 체크섬 같은 작은 메타데이터만 환경 데이터로 넘기고 워커가 필요한 리소스를 직접 로드하게 하세요.

## 마무리

`setEnvironmentData()`와 `getEnvironmentData()`는 Node.js 워커에 공통 설정을 전달하는 작지만 실용적인 도구입니다.
핵심은 역할을 분리하는 것입니다.
작업마다 달라지는 입력은 `workerData`나 메시지로 넘기고, 워커 생성 시점에 함께 복제되어야 하는 공통 정책만 환경 데이터로 둡니다.

실무에서는 세 가지 원칙만 지켜도 충분히 안정적으로 사용할 수 있습니다.
첫째, 워커 생성 전에 한곳에서 등록합니다.
둘째, 구조화 복제 가능한 작고 안전한 값만 넣습니다.
셋째, 워커 시작 시 필수 설정을 검증하고 민감정보는 절대 포함하지 않습니다.
이 기준을 지키면 워커 코드의 설정 전달 방식이 단순해지고, 성능 작업과 운영 디버깅도 더 예측 가능해집니다.

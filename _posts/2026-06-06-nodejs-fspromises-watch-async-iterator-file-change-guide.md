---
layout: post
title: "Node.js fsPromises.watch 가이드: 비동기 이터레이터로 파일 변경을 감지하는 법"
date: 2026-06-06 20:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-watch-async-iterator-file-change-guide
permalink: /development/blog/seo/2026/06/06/nodejs-fspromises-watch-async-iterator-file-change-guide.html
alternates:
  ko: /development/blog/seo/2026/06/06/nodejs-fspromises-watch-async-iterator-file-change-guide.html
  x_default: /development/blog/seo/2026/06/06/nodejs-fspromises-watch-async-iterator-file-change-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, fs, fspromises, watch, asynciterator, abortsignal, file]
description: "Node.js fsPromises.watch를 비동기 이터레이터와 AbortSignal로 안전하게 사용하는 방법을 정리했습니다. 파일 변경 감지, 이벤트 병합, 설정 리로드, 플랫폼 차이와 테스트 기준을 실무 예제로 설명합니다."
---

개발 도구나 작은 운영 자동화를 만들다 보면 파일 변경을 감지해야 할 때가 있습니다.
설정 파일이 바뀌면 다시 읽고, 특정 디렉터리에 파일이 들어오면 처리하며, 소스 코드가 수정되면 검사 작업을 다시 실행하는 식입니다.

Node.js에서는 오래전부터 `fs.watch()`를 사용할 수 있었습니다.
하지만 콜백 안에서 비동기 작업을 시작하면 중복 실행, 에러 전파, 종료 처리 흐름이 쉽게 흩어집니다.
`node:fs/promises`의 `watch()`는 파일 시스템 이벤트를 **비동기 이터레이터**로 제공해 `for await...of` 문 안에서 순서대로 다룰 수 있게 해 줍니다.

이 글에서는 Node.js `fsPromises.watch()`의 기본 사용법, `AbortSignal`을 이용한 종료, 짧은 시간에 몰리는 이벤트 처리, 설정 파일 리로드 패턴, 플랫폼 차이와 테스트 체크리스트를 정리합니다.
비동기 이터러블을 파일 목록 수집에 활용하는 방법이 궁금하다면 [Node.js fsPromises.glob 가이드](/development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html)도 함께 참고하세요.

## fsPromises.watch가 필요한 상황

### H3. 콜백 대신 for await...of로 이벤트를 읽는다

기존 `fs.watch()`는 변경이 발생할 때 등록한 콜백을 호출합니다.

```js
import { watch } from 'node:fs';

const watcher = watch('./config', (eventType, filename) => {
  console.log({ eventType, filename });
});
```

간단한 로그 출력에는 충분합니다.
하지만 콜백 안에서 설정 파일을 읽거나 외부 작업을 호출하면, 앞선 작업이 끝나기 전에 다음 이벤트가 들어올 수 있습니다.
종료할 때 watcher와 실행 중인 작업을 각각 관리해야 하는 문제도 생깁니다.

`fsPromises.watch()`는 같은 종류의 이벤트를 비동기 이터레이터로 제공합니다.

```js
import { watch } from 'node:fs/promises';

for await (const event of watch('./config')) {
  console.log(event.eventType, event.filename);
}
```

이 구조에서는 이벤트를 읽는 흐름과 비동기 작업을 한 함수 안에서 표현할 수 있습니다.
`await`를 사용하면 현재 처리가 끝난 뒤 다음 반복으로 넘어가므로, 작업 순서를 의도적으로 직렬화하기도 쉽습니다.

### H3. 파일 감지는 변경 사실을 알려 주는 신호로 본다

파일 시스템 감시 이벤트를 데이터베이스 변경 로그처럼 정확한 기록으로 보면 안 됩니다.
운영체제와 파일 시스템에 따라 한 번의 저장이 여러 이벤트를 만들 수 있고, 이벤트가 합쳐지거나 파일 이름이 제공되지 않을 수도 있습니다.

따라서 아래처럼 생각하는 편이 안전합니다.

- 이벤트는 "대상 상태를 다시 확인할 시점"을 알려 주는 신호다.
- 이벤트 한 건과 사용자 저장 동작 한 번이 정확히 대응한다고 가정하지 않는다.
- 이벤트 payload만 믿지 말고 필요하면 파일을 다시 읽거나 목록을 다시 조회한다.
- 짧은 시간에 들어오는 중복 이벤트를 합칠 정책을 둔다.

이 원칙을 먼저 세우면 편집기별 저장 방식이나 배포 환경 차이에도 덜 흔들리는 watcher를 만들 수 있습니다.

## 기본 사용법

### H3. node:fs/promises에서 watch를 가져온다

가장 작은 예제는 디렉터리를 감시하고 이벤트 종류와 파일 이름을 출력하는 코드입니다.

```js
import { watch } from 'node:fs/promises';

export async function logChanges(directory) {
  for await (const { eventType, filename } of watch(directory)) {
    console.log({
      eventType,
      filename: filename ?? '(unknown)'
    });
  }
}

await logChanges('./config');
```

일반적으로 `eventType`은 `rename` 또는 `change`입니다.
`rename`은 이름 변경만 뜻한다고 단정하면 안 됩니다.
파일 생성이나 삭제처럼 디렉터리 엔트리가 달라질 때도 관찰될 수 있으므로, 실제 상태를 다시 확인해야 합니다.

`filename`도 항상 존재한다고 가정하지 않는 편이 좋습니다.
파일 이름이 없으면 전체 대상 목록을 다시 조회하거나, 감시 대상이 단일 파일이라면 그 파일 자체를 다시 확인하는 fallback을 둘 수 있습니다.

### H3. encoding 옵션을 명시한다

파일 이름을 문자열로 다루려면 `encoding`을 명시해 코드 의도를 남길 수 있습니다.

```js
import { watch } from 'node:fs/promises';

for await (const event of watch('./incoming', {
  encoding: 'utf8'
})) {
  if (!event.filename) {
    continue;
  }

  console.log(`${event.eventType}: ${event.filename}`);
}
```

파일 이름은 신뢰할 수 있는 명령이나 경로라고 가정하지 않아야 합니다.
외부에서 파일을 만들 수 있는 디렉터리라면 이름을 로그에 남길 때 제어 문자를 제거하고, 셸 명령 문자열에 직접 이어 붙이지 않아야 합니다.
CLI 출력 정리가 필요하다면 [Node.js util.stripVTControlCharacters 가이드](/development/blog/seo/2026/05/31/nodejs-util-strip-vt-control-characters-cli-output-guide.html)의 원칙도 적용할 수 있습니다.

### H3. 특정 파일만 처리하더라도 디렉터리 상태를 다시 확인한다

설정 파일 하나를 감시할 때도 저장 과정에서 임시 파일 생성과 이름 교체가 일어날 수 있습니다.
그래서 이벤트 종류만 보고 처리 여부를 결정하기보다, 관심 파일이 현재 존재하는지 확인한 뒤 다시 읽는 편이 견고합니다.

```js
import { readFile, watch } from 'node:fs/promises';

async function readJson(path) {
  const text = await readFile(path, 'utf8');
  return JSON.parse(text);
}

export async function watchConfig(directory, filename) {
  const path = `${directory}/${filename}`;

  for await (const event of watch(directory)) {
    if (event.filename && event.filename !== filename) {
      continue;
    }

    try {
      const config = await readJson(path);
      console.log('config reloaded', {
        keys: Object.keys(config).length
      });
    } catch (error) {
      console.error('config reload failed', {
        code: error.code,
        message: error.message
      });
    }
  }
}
```

예제에서는 이해를 위해 경로를 단순하게 연결했습니다.
실제 코드에서는 `node:path`의 `join()`을 사용하고, 허용된 디렉터리 밖으로 경로가 벗어나지 않는지 확인하는 편이 좋습니다.

## AbortSignal로 watcher 종료하기

### H3. 무한 반복에는 명시적인 종료 조건이 필요하다

`for await...of` 감시 루프는 변경 이벤트를 계속 기다립니다.
서비스 종료, 테스트 정리, 작업 취소 시점에 루프를 끝낼 방법이 필요합니다.
`fsPromises.watch()`의 `signal` 옵션에 `AbortSignal`을 넘기면 외부에서 감시를 중단할 수 있습니다.

```js
import { watch } from 'node:fs/promises';

export async function runWatcher(directory, signal) {
  try {
    for await (const event of watch(directory, { signal })) {
      console.log(event);
    }
  } catch (error) {
    if (error.name === 'AbortError') {
      return;
    }

    throw error;
  }
}

const controller = new AbortController();

setTimeout(() => controller.abort(), 30_000);

await runWatcher('./config', controller.signal);
```

취소는 예상 가능한 종료 경로이므로 `AbortError`를 일반 장애와 구분합니다.
반대로 권한 문제, 존재하지 않는 경로, 운영체제 watcher 한도 초과 같은 에러까지 숨기면 실제 장애를 놓칠 수 있습니다.

### H3. 여러 종료 조건은 AbortSignal.any로 묶는다

감시 작업은 사용자 취소, 프로세스 종료, 최대 실행 시간처럼 여러 이유로 끝날 수 있습니다.
이때 여러 신호를 하나로 묶으면 watcher가 종료 조건을 하나만 받도록 만들 수 있습니다.

```js
import { watch } from 'node:fs/promises';

export async function watchForOneMinute(directory, parentSignal) {
  const signal = AbortSignal.any([
    parentSignal,
    AbortSignal.timeout(60_000)
  ]);

  try {
    for await (const event of watch(directory, { signal })) {
      await handleChange(event);
    }
  } catch (error) {
    if (error.name !== 'AbortError') {
      throw error;
    }
  }
}
```

취소 신호 조합 기준은 [Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)에서 더 자세히 다룹니다.
중요한 점은 watcher만 멈추는 것이 아니라, `handleChange()`가 긴 작업이라면 같은 신호를 하위 작업에도 전달하는 것입니다.

## 이벤트 중복과 처리 지연 다루기

### H3. 저장 한 번에 이벤트가 여러 번 올 수 있다

편집기는 파일을 바로 덮어쓰기도 하고, 임시 파일에 쓴 뒤 기존 파일과 교체하기도 합니다.
결과적으로 사용자가 저장 버튼을 한 번 눌러도 watcher에서는 여러 `change`와 `rename` 이벤트가 보일 수 있습니다.

설정 리로드처럼 마지막 상태만 중요하다면 짧은 debounce를 둘 수 있습니다.

```js
import { watch } from 'node:fs/promises';

const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

export async function watchWithDebounce(directory, signal) {
  let lastHandledAt = 0;

  for await (const event of watch(directory, { signal })) {
    const now = Date.now();

    if (now - lastHandledAt < 150) {
      continue;
    }

    await delay(50);
    await refreshState(event);
    lastHandledAt = Date.now();
  }
}
```

이 예제는 단순한 흐름을 보여 줍니다.
실제 요구사항에 따라 마지막 이벤트 이후 일정 시간을 기다리는 debounce, 일정 주기마다 한 번만 실행하는 throttle, 파일별 처리 상태를 관리하는 방식을 선택해야 합니다.

### H3. 느린 처리는 별도 큐와 정책이 필요하다

`for await...of` 안에서 매번 긴 작업을 `await`하면 이벤트 처리는 자연스럽게 직렬화됩니다.
동시 실행 충돌을 막는 장점이 있지만, 처리 속도보다 이벤트 유입 속도가 빠르면 중간 상태를 놓치거나 오래된 작업을 수행할 수 있습니다.

이 경우 먼저 요구사항을 구분하세요.

- 마지막 파일 상태만 중요하면 이벤트를 합치고 최신 상태를 다시 읽는다.
- 모든 파일을 한 번씩 처리해야 하면 별도 bounded queue를 둔다.
- 처리 중 같은 파일이 다시 바뀌면 완료 후 한 번 더 검사한다.
- 작업 실패 시 무한 재시도하지 않고 횟수와 대기 시간을 제한한다.

파일 감지 콜백에서 무제한 Promise를 만드는 방식은 피하는 것이 좋습니다.
처리량 제어가 필요하다면 [Node.js bounded queue 가이드](/development/blog/seo/2026/04/04/nodejs-bounded-queue-length-limit-overload-control-guide.html)의 큐 길이 제한 원칙을 적용할 수 있습니다.

## 설정 파일 리로드 패턴

### H3. 새 설정을 검증한 뒤 현재 설정과 교체한다

파일 변경을 감지했다고 기존 설정을 바로 버리면, 저장 중간 상태나 잘못된 JSON 때문에 서비스가 불안정해질 수 있습니다.
새 파일을 읽고 검증한 뒤 성공했을 때만 현재 설정을 교체하는 흐름이 안전합니다.

```js
import { readFile, watch } from 'node:fs/promises';
import { join } from 'node:path';

function validateConfig(value) {
  if (!Number.isInteger(value.timeoutMs) || value.timeoutMs < 100) {
    throw new Error('timeoutMs must be an integer of at least 100');
  }

  return Object.freeze({
    timeoutMs: value.timeoutMs,
    featureEnabled: value.featureEnabled === true
  });
}

export async function reloadOnChange(directory, signal) {
  const path = join(directory, 'runtime.json');
  let current = validateConfig(
    JSON.parse(await readFile(path, 'utf8'))
  );

  for await (const event of watch(directory, { signal })) {
    if (event.filename && event.filename !== 'runtime.json') {
      continue;
    }

    try {
      const candidate = validateConfig(
        JSON.parse(await readFile(path, 'utf8'))
      );

      current = candidate;
      console.log('runtime config updated');
    } catch (error) {
      console.error('runtime config rejected', {
        message: error.message
      });
    }
  }

  return current;
}
```

실제 서비스에서는 설정 원문이나 비밀 값을 로그에 남기지 않아야 합니다.
검증 실패 메시지는 어떤 필드가 잘못됐는지 알 수 있을 만큼만 구체적으로 만들고, API 키나 토큰 값 자체는 출력하지 않는 편이 좋습니다.

### H3. 시그널 기반 리로드와 파일 감시를 구분한다

파일 감시는 편리하지만 모든 배포 환경에서 같은 신뢰도를 보장하지 않습니다.
컨테이너 볼륨, 네트워크 파일 시스템, 심볼릭 링크 교체 방식에서는 기대한 이벤트가 오지 않을 수 있습니다.

운영 설정 리로드가 반드시 보장되어야 한다면 아래 선택지를 비교해야 합니다.

- 파일 감시 후 상태 재조회
- `SIGHUP` 같은 명시적 프로세스 시그널
- 관리자 API를 통한 검증된 리로드
- 새 설정으로 새 프로세스를 띄우는 재배포

프로세스 시그널을 활용하는 방식은 [Node.js SIGHUP 설정 리로드 가이드](/development/blog/seo/2026/06/01/nodejs-sighup-config-reload-guide.html)와 함께 비교하면 좋습니다.

## 플랫폼 차이와 운영 주의사항

### H3. 로컬에서 동작해도 배포 환경에서 다시 검증한다

파일 감시 기능은 운영체제의 파일 시스템 이벤트 기능에 의존합니다.
플랫폼, 파일 시스템, 마운트 방식에 따라 동작과 지원 범위가 달라질 수 있습니다.
특히 재귀 감시 옵션이나 네트워크 파일 시스템 동작을 모든 환경에서 같다고 가정하면 안 됩니다.

배포 전에는 실제 실행 환경에서 아래 항목을 확인하세요.

- 파일 생성, 수정, 삭제, 이름 교체가 각각 감지되는가?
- 편집기나 배포 도구가 사용하는 저장 방식에서도 동작하는가?
- 감시 대상이 사라졌다가 다시 생길 때 복구되는가?
- 디렉터리 수가 많아져도 운영체제 watcher 한도를 넘지 않는가?
- 컨테이너 볼륨과 네트워크 마운트에서 기대한 이벤트가 오는가?

핵심 기능이 파일 감지에 의존한다면 주기적인 상태 재조회나 재시작 복구 절차 같은 fallback도 준비하는 편이 좋습니다.

### H3. 감시 이벤트를 보안 경계로 사용하지 않는다

watcher는 변경을 빠르게 알아채는 도구이지, 접근 제어나 감사 로그를 완성하는 보안 도구가 아닙니다.
중요 파일의 무결성을 보장해야 한다면 파일 권한, 소유자, 배포 경로, 체크섬 검증, 변경 승인과 감사 기록을 별도로 설계해야 합니다.

또한 외부에서 쓰기 가능한 디렉터리를 감시한다면 다음 위험을 고려해야 합니다.

- 대량 파일 생성으로 이벤트와 처리 작업이 폭증할 수 있다.
- 파일 이름에 제어 문자나 예상하지 못한 문자가 들어올 수 있다.
- 심볼릭 링크나 경로 변경으로 의도하지 않은 파일을 읽을 수 있다.
- 처리 전에 파일이 삭제되거나 내용이 다시 바뀔 수 있다.

이런 입력은 항상 변할 수 있는 외부 상태로 보고, 처리 시점마다 경로와 파일 상태를 다시 확인해야 합니다.

## 테스트 체크리스트

### H3. watcher 테스트는 종료 가능하게 만든다

감시 루프 테스트가 끝나지 않는 가장 흔한 이유는 명시적인 종료 신호가 없기 때문입니다.
테스트에서는 임시 디렉터리와 `AbortController`를 사용하고, 기대한 이벤트를 확인한 뒤 watcher를 종료하세요.

```js
import assert from 'node:assert/strict';
import { mkdtemp, rm, watch, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { test } from 'node:test';

test('detects a file change', async () => {
  const directory = await mkdtemp(join(tmpdir(), 'watch-test-'));
  const controller = new AbortController();
  const watcher = watch(directory, {
    signal: controller.signal
  });

  try {
    const nextEvent = watcher.next();
    await writeFile(join(directory, 'config.json'), '{"enabled":true}');

    const { value, done } = await nextEvent;

    assert.equal(done, false);
    assert.ok(value.eventType === 'rename' || value.eventType === 'change');
  } finally {
    controller.abort();
    await rm(directory, { recursive: true, force: true });
  }
});
```

파일 감시 테스트는 플랫폼 타이밍의 영향을 받을 수 있습니다.
정확한 이벤트 개수나 순서를 지나치게 엄격하게 단정하기보다, 애플리케이션이 보장해야 하는 결과를 검증하는 편이 좋습니다.
내장 테스트 러너의 기본 흐름은 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)에서 이어서 볼 수 있습니다.

### H3. 운영 전 확인할 질문

`fsPromises.watch()`를 도입하기 전에는 아래 기준을 점검하세요.

- 이벤트가 아니라 최종 파일 상태를 기준으로 처리하는가?
- 같은 변경에서 이벤트가 여러 번 와도 결과가 안전한가?
- `filename`이 없을 때의 fallback이 있는가?
- watcher를 `AbortSignal`로 종료할 수 있는가?
- 취소 외의 감시 에러를 숨기지 않는가?
- 느린 작업에 동시성 제한이나 큐 길이 제한이 있는가?
- 파일 이름과 설정 원문에서 민감정보를 로그에 노출하지 않는가?
- 실제 배포 환경의 파일 시스템에서 검증했는가?

`fsPromises.watch()`의 장점은 콜백 기반 파일 감시를 비동기 이터레이터 흐름으로 읽기 쉽게 바꾸는 데 있습니다.
하지만 파일 시스템 이벤트 자체의 불확실성까지 없애 주는 API는 아닙니다.

정리하면 watcher 이벤트는 상태 변경을 다시 확인하라는 신호로 사용하고, `AbortSignal`로 수명을 관리하며, 중복 이벤트와 느린 처리에 별도 정책을 두는 것이 중요합니다.
이 기준을 지키면 설정 리로드, 개발 자동화, 파일 수집 작업을 더 예측 가능한 구조로 운영할 수 있습니다.

## 함께 읽기

- [Node.js fsPromises.glob 가이드: 비동기 이터레이터로 파일 목록을 안전하게 찾는 법](/development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html)
- [Node.js AbortSignal.any 가이드: timeout과 사용자 취소를 함께 처리하는 법](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js SIGHUP 설정 리로드 가이드: 프로세스 재시작 없이 설정을 갱신하는 법](/development/blog/seo/2026/06/01/nodejs-sighup-config-reload-guide.html)

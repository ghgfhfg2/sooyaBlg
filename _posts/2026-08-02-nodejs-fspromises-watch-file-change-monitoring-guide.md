---
layout: post
title: "Node.js fs.promises.watch 가이드: 파일 변경 감시를 안정적으로 처리하는 법"
date: 2026-08-02 08:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-watch-file-change-monitoring-guide
permalink: /development/blog/seo/2026/08/02/nodejs-fspromises-watch-file-change-monitoring-guide.html
alternates:
  ko: /development/blog/seo/2026/08/02/nodejs-fspromises-watch-file-change-monitoring-guide.html
  x_default: /development/blog/seo/2026/08/02/nodejs-fspromises-watch-file-change-monitoring-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, watch, file-watcher, abortsignal, filesystem, javascript]
description: "Node.js fs.promises.watch로 파일 변경 감시기를 만들 때 알아야 할 이벤트 처리, AbortSignal 종료, debounce, 누락 가능성, 재스캔 전략을 실무 예제로 정리합니다."
---

파일 변경 감시는 개발 서버, 정적 사이트 빌드, 설정 리로드, 캐시 무효화에서 자주 필요합니다.
하지만 감시 코드는 생각보다 쉽게 불안정해집니다.
저장 한 번에 이벤트가 여러 번 발생하거나, 편집기가 임시 파일을 만든 뒤 rename으로 교체하거나, 운영체제별 watcher 한도에 걸리면 "파일이 바뀌었는데 반영되지 않는" 문제가 생깁니다.

Node.js에서는 `fs.promises.watch()`를 사용해 파일 시스템 변경 이벤트를 비동기 반복자로 받을 수 있습니다.
콜백 기반 `fs.watch()`보다 `for await...of` 흐름에 잘 맞고, `AbortSignal`로 종료 시점을 명확하게 제어할 수 있다는 장점이 있습니다.
다만 watcher 이벤트는 최종 진실이 아니라 "변경이 있었을 가능성을 알려주는 신호"에 가깝게 다뤄야 합니다.

이 글에서는 `fs.promises.watch()`로 안정적인 파일 변경 감시기를 만들 때 필요한 종료 처리, debounce, 재스캔, 오류 처리 기준을 정리합니다.
파일을 안전하게 교체하는 흐름은 [Node.js rename 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-rename-atomic-publish-guide.html)를 함께 보면 좋고, 감시 대상의 실제 경로 검증은 [Node.js realpath 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-realpath-symlink-guide.html)와 연결해 설계할 수 있습니다.
변경 후 디렉터리 목록을 다시 읽어야 한다면 [Node.js opendir 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html)도 참고할 만합니다.

## fs.promises.watch가 필요한 상황

### 설정 파일 변경을 감지한다

서비스가 설정 파일을 읽고 동작한다면, 프로세스 재시작 없이 변경을 반영하고 싶을 때가 있습니다.
이때 파일 변경 이벤트를 받자마자 설정을 다시 읽으면 간단한 리로드 흐름을 만들 수 있습니다.
단, 편집기나 배포 도구가 파일을 여러 단계로 저장할 수 있으므로 이벤트 하나와 변경 한 번을 일대일로 보면 안 됩니다.

```js
import { watch, readFile } from 'node:fs/promises';

export async function watchConfig(configPath, { signal, onConfig }) {
  for await (const event of watch(configPath, { signal })) {
    if (event.eventType !== 'change' && event.eventType !== 'rename') {
      continue;
    }

    const body = await readFile(configPath, 'utf8');
    onConfig(JSON.parse(body));
  }
}
```

이 예제는 구조를 보여주기 위한 최소 코드입니다.
실무에서는 JSON 파싱 실패, 파일이 잠깐 사라지는 `ENOENT`, 같은 저장 작업에서 반복 발생하는 이벤트를 따로 처리해야 합니다.
감시기는 "바뀐 것 같다"는 신호만 받고, 실제 상태는 파일을 다시 읽어 확인하는 방식이 더 안전합니다.

### 개발 서버의 재빌드 트리거를 만든다

개발 서버나 문서 빌더는 여러 파일을 감시한 뒤 변경이 생기면 다시 빌드합니다.
이때 이벤트가 발생할 때마다 바로 빌드하면 저장 한 번에 빌드가 여러 번 실행될 수 있습니다.
짧은 debounce를 두고 마지막 이벤트 이후 한 번만 작업을 시작하는 편이 안정적입니다.

```js
export function createDebouncedTask(task, delayMs = 150) {
  let timer;

  return () => {
    clearTimeout(timer);
    timer = setTimeout(() => {
      task().catch((error) => {
        console.error('debounced task failed', error);
      });
    }, delayMs);
  };
}
```

debounce 시간은 길수록 중복 작업을 줄이지만 반응성은 떨어집니다.
개발 서버라면 100ms에서 300ms 사이로 시작하고, 대형 빌드나 원격 파일 시스템에서는 더 길게 잡을 수 있습니다.
중요한 것은 이벤트 수가 아니라 최종 파일 상태를 기준으로 작업한다는 점입니다.

## 안정적인 watcher 구조

### AbortSignal로 종료한다

감시기는 오래 살아 있는 작업입니다.
테스트, 개발 서버 종료, 배치 중단 시점에 watcher가 닫히지 않으면 프로세스가 끝나지 않거나 파일 디스크립터가 남을 수 있습니다.
`fs.promises.watch()`는 `signal` 옵션을 받을 수 있으므로 외부에서 종료를 제어하는 구조가 좋습니다.

```js
import { watch } from 'node:fs/promises';

export async function runWatcher(targetDir, { signal, onChange }) {
  try {
    for await (const event of watch(targetDir, { signal, recursive: false })) {
      await onChange(event);
    }
  } catch (error) {
    if (error?.name === 'AbortError') {
      return;
    }

    throw error;
  }
}
```

`AbortError`는 정상 종료로 취급하는 편이 자연스럽습니다.
반대로 권한 오류, watcher 한도 초과, 대상 경로 삭제 같은 오류는 로그와 재시도 정책으로 넘겨야 합니다.
종료와 실패를 구분해야 운영 로그에서 실제 장애를 빠르게 찾을 수 있습니다.

### 이벤트 payload를 과신하지 않는다

watcher 이벤트에는 `eventType`과 `filename`이 들어올 수 있지만, 모든 환경에서 항상 충분한 정보가 보장된다고 기대하면 안 됩니다.
`filename`이 없거나, 파일 저장 방식에 따라 `rename`으로 보이거나, 디렉터리 변경 이벤트만 들어올 수 있습니다.
따라서 이벤트 payload는 힌트로 쓰고 실제 판단은 재조회 결과로 해야 합니다.

```js
import { stat } from 'node:fs/promises';
import { join } from 'node:path';

export async function resolveChangedPath(rootDir, event) {
  if (!event.filename) {
    return { type: 'unknown' };
  }

  const path = join(rootDir, event.filename);

  try {
    const info = await stat(path);
    return {
      type: info.isDirectory() ? 'directory' : 'file',
      path
    };
  } catch (error) {
    if (error?.code === 'ENOENT') {
      return { type: 'removed', path };
    }

    throw error;
  }
}
```

파일이 삭제되거나 rename으로 교체되는 순간에는 `stat()`이 실패할 수 있습니다.
이 실패는 항상 장애가 아니라 변경 과정의 일부일 수 있습니다.
그래서 삭제, 생성, 변경을 모두 처리할 수 있는 작은 상태 모델을 두는 편이 좋습니다.

## 재스캔 전략

### 이벤트 후 전체 상태를 다시 계산한다

파일 감시 시스템은 플랫폼마다 제약이 다릅니다.
짧은 시간에 많은 파일이 바뀌면 이벤트가 합쳐지거나 누락될 수 있고, 네트워크 파일 시스템에서는 동작이 더 불안정할 수 있습니다.
정확도가 중요하다면 이벤트를 받은 뒤 관심 디렉터리의 전체 상태를 다시 계산해야 합니다.

```js
import { opendir } from 'node:fs/promises';
import { join } from 'node:path';

export async function listFiles(rootDir) {
  const files = [];
  const dir = await opendir(rootDir);

  for await (const entry of dir) {
    if (entry.isFile()) {
      files.push(join(rootDir, entry.name));
    }
  }

  return files.sort();
}
```

재스캔은 비용이 있으므로 모든 프로젝트에 필요한 것은 아닙니다.
파일 수가 작거나 정확도가 중요한 설정 디렉터리라면 매번 전체 목록을 다시 만드는 편이 단순합니다.
대형 저장소라면 변경 후보만 처리하되, 주기적으로 전체 스냅샷을 맞추는 절충안이 좋습니다.

### 스냅샷 비교로 변경을 확정한다

이벤트 자체를 작업 단위로 삼기보다, 이전 스냅샷과 현재 스냅샷을 비교하면 결과가 더 명확해집니다.
파일 목록, 수정 시간, 크기, 해시 중 무엇을 비교할지는 비용과 정확도 요구에 따라 정합니다.
개발 서버라면 수정 시간과 크기만으로 충분한 경우가 많습니다.

```js
import { stat } from 'node:fs/promises';

export async function createFileSnapshot(paths) {
  const snapshot = new Map();

  for (const path of paths) {
    const info = await stat(path);
    snapshot.set(path, {
      mtimeMs: info.mtimeMs,
      size: info.size
    });
  }

  return snapshot;
}
```

스냅샷 비교를 쓰면 이벤트가 여러 번 들어와도 "최종적으로 무엇이 달라졌는가"만 처리할 수 있습니다.
또한 watcher가 잠깐 실패한 뒤 재시작됐을 때도 현재 상태를 기준으로 복구하기 쉽습니다.
정확한 증분 동기화가 필요한 도구라면 이 패턴이 특히 유용합니다.

## 오류 처리와 운영 기준

### watcher 실패는 재시작 가능하게 만든다

감시 대상 디렉터리가 삭제되거나 권한이 바뀌면 watcher는 실패할 수 있습니다.
개발 도구라면 사용자에게 오류를 보여주고 종료해도 되지만, 장시간 실행되는 작업이라면 재시작 정책이 필요합니다.
다만 무한히 빠르게 재시도하면 로그와 CPU를 낭비하므로 backoff를 둬야 합니다.

```js
export async function sleep(ms) {
  await new Promise((resolve) => setTimeout(resolve, ms));
}

export async function retryWatcher(start, { signal, logger }) {
  let delayMs = 500;

  while (!signal.aborted) {
    try {
      await start();
      delayMs = 500;
    } catch (error) {
      if (error?.name === 'AbortError') return;

      logger.warn({
        event: 'watcher_failed',
        code: error?.code,
        delayMs
      });

      await sleep(delayMs);
      delayMs = Math.min(delayMs * 2, 10_000);
    }
  }
}
```

재시도 로그에는 전체 파일 경로나 사용자 입력을 그대로 남기지 않는 편이 좋습니다.
대상 종류, 오류 코드, 재시도 간격 정도면 대부분의 운영 진단에 충분합니다.
민감한 경로 구조가 로그 수집 시스템에 남지 않도록 기본값을 보수적으로 잡아야 합니다.

### 운영 환경에서는 watcher를 필수 경로로 두지 않는다

watcher는 편리하지만 운영의 유일한 정합성 장치로 삼기에는 약합니다.
배포 반영, 캐시 무효화, 인덱스 재생성처럼 중요한 흐름은 명시적인 작업 큐나 배포 훅으로 처리하는 편이 더 예측 가능합니다.
watcher는 보조 감지, 개발 편의, 빠른 피드백 용도로 두는 것이 안전합니다.

```js
export function shouldUseWatcher(env) {
  return env.NODE_ENV !== 'production' || env.ENABLE_FILE_WATCHER === 'true';
}
```

운영에서 watcher를 켠다면 관측 지표도 함께 둬야 합니다.
마지막 이벤트 시각, 마지막 재스캔 시각, 실패 횟수, 현재 감시 대상 수를 남기면 장애를 빨리 좁힐 수 있습니다.
감시기가 멈췄는데 서비스는 정상처럼 보이는 상황을 피하려면 health check에 watcher 상태를 포함하는 것도 방법입니다.

## 실무 체크리스트

### 구현 전 확인할 것

- 감시 대상이 파일 하나인지, 디렉터리인지 정한다.
- 변경 이벤트를 즉시 처리할지, debounce 후 처리할지 정한다.
- 이벤트 payload를 신뢰할 범위와 재스캔 범위를 나눈다.
- `AbortSignal`로 종료할 수 있게 만든다.
- watcher 실패 시 종료할지, backoff 후 재시작할지 정한다.

### 코드 리뷰에서 볼 것

- 저장 한 번에 작업이 여러 번 실행되어도 문제가 없는가?
- `AbortError`와 실제 실패가 구분되는가?
- `filename`이 없거나 대상 파일이 사라진 경우를 처리하는가?
- 운영 로그에 민감한 전체 경로가 과하게 남지 않는가?
- 중요한 정합성 요구를 watcher 이벤트 하나에만 의존하지 않는가?

## 마무리

`fs.promises.watch()`는 Node.js에서 파일 변경 감시기를 깔끔하게 만들 수 있는 좋은 도구입니다.
하지만 watcher 이벤트는 완전한 변경 이력이 아니라 작업을 시작하라는 신호로 보는 편이 안전합니다.
`AbortSignal`로 수명을 관리하고, debounce로 중복 작업을 줄이고, 필요한 곳에서는 재스캔과 스냅샷 비교로 최종 상태를 확정해야 합니다.

파일 변경 감시는 특히 개발 환경에서 강력합니다.
운영 환경에서는 배포 훅, 큐, 명시적 재빌드 명령을 기본 경로로 두고 watcher를 보조 장치로 설계하면 장애 가능성을 낮출 수 있습니다.
작은 감시기라도 수명, 오류, 재시도, 로그 기준을 갖추면 오래 실행되는 도구로 믿고 사용할 수 있습니다.

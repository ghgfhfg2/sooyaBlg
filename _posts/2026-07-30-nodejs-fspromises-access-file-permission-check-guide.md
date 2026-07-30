---
layout: post
title: "Node.js fs.promises.access 가이드: 파일 존재와 권한 확인을 안전하게 다루는 법"
date: 2026-07-30 20:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-access-file-permission-check-guide
permalink: /development/blog/seo/2026/07/30/nodejs-fspromises-access-file-permission-check-guide.html
alternates:
  ko: /development/blog/seo/2026/07/30/nodejs-fspromises-access-file-permission-check-guide.html
  x_default: /development/blog/seo/2026/07/30/nodejs-fspromises-access-file-permission-check-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, access, filesystem, permission, toctou, file-check, javascript]
description: "Node.js fs.promises.access로 파일 존재와 권한을 확인할 때 생기는 TOCTOU 경쟁 조건을 피하는 방법을 정리합니다. access가 적합한 경우, open/readFile/writeFile 우선 패턴, 에러 처리와 테스트 기준까지 실무 예제로 설명합니다."
---

파일을 읽거나 쓰기 전에 "이 파일이 있나?", "현재 프로세스가 읽을 수 있나?"를 먼저 확인하고 싶을 때가 있습니다.
Node.js에는 이 용도로 `fs.promises.access()`가 있습니다.
하지만 `access()`를 무조건 사전 검사로 넣으면 오히려 더 불안정한 코드가 됩니다.

공식 문서도 `fsPromises.access()`로 접근 가능 여부를 확인한 뒤 `open()`을 호출하는 방식은 권장하지 않는다고 설명합니다.
확인 시점과 실제 사용 시점 사이에 파일 상태가 바뀔 수 있기 때문입니다.
이 문제는 흔히 TOCTOU, 즉 time-of-check to time-of-use 경쟁 조건이라고 부릅니다.

이 글에서는 `fs.promises.access()`를 어디에 쓰면 좋은지, 파일을 실제로 읽고 쓸 때는 왜 `open()`, `readFile()`, `writeFile()` 결과를 직접 처리하는 편이 나은지 정리합니다.
파일 생성 전 디스크 여유 공간까지 확인해야 한다면 [Node.js statfs 가이드](/development/blog/seo/2026/07/30/nodejs-fspromises-statfs-disk-space-check-guide.html)를 함께 보고, 중복 실행 방지가 필요하면 [Node.js lock file 가이드](/development/blog/seo/2026/07/28/nodejs-lock-file-atomic-job-deduplication-guide.html)를 먼저 적용하는 것이 좋습니다.

## fs.promises.access가 하는 일

### H3. 파일의 접근 가능 여부만 확인한다

`fs.promises.access(path, mode)`는 지정한 경로에 대해 현재 프로세스가 접근 가능한지 확인합니다.
기본 모드는 `F_OK`이고, 파일이 존재하는지 확인하는 의미에 가깝습니다.
읽기 가능 여부는 `R_OK`, 쓰기 가능 여부는 `W_OK`, 실행 가능 여부는 `X_OK`를 사용할 수 있습니다.

```js
import { access, constants } from 'node:fs/promises';

export async function canReadConfig(path) {
  try {
    await access(path, constants.R_OK);
    return true;
  } catch {
    return false;
  }
}
```

이 함수는 설정 파일이 보이는지 빠르게 판단할 때 쓸 수 있습니다.
다만 이 결과는 "지금 이 순간 읽기 가능해 보인다"는 뜻일 뿐입니다.
다음 줄에서 실제로 `readFile()`을 호출하는 순간에는 파일이 삭제되거나 권한이 바뀌었을 수 있습니다.

### H3. mode를 조합할 수 있다

`mode`는 bitwise OR로 조합할 수 있습니다.
예를 들어 파일이 존재하고 읽기와 쓰기가 모두 가능한지 확인하려면 `R_OK | W_OK`를 넘깁니다.

```js
import { access, constants } from 'node:fs/promises';

export async function canUpdateFile(path) {
  try {
    await access(path, constants.R_OK | constants.W_OK);
    return true;
  } catch {
    return false;
  }
}
```

이런 함수는 UI에서 "수정 가능해 보이는 파일"을 표시하거나, 설정 점검 단계에서 친절한 메시지를 만드는 데 적합합니다.
하지만 실제 수정 작업에서는 이 검사만 믿지 말고 `open()`이나 `writeFile()`의 실패를 반드시 처리해야 합니다.

## 사전 검사로 쓰면 위험한 이유

### H3. 확인과 사용 사이에 파일 상태가 바뀔 수 있다

아래 코드는 자연스럽지만 위험합니다.
`access()`가 성공한 뒤 `readFile()`이 실행되기 전에 다른 프로세스가 파일을 지우거나 권한을 바꾸면 읽기는 실패합니다.

```js
import { access, readFile, constants } from 'node:fs/promises';

export async function readConfig(path) {
  await access(path, constants.R_OK);
  return readFile(path, 'utf8');
}
```

이 구조의 문제는 실패가 사라지지 않는다는 점입니다.
사전 검사를 추가했지만 결국 `readFile()`의 `ENOENT`, `EACCES`, `EPERM` 같은 오류는 여전히 처리해야 합니다.
그렇다면 읽기 전에 같은 성격의 I/O를 한 번 더 실행한 셈이 됩니다.

파일 시스템은 공유 자원입니다.
배포 스크립트, 백업 작업, 로그 회전, 테스트 정리 코드, 사용자의 수동 조작이 같은 경로를 동시에 건드릴 수 있습니다.
그래서 파일을 실제로 사용할 코드가 실패를 직접 감당하는 구조가 더 단순하고 안전합니다.

### H3. 읽기는 readFile 실패를 바로 처리한다

파일을 읽는 목적이라면 먼저 읽고, 실패 원인을 분기하는 편이 좋습니다.

```js
import { readFile } from 'node:fs/promises';

export async function loadJsonConfig(path) {
  try {
    const body = await readFile(path, 'utf8');
    return JSON.parse(body);
  } catch (error) {
    if (error?.code === 'ENOENT') {
      return {};
    }

    if (error?.code === 'EACCES' || error?.code === 'EPERM') {
      throw new Error('config file is not readable', { cause: error });
    }

    throw error;
  }
}
```

이 코드는 사전 검사 없이 실제 작업의 결과를 기준으로 판단합니다.
파일이 없으면 기본 설정을 반환하고, 권한 문제는 운영자가 알아볼 수 있는 메시지로 감쌉니다.
JSON 문법 오류는 파일 접근 문제가 아니므로 그대로 상위로 올립니다.

에러를 감쌀 때 원본 경로나 사용자 입력을 로그에 그대로 남기지 않는 것도 중요합니다.
경로에 사용자명, 프로젝트명, 고객 식별자 같은 민감한 정보가 섞일 수 있기 때문입니다.
필요하다면 파일 종류, 오류 코드, 작업 이름 정도만 기록합니다.

## 쓰기 작업에서의 안전한 패턴

### H3. 새 파일 생성은 wx 플래그로 원자적으로 처리한다

"없으면 만들기" 작업은 `access()`로 존재 여부를 확인한 뒤 `writeFile()`을 호출하면 경쟁 조건이 생깁니다.
두 프로세스가 동시에 확인하면 둘 다 "없다"고 판단할 수 있고, 뒤늦게 한쪽이 다른 쪽의 결과를 덮어쓸 수 있습니다.

새 파일을 한 번만 만들어야 한다면 `open(path, 'wx')`를 사용합니다.
`wx`는 파일이 이미 있으면 실패하도록 만드는 exclusive 생성 플래그입니다.

```js
import { open } from 'node:fs/promises';

export async function createOnce(path, body) {
  let file;

  try {
    file = await open(path, 'wx');
    await file.writeFile(body, 'utf8');
    return { created: true };
  } catch (error) {
    if (error?.code === 'EEXIST') {
      return { created: false };
    }

    throw error;
  } finally {
    await file?.close();
  }
}
```

락 파일도 같은 원리로 설계할 수 있습니다.
자세한 중복 실행 방지는 [Node.js lock file 가이드](/development/blog/seo/2026/07/28/nodejs-lock-file-atomic-job-deduplication-guide.html)에서 다뤘습니다.

### H3. 덮어쓰기는 임시 파일과 rename으로 분리한다

기존 파일을 갱신해야 한다면 "쓸 수 있는지 확인하고 덮어쓰기"보다 임시 파일을 만든 뒤 `rename()`으로 교체하는 흐름이 더 안전합니다.
쓰기 도중 실패해도 기존 파일을 유지할 수 있고, 성공한 결과만 마지막에 반영할 수 있습니다.

```js
import { rename, rm, writeFile } from 'node:fs/promises';

export async function writeJsonAtomic(path, value) {
  const tempPath = `${path}.${process.pid}.tmp`;
  const body = `${JSON.stringify(value, null, 2)}\n`;

  try {
    await writeFile(tempPath, body, {
      encoding: 'utf8',
      flag: 'wx'
    });

    await rename(tempPath, path);
  } catch (error) {
    await rm(tempPath, { force: true });
    throw error;
  }
}
```

파일 내구성까지 필요하면 단순 `writeFile()`만으로 충분하지 않을 수 있습니다.
그 경우에는 [Node.js writeFile flush 가이드](/development/blog/seo/2026/07/29/nodejs-fspromises-writefile-flush-durable-write-guide.html)처럼 flush, 임시 파일, rename, 실패 복구 범위를 함께 설계해야 합니다.

## access가 적합한 경우

### H3. 실행 전 환경 점검에는 유용하다

`access()`가 나쁜 API라는 뜻은 아닙니다.
실제 파일 작업을 대체하려고 쓰면 위험하지만, 실행 전 환경 점검이나 사용자 친화적인 안내에는 충분히 유용합니다.

예를 들어 CLI 시작 시 필요한 디렉터리와 설정 파일을 점검하고, 누락된 항목을 한 번에 보여줄 수 있습니다.

```js
import { access, constants } from 'node:fs/promises';

const checks = [
  { name: 'config', path: './config/app.json', mode: constants.R_OK },
  { name: 'output', path: './dist', mode: constants.W_OK },
  { name: 'template', path: './templates/base', mode: constants.R_OK }
];

export async function inspectWorkspace() {
  const results = await Promise.all(checks.map(async (check) => {
    try {
      await access(check.path, check.mode);
      return { name: check.name, ok: true };
    } catch (error) {
      return {
        name: check.name,
        ok: false,
        code: error.code
      };
    }
  }));

  return results;
}
```

이런 점검 결과는 "시작 전에 무엇이 부족한지 알려주는 정보"로만 사용합니다.
실제 빌드나 배포 단계에서는 여전히 각 파일 작업의 오류를 처리해야 합니다.

### H3. 권한 오류 메시지를 미리 다듬을 수 있다

사용자에게 작업 가능 여부를 미리 알려주는 도구라면 `access()`가 읽기 좋습니다.
예를 들어 로컬 개발 도구에서 output directory가 쓰기 가능한지 확인하고, 실패하면 권한 변경이나 경로 변경을 안내할 수 있습니다.

```js
import { access, constants } from 'node:fs/promises';

export async function assertWritableOutputDir(path) {
  try {
    await access(path, constants.W_OK);
  } catch (error) {
    throw new Error('output directory is not writable', {
      cause: error
    });
  }
}
```

다만 이 함수 이름도 중요합니다.
`ensureWritableOutputDir`처럼 실제 쓰기 성공을 보장하는 이름보다, `assertWritableOutputDir`처럼 현재 상태를 확인한다는 의미가 더 정확합니다.
이름이 보장 범위를 과장하면 호출하는 쪽에서 에러 처리를 빼먹기 쉽습니다.

## 테스트 기준

### H3. 파일 없음과 권한 오류를 분리한다

파일 접근 코드는 운영체제와 실행 계정에 영향을 받습니다.
테스트에서는 최소한 파일 없음, 이미 존재함, 읽기 실패, 쓰기 실패를 분리해 확인하는 것이 좋습니다.

```js
import assert from 'node:assert/strict';
import { mkdtemp, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import test from 'node:test';
import { createOnce, loadJsonConfig } from './files.js';

test('loadJsonConfig returns default object when file is missing', async () => {
  const dir = await mkdtemp(join(tmpdir(), 'access-guide-'));
  const config = await loadJsonConfig(join(dir, 'missing.json'));

  assert.deepEqual(config, {});
});

test('createOnce does not overwrite an existing file', async () => {
  const dir = await mkdtemp(join(tmpdir(), 'access-guide-'));
  const path = join(dir, 'result.txt');

  await writeFile(path, 'first\n');
  const result = await createOnce(path, 'second\n');

  assert.deepEqual(result, { created: false });
});
```

권한 오류 테스트는 플랫폼별 차이가 큽니다.
Windows, macOS, Linux, 컨테이너, CI runner의 파일 권한 모델이 다를 수 있으므로 모든 환경에서 같은 방식으로 검증하기 어렵습니다.
권한 테스트가 불안정하다면 작은 adapter를 만들고, 단위 테스트에서는 `EACCES`나 `EPERM`을 던지는 fake를 주입하는 방식이 더 안정적입니다.

## 실무 체크리스트

- `access()` 결과를 실제 읽기와 쓰기의 성공 보장으로 오해하지 않는다.
- 파일 읽기는 `readFile()` 실패를 기준으로 `ENOENT`, `EACCES`, `EPERM`을 분기한다.
- 새 파일 생성은 `access()` 사전 검사 대신 `open(path, 'wx')`를 우선 검토한다.
- 덮어쓰기는 임시 파일과 `rename()`으로 실패 복구 범위를 줄인다.
- 로그에는 전체 경로나 사용자 입력을 그대로 남기지 않는다.
- 환경 점검용 `access()`와 실제 작업 오류 처리를 분리한다.
- 테스트에서 파일 없음, 이미 존재함, 권한 실패, 정리 실패를 따로 확인한다.

## FAQ

### H3. fs.promises.access를 아예 쓰지 말아야 하나요?

그렇지 않습니다.
작업 전 환경 점검, CLI 진단, 사용자 안내처럼 "현재 접근 가능해 보이는지" 확인하는 용도에는 유용합니다.
다만 실제 파일을 읽거나 쓰기 직전에 성공 보장처럼 사용하는 패턴은 피하는 것이 좋습니다.

### H3. 파일 존재 여부만 확인하려면 access와 stat 중 무엇이 낫나요?

파일의 존재 여부만 필요하면 `access(path)`가 단순합니다.
파일 크기, 수정 시간, 파일인지 디렉터리인지 같은 메타데이터가 필요하면 `stat()`이 더 적합합니다.
하지만 둘 다 실제 사용 시점의 성공을 보장하지는 않습니다.

### H3. 쓰기 가능 여부를 확인한 뒤 writeFile을 호출하면 안 되나요?

사용자 안내 목적이라면 괜찮습니다.
하지만 실제 저장 로직에서는 `writeFile()` 자체의 실패를 반드시 처리해야 합니다.
권한, 디스크 부족, 경로 삭제, 파일 잠금 같은 상황은 사전 검사 뒤에도 발생할 수 있습니다.

## 정리

`fs.promises.access()`는 파일 접근 가능 여부를 확인하는 간단한 도구지만, 실제 파일 작업의 안전성을 보장하는 장치는 아닙니다.
확인과 사용 사이에 상태가 바뀔 수 있으므로 읽기와 쓰기 코드는 실제 I/O 실패를 기준으로 설계해야 합니다.

실무에서는 `access()`를 환경 점검과 안내에 쓰고, 읽기는 `readFile()` 오류 처리, 새 파일 생성은 `open(path, 'wx')`, 안전한 갱신은 임시 파일과 `rename()`으로 나누는 기준이 가장 다루기 쉽습니다.
이렇게 역할을 분리하면 코드가 짧아지고, 경쟁 조건과 운영 오류를 더 명확하게 처리할 수 있습니다.

---
layout: post
title: "Node.js child_process spawn AbortSignal 가이드: 외부 명령을 timeout과 함께 안전하게 중단하는 법"
date: 2026-07-25 08:00:00 +0900
lang: ko
translation_key: nodejs-child-process-spawn-abortsignal-timeout-guide
permalink: /development/blog/seo/2026/07/25/nodejs-child-process-spawn-abortsignal-timeout-guide.html
alternates:
  ko: /development/blog/seo/2026/07/25/nodejs-child-process-spawn-abortsignal-timeout-guide.html
  x_default: /development/blog/seo/2026/07/25/nodejs-child-process-spawn-abortsignal-timeout-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, child-process, spawn, abortsignal, timeout, cli, backend, reliability]
description: "Node.js child_process.spawn()에 AbortSignal과 timeout을 연결해 외부 명령을 안전하게 중단하고, stdout 제한, 종료 코드 분류, 권한 경계를 함께 설계하는 실무 패턴을 정리합니다."
---

Node.js 서비스와 배치 스크립트는 종종 외부 명령을 실행합니다.
이미지 변환, Git 정보 조회, 문서 빌드, 압축, 코드 생성처럼 Node.js 안에서 전부 다시 구현하기보다 검증된 CLI를 호출하는 편이 더 단순한 작업이 많습니다.
하지만 외부 프로세스는 한 번 실행하면 Node.js 이벤트 루프 밖에서 별도로 움직입니다.
timeout 없이 기다리거나, 출력 크기를 제한하지 않거나, 사용자 입력을 명령 문자열에 섞으면 작은 자동화가 운영 리스크가 될 수 있습니다.

이 글에서는 Node.js `child_process.spawn()`에 `AbortSignal`을 연결해 외부 명령을 중단 가능한 작업으로 감싸는 방법을 정리합니다.
핵심은 명령 실행을 단순한 `spawn()` 호출이 아니라, **시간 제한, 출력 제한, 종료 상태 분류가 있는 작은 실행 경계**로 만드는 것입니다.
[Node.js AbortSignal.any 가이드](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html), [Node.js timeout budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html), [Node.js Permission Model 가이드](/development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html)와 함께 보면 외부 실행 권한과 취소 정책을 더 일관되게 설계할 수 있습니다.

## child_process 실행에 AbortSignal이 필요한 이유

### H3. 외부 명령은 Node.js 함수처럼 자동으로 멈추지 않는다

`await`로 감싼 일반 함수는 호출자가 취소 흐름을 설계하면 비교적 좁은 범위에서 멈출 수 있습니다.
반대로 외부 프로세스는 OS 프로세스로 실행됩니다.
부모 Node.js 코드가 응답을 포기했더라도 자식 프로세스가 계속 CPU를 쓰거나 파일을 만들 수 있습니다.

예를 들어 업로드된 이미지를 변환하는 API에서 사용자가 요청을 취소했는데 변환 프로세스가 계속 돈다면 서버 리소스가 낭비됩니다.
작업 큐에서도 특정 명령이 멈추지 않으면 다음 작업이 밀리고, 워커 종료도 늦어집니다.
그래서 외부 명령에는 처음부터 중단 조건을 붙이는 편이 안전합니다.

Node.js의 `spawn()`은 옵션으로 `signal`을 받을 수 있습니다.
호출자가 `AbortController`를 중단하면 실행 중인 자식 프로세스에도 중단 신호를 전달할 수 있습니다.

### H3. shell 문자열보다 argv 배열이 기본이다

외부 명령을 실행할 때 가장 먼저 정해야 할 기준은 "어떻게 명령을 구성할 것인가"입니다.
사용자 입력이나 파일명을 포함해야 한다면 shell 문자열을 조합하는 방식은 피하는 편이 좋습니다.

```js
// 피해야 할 예: 사용자 입력이 shell 문법으로 해석될 수 있다.
spawn(`git log -- ${fileName}`, { shell: true });
```

대부분의 내부 도구에서는 명령과 인자를 분리해서 넘기면 충분합니다.

```js
spawn('git', ['log', '--', fileName], {
  stdio: ['ignore', 'pipe', 'pipe']
});
```

이 방식은 인자 경계가 명확합니다.
`fileName`에 공백이나 특수 문자가 있어도 shell 명령으로 재해석하지 않습니다.
정말 shell 기능이 필요한 경우가 아니라면 `shell: true`를 기본값처럼 쓰지 않는 편이 좋습니다.

## timeout과 호출자 취소를 함께 묶기

### H3. AbortSignal.timeout으로 명령별 상한을 둔다

가장 작은 기본형은 명령마다 timeout signal을 붙이는 것입니다.
아래 함수는 외부 명령을 실행하고 stdout, stderr, 종료 상태를 Promise로 돌려줍니다.

```js
import { spawn } from 'node:child_process';

export function runCommand(command, args, {
  cwd,
  env,
  timeoutMs = 10_000,
  signal
} = {}) {
  const timeoutSignal = AbortSignal.timeout(timeoutMs);
  const runSignal = signal
    ? AbortSignal.any([signal, timeoutSignal])
    : timeoutSignal;

  return new Promise((resolve, reject) => {
    const child = spawn(command, args, {
      cwd,
      env,
      signal: runSignal,
      stdio: ['ignore', 'pipe', 'pipe']
    });

    let stdout = '';
    let stderr = '';

    child.stdout.setEncoding('utf8');
    child.stderr.setEncoding('utf8');

    child.stdout.on('data', (chunk) => {
      stdout += chunk;
    });

    child.stderr.on('data', (chunk) => {
      stderr += chunk;
    });

    child.on('error', reject);

    child.on('close', (code, signalName) => {
      resolve({ code, signal: signalName, stdout, stderr });
    });
  });
}
```

이 코드는 요청 취소와 내부 timeout을 하나의 signal로 묶습니다.
상위 요청이 먼저 취소되면 자식 프로세스도 중단되고, 상위 요청이 살아 있더라도 `timeoutMs`를 넘기면 작업이 끊깁니다.

### H3. 전체 요청 예산과 명령 timeout을 분리한다

명령 timeout은 단독 숫자로 정하면 쉽게 흔들립니다.
API 요청 전체가 2초 안에 끝나야 하는데 외부 명령 하나에 2초를 주면 응답 직렬화, 로그 기록, fallback 처리 시간이 남지 않습니다.
따라서 명령 실행 함수는 남은 시간 예산을 받아서 timeout을 줄이는 구조가 더 안전합니다.

```js
function remainingMs(deadlineMs) {
  return Math.max(1, deadlineMs - Date.now());
}

export async function buildPreview(filePath, { deadlineMs, signal }) {
  const result = await runCommand('node', ['scripts/render-preview.js', filePath], {
    timeoutMs: Math.min(remainingMs(deadlineMs), 5_000),
    signal
  });

  if (result.code !== 0) {
    throw new Error(`preview command failed with exit code ${result.code}`);
  }

  return result.stdout;
}
```

이 패턴은 외부 명령이 전체 요청 시간을 독점하지 않게 만듭니다.
여러 명령을 순서대로 실행해야 한다면 각 단계가 남은 budget 안에서 동작하도록 제한해야 합니다.

## 출력 크기와 오류 분류

### H3. stdout과 stderr는 반드시 크기를 제한한다

외부 명령의 출력은 예상보다 커질 수 있습니다.
빌드 로그, 테스트 실패 메시지, `git diff`, 이미지 도구의 경고가 길어지면 문자열 누적으로 메모리를 많이 쓸 수 있습니다.
그래서 실행 래퍼에는 출력 상한을 두는 편이 좋습니다.

```js
function appendLimited(buffer, chunk, maxBytes) {
  const next = buffer + chunk;

  if (Buffer.byteLength(next, 'utf8') <= maxBytes) {
    return next;
  }

  return next.slice(0, maxBytes) + '\n[output truncated]';
}
```

실제 래퍼에서는 stdout과 stderr 각각에 상한을 둡니다.
상한을 넘는 순간 프로세스를 중단할지, 뒤쪽 출력만 버릴지는 작업 성격에 따라 정해야 합니다.
로그 분석용 명령이라면 중단이 낫고, 짧은 메타데이터만 필요하다면 잘라낸 출력으로 충분할 수 있습니다.

```js
const maxOutputBytes = 256 * 1024;

child.stdout.on('data', (chunk) => {
  stdout = appendLimited(stdout, chunk, maxOutputBytes);
});

child.stderr.on('data', (chunk) => {
  stderr = appendLimited(stderr, chunk, maxOutputBytes);
});
```

출력을 로그에 남길 때도 주의해야 합니다.
CLI 출력에는 파일 경로, 내부 호스트, 토큰, 사용자 식별자 같은 민감정보가 섞일 수 있습니다.
공개 문서나 사용자 응답으로 그대로 전달하지 말고 필요한 부분만 요약하거나 마스킹해야 합니다.

### H3. 종료 코드, signal, abort 오류를 분리한다

외부 명령 실패는 하나의 오류가 아닙니다.
명령을 찾지 못한 경우, 명령이 비정상 종료한 경우, timeout으로 중단된 경우, 호출자가 취소한 경우를 구분해야 합니다.

```js
export class CommandExitError extends Error {
  constructor(message, details) {
    super(message);
    this.name = 'CommandExitError';
    this.details = details;
  }
}

function assertCommandSucceeded(result, command) {
  if (result.code === 0) {
    return;
  }

  throw new CommandExitError(`${command} exited with ${result.code}`, {
    code: result.code,
    signal: result.signal,
    stderr: result.stderr.slice(0, 2000)
  });
}
```

`spawn()` 자체에서 발생하는 `error` 이벤트는 실행 시작 실패에 가깝습니다.
예를 들어 명령 파일을 찾지 못하면 `ENOENT`가 날 수 있습니다.
반대로 프로세스가 실행된 뒤 종료 코드 1로 끝나는 것은 명령이 실패 결과를 돌려준 것입니다.
운영 로그에서는 두 상황을 나눠야 재시도와 알림 기준을 다르게 잡을 수 있습니다.

## 실무 래퍼 설계

### H3. 허용 명령을 좁게 감싼다

범용 `runCommand(command, args)`를 아무 곳에서나 열어 두면 위험합니다.
서비스 코드에서는 허용된 명령을 작은 함수로 감싸고, 인자 검증을 그 함수 안에 두는 편이 좋습니다.

```js
const ALLOWED_FORMATS = new Set(['webp', 'jpeg', 'png']);

export async function convertImage(inputPath, outputPath, {
  format,
  signal,
  timeoutMs = 8_000
}) {
  if (!ALLOWED_FORMATS.has(format)) {
    throw new Error('unsupported image format');
  }

  const result = await runCommand('magick', [
    inputPath,
    '-strip',
    outputPath
  ], {
    signal,
    timeoutMs
  });

  assertCommandSucceeded(result, 'magick');
}
```

이 예시는 구조를 보여 주기 위한 코드입니다.
운영 환경에서는 `inputPath`와 `outputPath`가 허용된 작업 디렉터리 안에 있는지 확인해야 합니다.
사용자 입력이 파일 경로로 직접 이어지면 경로 우회나 민감 파일 접근 문제가 생길 수 있습니다.

### H3. 환경 변수는 필요한 값만 넘긴다

자식 프로세스는 기본적으로 부모 프로세스의 환경을 물려받을 수 있습니다.
그 안에는 API 토큰, 데이터베이스 URL, 내부 서비스 주소가 들어 있을 수 있습니다.
외부 명령이 정말 그 값들을 필요로 하지 않는다면 제한된 환경만 넘기는 편이 안전합니다.

```js
const safeEnv = {
  PATH: process.env.PATH,
  LANG: 'C.UTF-8',
  NODE_ENV: 'production'
};

await runCommand('node', ['scripts/build-public-index.js'], {
  env: safeEnv,
  timeoutMs: 15_000
});
```

`PATH`조차 고정된 실행 환경에서는 절대 경로로 대체할 수 있습니다.
CI나 서버에서 어떤 바이너리가 실행되는지 중요하다면 배포 이미지에 포함된 경로를 명확히 지정하고, 버전도 로그에 남겨야 합니다.

## 테스트와 운영 체크리스트

### H3. 성공보다 실패 경로를 먼저 테스트한다

외부 명령 래퍼는 정상 실행보다 실패 경로에서 값이 드러납니다.
테스트에서는 작은 Node.js 스크립트를 자식 프로세스로 실행해 timeout, 종료 코드, stderr 제한을 검증할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('runCommand reports non-zero exit code', async () => {
  const result = await runCommand(process.execPath, [
    '-e',
    'console.error("failed"); process.exit(3)'
  ]);

  assert.equal(result.code, 3);
  assert.match(result.stderr, /failed/);
});
```

timeout 테스트는 너무 긴 실제 대기 시간을 쓰지 않는 편이 좋습니다.
짧은 timeout을 두고 `setTimeout`으로 대기하는 자식 스크립트를 실행하면 빠르게 검증할 수 있습니다.

```js
test('runCommand can be aborted by timeout', async () => {
  await assert.rejects(
    runCommand(process.execPath, ['-e', 'setTimeout(() => {}, 10_000)'], {
      timeoutMs: 50
    }),
    /AbortError|aborted/i
  );
});
```

테스트 환경에 따라 abort 오류의 이름이나 메시지는 다르게 보일 수 있으므로, 실제 래퍼에서는 자체 오류 타입으로 한 번 감싸는 방법도 좋습니다.

### H3. 배포 전에 확인할 것

`child_process.spawn()`을 운영 코드에 넣기 전에는 아래 항목을 확인해야 합니다.

- `shell: true` 없이 command와 argv 배열로 실행하는가?
- timeout과 상위 호출자의 `AbortSignal`을 함께 연결했는가?
- stdout, stderr 크기 제한을 두었는가?
- 종료 코드, 실행 실패, abort, timeout을 로그에서 구분하는가?
- 자식 프로세스에 넘기는 환경 변수를 최소화했는가?
- 사용자 입력이 명령 이름, shell 문자열, 임의 파일 경로로 직접 이어지지 않는가?
- Node.js Permission Model이나 컨테이너 권한으로 child process 사용 범위를 제한할 수 있는가?

외부 명령 실행은 편리하지만, 권한이 큰 작업입니다.
작은 래퍼에서 시간, 출력, 권한, 오류 분류를 고정해 두면 호출부는 더 단순해지고 운영 리스크는 줄어듭니다.

## FAQ

### H3. execFile과 spawn 중 무엇을 써야 하나요?

출력이 작고 결과를 한 번에 받아도 되는 명령은 `execFile()`이 편할 수 있습니다.
출력이 크거나 스트림으로 읽어야 하거나, stdout과 stderr를 직접 제한하고 싶다면 `spawn()`이 더 적합합니다.
사용자 입력이 섞인다면 두 경우 모두 shell 문자열 대신 command와 args를 분리하는 기준이 중요합니다.

### H3. timeout이면 항상 재시도해도 되나요?

항상 그렇지는 않습니다.
읽기 전용 명령이나 임시 파일을 안전하게 정리하는 작업은 제한적으로 재시도할 수 있습니다.
하지만 외부 시스템에 쓰기 작업을 했거나, 일부 파일이 생성됐을 수 있다면 멱등성과 정리 정책을 먼저 확인해야 합니다.

### H3. stderr를 사용자에게 그대로 보여 줘도 되나요?

권장하지 않습니다.
stderr에는 내부 경로, 설정 이름, 라이브러리 버전, 민감한 입력 일부가 섞일 수 있습니다.
사용자에게는 짧은 실패 메시지를 보여 주고, 운영 로그에는 마스킹과 길이 제한을 적용한 진단 정보만 남기는 편이 안전합니다.

## 관련 글

- [Node.js AbortSignal.any 가이드: timeout과 사용자 취소를 함께 묶는 법](/development/blog/seo/2026/04/23/nodejs-abortsignal-any-timeout-cancellation-guide.html)
- [Node.js timeout budget 가이드: 요청 전체 시간 예산을 하위 작업에 전달하는 법](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)
- [Node.js Permission Model 가이드: 런타임 권한으로 파일·프로세스 접근을 제한하는 법](/development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html)

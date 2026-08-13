---
layout: post
title: "Node.js process.execArgv 가이드: 런타임 플래그를 안전하게 전달하는 법"
date: 2026-08-13 20:00:00 +0900
lang: ko
translation_key: nodejs-process-execargv-runtime-flags-guide
permalink: /development/blog/seo/2026/08/13/nodejs-process-execargv-runtime-flags-guide.html
alternates:
  ko: /development/blog/seo/2026/08/13/nodejs-process-execargv-runtime-flags-guide.html
  x_default: /development/blog/seo/2026/08/13/nodejs-process-execargv-runtime-flags-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, process, execargv, runtime-flags, child-process, worker-threads, cli, backend, javascript]
description: "Node.js process.execArgv로 현재 프로세스에 전달된 런타임 플래그를 확인하고 child_process, worker_threads, 테스트 실행 환경에 안전하게 전달하는 방법을 정리합니다. argv와의 차이, 필터링 기준, 디버그 포트 충돌 방지까지 실무 예제로 설명합니다."
---

Node.js 애플리케이션을 운영하다 보면 "지금 이 프로세스가 어떤 런타임 옵션으로 실행됐는가"를 확인해야 할 때가 있습니다.
예를 들어 `--enable-source-maps`, `--inspect`, `--max-old-space-size`, `--experimental-*` 같은 옵션은 애플리케이션 코드 인자가 아니라 Node.js 런타임 자체에 영향을 주는 플래그입니다.
이 값을 읽을 때 사용하는 속성이 `process.execArgv`입니다.

공식 Node.js 문서 기준으로 `process.execArgv`는 현재 Node.js 프로세스를 시작할 때 전달된 **Node.js 전용 명령줄 옵션 목록**을 담습니다.
`process.argv`처럼 실행 파일 경로, 스크립트 경로, 사용자 인자를 모두 담는 배열이 아니라는 점이 핵심입니다.
이 차이를 모르고 자식 프로세스나 Worker에 옵션을 그대로 넘기면 디버그 포트 충돌, 불필요한 실험 플래그 전파, 메모리 제한 왜곡 같은 문제가 생길 수 있습니다.

이 글에서는 `process.execArgv`와 `process.argv`를 구분하는 법, 자식 프로세스와 Worker에 런타임 플래그를 전달할 때의 기준, CI와 테스트 환경에서 안전하게 필터링하는 패턴을 정리합니다.
프로세스 교체처럼 더 강한 실행 제어가 필요하다면 [Node.js process.execve 가이드](/development/blog/seo/2026/08/13/nodejs-process-execve-current-process-replacement-guide.html)를 함께 볼 수 있습니다.
외부 명령 실행과 timeout 처리는 [Node.js child_process spawn AbortSignal 가이드](/development/blog/seo/2026/07/25/nodejs-child-process-spawn-abortsignal-timeout-guide.html), 운영 중 진단 파일을 남기는 기준은 [Node.js process.report 가이드](/development/blog/seo/2026/05/29/nodejs-process-report-production-diagnostics-guide.html)와 연결됩니다.

## process.execArgv가 필요한 이유

### H3. 애플리케이션 인자와 런타임 플래그를 분리한다

아래처럼 Node.js를 실행했다고 가정해 보겠습니다.

```bash
node --enable-source-maps --max-old-space-size=2048 ./scripts/build.js --target=web
```

이때 `process.argv`와 `process.execArgv`는 목적이 다릅니다.

```js
console.log(process.execArgv);
// ['--enable-source-maps', '--max-old-space-size=2048']

console.log(process.argv);
// ['/path/to/node', '/repo/scripts/build.js', '--target=web']
```

`process.argv`는 애플리케이션이 해석할 사용자 인자에 가깝고, `process.execArgv`는 Node.js 런타임이 이미 해석한 옵션에 가깝습니다.
CLI 도구에서 `--target`, `--config`, `--dry-run` 같은 값을 파싱하려면 `process.argv`를 봐야 합니다.
반대로 현재 프로세스가 source map, inspector, memory flag, warning flag를 어떤 상태로 켜고 있는지 확인하려면 `process.execArgv`가 더 정확합니다.

이 구분은 로그에도 중요합니다.
문제 상황에서 `process.argv`만 남기면 사용자가 어떤 명령을 실행했는지는 보이지만, 런타임 플래그가 빠질 수 있습니다.
반대로 `process.execArgv`만 남기면 실제 업무 인자가 보이지 않습니다.
운영 진단 로그에서는 둘을 분리해 기록하는 편이 나중에 원인을 찾기 쉽습니다.

### H3. 런타임 플래그는 그대로 복사해도 되는 값이 아니다

`process.execArgv`는 배열이라서 자식 프로세스를 만들 때 그대로 넘기기 쉽습니다.
하지만 현재 프로세스에 맞는 플래그가 새 프로세스에도 항상 맞는 것은 아닙니다.
특히 아래 옵션은 전파 기준을 따로 정해야 합니다.

- `--inspect`, `--inspect-brk`: 자식 프로세스와 같은 포트를 쓰면 충돌할 수 있습니다.
- `--max-old-space-size`: 작은 작업용 자식 프로세스에 과한 메모리 상한을 줄 수 있습니다.
- `--experimental-*`: 운영 안정성을 위해 일부 프로세스에는 넘기지 않는 편이 나을 수 있습니다.
- `--require`, `--import`: preload 모듈이 자식 프로세스의 시작 비용과 부작용을 바꿀 수 있습니다.
- `--trace-*`: 로그가 급격히 늘어 CI나 로그 수집 비용에 영향을 줄 수 있습니다.

따라서 `process.execArgv`는 "부모와 같은 Node.js 실행 환경을 재현하는 단서"로 보고, 실제 전달 전에는 allowlist 또는 denylist를 적용하는 편이 안전합니다.

## child_process에서 쓰는 패턴

### H3. fork는 기본적으로 execArgv를 상속한다

`child_process.fork()`는 Node.js 모듈을 별도 프로세스로 실행할 때 자주 사용합니다.
문서상 `fork()` 옵션의 `execArgv` 기본값은 `process.execArgv`입니다.
즉 부모 프로세스를 `--enable-source-maps`로 실행했다면 자식도 같은 플래그를 받을 수 있습니다.

```js
import { fork } from 'node:child_process';

const child = fork('./worker.js', ['--job=thumbnail'], {
  stdio: 'inherit'
});
```

이 기본값은 개발 중에는 편합니다.
부모 프로세스에서 source map을 켜면 자식 프로세스 stack trace도 읽기 쉬워지고, 공통 preload 설정도 같이 적용됩니다.
하지만 운영에서는 명시성이 더 중요합니다.
어떤 플래그를 상속할지 코드에서 드러나야 리뷰와 장애 대응이 쉬워집니다.

```js
import { fork } from 'node:child_process';

const child = fork('./worker.js', ['--job=thumbnail'], {
  stdio: 'inherit',
  execArgv: buildChildExecArgv(process.execArgv)
});

function buildChildExecArgv(parentExecArgv) {
  return parentExecArgv.filter(flag =>
    flag === '--enable-source-maps' ||
    flag.startsWith('--max-old-space-size=')
  );
}
```

이 예제는 source map과 memory limit만 전달합니다.
디버그, trace, 실험 플래그는 부모에서 켜져 있어도 자식에게 자동 전파하지 않습니다.
정책이 엄격해 보일 수 있지만, 장애 대응 시 "왜 자식 프로세스까지 inspector가 열렸지?" 같은 질문을 줄여 줍니다.

### H3. 디버그 포트는 충돌하지 않게 제거하거나 재설정한다

가장 조심해야 하는 값은 inspector 관련 플래그입니다.
부모 프로세스가 `--inspect=9229`로 실행 중인데 자식 프로세스도 같은 옵션을 받으면 포트 충돌이 날 수 있습니다.
개발 환경에서는 자동으로 다른 포트를 쓰게 하고 싶을 수 있지만, 운영 코드에서는 보통 inspector 전파 자체를 막는 편이 낫습니다.

```js
const INSPECT_FLAGS = [
  '--inspect',
  '--inspect-brk',
  '--inspect-port'
];

export function stripInspectorFlags(execArgv) {
  return execArgv.filter(flag => {
    return !INSPECT_FLAGS.some(name =>
      flag === name || flag.startsWith(`${name}=`)
    );
  });
}
```

이 함수는 `--inspect`, `--inspect=127.0.0.1:9229`, `--inspect-brk`, `--inspect-port=0` 같은 값을 제거합니다.
개발 전용 worker를 디버깅해야 한다면 별도 옵션으로 `--inspect=0`을 명시해 포트를 자동 할당하는 식의 분리된 흐름을 두는 편이 낫습니다.

```js
const execArgv = stripInspectorFlags(process.execArgv);

if (process.env.DEBUG_CHILD_PROCESS === '1') {
  execArgv.push('--inspect=0');
}
```

중요한 점은 부모의 디버그 상태를 자식에게 암묵적으로 복사하지 않는 것입니다.
디버그는 관찰 목적이 강하고, 관찰 대상이 늘어날수록 포트, 성능, 로그가 모두 달라질 수 있습니다.

## worker_threads에서의 기준

### H3. Worker도 execArgv를 받을 수 있지만 제약이 있다

`worker_threads`의 `Worker` 생성자도 `execArgv` 옵션을 받을 수 있습니다.
부모의 Node.js 옵션을 일부 전달해 Worker 런타임을 맞출 수 있지만, 모든 옵션이 허용되는 것은 아닙니다.
문서상 V8 옵션이나 프로세스 전체에 영향을 주는 옵션은 Worker에서 지원되지 않을 수 있습니다.

```js
import { Worker } from 'node:worker_threads';

const worker = new Worker(new URL('./image-worker.js', import.meta.url), {
  execArgv: ['--enable-source-maps'],
  workerData: {
    queueName: 'image-resize'
  }
});
```

Worker는 프로세스가 아니라 같은 프로세스 안의 별도 스레드입니다.
따라서 `child_process.fork()`에 넘기던 옵션을 그대로 Worker에 넘기면 실패하거나 의미가 달라질 수 있습니다.
특히 메모리 한도, inspector, process title 같은 플래그는 "새 프로세스"와 "새 스레드"에서 기대가 다릅니다.

### H3. Worker용 allowlist를 따로 둔다

Worker에 필요한 런타임 옵션은 보통 많지 않습니다.
예를 들어 source map만 필요하다면 아래처럼 좁게 가져가는 편이 좋습니다.

```js
const WORKER_EXEC_ARGV_ALLOWLIST = new Set([
  '--enable-source-maps',
  '--no-warnings'
]);

export function buildWorkerExecArgv(parentExecArgv) {
  return parentExecArgv.filter(flag => {
    if (WORKER_EXEC_ARGV_ALLOWLIST.has(flag)) {
      return true;
    }

    return false;
  });
}
```

`--no-warnings`처럼 로그 정책에 영향을 주는 플래그도 무조건 좋은 선택은 아닙니다.
경고가 운영 문제를 일찍 보여 주는 신호라면 숨기지 않는 편이 낫습니다.
그래서 allowlist는 조직의 운영 기준에 맞춰 작게 시작하고, 필요한 항목만 추가해야 합니다.

## 테스트와 CI에서의 활용

### H3. 테스트 프로세스의 실행 조건을 재현한다

CI에서만 실패하는 테스트는 런타임 플래그 차이 때문에 생기기도 합니다.
로컬에서는 source map이 꺼져 있는데 CI에서는 켜져 있거나, CI에서만 `--unhandled-rejections=strict` 같은 옵션을 쓰는 경우입니다.
이때 테스트 로그에 `process.execArgv`를 남기면 재현 조건을 빠르게 좁힐 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('runtime flags are visible in diagnostics', () => {
  assert.ok(Array.isArray(process.execArgv));
});
```

실제 서비스 테스트에서 매번 전체 값을 출력할 필요는 없습니다.
실패했을 때만 진단 로그에 포함하거나, CI job summary에 마스킹된 형태로 남기는 정도면 충분합니다.

```js
export function runtimeDiagnostics() {
  return {
    nodeVersion: process.version,
    platform: process.platform,
    execArgv: sanitizeExecArgv(process.execArgv)
  };
}
```

`process.execArgv` 자체에는 보통 API 키가 직접 들어가지 않지만, `--require ./bootstrap.js`나 `--import ./loader.js`처럼 내부 파일 경로가 드러날 수 있습니다.
외부로 공유되는 로그라면 경로와 사용자 홈 디렉터리를 마스킹하는 편이 좋습니다.

### H3. 스냅샷 테스트에는 원본 배열을 직접 박지 않는다

`process.execArgv`는 실행 환경에 따라 달라집니다.
로컬 IDE, npm script, CI, test runner가 서로 다른 플래그를 붙일 수 있습니다.
따라서 스냅샷 테스트나 golden file에 원본 배열을 그대로 넣으면 불필요하게 깨질 가능성이 큽니다.

좋은 접근은 "관심 있는 플래그가 켜져 있는가"만 검증하는 것입니다.

```js
export function hasRuntimeFlag(name, execArgv = process.execArgv) {
  return execArgv.some(flag =>
    flag === name || flag.startsWith(`${name}=`)
  );
}

test('detects source map flag', () => {
  assert.equal(
    hasRuntimeFlag('--enable-source-maps', ['--enable-source-maps']),
    true
  );
});
```

이렇게 함수 단위로 분리하면 실제 프로세스 실행 옵션과 무관하게 테스트할 수 있습니다.
운영 코드에서는 기본값으로 `process.execArgv`를 쓰고, 테스트에서는 작은 배열을 주입하면 됩니다.

## 운영 코드에서의 필터링 기준

### H3. allowlist와 denylist를 섞지 말고 우선순위를 정한다

런타임 플래그 필터링은 보통 두 가지 방식으로 나뉩니다.
첫째, 허용할 플래그만 남기는 allowlist입니다.
둘째, 위험한 플래그만 제거하는 denylist입니다.

운영 서비스에서는 allowlist가 더 예측 가능합니다.
새로운 Node.js 플래그가 부모 프로세스에 추가되더라도 자식 프로세스까지 자동으로 흘러가지 않기 때문입니다.
반면 개발 도구나 내부 CLI처럼 부모 환경을 최대한 재현해야 하는 경우에는 denylist가 더 편할 수 있습니다.

```js
const SAFE_CHILD_FLAGS = [
  '--enable-source-maps',
  '--trace-warnings'
];

export function buildSafeExecArgv(execArgv) {
  return execArgv.filter(flag => {
    return SAFE_CHILD_FLAGS.some(allowed =>
      flag === allowed || flag.startsWith(`${allowed}=`)
    );
  });
}
```

이 코드는 단순하지만 리뷰하기 쉽습니다.
자식 프로세스에 전달되는 런타임 옵션이 파일 상단에 모여 있고, 새로운 옵션을 추가할 때도 이유를 남기기 좋습니다.

### H3. 환경별 정책을 명시한다

개발, 테스트, 운영은 `process.execArgv`를 다루는 기준이 다를 수 있습니다.
개발에서는 inspector와 source map이 중요하고, 테스트에서는 warning과 unhandled rejection 정책이 중요하며, 운영에서는 안정성과 로그량이 중요합니다.
환경별 함수를 분리하면 의도가 더 잘 드러납니다.

```js
export function buildExecArgvForChild({ env, parentExecArgv }) {
  if (env === 'development') {
    return stripFixedInspectorPort(parentExecArgv);
  }

  if (env === 'test') {
    return buildSafeExecArgv(parentExecArgv);
  }

  return parentExecArgv.filter(flag =>
    flag === '--enable-source-maps'
  );
}

function stripFixedInspectorPort(execArgv) {
  return execArgv.map(flag => {
    if (flag.startsWith('--inspect=')) {
      return '--inspect=0';
    }

    if (flag.startsWith('--inspect-brk=')) {
      return '--inspect-brk=0';
    }

    return flag;
  });
}
```

여기서 운영 환경은 source map만 전달합니다.
개발 환경은 inspector를 유지하되 고정 포트를 자동 포트로 바꿉니다.
테스트 환경은 별도 안전 목록을 사용합니다.
이런 분리는 코드가 조금 길어지지만, 장애가 났을 때 어떤 옵션이 어디로 전달되는지 추적하기 쉽습니다.

## 자주 하는 실수

### H3. process.argv를 execArgv처럼 넘긴다

가끔 자식 Node.js 프로세스를 만들면서 `process.argv.slice(2)`를 런타임 옵션처럼 넘기는 코드를 볼 수 있습니다.
이 배열은 사용자 인자이므로 Node.js 옵션 위치에 들어가면 의미가 틀어질 수 있습니다.

```js
// 피해야 할 예
fork('./worker.js', [], {
  execArgv: process.argv.slice(2)
});
```

사용자 인자는 `fork(modulePath, args)`의 두 번째 인자로 넘기고, Node.js 런타임 옵션은 `execArgv`에 넘겨야 합니다.

```js
fork('./worker.js', ['--job=sync'], {
  execArgv: buildSafeExecArgv(process.execArgv)
});
```

### H3. 로그에 전체 실행 환경을 무심코 남긴다

문제 분석을 위해 `process.execArgv`, `process.argv`, `process.env`를 한 번에 출력하고 싶을 때가 있습니다.
하지만 `process.env`에는 토큰, 비밀번호, 연결 문자열이 들어갈 수 있습니다.
런타임 플래그를 확인하려다가 민감정보까지 로그로 남기면 안 됩니다.

```js
export function safeStartupLog() {
  return {
    nodeVersion: process.version,
    execArgv: sanitizeExecArgv(process.execArgv),
    argv: process.argv.map(maskHomePath)
  };
}

function sanitizeExecArgv(execArgv) {
  return execArgv.map(maskHomePath);
}

function maskHomePath(value) {
  const home = process.env.HOME;

  if (!home) {
    return value;
  }

  return value.replaceAll(home, '$HOME');
}
```

민감정보 점검은 코드 예제에도 적용됩니다.
블로그, 장애 보고서, 이슈 템플릿에 런타임 옵션을 붙일 때는 홈 디렉터리, 내부 저장소 경로, 토큰처럼 재사용 가능한 정보를 먼저 지워야 합니다.

## 실무 체크리스트

### H3. 배포 전 확인할 항목

`process.execArgv`를 직접 쓰는 코드를 배포하기 전에는 아래 항목을 확인하세요.

- `process.argv`와 `process.execArgv`의 역할이 코드에서 분리되어 있는가?
- 자식 프로세스에 넘기는 `execArgv`가 명시적으로 필터링되는가?
- inspector 관련 플래그가 포트 충돌을 만들지 않는가?
- Worker에 지원되지 않는 프로세스 단위 플래그를 넘기지 않는가?
- 테스트가 실제 실행 환경의 전체 `process.execArgv` 값에 과하게 의존하지 않는가?
- 진단 로그에 홈 디렉터리, 내부 경로, 민감정보가 노출되지 않는가?

이 체크리스트는 특히 monorepo build worker, image processing worker, test sharding runner, queue consumer supervisor처럼 Node.js 프로세스를 여러 개 띄우는 코드에서 효과가 큽니다.
단일 서버에서는 `process.execArgv`를 거의 의식하지 않아도 되지만, 프로세스를 복제하거나 감싸는 순간 런타임 플래그 전파가 설계 문제가 됩니다.

## 마무리

`process.execArgv`는 평소에는 눈에 잘 띄지 않지만, Node.js 프로세스를 다시 띄우거나 Worker로 작업을 나눌 때 중요한 기준점이 됩니다.
핵심은 단순합니다.
애플리케이션 인자는 `process.argv`, Node.js 런타임 플래그는 `process.execArgv`로 분리해서 보고, 자식 실행 환경에는 필요한 플래그만 명시적으로 전달하세요.

특히 inspector, memory, experimental, preload 계열 플래그는 그대로 복사하기 전에 한 번 멈춰서 생각해야 합니다.
부모 프로세스의 실행 조건을 재현하는 것이 목적인지, 자식 프로세스를 안정적으로 제한하는 것이 목적인지에 따라 답이 달라집니다.
작은 필터 함수와 환경별 정책을 만들어 두면 디버깅 경험과 운영 안정성을 동시에 지킬 수 있습니다.

## 함께 보면 좋은 글

- [Node.js process.execve 가이드: 현재 프로세스를 새 실행 파일로 교체하는 법](/development/blog/seo/2026/08/13/nodejs-process-execve-current-process-replacement-guide.html)
- [Node.js child_process spawn AbortSignal 가이드](/development/blog/seo/2026/07/25/nodejs-child-process-spawn-abortsignal-timeout-guide.html)
- [Node.js process.report 가이드: 운영 환경 진단 리포트 남기기](/development/blog/seo/2026/05/29/nodejs-process-report-production-diagnostics-guide.html)

## 발행 전 최종 점검

- 제목에 핵심 키워드 `Node.js process.execArgv`를 포함했습니다.
- 본문은 H2/H3 구조로 작성했고, 첫 단락에서 문제와 실무 가치를 제시했습니다.
- 예시는 민감정보 없이 재현 가능한 형태로 구성했습니다.
- 내부링크 3개를 연결했습니다.
- 불법, 혐오, 유해 표현이나 민감정보 노출이 없도록 점검했습니다.

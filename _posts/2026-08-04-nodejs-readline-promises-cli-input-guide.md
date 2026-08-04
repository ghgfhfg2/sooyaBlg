---
layout: post
title: "Node.js readline/promises 가이드: CLI 입력을 async/await로 안전하게 받는 법"
date: 2026-08-04 20:00:00 +0900
lang: ko
translation_key: nodejs-readline-promises-cli-input-guide
permalink: /development/blog/seo/2026/08/04/nodejs-readline-promises-cli-input-guide.html
alternates:
  ko: /development/blog/seo/2026/08/04/nodejs-readline-promises-cli-input-guide.html
  x_default: /development/blog/seo/2026/08/04/nodejs-readline-promises-cli-input-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, readline, cli, promises, async-await, stdin, terminal, javascript]
description: "Node.js readline/promises로 CLI 입력을 async/await 방식으로 받는 방법을 정리합니다. question, AbortSignal, 입력 검증, close 정리, 테스트 가능한 구조까지 실무 예제로 설명합니다."
---

CLI 도구는 옵션만으로 끝나지 않을 때가 많습니다.
토큰을 붙여 넣게 하거나, 배포 환경을 선택하게 하거나, 위험한 작업 전에 확인 문구를 입력받아야 합니다.
이런 흐름을 콜백 기반 `readline`으로 작성하면 분기와 정리 코드가 쉽게 흩어집니다.

Node.js의 `node:readline/promises`를 사용하면 터미널 입력을 `async/await` 흐름 안에서 다룰 수 있습니다.
질문을 순서대로 묻고, 입력을 검증하고, 타임아웃이나 취소 신호를 연결하는 구조가 훨씬 단순해집니다.
특히 배포 스크립트, 마이그레이션 도구, 내부 운영 CLI처럼 실패하면 영향이 큰 명령에서 유용합니다.

이 글에서는 Node.js `readline/promises`를 실무 CLI에 적용하는 기준을 정리합니다.
명령줄 옵션 파싱이 먼저 필요하다면 [Node.js util.parseArgs CLI 옵션 가이드](/development/blog/seo/2026/06/08/nodejs-util-parseargs-cli-options-guide.html)를 함께 보면 좋습니다.
터미널 출력 색상은 [Node.js util.styleText CLI 색상 출력 가이드](/development/blog/seo/2026/06/10/nodejs-util-styletext-cli-color-output-guide.html)와 연결해 정리할 수 있습니다.
로그 파일을 줄 단위로 읽는 작업은 [Node.js filehandle.readLines 대용량 로그 처리 가이드](/development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html)를 참고하세요.

## readline/promises가 필요한 상황

### 질문과 답변 흐름을 await로 정리한다

가장 기본적인 사용법은 `createInterface()`로 입출력 인터페이스를 만들고 `question()`을 기다리는 것입니다.
콜백을 중첩하지 않아도 되기 때문에 입력 순서가 코드 순서와 같아집니다.

```js
import { createInterface } from 'node:readline/promises';
import { stdin as input, stdout as output } from 'node:process';

export async function askProjectName() {
  const rl = createInterface({ input, output });

  try {
    const name = await rl.question('Project name: ');
    return name.trim();
  } finally {
    rl.close();
  }
}
```

`finally`에서 `rl.close()`를 호출하는 것이 중요합니다.
입력 인터페이스가 열린 채로 남으면 프로세스가 종료되지 않거나 테스트가 멈춘 것처럼 보일 수 있습니다.
CLI에서 "질문을 하나만 묻는다"는 작은 코드라도 정리 경로를 습관처럼 넣는 편이 안전합니다.

### 여러 질문을 순서대로 묻는다

실무 CLI는 한 번에 여러 값을 받는 경우가 많습니다.
예를 들어 배포 대상, 버전 이름, 최종 확인 여부를 차례로 물어야 할 수 있습니다.
`readline/promises`를 쓰면 이런 흐름을 일반 비동기 함수처럼 작성할 수 있습니다.

```js
import { createInterface } from 'node:readline/promises';
import { stdin as input, stdout as output } from 'node:process';

export async function collectDeployInput() {
  const rl = createInterface({ input, output });

  try {
    const environment = await rl.question('Environment (staging/production): ');
    const version = await rl.question('Version label: ');
    const confirmation = await rl.question('Type deploy to continue: ');

    return {
      environment: environment.trim(),
      version: version.trim(),
      confirmed: confirmation.trim() === 'deploy'
    };
  } finally {
    rl.close();
  }
}
```

입력 수집 함수는 실행 함수와 분리하는 편이 좋습니다.
값을 묻는 책임과 실제 배포를 수행하는 책임을 나누면 테스트와 재사용이 쉬워집니다.
또한 사용자가 입력한 문자열을 바로 실행 명령에 넣지 않고, 검증 단계를 거쳐 명확한 구조로 바꿀 수 있습니다.

## 입력 검증과 기본값 설계

### 허용 값은 명시적으로 검사한다

CLI 입력은 항상 문자열입니다.
사용자가 대소문자를 섞거나, 공백을 붙이거나, 예상하지 못한 값을 넣을 수 있습니다.
운영 작업에서는 "비슷한 값"을 추측해서 받아들이기보다 허용 목록을 좁게 두는 편이 낫습니다.

```js
const allowedEnvironments = new Set(['staging', 'production']);

export function parseEnvironment(value) {
  const environment = value.trim().toLowerCase();

  if (!allowedEnvironments.has(environment)) {
    throw new Error('Environment must be staging or production.');
  }

  return environment;
}
```

검증 함수는 `readline`과 분리해 두면 단위 테스트가 간단해집니다.
터미널을 띄우지 않고도 입력 문자열과 결과만 확인할 수 있기 때문입니다.
나중에 같은 검증을 `parseArgs()` 옵션 입력에도 재사용할 수 있습니다.

### 빈 입력의 기본값을 분명히 정한다

빈 입력을 허용할지 거부할지도 요구사항입니다.
예를 들어 배포 환경은 기본값을 줄 수 있지만, 삭제 확인 문구는 빈 입력을 허용하면 안 됩니다.
기본값이 있는 질문은 프롬프트에 표시하고, 반환 값도 한 곳에서 정리하는 편이 좋습니다.

```js
export function withDefault(value, fallback) {
  const trimmed = value.trim();
  return trimmed.length === 0 ? fallback : trimmed;
}
```

```js
const rawEnvironment = await rl.question('Environment [staging]: ');
const environment = parseEnvironment(withDefault(rawEnvironment, 'staging'));
```

기본값은 편리하지만 위험한 작업에는 조심해야 합니다.
특히 `production`, `delete`, `overwrite`처럼 되돌리기 어려운 동작은 기본값으로 선택되게 만들지 않는 편이 안전합니다.
사용자가 명시적으로 확인했다는 흔적을 남기는 것이 운영 사고를 줄입니다.

## 취소와 타임아웃 처리

### AbortSignal로 오래 걸리는 질문을 중단한다

자동화 환경에서 CLI가 입력을 기다리다 멈추면 배치나 CI가 계속 점유될 수 있습니다.
`question()`은 옵션으로 `signal`을 받을 수 있으므로, 일정 시간이 지나면 대기를 취소하게 만들 수 있습니다.

```js
import { createInterface } from 'node:readline/promises';
import { stdin as input, stdout as output } from 'node:process';

export async function askWithTimeout(prompt, timeoutMs = 30_000) {
  const rl = createInterface({ input, output });

  try {
    return await rl.question(prompt, {
      signal: AbortSignal.timeout(timeoutMs)
    });
  } finally {
    rl.close();
  }
}
```

타임아웃은 사용자 경험과 자동화 안정성 사이의 균형입니다.
사람이 직접 실행하는 도구라면 너무 짧은 제한이 불편할 수 있습니다.
반대로 CI나 cron에서 실행되는 명령이라면 무한 대기를 피하기 위해 명확한 제한이 필요합니다.

### 취소 오류를 사용자 메시지로 바꾼다

취소가 발생하면 내부 오류 객체를 그대로 보여 주기보다 CLI 사용자가 이해할 수 있는 메시지로 바꾸는 편이 좋습니다.
다만 오류를 삼켜 성공처럼 끝내면 안 됩니다.
명령은 실패 코드로 종료하되, 원인을 짧게 알려야 합니다.

```js
export async function main() {
  try {
    const answer = await askWithTimeout('Type continue: ', 10_000);

    if (answer.trim() !== 'continue') {
      throw new Error('Confirmation text did not match.');
    }
  } catch (error) {
    if (error?.name === 'AbortError') {
      console.error('Input timed out. Run the command again when ready.');
      process.exitCode = 1;
      return;
    }

    throw error;
  }
}
```

이 패턴은 실패를 숨기지 않으면서도 사용자가 다음 행동을 알 수 있게 합니다.
운영 CLI에서는 실패 메시지가 길 필요는 없지만, 재시도 가능 여부와 필요한 입력은 분명해야 합니다.

## 테스트 가능한 구조로 나누기

### 입출력 의존성을 주입한다

`stdin`과 `stdout`을 함수 내부에서 직접 고정하면 테스트가 어려워집니다.
대신 입력과 출력을 인자로 받을 수 있게 만들면, 테스트에서는 가짜 스트림을 넘길 수 있습니다.

```js
import { createInterface } from 'node:readline/promises';

export async function askDeploymentTarget({ input, output }) {
  const rl = createInterface({ input, output });

  try {
    const rawEnvironment = await rl.question('Environment [staging]: ');
    return parseEnvironment(withDefault(rawEnvironment, 'staging'));
  } finally {
    rl.close();
  }
}
```

실행 파일에서는 실제 `process.stdin`과 `process.stdout`을 넘기면 됩니다.
테스트에서는 `PassThrough` 같은 스트림을 이용해 입력을 흘려보내고 결과를 검증할 수 있습니다.
이렇게 하면 CLI 프롬프트 로직도 일반 비동기 함수처럼 테스트할 수 있습니다.

### 핵심 로직은 입력 수집 밖에 둔다

터미널 질문 함수 안에서 파일 삭제, 배포, 마이그레이션을 바로 실행하면 테스트와 검토가 어려워집니다.
입력 수집은 명령 객체를 만들고, 실제 실행은 별도 함수가 담당하게 나누는 편이 좋습니다.

```js
export async function buildDeployCommand(io) {
  const environment = await askDeploymentTarget(io);
  const confirmation = await io.ask('Type deploy to continue: ');

  return {
    environment,
    dryRun: confirmation.trim() !== 'deploy'
  };
}
```

이 예제처럼 질문 계층을 얇게 유지하면 나중에 웹 UI, 설정 파일, 명령줄 옵션에서 같은 실행 로직을 호출하기 쉬워집니다.
CLI는 입력 방법일 뿐이고, 비즈니스 동작의 유일한 출입구가 되면 안 됩니다.

## 운영 CLI 체크리스트

### 안전한 질문 문구를 사용한다

프롬프트는 짧아야 하지만 모호하면 안 됩니다.
위험한 작업에서는 대상 이름과 결과를 함께 보여 주는 편이 좋습니다.
예를 들어 `Continue?`보다 `Deploy production version v42? Type deploy:`가 더 안전합니다.

입력 확인 문구는 `y` 하나보다 명시적인 단어가 낫습니다.
특히 삭제, 덮어쓰기, 프로덕션 배포처럼 영향이 큰 작업은 사용자가 의도적으로 입력했다는 신호가 필요합니다.

### 비대화형 실행 경로를 제공한다

CLI가 항상 질문만 하면 자동화가 어렵습니다.
cron, CI, 배치에서는 `--yes`, `--environment`, `--config` 같은 비대화형 옵션이 필요할 수 있습니다.
대화형 질문은 사람이 직접 실행할 때의 보조 장치로 두고, 자동화 경로는 명확한 옵션으로 열어 두는 편이 좋습니다.

이때 옵션 파싱과 질문 입력의 검증 규칙은 같은 함수를 공유하세요.
같은 값인데 입력 경로에 따라 허용 기준이 달라지면 운영에서 혼란이 생깁니다.

## 자주 묻는 질문

### readline/promises는 콜백 readline을 완전히 대체하나요?

새 코드라면 대부분 `readline/promises`를 먼저 고려해도 됩니다.
다만 기존 콜백 기반 코드가 안정적으로 동작하고 있고, 이벤트 단위로 세밀하게 줄 입력을 처리해야 한다면 그대로 유지할 수도 있습니다.
대화형 질문이 중심인 CLI라면 Promise API가 더 읽기 쉽습니다.

### 비밀번호나 토큰 입력에도 그대로 쓰면 되나요?

민감한 값을 받을 때는 화면에 입력값이 그대로 보이는 문제가 있습니다.
`readline` 기본 프롬프트는 비밀 입력 마스킹 도구가 아니므로, 토큰이나 비밀번호를 받을 때는 전용 패키지나 터미널 echo 제어가 가능한 방식을 검토해야 합니다.
또한 입력값을 로그에 남기지 않는 규칙을 반드시 적용해야 합니다.

### 질문마다 createInterface를 새로 만들어야 하나요?

여러 질문을 한 흐름에서 묻는다면 하나의 인터페이스를 만들고 마지막에 닫는 편이 자연스럽습니다.
반대로 독립적인 헬퍼 함수가 질문 하나만 처리한다면 함수 내부에서 만들고 닫아도 됩니다.
중요한 기준은 `close()` 책임이 코드에서 분명히 보이는가입니다.

## 마무리

`node:readline/promises`는 Node.js CLI 입력을 `async/await` 흐름으로 정리해 주는 실용적인 API입니다.
질문 순서가 코드에 그대로 드러나고, 검증 함수와 실행 로직을 분리하기 쉬우며, `AbortSignal`을 통해 무한 대기 위험도 줄일 수 있습니다.

실무에서는 다음 기준을 지키면 좋습니다.

- `createInterface()`를 만들었다면 `finally`에서 닫는다
- 빈 입력, 기본값, 허용 목록을 명확히 검증한다
- 위험한 작업은 짧은 yes보다 명시적인 확인 문구를 요구한다
- 자동화 환경을 위해 비대화형 옵션 경로를 함께 제공한다
- 민감한 입력은 로그와 화면 노출 정책을 별도로 점검한다

CLI 도구가 작을 때는 입력 처리를 가볍게 넘기기 쉽습니다.
하지만 운영 명령일수록 질문, 검증, 취소, 정리 경로가 명확해야 합니다.
`readline/promises`를 기본 구조로 삼으면 작은 스크립트도 테스트 가능한 도구로 성장시키기 쉽습니다.

## 함께 보면 좋은 글

- [Node.js util.parseArgs CLI 옵션 가이드](/development/blog/seo/2026/06/08/nodejs-util-parseargs-cli-options-guide.html)
- [Node.js util.styleText CLI 색상 출력 가이드](/development/blog/seo/2026/06/10/nodejs-util-styletext-cli-color-output-guide.html)
- [Node.js filehandle.readLines 대용량 로그 처리 가이드](/development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html)

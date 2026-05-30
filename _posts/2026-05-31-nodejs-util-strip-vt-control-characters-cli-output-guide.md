---
layout: post
title: "Node.js stripVTControlCharacters 가이드: CLI 출력에서 ANSI 제어 문자를 안전하게 제거하는 법"
date: 2026-05-31 08:00:00 +0900
lang: ko
translation_key: nodejs-util-strip-vt-control-characters-cli-output-guide
permalink: /development/blog/seo/2026/05/31/nodejs-util-strip-vt-control-characters-cli-output-guide.html
alternates:
  ko: /development/blog/seo/2026/05/31/nodejs-util-strip-vt-control-characters-cli-output-guide.html
  x_default: /development/blog/seo/2026/05/31/nodejs-util-strip-vt-control-characters-cli-output-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, util, cli, ansi, logging, testing]
description: "Node.js util.stripVTControlCharacters로 CLI 출력과 로그에서 ANSI 제어 문자를 제거하는 방법을 정리했습니다. 테스트 스냅샷, 파일 로그, CI 출력 정리 기준과 주의사항을 예제로 설명합니다."
---

CLI 도구를 만들다 보면 색상, 굵게 표시, 커서 이동 같은 터미널 전용 표현을 자주 붙입니다.
사람이 터미널에서 볼 때는 이런 장식이 가독성을 높여 주지만, 같은 문자열을 로그 파일, 테스트 스냅샷, CI 아티팩트에 남기면 문제가 되기도 합니다.

예를 들어 실패 메시지에 색상 코드가 섞이면 텍스트 검색이 어려워지고, 테스트에서는 눈에 보이지 않는 제어 문자가 snapshot 차이를 만들 수 있습니다.
이럴 때 Node.js의 `util.stripVTControlCharacters()`를 사용하면 문자열에서 ANSI escape sequence 같은 VT 제어 문자를 제거할 수 있습니다.

이 글에서는 `stripVTControlCharacters()` 기본 사용법, CLI 출력과 파일 로그를 분리하는 패턴, 테스트에서 안정적으로 비교하는 방법, 그리고 색상 제거만으로 해결되지 않는 주의점을 정리합니다.
CLI 인자 구조까지 함께 다듬고 있다면 [Node.js util.parseArgs 가이드](/development/blog/seo/2026/04/28/nodejs-util-parseargs-cli-argument-parsing-guide.html)를 먼저 연결해 보면 좋습니다.

## stripVTControlCharacters가 필요한 순간

### H3. 터미널용 문자열과 저장용 문자열은 목적이 다르다

터미널 출력은 사용자가 지금 화면에서 읽는 것이 목적입니다.
성공은 초록색, 실패는 빨간색, 경고는 노란색으로 보여 주면 상태를 빠르게 구분할 수 있습니다.

하지만 저장용 문자열은 다릅니다.
로그 파일, 테스트 결과, 검색 인덱스, GitHub Actions 아티팩트는 장식보다 일관성이 중요합니다.
아래 같은 문제가 반복된다면 출력 정리 레이어를 두는 편이 좋습니다.

- 로그 검색에서 `ERROR`가 눈에 보이는데 실제 문자열 매칭이 실패한다.
- 테스트 스냅샷에 `\u001b[31m` 같은 escape 문자가 섞인다.
- CI 실패 메시지를 복사했을 때 보이지 않는 문자가 함께 붙는다.
- 터미널에서는 예쁘지만 파일로 열면 이상한 기호가 보인다.

핵심은 색상을 쓰지 말자는 것이 아닙니다.
**사람에게 보여 줄 출력과 기계가 읽을 출력을 분리하자**는 쪽에 가깝습니다.

### H3. 정규식 직접 구현보다 표준 유틸을 먼저 보는 편이 안전하다

ANSI 제어 문자는 단순히 `\x1b[`로 시작하는 문자열만 있는 것이 아닙니다.
직접 정규식을 작성해도 작은 예제에서는 동작하지만, 다양한 터미널 제어 시퀀스를 모두 다루려면 유지보수 부담이 생깁니다.

Node.js가 제공하는 `stripVTControlCharacters()`는 이런 문자열 정리 작업을 표준 라이브러리 쪽으로 옮겨 줍니다.
작은 내부 CLI나 배포 스크립트라면 외부 패키지를 추가하지 않고도 충분히 실용적인 기본값을 만들 수 있습니다.

## 기본 사용법

### H3. util 모듈에서 가져와 문자열을 정리한다

사용법은 단순합니다.
`node:util`에서 함수를 가져오고, 정리할 문자열을 넘기면 됩니다.

```js
import { stripVTControlCharacters } from 'node:util';

const colored = '\u001b[32mPASS\u001b[39m build completed';
const plain = stripVTControlCharacters(colored);

console.log(plain);
// PASS build completed
```

반환값은 제어 문자가 제거된 새 문자열입니다.
원본 문자열을 바꾸지 않으므로 터미널 출력용 값과 저장용 값을 따로 다루기 쉽습니다.

```js
import { appendFile } from 'node:fs/promises';
import { stripVTControlCharacters } from 'node:util';

function formatStatus(status) {
  if (status === 'pass') {
    return '\u001b[32mPASS\u001b[39m';
  }

  return '\u001b[31mFAIL\u001b[39m';
}

const terminalMessage = `${formatStatus('pass')} deployment check`;
const logMessage = stripVTControlCharacters(terminalMessage);

process.stdout.write(`${terminalMessage}\n`);
await appendFile('deploy.log', `${logMessage}\n`);
```

이 구조에서는 터미널 경험을 유지하면서도 파일 로그에는 깨끗한 텍스트만 남길 수 있습니다.

### H3. 출력 경로에 따라 색상 사용 여부를 나눈다

더 깔끔한 방식은 처음부터 출력 경로를 구분하는 것입니다.
터미널에는 색상을 허용하고, 파일이나 JSON에는 정리된 문자열만 보내는 헬퍼를 둘 수 있습니다.

```js
import { stripVTControlCharacters } from 'node:util';

function toPlainText(value) {
  return stripVTControlCharacters(String(value));
}

function writeHumanOutput(message) {
  process.stdout.write(`${message}\n`);
}

function writeMachineOutput(message) {
  process.stdout.write(`${toPlainText(message)}\n`);
}
```

실무에서는 `process.stdout.isTTY`를 함께 참고해 터미널 여부를 판단할 수 있습니다.
다만 `isTTY`만으로 모든 요구를 해결하려 하기보다, `--color`, `--no-color`, `--json` 같은 명시적 옵션을 두면 자동화에서 더 예측하기 쉽습니다.
이런 옵션 계약은 [Node.js util.parseArgs 가이드](/development/blog/seo/2026/04/28/nodejs-util-parseargs-cli-argument-parsing-guide.html)의 패턴과 잘 맞습니다.

## 테스트에서 활용하는 패턴

### H3. 색상 때문에 깨지는 스냅샷을 줄인다

CLI 테스트는 실제 출력 문자열을 비교하는 경우가 많습니다.
이때 색상 코드가 섞이면 운영체제, 터미널 감지, CI 환경에 따라 결과가 달라질 수 있습니다.

테스트에서는 비교 전에 제어 문자를 제거하면 의도한 문구만 검증할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { stripVTControlCharacters } from 'node:util';

function renderResult(name, passed) {
  const status = passed ? '\u001b[32mPASS\u001b[39m' : '\u001b[31mFAIL\u001b[39m';
  return `${status} ${name}`;
}

test('renders result text', () => {
  const output = renderResult('health check', true);

  assert.equal(
    stripVTControlCharacters(output),
    'PASS health check'
  );
});
```

이 테스트는 색상 표현 자체가 아니라, 사용자에게 전달해야 할 핵심 메시지를 검증합니다.
색상 정책을 별도로 테스트하고 싶다면 렌더링 함수의 옵션을 분리해 `color: true`일 때 escape sequence가 포함되는지만 좁게 확인하면 됩니다.

### H3. stderr와 stdout 비교를 단순하게 만든다

CLI 도구는 정상 결과를 `stdout`, 오류와 진단 메시지를 `stderr`로 나누는 경우가 많습니다.
테스트에서 두 스트림을 캡처한 뒤 문자열 비교를 할 때도 같은 정리 함수를 적용할 수 있습니다.

```js
import { stripVTControlCharacters } from 'node:util';

function normalizeCliOutput(output) {
  return stripVTControlCharacters(output)
    .replace(/\r\n/g, '\n')
    .trimEnd();
}
```

여기서 줄바꿈 정규화와 제어 문자 제거를 한곳에 모아 두면 테스트 파일마다 작은 정리 코드가 흩어지지 않습니다.
내장 테스트 러너와 조합한다면 [Node.js test runner 가이드](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)처럼 기본 검증 구조를 먼저 정리해 두는 편이 좋습니다.

## 로그와 CI에서 주의할 점

### H3. 민감정보 마스킹을 대신하지 않는다

`stripVTControlCharacters()`는 제어 문자를 제거하는 함수입니다.
API 키, 토큰, 이메일, 사용자 ID 같은 민감정보를 마스킹해 주지는 않습니다.

따라서 로그 저장 전에는 보통 두 단계를 분리합니다.

```js
import { stripVTControlCharacters } from 'node:util';

function redactSecrets(message) {
  return message.replace(/token=[^\s]+/g, 'token=[REDACTED]');
}

function sanitizeLogLine(message) {
  const plain = stripVTControlCharacters(message);
  return redactSecrets(plain);
}
```

순서는 팀 규칙에 맞게 정할 수 있지만, 중요한 점은 역할을 섞지 않는 것입니다.
제어 문자 제거, 민감정보 마스킹, 줄바꿈 정규화, JSON 직렬화는 각각 다른 문제입니다.
로그 예제를 공개 글이나 문서에 넣는다면 [로그 예제 민감정보 정리 가이드](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)도 함께 확인하세요.

### H3. JSON 출력은 처음부터 구조화하는 편이 낫다

자동화 도구가 읽을 출력이라면 색상이 있는 문자열을 만든 뒤 나중에 제거하는 방식보다, 처음부터 JSON을 별도 경로로 내보내는 편이 더 안정적입니다.

```js
function renderJsonResult(result) {
  return JSON.stringify({
    status: result.ok ? 'pass' : 'fail',
    name: result.name,
    durationMs: result.durationMs
  });
}
```

이렇게 하면 사람이 읽는 메시지와 자동화가 읽는 데이터 계약이 분리됩니다.
`stripVTControlCharacters()`는 사람이 읽는 출력에서 부득이하게 장식을 걷어 내야 할 때의 보조 도구로 두고, 기계용 출력은 구조화된 형식을 우선하는 편이 좋습니다.

## 실무 체크리스트

### H3. CLI 출력 정리 기준을 명시한다

작은 CLI라도 출력 정책을 정해 두면 나중에 테스트와 운영 로그가 덜 흔들립니다.
아래 항목을 기준으로 점검해 보세요.

- 터미널 출력과 파일 로그 출력이 같은 문자열을 공유해야 하는가?
- `--color`, `--no-color`, `--json` 같은 옵션이 필요한가?
- 테스트 비교 전에 줄바꿈과 VT 제어 문자를 정리하는 공통 함수가 있는가?
- CI 아티팩트에는 색상 코드가 제거된 결과를 남기는가?
- 민감정보 마스킹을 별도 단계로 처리하고 있는가?

이 기준이 잡히면 CLI의 사용자 경험과 자동화 안정성을 동시에 챙기기 쉬워집니다.

### H3. 내부 링크로 이어서 볼 글

- [Node.js util.parseArgs 가이드: CLI 인자를 표준 라이브러리로 깔끔하게 처리하는 법](/development/blog/seo/2026/04/28/nodejs-util-parseargs-cli-argument-parsing-guide.html)
- [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)
- [로그 예제 민감정보 정리 가이드](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)

## 마무리

`util.stripVTControlCharacters()`는 크고 화려한 API는 아니지만, CLI 도구를 운영 가능한 형태로 다듬을 때 꽤 유용합니다.
색상과 장식은 터미널에서 사람을 돕고, 정리된 텍스트는 로그와 테스트에서 자동화를 돕습니다.

중요한 건 모든 출력에서 색상을 없애는 것이 아니라, 출력의 목적을 분리하는 것입니다.
터미널에는 읽기 좋은 메시지를 주고, 파일·CI·테스트에는 비교와 검색이 쉬운 문자열을 남기면 작은 CLI도 훨씬 안정적으로 굴러갑니다.

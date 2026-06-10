---
layout: post
title: "Node.js util.styleText 가이드: CLI 색상 출력을 안전하게 적용하는 법"
date: 2026-06-10 20:00:00 +0900
lang: ko
translation_key: nodejs-util-styletext-cli-color-output-guide
permalink: /development/blog/seo/2026/06/10/nodejs-util-styletext-cli-color-output-guide.html
alternates:
  ko: /development/blog/seo/2026/06/10/nodejs-util-styletext-cli-color-output-guide.html
  x_default: /development/blog/seo/2026/06/10/nodejs-util-styletext-cli-color-output-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, util, styletext, cli, ansi, terminal, testing]
description: "Node.js util.styleText()로 CLI 메시지에 색상과 강조를 적용하는 방법을 정리합니다. TTY·NO_COLOR 처리, 파이프 출력, ANSI 제거 테스트와 접근성 기준을 실무 예제로 설명합니다."
---

CLI 도구의 성공, 경고, 실패 메시지에 색상을 더하면 사용자가 결과를 빠르게 구분할 수 있습니다.
하지만 ANSI 이스케이프 코드를 직접 붙이면 터미널이 아닌 파일이나 파이프 출력에도 제어 문자가 남고, 색상 비활성화 정책을 일관되게 적용하기 어려워집니다.

Node.js의 `util.styleText()`는 텍스트에 터미널 스타일을 적용하면서 출력 스트림이 색상을 지원하는지도 확인할 수 있는 내장 함수입니다.
별도 색상 라이브러리가 필요하지 않은 작은 CLI나 운영 스크립트에서 특히 유용합니다.

이 글에서는 `util.styleText()`의 기본 사용법, 여러 스타일 조합, TTY와 `NO_COLOR` 처리, 로그와 파이프 출력 기준, ANSI 코드를 제거해 테스트하는 방법을 정리합니다.
CLI 옵션을 함께 설계하고 있다면 [Node.js util.parseArgs 가이드](/development/blog/seo/2026/06/08/nodejs-util-parseargs-cli-options-guide.html)도 참고하세요.

## util.styleText가 필요한 이유

### H3. ANSI 코드를 직접 작성하면 출력 정책이 코드에 섞인다

터미널 색상은 ANSI 이스케이프 시퀀스로 표현할 수 있습니다.
다음 코드는 빨간색 오류 메시지를 출력하지만, 의미를 읽기 어렵고 비활성화 정책도 직접 구현해야 합니다.

```js
console.error('\u001b[31mBuild failed\u001b[39m');
```

이 문자열이 파일이나 로그 수집기로 전달되면 눈에 보이지 않는 제어 문자가 그대로 저장될 수 있습니다.
스타일 시작 코드와 종료 코드의 짝이 맞지 않으면 뒤에 출력되는 메시지까지 색상이 번질 수도 있습니다.

`styleText()`를 사용하면 스타일 이름과 텍스트를 분리해 의도를 드러낼 수 있습니다.

```js
import { styleText } from 'node:util';

console.error(styleText('red', 'Build failed'));
```

기본 동작에서는 출력 스트림이 색상 사용에 적합한지 확인합니다.
색상을 사용할 수 없는 환경이면 원본 텍스트를 반환하므로 같은 메시지를 터미널과 파이프 출력에 함께 사용할 수 있습니다.

### H3. 색상은 의미를 보조해야 한다

색상만으로 성공과 실패를 구분하면 색각 차이가 있는 사용자, 스크린 리더 사용자, 색상을 지원하지 않는 환경에서 의미가 사라질 수 있습니다.
메시지에는 상태를 설명하는 단어와 기호를 함께 넣고 색상은 보조 신호로 사용하는 편이 좋습니다.

```js
console.log(styleText('green', 'SUCCESS: build completed'));
console.error(styleText('red', 'ERROR: build failed'));
```

이 구조는 스타일이 제거돼도 결과를 이해할 수 있습니다.

## util.styleText 기본 사용법

### H3. 스타일 이름과 문자열을 전달한다

`styleText(format, text)`의 첫 번째 인자는 적용할 스타일, 두 번째 인자는 출력할 문자열입니다.

```js
import { styleText } from 'node:util';

console.log(styleText('green', 'deployment completed'));
console.warn(styleText('yellow', 'cache is stale'));
console.error(styleText('red', 'deployment failed'));
```

지원하지 않는 스타일 이름을 전달하면 예외가 발생합니다.
외부 입력을 스타일 이름으로 바로 사용하지 말고, 애플리케이션이 정한 상태와 스타일을 고정된 매핑으로 연결해야 합니다.

```js
const statusStyles = {
  success: 'green',
  warning: 'yellow',
  failure: 'red'
};

export function formatStatus(status, message) {
  const format = statusStyles[status];

  if (!format) {
    throw new TypeError('지원하지 않는 상태입니다.');
  }

  return styleText(format, `${status.toUpperCase()}: ${message}`);
}
```

상태 허용 목록을 두면 잘못된 스타일 이름 때문에 출력 시점에 실패하는 문제를 줄일 수 있습니다.

### H3. 배열로 여러 스타일을 조합한다

첫 번째 인자에 배열을 전달하면 굵게 표시하면서 색상을 입히는 식으로 여러 스타일을 조합할 수 있습니다.

```js
import { styleText } from 'node:util';

const heading = styleText(['bold', 'cyan'], 'Build summary');
const failure = styleText(['bold', 'red'], '3 checks failed');

console.log(heading);
console.error(failure);
```

강조를 너무 많이 사용하면 중요한 정보가 오히려 묻힐 수 있습니다.
제목, 경고, 실패처럼 제한된 상태에만 스타일을 적용하고 일반 설명과 상세 로그는 평문으로 유지하는 편이 읽기 쉽습니다.

## 출력 스트림과 색상 정책 다루기

### H3. 실제로 출력할 스트림을 명시한다

`styleText()`는 기본적으로 `process.stdout`을 기준으로 색상 사용 여부를 판단합니다.
하지만 오류 메시지를 `stderr`에 쓸 때는 검사 대상도 `process.stderr`로 맞추는 편이 정확합니다.

```js
import { styleText } from 'node:util';

function writeError(message) {
  const output = styleText('red', `ERROR: ${message}`, {
    stream: process.stderr
  });

  process.stderr.write(`${output}\n`);
}

writeError('configuration is invalid');
```

`stdout`은 정상 결과나 다른 명령으로 전달할 데이터에, `stderr`는 진단과 오류 메시지에 사용하는 것이 일반적입니다.
두 스트림을 구분하면 사용자가 정상 결과만 파이프로 넘기기도 쉬워집니다.

### H3. 파이프와 파일 출력에는 평문을 유지한다

CLI 결과는 터미널뿐 아니라 파일, CI 로그, 다른 프로세스의 입력으로도 전달됩니다.

```bash
node cli.js > result.txt
node cli.js | jq .
```

기본 스트림 검사를 유지하면 색상을 지원하지 않는 출력에서는 `styleText()`가 평문을 반환할 수 있습니다.
특별한 이유 없이 `{ validateStream: false }`를 사용해 색상을 강제하면 파이프 결과에 ANSI 코드가 섞일 수 있으므로 주의해야 합니다.

구조화된 JSON을 출력하는 명령이라면 데이터에는 색상을 넣지 말고, 필요한 진단 메시지만 `stderr`로 분리하는 편이 안전합니다.

```js
process.stdout.write(`${JSON.stringify({ status: 'ok' })}\n`);

process.stderr.write(`${styleText('dim', 'report generated', {
  stream: process.stderr
})}\n`);
```

### H3. NO_COLOR 같은 사용자 정책을 존중한다

터미널이 색상을 지원하더라도 사용자가 접근성, 로그 가독성, 자동화 호환성을 위해 색상을 끄고 싶을 수 있습니다.
Node.js의 색상 판단은 실행 환경의 색상 관련 설정을 고려하므로, 애플리케이션에서 무조건 ANSI 출력을 강제하지 않는 것이 좋습니다.

CLI에 별도 `--color` 옵션을 제공한다면 정책 우선순위를 문서화해야 합니다.
예를 들어 기본값은 자동 감지, `--no-color`는 비활성화, 명시적인 `--color`만 강제 활성화처럼 규칙을 정할 수 있습니다.

다만 강제 활성화는 사람이 터미널에서 보는 명령에만 제한하는 편이 좋습니다.
CI와 파이프라인에서 제어 문자가 섞이면 후속 파서가 실패하거나 로그 검색이 어려워질 수 있습니다.

## 재사용 가능한 CLI 메시지 포매터 만들기

### H3. 상태와 출력 스트림을 한곳에서 매핑한다

메시지마다 직접 색상과 스트림을 선택하면 시간이 지나면서 같은 상태가 서로 다른 형식으로 출력될 수 있습니다.
작은 포매터를 두면 상태 이름, 색상, 출력 스트림 정책을 한곳에서 관리할 수 있습니다.

```js
import { styleText } from 'node:util';

const levels = {
  info: { format: 'cyan', stream: process.stdout },
  success: { format: 'green', stream: process.stdout },
  warning: { format: 'yellow', stream: process.stderr },
  error: { format: ['bold', 'red'], stream: process.stderr }
};

export function writeMessage(level, message) {
  const config = levels[level];

  if (!config) {
    throw new TypeError('지원하지 않는 메시지 레벨입니다.');
  }

  const text = styleText(
    config.format,
    `${level.toUpperCase()}: ${message}`,
    { stream: config.stream }
  );

  config.stream.write(`${text}\n`);
}
```

메시지에 토큰, 비밀번호, 개인 식별 정보, 전체 요청 본문을 넣지 않아야 합니다.
색상 적용 여부와 관계없이 터미널 출력도 로그나 화면 녹화에 남을 수 있습니다.

### H3. 데이터 출력과 사람용 출력을 분리한다

자동화에 사용되는 CLI는 사람이 읽는 메시지와 프로그램이 소비하는 결과를 명확히 나눠야 합니다.
예를 들어 `--json` 모드에서는 `stdout`에 JSON만 출력하고, 진행 상태는 필요할 때만 `stderr`에 남깁니다.

```js
export function printResult(result, { json = false } = {}) {
  if (json) {
    process.stdout.write(`${JSON.stringify(result)}\n`);
    return;
  }

  writeMessage('success', `created ${result.count} files`);
}
```

이렇게 하면 색상 출력이 JSON 파싱을 깨뜨리는 문제를 피할 수 있습니다.
옵션 파싱 단계에서는 허용되지 않은 조합도 미리 거부해 명령의 동작을 예측 가능하게 만들어야 합니다.

## 색상 출력을 테스트하는 방법

### H3. 의미 텍스트를 먼저 검증한다

테스트 환경의 출력 스트림은 실제 터미널이 아닐 수 있으므로 ANSI 코드 자체를 항상 기대하면 테스트가 환경에 따라 달라질 수 있습니다.
가장 중요한 것은 색상을 제거해도 메시지 의미가 유지되는지 확인하는 것입니다.

Node.js의 `util.stripVTControlCharacters()`는 ANSI 같은 터미널 제어 문자를 제거합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import {
  stripVTControlCharacters,
  styleText
} from 'node:util';

test('failure message keeps its meaning without color', () => {
  const output = styleText(['bold', 'red'], 'ERROR: build failed', {
    validateStream: false
  });

  assert.equal(
    stripVTControlCharacters(output),
    'ERROR: build failed'
  );
});
```

`validateStream: false`는 테스트에서 ANSI 스타일을 의도적으로 생성하기 위해 제한적으로 사용했습니다.
운영 출력에서 같은 옵션을 무조건 적용하지 않아야 합니다.

제어 문자 제거 기준은 [Node.js util.stripVTControlCharacters 가이드](/development/blog/seo/2026/05/31/nodejs-util-strip-vt-control-characters-cli-output-guide.html)에서 더 자세히 확인할 수 있습니다.

### H3. 색상 비활성 환경과 스트림 분리도 확인한다

CLI 테스트에서는 다음 동작을 별도로 확인하는 편이 좋습니다.

- 색상이 없어도 상태와 의미를 이해할 수 있는가?
- JSON이나 파이프용 `stdout`에 ANSI 코드가 섞이지 않는가?
- 오류 메시지가 `stderr`로 출력되는가?
- 알 수 없는 메시지 레벨이나 스타일을 거부하는가?
- 민감정보가 메시지에 포함되지 않는가?

출력 문자열 전체를 ANSI 코드까지 고정해 비교하면 Node.js 버전이나 터미널 정책 변화에 테스트가 불필요하게 묶일 수 있습니다.
스타일 자체가 제품 요구사항인 부분만 제한적으로 검증하고, 나머지는 제어 문자를 제거한 의미 텍스트와 스트림 선택을 검사하는 것이 안정적입니다.

## 실무 체크리스트

- ANSI 코드를 직접 조합하지 않고 `util.styleText()`를 사용하는가?
- 색상 없이도 상태와 의미를 이해할 수 있는가?
- 오류 출력은 `process.stderr`를 기준으로 스타일을 판단하는가?
- 파이프, 파일, JSON 출력에 ANSI 코드가 섞이지 않는가?
- 사용자와 실행 환경의 색상 비활성화 정책을 존중하는가?
- 외부 입력을 검증 없이 스타일 이름으로 사용하지 않는가?
- 터미널 메시지에 토큰, 개인정보, 요청 본문을 출력하지 않는가?
- 테스트에서는 `stripVTControlCharacters()`로 의미 텍스트를 검증하는가?

## FAQ

### H3. util.styleText를 쓰면 항상 색상이 출력되나요?

아닙니다.
기본 동작은 출력 스트림과 실행 환경을 확인하고, 색상을 사용하지 않는 환경에서는 평문을 반환할 수 있습니다.
이 동작 덕분에 같은 메시지를 터미널과 파이프 출력에 비교적 안전하게 사용할 수 있습니다.

### H3. stdout과 stderr 중 무엇을 stream 옵션에 전달해야 하나요?

실제로 메시지를 쓸 스트림을 전달하는 편이 좋습니다.
정상 결과와 일반 안내를 `stdout`에 쓴다면 `process.stdout`, 경고와 오류를 `stderr`에 쓴다면 `process.stderr`를 사용하세요.

### H3. 색상이 적용됐는지 테스트에서 어떻게 확인하나요?

스타일 적용 자체를 검사해야 할 때는 테스트 안에서만 `validateStream: false`를 사용해 ANSI 출력을 만들 수 있습니다.
다만 대부분의 테스트에서는 `stripVTControlCharacters()`로 제어 문자를 제거한 뒤 의미 있는 텍스트와 출력 스트림을 검증하는 편이 더 안정적입니다.

## 마무리

Node.js `util.styleText()`는 작은 CLI와 운영 스크립트에 색상과 강조를 적용하면서 출력 환경에 따른 정책도 함께 다룰 수 있게 해 줍니다.
직접 ANSI 코드를 조합하는 대신 스타일 이름을 사용하면 코드 의도가 선명해지고, 파이프와 파일 출력에서 제어 문자가 섞이는 문제도 줄일 수 있습니다.

색상은 메시지의 의미를 대신하지 않아야 합니다.
상태 단어, 올바른 `stdout`·`stderr` 분리, 색상 비활성화 정책, 민감정보 없는 출력, ANSI 제거 테스트까지 함께 적용하면 사람이 읽기 쉽고 자동화에도 안정적인 CLI 출력을 만들 수 있습니다.

함께 읽으면 좋은 글:

- [Node.js util.parseArgs 가이드: CLI 옵션을 안전하게 파싱하는 법](/development/blog/seo/2026/06/08/nodejs-util-parseargs-cli-options-guide.html)
- [Node.js util.stripVTControlCharacters 가이드: ANSI 제어 문자 제거하기](/development/blog/seo/2026/05/31/nodejs-util-strip-vt-control-characters-cli-output-guide.html)
- [개발 글에서 CLI 출력값을 안전하게 공개하는 기준](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)

---
layout: post
title: "Node.js util.parseArgs 가이드: CLI 옵션 파싱을 안전하게 시작하는 법"
date: 2026-06-08 20:00:00 +0900
lang: ko
translation_key: nodejs-util-parseargs-cli-options-guide
permalink: /development/blog/seo/2026/06/08/nodejs-util-parseargs-cli-options-guide.html
alternates:
  ko: /development/blog/seo/2026/06/08/nodejs-util-parseargs-cli-options-guide.html
  x_default: /development/blog/seo/2026/06/08/nodejs-util-parseargs-cli-options-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, util, parseargs, cli, options, backend]
description: "Node.js util.parseArgs()로 CLI 옵션을 파싱하는 방법을 정리했습니다. boolean/string 옵션, short alias, positional 인자, strict 모드, 기본값 검증, 도움말 출력까지 실무 예제로 설명합니다."
---

Node.js로 작은 배포 스크립트, 데이터 마이그레이션, 로컬 자동화 도구를 만들다 보면 명령행 옵션 파싱이 필요합니다.
처음에는 `process.argv`를 직접 잘라 써도 충분해 보이지만, 옵션 이름, 짧은 별칭, 값 누락, 잘못된 플래그, 위치 인자를 다루기 시작하면 금방 복잡해집니다.

`util.parseArgs()`는 Node.js에 내장된 CLI 옵션 파서입니다.
외부 패키지를 추가하지 않고도 `--dry-run`, `--env production`, `-v` 같은 옵션을 구조화된 값으로 받을 수 있습니다.

이 글에서는 `util.parseArgs()`의 기본 사용법, 옵션 스키마 작성법, strict 모드, 위치 인자 처리, 기본값과 검증, 도움말 출력 패턴을 정리합니다.
스크립트 실행 결과를 로그나 문서에 남길 때는 [CLI 출력 민감정보 정리 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)도 함께 참고하면 좋습니다.

## util.parseArgs가 필요한 이유

### H3. process.argv 직접 파싱은 금방 깨진다

Node.js 스크립트는 `process.argv`로 명령행 인자를 받을 수 있습니다.
아래처럼 직접 파싱하는 코드는 간단해 보입니다.

```js
const args = process.argv.slice(2);
const dryRun = args.includes('--dry-run');
const envIndex = args.indexOf('--env');
const env = envIndex >= 0 ? args[envIndex + 1] : 'development';

console.log({ dryRun, env });
```

하지만 실제 운영 스크립트에서는 다음 질문이 바로 생깁니다.

- `--env` 뒤에 값이 없으면 어떻게 할 것인가?
- `--unknown` 같은 잘못된 옵션은 허용할 것인가?
- `-e production` 같은 짧은 옵션도 받을 것인가?
- `--` 뒤의 값은 옵션이 아니라 위치 인자로 볼 것인가?
- boolean 옵션에 실수로 값을 붙이면 어떻게 처리할 것인가?

이런 규칙을 매번 직접 구현하면 스크립트마다 동작이 달라집니다.
배포, 마이그레이션, 정리 작업처럼 실수가 비용으로 이어지는 CLI에서는 옵션 파싱 규칙을 명시하는 편이 안전합니다.

### H3. 내장 API라 작은 도구에 부담이 적다

`util.parseArgs()`는 Node.js 내장 모듈인 `node:util`에서 가져옵니다.
작은 내부 도구나 저장소 루트의 유지보수 스크립트에서 의존성을 늘리지 않고 옵션 파싱을 시작할 수 있습니다.

```js
import { parseArgs } from 'node:util';

const { values } = parseArgs({
  options: {
    env: { type: 'string' },
    dryRun: { type: 'boolean' }
  }
});

console.log(values);
```

`values`에는 정의한 옵션 이름을 기준으로 파싱된 결과가 들어갑니다.
정의하지 않은 옵션은 기본적으로 에러가 되므로, 오타가 조용히 무시되는 일을 줄일 수 있습니다.

## 기본 사용법

### H3. boolean 옵션과 string 옵션을 구분한다

`parseArgs()`의 핵심은 옵션 스키마입니다.
각 옵션은 `type`으로 `boolean` 또는 `string`을 지정합니다.

```js
import { parseArgs } from 'node:util';

const { values } = parseArgs({
  options: {
    dryRun: {
      type: 'boolean',
      short: 'd'
    },
    env: {
      type: 'string',
      short: 'e'
    }
  }
});

console.log({
  dryRun: values.dryRun ?? false,
  env: values.env ?? 'development'
});
```

이제 아래 두 명령은 같은 의도로 처리할 수 있습니다.

```bash
node deploy.js --dry-run --env staging
node deploy.js -d -e staging
```

boolean 옵션은 값 없이 존재 여부로 판단합니다.
string 옵션은 뒤따르는 값을 요구합니다.
값이 빠지거나 타입 규칙에 맞지 않으면 파서 단계에서 에러가 발생합니다.

### H3. 기본값은 파싱 뒤 명시적으로 채운다

`parseArgs()`는 옵션을 구조화해 주지만, 모든 업무 규칙을 대신 결정해 주지는 않습니다.
기본값은 파싱 결과를 받은 뒤 애플리케이션 코드에서 명시적으로 채우는 편이 읽기 쉽습니다.

```js
import { parseArgs } from 'node:util';

const { values } = parseArgs({
  options: {
    env: { type: 'string' },
    limit: { type: 'string' },
    verbose: { type: 'boolean', short: 'v' }
  }
});

const config = {
  env: values.env ?? 'development',
  limit: Number(values.limit ?? 100),
  verbose: values.verbose ?? false
};
```

주의할 점은 `string` 옵션으로 받은 숫자는 여전히 문자열이라는 것입니다.
숫자 범위, 허용 목록, 경로 존재 여부 같은 업무 검증은 별도 함수로 분리해야 합니다.

## 검증과 에러 처리

### H3. 허용 값을 좁히면 배포 실수를 줄인다

CLI 옵션은 사람이 직접 입력하는 값입니다.
따라서 파싱 성공과 업무적으로 유효한 값은 별개로 봐야 합니다.

```js
const allowedEnvs = new Set(['development', 'staging', 'production']);

function validateEnv(value) {
  if (!allowedEnvs.has(value)) {
    throw new Error(`env must be one of: ${Array.from(allowedEnvs).join(', ')}`);
  }

  return value;
}

function validateLimit(value) {
  const number = Number(value);

  if (!Number.isInteger(number) || number < 1 || number > 1000) {
    throw new Error('limit must be an integer between 1 and 1000');
  }

  return number;
}

const env = validateEnv(config.env);
const limit = validateLimit(config.limit);
```

특히 `production`을 대상으로 하는 스크립트라면 허용 값을 최대한 좁혀야 합니다.
오타가 난 환경 이름을 새 환경처럼 처리하거나, 너무 큰 `limit`으로 한 번에 과도한 작업을 실행하는 일을 막을 수 있습니다.

운영 스크립트에서 환경 변수와 CLI 옵션을 함께 다룬다면 [Node.js loadEnvFile 가이드](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)처럼 부팅 시점 검증을 한 곳에 모으는 방식도 유용합니다.

### H3. 에러 메시지는 짧고 실행 가능한 형태로 쓴다

CLI 도구의 에러는 길게 설명하기보다 사용자가 바로 고칠 수 있어야 합니다.
파싱과 검증을 `try/catch`로 감싸고, 실패 시 사용법을 함께 보여 주면 재시도가 쉬워집니다.

```js
import { parseArgs } from 'node:util';

function printHelp() {
  console.log(`
Usage:
  node deploy.js --env <development|staging|production> [--dry-run]

Options:
  -e, --env      Target environment
  -d, --dry-run  Print actions without applying changes
  -h, --help     Show this help message
`.trim());
}

try {
  const { values } = parseArgs({
    options: {
      env: { type: 'string', short: 'e' },
      dryRun: { type: 'boolean', short: 'd' },
      help: { type: 'boolean', short: 'h' }
    }
  });

  if (values.help) {
    printHelp();
    process.exit(0);
  }

  const env = validateEnv(values.env ?? 'development');
  console.log({ env, dryRun: values.dryRun ?? false });
} catch (error) {
  console.error(error.message);
  printHelp();
  process.exit(1);
}
```

내부 자동화 도구라도 도움말은 투자할 만합니다.
몇 달 뒤 다시 실행하는 사람은 작성자 본인일 가능성이 높고, 사용법이 코드 안에만 있으면 실수 확률이 올라갑니다.

## 위치 인자와 strict 모드

### H3. positional 인자는 allowPositionals로 받는다

옵션이 아닌 위치 인자가 필요한 CLI도 많습니다.
예를 들어 특정 파일을 변환하는 명령은 파일 경로를 위치 인자로 받을 수 있습니다.

```js
import { parseArgs } from 'node:util';

const { values, positionals } = parseArgs({
  allowPositionals: true,
  options: {
    format: { type: 'string', short: 'f' },
    dryRun: { type: 'boolean', short: 'd' }
  }
});

if (positionals.length !== 1) {
  throw new Error('exactly one input file is required');
}

const inputFile = positionals[0];
const format = values.format ?? 'json';
```

위치 인자를 허용할 때도 개수와 형식을 검증해야 합니다.
파일 경로라면 존재 여부, 확장자, 쓰기 대상 경로와의 충돌 여부까지 확인하는 편이 좋습니다.

### H3. strict 기본값은 유지하는 편이 안전하다

`parseArgs()`는 기본적으로 strict하게 동작합니다.
정의하지 않은 옵션이나 잘못된 값은 에러로 처리됩니다.

내부 도구에서는 이 기본값을 유지하는 편이 좋습니다.
옵션 오타가 조용히 무시되면 사용자는 명령이 의도대로 실행됐다고 착각할 수 있습니다.

```bash
node deploy.js --env production --dryrun
```

위 예시에서 `--dryrun`을 허용하지 않은 옵션으로 잡아내면 사용자가 `--dry-run` 오타를 바로 수정할 수 있습니다.
반대로 unknown 옵션을 허용하면 실제 배포가 dry run 없이 진행될 수 있습니다.

strict를 낮추는 것은 다른 도구로 인자를 넘겨야 하는 래퍼 CLI처럼 목적이 분명할 때만 검토하는 편이 안전합니다.

## 실무 패턴

### H3. 파싱, 검증, 실행을 나눈다

CLI 스크립트도 작은 애플리케이션입니다.
파싱과 검증, 실제 실행 로직을 나누면 테스트하기 쉽고, 에러 메시지도 일관되게 관리할 수 있습니다.

```js
import { parseArgs } from 'node:util';

export function parseCliArgs(args = process.argv.slice(2)) {
  const { values, positionals } = parseArgs({
    args,
    allowPositionals: true,
    options: {
      env: { type: 'string', short: 'e' },
      dryRun: { type: 'boolean', short: 'd' },
      limit: { type: 'string' }
    }
  });

  return {
    env: validateEnv(values.env ?? 'development'),
    dryRun: values.dryRun ?? false,
    limit: validateLimit(values.limit ?? '100'),
    files: validateFiles(positionals)
  };
}
```

`args`를 인자로 받을 수 있게 만들면 테스트에서 `process.argv`를 직접 바꾸지 않아도 됩니다.
실행 함수는 이미 검증된 설정 객체만 받도록 구성합니다.

```js
export async function main() {
  const options = parseCliArgs();
  await runJob(options);
}
```

이 구조는 테스트 러너와도 잘 맞습니다.
CLI 파싱 함수를 작은 단위로 검증하는 방법은 [Node.js test runner mock 함수 가이드](/development/blog/seo/2026/05/22/nodejs-test-runner-mock-fn-dependency-testing-guide.html)와 함께 적용할 수 있습니다.

### H3. 위험한 명령에는 dry run을 기본 경험에 넣는다

삭제, 배포, 마이그레이션, 대량 수정처럼 되돌리기 어려운 작업에는 `--dry-run` 옵션이 거의 필수입니다.
`parseArgs()`로 옵션을 받는 것에서 끝내지 말고, 실행 로그도 dry run 여부를 명확히 드러내야 합니다.

```js
function logPlan({ env, dryRun, targets }) {
  console.log(`Environment: ${env}`);
  console.log(`Mode: ${dryRun ? 'dry run' : 'apply'}`);
  console.log(`Targets: ${targets.length}`);
}
```

자동화 로그에는 토큰, 쿠키, 개인 식별 정보, 내부 URL처럼 민감할 수 있는 값을 그대로 남기지 않아야 합니다.
작업 대상이 많을 때도 전체 목록을 무제한 출력하기보다 개수와 샘플 몇 개만 보여 주는 편이 안전합니다.

## FAQ

### H3. util.parseArgs만으로 commander나 yargs를 대체할 수 있나요?

작은 내부 도구라면 충분한 경우가 많습니다.
하지만 중첩 명령, 풍부한 도움말, 자동 완성, 복잡한 옵션 그룹이 필요하다면 전용 CLI 프레임워크가 더 적합할 수 있습니다.
기준은 "옵션 몇 개를 안전하게 파싱하면 되는가, 아니면 제품 수준의 CLI 경험이 필요한가"입니다.

### H3. 숫자 옵션은 왜 string으로 받은 뒤 변환하나요?

`parseArgs()`의 옵션 타입은 `boolean`과 `string` 중심입니다.
따라서 숫자, 날짜, 경로, enum은 문자열로 받은 뒤 업무 규칙에 맞게 직접 검증하는 편이 명확합니다.
이 과정을 분리하면 에러 메시지도 더 친절하게 만들 수 있습니다.

### H3. 옵션 이름은 camelCase와 kebab-case 중 무엇이 좋나요?

명령행에서는 `--dry-run`처럼 kebab-case가 읽기 쉽습니다.
Node.js 코드에서는 `dryRun`이 자연스럽습니다.
프로젝트 안에서 한 가지 규칙을 정하고 도움말, 문서, 테스트에서 같은 이름을 유지하는 것이 더 중요합니다.

## 마무리

`util.parseArgs()`는 Node.js CLI 스크립트의 옵션 파싱을 단순하고 일관되게 만들어 줍니다.
작은 자동화 도구라면 외부 의존성을 늘리지 않고도 boolean 옵션, string 옵션, 짧은 별칭, 위치 인자를 안정적으로 다룰 수 있습니다.

핵심은 파싱만으로 끝내지 않는 것입니다.
파싱 뒤 기본값을 채우고, 허용 값을 검증하고, 에러와 도움말을 함께 설계해야 실무에서 반복 실행해도 안전한 CLI가 됩니다.
운영 자동화 도구를 만들 때는 `parseArgs()`를 시작점으로 삼고, 위험한 작업에는 strict 모드와 dry run, 민감정보를 숨기는 로그 정책을 함께 적용하세요.

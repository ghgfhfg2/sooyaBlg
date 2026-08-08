---
layout: post
title: "Node.js util.parseArgs 가이드: CLI 옵션 파싱을 의존성 없이 안전하게 처리하는 법"
date: 2026-08-09 08:00:00 +0900
lang: ko
translation_key: nodejs-util-parseargs-cli-options-guide
permalink: /development/blog/seo/2026/08/09/nodejs-util-parseargs-cli-options-guide.html
alternates:
  ko: /development/blog/seo/2026/08/09/nodejs-util-parseargs-cli-options-guide.html
  x_default: /development/blog/seo/2026/08/09/nodejs-util-parseargs-cli-options-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, util, parseargs, cli, command-line, javascript, backend, automation]
description: "Node.js util.parseArgs로 CLI 옵션을 의존성 없이 파싱하는 방법을 정리합니다. boolean/string 옵션, short alias, positional 인자, strict 검증, 민감정보 로그 처리까지 실무 예제로 설명합니다."
---

작은 운영 스크립트나 배포 자동화 도구를 만들 때 CLI 옵션 파싱 때문에 별도 패키지를 추가할지 고민할 때가 있습니다.
복잡한 서브커맨드, 도움말 생성, 자동 완성까지 필요하다면 전문 CLI 프레임워크가 편합니다.
하지만 `--dry-run`, `--env production`, `--limit 20`처럼 단순한 옵션을 안정적으로 읽는 목적이라면 Node.js 기본 모듈만으로도 충분한 경우가 많습니다.

`util.parseArgs()`는 Node.js의 `node:util` 모듈이 제공하는 명령행 인자 파서입니다.
옵션 타입, 짧은 별칭, 위치 인자, strict 검증을 선언적으로 다룰 수 있어 `process.argv`를 직접 문자열로 쪼개는 코드보다 실수를 줄일 수 있습니다.
이 글에서는 `util.parseArgs()`의 기본 사용법과 실무 CLI에서 자주 필요한 검증, 기본값, 민감정보 처리 기준을 정리합니다.

CLI 출력에 토큰이나 경로가 섞일 수 있다면 [CLI 출력값 민감정보 제거 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)를 함께 볼 수 있습니다.
환경변수와 실행 옵션의 경계를 정해야 한다면 [Node.js loadEnvFile 가이드](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)도 이어서 도움이 됩니다.

## util.parseArgs가 필요한 이유

### H3. process.argv 직접 파싱은 금방 흔들린다

가장 단순한 CLI는 `process.argv` 배열을 직접 읽어도 됩니다.

```js
const env = process.argv[2];
const dryRun = process.argv.includes('--dry-run');
```

하지만 옵션이 조금만 늘어도 문제가 생깁니다.
옵션 순서가 바뀌거나, 값이 빠지거나, 짧은 별칭을 추가하거나, 위치 인자를 함께 받아야 할 때 조건문이 빠르게 복잡해집니다.

```txt
node publish.js production --dry-run
node publish.js --dry-run --env production
node publish.js -e production --limit 20
```

세 명령을 같은 의미로 처리하려면 인자 배열의 모양이 아니라 "어떤 옵션을 허용할지"를 먼저 선언하는 편이 좋습니다.
`util.parseArgs()`는 그 선언을 코드 가까이에 둘 수 있게 해 줍니다.

### H3. 작은 자동화는 의존성이 적을수록 관리가 쉽다

사내 스크립트, GitHub Actions 보조 도구, 배치 작업 실행기처럼 오래 유지되는 작은 CLI는 설치 환경이 단순해야 합니다.
Node.js 기본 모듈만 사용하면 패키지 설치 실패, lockfile 변경, 공급망 점검 부담을 줄일 수 있습니다.

물론 모든 CLI를 기본 모듈만으로 만들 필요는 없습니다.
명령어가 많고 도움말, 프롬프트, 색상 출력, 플러그인 구조가 필요하다면 전용 라이브러리가 더 낫습니다.
기준은 단순합니다.
파싱 규칙이 작고 명확하면 `util.parseArgs()`로 시작하고, 사용자 경험이 CLI 제품 수준으로 커지면 프레임워크를 검토하면 됩니다.

## 기본 사용법

### H3. boolean 옵션과 string 옵션을 선언한다

`parseArgs()`에는 허용할 옵션을 `options` 객체로 넘깁니다.
각 옵션은 타입을 가집니다.

```js
import { parseArgs } from 'node:util';

const { values } = parseArgs({
  options: {
    'dry-run': {
      type: 'boolean'
    },
    env: {
      type: 'string'
    }
  }
});

console.log(values);
```

아래처럼 실행하면 `values`에 파싱 결과가 들어갑니다.

```sh
node publish.js --dry-run --env production
```

결과는 대략 이런 모양입니다.

```js
{
  'dry-run': true,
  env: 'production'
}
```

`--dry-run`처럼 하이픈이 들어간 옵션 이름은 점 표기법으로 읽기 어렵습니다.
실무에서는 한 번 정규화해서 애플리케이션 내부 이름으로 바꾸는 편이 좋습니다.

```js
export function readCliOptions() {
  const { values } = parseArgs({
    options: {
      'dry-run': { type: 'boolean' },
      env: { type: 'string' }
    }
  });

  return {
    dryRun: values['dry-run'] === true,
    env: values.env ?? 'staging'
  };
}
```

기본값을 이 단계에서 정하면 나머지 코드가 `undefined`를 계속 신경 쓰지 않아도 됩니다.

### H3. short 옵션은 shortName으로 연결한다

운영자가 자주 입력하는 옵션에는 짧은 별칭을 둘 수 있습니다.

```js
import { parseArgs } from 'node:util';

const { values } = parseArgs({
  options: {
    env: {
      type: 'string',
      short: 'e'
    },
    limit: {
      type: 'string',
      short: 'l'
    },
    verbose: {
      type: 'boolean',
      short: 'v'
    }
  }
});

console.log(values);
```

다음 두 명령은 같은 의미로 읽을 수 있습니다.

```sh
node job.js --env production --limit 50 --verbose
node job.js -e production -l 50 -v
```

`limit`처럼 숫자로 쓰고 싶은 값도 먼저 문자열로 받은 뒤 직접 검증하는 편이 안전합니다.
CLI 입력은 결국 외부 입력이기 때문입니다.

```js
function parsePositiveInteger(value, fallback) {
  if (value == null) {
    return fallback;
  }

  const number = Number(value);

  if (!Number.isInteger(number) || number <= 0) {
    throw new TypeError('limit must be a positive integer');
  }

  return number;
}
```

숫자 변환을 옵션 파싱과 분리하면 오류 메시지를 도메인에 맞게 다듬기 쉽습니다.

## 위치 인자 다루기

### H3. allowPositionals로 파일 경로나 작업 이름을 받는다

CLI에는 옵션이 아닌 위치 인자가 필요할 때가 있습니다.
예를 들어 마이그레이션 파일 경로나 배포 대상 서비스를 받을 수 있습니다.

```js
import { parseArgs } from 'node:util';

const { values, positionals } = parseArgs({
  allowPositionals: true,
  options: {
    env: {
      type: 'string',
      short: 'e'
    },
    'dry-run': {
      type: 'boolean'
    }
  }
});

const [taskName] = positionals;

if (!taskName) {
  throw new Error('task name is required');
}

console.log({
  taskName,
  env: values.env ?? 'staging',
  dryRun: values['dry-run'] === true
});
```

실행 예시는 아래와 같습니다.

```sh
node run-task.js rebuild-search-index --env production --dry-run
```

위치 인자는 자유도가 높기 때문에 검증을 빼먹기 쉽습니다.
파일 경로라면 허용 루트 안에 있는지 확인하고, 작업 이름이라면 허용 목록과 비교하는 것이 좋습니다.
경로 검증 기준은 [Node.js path.resolve/normalize 가이드](/development/blog/seo/2026/08/06/nodejs-path-resolve-normalize-traversal-prevention-guide.html)를 참고할 수 있습니다.

### H3. 옵션 뒤의 값을 위치 인자로 오해하지 않게 한다

직접 파싱할 때 흔한 버그는 옵션 값과 위치 인자를 섞어 읽는 것입니다.

```txt
node deploy.js api --env production
```

여기서 `api`는 위치 인자이고 `production`은 `env` 옵션의 값입니다.
`parseArgs()`는 옵션 선언을 기준으로 이 차이를 구분합니다.
따라서 호출부에서는 `values`와 `positionals`를 분리해서 다루면 됩니다.

```js
const command = positionals[0];
const env = values.env ?? 'staging';
```

명령어가 늘어나면 위치 인자 배열을 계속 직접 읽기보다 명령별 파싱 함수를 분리하는 편이 좋습니다.

```js
const COMMANDS = new Set(['publish', 'rollback', 'rebuild-search-index']);

function parseCommand(positionals) {
  const [command] = positionals;

  if (!COMMANDS.has(command)) {
    throw new Error(`unknown command: ${command ?? '(missing)'}`);
  }

  return command;
}
```

오류 메시지에는 사용자가 고칠 수 있는 정보를 담되, 민감한 인자 전체를 그대로 출력하지 않는 것이 좋습니다.

## strict 검증과 오류 처리

### H3. 알 수 없는 옵션은 기본적으로 실패시킨다

`parseArgs()`는 기본적으로 strict하게 동작합니다.
선언하지 않은 옵션을 넘기면 오류가 발생합니다.
운영 스크립트에서는 이 동작이 유리합니다.
오타가 조용히 무시되는 것보다 실행 전에 실패하는 편이 사고를 줄입니다.

```sh
node publish.js --env production --dryrun
```

`--dryrun`이 오타라면 실행이 멈춰야 합니다.
그렇지 않으면 dry run이라고 생각했는데 실제 배포가 진행되는 위험한 상황이 생길 수 있습니다.

오류를 사람이 읽기 좋게 감싸려면 진입점에서만 처리합니다.

```js
import { parseArgs } from 'node:util';

function parsePublishOptions() {
  const { values } = parseArgs({
    options: {
      env: { type: 'string', short: 'e' },
      'dry-run': { type: 'boolean' }
    }
  });

  return {
    env: values.env ?? 'staging',
    dryRun: values['dry-run'] === true
  };
}

try {
  const options = parsePublishOptions();
  console.log(options);
} catch (error) {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
}
```

라이브러리 함수 안에서 바로 `process.exit()`를 호출하면 테스트하기 어렵습니다.
파싱 함수는 오류를 던지고, CLI 진입점이 exit code를 정하는 구조가 유지보수에 좋습니다.

### H3. strict false는 래퍼 CLI에서만 제한적으로 쓴다

외부 도구에 옵션을 그대로 넘기는 래퍼 CLI라면 선언하지 않은 옵션을 허용해야 할 때가 있습니다.
이때는 `strict: false`를 검토할 수 있습니다.

```js
import { parseArgs } from 'node:util';

const { values, positionals } = parseArgs({
  strict: false,
  allowPositionals: true,
  options: {
    config: {
      type: 'string',
      short: 'c'
    }
  }
});

console.log({ values, positionals });
```

다만 일반 운영 스크립트에서는 strict를 끄지 않는 편이 좋습니다.
허용하지 않은 옵션을 받아들이면 오타, 폐기된 플래그, 잘못된 실행 문서가 조용히 통과할 수 있습니다.
특히 배포, 삭제, 권한 변경처럼 영향이 큰 작업은 입력을 좁게 받는 것이 안전합니다.

## 실무 예제: 배포 보조 CLI 만들기

### H3. 파싱과 검증을 한 함수로 모은다

다음 예제는 배포 보조 스크립트에서 자주 필요한 옵션을 읽습니다.

```js
import { parseArgs } from 'node:util';

const ALLOWED_ENVS = new Set(['staging', 'production']);

function parsePositiveInteger(value, fallback) {
  if (value == null) {
    return fallback;
  }

  const number = Number(value);

  if (!Number.isInteger(number) || number <= 0) {
    throw new TypeError('limit must be a positive integer');
  }

  return number;
}

export function parseDeployOptions(args = process.argv.slice(2)) {
  const { values, positionals } = parseArgs({
    args,
    allowPositionals: true,
    options: {
      env: { type: 'string', short: 'e' },
      limit: { type: 'string', short: 'l' },
      'dry-run': { type: 'boolean' },
      confirm: { type: 'boolean' },
      verbose: { type: 'boolean', short: 'v' }
    }
  });

  const service = positionals[0];
  const env = values.env ?? 'staging';

  if (!service) {
    throw new Error('service name is required');
  }

  if (!ALLOWED_ENVS.has(env)) {
    throw new Error('env must be staging or production');
  }

  return {
    service,
    env,
    limit: parsePositiveInteger(values.limit, 20),
    dryRun: values['dry-run'] === true,
    confirm: values.confirm === true,
    verbose: values.verbose === true
  };
}
```

`args`를 기본값으로 주입받게 만든 이유는 테스트를 쉽게 하기 위해서입니다.
테스트에서는 실제 `process.argv`를 바꾸지 않고도 여러 입력 조합을 검증할 수 있습니다.

### H3. 위험한 작업은 dry run 기본값을 신중하게 둔다

배포나 삭제처럼 되돌리기 어려운 작업은 dry run 정책을 명확히 해야 합니다.
예를 들어 production에서는 명시적인 `--confirm` 없이는 실행하지 않도록 만들 수 있습니다.

```js
export function assertExecutionAllowed(options) {
  if (options.env === 'production' && options.dryRun) {
    return;
  }

  if (options.env === 'production' && options.confirm !== true) {
    throw new Error('production deploy requires --confirm');
  }
}
```

이 예제에서 중요한 점은 옵션 이름이 아니라 정책입니다.
위험한 작업을 안전하게 만들려면 CLI 파싱 이후에 별도 검증 계층을 둬야 합니다.
파서는 문자열을 구조화해 줄 뿐, 업무 규칙까지 대신 판단하지 않습니다.

`--force`, `--yes`, `--confirm` 같은 옵션은 편하지만 사고 반경을 넓힐 수 있습니다.
사용한다면 로그에 남기고, CI 환경과 로컬 환경의 허용 기준을 분리하는 편이 좋습니다.

## 테스트 기준

### H3. process.argv 대신 args 주입으로 테스트한다

파싱 함수를 테스트할 때 전역 `process.argv`를 직접 바꾸면 테스트 간 간섭이 생길 수 있습니다.
앞의 예제처럼 `args`를 인자로 받게 만들면 입력과 기대값을 작게 고정할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { parseDeployOptions } from './deploy-options.js';

test('parses deploy options', () => {
  const options = parseDeployOptions([
    'api',
    '--env',
    'production',
    '--limit',
    '10',
    '--dry-run'
  ]);

  assert.deepStrictEqual(options, {
    service: 'api',
    env: 'production',
    limit: 10,
    dryRun: true,
    confirm: false,
    verbose: false
  });
});
```

Node.js 테스트 러너를 처음 쓴다면 [Node.js test runner describe/it 가이드](/development/blog/seo/2026/07/10/nodejs-test-runner-describe-it-suite-structure-guide.html)를 함께 보면 좋습니다.
값 비교 기준은 [Node.js assert.deepStrictEqual 가이드](/development/blog/seo/2026/06/26/nodejs-assert-deepstrictequal-object-comparison-test-guide.html)와도 연결됩니다.

### H3. 실패 케이스를 성공 케이스만큼 명시한다

CLI 파서는 잘못된 입력에서 더 중요해집니다.
알 수 없는 옵션, 누락된 값, 허용되지 않은 환경, 숫자가 아닌 limit을 테스트해야 합니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { parseDeployOptions } from './deploy-options.js';

test('rejects invalid limit', () => {
  assert.throws(
    () => parseDeployOptions(['api', '--limit', 'many']),
    /limit must be a positive integer/
  );
});

test('rejects unsupported env', () => {
  assert.throws(
    () => parseDeployOptions(['api', '--env', 'local']),
    /env must be staging or production/
  );
});
```

실패 테스트가 있으면 나중에 옵션을 추가할 때 기존 안전장치가 깨졌는지 바로 알 수 있습니다.
특히 자동 배포, 데이터 정리, 일괄 변경 스크립트는 성공 경로보다 실패 경로가 더 중요할 때가 많습니다.

## 로그와 민감정보 처리

### H3. 파싱 결과를 그대로 출력하지 않는다

디버깅 중에는 `console.log(values)`가 편합니다.
하지만 CLI 옵션에는 토큰, 파일 경로, 이메일, 내부 URL 같은 민감한 값이 섞일 수 있습니다.
운영 로그에 파싱 결과 전체를 그대로 남기는 습관은 피해야 합니다.

```js
function summarizeOptions(options) {
  return {
    service: options.service,
    env: options.env,
    dryRun: options.dryRun,
    verbose: options.verbose
  };
}

console.log('deploy options:', summarizeOptions(options));
```

이처럼 실행 판단에 필요한 값만 남기면 재현 가능성과 보안을 함께 지킬 수 있습니다.
토큰이나 비밀번호가 필요하다면 CLI 옵션보다 환경변수나 시크릿 저장소를 우선 검토하고, 출력에서는 항상 마스킹해야 합니다.

### H3. 도움말에도 내부 정보를 과하게 담지 않는다

간단한 CLI에는 직접 도움말 문자열을 둘 수 있습니다.

```js
const HELP = `
Usage:
  node deploy.js <service> --env <staging|production> [--dry-run] [--limit <n>]

Options:
  -e, --env       Target environment
  -l, --limit     Maximum number of items to process
  --dry-run       Print planned actions without applying them
  -v, --verbose   Print additional diagnostic logs
`;
```

도움말은 사용자가 명령을 고칠 수 있을 만큼만 구체적이면 됩니다.
내부 host, 계정명, 실제 배포 URL, 토큰 예시를 넣으면 문서가 복사되는 과정에서 정보가 퍼질 수 있습니다.

## 적용 체크리스트

### H3. 작은 CLI를 만들 때 확인할 것

- 허용할 옵션과 타입을 `options`에 선언했는가?
- `--dry-run`처럼 하이픈이 있는 이름을 내부 camelCase로 정규화했는가?
- 숫자와 enum 값은 파싱 뒤 별도 검증하는가?
- 위치 인자는 허용 목록이나 안전한 경로 기준으로 검증하는가?
- 알 수 없는 옵션을 조용히 무시하지 않는가?
- 테스트에서 `process.argv` 대신 `args`를 주입하는가?
- 로그에 토큰, 원문 경로, 개인정보가 그대로 남지 않는가?

## FAQ

### H3. util.parseArgs만으로 큰 CLI를 만들어도 되나요?

가능은 하지만 항상 좋은 선택은 아닙니다.
옵션 몇 개와 위치 인자 정도라면 충분합니다.
반대로 서브커맨드가 많고, 도움말 생성, 자동 완성, 프롬프트, 플러그인 구조가 필요하다면 전용 CLI 프레임워크가 유지보수에 더 적합합니다.

### H3. 옵션 값은 자동으로 숫자로 변환되나요?

아니요.
`type: 'string'`으로 받은 뒤 직접 `Number()` 변환과 범위 검증을 하는 편이 안전합니다.
그래야 `0`, 음수, 소수, 빈 문자열 같은 입력을 도메인 규칙에 맞게 처리할 수 있습니다.

### H3. strict false는 언제 쓰나요?

현재 CLI가 외부 명령을 감싸는 래퍼이고, 모르는 옵션을 그대로 넘겨야 할 때 제한적으로 씁니다.
일반적인 운영 스크립트에서는 strict 기본값을 유지해 오타와 폐기된 옵션을 빠르게 실패시키는 편이 좋습니다.

## 마무리

`util.parseArgs()`는 작은 Node.js CLI의 입력 처리를 단순하게 만들어 줍니다.
옵션 선언, 위치 인자 분리, strict 검증을 기본 모듈 안에서 해결할 수 있어 운영 스크립트와 자동화 도구에 잘 맞습니다.

핵심은 파싱과 업무 규칙을 분리하는 것입니다.
`parseArgs()`로 입력을 구조화하고, 숫자·환경·위험 작업 여부는 별도 함수에서 검증하면 테스트 가능한 CLI를 만들 수 있습니다.
여기에 민감정보 로그 기준까지 더하면 의존성 없이도 신뢰할 수 있는 Node.js 자동화 도구를 운영할 수 있습니다.

## 관련 글

- [CLI 출력값 민감정보 제거 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)
- [Node.js loadEnvFile 가이드: 기본 모듈로 환경변수 관리하기](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)
- [Node.js path.resolve/normalize 가이드](/development/blog/seo/2026/08/06/nodejs-path-resolve-normalize-traversal-prevention-guide.html)
- [Node.js test runner describe/it 가이드](/development/blog/seo/2026/07/10/nodejs-test-runner-describe-it-suite-structure-guide.html)

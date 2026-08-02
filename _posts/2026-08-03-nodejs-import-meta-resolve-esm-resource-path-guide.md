---
layout: post
title: "Node.js import.meta.resolve 가이드: ESM에서 리소스 경로를 안전하게 찾는 법"
date: 2026-08-03 08:00:00 +0900
lang: ko
translation_key: nodejs-import-meta-resolve-esm-resource-path-guide
permalink: /development/blog/seo/2026/08/03/nodejs-import-meta-resolve-esm-resource-path-guide.html
alternates:
  ko: /development/blog/seo/2026/08/03/nodejs-import-meta-resolve-esm-resource-path-guide.html
  x_default: /development/blog/seo/2026/08/03/nodejs-import-meta-resolve-esm-resource-path-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, esm, import-meta, module-resolution, file-url, tooling, javascript]
description: "Node.js import.meta.resolve로 ESM에서 템플릿, 설정 파일, 패키지 리소스 위치를 찾을 때 알아야 할 URL 처리, fileURLToPath 변환, cwd 의존 제거, 보안상 주의점을 실무 예제로 정리합니다."
---

ESM으로 작성한 Node.js 코드에서 파일 경로를 다룰 때 가장 자주 생기는 실수는 기준점을 `process.cwd()`로만 잡는 것입니다.
개발 중에는 명령을 항상 프로젝트 루트에서 실행하니 문제가 없어 보이지만, 테스트 러너, 작업 큐, 배포 스크립트, 패키지 내부 실행처럼 시작 위치가 달라지면 같은 상대 경로가 전혀 다른 파일을 가리킬 수 있습니다.
템플릿, fixture, 설정 샘플, 패키지 내부 리소스처럼 모듈과 함께 배포되는 파일은 "현재 명령을 어디서 실행했는가"보다 "이 모듈 기준으로 어디에 있는가"가 더 중요합니다.

Node.js ESM에서는 `import.meta.resolve()`를 사용해 모듈 specifier를 현재 모듈 기준의 URL로 해석할 수 있습니다.
상대 경로뿐 아니라 패키지 export 규칙을 거치는 specifier도 다룰 수 있어서, 작은 CLI 도구나 라이브러리 내부 리소스 경로를 정리할 때 유용합니다.
다만 결과는 일반 파일 경로가 아니라 URL 문자열이라는 점을 먼저 이해해야 합니다.

이 글에서는 `import.meta.resolve()`로 ESM 리소스 경로를 안전하게 찾는 방법을 정리합니다.
파일 자체의 디렉터리 기준 경로가 필요하다면 [Node.js import.meta.dirname 가이드](/development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html)를 먼저 보면 좋고, 패키지 메타데이터를 찾아야 한다면 [Node.js module.findPackageJSON 가이드](/development/blog/seo/2026/06/04/nodejs-module-findpackagejson-package-metadata-guide.html)와 연결해 볼 수 있습니다.
경로가 실제 허용 루트 안에 있는지 검증해야 한다면 [Node.js realpath 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-realpath-symlink-guide.html)도 함께 적용하는 편이 안전합니다.

## import.meta.resolve가 필요한 상황

### H3. 모듈 옆 리소스를 명확하게 찾는다

CLI나 서버 코드가 HTML 템플릿, SQL 파일, JSON schema 같은 리소스를 함께 읽는 경우가 많습니다.
이때 `./templates/email.html`을 그대로 `readFile()`에 넘기면 실행 위치에 따라 기준이 달라질 수 있습니다.
ESM에서는 모듈 기준으로 상대 specifier를 해석한 뒤, 파일 URL을 실제 경로로 바꾸는 흐름이 더 명확합니다.

```js
import { readFile } from 'node:fs/promises';
import { fileURLToPath } from 'node:url';

const templateUrl = import.meta.resolve('./templates/welcome.html');
const templatePath = fileURLToPath(templateUrl);

export async function loadWelcomeTemplate() {
  return readFile(templatePath, 'utf8');
}
```

`import.meta.resolve('./templates/welcome.html')`의 기준은 이 코드를 담고 있는 모듈입니다.
따라서 프로세스를 어느 디렉터리에서 시작하든 같은 리소스를 가리킵니다.
빌드 스크립트나 테스트 도구처럼 실행 위치가 자주 바뀌는 코드에서 특히 도움이 됩니다.

### H3. process.cwd 의존을 줄인다

`process.cwd()`는 "프로세스를 시작한 현재 작업 디렉터리"입니다.
사용자가 명령을 실행한 위치를 기준으로 파일을 찾고 싶을 때는 맞는 선택입니다.
하지만 코드와 함께 배포된 리소스를 찾는 기준으로는 흔들릴 수 있습니다.

```js
import { readFile } from 'node:fs/promises';

// 실행 위치가 프로젝트 루트라는 가정이 숨어 있다.
const config = await readFile('./config/default.json', 'utf8');
```

이 코드는 로컬에서는 잘 동작하다가 CI, 테스트, 패키지 실행 환경에서 깨질 수 있습니다.
리소스가 모듈과 함께 이동하는 파일이라면 `import.meta.resolve()`나 `import.meta.dirname`을 기준으로 삼는 편이 더 예측 가능합니다.

```js
import { readFile } from 'node:fs/promises';
import { fileURLToPath } from 'node:url';

export async function readDefaultConfig() {
  const url = import.meta.resolve('./config/default.json');
  return JSON.parse(await readFile(fileURLToPath(url), 'utf8'));
}
```

반대로 사용자가 넘긴 입력 파일, 명령 실행 위치의 설정 파일, 현재 저장소 루트의 산출물처럼 cwd 자체가 의미 있는 경우도 있습니다.
중요한 것은 둘 중 하나만 항상 쓰는 것이 아니라, 파일의 소유자가 누구인지에 따라 기준을 나누는 것입니다.

## URL과 파일 경로를 구분하기

### H3. resolve 결과는 URL 문자열이다

`import.meta.resolve()`는 경로 문자열이 아니라 URL 문자열을 반환합니다.
상대 파일 specifier를 넘기면 보통 `file:///...` 형태가 됩니다.
이 값을 그대로 `path.join()`에 넣으면 URL과 파일 경로 문맥이 섞입니다.

```js
const resolved = import.meta.resolve('./fixtures/user.json');

console.log(resolved);
// file:///Users/me/project/src/fixtures/user.json
```

파일 시스템 API에 넘길 경로가 필요하면 `fileURLToPath()`로 변환하세요.

```js
import { fileURLToPath } from 'node:url';

const fixturePath = fileURLToPath(
  import.meta.resolve('./fixtures/user.json')
);
```

URL을 URL로 다룰 때는 변환하지 않아도 됩니다.
예를 들어 `new URL()`로 다시 조합하거나, 동적 `import()`에 넘기는 specifier를 만들 때는 URL 문맥을 유지하는 편이 자연스럽습니다.
파일 API를 호출하는 순간에만 경로로 바꾸는 기준을 두면 코드 리뷰에서 실수를 찾기 쉽습니다.

### H3. path.join과 new URL을 섞지 않는다

URL과 파일 경로를 한 함수 안에서 계속 오가면 작은 버그가 생기기 쉽습니다.
특히 Windows 경로, 공백, 한글 파일명, URL escape 문자가 섞이면 문자열 조합으로는 안전하게 처리하기 어렵습니다.
리소스 위치를 만들 때는 URL 문맥이나 path 문맥 중 하나를 선택하고, 경계에서만 변환하는 편이 좋습니다.

```js
import { fileURLToPath } from 'node:url';

export function resolveResourcePath(relativeSpecifier) {
  const url = import.meta.resolve(relativeSpecifier);
  return fileURLToPath(url);
}
```

이런 작은 헬퍼를 두면 호출부는 파일 경로만 받습니다.
반대로 URL이 필요한 호출부에는 URL 문자열이나 `URL` 객체를 그대로 돌려주는 별도 함수를 둡니다.
한 함수가 "파일 경로를 돌려주는지, URL을 돌려주는지"를 이름으로 드러내면 유지보수가 쉬워집니다.

## 패키지 리소스와 함께 쓰기

### H3. 패키지 export 규칙을 존중한다

라이브러리 내부에서 다른 패키지의 공개 리소스를 찾고 싶을 때도 `import.meta.resolve()`가 도움이 됩니다.
bare specifier를 넘기면 Node.js의 모듈 해석 규칙을 거쳐 URL을 얻습니다.
다만 패키지가 `exports`를 제한한다면 공개되지 않은 내부 파일은 해석되지 않을 수 있습니다.

```js
const packageEntry = import.meta.resolve('some-package');
```

이 동작은 장점이기도 합니다.
패키지 관리자가 공개 API로 약속한 진입점만 사용하게 만들기 때문입니다.
`node_modules/some-package/...`를 직접 문자열로 조합하면 당장은 동작할 수 있어도 패키지 구조가 바뀌는 순간 깨집니다.

패키지 내부 리소스를 안정적으로 제공해야 한다면 패키지 쪽에서 export 경로를 명시하는 편이 좋습니다.

```json
{
  "name": "example-toolkit",
  "type": "module",
  "exports": {
    ".": "./src/index.js",
    "./schema": "./schema/config.schema.json"
  }
}
```

사용자는 다음처럼 공개된 specifier만 해석하면 됩니다.

```js
import { readFile } from 'node:fs/promises';
import { fileURLToPath } from 'node:url';

const schemaPath = fileURLToPath(
  import.meta.resolve('example-toolkit/schema')
);

export async function loadSchema() {
  return JSON.parse(await readFile(schemaPath, 'utf8'));
}
```

이 방식은 패키지 내부 폴더 구조를 외부 코드에 노출하지 않습니다.
리소스 파일을 옮겨도 export 경로만 유지하면 호출부 변경을 줄일 수 있습니다.

### H3. 동적 import와 파일 읽기를 분리한다

`import.meta.resolve()`는 "어디에 있는가"를 확인하는 도구입니다.
그 파일을 JavaScript 모듈로 실행할지, 일반 텍스트로 읽을지는 별개의 결정입니다.
설정 JS 파일처럼 실행이 필요한 대상은 동적 `import()`가 맞고, JSON schema나 템플릿처럼 데이터로 읽을 대상은 `readFile()`이 맞습니다.

```js
const pluginUrl = import.meta.resolve('./plugins/default.js');
const pluginModule = await import(pluginUrl);
```

반대로 HTML 템플릿을 `import()`하려고 하면 빌드 설정이나 import attributes가 필요해질 수 있습니다.
단순 파일 내용이 목적이면 파일 시스템 API를 쓰는 편이 분명합니다.

```js
import { readFile } from 'node:fs/promises';
import { fileURLToPath } from 'node:url';

const emailTemplate = await readFile(
  fileURLToPath(import.meta.resolve('./templates/reset-password.html')),
  'utf8'
);
```

실무에서는 "실행할 코드"와 "읽을 데이터"를 강하게 나누는 편이 안전합니다.
사용자 입력으로 받은 specifier를 동적 `import()`에 그대로 넘기는 구조는 피하고, 허용 목록이나 고정된 매핑을 두는 것이 좋습니다.

## 보안과 운영 기준

### H3. 사용자 입력을 specifier로 그대로 쓰지 않는다

`import.meta.resolve()`가 모듈 해석을 해 준다고 해서 입력 검증까지 대신해 주는 것은 아닙니다.
사용자가 보낸 문자열을 그대로 specifier로 넘기면 의도하지 않은 파일이나 패키지를 찾으려는 시도가 생길 수 있습니다.
특히 플러그인, 테마, 템플릿 이름처럼 외부 입력에서 리소스를 고르는 기능은 허용 목록을 먼저 두는 편이 안전합니다.

```js
import { readFile } from 'node:fs/promises';
import { fileURLToPath } from 'node:url';

const templates = {
  welcome: './templates/welcome.html',
  resetPassword: './templates/reset-password.html'
};

export async function loadTemplate(name) {
  const specifier = templates[name];

  if (!specifier) {
    throw new Error('Unknown template');
  }

  const url = import.meta.resolve(specifier);
  return readFile(fileURLToPath(url), 'utf8');
}
```

오류 메시지에는 허용되지 않은 원본 입력을 그대로 남기지 않는 편이 좋습니다.
템플릿 이름에 이메일, 토큰, 내부 경로 같은 값이 섞여 들어왔을 때 로그 시스템으로 퍼질 수 있기 때문입니다.
로그에는 `unknown_template` 같은 이벤트 이름과 요청 ID 정도만 남겨도 진단에는 충분합니다.

### H3. 실제 접근 권한은 별도로 검증한다

specifier 해석은 경로 접근 제어가 아닙니다.
어떤 URL이 만들어졌다는 사실과 그 파일을 읽어도 된다는 사실은 다릅니다.
사용자 입력 파일, 작업 디렉터리 바깥 파일, 심볼릭 링크가 섞이는 도구라면 실제 경로를 확인하고 허용 루트 안에 있는지 검증해야 합니다.

```js
import path from 'node:path';
import { realpath } from 'node:fs/promises';

export async function assertInsideRoot(rootDir, filePath) {
  const root = await realpath(rootDir);
  const target = await realpath(filePath);
  const relative = path.relative(root, target);

  if (relative.startsWith('..') || path.isAbsolute(relative)) {
    throw new Error('Resolved file is outside the allowed root');
  }

  return target;
}
```

라이브러리 내부 고정 리소스라면 이 검증이 과할 수 있습니다.
하지만 파일 선택권이 사용자에게 있거나, 플러그인 이름으로 리소스를 고르거나, 생성된 파일을 다시 읽는 자동화라면 루트 검증이 필요합니다.
`import.meta.resolve()`는 기준점을 안정화하고, `realpath()` 검증은 접근 범위를 확인하는 역할로 나누면 됩니다.

## 테스트 기준

### H3. cwd를 바꿔도 같은 파일을 읽는지 확인한다

이 글의 핵심 리스크는 실행 위치 의존입니다.
따라서 테스트에서도 `process.chdir()`로 cwd를 바꾼 뒤 리소스 로딩 함수가 같은 결과를 내는지 확인하면 좋습니다.

```js
import assert from 'node:assert/strict';
import { mkdtemp, rm } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import path from 'node:path';
import test from 'node:test';
import { loadWelcomeTemplate } from '../src/templates.js';

test('loadWelcomeTemplate is independent from cwd', async () => {
  const originalCwd = process.cwd();
  const tempDir = await mkdtemp(path.join(tmpdir(), 'cwd-test-'));

  try {
    process.chdir(tempDir);

    const template = await loadWelcomeTemplate();
    assert.match(template, /Welcome/);
  } finally {
    process.chdir(originalCwd);
    await rm(tempDir, { recursive: true, force: true });
  }
});
```

`process.chdir()`는 전역 상태를 바꾸므로 테스트 뒤 반드시 되돌려야 합니다.
테스트 파일을 병렬로 실행하는 프로젝트라면 cwd를 바꾸는 테스트는 별도 파일로 분리하거나 병렬 실행 영향을 줄이는 구성이 필요합니다.
이 기준은 [Node.js test runner randomize seed 가이드](/development/blog/seo/2026/07/27/nodejs-test-runner-randomize-seed-order-dependent-guide.html)의 전역 상태 관리 원칙과도 연결됩니다.

### H3. URL과 path 반환값을 따로 테스트한다

헬퍼 함수가 URL을 반환하는지 파일 경로를 반환하는지 명확히 테스트해 두면 리팩터링 중 실수를 줄일 수 있습니다.
예를 들어 `resolveTemplateUrl()`은 `file:` URL을 반환하고, `resolveTemplatePath()`는 일반 파일 경로를 반환하게 나누는 식입니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import {
  resolveTemplatePath,
  resolveTemplateUrl
} from '../src/resource-paths.js';

test('template helpers expose explicit URL and path forms', () => {
  assert.equal(new URL(resolveTemplateUrl('welcome')).protocol, 'file:');
  assert.ok(resolveTemplatePath('welcome').endsWith('welcome.html'));
});
```

이 테스트는 복잡하지 않지만 팀의 약속을 고정합니다.
파일 API 호출부에는 path 헬퍼만 쓰고, 동적 import나 URL 조합에는 URL 헬퍼만 쓰는 식으로 경계를 유지할 수 있습니다.

## 실무 체크리스트

`import.meta.resolve()`를 도입할 때는 다음 기준을 확인하세요.

- 모듈과 함께 배포되는 리소스는 cwd가 아니라 모듈 기준으로 찾는가?
- `import.meta.resolve()` 결과가 URL 문자열이라는 점을 코드에서 분명히 드러냈는가?
- 파일 시스템 API에 넘기기 전에 `fileURLToPath()`로 변환했는가?
- 사용자 입력을 specifier로 직접 넘기지 않고 허용 목록을 거치는가?
- 패키지 내부 파일을 직접 `node_modules` 경로로 조합하지 않는가?
- 실행할 코드와 데이터로 읽을 파일을 분리했는가?
- cwd가 달라져도 같은 리소스를 읽는 테스트가 있는가?
- 오류 로그에 토큰, 개인 정보, 내부 전체 경로가 그대로 남지 않는가?

이 체크리스트를 통과하면 ESM 전환 과정에서 자주 생기는 경로 흔들림을 꽤 줄일 수 있습니다.
특히 자동화 스크립트, 테스트 도구, 패키지로 배포되는 CLI에서는 리소스 기준점을 명확히 하는 것만으로도 재현성이 좋아집니다.

## FAQ

### H3. import.meta.resolve와 import.meta.dirname은 언제 다르게 쓰나요?

`import.meta.dirname`은 현재 모듈의 디렉터리 경로가 필요할 때 편합니다.
그 디렉터리를 기준으로 여러 파일 경로를 직접 조합해야 한다면 좋은 선택입니다.
반면 `import.meta.resolve()`는 특정 specifier를 Node.js 모듈 해석 규칙에 따라 URL로 바꾸고 싶을 때 적합합니다.
상대 리소스 하나를 찾거나 패키지 export 경로를 해석할 때는 `import.meta.resolve()`가 더 의도를 잘 드러냅니다.

### H3. import.meta.resolve 결과를 readFile에 바로 넘겨도 되나요?

파일 URL 문자열을 그대로 받는 API도 있지만, 팀 코드에서는 `fileURLToPath()`로 명시적으로 바꾸는 편을 권합니다.
호출부가 파일 경로를 다루는지 URL을 다루는지 선명해지고, `path` 모듈과 섞을 때의 실수를 줄일 수 있습니다.
URL이 필요한 흐름이라면 끝까지 URL로 유지하고, 파일 시스템 경로가 필요한 흐름이라면 경계에서 변환하세요.

### H3. process.cwd는 이제 쓰지 않아야 하나요?

아닙니다.
사용자가 명령을 실행한 위치를 기준으로 파일을 찾는 CLI에서는 `process.cwd()`가 자연스럽습니다.
예를 들어 현재 프로젝트의 `package.json`, 현재 폴더의 설정 파일, 사용자가 지정한 상대 입력 경로를 처리할 때는 cwd가 의미 있는 기준입니다.
모듈과 함께 배포되는 고정 리소스는 `import.meta.resolve()`나 `import.meta.dirname`을 쓰고, 사용자 작업 위치가 의미 있는 파일은 cwd를 쓰는 식으로 나누면 됩니다.

## 마무리

`import.meta.resolve()`는 ESM 코드에서 리소스 위치를 모듈 기준으로 해석하게 해 주는 작은 도구입니다.
핵심은 URL과 파일 경로를 섞지 않고, 실행 위치에 의존하지 않으며, 사용자 입력을 그대로 specifier로 넘기지 않는 것입니다.

ESM 기반 도구를 만들고 있다면 먼저 템플릿, fixture, schema, 기본 설정 파일부터 점검해 보세요.
상대 경로가 cwd에 기대고 있다면 `import.meta.resolve()`로 기준점을 옮기는 것만으로도 테스트와 배포 환경의 차이를 줄일 수 있습니다.

## 참고 자료

- [Node.js ECMAScript modules documentation](https://nodejs.org/api/esm.html)
- [MDN import.meta reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta)

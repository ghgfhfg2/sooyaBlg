---
layout: post
title: "Node.js watch 모드 가이드: 파일 변경 시 개발 서버를 자동 재시작하는 법"
date: 2026-05-13 08:00:00 +0900
lang: ko
translation_key: nodejs-watch-mode-auto-restart-guide
permalink: /development/blog/seo/2026/05/13/nodejs-watch-mode-auto-restart-guide.html
alternates:
  ko: /development/blog/seo/2026/05/13/nodejs-watch-mode-auto-restart-guide.html
  x_default: /development/blog/seo/2026/05/13/nodejs-watch-mode-auto-restart-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, watch, nodemon, dx, backend]
description: "Node.js 내장 watch 모드로 파일 변경 시 개발 서버와 스크립트를 자동 재시작하는 방법을 정리했습니다. nodemon과의 차이, --watch-path 사용법, 환경 변수와 테스트 실행 팁까지 실무 예제로 설명합니다."
---

Node.js 개발 서버를 고칠 때마다 터미널에서 프로세스를 끄고 다시 실행하는 일은 생각보다 흐름을 많이 끊습니다.
예전에는 이런 자동 재시작을 위해 `nodemon`, `tsx watch`, 프레임워크별 dev server를 따로 붙이는 경우가 많았습니다.
하지만 단순한 Node.js 서버나 CLI 스크립트라면 이제 **Node.js 내장 watch 모드**만으로도 충분한 경우가 많습니다.

`node --watch`는 지정한 엔트리 파일과 의존 파일이 바뀌었을 때 프로세스를 자동으로 재시작합니다.
외부 패키지를 추가하지 않아도 되기 때문에 작은 API 서버, 배치 스크립트, 학습용 예제, 운영 전 검증용 도구에서 특히 가볍게 쓸 수 있습니다.
이 글에서는 Node.js watch 모드의 기본 사용법, `nodemon`과의 선택 기준, `--watch-path`로 감시 범위를 좁히는 방법, 테스트·환경 변수와 함께 쓰는 실무 패턴을 정리합니다.
프로세스 종료 처리까지 함께 설계해야 한다면 [Node.js graceful shutdown 가이드: Kubernetes 무중단 종료를 위한 SIGTERM 처리](/development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html)도 같이 참고하면 좋습니다.

## Node.js watch 모드가 필요한 이유

### H3. 개발 중 재시작 반복을 줄인다

개발 중에는 같은 작업이 계속 반복됩니다.
라우터를 수정하고, 서버를 재시작하고, 요청을 다시 보내고, 로그를 확인합니다.
이때 재시작이 수동이면 작은 변경마다 집중이 끊깁니다.

기본 실행은 다음과 같습니다.

```bash
node server.js
```

watch 모드를 켜면 파일 변경 후 자동으로 다시 실행됩니다.

```bash
node --watch server.js
```

`server.js`가 import하는 파일이 바뀌어도 Node.js가 변경을 감지해 프로세스를 재시작합니다.
별도 의존성이 없고 명령이 짧기 때문에 프로젝트 초기 단계에서 바로 적용하기 좋습니다.

예를 들어 다음처럼 작은 HTTP 서버를 만들 수 있습니다.

```js
// server.js
import http from 'node:http';

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'content-type': 'application/json; charset=utf-8' });
  res.end(JSON.stringify({ ok: true, path: req.url }));
});

server.listen(3000, () => {
  console.log('dev server listening on http://localhost:3000');
});
```

실행은 이렇게 합니다.

```bash
node --watch server.js
```

이제 응답 메시지를 바꾸고 저장하면 Node.js가 자동으로 서버 프로세스를 다시 띄웁니다.
개발 서버가 단순할수록 이 방식은 `nodemon` 설정 파일 없이도 충분히 편합니다.

### H3. 의존성 없는 개발 환경을 만들 수 있다

작은 저장소에서 자동 재시작 하나만을 위해 개발 의존성을 추가하는 것은 부담이 될 수 있습니다.
팀원이 새로 clone했을 때 `npm install` 전에 간단한 스크립트를 돌려보고 싶을 수도 있습니다.

Node.js 내장 watch 모드는 이런 상황에 잘 맞습니다.

- 표준 Node.js만 설치되어 있으면 바로 실행한다.
- 설정 파일 없이 시작할 수 있다.
- 서버, CLI, 배치 스크립트 모두 같은 방식으로 실행한다.
- CI가 아니라 로컬 개발 편의 기능으로 분리하기 쉽다.

`package.json`에는 다음처럼 넣어둘 수 있습니다.

```json
{
  "scripts": {
    "dev": "node --watch server.js",
    "start": "node server.js"
  }
}
```

`dev`는 로컬 개발용, `start`는 일반 실행용으로 분리합니다.
운영 환경에서 watch 모드를 켜는 것은 권장하지 않습니다.
운영에서는 프로세스 매니저, 컨테이너 오케스트레이션, 헬스체크, 종료 신호 처리가 더 중요합니다.
배포 안정성 관점이 필요하다면 [readiness/liveness probe 가이드: 배포 신뢰성을 높이는 헬스체크 설계](/development/blog/seo/2026/03/10/readiness-liveness-probe-deploy-reliability-guide.html)를 함께 보면 좋습니다.

## 기본 사용법: node --watch로 자동 재시작하기

### H3. 엔트리 파일을 watch 모드로 실행한다

가장 단순한 형태는 `node --watch <entry>`입니다.

```bash
node --watch src/index.js
```

ESM 프로젝트라면 그대로 import를 사용할 수 있습니다.

```js
// src/index.js
import { createServer } from 'node:http';
import { getMessage } from './message.js';

const server = createServer((req, res) => {
  res.end(getMessage(req.url));
});

server.listen(3000, () => {
  console.log('listening on 3000');
});
```

```js
// src/message.js
export function getMessage(path) {
  return `hello from ${path}`;
}
```

`message.js`를 수정하고 저장하면 실행 중인 프로세스가 재시작됩니다.
개발자는 서버 종료와 재실행을 직접 반복하지 않아도 됩니다.

다만 watch 모드는 “프로세스가 재시작된다”는 점을 기억해야 합니다.
메모리에 있던 상태, 임시 캐시, 연결된 클라이언트는 재시작 과정에서 사라집니다.
따라서 개발 중 상태 보존이 필요한 도구라면 파일 기반 저장소나 테스트 fixture를 따로 준비하는 편이 좋습니다.

### H3. 화면 지우기를 원하지 않으면 --watch-preserve-output을 쓴다

watch 모드는 재시작 시 터미널 출력을 정리할 수 있습니다.
이전 로그까지 계속 보고 싶다면 `--watch-preserve-output`을 함께 사용할 수 있습니다.

```bash
node --watch --watch-preserve-output src/index.js
```

이 옵션은 다음 상황에서 유용합니다.

- 변경 전후 로그를 비교해야 한다.
- 초기화 순서를 계속 추적해야 한다.
- 에러가 재시작 직전에 찍혀 사라지는 것을 피하고 싶다.
- 테스트성 서버의 요청 로그를 누적해서 보고 싶다.

로그가 너무 많이 쌓이면 오히려 디버깅이 어려워질 수 있습니다.
그때는 `console.clear()`를 직접 넣기보다 로그 레벨을 조정하거나 `--watch-preserve-output`을 끄는 편이 낫습니다.
로그에는 토큰, 쿠키, 개인정보를 그대로 남기지 않아야 합니다.
민감한 출력 예시를 다룰 때는 [CLI 출력 정리 가이드: 블로그 예제에서 토큰과 개인정보를 안전하게 마스킹하기](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)를 참고할 수 있습니다.

## watch 범위 제어: --watch-path 사용법

### H3. 특정 디렉터리만 감시한다

기본 watch 모드는 실행 파일과 의존성 그래프를 중심으로 변경을 감지합니다.
하지만 프로젝트에 설정 파일, 템플릿, JSON fixture처럼 import되지 않는 파일이 있을 수 있습니다.
이때는 `--watch-path`로 감시 대상을 명시할 수 있습니다.

```bash
node --watch --watch-path=src --watch-path=config src/index.js
```

위 명령은 `src`와 `config` 경로 변경을 감시합니다.
설정 파일을 직접 `fs.readFile()`로 읽는 서버라면 이런 방식이 도움이 됩니다.

예를 들어 `config/app.json`을 읽는 서버가 있다고 가정합니다.

```js
import { readFile } from 'node:fs/promises';
import { createServer } from 'node:http';

async function loadConfig() {
  const raw = await readFile(new URL('../config/app.json', import.meta.url), 'utf8');
  return JSON.parse(raw);
}

const server = createServer(async (req, res) => {
  const config = await loadConfig();
  res.end(`service=${config.serviceName}`);
});

server.listen(3000);
```

`config/app.json`은 정적 import가 아니기 때문에 상황에 따라 변경 감지가 기대와 다를 수 있습니다.
이럴 때 `--watch-path=config`를 명시하면 의도를 더 분명하게 만들 수 있습니다.

파일 경로를 ESM 기준으로 안전하게 다루고 싶다면 [Node.js import.meta.dirname 가이드: ESM에서 __dirname 없이 경로 다루기](/development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html)를 함께 참고하세요.

### H3. 너무 넓은 감시는 피한다

프로젝트 루트 전체를 감시하면 편해 보이지만, 변경이 잦은 디렉터리까지 포함되어 재시작이 과하게 발생할 수 있습니다.
예를 들어 다음 경로는 watch 대상에서 빼거나 감시 범위를 좁히는 편이 좋습니다.

- `node_modules`
- 빌드 산출물 디렉터리
- 로그 파일 디렉터리
- 업로드 임시 파일
- 테스트 coverage 결과
- 데이터베이스 dump 또는 fixture 생성 결과

감시 범위가 너무 넓으면 파일 하나를 저장할 때 여러 번 재시작되거나, 빌드 산출물이 다시 watch를 트리거하는 루프가 생길 수 있습니다.
따라서 먼저 `src`, `config`, `scripts`처럼 실제 입력 파일이 있는 경로만 지정하는 것이 안전합니다.

```json
{
  "scripts": {
    "dev": "node --watch --watch-path=src --watch-path=config src/index.js"
  }
}
```

watch 모드는 개발 경험을 좋게 만드는 기능이지, 변경 감지 시스템 전체를 대체하는 기능은 아닙니다.
복잡한 번들링, TypeScript 변환, 프론트엔드 HMR이 필요하다면 각 도구의 dev server를 쓰는 편이 맞습니다.

## nodemon과 Node.js watch 모드 선택 기준

### H3. 단순한 서버와 스크립트는 내장 watch가 적합하다

Node.js 내장 watch 모드는 “일단 자동 재시작이 필요하다”는 요구에 가장 빠르게 답합니다.
특히 다음 조건이면 내장 기능부터 쓰는 것이 좋습니다.

- Node.js 파일을 직접 실행한다.
- TypeScript 트랜스파일 단계가 없다.
- 변경 후 전체 프로세스 재시작이면 충분하다.
- 복잡한 ignore/include 규칙이 필요 없다.
- 팀에 별도 도구를 추가하고 싶지 않다.

```bash
node --watch scripts/sync-cache.js
```

이런 간단한 스크립트는 `nodemon` 설정을 만드는 것보다 내장 watch가 더 명확합니다.
배치나 동기화 스크립트를 만들 때 취소 신호도 함께 고려한다면 [Node.js AbortController timeout 가이드: 요청 취소와 데드라인 처리 패턴](/development/blog/seo/2026/03/19/nodejs-abortcontroller-timeout-cancellation-pattern-guide.html)가 도움이 됩니다.

### H3. 복잡한 개발 워크플로는 nodemon이 여전히 편하다

반대로 다음 조건이라면 `nodemon` 같은 도구가 더 나을 수 있습니다.

- TypeScript 실행기와 함께 써야 한다.
- 확장자별 ignore/include 규칙이 많다.
- 재시작 전에 빌드, lint, codegen을 실행해야 한다.
- 특정 파일 변경에는 재시작하지 않아야 한다.
- 레거시 Node.js 버전을 지원해야 한다.

예를 들어 `src/**/*.ts` 변경 시 `tsx`로 실행하고, `generated` 디렉터리는 제외해야 한다면 전용 도구의 설정 능력이 더 편합니다.
내장 watch는 단순함이 장점이고, 복잡한 워크플로에서는 그 단순함이 한계가 됩니다.

따라서 선택 기준은 이렇게 잡으면 됩니다.

- 먼저 `node --watch`로 충분한지 확인한다.
- 감시 제외, 변환, 사전 작업이 늘어나면 `nodemon`이나 프레임워크 dev server로 옮긴다.
- 운영 실행 명령과 개발 실행 명령은 반드시 분리한다.

## 테스트와 함께 쓰는 패턴

### H3. node --test --watch로 테스트를 반복 실행한다

Node.js 내장 test runner도 watch 모드와 함께 사용할 수 있습니다.
테스트 파일을 고치면서 빠르게 피드백을 받고 싶다면 다음처럼 실행합니다.

```bash
node --test --watch
```

`package.json`에는 이렇게 둘 수 있습니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:watch": "node --test --watch"
  }
}
```

테스트 watch는 서버 watch와 목적이 다릅니다.
서버 watch는 앱을 다시 띄우고, 테스트 watch는 변경된 코드가 기대대로 동작하는지 반복 검증합니다.
새 기능을 만들 때는 두 터미널을 나눠 쓰는 방식도 좋습니다.

```bash
npm run dev
npm run test:watch
```

Node.js 내장 테스트 러너의 기본 구조가 필요하다면 [Node.js test runner 가이드: node:test로 기본 테스트를 빠르게 시작하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)를 먼저 읽어도 좋습니다.
커버리지까지 보고 싶다면 [Node.js test runner coverage 가이드: V8 커버리지로 테스트 빈틈 찾기](/development/blog/seo/2026/05/10/nodejs-test-runner-coverage-v8-guide.html)로 이어서 확장할 수 있습니다.

### H3. watch 환경에서는 부작용을 작게 만든다

watch 모드는 파일을 저장할 때마다 코드를 다시 실행합니다.
따라서 시작 시점에 외부 API 호출, 이메일 발송, 실제 결제, 운영 DB 변경 같은 부작용이 있으면 위험합니다.
개발용 실행에서는 다음 원칙을 지키는 편이 안전합니다.

- 기본값은 dry-run 또는 mock으로 둔다.
- 운영 API 키는 로컬 watch 명령에서 요구하지 않는다.
- 시작 시 자동 실행되는 destructive 작업을 넣지 않는다.
- 로그에는 토큰과 개인정보를 출력하지 않는다.
- `.env.example`로 필요한 변수 이름만 공유한다.

```js
const dryRun = process.env.DRY_RUN !== 'false';

if (dryRun) {
  console.log('dry-run mode: external write is disabled');
}
```

환경 변수 파일을 다룬다면 [Node.js loadEnvFile 가이드: 내장 dotenv로 환경 변수를 안전하게 불러오기](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)처럼 개발용 값과 운영용 값을 분리하세요.
watch는 편하지만 저장할 때마다 다시 실행된다는 특성 때문에, 부작용이 큰 코드는 더 보수적으로 설계해야 합니다.

## 실무 체크리스트

### H3. package.json 스크립트를 명확히 나눈다

팀 프로젝트에서는 실행 명령 이름만 봐도 목적이 드러나야 합니다.
다음처럼 분리하면 실수 가능성이 줄어듭니다.

```json
{
  "scripts": {
    "dev": "node --watch --watch-path=src --watch-path=config src/index.js",
    "start": "node src/index.js",
    "test": "node --test",
    "test:watch": "node --test --watch"
  }
}
```

`dev`는 로컬 자동 재시작, `start`는 일반 실행, `test:watch`는 테스트 반복 실행입니다.
운영 배포 스크립트에서 `npm run dev`를 호출하지 않도록 CI 설정도 분리해야 합니다.

### H3. 재시작 가능한 초기화 코드를 작성한다

watch 모드에서는 프로세스가 자주 시작되고 종료됩니다.
따라서 초기화 코드는 여러 번 실행되어도 안전해야 합니다.

- 포트 충돌이 나면 명확히 실패한다.
- 임시 파일은 시작 시 무조건 덮어쓰지 않는다.
- 로컬 DB migration은 자동 실행보다 명시 명령으로 분리한다.
- 캐시 warm-up 실패가 개발 서버 전체를 막지 않게 한다.
- 종료 신호 처리에서 서버와 연결을 정리한다.

```js
import { createServer } from 'node:http';

const server = createServer((req, res) => {
  res.end('ok');
});

server.listen(3000, () => {
  console.log('listening on 3000');
});

process.on('SIGTERM', () => {
  server.close(() => {
    console.log('server closed');
  });
});
```

개발 환경이라도 종료 처리를 넣어두면 운영 코드로 확장할 때 안정성이 좋아집니다.
watch 모드는 작은 개발 편의 기능이지만, 재시작 가능한 구조를 자연스럽게 훈련하게 해줍니다.

## FAQ

### H3. Node.js watch 모드는 운영 환경에서 써도 되나요?

권장하지 않습니다.
운영에서는 파일 변경 자동 재시작보다 프로세스 매니저, 컨테이너 재시작 정책, 헬스체크, 로그 수집, graceful shutdown이 더 중요합니다.
`node --watch`는 로컬 개발과 빠른 검증용으로 두는 것이 안전합니다.

### H3. nodemon을 완전히 대체할 수 있나요?

단순한 Node.js 서버와 스크립트라면 대체할 수 있습니다.
하지만 TypeScript 변환, 복잡한 ignore 규칙, 사전 빌드 작업, 레거시 런타임 지원이 필요하면 `nodemon`이나 프레임워크별 dev server가 더 적합합니다.

### H3. 변경할 때마다 두 번 이상 재시작되는 이유는 무엇인가요?

감시 범위가 너무 넓거나, 빌드 산출물·로그 파일·coverage 파일이 watch 대상에 포함됐을 가능성이 큽니다.
`--watch-path=src --watch-path=config`처럼 실제 입력 파일이 있는 경로만 감시하도록 좁혀 보세요.

### H3. 테스트도 watch 모드로 돌릴 수 있나요?

네.
Node.js 내장 test runner는 `node --test --watch`로 반복 실행할 수 있습니다.
서버 자동 재시작과 테스트 반복 실행은 목적이 다르므로, 필요하면 터미널을 분리해 `npm run dev`와 `npm run test:watch`를 함께 사용하는 방식이 좋습니다.

## 마무리

Node.js watch 모드는 개발 중 반복되는 재시작 비용을 줄이는 가장 가벼운 방법입니다.
외부 의존성 없이 `node --watch src/index.js`만으로 시작할 수 있고, 필요하면 `--watch-path`와 `--watch-preserve-output`으로 감시 범위와 로그 동작을 조정할 수 있습니다.

실무에서는 먼저 내장 watch로 충분한지 확인한 뒤, TypeScript 변환이나 복잡한 재시작 규칙이 필요해질 때 전용 도구로 넘어가는 흐름이 좋습니다.
그리고 watch 명령은 반드시 개발용으로만 두고, 운영 실행 명령과 분리하세요.
작은 자동화 하나라도 실행 범위와 부작용을 명확히 나누면 개발 경험과 배포 안정성을 함께 지킬 수 있습니다.

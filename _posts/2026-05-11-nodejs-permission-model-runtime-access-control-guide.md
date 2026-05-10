---
layout: post
title: "Node.js Permission Model 가이드: 런타임 권한으로 파일·프로세스 접근을 제한하는 법"
date: 2026-05-11 08:00:00 +0900
lang: ko
translation_key: nodejs-permission-model-runtime-access-control-guide
permalink: /development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html
alternates:
  ko: /development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html
  x_default: /development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, permission-model, security, runtime, backend]
description: "Node.js Permission Model로 파일 시스템, child_process, worker, addon 접근을 런타임에서 제한하는 방법을 정리했습니다. --permission 실행법, allow 옵션, 점진적 도입 전략, CI 점검까지 예제로 설명합니다."
---

Node.js 애플리케이션은 보통 실행 권한을 운영체제 사용자나 컨테이너 설정에 의존합니다.
하지만 실무에서는 한 프로세스 안에서도 더 좁은 경계가 필요할 때가 있습니다.
예를 들어 마크다운을 HTML로 변환하는 CLI가 프로젝트 전체 파일을 읽을 필요는 없고, 이미지 메타데이터를 처리하는 작업이 임의의 child process를 실행할 필요도 없습니다.

Node.js Permission Model은 이런 런타임 권한을 더 명시적으로 제한하기 위한 기능입니다.
`--permission`을 켜고 필요한 접근만 `--allow-*` 옵션으로 열어 두면, 의도하지 않은 파일 접근이나 프로세스 실행을 조기에 발견할 수 있습니다.
이 글에서는 파일 읽기·쓰기 제한, child process와 worker 허용, 점진적 도입 방법을 실무 흐름에 맞춰 정리합니다.
사용자 입력으로 경로를 만들거나 검증하는 코드가 함께 있다면 [Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)도 같이 보면 좋습니다.

## Node.js Permission Model이 필요한 이유

### H3. 런타임 권한은 마지막 방어선이 될 수 있다

권한 제어는 하나의 장치로 끝나지 않습니다.
컨테이너, OS 사용자, 시크릿 관리, 코드 리뷰, 의존성 점검이 모두 필요합니다.
그럼에도 애플리케이션 프로세스 내부에서 접근 범위를 줄여 두면 다음 상황에서 피해를 줄일 수 있습니다.

- 버그로 잘못된 파일 경로를 읽는 경우
- 플러그인이나 사용자 스크립트가 예상보다 넓은 파일에 접근하는 경우
- 빌드 도구가 임시 파일 외의 위치에 쓰기를 시도하는 경우
- 테스트 중 의도하지 않은 child process 실행이 섞이는 경우
- 서버리스나 배치 작업에서 최소 권한 원칙을 검증하고 싶은 경우

Permission Model은 보안 제품이 아니라 런타임 가드레일에 가깝습니다.
공격을 완전히 막는다고 기대하기보다, **필요한 권한을 코드 실행 명령에 문서화하고 벗어난 접근을 빠르게 실패시키는 도구**로 보는 것이 현실적입니다.

### H3. 의존성 많은 Node.js 프로젝트일수록 경계가 흐려진다

Node.js 프로젝트는 작은 라이브러리를 많이 조합하는 일이 흔합니다.
문제는 의존성이 늘어날수록 어떤 코드가 어느 파일을 읽고, 어떤 상황에서 외부 명령을 실행하는지 한눈에 보기 어려워진다는 점입니다.

예를 들어 문서 생성 스크립트가 다음 작업을 한다고 가정해 보겠습니다.

- `docs/` 디렉터리의 마크다운 읽기
- `dist/` 디렉터리에 HTML 쓰기
- 템플릿 파일 읽기
- 원격 API 호출 없이 로컬 변환만 수행

이 작업에는 홈 디렉터리, `.ssh`, 프로젝트 밖의 파일, child process 권한이 필요하지 않습니다.
Permission Model을 적용하면 실행 명령 자체가 “이 작업은 이 경로만 읽고 이 경로만 쓴다”는 운영 문서가 됩니다.

## 기본 실행법: --permission과 allow 옵션

### H3. 권한 시스템을 켜면 기본 접근이 제한된다

Node.js에서 Permission Model을 사용하려면 실행 시 `--permission` 옵션을 켭니다.
그다음 필요한 권한만 `--allow-*` 옵션으로 추가합니다.

```bash
node --permission app.js
```

권한을 주지 않은 상태에서 파일을 읽거나 쓰려는 코드는 실패합니다.
예를 들어 설정 파일만 읽어야 하는 앱이라면 다음처럼 시작할 수 있습니다.

```bash
node \
  --permission \
  --allow-fs-read=./config \
  app.js
```

여러 경로가 필요하면 쉼표로 나누거나 옵션을 반복하는 방식으로 명시합니다.
프로젝트 스크립트에서는 팀원이 그대로 실행할 수 있도록 `package.json`에 고정해 두는 편이 좋습니다.

```json
{
  "scripts": {
    "start:restricted": "node --permission --allow-fs-read=./config,./public app.js"
  }
}
```

운영체제와 셸 차이가 있는 팀이라면 긴 명령을 직접 넣기보다 Node.js 래퍼 스크립트나 배포 스크립트로 관리하는 편이 안전합니다.

### H3. 파일 읽기와 쓰기 권한을 분리한다

파일 시스템 권한은 읽기와 쓰기를 따로 생각해야 합니다.
템플릿과 입력 파일은 읽기 권한만 필요하고, 결과물 디렉터리는 쓰기 권한만 필요할 수 있습니다.

```bash
node \
  --permission \
  --allow-fs-read=./content,./templates \
  --allow-fs-write=./dist \
  scripts/build-site.js
```

이렇게 실행하면 빌드 스크립트가 `content/`와 `templates/`를 읽고 `dist/`에 결과를 쓰는 흐름이 명확해집니다.
반대로 실수로 프로젝트 루트 전체를 쓰거나, 홈 디렉터리의 파일을 읽으려 하면 권한 오류로 드러납니다.

경로가 동적으로 만들어지는 코드에서는 입력 검증도 함께 필요합니다.
정규식 기반 필터를 직접 만든다면 [Node.js RegExp.escape 가이드: 사용자 입력을 정규식에 안전하게 넣는 법](/development/blog/seo/2026/05/05/nodejs-regexp-escape-safe-dynamic-pattern-guide.html)처럼 사용자 입력을 그대로 패턴에 넣지 않는 습관도 중요합니다.

## 권한별 실무 사용 패턴

### H3. child_process는 기본적으로 닫아 둔다

`child_process`는 강력하지만 위험한 권한입니다.
외부 명령 실행이 꼭 필요하지 않은 웹 서버, API 핸들러, 데이터 변환 스크립트에서는 닫아 두는 편이 좋습니다.
필요한 경우에만 `--allow-child-process`를 명시합니다.

```bash
node \
  --permission \
  --allow-fs-read=./input \
  --allow-fs-write=./output \
  --allow-child-process \
  scripts/convert-with-tool.js
```

이 옵션을 추가했다면 코드 리뷰에서 다음 질문을 같이 확인해야 합니다.

- 실행하는 명령이 고정되어 있는가?
- 사용자 입력이 셸 문자열에 직접 합쳐지지 않는가?
- `shell: true`가 꼭 필요한가?
- 실패 시 stderr나 명령 인자를 로그에 과도하게 남기지 않는가?

Permission Model은 child process 사용 여부를 막거나 허용할 뿐, 명령 인자 주입까지 자동으로 해결하지는 않습니다.
명령 실행 코드 자체의 안전성은 별도로 검토해야 합니다.

### H3. worker와 addon 권한은 성능·확장 기능과 함께 본다

CPU 바운드 작업을 분리하려고 `worker_threads`를 쓰는 프로젝트는 `--allow-worker`가 필요할 수 있습니다.
예시는 다음과 같습니다.

```bash
node \
  --permission \
  --allow-worker \
  --allow-fs-read=./jobs \
  worker-runner.js
```

worker는 이벤트 루프 보호에 유용하지만, 권한 경계가 어떻게 적용되는지 팀 기준을 명확히 해야 합니다.
CPU 작업 분리 자체가 목적이라면 [Node.js worker_threads 가이드: CPU 바운드 작업으로 이벤트 루프 막지 않는 법](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)을 먼저 보고 구조를 단순하게 만든 뒤 권한을 붙이는 편이 좋습니다.

네이티브 addon을 사용하는 프로젝트는 `--allow-addons`가 필요할 수 있습니다.
이미지 처리, 암호화, 압축, 데이터베이스 드라이버처럼 네이티브 모듈을 포함한 의존성이 있다면 로컬에서 제한 모드로 한 번 실행해 보고 실제 필요한 권한을 확인해야 합니다.

## 코드에서 권한을 확인하는 방법

### H3. process.permission.has로 사전 점검한다

권한 오류를 사용자가 이해하기 쉬운 메시지로 바꾸고 싶다면 코드에서 권한을 확인할 수 있습니다.
Node.js는 `process.permission.has()`로 특정 권한이 있는지 확인하는 API를 제공합니다.

```js
const configPath = './config/app.json';

if (process.permission && !process.permission.has('fs.read', configPath)) {
  throw new Error(`config read permission is required: ${configPath}`);
}
```

이런 점검은 CLI나 배치 작업에서 특히 유용합니다.
권한이 없을 때 스택 트레이스만 보여 주는 대신, 사용자가 어떤 실행 옵션을 추가해야 하는지 안내할 수 있기 때문입니다.

```js
function assertCanWriteOutput(outputDir) {
  if (process.permission && !process.permission.has('fs.write', outputDir)) {
    throw new Error(
      `output directory write permission is required. ` +
      `Run with --allow-fs-write=${outputDir}`
    );
  }
}

assertCanWriteOutput('./dist');
```

단, 이 검사는 사용자 경험을 개선하기 위한 보조 장치입니다.
실제 권한 통제는 실행 옵션에서 이뤄져야 하며, 코드의 사전 검사만 믿고 운영 권한을 넓게 열어 두면 의미가 줄어듭니다.

### H3. 권한 오류를 테스트로 고정한다

제한 모드가 깨지지 않게 하려면 테스트나 CI에서 별도 명령을 실행하는 것이 좋습니다.
예를 들어 빌드 스크립트가 특정 디렉터리만 읽고 쓰는지 확인하려면 다음처럼 smoke test를 둘 수 있습니다.

```json
{
  "scripts": {
    "build": "node scripts/build-site.js",
    "build:restricted": "node --permission --allow-fs-read=./content,./templates --allow-fs-write=./dist scripts/build-site.js"
  }
}
```

CI에서는 일반 빌드와 제한 빌드를 모두 돌리거나, 최소한 제한 빌드를 야간 작업으로라도 실행할 수 있습니다.
테스트 범위를 함께 보고 싶다면 [Node.js test runner coverage 가이드: 내장 커버리지로 테스트 누락 찾는 법](/development/blog/seo/2026/05/10/nodejs-test-runner-coverage-v8-guide.html)처럼 변경된 코드가 실제로 실행됐는지도 같이 확인하면 좋습니다.

## 점진적 도입 전략

### H3. 웹 서버보다 짧은 스크립트부터 시작한다

처음부터 큰 웹 서버에 Permission Model을 적용하면 필요한 권한을 찾는 데 시간이 오래 걸릴 수 있습니다.
다음 순서로 좁은 작업부터 시작하는 편이 안전합니다.

1. 문서 생성, 코드 생성, 데이터 변환 같은 단일 목적 스크립트
2. 테스트 보조 스크립트와 릴리스 스크립트
3. CLI의 `--help`, `--version`, `lint` 같은 가벼운 명령
4. 서버리스 함수나 배치 작업
5. 장기 실행 API 서버

짧은 스크립트는 입력과 출력이 분명해서 권한 목록을 만들기 쉽습니다.
이 성공 사례를 쌓은 뒤 서버나 복잡한 CLI로 넓히면 팀 저항도 줄어듭니다.

### H3. 권한 목록을 문서가 아니라 실행 명령으로 관리한다

권한 정책을 README에만 적어 두면 실제 실행과 쉽게 어긋납니다.
가능하면 `package.json`, CI workflow, 배포 스크립트에 제한 모드 명령을 직접 넣어 두는 것이 좋습니다.

```json
{
  "scripts": {
    "docs:build": "node --permission --allow-fs-read=./docs,./templates --allow-fs-write=./site scripts/docs-build.js",
    "assets:resize": "node --permission --allow-fs-read=./assets/raw --allow-fs-write=./assets/public scripts/resize-assets.js"
  }
}
```

이 방식의 장점은 변경 리뷰가 쉬워진다는 점입니다.
누군가 `--allow-fs-read=.`처럼 권한을 넓히면 diff에서 바로 보입니다.
권한 변경이 필요한 이유를 PR 설명에 남기면 운영 지식도 함께 축적됩니다.

## 실무 체크리스트

### H3. 적용 전 확인할 것

Permission Model을 적용하기 전에는 다음 항목을 점검합니다.

- 현재 사용하는 Node.js 버전에서 `--permission`과 필요한 `--allow-*` 옵션을 지원하는가?
- 스크립트의 입력 경로와 출력 경로가 명확한가?
- 임시 파일, 캐시 디렉터리, 로그 디렉터리가 따로 필요한가?
- child process, worker, addon 사용 여부를 확인했는가?
- 제한 모드 실패를 CI에서 재현할 수 있는가?

Node.js 버전 차이는 옵션 지원과 동작 차이로 이어질 수 있습니다.
팀에서는 `.nvmrc`, `volta`, 컨테이너 이미지, CI 런타임 중 하나로 기준 버전을 맞추는 편이 좋습니다.
재현 가능한 설치 흐름이 필요하다면 [npm ci와 lockfile 가이드](/development/blog/seo/2026/03/06/npm-ci-lockfile-reproducible-build-guide.html)을 참고할 수 있습니다.

### H3. 운영에서 주의할 것

운영 환경에서는 권한을 너무 좁게 잡아 장애를 만들 수도 있고, 너무 넓게 잡아 의미를 잃을 수도 있습니다.
다음 기준으로 균형을 맞추는 것이 좋습니다.

- 처음에는 읽기·쓰기 경로를 명확히 아는 작업에만 적용한다.
- 권한 오류 메시지를 로그에 남기되, 민감한 전체 경로는 과도하게 노출하지 않는다.
- 배포 전 제한 모드 smoke test를 실행한다.
- 권한을 넓힐 때는 이유와 만료 가능성을 함께 기록한다.
- 보안 경계가 필요한 작업은 OS, 컨테이너, 네트워크 정책과 함께 설계한다.

Permission Model은 최소 권한 원칙을 코드 실행 단계로 끌어오는 좋은 출발점입니다.
다만 이 기능 하나만으로 샌드박스가 완성된다고 보기는 어렵습니다.
운영에서는 기존 보안 계층과 함께 쓰고, 개발 단계에서는 의도하지 않은 접근을 빨리 발견하는 도구로 활용하는 것이 가장 실용적입니다.

## FAQ

### H3. Node.js Permission Model을 켜면 모든 보안 문제가 해결되나요?

아닙니다.
Permission Model은 파일 시스템, child process, worker, addon 같은 런타임 접근을 제한하는 가드레일입니다.
인증, 권한 부여, 입력 검증, 의존성 취약점, 네트워크 정책을 대신하지 않습니다.
최소 권한 원칙을 보강하는 한 계층으로 보는 것이 안전합니다.

### H3. 기존 서버에 바로 적용해도 되나요?

가능은 하지만 추천 순서는 아닙니다.
큰 서버는 읽는 설정 파일, 로그 경로, 업로드 디렉터리, 네이티브 모듈, worker 사용 여부가 복잡할 수 있습니다.
먼저 빌드 스크립트나 배치 작업처럼 입력과 출력이 명확한 곳에서 권한 목록을 만들고, 실패 패턴을 익힌 뒤 서버로 확장하는 편이 좋습니다.

### H3. --allow-fs-read=.처럼 프로젝트 전체를 열어도 의미가 있나요?

상황에 따라 의미가 있습니다.
프로젝트 밖의 홈 디렉터리나 시스템 파일 접근을 막을 수 있기 때문입니다.
다만 더 좋은 목표는 `./config`, `./public`, `./content`처럼 실제 필요한 경로만 점점 좁히는 것입니다.
처음에는 넓게 시작하더라도 CI와 리뷰를 통해 권한을 줄이는 방향으로 운영하는 것이 좋습니다.

### H3. 권한 오류가 나면 어떻게 디버깅하나요?

먼저 오류가 난 작업이 정말 그 권한을 가져야 하는지 확인합니다.
필요한 접근이라면 실행 명령에 적절한 `--allow-*` 옵션을 추가하고, 불필요한 접근이라면 코드의 경로 계산이나 import 구조를 수정합니다.
권한을 넓히기 전에 “왜 이 파일을 읽는가”, “왜 외부 명령이 필요한가”를 확인하는 과정 자체가 Permission Model의 핵심 가치입니다.

## 마무리

Node.js Permission Model은 애플리케이션 실행 권한을 더 명확하게 만드는 실용적인 도구입니다.
`--permission`으로 기본 접근을 닫고, `--allow-fs-read`, `--allow-fs-write`, `--allow-child-process`, `--allow-worker` 같은 옵션으로 필요한 권한만 여는 방식은 작은 스크립트부터 큰 서비스까지 점진적으로 적용할 수 있습니다.

처음부터 완벽한 권한 목록을 만들 필요는 없습니다.
입력과 출력이 분명한 스크립트 하나를 제한 모드로 실행해 보고, CI에 smoke test를 추가하고, 권한 변경을 리뷰 가능한 diff로 남기는 것부터 시작하면 됩니다.
그 과정에서 런타임 접근 범위가 문서화되고, 의도하지 않은 파일 접근이나 프로세스 실행을 더 빨리 발견할 수 있습니다.

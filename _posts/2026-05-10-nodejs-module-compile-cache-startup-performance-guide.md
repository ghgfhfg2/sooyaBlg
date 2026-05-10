---
layout: post
title: "Node.js module compile cache 가이드: 서버리스 콜드 스타트와 CLI 시작 시간을 줄이는 법"
date: 2026-05-10 20:00:00 +0900
lang: ko
translation_key: nodejs-module-compile-cache-startup-performance-guide
permalink: /development/blog/seo/2026/05/10/nodejs-module-compile-cache-startup-performance-guide.html
alternates:
  ko: /development/blog/seo/2026/05/10/nodejs-module-compile-cache-startup-performance-guide.html
  x_default: /development/blog/seo/2026/05/10/nodejs-module-compile-cache-startup-performance-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, compile-cache, performance, serverless, cli]
description: "Node.js module compile cache로 서버리스 콜드 스타트와 CLI 시작 시간을 줄이는 방법을 정리했습니다. NODE_COMPILE_CACHE 설정, 측정 방법, CI·배포 환경 주의사항까지 예제로 설명합니다."
---

Node.js 애플리케이션의 성능 문제는 요청 처리 로직에서만 생기지 않습니다.
서버리스 함수, 배치 작업, CLI 도구처럼 프로세스가 자주 새로 뜨는 환경에서는 **시작 시간 자체**가 사용자 경험과 비용에 영향을 줍니다.
의존성이 많거나 모듈 로딩이 무거운 프로젝트라면 코드가 실제로 실행되기 전, 모듈을 읽고 파싱하고 컴파일하는 단계도 무시하기 어렵습니다.

Node.js의 module compile cache는 이 시작 비용을 줄이기 위한 기능입니다.
한 번 컴파일한 모듈 정보를 디스크 캐시에 저장해 두고, 다음 실행에서 재사용할 수 있게 도와줍니다.
이 글에서는 `NODE_COMPILE_CACHE`를 기준으로 기본 사용법, 측정 방법, 실무 적용 시 주의할 점을 정리합니다.
실행 시간 측정 기준이 먼저 필요하다면 [Node.js performance.now 가이드: 실행 시간을 정확하게 측정하는 법](/development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html)을 함께 보면 좋습니다.

## Node.js module compile cache가 필요한 이유

### H3. 콜드 스타트는 비즈니스 로직 전에 이미 시작된다

서버가 오래 떠 있는 전통적인 API 서버에서는 시작 시간이 몇백 밀리초 늘어도 크게 드러나지 않을 수 있습니다.
하지만 다음 환경에서는 이야기가 달라집니다.

- 서버리스 함수처럼 요청에 맞춰 프로세스가 자주 생성되는 환경
- 짧게 실행되고 종료되는 CLI 도구
- 크론 배치, 데이터 변환 스크립트, 마이그레이션 스크립트
- 테스트 실행처럼 반복적으로 Node.js 프로세스를 띄우는 워크플로

이런 작업은 실제 처리 시간이 짧을수록 시작 오버헤드 비중이 커집니다.
예를 들어 CLI가 200ms 안에 끝나는 작업인데 모듈 로딩에만 120ms가 든다면, 사용자는 기능보다 대기 시간을 먼저 느끼게 됩니다.

### H3. 모듈이 많을수록 파싱과 컴파일 비용이 누적된다

Node.js는 애플리케이션을 실행하면서 필요한 JavaScript 모듈을 읽고, 파싱하고, V8이 실행할 수 있는 형태로 준비합니다.
작은 스크립트에서는 거의 보이지 않지만, 다음 조건이 겹치면 비용이 누적됩니다.

- 진입점에서 많은 모듈을 즉시 import하는 구조
- CLI 실행마다 무거운 설정 파일과 플러그인을 읽는 구조
- 공통 유틸 패키지가 여러 계층으로 연결된 구조
- 서버리스 함수 번들에 사용하지 않는 코드가 많이 포함된 구조

module compile cache는 이 중 **반복 실행에서 발생하는 컴파일 비용**을 줄이는 데 초점을 둡니다.
네트워크 지연, 데이터베이스 연결, 비효율적인 초기화 코드까지 해결해 주는 만능 도구는 아니지만, 시작 시간을 줄이는 후보로 검토할 가치는 충분합니다.

## 기본 사용법: NODE_COMPILE_CACHE 설정

### H3. 캐시 디렉터리를 명시한다

가장 단순한 사용법은 실행 전에 `NODE_COMPILE_CACHE` 환경변수에 캐시 디렉터리를 지정하는 것입니다.

```bash
mkdir -p .node-compile-cache
NODE_COMPILE_CACHE=.node-compile-cache node app.js
```

CLI 도구라면 다음처럼 감쌀 수 있습니다.

```bash
NODE_COMPILE_CACHE=.node-compile-cache node ./bin/my-cli.js build
```

`package.json` 스크립트에는 운영체제 차이를 고려해 별도 래퍼 스크립트를 두는 편이 안전합니다.
간단한 macOS/Linux 전용 프로젝트라면 다음처럼 시작할 수 있습니다.

```json
{
  "scripts": {
    "start:cached": "NODE_COMPILE_CACHE=.node-compile-cache node app.js",
    "cli:cached": "NODE_COMPILE_CACHE=.node-compile-cache node ./bin/my-cli.js"
  }
}
```

Windows까지 고려한다면 `cross-env`를 쓰거나, Node.js로 작은 실행 래퍼를 만들어 환경변수를 설정하는 방식을 검토할 수 있습니다.

### H3. 캐시 위치는 실행 환경에 맞춰 정한다

캐시는 디스크에 저장됩니다.
따라서 캐시 디렉터리는 다음 조건을 만족해야 합니다.

- 프로세스가 쓸 수 있는 경로일 것
- 배포 산출물과 충돌하지 않을 것
- 여러 버전이 섞여도 문제가 없도록 관리할 수 있을 것
- 컨테이너나 서버리스 환경에서 재사용 가능성이 있을 것

로컬 개발에서는 프로젝트 내부 `.node-compile-cache`가 편합니다.
다만 Git에 커밋할 파일은 아니므로 `.gitignore`에 추가하는 편이 좋습니다.

```gitignore
.node-compile-cache/
```

서버 환경에서는 임시 디렉터리나 런타임별 캐시 경로를 사용할 수 있습니다.
중요한 점은 캐시를 공유한다고 해서 애플리케이션 상태를 공유하는 것은 아니라는 점입니다.
module compile cache는 실행 결과 캐시가 아니라, 모듈 컴파일을 재사용하기 위한 보조 데이터에 가깝습니다.

## 효과 측정 방법

### H3. 첫 실행과 두 번째 실행을 나눠 측정한다

compile cache는 첫 실행에서 캐시를 만들고, 이후 실행에서 재사용하는 구조입니다.
따라서 한 번만 실행해 보고 효과가 없다고 판단하면 안 됩니다.
최소한 다음처럼 캐시가 없는 상태와 있는 상태를 나눠 봐야 합니다.

```bash
rm -rf .node-compile-cache

# 캐시 생성 전 첫 실행
NODE_COMPILE_CACHE=.node-compile-cache time node app.js

# 캐시 재사용 가능성이 있는 두 번째 실행
NODE_COMPILE_CACHE=.node-compile-cache time node app.js
```

CLI라면 실제 사용자가 호출하는 명령을 기준으로 측정합니다.

```bash
rm -rf .node-compile-cache
NODE_COMPILE_CACHE=.node-compile-cache time node ./bin/my-cli.js --help
NODE_COMPILE_CACHE=.node-compile-cache time node ./bin/my-cli.js --help
```

`--help`처럼 외부 API나 데이터베이스를 호출하지 않는 명령은 시작 비용을 분리해서 보기 좋습니다.
반대로 네트워크 호출이 섞인 명령은 compile cache 효과보다 외부 지연이 더 크게 보일 수 있습니다.

### H3. 애플리케이션 내부 기준점도 함께 남긴다

운영에서 개선 효과를 보려면 단순한 체감보다 일관된 기준점이 필요합니다.
예를 들어 서버가 시작된 뒤 첫 요청을 받을 준비가 되는 시점까지 측정할 수 있습니다.

```js
import { performance } from 'node:perf_hooks';
import http from 'node:http';

const bootStart = performance.now();

const server = http.createServer((request, response) => {
  response.end('ok');
});

server.listen(3000, () => {
  const bootMs = performance.now() - bootStart;
  console.log(`ready in ${bootMs.toFixed(1)}ms`);
});
```

이 숫자는 module compile cache만의 효과를 정확히 분리하지는 못합니다.
하지만 배포 전후, 캐시 적용 전후의 시작 시간 변화를 추적하는 데는 충분히 유용합니다.
더 세밀한 성능 분석이 필요하다면 [Node.js worker_threads 가이드: CPU 바운드 작업으로 이벤트 루프 막지 않는 법](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)처럼 작업 종류를 분리해 보는 것도 좋습니다.

## 실무 적용 패턴

### H3. 서버리스에서는 재사용 가능한 경로를 확인한다

서버리스 환경에서는 파일 시스템이 완전히 영구적이지 않을 수 있습니다.
하지만 같은 인스턴스가 재사용되는 동안 임시 디렉터리가 유지되는 플랫폼도 있습니다.
이 경우 module compile cache가 warm start에 도움을 줄 수 있습니다.

예시는 다음과 같습니다.

```bash
NODE_COMPILE_CACHE=/tmp/node-compile-cache node handler.js
```

다만 플랫폼마다 `/tmp` 용량, 보존 시간, 동시 실행 모델이 다릅니다.
반드시 실제 배포 환경에서 다음을 확인해야 합니다.

- 캐시 디렉터리에 쓰기 권한이 있는가?
- 함수 재사용 사이에 캐시가 유지되는가?
- 콜드 스타트와 warm start를 나눠 측정했는가?
- 캐시 생성 비용이 첫 요청 지연을 과도하게 늘리지는 않는가?

서버리스 성능은 compile cache 하나로 결정되지 않습니다.
번들 크기, 초기 import 수, 외부 연결 초기화, 환경변수 로딩도 함께 봐야 합니다.
환경변수 관리가 섞여 있다면 [Node.js loadEnvFile 가이드: dotenv 없이 환경변수를 단순하게 관리하는 법](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)을 참고해 시작 로직을 단순화하는 편이 좋습니다.

### H3. CLI에서는 체감 속도가 큰 명령부터 적용한다

CLI 도구는 사용자가 명령을 입력한 뒤 첫 출력이 나올 때까지의 시간이 중요합니다.
특히 `lint`, `format`, `generate`, `deploy`, `preview`처럼 자주 실행하는 명령은 작은 지연도 피로로 쌓입니다.

적용 우선순위는 다음처럼 잡을 수 있습니다.

1. `--help`, `--version`처럼 즉시 끝나야 하는 명령
2. 설정 파일과 플러그인을 많이 읽는 명령
3. CI에서 반복 실행되는 짧은 스크립트
4. 개발자가 로컬에서 하루에 여러 번 실행하는 명령

단, compile cache를 켜기 전에 진입점 구조도 함께 점검해야 합니다.
`--version`을 출력하는 데 데이터베이스 클라이언트나 빌드 도구 전체를 import한다면, 캐시보다 import 지연 로딩이 더 큰 개선을 만들 수 있습니다.

```js
// 좋지 않은 예: 버전 출력에도 무거운 모듈을 먼저 로드한다
import { runBuild } from '../lib/build.js';
import { version } from '../package-version.js';

if (process.argv.includes('--version')) {
  console.log(version);
  process.exit(0);
}

await runBuild();
```

```js
// 개선 예: 가벼운 명령은 먼저 처리하고, 무거운 모듈은 필요할 때 로드한다
import { version } from '../package-version.js';

if (process.argv.includes('--version')) {
  console.log(version);
  process.exit(0);
}

const { runBuild } = await import('../lib/build.js');
await runBuild();
```

compile cache와 지연 import를 함께 쓰면 짧은 CLI 명령의 시작 시간을 더 안정적으로 줄일 수 있습니다.

## 주의할 점과 운영 체크리스트

### H3. 캐시는 커밋하지 않는다

compile cache는 실행 환경에서 만들어지는 산출물입니다.
프로젝트 소스와 함께 관리할 이유가 거의 없습니다.
캐시 파일을 Git에 넣으면 저장소가 불필요하게 커지고, 다른 OS나 Node.js 버전에서 의미 없는 파일이 섞일 수 있습니다.

```bash
printf "\n.node-compile-cache/\n" >> .gitignore
```

이미 실수로 추가했다면 캐시 디렉터리를 제거한 뒤 다시 커밋하는 편이 좋습니다.
민감정보를 담는 기능은 아니지만, 빌드 산출물을 소스처럼 다루면 배포 재현성이 흐려질 수 있습니다.

### H3. Node.js 버전과 배포 단위를 맞춘다

캐시는 Node.js 런타임과 코드 내용에 영향을 받습니다.
로컬에서 만든 캐시를 운영에 그대로 옮기는 방식보다는, 운영 환경에서 자연스럽게 생성되도록 두는 편이 안전합니다.
컨테이너 이미지에 캐시를 미리 넣는 전략도 가능하지만, 다음 조건을 만족할 때만 검토하는 것이 좋습니다.

- 빌드와 런타임 Node.js 버전이 동일하다
- 소스 경로와 실행 경로가 안정적이다
- 이미지 크기 증가보다 시작 시간 개선이 더 중요하다
- 캐시가 오래된 상태로 남았을 때 재생성 절차가 명확하다

대부분의 서비스에서는 먼저 런타임 캐시 디렉터리를 지정하고, 실제 수치가 충분히 개선되는지 확인하는 접근이 더 단순합니다.

### H3. 숫자로 검증하고 유지한다

성능 최적화는 적용했다는 사실보다 유지되는 숫자가 중요합니다.
다음 체크리스트를 발행 전후로 남겨 두면 회귀를 줄일 수 있습니다.

- 캐시 적용 전 첫 실행 시간과 두 번째 실행 시간
- 캐시 적용 후 첫 실행 시간과 두 번째 실행 시간
- 캐시 디렉터리 크기
- 배포 환경에서 쓰기 권한 오류 여부
- CI와 로컬 개발에서 스크립트가 동일하게 동작하는지 여부

테스트 자동화가 필요하다면 [Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하는 법](/development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html)처럼 기본 검증 스크립트를 먼저 정리하고, 성능 측정은 별도 작업으로 분리하는 편이 좋습니다.

## 자주 묻는 질문

### H3. module compile cache를 켜면 모든 Node.js 앱이 빨라지나요?

아닙니다.
이미 오래 떠 있는 서버이거나 시작 시간보다 데이터베이스, 네트워크, 렌더링 비용이 큰 앱에서는 체감 효과가 작을 수 있습니다.
반복 실행되는 CLI, 서버리스 함수, 짧은 배치처럼 프로세스 시작 비용 비중이 큰 작업에서 먼저 측정해 보는 것이 좋습니다.

### H3. 캐시를 지워도 애플리케이션이 깨지나요?

일반적으로 캐시는 보조 산출물이므로 지워도 애플리케이션은 다시 실행되면서 필요한 캐시를 만들 수 있습니다.
다만 캐시 디렉터리 쓰기 권한이 없거나 디스크가 꽉 찬 상황은 로그로 확인해야 합니다.
운영에서는 캐시 삭제를 장애 복구 절차 중 하나로 다룰 수 있게 문서화해 두면 좋습니다.

### H3. 번들러 최적화와 module compile cache 중 무엇을 먼저 해야 하나요?

둘은 목적이 다릅니다.
번들러 최적화는 배포 파일 크기와 import 구조를 줄이는 데 도움이 되고, module compile cache는 반복 실행에서 컴파일 비용을 줄이는 데 도움이 됩니다.
서버리스나 CLI에서는 먼저 진입점 import를 가볍게 만들고, 그다음 compile cache 효과를 측정하는 순서를 추천합니다.

## 마무리

Node.js module compile cache는 눈에 잘 보이지 않는 시작 비용을 줄이는 실용적인 선택지입니다.
특히 서버리스 콜드 스타트, CLI 응답성, 짧은 배치 실행처럼 프로세스가 자주 새로 뜨는 환경에서는 작은 개선도 누적 효과가 큽니다.

다만 성능 기능은 항상 측정과 함께 적용해야 합니다.
`NODE_COMPILE_CACHE`를 켜고, 첫 실행과 재실행을 나눠 비교하고, 캐시 디렉터리 운영 방식을 정리해 두면 안전하게 시작할 수 있습니다.
효과가 분명하다면 프로젝트의 표준 실행 스크립트나 배포 환경변수에 반영하고, 효과가 작다면 진입점 import 구조와 번들 크기부터 다시 점검하는 편이 좋습니다.

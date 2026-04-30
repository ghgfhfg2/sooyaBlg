---
layout: post
title: "Node.js import.meta.dirname 가이드: ESM에서 __dirname 대체를 가장 안전하게 처리하는 법"
date: 2026-04-30 08:00:00 +0900
lang: ko
translation_key: nodejs-import-meta-dirname-filename-esm-guide
permalink: /development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html
alternates:
  ko: /development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html
  x_default: /development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, esm, import-meta, dirname, filename, javascript, backend]
description: "Node.js ESM 환경에서 import.meta.dirname, import.meta.filename, import.meta.url을 활용해 __dirname 대체 패턴을 안전하게 구현하는 방법과 실무 주의사항을 예제와 함께 정리했습니다."
---

Node.js에서 CommonJS를 ESM으로 옮길 때 가장 자주 막히는 지점 중 하나가 `__dirname`과 `__filename`입니다.
기존 코드는 잘 동작했는데 `type: module`로 바꾸는 순간 경로 계산이 깨지고, 설정 파일·템플릿·정적 자산을 읽는 코드에서 바로 문제가 드러납니다.

이럴 때 핵심은 **ESM에서는 `import.meta.dirname`·`import.meta.filename`·`import.meta.url`을 기준으로 경로를 다뤄야 한다**는 점입니다.
결론부터 말하면 최신 Node.js에서는 `import.meta.dirname`을 우선 쓰고, 호환 범위를 넓혀야 하면 `new URL()` 또는 `fileURLToPath(import.meta.url)` 패턴을 함께 이해하는 것이 가장 안전합니다.

## Node.js ESM에서 __dirname이 사라지는 이유

### H3. CommonJS의 전역 값은 ESM에 그대로 제공되지 않는다

CommonJS에서는 아래 코드가 자연스럽습니다.

```js
console.log(__filename);
console.log(__dirname);
```

하지만 ESM에서는 이 값들이 기본 제공되지 않습니다.
이 차이는 단순한 문법 변화가 아니라, **모듈 시스템 자체가 다르기 때문**입니다.

그래서 기존 코드를 ESM으로 바꿀 때 아래 문제가 자주 생깁니다.

- 현재 파일 기준 상대 경로 계산 실패
- 템플릿 파일 로딩 오류
- CLI 설정 파일 탐색 실패
- 테스트 환경과 런타임 환경에서 경로 처리 불일치

즉 ESM 마이그레이션에서는 import/export만 바꾸면 끝나는 것이 아니라, **파일 경로 기준점도 함께 바꿔야** 합니다.

### H3. ESM에서는 import.meta가 현재 모듈 정보를 제공한다

ESM에서는 현재 모듈을 설명하는 메타데이터 접근점으로 `import.meta`를 사용합니다.
가장 기본이 되는 값은 `import.meta.url`입니다.

```js
console.log(import.meta.url);
```

보통 결과는 아래처럼 `file:` URL 형태입니다.

```txt
file:///Users/example/project/src/index.js
```

즉 ESM에서는 파일 시스템 경로를 바로 받는 대신, **현재 모듈의 URL 정보를 기준으로 경로를 계산한다**고 이해하면 훨씬 덜 헷갈립니다.

## import.meta.dirname과 import.meta.filename 기본 사용법

### H3. 최신 Node.js에서는 dirname과 filename을 더 직접적으로 쓸 수 있다

최신 Node.js에서는 `import.meta.dirname`과 `import.meta.filename`을 바로 사용할 수 있습니다.

```js
console.log(import.meta.filename);
console.log(import.meta.dirname);
```

이 패턴의 장점은 분명합니다.

- CommonJS의 `__filename`, `__dirname`와 의미가 비슷함
- 코드 가독성이 좋음
- 파일 기준 리소스 접근 코드가 간단해짐
- URL 변환 보일러플레이트를 줄일 수 있음

예를 들어 현재 파일 기준으로 템플릿 파일을 읽고 싶다면 아래처럼 쓸 수 있습니다.

```js
import { readFile } from 'node:fs/promises';
import path from 'node:path';

const templatePath = path.join(import.meta.dirname, 'templates', 'welcome.html');
const html = await readFile(templatePath, 'utf8');
```

대용량 파일을 줄 단위로 처리해야 한다면 이 경로 계산 결과를 [Node.js FileHandle.readLines 가이드](/development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html)에서 다룬 방식과 그대로 결합할 수 있습니다.

### H3. 파일 기준 자산 접근에서는 path.join 조합이 가장 읽기 쉽다

실무에서 자주 하는 일은 “현재 모듈 옆에 있는 파일”을 읽는 것입니다.
그럴 때는 `path.join(import.meta.dirname, ...)` 패턴이 가장 읽기 쉽습니다.

```js
import path from 'node:path';

const configPath = path.join(import.meta.dirname, 'config', 'default.json');
const workerPath = path.join(import.meta.dirname, 'workers', 'resize-worker.js');
```

다만 이 방식은 **파일 시스템 경로가 필요한 API에 넘길 때** 특히 잘 맞습니다.
반대로 `Worker`, `fetch`, 동적 import처럼 URL 기반 접근이 더 자연스러운 API라면 `new URL()` 패턴이 더 나을 수 있습니다.

## import.meta.url과 new URL()을 같이 이해해야 하는 이유

### H3. URL 기반으로 경로를 만들면 ESM답게 더 일관된 코드가 된다

ESM에서는 아래 패턴도 매우 자주 사용됩니다.

```js
const templateUrl = new URL('./templates/welcome.html', import.meta.url);
```

이 방식은 특히 다음 상황에서 좋습니다.

- 현재 모듈 기준 상대 위치를 명확히 표현하고 싶을 때
- URL 객체를 그대로 받는 API를 사용할 때
- 파일 경로와 웹 URL 개념을 일관되게 다루고 싶을 때

예를 들어 워커 스레드 시작 코드는 아래처럼 많이 작성합니다.

```js
import { Worker } from 'node:worker_threads';

const worker = new Worker(new URL('./worker-task.js', import.meta.url));
```

이 패턴은 [Node.js worker_threads 성능 가이드](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)처럼 워커 파일을 현재 모듈 기준으로 안전하게 지정해야 할 때 특히 깔끔합니다.

### H3. 파일 시스템 API가 문자열 경로를 요구하면 fileURLToPath 변환이 필요할 수 있다

일부 코드베이스는 URL 객체보다 문자열 경로를 선호합니다.
그럴 때는 `fileURLToPath()`를 사용할 수 있습니다.

```js
import { fileURLToPath } from 'node:url';
import path from 'node:path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
```

이 패턴은 `import.meta.dirname`이 없는 환경까지 넓게 호환해야 할 때 여전히 유효합니다.
즉 선택 기준은 대체로 아래처럼 정리할 수 있습니다.

- 최신 Node.js 중심이면 `import.meta.dirname`
- URL 기반 API면 `new URL(..., import.meta.url)`
- 구버전 호환이 중요하면 `fileURLToPath(import.meta.url)`

## 어떤 패턴을 선택해야 할까

### H3. 경로 문자열이 필요하면 import.meta.dirname을 우선 고려한다

예를 들어 아래와 같은 경우입니다.

- `fs.readFile()`에 넘길 파일 경로 만들기
- 템플릿/정적 자산 경로 조합
- 로그/설정 파일 위치 계산
- CLI에서 기본 경로 구성

이런 경우는 `path.join(import.meta.dirname, ...)`가 가장 단순합니다.
CLI 인자를 함께 처리한다면 [Node.js util.parseArgs 가이드](/development/blog/seo/2026/04/28/nodejs-util-parseargs-cli-argument-parsing-guide.html)와 조합해 기본 경로 + 사용자 입력 경로를 안정적으로 다룰 수 있습니다.

### H3. 모듈 상대 URL이 자연스러우면 new URL 패턴이 더 낫다

반대로 아래 상황은 URL 패턴이 더 적합합니다.

- 워커 파일 지정
- `import()` 대상 계산
- `fs`가 URL 객체도 받을 수 있는 코드 경로
- 브라우저/번들러 사고방식과 비슷하게 유지하고 싶을 때

핵심은 “무조건 하나만 써야 한다”가 아닙니다.
**문자열 경로가 필요한지, URL이 더 자연스러운지**에 따라 도구를 고르는 편이 실무적으로 맞습니다.

## ESM 경로 처리에서 자주 하는 실수

### H3. process.cwd()를 현재 파일 위치처럼 쓰면 배포 환경에서 깨지기 쉽다

가장 흔한 실수는 `process.cwd()`를 현재 모듈 위치처럼 쓰는 것입니다.

```js
const wrongPath = path.join(process.cwd(), 'templates', 'welcome.html');
```

이 코드는 로컬에서는 우연히 맞을 수 있지만, 실행 위치가 바뀌면 바로 깨집니다.
예를 들어 다음 상황이 위험합니다.

- 다른 디렉터리에서 CLI 실행
- PM2/systemd 등 서비스 매니저로 실행
- 테스트 러너가 루트를 다르게 잡는 경우
- monorepo 내부 패키지에서 개별 실행되는 경우

`process.cwd()`는 **프로세스가 시작된 작업 디렉터리**이지, 현재 파일 위치가 아닙니다.
현재 모듈 기준 리소스는 `import.meta.dirname` 또는 `import.meta.url` 기준으로 계산하는 편이 안전합니다.

### H3. path.join과 URL 조합을 섞을 때 타입을 헷갈리기 쉽다

또 다른 흔한 문제는 경로 문자열과 URL 객체를 뒤섞는 것입니다.

```js
const fileUrl = new URL('./data.json', import.meta.url);
const broken = path.join(fileUrl, 'extra');
```

이런 코드는 타입이 섞여 의도와 다르게 동작할 수 있습니다.
실무에서는 아래 원칙을 두면 사고를 줄이기 쉽습니다.

- 끝까지 URL로 갈 것인지
- 중간에 파일 시스템 경로 문자열로 바꿀 것인지
- 어느 API가 어떤 타입을 기대하는지 먼저 확인할 것인지

즉 한 함수 안에서라도 **URL 흐름과 path 문자열 흐름을 의식적으로 분리**하는 편이 좋습니다.

## 실무 추천 패턴

### H3. 애플리케이션 기본값은 최신 패턴, 라이브러리는 호환 범위를 더 보수적으로 본다

제가 추천하는 기준은 꽤 단순합니다.

애플리케이션 코드라면:

- 실행 Node.js 버전을 통제하기 쉬움
- `import.meta.dirname` 도입 이점이 큼
- 코드가 짧고 읽기 쉬워짐

라이브러리 코드라면:

- 사용자 런타임 버전이 더 다양할 수 있음
- `fileURLToPath(import.meta.url)` 폴백이 아직 유용함
- 문서에 지원 Node.js 버전을 분명히 적는 편이 안전함

### H3. 팀 규칙을 하나로 정하면 경로 관련 버그가 줄어든다

경로 처리 버그는 개별 코드보다 **팀 내 패턴 불일치**에서 더 자주 나옵니다.
그래서 아래처럼 규칙을 정해두는 편이 좋습니다.

- 파일 시스템 경로는 `import.meta.dirname` 우선
- 워커/동적 import는 `new URL(..., import.meta.url)` 우선
- `process.cwd()`는 사용자 입력 기준 루트가 필요할 때만 사용
- 구버전 호환이 필요하면 `fileURLToPath()` 폴백 허용

이 정도만 맞춰도 ESM 전환 후 경로 관련 오류를 꽤 줄일 수 있습니다.

## FAQ

### H3. import.meta.dirname만 쓰면 모든 Node.js 버전에서 안전한가

아닙니다.
프로젝트가 구버전 Node.js까지 지원해야 한다면 지원 범위를 먼저 확인해야 합니다.
호환성이 중요하면 `fileURLToPath(import.meta.url)` 기반 폴백을 고려하는 편이 안전합니다.

### H3. __dirname을 직접 다시 만들어서 계속 써도 괜찮은가

가능합니다.
다만 최신 Node.js만 대상으로 하는 애플리케이션이라면 `import.meta.dirname`을 직접 쓰는 편이 더 단순하고 읽기 쉽습니다.

### H3. process.cwd()는 언제 써야 하나

현재 파일 위치가 아니라 **사용자가 명령을 실행한 기준 디렉터리**가 필요할 때만 쓰는 편이 좋습니다.
예를 들어 CLI에서 상대 입력 파일을 해석할 때는 `process.cwd()`가 맞고, 모듈 옆 템플릿 파일을 읽을 때는 맞지 않습니다.

## 마무리

Node.js ESM에서 경로 처리는 “`__dirname`이 없네?” 수준의 작은 차이처럼 보여도, 실제로는 배포·테스트·CLI 동작 안정성에 직접 연결됩니다.
가장 실용적인 기준은 이렇습니다.

- 최신 Node.js 앱이면 `import.meta.dirname` 우선
- URL 중심 API면 `new URL(..., import.meta.url)` 사용
- 호환성이 중요하면 `fileURLToPath()` 폴백 준비

이 기준만 잡아도 ESM 전환 과정에서 나오는 경로 버그 대부분을 훨씬 차분하게 정리할 수 있습니다.

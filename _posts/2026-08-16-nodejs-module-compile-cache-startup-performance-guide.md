---
layout: post
title: "Node.js module.enableCompileCache 가이드: 시작 시간을 줄이는 컴파일 캐시 적용법"
date: 2026-08-16 08:00:00 +0900
lang: ko
translation_key: nodejs-module-compile-cache-startup-performance-guide
permalink: /development/blog/seo/2026/08/16/nodejs-module-compile-cache-startup-performance-guide.html
alternates:
  ko: /development/blog/seo/2026/08/16/nodejs-module-compile-cache-startup-performance-guide.html
  x_default: /development/blog/seo/2026/08/16/nodejs-module-compile-cache-startup-performance-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, module, compile-cache, startup-performance, performance, testing, backend, javascript]
description: "Node.js module.enableCompileCache()로 애플리케이션 시작 시간을 줄이는 방법을 정리합니다. 캐시 디렉터리 선택, 상태값 처리, 테스트 커버리지와의 충돌, 배포 환경에서의 운영 체크리스트까지 실무 예제로 설명합니다."
---

Node.js 애플리케이션은 모듈을 처음 불러올 때 소스 코드를 파싱하고 컴파일합니다.
서버가 오래 살아 있는 서비스라면 이 비용이 크게 보이지 않을 수 있습니다.
하지만 CLI, 서버리스 함수, 짧게 실행되는 배치 작업, 테스트 프로세스처럼 자주 시작하고 종료되는 프로그램에서는 시작 시간이 사용자 경험과 비용에 직접 영향을 줍니다.

`module.enableCompileCache()`는 Node.js가 모듈 컴파일 결과를 디스크에 저장해 다음 실행에서 재사용할 수 있게 해 주는 API입니다.
캐시는 기능 동작을 바꾸는 핵심 의존성이 아니라 시작 시간을 줄이는 최적화 장치로 보는 편이 안전합니다.
따라서 적용할 때도 "성공하면 빠르게, 실패해도 정상 실행"이라는 운영 원칙을 먼저 정해야 합니다.

이 글에서는 `module.enableCompileCache()`를 어디에서 호출할지, 캐시 디렉터리를 어떻게 선택할지, 상태값을 어떻게 로깅할지, 테스트 커버리지와 충돌할 때 어떻게 끄는지까지 정리합니다.
성능 측정 흐름은 [Node.js performance.timerify 가이드](/development/blog/seo/2026/06/11/nodejs-performance-timerify-function-duration-guide.html), 테스트 실행 기준은 [Node.js test runner mock timers 가이드](/development/blog/seo/2026/08/03/nodejs-test-runner-mock-timers-time-dependent-test-guide.html)와 함께 보면 좋습니다.
런타임 플래그를 관리하는 방식은 [Node.js process.execArgv 가이드](/development/blog/seo/2026/08/13/nodejs-process-execargv-runtime-flags-guide.html)도 이어서 참고할 수 있습니다.

## module.enableCompileCache가 필요한 순간

### 시작 비용이 반복해서 발생하는 프로세스에 우선 적용한다

컴파일 캐시는 모든 Node.js 서비스에 반드시 필요한 설정은 아닙니다.
이미 한 번 시작한 뒤 오래 실행되는 API 서버에서는 체감 효과가 작을 수 있습니다.
반대로 작은 CLI를 하루에 수백 번 실행하거나, 서버리스 함수가 콜드 스타트를 자주 겪거나, 배치 작업이 짧게 뜨고 사라지는 구조라면 시작 비용을 줄이는 가치가 커집니다.

```js
import module from 'node:module';

const result = module.enableCompileCache();

console.log(result);
// {
//   status: module.constants.compileCacheStatus.ENABLED,
//   directory: '/tmp/node-compile-cache'
// }
```

이 API는 캐시를 켜지 못해도 예외를 던지는 방식이 아니라 상태 객체를 반환합니다.
공식 문서에서도 컴파일 캐시는 미션 크리티컬 기능이 아닌 최적화로 다루는 흐름에 맞춰 설계되어 있습니다.
따라서 애플리케이션도 캐시 활성화 실패를 장애로 만들기보다, 원인을 로깅하고 정상 실행을 계속하는 쪽이 좋습니다.

### 진입점의 가장 앞에서 호출한다

컴파일 캐시는 모듈을 로드할 때 의미가 있습니다.
이미 많은 모듈을 불러온 뒤 뒤늦게 켜면 그 전에 로드된 모듈에는 효과가 제한됩니다.
그래서 가능한 한 애플리케이션 진입점의 맨 앞, 또는 별도 preload 파일에서 먼저 호출하는 편이 좋습니다.

```js
// register-compile-cache.js
import module from 'node:module';

module.enableCompileCache();
```

```bash
node --import ./register-compile-cache.js ./src/server.js
```

`--import`를 쓰면 애플리케이션 본문보다 먼저 캐시 설정을 실행할 수 있습니다.
ESM 프로젝트와 CLI 도구에서는 이 방식이 깔끔합니다.
CommonJS 중심 프로젝트라면 진입 파일 상단에서 `require('node:module').enableCompileCache()`를 호출하는 방식도 가능합니다.

## 캐시 디렉터리 운영 기준

### 기본값을 쓰되 환경변수로 덮어쓸 수 있게 둔다

일반적인 사용에서는 `module.enableCompileCache()`를 인자 없이 호출해도 됩니다.
Node.js는 환경변수 `NODE_COMPILE_CACHE`가 있으면 그 디렉터리를 사용하고, 없으면 임시 디렉터리 아래의 기본 캐시 경로를 사용합니다.

```js
import module from 'node:module';

export function enableStartupCache(logger = console) {
  const result = module.enableCompileCache();

  if (result.status === module.constants.compileCacheStatus.FAILED) {
    logger.warn('compile cache was not enabled', {
      message: result.message
    });
    return;
  }

  logger.info('compile cache configured', {
    status: result.status,
    directory: result.directory
  });
}
```

여기서 중요한 점은 환경변수 전체를 로그로 남기지 않는 것입니다.
캐시 디렉터리, 상태값, 실패 메시지는 진단에 도움이 됩니다.
하지만 `process.env`를 통째로 출력하면 토큰, 데이터베이스 주소, 서드파티 키 같은 민감정보가 노출될 수 있습니다.

운영 환경에서 캐시 위치를 명시하고 싶다면 실행 환경에만 값을 둡니다.

```bash
NODE_COMPILE_CACHE=/tmp/my-app-node-cache node ./src/server.js
```

코드 안에 절대 경로를 고정하면 컨테이너, 로컬 개발, CI, 서버리스 런타임 사이에서 이식성이 떨어집니다.
기본값과 환경변수 override를 함께 쓰면 같은 코드를 여러 실행 환경에 적용하기 쉽습니다.

### 프로젝트 이동 가능성이 있으면 portable 옵션을 검토한다

컴파일 캐시는 기본적으로 모듈의 절대 경로와 연결됩니다.
프로젝트 디렉터리가 바뀌면 기존 캐시를 재사용하지 못할 수 있습니다.
빌드 산출물을 다른 경로로 옮겨 실행하거나, CI 작업 디렉터리가 매번 달라지는 구조라면 portable 모드를 검토할 수 있습니다.

```js
import module from 'node:module';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

module.enableCompileCache({
  directory: join(tmpdir(), 'my-app-node-compile-cache'),
  portable: true
});
```

portable 모드는 프로젝트가 이동해도 캐시를 재사용하려는 최선 노력 방식입니다.
항상 모든 상황에서 캐시 적중을 보장하는 기능으로 이해하면 안 됩니다.
그래서 배포 최적화로 적용하더라도, 캐시가 비어 있는 첫 실행과 캐시가 채워진 이후 실행을 나눠 측정해야 합니다.

## 테스트와 커버리지에서 주의할 점

### 커버리지 수집 시에는 캐시를 끌 수 있게 만든다

컴파일 캐시는 테스트 속도를 줄이는 데 도움이 될 수 있지만, V8 기반 코드 커버리지와 함께 사용할 때 커버리지 정밀도가 낮아질 수 있습니다.
공식 문서도 테스트 커버리지를 정확히 수집해야 할 때는 컴파일 캐시를 끄는 것을 권장합니다.

가장 단순한 방법은 커버리지 스크립트에서 `NODE_DISABLE_COMPILE_CACHE=1`을 지정하는 것입니다.

```json
{
  "scripts": {
    "test": "node --test",
    "test:coverage": "NODE_DISABLE_COMPILE_CACHE=1 node --test --experimental-test-coverage"
  }
}
```

운영 실행과 테스트 커버리지 실행은 목적이 다릅니다.
운영에서는 시작 시간을 줄이는 것이 중요할 수 있고, 커버리지에서는 정확한 측정이 더 중요합니다.
두 스크립트를 분리하면 최적화 설정이 품질 지표를 흐리지 않게 만들 수 있습니다.

### 테스트 전용 부트스트랩을 분리한다

테스트 환경에서는 캐시를 항상 켜기보다 필요한 스위트에서만 켜는 편이 관리하기 쉽습니다.
특히 커버리지, 모듈 mocking, 시간 의존 테스트가 섞여 있다면 전역 preload 파일 하나가 예상 밖의 영향을 만들 수 있습니다.

```js
// test/register-fast-startup.js
import module from 'node:module';

if (process.env.NODE_DISABLE_COMPILE_CACHE !== '1') {
  module.enableCompileCache();
}
```

```bash
node --import ./test/register-fast-startup.js --test
```

이렇게 두면 일반 테스트에서는 캐시를 켤 수 있고, 커버리지 수집에서는 환경변수 하나로 끌 수 있습니다.
중요한 것은 "테스트가 왜 빠른가"와 "커버리지가 왜 정확한가"를 스크립트만 보고 이해할 수 있게 만드는 것입니다.

## 배포 환경 적용 패턴

### 캐시 성공 여부를 배포 실패 조건으로 삼지 않는다

컴파일 캐시는 디스크 권한, 임시 디렉터리 정책, 읽기 전용 파일 시스템 같은 이유로 실패할 수 있습니다.
이때 애플리케이션 자체가 캐시 없이는 동작하지 못하는 구조라면 최적화가 장애 원인이 됩니다.
캐시는 실패해도 서비스가 계속 떠야 합니다.

```js
import module from 'node:module';

export function configureCompileCache(logger) {
  const { compileCacheStatus } = module.constants;
  const result = module.enableCompileCache();

  switch (result.status) {
    case compileCacheStatus.ENABLED:
    case compileCacheStatus.ALREADY_ENABLED:
      logger.info('node compile cache enabled', {
        directory: result.directory
      });
      break;
    case compileCacheStatus.DISABLED:
      logger.info('node compile cache disabled by environment');
      break;
    case compileCacheStatus.FAILED:
      logger.warn('node compile cache failed', {
        message: result.message
      });
      break;
    default:
      logger.warn('unknown node compile cache status', {
        status: result.status
      });
  }
}
```

이 코드는 실패를 명확히 기록하지만 프로세스를 종료하지 않습니다.
운영자는 로그를 보고 캐시 디렉터리 권한이나 런타임 환경을 조정할 수 있고, 서비스는 캐시 없이도 정상 기능을 제공합니다.

### 배포 전후 시작 시간을 직접 측정한다

컴파일 캐시는 "켜면 무조건 빨라진다"가 아니라 "반복 실행에서 빨라질 수 있다"에 가깝습니다.
프로젝트 규모, 모듈 수, 디스크 성능, 캐시 적중률에 따라 결과가 달라집니다.
적용 전후에는 같은 조건에서 시작 시간을 측정해야 합니다.

```bash
hyperfine \
  'node ./src/cli.js --help' \
  'node --import ./register-compile-cache.js ./src/cli.js --help'
```

별도 도구를 쓰지 않는다면 `performance.now()`로 진입점 안에서 주요 초기화 구간을 나눠 기록해도 됩니다.
단, 첫 실행은 캐시를 생성하는 비용이 포함될 수 있으므로 두 번째 이후 실행 시간을 함께 봐야 합니다.
서버리스 환경에서는 콜드 스타트와 웜 스타트를 분리해서 측정하는 것이 좋습니다.

## 운영 점검 체크리스트

### 적용 전에 확인할 것

- Node.js 버전에서 `module.enableCompileCache()`와 필요한 옵션을 지원하는지 확인한다.
- 캐시가 실패해도 애플리케이션이 정상 실행되도록 처리한다.
- 캐시 디렉터리는 환경변수로 조정할 수 있게 둔다.
- 커버리지 수집 스크립트에서는 `NODE_DISABLE_COMPILE_CACHE=1`을 사용한다.
- 캐시 적용 전후의 시작 시간을 같은 조건에서 측정한다.

### 적용 후에 확인할 것

- 시작 로그에 캐시 상태와 디렉터리만 남고 민감정보가 섞이지 않는지 확인한다.
- 임시 디렉터리에 오래된 캐시가 과도하게 쌓이지 않는지 점검한다.
- 배포 경로가 자주 바뀌는 환경에서는 portable 옵션의 효과를 따로 측정한다.
- 테스트 커버리지 수치가 캐시 설정 때문에 갑자기 흔들리지 않는지 확인한다.

## FAQ

### module.enableCompileCache는 애플리케이션 로직을 바꾸나요?

아니요.
목적은 모듈 컴파일 결과를 재사용해 시작 비용을 줄이는 것입니다.
캐시 활성화에 실패해도 애플리케이션은 캐시 없이 실행될 수 있어야 합니다.

### 캐시 디렉터리는 저장소 안에 두는 게 좋나요?

대부분의 경우 저장소 안에 두지 않는 편이 좋습니다.
컴파일 캐시는 생성물이고 Node.js 버전이나 실행 경로에 영향을 받습니다.
임시 디렉터리나 런타임에서 제공하는 캐시 디렉터리를 사용하고, 필요하면 `NODE_COMPILE_CACHE`로 위치를 지정하는 방식이 안전합니다.

### 테스트에서도 항상 켜야 하나요?

항상 켤 필요는 없습니다.
일반 테스트 속도를 줄이고 싶을 때는 도움이 될 수 있지만, 코드 커버리지를 정확히 수집해야 하는 실행에서는 `NODE_DISABLE_COMPILE_CACHE=1`로 끄는 편이 좋습니다.

### portable 옵션은 언제 쓰나요?

프로젝트가 실행되는 절대 경로가 자주 바뀌는 환경에서 검토할 수 있습니다.
다만 캐시 재사용을 보장하는 만능 설정은 아니므로, 실제 배포 방식에서 시작 시간이 개선되는지 측정한 뒤 유지하는 것이 좋습니다.

## 마무리

`module.enableCompileCache()`는 Node.js 시작 시간을 줄이고 싶은 프로젝트에서 부담 없이 시도할 수 있는 최적화입니다.
다만 캐시는 기능의 필수 조건이 아니라 성능 보조 장치입니다.
진입점 앞단에서 켜고, 실패는 로깅만 하며, 테스트 커버리지에서는 끌 수 있게 두는 구조가 실무적으로 가장 다루기 쉽습니다.

특히 CLI, 서버리스 함수, 짧게 실행되는 배치 작업처럼 시작 비용이 반복되는 환경에서는 적용 전후를 측정해 볼 가치가 큽니다.
측정 결과가 분명하다면 캐시 디렉터리 정책과 로그 기준을 함께 정리해 배포 환경에 반영하면 됩니다.

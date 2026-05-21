---
layout: post
title: "Node.js source maps 가이드: 운영 에러 스택을 원본 TypeScript로 읽는 법"
date: 2026-05-22 08:00:00 +0900
lang: ko
translation_key: nodejs-enable-source-maps-production-debugging-guide
permalink: /development/blog/seo/2026/05/22/nodejs-enable-source-maps-production-debugging-guide.html
alternates:
  ko: /development/blog/seo/2026/05/22/nodejs-enable-source-maps-production-debugging-guide.html
  x_default: /development/blog/seo/2026/05/22/nodejs-enable-source-maps-production-debugging-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, source-maps, typescript, debugging, stack-trace, observability, javascript]
description: "Node.js에서 --enable-source-maps와 배포 산출물의 .map 파일을 함께 관리해 운영 에러 스택을 번들된 JavaScript가 아니라 원본 TypeScript 기준으로 추적하는 방법을 정리했습니다. 빌드 설정, 로그 보안, 배포 체크리스트까지 다룹니다."
---

운영 장애를 조사할 때 스택 트레이스가 `dist/index.js:1:28491`처럼 찍히면 원인을 찾는 시간이 길어집니다.
TypeScript나 번들러를 거친 Node.js 서비스에서는 실제로 수정해야 할 파일이 `src/payment/charge.ts`인데, 로그에는 압축되거나 변환된 JavaScript 위치만 남는 일이 흔합니다.
작은 서비스라면 감으로 따라갈 수 있지만, 배포 산출물이 커질수록 이 방식은 오래 버티기 어렵습니다.

Node.js의 `--enable-source-maps` 옵션은 에러 스택을 source map 기준으로 다시 매핑해 원본 파일과 라인을 보여 주는 기능입니다.
다만 옵션 하나만 켠다고 끝나지는 않습니다.
빌드 도구가 올바른 `.map` 파일을 만들고, 배포 이미지에 필요한 파일이 포함되어야 하며, 외부 로그에 소스 경로나 민감한 원본 코드가 과하게 노출되지 않도록 운영 기준도 함께 정해야 합니다.
이 글에서는 Node.js source maps를 운영 디버깅에 안전하게 적용하는 방법을 정리합니다.
에러를 감싸는 구조까지 같이 정리하려면 [Node.js Error cause 가이드](/development/blog/seo/2026/05/01/nodejs-error-cause-wrapped-errors-debugging-guide.html)를 함께 참고하세요.

## Node.js source maps가 필요한 이유

### 변환된 JavaScript 라인은 원인 추적을 느리게 만든다

TypeScript 프로젝트는 보통 `src`를 `dist`로 컴파일한 뒤 배포합니다.
번들러를 사용하면 여러 파일이 하나로 합쳐지고, 압축까지 적용하면 한 줄짜리 JavaScript가 만들어질 수도 있습니다.
이 상태에서 예외가 발생하면 스택은 배포된 파일 기준으로 출력됩니다.

```js
function parseAmount(input) {
  const amount = Number(input.value);

  if (!Number.isFinite(amount)) {
    throw new TypeError('invalid amount');
  }

  return amount;
}

parseAmount({ value: 'not-a-number' });
```

위 예제는 단순하지만, 실제 서비스에서는 이 코드가 컴파일과 번들링을 거쳐 전혀 다른 줄 번호에 놓일 수 있습니다.
장애 대응자는 에러 메시지, 배포 커밋, 번들 산출물을 동시에 대조해야 합니다.
source map이 있으면 이 과정을 줄여 원본 파일 기준으로 바로 접근할 수 있습니다.

### 스택 트레이스는 관측 가능성의 출발점이다

로그, 알림, APM, 에러 리포팅 도구는 대부분 스택 트레이스를 중심으로 사건을 묶습니다.
스택이 원본 파일 기준으로 읽히면 같은 장애를 더 빨리 그룹화하고, 담당자가 어떤 모듈을 봐야 하는지도 명확해집니다.

source maps는 로그를 예쁘게 만드는 기능이 아니라 복구 시간을 줄이는 운영 도구입니다.
프로세스 진단 자료까지 함께 남기는 팀이라면 [Node.js process report 가이드](/development/blog/seo/2026/05/14/nodejs-process-report-production-diagnostics-guide.html)와 연결해 장애 분석 흐름을 정리하는 것이 좋습니다.

## --enable-source-maps 기본 사용법

### 실행 옵션으로 source map 매핑을 켠다

Node.js에서 source map 기반 스택 매핑을 사용하려면 실행 시 `--enable-source-maps`를 전달합니다.
로컬에서 먼저 확인할 때는 다음처럼 실행할 수 있습니다.

```bash
node --enable-source-maps dist/server.js
```

운영에서는 `NODE_OPTIONS`로 주입하는 방식도 자주 사용합니다.
컨테이너, PaaS, systemd, PM2처럼 실행 명령을 중앙에서 관리하는 환경에서는 옵션을 한 곳에 모으기 쉽기 때문입니다.

```bash
NODE_OPTIONS="--enable-source-maps" node dist/server.js
```

단, `NODE_OPTIONS`는 모든 하위 Node.js 프로세스에 영향을 줄 수 있습니다.
테스트 러너, 워커, 마이그레이션 스크립트까지 같은 옵션을 받을 수 있으므로 배포 환경에서 의도한 범위인지 확인해야 합니다.
런타임 옵션을 환경별로 통제하는 방식은 [Node.js Permission Model 가이드](/development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html)처럼 실행 경계를 명확히 잡는 글과 함께 보면 좋습니다.

### 에러 스택이 원본 위치로 바뀌는지 확인한다

source map 적용 여부는 의도적으로 에러를 던지는 작은 파일로 확인하는 것이 가장 빠릅니다.
배포 전 검증 스크립트에서 `dist` 파일을 실행하고, 스택에 `src` 경로가 포함되는지 확인할 수 있습니다.

```js
export function assertPositiveAmount(amount) {
  if (amount <= 0) {
    throw new RangeError('amount must be positive');
  }

  return amount;
}

assertPositiveAmount(-1);
```

컴파일된 파일을 `node --enable-source-maps`로 실행했을 때 스택이 `src/...`를 가리키면 기본 연결은 된 것입니다.
반대로 여전히 `dist/...`만 보인다면 `.map` 파일이 없거나, 컴파일 결과의 `sourceMappingURL`이 잘못되었거나, 번들러가 source map 생성을 꺼 둔 상태일 수 있습니다.

## 빌드 설정에서 확인할 것

### TypeScript는 sourceMap을 켜야 한다

TypeScript만 사용하는 프로젝트라면 `tsconfig.json`에서 `sourceMap`을 켭니다.
운영 배포 파일에 원본 TypeScript를 포함하지 않으려면 `inlineSources`는 신중히 선택해야 합니다.

```json
{
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "sourceMap": true,
    "inlineSources": false,
    "declaration": false
  }
}
```

`sourceMap: true`는 `.js.map` 파일을 별도로 생성합니다.
`inlineSources: true`를 켜면 map 안에 원본 소스 내용이 들어갈 수 있어 디버깅은 편하지만, 배포 이미지나 에러 수집 도구로 원본 코드가 더 넓게 이동합니다.
비공개 서버 내부에서만 쓰는지, 외부 리포팅 도구로 업로드하는지에 따라 정책을 나누는 편이 안전합니다.

### 번들러는 hidden source map과 배포 방식을 구분한다

esbuild, Rollup, webpack 같은 번들러를 쓰면 source map 옵션 이름과 동작이 조금씩 다릅니다.
중요한 질문은 세 가지입니다.

- `.map` 파일이 실제로 만들어지는가?
- 실행되는 `.js` 파일이 그 `.map` 파일을 참조하는가?
- 운영 이미지나 서버에 `.map` 파일이 함께 배포되는가?

```js
// 예: esbuild 설정 형태
import { build } from 'esbuild';

await build({
  entryPoints: ['src/server.ts'],
  bundle: true,
  platform: 'node',
  target: ['node22'],
  outfile: 'dist/server.js',
  sourcemap: true,
});
```

번들러를 쓰는 경우 `dist/server.js.map`은 생성되었는데 Dockerfile의 `COPY` 규칙에서 빠지는 일이 있습니다.
빌드는 성공하지만 운영 스택은 매핑되지 않는 전형적인 원인입니다.
배포 산출물 검증에서 `.js`와 `.js.map`의 짝을 확인하는 단계를 넣어 두면 재발을 줄일 수 있습니다.

## 운영 로그와 보안 기준

### 원본 경로 노출 범위를 정한다

source map이 적용된 스택은 원본 파일 경로를 보여 줍니다.
대부분의 경우 `src/services/payment.ts` 같은 경로는 문제 되지 않지만, 사내 프로젝트명, 사용자 홈 디렉터리, 빌드 서버 경로가 그대로 노출되면 불필요한 정보가 외부 로그에 남을 수 있습니다.

```js
function sanitizeStack(stack) {
  return String(stack)
    .replaceAll(process.cwd(), '<app>')
    .replaceAll(process.env.HOME ?? '', '<home>');
}

try {
  throw new Error('example failure');
} catch (error) {
  console.error(sanitizeStack(error.stack));
}
```

외부로 전송되는 로그에는 절대경로를 줄이고, 내부 보안 로그나 APM에는 더 자세한 정보를 남기는 식으로 레벨을 나누는 것이 좋습니다.
로그 예시를 문서에 넣을 때는 [CLI 출력 sanitizing 가이드](/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html)의 원칙처럼 토큰, 계정명, 내부 호스트명을 마스킹해야 합니다.

### source map 파일을 공개 정적 경로에 두지 않는다

서버 사이드 Node.js source map은 런타임 디버깅을 위한 파일입니다.
웹 프런트엔드 source map처럼 CDN에 올릴 필요가 없습니다.
특히 서버 코드의 `.map` 파일이 정적 파일 경로로 공개되면 내부 구현이 노출될 수 있습니다.

운영 원칙은 단순합니다.
Node.js 프로세스가 읽어야 하는 위치에는 두되, 브라우저에서 접근 가능한 public 디렉터리에는 두지 않습니다.
컨테이너 안에서는 `dist` 옆에 보관하되, 웹 서버의 정적 루트와 분리합니다.
민감정보가 빌드 산출물에 섞이지 않도록 `.env`와 예시 파일을 나누는 방식은 [Node.js loadEnvFile 가이드](/development/blog/seo/2026/05/03/nodejs-loadenvfile-built-in-env-management-guide.html)와도 연결됩니다.

## 배포 파이프라인 체크리스트

### 빌드 결과에 map 파일이 있는지 검사한다

CI에서 가장 먼저 할 일은 산출물 존재 여부를 확인하는 것입니다.
다음 스크립트는 `dist` 안의 JavaScript 파일에 대응하는 `.map` 파일이 있는지 간단히 검사합니다.

```js
import { readdir, stat } from 'node:fs/promises';
import { join } from 'node:path';

async function walk(dir) {
  const entries = await readdir(dir);
  const files = [];

  for (const entry of entries) {
    const path = join(dir, entry);
    const info = await stat(path);

    if (info.isDirectory()) {
      files.push(...await walk(path));
    } else {
      files.push(path);
    }
  }

  return files;
}

const files = await walk('dist');
const jsFiles = files.filter((file) => file.endsWith('.js'));
const missing = jsFiles.filter((file) => !files.includes(`${file}.map`));

if (missing.length > 0) {
  console.error('missing source maps:', missing);
  process.exitCode = 1;
}
```

번들 결과가 일부 파일에만 source map을 만드는 구조라면 예외 목록을 명시해야 합니다.
중요한 것은 “없어도 그냥 넘어간다”가 아니라 “의도적으로 없는 파일과 실수로 빠진 파일을 구분한다”는 점입니다.

### 배포 후 실제 실행 옵션을 확인한다

빌드 산출물이 올바르더라도 운영 실행 명령에 `--enable-source-maps`가 빠지면 매핑은 동작하지 않습니다.
프로세스 시작 로그에 `process.execArgv`를 남기면 배포 후 옵션 누락을 빠르게 확인할 수 있습니다.

```js
console.info('node runtime options', {
  node: process.version,
  execArgv: process.execArgv,
  sourceMapsEnabled: process.execArgv.includes('--enable-source-maps') ||
    process.env.NODE_OPTIONS?.includes('--enable-source-maps') === true,
});
```

이 로그에는 비밀값을 넣지 말아야 합니다.
`NODE_OPTIONS` 자체는 보통 민감정보가 아니지만, 환경 변수 전체를 통째로 출력하는 습관은 위험합니다.
런타임 관측 항목을 정리할 때는 [Node.js diagnostics_channel 가이드](/development/blog/seo/2026/05/18/nodejs-diagnostics-channel-observability-guide.html)처럼 필요한 신호만 구조화하는 편이 좋습니다.

## 장애 대응 흐름에 연결하기

### 에러 cause와 함께 보면 문맥이 살아난다

source map은 “어디서” 실패했는지를 알려 줍니다.
하지만 “왜” 실패했는지는 에러 메시지, `cause`, 요청 ID, 외부 API 응답 상태 같은 문맥이 있어야 보입니다.
운영 코드에서는 낮은 수준의 에러를 그대로 던지기보다, 도메인 에러로 감싸면서 원인을 보존하는 편이 좋습니다.

```js
class PaymentCaptureError extends Error {
  constructor(orderId, cause) {
    super(`failed to capture payment for order ${orderId}`, { cause });
    this.name = 'PaymentCaptureError';
    this.orderId = orderId;
  }
}

async function capturePayment(order) {
  try {
    return await callPaymentGateway(order);
  } catch (error) {
    throw new PaymentCaptureError(order.id, error);
  }
}
```

이렇게 해 두면 source map으로 원본 라인을 찾고, `cause`로 하위 실패 원인을 따라갈 수 있습니다.
둘 중 하나만 있으면 조사가 반쪽이 됩니다.

### 릴리스마다 source map 보존 기간을 정한다

운영 장애는 배포 직후에만 발생하지 않습니다.
며칠 전 배포된 버전에서 뒤늦게 에러가 드러날 수 있습니다.
따라서 source map은 현재 버전 하나만 남기기보다, 최근 릴리스 몇 개 또는 일정 기간 동안 보존하는 정책이 필요합니다.

컨테이너 이미지를 릴리스 단위로 보관한다면 source map도 이미지 안에 함께 들어 있으므로 추적이 쉽습니다.
별도 아티팩트 저장소에 올린다면 커밋 SHA, 빌드 번호, 배포 환경을 키로 묶어야 합니다.
무중단 배포와 롤백 흐름을 관리한다면 [Node.js canary deployment 가이드](/development/blog/seo/2026/03/17/canary-deployment-nodejs-feature-flag-observability-guide.html)처럼 버전별 관측 신호를 같이 남기는 것이 좋습니다.

## 자주 묻는 질문

### source map을 켜면 성능이 크게 느려지나요?

일반 요청 처리 경로가 매번 느려지는 기능은 아닙니다.
주로 에러 스택을 만들고 해석할 때 영향을 줍니다.
다만 에러가 매우 많이 발생하는 서비스에서는 스택 생성 자체가 비용이 될 수 있으므로, 반복 에러를 줄이고 샘플링 정책을 함께 두는 것이 좋습니다.

### 운영 서버에 .map 파일을 꼭 배포해야 하나요?

`--enable-source-maps`로 런타임에서 스택을 매핑하려면 Node.js가 참조할 수 있는 source map이 필요합니다.
외부 에러 수집 도구에서 별도로 매핑한다면 서버에는 두지 않고 아티팩트 저장소나 에러 도구에 업로드하는 방식도 가능합니다.
중요한 것은 어느 쪽이든 릴리스 버전과 정확히 연결되어야 한다는 점입니다.

### inline source map을 써도 되나요?

로컬 개발에서는 편하지만 운영에서는 신중해야 합니다.
inline source map이나 `inlineSources`는 산출물 안에 원본 코드가 더 많이 들어갈 수 있습니다.
서버 내부 전용 이미지라면 허용할 수 있지만, 정적 공개 경로나 외부 로그 시스템으로 흘러갈 가능성이 있다면 별도 `.map` 파일과 접근 제어를 권장합니다.

## 마무리

Node.js source maps는 TypeScript와 번들러를 쓰는 서비스에서 운영 디버깅 시간을 줄여 주는 기본 장치입니다.
`--enable-source-maps`를 켜고, 빌드 도구가 `.map` 파일을 만들며, 배포 산출물에 필요한 파일이 포함되는지 확인해야 합니다.
동시에 원본 경로와 코드가 어디까지 노출되는지 정하고, 로그 마스킹과 아티팩트 보존 정책을 함께 운영해야 합니다.

추천 순서는 간단합니다.
먼저 로컬과 스테이징에서 source map 매핑을 확인하고, 다음으로 CI에서 map 파일 누락을 검사합니다.
그다음 운영 실행 옵션과 로그 마스킹을 점검하면, 장애가 발생했을 때 `dist`의 난해한 줄 번호가 아니라 실제 원본 TypeScript 위치에서 조사를 시작할 수 있습니다.

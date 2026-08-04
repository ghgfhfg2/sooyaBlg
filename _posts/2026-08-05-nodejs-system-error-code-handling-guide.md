---
layout: post
title: "Node.js 시스템 오류 코드 가이드: ENOENT와 EACCES를 안전하게 처리하는 법"
date: 2026-08-05 08:00:00 +0900
lang: ko
translation_key: nodejs-system-error-code-handling-guide
permalink: /development/blog/seo/2026/08/05/nodejs-system-error-code-handling-guide.html
alternates:
  ko: /development/blog/seo/2026/08/05/nodejs-system-error-code-handling-guide.html
  x_default: /development/blog/seo/2026/08/05/nodejs-system-error-code-handling-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, error-handling, errno, enoent, eacces, filesystem, backend, javascript]
description: "Node.js 시스템 오류 코드 ENOENT, EACCES, ENOSPC, EMFILE을 실무에서 안전하게 처리하는 방법을 정리합니다. 파일 작업 실패 분류, 사용자 메시지 변환, 재시도 기준, 로그 마스킹까지 예제로 설명합니다."
---

Node.js에서 파일을 읽고, 디렉터리를 만들고, 임시 파일을 옮기는 코드는 대부분 운영 환경을 만나는 순간 실패 가능성을 갖습니다.
파일이 없을 수도 있고, 권한이 부족할 수도 있으며, 디스크가 가득 찼거나 너무 많은 파일을 동시에 열었을 수도 있습니다.
이런 실패를 모두 `Error: failed`처럼 뭉뚱그리면 사용자는 무엇을 고쳐야 하는지 알 수 없고, 운영자는 장애 원인을 다시 로그에서 추적해야 합니다.

Node.js 시스템 오류에는 보통 `code`, `errno`, `syscall`, `path` 같은 단서가 들어 있습니다.
특히 `ENOENT`, `EACCES`, `ENOSPC`, `EMFILE` 같은 `code`는 오류를 분류하고 메시지를 다듬는 기준점이 됩니다.
이 글에서는 Node.js 시스템 오류 코드를 실무 서비스와 CLI에서 안전하게 다루는 방법을 정리합니다.

파일 존재 여부를 먼저 확인하는 흐름은 [Node.js fsPromises.access 파일 권한 확인 가이드](/development/blog/seo/2026/07/30/nodejs-fspromises-access-file-permission-check-guide.html)를 함께 보면 좋습니다.
파일을 완성한 뒤 공개 경로로 바꾸는 패턴은 [Node.js fsPromises.rename 원자적 배포 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-rename-atomic-publish-guide.html)와 연결됩니다.
디스크 여유 공간까지 점검해야 한다면 [Node.js fsPromises.statfs 디스크 공간 확인 가이드](/development/blog/seo/2026/07/30/nodejs-fspromises-statfs-disk-space-check-guide.html)를 참고하세요.

## 시스템 오류 코드를 분류해야 하는 이유

### 같은 파일 작업 실패도 대응이 다르다

파일 읽기 실패는 하나의 사건처럼 보이지만 원인은 여러 가지입니다.
파일이 없는 `ENOENT`는 경로 계산이나 입력값 문제일 수 있고, 권한 문제인 `EACCES`는 배포 사용자나 볼륨 마운트 설정을 봐야 합니다.
디스크가 가득 찬 `ENOSPC`는 애플리케이션 코드보다 운영 환경의 용량 관리가 먼저입니다.

```js
import { readFile } from 'node:fs/promises';

export async function readConfig(path) {
  try {
    return await readFile(path, 'utf8');
  } catch (error) {
    if (error?.code === 'ENOENT') {
      throw new Error(`Config file was not found: ${path}`);
    }

    if (error?.code === 'EACCES') {
      throw new Error(`Config file is not readable: ${path}`);
    }

    throw error;
  }
}
```

이 코드는 모든 실패를 성공처럼 삼키지 않습니다.
대신 자주 발생하는 실패만 사용자가 이해할 수 있는 메시지로 바꾸고, 나머지는 원래 오류를 유지합니다.
알 수 없는 오류까지 임의로 바꾸면 스택과 원인 정보가 사라져 디버깅이 더 어려워질 수 있습니다.

### code를 기준으로 정책을 만든다

`message` 문자열을 파싱하는 방식은 피하는 편이 좋습니다.
운영체제, Node.js 버전, 실행 환경에 따라 메시지 문구가 조금씩 달라질 수 있기 때문입니다.
반면 `error.code`는 `ENOENT`처럼 비교적 안정적인 분류 키로 쓰기 좋습니다.

```js
export function classifySystemError(error) {
  switch (error?.code) {
    case 'ENOENT':
      return 'not_found';
    case 'EACCES':
    case 'EPERM':
      return 'permission_denied';
    case 'ENOSPC':
      return 'storage_full';
    case 'EMFILE':
    case 'ENFILE':
      return 'file_descriptor_limit';
    default:
      return 'unknown';
  }
}
```

분류 함수는 로깅, 사용자 메시지, 재시도 정책에서 함께 재사용할 수 있습니다.
핵심은 `code`를 곧바로 화면에 노출하지 않고, 서비스가 이해하는 도메인 오류로 한 번 번역하는 것입니다.
이렇게 하면 API 응답과 CLI 출력의 표현을 일관되게 관리할 수 있습니다.

## 사용자 메시지와 운영 로그를 분리한다

### 사용자에게는 다음 행동을 알려 준다

사용자 메시지는 내부 스택을 보여 주는 곳이 아닙니다.
대신 무엇이 잘못됐고 어떤 입력이나 환경을 확인하면 되는지를 짧게 알려야 합니다.
예를 들어 `ENOENT`는 "파일이 없습니다"에서 끝나기보다 "경로를 확인하세요"까지 말해 주는 편이 낫습니다.

```js
export function toUserMessage(error) {
  const kind = classifySystemError(error);

  if (kind === 'not_found') {
    return '파일을 찾을 수 없습니다. 경로와 파일 이름을 확인하세요.';
  }

  if (kind === 'permission_denied') {
    return '파일에 접근할 권한이 없습니다. 실행 사용자와 권한 설정을 확인하세요.';
  }

  if (kind === 'storage_full') {
    return '저장 공간이 부족합니다. 불필요한 파일을 정리한 뒤 다시 시도하세요.';
  }

  if (kind === 'file_descriptor_limit') {
    return '동시에 열린 파일이 너무 많습니다. 잠시 후 다시 시도하세요.';
  }

  return '파일 작업 중 알 수 없는 오류가 발생했습니다.';
}
```

이 메시지는 제품 화면이나 CLI 출력에 사용할 수 있습니다.
사용자의 다음 행동이 분명해야 지원 요청도 줄어듭니다.
다만 파일 경로가 개인정보나 내부 서버 구조를 담을 수 있다면 화면에는 전체 경로를 그대로 보여 주지 않는 편이 안전합니다.

### 로그에는 원인 단서를 남기되 민감정보를 줄인다

운영 로그에는 문제를 재현할 수 있는 단서가 필요합니다.
하지만 사용자 홈 디렉터리, 업로드 파일명, 토큰이 섞인 URL을 그대로 남기면 개인정보와 민감정보 노출 위험이 생깁니다.
로그에는 분류, syscall, 마스킹된 path 정도만 남기는 구조가 현실적입니다.

```js
import { basename } from 'node:path';

export function toSafeLogFields(error) {
  return {
    errorCode: error?.code ?? 'UNKNOWN',
    errorKind: classifySystemError(error),
    syscall: error?.syscall,
    fileName: error?.path ? basename(error.path) : undefined
  };
}
```

파일명 자체도 민감할 수 있는 서비스라면 `basename()` 대신 해시나 내부 리소스 ID를 남기는 방식이 더 좋습니다.
로그의 목표는 "무엇이 반복되는지"를 보는 것이지, 사용자 데이터를 그대로 보관하는 것이 아닙니다.
특히 업로드, 백업, 변환 작업처럼 사용자 파일을 다루는 서비스에서는 로그 필드 설계를 별도로 검토해야 합니다.

## 재시도할 오류와 중단할 오류를 나눈다

### ENOENT와 EACCES는 보통 즉시 재시도 대상이 아니다

파일이 없거나 권한이 없는 상황은 같은 코드를 곧바로 다시 실행한다고 해결될 가능성이 낮습니다.
`ENOENT`는 경로 생성 순서, 입력값, 배포 파일 누락을 확인해야 하고, `EACCES`는 권한과 실행 사용자를 확인해야 합니다.
무작정 재시도하면 같은 오류 로그만 늘어나고 실제 원인은 가려집니다.

```js
export function shouldRetrySystemError(error) {
  const kind = classifySystemError(error);

  if (kind === 'not_found' || kind === 'permission_denied') {
    return false;
  }

  if (kind === 'file_descriptor_limit') {
    return true;
  }

  return false;
}
```

재시도는 "일시적일 수 있는 실패"에만 붙이는 편이 좋습니다.
파일 디스크립터 한도처럼 순간적인 동시성 문제는 짧은 backoff 뒤 성공할 수 있습니다.
반대로 권한 오류는 운영 설정이 바뀌기 전까지 반복 실패할 가능성이 큽니다.

### ENOSPC는 재시도보다 보호 장치가 먼저다

디스크 부족은 재시도보다 빠른 실패와 알림이 중요합니다.
작업을 계속 밀어 넣으면 임시 파일, 로그, 데이터베이스 쓰기까지 연쇄적으로 실패할 수 있습니다.
따라서 `ENOSPC`를 만나면 큐 처리량을 줄이거나, 추가 쓰기를 중단하거나, 운영 알림을 보내는 정책이 필요합니다.

```js
export function shouldPauseQueue(error) {
  return classifySystemError(error) === 'storage_full';
}
```

대용량 파일을 다루는 배치라면 작업 시작 전에 여유 공간을 확인하는 편이 더 낫습니다.
하지만 사전 확인만으로는 충분하지 않습니다.
확인 직후 다른 프로세스가 디스크를 사용할 수 있으므로, 실제 쓰기 실패도 반드시 처리해야 합니다.

## API와 CLI에 적용하는 패턴

### API 응답은 내부 오류를 감춘다

서버 API에서는 시스템 오류를 그대로 JSON으로 내보내지 않는 편이 안전합니다.
대신 내부 로그에는 원인을 남기고, 클라이언트에는 상태 코드와 짧은 메시지를 돌려줍니다.
파일이 없는 경우가 사용자 입력 문제라면 404나 400 계열로, 서버 저장소 문제라면 500 계열로 분리할 수 있습니다.

```js
export function toHttpError(error) {
  const kind = classifySystemError(error);

  if (kind === 'not_found') {
    return { status: 404, message: '요청한 파일을 찾을 수 없습니다.' };
  }

  if (kind === 'permission_denied') {
    return { status: 403, message: '파일에 접근할 수 없습니다.' };
  }

  if (kind === 'storage_full') {
    return { status: 507, message: '저장 공간이 부족해 요청을 처리할 수 없습니다.' };
  }

  return { status: 500, message: '파일 처리 중 오류가 발생했습니다.' };
}
```

상태 코드는 서비스 정책에 맞게 조정해야 합니다.
예를 들어 사용자가 소유하지 않은 파일을 요청했을 때 403을 줄지, 존재 여부를 감추기 위해 404를 줄지는 제품 보안 정책에 따라 달라집니다.
중요한 것은 내부 `path`, `syscall`, 스택을 그대로 응답에 싣지 않는 것입니다.

### CLI는 실패 코드와 짧은 힌트를 남긴다

CLI에서는 사용자가 바로 다음 명령을 실행할 수 있도록 실패 메시지를 구체적으로 쓰는 편이 좋습니다.
동시에 자동화에서 실패를 감지할 수 있도록 `process.exitCode`를 설정해야 합니다.
메시지만 출력하고 0으로 끝나면 CI나 cron은 성공으로 착각합니다.

```js
export async function runCli(task) {
  try {
    await task();
  } catch (error) {
    console.error(toUserMessage(error));
    process.exitCode = 1;
  }
}
```

운영 CLI라면 `--verbose` 옵션에서만 자세한 로그 필드를 보여 주는 방식도 좋습니다.
기본 출력은 짧게, 디버그 출력은 구조화해서 남기면 사람이 읽는 경험과 자동화 분석을 모두 챙길 수 있습니다.
민감한 경로나 입력값을 출력할 때는 verbose 모드에서도 마스킹 기준을 유지해야 합니다.

## 테스트 기준

### code 기반 분류를 단위 테스트한다

시스템 오류를 실제로 일으키는 통합 테스트도 의미가 있지만, 분류 함수는 작은 객체로 단위 테스트하는 편이 빠르고 안정적입니다.
Node.js 오류 객체와 완전히 같은 인스턴스일 필요는 없습니다.
정책 함수가 참조하는 필드만 갖춘 객체로 충분합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { classifySystemError, shouldRetrySystemError } from './system-error-policy.js';

test('classifies missing files', () => {
  assert.equal(classifySystemError({ code: 'ENOENT' }), 'not_found');
  assert.equal(shouldRetrySystemError({ code: 'ENOENT' }), false);
});

test('retries file descriptor pressure', () => {
  assert.equal(classifySystemError({ code: 'EMFILE' }), 'file_descriptor_limit');
  assert.equal(shouldRetrySystemError({ code: 'EMFILE' }), true);
});
```

이 테스트는 OS별 파일 권한 차이에 덜 흔들립니다.
실제 파일 시스템 통합 테스트는 대표 경로만 별도로 두고, 핵심 정책은 순수 함수로 고정하는 편이 유지보수에 좋습니다.
오류 분류가 API 응답과 CLI 출력의 공통 기반이라면 이 테스트의 가치는 더 커집니다.

### 로그 마스킹도 테스트 대상이다

민감정보 보호는 문서만으로 지켜지지 않습니다.
로그 필드 변환 함수가 전체 경로를 남기지 않는지 테스트해 두면 회귀를 줄일 수 있습니다.
특히 업로드 파일명, 임시 디렉터리, 사용자 홈 경로를 다루는 서비스에서는 이런 테스트가 실무적으로 중요합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { toSafeLogFields } from './system-error-policy.js';

test('does not expose full paths in log fields', () => {
  const fields = toSafeLogFields({
    code: 'EACCES',
    syscall: 'open',
    path: '/home/app/private/report.csv'
  });

  assert.equal(fields.fileName, 'report.csv');
  assert.equal(JSON.stringify(fields).includes('/home/app/private'), false);
});
```

테스트 이름에는 기대하는 보안 속성을 직접 적는 편이 좋습니다.
나중에 누군가 로그 필드를 늘릴 때도 "전체 경로를 노출하지 않는다"는 의도가 남아 있기 때문입니다.
보안 기준은 코드 리뷰에서만 기억하는 것보다 테스트로 묶어 두는 편이 더 오래갑니다.

## 마무리

Node.js 시스템 오류 코드는 단순한 예외 문자열이 아니라 운영 판단에 필요한 신호입니다.
`ENOENT`, `EACCES`, `ENOSPC`, `EMFILE`을 분류하면 사용자 메시지, HTTP 상태 코드, 재시도 정책, 운영 로그를 일관되게 만들 수 있습니다.
반대로 모든 오류를 하나의 메시지로 감추면 장애 분석과 사용자 지원이 모두 느려집니다.

실무에서는 먼저 `error.code` 기반 분류 함수를 만들고, 화면 출력과 로그 출력을 분리하세요.
재시도는 일시적 오류에만 제한하고, 권한이나 디스크 부족처럼 설정과 운영 조치가 필요한 오류는 빠르게 드러내는 편이 좋습니다.
마지막으로 전체 경로와 사용자 입력이 로그에 남지 않는지 테스트까지 붙이면 파일 작업 실패를 훨씬 안정적으로 다룰 수 있습니다.

## 함께 보면 좋은 글

- [Node.js fsPromises.access 파일 권한 확인 가이드](/development/blog/seo/2026/07/30/nodejs-fspromises-access-file-permission-check-guide.html)
- [Node.js fsPromises.rename 원자적 배포 가이드](/development/blog/seo/2026/08/01/nodejs-fspromises-rename-atomic-publish-guide.html)
- [Node.js fsPromises.statfs 디스크 공간 확인 가이드](/development/blog/seo/2026/07/30/nodejs-fspromises-statfs-disk-space-check-guide.html)

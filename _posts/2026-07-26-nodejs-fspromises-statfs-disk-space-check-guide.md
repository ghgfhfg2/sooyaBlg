---
layout: post
title: "Node.js fs.promises.statfs 가이드: 디스크 여유 공간을 배포 전에 확인하는 법"
date: 2026-07-26 20:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-statfs-disk-space-check-guide
permalink: /development/blog/seo/2026/07/26/nodejs-fspromises-statfs-disk-space-check-guide.html
alternates:
  ko: /development/blog/seo/2026/07/26/nodejs-fspromises-statfs-disk-space-check-guide.html
  x_default: /development/blog/seo/2026/07/26/nodejs-fspromises-statfs-disk-space-check-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, statfs, disk-space, filesystem, deployment, backend, automation]
description: "Node.js fs.promises.statfs()로 파일 시스템의 여유 공간을 계산하고, 배포·업로드·배치 작업 전에 디스크 부족을 안전하게 감지하는 방법을 정리합니다. bigint 옵션, 임계값 설계, 테스트와 운영 체크리스트까지 실무 예제로 설명합니다."
---

배포 스크립트, 이미지 변환 작업, 로그 압축 배치, 대용량 업로드 처리처럼 파일을 많이 쓰는 Node.js 작업은 디스크 여유 공간에 의존합니다.
문제는 디스크가 부족해지는 순간이 대개 작업 중간이라는 점입니다.
아카이브를 풀다가 실패하거나, 임시 파일을 만들다 멈추거나, 업로드 파일을 절반만 저장한 뒤 정리 로직까지 실패하면 원인 파악이 늦어집니다.

이 글에서는 Node.js `fs.promises.statfs()`로 파일 시스템의 여유 공간을 확인하고, 위험한 쓰기 작업 전에 빠르게 중단하는 패턴을 정리합니다.
핵심은 **파일 하나의 크기와 파일 시스템 전체의 남은 공간을 구분하고, 필요한 여유분을 작업 시작 전에 명시적으로 검사하는 것**입니다.
[Node.js fs.promises.opendir 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html), [Node.js fs.promises.glob 가이드](/development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html), [Node.js Permission Model 가이드](/development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html)와 함께 보면 파일 시스템 자동화를 더 안정적으로 설계할 수 있습니다.

## fs.promises.statfs가 필요한 이유

### H3. stat은 파일을 보고 statfs는 파일 시스템을 본다

`fs.promises.stat()`은 특정 파일이나 디렉터리의 정보를 알려줍니다.
예를 들어 파일 크기, 수정 시간, 파일 타입 같은 값이 필요할 때 사용합니다.

```js
import { stat } from 'node:fs/promises';

const info = await stat('dist/app.tar.gz');

console.log(info.size);
```

반면 `fs.promises.statfs()`는 해당 경로가 올라가 있는 파일 시스템의 정보를 반환합니다.
Node.js 공식 문서 기준으로 `statfs(path, options)`는 주어진 경로가 포함된 mounted file system 정보를 `fs.StatFs` 객체로 돌려줍니다.
즉 "이 파일이 얼마나 큰가"가 아니라 "이 경로에 앞으로 얼마나 더 쓸 수 있는가"를 볼 때 쓰는 API입니다.

```js
import { statfs } from 'node:fs/promises';

const stats = await statfs('/var/app/uploads');

const availableBytes = stats.bsize * stats.bavail;

console.log(availableBytes);
```

운영 코드에서는 이 차이가 중요합니다.
업로드 대상 디렉터리의 파일 수와 현재 파일 크기를 잘 관리하더라도, 같은 디스크를 쓰는 로그나 다른 프로세스가 공간을 차지하면 새 작업은 실패할 수 있습니다.
`statfs()`는 그런 위험을 작업 시작 전에 확인하는 작은 방어선이 됩니다.

### H3. 디스크 부족은 늦게 발견할수록 비싸다

디스크 부족은 단순한 실패가 아니라 후속 문제를 함께 만듭니다.

- 임시 파일이 남아 다음 작업의 공간을 더 줄인다.
- 부분 생성된 산출물이 정상 파일처럼 보일 수 있다.
- 로그 기록이 실패해 실제 원인을 잃는다.
- 재시도가 같은 디스크를 더 압박한다.
- 여러 worker가 동시에 쓰면 한 작업의 예상치가 쉽게 깨진다.

그래서 큰 파일을 쓰기 전에 "최소한 이 정도 공간은 남아 있어야 한다"는 기준을 코드로 표현하는 편이 좋습니다.
`statfs()`는 완벽한 예약 시스템은 아니지만, 위험한 작업을 시작하기 전에 빠른 fail-fast 판단을 내리게 해 줍니다.

## statfs 기본 사용법

### H3. bavail과 bsize를 곱해 사용 가능한 바이트를 계산한다

`fs.StatFs` 객체에는 여러 숫자 필드가 들어 있습니다.
디스크 여유 공간 판단에서 가장 자주 쓰는 값은 `bavail`과 `bsize`입니다.

- `bavail`: 일반 사용자에게 사용 가능한 free block 수
- `bfree`: 파일 시스템의 free block 수
- `blocks`: 전체 data block 수
- `bsize`: 전송 블록 크기
- `files`: 전체 file node 수
- `ffree`: 남은 file node 수

실무에서는 일반 애플리케이션 프로세스가 실제로 쓸 수 있는 공간을 보고 싶기 때문에 `bavail`을 우선 사용합니다.
계산은 단순합니다.

```js
import { statfs } from 'node:fs/promises';

export async function getAvailableBytes(path) {
  const stats = await statfs(path);

  return stats.bsize * stats.bavail;
}

console.log(await getAvailableBytes('/var/app/uploads'));
```

`bfree`는 시스템 관점의 free block에 가까워서, 운영 사용자에게 예약된 영역까지 포함될 수 있습니다.
애플리케이션이 실제 쓰기 가능성을 판단하려면 `bavail`을 기준으로 잡는 편이 더 보수적입니다.

### H3. 사람이 읽을 수 있는 단위는 표시용으로만 변환한다

운영 판단은 바이트 단위 숫자로 하고, 로그나 에러 메시지에서만 MB, GB로 변환하는 편이 안전합니다.
중간 계산을 문자열로 바꾸면 비교 로직이 흐려지기 쉽습니다.

```js
export function formatBytes(bytes) {
  const units = ['B', 'KiB', 'MiB', 'GiB', 'TiB'];
  let value = Number(bytes);
  let unitIndex = 0;

  while (value >= 1024 && unitIndex < units.length - 1) {
    value /= 1024;
    unitIndex += 1;
  }

  return `${value.toFixed(value >= 10 ? 1 : 2)} ${units[unitIndex]}`;
}
```

예를 들어 10GB 이상의 여유 공간이 필요한 작업이라면 비교 기준은 숫자로 둡니다.

```js
const minFreeBytes = 10 * 1024 * 1024 * 1024;

if (availableBytes < minFreeBytes) {
  throw new Error(
    `not enough disk space: required=${formatBytes(minFreeBytes)} available=${formatBytes(availableBytes)}`
  );
}
```

에러 메시지에는 절대 경로나 사용자 식별자를 그대로 넣지 않는 편이 좋습니다.
디스크 경로는 배포 환경 구조를 드러낼 수 있으므로, 공개 로그나 사용자 응답에는 짧은 작업 이름만 남기는 방식을 권장합니다.

## 배포 전 디스크 공간 검사 만들기

### H3. 작업별 최소 여유 공간을 코드로 표현한다

배포 산출물을 만들거나 압축을 푸는 작업은 결과 파일 크기보다 더 많은 임시 공간을 요구할 수 있습니다.
예를 들어 500MB 아카이브를 풀면 실제 디렉터리는 2GB가 될 수 있고, 빌드 중간에는 캐시와 임시 파일까지 생깁니다.
따라서 "예상 출력 크기"와 "운영 안전 여유분"을 나눠 계산하는 편이 좋습니다.

```js
import { statfs } from 'node:fs/promises';

const GiB = 1024 * 1024 * 1024;

export async function assertDiskBudget(path, options = {}) {
  const {
    expectedWriteBytes,
    safetyBytes = 2 * GiB,
    label = 'file operation'
  } = options;

  const stats = await statfs(path);
  const availableBytes = stats.bsize * stats.bavail;
  const requiredBytes = expectedWriteBytes + safetyBytes;

  if (availableBytes < requiredBytes) {
    throw new Error(
      `${label} needs ${formatBytes(requiredBytes)}, only ${formatBytes(availableBytes)} is available`
    );
  }

  return {
    availableBytes,
    requiredBytes
  };
}
```

이 함수는 쓰기 작업 전에 호출합니다.

```js
await assertDiskBudget('/var/app/releases', {
  expectedWriteBytes: 3 * GiB,
  safetyBytes: 5 * GiB,
  label: 'release build'
});

// build, copy, extract, upload 같은 실제 쓰기 작업은 그 다음에 실행한다.
```

중요한 점은 `statfs()` 호출과 실제 쓰기 사이에 다른 프로세스가 디스크를 사용할 수 있다는 사실입니다.
따라서 이 검사는 "성공 보장"이 아니라 "명백히 위험한 작업을 미리 막는 장치"로 봐야 합니다.

### H3. 업로드 요청은 Content-Length를 먼저 활용한다

HTTP 업로드 서버라면 요청 본문을 모두 받은 뒤 공간 부족을 발견하는 것보다, 가능한 경우 `Content-Length`를 먼저 사용해 대략적인 필요 공간을 계산하는 편이 낫습니다.

```js
import http from 'node:http';

const uploadDir = '/var/app/uploads';
const maxUploadBytes = 200 * 1024 * 1024;
const safetyBytes = 1024 * 1024 * 1024;

http.createServer(async (req, res) => {
  if (req.method !== 'POST' || req.url !== '/upload') {
    res.writeHead(404);
    res.end('not found');
    return;
  }

  const contentLength = Number(req.headers['content-length'] ?? 0);

  if (!Number.isSafeInteger(contentLength) || contentLength <= 0) {
    res.writeHead(411);
    res.end('content length required');
    return;
  }

  if (contentLength > maxUploadBytes) {
    res.writeHead(413);
    res.end('payload too large');
    return;
  }

  try {
    await assertDiskBudget(uploadDir, {
      expectedWriteBytes: contentLength,
      safetyBytes,
      label: 'upload'
    });
  } catch {
    res.writeHead(507);
    res.end('insufficient storage');
    return;
  }

  res.writeHead(202);
  res.end('accepted');
}).listen(3000);
```

여기서 `507 Insufficient Storage`는 서버가 요청을 저장할 공간이 부족하다는 의미를 표현하기에 적절합니다.
다만 클라이언트가 `Content-Length`를 보내지 않거나 압축, chunked transfer, 멀티파트 오버헤드가 붙으면 실제 사용량은 달라질 수 있습니다.
업로드 본문을 저장하는 코드에서도 별도 byte limit을 반드시 유지해야 합니다.

## bigint 옵션과 큰 파일 시스템

### H3. 큰 볼륨에서는 bigint 옵션을 고려한다

Node.js `statfs()`는 `options.bigint`를 받을 수 있습니다.
이 값을 `true`로 설정하면 반환 객체의 숫자 필드가 `number` 대신 `bigint`가 됩니다.

```js
import { statfs } from 'node:fs/promises';

const stats = await statfs('/var/app/data', { bigint: true });

const availableBytes = stats.bsize * stats.bavail;

console.log(availableBytes);
```

일반적인 애플리케이션 서버에서는 `number`로도 충분한 경우가 많습니다.
하지만 매우 큰 스토리지, 장기 보관 시스템, 데이터 처리 서버처럼 값이 커질 수 있는 환경에서는 `bigint`가 더 안전합니다.

주의할 점은 `bigint`와 `number`를 바로 섞어 계산할 수 없다는 것입니다.

```js
const GiB = 1024n * 1024n * 1024n;
const minFreeBytes = 50n * GiB;

if (availableBytes < minFreeBytes) {
  throw new Error('not enough disk space');
}
```

로그 표시를 위해 `Number()`로 바꿀 때도 값이 안전 범위를 넘지 않는지 고려해야 합니다.
운영 판단은 `bigint` 그대로 하고, 표시 함수만 `bigint`를 받을 수 있게 만드는 편이 깔끔합니다.

### H3. number와 bigint 버전을 분리하면 호출부가 단순해진다

공용 유틸에서 `number | bigint`를 동시에 반환하면 호출부가 계속 타입 분기를 해야 합니다.
프로젝트 안에서는 둘 중 하나를 표준으로 정하는 편이 낫습니다.

```js
import { statfs } from 'node:fs/promises';

export async function getDiskSpace(path) {
  const stats = await statfs(path, { bigint: true });

  return {
    availableBytes: stats.bsize * stats.bavail,
    freeBytes: stats.bsize * stats.bfree,
    totalBytes: stats.bsize * stats.blocks
  };
}
```

이렇게 반환 타입을 고정하면 테스트와 모니터링 코드도 단순해집니다.
특히 여러 작업에서 같은 임계값 정책을 공유한다면 `bigint` 기반 유틸 하나로 통일하는 것이 좋습니다.

## 운영 코드에서 주의할 점

### H3. statfs는 권한과 경로 검증을 대신하지 않는다

`statfs()`로 공간을 확인했다고 해서 그 경로에 파일을 쓸 수 있다는 뜻은 아닙니다.
디렉터리 권한, 소유자, 컨테이너 mount 설정, 읽기 전용 파일 시스템 여부는 별도 문제입니다.
쓰기 작업은 여전히 실패할 수 있습니다.

```js
import { access, constants } from 'node:fs/promises';

export async function assertWritableDirectory(path) {
  await access(path, constants.W_OK);
}
```

공간 검사와 권한 검사는 함께 두는 편이 좋습니다.

```js
await assertWritableDirectory('/var/app/uploads');
await assertDiskBudget('/var/app/uploads', {
  expectedWriteBytes: uploadSize,
  safetyBytes: 1024 * 1024 * 1024,
  label: 'upload'
});
```

사용자 입력으로 받은 경로를 그대로 `statfs()`에 넘기면 서버 내부 파일 시스템 구조를 탐색하는 기능이 될 수 있습니다.
업로드, export, 백업 같은 기능에서는 허용된 루트 아래의 경로만 검사하도록 제한해야 합니다.

### H3. 동시 쓰기 작업은 별도 한도를 둔다

`statfs()`는 현재 시점의 스냅샷입니다.
여러 worker가 동시에 2GB씩 쓰기 시작하면 각 worker가 검사할 때는 충분해 보여도 전체로는 부족할 수 있습니다.

따라서 다음 정책을 함께 두는 것이 좋습니다.

- 대용량 작업 동시 실행 수를 제한한다.
- 작업별 예상 쓰기 크기를 큐 메타데이터에 저장한다.
- 같은 디스크를 쓰는 작업은 하나의 semaphore로 묶는다.
- 실패한 작업의 임시 파일 정리 경로를 테스트한다.
- 재시도 전에 남은 공간을 다시 확인한다.

동시성 제어는 [Node.js concurrency limit 가이드](/development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)와 연결해서 설계할 수 있습니다.
디스크도 결국 공유 자원이므로 API 호출이나 DB 커넥션처럼 예산을 두고 다루는 편이 안전합니다.

## 테스트와 모니터링

### H3. 계산 로직은 순수 함수로 분리한다

`statfs()` 자체를 모든 테스트에서 실제 파일 시스템에 의존해 검증할 필요는 없습니다.
임계값 계산은 순수 함수로 분리하고, `statfs()`를 호출하는 얇은 경계만 통합 테스트로 확인하면 됩니다.

```js
export function evaluateDiskBudget({ availableBytes, expectedWriteBytes, safetyBytes }) {
  const requiredBytes = expectedWriteBytes + safetyBytes;

  return {
    ok: availableBytes >= requiredBytes,
    availableBytes,
    requiredBytes
  };
}
```

테스트는 작은 숫자로 충분합니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { evaluateDiskBudget } from './disk-budget.js';

test('fails when available space is below required budget', () => {
  const result = evaluateDiskBudget({
    availableBytes: 90,
    expectedWriteBytes: 80,
    safetyBytes: 20
  });

  assert.equal(result.ok, false);
  assert.equal(result.requiredBytes, 100);
});
```

실제 파일 시스템 통합 테스트는 환경에 따라 값이 달라지므로 "현재 디렉터리에서 양수 값을 반환한다" 정도로 제한하는 편이 안정적입니다.

```js
import assert from 'node:assert/strict';
import test from 'node:test';
import { getDiskSpace } from './disk-space.js';

test('reads disk space for the current working directory', async () => {
  const space = await getDiskSpace(process.cwd());

  assert.equal(typeof space.availableBytes, 'bigint');
  assert.ok(space.availableBytes > 0n);
});
```

### H3. 운영 알림은 비율과 절대값을 함께 본다

남은 공간을 모니터링할 때는 절대값과 비율을 함께 봐야 합니다.
1TB 볼륨의 5%는 50GB지만, 10GB 볼륨의 5%는 512MB 정도입니다.
작업 특성에 따라 어느 쪽이 더 중요한지가 달라집니다.

```js
export function summarizeDiskSpace({ availableBytes, totalBytes }) {
  const usedBytes = totalBytes - availableBytes;
  const usedRatio = Number(usedBytes * 10000n / totalBytes) / 100;

  return {
    availableBytes,
    totalBytes,
    usedRatio
  };
}
```

알림 기준은 다음처럼 나누면 실무에서 다루기 쉽습니다.

- 남은 공간이 작업 최소 요구량보다 작으면 작업 시작 전 실패 처리
- 남은 공간이 운영 기준보다 낮으면 경고 알림
- 사용률이 급격히 증가하면 배치나 로그 폭증 의심
- inode가 부족하면 작은 파일 누적이나 임시 파일 정리 실패 의심

`statfs.files`와 `statfs.ffree`는 inode에 해당하는 file node 관점의 값입니다.
작은 파일을 많이 만드는 시스템에서는 바이트 공간이 남아 있어도 inode 부족으로 파일 생성이 실패할 수 있으므로 함께 관찰할 가치가 있습니다.

## statfs 선택 기준

### H3. statfs가 잘 맞는 경우

`fs.promises.statfs()`는 다음 상황에서 특히 유용합니다.

- 큰 파일을 쓰기 전에 남은 디스크 공간을 확인한다.
- 업로드, 압축 해제, export, 백업 작업을 시작 전에 차단한다.
- 배포 서버의 release 디렉터리 공간을 점검한다.
- 로그 압축이나 정리 배치의 안전 기준을 만든다.
- inode 부족 가능성이 있는 작은 파일 누적 시스템을 관찰한다.

반대로 파일 하나의 크기나 수정 시간이 필요하다면 `stat()`이 맞습니다.
디렉터리 안의 파일 목록을 찾아야 한다면 `opendir()`나 `glob()`이 맞습니다.
`statfs()`는 파일 시스템 단위의 용량 정보를 볼 때 쓰는 도구입니다.

### H3. 운영 코드에 넣기 전 체크리스트

배포나 업로드 경로에 `statfs()` 검사를 넣기 전에는 아래 항목을 확인하세요.

- 검사 대상 경로가 실제 쓰기 경로와 같은 파일 시스템에 있는가?
- `bavail * bsize` 기준으로 일반 사용자에게 사용 가능한 공간을 계산하는가?
- 예상 쓰기 크기와 안전 여유분을 분리했는가?
- 큰 볼륨에서는 `bigint: true`를 사용할지 결정했는가?
- 권한 검사와 경로 allowlist를 함께 적용했는가?
- 동시 쓰기 작업 수와 재시도 정책을 제한했는가?
- 공간 부족 에러가 사용자에게 과도한 내부 경로를 노출하지 않는가?
- 부분 생성 파일과 임시 디렉터리 정리 테스트가 있는가?

이 기준을 정해 두면 디스크 공간 검사는 단순한 사전 확인이 아니라 배포와 파일 처리 작업의 안정성 경계가 됩니다.

## FAQ

### fs.promises.statfs와 fs.promises.stat은 어떻게 다른가요?

`stat()`은 파일이나 디렉터리 하나의 정보를 반환합니다.
`statfs()`는 그 경로가 위치한 파일 시스템의 용량, block, file node 정보를 반환합니다.
파일 크기는 `stat()`, 남은 디스크 공간은 `statfs()`를 쓰면 됩니다.

### bavail과 bfree 중 무엇을 써야 하나요?

애플리케이션이 실제로 사용할 수 있는 공간을 판단할 때는 `bavail`을 우선 추천합니다.
`bfree`는 파일 시스템 전체의 free block에 가까워서 예약 영역까지 포함될 수 있습니다.
운영 작업 차단 기준은 일반 사용자에게 사용 가능한 `bavail * bsize`가 더 보수적입니다.

### statfs로 확인하면 쓰기 작업이 반드시 성공하나요?

아닙니다.
`statfs()`는 현재 시점의 공간 정보만 보여줍니다.
권한 문제, 읽기 전용 mount, 동시 쓰기, 검사 이후 공간 변화 때문에 실제 쓰기는 여전히 실패할 수 있습니다.
그래서 공간 검사, 권한 검사, 동시성 제한, 실패 시 정리를 함께 설계해야 합니다.

### 디스크 공간 부족은 어떤 HTTP 상태 코드로 응답하나요?

업로드나 서버 저장 작업에서 저장 공간이 부족하면 `507 Insufficient Storage`를 사용할 수 있습니다.
다만 요청 크기 자체가 정책보다 크면 `413 Payload Too Large`가 더 정확합니다.
공간 부족 메시지에는 내부 절대 경로나 저장소 구조를 그대로 노출하지 않는 편이 안전합니다.

## 마무리

Node.js `fs.promises.statfs()`는 파일 시스템의 남은 공간을 코드 안에서 확인할 수 있게 해 주는 실용적인 API입니다.
큰 파일을 쓰는 작업, 배포 산출물 생성, 업로드 저장, 로그 압축처럼 디스크를 많이 쓰는 흐름에서는 작업 시작 전에 `bavail * bsize` 기준으로 여유 공간을 확인하는 것만으로도 실패를 더 일찍 발견할 수 있습니다.

다만 `statfs()`는 예약이나 락이 아닙니다.
검사 이후에도 공간은 변할 수 있으므로 동시성 제한, 권한 검사, 임시 파일 정리, 재시도 전 재검사를 함께 넣어야 합니다.
관련해서 파일 목록 수집은 [Node.js fs.promises.opendir 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html), 패턴 기반 검색은 [Node.js fs.promises.glob 가이드](/development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html), 공유 자원 동시성 제어는 [Node.js concurrency limit 가이드](/development/blog/seo/2026/04/03/nodejs-concurrency-limit-promise-pool-overload-control-guide.html)를 이어서 보면 좋습니다.

## 관련 글

- [Node.js fs.promises.opendir 가이드: 대량 파일 디렉터리 순회를 안전하게 처리하는 법](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html)
- [Node.js fs.promises.glob 가이드: 파일 검색을 의존성 없이 처리하는 법](/development/blog/seo/2026/05/28/nodejs-fs-promises-glob-file-search-guide.html)
- [Node.js Permission Model 가이드: 런타임 권한으로 파일·프로세스 접근을 제한하는 법](/development/blog/seo/2026/05/11/nodejs-permission-model-runtime-access-control-guide.html)
- [Node.js 공식 문서: File system API](https://nodejs.org/api/fs.html#fspromisesstatfspath-options)

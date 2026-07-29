---
layout: post
title: "Node.js statfs 가이드: 배치 작업 전 디스크 여유 공간을 확인하는 법"
date: 2026-07-30 08:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-statfs-disk-space-check-guide
permalink: /development/blog/seo/2026/07/30/nodejs-fspromises-statfs-disk-space-check-guide.html
alternates:
  ko: /development/blog/seo/2026/07/30/nodejs-fspromises-statfs-disk-space-check-guide.html
  x_default: /development/blog/seo/2026/07/30/nodejs-fspromises-statfs-disk-space-check-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, statfs, filesystem, disk-space, batch, cron, reliability, javascript]
description: "Node.js fs.promises.statfs로 배치 작업, 크론, 파일 생성 자동화 전에 디스크 여유 공간을 확인하는 방법을 정리합니다. bavail과 bfree 차이, bigint 옵션, 임계값 설계, 실패 처리와 테스트 기준까지 실무 예제로 설명합니다."
---

파일을 만드는 자동화는 디스크가 충분하다는 전제 위에서 움직입니다.
로그 압축, 리포트 생성, 이미지 변환, 백업, 크롤링 결과 저장, 정적 사이트 빌드처럼 파일을 많이 쓰는 작업은 여유 공간이 부족할 때 가장 애매하게 실패합니다.
처음부터 실패하면 차라리 낫지만, 중간까지 산출물을 만든 뒤 `ENOSPC`로 멈추면 임시 파일과 불완전한 결과까지 함께 정리해야 합니다.

Node.js에서는 `fs.promises.statfs()`로 특정 경로가 올라간 파일 시스템의 용량 정보를 확인할 수 있습니다.
공식 문서 기준 `statfs.bavail`은 일반 사용자에게 사용 가능한 free block 수이고, `statfs.bsize`와 곱하면 사용 가능한 바이트 수를 구할 수 있습니다.
이 글에서는 배치 작업 시작 전 디스크 여유 공간을 확인하는 패턴, `bavail`과 `bfree`의 차이, `bigint` 옵션을 언제 쓸지, 실패 시 운영 로그를 어떻게 남길지 정리합니다.

파일 쓰기 자체의 내구성이 필요하다면 [Node.js writeFile flush 가이드](/development/blog/seo/2026/07/29/nodejs-fspromises-writefile-flush-durable-write-guide.html)를 함께 보면 좋습니다.
중복 실행으로 같은 산출물을 동시에 쓰는 문제가 있다면 [Node.js lock file 가이드](/development/blog/seo/2026/07/28/nodejs-lock-file-atomic-job-deduplication-guide.html)를 먼저 적용하고, 임시 작업 디렉터리 관리는 [Node.js mkdtemp 가이드](/development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html)와 연결해서 설계할 수 있습니다.

## statfs가 필요한 상황

### H3. 쓰기 전에 실패 가능성을 줄인다

디스크 여유 공간 점검은 "완벽한 보장"이 아니라 "일찍 거절하기"에 가깝습니다.
점검 직후 다른 프로세스가 공간을 사용할 수 있으므로, `statfs()`를 통과했다고 해서 쓰기가 반드시 성공하는 것은 아닙니다.
그래도 큰 작업을 시작하기 전에 최소 요구량을 확인하면 실패 지점을 앞당길 수 있습니다.

특히 아래 작업은 사전 점검을 넣을 만합니다.

- 여러 파일을 만드는 정적 사이트 빌드
- 압축 파일, 리포트, CSV export 생성
- 이미지나 영상 썸네일 일괄 변환
- 크롤링 결과를 로컬 캐시에 저장하는 작업
- 큐 consumer가 임시 파일을 만든 뒤 업로드하는 흐름
- 백업 파일을 만든 뒤 원격 저장소로 옮기는 배치

핵심은 "쓰기 시작 후 실패"보다 "작업 시작 전 보류"가 복구하기 쉽다는 점입니다.
사전 점검에서 막히면 기존 파일은 그대로 두고, 작업 상태를 실패로 표시하고, 알림이나 재시도 정책으로 넘기면 됩니다.

### H3. path가 속한 파일 시스템을 기준으로 본다

`statfs(path)`는 그 경로가 위치한 파일 시스템의 정보를 반환합니다.
프로젝트 루트와 임시 디렉터리, 업로드 디렉터리, 마운트된 볼륨이 서로 다른 파일 시스템일 수 있습니다.
그래서 실제로 파일을 쓸 위치를 기준으로 확인해야 합니다.

예를 들어 빌드 결과는 `dist/`에 쓰고, 임시 변환 파일은 `/tmp`에 만들고, 최종 백업은 외부 볼륨에 저장한다면 세 위치를 같은 값으로 보면 안 됩니다.
각 경로의 여유 공간과 권한, 마운트 상태가 다를 수 있습니다.

```js
import { statfs } from 'node:fs/promises';

export async function getAvailableBytes(path) {
  const stats = await statfs(path);
  return stats.bsize * stats.bavail;
}

const availableBytes = await getAvailableBytes('./dist');
console.log({ availableBytes });
```

이 예제는 `./dist`가 속한 파일 시스템에서 일반 사용자가 실제로 사용할 수 있는 공간을 계산합니다.
운영 코드에서는 출력 값을 그대로 로그에 남기기보다 MiB나 GiB 단위로 변환해 사람이 읽기 쉽게 남기는 편이 좋습니다.

## bavail과 bfree 차이

### H3. 서비스 판단에는 bavail을 우선한다

`statfs()` 결과에는 `bavail`과 `bfree`가 함께 들어 있습니다.
둘 다 free block과 관련 있지만 의미가 다릅니다.
`bfree`는 파일 시스템의 전체 free block 수이고, `bavail`은 권한이 제한된 일반 사용자에게 사용 가능한 free block 수입니다.

서비스 프로세스가 root가 아닌 사용자로 실행된다면, 운영 판단에는 보통 `bavail`이 더 적합합니다.
파일 시스템은 관리자나 시스템 용도로 일부 블록을 예약할 수 있기 때문입니다.
`bfree`만 보고 여유가 있다고 판단했는데 실제 서비스 계정은 더 이상 쓸 수 없는 상황이 생길 수 있습니다.

```js
import { statfs } from 'node:fs/promises';

export async function readDiskSpace(path) {
  const stats = await statfs(path);

  return {
    availableBytes: stats.bsize * stats.bavail,
    freeBytes: stats.bsize * stats.bfree,
    totalBytes: stats.bsize * stats.blocks
  };
}
```

`availableBytes`는 지금 작업을 시작해도 되는지 판단하는 값으로 쓰고, `freeBytes`는 운영 진단이나 관리자 관점의 참고값으로 남기는 식이 현실적입니다.
대시보드나 알림에서도 서비스가 실제로 쓸 수 있는 공간과 전체 free 공간을 구분해 표시하면 원인 파악이 빨라집니다.

### H3. block size를 곱해야 바이트가 된다

`bavail`, `bfree`, `blocks`는 바이트 수가 아니라 block 수입니다.
바이트 단위로 비교하려면 `bsize`를 곱해야 합니다.
임계값을 `500 * 1024 * 1024`처럼 바이트로 잡아 놓고 block 수와 직접 비교하면 판단이 크게 틀어집니다.

아래처럼 단위 변환 함수를 작게 분리해 두면 실수를 줄일 수 있습니다.

```js
const MiB = 1024 * 1024;
const GiB = 1024 * MiB;

export function formatBytes(bytes) {
  if (bytes >= GiB) {
    return `${(bytes / GiB).toFixed(1)} GiB`;
  }

  return `${(bytes / MiB).toFixed(1)} MiB`;
}
```

로그에는 원본 바이트 값과 사람이 읽는 문자열을 함께 남기는 것이 좋습니다.
알림은 사람이 읽는 문자열이 편하고, 자동화된 정책이나 테스트는 바이트 값이 정확합니다.

## 배치 시작 전 점검 함수 만들기

### H3. 필요한 공간과 여유 버퍼를 분리한다

작업에 필요한 예상 용량만 임계값으로 두면 부족할 때가 많습니다.
파일 시스템 메타데이터, 임시 파일, 압축 중간 산출물, 로그 증가량까지 고려하면 예상치보다 조금 더 넉넉해야 합니다.
그래서 `requiredBytes`와 `reserveBytes`를 분리해 두는 편이 좋습니다.

```js
import { statfs } from 'node:fs/promises';

export class InsufficientDiskSpaceError extends Error {
  constructor({ path, availableBytes, minimumBytes }) {
    super('insufficient disk space');
    this.name = 'InsufficientDiskSpaceError';
    this.code = 'INSUFFICIENT_DISK_SPACE';
    this.path = path;
    this.availableBytes = availableBytes;
    this.minimumBytes = minimumBytes;
  }
}

export async function assertDiskSpace(path, {
  requiredBytes,
  reserveBytes = 256 * 1024 * 1024
}) {
  const stats = await statfs(path);
  const availableBytes = stats.bsize * stats.bavail;
  const minimumBytes = requiredBytes + reserveBytes;

  if (availableBytes < minimumBytes) {
    throw new InsufficientDiskSpaceError({
      path,
      availableBytes,
      minimumBytes
    });
  }

  return {
    availableBytes,
    minimumBytes
  };
}
```

`reserveBytes`는 작업 성격에 따라 다르게 잡습니다.
작은 JSON manifest를 쓰는 작업은 128MiB도 충분할 수 있지만, 이미지 변환이나 압축처럼 중간 파일이 큰 작업은 몇 GiB 이상이 필요할 수 있습니다.
정확한 숫자를 모른다면 처음에는 보수적으로 잡고, 운영 로그를 보며 조정하는 방식이 안전합니다.

### H3. 실패는 재시도 가능한 상태로 기록한다

디스크 부족은 보통 코드 버그라기보다 운영 상태 문제입니다.
즉시 같은 작업을 재시도해도 성공하지 않을 가능성이 큽니다.
대신 정리 작업, 용량 확장, 오래된 캐시 삭제, 작업량 축소 같은 조치가 먼저 필요합니다.

```js
import { assertDiskSpace } from './disk-space.js';

export async function runReportJob({ outputDir, logger }) {
  try {
    await assertDiskSpace(outputDir, {
      requiredBytes: 2 * 1024 * 1024 * 1024,
      reserveBytes: 512 * 1024 * 1024
    });
  } catch (error) {
    if (error?.code === 'INSUFFICIENT_DISK_SPACE') {
      logger.warn({
        event: 'report_job_skipped_low_disk',
        availableBytes: error.availableBytes,
        minimumBytes: error.minimumBytes
      });

      return {
        ok: false,
        retryable: true,
        reason: 'low_disk_space'
      };
    }

    throw error;
  }

  return createReportFiles(outputDir);
}
```

로그에는 전체 파일 경로나 사용자 입력을 그대로 넣지 않는 편이 좋습니다.
출력 디렉터리 이름이 내부 구조나 고객 식별자를 포함할 수 있기 때문입니다.
운영자가 필요한 값은 대개 이벤트명, 사용 가능 바이트, 최소 필요 바이트, 작업 종류, 실행 ID 정도입니다.

## bigint 옵션을 써야 하는 경우

### H3. 큰 파일 시스템에서는 bigint가 더 안전하다

`statfs(path, { bigint: true })`를 사용하면 numeric 값이 `bigint`로 반환됩니다.
대부분의 일반적인 서버에서는 `number`로도 충분하지만, 아주 큰 파일 시스템이나 정밀한 용량 계산이 필요한 환경에서는 `bigint`가 더 안전합니다.
JavaScript `number`는 큰 정수를 정확히 표현하는 데 한계가 있기 때문입니다.

```js
import { statfs } from 'node:fs/promises';

const GiB = 1024n * 1024n * 1024n;

export async function getAvailableGiB(path) {
  const stats = await statfs(path, { bigint: true });
  const availableBytes = stats.bsize * stats.bavail;

  return availableBytes / GiB;
}
```

`bigint`를 쓰면 숫자 리터럴도 `1024n`처럼 맞춰야 합니다.
`number`와 `bigint`는 바로 섞어 계산할 수 없으므로, 프로젝트 안에서 한쪽 기준을 정해 두는 편이 좋습니다.

### H3. JSON 로그에는 문자열로 변환한다

`bigint`는 `JSON.stringify()`가 기본으로 직렬화하지 못합니다.
구조화 로그에 그대로 넣으면 로거나 전송 계층에서 실패할 수 있습니다.
그래서 JSON으로 남길 값은 문자열 또는 안전한 범위의 number로 변환해야 합니다.

```js
export function toDiskSpaceLog(stats) {
  const availableBytes = stats.bsize * stats.bavail;
  const totalBytes = stats.bsize * stats.blocks;

  return {
    availableBytes: availableBytes.toString(),
    totalBytes: totalBytes.toString()
  };
}
```

사람이 보는 알림은 `formatBytes()` 같은 함수로 변환하고, 기계가 후속 처리하는 로그는 문자열 바이트 값을 남기면 됩니다.
이렇게 하면 큰 숫자의 정밀도를 잃지 않으면서 JSON 기반 로그 파이프라인도 깨지지 않습니다.

## 테스트와 운영 점검 기준

### H3. statfs 호출부를 주입 가능하게 둔다

디스크 부족 상황을 테스트하기 위해 실제 디스크를 채우는 것은 위험하고 느립니다.
대신 `statfs` 호출 함수를 주입 가능하게 만들면 작은 단위 테스트로 판단 로직을 검증할 수 있습니다.

```js
export function createDiskSpaceChecker({ statfs }) {
  return async function check(path, minimumBytes) {
    const stats = await statfs(path);
    const availableBytes = stats.bsize * stats.bavail;

    return {
      ok: availableBytes >= minimumBytes,
      availableBytes,
      minimumBytes
    };
  };
}
```

테스트에서는 `statfs`를 가짜 함수로 넣고, `bsize`, `bavail`, `blocks` 값을 작게 바꿔가며 검증하면 됩니다.
이 방식은 플랫폼별 파일 시스템 차이에 덜 흔들리고, CI에서도 안정적으로 동작합니다.

### H3. 실제 쓰기 실패 처리는 그대로 둔다

사전 점검이 있더라도 실제 파일 쓰기의 `ENOSPC`, `EDQUOT`, `EACCES` 처리는 유지해야 합니다.
점검과 쓰기 사이에 상태가 바뀔 수 있고, 디스크 공간이 충분해도 quota나 권한 문제로 실패할 수 있습니다.
따라서 `statfs()`는 쓰기 실패 처리를 대체하지 않습니다.

안전한 흐름은 아래 순서입니다.

1. 작업 대상 경로가 맞는지 확인한다.
2. `statfs()`로 최소 여유 공간을 확인한다.
3. 임시 파일이나 작업 디렉터리를 만든다.
4. 실제 쓰기 오류를 잡아 재시도 가능 여부를 분류한다.
5. 실패 시 임시 파일을 정리하고 상태를 남긴다.

크론이나 배치에서는 이 순서를 적용하면 실패 리포트가 훨씬 명확해집니다.
"작업 실패" 하나로 뭉뚱그리는 대신, 낮은 디스크 공간인지, 권한 문제인지, 쓰기 중간 실패인지 구분할 수 있습니다.

## FAQ

### H3. statfs를 매 요청마다 호출해도 되나요?

일반 웹 요청마다 호출하는 것은 권장하지 않습니다.
파일 시스템 조회도 비용이 있고, 요청 지연 시간에 영향을 줄 수 있습니다.
배치 시작 전, 업로드 세션 시작 전, 일정 주기의 health check처럼 빈도를 제한하는 편이 좋습니다.

### H3. bavail이 충분하면 writeFile은 반드시 성공하나요?

아닙니다.
다른 프로세스가 공간을 사용하거나, quota와 권한 문제가 있거나, 대상 디렉터리가 사라질 수 있습니다.
`statfs()`는 사전 방어이고, 실제 쓰기 오류 처리는 별도로 남겨야 합니다.

### H3. 임계값은 어떻게 정하나요?

작업이 만드는 최종 파일 크기, 임시 파일 크기, 로그 증가량, 실패 후 정리 여유를 합쳐 잡습니다.
처음에는 보수적으로 잡고, 성공/실패 로그의 실제 사용량을 보며 낮추거나 높이는 방식이 좋습니다.

## 마무리

`fs.promises.statfs()`는 파일을 쓰는 Node.js 자동화에 작은 안전장치를 더해 줍니다.
특히 배치, 크론, 정적 사이트 빌드, 리포트 생성처럼 중간 산출물이 있는 작업에서는 디스크 부족을 시작 전에 발견하는 것만으로도 복구 비용이 줄어듭니다.

실무 기준은 단순합니다.
실제로 쓸 경로를 기준으로 `bavail * bsize`를 계산하고, 필요한 공간과 예비 공간을 나눠 임계값을 정하고, 실제 쓰기 오류 처리는 그대로 유지하세요.
중요한 결과 파일은 원자적 교체와 flush를 함께 적용하면 더 안정적인 파일 쓰기 흐름을 만들 수 있습니다.

## 관련 글

- [Node.js writeFile flush 가이드](/development/blog/seo/2026/07/29/nodejs-fspromises-writefile-flush-durable-write-guide.html)
- [Node.js lock file 가이드](/development/blog/seo/2026/07/28/nodejs-lock-file-atomic-job-deduplication-guide.html)
- [Node.js mkdtemp 가이드](/development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html)

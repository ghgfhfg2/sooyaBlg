---
layout: post
title: "Node.js writeFile flush 가이드: 중요한 파일 쓰기를 디스크까지 밀어 넣는 법"
date: 2026-07-29 08:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-writefile-flush-durable-write-guide
permalink: /development/blog/seo/2026/07/29/nodejs-fspromises-writefile-flush-durable-write-guide.html
alternates:
  ko: /development/blog/seo/2026/07/29/nodejs-fspromises-writefile-flush-durable-write-guide.html
  x_default: /development/blog/seo/2026/07/29/nodejs-fspromises-writefile-flush-durable-write-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, writefile, flush, filesystem, durability, automation, reliability, javascript]
description: "Node.js fs.promises.writeFile의 flush 옵션으로 중요한 설정 파일, 매니페스트, 배치 결과를 디스크까지 반영하는 방법을 정리합니다. 원자적 파일 교체, fsync 비용, 임시 파일 패턴, CI 점검 기준까지 실무 예제로 설명합니다."
---

파일 쓰기는 겉으로는 단순해 보입니다.
`await writeFile('result.json', data)`가 끝났다면 파일이 저장됐다고 생각하기 쉽습니다.
하지만 운영체제와 파일 시스템은 성능을 위해 쓰기 내용을 잠시 버퍼에 둘 수 있고, 프로세스 관점의 쓰기 완료와 저장 장치까지의 반영은 같은 말이 아닙니다.

Node.js의 `fs.promises.writeFile()`에는 `flush` 옵션이 있습니다.
Node.js 공식 문서 기준으로 이 옵션을 `true`로 설정하면 파일을 닫기 전에 내부 파일 디스크립터를 flush합니다.
이 글에서는 `writeFile({ flush: true })`가 필요한 상황, 원자적 파일 교체와 함께 쓰는 패턴, 비용과 한계를 정리합니다.
파일 경로 충돌 자체를 막아야 한다면 [Node.js lock file 가이드](/development/blog/seo/2026/07/28/nodejs-lock-file-atomic-job-deduplication-guide.html)를 먼저 적용하고, 임시 작업 디렉터리 격리는 [Node.js mkdtemp 가이드](/development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html)를 함께 보면 좋습니다.

## writeFile flush가 필요한 이유

### H3. 쓰기 완료와 내구성은 다르다

대부분의 애플리케이션 파일 쓰기는 운영체제 캐시에 먼저 들어갑니다.
이 방식은 빠르고 일반적인 웹 요청, 캐시 파일, 일시적 산출물에는 충분합니다.
문제는 쓰기 직후 프로세스가 죽거나, 컨테이너가 중단되거나, 머신이 재부팅되는 상황입니다.
애플리케이션은 쓰기가 끝났다고 생각했지만 실제 저장 장치에는 아직 반영되지 않았을 수 있습니다.

모든 파일에 강한 내구성이 필요한 것은 아닙니다.
하지만 아래 파일은 조금 더 보수적으로 다루는 편이 좋습니다.

- 배치 작업의 마지막 완료 지점 파일
- 배포 산출물의 manifest 또는 checksum 파일
- 큐 consumer의 처리 offset 스냅샷
- 재시작 후 복구에 쓰는 작은 상태 파일
- 설정 변경을 반영한 런타임 config 파일
- 크론 작업의 마지막 성공 시각 기록

이런 파일은 "대충 언젠가 저장"보다 "쓰기 완료를 반환하기 전에 가능한 한 디스크까지 밀어 넣기"가 더 중요할 수 있습니다.

### H3. flush는 데이터 손실 가능성을 줄이는 옵션이다

`flush: true`는 `writeFile()`이 파일 내용을 쓴 뒤 닫기 전에 flush를 시도하게 만듭니다.
실무적으로는 작은 중요 파일에 적합합니다.
예를 들어 배치 결과를 `state.json`에 저장하고 다음 실행이 이 파일을 기준으로 이어서 처리한다면, flush 옵션은 재시작 직후의 불확실성을 줄이는 데 도움이 됩니다.

다만 flush는 마법 같은 백업 장치가 아닙니다.
저장 장치, 파일 시스템, 가상화 계층, 네트워크 파일 시스템의 동작에 따라 보장 수준은 달라질 수 있습니다.
그래서 중요한 파일 쓰기는 flush 하나만 붙이는 것이 아니라 임시 파일, 원자적 rename, 검증 가능한 JSON 구조, 재시도 정책을 함께 설계해야 합니다.

## 기본 사용법

### H3. fs.promises.writeFile에 flush 옵션을 둔다

가장 단순한 형태는 `writeFile()` 옵션에 `flush: true`를 추가하는 것입니다.
아래 예제는 작업 상태를 JSON 파일로 저장합니다.

```js
import { writeFile } from 'node:fs/promises';

export async function saveJobState(path, state) {
  const body = JSON.stringify({
    ...state,
    savedAt: new Date().toISOString()
  }, null, 2);

  await writeFile(path, `${body}\n`, {
    encoding: 'utf8',
    flush: true
  });
}
```

이 패턴은 작고 독립적인 파일에 잘 맞습니다.
파일 크기가 작고 쓰기 빈도가 낮다면 flush 비용도 관리하기 쉽습니다.
반대로 매 요청마다 큰 로그 파일을 flush하면 지연 시간과 저장 장치 부하가 커질 수 있습니다.

### H3. 지원 Node.js 버전을 먼저 확인한다

`flush` 옵션은 비교적 최신 Node.js에서 사용할 수 있습니다.
Node.js 공식 문서의 변경 이력에 따르면 `fsPromises.writeFile()`의 `flush` 옵션은 v21.0.0과 v20.10.0에서 추가됐습니다.
프로젝트가 오래된 Node.js LTS나 여러 런타임 버전을 지원한다면 바로 옵션을 넣기 전에 CI와 운영 이미지를 확인해야 합니다.

현실적인 기준은 아래처럼 나눌 수 있습니다.

- 사내 서비스: 런타임을 v20.10.0 이상으로 고정한 뒤 사용한다.
- 공개 패키지: 지원 버전 범위가 넓다면 수동 `FileHandle.sync()` fallback을 검토한다.
- CLI 도구: 사용자의 설치 Node.js 버전을 확인하고 안내 메시지를 둔다.
- 테스트 코드: CI Node.js 버전을 먼저 올린 뒤 작은 파일부터 적용한다.

버전이 섞인 환경에서는 "로컬에서는 저장 안정성을 높였는데 운영에서 옵션을 모르는 런타임이 실행된다"는 식의 실패가 생길 수 있습니다.
파일 내구성 개선은 런타임 표준화와 같이 진행하는 편이 안전합니다.

## 원자적 파일 교체와 함께 쓰기

### H3. 직접 덮어쓰기는 중간 상태를 노출할 수 있다

`writeFile(target, data)`는 대상 파일을 바로 덮어씁니다.
쓰기 중간에 프로세스가 죽으면 대상 파일이 비어 있거나 일부만 기록된 상태로 남을 수 있습니다.
상태 파일이나 매니페스트처럼 다음 실행이 바로 읽는 파일이라면 이 중간 상태가 복구 실패로 이어질 수 있습니다.

더 안전한 흐름은 같은 디렉터리에 임시 파일을 쓴 뒤, flush가 끝나면 `rename()`으로 교체하는 것입니다.
많은 파일 시스템에서 같은 디렉터리 안의 rename은 원자적 교체에 가깝게 동작하므로 독자는 기존 파일 또는 새 파일 중 하나를 보게 됩니다.

```js
import { rename, writeFile } from 'node:fs/promises';
import { dirname, join } from 'node:path';
import { randomUUID } from 'node:crypto';

export async function writeJsonAtomically(targetPath, value) {
  const dir = dirname(targetPath);
  const tempPath = join(dir, `.${randomUUID()}.tmp`);
  const body = `${JSON.stringify(value, null, 2)}\n`;

  await writeFile(tempPath, body, {
    encoding: 'utf8',
    flush: true,
    mode: 0o600
  });

  await rename(tempPath, targetPath);
}
```

이 코드는 임시 파일 내용이 디스크에 반영되도록 시도한 뒤 대상 경로로 옮깁니다.
`mode: 0o600`은 새 파일을 만들 때 소유자만 읽고 쓸 수 있게 하려는 의도입니다.
상태 파일에 민감정보가 없어야 한다는 원칙은 그대로지만, 권한도 보수적으로 잡는 편이 좋습니다.

### H3. 임시 파일은 같은 디렉터리에 만든다

임시 파일을 시스템 임시 디렉터리에 만들고 대상 디렉터리로 옮기면 다른 파일 시스템 경계를 넘을 수 있습니다.
이 경우 rename이 실패하거나 복사와 삭제에 가까운 동작이 필요해져 원자성이 약해질 수 있습니다.
그래서 원자적 교체 목적의 임시 파일은 대상 파일과 같은 디렉터리에 두는 편이 좋습니다.

임시 파일명에는 사용자 입력을 넣지 않습니다.
이메일, 토큰, 주문 번호, 내부 ID가 파일명에 들어가면 장애 로그와 파일 목록에 그대로 남을 수 있습니다.
위 예제처럼 무작위 값과 짧은 suffix만 사용하는 방식이 안전합니다.

작업 전체의 중간 산출물이 많다면 [Node.js mkdtemp 가이드](/development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html)처럼 작업용 디렉터리를 따로 만들고, 최종 교체 파일만 대상 디렉터리에서 처리하는 구조가 좋습니다.

## 실패 처리와 정리

### H3. rename 실패 시 임시 파일을 정리한다

임시 파일 쓰기까지 성공했더라도 `rename()`에서 실패할 수 있습니다.
권한이 없거나, 대상 디렉터리가 사라졌거나, 파일 시스템 경계가 다르거나, Windows에서 다른 프로세스가 대상 파일을 붙잡고 있을 수 있습니다.
이때 임시 파일을 그대로 방치하면 다음 실행 때 혼란을 만들 수 있습니다.

아래처럼 정리 함수를 분리하면 원래 실패와 정리 실패를 구분할 수 있습니다.

```js
import { rm } from 'node:fs/promises';

async function removeTempFile(path, logger = console) {
  try {
    await rm(path, { force: true });
  } catch (error) {
    logger.warn({
      event: 'atomic_write_temp_cleanup_failed',
      code: error?.code
    });
  }
}
```

정리 실패 로그에도 전체 경로나 파일 내용을 그대로 남기지 않는 편이 좋습니다.
경로 자체가 내부 디렉터리 구조나 작업 이름을 노출할 수 있기 때문입니다.
필요하다면 파일명 suffix나 작업명 정도로 줄여 기록합니다.

### H3. 쓰기 함수는 검증 가능한 값을 받는다

내구성을 높여도 잘못된 데이터를 튼튼하게 저장하면 문제는 그대로 남습니다.
상태 파일을 쓰기 전에는 최소한 필요한 필드가 있는지, 다음 실행이 읽을 수 있는 JSON인지, 버전 필드가 맞는지 확인해야 합니다.

```js
function createStateSnapshot(input) {
  if (!input || typeof input !== 'object') {
    throw new TypeError('state input must be an object');
  }

  if (typeof input.cursor !== 'string') {
    throw new TypeError('state cursor must be a string');
  }

  return {
    schemaVersion: 1,
    cursor: input.cursor,
    completedCount: Number(input.completedCount ?? 0),
    savedAt: new Date().toISOString()
  };
}
```

파일 쓰기 안정성은 데이터 검증과 짝입니다.
특히 복구 파일은 장애 상황에서 읽히므로 평소보다 더 단순하고 엄격한 구조가 좋습니다.

## flush 비용과 적용 범위

### H3. 모든 writeFile에 붙이지 않는다

flush는 저장 안정성을 높이는 대신 비용이 있습니다.
디스크 flush는 일반 메모리 쓰기보다 느리고, 저장 장치와 파일 시스템 상태에 따라 지연 시간이 튈 수 있습니다.
그래서 모든 파일 쓰기에 습관처럼 붙이는 방식은 좋지 않습니다.

적용 우선순위는 아래처럼 잡을 수 있습니다.

- 높음: 복구 상태, 배포 manifest, 작업 완료 marker, 작은 설정 파일
- 중간: 자주 바뀌지 않는 캐시 index, 수동 운영 스냅샷
- 낮음: 재생성 가능한 캐시 본문, 임시 변환 결과, 대량 로그

대량 로그는 파일마다 flush하기보다 로그 수집기, journald, stdout 기반 수집, 배치 flush 정책을 사용하는 편이 더 현실적입니다.
자동화 결과 파일도 큰 데이터 본문보다 작은 manifest와 checksum을 튼튼하게 저장하는 쪽이 비용 대비 효과가 좋습니다.

### H3. 반복 쓰기는 빈도를 낮춘다

작업 중간마다 상태 파일을 저장해야 한다면 매 항목마다 flush하지 말고 checkpoint 간격을 둡니다.
예를 들어 1만 개 항목을 처리하면서 1개마다 flush하면 처리량이 크게 떨어질 수 있습니다.
대신 100개 또는 1분마다 저장하고, 마지막 완료 시점에 한 번 더 flush하는 식으로 균형을 잡습니다.

```js
export function shouldCheckpoint({ processed, lastCheckpointAt }) {
  if (processed % 100 === 0) {
    return true;
  }

  return Date.now() - lastCheckpointAt > 60_000;
}
```

checkpoint 간격은 데이터 손실 허용 범위와 처리 비용을 함께 보고 정합니다.
재처리가 안전한 작업은 간격을 넓힐 수 있고, 외부 부작용이 큰 작업은 더 촘촘하게 잡아야 합니다.
외부 API 재시도와 실패 분류는 [Node.js fetch timeout/retry 가이드](/development/blog/seo/2026/07/24/nodejs-fetch-timeout-retry-error-classification-guide.html)를 함께 참고할 수 있습니다.

## 운영 점검 기준

### H3. 완료 marker는 lock file과 역할을 나눈다

락 파일은 "지금 실행 중"을 나타내고, 완료 marker는 "마지막으로 어디까지 성공했는지"를 나타냅니다.
두 역할을 한 파일에 섞으면 stale lock 처리와 복구 판단이 복잡해집니다.

추천 구조는 아래처럼 나누는 것입니다.

```text
var/
  run/
    daily-report.lock
  state/
    daily-report-state.json
```

락 파일은 작업 시작과 종료에 맞춰 만들고 지웁니다.
상태 파일은 처리 지점이 바뀔 때 원자적으로 교체합니다.
이렇게 분리하면 중복 실행 방지와 재시작 복구를 각각 테스트할 수 있습니다.

### H3. CI에서 손상 파일을 읽는 테스트를 둔다

원자적 쓰기 함수는 성공 케이스만 테스트하면 부족합니다.
최소한 아래 항목을 확인하는 테스트가 있으면 회귀를 줄일 수 있습니다.

- JSON 문자열 끝에 개행을 붙이는가?
- 필요한 schemaVersion이 들어가는가?
- 임시 파일이 대상 파일과 같은 디렉터리에 만들어지는가?
- 쓰기 실패나 rename 실패 후 임시 파일을 정리하는가?
- 기존 파일이 있을 때 새 파일로 교체되는가?
- 민감정보가 파일명과 로그에 들어가지 않는가?

파일 시스템 테스트는 병렬 실행에서 충돌하기 쉽습니다.
테스트별 고유 임시 디렉터리를 만들고 정리하는 패턴은 [Node.js mkdtemp 가이드](/development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html)를 그대로 적용하면 됩니다.

## 발행 전 체크리스트

### H3. 중요한 파일만 durable write로 다룬다

`writeFile({ flush: true })`는 작은 옵션이지만, 중요한 상태 파일을 더 조심스럽게 저장하는 출발점이 됩니다.
다만 모든 쓰기를 강하게 만드는 대신, 복구와 배포에 직접 영향을 주는 파일부터 선별해서 적용해야 합니다.

적용 전에는 아래 항목을 확인합니다.

- 대상 파일이 재시작 후 복구나 배포 판단에 쓰이는가?
- Node.js 런타임이 `flush` 옵션을 지원하는가?
- 임시 파일을 대상과 같은 디렉터리에 쓰는가?
- 임시 파일 쓰기 후 `rename()`으로 교체하는가?
- 파일명과 로그에 토큰, 이메일, 내부 식별자가 들어가지 않는가?
- checkpoint 저장 빈도가 처리량을 지나치게 떨어뜨리지 않는가?
- lock file, 임시 디렉터리, 완료 marker의 역할이 분리돼 있는가?
- 실패 후 재시도해도 같은 상태 파일을 안전하게 다시 만들 수 있는가?

Node.js 파일 쓰기에서 내구성은 "한 줄 옵션"만의 문제가 아닙니다.
`flush: true`, 같은 디렉터리 임시 파일, 원자적 rename, 보수적인 로그, 복구 테스트를 함께 묶을 때 운영에서 믿을 수 있는 상태 저장 흐름이 됩니다.

## 참고 자료

- [Node.js 공식 문서: File system API](https://nodejs.org/api/fs.html)
- [Node.js lock file 가이드](/development/blog/seo/2026/07/28/nodejs-lock-file-atomic-job-deduplication-guide.html)
- [Node.js mkdtemp 가이드](/development/blog/seo/2026/07/28/nodejs-mkdtemp-temporary-directory-cleanup-guide.html)

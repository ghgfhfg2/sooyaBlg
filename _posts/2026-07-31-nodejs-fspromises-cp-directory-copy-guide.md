---
layout: post
title: "Node.js fs.promises.cp 가이드: 디렉터리 복사와 배포 산출물 동기화를 안전하게 처리하는 법"
date: 2026-07-31 20:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-cp-directory-copy-guide
permalink: /development/blog/seo/2026/07/31/nodejs-fspromises-cp-directory-copy-guide.html
alternates:
  ko: /development/blog/seo/2026/07/31/nodejs-fspromises-cp-directory-copy-guide.html
  x_default: /development/blog/seo/2026/07/31/nodejs-fspromises-cp-directory-copy-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs-promises, cp, copy, filesystem, deployment, static-assets, javascript]
description: "Node.js fs.promises.cp로 디렉터리와 정적 산출물을 안전하게 복사하는 방법을 정리합니다. recursive, filter, force, errorOnExist, preserveTimestamps, 심볼릭 링크 옵션과 배포 자동화 체크리스트를 실무 예제로 설명합니다."
---

정적 사이트, 문서 빌드, 이미지 변환, CLI 템플릿 생성처럼 파일 묶음을 다루는 자동화에서는 "한 파일 복사"보다 "디렉터리 전체 복사"가 더 자주 필요합니다.
예전에는 직접 `readdir()`로 순회하거나 외부 패키지를 붙여 처리하는 경우가 많았지만, Node.js에서는 `fs.promises.cp()`로 디렉터리 구조 전체를 비동기 방식으로 복사할 수 있습니다.

공식 문서 기준 `fs.promises.cp()`는 Node.js 22.3.0부터 더 이상 experimental API가 아니며, `recursive`, `filter`, `force`, `errorOnExist`, `preserveTimestamps`, `dereference`, `verbatimSymlinks`, `mode` 같은 옵션을 제공합니다.
편리한 API지만 배포 산출물이나 사용자 파일을 다룰 때는 덮어쓰기 정책, 제외 규칙, 심볼릭 링크 처리 기준을 분명히 해야 합니다.

이 글에서는 `fs.promises.cp()`를 배포 산출물 복사, 템플릿 스캐폴딩, 정적 자산 동기화에 안전하게 쓰는 기준을 정리합니다.
복사 전에 기존 산출물을 지우는 흐름은 [Node.js fs.promises.rm 가이드](/development/blog/seo/2026/07/31/nodejs-fspromises-rm-safe-cleanup-guide.html)와 함께 보면 좋고, 복사 전 파일 존재와 권한 점검이 필요하다면 [Node.js fs.promises.access 가이드](/development/blog/seo/2026/07/30/nodejs-fspromises-access-file-permission-check-guide.html)를 참고하세요.
큰 디렉터리 순회가 필요하다면 [Node.js opendir 가이드](/development/blog/seo/2026/07/26/nodejs-fspromises-opendir-directory-walk-guide.html)도 같이 연결해 볼 만합니다.

## fs.promises.cp가 어울리는 작업

### 디렉터리 구조를 그대로 옮긴다

`fs.promises.cp(src, dest, options)`는 원본 경로의 파일과 하위 디렉터리를 대상 경로로 복사합니다.
디렉터리를 복사하려면 `recursive: true`를 명시해야 합니다.

```js
import { cp } from 'node:fs/promises';

export async function copyStaticAssets() {
  await cp('public', 'dist/public', {
    recursive: true
  });
}
```

이 코드는 `public/` 아래의 정적 파일을 `dist/public/`로 복사합니다.
작은 프로젝트에서는 이 정도만으로 충분하지만, 실무 배포에서는 대상 경로가 이미 있을 때 어떻게 할지, 어떤 파일을 제외할지, 실패 로그를 어디까지 남길지까지 정해야 합니다.

복사는 삭제보다 안전해 보이지만, 덮어쓰기까지 포함하면 결과를 되돌리기 어렵습니다.
특히 `dist/`, `build/`, `out/`처럼 배포와 연결된 디렉터리는 복사 전후의 상태가 명확해야 합니다.

### 템플릿 스캐폴딩에 사용할 수 있다

CLI가 새 프로젝트를 만들 때 템플릿 디렉터리를 통째로 복사하는 구조도 흔합니다.
이 경우에는 사용자가 이미 만든 파일을 덮어쓰지 않는 것이 중요합니다.

```js
import { cp } from 'node:fs/promises';

export async function scaffoldProject(templateDir, targetDir) {
  await cp(templateDir, targetDir, {
    recursive: true,
    force: false,
    errorOnExist: true
  });
}
```

`force: false`는 대상이 있을 때 덮어쓰지 않겠다는 뜻입니다.
여기에 `errorOnExist: true`를 함께 두면 이미 존재하는 파일을 만났을 때 조용히 넘어가지 않고 실패로 드러낼 수 있습니다.
초기 생성 도구에서는 이 편이 안전합니다.

반대로 캐시나 빌드 산출물처럼 언제든 다시 만들 수 있는 대상이라면 덮어쓰기를 허용할 수 있습니다.
중요한 기준은 파일의 성격입니다.
사용자가 직접 만든 파일은 보수적으로 다루고, 자동화가 만든 산출물은 재생성 가능성을 기준으로 정책을 정합니다.

## recursive와 filter 설계

### recursive는 복사 범위를 넓힌다

`recursive: true`는 하위 디렉터리 전체를 복사합니다.
이 옵션을 넣는 순간 복사 대상은 파일 하나가 아니라 트리 전체가 됩니다.
따라서 원본과 대상 경로를 모두 `resolve()`로 정규화하고, 의도한 프로젝트 내부인지 확인하는 가드가 있으면 좋습니다.

```js
import { cp } from 'node:fs/promises';
import { relative, resolve } from 'node:path';

function assertInside(baseDir, targetPath) {
  const base = resolve(baseDir);
  const target = resolve(targetPath);
  const rel = relative(base, target);

  if (rel === '' || rel.startsWith('..') || rel.startsWith('/')) {
    throw new Error('path is outside the allowed directory');
  }
}

export async function copyBuildAssets(projectRoot) {
  const source = resolve(projectRoot, 'public');
  const target = resolve(projectRoot, 'dist/public');

  assertInside(projectRoot, source);
  assertInside(projectRoot, target);

  await cp(source, target, {
    recursive: true
  });
}
```

삭제 작업만큼은 아니어도 복사 작업 역시 경로 계산 실수에 취약합니다.
대상이 잘못 계산되면 원치 않는 위치에 파일을 만들거나, 기존 파일을 덮어쓸 수 있습니다.
CI나 크론에서 환경변수로 경로를 받는다면 비어 있는 값, `.` 값, 예상 밖의 절대 경로를 먼저 거르는 편이 좋습니다.

### filter는 제외 규칙을 코드로 고정한다

배포 산출물을 만들 때 모든 파일을 복사하면 안 되는 경우가 많습니다.
소스맵, 테스트 fixture, 임시 파일, `.env`, 로컬 캐시, 운영에 필요 없는 원본 이미지가 대상이 될 수 있습니다.
`filter` 옵션은 복사할 파일과 디렉터리를 직접 결정할 수 있게 해 줍니다.

```js
import { cp } from 'node:fs/promises';
import { basename } from 'node:path';

const BLOCKED_NAMES = new Set([
  '.env',
  '.env.local',
  '.DS_Store'
]);

export async function copyPublicFiles(source, target) {
  await cp(source, target, {
    recursive: true,
    filter(src) {
      const name = basename(src);

      if (BLOCKED_NAMES.has(name)) {
        return false;
      }

      return !name.endsWith('.tmp');
    }
  });
}
```

`filter`가 디렉터리에 대해 `false`를 반환하면 그 안의 파일도 함께 건너뜁니다.
이 특성은 `node_modules/`, `.git/`, `.cache/` 같은 큰 디렉터리를 제외할 때 유용합니다.
성능에도 영향을 주기 때문에, 제외할 수 있는 상위 디렉터리는 최대한 빨리 걸러내는 편이 좋습니다.

비밀 값이 들어갈 수 있는 파일은 "나중에 배포 단계에서 빼자"보다 복사 단계에서 제외하는 것이 안전합니다.
배포용 디렉터리에 한 번 들어간 파일은 압축, 업로드, 캐시, 로그 과정에 남을 수 있습니다.

## 덮어쓰기 정책 정하기

### force 기본값을 의식한다

`fs.promises.cp()`의 `force` 기본값은 `true`입니다.
즉 별도 옵션을 주지 않으면 대상 파일을 덮어쓸 수 있습니다.
자동 배포 산출물처럼 덮어쓰기가 자연스러운 작업도 있지만, 템플릿 생성이나 사용자 작업 디렉터리 복사에서는 위험할 수 있습니다.

```js
await cp(templateDir, targetDir, {
  recursive: true,
  force: false,
  errorOnExist: true
});
```

이 조합은 "이미 있으면 실패"라는 의도를 코드에 분명히 남깁니다.
반면 `force: false`만 쓰면 대상이 있을 때 오류를 무시하고 넘어갈 수 있으므로, 사용자가 실패를 알아야 하는 흐름에서는 `errorOnExist: true`를 함께 고려해야 합니다.

배포 자동화에서는 다른 접근도 가능합니다.
대상 디렉터리에 바로 덮어쓰는 대신 새 임시 디렉터리에 복사하고 검증이 끝난 뒤 교체합니다.

```js
import { cp, rename, rm } from 'node:fs/promises';

export async function publishSnapshot(source, target) {
  const nextTarget = `${target}.next`;
  const previousTarget = `${target}.previous`;

  await rm(nextTarget, { recursive: true, force: true });

  await cp(source, nextTarget, {
    recursive: true,
    force: false,
    errorOnExist: true
  });

  await rm(previousTarget, { recursive: true, force: true });
  await rename(target, previousTarget).catch((error) => {
    if (error?.code !== 'ENOENT') throw error;
  });
  await rename(nextTarget, target);
}
```

이 예제는 단순화된 형태입니다.
실제 운영에서는 같은 파일 시스템 안에서 `rename()`하는지, 롤백 디렉터리를 얼마나 보관할지, 교체 중 읽기 요청이 들어올 수 있는지까지 확인해야 합니다.
그래도 핵심은 같습니다.
복사와 공개 반영을 한 단계로 섞지 않으면 실패 복구가 쉬워집니다.

### mode는 copyFile 동작을 조정할 때 쓴다

`mode` 옵션은 내부적으로 파일을 복사할 때 `fs.copyFile()`의 mode 인자처럼 동작을 조정합니다.
대표적으로 이미 있는 대상 파일을 실패로 처리하고 싶을 때 `COPYFILE_EXCL` 같은 플래그를 고려할 수 있습니다.

```js
import { constants } from 'node:fs';
import { cp } from 'node:fs/promises';

await cp('assets', 'dist/assets', {
  recursive: true,
  mode: constants.COPYFILE_EXCL
});
```

다만 디렉터리 단위 정책을 읽기 쉽게 만들려면 `force`와 `errorOnExist`를 먼저 명시하는 편이 보통 더 직관적입니다.
`mode`는 파일 복사 플래그까지 엄격하게 맞춰야 하는 특수한 경우에 보조 수단으로 두면 됩니다.

## 타임스탬프와 심볼릭 링크

### preserveTimestamps는 캐시 전략과 관련된다

`preserveTimestamps: true`를 설정하면 원본 파일의 타임스탬프를 대상 파일에 보존합니다.
문서 백업, 스냅샷 보관, 변경 시각이 의미 있는 파일 묶음에서는 유용할 수 있습니다.

하지만 배포 산출물에서는 조심해야 합니다.
일부 빌드나 배포 도구는 수정 시각을 기준으로 캐시 무효화나 증분 처리를 판단합니다.
타임스탬프를 보존하면 "새로 복사했지만 오래된 파일처럼 보이는" 상황이 생길 수 있습니다.

```js
await cp('docs', 'backup/docs', {
  recursive: true,
  preserveTimestamps: true
});
```

백업 목적이면 좋습니다.
반대로 웹 배포용 `dist/`를 만드는 목적이라면 기본값을 유지하고, 해시 파일명이나 manifest로 캐시를 제어하는 편이 더 명확합니다.

### 심볼릭 링크는 의도를 먼저 정한다

`dereference`는 심볼릭 링크를 따라가 실제 대상 파일을 복사할지에 관여합니다.
기본값은 `false`입니다.
`verbatimSymlinks`는 심볼릭 링크 경로 해석을 건너뛸지 정하는 옵션입니다.

심볼릭 링크가 포함된 디렉터리를 복사할 때는 보안과 재현성을 함께 봐야 합니다.
프로젝트 내부 링크라고 생각했지만 실제로는 바깥 경로를 가리킬 수 있고, 로컬에서는 동작하지만 CI에서는 깨질 수도 있습니다.

안전한 기본 전략은 배포 산출물에서 심볼릭 링크를 허용하지 않거나, 허용하더라도 사전에 검사하는 것입니다.

```js
import { lstat } from 'node:fs/promises';

export async function rejectSymlink(src) {
  const stat = await lstat(src);

  if (stat.isSymbolicLink()) {
    return false;
  }

  return true;
}
```

`filter` 안에서 이런 검사를 비동기로 호출할 수 있습니다.
심볼릭 링크가 꼭 필요한 monorepo나 패키지 작업이라면 링크를 보존할지, 링크가 가리키는 실제 파일을 복사할지 정책을 문서화해 두는 편이 좋습니다.

## 운영 코드에서 남길 로그

### 전체 경로 대신 작업 단위를 남긴다

파일 경로에는 사용자 이름, 프로젝트 코드명, 고객 식별자, 내부 서버 구조가 들어갈 수 있습니다.
따라서 실패 로그에 원본 경로와 대상 경로를 그대로 남기는 습관은 피하는 편이 좋습니다.
진단에 필요한 정보는 작업 이름, 오류 코드, 복사한 파일 종류, 실행 ID 정도로도 충분한 경우가 많습니다.

```js
import { cp } from 'node:fs/promises';

export async function copyWithLog({ source, target, logger }) {
  try {
    await cp(source, target, {
      recursive: true,
      force: false,
      errorOnExist: true
    });

    logger.info({
      event: 'copy_completed',
      job: 'static_assets'
    });
  } catch (error) {
    logger.error({
      event: 'copy_failed',
      job: 'static_assets',
      code: error?.code
    });

    throw error;
  }
}
```

운영자가 실제 경로를 알아야 하는 경우에는 접근 권한이 제한된 내부 로그나 디버그 모드에서만 남기는 식으로 분리합니다.
블로그 예제나 공개 저장소 코드에는 실서비스 경로, 계정명, 토큰이 드러나지 않도록 주의해야 합니다.

### 검증 단계를 복사 뒤에 붙인다

복사가 성공했다는 것은 파일 시스템 작업이 끝났다는 뜻이지, 배포 가능한 산출물이 완성됐다는 뜻은 아닙니다.
정적 사이트라면 필수 파일이 있는지, manifest가 유효한지, HTML이 생성됐는지, 금지 파일이 섞이지 않았는지 확인해야 합니다.

```js
import { access, constants } from 'node:fs/promises';
import { join } from 'node:path';

export async function assertStaticBuild(target) {
  await access(join(target, 'index.html'), constants.R_OK);
  await access(join(target, 'assets'), constants.R_OK);
}
```

이런 검증은 `cp()` 호출 뒤 곧바로 붙이는 것이 좋습니다.
검증을 배포 직전에만 두면 어느 단계에서 깨졌는지 찾기 어렵습니다.
복사, 검증, 압축, 업로드, 배포를 각각 분리하면 실패 지점도 선명해집니다.

## 테스트 기준

### 작은 fixture로 복사 정책을 확인한다

`fs.promises.cp()`를 쓰는 함수는 실제 파일 시스템을 대상으로 통합 테스트를 작성하기 쉽습니다.
임시 디렉터리를 만들고, 원본 fixture를 구성하고, 복사 뒤 결과를 확인하면 됩니다.

```js
import { mkdtemp, readFile, writeFile, mkdir } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';
import { test } from 'node:test';
import assert from 'node:assert/strict';

test('copies public files but skips env files', async () => {
  const root = await mkdtemp(join(tmpdir(), 'copy-test-'));
  const source = join(root, 'public');
  const target = join(root, 'dist');

  await mkdir(source);
  await writeFile(join(source, 'index.html'), '<h1>ok</h1>\n');
  await writeFile(join(source, '.env'), 'PLACEHOLDER=value\n');

  await copyPublicFiles(source, target);

  assert.equal(
    await readFile(join(target, 'index.html'), 'utf8'),
    '<h1>ok</h1>\n'
  );

  await assert.rejects(
    readFile(join(target, '.env'), 'utf8'),
    { code: 'ENOENT' }
  );
});
```

테스트에서는 실제 비밀 값을 넣지 않습니다.
예제 값도 `PLACEHOLDER=value`처럼 의미 없는 이름을 쓰는 편이 더 낫습니다.
중요한 것은 제외 규칙이 동작하는지 확인하는 것입니다.

### 실패 케이스도 테스트한다

복사 코드는 성공 케이스보다 실패 케이스에서 운영 품질이 갈립니다.
대상 파일이 이미 있을 때 실패해야 하는지, 필수 파일이 누락됐을 때 어떤 오류를 던질지, 심볼릭 링크를 거부할지 같은 정책을 테스트로 고정해 둡니다.

```js
import { mkdir, writeFile } from 'node:fs/promises';
import { join } from 'node:path';
import { test } from 'node:test';
import assert from 'node:assert/strict';

test('does not overwrite existing scaffold files', async () => {
  const root = await createTempWorkspace();
  const templateDir = join(root, 'template');
  const targetDir = join(root, 'app');

  await mkdir(templateDir);
  await mkdir(targetDir);
  await writeFile(join(templateDir, 'README.md'), 'template\n');
  await writeFile(join(targetDir, 'README.md'), 'user\n');

  await assert.rejects(
    scaffoldProject(templateDir, targetDir)
  );
});
```

이 테스트는 사용자가 이미 만든 파일을 보호한다는 정책을 보여 줍니다.
자동화는 편해야 하지만, 사용자 데이터를 조용히 덮어쓰면 신뢰를 잃습니다.

## 실무 체크리스트

`fs.promises.cp()`를 배포나 생성 자동화에 넣기 전에는 아래 항목을 확인하면 좋습니다.

- 원본 경로와 대상 경로가 의도한 기준 디렉터리 안에 있는가?
- 디렉터리 복사라면 `recursive: true`가 필요한 이유가 분명한가?
- 대상 파일이 이미 있을 때 덮어쓸지, 실패할지 정책이 명시됐는가?
- `.env`, 임시 파일, 캐시, 테스트 산출물 같은 제외 대상이 `filter`에 반영됐는가?
- 심볼릭 링크를 보존할지, 따라갈지, 거부할지 정했는가?
- `preserveTimestamps`가 캐시와 증분 빌드에 미치는 영향을 확인했는가?
- 복사 뒤 필수 파일 검증을 수행하는가?
- 실패 로그에 민감한 전체 경로와 비밀 값이 노출되지 않는가?

이 체크리스트는 정적 사이트 배포뿐 아니라 문서 사이트, 이미지 파이프라인, 템플릿 생성 CLI에도 그대로 적용할 수 있습니다.

## FAQ

### fs.promises.cp와 copyFile은 어떻게 다를까?

`copyFile()`은 파일 하나를 복사할 때 적합합니다.
`cp()`는 디렉터리 구조 전체를 복사할 수 있고, `recursive`와 `filter` 같은 디렉터리 단위 옵션을 제공합니다.
파일 하나만 복사한다면 `copyFile()`이 더 명확하고, 여러 파일과 하위 디렉터리를 함께 다룬다면 `cp()`가 더 자연스럽습니다.

### 배포 전에 dist를 지우고 cp로 다시 복사해도 될까?

자동화가 만든 산출물이라면 가능합니다.
다만 삭제 대상과 복사 대상이 프로젝트 내부인지 검증하고, `rm()`으로 지운 뒤 `cp()`로 채우는 흐름이 실패했을 때 빈 `dist/`가 남을 수 있다는 점을 고려해야 합니다.
중요한 배포라면 임시 디렉터리에 먼저 복사한 뒤 검증이 끝난 결과만 교체하는 방식이 더 안전합니다.

### filter에서 비동기 검사를 해도 될까?

가능합니다.
공식 문서 기준 `filter`는 boolean으로 평가 가능한 값이나 Promise를 반환할 수 있습니다.
예를 들어 `lstat()`으로 심볼릭 링크를 검사하거나, 파일 크기 기준으로 복사 여부를 판단할 수 있습니다.
다만 파일마다 비동기 검사가 추가되면 큰 디렉터리에서는 성능에 영향을 줄 수 있으므로 제외 규칙은 단순하게 유지하는 편이 좋습니다.

## 마무리

`fs.promises.cp()`는 Node.js만으로 디렉터리 복사 자동화를 만들 수 있게 해 주는 실용적인 API입니다.
하지만 배포 산출물, 템플릿, 정적 파일처럼 여러 파일을 한 번에 다룰수록 작은 옵션 하나가 운영 정책이 됩니다.

기본값을 그대로 두기보다 `recursive`, `filter`, `force`, `errorOnExist`, `preserveTimestamps`, 심볼릭 링크 처리 기준을 코드에 명확히 남겨야 합니다.
복사는 단순한 파일 이동이 아니라 "무엇을 공개하고, 무엇을 보호하며, 실패를 어디서 멈출지"를 정하는 배포 설계의 일부입니다.

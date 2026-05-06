---
layout: post
title: "Node.js fsPromises.glob 가이드: 파일 탐색 코드를 단순하게 만드는 법"
date: 2026-05-07 08:00:00 +0900
lang: ko
translation_key: nodejs-fspromises-glob-file-discovery-guide
permalink: /development/blog/seo/2026/05/07/nodejs-fspromises-glob-file-discovery-guide.html
alternates:
  ko: /development/blog/seo/2026/05/07/nodejs-fspromises-glob-file-discovery-guide.html
  x_default: /development/blog/seo/2026/05/07/nodejs-fspromises-glob-file-discovery-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, fs, glob, automation, backend]
description: "Node.js fsPromises.glob를 이용해 파일 탐색 코드를 단순하게 만드는 방법을 정리했습니다. 재귀 파일 검색, 필터링, 실무 예제, 주의사항, 내부 링크까지 한 번에 살펴봅니다."
---

빌드 스크립트나 콘텐츠 자동화 작업을 만들다 보면, 결국 해야 하는 일은 단순합니다.
**"어떤 패턴에 맞는 파일을 안전하게 찾고, 그 목록을 다음 단계로 넘기는 것"**입니다.

문제는 이 단순한 요구를 구현할 때 `readdir()` 재귀 순회, 경로 필터링, 확장자 체크가 뒤섞이면서 코드가 빠르게 복잡해진다는 점입니다.
`fsPromises.glob()`은 이런 파일 탐색 코드를 훨씬 짧고 읽기 좋게 정리할 때 꽤 유용합니다.

## fsPromises.glob가 필요한 이유

### H3. 파일 탐색 로직은 생각보다 금방 중복된다

스크립트를 몇 개만 만들어도 이런 코드가 반복됩니다.

- `_posts`에서 특정 날짜 파일만 찾기
- `src` 아래에서 확장자별 파일 수집하기
- 로그 폴더에서 패턴에 맞는 파일만 모으기
- 배포 전 생성 산출물만 따로 훑기

이때 디렉터리 순회를 직접 구현하면, 파일 탐색 자체보다 예외 처리와 경로 조합 코드가 더 많아지기 쉽습니다.

### H3. 패턴 기반 탐색은 의도를 더 직접적으로 드러낸다

`glob`의 장점은 분명합니다.

- 찾고 싶은 대상을 패턴으로 바로 표현할 수 있다
- 재귀 순회를 직접 구현할 필요가 줄어든다
- 후속 필터링 로직이 더 단순해진다

특히 ESM 기반 스크립트에서 경로 기준점을 분명히 잡으려면 [Node.js import.meta.dirname 가이드: ESM에서 현재 파일 경로를 안전하게 다루는 법](/development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html)과 함께 보는 편이 좋습니다.

## fsPromises.glob 기본 사용법

### H3. 특정 패턴의 파일 목록을 모은다

```js
import { glob } from 'node:fs/promises';

const files = [];

for await (const file of glob('src/**/*.js')) {
  files.push(file);
}

console.log(files);
```

핵심은 `glob()`이 **비동기 iterable**을 돌려준다는 점입니다.
그래서 한꺼번에 배열로 모을 수도 있고, 필요하면 순차 처리도 할 수 있습니다.

### H3. 배열로 모아 다음 단계에 넘긴다

```js
import { glob } from 'node:fs/promises';

const files = await Array.fromAsync(glob('content/**/*.md'));

for (const file of files) {
  console.log(file);
}
```

파일 개수가 아주 많지 않다면 이런 방식이 읽기 쉽습니다.
비동기 iterable을 배열로 수집하는 감각은 [Node.js Array.fromAsync 가이드: 비동기 iterable을 배열로 안전하게 모으는 법](/development/blog/seo/2026/05/05/nodejs-array-fromasync-async-iterable-collection-guide.html)과도 자연스럽게 이어집니다.

## 실무에서 유용한 패턴

### H3. Jekyll 포스트 후보 파일만 골라낸다

```js
import { glob } from 'node:fs/promises';

const posts = await Array.fromAsync(
  glob('_posts/2026-05-*.md')
);

console.log(posts);
```

콘텐츠 운영 자동화에서는 날짜 규칙에 맞는 파일만 찾는 일이 자주 있습니다.
이 패턴을 쓰면 포스트 생성 여부 점검, 누락 검사, 후처리 스크립트를 단순하게 만들 수 있습니다.

### H3. 순차 처리로 메모리와 제어 흐름을 단순하게 유지한다

```js
import { glob } from 'node:fs/promises';
import { readFile } from 'node:fs/promises';

for await (const file of glob('logs/**/*.log')) {
  const content = await readFile(file, 'utf8');
  console.log(file, content.length);
}
```

파일이 많을 때는 전부 배열에 담은 뒤 한꺼번에 읽기보다, 이렇게 **찾는 즉시 처리하는 흐름**이 더 안전할 때가 많습니다.
큰 로그 파일을 다룰 때는 [Node.js FileHandle.readLines 가이드: 대용량 로그를 메모리 부담 없이 처리하는 법](/development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html)처럼 읽기 단계도 스트리밍 관점으로 이어서 설계하면 좋습니다.

### H3. 찾기와 검증을 분리한다

```js
import { glob } from 'node:fs/promises';
import path from 'node:path';

const files = await Array.fromAsync(glob('src/**/*.*'));

const jsFiles = files.filter((file) =>
  ['.js', '.mjs', '.cjs'].includes(path.extname(file))
);
```

`glob` 패턴 하나로 모든 정책을 해결하려 하기보다, **탐색은 탐색대로 두고 검증은 다음 단계로 분리하는 편**이 유지보수에 유리합니다.
나중에 허용 확장자나 제외 규칙이 바뀌어도 수정 지점이 분명해집니다.

## fsPromises.glob를 쓸 때 주의할 점

### H3. 파일이 너무 많으면 무조건 배열화하지 않는 편이 낫다

`Array.fromAsync(glob(...))`는 편하지만, 결과 개수가 커지면 메모리 사용량이 늘어납니다.
작업 목적이 "모두 수집"이 아니라 "하나씩 처리"라면 `for await...of`가 더 나은 기본값입니다.

### H3. 패턴이 넓을수록 의도하지 않은 파일이 섞일 수 있다

예를 들어 `**/*` 같은 패턴은 편하지만, 캐시 파일·빌드 산출물·숨김 파일까지 함께 잡을 수 있습니다.
실무에서는 아래처럼 생각하는 편이 안전합니다.

1. 먼저 탐색 범위를 가능한 한 좁힌다
2. 그다음 확장자나 파일명 규칙을 검증한다
3. 마지막으로 처리 대상만 후속 단계에 넘긴다

### H3. 상대 경로 기준점을 애매하게 두면 CI에서 흔들린다

로컬에서는 되는데 CI에서 경로가 달라지는 경우가 꽤 많습니다.
특히 스크립트 실행 위치가 매번 같지 않다면, 현재 작업 디렉터리에 기대기보다 기준 경로를 명시하는 편이 낫습니다.
경로 기준을 정리하는 방법은 앞서 연결한 `import.meta.dirname` 글과 함께 보면 더 실전적으로 감이 잡힙니다.

## 추천 적용 체크리스트

### H3. 도입 전에 이 다섯 가지를 먼저 확인한다

1. 파일 탐색 결과를 배열로 모아야 하는가, 바로 순차 처리하면 되는가?
2. 패턴 범위가 너무 넓어서 불필요한 파일이 섞이지 않는가?
3. 탐색 단계와 검증 단계를 분리했는가?
4. CI와 로컬에서 같은 기준 경로를 쓰는가?
5. 대용량 파일은 읽기 단계도 별도로 최적화했는가?

이 다섯 가지만 점검해도 파일 자동화 스크립트의 안정성이 꽤 올라갑니다.

## 마무리

`fsPromises.glob()`은 화려한 기능이라기보다, **반복되는 파일 탐색 코드를 의도 중심으로 줄여 주는 실용 도구**에 가깝습니다.
직접 재귀 순회를 짜는 것보다 짧고, 나중에 읽었을 때도 "무엇을 찾는 코드인지"가 더 분명하게 보입니다.

정리하면 이렇게 기억하면 됩니다.

- 파일 탐색 의도는 패턴으로 먼저 표현한다
- 결과가 많으면 배열화보다 순차 처리를 우선한다
- 탐색, 검증, 읽기 단계를 분리하면 운영이 쉬워진다

## 함께 보면 좋은 글

- [Node.js Array.fromAsync 가이드: 비동기 iterable을 배열로 안전하게 모으는 법](/development/blog/seo/2026/05/05/nodejs-array-fromasync-async-iterable-collection-guide.html)
- [Node.js FileHandle.readLines 가이드: 대용량 로그를 메모리 부담 없이 처리하는 법](/development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html)
- [Node.js import.meta.dirname 가이드: ESM에서 현재 파일 경로를 안전하게 다루는 법](/development/blog/seo/2026/04/30/nodejs-import-meta-dirname-filename-esm-guide.html)

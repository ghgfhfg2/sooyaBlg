---
layout: post
title: "Node.js FileHandle.readLines 가이드: 대용량 로그 파일을 메모리 폭증 없이 읽는 법"
date: 2026-04-29 20:00:00 +0900
lang: ko
translation_key: nodejs-filehandle-readlines-large-log-processing-guide
permalink: /development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html
alternates:
  ko: /development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html
  x_default: /development/blog/seo/2026/04/29/nodejs-filehandle-readlines-large-log-processing-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs, readline, filehandle, log-processing, stream, backend, javascript]
description: "Node.js의 FileHandle.readLines()로 대용량 로그 파일을 한 줄씩 안전하게 읽는 방법, 메모리 사용을 줄이는 패턴, 실무 주의사항을 예제와 함께 정리했습니다."
---

대용량 로그 파일을 다룰 때 가장 흔한 실수는 파일 전체를 한 번에 메모리로 올리는 것입니다.
`readFile()`은 간단하지만 파일 크기가 커질수록 메모리 사용량이 급격히 늘고, 운영 환경에서는 분석 스크립트 하나가 서버를 느리게 만들 수도 있습니다.

이럴 때 기본 선택지로 보기 좋은 것이 **Node.js의 `FileHandle.readLines()`** 입니다.
결론부터 말하면 `readLines()`를 쓰면 **파일 전체를 통째로 읽지 않고도 줄 단위로 순회할 수 있어서**, 로그 검색·집계·샘플링 같은 작업을 훨씬 안전하게 처리할 수 있습니다.

## Node.js FileHandle.readLines가 왜 유용할까

### H3. readFile은 간단하지만 큰 파일에서는 비용이 커진다

예를 들어 에러 로그에서 특정 문자열 개수를 세고 싶다고 가정해 보겠습니다.
처음에는 아래처럼 쓰기 쉽습니다.

```js
import { readFile } from 'node:fs/promises';

const text = await readFile('./app.log', 'utf8');
const count = text
  .split('\n')
  .filter((line) => line.includes('ERROR'))
  .length;

console.log(count);
```

작은 파일에서는 문제없지만, 파일이 수백 MB 이상 커지면 단점이 바로 드러납니다.

- 파일 전체를 메모리에 올려야 함
- 줄 분리용 배열까지 추가로 만들어짐
- CI나 서버에서 메모리 스파이크가 생길 수 있음
- 중간에 원하는 조건을 찾았어도 끝까지 읽게 됨

즉 “읽기 쉽다”와 “운영에 안전하다”는 다른 문제입니다.

### H3. readLines는 줄 단위 처리에 더 잘 맞는다

`FileHandle.readLines()`는 파일 핸들을 열고, 각 줄을 순차적으로 순회하는 방식입니다.
이 패턴은 다음 같은 상황에서 특히 잘 맞습니다.

- 대용량 로그에서 특정 패턴 검색
- CSV/TSV 비슷한 단순 줄 기반 파일의 집계
- 최근 에러 샘플 몇 개만 추출
- 조건을 만족하는 첫 줄을 찾으면 바로 종료

특히 파일 크기를 정확히 모르는 운영 로그 분석에서는 “전체를 읽지 않아도 된다”는 점이 꽤 큰 장점입니다.

## FileHandle.readLines 기본 사용법

### H3. fs/promises의 open으로 파일 핸들을 연 뒤 순회한다

기본 패턴은 단순합니다.

```js
import { open } from 'node:fs/promises';

const file = await open('./app.log');

try {
  for await (const line of file.readLines()) {
    console.log(line);
  }
} finally {
  await file.close();
}
```

이 코드에서 중요한 점은 세 가지입니다.

- `open()`으로 `FileHandle`을 만든다
- `for await...of`로 줄 단위 순회한다
- `finally`에서 `close()`로 정리한다

짧은 스크립트라도 파일 핸들 정리는 습관처럼 넣는 편이 좋습니다.

### H3. 조건 기반 집계는 한 줄씩 누적하는 방식이 안전하다

실무에서는 출력보다 집계가 더 흔합니다.
예를 들어 `ERROR` 로그 수를 세는 코드는 아래처럼 작성할 수 있습니다.

```js
import { open } from 'node:fs/promises';

async function countErrorLines(path) {
  const file = await open(path);
  let count = 0;

  try {
    for await (const line of file.readLines()) {
      if (line.includes('ERROR')) {
        count += 1;
      }
    }

    return count;
  } finally {
    await file.close();
  }
}

const count = await countErrorLines('./app.log');
console.log({ count });
```

이 방식은 [Node.js stream backpressure 가이드](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)에서 다룬 것처럼, 큰 입력을 한꺼번에 처리하지 않고 점진적으로 소비한다는 점에서 운영 안정성에 유리합니다.

## 대용량 로그 분석에 적용하는 패턴

### H3. 최근 에러 몇 줄만 필요하면 조기 종료가 가능하다

파일 전체 통계가 아니라 샘플 몇 줄만 필요할 때도 많습니다.
그럴 때는 조건을 만족하면 바로 빠져나오는 방식이 효율적입니다.

```js
import { open } from 'node:fs/promises';

async function collectFirstErrors(path, limit = 5) {
  const file = await open(path);
  const errors = [];

  try {
    for await (const line of file.readLines()) {
      if (!line.includes('ERROR')) continue;

      errors.push(line);

      if (errors.length >= limit) {
        break;
      }
    }

    return errors;
  } finally {
    await file.close();
  }
}
```

이 패턴은 “필요한 만큼만 읽고 끝내기”에 적합합니다.
운영 장애 초반에 샘플 로그만 빨리 보고 싶을 때 특히 유용합니다.

### H3. CLI 스크립트와 결합하면 운영 도구로 쓰기 좋다

`readLines()`는 간단한 내부 도구와도 잘 맞습니다.
아래처럼 파일 경로와 검색어를 인자로 받아 grep 비슷한 스크립트를 만들 수 있습니다.

```js
import { open } from 'node:fs/promises';
import { parseArgs } from 'node:util';

const {
  values: { file: filePath, keyword },
} = parseArgs({
  options: {
    file: { type: 'string' },
    keyword: { type: 'string' },
  },
});

const file = await open(filePath);

try {
  for await (const line of file.readLines()) {
    if (line.includes(keyword)) {
      console.log(line);
    }
  }
} finally {
  await file.close();
}
```

인자 파싱은 [Node.js util.parseArgs 가이드](/development/blog/seo/2026/04/28/nodejs-util-parseargs-cli-argument-parsing-guide.html)와 함께 보면 더 정리하기 쉽습니다.
작은 운영 스크립트를 빠르게 만드는 데 조합이 깔끔합니다.

## readLines를 쓸 때 알아둘 제한 사항

### H3. 줄 단위 처리가 곧 모든 텍스트 포맷에 맞는 것은 아니다

`readLines()`는 이름 그대로 줄 단위 처리에 최적화된 도구입니다.
그래서 아래 경우에는 주의가 필요합니다.

- 줄바꿈 안에 인용부호가 포함된 복잡한 CSV
- 레코드 경계가 줄바꿈과 일치하지 않는 포맷
- 파일 전체 문맥을 함께 봐야 하는 파서
- 랜덤 접근이 필요한 바이너리/인덱스 파일

즉 “텍스트가 여러 줄이다”와 “줄 단위 처리로 충분하다”는 다릅니다.
복잡한 CSV라면 전용 파서를 쓰는 편이 안전합니다.

### H3. CPU 집약 후처리가 크면 읽기 방식만 바꿔서는 부족하다

파일을 한 줄씩 읽는다고 해서 전체 성능 문제가 자동으로 해결되지는 않습니다.
정규식이 매우 무겁거나, 각 줄마다 큰 JSON 파싱/압축/암호화가 들어가면 병목은 다른 곳으로 옮겨갈 수 있습니다.

이때는 다음을 같이 봐야 합니다.

- 줄당 처리 비용 줄이기
- 병렬 처리 수 제한하기
- CPU 작업이면 worker 분리 검토하기
- 긴 처리 시간 동안 event loop 지연을 관찰하기

특히 처리 중 애플리케이션이 굼떠진다면 [Node.js event loop lag monitoring 가이드](/development/blog/seo/2026/04/13/nodejs-event-loop-lag-monitoring-guide.html)를 함께 보는 편이 좋습니다.

## 실무에서 흔한 실수

### H3. close 정리를 빼먹거나 예외 흐름을 고려하지 않는 경우

테스트 스크립트에서는 잘 안 드러나도, 반복 실행되는 도구에서는 파일 핸들 누수가 문제가 됩니다.
그래서 `try/finally`로 닫는 패턴은 거의 기본값으로 두는 편이 안전합니다.

또한 아래 같은 실수도 흔합니다.

- `for await...of` 대신 일반 `for...of`를 사용함
- 파일 인코딩/로그 포맷 가정을 코드에 하드코딩함
- 검색만 필요한데 전체 결과를 배열에 계속 쌓음
- 찾자마자 끝내도 되는 작업인데 끝까지 순회함

이런 부분만 정리해도 운영 스크립트 체감 품질이 꽤 좋아집니다.

### H3. readLines 하나로 모든 대용량 처리 문제를 해결하려고 하면 안 된다

`readLines()`는 아주 좋은 기본 도구이지만 만능은 아닙니다.
예를 들어 파일 변환 파이프라인, 압축 해제 스트림, 네트워크 스트림 연결처럼 흐름 제어가 중요한 경우에는 `stream.pipeline()` 중심 설계가 더 자연스러울 수 있습니다.

즉 기준은 간단합니다.

1. 줄 단위 텍스트 순회가 핵심이면 `readLines()`
2. 여러 스트림을 연결하고 에러 전파/백프레셔가 중요하면 `pipeline()`
3. 포맷이 복잡하면 전용 파서 사용

도구를 문제 형태에 맞추는 것이 가장 중요합니다.

## 실무 체크리스트

### H3. FileHandle.readLines 도입 전에 확인할 것

대용량 파일 처리 코드를 바꾸기 전에 아래 항목을 먼저 보면 좋습니다.

- 파일 전체를 메모리에 올리고 있지는 않은가
- 정말 줄 단위 처리로 충분한 포맷인가
- 원하는 결과가 전체 집계인지, 일부 샘플인지 구분했는가
- 조기 종료가 가능한 작업인가
- 파일 핸들 정리와 예외 처리를 넣었는가

이 다섯 가지만 정리해도 단순한 로그 분석 스크립트의 안정성이 꽤 올라갑니다.

## 마무리

Node.js의 `FileHandle.readLines()`는 거창한 기능은 아니지만, 대용량 텍스트 파일을 다룰 때 기본기를 크게 개선해 주는 도구입니다.
전체 파일 읽기에서 줄 단위 순회로만 바꿔도 메모리 사용량, 장애 대응 속도, 스크립트 유지보수성이 함께 좋아지는 경우가 많습니다.

로그 분석이나 운영용 유틸리티를 자주 만든다면, 먼저 `readFile()` 습관부터 점검해 보길 권합니다.
파일이 커질 가능성이 조금이라도 있다면 `readLines()`가 더 안전한 출발점일 수 있습니다.

## 관련 글

- [Node.js stream backpressure 가이드: 대용량 처리에서 메모리 급증을 막는 법](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)
- [Node.js util.parseArgs 가이드: CLI 인자를 표준 라이브러리로 깔끔하게 처리하는 법](/development/blog/seo/2026/04/28/nodejs-util-parseargs-cli-argument-parsing-guide.html)
- [Node.js event loop lag monitoring 가이드: 느려지는 서버를 지표로 잡아내는 법](/development/blog/seo/2026/04/13/nodejs-event-loop-lag-monitoring-guide.html)

---
layout: post
title: "Node.js test runner 가이드: 내장 테스트 도구로 빠르게 검증하는 법"
date: 2026-05-09 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-built-in-testing-guide
permalink: /development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html
alternates:
  ko: /development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html
  x_default: /development/blog/seo/2026/05/09/nodejs-test-runner-built-in-testing-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, test, testing, node-test, backend]
description: "Node.js 내장 test runner로 별도 테스트 프레임워크 없이 빠르게 검증 코드를 작성하는 방법을 정리했습니다. 기본 테스트, 비동기 테스트, 서브테스트, 실무 운영 팁까지 함께 설명합니다."
---

작은 유틸 함수나 배포 스크립트를 만들 때마다 테스트 프레임워크를 새로 고르는 일은 생각보다 번거롭습니다.
패키지를 설치하고 설정 파일을 만들고 실행 스크립트를 맞추다 보면, 정작 검증하려던 코드보다 테스트 환경 준비가 더 커질 때도 있습니다.

Node.js의 내장 `node:test` runner는 이런 상황에서 좋은 기본값이 됩니다.
핵심은 **별도 의존성 없이 테스트 파일을 만들고, `node --test`로 바로 실행할 수 있다**는 점입니다.
처음부터 거대한 테스트 체계를 만들기보다, 작은 검증을 빠르게 쌓고 싶은 프로젝트에 특히 잘 맞습니다.

## Node.js test runner가 필요한 이유

### H3. 작은 프로젝트에도 테스트 실행 기준이 필요하다

개발 블로그 예제, 배치 스크립트, CLI 도구, 내부 자동화 코드는 규모가 작다는 이유로 테스트가 뒤로 밀리기 쉽습니다.
하지만 이런 코드도 한 번 깨지면 배포, 파일 생성, API 호출 같은 후속 작업에 영향을 줍니다.

내장 test runner를 쓰면 다음 부담을 줄일 수 있습니다.

- 테스트 프레임워크 설치 없이 시작할 수 있다
- Node.js 표준 실행 방식으로 검증 절차를 통일할 수 있다
- CI나 로컬 스크립트에서 `node --test`만 호출하면 된다
- 간단한 유닛 테스트와 비동기 테스트를 같은 방식으로 다룰 수 있다

파일 탐색이나 콘텐츠 생성 자동화처럼 작은 스크립트를 자주 만든다면 [Node.js fsPromises.glob 가이드: 파일 탐색 코드를 단순하게 만드는 법](/development/blog/seo/2026/05/07/nodejs-fspromises-glob-file-discovery-guide.html)와 함께 테스트 기준을 잡아 두는 것이 좋습니다.

### H3. 테스트 도구 선택보다 검증 습관이 먼저다

Jest, Vitest, Mocha처럼 강력한 도구가 필요한 프로젝트도 많습니다.
다만 모든 코드가 처음부터 큰 테스트 프레임워크를 요구하는 것은 아닙니다.

내장 test runner의 장점은 선택을 미루면서도 검증을 시작할 수 있다는 데 있습니다.
프로젝트가 커져 다른 도구가 필요해지더라도, 초기에 작성한 테스트 케이스는 요구사항을 설명하는 문서처럼 남습니다.

## Node.js test runner 기본 사용법

### H3. node:test와 assert로 첫 테스트를 작성한다

가장 단순한 테스트는 `node:test`의 `test()`와 `node:assert/strict`를 함께 쓰는 방식입니다.

```js
// sum.test.js
import test from 'node:test';
import assert from 'node:assert/strict';

function sum(a, b) {
  return a + b;
}

test('sum adds two numbers', () => {
  assert.equal(sum(2, 3), 5);
});
```

실행은 아래처럼 할 수 있습니다.

```bash
node --test sum.test.js
```

여러 테스트 파일을 자동으로 찾게 하고 싶다면 파일명을 `*.test.js`, `*.spec.js`처럼 일관되게 두고 `node --test`만 실행해도 됩니다.

```bash
node --test
```

### H3. package.json 스크립트로 실행 기준을 고정한다

팀이나 CI에서 같은 명령을 쓰려면 `package.json`에 스크립트를 두는 편이 안전합니다.

```json
{
  "scripts": {
    "test": "node --test"
  }
}
```

이렇게 해 두면 로컬에서는 `npm test`, CI에서는 같은 스크립트를 호출하면 됩니다.
테스트 명령을 문서에만 적어 두는 것보다 저장소 안에 고정하는 편이 재현성이 좋습니다.

## 비동기 코드 테스트 패턴

### H3. async test로 Promise 결과를 검증한다

Node.js 코드는 파일 I/O, 네트워크, 데이터베이스처럼 비동기 흐름이 많습니다.
`node:test`는 테스트 함수가 Promise를 반환하면 완료될 때까지 기다립니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { readFile } from 'node:fs/promises';

test('reads config file', async () => {
  const text = await readFile('./config.example.json', 'utf8');
  const config = JSON.parse(text);

  assert.equal(config.env, 'test');
});
```

비동기 작업의 실행 시간을 함께 보고 싶다면 [Node.js performance.now 가이드: 실행 시간을 정확하게 측정하는 법](/development/blog/seo/2026/05/08/nodejs-performance-now-measure-duration-guide.html)의 방식처럼 측정 코드를 별도로 분리하는 편이 좋습니다.
테스트 안에서 성능 수치를 검증할 때는 환경 차이 때문에 너무 빡빡한 기준을 두지 않는 것이 안전합니다.

### H3. 실패해야 하는 비동기 흐름은 assert.rejects로 확인한다

오류 처리가 중요한 함수는 성공 케이스만큼 실패 케이스도 검증해야 합니다.
Promise가 거부되어야 하는 상황은 `assert.rejects()`로 확인할 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

async function loadUser(id) {
  if (!id) {
    throw new Error('id is required');
  }

  return { id };
}

test('loadUser rejects empty id', async () => {
  await assert.rejects(
    () => loadUser(''),
    /id is required/
  );
});
```

이 패턴을 쓰면 에러가 실제로 발생했는지, 메시지가 기대한 범위 안에 있는지까지 함께 확인할 수 있습니다.

## 서브테스트로 관련 케이스 묶기

### H3. 같은 함수의 여러 입력을 한 테스트 안에서 정리한다

비슷한 입력값을 여러 개 검증할 때는 `t.test()`로 서브테스트를 만들 수 있습니다.
테스트 이름을 잘 붙이면 실패한 조건을 빠르게 찾을 수 있습니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function normalizeTag(tag) {
  return tag.trim().toLowerCase().replaceAll(' ', '-');
}

test('normalizeTag', async (t) => {
  await t.test('trims spaces', () => {
    assert.equal(normalizeTag(' nodejs '), 'nodejs');
  });

  await t.test('lowercases text', () => {
    assert.equal(normalizeTag('JavaScript'), 'javascript');
  });

  await t.test('replaces spaces with hyphen', () => {
    assert.equal(normalizeTag('web api'), 'web-api');
  });
});
```

서브테스트는 관련 케이스를 묶어 읽기 쉽게 만드는 데 유용합니다.
다만 너무 많은 검증을 하나의 테스트에 몰아넣으면 실패 지점이 흐려질 수 있으니, 기능 단위로 적당히 나누는 편이 좋습니다.

### H3. 테스트 데이터는 명확한 값으로 둔다

테스트에서 무작위 값을 쓰면 케이스가 더 현실적으로 보일 수 있지만, 실패 재현이 어려워질 수 있습니다.
식별자 생성 자체를 테스트하는 경우가 아니라면 입력값은 고정된 예제를 쓰는 편이 안전합니다.

운영 코드에서 안전한 ID가 필요할 때는 [Node.js crypto.randomUUID 가이드: 안전한 식별자를 직접 만드는 법](/development/blog/seo/2026/05/08/nodejs-crypto-randomuuid-safe-id-generation-guide.html)처럼 표준 API를 쓰되, 테스트에서는 생성 함수를 주입하거나 고정값으로 대체하는 방식을 고려합니다.

## 실무 적용 체크리스트

### H3. 테스트 파일 위치와 이름을 먼저 정한다

프로젝트 초기에 아래 기준을 정해 두면 테스트가 흩어지는 일을 줄일 수 있습니다.

- 소스 옆에 `*.test.js`로 둔다
- 또는 `tests/` 디렉터리에 기능별로 모은다
- 테스트 대상과 테스트 파일 이름을 비슷하게 맞춘다
- CI에서는 항상 같은 `npm test` 명령을 실행한다

정답은 하나가 아닙니다.
중요한 것은 팀과 자동화가 같은 위치를 바라보게 만드는 것입니다.

### H3. 외부 의존성은 가능한 한 경계 밖으로 뺀다

테스트가 느려지는 가장 흔한 이유는 외부 API, 실제 파일 시스템, 시간, 랜덤 값이 테스트 본문에 깊게 들어오기 때문입니다.
처음에는 다음 원칙만 지켜도 유지보수가 쉬워집니다.

- 순수 함수는 입력과 출력만 검증한다
- 파일이나 네트워크 호출은 작은 어댑터로 분리한다
- 현재 시각, UUID, 환경 변수는 함수 인자로 주입할 수 있게 만든다
- 실패 케이스를 최소 1개 이상 포함한다

시간이 들어가는 코드를 다룬다면 [Node.js test runner mock timers 가이드: 시간 의존 테스트를 안정적으로 만드는 법](/development/blog/seo/2026/04/29/nodejs-test-runner-mock-timers-guide.html)을 함께 참고하면 테스트 변동성을 줄이는 데 도움이 됩니다.

## 마무리

Node.js 내장 test runner는 모든 테스트 문제를 해결하는 도구는 아닙니다.
하지만 작은 프로젝트나 내부 자동화 코드에서 **검증을 시작하는 비용을 크게 낮춰 주는 기본 도구**입니다.

처음에는 `node:test`, `node:assert/strict`, `node --test` 세 가지만으로도 충분합니다.
그 위에 비동기 테스트, 서브테스트, 시간 의존 코드 분리 같은 습관을 조금씩 더하면 테스트는 부담스러운 의식이 아니라 배포 전 안전장치가 됩니다.

## FAQ

### H3. Node.js test runner만으로 충분한가요?

작은 라이브러리, 자동화 스크립트, 백엔드 유틸 함수 검증에는 충분한 경우가 많습니다.
스냅샷, 브라우저 테스트, 복잡한 mocking 생태계가 필요하다면 전용 프레임워크를 검토하는 편이 좋습니다.

### H3. 기존 Jest나 Vitest 프로젝트도 바꿔야 하나요?

그럴 필요는 없습니다.
이미 잘 운영되는 테스트 체계가 있다면 유지하는 것이 낫습니다.
내장 test runner는 새 프로젝트를 가볍게 시작하거나 작은 패키지에 검증 기준을 추가할 때 특히 유용합니다.

### H3. 테스트가 느려지면 무엇부터 봐야 하나요?

외부 API 호출, 실제 대용량 파일 처리, 불필요한 대기 시간부터 확인하는 것이 좋습니다.
테스트가 검증하려는 핵심 로직과 외부 경계를 분리하면 실행 시간과 실패 원인을 더 쉽게 관리할 수 있습니다.

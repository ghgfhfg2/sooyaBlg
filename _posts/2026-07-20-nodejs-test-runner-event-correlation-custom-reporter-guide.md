---
layout: post
title: "Node.js test runner 이벤트 상관관계 가이드: testId와 parentId로 CI 리포트 정리하기"
date: 2026-07-20 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-event-correlation-custom-reporter-guide
permalink: /development/blog/seo/2026/07/20/nodejs-test-runner-event-correlation-custom-reporter-guide.html
alternates:
  ko: /development/blog/seo/2026/07/20/nodejs-test-runner-event-correlation-custom-reporter-guide.html
  x_default: /development/blog/seo/2026/07/20/nodejs-test-runner-event-correlation-custom-reporter-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, custom-reporter, testid, parentid, ci, testing, observability]
description: "Node.js test runner의 testId, parentId, tags 이벤트 필드로 custom reporter와 CI 리포트에서 테스트 상관관계를 정리하는 방법을 설명합니다. 중첩 테스트, 병렬 실행, 실패 요약, 민감정보 없는 로그 기준을 예제로 정리합니다."
---

테스트 리포트가 읽기 어려워지는 순간은 실패가 많을 때만이 아닙니다.
`describe()`와 하위 테스트가 깊어지고, 병렬 실행과 shard가 섞이고, custom reporter가 이벤트를 순서대로만 처리하면 어느 실패가 어떤 suite 아래에서 나온 것인지 금방 흐려집니다.
특히 CI에서 테스트 이벤트를 JSON Lines로 저장하거나 PR 코멘트로 요약하려면 테스트 이름 문자열만으로는 부족할 때가 많습니다.

Node.js test runner의 이벤트에는 테스트를 서로 연결할 수 있는 메타데이터가 들어 있습니다.
공식 문서 기준으로 `test:start`, `test:pass`, `test:fail` 같은 이벤트의 `data`에는 `testId`, `parentId`, `name`, `file`, `tags` 같은 필드가 포함될 수 있습니다.
이 글에서는 이 필드들을 활용해 custom reporter에서 테스트 계층을 복원하고, CI 리포트를 더 안정적으로 정리하는 방법을 설명합니다.
[Node.js test runner custom reporter 가이드](/development/blog/seo/2026/07/12/nodejs-test-runner-custom-reporter-ci-report-guide.html), [Node.js test runner reporter 선택 가이드](/development/blog/seo/2026/07/18/nodejs-test-runner-reporters-spec-tap-dot-junit-guide.html), [Node.js test runner run() API 가이드](/development/blog/seo/2026/07/19/nodejs-test-runner-run-api-custom-runner-guide.html)와 함께 보면 테스트 결과 수집 흐름을 더 탄탄하게 설계할 수 있습니다.

## 테스트 이벤트 상관관계가 필요한 이유

### H3. 테스트 이름만으로는 충분하지 않다

작은 테스트 파일에서는 실패 이름만 봐도 원인을 찾을 수 있습니다.

```text
fails: returns 400 when email is invalid
```

하지만 큰 테스트 스위트에서는 같은 이름이 여러 suite에 반복될 수 있습니다.
예를 들어 `returns 400 when email is invalid`가 회원가입 API, 초대 API, 관리자 API 테스트에 모두 있을 수 있습니다.
이때 reporter가 이름만 저장하면 실패 목록을 다시 파일과 suite 구조로 맞춰 보는 시간이 늘어납니다.

테스트 이름에 모든 맥락을 길게 넣는 방법도 있지만, 이름이 지나치게 장황해집니다.
좋은 리포트는 사람이 읽기 쉬운 이름과 시스템이 연결할 수 있는 식별자를 함께 가져야 합니다.
`testId`와 `parentId`는 이 역할을 맡을 수 있습니다.

### H3. 병렬 실행에서는 이벤트 순서에 의존하면 위험하다

병렬 테스트에서는 이벤트가 선언 순서와 다르게 들어올 수 있습니다.
하위 테스트 A가 시작된 뒤 하위 테스트 B가 먼저 끝나고, A가 나중에 실패할 수 있습니다.
custom reporter가 "직전에 본 suite가 현재 테스트의 부모일 것이다"처럼 순서에 의존하면 잘못된 계층을 만들 수 있습니다.

Node.js test runner 이벤트의 `parentId`는 이런 상황에서 부모 테스트나 suite를 연결하는 단서입니다.
이벤트가 섞여 들어와도 `testId`를 key로 저장하고 `parentId`를 따라가면 리포트 계층을 다시 구성할 수 있습니다.

## 이벤트 필드 이해하기

### H3. testId는 한 테스트 인스턴스를 따라가는 키다

`testId`는 테스트 파일의 실행 프로세스 안에서 테스트 인스턴스를 구분하는 숫자 식별자입니다.
custom reporter에서는 이 값을 `Map`의 key로 두고, start/pass/fail 이벤트가 들어올 때 같은 테스트 정보를 갱신할 수 있습니다.

```js
const tests = new Map();

function rememberTest(event) {
  const data = event.data;
  if (!data?.testId) return;

  const previous = tests.get(data.testId) ?? {};
  tests.set(data.testId, {
    ...previous,
    testId: data.testId,
    parentId: data.parentId,
    name: data.name,
    file: data.file,
    tags: data.tags ?? [],
    type: data.type
  });
}
```

이 구조는 이벤트 타입과 무관하게 공통 메타데이터를 저장합니다.
나중에 실패 이벤트가 들어왔을 때 이미 저장된 start 이벤트의 정보를 함께 사용할 수 있습니다.

단, `testId`는 전역적으로 영구적인 ID가 아닙니다.
실행마다 달라질 수 있고, 파일이나 프로세스 범위 안에서 의미가 있습니다.
장기 저장용 고유 키가 필요하다면 `file`, 계층 이름, 테스트 이름, 태그를 조합한 별도 stable key를 만들어야 합니다.

### H3. parentId는 suite와 subtest 계층을 복원한다

`parentId`는 현재 테스트를 감싸는 부모 테스트 또는 suite의 `testId`를 가리킵니다.
부모가 없는 top-level 테스트라면 값이 없을 수 있습니다.
이 필드를 따라가면 중첩된 suite 이름을 배열로 만들 수 있습니다.

```js
function getPath(testId) {
  const path = [];
  let current = tests.get(testId);

  while (current) {
    path.unshift(current.name);
    current = current.parentId ? tests.get(current.parentId) : undefined;
  }

  return path;
}
```

실패 리포트에서는 이 경로를 `"User API > create user > returns 400"`처럼 보여 줄 수 있습니다.
파일 경로와 테스트 이름만 나열하는 것보다 훨씬 읽기 쉽고, 이름이 반복되어도 어느 suite 아래의 실패인지 분명해집니다.

### H3. tags는 리포트 필터링 기준이 된다

Node.js test runner의 tags는 테스트와 suite에 붙이는 라벨입니다.
이벤트 데이터의 `tags`는 상위 suite에서 물려받은 태그까지 포함할 수 있으므로, custom reporter에서 실패를 분류하기 좋습니다.

```js
function classifyFailure(test) {
  if (test.tags?.includes('flaky')) return 'flaky';
  if (test.tags?.includes('integration')) return 'integration';
  if (test.tags?.includes('slow')) return 'slow';
  return 'default';
}
```

CI 리포트에서 태그를 잘 쓰면 실패를 무작정 한 목록에 섞지 않아도 됩니다.
예를 들어 `integration` 실패는 외부 의존성 상태를 함께 확인하고, `flaky` 실패는 재시도 결과와 attempt 정보를 같이 보도록 분리할 수 있습니다.

## custom reporter에서 계층형 실패 요약 만들기

### H3. 이벤트를 먼저 축적한 뒤 출력한다

테스트 이벤트가 들어오는 즉시 출력하면 구현은 단순합니다.
하지만 부모 이벤트가 아직 처리되지 않았거나, 병렬 실행 때문에 경로 계산에 필요한 정보가 뒤늦게 들어올 수 있습니다.
따라서 실패 목록을 배열에 모아 두고, stream이 끝날 때 최종 요약을 출력하는 방식이 더 안전합니다.

```js
// reporters/failure-tree.js
import { Transform } from 'node:stream';

const tests = new Map();
const failures = [];

export default new Transform({
  writableObjectMode: true,
  transform(event, encoding, callback) {
    remember(event);

    if (event.type === 'test:fail') {
      failures.push({
        testId: event.data?.testId,
        message: sanitize(event.data?.details?.error?.message),
        durationMs: event.data?.details?.duration_ms
      });
    }

    callback();
  },
  flush(callback) {
    for (const failure of failures) {
      const test = tests.get(failure.testId);
      const payload = {
        type: 'test-failure',
        file: maskPath(test?.file),
        path: getPath(failure.testId),
        tags: test?.tags ?? [],
        durationMs: failure.durationMs,
        message: failure.message
      };

      this.push(`${JSON.stringify(payload)}\n`);
    }

    callback();
  }
});

function remember(event) {
  const data = event.data;
  if (!data?.testId) return;

  tests.set(data.testId, {
    ...(tests.get(data.testId) ?? {}),
    testId: data.testId,
    parentId: data.parentId,
    name: data.name,
    file: data.file,
    tags: data.tags ?? [],
    type: data.type
  });
}

function getPath(testId) {
  const path = [];
  let current = tests.get(testId);

  while (current) {
    path.unshift(current.name);
    current = current.parentId ? tests.get(current.parentId) : undefined;
  }

  return path;
}

function maskPath(file) {
  return file ? file.replace(process.cwd(), '<repo>') : undefined;
}

function sanitize(message) {
  if (!message) return undefined;

  return String(message)
    .replaceAll(/Bearer\s+[A-Za-z0-9._-]+/g, 'Bearer [REDACTED]')
    .replaceAll(/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/g, '[EMAIL]');
}
```

이 reporter는 실패 이벤트를 그대로 콘솔에 길게 출력하지 않습니다.
대신 실패한 테스트의 계층 경로, 파일, 태그, duration, 정리된 메시지만 JSON Lines로 남깁니다.
후속 스크립트는 이 파일을 읽어 PR 코멘트나 대시보드용 요약을 만들 수 있습니다.

### H3. reporter 실행 명령을 분리한다

custom reporter는 built-in reporter와 함께 쓰는 편이 좋습니다.
콘솔에는 사람이 바로 읽을 수 있는 `spec` reporter를 남기고, 파일에는 위에서 만든 구조화 reporter를 저장합니다.

```bash
mkdir -p reports

node --test \
  --test-reporter=spec \
  --test-reporter-destination=stdout \
  --test-reporter=./reporters/failure-tree.js \
  --test-reporter-destination=./reports/failures.ndjson
```

이렇게 하면 개발자는 CI 로그에서 기본 실패 내용을 바로 보고, 자동화는 `reports/failures.ndjson`를 안정적으로 읽습니다.
reporter가 실패를 숨기거나 성공 여부를 바꾸지 않는 것도 중요합니다.
custom reporter는 테스트 결과를 표현하는 계층이고, 테스트 판정은 test runner가 맡아야 합니다.

## CI 리포트에 넣을 필드 기준

### H3. 사람이 읽는 필드와 시스템 필드를 나눈다

CI 리포트에는 읽기 좋은 정보와 자동 처리용 정보가 함께 필요합니다.
아래 정도를 기본 필드로 잡으면 대부분의 실패 요약에 충분합니다.

```json
{
  "type": "test-failure",
  "file": "<repo>/test/users/create-user.test.mjs",
  "path": ["User API", "create user", "returns 400 when email is invalid"],
  "tags": ["integration"],
  "durationMs": 42.3,
  "message": "Expected values to be strictly equal"
}
```

`path`는 사람이 읽는 계층입니다.
`file`은 재현할 위치입니다.
`tags`는 실패를 분류하는 기준입니다.
`durationMs`는 느린 테스트와 flaky 의심 테스트를 추적하는 데 도움이 됩니다.
메시지는 길이를 제한하고 민감정보를 마스킹해야 합니다.

반면 환경 변수 전체, 원본 CI 컨텍스트, access token, 실제 사용자 이메일, 내부 hostname은 리포트에 넣지 않는 편이 좋습니다.
테스트 리포트는 artifact, 알림, PR 코멘트로 복사되기 쉬운 데이터입니다.

### H3. 장기 분석용 stable key는 따로 만든다

`testId`는 실행 중 이벤트를 연결하는 데 좋지만, 장기 분석용 key로는 부족합니다.
실행마다 값이 바뀔 수 있기 때문입니다.
flaky test 추세를 보려면 파일과 계층 경로를 조합해 stable key를 만들어야 합니다.

```js
import { createHash } from 'node:crypto';

function stableKey({ file, path }) {
  return createHash('sha256')
    .update(JSON.stringify({ file: maskPath(file), path }))
    .digest('hex')
    .slice(0, 16);
}
```

해시는 선택입니다.
내부 대시보드에서 사람이 읽는 key가 더 중요하다면 `file + path.join(' > ')`를 그대로 써도 됩니다.
중요한 점은 `testId`와 stable key의 역할을 섞지 않는 것입니다.

`testId`는 현재 실행의 이벤트 상관관계용입니다.
stable key는 여러 실행을 비교하는 분석용입니다.

## 병렬 실행과 shard에서 주의할 점

### H3. 파일 범위를 함께 저장한다

공식 문서 기준으로 test runner는 process isolation을 기본으로 사용할 수 있고, 병렬 실행이나 shard 구성에서는 여러 파일이 독립적으로 실행됩니다.
이때 `testId`만 저장하면 서로 다른 파일의 같은 숫자 ID가 충돌할 수 있습니다.
reporter에서는 항상 `file`과 함께 다루는 편이 안전합니다.

```js
function eventKey(data) {
  return `${data.file ?? '<unknown>'}:${data.testId}`;
}
```

앞의 예제처럼 단일 reporter 프로세스 안에서 `testId`만으로도 동작하는 경우가 있지만, 리포트 파일을 합치거나 shard별 결과를 모을 때는 파일 경로와 shard 정보를 함께 저장하세요.
특히 여러 CI job이 같은 `reports/failures.ndjson` 이름을 쓰면 결과가 덮어써질 수 있으므로 shard 번호를 파일 이름에 포함하는 편이 좋습니다.

```bash
node --test \
  --test-shard="${SHARD_INDEX}/${SHARD_TOTAL}" \
  --test-reporter=./reporters/failure-tree.js \
  --test-reporter-destination="reports/failures-${SHARD_INDEX}.ndjson"
```

### H3. 순서가 아니라 관계로 정렬한다

최종 리포트를 만들 때 이벤트 도착 순서를 그대로 보여 주면 실패 목록이 매번 달라질 수 있습니다.
이런 리포트는 PR 코멘트 diff가 지저분해지고, 같은 실패인지 비교하기 어렵습니다.

출력 직전에 정렬 기준을 고정하세요.

```js
failures.sort((a, b) => {
  const left = tests.get(a.testId);
  const right = tests.get(b.testId);

  return [
    left?.file ?? '',
    getPath(a.testId).join(' > ')
  ].join('\n').localeCompare([
    right?.file ?? '',
    getPath(b.testId).join(' > ')
  ].join('\n'));
});
```

파일 경로와 계층 경로를 기준으로 정렬하면 병렬 실행이어도 리포트가 안정됩니다.
리포트가 안정되면 실패 원인만 눈에 들어오고, 출력 순서 변화 때문에 리뷰가 흔들리지 않습니다.

## 로그 위생과 안전 점검

### H3. 실패 메시지는 원문 그대로 저장하지 않는다

테스트 실패 메시지는 개발자가 직접 작성한 assertion 메시지, 외부 API 응답, fixture 데이터, 환경 변수 일부를 포함할 수 있습니다.
따라서 custom reporter는 실패 메시지를 저장하기 전에 길이 제한과 마스킹을 적용해야 합니다.

```js
function safeMessage(error) {
  return String(error?.message ?? '')
    .slice(0, 500)
    .replaceAll(/sk-[A-Za-z0-9_-]+/g, 'sk-[REDACTED]')
    .replaceAll(/Bearer\s+[A-Za-z0-9._-]+/g, 'Bearer [REDACTED]')
    .replaceAll(/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/g, '[EMAIL]');
}
```

이 정규식이 모든 비밀을 완벽히 막지는 않습니다.
그래도 reporter 계층에서 기본적인 방어선을 두면 실수로 민감정보가 artifact에 남을 위험을 줄일 수 있습니다.
더 중요한 것은 테스트 데이터 자체를 처음부터 안전한 fixture로 만드는 것입니다.

### H3. 내부 경로는 필요한 만큼만 남긴다

로컬 사용자명, 임시 디렉터리, CI workspace 전체 경로는 리포트에 그대로 남길 필요가 거의 없습니다.
파일 위치를 재현할 수 있을 정도로만 정규화하세요.

```js
function normalizeFile(file) {
  if (!file) return undefined;
  return file.replace(process.cwd(), '<repo>');
}
```

PR 코멘트나 외부 알림으로 리포트를 보낼 계획이 있다면 더 보수적으로 접근하는 편이 좋습니다.
저장소 내부 상대 경로만 남기고, 절대 경로는 모두 제거하세요.

## 적용 전 체크리스트

### H3. reporter를 만들기 전에 확인할 것

- 실패 테스트가 중첩 suite 아래에서 반복되는가
- 병렬 실행 때문에 이벤트 순서가 흔들리는가
- CI 리포트가 사람용 출력과 시스템용 파일을 구분하는가
- `testId`는 실행 중 상관관계용으로만 쓰는가
- 장기 분석용 stable key를 별도로 만들었는가
- 파일 경로, 이메일, 토큰, 내부 URL을 마스킹하는가
- shard별 리포트 파일 이름이 서로 덮어쓰지 않는가

이 질문에 답하기 어렵다면 먼저 built-in `spec`과 `junit` reporter를 조합하고, 실패 요약이 정말 부족한 지점만 custom reporter로 보완하는 편이 좋습니다.

## FAQ

### H3. testId를 데이터베이스에 저장해도 되나요?

실행 중 이벤트를 연결하는 용도로는 좋지만, 장기 저장용 고유 ID로 쓰기에는 적합하지 않습니다.
실행마다 달라질 수 있으므로 파일 경로와 계층 경로를 조합한 stable key를 별도로 만드세요.

### H3. parentId가 없으면 잘못된 이벤트인가요?

그렇지 않습니다.
top-level 테스트나 suite는 부모가 없을 수 있습니다.
리포트에서는 해당 테스트 이름을 경로의 첫 항목으로 두면 됩니다.

### H3. custom reporter가 꼭 필요한가요?

대부분의 프로젝트는 `spec`, `tap`, `junit` 같은 built-in reporter 조합으로 충분합니다.
custom reporter는 중첩 테스트 계층, 태그별 실패 분류, PR 코멘트 자동화처럼 팀의 운영 요구가 분명할 때 작게 추가하는 편이 좋습니다.

## 마무리

Node.js test runner 이벤트의 `testId`와 `parentId`는 custom reporter를 더 신뢰할 수 있게 만드는 작은 기반입니다.
이벤트 순서에 의존하지 않고 테스트 계층을 복원하면, 병렬 실행과 shard가 섞인 CI에서도 실패 리포트를 안정적으로 만들 수 있습니다.

핵심은 역할을 나누는 것입니다.
`testId`는 현재 실행 안에서 이벤트를 연결하고, `parentId`는 suite 경로를 복원하며, stable key는 장기 분석에 사용합니다.
여기에 태그 분류와 로그 마스킹을 더하면 테스트 리포트는 단순한 실패 목록을 넘어 운영 가능한 품질 신호가 됩니다.

## 참고 자료

- [Node.js test runner 공식 문서](https://nodejs.org/api/test.html)
- [Node.js test runner custom reporter 가이드](/development/blog/seo/2026/07/12/nodejs-test-runner-custom-reporter-ci-report-guide.html)
- [Node.js test runner run() API 가이드](/development/blog/seo/2026/07/19/nodejs-test-runner-run-api-custom-runner-guide.html)

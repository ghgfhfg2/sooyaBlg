---
layout: post
title: "Node.js test runner workerId와 attempt 가이드: 병렬 CI와 재시도 테스트를 추적하는 법"
date: 2026-07-18 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-workerid-attempt-flaky-ci-guide
permalink: /development/blog/seo/2026/07/18/nodejs-test-runner-workerid-attempt-flaky-ci-guide.html
alternates:
  ko: /development/blog/seo/2026/07/18/nodejs-test-runner-workerid-attempt-flaky-ci-guide.html
  x_default: /development/blog/seo/2026/07/18/nodejs-test-runner-workerid-attempt-flaky-ci-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, workerid, attempt, flaky-test, ci, testing, javascript]
description: "Node.js test runner의 context.workerId와 context.attempt로 병렬 CI, 재시도 테스트, flaky test 원인을 추적하는 방법을 정리합니다. 테스트 로그 설계, 임시 리소스 분리, 민감정보 점검 기준까지 예제로 설명합니다."
---

CI에서만 실패하는 테스트는 원인을 좁히기 어렵습니다.
로컬에서는 항상 통과하는데 병렬 실행 worker 하나에서만 깨지거나, 첫 번째 실행은 실패하고 재시도에서는 통과하는 경우가 특히 그렇습니다.
이때 실패 로그에 테스트 이름만 남아 있으면 "어느 실행 환경에서, 몇 번째 시도에서, 어떤 파일이 문제였는지"를 다시 추적해야 합니다.

Node.js test runner는 테스트 컨텍스트에 `context.workerId`와 `context.attempt` 같은 실행 정보를 제공합니다.
공식 문서 기준으로 `workerId`는 현재 테스트 파일을 실행하는 worker 식별자이고, `attempt`는 `--test-rerun-failures`와 함께 현재 몇 번째 시도인지 볼 때 유용합니다.
이 글에서는 병렬 CI에서 `workerId`와 `attempt`를 로그와 fixture 설계에 반영하는 방법, flaky test를 추적할 때 남겨야 할 최소 정보, 민감정보를 피하는 기준을 정리합니다.
[Node.js test runner rerun failures 가이드](/development/blog/seo/2026/07/03/nodejs-test-runner-rerun-failures-ci-guide.html), [Node.js test runner shard 가이드](/development/blog/seo/2026/07/04/nodejs-test-runner-shard-ci-parallel-guide.html), [Node.js test runner custom reporter 가이드](/development/blog/seo/2026/07/12/nodejs-test-runner-custom-reporter-ci-report-guide.html)와 함께 보면 CI 테스트 추적 흐름을 더 안정적으로 만들 수 있습니다.

## workerId와 attempt가 필요한 상황

### H3. 병렬 실행에서는 실패 위치보다 실행 단서가 먼저 필요하다

테스트가 병렬로 실행되면 같은 테스트 파일이라도 worker 배치, 임시 디렉터리, 외부 포트, fixture 초기화 순서에 따라 결과가 달라질 수 있습니다.
실패 메시지에 assertion diff만 남아 있으면 실제 문제는 아래 중 어디인지 구분하기 어렵습니다.

- worker별 임시 파일 경로가 겹쳤는가?
- 병렬 테스트가 같은 포트나 같은 데이터베이스 이름을 공유했는가?
- 특정 worker에서만 환경 변수가 다르게 들어갔는가?
- 재시도에서는 통과하는 flaky test인가?
- shard 분산과 worker 병렬 실행이 함께 영향을 줬는가?

`workerId`와 `attempt`는 이런 질문에 답하기 위한 작은 단서입니다.
테스트 성공 여부를 바꾸는 값은 아니지만, 실패 로그를 읽을 때 원인 후보를 빠르게 줄여 줍니다.

### H3. 재시도 통과는 성공이 아니라 신호다

`--test-rerun-failures`를 쓰면 실패한 테스트만 다시 실행해 CI 시간을 줄일 수 있습니다.
하지만 두 번째 시도에서 통과했다고 해서 문제가 사라진 것은 아닙니다.
첫 번째 시도에서 실패하고 재시도에서 통과했다면 시간 의존성, 전역 상태 공유, 네트워크 지연, 리소스 경쟁을 의심해야 합니다.

`context.attempt`는 이런 상황을 로그에 남기는 데 유용합니다.
첫 시도는 `0`, 두 번째 시도는 `1`처럼 0부터 시작하는 숫자로 표현되므로, 실패 리포트에서 "재시도 덕분에 통과한 테스트"를 분리해 볼 수 있습니다.

## context.workerId로 병렬 실행 단서 남기기

### H3. 테스트 로그에 workerId를 구조화해서 넣는다

Node.js test runner의 테스트 함수는 컨텍스트 객체를 받을 수 있습니다.
이 객체의 `workerId`를 읽어 테스트 진단 로그나 JSON 로그에 포함하면 병렬 실행 중 어떤 worker에서 문제가 발생했는지 확인할 수 있습니다.

```js
// test/order-summary.test.mjs
import { test } from 'node:test';
import assert from 'node:assert/strict';

function createOrderSummary(items) {
  return {
    count: items.length,
    total: items.reduce((sum, item) => sum + item.price, 0)
  };
}

test('creates order summary', (t) => {
  t.diagnostic(JSON.stringify({
    test: t.fullName,
    file: t.filePath,
    workerId: t.workerId
  }));

  assert.deepStrictEqual(
    createOrderSummary([{ price: 1200 }, { price: 800 }]),
    { count: 2, total: 2000 }
  );
});
```

여기서 중요한 점은 payload를 넣지 않는 것입니다.
테스트 fixture 안에는 실제 이메일처럼 보이는 값, 토큰과 비슷한 문자열, 결제 식별자처럼 민감하게 취급해야 하는 값이 섞일 수 있습니다.
로그에는 테스트 이름, 파일, worker 번호, attempt 번호처럼 원인 분석에 필요한 최소 메타데이터만 남기는 편이 안전합니다.

### H3. workerId는 리소스 이름을 분리할 때도 쓸 수 있다

병렬 테스트에서 가장 흔한 문제는 공유 리소스 충돌입니다.
임시 파일, SQLite 데이터베이스, mock server port, cache key, object storage prefix를 모든 worker가 같이 쓰면 순서 의존 테스트가 됩니다.

`workerId`를 리소스 이름에 포함하면 worker 간 충돌 가능성을 줄일 수 있습니다.

```js
// test/helpers/temp-resource.mjs
import { mkdtemp } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

export async function createWorkerTempDir(t, label) {
  const safeLabel = label.replace(/[^a-z0-9-]/gi, '-').toLowerCase();
  const worker = t.workerId ?? 'single';
  return mkdtemp(join(tmpdir(), `node-test-${worker}-${safeLabel}-`));
}
```

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { createWorkerTempDir } from './helpers/temp-resource.mjs';

test('writes export file in isolated directory', async (t) => {
  const dir = await createWorkerTempDir(t, 'export');

  assert.match(dir, /node-test-/);
  t.diagnostic(`workerId=${t.workerId ?? 'single'}, tempDir=<tmp>`);
});
```

진단 메시지에는 실제 임시 경로 전체를 남기지 않고 `<tmp>`처럼 정규화한 값을 쓰는 것이 좋습니다.
절대 경로에는 사용자명, 프로젝트명, CI runner 내부 구조가 들어갈 수 있기 때문입니다.

## context.attempt로 재시도 흐름 추적하기

### H3. attempt를 실패 분석용 메타데이터로 남긴다

`context.attempt`는 테스트가 현재 몇 번째 시도인지 알려 줍니다.
`--test-rerun-failures`를 운영하는 CI에서는 이 값을 테스트 로그나 reporter 데이터에 넣어 두면 flaky test를 찾기 쉬워집니다.

```js
// test/payment-retry.test.mjs
import { test } from 'node:test';
import assert from 'node:assert/strict';

test('does not charge twice when retry token is reused', async (t) => {
  t.diagnostic(JSON.stringify({
    test: t.fullName,
    attempt: t.attempt,
    workerId: t.workerId
  }));

  const result = {
    status: 'deduplicated',
    chargedCount: 1
  };

  assert.equal(result.status, 'deduplicated');
  assert.equal(result.chargedCount, 1);
});
```

이 로그는 테스트 본문을 성공시키기 위한 우회가 아닙니다.
실패했을 때 "첫 번째 시도에서만 깨졌는지", "모든 attempt에서 반복 실패했는지", "특정 worker와 attempt 조합에서만 흔들렸는지"를 보는 관측 데이터입니다.

### H3. attempt에 따라 테스트 동작을 바꾸지 않는다

`attempt`를 읽을 수 있다고 해서 테스트 로직을 시도 횟수별로 다르게 만들면 안 됩니다.
예를 들어 첫 번째 시도에서는 실제 검증을 하고, 두 번째 시도에서는 느슨한 assertion을 적용하는 방식은 flaky test를 숨깁니다.

피해야 할 패턴은 다음과 같습니다.

```js
test('bad retry-aware assertion', (t) => {
  const responseTimeMs = 3200;

  if (t.attempt > 0) {
    // 재시도에서는 기준을 낮추는 나쁜 예시입니다.
    assert.ok(responseTimeMs < 5000);
    return;
  }

  assert.ok(responseTimeMs < 1000);
});
```

`attempt`는 관측과 리포팅에만 쓰고, 테스트의 기대값은 모든 시도에서 같아야 합니다.
재시도에서만 통과하는 테스트는 통과로 덮기보다 별도 flaky 목록으로 모아 원인을 고치는 편이 장기적으로 안전합니다.

## CI 로그와 reporter에 넣을 필드

### H3. 최소 필드부터 표준화한다

테스트 실패를 추적하려면 모든 로그가 같은 필드 이름을 쓰는 것이 중요합니다.
처음부터 많은 정보를 넣기보다 아래 정도의 최소 필드부터 표준화하세요.

- `test`: `t.fullName`
- `file`: `t.filePath`
- `workerId`: `t.workerId`
- `attempt`: `t.attempt`
- `tags`: 필요한 경우 `t.tags`
- `durationMs`: reporter나 계측 레이어에서 계산한 실행 시간

`filePath`는 절대 경로이므로 외부로 보내는 로그에서는 저장소 기준 상대 경로로 바꾸는 편이 좋습니다.
예를 들어 `/home/runner/work/app/app/test/order.test.mjs` 전체를 남기기보다 `test/order.test.mjs`만 남기면 충분합니다.

### H3. 실패 로그에는 원인 후보를 줄이는 값만 남긴다

테스트 로그가 너무 길면 CI에서 실패 원인을 찾기 어려워집니다.
특히 병렬 테스트에서는 여러 worker의 로그가 섞일 수 있으므로 한 줄 JSON처럼 검색 가능한 형식이 유리합니다.

```js
function diagnosticForTest(t, extra = {}) {
  t.diagnostic(JSON.stringify({
    event: 'test.context',
    test: t.fullName,
    file: t.filePath?.replace(process.cwd(), '<repo>'),
    workerId: t.workerId,
    attempt: t.attempt,
    ...extra
  }));
}
```

이 helper를 만들 때도 주의할 점이 있습니다.
`extra`에 요청 본문, 응답 payload, 사용자 fixture 전체를 그대로 넣을 수 있게 열어 두면 민감정보가 섞일 가능성이 커집니다.
허용 필드를 명확히 정하거나, `reason`, `resourceType`, `fixtureName`처럼 안전한 문자열만 받도록 제한하세요.

## 병렬 CI에서 리소스를 안전하게 나누는 기준

### H3. worker 단위로 격리하고 테스트 단위로 정리한다

`workerId`는 worker 단위 격리에 적합하지만, 모든 정리를 worker 종료에만 맡기면 실패 시 잔여 리소스가 남을 수 있습니다.
좋은 기본값은 worker별 namespace를 만들고, 각 테스트가 만든 리소스는 `afterEach`나 test cleanup 흐름에서 지우는 것입니다.

예를 들어 데이터베이스 schema 이름을 만들 때는 아래처럼 worker와 테스트 이름을 함께 반영할 수 있습니다.

```js
function createSchemaName(t) {
  const name = t.fullName
    .replace(/[^a-z0-9]+/gi, '_')
    .replace(/^_+|_+$/g, '')
    .toLowerCase()
    .slice(0, 48);

  return `test_w${t.workerId ?? 1}_a${t.attempt}_${name}`;
}
```

단, 이 이름을 외부 시스템에 오래 남기면 내부 테스트 구조가 노출될 수 있습니다.
CI 전용 리소스에만 쓰고, cleanup 실패를 감지하는 별도 점검을 두는 것이 좋습니다.

### H3. shard와 workerId를 혼동하지 않는다

CI shard는 전체 테스트 파일을 여러 job으로 나누는 단위이고, `workerId`는 한 Node.js test runner 실행 안에서 테스트 파일을 처리하는 worker 식별자입니다.
둘은 같은 값이 아닙니다.

예를 들어 GitHub Actions matrix가 `shard=2/4`로 실행되고, 그 안에서 Node.js test runner가 여러 worker를 만들 수 있습니다.
이 경우 로그에는 shard 정보와 `workerId`를 따로 남기는 편이 원인 분석에 좋습니다.

```js
t.diagnostic(JSON.stringify({
  shard: process.env.TEST_SHARD ?? 'local',
  workerId: t.workerId,
  attempt: t.attempt,
  test: t.fullName
}));
```

`TEST_SHARD` 같은 환경 변수는 CI 설정에서 주입하고, 테스트 코드에서는 없을 때 `local`처럼 안전한 기본값만 사용하세요.

## 발행 전 점검 체크리스트

### H3. workerId와 attempt 운영 기준

`workerId`와 `attempt`는 테스트를 더 복잡하게 만들기 위한 기능이 아니라, 병렬 실행과 재시도 흐름을 설명하기 위한 메타데이터입니다.
아래 기준을 만족하면 CI 실패 로그를 더 빠르게 읽을 수 있습니다.

- 실패 로그에 `test`, `file`, `workerId`, `attempt`가 남는가?
- 임시 파일, schema, cache key가 worker별로 충돌하지 않는가?
- `attempt`에 따라 assertion 기준이 달라지지 않는가?
- 재시도에서만 통과한 테스트를 따로 볼 수 있는가?
- shard 정보와 worker 정보를 분리해서 기록하는가?
- 절대 경로와 fixture payload를 로그에 그대로 남기지 않는가?

### H3. 민감정보와 유해 표현 점검

테스트 관측 로그는 운영 로그보다 더 느슨하게 관리되기 쉽습니다.
하지만 CI 로그도 저장되고 공유될 수 있으므로 API 키, 토큰, 실제 이메일, 고객 식별자, 내부 절대 경로를 그대로 남기면 안 됩니다.

이 글의 예시는 공개 가능한 테스트 메타데이터와 더미 값만 사용했습니다.
불법 행위, 혐오 표현, 폭력 조장, 개인정보 노출을 포함하지 않으며, 재현 가능한 Node.js 테스트 운영 기준에 초점을 맞췄습니다.

## FAQ

### H3. workerId가 undefined일 수도 있나요?

테스트 컨텍스트 밖에서 값을 읽거나 실행 방식에 따라 기대한 값이 없을 수 있습니다.
helper에서는 `t.workerId ?? 'single'`처럼 기본값을 두고, 테스트 컨텍스트 안에서만 사용하는 것이 안전합니다.

### H3. attempt가 1이면 두 번째 실행인가요?

네.
`attempt`는 0부터 시작하므로 첫 번째 시도는 `0`, 두 번째 시도는 `1`입니다.
재시도 테스트를 분석할 때 이 기준으로 로그를 해석하면 됩니다.

### H3. workerId를 테스트 기대값에 넣어도 되나요?

대부분의 경우 권장하지 않습니다.
`workerId`는 실행 환경을 설명하는 값이지 제품 코드의 계약이 아닙니다.
임시 리소스 분리와 실패 분석 로그에는 쓰되, 비즈니스 로직의 기대값을 worker별로 다르게 만들지는 마세요.

### H3. 모든 테스트에 diagnostic 로그를 넣어야 하나요?

아닙니다.
처음에는 flaky test가 자주 발생하는 영역, 외부 리소스를 쓰는 통합 테스트, 병렬 실행에서 충돌 가능성이 있는 테스트부터 적용하는 편이 좋습니다.
전역 custom reporter나 helper로 공통 필드를 붙일 수 있다면 개별 테스트에 반복 코드를 넣는 것보다 유지보수가 쉽습니다.

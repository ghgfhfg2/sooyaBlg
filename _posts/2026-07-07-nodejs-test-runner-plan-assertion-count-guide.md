---
layout: post
title: "Node.js test runner plan 가이드: 비동기 assertion 누락을 잡는 방법"
date: 2026-07-07 20:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-plan-assertion-count-guide
permalink: /development/blog/seo/2026/07/07/nodejs-test-runner-plan-assertion-count-guide.html
alternates:
  ko: /development/blog/seo/2026/07/07/nodejs-test-runner-plan-assertion-count-guide.html
  x_default: /development/blog/seo/2026/07/07/nodejs-test-runner-plan-assertion-count-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, plan, assertion-count, testing, javascript, async-test, ci]
description: "Node.js test runner의 t.plan()으로 예상 assertion 개수와 subtest 실행 수를 검증하는 방법을 정리합니다. 비동기 callback, event listener, stream 테스트에서 assertion 누락을 잡는 패턴과 timeout 조합을 예제로 설명합니다."
---

테스트가 통과했는데 실제로는 중요한 assertion이 실행되지 않은 적이 있다면 `t.plan()`을 검토할 만합니다.
비동기 callback, event listener, stream, timer를 다루는 테스트에서는 코드 경로가 예상과 다르게 지나가도 테스트 함수 자체는 정상 종료될 수 있습니다.
이때 assertion 개수를 명시하면 "검증이 실행되지 않은 통과"를 실패로 바꿀 수 있습니다.

[Node.js test runner 공식 문서](https://nodejs.org/api/test.html#contextplancount-options)는 `context.plan()`을 사용해 테스트 안에서 실행될 assertion과 subtest의 개수를 지정할 수 있다고 설명합니다.
공식 문서 기준으로 plan이 assertion을 세려면 일반 `assert` 모듈을 직접 쓰는 대신 test context의 `t.assert`를 사용해야 합니다.
즉 `t.plan()`은 단순한 주석이 아니라, 테스트가 실제로 검증을 수행했는지 확인하는 실행 기준입니다.

멈춘 테스트를 시간 제한으로 다루는 방법은 [Node.js test runner timeout 가이드](/development/blog/seo/2026/07/07/nodejs-test-runner-timeout-hanging-test-guide.html)에서 다뤘습니다.
테스트 파일과 child process 격리 기준은 [Node.js test runner isolation 가이드](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html)를 참고하세요.
CI에서 실패한 테스트만 다시 확인하는 흐름은 [Node.js test runner rerun failures 가이드](/development/blog/seo/2026/07/03/nodejs-test-runner-rerun-failures-ci-guide.html)와 함께 설계하면 좋습니다.

## t.plan()이 필요한 이유

### H3. 실행되지 않은 assertion을 통과로 두지 않는다

비동기 테스트에서 가장 위험한 실패는 "실패하지 않은 실패"입니다.
callback이 호출되지 않았는데 테스트 함수가 먼저 끝나면, assertion이 아예 실행되지 않았는데도 테스트가 통과처럼 보일 수 있습니다.
`t.plan()`은 이런 누락을 잡기 위한 간단한 장치입니다.

```js
import test from 'node:test';

test('calls completion callback', (t) => {
  t.plan(1);

  runJob(() => {
    t.assert.equal('done', 'done');
  });
});

function runJob(onComplete) {
  setImmediate(onComplete);
}
```

이 테스트는 `t.assert.equal()`이 정확히 한 번 실행되어야 통과합니다.
만약 `runJob()`의 버그 때문에 callback이 호출되지 않으면 plan에 지정한 assertion 개수를 채우지 못해 실패합니다.
테스트 통과 조건을 "함수가 에러 없이 끝났다"가 아니라 "필요한 검증이 실행됐다"로 바꾸는 셈입니다.

### H3. 비동기 분기마다 검증 수를 맞춘다

event listener나 callback 기반 API는 여러 분기를 가질 수 있습니다.
성공 이벤트와 완료 이벤트를 모두 확인해야 하는데 한쪽만 실행되어도 테스트가 끝나는 구조라면 누락이 생길 수 있습니다.
plan은 이런 테스트에서 기대하는 검증 개수를 명확히 드러냅니다.

```js
import { EventEmitter } from 'node:events';
import test from 'node:test';

test('emits data and close events', async (t) => {
  t.plan(2);

  const stream = new EventEmitter();

  stream.on('data', (chunk) => {
    t.assert.equal(chunk, 'hello');
  });

  stream.on('close', () => {
    t.assert.ok(true, 'stream closed');
  });

  stream.emit('data', 'hello');
  stream.emit('close');
});
```

이 예시는 `data`와 `close`가 모두 검증되어야 통과합니다.
둘 중 하나가 빠지면 assertion 개수가 부족해 실패하므로, 이벤트 순서나 경로 누락을 더 빨리 발견할 수 있습니다.
특히 stream, socket, queue consumer처럼 이벤트 기반 코드가 많은 프로젝트에서 유용합니다.

## t.assert를 함께 써야 하는 이유

### H3. 일반 assert는 plan 카운트에 포함되지 않는다

`t.plan()`은 test context가 추적할 수 있는 assertion을 기준으로 동작합니다.
그래서 `node:assert/strict`의 `assert.equal()`을 직접 호출하면 plan 카운트에 포함되지 않습니다.
plan을 쓰는 테스트에서는 `t.assert`를 쓰는 규칙을 분명히 두는 편이 안전합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('wrong plan example', (t) => {
  t.plan(1);

  assert.equal(1 + 1, 2);
});
```

이 코드는 일반 assertion 자체는 성공하지만, plan 입장에서는 추적된 assertion이 없습니다.
따라서 plan을 채우지 못한 테스트로 실패할 수 있습니다.
팀에서 plan을 도입한다면 "plan이 있는 테스트에서는 `t.assert`를 사용한다"는 규칙을 코드 리뷰 기준에 넣는 것이 좋습니다.

### H3. 기존 assert 스타일과 섞을 때 범위를 좁힌다

이미 많은 테스트가 `assert`를 직접 사용하고 있다면 모든 파일을 한 번에 바꿀 필요는 없습니다.
plan이 꼭 필요한 비동기 테스트부터 `t.assert`로 좁게 전환하는 편이 현실적입니다.
테스트 목적이 동기 계산 결과 확인이라면 기존 assertion만으로도 충분한 경우가 많습니다.

```js
import test from 'node:test';

test('notifies two subscribers', (t) => {
  t.plan(2);

  const subscribers = [
    (message) => t.assert.equal(message.type, 'CREATED'),
    (message) => t.assert.equal(message.id, 'order_123')
  ];

  for (const subscriber of subscribers) {
    subscriber({ type: 'CREATED', id: 'order_123' });
  }
});
```

이 테스트에서는 subscriber가 두 개 모두 호출되는지가 핵심입니다.
따라서 assertion 개수를 추적하는 것이 테스트의 의도와 맞습니다.
반대로 단순한 순수 함수 테스트에 plan을 남발하면 코드가 장황해질 수 있습니다.

## 비동기 테스트에서 plan 사용하기

### H3. callback 테스트에는 wait 옵션을 고려한다

비동기 assertion이 테스트 함수 종료 뒤에 실행될 수 있다면 `plan`의 wait 옵션을 사용할 수 있습니다.
공식 문서에 따르면 wait timeout은 테스트 함수 실행이 끝난 뒤 기대 assertion을 기다리는 최대 시간을 정합니다.
이 값은 무한 대기를 막으면서도 늦게 도착하는 callback을 검증할 여지를 줍니다.

```js
import test from 'node:test';

test('receives async notification', (t) => {
  t.plan(1, { wait: 500 });

  setTimeout(() => {
    t.assert.equal('notified', 'notified');
  }, 50);
});
```

이 테스트는 함수 본문이 먼저 끝나더라도 500ms 안에 assertion이 실행되면 통과할 수 있습니다.
callback이 오지 않으면 plan이 충족되지 않아 실패합니다.
다만 wait 값을 크게 잡으면 실패가 늦어지므로, 테스트의 정상 지연 시간을 기준으로 짧게 설정하는 것이 좋습니다.

### H3. test timeout과 plan wait를 구분한다

`timeout`과 `plan`의 wait는 역할이 다릅니다.
test timeout은 테스트 전체가 너무 오래 걸릴 때 실패시키는 상한입니다.
plan wait는 테스트 함수가 끝난 뒤 예상 assertion이 도착할 시간을 조금 더 기다리는 장치입니다.

```js
import test from 'node:test';

test('processes delayed callback once', {
  timeout: 1000
}, (t) => {
  t.plan(1, { wait: 300 });

  delayedCallback((result) => {
    t.assert.equal(result.status, 'ok');
  });
});

function delayedCallback(callback) {
  setTimeout(() => {
    callback({ status: 'ok' });
  }, 100);
}
```

이 구조에서는 전체 테스트가 1초를 넘기면 실패하고, 테스트 함수 종료 뒤 assertion은 300ms까지만 기다립니다.
둘을 함께 쓰면 멈춘 테스트와 누락된 assertion을 구분해서 볼 수 있습니다.
외부 I/O가 섞인 테스트라면 개별 작업의 `AbortSignal.timeout()`까지 별도로 두는 편이 더 읽기 쉽습니다.

## subtest와 plan 조합하기

### H3. subtest 실행 개수도 기대값에 포함한다

`t.plan()`은 assertion뿐 아니라 subtest 실행 개수도 기대값으로 다룰 수 있습니다.
동적으로 생성하는 subtest가 필터 조건이나 데이터 누락 때문에 예상보다 적게 만들어지는 경우를 잡는 데 유용합니다.
테스트 데이터가 비어도 조용히 통과하는 문제를 막을 수 있습니다.

```js
import test from 'node:test';

test('validates required endpoints', async (t) => {
  const endpoints = [
    { name: 'health', path: '/health' },
    { name: 'ready', path: '/ready' }
  ];

  t.plan(endpoints.length);

  for (const endpoint of endpoints) {
    await t.test(endpoint.name, (t) => {
      t.assert.ok(endpoint.path.startsWith('/'));
    });
  }
});
```

이 예시는 endpoint 개수만큼 subtest가 실행되어야 합니다.
데이터 로딩 버그로 `endpoints`가 비거나 일부가 빠지면 기대한 실행 수와 달라집니다.
동적 테스트를 만들 때는 "몇 개가 만들어져야 정상인가"를 plan으로 표현할 수 있습니다.

### H3. 부모 테스트가 먼저 끝나지 않게 await한다

subtest를 만들 때는 `await t.test()`를 빠뜨리지 않는 것이 중요합니다.
부모 테스트가 먼저 끝나면 아직 완료되지 않은 subtest가 취소되거나 실패로 처리될 수 있습니다.
plan을 쓰더라도 subtest 생명주기 자체를 올바르게 기다리는 습관이 먼저입니다.

```js
import test from 'node:test';

test('validates user roles', async (t) => {
  const roles = ['admin', 'editor', 'viewer'];

  t.plan(roles.length);

  for (const role of roles) {
    await t.test(`role ${role}`, (t) => {
      t.assert.match(role, /^[a-z]+$/);
    });
  }
});
```

이 테스트는 각 role마다 subtest를 하나씩 실행합니다.
`await` 덕분에 부모 테스트는 자식 테스트가 끝날 때까지 기다립니다.
동적 subtest와 plan을 함께 쓰면 테스트 구조와 기대 실행 수가 코드에서 함께 드러납니다.

## 실무 적용 기준

### H3. 모든 테스트에 plan을 넣을 필요는 없다

plan은 강력하지만 모든 테스트에 필요한 것은 아닙니다.
동기 함수의 반환값을 바로 검증하는 테스트에 plan을 반복해서 넣으면 읽기 비용만 늘어날 수 있습니다.
다음처럼 assertion이 누락될 위험이 높은 테스트에 우선 적용하는 편이 좋습니다.

- callback이나 event listener 안에서 assertion을 실행하는 테스트
- timer, stream, socket, queue consumer처럼 나중에 실행되는 코드가 있는 테스트
- 동적으로 subtest를 생성하는 테스트
- 실패해야 하는 경로와 성공해야 하는 경로를 모두 검증해야 하는 테스트
- CI에서 가끔 통과하지만 실제 검증이 부족했던 이력이 있는 테스트

이 기준은 [Node.js test runner watch mode 가이드](/development/blog/seo/2026/07/06/nodejs-test-runner-watch-mode-fast-feedback-guide.html)의 반복 실행 안정성과도 연결됩니다.
watch mode에서 테스트가 여러 번 통과하더라도 assertion이 실행되지 않았다면 신뢰할 수 없습니다.
plan은 로컬 피드백 루프의 품질을 높이는 보조 장치가 됩니다.

### H3. 실패 메시지가 설명하는 테스트 이름을 붙인다

plan 실패는 보통 "예상 개수와 실제 개수가 다르다"는 형태로 나타납니다.
이때 테스트 이름이 추상적이면 어떤 callback이나 이벤트가 빠졌는지 바로 알기 어렵습니다.
테스트 이름에는 대상, 조건, 기대 실행을 포함하는 편이 좋습니다.

```js
test('webhook handler calls success callback exactly once', (t) => {
  t.plan(1, { wait: 300 });

  handleWebhook({ type: 'payment.succeeded' }, (event) => {
    t.assert.equal(event.type, 'payment.succeeded');
  });
});

function handleWebhook(event, onSuccess) {
  setImmediate(() => onSuccess(event));
}
```

이름만 봐도 webhook handler가 success callback을 한 번 호출해야 한다는 기대가 드러납니다.
plan 실패가 발생하면 callback 호출 경로부터 확인하면 됩니다.
테스트 이름과 plan 개수가 함께 있으면 CI 로그만으로도 디버깅 방향을 잡기 쉽습니다.

## 발행 전 실무 체크리스트

### H3. plan을 넣은 테스트에서 확인할 항목

`t.plan()`은 테스트를 더 엄격하게 만들지만, 잘못 쓰면 오히려 의도를 흐릴 수 있습니다.
아래 항목을 기준으로 도입 범위를 점검하세요.

- plan이 있는 테스트에서 `t.assert`를 사용하고 있는가?
- assertion 개수가 테스트 의도를 설명하는가, 단순한 숫자 맞추기가 아닌가?
- callback이나 event listener가 호출되지 않는 경우 실패하는가?
- `plan`의 wait 값과 test timeout이 서로 다른 역할로 설정됐는가?
- 동적 subtest를 만들 때 `await t.test()`를 빠뜨리지 않았는가?
- 너무 단순한 동기 테스트에 plan을 남발하고 있지 않은가?

이 체크리스트는 코드 리뷰에서도 그대로 사용할 수 있습니다.
plan을 도입한 이유가 "assertion 누락 방지"인지, "테스트를 복잡하게 보이게 만드는 장식"인지 구분하는 것이 중요합니다.
숫자 자체보다 테스트가 어떤 검증을 반드시 실행해야 하는지가 핵심입니다.

## FAQ

### H3. t.plan()은 몇 개의 assertion을 세나요?

지정한 개수만큼 test context가 추적하는 assertion과 subtest가 실행되어야 합니다.
일반 `assert` 직접 호출은 plan 카운트에 들어가지 않으므로 `t.assert`를 사용하세요.
동적 subtest를 만들 때도 기대 실행 수를 plan으로 표현할 수 있습니다.

### H3. plan을 쓰면 timeout이 필요 없나요?

아닙니다.
plan은 assertion 개수 누락을 잡고, timeout은 테스트가 너무 오래 걸리는 상황을 잡습니다.
비동기 callback을 기다리는 테스트라면 `t.plan(count, { wait })`와 test `timeout`을 함께 쓰는 편이 안전합니다.

### H3. 모든 테스트에 t.plan()을 적용해야 하나요?

모든 테스트에 일괄 적용할 필요는 없습니다.
callback, event, stream, timer, 동적 subtest처럼 assertion이 늦게 실행되거나 누락될 수 있는 테스트에 우선 적용하세요.
단순한 동기 테스트는 명확한 assertion만으로 충분한 경우가 많습니다.

## 마무리

Node.js test runner의 `t.plan()`은 테스트가 정말 필요한 검증을 실행했는지 확인하는 장치입니다.
비동기 callback이 호출되지 않았거나 동적 subtest가 예상보다 적게 만들어졌을 때, 조용한 통과를 실패로 바꿔 줍니다.
특히 event listener, stream, timer, queue consumer 테스트에서는 작은 plan 하나가 CI 신뢰도를 크게 높일 수 있습니다.

다만 plan은 모든 테스트에 붙이는 규칙이 아니라, assertion 누락 위험이 있는 곳에 쓰는 도구입니다.
`t.assert`, wait 옵션, test timeout, 명확한 테스트 이름을 함께 사용하면 실패 로그가 훨씬 설명적입니다.
테스트가 "끝났다"가 아니라 "검증했다"는 사실을 보장하고 싶을 때 `t.plan()`을 선택하세요.

## 관련 글

- [Node.js test runner timeout 가이드: 멈춘 테스트를 빠르게 실패시키는 법](/development/blog/seo/2026/07/07/nodejs-test-runner-timeout-hanging-test-guide.html)
- [Node.js test runner isolation 가이드: 테스트 파일을 안전하게 격리 실행하는 방법](/development/blog/seo/2026/07/06/nodejs-test-runner-isolation-child-process-guide.html)
- [Node.js test runner rerun failures 가이드: 실패한 테스트만 다시 실행하는 CI 전략](/development/blog/seo/2026/07/03/nodejs-test-runner-rerun-failures-ci-guide.html)

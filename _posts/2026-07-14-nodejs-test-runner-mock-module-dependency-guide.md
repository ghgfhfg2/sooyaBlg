---
layout: post
title: "Node.js test runner mock.module 가이드: 의존 모듈을 안전하게 대체하는 법"
date: 2026-07-14 08:00:00 +0900
lang: ko
translation_key: nodejs-test-runner-mock-module-dependency-guide
permalink: /development/blog/seo/2026/07/14/nodejs-test-runner-mock-module-dependency-guide.html
alternates:
  ko: /development/blog/seo/2026/07/14/nodejs-test-runner-mock-module-dependency-guide.html
  x_default: /development/blog/seo/2026/07/14/nodejs-test-runner-mock-module-dependency-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, test-runner, mock-module, module-mock, testing, javascript, esm, commonjs]
description: "Node.js test runner의 mock.module()로 ESM, CommonJS, builtin 모듈 의존성을 테스트에서 안전하게 대체하는 방법을 정리합니다. --experimental-test-module-mocks, exports 옵션, cache, restore 기준을 예제로 설명합니다."
---

단위 테스트에서 가장 다루기 까다로운 의존성은 함수 인자가 아니라 모듈 import로 고정된 의존성입니다.
예를 들어 서비스 코드가 `node:fs/promises`, 외부 SDK, 설정 모듈을 직접 import한다면 테스트에서 그 의존성을 바꾸기 어렵습니다.
이때 무리하게 실제 파일 시스템이나 외부 API를 호출하면 테스트는 느려지고, 실패 원인도 흐려집니다.

Node.js 내장 test runner의 `mock.module()`은 테스트 안에서 특정 모듈의 export를 대체할 수 있는 API입니다.
공식 문서 기준으로 ESM, CommonJS, JSON, Node.js builtin 모듈 mock을 지원하지만, 아직 early development 상태이며 `--experimental-test-module-mocks` 플래그가 필요합니다.
따라서 모든 프로젝트의 기본 선택지라기보다, 의존성 주입으로 풀기 어려운 경계에서 신중하게 쓰는 도구로 보는 편이 안전합니다.

이 글에서는 `mock.module()`을 언제 쓰면 좋은지, `exports` 옵션으로 named export와 default export를 대체하는 방법, `cache`와 `restore()`를 다룰 때의 주의점, CI에서 실수하기 쉬운 부분을 정리합니다.
함수 단위 mock은 [Node.js test runner mock.fn call verification 가이드](/development/blog/seo/2026/07/09/nodejs-test-runner-mock-fn-call-verification-guide.html), 메서드 복구는 [Node.js test runner mock.method restore 가이드](/development/blog/seo/2026/07/09/nodejs-test-runner-mock-method-restore-guide.html), property 의존성은 [Node.js test runner mock.property 가이드](/development/blog/seo/2026/07/13/nodejs-test-runner-mock-property-access-guide.html)와 함께 보면 좋습니다.

## mock.module이 필요한 상황

### H3. import로 고정된 의존성을 바꿔야 할 때

가장 먼저 확인할 것은 정말 모듈 mock이 필요한지입니다.
함수 인자로 의존성을 주입할 수 있다면 그 방식이 더 단순합니다.
하지만 이미 많은 코드가 특정 모듈을 직접 import하고 있고, 테스트에서 그 모듈의 결과만 통제하고 싶다면 `mock.module()`이 도움이 됩니다.

예를 들어 아래 함수는 파일 시스템 모듈을 직접 import합니다.

```js
// profile-store.js
import { readFile } from 'node:fs/promises';

export async function loadProfileName(path) {
  const text = await readFile(path, 'utf8');
  const profile = JSON.parse(text);
  return profile.name;
}
```

실제 파일을 만들고 지우는 테스트도 가능하지만, 이 함수가 JSON 처리와 호출 흐름만 검증하면 되는 단위 테스트라면 `node:fs/promises`를 대체할 수 있습니다.
다만 mock은 import 전에 등록되어야 합니다.
이미 원본 모듈을 import한 참조에는 나중에 만든 mock이 영향을 주지 않습니다.

```js
// profile-store.test.js
import test from 'node:test';
import assert from 'node:assert/strict';

test('loads profile name from mocked file content', async (t) => {
  t.mock.module('node:fs/promises', {
    exports: {
      readFile: async () => JSON.stringify({ name: 'Soo' })
    }
  });

  const { loadProfileName } = await import('./profile-store.js');

  assert.equal(await loadProfileName('/tmp/profile.json'), 'Soo');
});
```

이 테스트를 실행하려면 플래그를 붙입니다.

```bash
node --test --experimental-test-module-mocks
```

테스트 코드만 보고는 플래그 누락을 알아차리기 어렵습니다.
그래서 `package.json` script에 명령을 고정해 두는 편이 좋습니다.

### H3. 실제 외부 효과를 피하고 계약만 확인한다

모듈 mock의 목적은 외부 효과를 숨기는 것이 아니라 테스트 범위를 분명히 하는 것입니다.
메일 발송, 결제 요청, 파일 삭제, 원격 API 호출처럼 테스트에서 실제로 실행하면 위험하거나 느린 작업은 mock으로 대체할 수 있습니다.

```js
// notifier.js
import { sendMail } from './mail-client.js';

export async function sendWelcomeMail(user) {
  if (!user.email) {
    return { sent: false, reason: 'missing-email' };
  }

  await sendMail({
    to: user.email,
    subject: 'Welcome',
    template: 'welcome'
  });

  return { sent: true };
}
```

테스트에서는 `mail-client.js`의 export를 바꾸고, 호출 여부는 mock function으로 확인합니다.

```js
import test from 'node:test';
import assert from 'node:assert/strict';

test('sends welcome mail through the mail client', async (t) => {
  const sendMail = t.mock.fn(async () => ({ id: 'test-message' }));

  t.mock.module('./mail-client.js', {
    exports: { sendMail }
  });

  const { sendWelcomeMail } = await import('./notifier.js');

  assert.deepEqual(await sendWelcomeMail({ email: 'user@example.test' }), {
    sent: true
  });
  assert.equal(sendMail.mock.callCount(), 1);
  assert.deepEqual(sendMail.mock.calls[0].arguments[0], {
    to: 'user@example.test',
    subject: 'Welcome',
    template: 'welcome'
  });
});
```

여기서 이메일 주소는 문서용 테스트 도메인을 사용했습니다.
실제 사용자 이메일, 토큰, 내부 URL을 assertion 값이나 실패 메시지에 넣지 않는 기준은 CI 로그에서도 중요합니다.

## exports 옵션으로 모듈 모양 맞추기

### H3. named export는 같은 이름으로 제공한다

`mock.module()`의 `exports` 객체에서 `default`를 제외한 enumerable property는 named export로 노출됩니다.
테스트 대상이 `import { getFeatureFlag } from './config.js'`처럼 named import를 사용한다면 mock도 같은 이름을 제공해야 합니다.

```js
t.mock.module('./config.js', {
  exports: {
    getFeatureFlag: () => true,
    region: 'ap-northeast-2'
  }
});
```

모듈 mock은 원본 모듈의 일부 export만 살짝 덮어쓰는 방식이 아닙니다.
mock으로 제공한 export가 테스트에서 보이는 모듈의 모양이 됩니다.
따라서 테스트 대상이 다른 export도 함께 import한다면 mock 객체에 빠짐없이 넣어야 합니다.

이 특성은 장점이기도 합니다.
테스트가 정말 필요한 의존성만 명시하게 만들고, 의존 모듈과의 결합을 눈에 보이게 합니다.
반대로 mock 객체가 너무 커진다면 테스트 대상이 너무 많은 모듈 세부 사항에 묶여 있다는 신호일 수 있습니다.

### H3. default export는 exports.default로 제공한다

default import를 쓰는 모듈은 `exports.default`에 값을 넣습니다.
공식 문서에는 `defaultExport`, `namedExports` 옵션도 있지만 더 이상 새 코드의 우선 선택지로 보기 어렵습니다.
새 테스트에서는 `exports` 하나로 default와 named export를 함께 표현하는 편이 읽기 쉽습니다.

```js
t.mock.module('./payment-client.js', {
  exports: {
    default: {
      charge: async () => ({ approved: true })
    },
    createIdempotencyKey: () => 'test-key'
  }
});
```

CommonJS나 builtin 모듈을 mock할 때는 default export가 `module.exports` 역할을 할 수 있다는 점도 기억해야 합니다.
ESM과 CommonJS가 섞인 코드베이스에서는 한 테스트가 어떤 import 방식을 검증하는지 명확히 적어 두는 편이 좋습니다.

## cache와 restore 기준

### H3. 기본 cache false는 매 import마다 새 mock을 만든다

`mock.module()`의 `cache` 기본값은 `false`입니다.
이 설정에서는 `require()`나 `import()`를 호출할 때마다 새 mock module이 생성될 수 있습니다.
대부분의 단위 테스트에서는 이 기본값이 테스트 간 오염을 줄이는 데 유리합니다.

공유 상태가 있는 mock을 일부러 유지해야 한다면 `cache: true`를 사용할 수 있습니다.
하지만 이때는 같은 mock이 여러 import에서 재사용되므로 테스트 순서와 병렬 실행에 더 민감해집니다.

```js
test('uses a cached module mock when shared state is intentional', async (t) => {
  const calls = [];

  t.mock.module('./audit-client.js', {
    cache: true,
    exports: {
      record: (event) => {
        calls.push(event.type);
      }
    }
  });

  const auditA = await import('./audit-client.js');
  const auditB = await import('./audit-client.js');

  auditA.record({ type: 'created' });
  auditB.record({ type: 'confirmed' });

  assert.deepEqual(calls, ['created', 'confirmed']);
});
```

공유 상태를 테스트하려는 의도가 없다면 `cache: true`는 피하는 편이 낫습니다.
특히 `--test-concurrency`를 쓰는 프로젝트에서는 mock 내부 배열이나 객체가 다른 테스트와 섞이지 않는지 확인해야 합니다.

### H3. restore는 원본 모듈로 돌아가는 경계다

`mock.module()`은 `MockModuleContext`를 반환하고, 이 객체의 `restore()`로 mock을 해제할 수 있습니다.
테스트 컨텍스트의 `t.mock`을 쓰면 테스트 종료 시 정리되지만, 한 테스트 안에서 mock 적용 전후를 모두 확인해야 한다면 직접 `restore()`를 호출할 수 있습니다.

```js
test('restores a builtin module mock', async (t) => {
  const mocked = t.mock.module('node:readline', {
    exports: {
      createInterface: () => ({ close() {} })
    }
  });

  let readline = await import('node:readline');
  assert.equal(typeof readline.createInterface, 'function');
  assert.equal(readline.cursorTo, undefined);

  mocked.restore();

  readline = await import('node:readline');
  assert.equal(typeof readline.cursorTo, 'function');
});
```

restore 이후에 다시 import하면 원본 모듈을 확인할 수 있습니다.
다만 이미 mock 상태에서 가져온 참조가 자동으로 원본처럼 바뀌는 것은 아닙니다.
테스트 코드에서는 mock 전후의 import 시점을 분명히 나누는 것이 좋습니다.

## CI 운영 체크리스트

### H3. 실험 플래그를 package script에 고정한다

`mock.module()` 테스트는 플래그가 없으면 실행 환경에서 실패합니다.
개발자마다 명령을 다르게 치지 않도록 script에 고정하세요.

```json
{
  "scripts": {
    "test": "node --test",
    "test:module-mock": "node --test --experimental-test-module-mocks"
  }
}
```

모든 테스트가 모듈 mock을 쓰지 않는다면 별도 script로 분리하는 방식도 괜찮습니다.
프로젝트 정책상 실험 API 사용을 제한한다면 해당 테스트 파일을 명확히 묶고, Node.js 버전 조건을 문서화합니다.

### H3. import 순서를 테스트 리뷰 기준에 넣는다

모듈 mock에서 가장 흔한 실수는 테스트 대상 모듈을 먼저 import한 뒤 mock을 등록하는 것입니다.
원본 참조가 이미 만들어진 뒤에는 나중에 만든 mock이 그 참조를 바꾸지 않습니다.

리뷰에서는 다음 순서가 지켜졌는지 확인하면 됩니다.

```text
1. t.mock.module()로 의존 모듈을 먼저 등록한다.
2. 테스트 대상 모듈을 동적 import로 불러온다.
3. 결과와 mock function 호출 기록을 검증한다.
4. 필요한 경우 restore() 후 원본 동작을 확인한다.
```

정적 import를 파일 상단에 두면 mock보다 먼저 실행됩니다.
따라서 모듈 mock 테스트에서는 테스트 대상 import를 테스트 함수 안의 `await import()`로 옮기는 패턴이 안전합니다.

### H3. 민감한 설정 모듈은 값 자체를 기록하지 않는다

설정 모듈을 mock할 때 실제 환경 변수 이름, 토큰 형식, 내부 서비스 URL을 그대로 예제와 assertion에 넣기 쉽습니다.
테스트에는 의미 있는 더미 값만 남기고, 실패 메시지는 동작 의도를 설명하는 수준으로 제한합니다.

```js
assert.equal(client.mock.callCount(), 1, 'client should be called once');
```

실패 로그에 필요한 것은 민감한 원문 값이 아니라 어떤 계약이 깨졌는지입니다.
이 기준을 지키면 테스트가 공개 CI나 협업 환경에서도 안전해집니다.

## 정리

`mock.module()`은 import로 고정된 의존성을 테스트 안에서 대체할 수 있는 강력한 도구입니다.
외부 API, 파일 시스템, 설정 모듈처럼 실제 효과를 피하고 싶은 경계에서 유용하지만, early development API이고 `--experimental-test-module-mocks` 플래그가 필요하다는 점을 운영 기준에 포함해야 합니다.

실무에서는 먼저 의존성 주입이나 작은 함수 mock으로 해결할 수 있는지 확인하세요.
그래도 모듈 경계를 바꿔야 한다면 `t.mock.module()`을 import 전에 등록하고, `exports`로 모듈 모양을 명확히 맞추고, `cache`는 필요한 경우에만 켜는 편이 안전합니다.
테스트가 끝난 뒤의 복구, Node.js 버전, CI 플래그, 민감정보 노출 여부까지 함께 점검하면 모듈 mock을 더 예측 가능하게 운영할 수 있습니다.

## FAQ

### H3. mock.module은 모든 Node.js 프로젝트에서 바로 써도 되나요?

아직 early development API이므로 보수적으로 도입하는 편이 좋습니다.
Node.js 버전과 실행 플래그를 맞출 수 있고, 모듈 mock이 꼭 필요한 테스트에 한해 사용하는 기준을 세우세요.

### H3. 정적 import와 함께 mock.module을 쓸 수 있나요?

테스트 대상 모듈을 파일 상단에서 정적으로 import하면 mock 등록보다 먼저 평가될 수 있습니다.
모듈 mock 테스트에서는 `t.mock.module()`을 먼저 호출한 뒤 `await import()`로 테스트 대상을 불러오는 패턴이 안전합니다.

### H3. mock.method와 mock.module 중 무엇을 먼저 선택해야 하나요?

객체 인스턴스나 이미 주입된 의존성을 바꾸는 경우에는 `mock.method()`가 더 단순합니다.
모듈 import 자체를 바꿔야 하는 경우에만 `mock.module()`을 고려하세요.

---
layout: post
title: "Node.js assert.match 가이드: 문자열 테스트를 정규식으로 덜 깨지게 검증하는 법"
date: 2026-06-23 20:00:00 +0900
lang: ko
translation_key: nodejs-assert-match-string-pattern-test-guide
permalink: /development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html
alternates:
  ko: /development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html
  x_default: /development/blog/seo/2026/06/23/nodejs-assert-match-string-pattern-test-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, assert, match, regex, test, string, backend]
description: "Node.js assert.match()와 assert.doesNotMatch()로 에러 메시지, 로그, CLI 출력처럼 변동이 있는 문자열을 안전하게 검증하는 방법을 정리합니다. exact 비교와 snapshot의 차이, 정규식 설계 기준, 테스트 예제를 함께 설명합니다."
---

테스트에서 문자열을 검증하는 일은 생각보다 까다롭습니다.
에러 메시지, 로그 라인, CLI 출력, validation 메시지는 사용자와 개발자에게 중요한 신호지만, 전체 문장을 그대로 고정하면 작은 문구 수정에도 테스트가 자주 깨집니다.
반대로 문자열 검증을 아예 빼면 실제로 필요한 정보가 빠졌는데도 테스트가 통과할 수 있습니다.

이때 Node.js `node:assert/strict`의 `assert.match()`를 쓰면 문자열 안에 꼭 포함되어야 하는 패턴만 검증할 수 있습니다.
[Node.js assert 공식 문서](https://nodejs.org/api/assert.html)에 따르면 `assert.match(string, regexp)`는 문자열이 정규식과 일치하는지 확인하고, 반대 검증에는 `assert.doesNotMatch()`를 사용할 수 있습니다.
이 글에서는 `assert.match()`를 에러 메시지, 로그, CLI 출력 테스트에 적용하는 방법과 정규식을 너무 느슨하거나 너무 빡빡하게 만들지 않는 기준을 정리합니다.

비동기 에러 검증은 [Node.js assert.rejects 가이드](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)를 함께 보면 좋습니다.
객체 응답의 일부 필드만 고정하고 싶다면 [Node.js assert.partialDeepStrictEqual 가이드](/development/blog/seo/2026/06/23/nodejs-assert-partialdeepstrictequal-api-response-test-guide.html)로 연결됩니다.
스택 트레이스 문구를 테스트할 때는 [Node.js Error.captureStackTrace 가이드](/development/blog/seo/2026/06/22/nodejs-error-capturestacktrace-custom-error-guide.html)도 참고할 수 있습니다.

## assert.match가 필요한 상황

### H3. 전체 문자열 비교는 문구 변경에 약하다

문자열 전체가 공개 계약인 경우에는 `assert.equal()`이 맞습니다.
예를 들어 CLI의 JSON 출력, 프로토콜 메시지, 문서화된 API 에러 문구처럼 한 글자 차이도 호환성 문제라면 exact 비교가 필요합니다.

하지만 많은 테스트에서 실제 의도는 전체 문장을 고정하는 것이 아닙니다.
보통은 "에러 메시지에 필드명이 들어가는가", "로그에 요청 ID가 포함되는가", "CLI 출력에 실패 원인이 보이는가"를 확인하고 싶을 뿐입니다.

```js
import assert from 'node:assert/strict';

const message = 'Invalid orderId: expected non-empty string';

assert.equal(
  message,
  'Invalid orderId: expected non-empty string'
);
```

위 테스트는 현재 문구에는 정확합니다.
하지만 나중에 메시지가 `Invalid orderId: must be a non-empty string`으로 조금 다듬어지면, 핵심 정보가 유지됐는데도 실패합니다.
테스트가 지켜야 하는 계약이 "orderId가 문제라는 사실"이라면 전체 문자열 비교는 의도보다 많은 것을 고정합니다.

### H3. 패턴 검증은 핵심 신호만 고정한다

`assert.match()`는 문자열이 정규식과 일치하는지만 확인합니다.
그래서 바뀌어도 되는 문구는 열어 두고, 반드시 남아야 하는 단어와 구조만 테스트 계약으로 둘 수 있습니다.

```js
import assert from 'node:assert/strict';

const message = 'Invalid orderId: expected non-empty string';

assert.match(message, /orderId/);
assert.match(message, /non-empty string/);
```

이 테스트는 메시지 앞뒤 표현이 조금 바뀌어도 통과할 수 있습니다.
하지만 `orderId`나 `non-empty string`이라는 핵심 힌트가 사라지면 실패합니다.
문자열 테스트가 구현 세부사항보다 사용자에게 필요한 신호를 중심으로 움직이게 됩니다.

## 기본 사용법

### H3. 문자열과 정규식을 함께 넘긴다

기본 형태는 단순합니다.
첫 번째 인자는 실제 문자열이고, 두 번째 인자는 `RegExp`입니다.

```js
import assert from 'node:assert/strict';

const output = 'Build completed in 842ms';

assert.match(output, /Build completed/);
assert.match(output, /\d+ms/);
```

테스트 의도를 더 선명하게 남기고 싶다면 세 번째 인자로 실패 메시지를 줄 수 있습니다.
다만 실패 메시지는 테스트 코드만 읽어도 의도가 애매할 때만 추가하는 편이 좋습니다.
정규식 이름과 테스트 이름이 충분히 설명적이면 별도 메시지가 없어도 됩니다.

```js
assert.match(
  output,
  /\d+ms/,
  'build output should include duration'
);
```

### H3. 포함되면 안 되는 패턴은 doesNotMatch로 검증한다

문자열 테스트에서는 "무엇이 있어야 하는가"만큼 "무엇이 없어야 하는가"도 중요합니다.
예를 들어 로그와 CLI 출력에는 API 키, 토큰, 비밀번호 같은 민감정보가 그대로 남으면 안 됩니다.

```js
import assert from 'node:assert/strict';

const logLine = 'user login failed userId=user_123 token=[redacted]';

assert.match(logLine, /login failed/);
assert.doesNotMatch(logLine, /sk_live_[a-z0-9]+/i);
assert.doesNotMatch(logLine, /password=/i);
```

이런 테스트는 보안 정책을 코드로 남기는 효과가 있습니다.
로그 redaction 로직을 바꿀 때도 민감정보가 다시 노출되는 회귀를 빠르게 잡을 수 있습니다.
로그 예제를 다룰 때는 실제 토큰처럼 보이는 문자열을 넣지 말고, 테스트용 더미 값이나 `[redacted]`를 사용하세요.

## 에러 메시지 테스트에 적용하기

### H3. 에러 타입과 핵심 문구를 나눠 검증한다

에러 테스트에서 가장 흔한 실수는 메시지 문자열 하나에 모든 의미를 몰아넣는 것입니다.
에러의 종류는 `instanceof`, `name`, `code` 같은 구조적 값으로 검증하고, 메시지는 사람이 읽어야 하는 힌트 중심으로 검증하는 편이 더 안정적입니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

class ValidationError extends Error {
  constructor(message, options = {}) {
    super(message);
    this.name = 'ValidationError';
    this.code = options.code ?? 'VALIDATION_ERROR';
  }
}

function parseOrder(input) {
  if (!input.orderId) {
    throw new ValidationError('orderId is required', {
      code: 'ORDER_ID_REQUIRED'
    });
  }

  return { orderId: input.orderId };
}

test('parseOrder reports missing orderId', () => {
  assert.throws(
    () => parseOrder({}),
    (error) => {
      assert.ok(error instanceof ValidationError);
      assert.equal(error.code, 'ORDER_ID_REQUIRED');
      assert.match(error.message, /orderId/);
      assert.match(error.message, /required/);
      return true;
    }
  );
});
```

이 테스트는 에러 계약을 세 층으로 나눕니다.
첫째, 에러 클래스가 맞아야 합니다.
둘째, 프로그램이 분기할 수 있는 `code`가 맞아야 합니다.
셋째, 사람이 읽는 메시지에는 `orderId`와 `required`라는 핵심 힌트가 있어야 합니다.

### H3. 비동기 에러는 assert.rejects 안에서 함께 쓴다

Promise 기반 함수도 같은 방식으로 검증할 수 있습니다.
`assert.rejects()`의 검증 함수 안에서 `assert.match()`를 호출하면 비동기 에러 메시지를 세밀하게 확인할 수 있습니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

async function loadUser(userId) {
  if (!userId.startsWith('user_')) {
    throw new TypeError(`Invalid user id: ${userId}`);
  }

  return { id: userId };
}

test('loadUser rejects invalid user id', async () => {
  await assert.rejects(
    () => loadUser('abc'),
    (error) => {
      assert.ok(error instanceof TypeError);
      assert.match(error.message, /Invalid user id/);
      assert.match(error.message, /abc/);
      return true;
    }
  );
});
```

여기서 `abc` 같은 입력값을 메시지에 포함하는 것이 실제 계약인지도 함께 판단해야 합니다.
사용자 입력이 길거나 민감할 수 있다면 메시지에 원문을 그대로 넣지 않는 편이 안전합니다.
그런 경우 테스트도 원문 포함이 아니라 마스킹 또는 필드명 포함 여부를 검증해야 합니다.

## 로그 테스트에 적용하기

### H3. 로그는 구조와 redaction을 함께 본다

운영 로그는 테스트하기 어렵다고 느끼기 쉽지만, 최소한의 규칙은 자동화해 둘 수 있습니다.
특히 장애 대응에 필요한 키와 민감정보 제거 정책은 테스트 가치가 큽니다.

```js
import assert from 'node:assert/strict';
import { test } from 'node:test';

function formatAuthLog({ userId, requestId, token }) {
  return [
    'auth failed',
    `userId=${userId}`,
    `requestId=${requestId}`,
    'token=[redacted]'
  ].join(' ');
}

test('auth log includes diagnostic fields without leaking token', () => {
  const line = formatAuthLog({
    userId: 'user_123',
    requestId: 'req_456',
    token: 'test-token-value'
  });

  assert.match(line, /auth failed/);
  assert.match(line, /userId=user_123/);
  assert.match(line, /requestId=req_456/);
  assert.match(line, /token=\[redacted\]/);
  assert.doesNotMatch(line, /test-token-value/);
});
```

이 테스트는 로그 문장 전체를 고정하지 않습니다.
하지만 장애 분석에 필요한 `userId`, `requestId`, 실패 이벤트명, redaction 결과는 고정합니다.
로그 포맷을 JSON으로 바꾸는 경우에는 문자열 정규식보다 JSON 파싱 후 객체 검증을 우선하세요.
문자열 검증은 실제 출력이 문자열일 때 가장 적절합니다.

### H3. 정규식은 과하게 일반화하지 않는다

`/.*/`에 가까운 정규식은 테스트로서 가치가 낮습니다.
반대로 전체 문장을 그대로 정규식으로 옮기면 `assert.equal()`과 다르지 않습니다.
좋은 정규식은 바뀌면 안 되는 단어, 숫자 형식, 키 이름, 구분자를 중심으로 좁게 잡습니다.

```js
const line = 'payment failed orderId=ord_123 retryable=false';

assert.match(line, /payment failed/);
assert.match(line, /orderId=ord_\d+/);
assert.match(line, /retryable=false/);
```

위 예시는 자연어 문장 전체가 아니라 이벤트명, ID 형식, 재시도 가능 여부를 검증합니다.
이 정도면 로그 메시지를 조금 다듬어도 테스트가 유지되고, 중요한 필드가 빠지면 실패합니다.

## CLI 출력 테스트에 적용하기

### H3. 사람용 출력은 핵심 라인만 검증한다

CLI는 사람이 읽는 출력과 기계가 읽는 출력을 구분해야 합니다.
`--json` 같은 옵션으로 제공되는 기계용 출력은 JSON 파싱 후 구조를 검증하는 편이 낫습니다.
반면 기본 help, 경고, 성공 메시지처럼 사람용 출력은 `assert.match()`로 핵심 라인을 확인할 수 있습니다.

```js
import assert from 'node:assert/strict';

const help = `
Usage: deploy [options]

Options:
  --env <name>      target environment
  --dry-run         print actions without applying changes
`;

assert.match(help, /Usage: deploy/);
assert.match(help, /--env <name>/);
assert.match(help, /--dry-run/);
```

이 테스트는 help 문구의 들여쓰기나 설명 문장 전체를 고정하지 않습니다.
하지만 사용자에게 필요한 명령 이름과 핵심 옵션이 빠지면 실패합니다.

### H3. 동적인 값은 형식만 검증한다

시간, 실행 시간, 파일 경로, 임시 디렉터리, 랜덤 ID는 실행할 때마다 달라질 수 있습니다.
이 값을 그대로 고정하면 테스트가 flaky해지기 쉽습니다.
정규식으로 형식만 검증하는 편이 더 안정적입니다.

```js
const output = 'Report written to /tmp/report-1842.json in 37ms';

assert.match(output, /Report written to/);
assert.match(output, /report-\d+\.json/);
assert.match(output, /in \d+ms/);
```

다만 파일 경로 자체가 계약이라면 다른 접근이 필요합니다.
예를 들어 특정 디렉터리에 반드시 저장해야 하는 기능이라면 경로의 상위 디렉터리까지 검증해야 합니다.
테스트가 지켜야 하는 계약이 무엇인지 먼저 정하고 정규식의 범위를 맞추는 것이 중요합니다.

## 정규식 설계 체크리스트

### H3. 먼저 계약이 되는 단어를 고른다

정규식부터 쓰기보다 테스트가 보호해야 하는 계약을 먼저 적어 보는 편이 좋습니다.
문자열 테스트에서 자주 계약이 되는 요소는 다음과 같습니다.

- 사용자가 조치할 수 있는 필드명
- 프로그램이 분기할 수 있는 에러 코드
- 장애 분석에 필요한 requestId, jobId, orderId
- 민감정보가 마스킹됐다는 표시
- CLI 옵션 이름과 명령 이름
- 시간, 크기, 개수 같은 값의 형식

이 목록이 정해지면 정규식은 자연스럽게 짧아집니다.
테스트도 "문장이 같은가"가 아니라 "필요한 신호가 유지되는가"를 검증하게 됩니다.

### H3. 대소문자와 공백 정책을 의식적으로 정한다

정규식의 `i` 플래그를 무심코 붙이면 대소문자 차이를 놓칠 수 있습니다.
반대로 대소문자가 사용자 경험이나 호환성에 중요하지 않은 문구라면 `i` 플래그가 테스트 유지보수에 도움이 됩니다.

```js
assert.match('ERROR: payment failed', /payment failed/i);
assert.match('retryable=false', /retryable=false/);
```

공백도 마찬가지입니다.
CLI help처럼 들여쓰기가 바뀔 수 있는 출력은 `\s+`를 쓸 수 있습니다.
하지만 CSV, fixed-width text, 프로토콜 메시지처럼 공백이 계약인 경우에는 정확히 검증해야 합니다.

### H3. 보안 검증에는 부정 패턴을 함께 둔다

문자열 테스트가 보안과 만나는 지점은 대부분 "나오면 안 되는 값"입니다.
API 키, 세션 토큰, 주민등록번호 같은 실제 민감정보 패턴은 예제에 넣지 않는 것이 좋습니다.
대신 테스트용 더미 값이 출력에 남지 않는지 검증하고, 일반적인 키 이름도 부정 패턴으로 확인할 수 있습니다.

```js
const output = 'request failed apiKey=[redacted]';

assert.match(output, /apiKey=\[redacted\]/);
assert.doesNotMatch(output, /apiKey=test-secret/);
assert.doesNotMatch(output, /Authorization:\s*Bearer/i);
```

이런 테스트는 완전한 보안 장치가 아닙니다.
하지만 로깅 포맷을 바꾸다가 마스킹이 빠지는 단순 회귀를 막는 데 효과적입니다.

## assert.match와 다른 검증 도구의 선택 기준

### H3. 구조화할 수 있으면 먼저 구조화한다

문자열로 받은 값이라도 JSON, URL, 날짜, 숫자처럼 구조화할 수 있다면 파싱 후 검증하는 편이 좋습니다.
정규식은 강력하지만, 데이터 구조를 이해하는 도구는 아닙니다.

```js
const json = '{"ok":true,"count":3}';
const payload = JSON.parse(json);

assert.deepStrictEqual(payload, {
  ok: true,
  count: 3
});
```

반대로 사람이 읽는 자유 문장, 로그 한 줄, CLI help처럼 구조화하기 어려운 출력은 `assert.match()`가 잘 맞습니다.
핵심은 "정규식으로 억지 파싱을 하고 있지는 않은가"를 점검하는 것입니다.

### H3. snapshot은 넓게 보고 match는 좁게 본다

snapshot 테스트는 큰 출력의 전체 모양을 빠르게 고정할 때 유용합니다.
하지만 출력이 자주 바뀌는 프로젝트에서는 승인 비용이 커질 수 있습니다.
`assert.match()`는 전체 모양보다 특정 신호를 좁게 검증할 때 유리합니다.

예를 들어 CLI help 전체를 snapshot으로 고정하면 옵션 설명의 작은 문구 수정도 snapshot 갱신으로 이어집니다.
반면 `assert.match()`로 핵심 옵션만 검증하면 출력 문장을 개선하면서도 중요한 옵션 누락은 잡을 수 있습니다.
둘 중 하나만 써야 하는 것은 아니며, 공개 계약의 넓이에 따라 선택하면 됩니다.

## 실무 적용 기준

### H3. 테스트 이름에 검증 의도를 드러낸다

정규식은 읽는 사람에게 의도가 바로 보이지 않을 때가 있습니다.
그래서 테스트 이름이 중요합니다.

```js
test('error message includes invalid field name', () => {
  const error = new Error('Invalid field: email');

  assert.match(error.message, /email/);
});
```

이 테스트는 정규식 자체보다 테스트 이름이 의도를 설명합니다.
나중에 메시지를 바꾸는 사람도 `email`이라는 필드 힌트가 계약이라는 사실을 이해하기 쉽습니다.

### H3. 정규식을 헬퍼로 감출 때는 신중하게 한다

여러 테스트에서 같은 패턴을 반복한다면 헬퍼가 도움이 됩니다.
하지만 너무 이른 추상화는 테스트 실패 원인을 흐릴 수 있습니다.
처음에는 테스트 안에 정규식을 직접 두고, 실제 중복이 쌓였을 때 공통 헬퍼로 빼는 편이 좋습니다.

```js
function assertRedactedToken(text) {
  assert.match(text, /token=\[redacted\]/);
  assert.doesNotMatch(text, /token=test-token/);
}
```

이런 헬퍼는 이름이 분명하고 검증 범위가 작을 때 좋습니다.
반대로 `assertValidLogLine()`처럼 너무 많은 규칙을 한 번에 숨기면 어떤 계약이 깨졌는지 찾기 어려워집니다.

## 자주 묻는 질문

### H3. assert.match는 문자열이 아닌 값에도 쓸 수 있나요?

문자열 검증용으로 생각하는 것이 안전합니다.
숫자, 객체, 배열을 정규식으로 검증하려고 문자열 변환을 끼워 넣기보다, 해당 타입에 맞는 assertion을 쓰는 편이 좋습니다.
객체 일부를 검증하려면 `assert.partialDeepStrictEqual()`, 값 하나를 검증하려면 `assert.equal()`이나 `assert.ok()`를 먼저 고려하세요.

### H3. 정규식을 너무 느슨하게 만들지 않는 기준은 무엇인가요?

테스트가 실패해야 하는 변경을 먼저 상상해 보세요.
예를 들어 에러 메시지에서 필드명이 빠지면 실패해야 한다면 필드명을 포함합니다.
로그에서 `requestId`가 빠지면 실패해야 한다면 `requestId=` 패턴을 넣습니다.
그 반대로 실패하지 않아도 되는 단어와 문장 순서는 정규식에서 빼는 편이 좋습니다.

### H3. 실제 민감정보 패턴을 테스트에 넣어도 되나요?

권장하지 않습니다.
테스트 코드도 저장소에 남고 검색됩니다.
실제 키, 실제 토큰, 실제 개인정보처럼 보이는 값을 넣지 말고, 명확한 더미 값과 `[redacted]` 같은 마스킹 표시를 사용하세요.
민감정보가 나오지 않아야 한다는 정책은 `assert.doesNotMatch()`로 검증하되 예제 값은 안전하게 유지해야 합니다.

## 정리

`assert.match()`는 문자열 테스트를 "전체 문장이 같은가"에서 "필요한 신호가 유지되는가"로 바꿔 줍니다.
에러 메시지, 로그, CLI 출력처럼 사람이 읽는 문자열에서는 이 차이가 테스트 유지보수성을 크게 좌우합니다.

실무에서는 다음 기준으로 적용하면 좋습니다.

- 공개 계약인 정확한 문자열은 `assert.equal()`로 검증한다.
- 바뀔 수 있는 사람용 문구는 `assert.match()`로 핵심 패턴만 검증한다.
- 민감정보가 없어야 하는 출력은 `assert.doesNotMatch()`를 함께 둔다.
- JSON이나 URL처럼 구조화할 수 있는 값은 파싱 후 타입에 맞게 검증한다.
- 정규식은 "실패해야 하는 변경"을 기준으로 좁고 읽기 쉽게 작성한다.

문자열 테스트는 너무 엄격하면 자주 깨지고, 너무 느슨하면 아무것도 지키지 못합니다.
`assert.match()`와 `assert.doesNotMatch()`를 적절히 쓰면 그 사이에서 실무적인 균형을 잡을 수 있습니다.

## 함께 읽기

- [Node.js assert.rejects 가이드: 비동기 에러 테스트를 정확하게 작성하는 법](/development/blog/seo/2026/05/24/nodejs-assert-rejects-async-error-testing-guide.html)
- [Node.js assert.partialDeepStrictEqual 가이드: API 응답 테스트를 덜 깨지게 만드는 부분 검증](/development/blog/seo/2026/06/23/nodejs-assert-partialdeepstrictequal-api-response-test-guide.html)
- [Node.js Error.captureStackTrace 가이드: 커스텀 에러의 스택 트레이스를 깔끔하게 만드는 법](/development/blog/seo/2026/06/22/nodejs-error-capturestacktrace-custom-error-guide.html)

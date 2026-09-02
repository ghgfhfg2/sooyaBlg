---
layout: post
title: "Node.js stream.duplexPair 가이드: 양방향 스트림을 메모리에서 테스트하기"
date: 2026-09-02 20:00:00 +0900
lang: ko
translation_key: nodejs-stream-duplexpair-bidirectional-testing-guide
permalink: /development/blog/seo/2026/09/02/nodejs-stream-duplexpair-bidirectional-testing-guide.html
alternates:
  ko: /development/blog/seo/2026/09/02/nodejs-stream-duplexpair-bidirectional-testing-guide.html
  x_default: /development/blog/seo/2026/09/02/nodejs-stream-duplexpair-bidirectional-testing-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, stream, duplexpair, duplex, socket-test, backpressure, integration-test]
description: "Node.js stream.duplexPair로 네트워크 없이 연결된 두 Duplex 스트림을 만들고, 요청·응답 프로토콜과 half-close, 오류, backpressure를 테스트하는 방법을 설명합니다."
---

TCP 소켓이나 IPC 위에 동작하는 프로토콜을 테스트할 때 매번 실제 포트를 열 필요는 없습니다.
테스트의 관심사가 주소 바인딩이나 운영체제 네트워크가 아니라 "한쪽이 쓴 데이터를 반대쪽이 정확히 읽는가"라면 메모리 안에서 연결된 스트림이 더 빠르고 단순합니다.

Node.js의 `node:stream` 모듈은 이런 상황을 위한 `duplexPair()`를 제공합니다.
이 함수는 서로 연결된 두 `Duplex` 스트림을 반환합니다.
한쪽에 쓴 데이터는 다른 쪽에서 읽을 수 있고, 반대 방향도 같은 방식으로 동작하므로 클라이언트와 서버의 양방향 연결을 작은 테스트 안에서 재현할 수 있습니다.

이 글에서는 Node.js `stream.duplexPair`의 기본 동작과 요청·응답 테스트, object mode, 종료·오류·backpressure 검증 기준을 정리합니다.
스트림 완료 처리는 [Node.js stream.finished 가이드](/development/blog/seo/2026/08/04/nodejs-stream-finished-completion-error-handling-guide.html), 취소 가능한 파이프라인은 [Node.js stream.pipeline AbortSignal 가이드](/development/blog/seo/2026/07/24/nodejs-stream-pipeline-abortsignal-cleanup-guide.html), Web Stream 연결은 [Node.js Readable.fromWeb/toWeb 가이드](/development/blog/seo/2026/04/30/nodejs-readable-fromweb-toweb-stream-bridge-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js Stream 공식 문서](https://nodejs.org/api/stream.html), [stream.duplexPair 공식 문서](https://nodejs.org/api/stream.html#streamduplexpairoptions), [Duplex 스트림 공식 문서](https://nodejs.org/api/stream.html#class-streamduplex)

## stream.duplexPair가 하는 일

### 서로 연결된 두 Duplex를 반환한다

`duplexPair(options)`의 반환값은 두 개의 `Duplex`가 들어 있는 배열입니다.
두 스트림은 대칭이므로 어느 쪽을 클라이언트로 사용할지는 호출자가 정하면 됩니다.

```js
import { duplexPair } from 'node:stream';

const [client, server] = duplexPair();

client.write('PING');

server.once('data', (chunk) => {
  console.log(chunk.toString()); // PING
});

server.write('PONG');

client.once('data', (chunk) => {
  console.log(chunk.toString()); // PONG
});
```

`client.write()`의 데이터는 `client` 자체가 아니라 `server`의 readable 쪽에서 나타납니다.
마찬가지로 `server.write()`의 결과는 `client`가 읽습니다.
실제 네트워크 연결에서 클라이언트가 보낸 바이트를 서버가 받고, 서버 응답을 클라이언트가 받는 흐름과 비슷합니다.

공식 문서 기준으로 `duplexPair()`는 Node.js 20.17.0과 22.6.0에 추가되었습니다.
프로젝트의 최소 Node.js 버전이 그보다 낮다면 테스트 및 배포 런타임을 먼저 확인해야 합니다.

### PassThrough 두 개와 목적이 다르다

`PassThrough` 하나는 자신에게 쓴 데이터를 자신의 readable 쪽으로 그대로 내보냅니다.
반면 `duplexPair()`는 한쪽의 writable이 반대쪽 readable에 연결됩니다.

양방향 연결을 `PassThrough` 두 개로 직접 조립할 수도 있지만, 종료 전파와 backpressure까지 포함해 소켓과 비슷한 인터페이스를 만들려면 연결 코드가 늘어납니다.
`duplexPair()`는 테스트 대상 함수가 `Duplex` 또는 소켓과 비슷한 객체를 받는 경우 의도를 더 분명하게 보여 줍니다.

## 요청·응답 프로토콜 테스트하기

### 줄바꿈으로 프레임 경계를 구분한다

스트림은 메시지 배열이 아니라 연속된 데이터 흐름입니다.
한 번의 `write()`가 한 번의 `data` 이벤트와 정확히 대응한다고 가정하면 안 됩니다.
간단한 텍스트 프로토콜이라면 줄바꿈 같은 프레임 구분자를 정하고 누적 버퍼에서 완성된 메시지를 꺼내야 합니다.

```js
export function serveCommands(stream) {
  let pending = '';

  stream.setEncoding('utf8');
  stream.on('data', (chunk) => {
    pending += chunk;

    while (pending.includes('\n')) {
      const newline = pending.indexOf('\n');
      const command = pending.slice(0, newline);
      pending = pending.slice(newline + 1);

      if (command === 'PING') {
        stream.write('PONG\n');
      } else {
        stream.write('ERROR unknown-command\n');
      }
    }
  });

  stream.on('end', () => {
    stream.end();
  });
}
```

이 함수는 실제 `net.Socket`에도, `duplexPair()`가 만든 한쪽 스트림에도 연결할 수 있습니다.
테스트에서는 포트 충돌이나 연결 재시도 없이 프로토콜 로직만 확인합니다.

```js
import assert from 'node:assert/strict';
import { once } from 'node:events';
import { duplexPair } from 'node:stream';
import { test } from 'node:test';
import { serveCommands } from './command-server.js';

test('PING에 PONG으로 응답한다', async (context) => {
  const [client, server] = duplexPair();
  context.after(() => {
    client.destroy();
    server.destroy();
  });

  serveCommands(server);
  client.setEncoding('utf8');

  client.write('PI');
  client.write('NG\n');

  const [response] = await once(client, 'data');
  assert.equal(response, 'PONG\n');
});
```

입력을 `PI`와 `NG\n`으로 나눠 보낸 이유는 chunk 경계에 의존하는 버그를 찾기 위해서입니다.
실제 네트워크에서도 하나의 메시지가 여러 chunk로 나뉠 수 있으므로 정상 입력뿐 아니라 의도적으로 분할한 입력을 테스트해야 합니다.

### 여러 메시지가 한 chunk에 들어오는 경우도 확인한다

반대 상황도 있습니다.
호출자가 여러 메시지를 연속으로 쓰면 수신 측에서 한 chunk처럼 관찰될 수 있습니다.
파서는 줄바꿈이 남아 있는 동안 반복해서 프레임을 꺼내야 합니다.

```js
test('연속된 명령을 모두 처리한다', async (context) => {
  const [client, server] = duplexPair();
  context.after(() => {
    client.destroy();
    server.destroy();
  });

  serveCommands(server);
  client.setEncoding('utf8');

  const responses = [];
  client.on('data', (chunk) => responses.push(chunk));

  client.end('PING\nUNKNOWN\n');
  await once(client, 'end');

  assert.equal(responses.join(''), 'PONG\nERROR unknown-command\n');
});
```

이 테스트는 입력 종료 뒤 서버가 응답을 마무리하는지까지 확인합니다.
프로토콜 구현이 요청 종료 후에도 writable 쪽을 닫지 않는다면 테스트가 끝나지 않을 수 있으므로 종료 규칙을 명시적으로 설계해야 합니다.

## object mode로 애플리케이션 메시지 테스트하기

### options는 양쪽 Duplex 생성자에 적용된다

`duplexPair(options)`에 전달한 옵션은 두 `Duplex` 생성자에 사용됩니다.
`objectMode: true`를 지정하면 Buffer나 문자열뿐 아니라 일반 객체를 그대로 주고받을 수 있습니다.

```js
import { duplexPair } from 'node:stream';

const [worker, coordinator] = duplexPair({ objectMode: true });

coordinator.write({ type: 'job', id: 42 });

worker.once('data', (message) => {
  console.log(message); // { type: 'job', id: 42 }

  worker.write({ type: 'done', id: message.id });
});

coordinator.once('data', (message) => {
  console.log(message); // { type: 'done', id: 42 }
});
```

이 패턴은 프로세스 내부 메시지 라우터나 테스트용 transport를 만들 때 편리합니다.
다만 실제 운영 연결이 JSON 또는 바이너리 프레임을 사용한다면 object mode 테스트만으로 직렬화 오류를 찾을 수 없습니다.
애플리케이션 상태 전이 테스트는 object mode로 빠르게 돌리고, 인코딩·디코딩은 byte stream 통합 테스트로 별도 검증하는 편이 좋습니다.

### 테스트용 transport와 프로덕션 transport의 계약을 맞춘다

프로덕션 코드가 `net.Socket`의 모든 속성에 직접 의존하면 `Duplex`만으로 대체하기 어렵습니다.
대신 비즈니스 로직이 필요한 최소 계약을 좁게 유지합니다.

```js
export async function exchange(stream, request) {
  stream.write(`${JSON.stringify(request)}\n`);

  for await (const chunk of stream) {
    return JSON.parse(String(chunk));
  }

  throw new Error('응답을 받기 전에 연결이 종료되었습니다.');
}
```

이 함수가 `write()`, async iteration, 종료·오류 이벤트처럼 표준 스트림 계약만 사용하면 실제 소켓과 메모리 pair 양쪽에서 재사용할 수 있습니다.
remote address나 socket timeout처럼 네트워크 전용 기능은 transport 어댑터 경계에 남겨 둡니다.

## 종료와 half-close 검증하기

### readable과 writable 수명 주기를 구분한다

`Duplex`에는 읽기와 쓰기 두 방향이 있습니다.
`end()`는 해당 스트림의 writable 쪽에 더 이상 데이터를 쓰지 않겠다는 뜻이며, 반대쪽이 보내는 데이터까지 즉시 읽지 못하게 만드는 명령은 아닙니다.

```js
import { duplexPair } from 'node:stream';

const [client, server] = duplexPair();

client.end('request');

server.on('end', () => {
  server.end('final-response');
});

let response = '';
client.on('data', (chunk) => {
  response += chunk;
});

client.on('end', () => {
  console.log(response); // final-response
});
```

요청 본문 전송이 끝난 뒤 최종 응답을 받는 프로토콜이라면 이런 half-close 동작이 중요합니다.
테스트에서는 양쪽의 `end`, `finish`, `close` 중 어떤 이벤트를 계약으로 삼는지 구분해야 합니다.

### 테스트가 끝나면 양쪽을 정리한다

실패한 assertion이나 timeout 때문에 한쪽이 열려 있으면 Node.js 테스트 프로세스가 종료되지 않을 수 있습니다.
`node:test`의 `context.after()`에 양쪽 `destroy()`를 등록하면 성공과 실패 경로 모두에서 정리할 수 있습니다.

```js
test('연결을 항상 정리한다', async (context) => {
  const [client, server] = duplexPair();

  context.after(() => {
    client.destroy();
    server.destroy();
  });

  // 테스트 본문
});
```

정상 종료 의미를 검증하는 테스트에서는 먼저 `end()`로 프로토콜 종료를 수행하고, `destroy()`는 최종 안전장치로만 사용합니다.
두 동작을 같은 의미로 취급하면 응답을 모두 flush하지 못하는 버그를 놓칠 수 있습니다.

## 오류와 backpressure 테스트하기

### 오류가 발생할 방향을 명시한다

실제 연결은 파싱 실패, 상대방 종료, 내부 처리 오류 등 여러 이유로 끊어집니다.
테스트에서 `destroy(error)`를 호출하면 오류 이벤트를 관찰할 listener 또는 완료 처리가 필요합니다.

```js
import assert from 'node:assert/strict';
import { finished } from 'node:stream/promises';

test('서버 오류를 연결 실패로 처리한다', async (context) => {
  const [client, server] = duplexPair();
  context.after(() => {
    client.destroy();
    server.destroy();
  });

  const clientCompletion = finished(client);
  const serverCompletion = finished(server);
  server.destroy(new Error('protocol failure'));

  await assert.rejects(serverCompletion, /protocol failure/);
  await assert.rejects(clientCompletion);
});
```

오류 객체에 인증 정보, 원문 요청, 개인정보를 그대로 붙이지 않도록 주의합니다.
운영 로그와 테스트 fixture에는 오류 코드, 메시지 종류, 익명화된 식별자처럼 진단에 필요한 최소 정보만 둡니다.

### 작은 highWaterMark로 느린 소비자를 재현한다

`options`에는 `highWaterMark` 같은 버퍼링 설정도 전달할 수 있습니다.
작은 값을 사용하고 반대쪽의 읽기를 늦추면 `write()`가 `false`를 반환하는 backpressure 상황을 비교적 작은 데이터로 만들 수 있습니다.

```js
import { once } from 'node:events';
import { duplexPair } from 'node:stream';

const [producer, consumer] = duplexPair({ highWaterMark: 8 });

async function writeSafely(stream, chunks) {
  for (const chunk of chunks) {
    if (!stream.write(chunk)) {
      await once(stream, 'drain');
    }
  }

  stream.end();
}

const writing = writeSafely(producer, [
  Buffer.alloc(16),
  Buffer.alloc(16),
  Buffer.alloc(16)
]);

consumer.resume();
await writing;
```

`write()`의 반환값을 무시하면 느린 소비자 앞에서 데이터가 메모리에 계속 쌓일 수 있습니다.
테스트는 `false` 반환 시 다음 쓰기를 멈추고 `drain` 뒤 재개하는지 확인해야 합니다.

## 실무 적용 체크리스트

### duplexPair가 잘 맞는 경우

- 소켓과 비슷한 양방향 프로토콜의 파서와 상태 전이를 테스트할 때
- 실제 포트 없이 클라이언트와 서버 함수를 한 프로세스에서 연결할 때
- object mode 메시지 transport를 테스트할 때
- 요청 방향 종료 후 응답을 계속 받는 half-close를 검증할 때
- 작은 `highWaterMark`로 backpressure 처리 코드를 재현할 때

반면 TLS handshake, 실제 DNS, 운영체제 socket option, 연결 timeout, 패킷 손실처럼 네트워크 계층 자체가 관심사라면 실제 소켓 통합 테스트가 필요합니다.
`duplexPair()`는 네트워크 전체를 모방하는 도구가 아니라 스트림 계약을 빠르게 검증하는 도구입니다.

### 발행 전 코드 점검 기준

1. 한 번의 `write()`와 한 번의 `data` 이벤트가 대응한다고 가정하지 않는다.
2. 분할된 프레임과 합쳐진 프레임을 모두 테스트한다.
3. 정상 `end()`와 비정상 `destroy(error)`를 구분한다.
4. 테스트 종료 시 양쪽 스트림을 정리한다.
5. `write()`가 `false`일 때 `drain`을 기다린다.
6. 실제 transport 전용 기능과 표준 `Duplex` 계약을 분리한다.
7. 최소 지원 Node.js 버전에서 `duplexPair()` 제공 여부를 확인한다.

## FAQ

### duplexPair는 실제 TCP 연결과 완전히 같나요?

아닙니다.
양방향 데이터와 스트림 수명 주기는 비슷하게 다룰 수 있지만 DNS, 포트, TLS, 실제 네트워크 지연과 손실은 재현하지 않습니다.
프로토콜 단위 테스트에는 `duplexPair()`, 네트워크 계층 검증에는 실제 socket 통합 테스트를 사용합니다.

### 어느 쪽이 client이고 어느 쪽이 server인가요?

정해져 있지 않습니다.
두 스트림은 대칭이므로 배열에서 받은 값을 코드에서 `client`, `server`처럼 역할에 맞게 이름 붙이면 됩니다.

### PassThrough와 무엇이 다른가요?

`PassThrough`는 자신에게 쓴 데이터를 자신의 readable에서 읽습니다.
`duplexPair()`는 A에 쓴 데이터를 B에서, B에 쓴 데이터를 A에서 읽습니다.
따라서 요청과 응답이 오가는 양방향 연결을 표현하기에 적합합니다.

### objectMode를 프로덕션 프로토콜 테스트에도 써도 되나요?

상태 전이와 메시지 라우팅을 검증하는 데는 유용합니다.
그러나 실제 연결이 byte 기반이라면 JSON 파싱, 문자 인코딩, 프레임 분할 문제를 놓칠 수 있으므로 byte stream 테스트도 함께 두어야 합니다.

## 마무리

Node.js `stream.duplexPair()`는 서로 연결된 두 `Duplex`를 만들어 양방향 스트림 코드를 메모리 안에서 검증하게 해 줍니다.
실제 포트를 열지 않고도 요청·응답, 입력 분할, half-close, 오류, backpressure를 빠르게 테스트할 수 있습니다.

핵심은 이 API를 네트워크 시뮬레이터가 아니라 스트림 계약 테스트 도구로 보는 것입니다.
프로토콜 로직을 표준 `Duplex` 인터페이스에 맞추고, 실제 네트워크 기능은 별도 통합 테스트로 보완하면 테스트 속도와 현실성을 균형 있게 유지할 수 있습니다.

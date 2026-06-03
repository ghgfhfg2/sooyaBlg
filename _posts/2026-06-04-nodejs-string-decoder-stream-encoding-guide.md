---
layout: post
title: "Node.js StringDecoder 가이드: 스트림 청크의 한글 깨짐을 안전하게 막는 법"
date: 2026-06-04 08:00:00 +0900
lang: ko
translation_key: nodejs-string-decoder-stream-encoding-guide
permalink: /development/blog/seo/2026/06/04/nodejs-string-decoder-stream-encoding-guide.html
alternates:
  ko: /development/blog/seo/2026/06/04/nodejs-string-decoder-stream-encoding-guide.html
  x_default: /development/blog/seo/2026/06/04/nodejs-string-decoder-stream-encoding-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, javascript, stream, encoding, stringdecoder, buffer, backend]
description: "Node.js StringDecoder로 스트림 Buffer 청크를 안전하게 문자열로 바꾸는 방법을 정리했습니다. UTF-8 한글 깨짐 원인, decoder.write와 decoder.end 사용법, 로그·파일·TCP 스트림 처리 체크리스트를 예제로 설명합니다."
---

Node.js에서 파일, TCP 소켓, 압축 해제 결과, HTTP 바디를 스트림으로 다루다 보면 데이터가 `Buffer` 청크 단위로 들어옵니다.
영문 로그만 처리할 때는 각 청크에 `chunk.toString('utf8')`을 바로 호출해도 별 문제가 없어 보일 수 있습니다.
하지만 한글, 이모지, 일부 다국어 문자가 청크 경계에 걸리면 문자열이 깨지거나 대체 문자로 바뀔 수 있습니다.

`node:string_decoder`의 `StringDecoder`는 이런 상황에서 사용할 수 있는 작고 실용적인 내장 도구입니다.
핵심은 **완성되지 않은 멀티바이트 문자를 다음 청크까지 보관했다가 안전하게 문자열로 내보내는 것**입니다.
이 글에서는 Node.js `StringDecoder`가 필요한 이유, 기본 사용법, 스트림 처리 패턴, 실무 체크리스트를 정리합니다.
Buffer와 인코딩 기초가 먼저 필요하다면 [Node.js Buffer base64url 인코딩 가이드](/development/blog/seo/2026/05/16/nodejs-buffer-base64url-encoding-guide.html)를 함께 참고하세요.

## StringDecoder가 필요한 이유

### H3. Buffer 청크는 문자 경계를 보장하지 않는다

스트림의 청크 경계는 사람이 보는 문자 경계와 다릅니다.
UTF-8에서 한글 한 글자는 여러 바이트로 표현됩니다.
스트림이 그 바이트 묶음 중간에서 잘리면, 현재 청크만으로는 완성된 문자를 만들 수 없습니다.

예를 들어 아래 코드는 일부러 한글 바이트를 중간에서 나눕니다.

```js
const text = '안녕하세요';
const buffer = Buffer.from(text, 'utf8');

const first = buffer.subarray(0, 2);
const second = buffer.subarray(2);

console.log(first.toString('utf8'));
console.log(second.toString('utf8'));
```

첫 번째 청크는 한 글자를 만들기에 부족합니다.
이 상태에서 `toString('utf8')`을 바로 호출하면 Node.js는 부족한 바이트를 알 수 없어서 깨진 문자열을 만들 수 있습니다.
로그 수집기, 파일 파서, TCP 프로토콜 처리기처럼 텍스트를 이어 붙이는 코드에서는 이런 문제가 실제 데이터 손상으로 이어질 수 있습니다.

### H3. StringDecoder는 미완성 문자를 보류한다

`StringDecoder`는 `Buffer`를 문자열로 바꾸되, 청크 끝에 완성되지 않은 문자가 있으면 내부에 보관합니다.
다음 청크가 들어왔을 때 남은 바이트와 합쳐 완성된 문자열을 반환합니다.

```js
import { StringDecoder } from 'node:string_decoder';

const decoder = new StringDecoder('utf8');
const text = '안녕하세요';
const buffer = Buffer.from(text, 'utf8');

const first = buffer.subarray(0, 2);
const second = buffer.subarray(2);

console.log(decoder.write(first));  // ''
console.log(decoder.write(second)); // '안녕하세요'
console.log(decoder.end());         // ''
```

이 차이는 데이터가 작을 때는 잘 보이지 않습니다.
하지만 파일이 커지고, 네트워크 상태가 달라지고, 청크 크기가 바뀌면 문자열 깨짐은 갑자기 나타날 수 있습니다.
그래서 스트림에서 다국어 텍스트를 직접 조립한다면 `StringDecoder`를 기본 선택지로 검토하는 편이 안전합니다.

## 기본 사용법

### H3. decoder.write로 청크를 누적 변환한다

가장 단순한 사용법은 디코더를 하나 만들고, 스트림에서 들어오는 `Buffer`마다 `decoder.write()`를 호출하는 것입니다.

```js
import { StringDecoder } from 'node:string_decoder';

const decoder = new StringDecoder('utf8');
let body = '';

for await (const chunk of readable) {
  body += decoder.write(chunk);
}

body += decoder.end();
```

여기서 `decoder.end()`는 마지막에 남아 있을 수 있는 바이트를 비우는 역할을 합니다.
완성되지 않은 문자가 끝까지 남아 있다면 적절한 대체 문자로 마무리됩니다.
따라서 스트림이 끝났을 때는 `end()`까지 호출하는 습관을 두는 것이 좋습니다.

### H3. 인코딩은 입력 데이터 기준으로 명시한다

`StringDecoder`를 만들 때는 데이터의 실제 인코딩을 지정해야 합니다.
대부분의 현대 웹·로그·JSON 데이터는 `utf8`이지만, 오래된 시스템이나 외부 연동에서는 다른 인코딩이 섞일 수 있습니다.

```js
import { StringDecoder } from 'node:string_decoder';

const utf8Decoder = new StringDecoder('utf8');
const utf16Decoder = new StringDecoder('utf16le');
const base64Decoder = new StringDecoder('base64');
```

다만 `StringDecoder`가 모든 문자 인코딩 문제를 해결해 주는 것은 아닙니다.
입력이 실제로 EUC-KR인데 `utf8`로 읽으면 경계 처리는 좋아져도 내용은 올바르게 해석되지 않습니다.
Node.js 내장 인코딩으로 처리하기 어려운 레거시 문자셋은 별도 변환 라이브러리나 upstream 인코딩 정리가 필요합니다.

## 스트림 처리 패턴

### H3. 파일 로그를 라인 단위로 자른다

로그 파일을 스트림으로 읽고 줄 단위로 처리할 때는 먼저 청크를 안전하게 문자열로 바꾼 뒤, 개행 기준으로 나누면 됩니다.

```js
import { createReadStream } from 'node:fs';
import { StringDecoder } from 'node:string_decoder';

export async function readLines(filePath, onLine) {
  const stream = createReadStream(filePath);
  const decoder = new StringDecoder('utf8');
  let pending = '';

  for await (const chunk of stream) {
    pending += decoder.write(chunk);

    const lines = pending.split(/\r?\n/);
    pending = lines.pop() ?? '';

    for (const line of lines) {
      onLine(line);
    }
  }

  pending += decoder.end();

  if (pending) {
    onLine(pending);
  }
}
```

이 패턴은 두 가지 경계를 나눠서 다룹니다.
문자 경계는 `StringDecoder`가 책임지고, 줄 경계는 `pending` 문자열이 책임집니다.
역할을 분리하면 "청크가 문자 중간에서 잘렸는가"와 "줄이 청크 중간에서 잘렸는가"를 동시에 억지로 처리하지 않아도 됩니다.
파일 탐색과 조합한다면 [Node.js fsPromises.glob 가이드](/development/blog/seo/2026/05/07/nodejs-fspromises-glob-file-discovery-guide.html)처럼 대상 파일 수집 단계를 분리하면 유지보수가 쉽습니다.

### H3. TCP 스트림에서는 메시지 경계를 따로 설계한다

TCP 소켓에서도 같은 문제가 나옵니다.
소켓은 메시지 단위가 아니라 바이트 흐름입니다.
`StringDecoder`는 바이트를 문자열로 안전하게 바꿔 주지만, 애플리케이션 메시지 하나가 어디서 끝나는지는 알 수 없습니다.

```js
import net from 'node:net';
import { StringDecoder } from 'node:string_decoder';

net.createServer((socket) => {
  const decoder = new StringDecoder('utf8');
  let pending = '';

  socket.on('data', (chunk) => {
    pending += decoder.write(chunk);

    let index;
    while ((index = pending.indexOf('\n')) !== -1) {
      const message = pending.slice(0, index);
      pending = pending.slice(index + 1);

      handleMessage(message);
    }
  });

  socket.on('end', () => {
    pending += decoder.end();

    if (pending) {
      handleMessage(pending);
    }
  });
});
```

이 예제는 개행을 메시지 구분자로 사용합니다.
길이 prefix, JSON lines, 구분자 기반 프로토콜 중 무엇을 쓰든 원칙은 같습니다.
`StringDecoder`는 문자 깨짐을 막고, 별도의 파서는 메시지 경계를 처리합니다.
스트림의 압력과 메모리 사용까지 함께 다룬다면 [Node.js stream backpressure 가이드](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)를 같이 보면 좋습니다.

## 실무 주의사항

### H3. 큰 문자열 누적은 메모리 사용량을 확인한다

`StringDecoder` 자체는 작지만, 결과 문자열을 계속 `+=`로 붙이면 메모리 사용량이 커질 수 있습니다.
작은 설정 파일이나 짧은 요청 바디는 괜찮지만, 대용량 로그나 업로드 파일 전체를 문자열 하나로 모으는 방식은 피하는 편이 좋습니다.

가능하면 아래처럼 처리 단위를 작게 유지하세요.

- 줄 단위로 읽고 바로 처리한다.
- 메시지 하나가 끝나면 즉시 파싱하고 버린다.
- 최대 허용 크기를 정하고 초과하면 중단한다.
- 원본 전체가 필요 없는 작업은 스트림 파이프라인으로 처리한다.

대용량 파일에서 일부만 읽는 작업이라면 [Node.js fs.readFile AbortSignal 가이드](/development/blog/seo/2026/05/19/nodejs-fs-readfile-abortsignal-cancel-file-read-guide.html)처럼 취소와 크기 제한 기준도 함께 두는 것이 좋습니다.

### H3. 텍스트가 아닌 데이터에는 쓰지 않는다

이미지, 압축 파일, 암호화된 바이트, 바이너리 프로토콜을 문자열로 바꾸면 데이터가 손상될 수 있습니다.
`StringDecoder`는 "텍스트로 해석해야 하는 Buffer"에 쓰는 도구입니다.

특히 업로드 파일 검증, 해시 계산, HMAC 검증처럼 바이트 정확도가 중요한 작업에서는 문자열 변환을 끼워 넣지 마세요.
웹훅 서명처럼 원문 바이트가 중요한 흐름은 [Node.js Web Crypto HMAC 가이드](/development/blog/seo/2026/05/30/nodejs-webcrypto-hmac-webhook-signature-guide.html)처럼 입력 바이트를 그대로 다루는 기준을 세워야 합니다.

## 발행 전 체크리스트

### H3. 스트림 텍스트 처리 전에 확인할 항목

- 입력 데이터가 실제로 텍스트인가?
- 입력 인코딩이 `utf8`인지, 다른 인코딩인지 확인했는가?
- 청크마다 `toString()`을 바로 호출해 멀티바이트 문자가 깨질 가능성은 없는가?
- 스트림 종료 시 `decoder.end()`를 호출하는가?
- 문자 경계와 메시지 경계를 별도 책임으로 나눴는가?
- 결과 문자열을 무제한으로 누적하지 않도록 크기 제한을 두었는가?
- 로그 예제에 실제 경로, 사용자명, 토큰 같은 민감정보가 섞이지 않았는가?

## FAQ

### H3. Buffer.toString('utf8')은 항상 위험한가요?

항상 위험한 것은 아닙니다.
완성된 전체 Buffer를 한 번에 문자열로 바꾸는 상황이라면 `buffer.toString('utf8')`이 자연스럽습니다.
문제는 스트림처럼 문자가 청크 경계에 걸릴 수 있는 상황입니다.
그때는 `StringDecoder`가 더 안전합니다.

### H3. readable.setEncoding('utf8')과 StringDecoder는 어떤 관계인가요?

Node.js 스트림의 `setEncoding('utf8')`도 내부적으로 청크 경계의 문자 처리를 고려합니다.
간단히 문자열 스트림으로 받고 싶다면 `setEncoding()`이 편할 수 있습니다.
반대로 직접 Buffer 청크를 다루면서 디코딩 시점과 메시지 파싱을 세밀하게 제어해야 한다면 `StringDecoder`를 명시적으로 쓰는 편이 읽기 좋습니다.

### H3. StringDecoder가 잘못된 인코딩도 자동으로 고쳐 주나요?

아닙니다.
`StringDecoder`는 지정한 인코딩 안에서 청크 경계 문제를 줄여 주는 도구입니다.
입력 데이터의 실제 문자셋이 다르면 올바른 결과를 보장할 수 없습니다.
외부 시스템과 연동할 때는 데이터 계약에 인코딩을 명시하고, 테스트 샘플에 한글 같은 멀티바이트 문자를 포함하는 것이 좋습니다.

## 정리

`StringDecoder`는 큰 프레임워크가 아니라, 스트림 텍스트 처리에서 자주 놓치는 경계 문제를 해결하는 작은 내장 API입니다.
파일과 네트워크 스트림은 바이트 청크를 줄 뿐이고, 그 청크가 사람이 읽는 문자 단위로 잘린다는 보장은 없습니다.

다국어 텍스트를 스트림에서 직접 조립한다면 청크마다 `toString()`을 호출하기보다 `StringDecoder`로 문자 경계를 보존하세요.
그리고 메시지 경계, 줄 경계, 크기 제한, 민감정보 로그 노출은 별도의 책임으로 분리해 점검하는 것이 좋습니다.

## 함께 읽기

- [Node.js Buffer base64url 인코딩 가이드: URL에 안전한 토큰 만들기](/development/blog/seo/2026/05/16/nodejs-buffer-base64url-encoding-guide.html)
- [Node.js fsPromises.glob 가이드: 파일 검색을 내장 API로 처리하는 법](/development/blog/seo/2026/05/07/nodejs-fspromises-glob-file-discovery-guide.html)
- [Node.js stream backpressure 가이드: 메모리 스파이크를 막는 기본 원리](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)

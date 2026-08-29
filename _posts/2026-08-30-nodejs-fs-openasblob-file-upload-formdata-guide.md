---
layout: post
title: "Node.js fs.openAsBlob 가이드: 파일 업로드를 FormData와 fetch로 단순하게 처리하는 법"
date: 2026-08-30 08:00:00 +0900
lang: ko
translation_key: nodejs-fs-openasblob-file-upload-formdata-guide
permalink: /development/blog/seo/2026/08/30/nodejs-fs-openasblob-file-upload-formdata-guide.html
alternates:
  ko: /development/blog/seo/2026/08/30/nodejs-fs-openasblob-file-upload-formdata-guide.html
  x_default: /development/blog/seo/2026/08/30/nodejs-fs-openasblob-file-upload-formdata-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, fs, openasblob, formdata, fetch, file-upload, blob, backend]
description: "Node.js fs.openAsBlob()로 로컬 파일을 Blob처럼 다루고 FormData와 fetch를 이용해 업로드하는 방법을 정리합니다. MIME 타입, 파일 변경 감지, 메모리 사용, 민감정보 점검 기준까지 실무 예제로 설명합니다."
---

Node.js에서 서버 간 파일 업로드를 구현할 때 예전에는 파일을 `Buffer`로 모두 읽거나, 스트림과 multipart 라이브러리를 직접 조합하는 경우가 많았습니다.
작은 파일 하나를 외부 API에 올리는 작업인데도 코드가 길어지고, `content-type`, 파일명, 메모리 사용 기준을 매번 다시 정해야 했습니다.

이때 검토할 만한 표준 API가 **`fs.openAsBlob()`**입니다.
공식 Node.js 문서 기준으로 `fs.openAsBlob(path, options)`는 파일에 의해 뒷받침되는 `Blob`을 Promise로 반환하며, Node.js 24.0.0과 22.17.0에서 stable로 표시되었습니다.
또 Node.js의 전역 `fetch()`는 `FormData`, `Blob`, `Headers`, `Request`, `Response`와 함께 사용할 수 있습니다.

이 글에서는 `openAsBlob()`으로 파일 업로드 코드를 단순하게 만드는 방법, `FormData`에 넣을 때의 기준, 파일 변경과 민감정보를 어떻게 점검해야 하는지 정리합니다.

파일을 전체 메모리로 읽는 기준은 [Node.js stream/consumers 가이드](/development/blog/seo/2026/06/07/nodejs-stream-consumers-text-json-buffer-guide.html), 업로드 요청의 timeout과 재시도 기준은 [Node.js fetch timeout 가이드](/development/blog/seo/2026/05/21/nodejs-fetch-abortsignal-timeout-retry-guide.html), 외부 API별 연결 풀 분리는 [Node.js fetch dispatcher 가이드](/development/blog/seo/2026/08/29/nodejs-undici-dispatcher-fetch-connection-pool-guide.html)와 함께 보면 좋습니다.

참고 문서: [Node.js fs.openAsBlob 공식 문서](https://nodejs.org/api/fs.html#fsopenasblobpath-options), [Node.js fetch와 FormData 공식 문서](https://nodejs.org/api/globals.html#fetch), [Undici fetch 문서](https://undici.nodejs.org/api/Fetch)

## fs.openAsBlob이 해결하는 문제

### H3. 파일을 Buffer로 모두 읽지 않아도 된다

파일 업로드를 급하게 만들면 아래처럼 `readFile()`로 전체 파일을 메모리에 올린 뒤 요청 본문에 넣는 코드가 나오기 쉽습니다.

```js
import { readFile } from 'node:fs/promises';

export async function uploadReport(path) {
  const bytes = await readFile(path);

  const form = new FormData();
  form.append('file', new Blob([bytes], { type: 'application/pdf' }), 'report.pdf');

  const response = await fetch('https://api.example.com/reports', {
    method: 'POST',
    body: form
  });

  if (!response.ok) {
    throw new Error(`Upload failed: ${response.status}`);
  }

  return response.json();
}
```

작은 파일에서는 충분히 동작합니다.
하지만 업로드 대상이 커지거나 동시에 여러 요청이 몰리면 파일 전체를 메모리에 올리는 방식은 부담이 됩니다.
코드만 보면 단순하지만, 운영 관점에서는 파일 크기 제한과 동시성 제한이 없으면 위험합니다.

`openAsBlob()`은 파일을 `Blob` 인터페이스로 다룰 수 있게 해 줍니다.
`FormData`가 `Blob`을 받을 수 있으므로, 파일 업로드 코드가 웹 표준 API 흐름과 더 가까워집니다.

### H3. FormData 기반 multipart 업로드가 자연스러워진다

외부 API가 multipart form 업로드를 요구한다면 `FormData`가 가장 직접적인 표현입니다.
`openAsBlob()`으로 만든 값을 그대로 append하면 파일 필드를 명확하게 만들 수 있습니다.

```js
import { openAsBlob } from 'node:fs';

export async function uploadInvoice(filePath, { signal } = {}) {
  const file = await openAsBlob(filePath, {
    type: 'application/pdf'
  });

  const form = new FormData();
  form.append('invoice', file, 'invoice.pdf');
  form.append('source', 'billing-job');

  const response = await fetch('https://api.example.com/invoices', {
    method: 'POST',
    body: form,
    signal
  });

  if (!response.ok) {
    throw new Error(`Invoice upload failed: ${response.status}`);
  }

  return response.json();
}
```

여기서 중요한 점은 `Content-Type: multipart/form-data` 헤더를 직접 넣지 않는 것입니다.
`FormData`를 요청 본문으로 넘기면 boundary가 포함된 헤더를 런타임이 구성합니다.
직접 헤더를 고정하면 boundary가 빠져 서버가 본문을 파싱하지 못할 수 있습니다.

## 기본 업로드 함수 설계

### H3. 파일명과 MIME 타입을 호출부에서 명시한다

`openAsBlob()`의 `type` 옵션에는 Blob의 MIME 타입을 넣을 수 있습니다.
자동 추론에 기대기보다 업로드 목적별로 허용 타입을 정하고, 호출부가 명시적으로 넘기게 만드는 편이 운영에 좋습니다.

```js
import { openAsBlob } from 'node:fs';
import { basename } from 'node:path';

const ALLOWED_MIME_TYPES = new Set([
  'application/pdf',
  'image/png',
  'image/jpeg',
  'text/csv'
]);

export async function createUploadForm({
  path,
  fieldName = 'file',
  filename = basename(path),
  mimeType
}) {
  if (!ALLOWED_MIME_TYPES.has(mimeType)) {
    throw new TypeError(`Unsupported upload type: ${mimeType}`);
  }

  const blob = await openAsBlob(path, { type: mimeType });
  const form = new FormData();

  form.append(fieldName, blob, filename);
  return form;
}
```

이 구조는 업로드 정책을 한곳에 모읍니다.
확장자만 보고 판단하지 않고 MIME 타입 allowlist를 두면 실수로 로그, 환경 파일, 임시 덤프 같은 파일을 올리는 사고를 줄일 수 있습니다.

물론 MIME 타입 문자열만으로 보안 검증이 끝나는 것은 아닙니다.
외부에서 받은 파일을 다시 업로드하는 경로라면 파일 크기, 확장자, 실제 포맷 검사, 백신 스캔, 저장 위치 검증까지 별도 단계로 두어야 합니다.

### H3. AbortSignal과 함께 요청 예산을 둔다

파일 업로드는 일반 JSON 요청보다 오래 걸릴 수 있습니다.
그래도 timeout 없이 무기한 기다리는 구조는 좋지 않습니다.
`AbortSignal.timeout()`을 사용해 업로드 요청 예산을 명확히 두면 장애 상황에서 대기열이 계속 쌓이는 일을 줄일 수 있습니다.

```js
export async function uploadFile({
  endpoint,
  form,
  signal,
  timeoutMs = 10_000
}) {
  const timeoutSignal = AbortSignal.timeout(timeoutMs);
  const requestSignal = signal
    ? AbortSignal.any([signal, timeoutSignal])
    : timeoutSignal;

  const response = await fetch(endpoint, {
    method: 'POST',
    body: form,
    signal: requestSignal
  });

  if (!response.ok) {
    throw new Error(`Upload failed with HTTP ${response.status}`);
  }

  return response.json();
}
```

업로드 timeout은 파일 크기와 네트워크 환경에 따라 달라져야 합니다.
예를 들어 100KB CSV 업로드와 20MB 이미지 업로드를 같은 timeout으로 묶으면 어느 한쪽에는 지나치게 느슨하거나 공격적인 기준이 됩니다.

## 파일 변경 감지를 이해해야 한다

### H3. Blob 생성 후 파일을 수정하면 읽기가 실패할 수 있다

`openAsBlob()`으로 만든 Blob은 지정한 파일을 기반으로 합니다.
Node.js 공식 문서는 Blob이 생성된 뒤 파일이 수정되면 Blob 데이터를 읽는 작업이 `DOMException`으로 실패할 수 있다고 설명합니다.
또 파일 변경 여부를 확인하기 위해 Blob 생성 시점과 읽기 전마다 동기 stat 작업을 수행합니다.

이 특성은 업로드 코드에서 꽤 중요합니다.
임시 파일을 만든 직후 `openAsBlob()`을 호출하고, 동시에 다른 작업이 같은 파일을 덮어쓰면 업로드가 중간에 실패할 수 있습니다.

```js
import { writeFile } from 'node:fs/promises';
import { openAsBlob } from 'node:fs';

export async function unsafeUpload(path) {
  const blob = await openAsBlob(path, { type: 'text/csv' });

  await writeFile(path, 'changed,data\n', 'utf8');

  const form = new FormData();
  form.append('file', blob, 'report.csv');

  return fetch('https://api.example.com/upload', {
    method: 'POST',
    body: form
  });
}
```

이런 코드는 피해야 합니다.
Blob을 만든 뒤에는 해당 파일을 읽기 전용 입력처럼 다루고, 업로드가 끝난 뒤에만 정리 작업을 수행하는 편이 안전합니다.

### H3. 임시 파일은 생성, 업로드, 삭제 순서를 분리한다

보고서 생성처럼 임시 파일을 만든 뒤 바로 업로드하는 작업이라면 생명주기를 분명히 나눕니다.

```js
import { mkdtemp, rm, writeFile } from 'node:fs/promises';
import { openAsBlob } from 'node:fs';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

export async function generateAndUploadReport(rows, { signal } = {}) {
  const directory = await mkdtemp(join(tmpdir(), 'report-'));
  const reportPath = join(directory, 'daily-report.csv');

  try {
    await writeFile(reportPath, rows.join('\n'), 'utf8');

    const report = await openAsBlob(reportPath, {
      type: 'text/csv'
    });

    const form = new FormData();
    form.append('file', report, 'daily-report.csv');

    return await uploadFile({
      endpoint: 'https://api.example.com/reports',
      form,
      signal,
      timeoutMs: 8_000
    });
  } finally {
    await rm(directory, { recursive: true, force: true });
  }
}
```

순서는 단순합니다.
파일을 완성하고, Blob을 만들고, 업로드하고, 마지막에 삭제합니다.
중간에 파일을 다시 쓰거나 append하는 단계가 끼면 실패 가능성이 생깁니다.

임시 디렉터리 정리 기준은 [Node.js mkdtempDisposable 가이드](/development/blog/seo/2026/08/09/nodejs-fspromises-mkdtempdisposable-temp-dir-cleanup-guide.html)와 함께 보면 더 깔끔하게 잡을 수 있습니다.

## 보안과 운영 체크리스트

### H3. 업로드 대상 파일을 allowlist로 제한한다

파일 업로드 자동화에서 가장 위험한 실수는 "경로만 받으면 그대로 올리는" 구조입니다.
사용자 입력이나 외부 이벤트가 파일 경로에 영향을 줄 수 있다면 반드시 업로드 가능한 루트를 제한해야 합니다.

```js
import { realpath } from 'node:fs/promises';
import { relative, resolve } from 'node:path';

const UPLOAD_ROOT = resolve('exports');

export async function assertUploadPathAllowed(filePath) {
  const root = await realpath(UPLOAD_ROOT);
  const target = await realpath(filePath);
  const relativePath = relative(root, target);

  if (relativePath.startsWith('..') || relativePath === '') {
    throw new Error('Upload path is outside the allowed export directory');
  }

  return target;
}
```

glob 패턴이나 확장자 검사는 "대상 파일 종류"를 고르는 데 도움이 됩니다.
하지만 실제 접근 제어는 `realpath()`와 허용 루트 검증처럼 파일 시스템 기준으로 확인해야 합니다.

### H3. 민감정보 파일은 업로드 전에 한 번 더 거른다

개발 환경에서는 아래 파일들이 실수로 업로드 후보에 섞일 수 있습니다.

- `.env`, `.env.local`, `.npmrc`
- 인증서, 개인 키, 토큰 파일
- 데이터베이스 덤프와 백업 파일
- 사용자 개인정보가 들어 있는 원본 로그
- 디버그용 heap snapshot과 process report

업로드 함수 안에서 파일명 denylist를 두는 것만으로 완벽하지는 않습니다.
그래도 마지막 방어선으로는 충분히 가치가 있습니다.

```js
const FORBIDDEN_UPLOAD_NAMES = [
  '.env',
  '.env.local',
  '.npmrc',
  'id_rsa'
];

export function assertSafeUploadName(filename) {
  if (FORBIDDEN_UPLOAD_NAMES.includes(filename)) {
    throw new Error(`Refusing to upload sensitive file: ${filename}`);
  }

  if (filename.endsWith('.pem') || filename.endsWith('.key')) {
    throw new Error(`Refusing to upload credential-like file: ${filename}`);
  }
}
```

실무에서는 여기에 크기 제한, MIME 타입 allowlist, 저장 루트 검증, 업로드 감사 로그를 함께 둡니다.
로그에는 파일 경로 전체나 사용자 원본 데이터를 그대로 남기지 말고, 파일 종류와 크기, 작업 ID 정도만 남기는 편이 안전합니다.

## openAsBlob을 쓰지 않는 편이 나은 경우

### H3. 파일을 변환하면서 스트리밍해야 한다면 pipeline이 더 맞다

`openAsBlob()`은 파일을 Blob으로 넘겨야 하는 FormData 업로드에 잘 맞습니다.
반대로 업로드 전에 압축, 암호화, 줄 단위 변환, 대용량 필터링을 해야 한다면 스트림 파이프라인이 더 자연스럽습니다.

예를 들어 로그 파일에서 개인정보를 제거하면서 업로드해야 한다면 `openAsBlob()`으로 바로 넘기기보다 `pipeline()`으로 변환 단계를 명확히 두는 편이 좋습니다.
이 경우 원본 파일을 그대로 외부로 보내지 않도록 변환 결과를 별도 임시 파일이나 스트림으로 관리해야 합니다.

### H3. 파일이 업로드 중 계속 바뀌는 구조라면 스냅샷을 먼저 만든다

운영 중인 로그 파일, 계속 쓰이는 SQLite 파일, 사용자가 수정 중인 문서처럼 내용이 바뀔 수 있는 파일은 바로 Blob으로 만들지 않는 편이 안전합니다.
`openAsBlob()`은 Blob 생성 후 파일 변경을 감지하면 읽기 실패로 이어질 수 있기 때문입니다.

이런 파일은 먼저 안정적인 복사본을 만든 뒤 그 복사본을 업로드합니다.
SQLite라면 단순 파일 복사보다 백업 API를 쓰는 편이 안전하며, 관련 기준은 [Node.js sqlite backup 가이드](/development/blog/seo/2026/08/20/nodejs-sqlite-backup-online-database-copy-guide.html)에서 더 자세히 볼 수 있습니다.

## 적용 전 점검표

### H3. 아래 항목을 통과하면 도입해도 좋다

`openAsBlob()` 기반 업로드를 넣기 전에는 아래 질문을 확인합니다.

- 배포 Node.js 버전이 `openAsBlob()` 안정화 버전을 지원하는가?
- 업로드 가능한 루트 디렉터리를 제한했는가?
- MIME 타입과 파일 확장자 allowlist가 있는가?
- 파일 크기와 동시 업로드 수 제한이 있는가?
- Blob 생성 후 파일을 다시 쓰지 않는가?
- `fetch()` 요청에 timeout과 취소 신호를 연결했는가?
- 민감정보 파일명과 credential-like 확장자를 차단하는가?
- 업로드 실패 시 파일 경로와 원본 데이터를 로그에 과하게 남기지 않는가?

이 기준은 코드 리뷰 체크리스트로도 쓸 수 있습니다.
파일 업로드는 "요청 하나 보내기"처럼 보여도 실제로는 파일 시스템, 네트워크, 개인정보 보호가 만나는 경계입니다.

## FAQ

### fs.openAsBlob은 fs.promises API인가요?

아닙니다.
`openAsBlob()`은 `node:fs`에서 가져옵니다.
반환값은 Promise이지만 import 경로는 `node:fs/promises`가 아니라 `node:fs`입니다.

### FormData를 쓰면 Content-Type을 직접 지정해야 하나요?

보통은 직접 지정하지 않는 편이 좋습니다.
`FormData` 본문에는 multipart boundary가 필요하고, 런타임이 이를 포함한 `Content-Type`을 구성합니다.
수동으로 `multipart/form-data`만 넣으면 서버가 본문을 파싱하지 못할 수 있습니다.

### openAsBlob은 대용량 업로드에 항상 안전한가요?

항상 그렇지는 않습니다.
파일 전체를 직접 `Buffer`로 만드는 방식보다 코드가 단순해질 수 있지만, 업로드 동시성, timeout, 외부 API 연결 풀, 파일 변경 가능성은 여전히 설계해야 합니다.
큰 파일을 변환하면서 보내야 한다면 스트림 기반 파이프라인이 더 적합할 수 있습니다.

## 마무리

`fs.openAsBlob()`은 Node.js에서 파일 업로드 코드를 웹 표준 API 흐름에 가깝게 정리해 주는 실용적인 도구입니다.
파일을 Blob으로 만들고, `FormData`에 붙이고, `fetch()`로 보내는 구조는 읽기 쉽고 테스트하기도 좋습니다.

다만 업로드 코드는 항상 운영 경계입니다.
파일 경로를 제한하고, MIME 타입과 크기를 검증하고, Blob 생성 후 파일을 수정하지 않으며, 요청 timeout과 민감정보 로그 방지 기준을 함께 두어야 합니다.
이 기준까지 갖추면 `openAsBlob()`은 작은 배치 작업부터 서버 간 문서 업로드까지 꽤 단정한 기본 선택지가 됩니다.

---
layout: post
title: "Node.js v8.writeHeapSnapshot 가이드: 메모리 누수 진단용 힙 스냅샷을 안전하게 남기는 법"
date: 2026-08-12 20:00:00 +0900
lang: ko
translation_key: nodejs-v8-writeheapsnapshot-memory-diagnostics-guide
permalink: /development/blog/seo/2026/08/12/nodejs-v8-writeheapsnapshot-memory-diagnostics-guide.html
alternates:
  ko: /development/blog/seo/2026/08/12/nodejs-v8-writeheapsnapshot-memory-diagnostics-guide.html
  x_default: /development/blog/seo/2026/08/12/nodejs-v8-writeheapsnapshot-memory-diagnostics-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, v8, writeheapsnapshot, heap-snapshot, memory-leak, diagnostics, observability, backend, javascript]
description: "Node.js node:v8의 writeHeapSnapshot()으로 메모리 누수 진단용 힙 스냅샷을 안전하게 남기는 방법을 정리합니다. 저장 경로, 이벤트 루프 차단, OOM 위험, worker thread, 민감정보 보호와 운영 체크리스트까지 실무 예제로 설명합니다."
---

Node.js 서비스에서 메모리 사용량이 계속 증가하면 로그와 메트릭만으로는 원인을 좁히기 어렵습니다.
어떤 객체가 남아 있는지, 어떤 참조 경로가 객체를 붙잡고 있는지 보려면 힙 스냅샷이 필요할 때가 있습니다.
이때 `node:v8`의 `writeHeapSnapshot()`을 사용하면 현재 V8 힙 상태를 파일로 남겨 Chrome DevTools 같은 도구에서 분석할 수 있습니다.

다만 힙 스냅샷은 가벼운 진단 로그가 아닙니다.
Node.js 공식 문서 기준으로 힙 스냅샷 생성은 동기 작업이라 이벤트 루프를 막고, 생성 시점 힙 크기의 약 두 배에 가까운 메모리가 필요할 수 있습니다.
또 파일 안에는 애플리케이션 객체 값이 들어갈 수 있으므로 토큰, 이메일, 세션 데이터 같은 민감정보 노출도 함께 고려해야 합니다.
이 글에서는 `writeHeapSnapshot()`을 언제 쓰면 좋은지, 운영 환경에서는 어떤 가드레일을 둬야 하는지, worker thread와 파일 보관 정책까지 정리합니다.

메모리 누수 분석의 전체 흐름은 [Node.js 메모리 누수 추적 실전 가이드](/development/blog/seo/2026/03/11/nodejs-memory-leak-heapdump-clinicjs-guide.html)와 함께 보면 좋습니다.
프로세스 상태를 먼저 넓게 확인하려면 [Node.js process.report 운영 진단 가이드](/development/blog/seo/2026/05/14/nodejs-process-report-production-diagnostics-guide.html)를 참고하세요.
메모리 압박을 사전에 감지하는 기준은 [Node.js process.availableMemory 가이드](/development/blog/seo/2026/05/21/nodejs-process-availablememory-memory-pressure-guide.html)와 연결됩니다.

## writeHeapSnapshot이 필요한 상황

### H3. 메모리 증가의 원인을 객체 단위로 보고 싶을 때

메모리 문제가 생겼다고 바로 힙 스냅샷부터 떠야 하는 것은 아닙니다.
먼저 RSS, heap used, GC 빈도, 요청량, 배포 시점, 캐시 크기, 큐 길이 같은 지표를 확인해야 합니다.
하지만 지표가 "메모리가 늘고 있다"까지만 말해 주고, 어떤 객체가 계속 남는지는 알려 주지 않는다면 힙 스냅샷이 필요합니다.

대표적인 상황은 아래와 같습니다.

- 배포 후 특정 라우트 호출이 반복될수록 `heapUsed`가 계단식으로 증가한다.
- 캐시 만료 정책을 넣었는데도 오래된 항목이 해제되지 않는다.
- 이벤트 리스너, 타이머, promise 대기열, AsyncLocalStorage 컨텍스트가 계속 누적되는 것 같다.
- worker thread나 job processor에서 작업이 끝난 뒤에도 큰 객체가 남아 있다.
- OOM 직전의 객체 그래프를 DevTools로 비교해야 한다.

`writeHeapSnapshot()`은 이런 질문에 답하기 위한 깊은 진단 도구입니다.
반대로 "현재 프로세스 설정이 무엇인가", "스레드와 네이티브 모듈 상태가 어떤가"처럼 넓은 진단은 `process.report`가 더 가볍고 안전한 첫 단계일 수 있습니다.

### H3. 스냅샷은 장애 대응 도구이지 상시 로깅 도구가 아니다

힙 스냅샷 파일은 커질 수 있고, 생성 중에는 애플리케이션이 멈춘 것처럼 보일 수 있습니다.
트래픽이 높은 API 서버에서 요청 처리 중 무제한으로 스냅샷을 남기면 문제를 진단하기 전에 장애를 키울 수 있습니다.

그래서 운영 기준은 단순해야 합니다.

- 자동 수집은 메모리 임계치와 쿨다운을 둔다.
- 수동 수집은 관리자 권한이 있는 내부 경로에서만 실행한다.
- 파일 저장 경로는 애플리케이션 정적 파일 경로와 분리한다.
- 생성 전 디스크 여유 공간과 현재 힙 크기를 확인한다.
- 생성 후 파일 권한, 보관 기간, 업로드 여부를 명확히 한다.

힙 스냅샷은 "필요할 때 꺼내는 정밀 도구"로 두는 편이 안전합니다.

## 기본 사용법

### H3. 명시적인 저장 경로를 사용한다

`writeHeapSnapshot()`은 인자를 생략하면 기본 파일명을 만들어 현재 작업 디렉터리에 저장합니다.
하지만 운영 코드에서는 현재 작업 디렉터리가 어디인지에 따라 파일 위치가 달라질 수 있으므로, 진단 전용 디렉터리를 명시하는 편이 좋습니다.

```js
import { mkdir } from 'node:fs/promises';
import { join } from 'node:path';
import { writeHeapSnapshot } from 'node:v8';

export async function saveHeapSnapshot({ dir = '/var/tmp/my-app-diagnostics' } = {}) {
  await mkdir(dir, { recursive: true, mode: 0o700 });

  const filename = join(dir, `heap-${Date.now()}-${process.pid}.heapsnapshot`);
  return writeHeapSnapshot(filename);
}
```

반환값은 실제 저장된 파일 경로입니다.
이 경로는 로그에 남길 수 있지만, 파일 내용을 로그로 출력하거나 외부 응답 본문으로 직접 내려주면 안 됩니다.
스냅샷 파일에는 런타임 객체 값이 포함될 수 있기 때문입니다.

### H3. 생성 전 메모리와 디스크 여유를 확인한다

공식 문서가 강조하는 핵심 위험은 두 가지입니다.
힙 스냅샷 생성은 동기적으로 이벤트 루프를 막고, 생성 시점 힙 크기의 약 두 배에 가까운 메모리가 필요할 수 있습니다.
따라서 이미 메모리가 한계에 가까운 프로세스에서 무작정 호출하면 OOM killer가 프로세스를 종료할 수 있습니다.

```js
import { statfs } from 'node:fs/promises';
import { writeHeapSnapshot } from 'node:v8';

const MIN_FREE_BYTES = 1024 * 1024 * 1024;

async function assertDiskHasRoom(dir) {
  const stats = await statfs(dir);
  const freeBytes = stats.bavail * stats.bsize;

  if (freeBytes < MIN_FREE_BYTES) {
    throw new Error('Not enough disk space for heap snapshot');
  }
}

export async function guardedHeapSnapshot(filename) {
  const { heapUsed, heapTotal, rss } = process.memoryUsage();

  if (rss > 1.5 * 1024 * 1024 * 1024) {
    throw new Error('Process memory is too high to take a safe heap snapshot');
  }

  await assertDiskHasRoom('/var/tmp/my-app-diagnostics');

  console.warn('writing heap snapshot', {
    heapUsed,
    heapTotal,
    rss,
    filename
  });

  return writeHeapSnapshot(filename);
}
```

임계치는 서비스의 메모리 제한, 컨테이너 limit, 평소 힙 크기에 맞게 조정해야 합니다.
중요한 것은 "스냅샷이 필요하다"와 "지금 이 프로세스에서 떠도 된다"를 분리하는 것입니다.

## 운영 환경에 붙이는 가드레일

### H3. 관리자용 엔드포인트는 인증과 쿨다운을 둔다

운영 중 수동으로 스냅샷을 떠야 한다면 HTTP 엔드포인트를 만들 수 있습니다.
하지만 이 엔드포인트는 일반 사용자 트래픽과 같은 수준으로 열려 있으면 안 됩니다.
최소한 내부 네트워크, 강한 인증, 감사 로그, 쿨다운을 둬야 합니다.

```js
import { mkdir } from 'node:fs/promises';
import { join } from 'node:path';
import { writeHeapSnapshot } from 'node:v8';

let lastSnapshotAt = 0;
const COOLDOWN_MS = 10 * 60 * 1000;
const SNAPSHOT_DIR = '/var/tmp/my-app-diagnostics';

export async function handleHeapSnapshotRequest(req, res) {
  if (!req.user?.roles?.includes('ops-admin')) {
    res.writeHead(403);
    res.end('forbidden');
    return;
  }

  const now = Date.now();
  if (now - lastSnapshotAt < COOLDOWN_MS) {
    res.writeHead(429);
    res.end('snapshot cooldown');
    return;
  }

  await mkdir(SNAPSHOT_DIR, { recursive: true, mode: 0o700 });
  const filename = join(SNAPSHOT_DIR, `heap-${now}-${process.pid}.heapsnapshot`);

  lastSnapshotAt = now;
  const savedTo = writeHeapSnapshot(filename);

  console.warn('heap snapshot written', {
    savedTo,
    requestedBy: req.user.id,
    requestId: req.id
  });

  res.writeHead(202, { 'content-type': 'application/json' });
  res.end(JSON.stringify({ savedTo }));
}
```

실무에서는 응답에 절대 다운로드 URL을 바로 넣지 않는 편이 안전합니다.
운영자가 서버 안에서 파일을 확인하거나, 별도 보안 저장소로 옮기는 절차를 두세요.
요청자, 요청 시간, 프로세스 ID, 저장 경로는 감사 로그에 남겨야 합니다.

### H3. 자동 수집은 임계치와 횟수 제한을 함께 둔다

메모리 사용량이 임계치를 넘을 때 자동으로 스냅샷을 남기는 방식도 가능합니다.
하지만 누수가 심한 상황에서는 임계치를 계속 넘기 때문에 스냅샷 파일이 폭증할 수 있습니다.
쿨다운, 최대 횟수, 디스크 여유 확인을 함께 둬야 합니다.

```js
import { mkdir } from 'node:fs/promises';
import { join } from 'node:path';
import { writeHeapSnapshot } from 'node:v8';

const SNAPSHOT_DIR = '/var/tmp/my-app-diagnostics';
const HEAP_USED_LIMIT = 700 * 1024 * 1024;
const COOLDOWN_MS = 15 * 60 * 1000;
const MAX_SNAPSHOTS = 3;

let snapshotCount = 0;
let lastSnapshotAt = 0;

export async function maybeWriteHeapSnapshot() {
  const now = Date.now();
  const memory = process.memoryUsage();

  if (memory.heapUsed < HEAP_USED_LIMIT) return null;
  if (snapshotCount >= MAX_SNAPSHOTS) return null;
  if (now - lastSnapshotAt < COOLDOWN_MS) return null;

  await mkdir(SNAPSHOT_DIR, { recursive: true, mode: 0o700 });

  const filename = join(SNAPSHOT_DIR, `auto-heap-${now}-${process.pid}.heapsnapshot`);
  const savedTo = writeHeapSnapshot(filename);

  snapshotCount += 1;
  lastSnapshotAt = now;

  console.warn('automatic heap snapshot written', {
    savedTo,
    heapUsed: memory.heapUsed,
    rss: memory.rss,
    snapshotCount
  });

  return savedTo;
}
```

이 함수는 주기적인 health loop나 운영자 명령에서 호출할 수 있습니다.
요청 처리 경로에서 매 요청마다 호출하는 식으로 붙이면 오히려 부하를 키울 수 있으니 주의하세요.

## worker thread와 isolate 기준

### H3. worker의 힙은 main thread 스냅샷에 포함되지 않는다

`writeHeapSnapshot()`은 현재 V8 isolate의 힙을 저장합니다.
worker thread를 사용한다면 main thread에서 만든 스냅샷은 worker 안의 객체를 포함하지 않습니다.
반대로 worker에서 만든 스냅샷도 main thread의 객체를 보여 주지 않습니다.

```js
import { writeHeapSnapshot } from 'node:v8';
import { Worker, isMainThread, parentPort } from 'node:worker_threads';
import { fileURLToPath } from 'node:url';

if (isMainThread) {
  const worker = new Worker(fileURLToPath(import.meta.url));

  worker.once('message', (workerSnapshot) => {
    console.log('worker heap snapshot:', workerSnapshot);
    console.log('main heap snapshot:', writeHeapSnapshot());
  });

  worker.postMessage('heapdump');
} else {
  parentPort.once('message', (message) => {
    if (message === 'heapdump') {
      parentPort.postMessage(writeHeapSnapshot());
    }
  });
}
```

worker 기반 이미지 처리, 압축, 검색 인덱싱, 대량 데이터 변환에서 메모리가 증가한다면 어느 thread에서 증가하는지 먼저 확인하세요.
메트릭도 main process 전체 RSS만 보지 말고 worker별 작업량, 재시작 횟수, queue backlog와 함께 봐야 원인을 좁힐 수 있습니다.

### H3. 여러 스냅샷은 같은 조건에서 비교한다

힙 스냅샷은 한 장만 봐도 단서가 나오지만, 누수 분석에서는 보통 여러 장을 비교합니다.
요청 전, 요청 반복 후, GC 유도 후처럼 조건을 나눠야 "정상적으로 커진 객체"와 "해제되지 않는 객체"를 구분할 수 있습니다.

예를 들어 재현 스크립트는 아래 순서로 설계할 수 있습니다.

```text
1. 서버를 시작하고 warm-up 요청을 보낸다.
2. baseline 힙 스냅샷을 남긴다.
3. 의심 라우트를 같은 입력으로 1,000회 호출한다.
4. 가능하면 GC 뒤 스냅샷을 남긴다.
5. 동일 작업을 한 번 더 반복하고 세 번째 스냅샷을 남긴다.
6. DevTools에서 retained size와 dominator tree를 비교한다.
```

운영 서버에서 임의로 GC를 강제하는 것은 신중해야 합니다.
재현 환경이나 staging에서는 비교 품질을 위해 쓸 수 있지만, production에서는 지연 시간과 사용자 영향까지 고려해야 합니다.

## 민감정보와 파일 보관

### H3. 힙 스냅샷은 비밀 파일처럼 다룬다

힙 스냅샷에는 문자열, 객체 속성, 요청 데이터, 캐시 데이터가 들어갈 수 있습니다.
운영 중 찍은 파일이라면 API 토큰, 세션 ID, 이메일, 이름, 내부 URL이 포함될 가능성을 배제하면 안 됩니다.

따라서 보관 기준은 보수적으로 잡아야 합니다.

- 저장 디렉터리 권한을 제한한다.
- 정적 웹 루트나 공개 다운로드 경로에 저장하지 않는다.
- 외부 분석 도구에 업로드하기 전에 보안 승인 절차를 둔다.
- 공유가 필요하면 만료 시간과 접근 권한이 있는 저장소를 사용한다.
- 분석이 끝난 파일은 보관 정책에 따라 삭제한다.

스냅샷을 압축하거나 암호화해도 민감정보가 사라지는 것은 아닙니다.
파일 접근 권한과 공유 절차가 핵심입니다.

### H3. 로그에는 경로와 메타데이터만 남긴다

스냅샷 파일의 내용을 로그에 남기면 안 됩니다.
운영 로그에는 파일 경로, 크기, 생성 시각, 프로세스 ID, 요청자, 관련 알림 ID 정도면 충분합니다.

```js
import { stat } from 'node:fs/promises';

export async function logSnapshotMetadata(savedTo, context) {
  const file = await stat(savedTo);

  console.warn('heap snapshot ready', {
    savedTo,
    sizeBytes: file.size,
    pid: process.pid,
    reason: context.reason,
    requestId: context.requestId
  });
}
```

운영 로그 시스템에 민감정보 마스킹이 있어도 스냅샷 파일 자체를 넣는 것은 피하세요.
큰 파일은 로그 비용과 전송 실패를 만들고, 무엇보다 비밀 데이터가 장기 보관 로그로 퍼질 수 있습니다.

## 실무 체크리스트

### H3. 배포 전에 정해야 할 기준

`writeHeapSnapshot()`을 코드에 넣기 전에 아래 항목을 정해 두면 장애 중 의사결정이 빨라집니다.

- 누가 스냅샷 생성을 요청할 수 있는가?
- 어떤 메모리 임계치에서 자동 수집할 것인가?
- 한 프로세스에서 최대 몇 번까지 수집할 것인가?
- 저장 디렉터리 권한은 어떻게 설정할 것인가?
- 파일은 어디로 옮기고 얼마 동안 보관할 것인가?
- worker thread를 쓰는 경우 어느 thread의 스냅샷을 남길 것인가?
- 스냅샷 생성 중 지연 시간 증가를 어디에 공지하거나 기록할 것인가?

이 기준이 없다면, 장애 중에 급하게 만든 진단 엔드포인트가 새로운 보안 리스크가 될 수 있습니다.

### H3. 발행 전 SEO와 안전 점검

- 제목과 설명에 `Node.js v8.writeHeapSnapshot` 핵심 키워드를 포함했다.
- H2/H3 구조로 문제, 기본 사용법, 운영 가드레일, worker thread, 민감정보 기준을 나눴다.
- 공식 문서 기준의 이벤트 루프 차단과 OOM 위험을 과장 없이 설명했다.
- 내부 링크를 3개 포함해 메모리 진단 관련 글로 연결했다.
- 예제 코드에는 실제 토큰, 계정, 내부 호스트 같은 민감정보를 넣지 않았다.
- 힙 스냅샷을 공개 경로에 저장하거나 외부로 자동 전송하지 않도록 안내했다.

## FAQ

### H3. writeHeapSnapshot은 운영 서버에서 써도 되나요?

쓸 수는 있지만 기본값으로 켜 두는 기능은 아닙니다.
생성 중 이벤트 루프가 막히고 추가 메모리가 필요하므로, 관리자 권한, 쿨다운, 디스크 여유 확인, 보관 정책을 갖춘 뒤 제한적으로 쓰는 편이 안전합니다.

### H3. getHeapSnapshot과 writeHeapSnapshot 중 무엇을 써야 하나요?

`getHeapSnapshot()`은 힙 스냅샷 JSON을 읽을 수 있는 stream을 반환하고, `writeHeapSnapshot()`은 파일로 저장한 뒤 파일명을 반환합니다.
운영 진단에서는 파일 보관과 감사 로그가 중요하므로 `writeHeapSnapshot()`이 단순한 선택일 때가 많습니다.
반면 별도 저장소로 stream을 직접 전달해야 하는 도구라면 `getHeapSnapshot()`을 검토할 수 있습니다.

### H3. 힙 스냅샷만 있으면 메모리 누수를 바로 찾을 수 있나요?

아닙니다.
스냅샷은 객체 그래프를 보여 주는 자료일 뿐이고, 원인 판단에는 재현 조건, 요청 패턴, 배포 변경, GC 이후 비교가 필요합니다.
한 장의 스냅샷보다 같은 조건에서 찍은 여러 장을 비교하는 방식이 더 실용적입니다.

## 마무리

`node:v8`의 `writeHeapSnapshot()`은 Node.js 메모리 누수 진단에서 강력하지만 무거운 도구입니다.
파일 하나만 남기면 끝나는 기능처럼 보이지만, 실제 운영 품질은 생성 조건, 권한, 저장 위치, 민감정보 보호, worker thread 범위, 분석 절차에서 결정됩니다.

먼저 메트릭과 `process.report`로 상황을 좁히고, 힙 객체 그래프가 필요하다고 판단되는 순간에만 제한적으로 스냅샷을 남기세요.
그렇게 해야 장애 중에도 서비스 영향과 보안 리스크를 통제하면서 필요한 단서를 확보할 수 있습니다.

## 참고 링크

- [Node.js 공식 문서: v8.writeHeapSnapshot](https://nodejs.org/api/v8.html#v8writeheapsnapshotfilenameoptions)
- [Node.js 메모리 누수 추적 실전: heap snapshot과 Clinic.js](/development/blog/seo/2026/03/11/nodejs-memory-leak-heapdump-clinicjs-guide.html)
- [Node.js process.report 운영 진단 가이드](/development/blog/seo/2026/05/14/nodejs-process-report-production-diagnostics-guide.html)
- [Node.js process.availableMemory 메모리 압박 가이드](/development/blog/seo/2026/05/21/nodejs-process-availablememory-memory-pressure-guide.html)

---
layout: post
title: "Node.js process.report 가이드: 운영 장애 순간 진단 리포트 남기는 법"
date: 2026-05-14 20:00:00 +0900
lang: ko
translation_key: nodejs-process-report-production-diagnostics-guide
permalink: /development/blog/seo/2026/05/14/nodejs-process-report-production-diagnostics-guide.html
alternates:
  ko: /development/blog/seo/2026/05/14/nodejs-process-report-production-diagnostics-guide.html
  x_default: /development/blog/seo/2026/05/14/nodejs-process-report-production-diagnostics-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, diagnostics, observability, production, backend, troubleshooting]
description: "Node.js process.report로 운영 장애 순간의 런타임 진단 리포트를 남기는 방법을 정리했습니다. 수동 생성, 시그널 트리거, 치명적 오류 자동 리포트, 민감정보 점검과 보관 전략까지 실무 예제로 설명합니다."
---

운영 장애를 분석할 때 가장 답답한 순간은 “문제가 난 바로 그때의 정보”가 남아 있지 않을 때입니다.
로그에는 에러 메시지 한 줄만 있고, 프로세스는 이미 재시작됐고, 메모리나 이벤트 루프 상태는 사라져 버립니다.
이런 상황에서는 추측이 늘어나고, 재현 환경을 만들기 전까지 원인을 좁히기 어렵습니다.

Node.js의 `process.report`는 이런 빈틈을 줄여 주는 내장 진단 기능입니다.
프로세스 정보, Node.js/V8 버전, 스택, 네이티브 모듈, 리소스 사용량 같은 런타임 상태를 JSON 리포트로 남길 수 있습니다.
이 글에서는 `process.report.writeReport()`를 수동으로 호출하는 방법부터 시그널 기반 트리거, 치명적 오류 자동 리포트, 보안 점검과 운영 보관 전략까지 정리합니다.
메모리 누수 분석이 주목적이라면 [Node.js 메모리 누수 분석 가이드: heapdump와 clinic.js로 원인 좁히기](/development/blog/seo/2026/03/11/nodejs-memory-leak-heapdump-clinicjs-guide.html)도 함께 참고하면 좋습니다.

## process.report가 해결하는 문제

### H3. 장애 순간의 런타임 상태를 파일로 남긴다

일반 로그는 애플리케이션이 직접 남긴 이벤트 중심입니다.
요청 ID, 에러 메시지, 처리 시간, 비즈니스 상태는 잘 남길 수 있지만 Node.js 런타임 자체의 상태는 충분히 담기 어렵습니다.
예를 들어 다음 질문은 로그만으로 답하기 힘든 경우가 많습니다.

- 장애 당시 Node.js와 V8 버전은 무엇이었나?
- 프로세스가 어떤 인자로 실행됐나?
- 현재 스택과 활성 핸들은 어떤 상태였나?
- 네이티브 모듈이나 공유 라이브러리는 무엇이 로드됐나?
- 힙, RSS, CPU 사용량은 어느 정도였나?

`process.report`는 이 정보를 JSON 파일로 모아 줍니다.
운영 중 문제가 생겼을 때 리포트를 확보하면, 재시작 이후에도 당시 프로세스 상태를 비교적 안정적으로 되돌아볼 수 있습니다.

```js
// scripts/write-diagnostic-report.js
const filename = process.report.writeReport();

console.log('diagnostic report written:', filename);
```

실행하면 현재 작업 디렉터리나 설정된 리포트 디렉터리에 진단 리포트 파일이 생성됩니다.
파일명은 기본적으로 타임스탬프와 프로세스 ID를 포함하므로, 여러 번 생성해도 구분하기 쉽습니다.

### H3. heapdump보다 가볍게 1차 단서를 확보한다

메모리 누수나 CPU 급증을 분석할 때 heapdump, profiler, tracing 도구가 필요할 수 있습니다.
하지만 모든 장애에서 처음부터 무거운 진단을 켤 필요는 없습니다.
heapdump는 파일이 크고, 민감한 객체 값이 포함될 수 있으며, 생성 순간 프로세스에 부담을 줄 수 있습니다.

반면 `process.report`는 운영 중 1차 단서를 확보하는 용도로 쓰기 좋습니다.
리포트만으로 모든 원인을 찾을 수는 없지만 다음 판단에는 도움이 됩니다.

- 특정 배포 버전에서만 문제가 나는지 확인
- 프로세스 시작 옵션과 환경 차이 확인
- 메모리 증가가 실제 RSS/heap 증가인지 구분
- 치명적 오류 직전의 네이티브 스택 단서 확보
- 추가로 heapdump나 profiler를 켤지 결정

즉 `process.report`는 최종 분석 도구라기보다 “장애 순간을 놓치지 않기 위한 기본 블랙박스”에 가깝습니다.
런타임 관측 지점을 코드에 심는 방법은 [Node.js diagnostics_channel 가이드: 라이브러리 계측을 안전하게 분리하는 법](/development/blog/seo/2026/04/25/nodejs-diagnostics-channel-observability-instrumentation-guide.html)과 함께 보면 흐름이 더 선명해집니다.

## 기본 사용법: 수동으로 진단 리포트 생성하기

### H3. writeReport로 현재 상태를 저장한다

가장 단순한 사용법은 `process.report.writeReport()`를 호출하는 것입니다.
장애 재현 스크립트, 운영 관리자용 엔드포인트, CLI 점검 도구에서 사용할 수 있습니다.

```js
import { mkdir } from 'node:fs/promises';
import { join } from 'node:path';

const reportDir = process.env.REPORT_DIR ?? './reports';
await mkdir(reportDir, { recursive: true });

process.report.directory = reportDir;
process.report.filename = `node-report-${Date.now()}.json`;

const filename = process.report.writeReport();
console.log(`report saved: ${filename}`);
```

운영에서는 리포트 저장 경로를 명확히 정해야 합니다.
컨테이너 환경이라면 컨테이너 내부 임시 디렉터리에만 남기지 말고, 로그 수집 에이전트가 읽을 수 있는 경로나 영속 볼륨을 고려합니다.
다만 무제한 저장은 피해야 합니다.
리포트가 반복 생성되면 디스크를 채울 수 있으므로 보관 기간과 최대 개수를 정해 두는 편이 안전합니다.

### H3. 장애 대응용 관리자 함수로 감싼다

실무에서는 진단 리포트 생성을 함수로 감싸고, 호출 위치를 제한하는 것이 좋습니다.
아무 요청에서나 리포트를 만들 수 있게 두면 디스크 사용량과 정보 노출 위험이 커집니다.

```js
import { mkdirSync } from 'node:fs';
import { join } from 'node:path';

const reportDir = process.env.REPORT_DIR ?? '/tmp/node-reports';
mkdirSync(reportDir, { recursive: true });

process.report.directory = reportDir;

export function writeDiagnosticReport(reason = 'manual') {
  const safeReason = reason.replace(/[^a-z0-9_-]/gi, '-').toLowerCase();
  const filename = `report-${safeReason}-${Date.now()}.json`;

  process.report.filename = filename;
  return process.report.writeReport();
}
```

이 함수는 다음처럼 내부 관리자 명령이나 온콜 스크립트에서만 호출하도록 제한할 수 있습니다.

```js
if (process.env.ENABLE_DIAGNOSTIC_REPORT === 'true') {
  const filename = writeDiagnosticReport('oncall-check');
  console.log('report:', filename);
}
```

중요한 점은 외부 사용자 입력을 그대로 파일명에 쓰지 않는 것입니다.
예제처럼 허용 문자만 남기거나, 아예 고정된 reason 값만 받는 방식이 안전합니다.
사용자 입력 URL이나 경로를 다룰 때도 예외 없는 검증이 필요합니다.
관련 패턴은 [Node.js URL.canParse 가이드: 사용자 입력 URL 검증을 예외 없이 안전하게 처리하는 법](/development/blog/seo/2026/05/04/nodejs-url-canparse-safe-url-validation-guide.html)에 정리해 두었습니다.

## 시그널로 운영 중 리포트 남기기

### H3. SIGUSR2 같은 내부 운영 신호를 활용한다

장애가 진행 중인 프로세스에 코드를 새로 배포하지 않고 리포트를 남기고 싶을 때는 시그널 기반 트리거가 유용합니다.
예를 들어 `SIGUSR2`를 받으면 진단 리포트를 쓰도록 설정할 수 있습니다.

```js
import { writeDiagnosticReport } from './diagnostics/report.js';

process.on('SIGUSR2', () => {
  try {
    const filename = writeDiagnosticReport('sigusr2');
    console.error(`diagnostic report written: ${filename}`);
  } catch (error) {
    console.error('failed to write diagnostic report:', error);
  }
});
```

이후 운영자가 프로세스 ID를 확인한 뒤 다음처럼 신호를 보낼 수 있습니다.

```bash
kill -USR2 12345
```

컨테이너나 프로세스 매니저를 사용한다면 실제로 어떤 신호가 애플리케이션에 전달되는지 확인해야 합니다.
PM2, systemd, Kubernetes, Docker는 각각 프로세스와 시그널 전달 방식이 다를 수 있습니다.
또한 `SIGTERM`은 종료 신호로 쓰이는 경우가 많으므로, 진단 리포트용 신호와 종료 처리용 신호를 혼동하지 않는 것이 좋습니다.
무중단 종료 흐름은 [Node.js graceful shutdown 가이드: Kubernetes 무중단 종료를 위한 SIGTERM 처리](/development/blog/seo/2026/03/11/graceful-shutdown-kubernetes-zero-downtime-guide.html)를 참고하세요.

### H3. 너무 자주 생성되지 않도록 간격 제한을 둔다

시그널 방식은 편하지만, 반복 호출되면 리포트 파일이 빠르게 쌓일 수 있습니다.
진단 함수에 간단한 rate limit을 넣어 두면 실수로 같은 신호를 여러 번 보내도 피해를 줄일 수 있습니다.

```js
let lastReportAt = 0;
const REPORT_COOLDOWN_MS = 60_000;

export function writeDiagnosticReportWithCooldown(reason) {
  const now = Date.now();

  if (now - lastReportAt < REPORT_COOLDOWN_MS) {
    throw new Error('diagnostic report cooldown is active');
  }

  lastReportAt = now;
  return writeDiagnosticReport(reason);
}
```

장애 대응 도구는 실패할 때도 명확해야 합니다.
“리포트 생성 실패”와 “쿨다운 때문에 건너뜀”을 로그에서 구분해 두면, 온콜 담당자가 불필요하게 같은 작업을 반복하지 않습니다.
운영 자동화 스크립트가 일정 시간 이상 멈추지 않게 하려면 [Node.js timers/promises setTimeout 가이드: 재시도와 지연을 깔끔하게 다루는 법](/development/blog/seo/2026/04/28/nodejs-timers-promises-settimeout-delay-retry-guide.html)도 함께 적용할 수 있습니다.

## 치명적 오류와 예외에서 자동 리포트 남기기

### H3. uncaughtException에서는 기록 후 종료를 우선한다

`uncaughtException`은 프로세스가 이미 예측 불가능한 상태에 들어갔다는 신호입니다.
이 이벤트에서 복구를 시도하며 계속 서비스하는 것은 위험할 수 있습니다.
대신 마지막 리포트와 로그를 남기고 프로세스를 종료해 재시작 정책에 맡기는 편이 일반적으로 안전합니다.

```js
import { writeDiagnosticReport } from './diagnostics/report.js';

process.on('uncaughtException', (error) => {
  console.error('uncaught exception:', error);

  try {
    const filename = writeDiagnosticReport('uncaught-exception');
    console.error('diagnostic report:', filename);
  } catch (reportError) {
    console.error('failed to write diagnostic report:', reportError);
  } finally {
    process.exitCode = 1;
    setTimeout(() => process.exit(1), 100).unref();
  }
});
```

여기서도 핵심은 복구가 아니라 증거 보존입니다.
이벤트 루프가 심하게 막혔거나 파일 시스템 접근이 실패하는 상황에서는 리포트 생성도 실패할 수 있습니다.
그래서 애플리케이션 로그, APM, 메트릭, 컨테이너 이벤트와 함께 보는 것이 좋습니다.
분산 추적을 함께 운영한다면 [Node.js OpenTelemetry 실전 가이드: 분산 추적으로 병목 찾기](/development/blog/seo/2026/03/14/opentelemetry-nodejs-distributed-tracing-practical-guide.html)가 좋은 보완책입니다.

### H3. process.report 설정 옵션을 부팅 시점에 고정한다

Node.js는 진단 리포트의 동작을 코드나 실행 옵션으로 설정할 수 있습니다.
운영에서는 부팅 시점에 “언제 리포트를 만들지”를 명확히 정하고, 런타임 중 임의 변경을 줄이는 편이 관리하기 쉽습니다.

```js
process.report.directory = process.env.REPORT_DIR ?? '/tmp/node-reports';
process.report.reportOnFatalError = true;
process.report.reportOnSignal = false;
process.report.reportOnUncaughtException = true;
```

팀마다 선호는 다를 수 있습니다.
치명적 오류와 uncaught exception에서는 자동으로 남기고, 운영 중 수동 리포트는 앞서 만든 `SIGUSR2` 핸들러로 제한하는 식의 조합이 실용적입니다.
중요한 것은 이 정책을 README나 운영 런북에 적어 두는 것입니다.
장애 순간에는 “어떤 신호를 보내야 하는지”, “리포트가 어디에 저장되는지”, “어느 로그 수집 경로로 올라가는지”가 바로 보여야 합니다.

## 리포트 파일을 안전하게 다루는 법

### H3. 민감정보 포함 가능성을 전제로 보관한다

진단 리포트에는 프로세스 인자, 환경, 경로, 스택, 네이티브 모듈 정보가 포함될 수 있습니다.
애플리케이션 구조에 따라 내부 URL, 사용자명, 파일 경로, 실행 옵션, 일부 환경 정보가 드러날 가능성도 있습니다.
따라서 리포트 파일은 일반 정적 파일 경로나 공개 버킷에 올리면 안 됩니다.

운영 기준은 보수적으로 잡는 편이 좋습니다.

- 리포트 디렉터리를 웹 루트 밖에 둔다.
- 접근 권한을 운영자와 제한된 자동화 계정으로 좁힌다.
- 장기 보관이 필요 없다면 보관 기간을 짧게 둔다.
- 외부 공유 전에는 환경 변수, 내부 호스트명, 경로를 검토한다.
- 이슈나 블로그에 첨부할 때는 원본 대신 필요한 일부만 발췌한다.

개발 글에 로그와 리포트 예시를 넣을 때도 같은 원칙이 적용됩니다.
공개 문서에 올릴 예시는 반드시 재현용 값으로 바꾸고, 실제 서비스의 토큰이나 내부 도메인을 포함하지 않아야 합니다.
예시 정리 기준은 [로그 예시 안전화 가이드: 신뢰도 높은 개발 글을 위한 민감정보 제거법](/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)을 참고하면 좋습니다.

### H3. 리포트 보관과 삭제 정책을 자동화한다

리포트는 필요할 때 없으면 곤란하지만, 무한히 쌓이면 또 다른 장애 원인이 됩니다.
운영 서버에서는 간단한 정리 작업을 두는 것이 좋습니다.

```js
import { readdir, stat, rm } from 'node:fs/promises';
import { join } from 'node:path';

const MAX_AGE_MS = 7 * 24 * 60 * 60 * 1000;

export async function cleanupOldReports(reportDir) {
  const entries = await readdir(reportDir);
  const now = Date.now();

  await Promise.all(entries.map(async (entry) => {
    if (!entry.endsWith('.json')) return;

    const file = join(reportDir, entry);
    const info = await stat(file);

    if (now - info.mtimeMs > MAX_AGE_MS) {
      await rm(file);
    }
  }));
}
```

정리 작업은 애플리케이션 요청 처리 경로와 분리하는 편이 안전합니다.
부팅 시 한 번 실행하거나, 별도 운영 크론으로 관리해도 됩니다.
다만 삭제 자동화는 경로 실수가 치명적일 수 있으므로, 리포트 전용 디렉터리만 대상으로 제한해야 합니다.
운영 스크립트의 재현성과 안전성을 높이는 기준은 [셸 명령어 안전 가이드: 개발 글에서 위험한 예제를 피하는 법](/development/blog/seo/2026/03/03/shell-command-safety-context-for-dev-guides.html)와도 연결됩니다.

## 운영 체크리스트

### H3. 도입 전에 결정할 것

`process.report`는 코드 한 줄로 시작할 수 있지만, 운영 기능으로 쓰려면 정책이 필요합니다.
다음 항목을 먼저 정리해 두면 장애 때 덜 흔들립니다.

- 리포트 저장 디렉터리와 권한
- 자동 생성 조건: fatal error, uncaught exception, 시그널 여부
- 수동 생성 방법: 내부 명령, 시그널, 관리자 CLI 중 선택
- 쿨다운과 파일 개수 제한
- 로그 수집/아카이브 경로
- 외부 공유 전 마스킹 절차
- 재시작 정책과 리포트 생성 순서

이 체크리스트는 복잡해 보이지만, 한 번 정해 두면 장애 대응 시간이 줄어듭니다.
특히 “프로세스가 죽기 직전에 무엇을 남기고 종료할 것인가”는 graceful shutdown, 로그 flush, 진단 리포트가 함께 설계되어야 합니다.

### H3. 리포트만 믿지 말고 다른 관측 신호와 연결한다

진단 리포트는 강력하지만 단독으로 모든 문제를 설명하지는 않습니다.
요청 단위 로그, 메트릭, tracing, 배포 이력, 알림 타임라인과 연결해야 원인이 보입니다.
예를 들어 특정 시간에 리포트가 남았다면 같은 시간대의 배포, 에러율, p95 latency, CPU throttling, 컨테이너 재시작 횟수를 함께 봐야 합니다.

실무에서는 다음 흐름이 좋습니다.

1. 알림 발생 시 로그와 메트릭으로 증상 범위를 확인한다.
2. 살아 있는 프로세스라면 수동 리포트를 남긴다.
3. 프로세스가 죽었다면 자동 리포트와 재시작 이벤트를 확인한다.
4. 리포트에서 런타임 단서를 확인한다.
5. 필요하면 heapdump, profiler, tracing으로 깊게 들어간다.

이렇게 계층을 나누면 장애 대응이 덜 감정적이고 더 재현 가능해집니다.
서비스가 느려지는 문제까지 함께 다뤄야 한다면 [Node.js event loop lag 모니터링 가이드: 서버가 느려지는 순간을 숫자로 잡기](/development/blog/seo/2026/04/13/nodejs-event-loop-lag-monitoring-guide.html)도 내부 링크로 이어 두면 좋습니다.

## 자주 묻는 질문

### H3. process.report는 성능에 부담이 큰가요?

항상 켜져 있는 관측 기능이라기보다, 특정 순간에 리포트 파일을 생성하는 기능으로 이해하는 것이 좋습니다.
평소 설정 자체의 부담은 크지 않지만, 리포트를 쓰는 순간에는 파일 생성과 정보 수집 비용이 발생합니다.
그래서 요청마다 호출하는 방식은 피하고, 장애 대응·치명적 오류·명시적 운영 신호처럼 제한된 상황에서 사용해야 합니다.

### H3. 진단 리포트가 heapdump를 대체하나요?

대체하지 않습니다.
`process.report`는 런타임 상태를 넓게 보여 주는 1차 진단 자료이고, heapdump는 메모리 객체를 깊게 분석하기 위한 자료입니다.
메모리 누수가 의심되면 리포트로 시점과 환경을 확인한 뒤, 필요할 때 heapdump나 profiler로 들어가는 순서가 실용적입니다.

### H3. 리포트 파일을 GitHub 이슈에 그대로 올려도 되나요?

권장하지 않습니다.
리포트에는 내부 경로, 실행 인자, 환경 정보, 스택 정보가 포함될 수 있습니다.
외부 공유가 필요하다면 원본을 그대로 올리지 말고 필요한 섹션만 발췌하고, 호스트명·토큰·사용자명·내부 URL 같은 민감정보를 검토한 뒤 공유해야 합니다.

## 마무리

`process.report`는 Node.js 운영 장애 대응에서 “그 순간의 런타임 단서”를 남기는 실용적인 내장 도구입니다.
수동 호출은 간단하지만, 운영에서 가치가 생기려면 저장 경로, 트리거, 쿨다운, 보관 기간, 민감정보 점검까지 함께 설계해야 합니다.

처음 도입한다면 치명적 오류와 uncaught exception에서 자동 리포트를 남기고, 온콜용 수동 시그널을 하나 추가하는 정도로 시작해 보세요.
그다음 로그·메트릭·트레이싱과 연결하면 장애 원인을 추측이 아니라 증거 기반으로 좁혀 갈 수 있습니다.

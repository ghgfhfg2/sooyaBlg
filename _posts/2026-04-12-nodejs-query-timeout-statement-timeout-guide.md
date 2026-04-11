---
layout: post
title: "Node.js Query Timeout 가이드: Statement Timeout으로 느린 쿼리 피해를 줄이는 법"
date: 2026-04-12 08:00:00 +0900
lang: ko
translation_key: nodejs-query-timeout-statement-timeout-guide
permalink: /development/blog/seo/2026/04/12/nodejs-query-timeout-statement-timeout-guide.html
alternates:
  ko: /development/blog/seo/2026/04/12/nodejs-query-timeout-statement-timeout-guide.html
  x_default: /development/blog/seo/2026/04/12/nodejs-query-timeout-statement-timeout-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, query-timeout, statement-timeout, database, postgres, mysql, performance, backend, resilience]
description: "Node.js에서 query timeout과 statement timeout을 어떻게 나눠야 하는지, 느린 쿼리와 lock 대기를 어떻게 제한할지, 실무 적용 기준과 예시 코드를 정리했습니다."
---

Node.js 서비스에서 응답 지연이 길어질 때 많은 팀이 애플리케이션 timeout만 먼저 조정합니다.
그런데 실제 장애는 이미 DB 안쪽에서 시작된 경우가 많습니다.
특정 쿼리가 너무 오래 실행되거나 lock 대기가 길어지면, 요청 하나의 지연이 끝나지 않고 connection pool 점유, 재시도 증가, tail latency 악화로 이어집니다.

이럴 때 중요한 것이 **query timeout**과 **statement timeout**입니다.
둘을 적절히 나누면 느린 쿼리가 시스템 전체를 붙잡는 시간을 제한하고, 장애를 더 작은 범위에서 멈출 수 있습니다.
이 글에서는 Node.js 백엔드에서 query timeout을 왜 따로 설계해야 하는지, 어떤 기준으로 값을 정해야 하는지, 그리고 운영에서 함께 봐야 할 지표를 실무 관점에서 정리합니다.

## Node.js Query Timeout이 왜 중요한가

### H3. 느린 쿼리 하나가 전체 요청 예산을 잠식한다

애플리케이션 전체 timeout이 2초라고 해도, DB 쿼리 하나가 1.8초를 써 버리면 남는 시간은 거의 없습니다.
이 상태에서는 직렬화, 후처리, 네트워크 전송까지 모두 촉박해지고, 결국 상위 계층 timeout이 연쇄적으로 발생합니다.

특히 아래 같은 징후가 보이면 DB timeout 정책을 먼저 의심할 만합니다.

- p95, p99 latency만 먼저 급증함
- DB CPU보다 lock wait, query duration이 먼저 늘어남
- connection acquire wait가 함께 증가함
- 같은 엔드포인트에서 가끔만 매우 느린 요청이 발생함

이 문제는 단순 성능 이슈가 아니라 **지연 전파 차단** 문제입니다.
요청 전체 예산을 관리하는 관점은 [Node.js Timeout Budget 가이드](/development/blog/seo/2026/03/31/nodejs-timeout-budget-deadline-propagation-guide.html)와도 직접 연결됩니다.

### H3. 애플리케이션 timeout만으로는 DB 자원 점유를 충분히 막지 못한다

Node.js에서 `AbortController`나 HTTP timeout을 걸어 두더라도, DB 서버 쪽 실행이 즉시 멈추지 않는 경우가 있습니다.
클라이언트는 기다리기를 포기했지만, DB는 이미 시작된 statement를 계속 수행할 수 있습니다.
그러면 사용자는 실패 응답을 받았는데도 DB 연결과 CPU는 계속 소비됩니다.

그래서 실무에서는 아래를 구분해야 합니다.

- 요청 전체 timeout
- DB 연결 획득 timeout
- DB statement 실행 timeout
- lock wait timeout

이렇게 나눠야 어디서 시간이 새는지 파악할 수 있고, 느린 쿼리가 pool 전체를 오래 붙잡는 문제를 줄일 수 있습니다.
관련해서 연결 고갈 관점은 [Node.js Connection Pool Exhaustion 가이드](/development/blog/seo/2026/04/11/nodejs-connection-pool-exhaustion-prevention-guide.html)도 함께 참고할 만합니다.

## Query Timeout과 Statement Timeout은 어떻게 다른가

### H3. query timeout은 클라이언트 관점의 대기 제한이다

라이브러리나 ORM에서 제공하는 query timeout은 대개 애플리케이션이 결과를 기다리는 최대 시간을 의미합니다.
즉 Node.js 프로세스가 얼마나 오래 기다릴지를 제한하는 장치에 가깝습니다.

장점은 적용이 쉽고, 요청 예산과 연결해 제어하기 좋다는 점입니다.
하지만 DB 엔진이 실제로 해당 statement를 즉시 중단하는지는 드라이버와 DB 설정에 따라 다를 수 있습니다.
그래서 이것만으로는 충분하지 않은 경우가 있습니다.

### H3. statement timeout은 DB 서버 관점의 실행 상한이다

PostgreSQL의 `statement_timeout` 같은 설정은 DB 서버가 statement 실행 시간을 직접 제한합니다.
이 방식은 느린 쿼리가 DB 내부에서 너무 오래 남아 있지 않도록 막는 데 유리합니다.

실무적으로는 아래처럼 이해하면 편합니다.

- query timeout: 앱이 얼마나 오래 기다릴지 정함
- statement timeout: DB가 얼마나 오래 실행을 허용할지 정함

둘은 대체 관계가 아니라 보완 관계입니다.
앱이 먼저 포기하더라도 DB가 계속 일하면 자원 낭비가 남고, 반대로 DB만 끊고 앱 예산 관리를 하지 않으면 사용자 경험이 불안정해질 수 있습니다.

## Node.js에서 timeout 값을 정하는 실무 기준

### H3. 전체 요청 timeout 안에서 DB 예산을 따로 배정한다

가장 단순하고 안전한 출발점은 요청 timeout을 먼저 정하고, 그 안에서 DB 예산을 나누는 것입니다.
예를 들어 아래처럼 시작할 수 있습니다.

- 전체 요청 timeout: 2000ms
- connection acquire timeout: 100~200ms
- statement timeout: 500~800ms
- 애플리케이션 후처리와 응답 전송: 나머지 예산

핵심은 DB가 요청 예산 대부분을 독점하지 못하게 하는 것입니다.
읽기 API와 쓰기 API는 허용 시간이 다를 수 있고, 배치성 작업은 별도 pool이나 별도 timeout 정책을 두는 편이 안전합니다.

### H3. 평균이 아니라 tail latency 기준으로 조정한다

timeout은 평균 응답 시간이 아니라 느린 상위 구간을 기준으로 잡아야 합니다.
평균이 80ms라고 해서 timeout을 100ms로 두면 정상적인 변동에도 자주 끊길 수 있습니다.
반대로 p99가 이미 2초를 넘는데 timeout이 10초면 장애 확산을 너무 오래 허용하게 됩니다.

실무에서는 아래 순서가 유용합니다.

1. 엔드포인트별 p95, p99 latency 확인
2. 쿼리별 duration 분포 확인
3. lock wait, deadlock, row count 패턴 확인
4. 그다음 timeout을 조정

이렇게 해야 단순히 숫자를 줄이는 것이 아니라, 실제 병목에 맞는 정책이 됩니다.

## PostgreSQL과 MySQL에서 같이 봐야 할 설정

### H3. PostgreSQL은 statement_timeout과 lock timeout을 함께 본다

PostgreSQL에서는 `statement_timeout`만 보지 말고 `lock_timeout`도 함께 보는 편이 좋습니다.
느린 쿼리 자체보다 lock 대기 때문에 지연이 커지는 경우가 적지 않기 때문입니다.

예를 들면 세션 단위로 아래처럼 둘 수 있습니다.

```sql
SET statement_timeout = '800ms';
SET lock_timeout = '200ms';
```

이 조합은 쿼리 실행과 lock 대기를 분리해 다루게 해 줍니다.
특히 쓰기 경합이 있는 서비스에서는 긴 lock wait를 일찍 끊는 것만으로도 장애 전파를 꽤 줄일 수 있습니다.

### H3. MySQL은 max execution time과 lock wait 정책을 분리해 생각한다

MySQL 계열에서는 쿼리 실행 제한과 lock wait 관련 설정이 PostgreSQL처럼 완전히 같은 방식은 아닙니다.
그래서 드라이버 timeout과 서버 설정을 함께 검토해야 합니다.

중요한 것은 제품별 문법보다 원칙입니다.

- 실행 시간이 너무 긴 쿼리를 제한한다
- lock 대기를 별도로 제한한다
- 애플리케이션 timeout과 서버 timeout을 맞춘다
- timeout 발생 시 재시도 정책을 무제한으로 두지 않는다

재시도는 잘못 다루면 느린 쿼리 문제를 더 크게 만들 수 있습니다.
이 부분은 [Node.js Retry Budget 가이드](/development/blog/seo/2026/04/05/nodejs-retry-budget-overload-control-guide.html)와 같이 설계하면 좋습니다.

## Node.js 코드에서 적용할 수 있는 패턴

### H3. 요청 단위 deadline을 만들고 DB 호출에 전달한다

DB timeout을 매번 하드코딩하기보다, 요청 단위 deadline을 먼저 만들고 그 안에서 남은 시간을 계산해 사용하는 편이 좋습니다.

```js
function remainingMs(deadlineAt) {
  return Math.max(0, deadlineAt - Date.now());
}

app.get('/orders/:id', async (req, res, next) => {
  const deadlineAt = Date.now() + 2000;

  try {
    const acquireTimeoutMs = 150;
    const statementTimeoutMs = Math.min(800, remainingMs(deadlineAt) - 200);

    const order = await fetchOrder({
      id: req.params.id,
      acquireTimeoutMs,
      statementTimeoutMs,
    });

    res.json(order);
  } catch (error) {
    next(error);
  }
});
```

이 방식의 장점은 서비스 전체 timeout 예산과 DB timeout이 따로 놀지 않게 만든다는 점입니다.
또한 요청별로 남은 시간을 로깅하면, 어디서 예산을 많이 쓰는지도 파악하기 쉬워집니다.

### H3. timeout 에러를 같은 장애로 뭉뚱그리지 않는다

운영에서는 timeout 에러를 하나로 묶으면 원인 파악이 느려집니다.
적어도 아래 정도는 분리해 두는 편이 좋습니다.

- connection acquire timeout
- statement timeout
- lock timeout
- upstream request timeout

예를 들어 statement timeout이 늘고 있다면 느린 쿼리나 인덱스 문제를 먼저 봐야 합니다.
반면 acquire timeout이 늘고 있다면 connection pool 경쟁이나 동시성 제어 부족을 먼저 의심해야 합니다.
N+1 패턴처럼 쿼리 수 자체가 비정상적으로 늘어난 경우도 함께 봐야 하므로 [Node.js N+1 Query 문제 가이드](/development/blog/seo/2026/04/11/nodejs-n-plus-one-query-detection-optimization-guide.html)와 연결해서 점검하는 것이 좋습니다.

## Timeout만 설정해서는 해결되지 않는 경우

### H3. 느린 쿼리 원인을 함께 줄여야 한다

timeout은 피해를 제한하는 장치이지, 근본 최적화 자체는 아닙니다.
자주 timeout이 나는 쿼리는 아래 항목을 같이 점검해야 합니다.

- 인덱스가 실제 필터와 정렬 조건에 맞는가
- 반환 row 수가 과도하지 않은가
- 불필요한 relation 조회가 붙어 있지 않은가
- transaction이 너무 길지 않은가
- 배치 작업과 사용자 요청이 같은 자원을 경쟁하지 않는가

즉 timeout을 넣는 이유는 느린 쿼리를 방치하기 위해서가 아니라, 느린 쿼리가 전체 시스템을 끌고 내려가지 못하게 하려는 것입니다.

### H3. 과도하게 짧은 timeout은 정상 트래픽도 깨뜨릴 수 있다

반대로 timeout을 너무 공격적으로 줄이면 평소에는 괜찮던 요청도 쉽게 실패합니다.
특히 cold cache, 피크 시간대, 배포 직후처럼 일시적으로 느려지는 구간에서 false positive가 늘 수 있습니다.

그래서 아래 원칙이 안전합니다.

- 전체 요청 예산보다 약간 보수적으로 DB timeout을 둔다
- 읽기와 쓰기 timeout을 같은 값으로 고정하지 않는다
- timeout 변경 전후의 실패율과 p95, p99를 함께 본다
- 배치 작업은 사용자 트래픽과 분리된 정책을 둔다

## 실무 점검 체크리스트

### H3. 발행 전이 아니라 운영 전 꼭 확인할 항목

DB timeout 정책을 적용할 때는 아래를 먼저 점검하면 시행착오를 줄일 수 있습니다.

- 엔드포인트별 요청 timeout과 DB timeout이 정렬되어 있는가
- 드라이버 timeout과 DB 서버 timeout이 충돌하지 않는가
- timeout 에러가 로그와 APM에서 구분되어 보이는가
- timeout 이후 재시도 budget이 제한되어 있는가
- lock wait와 느린 쿼리 로그를 함께 수집하고 있는가

이 다섯 가지가 정리되면 query timeout은 단순 설정값이 아니라, 서비스 안정성을 지키는 운영 장치가 됩니다.

## 마무리

Node.js 서비스에서 query timeout과 statement timeout은 성능 튜닝의 부가 옵션이 아닙니다.
느린 쿼리와 lock 대기가 전체 요청 예산을 집어삼키지 못하게 하는 **안전장치**에 가깝습니다.

핵심은 한 가지입니다.
애플리케이션이 기다리는 시간과 DB가 실행을 허용하는 시간을 분리해서 설계해야 합니다.
그 기준이 있어야 느린 쿼리를 더 빨리 발견하고, connection pool 고갈이나 재시도 폭증 같은 2차 장애도 줄일 수 있습니다.

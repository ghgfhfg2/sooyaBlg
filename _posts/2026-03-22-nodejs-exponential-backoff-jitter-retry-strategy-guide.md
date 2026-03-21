---
layout: post
title: "Node.js 재시도 전략 가이드: Exponential Backoff와 Jitter로 장애 증폭 막기"
date: 2026-03-22 08:00:00 +0900
lang: ko
translation_key: nodejs-exponential-backoff-jitter-retry-strategy-guide
permalink: /development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html
alternates:
  ko: /development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html
  x_default: /development/blog/seo/2026/03/22/nodejs-exponential-backoff-jitter-retry-strategy-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, retry, exponential-backoff, jitter, resilience, backend]
description: "Node.js 서비스에서 단순 재시도로 장애를 키우지 않기 위해 exponential backoff와 jitter를 어떻게 설계하고 적용할지 실무 기준으로 정리했습니다."
---

외부 API가 잠깐 느려졌다고 해서 모든 요청을 즉시 다시 보내면, 원래 작은 장애가 더 큰 장애로 번지는 경우가 많습니다.
문제는 재시도 자체가 아니라 **언제, 얼마나, 어떤 요청만 다시 시도할지** 기준 없이 붙이는 데 있습니다.
이 글에서는 Node.js 서비스에서 **재시도 전략(retry strategy)** 을 어떻게 설계하면 좋은지, `exponential backoff`와 `jitter`를 중심으로 장애 증폭을 줄이는 방법을 실무 기준으로 정리합니다.

## 왜 재시도 전략이 중요할까

### H3. 단순 재시도는 성공률을 높이기도 하지만 장애를 키우기도 한다

재시도는 분명 유용합니다.
네트워크 일시 장애, 짧은 타임아웃, 순간적인 upstream 과부하처럼 잠깐 흔들리는 문제는 한두 번의 재시도로 회복되는 경우가 많습니다.
하지만 아래처럼 구현하면 상황이 금방 나빠집니다.

- 실패하자마자 0ms 간격으로 즉시 재시도한다
- 모든 에러를 동일하게 재시도한다
- 모든 요청이 같은 시점에 같은 간격으로 몰린다
- 총 대기 시간과 최대 시도 횟수 제한이 없다

이 경우 재시도는 복구 장치가 아니라 **부하 증폭기**가 됩니다.
원래는 1번 실패할 요청이 3번, 5번, 10번씩 다시 나가면서 upstream 과 우리 서비스 둘 다 더 버거워집니다.

### H3. 특히 마이크로서비스와 외부 연동 구간에서 흔히 문제를 만든다

재시도 전략은 아래 같은 구간에서 자주 중요해집니다.

- 결제, 메시징, 이메일 같은 외부 API 호출
- 내부 마이크로서비스 간 HTTP/gRPC 통신
- Redis, DB 프록시, 메시지 브로커 접속 재시도
- 배치 작업이나 워커의 API pull/push 로직

이런 구간은 순간적인 실패가 드물지 않습니다.
그래서 재시도는 필요하지만, 무작정 넣으면 **실패를 숨기면서 시스템 전체를 흔드는 코드**가 되기 쉽습니다.

## Exponential Backoff는 무엇인가

### H3. 시도할수록 대기 시간을 늘려서 상대 시스템 회복 시간을 준다

`exponential backoff`는 재시도할 때마다 대기 시간을 점점 늘리는 방식입니다.
예를 들어 기본 지연을 200ms로 잡으면 다음처럼 갈 수 있습니다.

- 1차 재시도: 200ms
- 2차 재시도: 400ms
- 3차 재시도: 800ms
- 4차 재시도: 1600ms

핵심은 단순합니다.
실패가 계속되면 즉시 다시 때리지 말고, **조금씩 더 오래 기다리며 상대가 회복할 시간을 주자**는 것입니다.
이 방식은 순간적인 장애에는 비교적 빠르게 대응하면서도, 지속적인 장애에는 공격적인 재시도를 줄여줍니다.

### H3. 고정 간격 재시도보다 대개 운영상 안전하다

고정 간격 재시도도 구현은 쉽습니다.
하지만 모든 요청이 500ms마다 같은 패턴으로 재시도하면, 장애 구간에서 트래픽 파동이 반복적으로 생길 수 있습니다.
반면 exponential backoff는 실패가 이어질수록 간격이 벌어져서 아래 장점이 있습니다.

- upstream 복구 시간 확보
- 네트워크 혼잡 완화
- 불필요한 CPU·커넥션 사용 감소
- 전체 시스템의 장애 전파 속도 완화

즉, 성공률만 볼 게 아니라 **장애 시의 시스템 행동**까지 봐야 합니다.

## Jitter를 왜 꼭 같이 써야 할까

### H3. backoff만 쓰면 요청들이 여전히 같은 박자로 몰릴 수 있다

많은 팀이 exponential backoff를 넣고 안심하지만, 실제 운영에서는 `jitter`가 빠지면 여전히 문제가 남습니다.
모든 인스턴스가 같은 시점에 실패하고 같은 공식으로 대기 시간을 계산하면, 다음 재시도도 거의 동시에 몰릴 수 있기 때문입니다.

예를 들어 1000개의 요청이 한 번에 실패하면, backoff만 있는 경우 200ms 뒤에 1000개가 다시 오고, 그다음엔 400ms 뒤에 또 몰릴 수 있습니다.
이건 장애 순간에 **재시도 떼창**을 만드는 셈입니다.

### H3. jitter는 재시도 타이밍을 분산해 herd effect를 줄인다

`jitter`는 대기 시간에 약간의 무작위성을 넣는 방식입니다.
즉 400ms를 모두 정확히 기다리는 대신, 예를 들어 200~400ms 혹은 0~400ms 같은 범위 안에서 랜덤하게 기다리게 만듭니다.

이렇게 하면 아래 효과가 있습니다.

- 동시에 실패한 요청들의 재시도 시점이 분산된다
- upstream 입장에서는 스파이크보다 완만한 부하로 보인다
- recovery 구간에서 다시 죽이는 현상을 줄인다

실무에서는 backoff만 넣는 것보다 **backoff + jitter** 조합이 훨씬 안전한 기본값에 가깝습니다.

## Node.js에서 구현하는 기본 패턴

### H3. 재시도 가능한 에러와 아닌 에러를 먼저 구분한다

모든 실패를 재시도하면 안 됩니다.
예를 들어 인증 실패나 잘못된 요청 본문은 다시 보내도 성공하지 않습니다.
보통 아래처럼 구분합니다.

- **재시도 후보**: 네트워크 오류, 타임아웃, 429, 502, 503, 504
- **재시도 비권장**: 400, 401, 403, 404, 스키마 오류, 비즈니스 검증 실패

즉 재시도 로직의 첫 단계는 "몇 번 더 해볼까"가 아니라 **이 실패가 재시도할 가치가 있는가**입니다.

### H3. 최대 시도 횟수와 총 시간 상한을 함께 둔다

재시도는 제한이 있어야 합니다.
최대 시도 횟수만 두고 전체 시간을 안 막으면, 느린 호출이 워커 스레드나 요청 처리 시간을 오래 점유할 수 있습니다.
실무에서는 보통 아래 둘을 함께 둡니다.

- 최대 시도 횟수: 예) 3~5회
- 총 소요 시간 예산: 예) 2초, 5초, 10초

이렇게 해야 호출 하나가 시스템 전체의 지연을 끝없이 끌고 가지 않습니다.

### H3. 간단한 backoff + jitter 유틸 예시

아래는 Node.js에서 쓸 수 있는 단순 예시입니다.

```ts
function sleep(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function calculateDelay(attempt: number, baseMs = 200, capMs = 3000) {
  const exponential = Math.min(capMs, baseMs * 2 ** (attempt - 1));
  const jitter = Math.random() * exponential;
  return Math.floor(jitter);
}

function isRetryableError(error: any) {
  const status = error?.status ?? error?.response?.status;

  if (status && [429, 502, 503, 504].includes(status)) {
    return true;
  }

  if (error?.code && ['ECONNRESET', 'ETIMEDOUT', 'ECONNREFUSED', 'EAI_AGAIN'].includes(error.code)) {
    return true;
  }

  return false;
}

export async function retryWithBackoff<T>(
  task: () => Promise<T>,
  {
    maxAttempts = 4,
    baseMs = 200,
    capMs = 3000,
  }: {
    maxAttempts?: number;
    baseMs?: number;
    capMs?: number;
  } = {},
): Promise<T> {
  let lastError: unknown;

  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    try {
      return await task();
    } catch (error) {
      lastError = error;

      if (attempt === maxAttempts || !isRetryableError(error)) {
        throw error;
      }

      const delay = calculateDelay(attempt, baseMs, capMs);
      await sleep(delay);
    }
  }

  throw lastError;
}
```

이 코드는 아주 단순한 버전이지만, 최소한 아래 원칙은 담고 있습니다.

- 재시도 대상 에러를 제한한다
- 시도 횟수 제한을 둔다
- exponential backoff를 적용한다
- jitter로 타이밍을 분산한다

## 어떤 jitter 방식을 고를까

### H3. full jitter는 안전한 기본값으로 자주 쓰인다

jitter에도 여러 변형이 있습니다.
실무에서 자주 비교하는 방식은 아래 정도입니다.

- **no jitter**: 지연 시간 고정
- **equal jitter**: 일정 최소 지연 + 일부 랜덤성
- **full jitter**: 0 ~ backoff 범위 안에서 랜덤
- **decorrelated jitter**: 이전 지연값을 반영해 더 유연하게 변동

대부분의 HTTP 호출 재시도에서는 `full jitter`가 단순하고 안전한 기본값으로 자주 쓰입니다.
장애 구간에서 요청을 골고루 흩뜨리는 데 유리하기 때문입니다.

### H3. 너무 공격적인 base 값은 초반 재시도를 과밀하게 만든다

base delay를 너무 작게 잡으면 jitter가 있어도 초반 재시도가 과도하게 몰릴 수 있습니다.
예를 들어 20ms, 40ms, 80ms 식으로 가면 사용자 체감은 빨라 보일 수 있지만, upstream 장애 시에는 거의 즉시 재공격하는 셈이 됩니다.

보통은 호출 성격에 따라 다르게 잡습니다.

- 사용자 요청 경로: 짧은 예산 안에서 100~300ms base
- 백그라운드 작업: 더 여유 있게 300~1000ms base
- 외부 파트너 API: rate limit 정책과 `Retry-After`를 우선 고려

즉 숫자는 공식이 아니라 **호출 성격과 장애 비용에 맞는 운영 파라미터**입니다.

## AbortController와 함께 써야 하는 이유

### H3. 재시도는 각 시도와 전체 호출 모두 timeout 예산 안에 있어야 한다

재시도 로직만 있고 timeout이 없으면, 실패가 길어질 때 호출 하나가 너무 오래 매달릴 수 있습니다.
그래서 각 시도 자체의 timeout과 전체 재시도 체인의 시간 예산을 함께 관리하는 편이 좋습니다.

Node.js에서는 `AbortController`를 같이 쓰면 비교적 깔끔합니다.
각 시도에 개별 timeout을 두고, 상위 레벨에서는 전체 deadline을 관리하는 식입니다.
이 패턴을 안 두면 재시도가 실패 복구가 아니라 **지연 누적기**가 되기 쉽습니다.

## 운영에서 자주 놓치는 포인트

### H3. POST 재시도는 멱등성 없이는 위험하다

재시도에서 가장 위험한 부분 중 하나가 `POST`입니다.
결제, 주문 생성, 메시지 발송처럼 부작용이 있는 요청은 응답을 못 받았다고 해서 다시 보내면 중복 처리될 수 있습니다.
그래서 아래 같은 보호장치가 필요합니다.

- idempotency key 사용
- 서버 측 중복 요청 감지
- 재시도 가능한 API와 불가능한 API 명확화

즉 "네트워크 오류니까 다시 보내자"는 발상이 언제나 안전하지는 않습니다.
특히 쓰기 요청은 **재시도보다 멱등성 설계가 먼저**인 경우가 많습니다.

### H3. 429와 Retry-After는 서버 힌트를 우선해야 한다

상대 서버가 `429 Too Many Requests`와 함께 `Retry-After` 헤더를 준다면, 우리 쪽 임의 backoff보다 그 힌트를 우선하는 편이 보통 맞습니다.
이건 단순 예의 문제가 아니라, 상대가 허용하는 복구 리듬을 따르는 것이기 때문입니다.

즉 재시도 전략은 클라이언트의 일방적 결정이 아니라 **서버가 준 신호를 읽는 로직**까지 포함해야 합니다.

### H3. 관측성 없이 재시도만 넣으면 장애가 숨는다

재시도는 성공률을 올릴 수 있지만, 동시에 원래 실패를 로그에서 흐리게 만들 수 있습니다.
최종적으로 성공했다는 이유로 중간 실패를 버리면, upstream이 이미 흔들리고 있다는 신호를 놓치게 됩니다.

그래서 아래 지표를 함께 보는 편이 좋습니다.

- 재시도 횟수 분포
- 최종 성공 전 실패 비율
- upstream별 429/5xx 증가율
- 최대 시도 후 최종 실패율
- 재시도로 인한 추가 지연 시간

재시도는 에러를 지우는 기능이 아니라 **에러를 견디면서도 드러내는 기능**이어야 합니다.

## 언제 재시도를 줄이거나 아예 빼야 할까

### H3. 사용자 요청 경로에서 응답 지연 비용이 더 크다면 공격적으로 줄여야 한다

재시도는 성공률을 조금 높이는 대신 지연 시간을 늘립니다.
그래서 로그인, 결제 승인, 핵심 페이지 렌더링처럼 응답 지연이 치명적인 경로에서는 재시도 횟수를 보수적으로 잡거나, 아예 빠르게 실패시키고 상위 fallback 전략을 두는 편이 낫습니다.

이럴 때는 아래 조합이 더 현실적일 수 있습니다.

- 짧은 timeout
- 1회 이하의 제한적 재시도
- circuit breaker
- fallback response 또는 graceful degradation

즉 모든 경로에 같은 retry policy를 붙이는 건 대개 좋지 않습니다.

## 배포 전 체크리스트

### H3. 구현 점검

- 재시도 가능한 에러와 아닌 에러를 명확히 구분했는가
- 최대 시도 횟수와 전체 시간 예산이 있는가
- exponential backoff에 jitter를 함께 적용했는가
- `429`의 `Retry-After`를 우선 반영하는가
- POST/결제/주문 같은 쓰기 요청은 멱등성 없이 재시도하지 않는가

### H3. 보안·운영·콘텐츠 점검

- 코드 예시에 실제 토큰, 내부 URL, 개인정보가 포함되지 않았는가
- 특정 서비스 장애를 과장하거나 선동적으로 표현하지 않았는가
- 내부링크와 메타 설명, 태그가 일관되게 설정됐는가
- 독자가 바로 적용할 수 있는 숫자 기준과 운영 포인트가 포함됐는가

## 요약

Node.js에서 재시도는 실패를 줄이는 도구가 될 수도 있고, 장애를 증폭시키는 장치가 될 수도 있습니다.
안전한 기본값은 보통 **재시도 대상 제한, 최대 시도 횟수, timeout 예산, exponential backoff, jitter, 멱등성 고려**의 조합입니다.
특히 장애 구간에서 요청이 한꺼번에 다시 몰리는 문제를 줄이려면, 단순 retry보다 **backoff + jitter + 관측성**을 함께 설계하는 편이 훨씬 낫습니다.

## 내부 링크

- [Circuit Breaker 패턴 실전 가이드: Node.js 외부 API 장애 전파 줄이기](/development/blog/seo/2026/03/12/circuit-breaker-pattern-nodejs-external-api-stability-guide.html)
- [Node.js AbortController 실전 가이드: 타임아웃·취소 표준화로 장애 반경 줄이기](/development/blog/seo/2026/03/19/nodejs-abortcontroller-timeout-cancellation-pattern-guide.html)
- [Node.js 캐시 스탬피드 방지 가이드: Redis와 Singleflight로 트래픽 폭주 줄이기](/development/blog/seo/2026/03/20/nodejs-cache-stampede-prevention-redis-singleflight-guide.html)

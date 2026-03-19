---
layout: post
title: "Node.js 캐시 스탬피드 방지 가이드: Redis Singleflight로 트래픽 급증에도 안정성 지키기"
date: 2026-03-20 08:00:00 +0900
lang: ko
translation_key: nodejs-cache-stampede-prevention-redis-singleflight-guide
permalink: /development/blog/seo/2026/03/20/nodejs-cache-stampede-prevention-redis-singleflight-guide.html
alternates:
  ko: /development/blog/seo/2026/03/20/nodejs-cache-stampede-prevention-redis-singleflight-guide.html
  x_default: /development/blog/seo/2026/03/20/nodejs-cache-stampede-prevention-redis-singleflight-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, redis, caching, performance, backend, reliability]
description: "Node.js와 Redis 환경에서 캐시 스탬피드가 왜 발생하는지, singleflight·TTL jitter·stale-while-revalidate 패턴으로 DB 급증과 응답 지연을 줄이는 방법을 실무 예시와 함께 정리했습니다."
---

트래픽이 갑자기 몰릴 때 캐시가 있다고 해서 항상 안전한 것은 아닙니다.
오히려 **같은 키의 캐시가 동시에 만료되는 순간**, 수백 개 요청이 한꺼번에 DB나 외부 API로 쏠리면서 장애가 시작되기도 합니다.
이 글에서는 Node.js와 Redis 환경에서 자주 만나는 **캐시 스탬피드(cache stampede)** 문제를 어떻게 줄일지, 실무에서 바로 적용하기 쉬운 패턴 중심으로 정리합니다.

## 캐시 스탬피드가 왜 위험한가

### H3. 캐시 미스가 한 번이 아니라 동시에 폭발한다

평소에는 캐시 덕분에 응답 시간이 안정적이어도, 인기 상품 상세·메인 피드·랭킹 데이터처럼 많은 사용자가 같은 키를 보는 구조에서는 문제가 달라집니다.
캐시 TTL이 끝나는 순간 첫 요청 하나만 원본 데이터를 다시 가져오면 좋겠지만, 실제로는 같은 시점에 들어온 요청들이 모두 캐시 미스로 판단할 수 있습니다.

그 결과 아래 현상이 겹칩니다.

- 같은 SQL이나 외부 API 호출이 짧은 시간에 중복 실행된다
- DB connection pool이 빠르게 고갈된다
- 업스트림 지연이 다시 애플리케이션 타임아웃으로 번진다
- 캐시가 장애 완충재가 아니라 장애 증폭기가 된다

### H3. 문제는 평균 응답속도보다 피크 구간에서 드러난다

캐시 스탬피드는 평소 벤치마크에서는 잘 안 보입니다.
낮은 QPS에서는 몇 번 중복 호출이 발생해도 티가 나지 않지만, 트래픽이 몰리는 순간에는 같은 키 하나가 병목점이 됩니다.
그래서 이 문제는 단순 캐시 도입 여부보다 **만료 시점의 동시성 제어**를 어떻게 하느냐가 핵심입니다.

## Node.js에서 먼저 적용할 3가지 방어선

### H3. 첫 번째는 singleflight로 같은 계산을 한 번만 수행하게 만드는 것

가장 실용적인 기본 패턴은 **동일 키에 대한 재계산을 하나로 합치는 것**입니다.
Go 진영에서는 `singleflight`라는 이름으로 많이 알려져 있고, Node.js에서도 같은 아이디어를 쉽게 구현할 수 있습니다.

```ts
const inFlight = new Map<string, Promise<any>>();

async function singleflight<T>(key: string, loader: () => Promise<T>): Promise<T> {
  const existing = inFlight.get(key);
  if (existing) return existing;

  const promise = loader().finally(() => {
    inFlight.delete(key);
  });

  inFlight.set(key, promise);
  return promise;
}
```

이렇게 해두면 애플리케이션 인스턴스 하나 안에서는 같은 키에 대한 동시 요청이 들어와도 실제 원본 조회는 한 번만 실행됩니다.
작은 규모라면 이것만으로도 DB 급증을 꽤 줄일 수 있습니다.

### H3. 두 번째는 TTL jitter로 만료 시점을 분산하는 것

같은 종류의 캐시 키를 동일 TTL로 저장하면 특정 시점에 대량 만료가 겹치기 쉽습니다.
그래서 실무에서는 TTL에 약간의 랜덤 값을 더해 만료 시점을 분산합니다.

```ts
function withJitter(baseTtlSec: number, jitterSec = 30) {
  return baseTtlSec + Math.floor(Math.random() * jitterSec);
}

await redis.set(cacheKey, JSON.stringify(payload), {
  EX: withJitter(300, 45),
});
```

이 패턴의 장점은 단순합니다.
구현 비용이 거의 없는데도, 다수 키가 한꺼번에 사라지는 상황을 꽤 완화합니다.
특히 목록형 캐시나 집계성 캐시에서 효과가 좋습니다.

### H3. 세 번째는 stale-while-revalidate로 오래된 값을 잠깐 허용하는 것

무조건 최신 값만 반환하려고 하면 캐시 만료 순간마다 원본 저장소가 긴장하게 됩니다.
반면 사용자 경험상 수 초 정도의 지연된 최신성은 허용 가능한 경우가 많습니다.
이럴 때는 **stale-while-revalidate** 패턴이 잘 맞습니다.

핵심 아이디어는 이렇습니다.

1. 캐시가 신선하면 바로 반환한다
2. 약간 오래됐지만 허용 가능한 구간이면 오래된 값을 먼저 반환한다
3. 백그라운드에서 새 값을 다시 채운다

이 방식은 응답 지연을 낮추고, 피크 타임에 원본 저장소를 보호하는 데 도움이 됩니다.
다만 가격, 재고, 결제 가능 여부처럼 최신성이 아주 중요한 데이터에는 신중하게 적용해야 합니다.

## Redis 환경에서 많이 쓰는 구현 패턴

### H3. 분산 락은 짧고 단순하게 유지한다

애플리케이션 인스턴스가 여러 대라면 메모리 기반 singleflight만으로는 부족합니다.
이때 Redis 기반 락을 함께 써서 **클러스터 전체에서 한 인스턴스만 재생성**하도록 만들 수 있습니다.

```ts
import crypto from 'node:crypto';

async function refreshWithLock(cacheKey: string, loader: () => Promise<any>) {
  const lockKey = `lock:${cacheKey}`;
  const lockValue = crypto.randomUUID();

  const acquired = await redis.set(lockKey, lockValue, {
    NX: true,
    PX: 5000,
  });

  if (!acquired) {
    return null;
  }

  try {
    const fresh = await loader();
    await redis.set(cacheKey, JSON.stringify(fresh), { EX: withJitter(300, 45) });
    return fresh;
  } finally {
    const current = await redis.get(lockKey);
    if (current === lockValue) {
      await redis.del(lockKey);
    }
  }
}
```

여기서 중요한 건 락을 만능으로 보지 않는 것입니다.
락 시간이 너무 길면 오히려 대기열이 길어지고, 락 해제 로직이 느슨하면 경쟁 상태가 생길 수 있습니다.
그래서 락은 짧고, 재계산 함수는 멱등적으로 유지하는 편이 안전합니다.

### H3. 실패 시에는 빈 응답보다 마지막 정상값이 낫다

원본 조회가 실패했다고 캐시도 비워버리면 장애가 더 커집니다.
실무에서는 아래처럼 접근하는 편이 낫습니다.

- 최근 정상 캐시가 있으면 짧은 시간이라도 우선 반환한다
- 재생성 실패 횟수를 메트릭으로 남긴다
- 업스트림 장애가 길어지면 fallback 정책을 분리한다

즉, 캐시는 단순 성능 도구가 아니라 **장애 완화 레이어**로 다뤄야 합니다.

## 실전 예시: 캐시 조회 흐름 정리

### H3. 기본 흐름은 hit → stale 허용 → singleflight → 락 순서로 생각하면 된다

아래는 운영에서 이해하기 쉬운 흐름입니다.

```ts
async function getProductDetail(productId: string) {
  const cacheKey = `product:${productId}`;
  const cached = await redis.get(cacheKey);

  if (cached) {
    const parsed = JSON.parse(cached);

    if (parsed.expiresAt > Date.now()) {
      return parsed.data;
    }

    if (parsed.staleUntil > Date.now()) {
      void singleflight(cacheKey, async () => {
        await refreshWithLock(cacheKey, async () => {
          const fresh = await fetchProductDetail(productId);
          return {
            data: fresh,
            expiresAt: Date.now() + 60_000,
            staleUntil: Date.now() + 180_000,
          };
        });
      });

      return parsed.data;
    }
  }

  return singleflight(cacheKey, async () => {
    const fresh = await fetchProductDetail(productId);
    const payload = {
      data: fresh,
      expiresAt: Date.now() + 60_000,
      staleUntil: Date.now() + 180_000,
    };

    await redis.set(cacheKey, JSON.stringify(payload), { EX: withJitter(180, 30) });
    return payload.data;
  });
}
```

이 구조의 장점은 단순합니다.
요청 경로가 짧고, 신선도 정책과 동시성 제어가 코드상으로 구분돼 있어 운영 중 읽기 쉽습니다.

## 도입할 때 자주 놓치는 점

### H3. 캐시 키 설계가 흐리면 보호 효과도 약해진다

같은 데이터를 서로 다른 키 네이밍으로 저장하면 singleflight도, 락도, 지표도 분산됩니다.
그래서 아래 기준을 먼저 정리하는 것이 좋습니다.

- 어떤 단위까지 같은 데이터로 볼 것인가
- 사용자별 캐시와 공용 캐시를 어떻게 나눌 것인가
- 무효화 이벤트가 발생했을 때 어떤 키를 지울 것인가

캐시 스탬피드 대응은 코드 한 줄보다 **키 모델링 품질**에 더 크게 좌우됩니다.

### H3. 메트릭 없이는 개선 여부를 판단하기 어렵다

적용 후에는 반드시 수치로 확인해야 합니다.
최소한 아래 항목은 보는 편이 좋습니다.

- cache hit ratio
- stale response ratio
- lock acquisition success/failure
- loader execution count
- DB QPS와 p95 응답시간 변화

의도는 "캐시를 넣었다"가 아니라, **피크 타임의 원본 호출 수를 실제로 줄였는가**를 검증하는 데 있습니다.

## 배포 전 체크리스트

### H3. 코드와 운영 기준 점검

- 같은 키에 대한 재계산이 singleflight 또는 락으로 합쳐지는가
- TTL이 모두 동일하게 고정되지 않고 jitter가 적용되는가
- stale 허용 범위가 비즈니스 특성과 맞는가
- Redis 장애 시 fallback 동작과 타임아웃이 정리돼 있는가
- 코드 예시에 토큰, 내부 도메인, 민감정보가 포함되지 않았는가

### H3. SEO·콘텐츠 점검

- 제목에 `Node.js 캐시 스탬피드 방지`, `Redis`, `Singleflight` 의도가 자연스럽게 들어갔는가
- 첫 단락에서 문제와 실무 가치를 바로 설명하는가
- H2/H3 구조가 명확하고 짧은 문단으로 읽히는가
- 과장 표현 없이 재현 가능한 예시를 포함하는가

## 요약

캐시 스탬피드는 "캐시가 있으니 괜찮다"는 가정을 가장 쉽게 무너뜨리는 문제입니다.
실무에서는 거창한 솔루션보다도 **singleflight, TTL jitter, stale-while-revalidate** 같은 기본기를 조합하는 편이 효과적입니다.
Node.js와 Redis 조합이라면 이 세 가지 패턴만 잘 적용해도 피크 구간의 DB 급증, 응답 지연, 연쇄 타임아웃을 꽤 안정적으로 줄일 수 있습니다.

## 내부 링크

- [Node.js AbortController 실전 가이드: 타임아웃·취소 표준화로 장애 반경 줄이기](/development/blog/seo/2026/03/19/nodejs-abortcontroller-timeout-cancellation-pattern-guide.html)
- [Node.js 스트림 백프레셔 실전 가이드: 메모리 급증과 처리 지연 줄이기](/development/blog/seo/2026/03/18/nodejs-stream-backpressure-memory-spike-prevention-guide.html)
- [Node.js Rate Limiting 실전 가이드: Express와 Redis로 남용 트래픽 막기](/development/blog/seo/2026/03/12/rate-limiting-nodejs-express-redis-abuse-protection-guide.html)

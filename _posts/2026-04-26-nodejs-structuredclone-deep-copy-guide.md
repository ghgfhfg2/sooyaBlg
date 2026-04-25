---
layout: post
title: "Node.js structuredClone 가이드: JSON.parse(JSON.stringify()) 대신 안전하게 깊은 복사하는 법"
date: 2026-04-26 08:00:00 +0900
lang: ko
translation_key: nodejs-structuredclone-deep-copy-guide
permalink: /development/blog/seo/2026/04/26/nodejs-structuredclone-deep-copy-guide.html
alternates:
  ko: /development/blog/seo/2026/04/26/nodejs-structuredclone-deep-copy-guide.html
  x_default: /development/blog/seo/2026/04/26/nodejs-structuredclone-deep-copy-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, structuredClone, deep-copy, javascript, backend, data-serialization, performance]
description: "Node.js structuredClone으로 객체를 안전하게 깊은 복사하는 방법, JSON 기반 복사와의 차이, 타입 보존, 성능·주의사항을 실무 예제와 함께 정리했습니다."
---

Node.js에서 객체를 깊은 복사해야 할 때 아직도 `JSON.parse(JSON.stringify(obj))`를 먼저 떠올리는 경우가 많습니다.
하지만 이 방식은 `Date`, `Map`, `Set`, `undefined`, 순환 참조 같은 케이스에서 쉽게 깨집니다.
실무에서는 설정 스냅샷, 작업 payload 복제, 캐시 값 방어적 복사처럼 **복사 정확도와 타입 보존이 중요한 순간**이 자주 나오기 때문에, 더 안전한 기본값이 필요합니다.

이럴 때 먼저 검토할 만한 선택지가 `structuredClone()`입니다.
결론부터 말하면 Node.js의 `structuredClone()`은 **브라우저와 비슷한 구조화 복제 알고리즘으로 다양한 내장 타입을 더 안전하게 깊은 복사**하게 도와줍니다.
다만 모든 값을 복사할 수 있는 만능 함수는 아니고, 큰 객체를 무심코 복제하면 CPU와 메모리 비용이 커질 수 있습니다.

## Node.js에서 깊은 복사가 자주 문제 되는 이유

### H3. JSON 기반 복사는 간단하지만 타입을 잃어버리기 쉽다

가장 흔한 패턴은 아래와 같습니다.

```js
const cloned = JSON.parse(JSON.stringify(original));
```

짧고 익숙해서 편해 보이지만, 실제로는 아래 문제가 자주 생깁니다.

- `Date`가 문자열로 바뀜
- `Map`, `Set`이 기대한 구조를 유지하지 못함
- `undefined`, `Infinity`, `RegExp` 같은 값이 손실되거나 왜곡됨
- 순환 참조 객체에서 예외 발생
- 클래스 인스턴스의 의도가 흐려짐

즉 “대충 평범한 JSON 데이터”라면 지나갈 수 있어도, 백엔드 실무 객체는 생각보다 훨씬 복잡합니다.

### H3. 방어적 복사가 필요한 지점은 의외로 많다

Node.js 서비스에서는 아래 상황에서 깊은 복사가 자주 필요합니다.

- 요청 payload를 후속 처리 전에 스냅샷으로 남길 때
- 공통 설정 객체를 작업별로 안전하게 분기할 때
- 캐시에서 꺼낸 값을 수정하지 못하게 복사할 때
- 워커 스레드나 큐 작업으로 넘기기 전 데이터 형태를 고정할 때

이때 복사가 부정확하면 원본이 오염되거나, 나중에 직렬화 단계에서 예상 못 한 버그가 발생합니다.

## structuredClone은 무엇이 다른가

### H3. 더 많은 내장 타입과 순환 참조를 안전하게 다룬다

`structuredClone()`은 구조화 복제 알고리즘을 사용합니다.
JSON 복사와 비교하면 아래 차이가 실무에서 크게 체감됩니다.

```js
const original = {
  createdAt: new Date(),
  tags: new Set(['node', 'clone']),
  meta: new Map([['retries', 3]]),
};

original.self = original;

const cloned = structuredClone(original);

console.log(cloned.createdAt instanceof Date); // true
console.log(cloned.tags instanceof Set); // true
console.log(cloned.meta instanceof Map); // true
console.log(cloned.self === cloned); // true
```

즉 `structuredClone()`은 단순 문자열 왕복보다 **데이터 의미를 더 잘 유지하는 복사**에 가깝습니다.
순환 참조까지 다룰 수 있다는 점도 운영 코드에서는 꽤 중요합니다.

### H3. 그래도 함수와 일부 특수 객체는 복사 대상이 아니다

여기서 많이 오해하는 부분이 있습니다.
`structuredClone()`이 더 낫다고 해서 무엇이든 복사되는 것은 아닙니다.
예를 들어 함수는 복제할 수 없습니다.
일부 런타임 객체도 그대로 복사되지 않거나 예외가 발생할 수 있습니다.

```js
const config = {
  retry: 3,
  onError: () => console.error('fail'),
};

const cloned = structuredClone(config); // DataCloneError
```

따라서 `structuredClone()`은 “아무 객체나 깊은 복사”가 아니라, **구조화 복제가 가능한 데이터 중심 객체를 안전하게 복제하는 도구**로 이해하는 편이 정확합니다.

## 언제 structuredClone을 쓰는 게 좋은가

### H3. 설정 스냅샷이나 작업 입력값을 보호하고 싶을 때

여러 비동기 작업이 같은 객체를 공유할 때는 의도치 않은 변경이 자주 문제를 만듭니다.
이럴 때 작업 시작 시점에 복사본을 떠 두면 디버깅이 쉬워집니다.

```js
function buildJobPayload(requestBody, defaults) {
  const payload = structuredClone(defaults);

  payload.userId = requestBody.userId;
  payload.filters = structuredClone(requestBody.filters);

  return payload;
}
```

이 패턴의 장점은 “원본 보호”가 명확하다는 점입니다.
특히 공통 옵션 객체를 작업마다 조금씩 수정하는 코드에서 실수가 줄어듭니다.

### H3. 캐시나 공용 상태를 읽기 전용처럼 다루고 싶을 때

캐시에서 꺼낸 객체를 그대로 내려보냈다가, 호출한 쪽에서 값을 수정해 버리는 문제도 흔합니다.
이때 매번 원본을 직접 노출하지 않고 복사해서 넘기면 상태 오염을 줄일 수 있습니다.

다만 복사 비용은 분명히 존재합니다.
객체가 아주 크다면 매 요청마다 `structuredClone()`을 쓰는 방식이 병목이 될 수 있습니다.
이런 비용은 [Node.js Event Loop Lag 모니터링 가이드](/development/blog/seo/2026/04/13/nodejs-event-loop-lag-monitoring-guide.html)에서 다룬 것처럼 이벤트 루프 지연으로 이어질 수 있으니, 보호가 정말 필요한 경계에서만 쓰는 편이 좋습니다.

## JSON 복사보다 나은 점과 여전히 조심할 점

### H3. 타입 보존이 좋아도 성능 비용은 공짜가 아니다

`structuredClone()`은 편리하지만 결국 깊은 복사입니다.
즉 객체가 크고 중첩이 깊을수록 CPU와 메모리를 사용합니다.
특히 아래 상황에서는 주의가 필요합니다.

- 큰 배열이나 큰 중첩 객체를 요청마다 복제하는 경우
- 핫패스에서 캐시 값을 계속 복제하는 경우
- 대용량 바이너리나 버퍼를 반복 복제하는 경우
- 복사 후 곧바로 버리는 임시 객체가 많은 경우

이런 패턴은 메모리 사용량을 빠르게 키울 수 있습니다.
운영 중 힙이 자주 불어나면 [Node.js 메모리 누수·heapdump·Clinic.js 가이드](/development/blog/seo/2026/03/11/nodejs-memory-leak-heapdump-clinicjs-guide.html)처럼 실제 메모리 프로파일로 확인하는 편이 안전합니다.

### H3. 클래스 인스턴스는 “데이터”와 “행동”을 분리해서 보는 편이 낫다

실무에서 헷갈리는 또 다른 지점은 클래스 인스턴스입니다.
`structuredClone()`은 데이터 복제에는 유용하지만, 메서드와 프로토타입 의미까지 그대로 보존하는 용도로 기대하면 실망하기 쉽습니다.

그래서 아래 기준이 현실적입니다.

- 단순 데이터 전달: `structuredClone()` 적합
- 도메인 객체 행동 보존: 생성자/팩토리로 다시 만들기
- 라이브 리소스 핸들 보존: 복제보다 참조 관리 방식 재설계

즉 클래스 인스턴스 전체를 통째로 복사하려 하기보다, **복제 가능한 순수 데이터로 경계를 정리**하는 편이 장기적으로 유지보수에 유리합니다.

## worker_threads와 함께 볼 때 중요한 포인트

### H3. 큰 데이터를 자주 복제하면 워커 오프로딩 이점이 줄어든다

워커 스레드를 쓸 때도 복제 비용은 중요합니다.
메인 스레드에서 큰 객체를 준비해 워커로 보내는 순간, 데이터 이동 자체가 부담이 될 수 있기 때문입니다.

그래서 CPU 작업을 워커로 넘길 때는 “연산 비용”뿐 아니라 “전달 비용”도 같이 봐야 합니다.
관련해서는 [Node.js worker_threads 가이드](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)처럼 워커 사용 기준을 함께 보는 편이 좋습니다.
연산보다 복제 비용이 더 크면 기대한 성능 이득이 잘 안 나올 수 있습니다.

### H3. 복사가 필요 없는 구조가 더 좋은 답일 때도 많다

깊은 복사는 버그를 막는 쉬운 해법이지만, 항상 최선은 아닙니다.
아래처럼 구조를 바꾸면 아예 복제가 덜 필요할 수 있습니다.

- 변경 가능한 공용 객체를 없애기
- 함수 입력을 작고 명확한 데이터로 축소하기
- 캐시 값을 읽기 전용 규칙으로 관리하기
- 필요한 필드만 선택해 새 객체 만들기

즉 `structuredClone()`은 좋은 도구지만, **불필요한 공유 상태를 설계로 줄이는 것**이 더 근본적인 해결일 때가 많습니다.

## 자주 하는 실수

### H3. structuredClone을 무조건 안전한 기본값으로 남용하는 경우

“일단 복사하고 보자”는 식으로 핫패스에 넣으면, 장애는 줄었는데 응답 시간이 나빠질 수 있습니다.
복사는 숨은 비용이 커서, 특히 트래픽이 많을수록 체감됩니다.

안전한 기준은 이렇습니다.

- 경계 보호가 필요한 곳에만 사용
- 큰 객체 전체보다 필요한 필드만 선별
- 성능 민감 구간에서는 측정 후 도입

### H3. 함수가 섞인 설정 객체를 그대로 복제하려는 경우

설정 객체 안에 콜백, 로거, 클래스 인스턴스, 연결 핸들 같은 값이 섞이면 `structuredClone()`이 기대대로 동작하지 않을 수 있습니다.
이 경우에는 복제 대상과 실행 대상 구성을 분리하는 편이 낫습니다.

예를 들면 아래처럼 나누는 방식이 더 안전합니다.

- 복제 가능한 데이터 설정
- 런타임 핸들러/함수 레지스트리
- 외부 리소스 참조

### H3. 복제 후 원본과 의미가 완전히 같다고 가정하는 경우

깊은 복사는 값 분리를 돕지만, 모든 런타임 의미를 그대로 들고 가지는 않습니다.
따라서 “같은 모양의 데이터”와 “같은 의미의 객체”를 구분해야 합니다.
이 차이를 놓치면 테스트에서는 지나가는데 운영에서만 이상한 동작이 나올 수 있습니다.

## 도입 전 체크리스트

### H3. 아래 항목에 많이 해당하면 structuredClone이 잘 맞는다

- `JSON.parse(JSON.stringify())` 한계를 자주 만나는가?
- `Date`, `Map`, `Set`, 순환 참조를 다뤄야 하는가?
- 요청 payload나 설정 객체를 방어적으로 복사해야 하는가?
- 공용 상태 오염 때문에 디버깅이 어려웠는가?
- 복사 비용을 감당할 만큼 경계 보호 이점이 분명한가?

## 마무리

Node.js의 `structuredClone()`은 깊은 복사 문제를 훨씬 덜 위험하게 만들어 주는 좋은 기본 도구입니다.
특히 JSON 기반 복사에서 자주 깨지는 타입과 순환 참조를 더 안전하게 다룰 수 있다는 점이 큽니다.

다만 이것을 “아무 때나 써도 되는 공짜 복사”로 이해하면 곤란합니다.
복제 가능 대상인지 확인하고, 큰 객체에서는 성능 비용을 측정하고, 가능하면 애초에 공유 상태를 줄이는 방향으로 설계하는 편이 더 좋습니다.

함께 보면 좋은 글:

- [Node.js worker_threads 가이드](/development/blog/seo/2026/03/18/nodejs-worker-threads-cpu-bound-performance-guide.html)
- [Node.js Event Loop Lag 모니터링 가이드](/development/blog/seo/2026/04/13/nodejs-event-loop-lag-monitoring-guide.html)
- [Node.js 메모리 누수·heapdump·Clinic.js 가이드](/development/blog/seo/2026/03/11/nodejs-memory-leak-heapdump-clinicjs-guide.html)

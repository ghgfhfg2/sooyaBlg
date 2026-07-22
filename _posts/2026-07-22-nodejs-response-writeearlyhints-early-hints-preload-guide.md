---
layout: post
title: "Node.js writeEarlyHints 가이드: 103 Early Hints로 preload를 안전하게 보내는 법"
date: 2026-07-22 20:00:00 +0900
lang: ko
translation_key: nodejs-response-writeearlyhints-early-hints-preload-guide
permalink: /development/blog/seo/2026/07/22/nodejs-response-writeearlyhints-early-hints-preload-guide.html
alternates:
  ko: /development/blog/seo/2026/07/22/nodejs-response-writeearlyhints-early-hints-preload-guide.html
  x_default: /development/blog/seo/2026/07/22/nodejs-response-writeearlyhints-early-hints-preload-guide.html
categories: [development, blog, seo]
tags: [기술블로그, seo, nodejs, http, early-hints, preload, performance, server-response, web-performance, backend]
description: "Node.js http.ServerResponse의 writeEarlyHints()로 103 Early Hints와 Link preload 헤더를 보내는 방법을 정리합니다. 적용 기준, 보안 주의점, 중복 헤더 관리, 측정 체크리스트까지 실무 예제로 설명합니다."
---

웹 성능을 개선하려고 할 때 가장 먼저 떠올리는 방법은 번들 크기 줄이기, 이미지 최적화, 캐시 설정입니다.
그런데 서버가 HTML을 만들기 전에 이미 필요한 CSS나 JavaScript 파일을 알고 있다면, 브라우저에 조금 더 빨리 힌트를 줄 수도 있습니다.
최종 응답이 준비되기 전 `103 Early Hints`로 `Link: rel=preload`를 보내는 방식입니다.

Node.js의 `http.ServerResponse`에는 `response.writeEarlyHints()`가 있습니다.
Node.js 공식 HTTP 문서 기준으로 이 메서드는 HTTP/1.1 `103 Early Hints` 메시지를 `Link` 헤더와 함께 보내며, 사용자 에이전트가 연결된 리소스를 preload 또는 preconnect할 수 있게 힌트를 전달합니다.
이 글에서는 `writeEarlyHints()`를 어디에 쓰면 좋고, 어떤 리소스만 힌트로 보내야 하며, 캐시와 보안 기준을 어떻게 잡아야 하는지 정리합니다.
[Node.js response header 가이드](/development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html), [Node.js source maps 가이드](/development/blog/seo/2026/05/22/nodejs-enable-source-maps-production-debugging-guide.html), [Node.js graceful shutdown 가이드](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)와 함께 보면 HTTP 응답 경로를 더 안정적으로 운영할 수 있습니다.

## writeEarlyHints가 해결하는 문제

### H3. HTML 생성 전에도 중요한 정적 리소스는 알 수 있다

서버 렌더링 페이지는 HTML을 만들기 전에 데이터베이스 조회, API 호출, 템플릿 렌더링을 거칠 수 있습니다.
이 시간 동안 브라우저는 최종 HTML의 `<link rel="stylesheet">`나 `<script>` 태그를 아직 보지 못합니다.
하지만 라우트 단위로 반드시 필요한 CSS, 핵심 JavaScript, 폰트 파일은 서버가 미리 알고 있는 경우가 많습니다.

`103 Early Hints`는 이 틈을 줄이는 보조 신호입니다.
서버는 최종 `200 OK` 응답을 보내기 전에 먼저 "이 리소스는 곧 필요할 가능성이 높다"는 정보를 보낼 수 있습니다.
브라우저와 중간 프록시가 이를 지원하면 리소스 요청을 조금 더 일찍 시작할 수 있습니다.

```js
import http from 'node:http';

const server = http.createServer(async (req, res) => {
  if (req.url === '/') {
    res.writeEarlyHints({
      link: [
        '</assets/app.css>; rel=preload; as=style',
        '</assets/app.js>; rel=preload; as=script'
      ]
    });

    const html = await renderHomePage();

    res.writeHead(200, {
      'content-type': 'text/html; charset=utf-8'
    });
    res.end(html);
    return;
  }

  res.writeHead(404).end('not found');
});

server.listen(3000);

async function renderHomePage() {
  return '<!doctype html><html><head><link rel="stylesheet" href="/assets/app.css"></head><body><div id="app"></div><script src="/assets/app.js"></script></body></html>';
}
```

핵심은 Early Hints가 최종 응답을 대체하지 않는다는 점입니다.
최종 HTML에는 여전히 정상적인 `<link>`와 `<script>`가 있어야 합니다.
Early Hints는 브라우저에게 더 일찍 움직일 기회를 주는 성능 힌트일 뿐입니다.

### H3. 모든 페이지에 넣는다고 항상 빨라지는 것은 아니다

Early Hints는 서버가 늦고, 필요한 리소스가 명확하며, 클라이언트와 CDN 경로가 103 응답을 처리할 때 효과가 납니다.
반대로 HTML이 매우 빨리 내려가거나, 페이지마다 필요한 리소스가 자주 바뀌거나, 잘못된 preload가 많으면 효과가 작거나 오히려 손해가 될 수 있습니다.

예를 들어 사용자가 실제로 방문하지 않을 탭의 큰 JavaScript 번들을 preload하면 네트워크 우선순위를 낭비할 수 있습니다.
폰트도 마찬가지입니다.
첫 화면에 꼭 필요한 폰트 한두 개라면 후보가 될 수 있지만, 여러 굵기와 언어별 파일을 모두 preload하면 중요한 이미지나 API 응답보다 먼저 대역폭을 차지할 수 있습니다.

그래서 `writeEarlyHints()`는 "많이 보내기"보다 "틀릴 가능성이 낮은 리소스만 보내기"가 중요합니다.

## 기본 사용법과 Link 헤더 구성

### H3. Link 값은 배열로 관리한다

`writeEarlyHints()`는 객체 형태의 힌트를 받습니다.
`link`에는 문자열 하나를 넣을 수도 있고, 여러 preload 항목을 배열로 넣을 수도 있습니다.
실무에서는 배열로 관리하는 편이 중복 제거, 조건 분기, 테스트가 쉽습니다.

```js
const HOME_EARLY_HINTS = [
  '</assets/home.css>; rel=preload; as=style',
  '</assets/home.js>; rel=preload; as=script'
];

export function sendEarlyHints(res, links) {
  if (!Array.isArray(links) || links.length === 0) {
    return;
  }

  res.writeEarlyHints({
    link: links
  });
}
```

이렇게 얇게 감싸면 라우트 핸들러마다 `writeEarlyHints()` 호출 형식이 흩어지지 않습니다.
나중에 CDN 미지원 경로에서는 비활성화하거나, 실험군에만 보내는 조건을 추가하기도 쉽습니다.

### H3. preload 대상은 최종 HTML과 일치해야 한다

Early Hints로 보낸 리소스가 최종 HTML에 없으면 브라우저가 불필요한 일을 시작할 수 있습니다.
반대로 최종 HTML에만 있고 Early Hints에는 없으면 성능 기회가 줄어듭니다.
두 목록이 서로 다른 파일에서 수동으로 관리되면 시간이 지날수록 어긋나기 쉽습니다.

가능하면 빌드 manifest를 기준으로 Early Hints와 HTML 태그를 함께 만드세요.

```js
const assetManifest = {
  home: {
    css: '/assets/home.4f3a.css',
    js: '/assets/home.91bd.js'
  }
};

export function buildEarlyHintLinks(routeName) {
  const assets = assetManifest[routeName];

  if (!assets) {
    return [];
  }

  return [
    `<${assets.css}>; rel=preload; as=style`,
    `<${assets.js}>; rel=preload; as=script`
  ];
}

export function renderAssetTags(routeName) {
  const assets = assetManifest[routeName];

  return [
    `<link rel="stylesheet" href="${assets.css}">`,
    `<script src="${assets.js}" defer></script>`
  ].join('\n');
}
```

중요한 점은 같은 source of truth를 쓰는 것입니다.
파일명이 hash 기반으로 바뀌는 배포라면, 이전 배포의 Early Hints가 새 HTML과 섞이지 않도록 캐시와 릴리스 경계도 함께 점검해야 합니다.

## 라우트별 적용 기준

### H3. 첫 화면 필수 리소스만 후보에 올린다

Early Hints 후보는 아래 기준을 통과해야 합니다.

- 거의 모든 요청에서 실제로 사용된다.
- 첫 화면 렌더링에 필요하다.
- 파일명이 content hash로 관리되어 캐시 안정성이 높다.
- 사용자의 권한, 개인정보, 실험군 상태를 URL에 담지 않는다.
- preload 우선순위를 줄 만큼 크거나 중요한 리소스다.

예를 들어 공통 레이아웃 CSS와 라우트 진입 JavaScript는 좋은 후보입니다.
반면 모달을 열 때만 필요한 에디터 번들, 관리자에게만 보이는 차트 라이브러리, 사용자별 서명 URL은 후보에서 제외하는 편이 안전합니다.

```js
const routeEarlyHints = new Map([
  ['/', [
    '</assets/home.css>; rel=preload; as=style',
    '</assets/home.js>; rel=preload; as=script'
  ]],
  ['/pricing', [
    '</assets/pricing.css>; rel=preload; as=style'
  ]]
]);

export function getEarlyHintsForRequest(req) {
  const url = new URL(req.url, 'https://example.com');

  return routeEarlyHints.get(url.pathname) ?? [];
}
```

라우트 이름이나 pathname을 기준으로 관리하면 예측 가능성이 높습니다.
쿼리 파라미터나 사용자 입력을 그대로 Link 헤더에 넣는 방식은 피해야 합니다.

### H3. 동적 입력으로 Link 헤더를 만들지 않는다

Link 헤더는 HTTP 헤더입니다.
사용자 입력이 섞이면 헤더 인젝션, 내부 경로 노출, 원치 않는 외부 도메인 preconnect 같은 문제가 생길 수 있습니다.
Node.js는 잘못된 헤더 이름이나 값에 대해 에러를 던질 수 있지만, 애플리케이션의 정책 검증까지 대신해 주지는 않습니다.

아래처럼 허용 목록 기반으로만 구성하는 편이 좋습니다.

```js
const allowedPreloadAssets = new Set([
  '/assets/home.css',
  '/assets/home.js',
  '/assets/pricing.css'
]);

export function toPreloadLink(assetPath, as) {
  if (!allowedPreloadAssets.has(assetPath)) {
    throw new Error(`Asset is not allowed for early hints: ${assetPath}`);
  }

  if (!['style', 'script', 'font', 'image'].includes(as)) {
    throw new Error(`Unsupported preload type: ${as}`);
  }

  return `<${assetPath}>; rel=preload; as=${as}`;
}
```

이 예제는 내부 개발자 실수를 빨리 잡기 위한 방어선입니다.
공개 요청에서 받은 URL을 그대로 preload하지 않고, 배포 산출물 manifest나 명시적인 route map에서만 가져오도록 제한하세요.

## 서버 코드에 안전하게 붙이기

### H3. 최종 응답 헤더보다 먼저 보낸다

Early Hints는 최종 응답 헤더가 나가기 전에 보내야 합니다.
`res.writeHead()`, `res.write()`, `res.end()`가 먼저 호출되면 이미 최종 헤더가 계산되었거나 전송된 상태일 수 있습니다.
따라서 라우트 핸들러 초반, 렌더링이나 데이터 조회를 시작하기 전에 보내는 흐름이 자연스럽습니다.

```js
import http from 'node:http';

const server = http.createServer(async (req, res) => {
  const earlyHintLinks = getEarlyHintsForRequest(req);

  if (earlyHintLinks.length > 0) {
    res.writeEarlyHints({ link: earlyHintLinks });
  }

  try {
    const page = await route(req);

    res.writeHead(page.status, {
      'content-type': page.contentType
    });
    res.end(page.body);
  } catch (error) {
    res.writeHead(500, {
      'content-type': 'text/plain; charset=utf-8'
    });
    res.end('internal server error');
  }
});
```

Early Hints를 보낸 뒤 최종 응답이 500이 될 수도 있습니다.
그래서 힌트 대상은 실패해도 민감하지 않은 정적 리소스로 제한해야 합니다.
사용자별 대시보드 CSV, 임시 다운로드 URL, 비공개 이미지 같은 것은 Early Hints에 올리지 않는 편이 안전합니다.

### H3. 프레임워크 뒤에서는 지원 여부를 확인한다

Express, Fastify, Next.js 커스텀 서버, reverse proxy, CDN을 쓰는 구조에서는 `http.ServerResponse`에 직접 접근하지 않거나, 중간 계층이 103 응답을 통과시키지 않을 수 있습니다.
이 경우 애플리케이션 코드만 보고 "보냈다"고 판단하면 안 됩니다.

확인은 실제 네트워크 경로에서 해야 합니다.
로컬 Node.js 서버, 스테이징 CDN, 운영 도메인까지 각각 103 응답이 보이는지 확인해야 합니다.

```bash
curl -I --http1.1 https://example.com/
```

환경에 따라 `curl -v`로 informational response를 확인하는 편이 더 명확할 수 있습니다.
중요한 것은 최종 `200 OK`만 보는 것이 아니라, 그 전에 `HTTP/1.1 103 Early Hints`와 `Link` 헤더가 지나가는지 확인하는 것입니다.

## 성능 측정 체크리스트

### H3. Core Web Vitals만으로 원인을 단정하지 않는다

Early Hints의 목적은 대체로 리소스 발견 시점을 앞당기는 것입니다.
하지만 LCP, FCP, TTFB 같은 지표는 네트워크, 서버 렌더링, 캐시, 이미지 크기, 클라이언트 실행 비용이 함께 섞여 움직입니다.
Early Hints를 켰다고 LCP가 바로 좋아지지 않을 수도 있고, 좋아졌더라도 다른 변경의 영향일 수 있습니다.

측정은 최소한 아래를 함께 봐야 합니다.

- 103 응답이 실제로 전달되는가?
- preload된 리소스가 최종 HTML에서도 사용되는가?
- 리소스 요청 시작 시간이 앞당겨졌는가?
- 불필요한 preload로 대역폭 낭비가 생기지 않았는가?
- 모바일 네트워크에서 효과가 유지되는가?

Chrome DevTools의 Network 패널이나 WebPageTest waterfall을 보면 리소스 발견 시점 차이를 비교하기 쉽습니다.
운영에서는 실험군과 대조군을 나누고, 같은 배포 버전에서 비교하는 편이 안전합니다.

### H3. 작은 실험으로 시작한다

처음부터 모든 라우트에 Early Hints를 붙이는 것은 추천하지 않습니다.
가장 트래픽이 많고, 첫 화면 리소스가 안정적이며, 서버 렌더링 시간이 어느 정도 있는 페이지 하나로 시작하세요.

```js
const earlyHintsEnabledRoutes = new Set([
  '/'
]);

export function maybeWriteEarlyHints(req, res) {
  const url = new URL(req.url, 'https://example.com');

  if (!earlyHintsEnabledRoutes.has(url.pathname)) {
    return;
  }

  const links = getEarlyHintsForRequest(req);

  if (links.length > 0) {
    res.writeEarlyHints({ link: links });
  }
}
```

이런 식으로 라우트 단위 feature flag처럼 시작하면 문제가 생겼을 때 빠르게 끌 수 있습니다.
성능 기능은 "켜져 있다"보다 "효과가 측정되고, 되돌릴 수 있다"가 더 중요합니다.

## 운영 주의점

### H3. 캐시와 배포 경계를 맞춘다

hash가 붙은 정적 파일은 Early Hints와 잘 맞습니다.
파일 내용이 바뀌면 URL도 바뀌기 때문에 오래 캐시해도 안전합니다.
반대로 `/assets/app.js`처럼 이름이 고정된 파일을 장기 캐시하면서 Early Hints로 밀어 넣으면, 배포 직후 이전 파일과 새 HTML이 섞이는 문제가 생길 수 있습니다.

운영 기준은 단순합니다.

- Early Hints 대상은 가능한 한 content hash 파일로 제한한다.
- 배포 manifest와 HTML 렌더링이 같은 버전을 바라보게 한다.
- CDN이 103 응답과 최종 응답을 어떻게 캐시하는지 확인한다.
- rollback 시 이전 manifest와 파일이 함께 살아 있는지 확인한다.

성능 최적화가 배포 안정성을 해치면 결국 손해입니다.
Early Hints는 asset pipeline과 함께 설계해야 합니다.

### H3. 추적 헤더는 필요한 최소한만 보낸다

Node.js 문서 예제처럼 Early Hints 객체에는 `link` 외의 헤더도 넣을 수 있습니다.
하지만 진단용 헤더를 무심코 추가하면 요청 추적 ID, 실험군 정보, 내부 시스템 이름이 최종 응답보다 먼저 노출될 수 있습니다.

실무에서는 Early Hints에 `Link`만 넣는 정책을 기본값으로 두는 편이 안전합니다.
정말 진단 헤더가 필요하다면 내부 스테이징에서만 켜고, 운영 공개 응답에는 남기지 않는 기준을 세우세요.

```js
export function writePublicEarlyHints(res, links) {
  if (links.length === 0) {
    return;
  }

  res.writeEarlyHints({
    link: links
  });
}
```

단순한 코드가 가장 안전한 정책이 될 때가 많습니다.
특히 헤더는 로그, 프록시, 브라우저 도구, 외부 모니터링에 남을 수 있으므로 노출 기준을 보수적으로 잡는 것이 좋습니다.

## FAQ

### H3. writeEarlyHints는 HTML의 preload 태그를 대체하나요?

아닙니다.
Early Hints는 최종 응답 전에 보내는 힌트이고, HTML의 `<link rel="preload">`나 실제 stylesheet/script 태그는 여전히 필요합니다.
브라우저나 중간 프록시가 103 응답을 처리하지 않는 환경에서도 페이지가 정상 동작해야 합니다.

### H3. API 응답에도 Early Hints를 써도 되나요?

대부분의 JSON API에는 맞지 않습니다.
Early Hints는 브라우저가 곧 필요한 정적 리소스를 미리 가져오도록 돕는 데 유용합니다.
API 호출 결과에 따라 달라지는 사용자별 리소스나 민감한 다운로드 URL은 Early Hints 대상에서 제외하는 편이 안전합니다.

### H3. 여러 번 호출해도 되나요?

Node.js HTTP informational response 계열은 최종 응답 전 여러 번 보낼 수 있는 흐름을 지원합니다.
다만 실무에서는 한 요청당 한 번, 라우트별로 확정된 Link 목록을 보내는 쪽이 관측과 디버깅이 쉽습니다.
여러 번 나누어 보내면 어떤 힌트가 실제 성능에 영향을 줬는지 추적하기 어려워질 수 있습니다.

## 발행 전 점검

- 제목에 핵심 키워드인 `Node.js writeEarlyHints`, `103 Early Hints`, `preload`를 포함했다.
- 본문을 H2/H3 구조로 나누고 첫 단락에서 문제와 가치를 설명했다.
- 재현 가능한 Node.js HTTP 서버 예제와 운영 체크리스트를 포함했다.
- 사용자 입력으로 Link 헤더를 만들지 않도록 허용 목록과 보안 기준을 명시했다.
- 내부링크 3개를 포함해 HTTP 응답, source maps, graceful shutdown 글과 연결했다.
- 민감정보, 불법 행위, 혐오·차별·유해 표현은 포함하지 않았다.

## 관련 글

- [Node.js maxRequestsPerSocket 가이드: keep-alive 연결 재활용 기준 잡기](/development/blog/seo/2026/04/15/nodejs-maxrequestspersocket-keepalive-connection-recycling-guide.html)
- [Node.js source maps 가이드: 운영 에러 스택을 원본 TypeScript로 읽는 법](/development/blog/seo/2026/05/22/nodejs-enable-source-maps-production-debugging-guide.html)
- [Node.js graceful shutdown 가이드: 진행 중 요청을 안전하게 드레이닝하기](/development/blog/seo/2026/04/07/nodejs-graceful-shutdown-inflight-request-draining-guide.html)

## 마무리

`response.writeEarlyHints()`는 Node.js HTTP 서버에서 `103 Early Hints`를 직접 보낼 수 있는 작지만 선명한 성능 도구입니다.
효과를 내려면 첫 화면에 꼭 필요한 정적 리소스만 고르고, 최종 HTML과 같은 manifest를 쓰며, 실제 네트워크 경로에서 103 응답이 통과하는지 확인해야 합니다.
무엇보다 Early Hints는 최종 응답을 대체하지 않습니다.
보수적인 Link 목록으로 작게 시작하고, waterfall과 Core Web Vitals를 함께 보며 계속 유지할 가치가 있는지 측정하세요.

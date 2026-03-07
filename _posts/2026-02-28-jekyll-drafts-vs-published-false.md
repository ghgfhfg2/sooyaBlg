---
layout: post
title: "Jekyll에서 초안과 비공개 글을 구분하는 방법"
date: 2026-02-28 20:00:00 +0900
lang: ko
translation_key: jekyll-drafts-vs-published-false
permalink: /jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
alternates:
  ko: /jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
  en: /en/jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
  ja: /ja/jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
  x_default: /jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
categories: [jekyll, blog, seo]
tags: [jekyll, drafts, published, github-pages, seo]
description: "Jekyll에서 _drafts와 published:false를 역할별로 분리해 공개 사고를 줄이고 운영 신뢰도를 지키는 기준을 정리합니다."
---

## 문제

Jekyll에서는 작성 단계와 배포 단계를 섞어 운영하면, 검수 전 글이 의도치 않게 노출되기 쉽습니다.
특히 `_drafts`와 `published: false`를 같은 용도로 쓰면 팀 내 기준이 흔들립니다.

## 설명

핵심은 두 기능의 역할을 분리하는 것입니다.

1. **`_drafts`**
   - 구조가 아직 불안정한 글, 메모성 초안
   - 기본 빌드/배포에 포함되지 않음
2. **`published: false` in `_posts`**
   - 글은 거의 완성됐지만 검수·승인 대기 상태
   - 배포 파이프라인에서 명시적으로 비공개 유지

운영 기준:
- 미완성 문서 → `_drafts`
- 발행 직전 검수 문서 → `_posts` + `published: false`
- 공개 승인 완료 → `_posts` + `published: true`(또는 생략)

## 요약

`_drafts`는 작성 안전장치, `published: false`는 배포 제어 스위치입니다.
역할을 나눠 쓰면 검색 노출 실수를 줄이고 블로그 신뢰도를 안정적으로 유지할 수 있습니다.

## 내부 링크 후보

- [개발 글 신뢰도를 높이는 에러 재현 정보 작성법](/development/blog/seo/2026/03/02/api-error-troubleshooting-context.html)
- [기술 블로그 신뢰도를 높이는 코드 예제 재현성 체크리스트](/development/blog/seo/2026/03/01/reproducible-code-example-checklist.html)

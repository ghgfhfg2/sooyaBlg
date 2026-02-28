---
layout: post
title: "Jekyll에서 초안과 비공개 글을 구분하는 방법"
date: 2026-02-28 20:00:00 +0900
categories: [jekyll, blog, seo]
tags: [jekyll, drafts, published, github-pages, seo]
description: "Jekyll에서 _drafts와 published:false를 어떻게 다르게 써야 하는지, 운영 실수 없이 블로그 신뢰도를 지키는 기준을 정리합니다."
---

## 문제

기술 블로그를 운영하다 보면 아직 검수되지 않은 글이 검색엔진에 노출되는 사고가 자주 발생합니다.  
특히 Jekyll에서는 `_drafts` 폴더와 `published: false` 설정을 혼용할 때 기준이 없으면, 의도와 다르게 글이 빌드되거나 배포될 수 있습니다.

## 설명

Jekyll에서 공개 제어는 보통 두 가지 축으로 관리합니다.

1. **`_drafts` 폴더**
   - 발행 전 초안을 두는 위치입니다.
   - 일반 빌드에서는 포함되지 않고, 로컬에서 `--drafts` 옵션을 사용할 때만 확인할 수 있습니다.

2. **front matter의 `published: false`**
   - 글이 `_posts`에 있더라도 발행 대상에서 제외할 수 있습니다.
   - 임시 비공개, 검수 대기, 예약 전 상태 관리에 유용합니다.

실무에서는 보통 아래 기준이 안전합니다.

- 아직 구조가 완성되지 않은 글: `_drafts`
- 내용은 완성됐지만 검수/승인 대기: `_posts` + `published: false`
- 최종 발행 승인 완료: `_posts` + `published: true`(또는 생략)

이렇게 구분하면 작성 단계와 배포 단계를 분리할 수 있어, 검색 노출 사고를 줄이는 데 도움이 됩니다.

## 예시

### 예시 1) 초안 단계

```text
_drafts/jekyll-draft-published-false-seo-trust.md
```

- 아이디어 정리, 문장 다듬기, 코드 샘플 정리 단계
- 아직 배포 파이프라인에 올리지 않음

### 예시 2) 발행 직전 검수 단계

```yaml
---
title: "배포 직전 체크리스트"
date: 2026-03-01 09:00:00 +0900
published: false
---
```

- `_posts`로 이동해도 `published: false`이면 공개되지 않음
- 맞춤법/링크/이미지/메타데이터 검수 후 공개 전환

## 요약

- `_drafts`는 작성 중인 문서를 안전하게 분리하는 공간입니다.
- `published: false`는 배포 직전 검수 상태를 명시하는 스위치입니다.
- 두 방식을 역할별로 나누면, 운영 실수를 줄이고 기술 블로그의 신뢰도를 안정적으로 유지할 수 있습니다.

---

### 내부 링크 후보

- `/private/test/비밀-테스트-글/` (비공개 상태 관리 예시 글)
- `/start/첫-글/` (기본 Jekyll 포스트 구조 참고)

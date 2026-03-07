---
layout: post
title: "Jekyll Drafts vs `published: false`: A Safe Publishing Workflow"
date: 2026-02-28 20:00:00 +0900
lang: en
translation_key: jekyll-drafts-vs-published-false
permalink: /en/jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
alternates:
  ko: /jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
  en: /en/jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
  ja: /ja/jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
  x_default: /jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
categories: [jekyll, blog, seo]
tags: [jekyll, drafts, publishing, github-pages, seo]
description: "Use `_drafts` for unfinished writing and `published: false` for pre-release review to reduce accidental exposure and improve editorial reliability."
---

## Problem

In Jekyll blogs, accidental publication usually happens when drafting and release controls are mixed together.

## Practical Split

1. **`_drafts`** for unfinished structure and rough writing
2. **`published: false` in `_posts`** for content-complete posts waiting for review

This split gives a clear handoff from writing to publishing.

## Recommended Policy

- Unfinished content → `_drafts`
- Ready but unapproved content → `_posts` + `published: false`
- Approved content → `_posts` with publish enabled

## Summary

`_drafts` is your writing safety zone, while `published: false` is your release gate. Using both by role reduces indexing mistakes and supports long-term SEO trust.

## Related reading

- [How to document API errors with reproducible context](/en/development/blog/seo/2026/03/02/api-error-troubleshooting-context.html)
- [Reproducible code example checklist](/en/development/blog/seo/2026/03/01/reproducible-code-example-checklist.html)

---
layout: post
title: "Jekyllで下書きと非公開投稿を分ける運用ルール"
date: 2026-02-28 20:00:00 +0900
lang: ja
translation_key: jekyll-drafts-vs-published-false
permalink: /ja/jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
alternates:
  ko: /jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
  en: /en/jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
  ja: /ja/jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
  x_default: /jekyll/blog/seo/2026/02/28/jekyll-drafts-vs-published-false.html
categories: [jekyll, blog, seo]
tags: [jekyll, drafts, 公開管理, github-pages, seo]
description: "`_drafts`と`published: false`を役割で分離すると、誤公開を防ぎながら編集品質とSEO信頼性を維持できます。"
---

## 課題

Jekyllでは、執筆中の管理と公開制御を同じ手段で扱うと、確認前の記事が公開される事故が起きやすくなります。

## 運用の分離

1. **`_drafts`**: 構成が未完成な下書き
2. **`published: false`（`_posts`内）**: 内容は完成済みだがレビュー待ち

この分離で、執筆フェーズと公開フェーズを明確にできます。

## 推奨ルール

- 未完成記事 → `_drafts`
- 公開直前レビュー → `_posts` + `published: false`
- 承認後に公開

## まとめ

`_drafts`は執筆の安全領域、`published: false`は公開ゲートです。役割分担を徹底すると、誤公開を減らし、ブログ全体の信頼性を守れます。

## 関連記事

- [エラー再現情報の書き方](/ja/development/blog/seo/2026/03/02/api-error-troubleshooting-context.html)
- [再現可能なコード例チェックリスト](/ja/development/blog/seo/2026/03/01/reproducible-code-example-checklist.html)

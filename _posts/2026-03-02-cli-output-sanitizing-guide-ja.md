---
layout: post
title: "技術ブログでCLI出力を安全に公開する基準"
date: 2026-03-02 20:00:00 +0900
lang: ja
translation_key: cli-output-sanitizing-guide
permalink: /ja/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html
alternates:
  ko: /development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html
  en: /en/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html
  ja: /ja/development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html
  x_default: /development/blog/seo/2026/03/02/cli-output-sanitizing-guide.html
categories: [development, blog, seo]
tags: [技術ブログ, seo, 信頼性, cli, セキュリティ]
description: "CLIログ公開時に機密情報を伏せつつ、原因分析に必要な情報を残すための実務ルールを解説します。"
---

## 公開ルール

CLIログは「そのまま公開」も「全部伏せる」も問題です。
重要なのは**機密情報の除去と診断文脈の維持**です。

### 除去する情報
- トークン、APIキー、セッション値
- 内部IP/ドメイン、個人識別情報

### 残す情報
- エラーコード
- バージョン情報
- 失敗したコマンドと実行順序

## まとめ

置換ルールを明記すると、安全性を保ちながら読者の再現判断も支援できます。

## 関連記事

- [エラー再現情報の書き方](/ja/development/blog/seo/2026/03/02/api-error-troubleshooting-context.html)
- [ログ例を安全に公開する基準](/ja/development/blog/seo/2026/03/03/log-example-sanitization-for-trustworthy-dev-posts.html)

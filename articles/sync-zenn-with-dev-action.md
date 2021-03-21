---
dev_article_id: 640945
title: "Zenn の記事を DEV に自動的に同期させる GitHub Actions 作ってみた"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "githubactions", "forem"]
published: false
---

# はじめに

[DEV](https://dev.to/) のアカウントを作成していたものの、今まで全く有効活用出来ていませんでした。

DEV には [カノニカル URL](https://dev.to/michaelburrows/comment/125j0) を設定出来るので、Zenn の記事を投稿する際にクロスポストしたいなと常々考えておりました。
そこで、Zenn に記事を投稿したら、自動的に DEV にも記事を同期する GitHub Action を作ってみました。

[sync-zenn-with-dev-action](https://github.com/nikaera/sync-zenn-with-dev-action)

今回初めて GitHub Action を自作したのですが、その中で得た知見を残す形で記事を書くことにしました。

# TypeScript で GitHub Action を作る

GitHub 公式が TypeScript で GitHub Action を作るための [テンプレートプロジェクト](https://github.com/actions/typescript-action) を用意してくれています。

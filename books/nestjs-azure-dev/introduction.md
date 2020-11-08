---
title: "はじめに"
---

# 概要

[PlayFab](https://playfab.com/) の [CloudScript](https://docs.microsoft.com/ja-jp/gaming/playfab/features/automation/cloudscript/quickstart) 向けに [Azure Functions](https://azure.microsoft.com/ja-jp/services/functions/) の開発をすることになり、当初は [.NET Azure Functions](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-dotnet-class-library) の採用を検討していました。

しかし、開発速度が求められる案件であり C# を書ける人材がいなくて Mac 使いが多く Node.js を使いたいとのことだったので、その中で良さそうな開発ツールを選定しました。

結果 NestJS の [Azure Functions HTTP module](https://github.com/nestjs/azure-func-http) を採用しようとなりました。
最終的には下記が決め手でした。

1. Azure Functions へのアクセス手段として [HTTP Trigger](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=javascript) を使用する
2. Webアプリケーション開発の感覚で Azure Functions の開発が可能である
3. [Azure Cosmos DB](https://docs.microsoft.com/ja-jp/azure/cosmos-db/introduction) へは [MongoDB API](https://docs.microsoft.com/ja-jp/azure/cosmos-db/mongodb-introduction) を用いれば [TypeORM Module](https://github.com/nestjs/typeorm) を用いて接続可能である
4. ユニットテストや E2E テストを書くのが容易である

本記事では下記について記載していきます。

* NestJS の導入から開発環境のセットアップ
* Azure Cosmos DB を絡めた Azure Functions の開発からテスト環境の構築
* GitHub Actions を用いた CI 環境の構築
* Azure KeyVault を用いたシークレット情報の管理までの手順

# 動作環境

* Azure Functions Core Tools
  * core 2.11.1
  * func 3.0.2931
* Node.js 10.22.1
* MongoDB 4.4.1
* Docker 19.03.13
* Docker Compose 1.27.4

まずは NestJS + Azure Functions をローカルで開発するための環境をセットアップします。

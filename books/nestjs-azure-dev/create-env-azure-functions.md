---
title: "Azure Portal で Azure Functions の環境を構築する"
---

## Azure Portal で Azure Funtions のデプロイ先を作成する

既に Azure アカウントは作成済み前提で進めていきます。まず Azure Portal の[トップページ](https://portal.azure.com/#home)にアクセスして、Azure Functions (関数アプリ) のページに遷移してから下記の作業します。

1. [関数アプリの新規作成ページ](https://portal.azure.com/#create/Microsoft.FunctionApp) から関数アプリを作成する
![1. 関数アプリの新規作成ページから関数アプリを作成する](https://i.gyazo.com/48ad899aee1de9617b9cb854dac414c3.png)

2. 関数アプリの作成が成功して、各種リソースがデプロイされたことを確認する
![2. 関数アプリの作成が成功して各種リソースがデプロイされたことを確認する](https://i.gyazo.com/2a3245f0eb0217770c38136a3de169d9.png)

## Azure Functions の環境をセットアップする

Azure Cosmos DB アカウントを作成した後、`MONGODB_URI` をシークレットとして [Azure KeyVault](https://docs.microsoft.com/ja-jp/azure/key-vault/) に設定します。Azure KeyVault で設定した値を Azure Functinos (関数アプリ) に環境変数としてセットして利用可能にします。

### Azure Cosmos DB アカウントを作成する

1. [Azure Cosmos DB アカウントの作成ページ](https://portal.azure.com/#create/Microsoft.DocumentDB)からアカウント作成する。API には `MongoDB~` を選択する。
![1. Azure Cosmos DB アカウントの作成ページから Azure Cosmos DB アカウントを作成する。API には `MongoDB~` を選択する](https://i.gyazo.com/91667a60dc987f0ce2310ce8f24332e4.png)

2. 作成した Azure Cosmos DB アカウントのプライマリ接続文字列 (MongoDB URI) を取得して控えておく
![2. Azure Cosmos DB の MongoDB URI を取得して控えておく](https://i.gyazo.com/f7408c4d2ea9d1d89b52a3f43d502bf9.png)

### プライマリ接続文字列を Azure KeyVault でシークレットに登録する

Azure Cosmos DB のプライマリ接続文字列はセキュアな情報なので Azure KeyVault (キーコンテナー) で管理します。

1. [キーコンテナーの作成ページ](https://portal.azure.com/#create/Microsoft.KeyVault)からキーコンテナーを作成する
![1. キーコンテナーの作成ページからキーコンテナーを作成する](https://i.gyazo.com/bb444f25e12715d366093cc41b227e66.png)

2. 1. で作成したキーコンテナーのシークレット登録画面に遷移する
![2. プライマリ文字列を登録するため、1. で作成したキーコンテナーのシークレット登録画面に遷移する](https://i.gyazo.com/b0e632a60911c08993af82cbcdac1055.png)

3. 控えておいた Azure Cosmos DB のプライマリ文字列 (MongoDB URI) をシークレットに登録する
![3. 控えておいた Azure Cosmos DB のプライマリ文字列 (MongoDB URI) をシークレットに登録する](https://i.gyazo.com/7414ee3f76753e1f383fcf3c295afb7c.png =300x)

4. 後に関数アプリで値を設定するため、登録したシークレットの識別子を控えておく
![4. 後に関数アプリで値を設定するため、登録したシークレットの識別子を控えておく](https://i.gyazo.com/a9e53f7ccdde0139630e3d6e17f3f791.png =300x)

### Azure Functions から Azure KeyVault が参照できるようにする

**Azure Functions (関数アプリ) では、Azure KeyVault の値を環境変数の設定が可能です。**[公式ページの手順](https://docs.microsoft.com/ja-jp/azure/app-service/app-service-key-vault-references)に沿って設定作業を進めていきます。

1. 関数アプリのシステム割り当て済みの状態をオンにする
![1. 関数アプリのシステム割り当て済みの状態をオンにする](https://i.gyazo.com/d4714472dd7eec4791ca9d0c957f8f27.png)

2. [キーコンテナーの一覧ページ](https://portal.azure.com/#blade/HubsExtension/BrowseResource/resourceType/Microsoft.KeyVault%2Fvaults) から該当するキーコンテナーのアクセスポリシー追加画面に遷移する
![2. キーコンテナーの一覧ページから該当するキーコンテナーのアクセスポリシー追加画面に遷移する](https://i.gyazo.com/2645dac181b63e00f0ff8c6be54d2b67.png)

3. 該当する関数アプリにアクセスポリシーを設定する
![3. 該当する関数アプリにアクセスポリシーを設定する](https://i.gyazo.com/76b23205edc2936327b7c0050467b766.png)

:::message alert
アクセスポリシー設定時に `承認されているアプリケーション` の設定項目がありますが、ここには何も設定しないでください。設定してしまうと関数アプリからキーコンテナーのシークレットが読み込めなくなるためです。
:::

### Azure KeyVault に登録した値を Azure Functions から参照する

1. 該当する関数アプリのアプリケーション設定追加画面に遷移する
![1. 関数アプリのアプリケーション設定追加画面に遷移する](https://i.gyazo.com/e5b7169061ce0a5802ad7de3753006f7.png)

2. 該当する関数アプリのアプリケーション設定にキーコンテナーのシークレットを追加する。
キーコンテナーのシークレットをアプリケーション設定から追加する際、値のフォーマットは `@Microsoft.KeyVault(SecretUri=<KeyVault で値を設定した時に取得可能なシークレット識別子>)` になります。
![2. 関数アプリのアプリケーション設定に KeyVault のシークレットを追加する](https://i.gyazo.com/d70d1db1a18f8b060e926d432307a828.png)

3. 該当する関数アプリのアプリケーション設定の変更内容を関数アプリに反映させる
![3. 該当する関数アプリのアプリケーション設定の変更内容を関数アプリに反映させる](https://i.gyazo.com/fe3ae44366040b9f9d17f6901bd467a6.png)

Azure の環境構築も出来たので、いよいよ Azure Functions へデプロイしてみます。デプロイも CI で回せるようにします。

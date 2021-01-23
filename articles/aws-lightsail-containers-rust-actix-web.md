---
title: "AWS Lightsail Containers に Actix web をデプロイする"
emoji: "⛵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "rust", "aws", "lightsail"]
published: false
---

# はじめに

[Actix web](https://github.com/actix/actix-web) で Web アプリケーションを作ったのですが、技術勉強も兼ねていたので、デプロイ先も今まで試したことがないものを試そうとしていました。そこで、日頃業務でも AWS を利用しているということもあり、去年末に発表された [AWS Lightsail Containers](https://aws.amazon.com/jp/about-aws/whats-new/2020/11/announcing-amazon-lightsail-containers/) をデプロイ先に採用しました。

AWS Lightsail Containers へのデプロイ自体は非常に簡単でした。また、デプロイにあたり Rust の Docker イメージ作成のやり方も学べました。今回はそのあたりの手順をまとめる形で記事として書き残しておくことにしました。

# Actix web の Docker イメージを作成する

開発したアプリケーションでは React でフロントエンド開発をしていて、ビルドしたものを Actix web の public フォルダに配置する形で公開しています。そのため、下記の Dockerfile ではマルチステージビルドを利用しておりますが、本質的には `FROM rust:1.49` 以降の記述が Actix web に関するものとなります。

```Dockerfile:Dockerfile
# React ビルド用のイメージ
FROM node:14.15.4-alpine3.10 as client_builder

ARG REACT_APP_API_URL
ARG REACT_APP_GYAZO_AUTH_URL
ARG REACT_APP_GA_UNIVERSAL_ID

WORKDIR /client
COPY ./client/package*.json .
RUN yarn install

ADD ./client .
RUN yarn build

# Actix web ビルド用のイメージ
FROM rust:1.49

# Actix web にアクセスするためのポートを公開する
EXPOSE 8080

# Actix web プロジェクトのフォルダをイメージに追加する
WORKDIR /server
ADD ./server .

# プロジェクトフォルダ内で `cargo install` してビルドを生成する
RUN cargo install --path .

# 不要になったファイル群を削除する
RUN ls | grep -v -E 'templates' | xargs rm -r

# React ビルド用のイメージでビルドした内容を Actix web ビルド用イメージに追加する
COPY --from=client_builder /client/build ./build
RUN mkdir tmp

# `cargo install` コマンドで生成したビルドを実行して Actix web を起動する
# 下記のコマンド名称は Cargo.toml 内の [package.name] に準ずる
CMD ["bloggimg-server"]

```

また、**Docker ビルド時のオプション管理を楽にするため、Docker Compose を利用しました。単一の Docker イメージをビルドする際にも利用しておくことで、後々コンテナを追加して連携させたいときにも即座に対応できたりでオススメです。**

```yml:docker-compose.yml
# context に Actix web プロジェクトのパスを指定する
# args に Docker ビルド時に利用したい ARGS の値を環境変数で設定する
# image に Docker の <イメージ名:タグ名> を指定する (今回は Docker Hub にデプロイする想定)
# env_file に開発/動作検証時に利用したい dotenv ファイルを指定する
# ports にポートマッピングの設定を書く

version: '3.8'
services:
  app:
    build:
      context: ./
      args:
        - REACT_APP_API_URL=${REACT_APP_API_URL}
        - REACT_APP_GYAZO_AUTH_URL=${REACT_APP_GYAZO_AUTH_URL}
        - REACT_APP_GA_UNIVERSAL_ID=${REACT_APP_GA_UNIVERSAL_ID}
    image: n1kaera/bloggimg:v1.0.0
    env_file:
      - ./server/.env
    ports:
      - 8080:8080
 
```

上記を自分の Actix web プロジェクトに応じて改変し `docker-compose up` して動作検証します。動作検証ができ次第、`docker-compose build` を実行して Docker イメージをビルドします。ビルドに成功したら次は Docker Hub にイメージを push します。

# Docker Hub にビルドしたイメージを push する

今回は AWS Lightsail Containers で使用するイメージの管理に [Docker Hub](https://hub.docker.com/) を利用します。Docker Hub へ push する前に `docker login --username=<Docker Hub のユーザ名>` コマンドで Docker Hub へのログインを済ませておきます。

その後 `docker-compose push` コマンドで Docker イメージを Docker Hub に push します。

![スクリーンショット 2021-01-23 20.37.39.png](https://i.gyazo.com/06ce4ca43a26c73c227b9eb768f65685.png)
**Docker Hub のページから、正常に Docker イメージが push できていそうか確認する**

Docker イメージの push が成功していることを確認できたら、残りは AWS Console 上での作業になります。

# AWS Console から Lightsail Containers Service を作成する

AWS Console にログイン後、[Lightsail サービス](https://lightsail.aws.amazon.com/ls/webapp/home/containers) を選択して Lightsail サービスのトップページへ遷移します。遷移したら Containers タブを選択し、`Create container services` ボタンから Container Service を作成します。

![スクリーンショット 2021-01-23 18.55.16.png](https://i.gyazo.com/42a8e6fa6224a6a52212cdb7461a8515.png)
**AWS Console へログイン後 Lightsail のページに遷移して、Containers タブを選択する**

![スクリーンショット 2021-01-23 18.57.06.png](https://i.gyazo.com/25c1b55863c4d77249d3105ba7a97afd.png)
**Containers タブを選択すると出てくる、`Create container services` ボタンをクリックする**

`Create container services` ボタンをクリックした遷移先の画面で、リージョンやキャパシティ (Micro であれば 3ヶ月間のみ無料で利用可能) 等を選択して、名称を入力します。今回は最初にデプロイのための準備をすでに済ませているので、Container Service を作成するついでにデプロイ設定も行います。

デプロイ設定は `Set up your first deployment` の項目から行うことが可能です。

![スクリーンショット 2021-01-23 19.07.53.png](https://i.gyazo.com/e8e4ab4e3b29afaf6298f7eb75355c34.png)
**`Set up deployment` の部分をクリックして、デプロイの設定項目を表示する**

![スクリーンショット 2021-01-23 19.12.14.png](https://i.gyazo.com/da0fd3ce440f441082a367a0dfe350b6.png)
**Docker Hub イメージを利用してデプロイする際に必要な設定項目を入力する**

![スクリーンショット 2021-01-23 19.16.16.png](https://i.gyazo.com/eefec162ec66adab937d5139f70763cc.png)
**コンテナのヘルスチェックのための情報を入力する**

![スクリーンショット 2021-01-23 19.19.17.png](https://i.gyazo.com/0c532eb818754d554a61028f6a177dde.png)
**すべての情報入力が完了したら `Create container services` ボタンをクリックする**

![スクリーンショット 2021-01-23 19.22.48.png](https://i.gyazo.com/102978a025c8babbba40886e1b551ae6.png)
**遷移後の画面下部の `Deployment versions` からデプロイ状況の確認が行える**

![スクリーンショット 2021-01-23 19.39.55.png](https://i.gyazo.com/eaaf0b1f1d2b2d37807d49764cafa681.png)
**正常にデプロイできていれば `Deployment versions` の項目が Active になる**

デプロイが完了したら `Public domain` が発行されているはずなので、正常にアクセスして Web アプリケーションが利用できそうか確認します。`Public domain` は該当する Container Service のトップページから確認できます。

![スクリーンショット 2021-01-23 19.48.23.png](https://i.gyazo.com/9ee36e7f1018c47a9c12e51ebe6df6d0.png)
**AWS Lightsail Containers のトップページにある `Public domain` から動作検証する**

![スクリーンショット 2021-01-23 19.51.27.png](https://i.gyazo.com/a381f52745c002ce47addf7721b7ec0b.png)
**一通りの動作検証を行い、正常にデプロイできていそうか確認する**

これで作業は完了です。新しい Docker イメージでデプロイし直したい場合は、`Deployments` タブの `Modify your deployment` リンクをクリックすれば可能です。

# (おまけ) 独自ドメインで Container Service へアクセス可能にする

AWS Lightsail Containers では独自ドメインの紐付け及び、HTTPS 化も簡単に設定できます。`Custom domains` タブを選択した後、画面下部にある `Create certificate` リンクをクリックすることで設定画面を表示します。

![スクリーンショット 2021-01-23 20.07.22.png](https://i.gyazo.com/93019714418209872be82c396931beee.png)
**`Custom domain` タブをクリックしてから、`Create certificate` リンクをクリックする**

![スクリーンショット 2021-01-23 20.10.22.png](https://i.gyazo.com/b6cc81261f5d3bec4b236e04c0409372.png)
**各種設定項目の入力が完了したら `Create` ボタンをクリックする**

![スクリーンショット 2021-01-23 20.16.02.png](https://i.gyazo.com/d8075a0e1eedeb1e518e7f0ae22554a1.png)
**ドメイン検証のために、CNAME レコードの設定を求められるので各自で設定作業を行う**

![スクリーンショット 2021-01-23 20.20.26.png](https://i.gyazo.com/3aa4608c22ab83193d75b3e968d47b77.png)
**正常に CNAME レコードを設定した後、しばらく経つと Status が Valid になる**

上記まで確認したら、`Create certificate` で設定したドメインの CNAME レコードに `Public domain` の値を設定しておきます。設定内容が反映され次第、独自ドメインへアクセスすることで HTTPS 経由で Container Service へアクセスできるようになります。

![スクリーンショット 2021-01-23 20.27.26.png](https://i.gyazo.com/21d69cbc2370b588e01cfae43afa50ca.png)
**`Custom domains` で Container Service で起動しているサービスにアクセスできることを確認する**

# おわりに

AWS Lightsail Containers を利用して Actix web プロジェクトをデプロイする手順について簡単にまとめてみました。便利ではあるものの、個人開発で利用する分には価格面及び性能面で Lightsail Instance のほうが良いなと現時点では感じてしまいました。

しかし、日本リージョンが用意されていたりロードバランサーを備えていたり、簡単にスケールさせやすくかつ定額で利用可能なサービスであるというメリットを活かせる場面があれば有効活用できそうだなと感じました。

# 参考リンク

* [Lightsail コンテナ: クラウドでコンテナを実行する簡単な方法 \| Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/lightsail-containers-an-easy-way-to-run-your-containers-in-the-cloud/)
* [rust \- Docker Hub](https://hub.docker.com/_/rust)
* [Enabling custom domains for your Amazon Lightsail distributions \| Lightsail Documentation](https://lightsail.aws.amazon.com/ls/docs/en_us/articles/amazon-lightsail-enabling-distribution-custom-domains)

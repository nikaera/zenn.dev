---
title: "Azure Cosmos DB の開発環境を構築する"
---

イベントの CRUD 機能実装にあたって、**今回はデータベースに Azure Cosmos DB を採用しますが、ローカルで開発する際は MongoDB を利用します。**

そのため、まずは開発用に [MongoDB の Docker イメージ](https://hub.docker.com/_/mongo) を使用できるようにします。今後 E2E テストを行う際に [Docker Compose](https://docs.docker.com/compose/) を利用する想定のため、`devenv/docker-compose.yml` に MongoDB を利用する記述を書いていきます。

```yml:devenv/docker-compose.yml
version: '3.8'
services:
  mongodb:
    image: mongo:latest
    ports:
        - "27017:27017"
    environment:
        MONGO_INITDB_ROOT_USERNAME: azure-sample
        MONGO_INITDB_ROOT_PASSWORD: azure-sample
        MONGO_INITDB_DATABASE: azure-sample
        TZ: Asia/Tokyo
```

`devenv/docker-compose.yml` の記述が終わったら、下記コマンドで MongoDB のコンテナをデーモン状態で起動します。

```bash
# Docker Compose で MongoDB コンテナをデーモンで起動する
cd devenv
docker-compose up -d

# MongoDB のコンテナが起動できたか確認する (NAMES の欄に devenv_mongodb_1 が表示されていれば成功)
docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                      NAMES
f78f09cd4f5b        mongo:latest        "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:27017->27017/tcp   devenv_mongodb_1
```

無事起動できれば `devenv_mongodb_1` という名前のコンテナが起動していることが `docker ps` コマンドで確認できるはずです。

これで MongoDB で開発する環境は整ったのですが、今はまだ使用しないので `docker-compose down` コマンドでコンテナを停止しておきます。

```bash
# Docker Compose で起動したコンテナを停止する
cd devenv
docker-compose down
Stopping devenv_mongodb_1 ... done
Removing devenv_mongodb_1 ... done
Removing network devenv_default
```

ローカルで検証可能な環境が整ったので、次は TypeORM を用いたイベントの CRUD 機能を実装していきます。

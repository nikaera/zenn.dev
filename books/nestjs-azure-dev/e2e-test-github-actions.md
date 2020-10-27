---
title: "GitHub Actions で E2E テストを実行する"
---

本記事の内容を採用したプロジェクトではソースコード管理を GitHub で行っていたので、CI 環境として GitHub Actions を採用していました。

そのため、今回は GitHub Actions で Docker Compose の E2E テストを実行できるようにしていきます。

```yml:.github/workflows/build-and-test.yml
# Pull Request が更新されるたびに走らせる
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  build-and-test:
    name: Build and Test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [ '10.22.1' ]
    steps:
    # Pull Request 出された branch の最新の commit のソースコードを使用する
    - name: checkout pushed commit
      uses: actions/checkout@v2
      with:
        ref: ${{ github.event.pull_request.head.sha }}
    - name: Use Node.js ${{ matrix.node }}
      uses: actions/setup-node@v1
      with:
        node-version: ${{ matrix.node }}
    # Node.js のビルドが通るか検証する
    - name: npm install, build.
      run: |
        npm install
        npm run build --if-present
    # E2E テストを Docker Compose で実行する
    - name: run test on docker-compose
      run: |
        docker-compose build
        docker-compose up --abort-on-container-exit
      working-directory: ./devenv
```

上記ファイル作成後、適当にブランチを切ってコミットしてからリモートリポジトリに push して PR を出すと、下記のように GitHub Actions が動いていることが確認できるはずです。

![GitHub Actions で E2E テストが動いているか確認する](https://i.gyazo.com/eef4c3788e92148bc906ee6a1dffd95b.png)
*GitHub Actions で E2E テストが動いているか確認する*

PR のたびに E2E テストが実行されるため、レビューが楽になりました。これで一通りの開発環境は揃ったため、次は実際に Azure Portal 上でデプロイ環境を整備していきます。

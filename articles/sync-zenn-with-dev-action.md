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

DEV には [カノニカル URL](https://dev.to/michaelburrows/comment/125j0) を設定出来るので、Zenn の記事を投稿する際にクロスポストしたいなと常々考えておりました。そこで、Zenn に記事を投稿したら、自動的に DEV にも記事を投稿 & 同期する GitHub Action を作ってみました。

[sync-zenn-with-dev-action](https://github.com/nikaera/sync-zenn-with-dev-action)

今回初めて GitHub Action を自作したのですが、その中で得た知見を残す形で記事を書くことにしました。また、GitHub Action は TypeScript で作成しました。

# 今回作成した GitHub Action の概要

まずはザッとどのような GitHub Action を作成したのか、概要について説明いたします。

GitHub リポジトリで管理している Zenn の記事を DEV に同期して投稿する GitHub Action を作成しました。その際に DEV へ投稿する記事には Zenn の該当記事へのカノニカル URL も自動で設定できます。これにより DEV と Zenn へ記事をシームレスにクロスポストすることが可能となります。

今回作成した GitHub Action を利用するワークフローファイルの一例は下記となります。

```yml
name: "Sync all Zenn articles to DEV"
on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: checkout my project
        uses: actions/checkout@v2
      - name: dev.to action step
        uses: nikaera/sync-zenn-with-dev-action@v1
        # id を設定することで、後のジョブで Output で指定した値が参照可能になる
        id: dev-to
        with:
          # DEV の API キーを指定する
          api_key: ${{ secrets.api_key }}
          # (オプション) DEV に記事を投稿した際に Zenn のカノニカル URL を設定したい場合に指定する
          # username: nikaera
          # (オプション) 改行区切りで指定した articles フォルダ内のファイルパスを記載した txt ファイルを指定することで、記載された記事のみを同期するようになる。
          # 他プラグインと組み合わせることで差分のみを txt ファイルに載せることが可能。詳細については後述の Outputs の項目に記載。
          # added_modified_filepath: ./added_modified.txt
          # (オプション) Zenn の articles 以下全ての記事を常に DEV に同期するか指定する
          # update_all が true のときは added_modified_filepath は無視される。
          # update_all: false
        # 上記アクション実行時に DEV に新規で同期する記事に関しては Zenn のマークダウンヘッダに
        # 該当する DEV の記事の ID が dev_article_id として記載されるようになる。
        # 今後はその ID を元に同期するようになるため、該当する Zenn の記事をコミットする。
        # 新規で同期する記事が無ければ、このジョブは実行しない。
      - name: write article id of DEV to articles of Zenn.
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          git add ${{ steps.dev-to.outputs.newly-sync-articles }}
          git commit -m "sync: Zenn with DEV [skip ci]"
          git push
        if: steps.dev-to.outputs.newly-sync-articles
        # Outputs には DEV の記事情報 (title, url) が含まれるようになるため、
        # 最後に出力して実行結果の内容を確認することもできる
      - name: Get the output articles.
        # dev-to という id が紐付いたジョブの Outputs を取得して echo で内容を出力する
        run: echo "${{ steps.dev-to.outputs.articles }}"
```

簡単に `nikaera/sync-zenn-with-dev-action@v1` の内部処理について説明いたします。

1. Inputs の `update_all` 及び `added_modified_filepath` で受け取った情報を元に、
   どの記事を DEV に同期させるか判定する
1. DEV に同期する記事のファイルパス一覧を取得して、
   それぞれの記事のヘッダに `dev_article_id` の記載があるか判定する
1. Inputs の `api_key` を利用して、Zenn の記事に `dev_article_id` が含まれていれば、
   該当する DEV の記事を更新する。含まれていなければ DEV に新規で記事を作成する
1. Inputs の `username` を利用して、
   DEV の記事に該当する Zenn 記事のカノニカル URL を設定する
1. DEV に新規で記事を作成した場合は、
   Zenn の該当記事のヘッダに `dev_article_id` を書き込む
1. DEV に記事を投稿する際、Zenn は対応しているが、
   DEV では対応していない記述は削除する (`:::` 記法や一部のコード記法)
1. 記事の公開ステータス及びタグなどについても DEV の記事に反映する[^1]
1. 新規で DEV に記事を同期した Zenn 記事のファイルパスを、
   Outputs の `newly-sync-articles` に設定する
   (後のジョブで `dev_article_id` の含まれた記事をコミットしたいため)
1. ワークフローで同期された記事情報は Outouts の `articles` に設定する

Inputs と Outputs の内容一覧については下記になります。

**Inputs**

| キー                    | 説明                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | 必須 |
| :---------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--: |
| api_key                 | DEV の [API Key](https://docs.forem.com/api/#section/Authentication) を設定する                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |  o   |
| username                | Zenn の **自分のアカウント名** を設定する (DEV に同期する記事に Zenn のカノニカル URL を設定したい場合のみ)                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |  x   |
| added_modified_filepath | 改行区切りで指定した articles フォルダ内のファイルパスを記載した txt ファイルを指定することで、記載された記事のみを同期するようになる。**PR やコミット差分のファイルのみを取得するための GitHub Action [jitterbit/get-changed-files@v1](https://github.com/jitterbit/get-changed-files) と組み合わせることで、更新差分のあった記事のみを随時同期することも可能。[^2]** 更新差分のあった記事のみを随時同期するための[実際のワークフローファイルはこちら](<[該当するワークフローファイル](https://github.com/nikaera/zenn.dev/blob/main/.github/workflows/sync-zenn-with-dev.yml)>) |  x   |
| update_all              | Zenn の全ての記事をどうきするかどうかを設定する。GitHub Action 初回実行時のみ true にする使い方を想定している。デフォルトは true。**`added_modified_filepath` よりも `update_all` が優先されるため `added_modified_filepath` を設定する場合は false を設定する必要あり`**                                                                                                                                                                                                                                                                                                         |  x   |

**Outputs**

| キー                | 説明                                                                                                                                                                                                                                                                                 |
| :------------------ | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| articles            | 同期された DEV の記事のタイトル及び URL が格納された配列                                                                                                                                                                                                                             |
| newly-sync-articles | DEV で新たに新規作成された Zenn 記事のファイルパスが格納された配列。**[実際のワークフローファイルの該当する記述](https://github.com/nikaera/sync-zenn-with-dev-action/blob/main/.github/workflows/test.yml#L31-L38)のように、必ずコミットに含めるようにする必要がある (理由は後述)** |

ザッと本記事で紹介する GitHub Action の説明は上記の通りです。

## Zenn の記事を DEV に同期するための仕組み

Zenn の記事を新規で DEV に同期する際は、DEV に記事を新規作成する必要があります。その際に Zenn の記事と DEV の記事を紐付けるための何らかの仕組みが必要となります。そうしないと、今後 Zenn の記事内容を DEV のどの記事に同期させるようにすればよいかが不明なためです。

そこで記事を同期するための仕組みとして、`dev_article_id` というフィールドを Zenn のマークダウンヘッダに追記することで DEV の同期すべき記事との紐付けを行うことにしました。`dev_article_id` には [DEV の記事作成 API](https://docs.forem.com/api/#operation/createArticle) 実行時の返り値である `id` を設定します。

一度 `dev_article_id` を紐付けてしまえば、次回以降に記事の同期を行う際は [DEV の記事更新 API](https://docs.forem.com/api/#operation/updateArticle) を利用することが可能になります。

また、Outputs の `newly-sync-articles` には新規で作成された DEV 記事の `id` が追記された Zenn 記事のファイルパスが格納されています。そのため、`nikaera/sync-zenn-with-dev-action@v1` 実行後は、下記のように `steps.dev-to.outputs.newly-sync-articles` 内に格納されたファイル群をコミットに反映させる必要があります。

```yml
- name: write article id of DEV to articles of Zenn.
    run: |
        git config user.name github-actions
        git config user.email github-actions@github.com
        git add ${{ steps.dev-to.outputs.newly-sync-articles }}
        git commit -m "sync: Zenn with DEV [skip ci]"
        git push
```

::: message alert

`newly-sync-articles` に格納された `dev_article_id` が追記された Zenn 記事は常にコミットに反映しないと、**Zenn の全ての記事が同期毎 DEV に新規作成され続けるという挙動を引き起こしてしまうので、ご注意ください**

:::

# GitHub Action を自作するにあたり行った手順

サクッと作りたかったため、[Docker コンテナを利用する方法](https://docs.github.com/ja/actions/creating-actions/creating-a-docker-container-action) ではなく、[JavaScript を利用する方法](https://docs.github.com/ja/actions/creating-actions/creating-a-javascript-action) で開発を進めていくことにしました。

## TypeScript で GitHub Action を作る

GitHub 公式が TypeScript で GitHub Action を作るための [テンプレートプロジェクト](https://github.com/actions/typescript-action) を用意してくれています。今回はこのテンプレートプロジェクトを利用する形でプロジェクトを作成しました。

::: message info

GitHub Action では [Docker コンテナ](https://docs.github.com/ja/actions/creating-actions/creating-a-docker-container-action) を用いてワークフローが実行可能です。そのため、実行環境は自由に設定出来ます。(Go, Python, Ruby, etc.)

:::

早速 TypeScript のテンプレートプロジェクトを元に自分の GitHub Action プロジェクトを作成します。

![スクリーンショット 2021-03-21 13.25.54.png](https://i.gyazo.com/359f6f795bab9807c9f480c0f922973a.png)
**1. テンプレートプロジェクトを元に GitHub Action の TypeScript プロジェクトを作成する**

![スクリーンショット 2021-03-21 13.29.43.png](https://i.gyazo.com/76aecd3d63db95d34c086774ded122d4.png)
**2. プロジェクトの作成後 `git clone` してきて開発する準備を整える**

## GitHub Action プロジェクトの開発を進めるための準備を行う

テンプレートプロジェクトを `git clone` したら、まずは `action.yml` の内容を変更します。
今回作成した GitHub Action の `action.yml` は下記となっております。

```yml:action.yml
# GitHub Action のプロジェクト名
name: 'Sync Zenn articles to DEV'
# GitHub Action のプロジェクト説明文
description: 'Just sync Zenn articles to DEV.'
# GitHub Action の作者
author: 'nikaera'
# GitHub Action に渡せる引数の値定義
inputs:
  api_key:
    # フィールドの指定が必須であれば true、必須でなければ false を設定する
    # DEV の API キーは同期を行う際に必須なため、true を設定している
    required: true
    # フィールドの説明文
    description: 'The api_key required to use the DEV API (https://docs.forem.com/api/#section/Authentication)'
  username:
    required: false
    description: "Zenn user's account name. (Fields to be filled in if canonical url is set.)"
  articles:
    required: false
    description: "The directory where Zenn articles are stored."
    # フィールドにはデフォルト値を指定することも可能
    # Zenn の記事がデフォで格納されているフォルダ名を指定している
    default: articles
  update_all:
    require: false
    description: "Whether to synchronize all articles."
    default: true
  added_modified_filepath:
    required: false
    description: |
      Synchronize only the articles in the file path divided by line breaks.
      You can use jitterbit/get-changed-files@v1 to get only the file paths of articles that have changed in the correct format.
      (https://github.com/jitterbit/get-changed-files)
# GitHub Action 実行後に参照可能になる値定義
outputs:
  articles:
    description: 'A list of URLs of dev.to articles that have been created or updated'
  newly-sync-articles:
    description: 'File path list of newly synchronized articles.'
# GitHub Action の実行環境
runs:
  using: 'node12'
  # テンプレートプロジェクトでは コンパイル先が dist になるため `dist/index.js` を指定している
  main: 'dist/index.js'
```

TypeScript のテンプレートプロジェクトでは、バンドルツールとして [`ncc`](https://github.com/vercel/ncc) が採用されています。GitHub Action 実行時に使用されるのは ncc によりコンパイルされた単一の JavaScript ファイル (`dist/index.js`) になります。

あとは `src` フォルダ内でプログラムを書いて、`npm run all && node dist/index.js` のようにコマンド実行しながら開発を進めていくだけです。

::: message info

(余談) 開発ツールとして Docker を利用した [`act`](https://github.com/nektos/act) というものが存在するようです。ローカル環境で検証する際は [既知の問題](https://github.com/nektos/act#known-issues) に対応する必要がありそうですが、GitHub Action の開発で非常に有効活用できそうで気になっております。

今回の開発では利用しなかったのですが、今後開発を進めていく中で利用する機会も出てきそうなので、その際は本記事内容を更新する形で知見を追記したいと考えております。

:::

## GitHub Action を実装する際に利用した機能

GitHub Action を実装する際に利用した機能を、実際のコード内容を抜粋して簡単に説明していきます。
下記で紹介する内容は [GitHub Actions Toolkit](https://github.com/actions/toolkit) の機能です。

```typescript:main.ts
/**
下記の yml の with で指定した値は core.getInput で受け取ることが可能。

- name: dev.to action step
    uses: nikaera/sync-zenn-with-dev-action@v1
    id: dev-to
    with:
        # DEV の API キーを指定する
        api_key: ${{ secrets.api_key }}
*/
core.getInput("api_key", { required: true });
core.getInput("articles", { required: false });

/**
下記の yml の steps.<ジョブで指定した id>.outputs で参照可能な値をセットすることが可能。
セットする内容は文字列である必要がある。

- name: dev.to action step
    uses: nikaera/sync-zenn-with-dev-action@v1
    id: dev-to
- name: Get the output articles.
    run: echo "${{ steps.dev-to.outputs.articles }}"
*/

core.setOutput("articles", JSON.stringify(devtoArticles, undefined, 2));
core.setOutput("newly-sync-articles", newlySyncedArticles.join(" "));

/**
GitHub Action 実行時に出力されるログをレベルごとに出力することが可能
core.debug はローカル実行時のみに出力内容を確認することができる
*/
core.debug("debug");
core.info(`update_all: ${updateAll}`);
core.error(JSON.stringify(error));
```

上記の機能だけ把握してれば GitHub Action の開発は問題なく行うことができました。

# 作成した GitHub Action を実際に GitHub 上で実行可能にする

ローカル環境で一通り開発が完了したら、リモートリポジトリに push した後タグ付けを行います。**GitHub Action はタグを指定しないと実行できないため必要な作業となります。** 今回タグ付けは GitHub 上で行いました。

![スクリーンショット 2021-03-22 8.11.58.png](https://i.gyazo.com/252514cca20470154475db75b133e9b3.png)
**1. タグの項目をクリックする**

![スクリーンショット 2021-03-22 8.14.47.png](https://i.gyazo.com/a8c145f8d6ae7b1f6253eb4866bcbb51.png)
**2. `Create a new release` ボタンをクリックしてタグ作成画面に遷移する**

![スクリーンショット 2021-03-22 8.20.00.png](https://i.gyazo.com/25e019e42227ad7e0b0b634e68bb1873.png)
**3. `Publish release` ボタンをクリックしてタグの作成を完了する**

上記の例では `v1` というタグを作成したので `nikaera/sync-zenn-with-dev-action@v1` のような記述で GitHub Action を利用可能になりました。私は Zenn の記事を [`zenn.dev`](https://github.com/nikaera/zenn.dev/) というリポジトリで管理しているため、早速このリポジトリに GitHub Action を導入してみます。

## Zenn の全ての記事を DEV に同期するためのワークフロー

まず GitHub Action では DEV の API キーを設定する必要があるため、事前にシークレットへ `API_KEY` という名称で値を登録しておきます。[公式サイトの手順](https://docs.github.com/ja/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository) に従い シークレットの登録が完了したら、該当リポジトリに `.github/workflows/sync-zenn-with-dev-all.yml` というワークフローファイルを作成します。

```yml:.github/workflows/sync-zenn-with-dev-all.yml
name: "Sync-All Zenn with DEV"
on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    if: contains(github.event.head_commit.message, '[skip ci]') == false
    steps:
      - name: setup node project
        uses: actions/checkout@v2
      - name: dev.to action step
        uses: nikaera/sync-zenn-with-dev-action@v1
        id: dev-to
        with:
          api_key: ${{ secrets.api_key }}
          username: nikaera
          update_all: true
      - name: write article id of DEV to articles of Zenn.
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          git add ${{ steps.dev-to.outputs.newly-sync-articles }}
          git commit -m "sync: Zenn with DEV [skip ci]"
          git push
        if: steps.dev-to.outputs.newly-sync-articles
      - name: Get the output articles.
        run: echo "${{ steps.dev-to.outputs.articles }}"
```

上記は手動実行が可能な Zenn の記事を全て DEV に同期するためのワークフローファイルになります。作成が完了したらワークフローファイルを実行して、記事が正常に同期できているか確認してみます。

![スクリーンショット 2021-03-22 8.35.38.png](https://i.gyazo.com/e0740fe369752db67514e0ab50e88772.png)
**1. `Actions` タブをクリックする**

![スクリーンショット 2021-03-22 8.37.24.png](https://i.gyazo.com/2aeadddccf08c0dddc54dddab464edee.png)
**2. 作成した `Sync-All Zenn with DEV` ワークフローファイルを選択する**

![スクリーンショット 2021-03-22 8.40.11.png](https://i.gyazo.com/d9e138de1f6e0baac8b0c3bc6742f4f2.png)
**3. `Run workflow` ボタンを実行して、ワークフローを実行する**

![スクリーンショット 2021-03-22 8.45.23.png](https://i.gyazo.com/e67df0f6e33d1f57a46000d20ef2c824.png)
**4. ワークフローの実行が正常に完了していれば、ログに DEV の記事情報が出力される**

![スクリーンショット 2021-03-22 8.50.37.png](https://i.gyazo.com/4acc0b0de3acd6731918a9347e7baea8.png)
**5. `dev.to` にアクセスして Zenn の記事情報が正常に同期されていることを確認する**

正常に Zenn の全ての記事が DEV に同期されていることが確認できたら、次は Zenn の記事が新規作成されたり、更新されたときのみに DEV に同期するためのワークフローファイルを作成します。

## 更新差分のあった Zenn 記事のみを DEV に同期するためのワークフロー

`main` ブランチが更新されたときのみ更新された記事のみを DEV に同期するためのワークフローファイルを作成します。該当リポジトリに `.github/workflows/sync-zenn-with-dev.yml` というワークフローファイルを作成します。

```yml:.github/workflows/sync-zenn-with-dev.yml
name: "Sync Zenn with DEV"
on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    if: contains(github.event.head_commit.message, '[skip ci]') == false
    steps:
      - name: setup node project
        uses: actions/checkout@v2
      - name: get modified files
        id: files
        uses: jitterbit/get-changed-files@v1
      - name: output modified files to text
        run: |
          for changed_file in ${{ steps.files.outputs.added_modified }}; do
            echo "${changed_file}" >> added_modified.txt
          done
      - name: dev.to action step
        uses: nikaera/sync-zenn-with-dev-action@v1
        id: dev-to
        with:
          api_key: ${{ secrets.api_key }}
          username: nikaera
          added_modified_filepath: ./added_modified.txt
          update_all: false
      - name: write article id of DEV to articles of Zenn.
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          git add ${{ steps.dev-to.outputs.newly-sync-articles }}
          git commit -m "sync: Zenn with DEV [skip ci]"
          git push
        if: steps.dev-to.outputs.newly-sync-articles
      - name: Get the output articles.
        run: echo "${{ steps.dev-to.outputs.articles }}"
```

上記ワークフローファイルの作成が完了したら、早速動作確認のために、まさに今執筆中の本記事をリポジトリに push してみます。

# おわりに

# 参考リンク

[^1]: DEV (Forem) の仕様上、タグは最大でも 4 つまでしか設定できないため、Zenn で設定したタグの先頭 4 つまで DEV の記事には設定している
[^2]: 当然といえば当然ですが [jitterbit/get-changed-files@v1](https://github.com/jitterbit/get-changed-files) は `workflow_dispatch` でワークフローを手動実行したり、`force push` 等でファイル差分を正確に特定できない操作には対応しておりませんので、その場合はエラーが発生します。

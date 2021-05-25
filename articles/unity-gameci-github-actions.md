---
dev_article_id: 707694
title: "GameCI で Unity の CI 環境を GitHub Actions で構築する"
emoji: "🎮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unity", "githubactions"]
published: false
---

# はじめに

先日 [ようさん](https://twitter.com/ayousanz) が [GameCI](https://game.ci/) について教えてくれました。早速試してみたところ、確かに GitHub Actions を用いた Unity の CI 環境が楽に構築できることが分かりました。

GameCI には現状下記の GitHub Actions が用意されているようです。

| 機能                                                                   | 概要                                                                                                                                                         |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [Activation](https://game.ci/docs/github/activation)                   | Unity ライセンスを任意のバージョンで発行する                                                                                                                 |
| [Test runner](https://game.ci/docs/github/test-runner)                 | Unity の PlayMode 及び EditMode のテストを実行する (テスト結果の出力にも対応)                                                                                |
| [Builder](https://game.ci/docs/github/builder)                         | 任意の Platform ビルドを実行する ([アーティファクト](https://docs.github.com/ja/actions/guides/storing-workflow-data-as-artifacts) 利用でダウンロードも可能) |
| [Returning a license](https://game.ci/docs/github/returning-a-license) | Unity ライセンスの返却ができる (発行上限数に引っかかった際などに有効活用できる)                                                                              |
| [Remote builder](https://game.ci/docs/github/remote-builder)           | GitHub Actions のスペックでは満足のいくビルドができない際に AWS 環境でハイスペックなマシンを用意してビルドできる (GCP, Azure にも対応予定とのこと)           |
| [Deployment](https://game.ci/docs/github/deployment/android)           | Unity ビルドを各種 Platform 向けにデプロイする (現在は Android, iOS のみ対応)                                                                                |

上記を見ると既に **GameCI には開発者として Unity CI に欲しい機能は最低限揃っているように見受けられました。** また本記事では **Activation と Test runner** についてのみ紹介しております。

検証時に利用したプロジェクトは GitHub にアップ済みですので、手っ取り早く把握されたい方は下記をご参照くださいませ。

https://github.com/nikaera/Unity-CISample

業務でも利用できそうなので、GameCI を利用して CI 環境を構築する手順を記事でまとめました。

# GameCI を利用して Unity License のアクティベーションを行う

GameCI で Unity ライセンスをアクティベートするには [Activation](https://game.ci/docs/github/activation) を利用します。早速ドキュメントの手順に沿って作業を進めていきます。

1. CI を導入したい GitHub 上の Unity プロジェクトの `.github/workflows` 内に Unity ライセンスアクティベート用のワークフローを作成して [デフォルトブランチ](https://docs.github.com/ja/github/setting-up-and-managing-your-github-user-account/managing-user-account-settings/managing-the-default-branch-name-for-your-repositories#about-management-of-the-default-branch-name) にプッシュする (恐らく `main` or `master` ブランチ)

```yml:.github/workflows/activation.yml
name: Acquire activation file
on:
  workflow_dispatch: {}
jobs:
  activation:
    name: Request manual activation file 🔑
    runs-on: ubuntu-latest
    steps:
      # Request manual activation file
      - name: Request manual activation file
        id: getManualLicenseFile
        uses: game-ci/unity-request-activation-file@v2
        with:
          # Unity プロジェクトのバージョンを指定する
          # ProjectSettings/ProjectVersion.txt に記載されているバージョンを入力すれば OK
          unityVersion: 2020.3.5f1
      # Upload artifact (Unity_v20XX.X.XXXX.alf)
      - name: Expose as artifact
        uses: actions/upload-artifact@v2
        with:
          name: ${{ steps.getManualLicenseFile.outputs.filePath }}
          path: ${{ steps.getManualLicenseFile.outputs.filePath }}
```

2. ブラウザから GitHub リポジトリにアクセスして、Unity ライセンスアクティベート用のワークフローを実行して `alf` ファイルを生成する

![Unity ライセンスアクティベート用のワークフローを実行する](https://i.gyazo.com/bd276ca6dcf6a2c12ce9ff9569e08ce3.png)
**Unity ライセンスアクティベート用のワークフローを実行して `alf` ファイルを生成する**

3. ワークフローの実行が成功したら、詳細画面に遷移した後、アーティファクトから `alf` ファイルをダウンロードする。後述する `ulf` ファイルの生成に必須となる。

![ワークフローの実行結果確認のための画面に遷移する](https://i.gyazo.com/2271b3cb35efc7f1c9d51702662cdac9.png)
**ワークフローの実行結果確認のための画面に遷移する**

![`Artifacts` の項目から `alf` ファイルをダウンロードする](https://i.gyazo.com/71b9dff8266c9bc990e1b709a5191535.png)
**`Artifacts` の項目から `alf` ファイルをダウンロードする**

4. [Unity license manual activation webpage](https://license.unity3d.com) からログインして `alf` ファイルをアップロードする

![ログイン後の画面から `alf` ファイルをアップロードする](https://i.gyazo.com/cfec48fc58f2a17560ea2e7d0f71cc41.png)
**ログイン後の画面から `alf` ファイルをアップロードする**

5. Unity ライセンスの利用用途に応じて適切な選択肢を入力する (本記事では Personal ライセンスを選択)

![利用するライセンス形態を選択する (本記事では Personal ライセンスを選択)](https://i.gyazo.com/72239a40ef5b2474f34c814a68f8c61d.png)
**利用するライセンス形態を選択する (本記事では Personal ライセンスを選択)**

6. `Download license` ボタンをクリックして `ulf` ファイルをダウンロードする

![`Download license` ボタンをクリックして `ulf` ファイルをダウンロードする](https://i.gyazo.com/9ddc63dc321a68986bfedfb8a97c8f00.png)

これで Unity ライセンスファイルのアクティベートは完了です。次にアクティベートしたライセンスファイルを元に GitHub リポジトリの [Secrets](https://docs.github.com/ja/actions/reference/encrypted-secrets) に登録して、GameCI でテストが実行できるようにしていきます。

# GameCI を利用して Unity の PlayMode 及び EditMode テストを実行する

GitHub Actions 上でテストを実行するために、先ほどアクティベートした Unity ライセンスの情報を GitHub リポジトリ上で扱えるようにする必要があります。そのため、まずは Secrets に `ulf` ファイルの内容を登録することから始めます。

1. GameCI では Unity ライセンスの情報を Secrets から読み取るため、予め `ulf` ファイルの中身を登録しておく

![Github リポジトリの `Secrets` 登録画面に遷移する](https://i.gyazo.com/e126ae5e2fe9339d56047b8497808100.png)
**Github リポジトリの `Secrets` 登録画面に遷移する**

2. `Name` を `UNITY_LICENSE` で `Value` に `ulf` ファイルの中身をコピー & ペーストする。`ulf` ファイルの実態は XML ファイルでありテキストエディタでファイルを開くことができる

![`UNITY_LICENSE` として `ulf` ファイルの中身を登録する](https://i.gyazo.com/f52356a229caa4e31e3ae8268d53a4e6.png)
**`UNITY_LICENSE` として `ulf` ファイルの中身を登録する**

3. これで GameCI でテストやビルドの実行を行える環境が整った。今回は下記の各種テスト実行用のワークフローファイルを追加して動作検証する。

```yml:.github/workflows/test.yml
name: Run EditMode and PlayMode Test
on:
  workflow_dispatch: {}
jobs:
  test:
    name: Run EditMode and PlayMode Test
    runs-on: ubuntu-latest
    steps:
      - name: Check out my unity project.
        # actions/checkout@v2 を利用して作業ディレクトリに
        # Unity プロジェクトの中身をダウンロードしてくる
        uses: actions/checkout@v2
      - name: Run EditMode and PlayMode Test
        uses: game-ci/unity-test-runner@v2
        env:
          # 2. の手順で Secrets に登録した Unity ライセンスの情報を指定する
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
        with:
          projectPath: .
          githubToken: ${{ secrets.GITHUB_TOKEN }}
          # Unity プロジェクトのバージョンを指定する
          # ProjectSettings/ProjectVersion.txt に記載されているバージョンを入力すれば OK
          unityVersion: 2020.3.5f1
      # テストの実行結果をアーティファクトにアップロードして後から参照可能にする
      - uses: actions/upload-artifact@v2
        if: always()
        with:
          name: Test results
          path: artifacts
```

4. 3. のワークフローファイルをリモートリポジトリに push してワークフローを実行する。

![Unity のテスト実行を行うためのワークフローファイルを実行する](https://i.gyazo.com/9bace70734f1e99955f5b9aa068b31de.png)
**Unity のテスト実行を行うためのワークフローファイルを実行する**

5. ワークフローの実行が成功したら、詳細画面に遷移した後、`Test Results` の項目から実行結果を確認する。

![Unity の EditMode テストが正常に実行されている様子](https://i.gyazo.com/ffc31b5919431fedb516f1932a13ce62.png)
**Unity の EditMode テストが正常に実行されている様子**

テスト実行のためのワークフローファイルでは [`workflow_dispatch`](https://docs.github.com/ja/actions/reference/events-that-trigger-workflows#manual-events) で実行可能にしていますが、`pull_request` を利用すれば PR 時にテスト実行させたりすることが可能になります。

# おわりに

以前 Unity コマンドを駆使して自分で CI 環境を構築した経験があるのですが、GameCI を利用した方が全然楽に Unity CI 環境構築を GitHub Actions 上で行えました。

もちろん、[Unity Cloud Build](https://unity3d.com/jp/unity/features/cloud-build) を利用すれば CI 環境の構築は以前から楽に行えました。しかし、選択肢の 1 つとして GameCI を持っておくことで、サクッと GitHub Actions に統合する形で Unity の CI 環境を導入できるのは他には無いメリットを感じました。

ちなみに GameCI で利用されている Docker イメージは以前からよく使われていた [gableroux/unity3d](https://hub.docker.com/r/gableroux/unity3d/) が元になっているようでした。

本記事が Unity で CI 環境構築を求められている方の助けになれれば幸いです。

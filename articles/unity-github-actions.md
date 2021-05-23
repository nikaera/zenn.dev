---
title: "爆速で Unity の CI 環境を GitHub Actions で構築する"
emoji: "💨"
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

![`alf` ファイルをダウンロードする](https://i.gyazo.com/71b9dff8266c9bc990e1b709a5191535.png)
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

# おわりに

Unity での CI 環境構築にお悩みの方の参考になれれば幸いです。

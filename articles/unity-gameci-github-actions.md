---
dev_article_id: 707694
title: "GameCI で Unity の CI 環境を GitHub Actions で構築する"
emoji: "🎮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unity", "githubactions"]
published: false
---

# はじめに

先日同僚が Unity の CI 環境を構築するためのライブラリである [GameCI](https://game.ci/) について教えてくれました。早速 GameCI の GitHub Actions を利用して、サンプルプロジェクトで色々動作検証してみたところ、確かに Unity の CI 環境を楽に構築できることが分かりました。

もちろん、[Unity Cloud Build](https://unity3d.com/jp/unity/features/cloud-build) を利用すれば CI 環境の構築は以前から楽にできました。しかし、選択肢の 1 つとして GameCI を持っておくことで、**サクッと GitHub Actions に統合する形で Unity の CI 環境を導入できるのは他には無いメリットを感じました。**

本記事で紹介しているソースコード、及び検証時に利用したプロジェクトは GitHub にアップ済みですので、手っ取り早く内容を把握されたい方は下記をご参照くださいませ。

https://github.com/nikaera/Unity-GameCI-Sample

業務でも利用できそうなので、GameCI を利用して CI 環境を構築する手順を記事でまとめました。

# GameCI に備わっている機能紹介

GameCI には現状下記の GitHub Actions が用意されているようです。

| 機能                                                                   | 概要                                                                                                                                                                                                                                                                                        |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Activation](https://game.ci/docs/github/activation)                   | Unity ライセンスを任意の Unity バージョンで発行する                                                                                                                                                                                                                                         |
| [Test runner](https://game.ci/docs/github/test-runner)                 | Unity の PlayMode 及び EditMode のテストを実行する (テスト結果の出力にも対応)                                                                                                                                                                                                               |
| [Builder](https://game.ci/docs/github/builder)                         | 任意の Platform ビルドを実行する ([アーティファクト](https://docs.github.com/ja/actions/guides/storing-workflow-data-as-artifacts) 利用でダウンロードも可能)                                                                                                                                |
| [Returning a license](https://game.ci/docs/github/returning-a-license) | Unity ライセンスの返却ができる (Professional License のみ対応)                                                                                                                                                                                                                              |
| [Remote builder](https://game.ci/docs/github/remote-builder)           | GitHub Actions のスペックでは満足のいくビルドができない際に AWS 環境でハイスペックなマシンを用意してビルドできる。ビルドのためのインフラ構築には [AWS CloudFormation](https://aws.amazon.com/jp/cloudformation/) を使用している (現在は AWS のみ対応。今後 GCP, Azure にも対応予定とのこと) |
| [Deployment](https://game.ci/docs/github/deployment/android)           | Unity ビルドを各種 Platform 向けにデプロイする (iOS 及び Android のみ記載あり。厳密に言うと `Builder` でビルド出力した内容を [`fastlane`](https://fastlane.tools/) を用いてデプロイするためのワークフロー紹介になっている)                                                                  |

上記を見ると既に **GameCI には開発者として Unity CI に欲しい機能は最低限揃っているように見受けられました。** また本記事では、今後機会があれば試してみたいと考えていますが **Remote builder 及び Deployment** については言及していません。

今回は実例を交えながら **Activation 及び Test runner、Builder、Returning a license** の使用方法について紹介していきます。

## Activation: GameCI で必要となる Unity License のアクティベーションを行う

GameCI で Unity ライセンスをアクティベートするには [Activation](https://game.ci/docs/github/activation) を利用します。早速ドキュメントの手順に沿って作業を進めていきます。

まず CI を導入したい GitHub 上の Unity プロジェクトの `.github/workflows` 内に Unity ライセンスアクティベート用のワークフローファイルを作成します。

```yml:.github/workflows/activation.yml
name: Acquire activation file
on:
  workflow_dispatch: {}
jobs:
  activation:
    name: Request manual activation file 🔑
    runs-on: ubuntu-latest
    steps:
      # GameCI の Activation を利用して alf ファイルを発行する
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

その後、[デフォルトブランチ](https://docs.github.com/ja/github/setting-up-and-managing-your-github-user-account/managing-user-account-settings/managing-the-default-branch-name-for-your-repositories#about-management-of-the-default-branch-name) にプッシュして GitHub Actions で実行可能にしたら、下記手順に従い Unity ライセンスファイルのアクティベート及びダウンロードを行います。

![1. ブラウザから GitHub リポジトリにアクセスして、Unity ライセンスアクティベート用のワークフローを実行して `alf` ファイルを生成する](https://i.gyazo.com/bd276ca6dcf6a2c12ce9ff9569e08ce3.png)
**1. ブラウザから GitHub リポジトリにアクセスして、Unity ライセンスアクティベート用のワークフローを実行して `alf` ファイルを生成する**

![2. ワークフローの実行に成功したら、該当項目をクリックして詳細画面に遷移する](https://i.gyazo.com/2271b3cb35efc7f1c9d51702662cdac9.png)
**2. ワークフローの実行に成功したら、該当項目をクリックして詳細画面に遷移する**

![3. `Artifacts` の項目から `alf` ファイルをダウンロードする](https://i.gyazo.com/71b9dff8266c9bc990e1b709a5191535.png)
**3. `Artifacts` の項目から `alf` ファイルをダウンロードする**

![4. [Unity license manual activation webpage](https://license.unity3d.com) からログインして `alf` ファイルをアップロードする](https://i.gyazo.com/cfec48fc58f2a17560ea2e7d0f71cc41.png)
**4. [Unity license manual activation webpage](https://license.unity3d.com) からログインして `alf` ファイルをアップロードする**

![5. Unity ライセンスの用途に応じて適切な選択肢を入力する (本記事では Personal ライセンスを選択)](https://i.gyazo.com/72239a40ef5b2474f34c814a68f8c61d.png)
**5. Unity ライセンスの利用用途に応じて適切な選択肢を入力する (本記事では Personal ライセンスを選択)**

![6. `Download license` ボタンをクリックして `ulf` ファイルをダウンロードする](https://i.gyazo.com/9ddc63dc321a68986bfedfb8a97c8f00.png)
**6. `Download license` ボタンをクリックして `ulf` ファイルをダウンロードする**

これで Unity ライセンスファイルのアクティベートは完了です。**次にアクティベートしたライセンスファイルを GitHub リポジトリの [Secrets](https://docs.github.com/ja/actions/reference/encrypted-secrets) に登録して、GameCI で PlayMode 及び EditMode のテストが実行できるようにしていきます。**

::: message

`alf` 拡張子のファイルがライセンスリクエストファイルを指していて、Unity ライセンスの発行に必要となるファイルです。`ulf` 拡張子のファイルが Unity ライセンスのファイルです。[^1]

:::

## Test runner: PlayMode 及び EditMode テストを実行して結果を参照する

GitHub Actions 上でテストを実行するために、**先ほどアクティベートした Unity ライセンスの情報を ワークフロー上で扱えるようにする必要があります。そのため、まずは Secrets に `ulf` ファイルの内容を登録することから始めます。**

![1. Unity ライセンスの情報登録のため、Github リポジトリの `Secrets` 登録画面に遷移する](https://i.gyazo.com/e126ae5e2fe9339d56047b8497808100.png)
**1. Unity ライセンスの情報登録のため、Github リポジトリの `Secrets` 登録画面に遷移する**

![2. GameCI はライセンス情報参照のため `Secrets` の `UNITY_LICENSE` を参照する。そのため、`Name` を `UNITY_LICENSE` で `Value` に `ulf` ファイルの中身をコピー & ペーストしておく](https://i.gyazo.com/f52356a229caa4e31e3ae8268d53a4e6.png)
**2. GameCI はライセンス情報参照のため、デフォルト設定では `Secrets` の `UNITY_LICENSE` を参照する。そのため、`Name` が `UNITY_LICENSE`、`Value` には `ulf` ファイルの中身を登録する[^2]**

上記作業で GameCI でテストやビルド実行を行える環境が整ったので、動作検証のためテスト実行用のワークフローファイルを作成します。

```yml:.github/workflows/test.yml
name: Run EditMode and PlayMode Test
on:
  workflow_dispatch: {}
jobs:
  test:
    name: Run EditMode and PlayMode Test
    runs-on: ubuntu-latest
    steps:
      # actions/checkout@v2 を利用して作業ディレクトリに
      # Unity プロジェクトの中身をダウンロードしてくる
      - name: Check out my unity project.
        uses: actions/checkout@v2

      # GameCI の Test runner を利用して
      # EditMode 及び PlayMode のテストを実行する
      - name: Run EditMode and PlayMode Test
        uses: game-ci/unity-test-runner@v2
        env:
          # 2. の手順で Secrets に登録した Unity ライセンスの情報を指定する
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}

          # もし Professional license を使いたい場合は、
          # メールアドレス、パスワード、シリアルナンバーを入力する必要がある
          # ref: https://game.ci/docs/github/test-runner#professional-license
          # UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          # UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
          # UNITY_SERIAL: ${{ secrets.UNITY_SERIAL }}
        with:
          projectPath: .
          # テストの実行結果もみたい場合は githubToken を指定する
          # secrets.GITHUB_TOKEN は Secrets 未登録でも利用可能
          githubToken: ${{ secrets.GITHUB_TOKEN }}

          # Unity プロジェクトのバージョンを指定する
          # ProjectSettings/ProjectVersion.txt に記載されているバージョンを入力すれば OK
          unityVersion: 2020.3.5f1

          # 実行したいテストの種類を指定できる
          # 指定可能な値は All, PlayMode, EditMode
          # testMode: All

          # テスト実行時に利用したい Docker イメージを明示的に指定できる
          # customImage: 'unityci/editor:2020.1.14f1-base-0'

      # テストの実行結果をアーティファクトにアップロードしてダウンロードして参照できるようにする
      - uses: actions/upload-artifact@v2
        if: always()
        with:
          name: Test results
          path: artifacts
```

上記のワークフローファイルを GitHub Actions 上で動作検証する際の手順は下記になります。

![1. Unity のテストを実行するためのワークフローを選択して実行する](https://i.gyazo.com/9bace70734f1e99955f5b9aa068b31de.png)
**1. Unity のテストを実行するためのワークフローを選択して実行する**

![2. ワークフローの実行が成功したら、詳細画面に遷移した後、`Test Results` の項目からテストの実行結果を確認する](https://i.gyazo.com/ffc31b5919431fedb516f1932a13ce62.png)
**2. ワークフローの実行が成功したら、詳細画面に遷移した後、`Test Results` の項目からテストの実行結果を確認する**

テスト実行用のワークフローファイルでは [`workflow_dispatch`](https://docs.github.com/ja/actions/reference/events-that-trigger-workflows#manual-events) で実行可能にしていますが、`pull_request` を利用すればプルリク時にテストを実行させることが可能になります。

## Builder: プロジェクトのビルドを実行して出力結果を確認する

GameCI にはプロジェクトのビルドを行うための GitHub Actions も用意されています。**実際に GameCI で WebGL ビルドを行いその内容を GitHub Pages で確認できるようにして動作検証していきます。**

早速 WebGL ビルドを行うためのワークフローファイルを作成していきます。

```yml:.github/workflows/webgl_build.yml
name: Run the WebGL build
on:
  workflow_dispatch: {}
jobs:
  build:
    name: Run the WebGL build
    runs-on: ubuntu-latest
    steps:
      # actions/checkout@v2 を利用して作業ディレクトリに
      # Unity プロジェクトの中身をダウンロードしてくる
      - name: Check out my unity project.
        uses: actions/checkout@v2

      # GameCI の Builder を利用して、
      # Unity プロジェクトのビルドを実行する
      - name: Run the WebGL build
        uses: game-ci/unity-builder@v2
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
        with:
          # 今回は WebGL ビルドを行いたいため WebGL を指定する
          # WebGL 以外の指定可能な値は下記に記載の値が利用可能
          # ref: https://docs.unity3d.com/ScriptReference/BuildTarget.html
          targetPlatform: WebGL

          unityVersion: 2020.3.5f1
      # Builder で出力した WebGL ビルドを GitHub Pages にデプロイする
      - name: Deploy to GitHub Pages
        uses: JamesIves/github-pages-deploy-action@4.1.3
        with:
          # GitHub Pages デプロイ用の Orphan ブランチ名を指定する
          branch: gh-pages

          # デプロイ用ビルドフォルダパスを指定する
          # GameCI の Builder はデフォルトでは build フォルダにビルド内容を出力する
          folder: build

      # Builder で出力した WebGL ビルドをアーティファクトでダウンロード可能にする
      - name: Upload the WebGL Build
        uses: actions/upload-artifact@v2
        with:
          name: Build
          path: build

```

上記のワークフローファイルを GitHub Actions 上で動作検証する際の手順は下記になります。

![1. Unity の WebGL ビルドを実行するためのワークフローを実行する](https://i.gyazo.com/b8f43baef2e2edbf8d5510ce8f58172d.png)
**1. Unity の WebGL ビルドを実行するためのワークフローを実行する**

![2. ワークフローの実行が成功したら、詳細画面に遷移した後、ビルド内容が正常か確認する](https://i.gyazo.com/a102d1f5cb4d0581746770d1d9cad965.png)
**2. ワークフローの実行が成功したら、詳細画面に遷移した後、ビルド内容が正常そうか確認する**

![3. ビルド内容を確認するための GitHub Pages の設定を Settings から行う](https://i.gyazo.com/7d39c035b52ba3862d5dad18213a2041.png)
**3. ビルド内容を確認するための GitHub Pages の設定を Settings から行う**

![4. GitHub Pages でブラウザから WebGL ビルドの動作確認をする](https://i.gyazo.com/b2e1bd86ca9d3b3f207a5ddbb1de3ba7.png)
**4. GitHub Pages でブラウザから WebGL ビルドの動作確認をする**

**上記のように Builder を利用することで WebGL ビルドの成否及び、最新のビルド内容を常に GitHub Pages で見られるようにできます。** すると WebGL ビルドが正常かどうかの確認が常に GitHub Pages を見れば把握できるようになるため、[Unity1 週間ゲームジャム](https://unityroom.com/unity1weeks) などに参加する際で便利に活用できそうです。

:::message

WebGL ビルドを行う際、Unity バージョンやアセットの対応状況によっては正しくブラウザ上で動作しないビルドが出力されます。**ただし、ブラウザで発生するエラー内容によっては WebGL のビルド設定を見直すだけで解決できる場合があります。** 例えば `unityframework is not defined` というエラーが発生した際は、[この記事](https://qiita.com/aguroshou0413/items/1451a6779a92acb96b78) のように WebGL の `Build Settings` を見直すことで解決できる場合があります。

:::

:::message

私の環境では [`JamesIves/github-pages-deploy-action`](https://github.com/JamesIves/github-pages-deploy-action) で GitHub Pages へのデプロイを行った際、**デフォルトでは `/WebGL/WebGL` フォルダにビルド内容が出力されました。そのため、ブラウザから WebGL ビルドにアクセスする際、`<GitHub Pages の URL>/WebGL/WebGL` のような URL にアクセスする必要がありました。**

:::

## Returning a license: GameCI で利用している Unity License を返却する

**通常利用することは無いと[公式サイト](https://game.ci/docs/github/returning-a-license)にも書かれていますが、Professional License の返却も GameCI で行うことが可能です。** 今回は Personal License を利用したため使用しませんでしたが、下記をワークフローのステップに組み込むことで Professional License を返却できるようです。

```yml
# ...
# どこかのタイミングでライセンスのアクティベートを行う
- name: Activate Unity
  uses: game-ci/unity-activate@v1
  env:
    UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}

#...
# ステップの最後などに game-ci/unity-return-license@v1 を呼び出して
# アクティベート済みのライセンスを返却する
- name: Return license
  uses: game-ci/unity-return-license@v1
  if: always()
```

# おわりに

以前 Unity コマンドを駆使して自分で CI 環境を構築した経験があるのですが、
GameCI を利用した方が全然楽に Unity CI 環境構築を GitHub Actions 上で行えました。

ちなみに [GameCI で利用されている Docker イメージ](https://game.ci/docs/docker/docker-images) は以前からよく使われていた [gableroux/unity3d](https://hub.docker.com/r/gableroux/unity3d/) が元になっているようでした。ってか [GabLeRoux さんのホームページ](https://gableroux.com/about/) を見たら、GameCI の開発を始めた方のようでした。すごい。

本記事が GitHub Actions で Unity CI 環境構築を始めようとしている方の助けになれれば幸いです。

# 参考リンク

- [GameCI - The fastest and easiest way to automatically test and build your game projects](https://game.ci/)
- [Services \- Cloud Build \- Unity](https://unity3d.com/jp/unity/features/cloud-build)
- [AWS CloudFormation（テンプレートを使ったリソースのモデル化と管理）\| AWS](https://aws.amazon.com/jp/cloudformation/)
- [fastlane \- App automation done right](https://fastlane.tools/)
- [リポジトリのデフォルトブランチ名を管理する \- GitHub Docs](https://docs.github.com/ja/github/setting-up-and-managing-your-github-user-account/managing-user-account-settings/managing-the-default-branch-name-for-your-repositories#about-management-of-the-default-branch-name)
- [Unity でパーソナルライセンスのシリアルナンバーを発行する \| Yucchiy's Note](https://blog.yucchiy.com/2019/01/08/how-to-get-unity-free-license/)
- [Unity license manual activation webpage](https://license.unity3d.com)
- [暗号化されたシークレット \- GitHub Docs](https://docs.github.com/ja/actions/reference/encrypted-secrets)
- [Unity \- Scripting API: BuildTarget](https://docs.unity3d.com/ScriptReference/BuildTarget.html)
- [Unity 1 週間ゲームジャム \| フリーゲーム投稿サイト unityroom](https://unityroom.com/unity1weeks)
- [Unity2020 WebGL 9 割まで読み込めるがアプリが起動しない不具合の解決方法 \- Qiita](https://qiita.com/aguroshou0413/items/1451a6779a92acb96b78)
- [Deploy to GitHub Pages · Actions · GitHub Marketplace](https://github.com/marketplace/actions/deploy-to-github-pages)

[^1]: `alf` ファイル及び `ulf` ファイルの実態は XML ファイルです。
[^2]: 適当なテキストエディタで `ulf` ファイルを開き全文をコピー & ペーストします。

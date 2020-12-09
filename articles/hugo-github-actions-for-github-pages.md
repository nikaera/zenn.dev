---
title: "Hugo + GitHub Pages + GitHub Actions で独自ドメインのウェブサイトを構築する"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hugo", "githubactions", "githubpages"]
published: true
---

# はじめに

![Hugo のトップページ](https://i.gyazo.com/ffe8fe276b9d008461880581002430ec.png)

Zenn や Qiita, note など様々なウェブサービスで記事を書くにつれて、ふと雑多な内容で自分の好き勝手に記事を書いて公開できる自分のブログが欲しくなりました。そこで、自分のブログを作ろうと思い調査したところ、SSG で作るのが手っ取り早そうだったのと、その中でも一番ラクにウェブサイトが構築できそうな [Hugo](https://gohugo.io/) を採用しました。

また、デプロイは簡単に行いたかったので、デプロイ先として [GitHub Pages](https://docs.github.com/ja/free-pro-team@latest/github/working-with-github-pages/about-github-pages) を採用しました。
独自ドメインの割り当てから HTTPS 対応まで無料でできるかつ、使い慣れている GitHub をデプロイ先に使えることが決め手でした。

Hugo で自分のブログを構築して GitHub Pages で公開できるようになったのですが、ブログ内容を更新したり記事を書くたびにビルドしてデプロイをするのが、意外と面倒なことに気づきました。そこで、[GitHub Pages action](https://github.com/marketplace/actions/github-pages-action) を用いて、ビルドしてデプロイするという作業は自動化しました。

上記までの作業をすることで、自分のブログを書いたり更新することだけに集中できる環境を整えることができました。ウェブサイトを作りこむ以外は、簡単ないくつかの作業をするだけで Hugo で満足のいく自分のブログを書く環境が整えられたので、その手順についてまとめてみました。

ちなみに本記事の手順で実際に作成した私のブログは下記です。
https://nikaera.com/

# Hugo を PC にインストールする

私は Windows には [Chocolatey](https://chocolatey.org/)、Mac では [Homebrew](https://brew.sh/index_ja) で Hugo をインストールしました。Chocolatey や Homebrew を利用したインストール方法については [公式サイトの手順](https://gohugo.io/getting-started/installing/) で公開されています。

```bash:install-with-homebrew.sh
# Mac で Homebrew を使って Hugo をインストールする
brew install hugo
```

```powershell:install-with-chocolatey.ps1
# Windows で Chocolatey を使って Hugo をインストールする
choco install hugo -confirm

# Sass/SCSS を用いてウェブサイトのデザイン改修を行いたい場合は
# 下記で Chocolatey を使って Hugo をインストールする
choco install hugo-extended -confirm
```

# Hugo で自分のウェブサイトを構築する

## Hugo プロジェクトを作成する

下記コマンドで Hugo のプロジェクトフォルダを生成できます。

```bash:create-new-site.sh
hugo new site <プロジェクトフォルダのパス>
```

Hugo のコンフィグファイルのデフォルトフォーマットは [TOML](https://github.com/toml-lang/toml.io/blob/main/specs/ja/v1.0.0-rc.2.md) 形式なのですが、`hugo new site <プロジェクトフォルダへのパス> -f json` のように `-f` オプションを付与することで JSON フォーマットでもコンフィグファイルを生成可能です。

後述しますが、一部の Hugo のテーマはコンフィグファイルのサンプルが JSON ファイルで書かれています。その場合は、新規で設定するコンフィグファイルのフォーマットも JSON で統一しておくと各種設定項目の調整が楽になりそうです。

:::message info

もしくは[こちら](https://github.com/sclevine/yj)のようなコンバーターを使用したり、[こちら](https://pseitz.github.io/toml-to-json-online-converter/)のようなウェブのコンバーターを使用して、設定ファイルを JSON から TOML フォーマットに変更しても良さそうです。

:::

## ウェブサイトのテーマを探す

テーマとは、ウェブサイトのデザインテンプレートのことです。テーマは [Hugo Themes](https://themes.gohugo.io/) で公開されているので、この中から自分の好みを選びます。Hugo ではテーマの内容をカスタマイズしたり差し替えることもできるので、あとから一部デザインを自分好みに修正できます。

![Hugo Themes にアクセスした時のページ](https://i.gyazo.com/294b076125052fe8bd9574c5bd0d1f0b.png)
**[Hugo Themes](https://themes.gohugo.io/) にアクセスした時のページ**

お気に入りのテーマが見つかったら、`Download` ボタンをクリックしてテーマの GitHub URL を控えておきます。

![Kiera というテーマのページ](https://i.gyazo.com/36ce79358e416ff6c92e7ad405c2abfa.png)
**[Kiera](https://themes.gohugo.io/hugo-kiera/) というテーマのページ**

## Hugo プロジェクトにテーマを適用する

テーマの適用方法については [公式サイトの手順](https://gohugo.io/getting-started/quick-start/#step-3-add-a-theme) で紹介されていますが、Hugo のプロジェクトフォルダのルートで下記コマンドを実行して、コンフィグファイルに `theme` を追記するだけです。

```bash:apply-template.sh
git clone <テーマの GitHub URL> themes/<テーマ名> --depth=1
echo 'theme = "<テーマ名>"' >> config.toml
```

どのテーマも基本的に適用方法は同じで、例えば私が採用したテーマである [PaperMod](https://themes.gohugo.io/hugo-papermod/) を適用する場合は下記コマンドを実行することになります。

```bash:apply-papermod-template.sh
git clone https://github.com/adityatelange/hugo-PaperMod themes/hugo-PaperMod --depth=1
echo 'theme = "hugo-PaperMod"' >> config.toml
```

これで自分のウェブサイトを作り込んでいくための準備が整いました。

## ウェブサイトを作り込む

**各テーマには `exampleSite` というウェブサイト作成の参考になる Hugo のプロジェクトフォルダが存在します。** 最初は `exampleSite` フォルダ内に存在する各種ファイルを自分のプロジェクトにコピー&ペーストして、改変しながらウェブサイトを作り込んでいくとスムーズに作業が進められます。

`exampleSite` フォルダは大抵 GitHub プロジェクトのルートに存在しているのですが、**GitHub のブランチで管理しているものもあります。** 私の採用した PaperMod は GitHub の `exampleSite` ブランチで管理していました。

上記のような詳細は Hugo のテーマページに記載があるはずなので、事前に `exampleSite` フォルダが配置されている場所や設定可能な項目等をチェックしておくことをオススメします。実際のウェブサイトの見た目を確認するには、Hugo のプロジェクトフォルダのルートで下記コマンドを実行します。

```bash
hugo server
```

`hugo server` 実行中に、ウェブサイトの設定を変更したり記事を追加すると、自動的にビルドが実行されるので常に最新のウェブサイトの見た目を確認できます。また、その際にエラーもコンソールに出力されるので、適宜修正しながらウェブサイトの作成を進めていくことが可能です。

# GitHub にデプロイ環境を整える

ウェブサイトの構築が出来ればあとはデプロイするだけです。今回は GitHub Actions でデプロイするため、残りの作業は GitHub 上で進めていきます。一応 GitHub Actions を使わずに手動でデプロイする手順については、Hugo の [公式サイトの手順](https://gohugo.io/hosting-and-deployment/hosting-on-github/) にて紹介されています。

GitHub Actions を用いてデプロイできるようにした際の利点としては下記があります。

- デプロイ時に毎回手動で複数コマンド実行する手間が省ける
- 自動化することでデプロイ時のコマンドミスを防ぐことが可能
- **常に統一した Hugo のビルド環境が利用可能**

最初のうちは私も公式サイトの手順通りに Mac で手動デプロイしていました。しかし、Windows 環境でデプロイ作業した際、本番環境デプロイ時に [SRI](https://gohugo.io/hugo-pipes/fingerprint/) 関連のエラーが発生してしまい、ウェブサイトに stylesheet が適用されないバグが発生してしまいました。

結局原因はよく分からなかったのですが、GitHub Actions 経由でデプロイするようにしたところ直りました。**CI 経由でデプロイできるようになると、こういった実行環境の違いによる挙動も気にする必要が無くなります。**

:::message info

一応 Windows 環境で発生した SRI 関連のバグは Hugo で該当するテンプレートファイルを `layouts` フォルダを利用して差し替えて、integrity の設定内容を空にすることで、本番環境でも stylesheet が適用できるようになったことは確認しました。[詳細はこちら](https://stackoverflow.com/a/65052963)。

:::

## 自分のウェブサイトをデプロイするためのリポジトリを作成する

GitHub の [公式サイトの手順](https://pages.github.com/) に従って、自身のウェブサイトをデプロイするためのリポジトリを作成します。また本記事では、**Hugo プロジェクトのリポジトリはデプロイ用リポジトリとは別で扱うため、Hugo プロジェクトのリポジトリも新たに作成します。**

本記事では Hugo プロジェクトのリポジトリ名は `hugo-blog` という名前に設定した前提で進めていきます。また作成したリモートリポジトリの情報は、下記コマンドで Hugo プロジェクトに登録しておきます。

```bash
# Hugo プロジェクトフォルダのルートで Git リポジトリを作成する
git init

# Hugo プロジェクトのリポジトリ情報を登録しておく (hugo-blog の GitHub URL)
git remote add origin <Hugo プロジェクトのリポジトリの URL>
```

## GitHub Actions で Hugo のビルドからデプロイまでを自動化するための環境を整える

今回は `hugo-blog` という Hugo プロジェクトのリポジトリから、デプロイ用リポジトリへデプロイしたいため、まずはデプロイ用リポジトリでデプロイキーを登録します。[公式サイトの手順](https://docs.github.com/ja/free-pro-team@latest/developers/overview/managing-deploy-keys#%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%82%AD%E3%83%BC) に従いデプロイキーを登録したら、秘密鍵を `hugo-blog` リポジトリのシークレットに登録します。

シークレットは [公式サイトの手順](https://docs.github.com/ja/free-pro-team@latest/actions/reference/encrypted-secrets#%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E3%81%AE%E6%9A%97%E5%8F%B7%E5%8C%96%E3%81%95%E3%82%8C%E3%81%9F%E3%82%B7%E3%83%BC%E3%82%AF%E3%83%AC%E3%83%83%E3%83%88%E3%81%AE%E4%BD%9C%E6%88%90) に従って登録します。今回は秘密鍵をシークレットに登録する際の名前に `ACTIONS_DEPLOY_KEY` を設定した前提で進めていきます。

次に [GitHub Pages action](https://github.com/marketplace/actions/github-pages-action) を導入して Hugo のビルドから公開までを自動化する環境をセットアップします。

`hugo-blog` リポジトリに GitHub Pages action の [Getting started](https://github.com/marketplace/actions/github-pages-action#getting-started) を参考にワークフローファイルを追加します。`- name: Deploy` の項目のみデプロイ用リポジトリへデプロイするために設定内容を変更します。

```yaml:.github/workflows/gh-pages.yml
name: github pages

on:
  push:
    branches:
      - main  # Set a branch name to trigger deployment

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.78.2'

      - name: Build
        run: hugo --minify

    #   - name: Deploy
    #     uses: peaceiris/actions-gh-pages@v3
    #     with:
    #       github_token: ${{ secrets.GITHUB_TOKEN }}
    #       publish_dir: ./public
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          external_repository: nikaera/nikaera.github.io
          publish_branch: main
          cname: nikaera.com
```

`- name: Deploy` の項目で設定した各種パラメータは下記になります。

| キー | 説明 |
| ------- | ------- |
| deploy_key | デプロイ時に使用する秘密鍵 |
| external_repository | デプロイ先のリモートリポジトリ |
| publish_branch | デプロイ先のリモートリポジトリのブランチ |
| cname | 設定するカスタムドメイン名 |

`deploy_key` にはシークレットに登録した秘密鍵を設定します。`external_repository` には Hugo をビルドした際のデプロイ先リポジトリを `<ユーザ名>/<リポジトリ名>` のフォーマットで指定します。`publish_branch` はデプロイ先として使用するブランチ名になります。`cname` には自分が設定したいドメイン名を指定します。`cname` の [詳細](https://docs.github.com/ja/free-pro-team@latest/github/working-with-github-pages/managing-a-custom-domain-for-your-github-pages-site) はこちらからご確認いただけます。

## カスタムドメインで設定した内容で DNS 設定を書き換える

GitHub Pages にカスタムドメインを利用する際は、該当するドメインの DNS レコードの設定で CNAME レコードもしくは A レコードを設定する必要があります。[公式サイトの手順](https://docs.github.com/ja/free-pro-team@latest/github/working-with-github-pages/managing-a-custom-domain-for-your-github-pages-site) に従って設定します。

また、カスタムドメインの設定後は特別な理由がない限りは、デプロイ用リポジトリで [HTTPS 強制の設定](https://docs.github.com/ja/free-pro-team@latest/github/working-with-github-pages/securing-your-github-pages-site-with-https) をしておくことオススメします。

:::message info

ちなみに GitHub Pages で利用している証明書は [Let's Encrypt](https://github.blog/2018-05-01-github-pages-custom-domains-https/) のものになります。

:::

設定作業はこれで完了です。あとは実際に Hugo プロジェクトを更新後、`hugo-blog` リポジトリに push することでビルドからデプロイまで GitHub Actions で行われるようになったかを確認していきます。


## Hugo プロジェクトの更新時に自動でデプロイが行われるか確認する

Hugo プロジェクトには既に `hugo-blog` リポジトリの GitHub URL が `origin` として設定されているはずなので、下記で実際に `hugo-blog` リポジトリへ push してみます。

```bash
# Hugo プロジェクトを hugo-blog リポジトリに push する
git add -A
git commit -m "initial commit"
git push origin main
```

次は実際に GitHub リポジトリの `Actions` タブから、GitHub Pages action のワークフローが実行されているか確認します。

![GitHub Pages actions の実行に成功した様子](https://i.gyazo.com/d4c3c5477c0f149ae84464fcc63c593b.png)
**GitHub Pages actions の実行に成功した様子**

無事ワークフローの実行に成功したことを確認したら、デプロイ用リポジトリの様子を確認します。

![デプロイ用リポジトリに Hugo の最新ビルドが push されていることを確認する](https://i.gyazo.com/3a0eb5ca58701a39bb922ea14477488c.png)
**デプロイ用リポジトリに Hugo の最新ビルドが push されていることを確認する**

最新コミットのメッセージに `deploy: <ユーザ名>/hugo-blog@<コミットハッシュ>` のコメントが表示されているはずです。コメントのリンクをクリックすると、`hugo-blog` リポジトリの最新コミットの確認画面に遷移するはずです。

ここまで確認できれば、あとはローカルで `hugo server` して確認していたウェブサイトと同じように、GitHub Pages 上でもウェブサイトが見えるか実際に確認してみます。

デプロイ用リポジトリ名が `<ユーザ名>/<ユーザ名>.github.io` であった場合、`https://<ユーザ名>.github.io` でウェブサイトにアクセスできます。私の場合は `https://nikaera.github.io` になります。その際、カスタムドメインにリダイレクトすることも確認できれば、カスタムドメインの設定も正常に反映されています。

![ページが正常に表示できカスタムドメインでもアクセスできている様子](https://i.gyazo.com/b3c061569dae33a1dd58f4741c6d77cc.png)
**ページが正常に表示されていてカスタムドメインでアクセスできた様子**

アクセス時に意図したページが表示されていることが確認できれば大丈夫です。これで自分のウェブサイトを更新したら `hugo-blog` リポジトリに Hugo プロジェクトを push するだけで自分のウェブサイトを自動更新できるようになりました。

# おわりに

今回は Hugo を例にしましたが、本記事内で紹介した [GitHub Pages action](https://github.com/marketplace/actions/github-pages-action) は [様々な SSG](https://jamstack.org/generators/) に対応しています。そのため Hugo ではウェブサイトのカスタマイズに限界が来たなと感じたら [Next.js](https://nextjs.org/) や [Gatsby](https://www.gatsbyjs.com/) に乗り換えるといったことも可能です。

また Hugo では何も考えずとも、マークダウンで記事が書けてビルドも高速なので、手っ取り早く自分のウェブサイトを構築してみたいという用途にはピッタリだと感じました。

関係ないですが、Hugo でウェブサイト構築する際の知見も記事に含めてしまったせいで、文章量が意図した倍近い量になってしまいました。。簡潔で分かりやすい文章が書けるようにならなきゃ。。

# 参考リンク

- [The world’s fastest framework for building websites | Hugo](https://gohugo.io/)
- [Complete List | Hugo Themes](https://themes.gohugo.io/)
- [GitHub Pages について - GitHub Docs](https://docs.github.com/ja/free-pro-team@latest/github/working-with-github-pages/about-github-pages)
- [GitHub Pages action](https://github.com/marketplace/actions/github-pages-action)
- [Install Hugo | Hugo](https://gohugo.io/getting-started/installing/)
- [Quick Start | Hugo](https://gohugo.io/getting-started/quick-start/#step-3-add-a-theme)
- [Host on GitHub | Hugo](https://gohugo.io/hosting-and-deployment/hosting-on-github/)
- [GitHub Pages | Websites for you and your projects, hosted directly from your GitHub repository. Just edit, push, and your changes are live.](https://pages.github.com/)
- [デプロイキーの管理 - GitHub Docs](https://docs.github.com/ja/free-pro-team@latest/developers/overview/managing-deploy-keys#%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%82%AD%E3%83%BC)
- [暗号化されたシークレット - GitHub Docs](https://docs.github.com/ja/free-pro-team@latest/actions/reference/encrypted-secrets#%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E3%81%AE%E6%9A%97%E5%8F%B7%E5%8C%96%E3%81%95%E3%82%8C%E3%81%9F%E3%82%B7%E3%83%BC%E3%82%AF%E3%83%AC%E3%83%83%E3%83%88%E3%81%AE%E4%BD%9C%E6%88%90)
- [GitHub Pages サイトのカスタムドメインを管理する - GitHub Docs](https://docs.github.com/ja/free-pro-team@latest/github/working-with-github-pages/managing-a-custom-domain-for-your-github-pages-site)
- [HTTPS で GitHub Pages サイトを保護する - GitHub Docs](https://docs.github.com/ja/free-pro-team@latest/github/working-with-github-pages/securing-your-github-pages-site-with-https)

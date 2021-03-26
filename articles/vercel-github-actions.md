---
dev_article_id: 640773
title: "Vercel の定期デプロイを GitHub Actions で実現する"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vercel", "githubactions"]
published: true
---

# はじめに

最近 [catnose99](https://zenn.dev/catnose99) さんの [チーム個々人のテックブログを RSS で集約するサイトを作った（Next.js）](https://zenn.dev/catnose99/articles/cb72a73368a547756862) を利用させていただく形で [会社のテックブログ](https://tech.kadinche.com/) を構築しました。

https://tech.kadinche.com/

[Team Blog Hub](https://github.com/catnose99/team-blog-hub) が [Next.js](https://nextjs.org/) で開発されているので、デプロイ先は [Vercel](https://vercel.com/) に決めました。その際に記事を自動的に更新するために、[GitHub Actions](https://github.co.jp/features/actions) で定期的にビルドを回すようにしました。また、独自ドメインの設定も行いました。

**本記事では、Vercel プロジェクトを GitHub Actions でビルドしてデプロイするための手順についてまとめていきます。またおまけとして、Vercel での独自ドメインの設定方法、及び弊社の Team Blog Hub の運用方針についても書いていきます。**

なお、**記事は下記の前提で書いておりますのでご注意くださいませ。**

- Vercel でサインアップ済み
  - サインアップ済みでない方は[こちら](https://vercel.com/signup)からサインアップ可能です
- Vercel でプロジェクト作成済み
  - プロジェクト未作成の方は[こちら](https://vercel.com/docs/platform/projects)から作成可能です
- [Vercel の GitHub 連携](https://vercel.com/docs/git#deploying-a-git-repository)でプロジェクトへのリポジトリの紐付け済み

# 動作環境

- Node.js v15.6.0
- Vercel CLI 21.1.0

# GitHub Actions で Vercel へデプロイする準備を行う

まずは GitHub Actions で Vercel へのビルド & デプロイを行うための環境を整える必要があります。そのためには、**各種必要となる情報を予め Vercel から取得しておく必要があります。**

## Vercel デプロイ時に必要となるトークンを取得する

Vercel にログイン後、トップページ右上のアイコンからトークンを発行画面に遷移します。

![スクリーンショット 2021-01-27 5.14.38.png](https://i.gyazo.com/b5b135d2b441d810f516d12a4523f7f8.png)
**トップページ右上のアイコンから Settings メニューをクリックする**

![スクリーンショット 2021-01-27 5.17.27.png](https://i.gyazo.com/a5817e0dd7afb924534f15cd955087e4.png)
**トークンを発行して、その内容を控える (GitHub Actions での Vercel デプロイ時に必要となる)**

発行したトークンは GitHub Actions でのデプロイ時に利用します。

## Vercel デプロイ時に必要となる情報を取得する

Verce には[公式が提供している CLI](https://vercel.com/download) が存在するので、まずはそちらをインストールします。
インストールコマンドは下記になります。

```bash
# npm の場合
npm i -g vercel

# yarn の場合
yarn global add vercel
```

その後、Vercel プロジェクトに紐付けた GitHub リポジトリのルートで `vercel` コマンドを実行して、手順に従って Vercel プロジェクトとの紐付けを行います。コマンドの実行に成功すると `.vercel/project.json` というファイルの生成が確認できるはずです。

**`.vercel/project.json` には GitHub Actions 経由でのデプロイ時に必要となる [vercel-project-id と vercel-org-id](https://github.com/amondnet/vercel-action#inputs) の内容が記載されています。**

```json:.vercel/project.json
{"projectId":"<vercel-project-id の内容>","orgId":"<vercel-org-id の内容>"}
```

上記の内容を GitHub リポジトリの [Secrets](https://docs.github.com/ja/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository) に登録していきます。

# GitHub Actions で定期的に Vercel へデプロイする

GitHub リポジトリの Secrets に `.vercel/project.json` の内容を登録します。今回は `ORG_ID` `PROJECT_ID` `VERCEL_TOKEN` を Secrets に登録します。

| キー         | 値                                    |
| ------------ | ------------------------------------- |
| ORG_ID       | `.vercel/project.json` の `orgId`     |
| PROJECT_ID   | `.vercel/project.json` の `projectId` |
| VERCEL_TOKEN | Vercel で発行したトークン             |

![スクリーンショット 2021-01-27 5.23.06.png](https://i.gyazo.com/f630e51d4c02f1ec0362712ef2cd630c.png)
**GitHub Actions 経由で Vercel へデプロイするのに必要な値を Secrets に登録しておく**

シークレットへ必要な情報が登録できたら、GitHub Actions のワークフローファイルを作成します。**今回は Vercel 公式が用意している [Vercel Action](https://github.com/marketplace/actions/vercel-action) を利用します。**

```yml:deploy_website.yml
name: deploy website
on:
  # 一応動確のために手動で GitHub Actions を実行可能にする
  # その際の引数として checkout 時の ref を渡している
  # default 部分はリポジトリに設定されているデフォルトブランチを指定する
  workflow_dispatch:
    inputs:
      ref:
        description: branch|tag|SHA to checkout
        default: 'main'
        required: true
  # 毎日日本時間の 11時 に GitHub Actions が実行される (cron の時刻は UST)
  # 実行の際に参照されるブランチは上記の default で指定したものが使用される
  schedule:
    - cron:  '0 2 * * *'
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          ref: ${{ github.event.inputs.ref }}
      - uses: actions/setup-node@v2
        with:
          node-version: '15'
      # 投稿内容を更新するために npm run build:posts を走らせる
      - name: Recreate all posts
        shell: bash
        run: |
          npm install
          npm run build:posts
      - uses: amondnet/vercel-action@v20
        with:
          # GitHub Actions の Secrets で作成した値を参照する形で
          # Vercel デプロイ時に必要となる各種パラメタを設定する
          vercel-token: ${{ secrets.VERCEL_TOKEN }} # Required
          vercel-args: '--prod' # Optional
          vercel-org-id: ${{ secrets.ORG_ID}}  #Required
          vercel-project-id: ${{ secrets.PROJECT_ID}} #Required
          working-directory: ./

```

上記ファイルを作成次第、GitHub リポジトリのデフォルトブランチにコミットして、GitHub Actions のワークフローを手動実行してみて正常にデプロイできそうか確認していきます。

![スクリーンショット 2021-01-27 5.46.23.png](https://i.gyazo.com/ce6d4ef418f097e0254a0554d0692b0f.png)
**GitHub リポジトリのページに遷移後、実際に GitHub Actions のワークフローを実行してみる**

![スクリーンショット 2021-01-27 6.00.48.png](https://i.gyazo.com/87647579244d9f2e5448d7c9de337499.png)
**GitHub Actions のワークフローが実行されていそうか確認する**

![スクリーンショット 2021-01-27 6.04.43.png](https://i.gyazo.com/8001e72df8b0cd733a75cb419efaef2c.png)
**GitHub Actions のワークフローが正常に実行完了していることを確認する**

GitHub Actions が正常に実行されていそうなことが確認できたら、Vercel 側で正常にデプロイが完了していそうかを確認します。**Vercel のトップページに遷移後、該当するプロジェクトページに遷移して、 `Deployments` タブからデプロイ履歴を確認します。**

![スクリーンショット 2021-01-27 6.09.55.png](https://i.gyazo.com/c87b2faa3d1d7e0a06f056a218849ac8.png)
**Vercel で該当するプロジェクトページに遷移して、デプロイ履歴を確認する**

これで一通りの作業は終了です。お疲れさまでした。

# (おまけ) 独自ドメインを紐付ける

Vercel には [Custom Domains](https://vercel.com/docs/custom-domains) という独自ドメインを紐付けるための機能が備わっています。こちらを利用すると独自ドメイン経由で Vercel プロジェクトにアクセスさせることが可能になります。

[ルートドメインの場合は A レコードを、サブドメインの場合は CNAME レコード](https://vercel.com/docs/custom-domains#step-4:-configuring-the-domain)を独自ドメインの DNS レコード設定に追加するだけで、独自ドメイン経由で HTTPS アクセス可能になります。

![スクリーンショット 2021-01-27 6.20.50.png](https://i.gyazo.com/e91580925ee7550102f986e3947d7f16.png)
**Custom Domains を設定して独自ドメイン経由でアクセスできるようにする (サブドメイン設定時の場合の設定方法)**

Custom Domains の状態が `Valid Configuration` になり次第、設定した独自ドメインへアクセスしたときに Vercel デプロイしたページが表示されるか確認する。

# (おまけ) Team Blog Hub の実際の運用

社内で話し合った結果、**`master` ブランチはオリジナルリポジトリの最新内容を反映する場所として、`release` ブランチは実際のデプロイに利用するブランチとして運用することにしました。**

`master` ブランチと分離して、自分たちの会社の独自改修については `release` ブランチに取り込むことで、本家リポジトリに PR が出しやすいという利点があります。また、自分たちのペースで本家リポジトリの最新の `master` ブランチの取り込み作業ができるという利点もあります。

ブランチを分離しておくだけで、**最新の本家リポジトリの新機能の取り込みが楽になる恩恵が受けられるようになり、かつ自社の独自改修の内容についても安心して `release` ブランチへコミットできるようになるためオススメです。**

# おわりに

[Team Blog Hub](https://github.com/catnose99/team-blog-hub) を用いることで、簡単にカッコいいデザインで会社のテックブログを構築できました。OSS として公開してくださっている [catnose](https://zenn.dev/catnose99) さんには本当に感謝です 🙏🏻

無理なく継続更新され続ける自社テックブログとして、ありがたく Team Blog Hug を活用させていただきつつ、PR チャンスがあれば積極的に狙っていきたいと考えています 🔫

# 参考リンク

- [catnose99/team-blog-hub: RSS based blog starter kit for teams](https://github.com/catnose99/team-blog-hub)
- [Introduction to Vercel - Vercel Documentation](https://vercel.com/docs)
- [Vercel Action · Actions · GitHub Marketplace](https://github.com/marketplace/actions/vercel-action)

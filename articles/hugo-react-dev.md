---
title: "Hugo で React + TypeScript を利用してサクッとウェブサイトに RSS リーダーを追加する"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hugo", "react", "rss", "javascript", "typescript"]
published: true
---

# はじめに

Hugo のウェブサイトに組み込む RSS リーダーを TypeScript で開発してみたいと思い調査したところ、Hugo の最新版には [ESBuild](https://github.com/evanw/esbuild) が組み込まれていて、**非常に手厚く JavaScript の開発環境がサポートされていることが分かりました。** 本記事では紹介していませんが [Babel](https://gohugo.io/hugo-pipes/babel/) も利用できるようです。

また、NPM パッケージも利用できるため、普段のウェブ開発と同様の流れで開発ができ、各種ライブラリを用いた開発も非常に楽でした。
今回は Hugo で JavaScript 開発する方法を RSS リーダーの開発を例に上げ、そこで得た知見についても交える形で記事として残しておくことにしました。

**ちなみに本記事内容は Hugo で JavaScript 開発する方法に焦点を絞ったものなのですが、ウェブサイトに RSS リーダーを組み込むことに焦点を絞って見たい方は [`RSS リーダーを Hugo の Data Templates で実装する`](#(%E4%BD%99%E8%AB%87)-rss-%E3%83%AA%E3%83%BC%E3%83%80%E3%83%BC%E3%82%92-hugo-%E3%81%AE-data-templates-%E3%81%A7%E5%AE%9F%E8%A3%85%E3%81%99%E3%82%8B) から見ていただくことをオススメします。**

# Hugo で JavaScript (React + TypeScript) の開発環境を整える

まず、**TypeScript のビルドは ESBuild に任せることができるため何も行う必要はありません。** そのため React 開発用パッケージのインストールのみ行えば大丈夫です。

Hugo プロジェクトのルートディレクトリで下記コマンドを実行し、`package.json` を作成してから、React の開発に必要なパッケージをインストールします。

```bash
npm init -y
npm install --save react react-dom
```

無事パッケージのインストールが完了したら、早速 TSX ファイルを `assets/js/App.tsx` に作成してしまいます。

```typescript:assets/js/App.tsx
import * as React from "react";
import * as ReactDOM from "react-dom";

function App() {
    return (
        <>
        Hello React!
        </>
    );
}

ReactDOM.render(
    <App />,
    document.getElementById("react")
);
```

上記のコードを見てもらえば分かる通り、レンダリング先に `id` が `react` の DOM ノードを指定しています。そのため Hugo 側で該当する DOM ノードを用意する必要があります。その際の HTML テンプレートは下記になります。

```html
<!-- ... -->

<!-- 利用するリソースを指定する -->
{{ with resources.Get "js/App.tsx" }}

<!-- id が react の div 要素を用意する -->
<div id="react"></div>

<!-- TSX を ESBuild でビルドする際の Hugo のオプションを指定する -->
{{ $options := dict "targetPath" "js/app.js" "minify" true "defines" (dict "process.env.NODE_ENV" "\"development\"") }}

<!-- TSX のビルドを Hugo のオプションで指定した内容で実行する -->
{{ $js := resources.Get . | js.Build $options }}

<!-- 一応 SRI を有効化した状態でビルドした JS を読み込む -->
{{ $secureJS := $js | resources.Fingerprint "sha512" }}
<script src="{{ $secureJS.Permalink }}" integrity="{{ $secureJS.Data.Integrity }}"></script>

{{ end }}

<!-- ... -->
```

:::message info

ちなみに `$options` で指定している ESBuild でビルド時に指定可能なオプションは [Hugo の公式ページ](https://gohugo.io/hugo-pipes/js/) に記載されています。

:::

上記 HTML の記述を RSS リーダーを埋め込みたいページに追加します。
この状態で該当ページにアクセスすると下記のような表示が確認できるはずです。

![Hello React! と画面に表示される](https://i.gyazo.com/7e196a2a52f492771deb5dd6913bbe60.png)
**App.tsx で定義した内容が画面に表示される**

これで React + TypeScript の開発環境が整いました。

# RSS リーダーを実装する

あとは一般的な Web フロントエンド開発の流れで RSS リーダーの開発を進めていくだけです。

## ウェブサイトで読み込みたい RSS フィードを準備する

:::message alert

RSS フィードを利用する際は必ず提供しているサービスの利用規約をご確認ください。
[Qiita](https://qiita.com/terms) 及び [Zenn](https://zenn.dev/terms) については個人利用かつ自分の情報のみを扱う範囲内であれば利用が許可されているように見受けられました。[^1]

:::

[^1]: もし認識に誤りがあればコメント欄等でご教授いただけますと幸いです。

下準備としてウェブサイトで読み込みたい RSS フィードを事前にダウンロードするためのバッチを作成します。バッチは NPM を利用して作成していきます。**NPM を導入したので Hugo で利用する簡易なバッチは JavaScript でサクッと作成していきます。**

まずはスクリプト作成の際に必要となるパッケージを事前にいくつかインストールします。

```bash
# html をテキスト変換にするパッケージと RSS フィードのパーサーをインストールする
npm i -D --save html-to-text rss-parser
```

実際のコードは下記になります。ファイル名末尾が `.mjs` なのは [Top-Level Await](https://dev.to/mikeesto/top-level-await-in-node-2jad) を使用したいからです。

```typescript:scripts/update-rss.mjs
import { writeFileSync } from 'fs';

import pkg from 'html-to-text';
const { htmlToText } = pkg;

import Parser from 'rss-parser';
const parser = new Parser();

// 自ブログで読み込みたい RSS フィードの情報を設定する
const rssFeed = {
    Zenn: {
        rss_url: 'https://zenn.dev/nikaera/feed',
        profile_url: 'https://zenn.dev/nikaera',
    },
    Qiita: {
        rss_url: 'https://qiita.com/nikaera/feed.atom',
        profile_url: 'https://qiita.com/nikaera',
    }
}

try {
    const jsonFeed = {}

    // RSS フィード内の description を 73字で切り取り末尾に ... を付与する関数
    const spliceContent = (content) => `${htmlToText(content).slice(0, 73)}...`

    // rssFeed 変数で定義されてる情報を繰り返し処理する
    for (const [site, info] of Object.entries(rssFeed)) {

        // RSS フィードの URL から必要な情報を取得する
        const feed = await parser.parseURL(info.rss_url);

        // RSS フィードに登録されている項目で必要な情報のみを取得する
        const items = feed.items.map((i) => {
            return {
                title: i.title,
                content: spliceContent(i.content),
                url: i.link,
                date: i.pubDate
            }
        })

        // 取得内容は jsonFeed に格納する
        const { rss_url, profile_url } = info
        jsonFeed[site] = { rss_url, profile_url, items };
    }

    // 最後に jsonFeed に格納された内容を JSON 文字列として static/rss.json に出力する
    writeFileSync('./static/rss.json', JSON.stringify(jsonFeed));
} catch(err) {
    console.error(err);
}
```

次に `package.json` の `scripts` に登録してコマンドとして実行可能にします。

```json:package.json
{
    "scripts": {
        "update-rss": "node ./scripts/update-rss.mjs"
    }
}
```

これで `npm run update-rss` を実行すれば自ブログで表示する際に用いる JSON ファイルとして RSS フィードの内容を `static/rss.json` に出力できます。また、JSON ファイルは `static` フォルダに出力しているため `http://localhost:1313/rss.json` でアクセスできます。

![npm run update-rss を実行して出力した rss.json](https://i.gyazo.com/508ba87c41f1c1e410b89ff1bb56be4e.png)
**npm run update-rss を実行して出力した rss.json**

![npm run update-rss を実行して出力した rss.json にブラウザからアクセスする](https://i.gyazo.com/9b7ebeedce1cb69b6b3ab8acacb0b1d1.png)
**`http://localhost:1313/rss.json` にアクセスして出力した rss.json が参照可能なことを確認する**

## RSS リーダーを React + TypeScript で実装する

準備が整ったので、早速 RSS リーダーを作成していきます。

下記は Hugo のテーマの 1つである [hugo-PaperMod](https://themes.gohugo.io/hugo-papermod/) の `archives` テンプレートを利用してページに埋め込むことを想定した RSS リーダーのコードです。

```typescript:assets/js/Rss.tsx
import React, { useMemo, useState } from 'react'

import * as superagent from 'superagent';

const Rss = (props) => {
    const [feed, setFeed] = useState({});
    const { name } = props;

    useMemo(() => {
        (async () => {
            try {
                const res = await superagent.get('/rss.json');
                setFeed(res.body[name]);
            } catch (err) {
                console.error(err);
            }
        })()
    }, [name]);

    if (!("items" in feed)) return null

    return (
        <div className="archive-month">
            <h3 className="archive-month-header">
                <a href={feed.profile_url} target="_blank" rel="noopener noreferrer">{name}</a> - <a href={feed.rss_url} target="_blank" rel="noopener noreferrer">RSS</a>
            </h3>
            <div className="archive-posts">
                {feed.items.map((item) => {
                    return <div className="archive-entry" key={item.url}>
                        <h3 className="archive-entry-title">{item.title}</h3>
                        <div className="archive-meta">{item.date} - {item.content}</div>
                        <a className="entry-link" href={item.url} target="_blank" rel="noopener noreferrer">&nbsp;</a>
                    </div>
                })}
            </div>
        </div>
    )
}

export default Rss

```

次に `assets/js/App.tsx` で `assets/js/Rss.tsx` を読み込み画面に表示できるよう改修します。

```typescript:assets/js/App.tsx
import Rss from './Rss';

import * as React from "react";
import * as ReactDOM from "react-dom";

function App() {
    return (
        <>
            <div class="archive-year">
                <h2 class="archive-year-header">
                    Tech 🦾
                </h2>
                <Rss name="Zenn" />
                <Rss name="Qiita" />
            </div>
        </>
    );
}

ReactDOM.render(
    <App />,
    document.getElementById("react")
);

```

これで RSS リーダーを埋め込んだページを閲覧すると下記のような画面が表示されるはずです。

![hugo-PaperMod で archives テンプレートを用いて RSS リーダーを表示する](https://i.gyazo.com/0a6b8923d141ae70f5e298637f5acc69.png)
**hugo-PaperMod で `archives` テンプレートを用いて RSS リーダーを表示したときの画面**

もし他の RSS フィードを追加したい場合は `scripts/update-rss.mjs` の `rssFeed` 変数に情報を追加して、`App.tsx` に `<Rss name="<rssFeed 変数で定義した RSS Feed 名>" />` を定義することで対応できます。

# RSS フィードの内容を自動で更新する

`npm run update-rss` を手元で実行して `static/rss.json` を更新して公開すれば、最新の RSS フィードの内容をページに反映できる状態ですが、都度手動で更新するのは面倒な作業です。

そこで今回は GitHub Actions の `schedule` を用いて `static/rss.json` の更新を自動化します。

## GitHub Actions のワークフローファイルを作成する

実際のワークフローファイルは下記になります。`schedule` の項目で設定している内容がワークフローの実行スケジュールになります。今回は半日毎に更新が走るようにしました。

```yml:.github/workflows/update-rss.yml
name: update rss json file

on:
  push:
    branches:
      - main  # Set a branch name to trigger deployment
  schedule:
    - cron: '0 */12 * * *' # 今回は半日に 1回のタイミングで更新するようにした

jobs:
  build:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          ref: main
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Use Node.js 14.10.1
        uses: actions/setup-node@v1
        with:
          node-version: 14.10.1

      - name: Install dependencies
        run: npm install

      - name: Update RSS Feeds
        run: npm run update-rss

      - name: Commit files
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add static/rss.json
          STATUS=$(git status -s)
          if [ -n "$STATUS" ]; then
            git commit -m "Update rss.json `date +'%Y-%m-%d %H:%M:%S'`" -a
            git push origin main
          fi

```

上記ワークフローファイルをプロジェクトに追加して、リモートリポジトリにプッシュした後は、ワークフローが実行されるタイミングを待ちます。

無事にワークフローの実行が完了すると下記のようなコミットが追加されているはずです。

![GitHub Actions が JSON ファイルを更新してコミットしている](https://i.gyazo.com/ebb7cb2e64b13e4a1e1a592836f511f5.png)
**GitHub Actions が JSON ファイルを更新してコミットしている**

![コミットの詳細を見ると正常に JSON ファイルが更新されていることを確認できる](https://i.gyazo.com/1a74399a3cf1053e0480a01590086fbe.png)
**コミットの詳細を見ると正常に JSON ファイルが更新されていることが確認できる**

![コミット後 Hugo をビルド & デプロイするとページが更新されていることを確認できる](https://i.gyazo.com/bf86668d5fc32ca09b6d2cfcf71262ce.png)
**コミット後 Hugo をビルド & デプロイするとページが更新されていることを確認できる**

これで Zenn や Qiita 等に記事を書いた際に、都度手動で `static/rss.json` を更新してページに最新の内容を反映させる作業は必要なくなりました。

# (余談) RSS リーダーを Hugo の Data Templates で実装する

ちなみに Hugo には [Data Templates](https://gohugo.io/templates/data-templates/) という仕組みがあり、これを用いることで実は JavaScript を利用しなくても HTML テンプレートで RSS リーダーを実現できるということを後から知りました。

そこで最後に Data Template での RSS リーダーの実装方法について記載します。

まずは、`scripts/update-rss.mjs` の内容を書き換えます。

```typescript:scripts/update-rss.mjs
import { writeFileSync } from 'fs';

import pkg from 'html-to-text';
const { htmlToText } = pkg;

import Parser from 'rss-parser';
const parser = new Parser();

const rssFeed = {
    Zenn: {
        rss_url: 'https://zenn.dev/nikaera/feed',
        profile_url: 'https://zenn.dev/nikaera'
    },
    Qiita: {
        rss_url: 'https://qiita.com/nikaera/feed.atom',
        profile_url: 'https://qiita.com/nikaera'
    }
}

try {
    const jsonFeed = {}

    const spliceContent = (content) => `${htmlToText(content).slice(0, 73)}...`
    for (const [site, info] of Object.entries(rssFeed)) {
        const feed = await parser.parseURL(info.rss_url);
        const items = feed.items.map((i) => {
            console.log(i);
            return {
                title: i.title,
                content: spliceContent(i.content),
                url: i.link,
                date: i.pubDate
            }
        })
        const { rss_url, profile_url } = info
        jsonFeed[site] = { rss_url, profile_url, items };

        /*
        最終的な JSON ファイルの出力先は data フォルダとなり、RSS フィード毎に出力する
        例: ./data/Qiita.json, ./data/Zenn.json, etc.
        */
        writeFileSync(`./data/${site}.json`, JSON.stringify(jsonFeed[site]));
    }
} catch(err) {
    console.error(err);
}

```

上記を実行することで `data/Qiita.json` や `data/Zenn.json` にファイルが出力されます。

Hugo の Data Template を用いると `data` フォルダ内に配置した `json`, `yaml`, `toml` 形式のファイルは Go の HTML テンプレートで読み込めるようになります。

例えば、**`data/Qiita.json` に配置された JSON ファイルを読み込みたい場合は Go のテンプレートで `$Qiita := $.Site.Data.Qiita` のような記述でできます。**

次に RSS リーダーを埋め込んでいたページを下記のように書き換えます。

```html
<!-- ... -->

<!-- React 関連の記述を全て削除する -->
<!--
{{ with resources.Get "js/App.tsx" }}
<div id="react"></div>
{{ $options := dict "targetPath" "js/app.js" "minify" true "defines" (dict "process.env.NODE_ENV" "\"development\"") }}
{{ $js := resources.Get . | js.Build $options }}
{{ $secureJS := $js | resources.Fingerprint "sha512" }}
<script src="{{ $secureJS.Permalink }}" integrity="{{ $secureJS.Data.Integrity }}"></script>
{{ end }}
-->

<div class="archive-year">
    <h2 class="archive-year-header">
        Tech 🦾
    </h2>
    <div class="archive-month">
        <!-- data/Zenn.json の内容を読み込む -->
        {{ $Zenn := $.Site.Data.Zenn }}
        <h3 class="archive-month-header">
            <a href="{{ $Zenn.profile_url }}" target="_blank" rel="noopener noreferrer">Zenn</a> - <a
                href="{{ $Zenn.rss_url }}" target="_blank" rel="noopener noreferrer">RSS</a>
        </h3>
        <div class="archive-posts">
        <!-- 配列で格納されている記事情報を繰り返し処理で取得する -->
            {{- range $Zenn.items }}
            <div class="archive-entry" key="{{ .url }}">
                <h3 class="archive-entry-title">{{ .title }}</h3>
                <div class="archive-meta">{{ .date }} - {{ .content }}</div>
                <a class="entry-link" aria-label="{{ .content }}" href="{{ .url }}" target=" _blank"
                    rel="noopener noreferrer"></a>
            </div>
            {{- end }}
        </div>
    </div>
    <div class="archive-month">
        <!-- data/Qiita.json の内容を読み込む -->
        {{ $Qiita := $.Site.Data.Qiita }}
        <h3 class="archive-month-header">
            <a href="{{ $Qiita.profile_url }}" target="_blank" rel="noopener noreferrer">Qiita</a> - <a
                href="{{ $Qiita.rss_url }}" target="_blank" rel="noopener noreferrer">RSS</a>
        </h3>
        <div class="archive-posts">
        <!-- 配列で格納されている記事情報を繰り返し処理で取得する -->
            {{- range $Qiita.items }}
            <div class="archive-entry" key="{{ .url }}">
                <h3 class="archive-entry-title">{{ .title }}</h3>
                <div class="archive-meta">{{ .date }} - {{ .content }}</div>
                <a class="entry-link" aria-label="{{ .content }}" href="{{ .url }}" target=" _blank"
                    rel="noopener noreferrer"></a>
            </div>
            {{- end }}
        </div>
    </div>
</div>

<!-- ... -->
```

また GitHub Actions のワークフローを用いて RSS フィードの情報を更新していた場合は、`.github/workflows/update-rss.yml` ファイルの更新も必要になります。

```yml:.github/workflows/update-rss.yml
name: update rss json file

on:
  push:
    branches:
      - main  # Set a branch name to trigger deployment
  schedule:
    - cron: '0 */12 * * *'

jobs:
  build:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          ref: main
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Use Node.js 14.10.1
        uses: actions/setup-node@v1
        with:
          node-version: 14.10.1

      - name: Install dependencies
        run: npm install

      - name: Update RSS Feeds
        run: npm run update-rss

        # Git で追加する内容を data フォルダに変更する
        # git add static/rss.json -> git add data/
      - name: Commit files
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add data/
          STATUS=$(git status -s)
          if [ -n "$STATUS" ]; then
            git commit -m "Update data folder `date +'%Y-%m-%d %H:%M:%S'`" -a
            git push origin main
          fi

```

これで JavaScript で作成した RSS リーダーから、Hugo の Data Templates を用いて作成した RSS リーダーへ移行できました。

# おわりに

Hugo で React + TypeScript 開発を楽にできそうなことが分かり、テンションが上がってしまい、そのままのノリで実際に RSS リーダーを自ブログ向けに作成してみました。

しかし、本記事内容で RSS リーダーを実装するのであれば、Hugo の Data Templates を利用することがベストなことに後から気づきました。ただ Hugo での JavaScript を用いた開発手法が理解でき勉強になったので結果ヨシとしました。

Hugo での JavaScript 開発環境は相当充実していることが分かったので、また何かアイデアを思いついたら気軽に作って自ブログに取り込んでいきます。今はザックリ WebGL/WebVR とかで何か面白いもの作れそうだなと考えています。

# 参考リンク

- [esbuild \- An extremely fast JavaScript bundler](https://esbuild.github.io/)
- [Data Templates \| Hugo](https://gohugo.io/templates/data-templates/)
- [Functions Quick Reference \| Hugo](https://gohugo.io/functions/)
- [JavaScript Building \| Hugo](https://gohugo.io/hugo-pipes/js/)
- [Introducing Hooks – React](https://reactjs.org/docs/hooks-intro.html)
- [rbren/rss\-parser: A lightweight RSS parser, for Node and the browser](https://github.com/rbren/rss-parser)
- [html\-to\-text/node\-html\-to\-text: Advanced html to text converter](https://github.com/html-to-text/node-html-to-text)

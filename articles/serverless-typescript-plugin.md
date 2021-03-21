---
dev_article_id: 640771
title: "Serverless のプラグインを TypeScript で作成する方法"
emoji: "🔨"
type: "tech"
topics: ["serverless", "typescript", "nodejs"]
published: true
---

# はじめに

[Serverless Framework](https://www.serverless.com/) を使っていて、度々デプロイ時に手動で設定していた作業内容を自動化したいなと思い、プラグイン作成の知識習得も兼ねてライブラリを作成し [NPM](https://www.npmjs.com/) で公開してみました。

[serverless-amplify-auth 🔑](https://www.npmjs.com/package/serverless-amplify-auth)

今後も開発する可能性はありそうなので Serverless のプラグインを TypeScript で作成する際の手順をまとめておきました。各手順はザックリと紹介しつつ、**主にその過程でハマった点や工夫した点に重きをおいて記事を書いていきます。**

# 動作環境

- Node.js 12.19.0
- Serverless Framework
  - Framework Core: 2.10.0
  - Plugin: 4.1.1
  - SDK: 2.3.2
  - Components: 3.3.0

# 開発環境を整える

本記事の内容を最後まで実践した際の最終的なプロジェクトのディレクトリ構造は下記になります。

```bash
tree -I node_modules -L 2 ./
./
├── example # ライブラリの動作検証用のサンプルコードを配置するフォルダ
│   ├── handler.js
│   ├── package.json
│   └── serverless.yml
├── lib     # src フォルダ内のファイルをコンパイルした結果を配置するフォルダ (ライブラリとして利用する際に含まれるソースコード群)
│   ├── index.js
│   └── index.js.map
├── package-lock.json
├── package.json
├── src     # Serverless プラグインのソースコードを配置するフォルダ
│   └── index.ts
└── tsconfig.json
```

基本的には [TypeScript で Serverless Framework の Plugin を書いてみる | Developers.IO](https://dev.classmethod.jp/articles/create-serverless-framework-plugin-by-typescript/) の手順をなぞっていくだけで環境構築自体は可能です。そこで、ここでは自分なりに工夫した箇所について記載していきます。

まずは、開発に必要なパッケージを下記コマンドでまとめてインストールします。

```bash
# TypeScript の開発に必要なパッケージインストール
npm i -D typescript

# TypeScript の型定義ファイルのインストール
npm i -D @types/node @types/serverless

# 今回は AWS プロバイダー向けの開発を行うため SDK をインストールする
npm i --save aws-sdk
```

TypeScript のコンパイル時に必要となる `tsconfig.json` は下記のように設定しました。

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
    "moduleResolution": "node",

    "strict": true,

    "strictBindCallApply": false,
    "strictNullChecks": false,

    "outDir": "lib",

    "sourceMap": true
  },
  "include": [
    "src/**/*"
  ]
}
```

**`compilerOptions.strict` には `true` を設定しつつ、`compilerOptions.strictNullChecks` 等には `false` を設定することで、部分的に TypeScript のコンパイルチェックを外すようにしました。**

`outDir` には `lib` を指定することで、コンパイルされた TypeScript ファイルは `lib` フォルダに出力されるよう設定しました。

`include` には `src/**/*` を明示的に指定しており、src フォルダ内の全ファイルをコンパイル対象にしております。

---

`package.json` の内容は部分的に抜粋し、説明が必要そうな項目について説明いたします。
全容を把握したい方は [こちら](https://github.com/nikaera/serverless-amplify-auth/blob/main/package.json) からご確認いただけます。

```json:package.json (一部抜粋)
{
  "main": "lib/index.js",
  "files": [
    "lib"
  ],
  "scripts": {
    "build": "rm -rf lib && tsc",
    "test": "echo \"Error: no test specified\" && exit 1"
  }
}
```

`main` には `src/index.ts` をコンパイルすると生成される `lib/index.js` を指定しました。そのため、ライブラリのエントリーポイントは `lib/index.js` が設定されます。

`files` には `lib` フォルダを指定することで、TypeScript をコンパイルした結果のみがライブラリのソースコードとして取り込まれるようになります。

# Serverless プラグインの開発を進める

開発環境が整ったところで早速 Serverless Plugin のソースコードを書いていきます。TypeScript のソースコードは `src/index.ts` に配置します。

## Serverless プラグインのプログラムを書く

```typescript:src/index.ts
import * as Serverless from 'serverless'
import {
  SharedIniFileCredentials,
  config,
} from 'aws-sdk'

/**
 * serverless.yml の custom property の型定義
 */
interface Variables {
  value1: string
  value2: number
  value3: boolean
  profile?: string
}

export default class Plugin {
  serverless: Serverless
  options: Serverless.Options
  hooks: {
    [event: string]: () => Promise<void>
  }
  variables: Variables

  /**
   * プラグインの初期化関数。
   * 注意点として、初期化関数内では serverless.yml 内の変数展開が行われないので、
   * ${ssm:~} 等で設定した値を呼び出しても、適切に値が設定されない状態で呼び出すことになる。
   */
  constructor(serverless: Serverless, options: Serverless.Options) {
    this.serverless = serverless
    this.options = options

    /**
     * serverless.service.custom 内の特定プロパティを取得するための記述
     * 今回は Serverless のプラグイン名に serverless-typescript を設定したため、
     * serverless-typescript 文字列をキーとして指定する。
     */
    this.variables = serverless.service.custom['serverless-typescript']

    /**
     * プラグインがフックする関数を指定する。複数指定することも可能だが、
     * 今回は before:package:createDeploymentArtifacts を指定して、
     * パッケージングの手前の処理を定義した run 関数でフックする。
     */
    this.hooks = {
      'before:package:createDeploymentArtifacts': this.run.bind(this),
    }
  }

  /**
  * before:package:createDeploymentArtifacts 時に実行される関数
  */
  async run() {
    /**
    * プラグイン実行時に必要となるフィールドがセットされていなければ処理をスキップする
    */
    if (!this.variables) {
      this.serverless.cli.log(
        `serverless-typescript: Set the custom.serverless-typescript field to an appropriate value.`,
      )
      return
    }

    /**
     * this.serverless.getProvider 関数を用いることで、
     * デプロイ時のアカウントの各種情報について取得することが出来る
     */
    const awsProvider = this.serverless.getProvider('aws')
    const region = await awsProvider.getRegion()
    const accountId = await awsProvider.getAccountId()
    const stage = await awsProvider.getStage()

    /**
     * serverless.yml で指定した値や AWS 情報が取得できているか、
     * 確認するために標準出力する
     */
    this.serverless.cli.log(
      `serverless-typescript values: ${JSON.stringify({
        stage: stage,
        region: region,
        accountId: accountId,
        variables: this.variables,
      })}`,
    )

    /**
     * プラグイン内で処理を実行する際、別の特定 Profile を用いたい際は、
     * AWS SDK の SharedIniFileCredentials を用いて切り替えると楽に切替可能。
     * その際は process.env.AWS_SDK_LOAD_CONFIG に値を設定しておくこと
     */
    if (this.variables.profile) {
      process.env.AWS_SDK_LOAD_CONFIG = 'true'
      const credentials = new SharedIniFileCredentials({
        profile: this.variables.profile,
      })
      config.credentials = credentials
    }
  }
}

module.exports = Plugin
```

ソースコード内にいくつかコメントを残しましたが、何点か補足の説明をしていきます。

`serverless.service.custom['serverless-typescript']` を呼び出すことで、`serverless.yml` 内の下記の記述内容を `Object` として取得できます。

```yml:serverless.yml
custom:
    # custom.serverless-typescript 内の定義を Object として取得可能
    serverless-typescript:
        value1: "value1"
        value2: 0
        value3: true
        # profile: default (optional)
```

`this.hooks` には必要に応じてフックを指定します。フックの書き方については [公式ドキュメント](https://www.serverless.com/framework/docs/providers/aws/guide/plugins#lifecycle-events) に詳細が記載されています。フックの種類については [Gist](https://gist.github.com/HyperBrain/50d38027a8f57778d5b0f135d80ea406) でまとめてくださっている方がいました。

**`this.serverless.getProvider('aws')` を用いることで、デプロイ時にアカウントの各種情報について取得することが出来ます。この記述を利用することで [Serverless Pseudo Parameters](https://www.serverless.com/plugins/serverless-pseudo-parameters) のようなシンタックスを自身のプラグインに取り込むことが可能になります。**
私が作成したプラグインでも [serverless.yml](https://github.com/nikaera/serverless-amplify-auth/blob/main/example/serverless.yml#L21-L23) で [ARN](https://docs.aws.amazon.com/ja_jp/general/latest/gr/aws-arns-and-namespaces.html) を構築する際に利用していて、[index.ts](https://github.com/nikaera/serverless-amplify-auth/blob/main/src/index.ts#L126-L128) 内で利用しました。

また、**プラグイン内でデプロイ時とは異なる Profile を使用したいケースもあるかと存じます。それは AWS SDK の [SharedIniFileCredentials](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/SharedIniFileCredentials.html) を用いることで簡易に実装できました。**

:::message alert
注意点として、`SharedIniFileCredentials` を用いてプロファイルを切り替える時は、環境変数に [AWS_SDK_LOAD_CONFIG="true"](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/loading-node-credentials-configured-credential-process.html) を設定する必要がありました。
設定しないと `ConfigError: Missing region in config` というエラーが発生してしまい、プロファイルを切り替えることが出来ませんでした。
:::

それでは、次にプラグインの動作検証用コードを `example` フォルダに配置していきます。

## Serverless プラグインの動作検証用プログラムを書く

`example` フォルダ内には検証用プロジェクトを作成するので、その前準備として `example/package.json` を作成します。

```bash
# package.json ファイルを作成する
cd example && npm init -y
```

`example/package.json` ファイルを作成したら開発用のスクリプトを `example/package.json` に追記します。

```json:example/package.json
{
  "scripts": {
    "prestart": "cd ../ && npm run build",
    "start": "sls package",
    "test": "echo \"Error: no test specified\" && exit 1"
  }
}

```

`scripts` 内の `prestart` は `start` スクリプト実行前に実行されるスクリプトです。`npm start` を実行すると `prestart` でプラグインの `build` タスクを実行した後、 Serverless Framework のパッケージングを行うことでプラグインの動作確認が行えます。

:::message
今回は Serverless の `before:package:createDeploymentArtifacts` フックを利用しているので、`sls package` コマンドで動作検証が可能となっている。`before:deploy:deploy` 等のデプロイ中に実行されるフックを利用する際は `sls deploy --noDeploy` コマンド等で動作検証を行う必要がある。
:::

次に動作検証用の `serverless.yml` を `example` フォルダに配置します。

```yml:serverless.yml
service:
    name: serverless-typescript
    publish: false

# プラグイン内で利用する設定値を定義する
custom:
    serverless-typescript:
        value1: "value1"
        value2: 0
        value3: true
        profile: custom_profile

provider:
    name: aws
    runtime: nodejs12.x
    region: ap-northeast-1

# プラグインのパスを指定して読み込む
plugins:
    localPath: '../../'
    modules:
        - serverless-typescript

# 何でも良いので動作検証用の関数を定義する (関数の定義は後述)
functions:
    hello:
        handler: handler.hello

```

`example` フォルダ内に `handler.js` を配置して `functions.hello.handler` で用いる検証用の関数を定義します。

```javascript:example/handler.js
'use strict';

// 検証用の関数。serverless.yml 内では handler.hello で参照可能
module.exports.hello = (event, context, callback) => {
  callback(null, {
    statusCode: 200,
    body: "Hello World!"
  });
};

```

上記作業が完了次第、`cd example && npm start` を実行して動作検証してみます。

```bash
cd example && npm start

> example@1.0.0 prestart /Users/nika/Desktop/serverless-typescript/example
> cd ../ && npm run build


> serverless-typescript@1.0.0 build /Users/nika/Desktop/serverless-typescript
> rm -rf lib && tsc


> example@1.0.0 start /Users/nika/Desktop/serverless-typescript/example
> sls package

Serverless: Configuration warning at 'service': unrecognized property 'publish'
Serverless:
Serverless: Learn more about configuration validation here: http://slss.io/configuration-validation
Serverless:
# src/index.ts 内の this.serverless.cli.log の出力内容
# 各種値が正常にセットされていることが確認出来る
Serverless: serverless-typescript values: {"stage":"dev","region":"ap-northeast-1","accountId":"XXXXXXXXXX","variables":{"value1":"value1","value2":0,"value3":true,"profile":"custom_profile"}}
Serverless: Packaging service...
Serverless: Excluding development dependencies...
```

標準出力にあるプラグイン内で出力したログから、適切に値が取得出来ていることが確認出来れば OK です。

## AWS Profile の切り替えができるか確認してみる

Serverless プラグインでの Profile の切り替えについて、動作検証がまだ出来ていないので確認していきます。

`serverless.yml` 内の `custom.serverless-typescript.profile` に設定箇所は既に用意してあるので、`~/.aws/credentials` に実在する Profile 名を指定します。

```yml:serverless.yml (一部抜粋)
custom:
    serverless-typescript:
        profile: <プラグイン実行時に使用したい Profile 名>
```

動作検証のため、`src/index.ts` 内にログ出力の記述を加えます。

```typescript:src/index.ts
import * as Serverless from 'serverless'
import {
  SharedIniFileCredentials,
  config,
} from 'aws-sdk'

interface Variables {
  value1: string
  value2: number
  value3: boolean
  profile?: string
}

export default class Plugin {
  serverless: Serverless
  options: Serverless.Options
  hooks: {
    [event: string]: () => Promise<void>
  }
  variables: Variables

  constructor(serverless: Serverless, options: Serverless.Options) {
    this.serverless = serverless
    this.options = options

    this.variables = serverless.service.custom['serverless-typescript']
    this.hooks = {
      'before:package:createDeploymentArtifacts': this.run.bind(this),
    }
  }

  async run() {
    if (!this.variables) {
      this.serverless.cli.log(
        `serverless-typescript: Set the custom.serverless-typescript field to an appropriate value.`,
      )
      return
    }

    const awsProvider = this.serverless.getProvider('aws')
    const region = await awsProvider.getRegion()
    const accountId = await awsProvider.getAccountId()
    const stage = await awsProvider.getStage()

    this.serverless.cli.log(
      `serverless-typescript values: ${JSON.stringify({
        stage: stage,
        region: region,
        accountId: accountId,
        variables: this.variables,
      })}`,
    )

    if (this.variables.profile) {
      process.env.AWS_SDK_LOAD_CONFIG = 'true'
      const credentials = new SharedIniFileCredentials({
        profile: this.variables.profile,
      })
      config.credentials = credentials

      // Profile が切り替えられたか確認するためにログを出力する
      this.serverless.cli.log(`serverless-typescript profile: ${JSON.stringify(config.credentials)}`);
    }
  }
}

module.exports = Plugin
```

早速 `cd example && npm start` を実行して正常に profile が切り替えられていそうか確認してみます。

```bash
# 成功時の実行結果
cd example && npm start

# ...
# accessKeyId のフィールドに ~/.aws/credentials 内に存在する値が出力されている
Serverless: serverless-typescript profile: {"expired":false,"expireTime":null,"refreshCallbacks":[],"accessKeyId":"XXXXXXXXXXXXXX","profile":"XXXXXXXXXXXXXX","disableAssumeRole":false,"preferStaticCredentials":false,"tokenCodeFn":null,"httpOptions":null}
# ...
```

ちなみに存在しない Profile を指定した場合の出力は下記のようになります。

```bash
# 失敗時の実行結果
cd example && npm start

# ...
# accessKeyId のフィールドが存在しない時は Profile が正しく設定出来ていない
Serverless: serverless-typescript profile: {"expired":false,"expireTime":null,"refreshCallbacks":[],"profile":"custom_profile","disableAssumeRole":false,"preferStaticCredentials":false,"tokenCodeFn":null,"httpOptions":null}
# ...
```

# おわりに

今回初めて Serverless プラグインの開発をしてみて、手軽に出来ることが分かったので自動化出来そうな作業は積極的にプラグイン化していきたいなと感じました。

プラグイン化した後は Git リポジトリにアップするだけでなく、[NPM のパッケージ](https://docs.npmjs.com/packages-and-modules/contributing-packages-to-the-registry) や [GitHub Packages](https://docs.github.com/ja/free-pro-team@latest/packages/using-github-packages-with-your-projects-ecosystem/configuring-npm-for-use-with-github-packages) として公開しておくと、後々プラグインを利用する際に便利です。また、公開してライブラリのスタッツを見るのは案外楽しく開発のモチベーションにも繋がるのでオススメです。

# 参考リンク

- [TypeScript で Serverless Framework の Plugin を書いてみる | Developers.IO](https://dev.classmethod.jp/articles/create-serverless-framework-plugin-by-typescript/)
- [typescript 導入した private な npm パッケージの作り方 - 30 歳 SIer から WEB エンジニアで奮闘](https://karuta-kayituka.hatenablog.com/entry/2020/04/05/124531)
- [How To Write Your First Plugin For The Serverless Framework - Part 1](https://www.serverless.com/blog/writing-serverless-plugins)

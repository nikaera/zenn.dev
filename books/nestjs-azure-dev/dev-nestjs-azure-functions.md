---
title: "NestJS + Azure Functions の開発環境を構築する"
---

## Azure Functions Core Tools の導入

Azure Functions の開発をする上で、ローカルで動作確認を行ったり、CI 環境構築をするためにターミナルからデプロイ作業を済ませたくなる状況が発生します。

そのため、まずは上記のためのツールである [Azure Functions Core Tools](https://github.com/Azure/azure-functions-core-tools) を[公式サイトの手順](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-run-local?tabs=macos%2Ccsharp%2Cbash#v2)に従ってインストールします。__本記事では v3.x 系を利用しています。__

![公式サイトの手順](https://i.gyazo.com/96063b5c81bdc323174440c7b2709eb1.png)
*公式サイトの Azure Functions Core Tools のインストール手順*
## NestJS のインストール及び Azure Functions HTTP module の導入

[公式サイト](https://docs.microsoft.com/ja-jp/azure/azure-functions/durable/quickstart-js-vscode)を見る限り、現時点 (2020/10/23) では __Node.js のバージョンとして 10.x もしくは 12.x を利用する必要があります。本記事では v10.22.1 を利用しました。__

> Node.js のバージョン 10.x または 12.x がインストールされていることを確認します。

何はともあれ NestJS では CLI を用いて開発していくので、まずは CLI ツールをグローバルインストールします。

```bash
npm install @nestjs/cli -g
```

インストールが完了したら、CLI 経由で NestJS プロジェクトを作成し、Azure Functions HTTP module のインポートまで行います。

```bash
# NestJS アプリケーションを作成する
nest new azure-sample

# NestJS アプリケーションに Azure Functions HTTP module をインポートする
cd azure-sample
nest add @nestjs/azure-func-http
```

無事にコマンドの実行に成功していれば、プロジェクトフォルダ内は下記の構造になっているはずです。

```bash
# tree コマンドでプロジェクトフォルダ構成を確認する
tree -I node_modules -L 2 ./
./
├── README.md
├── host.json
├── local.settings.json
├── main
│   ├── function.json
│   ├── index.ts
│   └── sample.dat
├── node_modules
├── nest-cli.json
├── package-lock.json
├── package.json
├── proxies.json
├── src
│   ├── app.controller.spec.ts
│   ├── app.controller.ts
│   ├── app.module.ts
│   ├── app.service.ts
│   ├── main.azure.ts
│   └── main.ts
├── test
│   ├── app.e2e-spec.ts
│   └── jest-e2e.json
├── tsconfig.build.json
└── tsconfig.json
```

これで Azure Functions の開発環境は整いました。
早速動作検証が可能な状態となっているか確かめてみましょう。

### ローカル環境で Azure Functions の環境を起動する

`src/app.controller.ts` の中に既に `getHello` 関数が存在しているので、`helloworld` というパスでアクセス可能な状態にしてみます。

```typescript:src/app.controller.ts
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  // @Get 内に helloworld と記載することで、
  // http://localhost:7071/api/helloworld のようなエンドポイントでアクセス可能になる
  @Get('helloworld')
  getHello(): string {
    return this.appService.getHello();
  }
}
```

上記改修が完了したら `npm run start:azure` コマンドを実行してみます。

コマンドの実行が成功すると `http://localhost:7071/api/{*segments}` で各種 API へアクセス可能になります。

早速先程アクセス可能にした API へ `http://localhost:7071/api/helloworld` でアクセスしてみましょう。

![http://localhost:7071/api/helloworld のアクセス結果](https://i.gyazo.com/e1146e67fcc4c0d0fec133660f8d4702.png)
*http://localhost:7071/api/helloworld のアクセス結果*

画面上に `Hello World!` の文字列が表示されていれば成功です。

## NestJS で機能開発を行うための準備をする

本記事ではイベントを CRUD する機能を開発します。

NestJS では [モジュール](https://docs.nestjs.com/modules) 単位で開発します。**プログラムをモジュール単位で区切り、組み合わせることで、機能実装を進めていきます。** モジュールを区切る基準は開発者に委任されているため、採用するアーキテクチャによって異なります。

それでは早速 `nest g module events` コマンドでモジュールを新規作成します。

```bash
# モジュールを作成する
nest g module events
CREATE src/events/events.module.ts (82 bytes)
UPDATE src/app.module.ts (370 bytes)
```

すると、モジュールが新規作成されるのと同時に、そのモジュールを読み込むための記述が自動で `src/app.module.ts` に書き込まれます。

:::message
`AppModule` は NestJS でルートモジュールと呼ばれるもので、
アプリ起動時、最初に読み込まれるモジュールとなっております。
:::

```typescript:src/app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
// 新規追加された EventsModule を import で読み込む
import { EventsModule } from './events/events.module';

@Module({
  // ルートモジュールである AppModule から EventsModule を読み込む
  imports: [EventsModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

次に `EventsModule` の実体を作成するために [コントローラー](https://docs.nestjs.com/controllers) を作成します。**NestJS のコントローラーは入出力の制御する役割を担います。** 具体的にはルーティングやリクエストに応じたハンドリングを行います。

コントローラーもモジュールの時と同様 NestJS の `nest g controller events` コマンドで新規作成します。

```bash
# コントローラーを作成する
nest g controller events
CREATE src/events/events.controller.spec.ts (485 bytes)
CREATE src/events/events.controller.ts (99 bytes)
UPDATE src/events/events.module.ts (170 bytes)
```

:::message
`*.spec.ts` ファイルは生成したクラスのテストファイルになります。
NestJS はテストフレームワークに [Jest](https://github.com/facebook/jest) を採用しています。
:::

コントローラーが作成され、コントローラーを読み込むための記述が、自動で先程作成したモジュールに書き込まれました。**このように NestJS では CLI でスキャフォールド (開発に必要なファイル群の生成) する際に同名パスを指定すると自動的にインポートの記述が追加されていきます。**

```typescript:src/events/events.module.ts
import { Module } from '@nestjs/common';
// 新規追加された EventsController を import で読み込む
import { EventsController } from './events.controller';

@Module({
  // EventsController を読み込むための記述が追加されている
  controllers: [EventsController]
})
export class EventsModule {}
```

次に、コントローラーに機能を実装していく前に、新たに [プロバイダ](https://docs.nestjs.com/providers) を作成します。**NestJS のプロバイダはいわゆるサービス層の役割を担うクラス全般を指します。** リポジトリやファクトリー、ヘルパー等を指します。

今回は `EventsService` というプロバイダを新規作成して、イベントの CRUD を行うための実装を書いていきます。コントローラーで `EventsService` を用いることで、CRUD の処理をコントローラー経由で実行可能にします。`nest g service events` コマンドでプロバイダを新規作成します。

```bash
# プロバイダ (サービス) を作成する
nest g service events
CREATE src/events/events.service.spec.ts (453 bytes)
CREATE src/events/events.service.ts (89 bytes)
UPDATE src/events/events.module.ts (247 bytes)
```

`EventsService` が作成され、`EventsService` を読み込むための記述が自動で `EventsModule` に書き込まれます。

```typescript:src/events/events.module.ts
import { Module } from '@nestjs/common';
import { EventsController } from './events.controller';
// 新規追加された EventsService を import で読み込む
import { EventsService } from './events.service';

@Module({
  controllers: [EventsController],
  // EventsService を読み込むための記述が追加されている
  providers: [EventsService]
})
export class EventsModule {}
```

これでイベントの CRUD 機能を Azure Functions の関数として実装していくための準備が整いました。

上述の通り、**NestJS では `nest` コマンドを用いてスキャフォールド (開発に必要なファイル群の生成) した後に実装を進めるというフローを繰り返すのが一般的なフローとなっています。**

次はローカルに Azure Cosmos DB の開発環境を構築していきます。ローカルでの開発時は MongoDB を利用します。

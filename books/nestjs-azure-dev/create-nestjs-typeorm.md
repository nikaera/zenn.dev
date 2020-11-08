---
title: "TypeORM を用いて CRUD 機能を実装する"
---

# NestJS の Configuration module でデータベースの接続情報を管理する

まずは TypeORM で MongoDB に接続するための情報を NestJS の [Configuration module](https://docs.nestjs.com/techniques/configuration) を使って管理できるようにします。

```bash
# Configuration module をインストールする
npm i --save @nestjs/config
```

Configuration module は内部的に [dotenv](https://github.com/motdotla/dotenv) を利用しています。そのため、記法は dotenv と同じになります。

```dotenv:.env
MONGODB_URI=mongodb://azure-sample:azure-sample@localhost:27017/azure-sample?authSource=admin
```

MongoDB への接続 URI を `.env` ファイルで `MONGODB_URI` として記載しました。NestJS で上記を読み込むための記述を `src/app.module.ts` に追記します。

```typescript:src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { EventsModule } from './events/events.module';

@Module({
  imports: [
    // ConfigModule の記述を追加することで .env の変数を自動で読み込むようになる
    ConfigModule.forRoot(),
    EventsModule
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

**ルートモジュールで読み込んでいるため、`ConfigModule` で読み込んだ `.env` ファイルに記載した変数は全てのモジュールで利用可能です。** 試しに正常に読み込めていそうか `EventsModule` に `console.log` を追記して確かめてみます。

```typescript:src/app.controller.ts
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get('helloworld')
  getHello(): string {
    return this.appService.getHello();
  }

  // http://localhost:7071/api/database-uri にアクセスした時に変数の値を確認出来る API を追加
  @Get('database-uri')
  getDatabaseURI(): string {
    return process.env.MONGODB_URI;
  }
}
```

```bash
# Azure Functions のローカル環境を起動する
npm run start:azure
```

`src/app.controller.ts` を書き換えて、`npm run start:azure` した後に `http://localhost:7071/api/database-uri` へアクセスした結果は下記になるはずです。

![http://localhost:7071/api/database-uri のアクセス結果](https://i.gyazo.com/5023493e3c711e1fb5d9b089d9d9a658.png)
*http://localhost:7071/api/database-uri のアクセス結果*

:::message alert
無事に変数が読み込めていそうなことが確認できたら `getDatabaseURI` 関数は削除しておきましょう。本番環境でこの API が残っているとセキュリティ上問題があります。
:::

これで TypeORM で MongoDB に接続するための情報を `ConfigModule` で管理できることが確認できました。

# NestJS の TypeORM Module を導入してテーブルの定義を行う

次に NestJS の [TypeORM Module](https://github.com/nestjs/typeorm) をインストールします。また今回は MongoDB に接続するため [Mongoose](https://github.com/Automattic/mongoose) も同時にインストールしておきます。

:::message
TypeORM でサポートしているデータベースであれば、基本的に NestJS の TypeORM Module で利用可能です。たとえば MongoDB 以外にも PostgreSQL, MySQL, SQLite 等々がサポートされているようです。
:::

```bash
# NestJS で TypeORM Module を Mongoose で利用するのに必要なライブラリをインストールする
npm i --save @nestjs/typeorm typeorm @nestjs/mongoose mongoose

# TypeScript で利用するため Mongoose の型情報を追加インストールする
npm i --save-dev @types/mongoose
```

TypeORM で MongoDB に接続するための記述からイベントテーブルの定義までを[公式ページの手順](https://docs.nestjs.com/techniques/mongodb)に沿って進めます。まずは MongoDB への接続処理を `src/app.module.ts` に記載します。

```typescript:src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { EventsModule } from './events/events.module';

@Module({
  imports: [
    ConfigModule.forRoot(),
    // MongooseModule を用いて MongoDB との接続を行う
    MongooseModule.forRoot(process.env.MONGODB_URI),
    EventsModule
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

イベントテーブルの定義を `src/events/events.schema.ts` に書いていきます。イベントテーブルには　`name` 属性のみが存在します。

```typescript:src/events/events.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

// イベントテーブルの定義
@Schema()
export class Event extends Document {
  // イベント名を保持する必須フィールドのみを持つ
  @Prop({ required: true })
  name: string;
}

export const EventSchema = SchemaFactory.createForClass(Event);
```

次にイベントテーブルの操作を扱うモジュール `EventsModule` に `EventSchema` を利用するための記述を追記します。

```typescript:src/events/events.module.ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { EventsController } from './events.controller';
import { EventsService } from './events.service';
import { Event, EventSchema } from './events.schema'

@Module({
  // 先程定義した EventSchema を MongooseModule でインポートする
  // これで Controller や Provider で Event が利用可能になる
  // 今回は Provider である EventsService で利用する想定
  imports: [
    MongooseModule.forFeature([
      { name: Event.name, schema: EventSchema },
    ]),
  ],
  controllers: [EventsController],
  providers: [EventsService]
})
export class EventsModule {}
```

## TypeORM でイベントの CRUD 機能を実装する

イベントテーブルを定義したクラス `Event` を用いて、まずは `EventsService` に CRUD を実装していきます。

```typescript:src/events/events.service.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { Event } from './events.schema'

@Injectable()
export class EventsService {
    // コンストラクターインジェクションで EventModule で import した Model<Event> を生成する
    constructor(
        // MongooseModule でインポートした場合は @nestjs/mongoose に用意されている
        // @InjectModel デコレータでインジェクション時に名前を定義する必要がある
        @InjectModel(Event.name)
        private eventModel: Model<Event>,
    ) {}

    // Event を作成する
    async create(name: string): Promise<Event> {
        const createdEvent = new this.eventModel({ name: name });
        return createdEvent.save();
    }

    // Id 指定した Event を読み込む
    async readOne(id: string): Promise<Event> {
        return this.eventModel.findById(id).exec();
    }

    // Event を全件読み込む
    async readAll(): Promise<Event[]> {
        return this.eventModel.find().exec();
    }

    // Id 指定した Event を更新する
    async update(id: string, name: string): Promise<Event> {
        // データ更新時に、更新後のデータを返却するためのオプションとして { new: true } を指定する
        // { new: true } を指定しないと更新前のデータが返却されるようになる
        return this.eventModel.findByIdAndUpdate(
            id, { name: name }, { new: true }
        ).exec();
    }

    // Id 指定した Event を削除する
    async delete(id: string): Promise<Event> {
        return this.eventModel.findByIdAndDelete(id).exec();
    }
}
```

イベントの CRUD 機能を実装したので、次はコントローラーの実装を進めていきます。

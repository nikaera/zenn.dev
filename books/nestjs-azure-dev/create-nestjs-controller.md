---
title: "CRUD 機能のエンドポイントを追加する"
---

## コントローラー経由で CRUD を実行可能に改修する

次に `EventsController` を実装していきたいのですが、その前にリクエストやレスポンスのデータバリデーションも兼ねて、各種 DTO を作成します。

```typescript:src/events/events.dto.ts
// POST events の際の Body Request 定義
export interface CreateRequest {
    name: string
}

// PATCH events/:id の際の Body Request 定義
export interface UpdateEventDto {
    name: string
}
```

リクエストパラメーターの DTO を作成したら `EventsController` を実装します。

```typescript:src/events/events.controller.ts
import { Param, Body, Controller, Get, Post, Patch, Delete } from '@nestjs/common';
import { EventsService } from './events.service'
import { Event } from './events.schema'
import {
    CreateRequest,
    IdParams,
    UpdateRequest
} from './events.request'

@Controller('events')
export class EventsController {
    // コンストラクターインジェクションで EventsService を生成して EventsController で利用する
    constructor(private readonly eventsService: EventsService) {}

    // POST events へアクセス時に呼び出される関数
    @Post()
    async create(@Body() request: CreateEventDto): Promise<Event> {
        return this.eventsService.create(request.name)
    }

    // GET events/:id へアクセス時に呼び出される関数
    @Get(':id')
    async readOne(@Param('id') id: string): Promise<Event> {
        return this.eventsService.readOne(id)
    }

    // GET events へアクセス時に呼び出される関数
    @Get()
    async readAll(): Promise<Event[]> {
        return this.eventsService.readAll()
    }

    // PATCH events/:id へアクセス時に呼び出される関数
    @Patch(':id')
    async update(@Param('id') id: string, @Body() request: UpdateEventDto): Promise<Event> {
        return this.eventsService.update(id, request.name)
    }

    // DELETE events/:id へアクセス時に呼び出される関数
    @Delete(':id')
    async delete(@Param('id') id: string): Promise<Event> {
        return this.eventsService.delete(id)
    }
}
```

:::message
NestJS では `events/:id` のように定義した際、`:id` の内容は `@Param` デコレータを利用することで、引数から値を取得できます。
:::

## 各種 API の正常系の動作検証を `curl` で行う

各種 API の正常系の動作確認を `curl` で行ってみます。

```bash
# 1. Azure Functions のローカル環境を起動する
npm run start:azure

# 2. Docker Compose で MongoDB インスタンスを起動する
cd devenv
docker-compose up -d

# 3. Event を 2つ作成する
curl -X POST -H "Content-Type: application/json" -d '{"name": "test-event-1"}' http://localhost:7071/api/events
{"_id":"5f945260a503fa9ceae111ae","name":"test-event-1","__v":0}

curl -X POST -H "Content-Type: application/json" -d '{"name": "test-event-2"}' http://localhost:7071/api/events
{"_id":"5f945278a503fa9ceae111af","name":"test-event-2","__v":0}

# 4. 特定の Event を読み込む
curl -X GET http://localhost:7071/api/events/5f945260a503fa9ceae111ae
{"_id":"5f945260a503fa9ceae111ae","name":"test-event-1","__v":0}

# 5. 登録されてる Event を全件読み込む
curl -X GET http://localhost:7071/api/events
[{"_id":"5f945260a503fa9ceae111ae","name":"test-event-1","__v":0},{"_id":"5f945278a503fa9ceae111af","name":"test-event-2","__v":0}]

# 6. 特定の Event を更新する
curl -X PATCH -H "Content-Type: application/json" -d '{"name": "test-event-3"}' http://localhost:7071/api/events/5f945260a503fa9ceae111ae
{"_id":"5f945260a503fa9ceae111ae","name":"test-event-3","__v":0}

# 6-1 特定の Event が更新されたかどうかを確認する
curl -X GET http://localhost:7071/api/events/5f945260a503fa9ceae111ae

# 先程は test-event-1 だった値が test-event-3 に更新されていることが確認できた
{"_id":"5f945260a503fa9ceae111ae","name":"test-event-3","__v":0}

# 7. 特定の Event を削除する
curl -X DELETE http://localhost:7071/api/events/5f945260a503fa9ceae111ae
{"_id":"5f945260a503fa9ceae111ae","name":"test-event-3","__v":0}

# 7-1 特定の Event が削除されたかどうかを確認する
curl -X GET http://localhost:7071/api/events

# イベント全件取得した際に 2件登録したうちの 1件しか返却されず、7. で削除した id を含むデータは返却されないことが確認できた
[{"_id":"5f945278a503fa9ceae111af","name":"test-event-2","__v":0}]

# 8. Docker Compose で MongoDB インスタンスを破棄する
cd devenv
docker-compose down
```

イベントの CRUD 機能の実装及び動作検証は完了しましたが、機能改修のたびにこれらの動作検証を手動で行うのは面倒なので、E2E テストを実装していきます。

また、E2E のテスト環境は Docker Compose で構築していきます。

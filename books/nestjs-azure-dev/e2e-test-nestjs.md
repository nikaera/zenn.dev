---
title: "NestJS で E2E テストを実装する"
---

# CRUD 機能の E2E テストを実装する

それではイベントの CRUD API の正常系及び異常系の E2E テストを書いていきます。

NestJS では E2E テストを書く場所は `test` フォルダの中となっています。**また、テスト実行時は環境変数を `.env` ではなく `.env.test` で管理するようにします。** ローカルでの検証環境とテスト環境でのコンフィグは分けておきたいからです。

```dotenv:.env.test
# 現状は .env と同じ内容を .env.test にも記載する
MONGODB_URI=mongodb://azure-sample:azure-sample@localhost:27017/azure-sample?authSource=admin
```

```typescript:app.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { ConfigModule } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';
import { EventsModule } from '../src/events/events.module';
import { AppController } from '../src/app.controller';
import { AppService } from '../src/app.service';
import { CreateRequest, UpdateRequest } from '../src/events/events.request';
import { Event } from '../src/events/events.schema'

describe('AppController (e2e)', () => {
  let app: INestApplication;

  beforeEach(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [
          // テスト用の dotenv ファイルである .env.test をコンフィグとして読み込む
        ConfigModule.forRoot({
          envFilePath: '.env.test',
        }),
        MongooseModule.forRoot(process.env.MONGODB_URI),
        EventsModule,
      ],
      controllers: [AppController],
      providers: [AppService],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  // イベント全件取得のための関数
  const getEventAll = async (): Promise<Array<Event>> => {
    const res = await request(app.getHttpServer()).get('/events');
    expect(res.status).toEqual(200);

    return res.body as Array<Event>;
  }

  describe('EventsController (e2e)', () => {
    // イベントの Create API に関するテスト
    describe('Create API of Event', () => {
      // API の実行に成功する
      it('OK /events (POST)', async () => {
        const body: CreateRequest = {
          name: "test-event"
        }
        const res = await request(app.getHttpServer())
          .post('/events')
          .set('Accept', 'application/json')
          .send(body);
        expect(res.status).toEqual(201);

        const eventResponse = res.body as Event;
        expect(eventResponse).toHaveProperty('_id');
        expect(eventResponse.name).toEqual(body.name);
      });

      // 不正なパラメタで API の実行に失敗する
      it('NG /events (POST): Incorrect parameters', async () => {
        const body = {
          namee: "test-event"
        }
        const res = await request(app.getHttpServer())
          .post('/events')
          .set('Accept', 'application/json')
          .send(body);
        expect(res.status).toEqual(400);
      });

      // 空のパラメタで API の実行に失敗する
      it('NG /events (POST): Empty parameters.', async () => {
        const body = {}
        const res = await request(app.getHttpServer())
          .post('/events')
          .set('Accept', 'application/json')
          .send(body);
        expect(res.status).toEqual(400);
      });
    });

    // イベントの Read API に関するテスト
    describe('Read API of Event', () => {
      // API の実行に成功する
      it('OK /events (GET)', async () => {
        const eventsResponse = await getEventAll();
        expect(eventsResponse.length).toEqual(1);
      });

      // API の実行に成功する
      it('OK /events/:id (GET)', async () => {
        const eventsResponse = await getEventAll();

        const res = await request(app.getHttpServer())
          .get(`/events/${eventsResponse[0]._id}`);
        expect(res.status).toEqual(200);

        const eventResponse = res.body as Event;
        expect(eventResponse).toHaveProperty('_id');
        expect(eventResponse.name).toEqual('test-event');
      });

      // 不正な id で API の実行に失敗する
      it('NG /events/:id (GET): Invalid id.', async () => {
        const res = await request(app.getHttpServer())
          .get('/events/XXXXXXXXXXX');
        expect(res.status).toEqual(400);
      });

      // 存在しない id で API の実行に失敗する
      it('NG /events/:id (GET): id that doesn\'t exist.', async () => {
        const res = await request(app.getHttpServer())
          .get('/events/5349b4ddd2781d08c09890f4');
        expect(res.status).toEqual(404);
      });
    });

    // イベントの Update API に関するテスト
    describe('Update API of Event', () => {
        // API の実行に成功する
        it('OK /events/:id (PATCH)', async () => {
          const eventsResponse = await getEventAll();

          const body: UpdateRequest = {
            name: "new-test-event"
          }
          const res = await request(app.getHttpServer())
            .patch(`/events/${eventsResponse[0]._id}`)
            .set('Accept', 'application/json')
            .send(body);
          expect(res.status).toEqual(200);

          const eventResponse = res.body as Event;
          expect(eventResponse).toHaveProperty('_id');
          expect(eventResponse.name).toEqual(body.name);
        });

        // 不正な id で API の実行に失敗する
        it('NG /events/:id (PATCH): Incorrect parameters', async () => {
          const eventsResponse = await getEventAll();

          const body = {
            namee: "new-test-event"
          }
          const res = await request(app.getHttpServer())
            .patch(`/events/${eventsResponse[0]._id}`)
            .set('Accept', 'application/json')
            .send(body);
          expect(res.status).toEqual(400);
        });

        // 空のパラメタで API の実行に失敗する
        it('NG /events/:id (PATCH): Empty parameters.', async () => {
          const eventsResponse = await getEventAll();

          const body = {}
          const res = await request(app.getHttpServer())
            .patch(`/events/${eventsResponse[0]._id}`)
            .set('Accept', 'application/json')
            .send(body);
          expect(res.status).toEqual(400);
        });

        // 空の id で API の実行に失敗する
        it('NG /events/:id (PATCH): Empty id.', async () => {
          const eventsResponse = await getEventAll();

          const body = {
            namee: "new-test-event"
          }
          const res = await request(app.getHttpServer())
            .patch('/events')
            .set('Accept', 'application/json')
            .send(body);
          expect(res.status).toEqual(404);
        });

        // 不正な id で API の実行に失敗する
        it('NG /events/:id (PATCH): Invalid id.', async () => {
          const body: UpdateRequest = {
            name: "new-test-event"
          }
          const res = await request(app.getHttpServer())
            .patch('/events/XXXXXXXXXXX')
            .set('Accept', 'application/json')
            .send(body);
          expect(res.status).toEqual(400);
        });

        // 存在しない id で API の実行に失敗する
        it('NG /events/:id (PATCH): id that doesn\'t exist.', async () => {
          const body = {}
          const res = await request(app.getHttpServer())
            .patch('/events/5349b4ddd2781d08c09890f4')
            .set('Accept', 'application/json')
            .send(body);
          expect(res.status).toEqual(404);
        });
    });

    // イベントの Delete API に関するテスト
    describe('Delete API of Event', () => {
        // API の実行に成功する
        it('OK /events/:id (DELETE)', async () => {
          const eventsResponse = await getEventAll();

          const res = await request(app.getHttpServer())
            .delete(`/events/${eventsResponse[0]._id}`);
          expect(res.status).toEqual(200);
        });

        // 空の id で API の実行に失敗する
        it('NG /events/:id (DELETE): Empty id.', async () => {
          const res = await request(app.getHttpServer())
            .delete('/events')
          expect(res.status).toEqual(404);
        });

         // 不正な id で API の実行に失敗する
        it('NG /events/:id (DELETE): Invalid id.', async () => {
          const res = await request(app.getHttpServer())
            .delete('/events/XXXXXXXXXXX')
          expect(res.status).toEqual(400);
        });

        // 存在しない id で API の実行に失敗する
        it('NG /events/:id (DELETE): id that doesn\'t exist.', async () => {
          const res = await request(app.getHttpServer())
            .delete('/events/5349b4ddd2781d08c09890f4');
          expect(res.status).toEqual(404);
        });
    });
  })

  // 各テスト実行後は app を破棄する　(DB コネクションの解放を行わないと、テストを完了することが出来ない)
  afterEach(async () => {
    await app.close();
  });
});
```

上記内容で E2E テストを実行してみます。E2E テストには理想の結果を書いたので、正常系は手動で確認したとき同様成功するはずですが、異常系のテストはほとんど失敗しているはずです。

```bash
# 1. Docker Compose で MongoDB を起動する
cd devenv
docker-compose up -d

# 2. E2E テストを実行する
npm run test:e2e

# 3. Docker Compose で MongoDB の破棄する
cd devenv
docker-compose down
```

![E2E テストの実行結果](https://i.gyazo.com/59c1cf9a4ac81f194af6de75b4ae0098.png)
*E2E テストの実行結果 (失敗)*

## E2E テストが通るようにイベントの CRUD 機能を改修する

テストを通すために、例外処理周りの記述を追記していく際、NestJS の [Exception filters](https://docs.nestjs.com/exception-filters) を利用していきます。**NestJS の Exception filters を利用することで簡易に例外処理を実装することが可能になると同時に、適切な範囲でそれらの適用範囲を設定することが可能になります。**

今回は主に Mongoose 周りのエラーをフィルタリングしていきたいので Mongoose に関する `ExceptionFilter` を作成します。

```typescript:src/mongoose.exception.filter.ts
import { ArgumentsHost, Catch, ExceptionFilter, HttpStatus } from '@nestjs/common';
import { Error } from 'mongoose';

@Catch(Error)
export class MongooseExceptionFilter implements ExceptionFilter {
  catch(exception: Error, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse();
    switch(exception.name) {
      // Mongoose の検証エラーが発生したら HTTP BadRequest エラーを返却する
      case Error.ValidationError.name:
      case Error.CastError.name:
        response.status(HttpStatus.BAD_REQUEST).json(null);
        break;
      // Mongoose でデータが見つからなかった時に HTTP NotFound エラーを返却する
      case Error.DocumentNotFoundError.name:
        response.status(HttpStatus.NOT_FOUND).json(null);
        break;
    }
  }
}
```

`EventsController` に先程作成した `MongooseExceptionFilter` を適用します。これにより、MongoDB クエリの例外処理をコントローラー全体に一括で設定可能です。

```typescript:src/events/events.controller.ts
import { Param, Body, Controller, Get, Post, Patch, Delete, UseFilters } from '@nestjs/common';
import { EventsService } from './events.service'
import { Event } from './events.schema'
import {
    CreateEventDto,
    UpdateEventDto
} from './events.dto'
import { MongooseExceptionFilter } from '../mongoose.exception.filter';

@Controller('events')
// 関数全てに適用したいので class 定義の上で @UseFilters デコレータを用いて
// コントローラーの関数全てに MongooseExceptionFilter を適用する
@UseFilters(MongooseExceptionFilter)
export class EventsController {
    constructor(private readonly eventsService: EventsService) {}

    @Post()
    async create(@Body() request: CreateEventDto): Promise<Event> {
        return this.eventsService.create(request.name)
    }

    @Get(':id')
    async readOne(@Param('id') id: string): Promise<Event> {
        return this.eventsService.readOne(id)
    }

    @Get()
    async readAll(): Promise<Event[]> {
        return this.eventsService.readAll()
    }

    @Patch(':id')
    async update(@Param('id') id: string, @Body() request: UpdateEventDto): Promise<Event> {
        return this.eventsService.update(id, request.name)
    }

    @Delete(':id')
    async delete(@Param('id') id: string): Promise<Event> {
        return this.eventsService.delete(id)
    }
}
```

また Update 時に name の null チェックを行いたいので `EventsService` を改修します。null 時は明示的に Mongoose の Error を作成して `throw` することで、例外を `MongooseExceptionFilter` に補足させます。

```typescript:src/events/events.service.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model, Error } from 'mongoose';
import { Event } from './events.schema'

@Injectable()
export class EventsService {
    constructor(
        @InjectModel(Event.name)
        private eventModel: Model<Event>,
    ) {}

    async create(name: string): Promise<Event> {
        const createdEvent = new this.eventModel({ name: name });
        return createdEvent.save();
    }

    async readOne(id: string): Promise<Event> {
        return this.eventModel.findById(id).orFail().exec();
    }

    async readAll(): Promise<Event[]> {
        return this.eventModel.find().exec();
    }

    async update(id: string, name: string): Promise<Event> {
        // name の null チェックをおこなう
        // https://github.com/Automattic/mongoose/issues/6161#issuecomment-368242099
        if(name == null) {
            const validationError = new Error.ValidationError(null);
            validationError.addError('docField', new Error.ValidatorError({ message: 'Empty name.' }));

            throw validationError;
        }
        return this.eventModel.findByIdAndUpdate(
            id, { name: name }, { new: true }
        ).orFail().exec();
    }

    async delete(id: string): Promise<Event> {
        return this.eventModel.findByIdAndDelete(id).orFail().exec();
    }
}
```

再度 E2E テストを実行してみます。今度は無事 E2E テストが全て通ることを確認できるはずです。

```bash
# 1. Docker Compose で MongoDB を起動する
cd devenv
docker-compose up -d

# 2. E2E テストを実行する
npm run test:e2e

# 3. Docker Compose で MongoDB の破棄する
cd devenv
docker-compose down
```

![E2E テストの実行結果](https://i.gyazo.com/6a835d625b1363f8a06079f517338e72.png)
*E2E テストの実行結果 (成功)*

しかし、現状だと手動で Docker Compose で MongoDB を起動して閉じるフローが挟まっているため、E2E テストを走らせるためには手動でいくつかの作業が必要な状況です。

流石に E2E テストの実行が面倒な上、CI で走らせることが出来ないため NestJS アプリケーション自体も Docker Compose に乗せていきます。それにより、自動で MongoDB と NestJS アプリケーションを同時に起動して E2E テストが実行できる環境を構築します。
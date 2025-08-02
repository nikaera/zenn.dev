---
dev_article_id: 2747536
title: "OpenNext + Drizzle で Cloudflare D1開発・テスト環境をシームレスに構築する"
emoji: "🌧️"
type: "tech"
topics: ["drizzleorm", "cloudflare", "d1", "nextjs", "typescript"]
published: false
---

# はじめに

OpenNext + Cloudflare D1 を使った開発で、ローカル環境での開発・テスト・本番への移行を効率的に行いたいと考えたことはありませんか。

愚直に実現しようとすると、環境ごとに異なる設定ファイルを用意したり、テストデータの管理に手間がかかります。**[Drizzle](https://orm.drizzle.team/) のエコシステム（`drizzle-orm`、`drizzle-kit`、`drizzle-seed`）を活用することで、これらの課題を解決し、シームレスな開発体験を実現する方法を紹介します。**

この記事では、実際の [`@opennextjs/cloudflare`](https://opennext.js.org/cloudflare) プロジェクト構成を参考に、ローカル開発からテスト実装、本番デプロイまでの一連の流れを解説します。

# 動作環境

```json:package.json
{
  "dependencies": {
    "@libsql/client": "^0.15.10",
    "@opennextjs/cloudflare": "^1.6.2",
    "drizzle-orm": "^0.44.4"
  },
  "devDependencies": {
    "drizzle-kit": "^0.31.4",
    "drizzle-seed": "^0.3.1",
    "vitest": "^3.2.4",
    "wrangler": "^4.26.1"
  }
}
```

# プロジェクト構成

この記事で構築する最終的な関連したプロジェクト構造は以下のようになります。

```bash
./
├── drizzle/
│   ├── migrations/           # マイグレーションファイルの格納先
│   │   └── 0000_init.sql
│   └── schema.ts            # データベーススキーマの定義
├── tests/
│   ├── db.test.ts           # drizzle-seed を使ったテスト
│   └── db.ts                # テスト用 DB のユーティリティ
├── drizzle.config.ts        # 各種 DB の設定ファイル
├── wrangler.jsonc           # Cloudflare Workers 設定 (D1)
└── package.json
```

## 1. 必要なパッケージのインストール

まず、必要なパッケージをインストールします。

```bash
npm install drizzle-orm @libsql/client
npm install -D drizzle-kit drizzle-seed vitest
```

## 2. Cloudflare D1 の設定

`wrangler.jsonc` に D1 データベースの情報を追記します。

```json:wrangler.jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "<your-app-name>",
  "main": ".open-next/worker.js",
  "compatibility_date": "2025-03-01",
  "compatibility_flags": ["nodejs_compat", "global_fetch_strictly_public"],
  // Cloudflare D1 データベースの情報
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "<your-database-name>",
      "database_id": "<your-database-id>",
      "migrations_dir": "drizzle/migrations"
    }
  ]
}
```

`<your-database-name>` と `<your-database-id>` には、  
D1 データベースの「データベース名」と「データベースID」をそれぞれ指定します。

### 値の確認・発行方法

1. **Cloudflare ダッシュボードから確認**  
    - [Cloudflare Dashboard](https://dash.cloudflare.com/) にログインし、  
    対象アカウントの「ストレージとデータベース」→「D1 SQL データベース」へ移動する
    - 作成済みのデータベース一覧から、該当データベースを選択する
    - `<your-database-name>` は一覧や詳細画面での表示名
      - 例: `my-app-dev`
    - `<your-database-id>` は詳細画面の「UUID」
      - 例: `e216461a-74c3-40b2-8819-9fa351827304`

2. **Wrangler CLI で新規作成する場合**  
    - 下記でコマンド成功時に出力される JSON フィールドの値を用いる 
      - `database_name` が `<your-database-name>`
      - `database_id` が `<your-database-id>`
    ```bash
    npx wrangler d1 create my-app-dev
    # ...
    Successfully created DB 'my-app-dev' in region APAC
    Created your new D1 database.

    {
      "d1_databases": [
        {
          "binding": "DB",
          "database_name": "my-app-dev",
          "database_id": "e216461a-74c3-40b2-8819-9fa351827304"
        }
      ]
    }
    ```

これらの値を `wrangler.jsonc` の該当箇所にコピー&ペーストしてください。

## 3. 環境別のデータベース設定

`drizzle.config.ts` に**本番環境とローカル環境の設定を記載します。**  

環境の切り分けには `NODE_ENV` 環境変数を利用しており、  
`NODE_ENV=production` の場合は本番用、そうでない場合はローカル用の設定が適用されます。

```typescript:drizzle.config.ts
import { readdirSync } from "fs"
import { defineConfig } from 'drizzle-kit'

const isProduction = process.env.NODE_ENV === 'production';
// `sqliteDirPath` は、wrangler で D1 環境を設定し、`dev` コマンドを実行した際に
// SQLite ファイルが生成されるディレクトリパスを指します。
const sqliteDirPath = '.wrangler/state/v3/d1/miniflare-D1DatabaseObject';
const sqliteFilePath = readdirSync(sqliteDirPath).find(file => file.endsWith('.sqlite'));

const config = isProduction ? defineConfig({
  out: './drizzle/migrations',
  schema: './drizzle/schema.ts',
  dialect: 'sqlite',
  driver: 'd1-http',
  dbCredentials: {
    accountId: process.env.CLOUDFLARE_ACCOUNT_ID!,
    databaseId: process.env.CLOUDFLARE_DATABASE_ID!,
    token: process.env.CLOUDFLARE_D1_TOKEN!,
  }
}) : defineConfig({
  out: './drizzle/migrations',
  schema: './drizzle/schema.ts',
  dialect: 'sqlite',
  dbCredentials: {
    url: `${sqliteDirPath}/${sqliteFilePath!}`,
  }
});

export default config;
```

:::message
`wrangler` のバージョンや環境によっては、SQLite ファイルの生成先ディレクトリが `.wrangler/state/v3/d1/miniflare-D1DatabaseObject` ではなく、`.wrangler` 配下の異なるパスになる可能性があります。

`.wrangler` 配下の SQLite ファイルのパスが不明な場合は、`find` コマンドで検索し特定できます。たとえば、以下のように実行すると、`.sqlite` 拡張子のファイルを探せます。

```bash
find .wrangler -type f -name "*.sqlite"
```
:::

Cloudflare D1 の本番環境の設定には、以下の環境変数が必要です。

- `CLOUDFLARE_ACCOUNT_ID`
- `CLOUDFLARE_DATABASE_ID`
- `CLOUDFLARE_D1_TOKEN`

本番環境設定の取得方法に関しては下記を参考にしてください。

https://zenn.dev/arafipro/articles/2024-07-24-drizzle-d1-table#drizzle.config.ts-1

## 4. データベーススキーマの定義

`drizzle/schema.ts` で以下の2つのテーブルを定義します。

- **conversations テーブル**  
  チャットの会話単位を管理するテーブル

- **messages テーブル**  
  各会話に紐づくメッセージを管理するテーブル、  
  `conversationId` で `conversations` テーブルとリレーションする。

```typescript:drizzle/schema.ts
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';
import { sql } from 'drizzle-orm';

export const conversations = sqliteTable('conversations', {
  id: text('id').primaryKey(),
  title: text('title'),
  createdAt: integer('created_at', { mode: 'timestamp' })
    .notNull()
    .default(sql`(unixepoch())`),
  updatedAt: integer('updated_at', { mode: 'timestamp' })
    .notNull()
    .default(sql`(unixepoch())`),
});

export const messages = sqliteTable('messages', {
  id: text('id').primaryKey(),
  conversationId: text('conversation_id')
    .notNull()
    .references(() => conversations.id, { onDelete: 'cascade' }),
  role: text('role', { enum: ['user', 'system'] }).notNull(),
  content: text('content').notNull(),
  createdAt: integer('created_at', { mode: 'timestamp' })
    .notNull()
    .default(sql`(unixepoch())`),
});
```

## 5. テスト環境の構築

`tests/db.ts` でテスト用のユーティリティ関数を作成します。

```typescript:tests/db.ts
import { createClient } from '@libsql/client/node';
import { drizzle } from 'drizzle-orm/libsql';
import * as schema from '../drizzle/schema';
import { readFileSync } from 'fs';
import { join } from 'path';
import { seed, reset } from 'drizzle-seed';

export async function createTestDatabase() {
  const client = createClient({ url: ':memory:' });
  const db = drizzle(client, { schema });
  
  // `drizzle-kit` で生成された、実際のマイグレーションファイルを読み込む
  const migrationsDir = join(process.cwd(), 'drizzle/migrations');
  const migrationFiles = (
    await Promise.resolve(
      require('fs').readdirSync(migrationsDir)
        .filter((file: string) => file.endsWith('.sql'))
        .sort()
    )
  );

  // 正しい順序で逐次マイグレーションファイルを実行する
  for (const file of migrationFiles) {
    const migrationSQL = readFileSync(join(migrationsDir, file), 'utf-8');
    const statements = migrationSQL
      .split('--> statement-breakpoint')
      .map(stmt => stmt.trim())
      .filter(stmt => stmt.length > 0);

    for (const statement of statements) {
      await client.execute(statement);
    }
  }
  
  return { client, db };
}

export async function seedTestDataWithDrizzleSeed(
  db: any, 
  options?: { count?: number; seed?: number }
) {
  const { count = 3, seed: seedValue = 12345 } = options || {};
  
  // `drizzle-seed` の `refine` でテスト時に利用するシードデータを生成
  await seed(db as any, schema, { count, seed: seedValue }).refine((f) => ({
    conversations: {
      count: 3,
      columns: {
        title: f.valuesFromArray({
          values: [
            'AI System Chat',
            'Technical Q&A Session', 
            'Creative Writing Help',
            'Code Review Discussion',
            'Product Planning Talk'
          ]
        }),
      }
    },
    messages: {
      count: 6,
      columns: {
        role: f.valuesFromArray({
          values: ['user', 'system']
        }),
        content: f.valuesFromArray({
          values: [
            'Hello! How can you help me today?',
            'I\'m an AI system, happy to help with various tasks!',
            'Can you explain how TypeScript interfaces work?',
            'TypeScript interfaces define the shape of objects...',
            'What are the best practices for React development?',
            'Some key React best practices include using functional components...',
          ]
        }),
      }
    }
  }));
  
  return db;
}

export async function resetDatabase(db: any) {
  await reset(db as any, schema);
}
```

## 6. テストの実装

`tests/db.test.ts` で実際のテストを実装します。

```typescript:tests/db.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { conversations, messages } from '../drizzle/schema';
import { eq } from 'drizzle-orm';
import { 
  createTestDatabase,
  seedTestDataWithDrizzleSeed, 
  resetDatabase 
} from './db';

describe('Drizzle Seed Test', () => {
  let client: any;
  let db: any;

  beforeEach(async () => {
    const testDb = await createTestDatabase();
    client = testDb.client;
    db = testDb.db;
  });

  afterEach(() => {
    client.close();
  });

  it('should seed test data using drizzle-seed', async () => {
    await seedTestDataWithDrizzleSeed(db, { count: 3, seed: 12345 });

    // 会話データの検証
    const allConversations = await db.select().from(conversations);
    expect(allConversations.length).toBeGreaterThan(0);

    // メッセージデータの検証
    const allMessages = await db.select().from(messages);
    expect(allMessages.length).toBeGreaterThan(0);

    // 外部キー制約の検証
    for (const message of allMessages) {
      const conversation = allConversations.find((c: any) => 
        c.id === message.conversationId
      );
      expect(conversation).toBeDefined();
    }

    // ロール値の検証
    for (const message of allMessages) {
      expect(['user', 'system']).toContain(message.role);
    }
  });

  it('should reset database correctly', async () => {
    await seedTestDataWithDrizzleSeed(db);

    let allConversations = await db.select().from(conversations);
    expect(allConversations.length).toBeGreaterThan(0);

    await resetDatabase(db);

    allConversations = await db.select().from(conversations);
    const allMessages = await db.select().from(messages);

    expect(allConversations).toHaveLength(0);
    expect(allMessages).toHaveLength(0);
  });
});
```

# 実際の開発フロー

**予め `package.json` の `scripts` へ下記を登録しておきます。**

```json:package.json
{
  "scripts": {
    "test": "vitest run",
    "db:gen": "drizzle-kit generate",
    "db:mgn": "drizzle-kit migrate",
    "db:mgn:prd": "NODE_ENV=production drizzle-kit migrate",
    "db:client": "drizzle-kit studio"
  }
}
```

## 1. DB のローカル環境構築

```bash
# Wrangler dev でローカル開発環境を起動
npx wrangler dev

# 別ターミナルでマイグレーションを実行
npm run db:gen # 必要に応じて `-- --name <ファイル名>`
npm run db:mgn

# 適宜 DB 絡むテストを実行
npm test
```

Drizzle には、[Drizzle Studio](https://orm.drizzle.team/drizzle-studio/overview) という GUI で DB を確認可能なツールが存在します。  
Drizzle Studio は下記コマンドで起動可能です。

```bash
# DB クライアント (Drizzle Studio) の起動
npm run db:client
```

## 2. DB の本番環境への反映

```bash
# 本番環境でのマイグレーション
npm run db:mgn:prd
```

# まとめ

**Drizzle のエコシステムを活用することで、OpenNext + Cloudflare D1 を使った開発・テスト・本番環境をシームレスに構築できました。**

OpenNext を利用したい動機として、複雑な設定や環境構築に時間を取られることなく、迅速かつ効率的にアプリケーションの実装を進めたいというモチベで採用されることが大半だと考えます。

本記事の手法を使うことで、**データベース周りの煩雑な設定や運用から解放されます。**  
そのため、アプリケーションロジックの実装に集中できます。爆速で開発を進めていきましょう。

# 参考リンク

- [Drizzle ORM \- next gen TypeScript ORM\.](https://orm.drizzle.team/)
- [Overview · Cloudflare D1 docs](https://developers.cloudflare.com/d1/)
- [Drizzle ORM \- Migrations with Drizzle Kit](https://orm.drizzle.team/docs/kit-overview)
- [Drizzle ORM \- Drizzle Seed](https://orm.drizzle.team/docs/seed-overview)
- [Index \- OpenNext](https://opennext.js.org/cloudflare)

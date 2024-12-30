---
title: "GraphQLで適当にアプリを作ってみた"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "Docker", "React", "PostgreSQL"]
published: false
---

## はじめに
- GraphQLにさわる機会があったので、勉強がてら遊んでみることにした。

## 構成

## 用語

## 環境
- OS: Windows 11 Pro (ビルド：22631)
- Terminal: Ubuntu 24.04.1 LTS (WSL) 
- Docker Compose: v2.32.1
- Nodejs: 18.19.1
- npm: 9.2.0

## 手順

```bash
$ mkdir graphql-app
$ cd graphql-app/
$ npm init -y
$ npm install --save-dev typescript @types/node ts-node
$ npx tsc --init
```
と打つと、

```bash
Created a new tsconfig.json with:                                                                                           
                                                                                                                     TS     
  target: es2016
  module: commonjs
  strict: true
  esModuleInterop: true
  skipLibCheck: true
  forceConsistentCasingInFileNames: true
```
必要なパッケージをインストールします。

```bash
npm install express @types/express
npm install @apollo/server graphql
npm install --save-dev prisma
npm install --save-dev cors nodemon @types/cors

```

package.jsonに起動スクリプト追加

```diff_json:    package.json
"scripts": {
+    "start": "nodemon --exec ts-node --esm index.ts",
    "test": "echo \"Error: no test specified\" && exit 1"
},
```

docker-composeにて、postgresのdbを起動するための設定を記載。

```yaml
version: '3'

services:
  db:
    image: postgres:17.2
    container_name: postgres_container
    ports:
      - 5432:5432
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: password

```

- 起動する。

```bash
$ docker compose up -d 
```

- DBにログインする。

```bash
$ docker exec -it postgres_container psql -U myuser -d postgres
```

### Prisma

```bash
npx prisma init
```

```prisma:schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

```env: .env
- DATABASE_URL="postgresql://johndoe:randompassword@localhost:5432/mydb?schema=public"
+ DATABASE_URL="postgresql://myuser:password@localhost:5432/postgres?schema=public"
```

- モデルの定義

```prisma: schema.prisma

model task {
  id       Int    @id @default(autoincrement())
  title    String
  deadline String
  complete Boolean
}

```

- マイグレーション

```bash

npx prisma migrate dev

# The following migration(s) have been created and applied from new schema changes:

# migrations/
#   └─ 20241230025115_1/
#     └─ migration.sql

# Your database is now in sync with your schema.

# Running generate... (Use --skip-generate to skip the generators)

# ✔ Generated Prisma Client (v6.1.0) to ./node_modules/@prisma/client in 381ms

```

以下のコードを記載する。

```typescript: index.ts
import express from 'express';
import { ApolloServer, BaseContext } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import cors from 'cors';
import { PrismaClient } from '@prisma/client';

const prisma: PrismaClient = new PrismaClient();

const app = express();
const port: number = 3000;

// ここに今からコードを書いていく。

const typeDefs = `
  type Task {
    id: Int!
    title: String!
    deadline: String
    complete: Boolean!
  }

  type Query {
    hello(name: String): String
    greeting: String
    getTasks: [Task]
  }

  type Mutation {
    addTask(title: String!, deadline: String): Task
    deleteTask(id: Int!): Task
    updateTask(id: Int!, title: String!, deadline: String, Complete: Boolean!): Task
  }
`;

const greetings: string[] = [
    'Hello!',
    '¡Hola!',
    'こんにちは!',
    '你好!',
    'bonjour!',
];
  
// ランダムな数字を返す
const getRandomValue = (max: number): number => {
    return Math.floor(Math.random() * max);
};

const resolvers = {
    Query: {
      hello: (parent: any, args: { name: string }) => {
        return `Hello, ${args.name}!`;
      },
      // ランダムであいさつを返してもらう。
      greeting: () => {
        const max: number = greetings.length;
        return greetings[getRandomValue(max)];
      },
      // タスク全部取得
      getTasks: () => prisma.task.findMany(),
    },
    Mutation: {
      // タスク登録
      addTask: (parent: any, args: { title: string; deadline: string }) => {
        return prisma.task.create({
          data: {
            title: args.title,
            deadline: args.deadline,
            complete: false
          },
        });
      },
  
      // タスク削除
      deleteTask: (parent: any, args: { id: number }) => {
        return prisma.task.delete({
          where: {
            id: args.id,
          },
        });
      },
  
      // タスク更新
      updateTask: (parent: any, args: { id: number; title: string; deadline: string; complete: boolean }) => {
        return prisma.task.update({
          where: {
            id: args.id,
          },
          data: {
            id: args.id,
            title: args.title,
            deadline: args.deadline,
            complete: args.complete
          },
        });
      },
    }
  };

// ApolloServer初期化
const server = new ApolloServer<BaseContext>({
    typeDefs,
    resolvers,
});
  
// ApolloServer起動
await server.start();
  
// Expressのミドルウェア設定
app.use(
    '/api',
    cors<cors.CorsRequest>(),
    express.json(),
    expressMiddleware(server)
);

// サーバ起動
await new Promise<void>((resolve) => app.listen({ port: port }, resolve));
console.log(`🚀 Server ready at http://localhost:${port}/`);
```

- awaitのところで以下のエラーが
```bash
Top-level 'await' expressions are only allowed when the 'module' option is set to 'es2022', 'esnext', 'system', 'node16', 'nodenext', or 'preserve', and the 'target' option is set to 'es2017' or higher.ts(1378)
```

- tsconfigの`module`オプションと、`target`を変更せよ、とのことなので、それぞれ、`NodeNext`, `ESNext`に変更する。

- 今度は以下のエラーが。
```bash
The current file is a CommonJS module and cannot use 'await' at the top level.ts(1309)
```

- package.jsonに、`"type": "module"` を追加した。
- `expressMiddleware` について、expressと、apolloのexpressで競合して要るっぽいエラーが出るがここは無視しても動いたので無視する。

ここで、アプリケーションを立ち上げてみる。

```bash
$ npm start
```
すると、以下のエラーが。

```bash
TypeError [ERR_UNKNOWN_FILE_EXTENSION]: Unknown file extension ".ts" for ./graphql-app/index.ts
    at new NodeError (node:internal/errors:405:5)
    at Object.getFileProtocolModuleFormat [as file:] (node:internal/modules/esm/get_format:136:11)
    at defaultGetFormat (node:internal/modules/esm/get_format:182:36)
    at defaultLoad (node:internal/modules/esm/load:101:20)
    at nextLoad (node:internal/modules/esm/hooks:864:28)
    at load (/mnt/c/Users/soshi/Documents/Program/graphql-app/node_modules/ts-node/dist/child/child-loader.js:19:122)
    at nextLoad (node:internal/modules/esm/hooks:864:28)
    at Hooks.load (node:internal/modules/esm/hooks:447:26)
    at MessagePort.handleMessage (node:internal/modules/esm/worker:196:24)
    at [nodejs.internal.kHybridDispatch] (node:internal/event_target:786:20) {
  code: 'ERR_UNKNOWN_FILE_EXTENSION'

```

いろいろ解決策を調べたが、以下で何とかなった。

```bash
$ npm install --save-dev tsx
```
package.jsonを変更する。

```diff_json: package.json
  "scripts": {
-    "start": "nodemon --exec ts-node --esm index.ts",
+    "start": "npx tsx index.ts"
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```

- これで起動した

```bash
$ npm start

> graphql-app@1.0.0 start
> npx tsx index.ts

🚀 Server ready at http://localhost:3000/
```

![](/images/articles/graphql-app/apollo-server-top.png)

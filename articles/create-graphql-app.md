---
title: "GraphQLで適当にアプリを作ってみた"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "Docker", "Vue.js", "PostgreSQL"]
published: false
---

## はじめに
AWS AppSyncについて調べてみて、結局GraphQLをマネージドでできるサービスなんだからGraphQL触ったらええやん、ってなり遊んでみることにしました。

## 作成するアプリおよび構成

### 作成するアプリケーション
今回は超簡単なタスク管理アプリを作成してみます。以下のような画面イメージ。

![](/images/articles/graphql-app/vue_3.png)

### 構成図
せっかくなので、フロントエンド、バックエンド、およびデータベースをすべてdockerコンテナ化し、docker-composeで管理することとします。

![](/images/articles/graphql-app/architecture.png)

シンプルですが絵で表現するの大事ですね。


### フォルダ構成

今回作成したアプリケーションのフォルダ構成はこんな感じ。<br>
- `graphql-app`：graphqlのAPIのソースコードを管理しているディレクトリ
- `graphql-frontend`：フロントエンドのソースコードを管理しているディレクトリ


```bash
graphql
├── docker-compose.yml
├── graphql-app/
│   ├── dockerfile
│   ├── index.ts
│   ├── package-lock.json
│   ├── package.json
│   ├── node_module/
│   ├── prisma/
│   │   └── schema.prisma
│   └── tsconfig.json
└── graphql-frontend/
    ├── node_module/
    ├── dockerfile
    ├── index.html
    ├── jsconfig.json
    ├── package-lock.json
    ├── package.json
    ├── public/
    │   └── favicon.ico
    ├── src/
    │   ├── App.vue
    │   └── main.js
    └── vite.config.js
```

## その前に

### GarphQLとは

GraphQLは[公式HP](https://graphql.org/)に記載の通り、APIのためのクエリ言語となります。
APIを作成するための決まった構文があるんだなーと思えばいいかと思います。

よく対比されるのがREST APIですが、これは、エンドポイントを分けたり、HTTPのMethod (GET, POST, OPTIONS, etc.)を使い分けたりすることでデータのやり取りを行います。
それに対して、GraphQLは、エンドポイントは同一で、POST Methodの中にクエリ言語を埋め込むことでデータのやり取りを行います。

それぞれのメリット/デメリットは[こちら](https://kinsta.com/jp/blog/graphql-vs-rest/)、
REST APIとの挙動の違いは[こちら](https://hasura.io/learn/ja/graphql/intro-graphql/graphql-vs-rest/)を見るとわかりやすいかと思います。

また、GraphQLを理解するために必須の用語について、以下に整理しておきます。

### 用語

|用語|意味|
|---|---|
|Schema|APIのデータ構造の定義。GraphQLスキーマ言語（SDL）で記述される。|
|Query|オペレーションの１つ。データを取得する際に利用する。|
|Mutation|オペレーションの1つ。データを作成、更新、削除するときに利用する。|
|Subscription|オペレーションの1つ。リアルタイム更新を可能にする。|
|Resolver|オペレーションの中身(挙動)を定義する。|


## 環境
- OS: Windows 11 Pro (ビルド：22631)
- Terminal: Ubuntu 24.04.1 LTS (WSL) 
- Docker Compose: v2.32.1
- Nodejs: 18.19.1
- npm: 9.2.0

## 実際に構築してみる

### バックエンド (Apollo Server, PostgreSQL)

まずは、Apollo Server (GraphQLのサーバーサイド実装のためのオープンソースライブラリ) を使って、バックエンド処理を作成してきます。

#### プロジェクトの作成

以下のようにTypescriptのプロジェクトを作成します。

```bash
$ mkdir graphql-app
$ cd graphql-app/
$ npm init -y
$ npm install --save-dev typescript @types/node ts-node
$ npx tsc --init
```
を実行すると、以下の用にセットアップが完了します。

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
以下のように、必要なパッケージをインストールします。

```bash
npm install express @types/express
npm install @apollo/server graphql
npm install --save-dev prisma
npm install --save-dev cors ts-node nodemon @types/cors
```
- `express`は、node.jsのためのWebアプリケーションライブラリで、`@types/express`はtypescriptの型定義パッケージです。
- `prisma`は、ORM (Object-Relational-Mapping)のライブラリで、データベースとのやり取りを扱いやすくするために使用します。
- `cors`はCORS対応のため、`ts-node`はtypescriptを直接実行するため、 `nodemone` はコードの変更を自動で検知してnode.jsのプロセスを再起動するために使用します。（後程書きますが、結局 `ts-node`, `nodemon`は使いませんでした。)

package.jsonに起動をスクリプトを追加します。
これにより、`npm start` コマンドで、記載したコマンドが実行されるようになります。

```diff json:package.json
"scripts": {
+   "start": "nodemon --exec ts-node --esm index.ts",
    "test": "echo \"Error: no test specified\" && exit 1"
},
```
#### データベースの作成

次に、データベース (PostgreSQL) を起動します。<br>
docker-composeで、PostgreSQLのDBを起動するための設定を記載します。

※ `docker-compoes.yml`はディレクトリ階層が `grapqhl-app`より1つ上になってます。


```yaml:docker-compose.yml
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
    volumes:
      - /usr/local/src/postgres:/var/lib/postgresql/data

```

以下コマンドで起動する。

```bash
$ docker compose up -d 
```

以下コマンドでDBにログインできるようになります。

```bash
$ docker exec -it postgres_container psql -U myuser -d postgres
```

#### Prisma

次にPrismaの設定を追加していきます。
再度、`graphql-app` ディレクトリに移動し、以下コマンドを実行します。

```bash
$ npx prisma init
```
各種ファイルが生成されます。
localhostで起動しているデータベースの設定を追加していきます。

```:schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

```diff :.env
- DATABASE_URL="postgresql://johndoe:randompassword@localhost:5432/mydb?schema=public"
+ DATABASE_URL="postgresql://myuser:password@localhost:5432/postgres?schema=public"
```

さらに、モデルの定義を追加してきます。<br>
ここに記載する内容が、PostgreSQLのDBのテーブルとなります。<br>

```prisma: schema.prisma
model task {
  id       Int    @id @default(autoincrement())
  title    String
  deadline String
  complete Boolean
}
```
`complete`なるタスク完了フラグを準備しましたが、実は今回使いませんでした。。。


次に、以下の通り、マイグレーションを実行します。<br>
こうすることで、PostgreSQL側でテーブルが作成されます。

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

#### ソースコードの記述

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

const typeDefs = `
  type Task {
    id: Int!
    title: String!
    deadline: String
    complete: Boolean!
  }

  type Query {
    getTasks: [Task]
  }

  type Mutation {
    addTask(title: String!, deadline: String): Task
    deleteTask(id: Int!): Task
    updateTask(id: Int!, title: String!, deadline: String, Complete: Boolean!): Task
  }
`;


const resolvers = {
    Query: {
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

ソースコードについて簡単に解説します。

※ このタイミングで躓き事項は[トラブルシューティング](#トラブルシューティング)に記載しました。

まず、



そして、以下のように `npm start`でサーバを立ち上げます。

```bash
$ npm start

> graphql-app@1.0.0 start
> npx tsx index.ts

🚀 Server ready at http://localhost:3000/
```

そして、`http://localhost:3000/api`にブラウザでアクセスすると、以下のキャプチャのようにApollo Serverの画面が立ち上がります。
この画面で、ソースコードで定義したQuery, Mutationが画面で実行できるようになります。

![](/images/articles/graphql-app/apollo-server-top.png)

Query, Mutaionを実行してみます。
まず、Queryからやってみます。

画面に以下を入力して、再生ボタンを押下し、`getTask`を実行してみます。

```bash
query getTask{
  getTasks {
    id,
    title,
    deadline,
    complete
  }
}
```

すると、以下のように、まだ何もタスクがデータベースに格納されていないので、空の値が返ってきます。
![](/images/articles/graphql-app/apollo-server-sample2.png)

次に、Mutationでタスクを挿入してみます。

```bash
mutation addTask($title: String!,  $deadline: String) {
  addTask(title: $title, deadline: $deadline) {
    id,
    title,
    deadline,
    complete
  }
}
```
を、入力し、画面下部の`Variables`に以下を入力後、同様に実行します。

```bash
{
  "title": "年越しそば",
  "deadline": "2024/12/31"
}
```

- 値が入力されたことがわかります（idが5なのはその前に遊んでいたから）。
![](/images/articles/graphql-app/apollo-server-sample3.png)

- 再度、`getTask`を実行すると、値が挿入されていることがわかります。
![](/images/articles/graphql-app/apollo-server-sample4.png)


次に、このapollo-serverをdockerコンテナ化して、PostgreSQLコンテナが稼働しているコンテナと同時に起動するようにdocker-composeを編集します。
まず、以下のようにdockerfileを作成します。

```dockerfile
FROM node:20-alpine3.19
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

次に、docker-compose.yamlを以下のように修正します。

```diff yaml: docker-compose.yml
version: '3'

services:
+ app:
+   build:
+     context: ./graphql-app
+      dockerfile: dockerfile
+   container_name: graphql
+   ports:
+     - "3000:3000"

  db:
    image: postgres:17.2
    container_name: postgres_container
    ports:
      - 5432:5432
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: password
    volumes:
      - /usr/local/src/postgres:/var/lib/postgresql/data

```

ここで、prismaの`.env`ファイルにて、dbの接続情報に記載の `localhost`を `db`に変更する。
docker-composeに組み込まれると、Apollo Serverはdocker-composeに記載の、`db`という名前で解決されるようになります。

```diff env:.env
- DATABASE_URL="postgresql://myuser:password@localhost:5432/postgres?schema=public"
+ DATABASE_URL="postgresql://myuser:password@db:5432/postgres?schema=public"
```
以下コマンドで、実行します。
```bash
$ docker compose up -d
```

うまく起動しないので、
```bash
$ docker compose logs -f 
```
でログを確認すると、以下のようなエラーが発生していました。

```bash
graphql-app         | PrismaClientInitializationError: Prisma Client could not locate the Query Engine for runtime "linux-musl-openssl-3.0.x".   
graphql-app         |
graphql-app         | This happened because Prisma Client was generated for "debian-openssl-3.0.x", but the actual deployment required "linux-musl-openssl-3.0.x".
graphql-app         | Add "linux-musl-openssl-3.0.x" to `binaryTargets` in the "schema.prisma" file and run `prisma generate` after saving it:   
graphql-app         |
graphql-app         | generator client {
graphql-app         |   provider      = "prisma-client-js"
graphql-app         |   binaryTargets = ["native", "linux-musl-openssl-3.0.x"]
postgres_container  | selecting dynamic shared memory implementation ... posix
postgres_container  | selecting default "max_connections" ... 100
graphql-app         | }
```

なにやらopenssl関連のエラーが出ていますので、エラー文言に記載の通り、`prisma.schema`に以下の記載します。

```diff schema:prisma.schema
generator client {
  provider = "prisma-client-js"
+ binaryTargets = ["native", "linux-musl-openssl-3.0.x"]
}
```

prisma関連のファイルを修正したので、以下コマンドを実行します。
```bash
$ npx prisma generate
```

`docker compose down` 実行後、`docker rmi` でイメージを消して再度、`docker compose up -d` を実行したところうまく起動しました。


## フロントエンド

次に、フロントエンドをVue.jsで構築していきます。
`graphql`ディレクトリの下で以下を実行します。
今回はプロジェクト名を`graphql-frontend`にしました。

```bash
npm create vue@latest

✔ Project name: … graphql-frontend
✔ Add TypeScript? … No / Yes
✔ Add JSX Support? … No / Yes
✔ Add Vue Router for Single Page Application development? … No / Yes
✔ Add Pinia for state management? … No / Yes
✔ Add Vitest for Unit Testing? … No / Yes
✔ Add an End-to-End Testing Solution? › No
✔ Add ESLint for code quality? › No
```

その後、以下コマンドで依存パッケージをインストール後、起動します。

```bash
$ cd graphql-frontend/
$ npm install
$ npm run dev
```
一旦、`http://localhost:5173/` にアクセスして、vue.jsのサンプル画面が表示されるか確認してみます。

![](/images/articles/graphql-app/vue_1.png)

うまく起動できていました。
続いて、実際にコードを書いてみたいと思います。


```bash
$ npm install @vue/apollo-composable @apollo/client graphql graphql-tag
```

```javascript: main.js
import { createApp, provide, h } from 'vue';
import { DefaultApolloClient } from '@vue/apollo-composable';
import { ApolloClient, InMemoryCache } from '@apollo/client';
import App from './App.vue';

const cache = new InMemoryCache();

const apolloClient = new ApolloClient({
  uri: 'http://localhost:3000/api',
  cache,
});

const app = createApp({
  setup() {
    provide(DefaultApolloClient, apolloClient);
  },
  render: () => h(App),
});

app.mount('#app');
```

とし、

```vue: App.vue
<template>
  <div>
    <h1>Task Management</h1>
    <ul>
      <li v-for="task in tasks" :key="task.id">
        {{ task.title }} ({{ task.deadline }})
        <button @click="deleteTask(task.id)">Delete</button>
      </li>
    </ul>
    <form @submit.prevent="addNewTask">
      <input v-model="newTaskTitle" placeholder="Task Title" required />
      <input v-model="newTaskDeadline" type="date" placeholder="Deadline" />
      <button type="submit">Add Task</button>
    </form>
  </div>
</template>

<script>
import { ref, watch } from 'vue';
import { gql } from 'graphql-tag';
import { useQuery, useMutation} from '@vue/apollo-composable';

const GET_TASKS = gql`
  query GetTasks {
    getTasks {
      id
      title
      deadline
    }
  }
`;

const ADD_TASK = gql`
  mutation AddTask($title: String!, $deadline: String) {
    addTask(title: $title, deadline: $deadline) {
      id
      title
      deadline
    }
  }
`;

const DELETE_TASK = gql`
  mutation DeleteTask($id: Int!) {
    deleteTask(id: $id) {
      id
    }
  }
`;

export default {
  setup() {
    const { result } = useQuery(GET_TASKS);

    const { mutate: addTask } = useMutation(ADD_TASK, {
      update(cache, { data: { addTask } }) {
        const data = cache.readQuery({ query: GET_TASKS });
        cache.writeQuery({
          query: GET_TASKS,
          data: {
            getTasks: [...data.getTasks, addTask],
          },
        });
      },
    });

    const { mutate: deleteTask } = useMutation(DELETE_TASK, {
      update(cache, { data: { deleteTask } }) {
        const data = cache.readQuery({ query: GET_TASKS });
        cache.writeQuery({
          query: GET_TASKS,
          data: {
            getTasks: data.getTasks.filter(task => task.id !== deleteTask.id),
          },
        });
      },
    });

    const tasks = ref([]);
    const newTaskTitle = ref('');
    const newTaskDeadline = ref('');

    watch(result, (value) => {
      tasks.value = value.getTasks;
    });

    const addNewTask = () => {
      addTask({
          title: newTaskTitle.value,
          deadline: newTaskDeadline.value,
      });

      newTaskTitle.value = '';
      newTaskDeadline.value = '';
    };

    const removeTask = (id) => { deleteTask({id}) };

    return {
      tasks,
      newTaskTitle,
      newTaskDeadline,
      addNewTask,
      deleteTask: removeTask,
    };
  },
};
</script>
```


```bash
 Uncaught Error: Could not resolve "react" imported by "rehackt". Is it installed?
```
とでたので、
```bash
npm install react
```
を実行してみます。

すると、うまくいきました。
タスクを追加したり、削除ができるようになっていますね。

![](/images/articles/graphql-app/vue_2.png)

画面がちょっとまだダサいので、ちょっとアレンジしてます。
`npm`で`vuetify`をインストールして、各ファイルを次の通り修正します。
ソースコードの細かい解説は割愛します。

（いまどき生成AIとかでチャット生成できますしね）

```bash
npm install vuetify@next
npm install vite-plugin-vuetify
```

`vite.cinfig.js`を以下のように修正します。

```diff js:vite.config.js
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueDevTools from 'vite-plugin-vue-devtools'
+ import vuetify from 'vite-plugin-vuetify'

// https://vite.dev/config/
export default defineConfig({
  plugins: [
    vue(),
    vueDevTools(),
+    vuetify({ autoImport: true })
  ],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    },
  },
})
```

そして、`main.js`を以下のように修正します。

```diff js:main.js
import { createApp, provide, h } from 'vue';
import { DefaultApolloClient } from '@vue/apollo-composable';
import { ApolloClient, InMemoryCache } from '@apollo/client';
import App from './App.vue';
+ import 'vuetify/styles';
+ import { createVuetify } from 'vuetify';

const cache = new InMemoryCache();
+ const vuetify = createVuetify();

const apolloClient = new ApolloClient({
  uri: 'http://localhost:3000/api',
  cache,
});

const app = createApp({
  setup() {
    provide(DefaultApolloClient, apolloClient);
  },
  render: () => h(App),
});

+ app.use(vuetify)
app.mount('#app');

```

`App.vue`も以下のように修正します。

```vue:App.vue
<template>
  <v-container>
    <v-row>
      <v-col cols="12">
        <v-card class="pa-4">
          <v-card-title>Task Management</v-card-title>
          <v-card-text>
            <v-list>
              <v-list-item v-for="task in tasks" :key="task.id">
                <v-row align="center"> 
                  <v-col cols="8">
                    <v-list-item-title >{{ task.title }}</v-list-item-title>
                    <v-list-item-subtitle>{{ task.deadline }}</v-list-item-subtitle>
                  </v-col>
                  <v-col cols="4" class="text-right">
                    <v-list-item-action>
                      <v-btn icon @click="deleteTask(task.id)">
                        削除
                      </v-btn>
                    </v-list-item-action>
                  </v-col>
                </v-row>
              </v-list-item>
            </v-list>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
    <v-row>
      <v-col cols="12">
        <v-form @submit.prevent="addNewTask">
          <v-text-field v-model="newTaskTitle" label="Task Title" required></v-text-field>
          <v-text-field
            v-model="newTaskDeadline"
            label="Deadline"
            type="date"
            required
          ></v-text-field>
          <v-btn color="primary" type="submit">Add Task</v-btn>
        </v-form>
      </v-col>
    </v-row>
  </v-container>
</template>

```

この通り、タスク管理アプリが完成しました。

![](/images/articles/graphql-app/vue_3.png)

## まとめ

今回は、Apollo Severを使って、簡単なgraphqlアプリを作りました。
ついでにdockerを使ってコンテナ化もチャレンジしました。
わりとすっとAPIが作成できて便利ですね。


## 参考

- https://zenn.dev/azunasu/articles/26556e1d21da0d

## トラブルシューティング

- awaitのところで以下のエラーが
```bash
Top-level 'await' expressions are only allowed when the 'module' option is set to 'es2022', 'esnext', 'system', 'node16', 'nodenext', or 'preserve', and the 'target' option is set to 'es2017' or higher.ts(1378)
```

- tsconfigの`module`オプションと`target`オプションを変更せよ、とのことなので、それぞれ、`NodeNext`, `ESNext`に変更する。

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

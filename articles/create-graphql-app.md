---
title: "GraphQLで適当にアプリを作ってみた"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "Docker", "Vue.js", "PostgreSQL"]
published: false
---

## はじめに
- GraphQLにさわる機会があったので、勉強がてら遊んでみることにした。

## 構成
- 今回目指す構成は以下である。

### フォルダ構成
- 今回作成したアプリケーションのフォルダ構成はこんな感じ。

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
    volumes:
      - /usr/local/src/postgres:/var/lib/postgresql/data

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

- ためしに、query, mutaionを実行してみる。
- まず、Queryから。以下を実行してみる。
```bash 
query greeting {
  greeting
}
```
- 以下のように、右ペインにResponseが表示される。

![](/images/articles/graphql-app/apollo-server-sample1.png)

- 次に、`getTask`を実行する。

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

- まだ何もタスクを入力していないので、空の値が返ってくる。
![](/images/articles/graphql-app/apollo-server-sample2.png)

- タスクを挿入してみる。

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
を、入力し、画面下部の`Variables`に以下を入力後、実行する。

```bash
{
  "title": "年越しそば",
  "deadline": "2024/12/31"
}
```

- 値が入力されたことがわかる（idが5なのはその前に遊んでいたから）。
![](/images/articles/graphql-app/apollo-server-sample3.png)

- 再度、`getTask`を実行すると、値が挿入されていることがわかる。
![](/images/articles/graphql-app/apollo-server-sample4.png)


- 次に、このapollo-serverをdockerコンテナ化する。以下のようにdockerfileを作成する。
```dockerfile
FROM node:20-alpine3.19
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

- docker-compose.yamlを以下のように修正する。
```diff_yaml: docker-compose.yml
version: '3'

services:
+  app:
+      build:
+        context: ./graphql-app
+        dockerfile: dockerfile
+      container_name: graphql
+      ports:
+        - "3000:3000"

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

- ここで、prismaの`.env`ファイルにて、dbの接続情報に記載の `localhost`を `db`に変更する。
```diff_env:.env
- DATABASE_URL="postgresql://myuser:password@localhost:5432/postgres?schema=public"
+ DATABASE_URL="postgresql://myuser:password@db:5432/postgres?schema=public"
```

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

- なにやらopenssl関連のエラーが出ているが、エラーの通り、`prisma.schema`に以下の記載をする。
```diff_schema:prisma.schema
generator client {
  provider = "prisma-client-js"
+  binaryTargets = ["native", "linux-musl-openssl-3.0.x"]
}
```

- 以下コマンドを実行する。
```bash
$ npx prisma generate
```

docker compose down実施後、docker rmi でイメージを消して再度、docker compose up -d を実行

- うまくいった。


## フロントエンド

- graphqlフォルダの下で以下を実行

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

```bash
$ cd graphql-frontend/
$ npm install
$ npm run dev
```
一旦、`http://localhost:5173/` にアクセスして、vue.jsのサンプル画面が表示されるか確認する。

![](/images/articles/graphql-app/vue_1.png)

続いて、実際にコードを書いてみたいと思う。、

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
を実行する。

すると、うまくいった。
タスクを追加したり、削除ができるようになっているかと思う。

![](/images/articles/graphql-app/vue_2.png)


```bash
npm install vuetify@next
npm install vite-plugin-vuetify
```

```js:vite.config.js
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

```js:main.js
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


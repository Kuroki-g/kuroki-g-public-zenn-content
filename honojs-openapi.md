# OpenAPIとHono.jsの利用

## OpenAPIとの統合の方針について

作成の方法によって変わる。現実的にフロントから作る場合はないと思う。

- backendから作る場合
    - バックエンド側がOpenAPIを生成する機構が必要
- OpenAPIから作る場合
    - OpenAPIを手動で作成し、バックエンドで統合する。


## Init project

空のディレクトリを作っておく

```bash
bash>~/openapi-sample/honojs$ ls
```

`npm create hono`で初期化。自分の好みでテンプレを選択。

```bash
bash>~/openapi-sample/honojs$ npm create hono@latest backend
Need to install the following packages:
create-hono@0.15.2
Ok to proceed? (y) 


> npx
> create-hono backend

create-hono version 0.15.2
✔ Using target directory … backend
? Which template do you want to use? nodejs
? Do you want to install project dependencies? no
✔ Cloning the template
```

npm i

```bash
bash>~/openapi-sample/honojs/backend$ npm i
```

一応この時点でエラーがないか動作確認

```bash
bash>~/openapi-sample/honojs/backend$ npm run dev

> dev
> tsx watch src/index.ts

Server is running on http://localhost:3000
```

### backendから作る場合
---
title: "チュートリアル: ASP.NET Core を使って最小 API を作成する - F#"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fsharp", "dotnet", "aspnetcore", "openapi"]
published: true
published_at : "2025-03-20 20:00"
---

## 概要

[チュートリアル: ASP.NET Core を使って最小 API を作成する](https://learn.microsoft.com/ja-jp/aspnet/core/tutorials/min-web-api?view=aspnetcore-9.0&tabs=visual-studio-code) のF#版です。
重複する内容については、改変しています。

本ページで作成されるコードは以下に公開しています。
<https://github.com/Kuroki-g/kuroki-g-public-zenn-code/tree/main/articles/fsharp-min-web-api>

## 必須コンポーネント

[Visual Studio Code で F# を始める](https://learn.microsoft.com/ja-jp/dotnet/fsharp/get-started/get-started-vscode)を参照し、Visual Studio CodeでF#を書くための準備をしてください。

:::message
F# Null許容参照型は.NET 9以上で利用可能です。
<https://devblogs.microsoft.com/dotnet/nullable-reference-types-in-fsharp-9/#welcome-f#-nullable-reference-types-in-f#-9>
:::

## API プロジェクトを作成する

次のコマンドを実行します。

```bash
dotnet new sln -o TodoApi
code TodoApi
```

:::message
ソリューションファイルは元のドキュメントには無いのですが、おおよそのプロジェクトの場合には作ることになるため、作成してしまいます。
:::

次のディレクトリ構造が生成されます。

```bash
.
└── TodoApi
    └── TodoApi.sln
```

ターミナルで、次のコマンドを実行します。

```bash
dotnet new web -lang "F#" -o src/App
dotnet new gitignore -o src/App/
```

この時点で、次のディレクトリ構造が生成されます。

```bash
.
├── TodoApi.sln
└── src
    └── App
        ├── App.fsproj
        ├── Program.fs
        ├── Properties
        │   └── launchSettings.json
        ├── appsettings.Development.json
        └── appsettings.json
```

dotnet sln add コマンドを使用して、TodoApi ソリューションに App プロジェクトを追加します。

```bash
dotnet sln add src/App/App.fsproj 
```

この時点で一度ビルドを行い、プロジェクトが正しく設定されていることを確認します。
正しく設定されていれば、`src/App/bin`にビルドされたアプリケーションが生成されます。

```bash
dotnet build
```

### コードを確認する

`src/App/Program.fs` ファイルには、次のコードが含まれています。

```F#
open System
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.Hosting

[<EntryPoint>]
let main args =
    let builder = WebApplication.CreateBuilder(args)
    let app = builder.Build()

    app.MapGet("/", Func<string>(fun () -> "Hello World!")) |> ignore

    app.Run()

    0 // Exit code

```

上記のコードでは次の操作が行われます。

- 事前に構成された既定値で `WebApplicationBuilder` と `WebApplication` を作成します。
- / を返す HTTP GET エンドポイント Hello World! を作成します。

### Null許容参照型を設定する

F#のプロジェクトでは、null許容参照型はONになっていません。
C#との相互運用性を高めるために導入します。

```xml
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable> <!-- <= add this -->
  </PropertyGroup>
```

### アプリを実行する

次のコマンドを実行し、HTTPS 開発証明書を信頼します。

```bash
dotnet dev-certs https --trust
```

:::message
詳細は<https://learn.microsoft.com/ja-jp/aspnet/core/tutorials/min-web-api?view=aspnetcore-8.0&tabs=visual-studio-code#run-the-app>を参照してください。
:::

## NuGet パッケージを追加する

次のコマンドを実行します。

```bash
cd src/App/
dotnet add package Microsoft.EntityFrameworkCore.InMemory --version 9.0.3
dotnet add package Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore --version 9.0.3
```

:::message
バージョンをSDKに合わせてください。
:::

- [Microsoft.EntityFrameworkCore.InMemory](http://nuget.org/packages/Microsoft.EntityFrameworkCore.InMemory/9.0.3)
- [Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore](https://www.nuget.org/packages/Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore/9.0.3)

この時点でビルドを行います。

```bash
dotnet build
```

## モデルおよびデータベース コンテキスト クラス

`Models`フォルダーを作成し、次のコードを含む `Todo.fs` という名前のファイルを作成します。

```F#
namespace TodoApi.Models

type Todo() =
    member val Id: int = 0 with get, set
    member val Name: string | null = null with get, set
    member val IsComplete: bool = false with get, set

```

:::message
`option`型ではなく`string?`型を使用しています。
[How to handle F# Option type with Entity Framework Core?](https://stackoverflow.com/questions/63352175/how-to-handle-f-option-type-with-entity-framework-core)を見る限り、option型の対応にはEFCore.FSharpが必要なようで、このpackageはEF core 8以上に対応していません。
:::

`App.fsproj`を開き、コンパイルできるように設定を追加します。
F#ではコンパイラに渡されるファイルの順番が重要なため、`Program.fs`の上に追加します。

```xml
<ItemGroup>
    <Compile Include="Models/*.fs" /> <!-- <= add this -->
    <Compile Include="Program.fs" />
</ItemGroup>
```

:::message
コンパイラに渡されるファイルの順番については、以下を読んでください。
<https://stackoverflow.com/a/6823306>
:::

`Data`フォルダーを作成し、次のコードのファイルを、`TodoDb.fs` という名前で作成します。

```F#
namespace TodoApi.Data

open Microsoft.EntityFrameworkCore
open TodoApi.Models

type TodoDb(options: DbContextOptions<TodoDb>) =
    inherit DbContext(options)

    member _.Todos = base.Set<Todo>()
```

`Models`の設定の後に追加します。

```xml
<ItemGroup>
    <Compile Include="Models/*.fs" />
    <Compile Include="Data/*.fs" /> <!-- <= Add here! -->
    <Compile Include="Program.fs" />
</ItemGroup>
```

コンパイルの設定が正しいか確認するため、この時点でビルドを行います。

```bash
dotnet build
```

## API コードを追加する

### データベース接続の追加

`Program.fs`ファイルの内容を次のコードに置き換えます。

```F#
open System
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection // add here
open Microsoft.Extensions.Hosting
open Microsoft.EntityFrameworkCore // add here
open TodoApi.Data // add here

[<EntryPoint>]
let main args =
    let builder = WebApplication.CreateBuilder(args)
    
    // add codes from here...
    // AddDbContextのためにMicrosoft.Extensions.DependencyInjectionを追加する。
    builder.Services.AddDbContext<TodoDb>(fun opt -> opt.UseInMemoryDatabase("TodoList") |> ignore)
    |> ignore

    builder.Services.AddDatabaseDeveloperPageExceptionFilter() |> ignore

    // Next codes...
    // let app = builder.Build()
```

この時点で、インメモリデータベースへの接続設定が追加されます。

### APIの追加

APIを追加していきます。

#### GETの追加

GETリクエストを行うためのAPIのコードを追加します。

```F#
open System
open System.Linq // add here
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Http // add here
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open Microsoft.EntityFrameworkCore
open System.Threading.Tasks // add here
open TodoApi.Data

// ...
    let app = builder.Build()
    // add from here...
    app.MapGet("/todoitems", Func<TodoDb, Task<Collections.Generic.List<Todo>>>(fun db -> db.Todos.ToListAsync()))
    |> ignore

    app.MapGet(
        "/todoitems/complete",
        Func<TodoDb, Task<Collections.Generic.List<Todo>>>(fun db ->
            db.Todos.Where(fun t -> t.IsComplete).ToListAsync())
    )
    |> ignore

    app.MapGet(
        "/todoitems/{id}",
        Func<int, TodoDb, Task<IResult>>(fun id db ->
            task {
                match! db.Todos.FindAsync id with
                | null -> return Results.NotFound()
                | todo -> return Results.Ok todo
            })
    )
    |> ignore
    
    // next codes...
    app.Run()
```

F#の書き方については、以下を確認してください。

- [パターン マッチング](https://learn.microsoft.com/ja-jp/dotnet/fsharp/language-reference/pattern-matching)
- [.NET タスクを直接 F# で記述する](https://learn.microsoft.com/ja-jp/dotnet/fsharp/tutorials/async#writing-net-tasks-directly-in-f)
- [F# 6でtask式なるものが追加されていたので、F#と併せて紹介してみる](https://blog.nextscape.net/archives/2021/12/10/000728)

ここで、実際に動きを確認してみましょう。
開発環境用サーバーを起動してみます。

```bash
dotnet run
```

curlでリクエストをしてみましょう。
データは投入していませんので、空のデータ、もしくは404が返って来ます。

```bash
$ curl --fail http://localhost:5058/todoitems
[]
$ curl --fail http://localhost:5058/todoitems/complete
[]
$ curl --fail http://localhost:5058/todoitems/1
curl: (22) The requested URL returned error: 404
```

#### 残りのAPIの追加

続いて、残りのAPIも加えます。

```F#
    // ...
    // next to get api
    app.MapPost(
        "/todoitems",
        Func<Todo, TodoDb, Task<IResult>>(fun todo db ->
            task {
                db.Todos.Add todo |> ignore
                let! result = db.SaveChangesAsync()

                return Results.Created($"/todoitems/{todo.Id}", result)
            })
    )
    |> ignore

    app.MapPut(
        "/todoitems/{id}",
        Func<int, Todo, TodoDb, Task<IResult>>(fun id inputTodo db ->
            task {
                match! db.Todos.FindAsync id with
                | null -> return Results.NotFound()
                | todo ->
                    todo.Name <- inputTodo.Name
                    todo.IsComplete <- inputTodo.IsComplete
                    db.SaveChangesAsync() |> ignore
                    return Results.NoContent()
            })
    )
    |> ignore

    app.MapDelete(
        "/todoitems/{id}",
        Func<int, TodoDb, Task<IResult>>(fun id db ->
            task {
                match! db.Todos.FindAsync id with
                | null -> return Results.NotFound()
                | todo ->
                    db.Todos.Remove todo |> ignore
                    db.SaveChangesAsync() |> ignore
                    return Results.NoContent()
            })
    )
    |> ignore

    app.Run()
```

データを投入し、GETで見てみましょう。

```bash
$ curl --fail --request POST --header "Content-Type: application/json" --data '{ "Name":"Jone", "IsComplete":true }' http://localhost:5058/todoitems
1
$ curl --fail --request POST --header "Content-Type: application/json" --data '{ "Name":"Bob", "IsComplete":false }' http://localhost:5058/todoitems
1
$ curl -f http://localhost:5058/todoitems
[{"id":1,"name":"Jone","isComplete":true},{"id":2,"name":"Bob","isComplete":false}]
$ curl -f http://localhost:5058/todoitems/complete
[{"id":1,"name":"Jone","isComplete":true}]
```

データを投入したデータを編集してみましょう。

```bash
$ curl --fail --request PUT --header "Content-Ty
pe: application/json" --data '{ "IsComplete":false }' http://localhost:5058/todoitems/1
$ curl -f http://localhost:5058/todoitems/1
{"id":1,"name":null,"isComplete":false}
```

削除をすると、404となります。

```bash
$ curl -f --request DELETE http://localhost:5058
/todoitems/1
$ curl -f http://localhost:5058/todoitems/1
curl: (22) The requested URL returned error: 404
```

これで、C#の場合と同様に、CURD操作ができることが確かめられました。

### Swagger を使って API テスト UI を作成する

#### Swagger ツールをインストールする

以下のコマンドを実行します。

```bash
dotnet add package NSwag.AspNetCore --version 14.2.0
```

- [NSwag.AspNetCore](https://www.nuget.org/packages/NSwag.AspNetCore/14.2.0)

#### Swagger ミドルウェアを構成する

```F#
    builder.Services.AddDatabaseDeveloperPageExceptionFilter() |> ignore
    // ...
    // add codes

    builder.Services.AddEndpointsApiExplorer() |> ignore

    builder.Services.AddOpenApiDocument(fun config ->
        config.DocumentName <- "TodoAPI"
        config.Title <- "TodoAPI v1"
        config.Version <- "v1")
    |> ignore
 
    let app = builder.Build()

    if app.Environment.IsDevelopment() then
        app.UseOpenApi() |> ignore

        app.UseSwaggerUi(fun config ->
            config.DocumentTitle <- "TodoAPI"
            config.Path <- "/swagger"
            config.DocumentPath <- "/swagger/{documentName}/swagger.json"
            config.DocExpansion <- "list")
        |> ignore
```

### Swagger UIでデータの API をテストする

Swagger UIを見てみましょう。
先ほどはHTTPで確認しましたが、HTTPSで起動します。

```bash
dotnet run --launch-profile https
```

ブラウザで<<https://localhost>:port/swagger/index.html>にアクセスします。
portは`src/App/Properties/launchSettings.json`の中の`profiles.https.applicationUrl`から確認できます。

Swagger UIでの確認手順については、C#向けの記述を確認してください。

- [データの POST をテストする](https://learn.microsoft.com/ja-jp/aspnet/core/tutorials/min-web-api?view=aspnetcore-9.0&tabs=visual-studio-code#test-posting-data)
- [GET エンドポイントを検証する](https://learn.microsoft.com/ja-jp/aspnet/core/tutorials/min-web-api?view=aspnetcore-9.0&tabs=visual-studio-code#examine-the-get-endpoints)
- [PUT エンドポイントをテストする](https://learn.microsoft.com/ja-jp/aspnet/core/tutorials/min-web-api?view=aspnetcore-9.0&tabs=visual-studio-code#test-the-put-endpoint)
- [DELETE エンドポイントの検証とテスト](https://learn.microsoft.com/ja-jp/aspnet/core/tutorials/min-web-api?view=aspnetcore-9.0&tabs=visual-studio-code#examine-and-test-the-delete-endpoint)

## MapGroup API を使う

MapGroup メソッドを使うことで、共通のURLプレフィックスを整理することができます。
`todoitems`URLプレフィックスを共通化してみましょう。

```F#

    let todoItems = app.MapGroup "/todoitems"

    todoItems.MapGet("/", Func<TodoDb, Task<Collections.Generic.List<Todo>>>(fun db -> db.Todos.ToListAsync()))
    |> ignore

    // change rest of api
```

## Moduleを使う

Delegate部分をmoduleに切り出すことで、コード全体の見通しを良くすることができます。

```F#
module TodoItems =
    let getAllTodos =
        Func<TodoDb, Task<Collections.Generic.List<Todo>>>(fun db -> db.Todos.ToListAsync())

// ...
todoItems.MapGet("/", TodoItems.getAllTodos) |> ignore
```

## TypedResults API を使う

先ほど`module`へと切り出した関数に対して、`TypedResults`を使えるように変更してみます。

```F#
module TodoItems =
    let getAllTodos =
        Func<TodoDb, Task<IResult>>(fun db -> task { return db.Todos.ToListAsync() |> TypedResults.Ok :> IResult })

    let getCompleteTodos =
        Func<TodoDb, Task<IResult>>(fun db ->
            task { return db.Todos.Where(fun t -> t.IsComplete).ToListAsync() |> TypedResults.Ok :> IResult })

    let getTodo =
        Func<int, TodoDb, Task<IResult>>(fun id db ->
            task {
                match! db.Todos.FindAsync id with
                | null -> return TypedResults.NotFound() :> IResult
                | todo -> return TypedResults.Ok todo :> IResult
            })

    let createTodo =
        Func<Todo, TodoDb, Task<IResult>>(fun todo db ->
            task {
                db.Todos.Add todo |> ignore
                let! result = db.SaveChangesAsync()

                return TypedResults.Created($"/todoitems/{todo.Id}", result) :> IResult
            })

    let updateTodo =
        Func<int, Todo, TodoDb, Task<IResult>>(fun id inputTodo db ->
            task {
                match! db.Todos.FindAsync id with
                | null -> return TypedResults.NotFound() :> IResult
                | todo ->
                    todo.Name <- inputTodo.Name
                    todo.IsComplete <- inputTodo.IsComplete
                    db.SaveChangesAsync() |> ignore
                    return TypedResults.NoContent() :> IResult
            })

    let deleteTodo =
        Func<int, TodoDb, Task<IResult>>(fun id db ->
            task {
                match! db.Todos.FindAsync id with
                | null -> return TypedResults.NotFound() :> IResult
                | todo ->
                    db.Todos.Remove todo |> ignore
                    db.SaveChangesAsync() |> ignore
                    return TypedResults.NoContent() :> IResult
            })
```

ここで用いられているキャスト（`:>`）に関しては、[アップキャスト](https://learn.microsoft.com/ja-jp/dotnet/fsharp/language-reference/casting-and-conversions#upcasting)を参照してください。

## 過剰な投稿を防止する

DTOを導入します。
詳細は[過剰な投稿を防止する](https://learn.microsoft.com/ja-jp/aspnet/core/tutorials/min-web-api?view=aspnetcore-9.0&tabs=visual-studio-code#prevent-over-posting)を参照してください。

```F#
namespace TodoApi.Dtos

open TodoApi.Models

type TodoItemDTO =
    { Id: int
      Name: string | null
      IsComplete: bool }

    static member Create(v: Todo) =
        { Id = v.Id
          Name = v.Name
          IsComplete = v.IsComplete }

```

モジュールを以下のように変更します。

```F#
module TodoItems =
    let getAllTodos =
        Func<TodoDb, Task<IResult>>(fun db ->
            task { return db.Todos.Select(TodoItemDTO.Create).ToListAsync() |> TypedResults.Ok :> IResult })

    let getCompleteTodos =
        Func<TodoDb, Task<IResult>>(fun db ->
            task {
                return
                    db.Todos.Where(fun t -> t.IsComplete).Select(TodoItemDTO.Create).ToListAsync()
                    |> TypedResults.Ok
                    :> IResult
            })

    let getTodo =
        Func<int, TodoDb, Task<IResult>>(fun id db ->
            task {
                match! db.Todos.FindAsync id with
                | null -> return TypedResults.NotFound() :> IResult
                | todo -> return todo |> TodoItemDTO.Create |> TypedResults.Ok :> IResult
            })

    let createTodo =
        Func<TodoItemDTO, TodoDb, Task<IResult>>(fun todoItemDTO db ->
            task {
                let todoItem = new Todo()
                todoItem.Name <- todoItemDTO.Name
                todoItem.IsComplete <- todoItemDTO.IsComplete
                db.Todos.Add todoItem |> ignore
                let! result = db.SaveChangesAsync()

                return TypedResults.Created($"/todoitems/{todoItem.Id}", result) :> IResult
            })

    let updateTodo =
        Func<int, TodoItemDTO, TodoDb, Task<IResult>>(fun id todoItemDTO db ->
            task {
                match! db.Todos.FindAsync id with
                | null -> return TypedResults.NotFound() :> IResult
                | todo ->
                    todo.Name <- todoItemDTO.Name
                    todo.IsComplete <- todoItemDTO.IsComplete
                    db.SaveChangesAsync() |> ignore
                    return TypedResults.NoContent() :> IResult
            })

    let deleteTodo =
        Func<int, TodoDb, Task<IResult>>(fun id db ->
            task {
                match! db.Todos.FindAsync id with
                | null -> return TypedResults.NotFound() :> IResult
                | todo ->
                    db.Todos.Remove todo |> ignore
                    db.SaveChangesAsync() |> ignore
                    return TypedResults.NoContent() :> IResult
            })

```

## 感想・気になることろ

### 感想

WEB APIに、C#のコードに沿って書くやり方では、C#でstatic classで定義しているのとあまり変わらない、という印象が強いです。
C#と相互運用するというのは前提条件であるので、letバインド、Option型、Result型、`|>`、パターンマッチにどれほどの価値を見出すかということになります。
OPPするというC#の書き方を頭の端っこに残しつつ、F#での書き方を強く意識する必要があるでしょう。

### 本質的ではないところで気になったところ

#### エディタについて

- 拡張メソッドでの`open`の補完が効かない。`System.Linq`も手動でインポートが必要。C#での名前空間何だっけな…と調べて手動で記述ということで非効率です。
- 暗黙的なアップキャストが貧弱に感じます。`IResult`に対して入れないとコンパイルが通らないというのは煩雑です。
- C#資産を使えないと片手落ちにもかかわらず、null許容参照型がまだまだ弱いです。
  - ただ、.NET 10 LTSでもnull許容参照型は維持されれば、導入は広まるでしょう。
  - null 安全はデフォルトでオンにしておいて欲しいところです。
- 公式のドキュメントが少なすぎる。
  - Microsoftのドキュメントで、明らかにF#のドキュメントの量が少なく、新しく学ぶ人へのサポートが貧弱です。
  - そもそも、F#単体で完結させるやり方をするのが基本なのかもしれません。

#### コミュニティの状況

ランキング[^1][^2]でも、関数型プログラミング言語の使用率はかなり低いものになっています。
会社が Scala[^3][^5]、Elixir[^4]、Clojure[^5]を使っているというものでもない限り使うことはないでしょう。F#もひっそり使っている方がいらっしゃるよう[^6]です。
コミュニティ層の薄さというのは、行き詰ったときの情報源の少なさに直結します。

新しく学ぶ人へのサポートが貧弱です。Why F#?というのをもっと強く推さないと、使用人口は増えないように思えます。Wikipediaにも利用実績書いてないし[^8]。

- Erlangだと分散処理だよね、といったものがないです。
- Microsoftの機械学習ライブラリSynapseMLはScalaで開発されており、「Many of us feel that scala is the queen of the languages so we almost exclusively code in scala.」と述べています[^7]

これくらい言えるとおっ？となりますね。

#### 利用事例

実際の利用についてですが、Deep Researchに探してもらい見つけてもらいました。

- Goldman Sachsで使われているという記述[^9]：<https://medium.com/@robertdennyson/where-f-outshines-other-languages-a-deep-dive-with-use-cases-00f8ab187fc9>
- Jet.com[^10]
- Credit Suisse quants were the first users of a new programming language[^11]
- <https://fsharp.org/testimonials/>
- <https://theirstack.com/en/technology/f-sharp>
- <https://github.com/fsprojects/fsharp-companies>
  - <https://sessionize.com/eric-harding/>

もう少し推せばいいのに…と思いますね。

## Links

- [ASP.NET CoreにおけるWeb APIの最新情報](https://speakerdeck.com/tomokusaba/asp-net-coreniokeruwebapinozui-xin-qing-bao)
- [F# で Entity Framework Coreる](https://pocketberserker.hatenablog.com/entry/2017/12/06/000405)
- [ASP.NET Core の C# を F# に書き換える](https://qiita.com/narimasu/items/85c0b1599f76f4e804bb)
- [F#でのOOP](https://qiita.com/aoi_erimiya/items/62ac073ecae710b55ca3)

## 参考文献

[^1]: [The Top Programming Languages 2024](https://spectrum.ieee.org/top-programming-languages-2024)
[^2]: [TIOBE Index for March 2025](https://www.tiobe.com/tiobe-index/https://www.tiobe.com/tiobe-index/)
[^3]: [Scala先駆者インタビュー VOL.7 水島さん（株式会社ドワンゴ／一般社団法人Japan Scala Association）](https://www.atware.co.jp/blog/2016/9/6/scala-vol7-kmizu)
[^4]: [モダン Erlang/OTP](https://zenn.dev/shiguredo/articles/modern-erlang)
[^5]: [JavaからScala、そしてClojureへ: 実務で活きる関数型プログラミング](https://www.docswell.com/s/lagenorhynque/5Q74YZ-from-java-through-scala-to-clojure)
[^6]: [仕事と F# と私](https://qiita.com/u_1roh/items/eb90672fc40fc7f2ea86)
[^7]: [Microsoft Announces General Availability of SynapseML](https://www.reddit.com/r/scala/comments/qwtzqk/comment/hl9xj28/?utm_source=share&utm_medium=web2x&context=3%3AMicrosoft)
[^8]: [F Sharp (programming language)](https://en.wikipedia.org/wiki/F_Sharp_(programming_language))
[^9]: [Where F# Outshines Other Languages: A Deep Dive with Use Cases](https://medium.com/@robertdennyson/where-f-outshines-other-languages-a-deep-dive-with-use-cases-00f8ab187fc9)
[^10]: [Case Study: Writing Microservices with F#](https://www.codemag.com/article/1611071/Case-Study-Writing-Microservices-with-F)
[^11]: [Credit Suisse quants were the first users of a new programming language](https://www.efinancialcareers.com/news/2020/12/credit-suisse-quant-jobs)

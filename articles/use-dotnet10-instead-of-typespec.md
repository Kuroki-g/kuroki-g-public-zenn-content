---
title: "API仕様書 (OpenAPI) の管理に.NET 10を用いる"
emoji: "🦔"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["typespec", "openapi", "dotnet", "swagger"]
published: false
---

## はじめに

OpenAPI / Swagger使ってますか？
最近はAPI仕様書として、OpenAPIを用いて管理するようになりつつあります。
一方で、yaml / jsonを用いて記述するため、以下の問題点もあります。

- そもそもyaml / jsonを直接書くのがつらい
- 長くて読みにくい
- 生成AIに気軽に食わせられない

また、gRPCの表現ができない、という弱点もあります。
REST、OpenAPI、gRPCなどのプロトコルで共通するAPI形状を記述するために開発されたのが、[TypeSpec](https://typespec.io/)です。

## そもそもTypeSpecとは？

TypeSpec (<https://typespec.io/>, <https://github.com/microsoft/typespec/>)とは、MS謹製のAPIを定義するための言語です。
複数のプロトコルを利用しているケースや、大規模なAPI仕様書の管理に使えるのではないかと期待されます。個人的には、[4万行超のopenapi.yamlをTypeSpecに移行した話](https://zenn.dev/yuta_takahashi/articles/migrate-to-typespec)が印象に残っています。

しかし、まだまだ事例が少ないようで、2025/11/16時点で、Zennでトピックで見つかるのはOpenAPI 423件に対しTypeSpecは11件にすぎません。

メジャーバージョンになったのが2025/05/07 ([typespec-stable@1.0.0](https://github.com/microsoft/typespec/releases/tag/typespec-stable%401.0.0))と最近で、周辺ライブラリも0系と発展途上です。
TypeSpecのジェネレーターは軒並み0系のため、現時点では、例えば[openapi-generator-cli](https://github.com/OpenAPITools/openapi-generator-cli)のような別のツールを用いることになるでしょう。

最近は、生成AIにドキュメントを渡すというのが重要であることからも、なるべく文字数が少なく、かつ高品質な情報を渡せるというのは重要かと思います。

執筆時点でスター数5.5kであることから、非常に期待がなされていると言ってよいかと思います。

## それ、.NET 10で実現できませんか？

一方で、ほとんどの案件では、複数のプロトコルを同時に使うことや、数万行のOpenAPI仕様書を見る機会は少ないでしょう。
どちらかと言えば、保守性・生成の品質・柔軟性などの方が重要なはずです。

そこで、.NET 10での管理を提案します。
.NET 10では、非常に簡単にOpenAPIの生成が可能になっています。

## TypeSpecを.NET 10に書き換えてみた

サンプルとして、[クイック スタート: TypeSpec と .NET を使用して新しい API プロジェクトを作成する](https://learn.microsoft.com/ja-jp/azure/developer/typespec/quickstart-scaffold-dotnet#developing-with-typespec) での自動生成されるコードを丸っと.NET 10に置き換えてみます。

例えば、この`model`の表現では以下のようになります。
`[Range]`属性を用いてのバリデーションも可能で、表現力としては十分なものがあります。

C#では文字列リテラルのユニオン型が使えないため、`enum`で明示的に定義するのが明確な差異となります。

```typescript
model Widget {
  id: string;
  weight: int32;
  color: "red" | "blue";
}
```

```csharp
public record Widget(
 string Id,
 int Weight,
 WidgetColor Color
);

[JsonConverter(typeof(JsonStringEnumConverter<WidgetColor>))]
public enum WidgetColor
{
    Red,
    Blue
}
```

APIエンドポイントではこうです。.NET側では、数行追記が必要です。
C#側の、4xx系のエラーの追記、APIエンドポイントや、パスパラメーターへの`description`の記述も可能ですが、今回は省いています。

```typescript
@route("/widgets")
@tag("Widgets")
interface Widgets {
  @get list(): WidgetList | Error;
}
```

```csharp
[ApiController]
[Route("/widgets")]
[Tags("Widgets")]
[Produces("application/json")]
public class WidgetsController : Controller
{
    [HttpGet]
    public WidgetList List()
    {
        throw new NotImplementedException();
    }
}
```

左：TypeSpecの生成、右：.NET 10の生成です。
（注意：.NET 10は、現時点ではyaml形式の直接のCLI生成をサポートしていないので、手動変換している）

![Modelの場合](/images/use-dotnet10-instead-of-typespec/yaml-1.png)

![Pathの場合](/images/use-dotnet10-instead-of-typespec/yaml-2.png)

## .NET 10である理由

OpenAPIの生成であれば、最近のフレームワークで対応しているものが多数あります。

- Golang × swag <https://github.com/swaggo/swag>
- Laravel × Scribe <https://scribe.knuckles.wtf/laravel/>
- Hono × hono-openapi <https://hono.dev/examples/hono-openapi>

しかし、.NET10ならではのメリットがあります。

- フレームワーク自体に内蔵
- C#の厳密な型定義
- LTS採用による後方互換性・保守性

での生成をするのは難しいと考えています。

※ .NET 8 LTSではSwashbuckle等採用のため、all-in-oneではなかった  
※ .NET9から、`Microsoft.AspNetCore.OpenApi`と`Microsoft.Extensions.ApiDescription.Server`はあるが、LTSではない

## 備考

- .NET 10におけるOpenAPIのyamlでの生成は<https://github.com/dotnet/aspnetcore/issues/61041>で対応中のようですので、そのうちサポートされるかと思われます。

## サンプルコード

- <https://github.com/Kuroki-g/kuroki-g-public-zenn-code/tree/main/articles/use-dotnet10-instead-of-typespec>

## 参考文献

- [.NET9における異なるOpenAPI](https://qiita.com/axzxs2001/items/6487dc101fa03e3fd4ba)
- [HonoでAPIのドキュメントを自動生成する](https://zenn.dev/praha/articles/d1d6462a27e37e)
- [25000行超えのAPIドキュメントを分割した話](https://zenn.dev/counterworks/articles/ce4f359947fcfe)
- [4万行超のopenapi.yamlをTypeSpecに移行した話](https://zenn.dev/yuta_takahashi/articles/migrate-to-typespec)
- [OpenAPI ドキュメントを生成する](https://learn.microsoft.com/ja-jp/aspnet/core/fundamentals/openapi/aspnetcore-openapi?view=aspnetcore-10.0&tabs=visual-studio%2Cvisual-studio-code)
- [ビルド時に OpenAPI ドキュメントを生成する](https://learn.microsoft.com/ja-jp/aspnet/core/fundamentals/openapi/aspnetcore-openapi?view=aspnetcore-10.0&tabs=net-cli%2Cnetcore-cli#generate-openapi-documents-at-build-time)
- [クイック スタート: TypeSpec と .NET を使用して新しい API プロジェクトを作成する](https://learn.microsoft.com/ja-jp/azure/developer/typespec/quickstart-scaffold-dotnet#developing-with-typespec)

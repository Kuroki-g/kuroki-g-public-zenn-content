---
title: "OpenAPI生成で.NET 10を用いてみる"
emoji: "🦔"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["typespec", "openapi", "dotnet"]
published: false
---

## はじめに

OpenAPI使ってますか？
最近はOpenAPI仕様書を用いたAPIの管理が非常に盛んになってきている気がします。
一方で、yaml / jsonを用いて記述するため、以下の問題点もあります。

- そもそもyamlを直接書くのがつらい
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
どちらかと言えば、果たして維持されるのか？品質はどうなのか？柔軟性は？などの方が重要なはずです。

そこで、.NETはどうだろう、というのを考えてみます。
.NET 10になり（厳密にはSTSの.NET 9の時点での強化）、OpenAPIの機能が大幅に強化されました。

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
APIエンドポイントや、パスパラメーターへの`description`の記述も可能ですが、今回は省いています。

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

NOTE: .NET 10におけるOpenAPIのyamlでの生成は<https://github.com/dotnet/aspnetcore/issues/61041>で対応中のようですので、そのうちサポートされるかと思われます。

## サンプルコード

- <https://github.com/Kuroki-g/kuroki-g-public-zenn-code/tree/main/articles/use-dotnet10-instead-of-typespec>

## 参考文献

- [4万行超のopenapi.yamlをTypeSpecに移行した話](https://zenn.dev/yuta_takahashi/articles/migrate-to-typespec)
- [ビルド時に OpenAPI ドキュメントを生成する](https://learn.microsoft.com/ja-jp/aspnet/core/fundamentals/openapi/aspnetcore-openapi?view=aspnetcore-10.0&tabs=net-cli%2Cnetcore-cli#generate-openapi-documents-at-build-time)
- [クイック スタート: TypeSpec と .NET を使用して新しい API プロジェクトを作成する](https://learn.microsoft.com/ja-jp/azure/developer/typespec/quickstart-scaffold-dotnet#developing-with-typespec)

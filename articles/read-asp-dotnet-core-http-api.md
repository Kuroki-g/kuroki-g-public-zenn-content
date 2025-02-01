---
title: "ASP.NET CoreのHTTPリクエストを読む"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["http", "csharp", "aspdotnetcore"]
published: false
---

## 概要

WEB API が実際のリクエストからレスポンスまでどのように返しているかを読み直します。

## サンプルの作成

### 検証環境

- Windows11 WSL (22.04.5 LTS (Jammy Jellyfish))
- .NET 8.0.112
- curl

### サンプルプロジェクトの作成

公式ドキュメント通りにWEB API を作成します。
SEE: [チュートリアル: ASP.NET Core を使って最小 API を作成する](https://learn.microsoft.com/ja-jp/aspnet/core/tutorials/min-web-api?view=aspnetcore-8.0&tabs=visual-studio-code)

```bash
dotnet new web -o TodoApi.Web
```

:::message
`dotnet new`には`webapi`テンプレートもありますが、今回はシンプルにするために`web`テンプレートを使用します。
:::

### APIの統合テストを行うプロジェクトの作成

デバッグで確認できるように統合テストをするためのテストプロジェクトも作成します。
手順は [ASP.NET Core での統合テスト](https://learn.microsoft.com/ja-jp/aspnet/core/test/integration-tests?view=aspnetcore-8.0)に従います。

#### テストプロジェクトの作成

```bash
dotnet new xunit3 --name TodoApi.Web.ApiTest.Tests
cd TodoApi.Web.ApiTest.Tests
# add package for testing. change version if needed.
dotnet add package Microsoft.AspNetCore.Mvc.Testing --version 8.0.12

# add reference to the project
dotnet add reference ../TodoApi.Web
Reference `..\TodoApi.Web\TodoApi.Web.csproj` added to the project.
```

Program.csに以下のコードを追加します。

```csharp
app.Run();

// この行を追加
public partial class Program { }
```

#### テストの作成

`UnitTest1.cs`を以下のように書き換えます。

```csharp
using Microsoft.AspNetCore.Mvc.Testing;

namespace TodoApi.Web.ApiTest.Tests;

public class BasicTests(WebApplicationFactory<Program> factory)
        : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory = factory;

    [Theory]
    [InlineData("/")]
    public async Task Get_EndpointsReturnSuccessAndCorrectContentType(string url)
    {
        // Arrange
        var client = _factory.CreateClient();

        // Act
        var response = await client.GetAsync(url);

        // Assert
        response.EnsureSuccessStatusCode(); // Status Code 200-299

        var contentType = response.Content.Headers.ContentType?.ToString();
        Assert.Equal("text/plain; charset=utf-8", contentType);
    }
}
```

この状態でテストが通ることを確認します。

```bash
dotnet test
```

VSCodeでのデバッガーを用いることで、きちんとAPIのレスポンスの部分にブレークポイントを設定してデバッグすることができます。

<!-- add image. path is articles/read-asp-dotnet-core-http-api/image.png -->
![API response debugging in VSCode](/images/read-asp-dotnet-core-http-api/image.png)

###

開発環境用HTTPサーバーを起動します。

```bash
dotnet watch run
```
---
title: ".NET 10 (C# 14) の新機能をまとめる"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## 概要

### 単一ファイルの実行

単一ファイルの実行が可能になりました。

```cshap
Console.WriteLine("Hello, world!");
```

```bash
$ dotnet run hello.cs
Hello, world!
```

- [Announcing dotnet run app.cs – A simpler way to start with C# and .NET 10] <https://devblogs.microsoft.com/dotnet/announcing-dotnet-run-app/>

### C# 14 の新機能

以下の内容をコードに起こしました。一部は省略しています。

- [.NET 10の新機能] <https://learn.microsoft.com/ja-jp/dotnet/core/whats-new/dotnet-10/overview>
- [C# 14 の新機能] <https://learn.microsoft.com/ja-jp/dotnet/csharp/whats-new/csharp-14>

#### 拡張メンバー

```csharp
namespace Dotnet10Feature;

public partial class MorePartialMembers
{
    // partial classが非常に長くなる場合に、このようにメソッドのみの宣言を事前にしておくことが可能です。
    // NOTE: staticは不可です。
    partial void PartialMethod(string s);

    // partial classで部分コンストラクターの宣言が可能になりました。
    // https://learn.microsoft.com/ja-jp/dotnet/csharp/programming-guide/classes-and-structs/constructors#partial-constructors
    public partial MorePartialMembers();
}

public partial class MorePartialMembers
{
    partial void PartialMethod(string s) => Console.WriteLine($"Something happened: {s}");

    public partial MorePartialMembers() // base()又はthis()の使用をする場合には、こちらに追加する必要があります。
    {
        // ここに実装宣言に追加することができます。   
    }
}
```

- <https://github.com/Kuroki-g/kuroki-g-public-zenn-code/blob/main/articles/dotnet10-feature/MorePartialMembers.cs>

#### Null 条件付き割り当て

```csharp
namespace Dotnet10Feature;

/// <summary>
/// Null 条件付き割り当て
/// </summary>
/// <see href="https://learn.microsoft.com/ja-jp/dotnet/csharp/whats-new/csharp-14#null-conditional-assignment"/>
class NullConditionalAssignment
{
    void NullAssign(Customer? customer)
    {
        // Null check can be simplified (IDE0031) が出るようになりました。
        if (customer is not null)
        {
            customer.Order = GetCurrentOrder();
        }

        customer?.Order = GetCurrentOrder();
    }

    class Customer
    {
        public string? Order { get; set; } = null;
    }

    static string GetCurrentOrder()
    {
        return "CurrentOrder";
    }
}
```

- <https://github.com/Kuroki-g/kuroki-g-public-zenn-code/blob/main/articles/dotnet10-feature/NullConditionalAssignment.cs>

#### nameof は、バインドされていないジェネリック型をサポートします

```csharp
namespace Dotnet10Feature;

/// <summary>
/// バインドされていないジェネリック型と nameof
/// </summary>
/// <see href="https://learn.microsoft.com/ja-jp/dotnet/csharp/whats-new/csharp-14#unbound-generic-types-and-nameof"/> 
public static class UnboundGenericTypesAndNameof
{
    public static void ShowExample()
    {
        // C# 14 以降では、 nameof する引数はバインドされていないジェネリック型にすることができます。
        var nameofList = nameof(List<>);
        // 以前のバージョンでは閉じたジェネリック型のみが使用可能でした。
        // NOTE: Use unbound generic type (IDE0340) の警告が出ます。
        var nameofListInt = nameof(List<int>);
    }
}
```

- <https://github.com/Kuroki-g/kuroki-g-public-zenn-code/blob/main/articles/dotnet10-feature/UnboundGenericTypesAndNameof.cs>


#### 単純なラムダ パラメーターの修飾子

#### field でサポートされるプロパティ

```csharp
/// <summary>
/// field キーワード
/// </summary>
/// <see href="https://learn.microsoft.com/ja-jp/dotnet/csharp/whats-new/csharp-14#the-field-keyword"/>
class FieldFeature
{
    #region 
    // fieldキーワードを使用した新しい形式です。
    public string NewFormatProperty
    {
        get;
        // fieldキーワードを使うと簡略化することができます。
        // WARNING: `field`という名前のシンボルを含む型のコードがある場合には、ワークアラウンドが必要です。
        // NOTE: null許容参照の警告が出ます。
        set => field = value ?? throw new ArgumentNullException(nameof(value));
    }

    // 古い形式のプロパティ。VSCodeは、infoレベルでの警告となります。
    // 自動的に新しい形式に変更することが可能です。
    // NOTE: null許容参照の警告が出ます。
    private string _msg; // この形式の場合には、バッキングフィールドの宣言が必要です。

    public string OldFormatProperty
    {
        get => _msg;
        set => _msg = value ?? throw new ArgumentNullException(nameof(value));
    }
    #endregion
}
```

- <https://github.com/Kuroki-g/kuroki-g-public-zenn-code/blob/main/articles/dotnet10-feature/FieldFeature.cs>

#### partial イベントとコンストラクター

```csharp
namespace Dotnet10Feature;

public partial class MorePartialMembers
{
    // partial classが非常に長くなる場合に、このようにメソッドのみの宣言を事前にしておくことが可能です。
    // NOTE: staticは不可です。
    partial void PartialMethod(string s);

    // partial classで部分コンストラクターの宣言が可能になりました。
    // https://learn.microsoft.com/ja-jp/dotnet/csharp/programming-guide/classes-and-structs/constructors#partial-constructors
    public partial MorePartialMembers();
}

public partial class MorePartialMembers
{
    partial void PartialMethod(string s) => Console.WriteLine($"Something happened: {s}");

    public partial MorePartialMembers() // base()又はthis()の使用をする場合には、こちらに追加する必要があります。
    {
        // ここに実装宣言に追加することができます。   
    }
}
```

- <https://github.com/Kuroki-g/kuroki-g-public-zenn-code/blob/main/articles/dotnet10-feature/MorePartialMembers.cs>

#### ユーザー定義複合代入演算子

```csharp
namespace Dotnet10Feature;

/// <summary>
/// ユーザー定義複合割り当て
/// </summary>
/// <see href="https://learn.microsoft.com/ja-jp/dotnet/csharp/language-reference/operators/operator-overloading"/>
class UserDefinedCompoundAssignment
{
    class C1
    {
        public int Value;

        public static C1 operator +(C1 operand) => operand;

        public void operator +=(int x)
        {
            Value += x;
        }
    }
}
```

- <https://github.com/Kuroki-g/kuroki-g-public-zenn-code/blob/main/articles/dotnet10-feature/UserDefinedCompoundAssignment.cs>

#### その他

- `Span<T>` および `ReadOnlySpan<T>` のより暗黙的な変換

### EF Core 10

EF core 9 / 10の両方の破壊的変更の確認が必要そうです。

- [EF Core 9 (EF9) での破壊的変更] <https://learn.microsoft.com/ja-jp/ef/core/what-is-new/ef-core-9.0/breaking-changes>
- [EF Core 10 の破壊的変更 (EF10)] <https://learn.microsoft.com/ja-jp/ef/core/what-is-new/ef-core-10.0/breaking-changes>

## 参考リンク

- [C# 14 / .NET 10 の新機能 (RC 1 時点)] <https://speakerdeck.com/nenonaninu/net-10-noxin-ji-neng-rc-1-shi-dian>
  - RC 1時点の、何縫ねの。氏による解説です。

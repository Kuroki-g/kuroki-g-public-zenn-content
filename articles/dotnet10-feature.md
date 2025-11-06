---
title: ".NET 10 (C# 14) の新機能をまとめる"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "dotnet"]
published: true
published_at : "2025-11-06 09:00"
---

## 概要

.NET 10 (RC2) 時点の内容です。
[.NET 10 の新機能] <https://learn.microsoft.com/ja-jp/dotnet/core/whats-new/dotnet-10/overview> から、抜粋します。詳細はリンク先を参照してください。
本記事のコードは <https://github.com/Kuroki-g/kuroki-g-public-zenn-code/tree/main/articles/dotnet10-feature> にまとめてあります。

### .NET ライブラリ

以下の内容をコードに起こしました。一部は省略しています。

- [.NET 10 用 .NET ライブラリの新機能] <https://learn.microsoft.com/ja-jp/dotnet/core/whats-new/dotnet-10/libraries>

#### グローバリゼーションと日付/時刻

##### 文字列比較の数値の順序付け

```csharp
StringComparer numericStringComparer = StringComparer.Create(
    System.Globalization.CultureInfo.CurrentCulture,
    System.Globalization.CompareOptions.NumericOrdering // NumericOrderingオプションが追加されました。
);

Console.WriteLine(numericStringComparer.Equals("02", "2")); // これはTrueと判定されます。

var sorted = new[] { "Windows XP", "Windows 10", "Windows 8", "Windows 11" }.Order(numericStringComparer);
// 10、11の数字が考慮され、以下の順番で出力されます:
// Windows 8
// Windows 10
// Windows 11
// Windows XP
foreach (string os in sorted)
{
    Console.WriteLine(os);
}

HashSet<string> set = new(numericStringComparer) { "007", "07", "7" };
Console.WriteLine(string.Join(",", set.ToArray())); // Output: 007 <- 順番が考慮されます。
Console.WriteLine(set.Contains("7")); // Output: True

HashSet<string> set2 = new(numericStringComparer) { "7", "007", "07",  };
Console.WriteLine(string.Join(",", set2.ToArray())); // Output: 7 <- 順番が考慮されます。
Console.WriteLine(set2.Contains("007")); // Output: True

```

#### シリアル化

##### JSON プロパティの重複を禁止するオプション

```csharp
record MyRecord(int Value);

string json = """{ "Value": 1, "Value": -1 }""";
Console.WriteLine(JsonSerializer.Deserialize<MyRecord>(json)?.Value); // -1

// AllowDuplicatePropertiesを指定すると、例外となります。
JsonSerializerOptions options = new() { AllowDuplicateProperties = false };
try
{
    // JsonObject、Dictionary<string, int>でも同様にJsonExceptionとなります。
    JsonSerializer.Deserialize<MyRecord>(json, options);
}
catch (JsonException ex)
{
    Console.WriteLine(ex.Message);
}

// JsonDocumentでも同様に、AllowDuplicatePropertiesの設定が可能です。
JsonDocumentOptions docOptions = new() { AllowDuplicateProperties = false };
try
{
    JsonDocument.Parse(json, docOptions);
}
catch (JsonException ex)
{
    Console.WriteLine(ex.Message);
}
```

##### 厳密な JSON シリアル化オプション

```csharp
string json = """{ "Value": 1, "Value": -1 }""";
JsonSerializer.Deserialize<MyRecord>(json, JsonSerializerOptions.Strict);
// JsonSerializerOptions.Strict プリセットが追加されました。
// これは、以下のものに等しいです。
// JsonUnmappedMemberHandling.Disallow
// + JsonSerializerOptions.AllowDuplicateProperties = false
// + case sensitive (大文字と小文字の区別)
// + JsonSerializerOptions.RespectNullableAnnotations
// + JsonSerializerOptions.RespectRequiredConstructorParameters
// <https://github.com/dotnet/dotnet/blob/89c8f6a112d37d2ea8b77821e56d170a1bccdc5a/src/runtime/src/libraries/System.Text.Json/src/System/Text/Json/Serialization/JsonSerializerOptions.cs#L180>
```

#### その他

他、複数の追加があります。

- System.Numerics
- オプションの検証
- 診断
- ZIP ファイル
- Windows プロセス管理
- WebSocket の機能強化
- TLS の機能強化

詳細は、[.NET 10 用 .NET ライブラリの新機能] <https://learn.microsoft.com/ja-jp/dotnet/core/whats-new/dotnet-10/libraries> を参照してください。

### C# 14 の新機能

以下の内容をコードに起こしました。一部は省略しています。

- [.NET 10の新機能] <https://learn.microsoft.com/ja-jp/dotnet/core/whats-new/dotnet-10/overview>
- [C# 14 の新機能] <https://learn.microsoft.com/ja-jp/dotnet/csharp/whats-new/csharp-14>

#### 拡張メンバー

```csharp
namespace Dotnet10Feature.Extensions;

/// <summary>
/// 拡張メンバー
/// C# 14 では、 拡張メンバーを定義するための新しい構文が追加されています。 
/// </summary>
/// <see href="https://learn.microsoft.com/ja-jp/dotnet/csharp/whats-new/csharp-14#extension-members"/> 
public static class Enumerable
{
    // 新しく、extensionブロックを用いて拡張メンバーを宣言することができるようになりました。
    // Extension block
    // https://learn.microsoft.com/ja-jp/dotnet/csharp/programming-guide/classes-and-structs/extension-methods#declare-extension-members
    extension<TSource>(IEnumerable<TSource> source) // extension members for IEnumerable<TSource>
    {
        // 新しく拡張プロパティの宣言ができるようになりました:
        public bool IsEmpty => !source.Any();

        // 拡張メソッドの宣言をextensionブロックに書くことができます:
        public IEnumerable<TSource> Where(Func<TSource, bool> predicate)
        {
            throw new NotImplementedException();
        }
    }

    // C# 14より前の場合、拡張プロパティはないのでメソッドで宣言する必要があります。
    public static bool BeforeC14IsEmpty<TSource>(this IEnumerable<TSource> source)
        => !source.Any();

    // 既存のC# 14より前のバージョンの拡張メソッドの宣言はこれまで通りです。
    public static IEnumerable<TSource> BeforeC14Where<TSource>(this IEnumerable<TSource> source, Func<TSource, bool> predicate)
    {
        throw new NotImplementedException();
    }

    // 静的メンバー + operatorのみの場合には、レシーバー型のみの、extensionブロックで表現することもできる。
    extension<TSource>(IEnumerable<TSource>)
    {
        // static拡張メソッド:
        public static IEnumerable<TSource> Combine(IEnumerable<TSource> first, IEnumerable<TSource> second)
        {
            throw new NotImplementedException();
        }

        // static拡張プロパティが定義可能です:
        public static IEnumerable<TSource> Identity => System.Linq.Enumerable.Empty<TSource>();

        // ユーザー定義のoperatorの定義ができるようになりました:
        // NOTE: [Extensions (拡張型) 未確認飛行 C]<https://ufcpp.net/blog/2024/3/extensions/> を見るに、しばらく前から要望があったようです。
        public static IEnumerable<TSource> operator +(IEnumerable<TSource> left, IEnumerable<TSource> right) => left.Concat(right);
    }
}
```

- <https://github.com/Kuroki-g/kuroki-g-public-zenn-code/blob/main/articles/dotnet10-feature/Extensions/Enumerable.cs>

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

### .NET SDK

#### 単一ファイルの実行

単一ファイルの実行が可能になりました。

```cshap
Console.WriteLine("Hello, world!");
```

```bash
$ dotnet run hello.cs
Hello, world!
```

- [Announcing dotnet run app.cs – A simpler way to start with C# and .NET 10] <https://devblogs.microsoft.com/dotnet/announcing-dotnet-run-app/>

### .NET ランタイム

以下の改善・機能強化がなされたとのことです。
詳細は[.NET 10 ランタイムの新機能] <https://learn.microsoft.com/ja-jp/dotnet/core/whats-new/dotnet-10/runtime>を参照してください。

- JIT コンパイラの機能強化
- AVX10.2 のサポート
- スタックの割り当て
- NativeAOT 型の事前初期化機能の改善
- Arm64 書き込みバリアの機能強化

## 参考リンク

- [C# 14 / .NET 10 の新機能 (RC 1 時点)] <https://speakerdeck.com/nenonaninu/net-10-noxin-ji-neng-rc-1-shi-dian>
  - RC 1時点の、何縫ねの。氏による解説です。

## 変更履歴

2025/11/06 拡張メンバーのコードが別のものになっていたため差し替えを実施

---
title: "BlazorWebAssemblyでExcelファイルを読み取る - C#"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "dotnet", "blazorwebassembly", "blazor", "excel"]
published: true
published_at : "2025-03-16 18:30"
---

## 概要

ブラウザ上でExcel操作をするライブラリといえば[SheetJs](https://sheetjs.com/)や、[ExcelJS](https://github.com/exceljs/exceljs
)です。

しかし、どうせなら公式というのにこだわりたいところです。

さて、Microsoft謹製のExcel操作のライブラリといえば[OpenXML](https://github.com/dotnet/Open-XML-SDK)です。

Blazorならそんなこだわりも叶えることができます。

## 前提

- WSL Ubuntu 22.04.5 LTS (Jammy Jellyfish)
- dotnet 8.0.112

## コード

この記事のコードは以下に配置しています。
<https://github.com/Kuroki-g/kuroki-g-public-zenn-code/tree/main/>

## .NETプロジェクトの作成

### ひな形の作成

ざっとプロジェクトを作ります。

```bash
$ dotnet new blazorwasm --name dotnet-blazorwasm-openxml
$ dotnet new gitignore
$ dotnet new editorconfig
```

### ライブラリの追加

[NugetのDocumentFormat.OpenXml](https://www.nuget.org/packages/DocumentFormat.OpenXml/)で最新版を確認して依存関係に追加します。

```bash
$ dotnet add package DocumentFormat.OpenXml --version 3.3.0
```

## BlazorWebAssemblyでのExcel生成コードの作成

### ページの作成

適当にひな形のCounter.razorをコピーして作ります。

Excel.razor

```razor
@page "/excel"

<PageTitle>Excel</PageTitle>

<h1>Excel</h1>

<button class="btn btn-primary" @onclick="GenerateSheet">Click me</button>

@code {
    private void GenerateSheet()
    {
        // Generate code here!
    }
}
```

### シートの生成

OpenXMLはかなり使いにくいので、コードサンプルを用意するのも面倒です。
なので、先人の遺産を参考にしましょう。

- [OpenXMLでExcelファイルを操作しよう (1) - Excelファイルを新規作成したい](https://qiita.com/gushwell/items/5c689e86014105313017#%E6%BA%96%E5%82%99)
- [ファイル名を指定してスプレッドシート ドキュメントを作成する](https://learn.microsoft.com/ja-jp/office/open-xml/spreadsheet/how-to-create-a-spreadsheet-document-by-providing-a-file-name?tabs=cs-0%2Ccs-100%2Ccs-1%2Ccs)

今回は適当なシートを作成するクラスを作成しました。

```csharp
using DocumentFormat.OpenXml;
using DocumentFormat.OpenXml.Spreadsheet;
using DocumentFormat.OpenXml.Packaging;

namespace Models;

sealed class SampleBook : IDisposable
{
    private SampleBook(string filePath)
    {
        _document = SpreadsheetDocument.Create(filePath, SpreadsheetDocumentType.Workbook);
        _workbookPart = _document.AddWorkbookPart();
        _workbookPart.Workbook = new Workbook();
        _sheets = _workbookPart.Workbook.AppendChild<Sheets>(new Sheets());

        // シート作成のための準備
        _worksheetPart = _workbookPart.AddNewPart<WorksheetPart>();
        _worksheetPart.Worksheet = new Worksheet(new SheetData());

    }

    private SpreadsheetDocument _document;

    private WorkbookPart _workbookPart;

    private WorksheetPart _worksheetPart;

    private Sheets _sheets;

    /// <summary>
    /// サンプルExcelのBookを新規作成する。
    /// </summary>
    /// <param name="filePath"></param>
    /// <returns></returns>
    public static SampleBook CreateBook(string filePath) {
        var book = new SampleBook(filePath);
        book.CreateSheet("シート1");

        return book;
    }

    public void CreateSheet(string sheetName)
    {

        // シートを作成し追加する
        var max = (uint)(_sheets.Count() + 1);
        var sheet = new Sheet {
            Id = _workbookPart.GetIdOfPart(_worksheetPart),
            SheetId = max,
            Name = sheetName
        };
        _sheets.Append(sheet);
    }

    /// <summary>
    /// 現在のbookを保存し、streamに渡すためのバイト列を生成する。
    /// </summary>
    public async Task<byte[]> SaveAndGetBytesAsync()
    {
        _workbookPart.Workbook.Save();
        
        using var stream = new MemoryStream();
        _document.Clone(stream);

        return stream.ToArray();
    }

    /// <summary>
    /// Book全体の保存を行う。
    /// </summary>
    public void Save() {
        _workbookPart.Workbook.Save();
    }
    
    private bool _isDisposed = false;

    public void Dispose() => Dispose(true);   

    private void Dispose(bool disposing)
    {
        if (!_isDisposed)
        {
            if (disposing)
            {
                _document?.Dispose();
            }
            _isDisposed = true;
        }
    }
}

```

### ダウンロード用関数の追加

JavaScriptコードを作成します。

```js
function DownloadFile(filename, contentType, content) {
    // Create a new Blob object from the byte array
    const data = new Uint8Array(content);
    const blob = new Blob([data], { type: contentType });

    // Create a URL for the blob
    const url = window.URL.createObjectURL(blob);

    // Create a temporary anchor element and trigger the download
    const a = document.createElement('a');
    a.href = url;
    a.download = filename; // Set the filename
    document.body.appendChild(a);
    a.click();

    // Clean up
    document.body.removeChild(a);
    window.URL.revokeObjectURL(url);
}
```

BlazorからJavaScriptの、ダウンロード用コードを呼び出せるようにします。
[ASP.NET Core Blazor で .NET メソッドから JavaScript 関数を呼び出す](https://learn.microsoft.com/ja-jp/aspnet/core/blazor/javascript-interoperability/call-javascript-from-dotnet?view=aspnetcore-9.0)を参考に。

そして適当にExcelブックを生成してダウンロードするためのJavaScript関数を呼び出します。

```razor
@using Models
@inject IJSRuntime JS
@code {
    private async void GenerateExcel()
    {
        Console.WriteLine("called GenerateExcel");
        
        using var book = SampleBook.CreateBook("example.xlsx");
        book.CreateSheet("シート2");
        book.Save();

        byte[] fileBytes = await book.SaveAndGetBytesAsync();
        
        await JS.InvokeVoidAsync(
            "DownloadFile",
            "example.xlsx",
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            fileBytes
        );
    }
}
```

### ブラウザ側からの挙動の確認

ローカル開発環境用サーバを立ち上げ、ブラウザから <http://localhost:5043/excel> にアクセスします。

```shell
dotnet run
```

![ブラウザの画面](/images/dotnet-blazorwasm-openxml/image0.png)

ボタンを押してダウンロードできることを確認します。

![ダウンロードの確認](/images/dotnet-blazorwasm-openxml/image1.png)

## 参考文献

- [OpenXMLでExcelファイルを操作しよう (1) - Excelファイルを新規作成したい](https://qiita.com/gushwell/items/5c689e86014105313017#%E6%BA%96%E5%82%99)
- [ファイル名を指定してスプレッドシート ドキュメントを作成する](https://learn.microsoft.com/ja-jp/office/open-xml/spreadsheet/how-to-create-a-spreadsheet-document-by-providing-a-file-name?tabs=cs-0%2Ccs-100%2Ccs-1%2Ccs)
- [ASP.NET Core Blazor で .NET メソッドから JavaScript 関数を呼び出す](https://learn.microsoft.com/ja-jp/aspnet/core/blazor/javascript-interoperability/call-javascript-from-dotnet?view=aspnetcore-9.0)

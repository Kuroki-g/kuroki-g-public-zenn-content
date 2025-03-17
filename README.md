# Zenn CLI

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

## Usage

```bash
Command:
  zenn init           コンテンツ管理用のディレクトリを作成. 初回のみ実行
  zenn preview        コンテンツをブラウザでプレビュー
  zenn new:article    新しい記事を追加
  zenn new:book       新しい本を追加
  zenn list:articles  記事の一覧を表示
  zenn list:books     本の一覧を表示
  zenn --version, -v  zenn-cliのバージョンを表示
  zenn --help, -h     ヘルプ

  👇  詳細
  https://zenn.dev/zenn/articles/zenn-cli-guide
```

本の雛形をコマンドで作成する

```bash
npx zenn new:book
```

本と各チャプターをプレビューする
公開範囲もあるのでドキュメント一読推奨

```bash
npx zenn preview
```

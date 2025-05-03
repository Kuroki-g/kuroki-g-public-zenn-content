---
title: "Steamのネタバレ防止を無視するChrome拡張機能「steam-spoiler-cleane」の作成"
emoji: "🔖"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["steam", "chrome拡張", "chromeextension"]
published: true
published_at : "2025-05-03 20:30"
---

## 概要

海外のゲームの攻略を見たいときにSteamのネタバレが大量にある、という場合に全てのspoilerを消し去る拡張機能です。

## コード

ソースコードは以下のGithubレポジトリで公開しています。
MITライセンスです。

<https://github.com/Kuroki-g/steam-spoiler-cleaner>

## 機能

- `span`タグかつ`class`が`bb_spoiler`のものを取り除き、親要素に取り込みます。

## インストール

パッケージ化していないので直接読み込んでください。
<https://developer.chrome.com/docs/extensions/get-started/tutorial/hello-world?hl=ja#load-unpacked>

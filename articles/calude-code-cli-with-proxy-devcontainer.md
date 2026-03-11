---
title: "iptablesなしでOK！SquidプロキシでClaude Codeの通信を制御するDevContainer構築"
emoji: "🦑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claudecode", "devcontainer", "squid", "docker"]
published: true
published_at : "2026-03-01 19:00"
---

## はじめに

Claude Code をDevContainer上で動かす際のアウトバウンド制限に、Squidプロキシを使うアプローチを紹介します。

https://github.com/Kuroki-g/kuroki-g-public-zenn-code/tree/main/articles/calude-code-cli-with-proxy-devcontainer

## 【前提】 Anthropicの公式アプローチ：iptables + ipset

Anthropicの[開発コンテナ](https://code.claude.com/docs/ja/devcontainer)では、[init-firewall.sh](https://github.com/anthropics/claude-code/blob/main/.devcontainer/init-firewall.sh) によるiptablesベースの制御が採用されています。

```
コンテナ起動時
  ↓
GitHub APIからIPレンジを取得（動的）
  ↓
dig でその他ドメインをIPに解決
  ↓
ipset (hash:net) に追加
  ↓
iptables OUTPUT チェーンで ipset 以外を DROP
```

- ネットワーク層（L3/4）での制御。カーネルレベルなのでアプリからの回避が困難
- 全プロトコル（TCP/UDP/ICMP等）を対象にできる
- コンテナ内で完結する1コンテナ構成

## Squidをプロキシサーバーに用いた場合

Squidをフォワードプロキシとして別コンテナに立て、Dockerのネットワーク分離と組み合わせます。IPではなくドメイン名でフィルタリングするため、CDNやパッケージレジストリのIP変更に追従できます。

### ネットワーク構成とcompose.yml（抜粋）

![ネットワーク構成](/images/calude-code-cli-with-proxy-devcontainer/image0.drawio.png)

`internal: true` でDockerがデフォルトゲートウェイを付与しないため、`claude-dev` はSquid経由でしか外部に出られません。

```yaml
services:
  http-proxy:
    image: ubuntu/squid:6.13-25.04_beta
    restart: unless-stopped
    command: ["squid", "-N"]
    networks:
      - internal
      - external
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
      - SETGID
      - SETUID
    security_opt:
      - no-new-privileges:true

  claude-dev:
    networks:
      - internal   # internalのみ。外部への直接経路なし
    environment:
      - http_proxy=http://http-proxy:3128
      - https_proxy=http://http-proxy:3128
      - HTTP_PROXY=http://http-proxy:3128
      - HTTPS_PROXY=http://http-proxy:3128

networks:
  internal:
    internal: true  # ← ここが肝
  external:
    driver: bridge
```

### squid.conf（抜粋）

:::message alert
`dstdomain` はドットの有無で挙動が変わります。`.example.com` はサブドメインを含み、`example.com`（ドットなし）は完全一致のみです。

`http_access allow CONNECT` はHTTPSトンネリング専用のため、平文HTTP（ポート80）は通過しません。HTTP専用エンドポイントが必要な場合は別途ルールを追加してください。
:::

```squid
acl internal_net src 172.16.0.0/12

acl allowed_domains dstdomain \
    .anthropic.com \
    platform.claude.com \
    .github.com \
    .githubusercontent.com \
    # ...プロジェクトに必要なドメインを追加...

http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow CONNECT internal_net allowed_domains
http_access deny all

forwarded_for off
via off
```

## モチベーション

開発コンテナに `NET_ADMIN` と `sudo` を与えたくないのが主な理由です。iptablesアプローチはカーネルレベルの強制力が強みですが、[devcontainer.json](https://github.com/anthropics/claude-code/blob/main/.devcontainer/devcontainer.json) を見ると `sudo` が必要です。プロキシを別コンテナに分離することで、開発コンテナ側に特権を与えずに制御できます。ライセンスの都合で公式コードを流用できなかったことも一因です。

| 観点                 | iptables + ipset           | Squid（本記事）           |
| -------------------- | -------------------------- | ------------------------- |
| 制御レイヤー         | ネットワーク層（L3/4）     | アプリケーション層（L7）  |
| ホワイトリストの基準 | IPアドレス（起動時に解決） | ドメイン名（接続時に解決）|
| 制御できるプロトコル | 全プロトコル（TCP/UDP等）  | HTTP/HTTPSのみ            |

## Claude CodeのOAuth認証問題と回避策

`claude login` 実行時、CLIはOAuthコールバック用サーバーを `[::1]`（IPv6ループバック）のエフェメラルポートで起動します。VS Codeは `[::1]` bindのポートを自動転送できず、`internal` ネットワーク内からホストへの経路もないため、ブラウザからコールバックURLに到達できません。

### 回避策：socatでリバインド


~~`0.0.0.0` へのリバインドでVS Codeの自動転送を有効にします。~~

2026/03/12追記: リバインド不要になったようです。

```bash
#!/bin/bash
# change-network.sh
#
# 使い方:
#   1. このスクリプトを実行して待機させる
#   2. 別タブで `claude login` を実行する
#   3. ポートを検知したらsocatで0.0.0.0にリバインドし、VS Codeが自動フォワード
#   4. ブラウザでOAuth認証を完了したらCtrl+Cで終了

EXPECTED_BIND="[::1]"

while true; do
    line=$(ss -tlnp 2>/dev/null | grep '"claude"' || true)

    if [[ -n "$line" ]]; then
        bind_addr=$(echo "$line" | awk '{print $4}' | rev | cut -d: -f2- | rev)
        port=$(echo "$line" | awk '{print $4}' | rev | cut -d: -f1 | rev)

        echo "Detected: ${bind_addr}:${port}"
        echo "Starting socat relay on 0.0.0.0:${port} ..."

        socat "TCP-LISTEN:${port},bind=0.0.0.0,reuseaddr,fork" \
              "TCP6:[::1]:${port}" &

        echo "Ready. Complete the OAuth flow in your browser."
        wait $!
        break
    fi

    sleep 0.5
done
```

## 変更履歴

- 2026/03/06 タイトルを変更
- 2026/03/12 リバインドについて追記

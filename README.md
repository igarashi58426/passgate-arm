# passgate-arm

[WebAuthn(FIDO2) パスキー認証ゲート Passgate](https://github.com/yankeguo/passgate) の arm64 対応 Docker イメージ。

> **このリポジトリは Passgate の再実装・フォークではありません。**
> 上流のソースコード・Dockerfile を**無改修のまま** clone し、`linux/arm64` 向けにビルドして公開するだけの、薄いビルドラッパーです。すべての設計・実装のクレジットは上流の作者 [Y.-K. Guo (@yankeguo)](https://github.com/yankeguo) に帰属します。

## Passgate について（上流 README より）

> A single-user PassKey gate that adds authentication to any web service — first visit registers your key, every visit after requires it.

Passgate は、認証のないセルフホスト Web サービスの前面に立つ**単一ユーザー向けパスキー認証ゲート**です。Caddy basic auth のような「覚えるパスワードが増える」方式の代わりに WebAuthn を採用し、パスキーによる認証が成功するまですべてのリクエストを遮断し、成功後は JWT クッキーでサインインを維持したままリバースプロキシします。

- **初回アクセスで登録** — 認証情報が無い状態では、起動時に表示されるセットアップキー(128-bit)の入力後にのみパスキーを登録できる。パスキー登録後、キーは自動的に無効化される
- **セットアップキーはメモリ上のみ** — ディスクに書かれず、登録成功時に消去される
- **認証済みリクエストはそのまま転送** — `net/http/httputil` による透過プロキシ(WebSocket 対応)。Passgate 自身のパスはすべて `/__passgate/` 配下に集約され、上流アプリのルートと衝突しない
- **状態は単一ファイル** — credential と JWT 署名鍵を `$PASSGATE_DATA_DIR/state.json` に保存(atomic write)。ユーザーは常に 1 人

上流は「完全に透明なプロキシであり、いかなるトラフィック制御も課さない」という設計思想を掲げています。当リポジトリもこの思想を尊重し、**上流への差分はゼロ**を維持しています。

- 上流リポジトリ: `https://github.com/yankeguo/passgate` (MIT License, Copyright (c) 2026 Y.-K. Guo)
- パッチ: **なし**。上流のマルチステージ Dockerfile(`oven/bun` でフロントエンド静的アセット → `golang` で `CGO_ENABLED=0` の単一バイナリ)が arm64 のまま動作するため

## なぜこのリポジトリが存在するのか

公式イメージ `ghcr.io/yankeguo/passgate` は **amd64 のみ**が公開されており、Raspberry Pi(aarch64) では動作しません。
そこで本リポジトリが arm64 版を日次で自動ビルドし、GHCR に公開しています。

GitHub Actions により毎日 1 回、上流リポジトリの最新ソースを clone → ビルド → push します。
上流が更新されると翌朝には arm64 イメージに反映されるため、マージコストや追従の手間はありません。

## イメージ

```
ghcr.io/igarashi58426/passgate-arm:latest
```

`YYYY-MM-DD` 形式の日付タグも付与され、過去のビルドが蓄積されていきます（例: `2026-10-28`）。

## 使い方

raspi-sava の `auth-passgate` から、本イメージを認証ゲートとして使用します。

```yaml
services:
  webssh-passgate:
    image: ghcr.io/igarashi58426/passgate-arm:latest
    container_name: webssh-passgate
    hostname: webssh.lan
    restart: always
    dns: [172.16.0.54]
    networks: [raspi_sava_net]
    expose: [80]
    env_file:
      - ../auth-passgate/env/passgate.env
    environment:
      - PASSGATE_UPSTREAM=http://webssh-app.lan
      - PASSGATE_ORIGIN=https://webssh.sava4re.uk
    volumes:
      - webssh_passgate:/data # state.json(認証情報・署名秘密鍵)の永続化
    healthcheck: {test: "curl -f http://localhost:80/__passgate/healthz || exit 1"}
```

主要な環境変数（上流の設定をそのまま引き継ぎます）:

| 変数 | 既定値 | 説明 |
|---|---|---|
| `PASSGATE_LISTEN` | `:8080` | リッスンアドレス（`env/passgate.env` で `:80` に上書き） |
| `PASSGATE_UPSTREAM` | （必須） | 認証成功後に転送するアプリの URL |
| `PASSGATE_DATA_DIR` | `./data` | `state.json` の保存先 |
| `PASSGATE_SESSION_TTL` | `168h` | セッションクッキーの有効期間 |
| `PASSGATE_ORIGIN` | per-request | 外部オリジンを固定する場合に設定（WebAuthn RP ID 検証の根拠） |
| `PASSGATE_TITLE` | `passgate` | ゲートページのタイトル |

## 初回パスキー登録

1. コンテナを起動すると、**セットアップキー(128-bit)** がログに出力される
   ```bash
   docker logs webssh-passgate
   # no passkey registered; enter this setup key to register the first one:
   # XXXX-XXXX-XXXX-XXXX-XXXX-XXXX-XXXX-XXXX
   ```
2. ブラウザで `https://webssh.sava4re.uk/` にアクセス → `/__passgate/` へリダイレクトされる
3. セットアップキーを入力し、生体認証/PIN でパスキーを登録
4. 登録完了後、セットアップキーは自動的に無効化される

> ⚠️ パスキーは RP ID(ホスト名)に紐付きます。ホスト名を変えると登録済みキーが一致せずロックアウトされるため、その場合は `state.json` を削除して再登録してください（raspi-sava では `raspi-sava-volume-backup` により `state.json` が日次バックアップされています）。

## CI/CD

`.github/workflows/build.yml` のスケジュール（毎日 21:00 UTC / JST 06:00）で自動ビルド・push されます。
手動実行も `workflow_dispatch` に対応しています。

| ステップ | 内容 |
|---|---|
| Clone | 上流リポジトリを `--depth 1` で取得（パッチは適用しない） |
| Build | 上流の Dockerfile を `linux/arm64` 向けにビルド |
| Push | `ghcr.io/igarashi58426/passgate-arm` へ `latest` + 日付タグ |

## クレジットとライセンス

- **[Passgate](https://github.com/yankeguo/passgate)** — [@yankeguo](https://github.com/yankeguo) による素晴らしいプロジェクト。この arm64 イメージは上流がなければ存在しません。問題報告・機能要望・寄付は本リポジトリではなく、**上流へ**お願いします
- 本リポジトリのビルド定義・ワークフロー: MIT License（[LICENSE](./LICENSE)）
- Passgate 本体: MIT License（Copyright (c) 2026 Y.-K. Guo）

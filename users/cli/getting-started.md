# CLI を使いはじめる

このガイドでは、zERC20 コマンドラインインターフェース（CLI）のインストールと使い方を説明します。

## インストール

```bash
git clone https://github.com/kbizikav/zerc20.git
cd zerc20/cli
cargo install --path .
```

CLI バイナリは `zerc20-cli` としてインストールされます（古いバージョンでは `cli` の場合があります）。リポジトリから直接実行する場合：

```bash
cd zerc20/cli
cargo run -r -- <command> ...
```

## 事前準備

### Circuit アーティファクト

引き出し証明の生成には Circuit アーティファクトが必要です。CLI を使う前にダウンロードしてください：

```bash
cargo install --path ../circuit-setup
zerc20-circuit-setup download --version 1.1.0
```

詳細は [Circuit セットアップ](../../developers/circuit-setup.md) を参照してください。

## 設定

### 環境変数

`.env` から環境変数を読み込みます（`cli` ディレクトリの `.env.example` を参照）：

```bash
# トークン設定
export TOKENS_FILE_PATH=../config/tokens.json

# Internet Computer エンドポイント（必須）
export IC_REPLICA_URL=<IC_URL>
export KEY_MANAGER_CANISTER_ID=<CANISTER_ID>
export STORAGE_CANISTER_ID=<CANISTER_ID>

# 資金の受け取りに必要（ダウンロードした Circuit アーティファクトのパス）
export NOVA_ARTIFACTS_DIR=/path/to/nova_artifacts
```

Mainnet / Testnet の値については [ICP Canister ID](../../reference/addresses.md#icp-canister-ids) を参照してください。

## 基本コマンド

### Invoice を発行する

支払いを受け取るためのバーンアドレス（Burn Address）を生成します：

```bash
zerc20-cli invoice issue --chain-id <CHAIN_ID>
```

### Invoice 一覧を確認する

発行済みの Invoice を表示します：

```bash
zerc20-cli invoice ls --chain-id <CHAIN_ID>
```

### 送金する

バーンアドレスに zERC20 を送金します：

```bash
zerc20-cli transfer \
  --chain-id <CHAIN_ID> \
  --to <BURN_ADDRESS> \
  --amount <AMOUNT_IN_WEI>
```

### Invoice ステータスを確認する

```bash
zerc20-cli invoice status --chain-id <CHAIN_ID> --invoice-id <INVOICE_ID>
```

### 資金を受け取る

Proof を生成して引き出しを実行します：

```bash
zerc20-cli invoice receive --chain-id <CHAIN_ID> --invoice-id <INVOICE_ID>
```

## クイックスタート例

```bash
# 1. Invoice を発行する
zerc20-cli invoice issue --chain-id 1

# 2. Invoice 一覧で Invoice ID とバーンアドレスを確認する
zerc20-cli invoice ls --chain-id 1

# 3. バーンアドレスに資金を送金する
zerc20-cli transfer \
  --chain-id 1 \
  --to 0x1234567890abcdef1234567890abcdef12345678 \
  --amount 1000000000000000000

# 4. ステータスを確認する
zerc20-cli invoice status --chain-id 1 --invoice-id inv-01

# 5. 資金を受け取る
zerc20-cli invoice receive --chain-id 1 --invoice-id inv-01
```

## 重要な注意事項

- **クロスチェーン対応**：あるチェーンで送金し、別のチェーンで受け取ることができます
- **処理時間**：Mainnet でのプライベート転送（Private Transfer）は通常30分〜1時間かかります
- **Testnet の制限**：LayerZero の不安定性により、Testnet では処理に時間がかかる場合があります

## 次のステップ

- [FAQ](../faq.md) — よくある質問とトラブルシューティング
- [CLI README](https://github.com/InternetMaximalism/zerc20/tree/main/cli) — CLI の完全なドキュメント

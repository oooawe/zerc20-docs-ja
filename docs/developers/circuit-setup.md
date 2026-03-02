---
icon: brain-circuit
---

# Circuit セットアップ

このガイドでは、`zerc20-circuit-setup` CLI ツールを使って zERC20 の Circuit アーティファクトをダウンロードしてテストする方法を説明します。

## Circuit Artifacts が必要な理由

Circuit Artifacts は以下に必要です：

* **Indexer の実行**：Indexer が転送 Root 遷移の Nova Proof を生成するために使用
* **Decider Prover の実行**：Decider Prover がオンチェーン検証用に Nova Proof を Decider Proof に最終化するために使用
* **CLI での Proof 生成**：CLI が引き出し Proof（Nova・Groth16 両方）を生成するために使用

これらのアーティファクトがなければ、Indexer・Decider Prover の実行も、CLI での Proof 生成もできません。

## 公式マニフェストダイジェスト

`download` コマンド実行時に表示されるマニフェストダイジェストが、以下の公式値と一致することを必ず確認してください。

| バージョン   | マニフェストダイジェスト                                                       |
| ------- | ------------------------------------------------------------------ |
| `1.0.0` | `f8181f89d502cd5bebc4445c4305c6c692f92deb202a18ce5d7c41694b10a7a4` |
| `1.1.0` | `f22882b901ac1585f665f3aa7c812dc688a0cc08d91f84c2af3c390448a85373` |

## インストール

```bash
# Clone the repository
git clone https://github.com/kbizikav/zerc20.git
cd zerc20

# Install the CLI
cargo install --path circuit-setup
```

## 設定

環境変数を設定するか、`.env` ファイルを使用します。

```bash
# Copy the example environment file
cp circuit-setup/.env.example circuit-setup/.env
```

`.env.example` の内容：

```bash
# Artifact version
ARTIFACTS_VERSION=1.1.0

# Local directory for circuit artifacts
NOVA_ARTIFACTS_DIR=../nova_artifacts

# Base URL for download (public HTTP/HTTPS)
ARTIFACTS_BASE_URL=https://zerc20-prod-public-uploads.s3.ap-southeast-1.amazonaws.com/circuit-setup

# Logging level (error, warn, info, debug, trace)
RUST_LOG=info
```

## コマンド

### ダウンロード

公開 URL から Circuit アーティファクトをダウンロードし、SHA256 ハッシュを自動検証します。

```bash
zerc20-circuit-setup download --version 1.1.0
```

URL を明示的に指定する場合：

```bash
zerc20-circuit-setup download \
  --version 1.1.0 \
  --base-url https://zerc20-prod-public-uploads.s3.ap-southeast-1.amazonaws.com/circuit-setup \
  --artifacts-dir ./nova_artifacts
```

| オプション             | 環境変数                 | 説明              |
| ----------------- | -------------------- | --------------- |
| `--version`       | `ARTIFACTS_VERSION`  | ダウンロードするバージョン   |
| `--base-url`      | `ARTIFACTS_BASE_URL` | アーティファクトの公開 URL |
| `--artifacts-dir` | `NOVA_ARTIFACTS_DIR` | ローカルの出力ディレクトリ   |

### テスト

ダウンロードしたアーティファクトを使ってダミー Proof を生成・検証します。

```bash
zerc20-circuit-setup test --artifacts-dir ./nova_artifacts
```

## 詳細情報

完全な使用方法は [README](https://github.com/InternetMaximalism/zerc20/blob/main/circuit-setup/README.md) を参照してください。

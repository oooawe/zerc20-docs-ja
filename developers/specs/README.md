# 技術仕様

zERC20 各コンポーネントの詳細な技術仕様です。

## 仕様一覧

- [コントラクト仕様](contract-spec.md) — スマートコントラクトのインターフェースと挙動
- [ZKP 仕様](zkp-spec.md) — ゼロ知識回路の仕様
- [ICP ストレージ仕様](icp-storage-spec.md) — Internet Computer 上のステルスメッセージングレイヤー

## クイックリファレンス

### 値の制約

| 制約 | 値 | 理由 |
|------|----|------|
| 最大転送量 | 2^248 - 1 | BN254 スカラーフィールドに収まる必要がある |
| アドレスサイズ | 160 ビット | Ethereum アドレスの幅 |
| Merkle Tree の深さ | 40（ローカル）/ 46（グローバル） | トークンごとのツリーとグローバルツリー |
| PoW 難易度 | 16 ビット | バーンアドレス（Burn Address）の衝突耐性 |

### Proof の種類

| Proof の種類 | 回路 | ユースケース |
|------------|------|------------|
| Root Transition | Nova | 新しい転送ルートの証明 |
| Batch Withdraw | Nova | 複数の引き出しを 1 つの Proof にまとめる |
| Single Withdraw | Groth16 | 単一の引き出し（高速） |

### 主要コントラクト

| コントラクト | 主な関数 |
|------------|---------|
| zERC20 | `transfer`、`teleport`、`mint`、`burn` |
| Verifier | `proveTransferRoot`、`teleport`、`singleTeleport` |
| Hub | `broadcast`、`registerToken` |

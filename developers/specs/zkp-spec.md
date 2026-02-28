# ZKP 仕様

## 概要

zERC20 はゼロ知識証明（Zero-Knowledge Proof）を用いて、送信者と受信者の関係を明かすことなくプライベート転送（Private Transfer）を検証します。すべての回路は BN254 スカラーフィールド上で動作します。

**ファイル**: `zkp/src/circuits/`

## 制約

| 制約 | 値 | 理由 |
|------|----|------|
| 最大値 | 248 ビット（31 バイト） | BN254 スカラーフィールドに収まる必要がある |
| アドレスサイズ | 160 ビット | Ethereum アドレスの幅 |
| PoW 難易度 | 16 ビット | バーンアドレス（Burn Address）の衝突耐性 |

## 回路

### 1. バーンアドレスの導出

有効なバーンアドレスをプルーフ・オブ・ワーク（PoW）難易度付きで決定論的に生成します。

```
burn_address = truncate_160(Poseidon3(domain_separator, recipient, secret))

where:
  domain_separator = field_encode("burn")
  recipient = Poseidon3(chain_id, address, tweak)
```

**PoW 制約**：ビット `[160, 160 + POW_DIFFICULTY)` はゼロでなければならず、有効な `(recipient, secret)` ペアを見つけるのに約 2^16 回の試行が必要です。

### 2. Single Withdraw（Groth16）

Merkle Tree 内に、バーンアドレスへの単一の転送が存在することを証明します。

**公開入力**：
- `merkle_root`：現在の転送ツリーのルート
- `recipient`：GeneralRecipient のハッシュ
- `withdraw_value`：引き出し量

**秘密入力**：
- `from_address`、`value`、`delta`
- `secret`、`leaf_index`、`siblings[]`

**制約**：
1. `(recipient, secret)` から PoW チェック付きでバーンアドレスを再計算
2. リーフをハッシュ：`Poseidon3(from_address, burn_address, value)` — `burn_address` は転送リーフの `to` フィールド（[インテグレーションガイド](../integration.md#leaf-structure)を参照）
3. `merkle_root` に対して Merkle インクルージョンを検証
4. `withdraw_value = value - delta` を出力

### 3. Withdraw Step（Nova）

バッチ引き出し（Batch Withdrawal）用のフォールディングガジェット。複数の転送を順序を保ちながら集約します。

**アキュムレータの状態**：
- `merkle_root`：転送ツリーのルート
- `recipient`：GeneralRecipient（バッチ全体で一定）
- `leaf_index_with_offset`：最後に処理したインデックス + 1
- `total_value`：累積値の合計

**ステップごとの入力**：
- `from_address`、`value`、`secret`
- `leaf_index`、`siblings[]`
- `is_dummy`：本検証をスキップ（パディング用）

**制約**：
1. 順序の強制：`prev_leaf_index_with_offset < leaf_index + 1`
2. `is_dummy = false` の場合：
   - バーンアドレスの PoW を検証
   - Merkle インクルージョンを検証
   - `value` を合計に加算
3. `is_dummy = true` の場合：
   - PoW および Merkle チェックをスキップ
   - プライバシー調整のため合計から `value` を減算

**ダミーステップ**：バッチ長の秘匿と、Merkle ルートに触れることなく端数の余剰調整を可能にします。

### 4. Root Transition Step（Nova）

オンチェーンの `IndexedTransfer` イベントとオフチェーンの Merkle Tree を連携させます。

**アキュムレータの状態**：
- `index`：現在の転送インデックス
- `hash_chain`：SHA-256 チェーン値（248 ビット）
- `root`：現在の Merkle ルート

**ステップごとの入力**：
- `from_address`、`to_address`、`value`
- `siblings[]`
- `is_dummy`：本検証をスキップ

**制約**：
1. `is_dummy = false` の場合：
   - `index` でのゼロリーフが存在する前のルートを検証
   - リーフを計算：`Poseidon3(from_address, to_address, value)`
   - 新しいリーフでルートを更新
   - ハッシュチェーンを更新：`SHA256(hash_chain || from_address || to_address || value)[0:248]`
   - インデックスをインクリメント
2. `is_dummy = true` の場合：
   - 状態をそのまま引き継ぐ

## Proof の種類

| 種類 | 回路 | プルーバー | ユースケース |
|------|------|----------|------------|
| Single Withdraw | Groth16 | ローカル（WASM） | 単一の転送、高速 |
| Batch Withdraw | Nova + Decider | サーバー | 複数の転送をまとめて処理 |
| Root Transition | Nova + Decider | サーバー | 新しいルートの証明 |

## 検証パス

### Single Teleport

```
Client → Groth16 proof → Verifier.singleTeleport() → singleWithdrawVerifier contract
```

### Batch Teleport

```
Client → Nova IVC proof → Decider Prover → Decider proof → Verifier.teleport() → withdrawDecider contract
```

### Root Proving

```
Indexer → Nova IVC proof → Decider Prover → Decider proof → Verifier.proveTransferRoot() → rootDecider contract
```

## セキュリティに関する注記

- **PoW セキュリティ**：16 ビット PoW により、衝突耐性は約 2^89（詳細はソース仕様の分析を参照）
- **パディング**：バッチサイズの漏洩を防ぐため、Nova Proof は最大インデックスまでパディングを推奨
- **ハッシュチェーン**：BN254 との互換性のため 248 ビットに切り捨て

---
icon: head-side-circuit
---

# Proof of Innocence（予定）

> **⚠️ Warning：** 未実装の機能です。このページは計画中の機能を説明しています。以下のインターフェースと設計は変更される可能性があり、現時点では使用できません。

Proof of Innocence は AMLポリシーとして機能するオフチェーンのコンプライアンス機能です。取引の詳細を明かさずに、受け取った資金が制裁対象アドレス（例：OFACリスト）に由来しないことを証明できます。

## 概要

| プロパティ      | 説明                             |
| ---------- | ------------------------------ |
| **目的**     | 送信者アドレスが制裁リストに含まれていないことを証明     |
| **証明システム** | Groth16（単体 Proof） / Nova（集約）   |
| **検証方法**   | オフチェーン CLI Verifier            |
| **データ構造**  | 制裁リスト用 Sparse Merkle Tree（SMT） |

## 仕組み

```
┌─────────────────────────────────────────────────────────────────┐
│                    Proof of Innocence Flow                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Prove that `totalTeleported` per recipient (hash of         │
│     `GeneralRecipient`) is not originated from OFAC-sanctioned  │
│     sources.                                                    │
│                        ↓                                        │
│  2. Commit the OFAC list using a commitment scheme that         │
│     supports non-membership proofs (e.g., Exclusion Tree).      │
│                        ↓                                        │
│  3. For each `from_address`, generate non-membership proof.     │
│                        ↓                                        │
│  4. For each teleport, prove that `transfer_leaf.from` is not   │
│     in the OFAC list, then aggregate the steps with Nova.       │
│                        ↓                                        │
│  5. The Verifier checks the Nova proof with public inputs:      │
│     `recipient`, `totalTeleported`, and the trusted OFAC SMT    │
│     root.                                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 暗号設計

### 除外ツリー（Exclusion Tree）

「OFAC 非メンバーシップツリー」は、OFACリストの要素間のギャップを表すソート済みの互いに素な（`start`、`end`）ペアからなる Merkle Tree です。各領域 `start..=end` の和集合が制裁リストの補集合と完全に一致します。

OFACリストの各ツリーのリーフは以下のように計算されます：

```rust
leaf = poseidon2(start, end)
```

制裁リストに **k** 個のアドレスが含まれる場合、除外ツリーには **k + 1** 個のギャップ（最初のアドレスの前に1つ、連続するペア間に各1つ、最後のアドレスの後に1つ）があります。最小ツリーの高さは **⌈log₂(k + 1)⌉** です。リストの増加に対応してアーティファクトを再生成しなくて済むよう、実際には2〜3レベルの余裕を持たせることを推奨します。

### 非メンバーシップ証明（Nova ステップ）

受信者への各転送は `transfer_leaf` = `(from_address, burn_address, value)` を持ちます（`burn_address` は汎用転送リーフの `to` フィールドに相当）。

`recipient` ごとの `totalTeleported` が制裁対象から由来しないことを証明するために、各転送で送信者（`transfer_leaf.from`）が OFACリストに含まれていないことを証明します。

**パブリック入力（全ステップで共通）：**

1. `ofac_root` — OFAC 非メンバーシップツリーの Root
2. `recipient` — GeneralRecipient のハッシュ

**アキュムレータ状態：**

1. `totalTeleported` — 転送金額の累計

**ステップごとのプライベート入力：**

1. `from_address` — この転送の送信者（`transfer_leaf.from`）
2. `value` — 転送金額
3. `start`、`end` — `from_address` を含むギャップの境界値
4. `path`、`position` — ギャップの Merkle Proof

**ステップごとの制約：**

1. **送信者が制裁対象外**：`start < from_address < end` を `ofac_root` に対して検証
2. **金額の累計**：`totalTeleported_new = totalTeleported_old + value`

## CLI 使用方法（予定）

> **📃 Note：** これは MVP インターフェースです。ユーザーはウィットネスファイルを手動で組み立てる必要があります。

### Proof of Innocence を生成する

受信者への全転送が制裁対象外アドレスから発生していることを示す Nova Proof を生成します。

```bash
# Generate proof for a recipient
zerc20-cli proof-of-innocence generate \
  --recipient <GENERAL_RECIPIENT_HASH> \
  --ofac-root <OFAC_TREE_ROOT> \
  --transfers-file <TRANSFERS_JSON> \
  --exclusion-proofs-file <EXCLUSION_PROOFS_JSON> \
  --output <PROOF_OUTPUT_PATH>
```

| 引数                        | 説明                                       |
| ------------------------- | ---------------------------------------- |
| `--recipient`             | GeneralRecipient のハッシュ（引き出しフローから取得）      |
| `--ofac-root`             | 信頼できる OFAC 除外ツリーの Root                   |
| `--transfers-file`        | （from\_address、value）ペアのリストを含む JSON ファイル |
| `--exclusion-proofs-file` | OFACツリーサービスからの除外 Proof を含む JSON ファイル     |
| `--output`                | 生成した Proof の出力パス                         |

### Proof of Innocence を検証する

```bash
# Verify a proof
zerc20-cli proof-of-innocence verify \
  --proof <PROOF_PATH> \
  --recipient <GENERAL_RECIPIENT_HASH> \
  --total-teleported <TOTAL_VALUE> \
  --ofac-root <OFAC_TREE_ROOT>
```

| 引数                   | 説明                     |
| -------------------- | ---------------------- |
| `--proof`            | Proof ファイルのパス          |
| `--recipient`        | 期待される受信者ハッシュ           |
| `--total-teleported` | 期待される合計金額（wei）         |
| `--ofac-root`        | 信頼できる OFAC 除外ツリーの Root |

### 使用例

```bash
# 1. Generate proof
zerc20-cli proof-of-innocence generate \
  --recipient 0x1a2b3c...recipient_hash \
  --ofac-root 0x5678...ofac_root \
  --transfers-file ./my_transfers.json \
  --exclusion-proofs-file ./exclusion_proofs.json \
  --output ./innocence_proof.bin

# 2. Verify proof
zerc20-cli proof-of-innocence verify \
  --proof ./innocence_proof.bin \
  --recipient 0x1a2b3c...recipient_hash \
  --total-teleported 1000000000000000000 \
  --ofac-root 0x5678...ofac_root

# Output:
# ✓ Proof valid
# recipient: 0x1a2b3c...
# totalTeleported: 1000000000000000000 (1.0 ETH)
# ofac_root: 0x5678...
```

### 入力ファイル形式

`transfers.json` — 証明対象の転送リスト：

```json
[
  { "from_address": "0xabc...", "value": "500000000000000000" },
  { "from_address": "0xdef...", "value": "500000000000000000" }
]
```

`exclusion_proofs.json` — OFACツリーサービスからの除外 Proof：

```json
[
  {
    "from_address": "0xabc...",
    "start": "0x000...",
    "end": "0x111...",
    "path": ["0x...", "0x..."],
    "position": 5
  },
  {
    "from_address": "0xdef...",
    "start": "0x222...",
    "end": "0x333...",
    "path": ["0x...", "0x..."],
    "position": 12
  }
]
```

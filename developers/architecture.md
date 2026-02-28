# アーキテクチャ概要

このドキュメントでは、zERC20 のオンチェーンコントラクト・オフチェーンサービス・Internet Computer（ICP）ステルスメッセージングレイヤーにまたがるアーキテクチャを説明します。

- **プライバシー保護転送**：送信者はステルスアドレスに zERC20 をバーン。受信者は同一チェーンまたは別チェーンで、オンチェーン上に紐付け可能なメタデータを残さずにミント
- **ERC-20 互換性**：標準トークンインターフェースで既存のウォレット・DEX・DeFiプロトコルと連携
- **証明可能な整合性**：すべてのミントは ZKP（Zero-Knowledge Proof）（Nova / Groth16）に基づく
- **スケーラビリティ**：Poseidon Merkle Tree と IVC（Incrementally Verifiable Computation）が数千の転送を1つの Proof にまとめる
- **クロスチェーン**：LayerZero ベースの Hub が全チェーンの Root を集約

## オンチェーンコントラクト

| コントラクト | 役割 |
|----------|---------|
| **zERC20** | `IndexedTransfer` イベントを発行し、SHA-256 Hash Chain を管理し、検証済みミント用の `teleport` を公開するアップグレード可能な ERC-20 |
| **Verifier** | Nova / Groth16 Proof を検証し、受信者ごとの Teleport 済み金額を追跡し、Root を Hub にリレーする LayerZero OApp |
| **Hub** | 全チェーンの転送 Root を Poseidon ツリーに集約し、グローバル Root を全 Verifier にブロードキャスト |
| **LiquidityManager** | 流動性ポリシーを管理し、原資産トークンの Wrap / Unwrap を処理 |
| **Adaptor** | 別チェーンの方が流動性が有利な場合に Stargate 経由でクロスチェーン出金 |

## オフチェーンサービス

| サービス | 役割 |
|---------|---------|
| **Indexer** | Actix HTTP サーバー + Postgres。オンチェーンイベントを同期し、Merkle Tree を管理し、Root Proof を生成 |
| **Decider Prover** | オンチェーン検証用に Nova Proof を最終化する HTTP ワーカー |
| **Cross-chain Job** | 転送 Root を Hub にリレーし、ブロードキャストをトリガー |

## ICP ステルスストレージ

| コンポーネント | 役割 |
|-----------|---------|
| **Key Manager Canister** | EVM アドレスごとに VetKD バックの IBE 鍵を導出 |
| **Storage Canister** | 暗号化されたアナウンスと署名済み Invoice を保管 |

## データフロー

### 1. 転送のエミット

```
zERC20._update() → emits IndexedTransfer → updates hashChain
```

すべての転送（ミント / 転送 / バーン）は SHA-256 Hash Chain に追記され、インデックス付きイベントを発行します。

### 2. Root の証明

<div align="center">
  <img src="../assets/design/merkle_tree.png" alt="Merkle Tree and Hash Chain" width="700">
</div>

Indexer は Poseidon Merkle Tree を管理し、IVC Proof を使って定期的に新しい転送 Root をオンチェーンで証明します。

### 3. クロスチェーン集約

<div align="center">
  <img src="../assets/design/crosschain.png" alt="Cross-chain Architecture" width="700">
</div>

各チェーンの Verifier が転送 Root を Hub にリレーし、Hub はそれらを集約してグローバル Root を全 Verifier にブロードキャストします。

### 4. プライベート転送

```
Sender → zERC20 transfer to burn address → Recipient scans ICP storage →
Generates ZKP → Verifier.teleport() → zERC20 mints to recipient
```

## 暗号プリミティブ

| プリミティブ | 用途 |
|-----------|-------|
| **Poseidon Hash** | Merkle Tree、バーンアドレスの導出、受信者バインディング |
| **SHA-256** | Hash Chain のコミットメント（BN254 に合わせて248ビットに切り詰め） |
| **Nova Folding** | バッチ引き出し（Batch Withdrawal）Proof、Root 遷移 Proof |
| **Groth16** | 単体引き出し Proof |
| **VetKD / IBE** | ICP 上の暗号化ステルスメッセージング |

## トラストモデル

| アクター | 信頼の前提 |
|-------|------------------|
| Contract Owner | コントラクトのアップグレード、Verifier のローテーションが可能 |
| Indexer Operator | 送信者 / バーンアドレス / 金額を観察。クエリ時に受信者を把握 |
| ICP Canisters | 暗号化データを保管。受信者の鍵なしでは復号不可 |

**最大限のプライバシーを確保するには**：送受信者の紐付け漏洩を避けるため、自前の Indexer インスタンスを実行してください。

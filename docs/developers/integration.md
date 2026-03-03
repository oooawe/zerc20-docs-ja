---
icon: timeline-arrow
description: zERC20 をアプリケーションにインテグレーションする方法
---

# インテグレーションガイド

{% hint style="info" %}
SDK を使いたい場合（プライベート送信・Scan・Wrap / Unwrap など）は [SDK ガイド](sdk/quickstart.md) を参照してください。このページはより低レベルなコントラクトインテグレーションとオラクル利用を対象としています
{% endhint %}

## 概要

zERC20 インテグレーションには3つの方法があります：

1. **トークンインテグレーション**：DeFiプロトコル・ウォレット・dApp で zERC20 を標準 ERC-20 トークンとして利用
2. **オラクルインテグレーション**：zERC20 の Transfer Merkle Tree をオンチェーンオラクルとして活用し、ZKP で転送履歴を検証
3. **セルフホスト型 Indexer**：最大限のプライバシー確保のために自前の Indexer ノードを実行し、送受信者の紐付け漏洩を防止

## 新しい zERC20 トークンを作成する

独自トークンの zERC20 バージョン（例：zUNI、zUSDT）を立ち上げたい場合は、[zerc20.io/integrate](https://zerc20.io/integrate) から zERC20 チームにお問い合わせください。デプロイと設定をサポートします。

## トークンインテグレーション

zERC20 は完全な ERC-20 互換です。ERC-20 トークンをサポートするあらゆるアプリケーションで zERC20 を使用できます。

### コントラクトアドレス

各チェーンのデプロイアドレスは [コントラクトアドレス](../reference/addresses.md) を参照してください。

### 標準 ERC-20 インターフェース

```javascript
interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}
```

### Wrap と Unwrap

`LiquidityManager` コントラクトを使って、原資産トークンと zERC20 を相互変換できます。

```javascript
interface ILiquidityManager {
    /// @notice Wrap underlying tokens to receive zERC20
    /// @param amount Amount of underlying tokens to wrap
    /// @param receiver Address to receive the zERC20 tokens
    /// @return amountOut Amount of zERC20 tokens received (includes reward)
    function wrap(uint256 amount, address receiver) external payable returns (uint256 amountOut);

    /// @notice Wrap with slippage protection
    /// @param amount Amount of underlying tokens to wrap
    /// @param minOut Minimum zERC20 tokens to receive (reverts if less)
    /// @param receiver Address to receive the zERC20 tokens
    /// @return amountOut Amount of zERC20 tokens received
    function wrapWithMinOut(uint256 amount, uint256 minOut, address receiver)
        external
        payable
        returns (uint256 amountOut);

    /// @notice Unwrap zERC20 to receive underlying tokens
    /// @param amount Amount of zERC20 tokens to unwrap
    /// @param receiver Address to receive the underlying tokens
    /// @return amountOut Amount of underlying tokens received (minus fee)
    function unwrap(uint256 amount, address receiver) external returns (uint256 amountOut);

    /// @notice Unwrap with slippage protection
    /// @param amount Amount of zERC20 tokens to unwrap
    /// @param minOut Minimum underlying tokens to receive (reverts if less)
    /// @param receiver Address to receive the underlying tokens
    /// @return amountOut Amount of underlying tokens received
    function unwrapWithMinOut(uint256 amount, uint256 minOut, address receiver)
        external
        returns (uint256 amountOut);

    /// @notice Get the reward amount for wrapping
    function quoteWrapReward(uint256 amount) external view returns (uint256 rewardAmount);

    /// @notice Get the fee amount for unwrapping
    function quoteUnwrapFee(uint256 amount) external view returns (uint256 feeAmount);
}
```

**使用例：**

```javascript
// Wrap: Convert 100 USDC to zUSDC
IERC20(usdc).approve(address(liquidityManager), 100e6);
uint256 zAmount = liquidityManager.wrap(100e6, msg.sender);

// Unwrap: Convert 100 zUSDC back to USDC
uint256 usdcAmount = liquidityManager.unwrap(100e6, msg.sender);
```

Wrapリワードと Unwrap手数料の詳細は [手数料とリワード](../users/fees-and-rewards.md) を参照してください。

## オラクルとしての zERC20

zERC20 は ZKP に適した Poseidon Hash 関数を用いて、全転送の完全な履歴を Merkle Tree として管理しています。外部開発者はこの Transfer Merkle Tree をオンチェーンオラクルとして活用し、転送履歴を検証できます。

### リーフ構造

Transfer Merkle Tree の各リーフは1件の転送を表します：

```
leaf_hash = Poseidon3(from, to, value)
```

| フィールド   | 型         | 説明                  |
| ------- | --------- | ------------------- |
| `from`  | `address` | 送信者アドレス（フィールド要素に変換） |
| `to`    | `address` | 受信者アドレス（フィールド要素に変換） |
| `value` | `uint256` | 転送金額（フィールド要素に変換）    |

### ツリー構造

Merkle Root には2種類あります：

| Root の種別                 | 説明                    | ツリーの高さ     |
| ------------------------ | --------------------- | ---------- |
| **Local Transfer Root**  | チェーンごとの転送 Merkle Root | 40         |
| **Global Transfer Root** | クロスチェーン集約 Merkle Root | 46（40 + 6） |

Global Transfer Tree は、全チェーンの Local Transfer Root を集約ツリー（高さ6・最大64チェーン対応）で集約して構築されます。

```
Global Transfer Tree Structure:

         [Global Root]
              │
    ┌─────────┴─────────┐
    │  Aggregation Tree │  (height: 6)
    │   (up to 64 chains)│
    └─────────┬─────────┘
              │
   ┌──────────┼──────────┐
   │          │          │
[Chain 0]  [Chain 1]  [Chain N]
   │          │          │
[Local     [Local     [Local
 Transfer   Transfer   Transfer
 Root]      Root]      Root]
   │          │          │
[Transfer  [Transfer  [Transfer
 Tree]      Tree]      Tree]
 (h:40)     (h:40)     (h:40)
```

### コントラクトから Merkle Root を読み取る

外部コントラクトは Verifier コントラクトから証明済み Merkle Root を照会できます：

```javascript
interface IVerifier {
    /// @notice Get the local transfer root for a given index
    /// @param index The tree index (increments with each proof)
    /// @return The local transfer Merkle root
    function provedTransferRoots(uint64 index) external view returns (uint256);

    /// @notice Get the global transfer root for a given index
    /// @param index The aggregation sequence number
    /// @return The global transfer Merkle root
    function globalTransferRoots(uint64 index) external view returns (uint256);
}
```

**使用例：**

Merkle Proof の検証は**スマートコントラクト上でオンチェーン**に行う方法と、**ZKP 回路を使ってオフチェーン**で行う方法があります。用途に応じて選択してください：

* **オンチェーン検証**：ガスコストが許容範囲内のシンプルなメンバーシップ証明に適しています
* **ZKP 検証**：プライバシー保護アプリケーションや、オンチェーンでは高コストになる複雑なロジックに最適です

```javascript
// On-chain verification example
contract MyContract {
    IVerifier public verifier;

    function verifyLocalTransfer(
        uint64 treeIndex,
        bytes32[] calldata siblings,
        uint64 leafIndex,
        address from,
        address to,
        uint256 value
    ) external view returns (bool) {
        uint256 expectedRoot = verifier.provedTransferRoots(treeIndex);
        // Verify Merkle proof against expectedRoot using Poseidon hash
        // ...
    }
}
```

### Poseidon Hash の互換性

zERC20 が使用する Poseidon Hash は [circomlib の Poseidon ライブラリ](https://github.com/iden3/circomlib/blob/master/circuits/poseidon.circom) と完全に互換性があります：

| 用途      | circomlib テンプレート | 説明                          |
| ------- | ---------------- | --------------------------- |
| リーフハッシュ | `Poseidon(3)`    | `Poseidon(from, to, value)` |
| ノードハッシュ | `Poseidon(2)`    | `Poseidon(left, right)`     |

この互換性により、開発者は circomlib を使ったカスタム ZK 回路を構築して、zERC20 の Transfer Merkle Tree に対するメンバーシップ証明を検証できます。

### Merkle Proof を取得する

#### Local Transfer Merkle Proof

Indexer ノードに照会して Local Transfer Merkle Proof を取得します：

**エンドポイント：** `POST /proofs`

**リクエスト：**

```json
{
  "chain_id": 1,
  "token_address": "0x...",
  "target_index": 100,
  "leaf_indices": [42, 43, 44]
}
```

**レスポンス：**

```json
[
  {
    "target_index": 100,
    "leaf_index": 42,
    "root": "0x...",
    "hash_chain": "0x...",
    "siblings": ["0x...", "0x...", ...]
  }
]
```

| フィールド          | 説明                             |
| -------------- | ------------------------------ |
| `target_index` | 証明対象のツリーインデックス（スナップショット）       |
| `leaf_index`   | ツリー内のリーフの位置                    |
| `root`         | target\_index 時点の Merkle Root  |
| `hash_chain`   | target\_index 時点の Hash Chain 値 |
| `siblings`     | 証明パスの40個の兄弟ハッシュの配列             |

**補助エンドポイント — ツリーインデックスを取得：**

`GET /tree-index?chain_id={chainId}&token_address={address}&transfer_root={root}`

指定した Merkle Root に対応するツリーインデックスを返します。

#### Global Transfer Merkle Proof

Global Merkle Proof は以下を連結して構築します：

1. **Local Transfer Merkle Proof**（40個の兄弟ハッシュ）
2. **集約ツリー Proof**（6個の兄弟ハッシュ）

**構築アルゴリズム：**

```
1. Fetch local_merkle_proof from indexer (40 siblings)
2. Determine aggregation_index from chain_id position in Hub contract
3. Compute aggregation_merkle_proof from Hub's AggregationRootUpdated event
4. Concatenate: global_proof = local_proof ++ aggregation_proof (46 siblings)
5. Compute global_leaf_index:
   global_leaf_index = (aggregation_index << 40) + local_leaf_index
```

**Hub コントラクトの集約状態：**

Hub コントラクトの `AggregationRootUpdated` イベントが、全 Local Root のスナップショットを提供します：

```javascript
event AggregationRootUpdated(
    uint256 indexed root,
    uint64 indexed aggSeq,
    uint256[] transferRootsSnapshot,
    uint64[] transferTreeIndicesSnapshot
);
```

## 自前の Indexer を実行する

送受信者の紐付け漏洩を防いで最大限のプライバシーを確保するには、自前の Indexer インスタンスを実行してください。

### Docker Compose

```bash
git clone https://github.com/kbizikav/zerc20.git
cd zerc20
docker compose up -d
```

### 設定

環境変数を設定します：

```bash
DATABASE_URL=postgres://...
RPC_URL=https://...
TOKENS_FILE_PATH=./config/tokens.json
```

完全なサービス設定はリポジトリルートの `docker-compose.yml` を参照してください。

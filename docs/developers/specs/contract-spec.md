# コントラクト仕様

## 概要

zERC20 コントラクトシステムは以下で構成されます：

- **zERC20**：プライバシー機能を備えた ERC-20 トークン
- **Verifier**：Proof 検証と Teleport の実行
- **Hub**：クロスチェーンルートの集約
- **LiquidityManager**：流動性の入出金ポリシー管理
- **Adaptor**：Stargate 経由のクロスチェーン出金
- **Fee Manager**（オフチェーン）：流動性ターゲットの動的調整

## zERC20

**ファイル**: `contracts/src/zERC20.sol`

ZKP（Zero-Knowledge Proof）検証のためにすべての転送をハッシュチェーンで追跡する、アップグレード可能な ERC-20 コントラクトです。

### 主な機能

- すべての転送に対して `IndexedTransfer(index, from, to, value)` イベントを発行
- SHA-256 の切り捨てハッシュチェーンを維持：`hashChain = SHA256(hashChain || from || to || value)[0:248]`
- Verifier が起動する Mint のために `teleport` を公開

### 関数

```javascript
// Called by Verifier after successful proof verification
function teleport(address to, uint256 value) external onlyVerifier;

// Called by the configured `minter` (typically LiquidityManager) for deposit/withdraw flows
function mint(address to, uint256 amount) external onlyMinter;
function burn(address from, uint256 amount) external onlyMinter;
```

### イベント

```javascript
event IndexedTransfer(uint256 indexed index, address indexed from, address indexed to, uint256 value);
event Teleport(address indexed to, uint256 value);
```

### 制約

- すべての転送量は ≤ 2^248 - 1 でなければならない（BN254 スカラーフィールドに収まる必要がある）
- 超過した場合は `ValueTooLarge` でリバート

## Verifier

**ファイル**: `contracts/src/Verifier.sol`

ZK Proof を検証し、Teleport を管理する LayerZero OApp です。

### 主な機能

1. Proof のアンカリング（基準点として記録）のためにハッシュチェーンのチェックポイントを記録
2. 転送ルートの遷移に Nova Proof を検証
3. 引き出しに Nova / Groth16 Proof を検証
4. LayerZero 経由で Hub にルートを中継

### 関数

```javascript
// Reserve the latest zERC20 (index, hashChain) checkpoint for proof anchoring
function reserveHashChain() external returns (uint64 index, uint256 hashChain);

// Prove a new transfer root (called by indexer)
function proveTransferRoot(bytes calldata proof) external;

// Batch withdrawal with Nova proof
function teleport(
    bool isGlobal,
    uint64 rootHint,
    GeneralRecipient calldata gr,
    bytes calldata proof
) external;

// Single withdrawal with Groth16 proof
function singleTeleport(
    bool isGlobal,
    uint64 rootHint,
    GeneralRecipient calldata gr,
    bytes calldata proof
) external;

// Relay local root to Hub
function relayTransferRoot(bytes calldata options) external payable;
function relayTransferRoot(bytes calldata options, address refundAddress) external payable;
```

### ステート

```javascript
mapping(uint64 => uint256) public reservedHashChains;     // index → hashChain snapshot (248-bit packed in uint256)
mapping(uint64 => uint256) public provedTransferRoots;    // index → merkle root (field element)
mapping(uint64 => uint256) public globalTransferRoots;    // aggSeq → global root (field element)
mapping(uint256 => uint256) public totalTeleported;       // recipientHash → total minted (cumulative)
```

### GeneralRecipient

引き出しを特定の送金先に紐付けます：

```javascript
struct GeneralRecipient {
    uint256 chainId;
    address addr;
    uint256 tweak;
}
// Hash: Poseidon(chainId, addr, tweak)
```

### 緊急時の対応

Proof の不整合（同一インデックスに対してルートが一致しない）が検出された場合、コントラクトは自動的に停止します。
再開するにはオーナーが検証者をローテートし、`deactivateEmergency` を呼び出す必要があります。

## Hub

**ファイル**: `contracts/src/Hub.sol`

クロスチェーン転送ルートの中央集約コントラクトです。

### 主な機能

- LayerZero 経由ですべてのチェーンの Verifier からルートを受信
- Poseidon ツリー（最大 64 リーフ）に集約
- グローバルルートをすべての Verifier にブロードキャスト

### 関数

```javascript
struct TokenInfo {
    uint64 chainId;
    uint32 eid;
    address verifier;
    address token;
}

// Register or update token metadata (owner only)
function registerToken(TokenInfo calldata info) external onlyOwner;
function updateToken(TokenInfo calldata info) external onlyOwner;

// Broadcast current aggregation to all chains
function broadcast(uint32[] calldata targetEids, bytes calldata lzOptions) external payable;
function broadcast(uint32[] calldata targetEids, bytes calldata lzOptions, address refundAddress) external payable;

// Estimate broadcast fee
function quoteBroadcast(uint32[] calldata targetEids, bytes calldata lzOptions) external view returns (uint256 totalNativeFee);
```

### イベント

```javascript
event AggregationRootUpdated(
    uint256 indexed root,
    uint64 indexed aggSeq,
    uint256[] transferRootsSnapshot,
    uint64[] transferTreeIndicesSnapshot
);
```

## LiquidityManager

**ファイル**: `contracts/src/liquidity/LiquidityManager.sol`

インセンティブカーブを用いて流動性の入出金を管理するコントラクトです。

### 主な機能

- 原資産トークンを zERC20 に Wrap
- zERC20 を原資産トークンに Unwrap
- 流動性ターゲットに基づいたインセンティブ手数料の適用

### 関数

```javascript
// Wrap underlying → zERC20
function wrap(uint256 amount, address receiver) external payable returns (uint256 amountOut);

// Unwrap zERC20 → underlying
function unwrap(uint256 amount, address receiver) external returns (uint256 amountOut);

// Quote incentive amounts (view)
function quoteWrapReward(uint256 amount) external view returns (uint256 rewardAmount);
function quoteUnwrapFee(uint256 amount) external view returns (uint256 feeAmount);
```

### インセンティブカーブ（Incentive Curve）

線形インセンティブ密度：`density(x) = k * (1 - x / T)`（`x < T` の場合）

- ターゲット未満：Wrap に報酬、Unwrap に手数料を適用
- ターゲット以上：報酬・手数料なし
- 手数料は `feeSurplus` に蓄積され、将来のインセンティブとして使用

## Fee Manager

**ファイル**: `fee-manager/`（オフチェーンサービス）

すべてのチェーンの `LiquidityManager` コントラクトに対して `targetLiquidity` を動的に調整する定期メンテナンスワーカーです。

### 目的

Fee Manager はすべてのチェーン間で流動性の均衡を保ち、手数料パラメータを自動的に更新します：

```
targetLiquidity = total_underlying_balance / number_of_chains * target_ratio_bps / 10000
```

これにより手動でのパラメータ調整が不要になり、チェーン間での流動性の移動に合わせてインセンティブが自動調整されます。

### 動作の流れ

1. **残高の取得**：各 LiquidityManager から原資産トークンの残高（`balance - feeSurplus`）を取得
2. **ターゲットの計算**：`targetLiquidity = total / chain_count * target_ratio_bps / 10000` を計算
3. **パラメータの更新**：各 LiquidityManager に対して `setFeeParams({ targetLiquidity, k })` を呼び出す
4. **繰り返し**：設定したインターバル（デフォルト：1 時間）スリープ後に繰り返す

### FeeParams 構造体

```javascript
struct FeeParams {
    uint256 targetLiquidity;  // Liquidity level where incentives fade to zero
    uint256 k;                // Incentive strength coefficient (basis points)
}
```

- `targetLiquidity`：Wrap 報酬と Unwrap 手数料がゼロに近づく流動性レベル
- `k`：インセンティブカーブの傾きを制御（値が高いほど、流動性がターゲットから乖離した際のインセンティブが強くなる）

### 設定

| 変数 | デフォルト | 説明 |
|------|----------|------|
| `FEE_MANAGER_PRIVATE_KEY` | — | 各 LiquidityManager の `FEE_MANAGER_ROLE` を持つ秘密鍵 |
| `FEE_MANAGER_INTERVAL_SECS` | `3600` | 更新間隔（秒） |
| `FEE_MANAGER_K_BPS` | `1000` | インセンティブ係数 k（ベーシスポイント、1000 = 10%） |
| `FEE_MANAGER_TARGET_RATIO_BPS` | `8000` | ターゲット流動性比率（ベーシスポイント、8000 = 80%） |

### ネイティブトークンのサポート

Fee Manager は LiquidityManager がネイティブ ETH を使用しているか ERC20 を使用しているかを自動的に検出します：
- **ネイティブ ETH**：ERC-7528 センチネルアドレス `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` で検出
- **ERC20**：原資産トークンの `balanceOf()` で残高を取得

### 権限

Fee Manager は各 LiquidityManager に対して `FEE_MANAGER_ROLE` が必要です：

```javascript
liquidityManager.grantRole(FEE_MANAGER_ROLE, feeManagerAddress);
```

## 主要フロー

### 1. 転送ルートの証明

```
1. zERC20 の転送が IndexedTransfer を発行し、hashChain を更新
2. インデクサがイベントを同期し、Merkle Tree を構築
3. インデクサが Verifier.reserveHashChain(index) を呼び出す
4. インデクサがルート遷移の Nova Proof を生成
5. インデクサが Verifier.proveTransferRoot() を呼び出す
6. Verifier が新しいルートを provedTransferRoots に格納
```

### 2. ローカル Teleport

```
1. ユーザーが zERC20 をバーンアドレスに送信
2. 受信者がバーンアドレスの所有権を証明する ZKP を生成
3. 受信者が Verifier.teleport() または singleTeleport() を呼び出す
4. Verifier が provedTransferRoots に対して Proof を検証
5. Verifier が zERC20.teleport() を呼び出してトークンを Mint
```

### 3. グローバル Teleport（クロスチェーン）

```
1. Verifier.relayTransferRoot() がローカルルートを Hub に送信
2. Hub.broadcast() が集約してグローバルルートを送信
3. 受信者が isGlobal=true で ZKP を生成
4. Verifier が globalTransferRoots に対して検証
5. Verifier が zERC20.teleport() 経由でトークンを Mint
```

## セキュリティに関する注記

- **値の範囲**：すべての値を 248 ビット制限で検証
- **二重支払い防止**：`totalTeleported` が受信者ごとの累積 Mint 量を追跡
- **LayerZero セキュリティ**：既知のエンドポイントからのメッセージのみ受け付ける
- **アップグレードの安全性**：UUPS パターンとオーナー専用アップグレードを採用

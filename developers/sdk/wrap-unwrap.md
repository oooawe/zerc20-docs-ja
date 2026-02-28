# Wrap と Unwrap

`LiquidityManager` コントラクトは原資産トークン（USDC・ETH・BNB）と対応する zERC20（zUSDC・zETH・zBNB）を相互変換します。Wrap では原資産トークンをデポジットして zERC20 をミントし、プロトコルはデポジットを促す**リワード**インセンティブを付与します。Unwrap では zERC20 をバーンして原資産トークン（手数料を差し引いた額）を返します。リワードと手数料はどちらも、プールの残高を健全に保つための流動性目標値カーブによって決まります。

## Wrap

原資産トークンを対応する zERC20 に変換します。

```typescript
function wrapWithLiquidityManager(
  params: WrapWithLiquidityManagerParams,
): Promise<LiquidityActionResult>;
```

### WrapWithLiquidityManagerParams

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `walletClient` | `WalletClient` | トランザクションの署名・送信に使用するウォレットクライアント |
| `publicClient?` | `PublicClient` | 読み取り呼び出し用のオプションのパブリッククライアント（省略時は `walletClient` から導出） |
| `liquidityManagerAddress` | `string` | LiquidityManager コントラクトのアドレス |
| `amount` | `bigint \| number \| string` | Wrap する原資産トークンの金額（基本単位） |
| `underlyingTokenAddress?` | `string` | 原資産トークンアドレスのオーバーライド（省略時はコントラクトから自動読み取り） |
| `recipient?` | `string` | ミントされた zERC20 の受取人（デフォルトは送信者） |
| `feeOverrides?` | `FeeOverrides` | ガス価格・ガスリミットのオプションオーバーライド |

**ネイティブ ETH の Wrap** の場合、SDK は自動的に `msg.value` を送信します — ERC-20 の承認は不要です。ERC-20 の原資産トークンの場合、SDK は現在の allowance を確認し、必要に応じて承認トランザクションを先に送信します。

### LiquidityActionResult

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `transactionHash` | `Hex` | Wrap（または Unwrap）トランザクションのハッシュ |
| `approvalTransactionHash?` | `Hex` | ERC-20 承認トランザクションのハッシュ（送信した場合のみ） |

### 見積もり

Wrap 前にオンチェーンの `quoteWrapReward` を照会してリワードをプレビューできます：

```typescript
import { getLiquidityManagerContract } from "zerc20-client-sdk";

const manager = getLiquidityManagerContract(entry.liquidityManagerAddress, publicClient);
const reward = await manager.read.quoteWrapReward([100_000_000n]);
console.log("Reward:", reward);
```

### コード例 — 100 USDC を zUSDC に Wrap する

```typescript
import {
  normalizeTokens,
  findTokenByChain,
  wrapWithLiquidityManager,
} from "zerc20-client-sdk";

// 1. トークンレジストリからアドレスを解決
const tokensFile = await import("./tokens.json");
const { tokens } = normalizeTokens(tokensFile);
const entry = findTokenByChain(tokens, 42161n); // Arbitrum

// 2. Wrap を実行
const result = await wrapWithLiquidityManager({
  walletClient,
  liquidityManagerAddress: entry.liquidityManagerAddress,
  amount: 100_000_000n, // 100 USDC (6 decimals)
});
console.log("Wrap tx:", result.transactionHash);
if (result.approvalTransactionHash) {
  console.log("Approval tx:", result.approvalTransactionHash);
}
```

## Unwrap

zERC20 をバーンして原資産トークンを受け取ります。

```typescript
function unwrapWithLiquidityManager(
  params: LocalUnwrapParams,
): Promise<LiquidityActionResult>;
```

### LocalUnwrapParams

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `walletClient` | `WalletClient` | トランザクションの署名・送信に使用するウォレットクライアント |
| `publicClient?` | `PublicClient` | 読み取り呼び出し用のオプションのパブリッククライアント |
| `liquidityManagerAddress` | `Address` | LiquidityManager コントラクトのアドレス |
| `zerc20TokenAddress` | `Address` | バーンする zERC20 トークンのアドレス |
| `amount` | `bigint \| number \| string` | Unwrap する zERC20 の金額（基本単位） |
| `recipient?` | `Address` | 原資産トークンの受取人（デフォルトは送信者） |
| `feeOverrides?` | `FeeOverrides` | ガス価格・ガスリミットのオプションオーバーライド |
| `minAmountOut?` | `bigint` | 受け取る原資産の最低額。送信前に**プリフライトスリッページチェック**を実施 |

`minAmountOut` を設定すると、SDK はまずオフチェーンで Unwrap をシミュレートします。予想出力がしきい値を下回る場合、オンチェーンでリバートするトランザクションを送信する代わりに、即座にスリッページエラーを返します。

### 見積もり

```typescript
function quoteLocalUnwrap(
  params: LocalUnwrapQuoteParams,
): Promise<LocalUnwrapQuote>;
```

`LocalUnwrapQuote` の内容：

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `fee` | `bigint` | Unwrap 金額から差し引かれる手数料 |
| `expectedOut` | `bigint` | 受け取れる原資産トークン（`amount - fee`） |

## クロスチェーンUnwrap

現在のチェーンで zERC20 を Unwrap し、Stargate ブリッジ経由で原資産トークンを別のチェーンにブリッジします。

### 見積もりを取得する

```typescript
function buildCrossUnwrapQuote(
  params: CrossUnwrapQuoteParams,
): Promise<CrossUnwrapQuote>;
```

`CrossUnwrapQuote` の内容：

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `tokenUnwrapFee` | `bigint` | LiquidityManager の Unwrap手数料（ローカル Unwrap と同じ） |
| `nativeBridgeFee` | `bigint` | Stargate / LayerZero が要求するネイティブガス手数料 |
| `tokenBridgeFee` | `bigint` | トークン建てのブリッジ手数料 |
| `sendNativeFee` | `bigint` | トランザクションに付ける必要がある合計ネイティブ値 |
| `expectedOut` | `bigint` | 目的チェーンで受け取れる原資産トークンの見積もり |
| `sendParam` | `SendParam` | エンコードされた Stargate 送信パラメータ |
| `bridgeRequest` | `BridgeRequest` | 送信準備が完了したブリッジリクエスト構造体 |

### 実行する

```typescript
function sendCrossUnwrap(
  params: CrossUnwrapSendParams,
): Promise<LiquidityActionResult>;
```

`CrossUnwrapQuote` に `walletClient` とコントラクトアドレスを渡します。SDK は自動的に `sendNativeFee` を `msg.value` として付与します。

## 手数料の見積もり

トランザクション送信**前**に見積もりヘルパーでコストを確認します：

| 操作 | ヘルパー | 主なフィールド |
|-----------|--------|------------|
| ローカル Wrap | `getLiquidityManagerContract().read.quoteWrapReward()` | リワード金額 |
| ローカル Unwrap | `quoteLocalUnwrap()` | `fee`、`expectedOut` |
| クロスチェーンUnwrap | `buildCrossUnwrapQuote()` | `tokenUnwrapFee`、`nativeBridgeFee`、`tokenBridgeFee`、`expectedOut` |

### プールの残高

LiquidityManager プールの現在の状態を確認します：

```typescript
function fetchLiquidityManagerBalances(
  params: FetchLiquidityBalancesParams,
): Promise<LiquidityBalances>;
```

`LiquidityBalances` の内容：

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `underlyingAddress` | `Address` | 原資産 ERC-20 のアドレス（ネイティブ ETH の場合はゼロアドレス） |
| `underlyingBalance` | `bigint` | プールが保有する原資産トークンの現在残高 |
| `underlyingDecimals` | `number` | 原資産トークンのデシマル |
| `zerc20Balance` | `bigint` | プールが保有する zERC20 の現在残高 |
| `zerc20Decimals` | `number` | zERC20 トークンのデシマル |

## エラーハンドリング

| シナリオ | 動作 |
|----------|-----------|
| **担保不足のプール** | Wrap リワードがゼロになるか負になる場合があります（ボーナスなし）。トランザクション自体は成功しますが、デポジットした金額より少ない zERC20 を受け取ります。 |
| **スリッページ超過** | `minAmountOut` が設定されていてシミュレーション結果がしきい値を下回る場合、SDK はトランザクションを送信せずに `"Price changed beyond slippage tolerance"` をスローします。 |
| **承認の拒否** | SDK は Wrap / Unwrap 前に自動で ERC-20 `approve` トランザクションを送信します。ウォレットが承認ポップアップを拒否すると、操作全体が失敗します。 |

## 次のステップ

- [SDK クイックスタート](quickstart.md) — インストールとはじめてのプライベート送信
- [受け取り](receiving.md) — 受け取った転送の Scan と請求
- [Proof 生成](proof-generation.md) — Teleport 用の ZKP 生成

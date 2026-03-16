---
hidden: true
icon: coins
description: サポートトークンの読み込みと管理
---

# Token Registry

SDK にはトークンレジストリモジュールが含まれており、zERC20 トークン設定の読み込み・正規化・クエリができます。トークン設定ファイルには、各トークンのデプロイ済みコントラクト、チェーン ID、RPC エンドポイントが記述されています。

## トークンの読み込み

### 圧縮データからの読み込み

本番フロントエンドでは、トークンを Base64 エンコード済み gzip 文字列として配布しています。`TokensCacheManager.load()` で展開・正規化できます。

```typescript
import { TokensCacheManager } from "zerc20-client-sdk";

const cache = new TokensCacheManager();
const { hub, tokens } = await cache.load(compressedString, {
  cacheKey: "zusdc-main",
});
```

### 生の JSON からの読み込み

`tokens.json` ファイルがある場合（例：独自デプロイ環境）は、`normalizeTokens` を使います。

```typescript
import { normalizeTokens } from "zerc20-client-sdk";

const tokensFile = await import("./tokens.json");
const { hub, tokens } = normalizeTokens(tokensFile);
```

### RPC URL のオーバーライド

`normalizeTokensWithOverrides` を使うと、実行時にデフォルトの RPC URL を差し替えられます。カスタム RPC エンドポイント、プライベート RPC、フォールバックプロバイダーの利用に便利です。

```typescript
import { normalizeTokensWithOverrides } from "zerc20-client-sdk";

const tokensFile = await import("./tokens.json");
const { hub, tokens } = normalizeTokensWithOverrides(tokensFile, {
  tokens: {
    "Arbitrum One": ["https://my-rpc.example.com/arb"],
    "Ethereum": ["https://my-rpc.example.com/eth"],
    "Base": ["https://my-rpc.example.com/base"],
  },
  hub: ["https://my-rpc.example.com/base"],
});
```

`overrides.tokens` のキーは、トークン設定の `label` フィールドと照合されます。一致するエントリのみ RPC URL が置き換わります。

## トークンの検索

```typescript
import { findTokenByChain } from "zerc20-client-sdk";

const entry = findTokenByChain(tokens, 42161n); // Arbitrum
// entry.tokenAddress        — zERC20 コントラクトアドレス
// entry.verifierAddress     — Verifier コントラクトアドレス
// entry.liquidityManagerAddress — LiquidityManager アドレス
// entry.rpcUrls             — RPC エンドポイント URL
// entry.chainId             — チェーン ID (bigint)
```

`findTokenByChain` は、指定したチェーン ID に一致するトークンがない場合にエラーをスローします。

## 型定義

### NormalizedTokens

```typescript
interface NormalizedTokens {
  hub?: HubEntry;       // Hub コントラクトのメタデータ（シングルチェーン構成では undefined の場合あり）
  tokens: TokenEntry[]; // チェーンごとのトークンエントリ
  raw: TokensFile;      // 元の入力データ（参照用に保持）
}
```

### TokenEntry

各 `TokenEntry` は、特定チェーン上のトークンデプロイメントを表します。

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `tokenAddress` | `string` | zERC20 コントラクトアドレス |
| `verifierAddress` | `string` | Verifier コントラクトアドレス |
| `minterAddress` | `string \| undefined` | Minter コントラクトアドレス（任意） |
| `liquidityManagerAddress` | `string \| undefined` | LiquidityManager コントラクトアドレス（任意） |
| `adaptorAddress` | `string \| undefined` | Adaptor コントラクトアドレス（クロスチェーン Unwrap 用、任意） |
| `eid` | `number \| undefined` | LayerZero エンドポイント ID（任意） |
| `chainId` | `bigint` | EVM チェーン ID |
| `deployedBlockNumber` | `bigint` | デプロイブロック番号 |
| `label` | `string` | 人間が読めるチェーン名（例：`"Arbitrum One"`） |
| `rpcUrls` | `string[]` | このチェーンの RPC エンドポイント URL |
| `legacyTx` | `boolean` | このチェーンでレガシー tx フォーマットを使うかどうか |

### HubEntry

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `hubAddress` | `string` | Hub コントラクトアドレス |
| `chainId` | `bigint` | Hub がデプロイされているチェーン ID |
| `eid` | `number \| undefined` | LayerZero エンドポイント ID（任意） |
| `rpcUrls` | `string[]` | RPC エンドポイント URL |

### RpcOverrides

```typescript
interface RpcOverrides {
  tokens?: Record<string, string[]>;
  hub?: string[];
}
```

## 関数シグネチャ

```typescript
function normalizeTokens(file: TokensFile): NormalizedTokens;

function normalizeTokensWithOverrides(
  file: TokensFile,
  overrides?: RpcOverrides,
): NormalizedTokens;

function findTokenByChain(
  tokens: readonly TokenEntry[],
  chainId: bigint,
): TokenEntry;
```

## 関連ページ

- [SDK クイックスタート](quickstart.md) — インストールと最初のトークン読み込み
- [コントラクトアドレス](../../reference/addresses.md) — チェーンごとのデプロイ済みアドレス

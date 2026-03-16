---
icon: rocket
description: zerc20-client-sdk を使って TypeScript アプリケーションからプライベート zERC20 転送（Private Transfer）を送信する方法
---

# SDK ガイド

## 前提条件

| 要件            | 詳細                                  |
| ------------- | ----------------------------------- |
| **Node.js**   | >= 18                               |
| **EVM ウォレット** | メッセージに署名できるウォレット（MetaMask、Rabby など） |
| **サポートチェーン**  | Ethereum、Arbitrum、または Base          |

## インストール

```bash
npm install zerc20-client-sdk
```

## SDK を初期化する

`createSdk()` を呼び出して `Zerc20Sdk` インスタンスを取得します。すべてのオプションはオプショナルで、省略した場合は適切なデフォルト値が使用されます。

```typescript
import { createSdk } from "zerc20-client-sdk";

const sdk = createSdk();
```

`Zerc20SdkOptions` には以下のオプションフィールドを指定できます：

| フィールド            | 型                      | 説明                         |
| ---------------- | ---------------------- | -------------------------- |
| `wasm`           | `WasmRuntime`          | カスタム WASM ランタイム（省略時は自動検出）  |
| `teleportProofs` | `TeleportProofClient`  | カスタム Teleport Proof クライアント |
| `decider`        | `HttpDeciderClient`    | Decider Prover エンドポイント     |
| `stealth`        | `StealthClientFactory` | カスタムステルスクライアントファクトリ        |

返された `Zerc20Sdk` は以下のフィールドに直接アクセスできます：

```typescript
interface Zerc20Sdk {
  wasm: WasmRuntime;
  teleportProofs: TeleportProofClient;
  decider?: HttpDeciderClient;
  stealth: StealthClientFactory;
}
```

## ICP に接続する

`StealthCanisterClient` を作成して ICP のストレージ canister・Key Manager canister とやり取りします：

```typescript
import { HttpAgent } from "@dfinity/agent";

const agent = await HttpAgent.create({ host: "https://icp-api.io" });

const stealthClient = sdk.createStealthClient({
  agent,
  storageCanisterId: "your-storage-canister-id",
  keyManagerCanisterId: "your-key-manager-canister-id",
});
```

`agent`・`storageCanisterId`・`keyManagerCanisterId` フィールドは**必須**です。いずれかが欠けている場合（かつファクトリのデフォルトが設定されていない場合）、`createStealthClient()` はエラーをスローします。

```typescript
sdk.createStealthClient(config?: Partial<StealthClientConfig>): StealthCanisterClient
```

## EVM プロバイダー

SDK はライブラリ非依存のプロバイダーインターフェースを使っており、viem 以外のライブラリとも連携できます。

| インターフェース | 用途 | viem の対応クラス |
|-----------|---------|-----------------|
| `EvmReadProvider` | コントラクト読み取り・残高クエリ・手数料推定 | `PublicClient` |
| `EvmWriteProvider` | トランザクションの署名・送信 | `WalletClient` |

viem の `PublicClient` と `WalletClient` はこれらのインターフェースを構造的に満たすため、アダプターは不要です。他のライブラリを使う場合は、必要なメソッドを実装する薄いアダプターを用意します。

```typescript
import type { EvmReadProvider, EvmWriteProvider } from "zerc20-client-sdk";
import { createPublicClient, createWalletClient, custom, http } from "viem";
import { arbitrum } from "viem/chains";

// EvmReadProvider / EvmWriteProvider としてそのまま使えます
const readProvider = createPublicClient({ chain: arbitrum, transport: http() });
const writeProvider = createWalletClient({ chain: arbitrum, transport: custom(window.ethereum!) });
```

## トークンを読み込む

SDK はトークン設定を読み込む複数の方法を提供しています：

### 方法A：`TokensCacheManager` で圧縮データから読み込む

`TokensCacheManager.load()` はトークン設定 JSON を含む **Base64 エンコードされた gzip** 文字列を受け取ります。これはプロダクションのフロントエンドで使われている形式です。

```typescript
import { TokensCacheManager, findTokenByChain } from "zerc20-client-sdk";

// `compressed` は tokens.json の Base64 エンコード gzip 文字列
const cache = new TokensCacheManager();
const { hub, tokens } = await cache.load(compressedTokensString, {
  cacheKey: "zusdc-main",
});
```

### 方法B：`normalizeTokens` で生の JSON から読み込む

自前のデプロイからの `tokens.json` ファイルがある場合は、`normalizeTokens()` を使います：

```typescript
import { normalizeTokens, findTokenByChain } from "zerc20-client-sdk";

const tokensFile = await import("./tokens.json");
const { hub, tokens } = normalizeTokens(tokensFile);
// hub?: HubEntry           -- Hub コントラクトのメタデータ（Base チェーン）
// tokens: TokenEntry[]     -- チェーンごとのトークンエントリ
```

### 方法C：RPC URL をオーバーライドする

`normalizeTokensWithOverrides()` を使うと、実行時にデフォルトの RPC URL を差し替えられます：

```typescript
import { normalizeTokensWithOverrides, findTokenByChain } from "zerc20-client-sdk";

const tokensFile = await import("./tokens.json");
const { hub, tokens } = normalizeTokensWithOverrides(tokensFile, {
  tokens: {
    "Arbitrum One": ["https://my-rpc.example.com/arb"],
    "Ethereum": ["https://my-rpc.example.com/eth"],
  },
  hub: ["https://my-rpc.example.com/base"],
});
```

### トークンの検索

```typescript
// Arbitrum（チェーン ID 42161）のトークンエントリを取得
const entry = findTokenByChain(tokens, 42161n);
// entry.tokenAddress, entry.liquidityManagerAddress, etc.
```

トークン API の詳細は [Token Registry](token-registry.md) を参照してください。

### 型シグネチャ

```typescript
function loadTokens(compressed: string, options?: LoadTokensOptions): Promise<NormalizedTokens>;
function normalizeTokens(file: TokensFile): NormalizedTokens;

interface NormalizedTokens {
  hub?: HubEntry;
  tokens: TokenEntry[];
  raw: TokensFile;
}

function findTokenByChain(
  tokens: readonly TokenEntry[],
  chainId: bigint,
): TokenEntry;

function createProviderForToken(entry: TokenEntry): PublicClient;
```

## はじめてのプライベート送信

以下はエンドツーエンドの簡潔なコード例です。各ヘルパーの詳細は [プライベート送信](private-send.md) ページを参照してください。

```typescript
import {
  createSdk,
  normalizeTokens,
  findTokenByChain,
  createProviderForToken,
  getSeedMessage,
  preparePrivateSend,
  submitPrivateSendAnnouncement,
} from "zerc20-client-sdk";
import { encodeFunctionData, erc20Abi, keccak256, toBytes } from "viem";
import { HttpAgent } from "@dfinity/agent";

// 1. 初期化
const sdk = createSdk();
const agent = await HttpAgent.create({ host: "https://icp-api.io" });
const client = sdk.createStealthClient({
  agent,
  storageCanisterId: "your-storage-canister-id",
  keyManagerCanisterId: "your-key-manager-canister-id",
});
const tokensFile = await import("./tokens.json");
const { tokens } = normalizeTokens(tokensFile);
const entry = findTokenByChain(tokens, 42161n);   // Arbitrum

// 2. ウォレット署名から seed を導出（32 バイトにハッシュ化）
const seedMsg = await getSeedMessage();
const signature = await walletClient.signMessage({ message: seedMsg });
const seedHex = keccak256(toBytes(signature));

// 3. プライベート送信を準備
const preparation = await preparePrivateSend({
  client,
  recipientAddress: "0xRecipient...",
  recipientChainId: 42161n,
  seedHex,
});

// 4. バーンアドレスに zERC20 を送金
const txHash = await walletClient.sendTransaction({
  to: entry.tokenAddress,
  data: encodeFunctionData({
    abi: erc20Abi,
    functionName: "transfer",
    args: [preparation.burnAddress, 100_000_000n], // 100 zUSDC
  }),
});

// 5. 暗号化アナウンスを送信
const result = await submitPrivateSendAnnouncement({
  client,
  preparation,
});
console.log("Announcement submitted:", result);
```

## 次のステップ

* [プライベート送信](private-send.md) — 各ステップの詳細な説明
* [受け取り](receiving.md) — 受け取った転送の Scan と請求
* [インテグレーションガイド](../integration.md) — オンチェーンオラクルとセルフホスト型 Indexer
* [コントラクトアドレス](../../reference/addresses.md) — チェーンごとのデプロイ済みアドレス

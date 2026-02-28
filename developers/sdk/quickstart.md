# SDK クイックスタート

`zerc20-client-sdk` を使って TypeScript アプリケーションからプライベート zERC20 転送（Private Transfer）を送信する方法を説明します。

## 前提条件

| 要件 | 詳細 |
|-------------|---------|
| **Node.js** | >= 18 |
| **EVM ウォレット** | メッセージに署名できるウォレット（MetaMask、Rabby など） |
| **サポートチェーン** | Ethereum、Arbitrum、または Base |

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

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `wasm` | `WasmRuntime` | カスタム WASM ランタイム（省略時は自動検出） |
| `teleportProofs` | `TeleportProofClient` | カスタム Teleport Proof クライアント |
| `decider` | `HttpDeciderClient` | Decider Prover エンドポイント |
| `stealth` | `StealthClientFactory` | カスタムステルスクライアントファクトリ |

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

## トークンを読み込む

SDK はトークン設定を読み込む2つの方法を提供しています：

### 方法A：`loadTokens` で圧縮データから読み込む

`loadTokens(compressed)` はトークン設定 JSON を含む **Base64 エンコードされた gzip** 文字列を受け取ります。これはプロダクションのフロントエンドで使用されている形式です。

```typescript
import {
  loadTokens,
  findTokenByChain,
  createProviderForToken,
} from "zerc20-client-sdk";

// `compressed` は tokens.json の Base64 エンコード gzip 文字列
const { hub, tokens } = await loadTokens(compressedTokensString);
```

### 方法B：`normalizeTokens` で生の JSON から読み込む

自前のデプロイからの `tokens.json` ファイルがある場合は、代わりに `normalizeTokens()` を使います：

```typescript
import {
  normalizeTokens,
  findTokenByChain,
  createProviderForToken,
} from "zerc20-client-sdk";

const tokensFile = await import("./tokens.json");
const { hub, tokens } = normalizeTokens(tokensFile);
// hub?: HubEntry           -- Hub コントラクトのメタデータ（Base チェーン）
// tokens: TokenEntry[]     -- チェーンごとのトークンエントリ
```

### トークンの検索とプロバイダーの作成

```typescript
// Arbitrum（チェーン ID 42161）のトークンエントリを取得
const entry = findTokenByChain(tokens, 42161n);

// そのチェーン用に設定された viem PublicClient を作成
const publicClient = createProviderForToken(entry);
```

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

- [プライベート送信](private-send.md) — 各ステップの詳細な説明
- [受け取り](receiving.md) — 受け取った転送の Scan と請求
- [インテグレーションガイド](../integration.md) — オンチェーンオラクルとセルフホスト型 Indexer
- [コントラクトアドレス](../../reference/addresses.md) — チェーンごとのデプロイ済みアドレス

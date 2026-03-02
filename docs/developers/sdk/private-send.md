# プライベート送信

このページでは、SDK を使ったプライベート zERC20 転送（Private Transfer）の各ステップを詳しく説明します。

## 仕組み

プライベート送信は、オンチェーンとオフチェーンにまたがる3つのフェーズで処理されます：

1. **バーンアドレス（Burn Address）の導出** — 送信者が受信者のアドレスとランダムな secret から、Poseidon Hash（16ビットの Proof-of-Work チェック付き）を使って決定論的なバーンアドレスを計算します。
2. **バーンアドレスへの zERC20 送金** — 標準の ERC-20 `transfer` でトークンをバーンアドレスに送ります。Indexer が転送リーフを記録します。
3. **暗号化アナウンスの送信** — 送信者が転送メタデータ（secret・金額など）を ICP canister 経由で暗号化し、受信者だけが復号して後から資金を請求できるようにします。

> 受信者はその後 ICP ストレージ canister を Scan し、アナウンスを復号して ZKP（Zero-Knowledge Proof）を生成し、`Verifier.teleport()` 経由で同額の zERC20 をミントできます。

## ステップ1：Seed を導出する

すべてのプライベート送信は **seed** から始まります。seed はウォレットで署名したメッセージから決定論的にステルス鍵を導出します。

```typescript
import { getSeedMessage } from "zerc20-client-sdk";
import { keccak256, toBytes } from "viem";

// getSeedMessage() は async で、ウォレットが署名する人間可読な文字列を返す
const message = await getSeedMessage();
const signature = await walletClient.signMessage({ message });

// 65 バイトの署名を 32 バイトにハッシュ化 — SDK は 32 バイトの hex seed を要求する
const seedHex = keccak256(toBytes(signature));
```

`getSeedMessage()` は `Promise<string>` を返す非同期関数です。ウォレット署名（65バイト）は `keccak256` でハッシュ化して32バイトの hex 文字列（`seedHex`）にする必要があります。SDK は `seedHex` が正確に32バイトであることを検証し、そうでなければエラーをスローします。

## ステップ2：プライベート送信を準備する

`preparePrivateSend()` を呼び出してバーンアドレスを導出し、secret を生成し、暗号化アナウンスのペイロードを構築します。

```typescript
import { preparePrivateSend } from "zerc20-client-sdk";

const preparation = await preparePrivateSend({
  client,                               // StealthCanisterClient
  recipientAddress: "0xRecipient...",   // 受信者の EVM アドレス
  recipientChainId: 42161n,             // 受信者が請求するチェーン ID
  seedHex,                              // 32 バイト hex seed（ステップ1の署名を keccak256 したもの）
});
```

### パラメータ

`preparePrivateSend` は `PreparePrivateSendParams` オブジェクトを受け取ります：

| フィールド | 型 | 必須 | 説明 |
|-------|------|----------|-------------|
| `client` | `StealthCanisterClient` | 必須 | `sdk.createStealthClient()` で作成した ICP ステルスクライアント |
| `recipientAddress` | `string` | 必須 | 受信者の EVM アドレス |
| `recipientChainId` | `number \| bigint` | 必須 | 受信者が請求するチェーン ID |
| `seedHex` | `string` | 必須 | 32 バイトの hex 文字列（ステップ1のウォレット署名を `keccak256` したもの） |
| `paymentAdviceIdHex` | `string` | オプション | 支払い通知の識別子 |
| `vetkdKeyIdName` | `string` | オプション | VetKD キー ID 名のオーバーライド |

### 戻り値

`preparePrivateSend` は `PreparedPrivateSend` オブジェクトを返します：

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `burnAddress` | `string` | zERC20 を送金する決定論的アドレス |
| `burnPayload` | `Uint8Array` | シリアライズされたバーンデータ |
| `secret` | `bigint` | バーンアドレスにバインドされたランダムな secret |
| `tweak` | `bigint` | Poseidon 導出の tweak 値 |
| `generalRecipient` | `string` | 汎用化された受信者識別子 |
| `announcement` | `object` | 送信準備が完了した暗号化アナウンス |
| `sessionKey` | `Uint8Array` | 一時的なセッションキー |
| `paymentAdviceId` | `string` | 解決済みの支払い通知識別子 |
| `paymentAdviceIdBytes` | `Uint8Array` | バイト配列形式の支払い通知識別子 |

### シグネチャ

```typescript
function preparePrivateSend(
  params: PreparePrivateSendParams,
): Promise<PreparedPrivateSend>;
```

## ステップ3：バーンアドレスに zERC20 を送金する

任意の EVM ライブラリ（viem・ethers など）を使って、標準の ERC-20 `transfer` で `preparation.burnAddress` にトークンを送金します。

```typescript
import { encodeFunctionData, erc20Abi } from "viem";

const txHash = await walletClient.sendTransaction({
  to: tokenAddress,   // 送信者のチェーン上の zERC20 コントラクトアドレス
  data: encodeFunctionData({
    abi: erc20Abi,
    functionName: "transfer",
    args: [preparation.burnAddress, amount],
  }),
});

// トランザクションの確認を待機
await publicClient.waitForTransactionReceipt({ hash: txHash });
```

> **📃 Note：** 送金金額はアナウンスにエンコードされません。Indexer がオンチェーンのイベントから金額を検出します。1回のトランザクションで任意の金額を送金できます。

## ステップ4：アナウンスを送信する

オンチェーンの転送が確認された後、暗号化アナウンスを ICP ストレージ canister に送信して、受信者が転送を発見できるようにします。

```typescript
import { submitPrivateSendAnnouncement } from "zerc20-client-sdk";

const result = await submitPrivateSendAnnouncement({
  client,       // StealthCanisterClient
  preparation,  // ステップ2の PreparedPrivateSend
  tag: "myApp", // オプション：アプリケーションレベルのタグ
});
```

### パラメータ

`submitPrivateSendAnnouncement` は `SubmitPrivateSendParams` オブジェクトを受け取ります：

| フィールド | 型 | 必須 | 説明 |
|-------|------|----------|-------------|
| `client` | `StealthCanisterClient` | 必須 | ICP ステルスクライアント |
| `preparation` | `PreparedPrivateSend` | 必須 | `preparePrivateSend()` の戻り値 |
| `tag` | `string` | オプション | アナウンスのフィルタリング用タグ |

### 戻り値

アナウンスが保存されたことを確認する `PrivateSendResult` を返します。

### シグネチャ

```typescript
function submitPrivateSendAnnouncement(
  params: SubmitPrivateSendParams,
): Promise<PrivateSendResult>;
```

## 完全なコード例

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
import { createWalletClient, custom, encodeFunctionData, erc20Abi, keccak256, toBytes } from "viem";
import { arbitrum } from "viem/chains";
import { HttpAgent } from "@dfinity/agent";

// --- セットアップ ---
const sdk = createSdk();
const agent = await HttpAgent.create({ host: "https://icp-api.io" });
const stealthClient = sdk.createStealthClient({
  agent,
  storageCanisterId: "your-storage-canister-id",
  keyManagerCanisterId: "your-key-manager-canister-id",
});

const walletClient = createWalletClient({
  chain: arbitrum,
  transport: custom(window.ethereum!),
});

// トークンを読み込む（自前の設定ファイルまたは組み込みの圧縮データから）
const tokensFile = await import("./tokens.json");
const { tokens } = normalizeTokens(tokensFile);
const entry = findTokenByChain(tokens, 42161n);
const publicClient = createProviderForToken(entry);

// --- ステップ1：Seed を導出 ---
const seedMsg = await getSeedMessage();
const [account] = await walletClient.getAddresses();
const signature = await walletClient.signMessage({
  account,
  message: seedMsg,
});
const seedHex = keccak256(toBytes(signature));

// --- ステップ2：準備 ---
const preparation = await preparePrivateSend({
  client: stealthClient,
  recipientAddress: "0xAbC123...def",
  recipientChainId: 42161n,
  seedHex,
});

// --- ステップ3：送金 ---
const txHash = await walletClient.sendTransaction({
  account,
  to: entry.tokenAddress,
  data: encodeFunctionData({
    abi: erc20Abi,
    functionName: "transfer",
    args: [preparation.burnAddress, 100_000_000n], // 100 zUSDC (6 decimals)
  }),
});
await publicClient.waitForTransactionReceipt({ hash: txHash });

// --- ステップ4：アナウンスを送信 ---
const result = await submitPrivateSendAnnouncement({
  client: stealthClient,
  preparation,
});

console.log("Private send complete:", result);
```

## エラーハンドリング

| エラー | 原因 | 対処法 |
|-------|-------|------------|
| `SeedSignatureRejected` | ユーザーがウォレットの署名プロンプトを拒否 | 再度サインを促す。seed メッセージは決定論的で安全に署名できます |
| `BurnAddressPoWFailed` | 導出したバーンアドレスの Proof-of-Work チェックが失敗 | `preparePrivateSend()` を再試行 — 新しい secret がサンプリングされます |
| `StealthClientNotConnected` | `createStealthClient()` が呼ばれていないか、ICP エージェントに接続できない | ICP エージェントのホストと canister ID を確認する |
| `AnnouncementSubmissionFailed` | ICP ストレージ canister がアナウンスを拒否 | canister が利用可能でアナウンスペイロードが正しい形式であることを確認する |
| `InsufficientBalance` | 送信者が送信元チェーンに十分な zERC20 を保有していない | `LiquidityManager.wrap()` でより多くの原資産トークンを Wrap するか、別チェーンからブリッジする |
| `TransactionReverted` | ERC-20 `transfer` 呼び出しがオンチェーンでリバート | トークンの承認・残高・バーンアドレスの有効性を確認する |

## 関連ページ

- [SDK クイックスタート](quickstart.md) — インストールとはじめの一歩
- [受け取り](receiving.md) — アナウンスの Scan と資金の請求
- [アーキテクチャ概要](../architecture.md) — システムレベルの設計
- [ZKP 仕様](../specs/zkp-spec.md) — Nova と Groth16 Proof の詳細

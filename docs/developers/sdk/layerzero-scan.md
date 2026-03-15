---
icon: layer-group
description: クロスチェーンメッセージの追跡・デコードに使う LayerZero Scan モジュールの使い方
---

# LayerZero Scan

SDK には、クロスチェーンメッセージを追跡するための **LayerZero Scan** モジュールが含まれています。トランザクション履歴 UI の構築・ブリッジ進捗の監視・OFT（Omnichain Fungible Token）ペイロードのデコードに活用できます。

{% hint style="info" %}
このモジュールは公式フロントエンド（develop-sdk）でも使われており、フロントエンド専用サービスから共有 SDK に切り出されたものです。
{% endhint %}

## 概要

zERC20 トークンがクロスチェーン転送される（例：Stargate 経由のクロスチェーン Unwrap）と、トランザクションは LayerZero のメッセージングプロトコルを経由します。LayerZero Scan モジュールが提供する機能：

- **API クライアント** — LayerZero Scan API へのクエリ（ウォレット・TX 単位）
- **ペイロードデコード** — OFT の `send()` calldata・compose メッセージ・ブリッジリクエストのデコード
- **ステータス統合** — `fetchWalletStatus()` でウォレット全メッセージを取得・フィルタ・デコード

## 設定

このモジュールは環境変数ではなく `LayerZeroScanConfig` オブジェクトを受け取ります。フレームワークに依存しない設計です。

```typescript
import type { LayerZeroScanConfig } from "zerc20-client-sdk";

const scanConfig: LayerZeroScanConfig = {
  baseUrl: "https://scan.layerzero-api.com",
  apiKey: "your-api-key",  // 省略可
};
```

## fetchWalletStatus()

メインのエントリーポイントです。ウォレットの全 LayerZero メッセージを取得し、トークンでフィルタリングしてペイロードをデコードします：

```typescript
import { fetchWalletStatus } from "zerc20-client-sdk";
import type { EvmReadProvider } from "zerc20-client-sdk";

// createReadProvider は TokenEntry を受け取り EvmReadProvider を返すファクトリ関数
function createReadProvider(token: TokenEntry): EvmReadProvider {
  return createPublicClient({ chain: ..., transport: http(token.rpcUrls[0]) });
}

const result = await fetchWalletStatus({
  address: "0xYourWallet...",
  tokens,                           // normalizeTokens() で取得した TokenEntry[]
  scanConfig: {
    baseUrl: "https://scan.layerzero-api.com",
    apiKey: "your-api-key",
  },
  createReadProvider,               // (token: TokenEntry) => EvmReadProvider
  limit: 20,                        // 省略可：1ページのメッセージ数
  nextToken: undefined,             // 省略可：ページネーションカーソル
  filterByToken: true,              // 省略可：トークンアドレスでフィルタ
});
// result.items: LayerZeroMessageSummary[]
// result.nextToken: string | undefined
// result.walletUrl: string
```

### FetchWalletStatusParams

| フィールド | 型 | 必須 | 説明 |
|-------|------|----------|-------------|
| `address` | `string` | Yes | クエリ対象のウォレットアドレス |
| `tokens` | `TokenEntry[]` | Yes | フィルタリングとデコードに使うトークン設定 |
| `scanConfig` | `LayerZeroScanConfig` | Yes | API ベース URL と API キー（任意） |
| `createReadProvider` | `(token: TokenEntry) => EvmReadProvider` | Yes | トークンごとの Read Provider を生成するファクトリ関数 |
| `limit` | `number` | No | 1ページあたりのメッセージ数（デフォルト：25） |
| `nextToken` | `string` | No | 前回レスポンスのページネーションカーソル |
| `filterByToken` | `boolean` | No | 設定済みトークンに関連するメッセージのみに絞り込む |

### WalletStatusResult

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `items` | `LayerZeroMessageSummary[]` | デコード済みメッセージのサマリー一覧 |
| `nextToken` | `string \| undefined` | 次ページのページネーションカーソル |
| `walletUrl` | `string` | このウォレットの LayerZero Scan URL |

### LayerZeroMessageSummary

デコード済みの各メッセージには以下のフィールドが含まれます：

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `guid` | `string` | メッセージの一意な識別子 |
| `pathway` | `object` | 送信元・送信先チェーン情報（`src`, `dst`, `nonce`, `srcEid`, `dstEid`） |
| `sourceTx` | `string` | 送信元のトランザクションハッシュ |
| `sourceBlock` | `string` | 送信元のブロック情報 |
| `destinationTx` | `string \| undefined` | 送信先のトランザクションハッシュ（配信完了後） |
| `send` | `SendPayloadSummary \| null` | デコード済みの送信ペイロード（金額、アドレス） |
| `composeFollowups` | `ComposeFollowupSummary[]` | compose メッセージのフォローアップ（例：ブリッジリクエスト） |
| `status` | `string` | メッセージステータス（例：`"DELIVERED"`, `"INFLIGHT"`） |
| `raw` | `LayerZeroScanMessage` | API の生レスポンス |

## API クライアント

LayerZero Scan API を直接クエリします：

```typescript
import { fetchWalletMessages, fetchTxMessages, getWalletMessagesUrl } from "zerc20-client-sdk";

// ウォレットのメッセージを取得
const response = await fetchWalletMessages(scanConfig, "0xWallet...", {
  limit: 10,
  nextToken: undefined,
});

// 特定トランザクションのメッセージを取得
const txMessages = await fetchTxMessages(scanConfig, "0xTxHash...");

// LZ Scan エクスプローラーの URL を生成
const url = getWalletMessagesUrl(scanConfig, "0xWallet...");
```

## ペイロードデコード

OFT の送信ペイロードと compose メッセージをデコードします：

```typescript
import { decodeSendSummary, tryDecodeBridgeRequest, fetchOftSentAmount } from "zerc20-client-sdk";

// OFT の send() calldata をデコード
const summary = decodeSendSummary(txInputData);
// → { dstEid, to, amount, minAmount, compose, source: "tx" | "payload" }

// compose メッセージをブリッジリクエストとしてデコード
const bridge = tryDecodeBridgeRequest(composeMsg);
// → { dstEid, to, refundAddress, minAmountOut } | null

// トランザクションログから OFTSent イベントの金額を読み取る
const amount = await fetchOftSentAmount(readProvider, txHash);
```

## フォーマットユーティリティ

```typescript
import { endpointChain, destinationTx, formatPathway, isMessageForTokens } from "zerc20-client-sdk";

// LZ エンドポイントからチェーン名を取得
const chain = endpointChain(message.pathway?.sender, "src");

// 送信先トランザクションハッシュを抽出
const dstTx = destinationTx(message);

// 経路の文字列表現を取得（例："Arbitrum → Ethereum"）
const path = formatPathway(message);

// メッセージが設定済みトークンに関連するか確認
const relevant = isMessageForTokens(message, tokens);
```

## 関連ページ

- [SDK クイックスタート](quickstart.md) — インストールとセットアップ
- [Wrap と Unwrap](wrap-unwrap.md) — クロスチェーン Unwrap で LayerZero メッセージが発生する
- [API リファレンス](api-reference.md) — 関数一覧

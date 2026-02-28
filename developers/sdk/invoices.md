# Invoice

Invoice を使うと、受信者がバーンアドレス（Burn Address）を事前に生成して送信者に共有でき、送信者はどこにトークンを送ればよいか正確に把握できます。これはプライベート転送（Private Transfer）を両者間で調整するための推奨された方法です。

## Invoice とは

Invoice は受信者が事前に生成した1つ以上のバーンアドレスをまとめたものです。送信者が Invoice を受け取ると、記載されたバーンアドレスに zERC20 トークンを送金します。

Invoice の種類は2つあります：

| 種類 | バーンアドレス数 | 用途 |
| ---------- | -------------- | --------------------------------------------- |
| **Single** | 1 | 1つのバーンアドレスへの1回限りの支払い |
| **Batch** | 10 | 複数の支払いや金額の分割 |

Batch Invoice は常に正確に `NUM_BATCH_INVOICES = 10` 個のバーンアドレスを含みます。

Invoice 内の各バーンアドレスは受信者のアドレス・secret・tweak から導出されており、そのバーンアドレスに送られたトークンを後から Redeem できるのは受信者のみです。

## Invoice を準備する

ICP ストレージ canister に送信する前に、Invoice のアーティファクトをローカルで生成します。このステップでバーンアドレスを導出し、ウォレット署名用のメッセージを準備します。

```typescript
import { prepareInvoiceIssue } from "zerc20-client-sdk";

const artifacts = await prepareInvoiceIssue({
  client,            // StealthCanisterClient
  seedHex,           // 32 バイト hex seed（ウォレット署名を keccak256 したもの）
  recipientAddress,  // 受信者の EVM アドレス
  recipientChainId,  // 受信者が Redeem するチェーン ID
  isBatch: false,    // バッチ（10アドレス）の場合は true、Single の場合は false
  tag: undefined,    // オプション、フィルタリング用タグ
  randomBytes: undefined, // オプション、カスタムランダムバイト列
  maxRetries: undefined,  // オプション、PoW の最大再試行回数
});
```

**シグネチャ：**

```typescript
function prepareInvoiceIssue(
  params: InvoiceIssueParams,
): Promise<InvoiceIssueArtifacts>;
```

**InvoiceIssueParams：**

| フィールド | 型 | 必須 | 説明 |
| ------------------ | ----------------------- | -------- | ---------------------------------------------- |
| `client`           | `StealthCanisterClient` | 必須 | ICP canister クライアント |
| `seedHex`          | `string`                | 必須 | 32 バイト hex seed（導出方法は [プライベート送信](private-send.md#ステップ1seed-を導出する)を参照） |
| `recipientAddress` | `string`                | 必須 | 受信者の EVM アドレス |
| `recipientChainId` | `number \| bigint`      | 必須 | 受信者が Redeem するチェーン ID |
| `isBatch`          | `boolean`               | 必須 | バッチ（10アドレス）の場合は `true`、Single の場合は `false` |
| `tag`              | `string \| undefined`   | オプション | Invoice の分類・フィルタリング用タグ |
| `randomBytes`      | `(length: number) => Uint8Array` | オプション | バーンアドレス導出用のカスタムランダムバイト生成関数 |
| `maxRetries`       | `number \| undefined`   | オプション | バーンアドレス生成の PoW 最大再試行回数 |

**InvoiceIssueArtifacts：**

| フィールド | 型 | 説明 |
| ------------------ | ---------------------------- | ----------------------------------------------- |
| `invoiceId`        | `string`                     | Invoice の一意な識別子 |
| `recipientAddress` | `string`                     | 受信者の EVM アドレス |
| `recipientChainId` | `bigint`                     | Redeem するチェーン ID |
| `burnAddresses`    | `InvoiceBatchBurnAddress[]`  | バーンアドレスエントリの配列 |
| `signatureMessage` | `string`                     | 受信者のウォレットで署名するメッセージ |
| `tag`              | `string \| undefined`        | 指定した場合のタグ |

`burnAddresses` の各エントリは `InvoiceBatchBurnAddress` です：

| フィールド | 型 | 説明 |
| -------------- | -------- | ------------------------------------------ |
| `subId`        | `number` | Invoice 内のサブインデックス（0始まり） |
| `burnAddress`  | `string` | 導出されたバーンアドレス |
| `secret`       | `string` | バーンアドレス導出に使用した secret |
| `tweak`        | `string` | アドレスの一意性を保証する tweak 値 |

Single Invoice の場合、`burnAddresses` は1エントリです。Batch Invoice の場合は正確に10エントリです。

## Invoice を送信する

Invoice を準備した後、受信者のウォレットで `signatureMessage` に署名して ICP ストレージ canister に Invoice を送信します。

```typescript
import { submitInvoice } from "zerc20-client-sdk";
import { hexToBytes } from "viem";

// 準備ステップで取得したメッセージに署名
const signature = await walletClient.signMessage({
  message: artifacts.signatureMessage,
});

// canister に送信（hex 形式の署名を Uint8Array に変換）
await submitInvoice(
  client,                       // StealthCanisterClient
  artifacts.invoiceId,          // hex エンコードされた Invoice ID
  hexToBytes(signature),        // Uint8Array 形式の署名
  artifacts.tag,                // オプションのタグ
);
```

**シグネチャ：**

```typescript
function submitInvoice(
  client: StealthCanisterClient,
  invoiceIdHex: string,
  signature: Uint8Array,
  tag?: string,
): Promise<void>;
```

canister は Invoice に埋め込まれた受信者アドレスに対して署名を検証してから受け入れます。

## Invoice を一覧表示する

特定のアドレスが所有する Invoice ID を取得します。チェーン ID やタグでフィルタリングもできます。

```typescript
import { listInvoices } from "zerc20-client-sdk";

const invoiceIds = await listInvoices(
  client,          // StealthCanisterClient
  ownerAddress,    // Invoice を作成した EVM アドレス
  chainId,         // オプション、受信者チェーン ID でフィルタリング
  tag,             // オプション、タグでフィルタリング
);

// invoiceIds: string[] — hex エンコードされた Invoice ID の配列
```

**シグネチャ：**

```typescript
function listInvoices(
  client: StealthCanisterClient,
  ownerAddress: string,
  chainId?: number | bigint,
  tag?: string,
): Promise<string[]>;
```

Invoice ID を hex 文字列の配列で返します。これらの ID を使って特定の Invoice を検索・共有できます。

## 完全なコード例

Invoice の準備 → 署名 → 送信 → 一覧表示のエンドツーエンドフロー：

```typescript
import {
  prepareInvoiceIssue,
  submitInvoice,
  listInvoices,
} from "zerc20-client-sdk";
import { hexToBytes } from "viem";

// 1. Invoice を準備する
const artifacts = await prepareInvoiceIssue({
  client,
  seedHex: "0xabcdef1234567890...",
  recipientAddress: "0xRecipient...",
  recipientChainId: 8453, // Base
  isBatch: false,
});

console.log("Invoice ID:", artifacts.invoiceId);
console.log("Burn address:", artifacts.burnAddresses[0].burnAddress);

// 2. 受信者のウォレットで署名メッセージに署名する
const signature = await walletClient.signMessage({
  message: artifacts.signatureMessage,
});

// 3. 署名済み Invoice を ICP ストレージ canister に送信する
await submitInvoice(
  client,
  artifacts.invoiceId,
  hexToBytes(signature),  // hex 文字列を Uint8Array に変換
  artifacts.tag,
);

console.log("Invoice submitted successfully");

// 4. このアドレスの全 Invoice を一覧表示する
const invoiceIds = await listInvoices(
  client,
  "0xRecipient...",
);

console.log("All invoices:", invoiceIds);
```

送信後、送信者は Invoice を取得してバーンアドレスを確認し、[プライベート送信](private-send.md)を実行できます。SDK クライアントのセットアップ方法は [SDK クイックスタート](quickstart.md) を参照してください。

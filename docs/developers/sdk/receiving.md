---
icon: inbox
description: SDK を使ったプライベート転送の受け取り・Withdrawal の手順
---

# 受け取り

zERC20 トークンの受け取りは2つのフェーズで行います：

1. **認証 + Scan**：ICP ステルスストレージ canister で認証し、VetKey を復号して、あなた宛のアナウンスを Scan します。
2. **Redeem**：引き出し可能な転送のオンチェーンコンテキストを収集し、ZKP（Zero-Knowledge Proof）を生成して、Verifier コントラクトの `teleport` を呼び出してトークンをミントします。

## ステップ1：認証ペイロードを作成する

受信者のウォレットで署名する、有効期限付きの認証ペイロードを構築します。`ttlSeconds` パラメータで認証の有効期間を制御します。

```typescript
import { createAuthorizationPayload } from "zerc20-client-sdk";

const payload = await createAuthorizationPayload(
  client,          // StealthCanisterClient
  address,         // 受信者の EVM アドレス
  ttlSeconds,      // オプション、デフォルト値あり
);
```

**シグネチャ：**

```typescript
function createAuthorizationPayload(
  client: StealthCanisterClient,
  address: string,
  ttlSeconds?: number,
): Promise<AuthorizationPayload>;
```

**AuthorizationPayload：**

| フィールド              | 型            | 説明                         |
| ------------------ | ------------ | -------------------------- |
| `message`          | `string`     | ウォレットのプロンプトに表示する人間可読なメッセージ |
| `canonicalMessage` | `Uint8Array` | canister 上での検証に使用する正規形式    |
| `expiryNs`         | `bigint`     | ナノ秒単位の有効期限タイムスタンプ          |
| `nonce`            | `bigint`     | リプレイ攻撃を防ぐためのランダムなノンス       |
| `transport`        | `object`     | VetKey 復号用の一時的なトランスポートキーペア |

## ステップ2：認証メッセージに署名する

EIP-191 personal sign を使って認証メッセージに署名します。標準的なウォレットクライアントであれば使用できます。

```typescript
const signature = await walletClient.signMessage({
  message: payload.message,
});
```

ウォレットはユーザーへの承認確認として `payload.message` を表示します。取得した署名は次のステップでアドレスの所有を証明するために使用します。

## ステップ3：VetKey を要求する

署名済みの認証を ICP canister に送信して VetKey を取得します。VetKey はステップ1の一時的なトランスポートキーペアを使って送信時に暗号化され、ローカルで復号されます — canister が平文の鍵を見ることはありません。

```typescript
import { requestVetKey } from "zerc20-client-sdk";

const vetKey = await requestVetKey(
  client,     // StealthCanisterClient
  address,    // 受信者の EVM アドレス
  payload,    // ステップ1の AuthorizationPayload
  signature,  // Uint8Array、署名の生のバイト列
);
```

**シグネチャ：**

```typescript
function requestVetKey(
  client: StealthCanisterClient,
  address: string,
  payload: AuthorizationPayload,
  signature: Uint8Array,
): Promise<VetKey>;
```

返された `VetKey` は ICP に保管されたアナウンスの復号に使用します。

## ステップ4：アナウンスを Scan する

ステルスストレージ canister を Scan して、あなたの VetKey で復号できるアナウンスを探します。各アナウンスはあなた宛のプライベート転送（Private Transfer）に対応しています。

```typescript
import { scanReceivings } from "zerc20-client-sdk";

const announcements = await scanReceivings({
  client,             // StealthCanisterClient
  vetKey,             // ステップ3の VetKey
  pageSize: 100,      // オプション、デフォルト: 100
  startAfter: undefined, // オプション、前回の Scan の続きから再開
  tag: undefined,     // オプション、タグでフィルタリング
});
```

**シグネチャ：**

```typescript
function scanReceivings(
  params: ScanReceivingsParams,
): Promise<ScannedAnnouncement[]>;
```

**ScanReceivingsParams：**

| フィールド        | 型                       | 必須    | 説明                          |
| ------------ | ----------------------- | ----- | --------------------------- |
| `client`     | `StealthCanisterClient` | 必須    | ICP canister クライアント         |
| `vetKey`     | `VetKey`                | 必須    | ステップ3の復号キー                  |
| `pageSize`   | `number`                | オプション | 1ページあたりのアナウンス数（デフォルト: 100）  |
| `startAfter` | `bigint \| undefined`   | オプション | 前回の Scan 後のアナウンス ID（続きから再開） |
| `tag`        | `string \| undefined`   | オプション | タグによるアナウンスのフィルタリング          |

**ScannedAnnouncement：**

| フィールド              | 型        | 説明                       |
| ------------------ | -------- | ------------------------ |
| `id`               | `bigint` | アナウンスの一意な識別子             |
| `burnAddress`      | `string` | 切り詰めたバーンアドレス（オンチェーンの送金先） |
| `fullBurnAddress`  | `string` | 切り詰め前の完全なバーンアドレス         |
| `createdAtNs`      | `bigint` | ナノ秒単位の作成タイムスタンプ          |
| `recipientChainId` | `bigint` | 受信者が Redeem するチェーン ID    |

## ステップ5：Redeem コンテキストを収集する

Scan した各アナウンスについて、Redeem Proof の生成に必要なオンチェーンコンテキストを収集します。Indexer とコントラクトを照会して、どの転送が Redeem 可能かを判定します。

```typescript
import { collectRedeemContext } from "zerc20-client-sdk";

const redeemContext = await collectRedeemContext({
  burn,               // Scan したアナウンスから導出した BurnArtifacts
  tokens,             // トークン設定
  hub,                // Hub コントラクトアドレスまたは設定
  verifierContract,   // Verifier コントラクトインスタンス
  indexerUrl,         // Indexer エンドポイント URL
  indexerFetchLimit,  // オプション、Indexer リクエストあたりの最大イベント数
  eventBlockSpan,     // オプション、スキャンあたりのブロック範囲
});
```

**シグネチャ：**

```typescript
function collectRedeemContext(
  params: RedeemContextParams,
): Promise<RedeemContext>;
```

**RedeemContextParams：**

| フィールド               | 型                  | 必須    | 説明                       |
| ------------------- | ------------------ | ----- | ------------------------ |
| `burn`              | `BurnArtifacts`    | 必須    | アナウンスの BurnArtifacts     |
| `tokens`            | `TokenConfig`      | 必須    | トークン設定                   |
| `hub`               | `HubConfig`        | 必須    | Hub コントラクトアドレスまたは設定      |
| `verifierContract`  | `Contract`         | 必須    | Verifier コントラクトインスタンス    |
| `indexerUrl`        | `string`           | 必須    | Indexer の HTTP エンドポイント   |
| `indexerFetchLimit` | `number`           | オプション | Indexer リクエストあたりの最大イベント数 |
| `eventBlockSpan`    | `bigint \| number` | オプション | イベントスキャンあたりのブロック範囲       |

**RedeemContext：**

| フィールド                | 型                  | 説明                                           |
| -------------------- | ------------------ | -------------------------------------------- |
| `token`              | `TokenInfo`        | 解決済みのトークンメタデータ                               |
| `aggregationState`   | `AggregationState` | Hub の現在の集約スナップショット                           |
| `events`             | `object`           | `eligible`（引き出し可能）と `ineligible`（未確定）のイベント配列 |
| `globalProofs`       | `GlobalProof[]`    | 引き出し可能なイベントのグローバル Merkle Proof               |
| `eligibleProofs`     | `EligibleProof[]`  | ZKP 生成準備済みのイベントごとの Proof                     |
| `totalEligibleValue` | `bigint`           | 今すぐ Redeem できる金額の合計                          |
| `totalPendingValue`  | `bigint`           | まだ証明済み Root に含まれていない金額の合計                    |
| `totalIndexedValue`  | `bigint`           | このバーンアドレスのインデックス済み全金額の合計                     |
| `totalTeleported`    | `bigint`           | 受信者にすでに Teleport（ミント）された金額                   |
| `chains`             | `ChainBreakdown[]` | チェーンごとの引き出し可能・保留中の金額内訳                       |

* **Eligible イベント**：Merkle Root がオンチェーンで証明済みで Hub に集約済みの転送。今すぐ Redeem できます。
* **Ineligible イベント**：インデックスは済んでいるが、Root がまだ証明または集約されていない転送。Indexer とクロスチェーンジョブが追いつき次第、Eligible になります。

## ステップ6：Redeem トランザクションを準備・送信する

SDK は Redeem に prepare / submit の2ステップパターンを提供しています。`prepareRedeemTransaction()` が ZKP を生成してトランザクションデータを組み立て、`submitRedeemTransaction()` が署名してオンチェーンに送信します。

```typescript
import {
  prepareRedeemTransaction,
  submitRedeemTransaction,
} from "zerc20-client-sdk";

// Prepare：Proof を生成してトランザクションオブジェクトを構築
const redeemTx = await prepareRedeemTransaction({
  redeemContext,
  burn,                       // BurnArtifacts
  teleportProofClient: sdk.teleportProofs,
  decider: sdk.decider,       // 省略可：バッチ Proof に必要
});

// Submit：トランザクションに署名してオンチェーンに送信
const { transactionHash } = await submitRedeemTransaction({
  writeProvider,              // EvmWriteProvider
  tx: redeemTx,               // prepare ステップの RedeemTransaction
  readProvider,               // 省略可：レシートポーリング用
  feeOverrides,               // 省略可：ガス価格オーバーライド
});
```

SDK は eligible イベントの数に応じて Proof モードを自動選択します：

- **1件の eligible イベント** — Groth16 シングル Proof（`Verifier.singleTeleport()`）
- **複数の eligible イベント** — Nova バッチ Proof（`Verifier.teleport()`、Decider が必要）

### prepareRedeemTransaction

```typescript
function prepareRedeemTransaction(
  params: PrepareRedeemTransactionParams,
): Promise<RedeemTransaction>;
```

**PrepareRedeemTransactionParams：**

| フィールド | 型 | 必須 | 説明 |
|-------|------|----------|-------------|
| `redeemContext` | `RedeemContext` | Yes | `collectRedeemContext()` の結果 |
| `burn` | `BurnArtifacts` | Yes | アナウンスの Burn アーティファクト |
| `teleportProofClient` | `TeleportProofClient` | Yes | Proof 生成クライアント（`sdk.teleportProofs`） |
| `decider` | `HttpDeciderClient` | No | バッチ Proof に必要。シングルのみの場合は省略可 |

**RedeemTransaction：**

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `mode` | `"single" \| "batch"` | 使われた Proof モード |
| `address` | `Hex` | Verifier コントラクトアドレス |
| `abi` | `object` | Verifier ABI |
| `functionName` | `"singleTeleport" \| "teleport"` | 呼び出すコントラクト関数 |
| `args` | `readonly [...]` | コントラクト呼び出し用のエンコード済み引数 |

### submitRedeemTransaction

```typescript
function submitRedeemTransaction(
  params: SubmitRedeemTransactionParams,
): Promise<{ transactionHash: Hex }>;
```

**SubmitRedeemTransactionParams：**

| フィールド | 型 | 必須 | 説明 |
|-------|------|----------|-------------|
| `writeProvider` | `EvmWriteProvider` | Yes | 署名・送信用のウォレットプロバイダー |
| `tx` | `RedeemTransaction` | Yes | `prepareRedeemTransaction()` の結果 |
| `feeOverrides` | `FeeOverrides` | No | ガス価格オーバーライド（任意） |
| `readProvider` | `EvmReadProvider` | No | レシートポーリング用プロバイダー |

### 事前構築済み Proof でのバッチ Redeem

外部サービスなどで Decider Proof を既に持っている場合は、`buildBatchRedeemTransaction()` でトランザクションを直接組み立てられます：

```typescript
import { buildBatchRedeemTransaction } from "zerc20-client-sdk";

const redeemTx = buildBatchRedeemTransaction({
  redeemContext,
  burn,
  deciderProof: proofBytes,  // Decider からの Uint8Array
});
```

## ステータス確認

Proof の収集・生成をスキップして軽量にステータスを確認するには、`getAnnouncementStatus` を使います。`collectRedeemContext` のオーバーヘッドなしに残高の表示や準備完了のポーリングに適しています。

```typescript
import { getAnnouncementStatus } from "zerc20-client-sdk";

const status = await getAnnouncementStatus({
  // AnnouncementStatusParams
  burn,
  tokens,
  hub,
  verifierContract,
  indexerUrl,
});
```

**シグネチャ：**

```typescript
function getAnnouncementStatus(
  params: AnnouncementStatusParams,
): Promise<AnnouncementStatus>;
```

**AnnouncementStatus：**

| フィールド                | 型        | 説明                         |
| -------------------- | -------- | -------------------------- |
| `totalEligibleValue` | `bigint` | 今すぐ Redeem できる金額の合計        |
| `totalPendingValue`  | `bigint` | まだ証明済み Root に含まれていない金額の合計  |
| `totalIndexedValue`  | `bigint` | このバーンアドレスのインデックス済み全金額の合計   |
| `totalTeleported`    | `bigint` | 受信者にすでに Teleport（ミント）された金額 |

---
icon: toilet-paper-check
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

## ステップ6：Proof を生成して Teleport する

`RedeemContext` の eligible イベントとグローバル Proof を使って ZKP を生成し、オンチェーンに送信します。完全な API リファレンスは [Proof 生成](proof-generation.md) を参照してください。

2つの Redeem 方法があります：

### Single Teleport

Groth16 Proof で1件の eligible イベントを Redeem します：

```typescript
const proof = await createSingleTeleportProof(/* ... */);
// Verifier.singleTeleport() に送信
```

### Batch Teleport

Nova バッチ Proof で複数の eligible イベントを一度に Redeem します：

```typescript
const proof = await createBatchTeleportProof(/* ... */);
// Verifier.teleport() に送信
```

どちらの関数も `RedeemContext` の `eligibleProofs` と `globalProofs` を入力として受け取ります。

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

# API リファレンス

`zerc20-client-sdk` のすべての公開エクスポートの完全なリファレンスです。

## Core

| 名称 | シグネチャ | 戻り値 | 説明 |
|------|-----------|--------|------|
| `createSdk` | `(options?: Zerc20SdkOptions)` | `Zerc20Sdk` | SDK インスタンスを作成 |
| `Zerc20Sdk` | クラス | - | WASM・Proof・Decider・Stealth をまとめたメイン SDK |
| `Zerc20SdkOptions` | インターフェース | - | オプション：`wasm?`、`teleportProofs?`、`decider?`、`stealth?` |

## ICP / Stealth

| 名称 | シグネチャ | 戻り値 | 説明 |
|------|-----------|--------|------|
| `StealthCanisterClient` | クラス | - | アナウンスメント・Invoice・鍵管理用 ICP キャニスタークライアント |
| `StealthClientFactory` | クラス | - | StealthCanisterClient インスタンスを作成するファクトリ |
| `StealthClientConfig` | インターフェース | - | 設定：`agent`、`storageCanisterId`、`keyManagerCanisterId` |
| `createAuthorizationPayload` | `(client, address, ttlSeconds?)` | `Promise<AuthorizationPayload>` | VetKey リクエスト用の認証ペイロードを作成 |
| `requestVetKey` | `(client, address, payload, signature)` | `Promise<VetKey>` | VetKey をリクエストして復号 |
| `scanReceivings` | `(params: ScanReceivingsParams)` | `Promise<ScannedAnnouncement[]>` | アナウンスメントをスキャンして復号 |

## Private Send

| 名称 | シグネチャ | 戻り値 | 説明 |
|------|-----------|--------|------|
| `preparePrivateSend` | `(params: PreparePrivateSendParams)` | `Promise<PreparedPrivateSend>` | バーンアドレス（Burn Address）を導出してアナウンスメントを暗号化 |
| `submitPrivateSendAnnouncement` | `(params: SubmitPrivateSendParams)` | `Promise<PrivateSendResult>` | アナウンスメントをストレージキャニスタに送信 |

## Invoice

| 名称 | シグネチャ | 戻り値 | 説明 |
|------|-----------|--------|------|
| `prepareInvoiceIssue` | `(params: InvoiceIssueParams)` | `Promise<InvoiceIssueArtifacts>` | バーンアドレス付きの Invoice を生成 |
| `submitInvoice` | `(client, invoiceIdHex, signature, tag?)` | `Promise<void>` | 署名済み Invoice を送信 |
| `listInvoices` | `(client, ownerAddress, chainId?, tag?)` | `Promise<string[]>` | Invoice ID の一覧を取得 |

## Liquidity

| 名称 | シグネチャ | 戻り値 | 説明 |
|------|-----------|--------|------|
| `wrapWithLiquidityManager` | `(params: WrapWithLiquidityManagerParams)` | `Promise<LiquidityActionResult>` | 原資産トークンを zERC20 に Wrap |
| `unwrapWithLiquidityManager` | `(params: LocalUnwrapParams)` | `Promise<LiquidityActionResult>` | zERC20 を原資産トークンに Unwrap |
| `quoteLocalUnwrap` | `(params)` | `Promise<LocalUnwrapQuote>` | Unwrap 手数料の見積もりを取得 |
| `buildCrossUnwrapQuote` | `(params)` | `Promise<CrossUnwrapQuote>` | クロスチェーン Unwrap の見積もりを取得 |
| `sendCrossUnwrap` | `(params)` | `Promise<LiquidityActionResult>` | クロスチェーン Unwrap を実行 |
| `fetchLiquidityManagerBalances` | `(params)` | `Promise<LiquidityBalances>` | トークン残高とデシマルを取得 |

## Receive

| 名称 | シグネチャ | 戻り値 | 説明 |
|------|-----------|--------|------|
| `collectRedeemContext` | `(params: RedeemContextParams)` | `Promise<RedeemContext>` | Redeem に必要なイベントと Proof をまとめて取得 |
| `getAnnouncementStatus` | `(params: AnnouncementStatusParams)` | `Promise<AnnouncementStatus>` | アナウンスメントのステータスを軽量チェック |

## Proofs

| 名称 | シグネチャ | 戻り値 | 説明 |
|------|-----------|--------|------|
| `TeleportProofClient` | クラス | - | 高レベルの Proof クライアント |
| `createTeleportProofClient` | `(options?)` | `TeleportProofClient` | Proof クライアントを作成 |
| `HttpDeciderClient` | クラス | - | Decider サービスの HTTP クライアント |

## WASM

| 名称 | シグネチャ | 戻り値 | 説明 |
|------|-----------|--------|------|
| `WasmRuntime` | クラス | - | WASM ライフサイクルマネージャ |
| `getSeedMessage` | `()` | `Promise<string>` | シード導出用の署名メッセージを取得 |
| `derivePaymentAdvice` | `(seedHex, paymentAdviceIdHex, chainId, address)` | `Promise<SecretAndTweak>` | ペイメントアドバイスを導出 |
| `buildFullBurnAddress` | `(chainId, address, secret, tweak)` | `Promise<BurnArtifacts>` | バーンアドレスを構築 |
| `decodeFullBurnAddress` | `(fullBurnAddressHex)` | `Promise<BurnArtifacts>` | バーンアドレスをデコード |

## Registry

| 名称 | シグネチャ | 戻り値 | 説明 |
|------|-----------|--------|------|
| `loadTokens` | `(compressed, options?)` | `Promise<NormalizedTokens>` | 圧縮済み（Base64 gzip）文字列からトークンを読み込む |
| `normalizeTokens` | `(file: TokensFile)` | `NormalizedTokens` | 生のトークン設定を正規化 |
| `findTokenByChain` | `(tokens, chainId)` | `TokenEntry` | チェーン ID でトークンを検索 |
| `createProviderForToken` | `(entry)` | `PublicClient` | RPC プロバイダを作成 |
| `clearTokensCache` | `()` | `void` | トークンキャッシュをクリア |

## Contracts

| 名称 | シグネチャ | 戻り値 | 説明 |
|------|-----------|--------|------|
| `getZerc20Contract` | `(address, client)` | コントラクトインスタンス | zERC20 コントラクトを取得 |
| `getVerifierContract` | `(address, client)` | コントラクトインスタンス | Verifier コントラクトを取得 |
| `getHubContract` | `(address, client)` | コントラクトインスタンス | Hub コントラクトを取得 |
| `getLiquidityManagerContract` | `(address, client)` | コントラクトインスタンス | LiquidityManager コントラクトを取得 |
| `getAdaptorContract` | `(address, client)` | コントラクトインスタンス | Adaptor コントラクトを取得 |

## Types

主なエクスポート型とインターフェース：

- `SecretAndTweak` — ペイメントアドバイスから導出されたシークレットとツイークのペア
- `GeneralRecipient` — 送信フローと Invoice フローで共通して使用される受信者ディスクリプタ
- `BurnArtifacts` — バーンアドレスの構築またはデコード時に生成されるアーティファクト
- `PrivateSendPreparation` — プライベート送信（Private Transfer）の中間準備データ
- `PreparedPrivateSend` — オンチェーン転送とアナウンスメント送信の準備が完了したプライベート送信
- `PrivateSendResult` — プライベート送信のアナウンスメント送信後の結果
- `InvoiceIssueArtifacts` — Invoice 準備で生成されるアーティファクト（バーンアドレスと暗号化データを含む）
- `InvoiceBatchBurnAddress` — バッチ Invoice 内の個別のバーンアドレス
- `ScannedAnnouncement` — スキャン中に検出・復号されたアナウンスメント
- `AggregationTreeState` — クロスチェーン集約 Merkle Tree の状態
- `GlobalTeleportProof` — グローバル（集約）ツリールートに対して有効な Proof
- `IndexedEvent` — インデクサによってインデックスされたオンチェーン転送イベント
- `SingleTeleportArtifacts` — 単一 Groth16 Teleport Proof のアーティファクト
- `SingleTeleportParams` — 単一 Teleport Proof 生成のパラメータ
- `NovaProverInput` — Nova バッチプルーバへの入力
- `NovaProverOutput` — Nova バッチプルーバからの出力
- `BatchTeleportArtifacts` — バッチ（Nova + Groth16）Teleport Proof のアーティファクト
- `BatchTeleportParams` — バッチ Teleport Proof 生成のパラメータ
- `RedeemContext` — Redeem 実行に必要なフルコンテキスト（イベント・Proof・チェーンデータ）
- `RedeemChainContext` — Redeem コンテキストのチェーン別サブセット
- `AnnouncementStatus` — アナウンスメントの軽量ステータス（pending・redeemable・redeemed）
- `EventsWithEligibility` — 受け取り可否が付加されたイベント
- `TokenEntry` — 個別トークンのデプロイ設定
- `HubEntry` — Hub コントラクトのデプロイ設定
- `TokensFile` — 生のデシリアライズ済みトークン設定ファイル
- `NormalizedTokens` — `bigint` フィールドがパース済みの正規化されたトークン設定

## Constants

| 名称 | 値 | 説明 |
|------|----|------|
| `AGGREGATION_TREE_HEIGHT` | `6` | 集約ツリーのレベル数 |
| `TRANSFER_TREE_HEIGHT` | `40` | チェーンごとの転送ツリーのレベル数 |
| `GLOBAL_TRANSFER_TREE_HEIGHT` | `46` | グローバルツリー（40 + 6） |
| `NUM_BATCH_INVOICES` | `10` | バッチ Invoice あたりのバーンアドレス数 |
| `DEFAULT_INDEXER_FETCH_LIMIT` | `20` | インデクサリクエストあたりの最大イベント数 |
| `DEFAULT_EVENT_BLOCK_SPAN` | `5000n` | 集約クエリのブロックスパン |
| `DEFAULT_DECIDER_TIMEOUT_MS` | `300000` | Decider ジョブのタイムアウト（5 分） |
| `DEFAULT_DECIDER_POLL_INTERVAL_MS` | `1000` | Decider のポーリング間隔 |
| `DEFAULT_ZSTORAGE_TAG` | `"v1"` | デフォルト IC ストレージタグ |

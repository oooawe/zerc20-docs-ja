# ICP ストレージ仕様

## 概要

Internet Computer（ICP）は zERC20 のステルスメッセージングレイヤーを提供します。2 つのcanister（キャニスター: IPC上でのスマートコントラクト）が、送信者と受信者の関係をオンチェーンで明かすことなく、暗号化された通信を処理します。

**ファイル**: `zstorage/`

## コンポーネント

### Key Manager キャニスター

**ファイル**: `zstorage/backend/key_manager/`

VetKD（Verifiable Encrypted Threshold Key Derivation）を用いてアイデンティティベース暗号（IBE）鍵を導出します。

**機能**：
- EVM アドレスごとに Boneh-Franklin IBE シークレットを導出
- 鍵リクエストに Nonce と TTL を強制
- 受信者が EVM 署名で認証して閲覧鍵（View Key）を取得

### Storage キャニスター

**ファイル**: `zstorage/backend/storage/`

暗号化されたアナウンスメントと署名済み Invoice を永続化します。

**機能**：
- 暗号化アナウンスメントの保存・取得
- 署名済み Invoice の保存・取得
- 受信者向けのページネーション付きスキャン

## データ構造

### Invoice

受信者が起点となる支払いリクエストです。

```
Invoice {
    invoice_id: String,
    signer: Address,           // EVM address that signed
    burn_addresses: Vec<Address>,
    mode: InvoiceMode,         // Single or Batch
    created_at: Timestamp,
}
```

### Announcement

送信者が起点となる暗号化ペイロードです。

```
Announcement {
    announcement_id: String,
    recipient: Address,        // EVM address
    ibe_ciphertext: Bytes,     // IBE-encrypted AES key
    aes_payload: Bytes,        // AES-GCM encrypted stealth payload
    created_at: Timestamp,
}
```

### Stealth Payload

アナウンスメントを復号した内容です。

```
StealthPayload {
    chain_id: u256,
    recipient_address: Address,
    tweak: u256,
    secret: u256,
    burn_address: Address,
}
```

## ワークフロー

### Invoice フロー（受信者起点）

```
1. 受信者が EVM ウォレットで Invoice リクエストに署名
2. 受信者が storage.submit_invoice() を呼び出す
3. Storage がバーンアドレスとともに Invoice を永続化
4. 受信者がバーンアドレスを支払者に共有
5. 支払者が zERC20 をバーンアドレスに送信
6. 受信者が後ほど ZKP で Redeem
```

**モード**：
- **Single**：Invoice あたり 1 つのバーンアドレス
- **Batch**：最大 10 個のバーンアドレス（サブ ID：0〜9）

### Payment Advice フロー（送信者起点）

```
1. 送信者が自身のシードから（tweak, secret）を導出
2. 送信者が受信者向けのバーンアドレスを計算
3. 送信者が Key Manager から受信者の IBE 公開鍵を取得
4. 送信者がステルスペイロードを暗号化：
   - ランダムな AES 鍵を生成
   - AES-GCM でペイロードを暗号化
   - IBE で AES 鍵を暗号化
5. 送信者が storage.submit_announcement() を呼び出す
6. 送信者が zERC20 をバーンアドレスに転送
7. 受信者がアナウンスメントをスキャンし、復号して Redeem
```

### 受信者によるスキャン

```
1. 受信者が Key Manager に認証（EVM 署名 + トランスポート鍵）
2. Key Manager が暗号化された View Key を返す
3. 受信者が View Key を復号
4. 受信者が Storage からアナウンスメントを取得（ページネーション）
5. 各アナウンスメントに対して：
   - IBE 暗号文を復号して AES 鍵を取得
   - AES ペイロードを復号してステルスペイロードを取得
   - インデクサ経由でバーンアドレスに残高があるか確認
6. 受信者が一致したペイロードをローカルに保存
```

## 暗号化スキーム

### IBE（Identity-Based Encryption）

- **方式**：Boneh-Franklin IBE
- **アイデンティティ**：受信者の EVM アドレス
- **鍵導出**：ICP サブネット鍵からの VetKD

### ペイロード暗号化

```
1. ランダムな 256 ビット AES 鍵を生成
2. AES-256-GCM でステルスペイロードを暗号化
3. 受信者の IBE 公開鍵で AES 鍵を暗号化
4. （IBE 暗号文、AES 暗号文）をアナウンスメントとして保存
```

### 復号

```
1. Key Manager に認証済みで暗号化された View Key をリクエスト
2. トランスポート秘密鍵で View Key を復号
3. View Key で IBE 暗号文を復号 → AES 鍵を取得
4. AES 鍵でペイロードを復号 → ステルスペイロードを取得
```

## クライアントライブラリ

### Rust クライアント

**ファイル**: `zstorage/frontend/`（Rust）

```rust
use zstorage::StealthCanisterClient;

let client = StealthCanisterClient::new(ic_url, key_manager_id, storage_id);

// Issue invoice
client.submit_invoice(chain_id, mode, signature).await?;

// Publish announcement
client.submit_announcement(recipient, encrypted_payload).await?;

// Scan for incoming
let announcements = client.scan_announcements(my_address, page).await?;
```

### TypeScript クライアント

**ファイル**: `frontend/src/services/sdk/storage/`

同等の機能を持つブラウザ対応クライアントです。

## セキュリティに関する考慮事項

- **Key Manager の信頼**：ICP サブネットがマスター鍵を分散保有するため、単一ノードによる復号は不可能
- **Storage のプライバシー**：キャニスターは暗号化データのみ保存し、内容を読むことはできない
- **認証**：鍵リクエストには EVM 署名が必要
- **Nonce / TTL**：鍵導出リクエストへのリプレイ攻撃を防止

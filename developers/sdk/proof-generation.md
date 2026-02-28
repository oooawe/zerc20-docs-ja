# Proof 生成

受け取り（[受け取り](receiving.md)を参照）で転送をスキャンした後、トークンをオンチェーンで請求するためのゼロ知識証明（Zero-Knowledge Proof）を生成します。SDK は Proof 生成を内部で処理します。`collectRedeemContext` から得たデータを渡すと、即座に送信できるコールデータが返ってきます。

## 2 つの Proof モード

| モード | ユースケース | ガスコスト |
|--------|------------|----------|
| **Single**（Groth16） | 転送を 1 件ずつ請求 | 1 件あたりのコストが高い |
| **Batch**（Nova + Decider） | 複数の転送をまとめて請求 | 1 件あたりのコストが低い |

## Single Proof

単一の受け取り可能な転送を Redeem する際に使用します。

```typescript
const sdk = createSdk();

// `redeemContext` comes from collectRedeemContext() -- see Receiving page
const artifacts = await sdk.teleportProofs.createSingleTeleportProof({
  aggregationState: redeemContext.aggregationState,
  recipientFr,
  secretHex,
  event: redeemContext.events.eligible[0],
  proof: redeemContext.globalProofs[0],
});
```

オンチェーンに送信：

```typescript
import { getVerifierContract } from "zerc20-client-sdk";

const verifier = getVerifierContract({ publicClient, walletClient, address: verifierAddress });

const txHash = await verifier.write.singleTeleport([
  artifacts.proofCalldata,
  artifacts.publicInputs,
  artifacts.treeDepth,
]);
```

## Batch Proof

複数の受け取り可能な転送をまとめて Redeem する際に使用します。Batch Proof は **Decider** サービスを通じて最終化され、オンチェーン検証のために Nova IVC Proof を Groth16 Proof に変換します。

```typescript
import { createSdk, HttpDeciderClient } from "zerc20-client-sdk";

const sdk = createSdk();
const decider = new HttpDeciderClient("https://decider.intmax.io");

const artifacts = await sdk.teleportProofs.createBatchTeleportProof({
  aggregationState: redeemContext.aggregationState,
  recipientFr,
  secretHex,
  events: redeemContext.events.eligible,
  proofs: redeemContext.globalProofs,
  decider,
});
```

オンチェーンに送信：

```typescript
const verifier = getVerifierContract({ publicClient, walletClient, address: verifierAddress });

const txHash = await verifier.write.teleport([
  artifacts.deciderProof,
  artifacts.finalState,
  artifacts.steps,
]);
```

## 次のステップ

- [受け取り](receiving.md) — 受信転送のスキャンと Proof 入力の構築
- [SDK クイックスタート](quickstart.md) — インストールとはじめてのプライベート送信
- [Wrap と Unwrap](wrap-unwrap.md) — 原資産トークンと zERC20 の相互変換

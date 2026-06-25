---
status: superseded
updated_at: 2026-06-26
updated_by: Claude
remark: 本草稿已由 v1.3 的 superjson-wire-codec-proposal.md 重寫取代；保留原位作歷史記錄，內容不再維護。
---

# Contract-RPC SuperJSON Adoption Proposal

> **Superseded** — 本草稿已由 [v1.3 superjson-wire-codec-proposal.md](../../v1.3/architecture-improvement/superjson-wire-codec-proposal.md) 重寫取代並擴張為 RPC + event 聯合提案。方向之審議在新文進行；此文僅留作歷史素材,內容(scope、cleanup 主張、codec API)多已被新文修正,勿據此實作。

## 核心判斷

導入 SuperJSON 的範圍應限縮為：

- `@esingpay/contract-rpc`
- `@esingpay/service-kit/rpc`
- 使用 `@InjectRpcClient(xxxRpc)` 的 typed RPC caller
- 使用 `RpcServerOf<typeof xxxRpc>` + `@MessagePattern(createRpcTopicMap(...))` 的 provider controller

不包含：

- `@esingpay/service-kit/rest-rpc`
- API gateway 的 `RestRpcClient`
- legacy `libs/microservices/client/**`
- queue / event payload
- public REST DTO

## Why

目前 `contract-rpc` 的很多 DTO/mapper 是 transport workaround，不是 domain contract。

典型例子：

- `DepositDto = Serialized<Deposit>`
- `WithdrawalIntentDto = Serialized<WithdrawalIntent>`
- `WalletAllocationDto = Serialized<WalletAllocation>`
- `NetworkTransactionDto = Serialized<NetworkTransaction>`

這些 `Serialized<T>` 主要只是為了：

- `bigint -> string`
- `Date -> ISO string`
- caller 再 `BigInt(...)`
- caller 再 `new Date(...)`

這些轉換目前散在：

- server RPC mapper
- caller use-case
- shared RPC mapper
- tests / stubs

所以 SuperJSON 應該把這件事收回到 `service-kit/rpc` adapter boundary。

## Proposed Boundary

`contract-rpc` 本身仍只放宣告，不放 SuperJSON runtime code。

建議分工：

- `libs/contract-rpc`
  - 定義 semantic RPC contract
  - 逐步允許 `bigint` / `Date`
  - 保留 `ResultDto`、code union、method-specific contract
- `libs/service-kit/src/lib/rpc`
  - 新增 SuperJSON codec
  - typed RPC client encode request / decode response
  - typed RPC server wrapper decode request / encode response
- `apps/esingpay-cradle/src/*/rpc/server/**`
  - 逐步移除純 serialization mapper
  - 保留 error mapping / projection mapping / semantic validation

## Guide Change Needed

目前 `guide/contract-structure.md` 的規則是：

- RPC payload 必須 JSON-safe
- `Date` / `bigint` 不直接出現在 RPC wire DTO

導入 SuperJSON 後應改成：

- RPC semantic contract 可以使用 `bigint` / `Date`
- wire serialization 由 `service-kit/rpc` codec 負責
- public REST / event / queue 是否 JSON-safe 另依各自 guide，不受此變更影響

這是必要的 guide 修訂，不然會跟現行規範衝突。

## Recommended Architecture

新增：

```text
libs/service-kit/src/lib/rpc/rpc-codec.ts
```

概念 API：

```ts
export interface RpcCodec {
  encode(value: unknown): unknown;
  decode(value: unknown): unknown;
}

export const RpcSuperJsonCodec: RpcCodec = {
  encode(value) {
    return {
      __rpcCodec: 'superjson@1',
      payload: SuperJSON.serialize(value),
    };
  },

  decode(value) {
    if (!isSuperJsonRpcPacket(value)) return value;
    return SuperJSON.deserialize(value.payload);
  },
};
```

client side：

```ts
async call(topic: string, input: unknown): Promise<unknown> {
  const wireInput = this.codec.encode(input);
  const wireOutput = await this.send(topic, wireInput);
  return this.codec.decode(wireOutput);
}
```

server side：

```ts
export async function handleContractRpc<TInput, TOutput>(
  wireInput: unknown,
  handler: (input: TInput) => Promise<TOutput> | TOutput,
): Promise<unknown> {
  const input = RpcSuperJsonCodec.decode(wireInput) as TInput;
  const output = await handler(input);
  return RpcSuperJsonCodec.encode(output);
}
```

第一階段可先用 helper，不急著做 decorator magic 或全域 serializer。

## DTO Cleanup Rule

可以移除或弱化的 DTO：

- 純 `Serialized<T>` alias
- 只為 `bigint` / `Date` string 化存在的 mapper
- caller-side `BigInt(dto.id)` / `new Date(dto.createdAt)` 還原邏輯

應保留的 contract：

- `ResultDto`
- RPC code enum / error union
- method-specific input/output shape
- permission masking / projection / field rename
- pagination / batch / partial response
- capability command shape，例如 `InstructNetworkTransactionDto`

也就是：**SuperJSON 取代 transport workaround，不取代語意 contract。**

## Good First Target

我會選：

```text
walletAllocationRpc.listByFundCase
```

原因：

- caller 現在傳 `identifier: depositId.toString()`
- server mapper 再 `BigInt(input.identifier)`
- response `WalletAllocationDto` 內有 nested `bigint` / `Date`
- fund caller 又會 `BigInt(existing.id)`
- blast radius 比 `reserve` / `settle` 小

第一個 PR 不要碰全部 `walletAllocationRpc`，只證明 typed channel 可行。

## First PR Scope

修改：

- `package.json`
  - add `superjson`
- `libs/service-kit/src/lib/rpc/rpc-codec.ts`
  - codec + packet guard
- `libs/service-kit/src/lib/rpc/rpc-client.ts`
  - opt-in encode/decode
- `libs/service-kit/src/lib/rpc/index.ts`
  - export codec if needed
- `walletAllocationRpc.listByFundCase`
  - contract input/output POC 改 native type
- 對應 server controller/helper
- 對應 targeted tests

不碰：

- `service-kit/rest-rpc`
- API gateway
- legacy microservice clients
- queue/event
- all `Serialized<T>` cleanup
- global Nest serializer/deserializer

## Acceptance Criteria

- caller 傳 `identifier: bigint`
- server handler 收到 `bigint`
- response 回來後 caller 看到 `WalletAllocation.id: bigint`
- `Date` 欄位是 `Date`
- plain payload fallback 還能 decode，方便 rollback
- legacy / rest-rpc 完全不受影響

## Bottom Line

整理後的方向是：

**只讓 `contract-rpc + service-kit/rpc` 這條 typed RPC channel 使用 SuperJSON。**

這樣可以吃到主要收益，也不會污染 REST、HTTP、legacy RPC 或 queue/event。這是目前最合理的草創期方案。

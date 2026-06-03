---
status: draft
updated_at: 2026-06-03
updated_by: Codex
---

# External API — Stage 4 Blueprint

## 目標

Stage 4 完成 external API key 的 backend 基礎能力：

- API key create。
- API key list。
- API key delete。
- API key to merchant resolution。
- API Gateway external guard 可透過 RPC 向 cradle 查詢 API key 是否可用。
- API Gateway 可將驗證結果解析成 `merchant-agent` identity。

第一版不做：

- API key edit。
- API key scope。
- API key prefix。
- IP whitelist。
- Idempotency key。

IP whitelist 會在 Stage 5 處理。Console UI 會在 Stage 6 處理。Docs website 會在 Stage 7 處理。

## Ownership Decision

Stage 4 的 API key 資料表應建立在 `esingpay-cradle`，由 cradle 的 `src/external` module 擁有。

理由：

- External API 是新 merchant system integration boundary。
- `esingpay-cradle` 會是未來 external API 的主要開發點。
- 舊 account management 之後會搬遷，不應把新 external access policy 再放回舊服務。
- API key 是 external API credential，不是 fund / wallet domain 資料，也不是 merchant portal user session。
- API Gateway 不應直接查 DB，只應透過 RPC 詢問 credential owner。

因此 Stage 4 以 cradle external module 作為 external API credential/access policy 的 source of truth。

## Existing Account Management API Key

現有 codebase 已存在舊的 account-management API key 雛形：

```text
merchant_api_keys
```

但 Stage 4 不應直接沿用它作為正式 external API key 設計。

主要原因：

- 目前欄位使用 `plaintext` 儲存 key。
- 查詢也是用 plaintext 比對。
- 現有能力只有 create / get。
- 沒有 list / delete / guard verify。
- 沒有 IP whitelist 延伸點。
- 這套資料位於舊 account-management boundary，與未來 cradle ownership 不一致。

Stage 4 可以把它視為歷史參考，但不作為 external API key 的正式 schema baseline。

## Data Model

新增 cradle external API key table：

```text
external_api_keys
```

建議欄位：

```text
id
merchant_id
name
key_hash
key_last4
created_at
deleted_at
last_used_at
```

欄位語意：

- `id`：API key record id，對外 list / delete 使用。
- `merchant_id`：API key 所屬 merchant。
- `name`：Console 顯示用名稱。
- `key_hash`：API key hash 後的值，不存 plaintext。
- `key_last4`：Console list 顯示用。
- `created_at`：建立時間。
- `deleted_at`：soft delete 時間。
- `last_used_at`：最後成功驗證時間。

第一版 delete 採 soft delete：

```text
delete -> set deleted_at
```

不直接刪除資料列，避免 audit 與追查困難。

## API Key Generation

Create API key 時：

- server 產生 API key plaintext。
- 只在 create response 回傳一次 plaintext。
- DB 只保存 `key_hash` 與 `key_last4`。
- 後續 list 不回傳 plaintext。

API key plaintext 遺失時，不提供查看或編輯，只能 delete 後重新 create。

第一版不使用 prefix，所以 API Gateway guard 不能靠 prefix routing；它必須把完整 key 送到 cradle verify RPC，由 cradle hash 後查詢。

## Cradle External Module

Stage 4 在 cradle `src/external` 底下新增 API key capability。

概念檔案：

```text
apps/esingpay-cradle/src/external/api-key/api-key.module.ts
apps/esingpay-cradle/src/external/api-key/entity/external-api-key.entity.ts
apps/esingpay-cradle/src/external/api-key/repository/external-api-key.repository.ts
apps/esingpay-cradle/src/external/api-key/service/external-api-key.service.ts
apps/esingpay-cradle/src/external/api-key/use-case/create-external-api-key.use-case.ts
apps/esingpay-cradle/src/external/api-key/use-case/list-external-api-keys.use-case.ts
apps/esingpay-cradle/src/external/api-key/use-case/delete-external-api-key.use-case.ts
apps/esingpay-cradle/src/external/api-key/use-case/verify-external-api-key.use-case.ts
apps/esingpay-cradle/src/external/api-key/rpc/external-api-key.controller.ts
apps/esingpay-cradle/src/external/api-key/rpc/external-api-key.mapper.ts
```

職責：

- Entity / repository：保存 external API key。
- Create use case：產生 key、hash、保存、回傳一次性 plaintext。
- List use case：列出 merchant 自己的 API keys，不回傳 plaintext。
- Delete use case：確認 merchant ownership 後 soft delete。
- Verify use case：給 API Gateway guard 使用，解析 API key 對應 merchant。
- RPC controller：提供 API Gateway 呼叫的 guard verification endpoint。

## Gateway Guard RPC

API Gateway external guard 不直接查 DB。

Guard flow：

```text
external request
  -> parse API key from request header
  -> resolve client IP
  -> call cradle verify RPC
  -> receive merchant identity
  -> attach merchant-agent identity to request context
  -> continue external controller
```

Stage 4 的 verify RPC input：

```text
apiKey
clientIp
```

Stage 4 中 `clientIp` 可先傳入但不檢查，保留給 Stage 5 IP whitelist 使用。

Stage 4 的 verify RPC output：

```text
valid
apiKeyId
merchantId
rejectionReason
```

成功時：

```text
valid = true
apiKeyId = external API key id
merchantId = API key owner merchant id
```

失敗時：

```text
valid = false
rejectionReason = missing | invalid | deleted
```

API Gateway guard 應將失敗結果轉成 external auth error，不應把 internal rejection detail 原樣暴露給 external caller。

## RPC Contract

API key guarding 是 internal service-to-service capability，不是 public external REST endpoint。

因此 Stage 4 應新增 contract-rpc 定義，而不是放進 external contract-rest endpoint：

```text
libs/contract-rpc/src/lib/external/rpc/external-api-key.rpc.ts
```

概念 RPC：

```text
externalApiKey.verify
```

此 RPC 供 API Gateway guard 呼叫 cradle external module。

External public endpoints 仍維持 Stage 1 的 decision：

```text
API Gateway external REST
  -> contract-rest based RestRpcClient
  -> cradle src/external REST-RPC server
```

API key guard verification 則是：

```text
API Gateway guard
  -> contract-rpc
  -> cradle src/external/api-key/rpc
```

## API Gateway Integration

API Gateway 新增 external API key guard。

概念檔案：

```text
apps/esing-pay-api-gateway/src/rest/external/guard/external-api-key.guard.ts
apps/esing-pay-api-gateway/src/rest/external/identity/merchant-agent.identity.ts
apps/esing-pay-api-gateway/src/rest/external/rpc/external-api-key.client.ts
```

Guard 負責：

- 從 header 解析 API key。
- 呼叫 cradle verify RPC。
- 將成功結果轉成 `merchant-agent` identity。
- 將 identity 掛到 request context。
- 將 missing / invalid / deleted key 轉成 external auth error。

`merchant-agent` identity 第一版至少包含：

```text
merchantId
apiKeyId
```

後續 Stage 5 可在同一個 identity 或 request context 中保留 IP whitelist 驗證結果，但不應影響 Stage 4 的 API shape。

## Console Management API

Stage 4 需要提供 Console side 可呼叫的 backend capability：

- create API key。
- list API keys。
- delete API key。

Console UI 本身放在 Stage 6。

Stage 4 僅定義 backend 行為：

- Console caller 必須是已登入 merchant user。
- API key create / list / delete 都只能作用在 caller 所屬 merchant。
- create response 只回傳一次 plaintext。
- list response 不回傳 plaintext。
- delete 是 soft delete。
- 不提供 edit。

Console 管理 API 可以先走既有 Console / merchant auth boundary，再由 gateway 或後端 adapter 呼叫 cradle external API key use cases。具體路由放置可在 plan 階段依現有 Console gateway 結構確認。

## Error Mapping

External API request guard error：

- Missing API key -> unauthorized。
- Invalid API key -> unauthorized。
- Deleted API key -> unauthorized。

Console management error：

- API key not found -> not found。
- API key belongs to another merchant -> not found 或 forbidden，plan 階段依現有 API 標準決定。
- Deleted key cannot be deleted again -> idempotent success 或 not found，plan 階段決定。

Guard error 不應回傳 key 是否存在、是否刪除等細節給 external caller。

## Validation Target

Stage 4 完成時，應能證明：

```text
Console merchant user
  -> create API key
  -> receive plaintext once
  -> list API keys without plaintext
  -> delete API key by id
```

以及：

```text
Merchant system request
  -> API Gateway external API key guard
  -> cradle externalApiKey.verify RPC
  -> merchant-agent identity { merchantId, apiKeyId }
  -> external controller
```

Stage 4 完成後，External API 的 read / create endpoints 仍由原本 external gateway-to-cradle flow 處理；API key guard 只是在進入 external controller 前建立可信任的 merchant identity。

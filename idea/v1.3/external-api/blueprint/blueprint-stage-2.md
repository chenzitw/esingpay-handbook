---
status: draft
updated_at: 2026-06-03
updated_by: Codex
---

# External API — Stage 2 Blueprint

## 目標

Stage 2 完成 external read APIs：

- `GET /external/v1/deposits`
- `GET /external/v1/deposits/{id}`
- `GET /external/v1/withdrawal-intents`
- `GET /external/v1/withdrawal-intents/{id}`
- `GET /external/v1/wallets`
- `GET /external/v1/wallets/{id}`

Stage 2 延續 Stage 1 建立的 boundary：

```text
esing-pay-api-gateway src/rest/external
  -> RestRpcClient
  -> esingpay-cradle src/external
  -> existing fund / wallet capability
  -> external DTO response
```

## Scope

範圍內：

- 補齊 external deposit list / detail。
- 補齊 external withdrawal intent list / detail。
- 補齊 external wallet list / detail。
- 實作 external read DTO mapping。
- 在 cradle external adapter 內做 merchant scoping。
- 確保 external response 不直接輸出 portal DTO。

範圍外：

- `POST /external/v1/withdrawal-intents`。
- API key persistence and validation。
- IP whitelist enforcement。
- Console API key management。
- Docusaurus docs site。
- `Idempotency-Key`。
- API key scope。

## Capability Reuse Decision

Stage 2 不新增不必要的 application capability。

Deposit 與 withdrawal intent 已經有 merchant-scoped use case，可以直接從 cradle external adapter 透過 DI 使用。

Wallet 目前沒有 merchant-scoped wallet read use case，但已有 `WalletQueryService` 與 `WalletComposer`。依目前 guide，這類多個 read method 的 capability 本來就適合是 query service，不需要為了形式一致硬包一層 use case。

## External Deposit

### APIs

- `GET /external/v1/deposits`
- `GET /external/v1/deposits/{id}`

### 使用既有 capability

Cradle external deposit adapter 應 import fund deposit context，並直接注入：

- `MerchantSearchDepositsUseCase`
- `MerchantGetDepositUseCase`

### Adapter responsibility

Cradle external deposit adapter 負責：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取得 `merchantId`。
- 將 external query params 轉成 merchant-scoped deposit query。
- 呼叫既有 merchant deposit use case。
- 將 `DepositComposed` map 成 `ExternalDepositDto`。
- 將 use-case error map 成 external REST result code。

### Mapping 注意事項

External deposit response 不應直接重用 portal `DepositDto`。

External mapper 應隱藏：

- actor details。
- full status history。
- internal correlation details。
- network endpoint lifecycle。

## External Withdrawal Intent

### APIs

- `GET /external/v1/withdrawal-intents`
- `GET /external/v1/withdrawal-intents/{id}`

### 使用既有 capability

Cradle external withdrawal intent adapter 應 import fund withdrawal intent context，並直接注入：

- `MerchantSearchWithdrawalIntentsUseCase`
- `MerchantGetWithdrawalIntentUseCase`

### Adapter responsibility

Cradle external withdrawal intent adapter 負責：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取得 `merchantId`。
- 將 external query params 轉成 merchant-scoped withdrawal intent query。
- 呼叫既有 merchant withdrawal intent use case。
- 將 `WithdrawalIntentComposed` map 成 `ExternalWithdrawalIntentDto`。
- 將 use-case error map 成 external REST result code。

### Mapping 注意事項

External withdrawal intent response 不應直接重用 portal `WithdrawalIntentDto`。

External mapper 應隱藏：

- actor details。
- full status history。
- wallet allocation id。
- fee payer wallet strategy。
- internal network endpoint lifecycle。

## External Wallet

### APIs

- `GET /external/v1/wallets`
- `GET /external/v1/wallets/{id}`

### 使用既有 capability

Wallet read 目前沒有 merchant-scoped use case。

Cradle external wallet adapter 應 import wallet context，並直接注入：

- `WalletQueryService`
- `WalletComposer`

不需要為了 external API 形式一致新增只包一層 `WalletQueryService + WalletComposer` 的 wallet use case。

理由：

- `WalletQueryService` 已經是 application-layer read-only capability。
- 它提供多個 public read methods，依 guide 應視為 query service，而不是 use case。
- External adapter 可以直接使用 query service 處理簡單 read flow。
- 新增 pass-through use case 只會增加維護成本，沒有增加業務語意。

### Adapter responsibility

Cradle external wallet adapter 負責：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取得 `merchantId`。
- List 時用 `merchantId` 查詢商戶 wallet。
- Detail 時先 decode wallet id，再用 `WalletQueryService.getOneById` 查詢。
- Detail 必須驗證 wallet 的 `merchantId` 等於 identity merchantId。
- 使用 `WalletComposer` compose wallet detail。
- 將 composed wallet map 成 `ExternalWalletDto`。
- 將 not found / invalid params map 成 external REST result code。

### Mapping 注意事項

External wallet response 不應直接重用 portal `WalletDto`。

External mapper 應隱藏：

- top-level `merchant`。
- `networkEndpoint.status`。
- HD wallet path / account / index。
- held / heldLocked balance。

## Gateway

Stage 2 gateway external module 應延伸 Stage 1 結構，新增或補齊：

- external deposit controller / proxy / query DTO。
- external withdrawal intent detail method。
- external wallet controller / proxy / query DTO。

Gateway external controllers 應依 resource 拆檔，例如：

- `controller/deposit.controller.ts`
- `controller/withdrawal-intent.controller.ts`
- `controller/wallet.controller.ts`

Gateway 不做 business mapping，也不直接呼叫 fund / wallet portal controllers。

## Cradle

Stage 2 cradle external module 應延伸 Stage 1 結構，新增或補齊：

- external deposit RPC adapter。
- external withdrawal intent detail adapter。
- external wallet RPC adapter。
- external read mappers。

Cradle external adapters 不呼叫 fund / wallet REST adapters。

正確方向：

```text
external adapter
  -> existing use case / query service / composer
  -> external mapper
```

錯誤方向：

```text
external adapter
  -> fund or wallet REST adapter
```

## Validation Target

Stage 2 完成時，應能證明：

```text
GET /external/v1/deposits
GET /external/v1/deposits/{id}
GET /external/v1/withdrawal-intents
GET /external/v1/withdrawal-intents/{id}
GET /external/v1/wallets
GET /external/v1/wallets/{id}
```

都能走：

```text
gateway src/rest/external
  -> cradle src/external
  -> existing fund / wallet capability
  -> external DTO response
```

其中 wallet read 明確使用 `WalletQueryService + WalletComposer`，不新增 pass-through wallet use case。

---
status: draft
updated_at: 2026-06-03
updated_by: Codex
---

# External API — Stage 3 Blueprint

## 目標

Stage 3 完成 external withdrawal intent create：

- `POST /external/v1/withdrawal-intents`

這個 stage 的重點不是複製既有 merchant submit use case，而是把 withdrawal intent submit 的共用核心流程抽出來，讓 merchant portal submit 與 external API submit 共用同一份 submit mechanics。

## 背景

既有 `MerchantSubmitWithdrawalIntentUseCase` 目前同時承載兩件事：

- withdrawal intent submit 的核心流程。
- merchant portal actor 建立，也就是 `createMerchantUserActor(command.merchantUserId)`。

External API 使用 `merchant-agent` identity。它代表商戶系統透過 API key 呼叫，不是 merchant portal user。

因此 external submit 不應把 `merchantId` 塞進 `merchantUserId`，也不應讓 `merchantUserId` 變成 nullable 後在同一個 use case 裡隱式切換 actor。

## Actor Strategy

Stage 3 第一版採用：

```text
external submit -> submittedBy = createSystemActor()
```

理由：

- 不新增 ActorType。
- 不新增 actor persistence 欄位。
- 不需要 migration。
- composer 不會因為 merchantUserId 而去 account management 查 merchant user refs。
- portal REST mapper 仍能正常顯示 actor 為 System。

此策略的限制：

- withdrawal intent 的 `submittedBy` 只會顯示 System。
- 無法從 actor 欄位直接知道是哪把 API key 建立。
- 若正式 audit 需要追 API key，應另外設計 API key audit / request log，而不是把 API key 硬塞進 `merchantUserId`。

## Submit Flow Decision

Stage 3 應抽出共用 submit service，而不是 copy 一份 external submit use case。

### 共用核心

新增共用 submitter service，概念上負責：

```text
merchantId
senderWalletId
destination
currencyCode
amount
submittedBy
```

它承載原本 merchant submit use case 裡的大段 submit mechanics：

- sender wallet ownership check。
- amount normalize。
- sender network endpoint lookup。
- withdrawal category derive。
- fee payer resolve。
- network fee estimate。
- withdrawal intent evaluation。
- merchant withdrawal limit lookup。
- wallet allocation reserve。
- withdrawal intent persist。
- wallet allocation bind。
- acceptance job dispatch。
- result compose。
- submit error convergence。

共用 submitter service 不決定 caller 是 portal 還是 external，也不建立 actor。它只接受已決定好的 `submittedBy`。

### Merchant Portal Use Case

`MerchantSubmitWithdrawalIntentUseCase` 保留 merchant portal 語意。

它負責：

- 接收 `merchantId`。
- 接收 `merchantUserId`。
- 建立 `createMerchantUserActor(merchantUserId)`。
- 呼叫共用 submitter service。

### External Use Case

新增 external submit use case。

它負責：

- 接收 `merchant-agent` 對應的 `merchantId`。
- 接收 external request mapping 後的 wallet / destination / amount。
- 建立 `createSystemActor()`。
- 呼叫同一個共用 submitter service。

External use case 不接收 `merchantUserId`。

## Cradle External Adapter

`src/external` 的 withdrawal intent create adapter 負責：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取得 `merchantId`。
- 將 external create DTO 轉成 external submit use case command。
- 呼叫 external submit use case。
- 將 result map 成 `ExternalWithdrawalIntentDto`。
- 將 use-case error map 成 external REST result code。

External adapter 不直接呼叫既有 `MerchantSubmitWithdrawalIntentUseCase`。

## Contract REST

Stage 3 需要在 external withdrawal intent API contract 補上：

- `POST /external/v1/withdrawal-intents`

External create request DTO 依 design 文件定義，第一版支援範圍以 Tron USDT blockchain address 為主。

第一版不做：

- `Idempotency-Key`。
- API key scope。
- API key prefix。

## Reuse

Stage 3 應重用：

- 現有 withdrawal intent submit mechanics。
- 現有 wallet ownership / fee / evaluation / allocation / queue dispatch dependencies。
- 現有 withdrawal intent composer。
- 現有 submit error type 與 result mapping 可用部分。

Stage 3 不應重用：

- `merchantUserId` 作為 external API caller identity。
- Merchant portal REST DTO 作為 external request / response DTO。
- Merchant portal submit use case 作為 external adapter 的直接 dependency。
- Copy-pasted submit implementation。

## Validation Target

Stage 3 完成時，應能證明：

```text
POST /external/v1/withdrawal-intents
  -> gateway external withdrawal-intent.controller
  -> cradle external withdrawal-intent adapter
  -> ExternalSubmitWithdrawalIntentUseCase
  -> shared WithdrawalIntentSubmitterService
  -> ExternalWithdrawalIntentDto response
```

同時，merchant portal submit 仍應走：

```text
MerchantSubmitWithdrawalIntentUseCase
  -> createMerchantUserActor(merchantUserId)
  -> shared WithdrawalIntentSubmitterService
```

兩條路徑共用 submit mechanics，但 actor strategy 各自明確。

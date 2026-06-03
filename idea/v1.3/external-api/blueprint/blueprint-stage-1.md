---
status: draft
updated_at: 2026-06-03
updated_by: Codex
---

# External API — Stage 1 Blueprint

## 目標

Stage 1 建立 external API boundary 的基本骨架。

第一條驗證路徑使用：

- `GET /external/v1/withdrawal-intents`

這條路徑用來證明：

```text
esing-pay-api-gateway src/rest/external
  -> RestRpcClient
  -> esingpay-cradle src/external
  -> existing fund withdrawal intent capability
  -> external response DTO
```

Stage 1 不完成所有 external API resource。它只先證明 boundary 成立，並提供後續 stage 可延伸的穩定結構。

## 邊界

External API Gateway 程式碼放在：

```text
apps/esing-pay-api-gateway/src/rest/external
```

External cradle adapter 程式碼放在：

```text
apps/esingpay-cradle/src/external
```

Stage 1 不得把 external controllers 或 adapters 放在：

```text
apps/esing-pay-api-gateway/src/rest/fund
apps/esing-pay-api-gateway/src/rest/wallet
apps/esingpay-cradle/src/fund
apps/esingpay-cradle/src/wallet
```

Fund 和 wallet 仍是 business capability owners。External 是 integration adapter boundary。

## Stage 1 Scope

範圍內：

- 定義 external withdrawal intent list REST contract。
- 新增 gateway external REST module。
- 新增 gateway external withdrawal intent controller，並以獨立 controller file 表示。
- 新增 gateway external withdrawal intent proxy。
- 新增 cradle external module。
- 新增 cradle external withdrawal intent RPC adapter。
- 建立 boundary 使用的 `merchant-agent` identity。
- 將 `GET /external/v1/withdrawal-intents` 從 gateway 串到 cradle。

範圍外：

- 完整 API key persistence 與 validation。
- 完整 IP whitelist enforcement。
- Console API key management。
- Docusaurus docs site。
- `Idempotency-Key`。
- API key scope。
- API key prefix。
- 其他 external read resources。
- Withdrawal intent create。

## Contract REST

Stage 1 應在 external namespace 下建立 external REST contract，不放在 fund 或 wallet portal API definitions 底下。

### 新增 `libs/contract-rest/src/lib/external/namespace.ts`

定義 external REST contract namespace。

### 新增 `libs/contract-rest/src/lib/external/api/withdrawal-intent.api.ts`

定義 public external withdrawal intent API：

```text
GET /external/v1/withdrawal-intents
```

API identity 應使用 `merchant-agent`，不是 merchant user。

### 新增 `libs/contract-rest/src/lib/external/dto/withdrawal-intent.dto.ts`

對應 `idea/v1.3/external-api/design/design-withdrawal-intent-dto.md`

### 新增 `libs/contract-rest/src/lib/external/dto/search-withdrawal-intent-params.dto.ts`

定義 `GET /external/v1/withdrawal-intents` 的 query params。

Stage 1 可以先保持 params 最小化。除非 plan 發現既有 pattern 有必要最低欄位，否則 pagination 足以證明 boundary。

### 新增 `libs/contract-rest/src/lib/external/api/index.ts`

匯出 external API definitions。

### 新增 `libs/contract-rest/src/lib/external/dto/index.ts`

匯出 external DTOs。

### 新增 `libs/contract-rest/src/lib/external/index.ts`

匯出 external namespace、APIs、DTOs。

## Identity

Stage 1 應建立 external API calls 使用 `merchant-agent` identity，而不是 merchant user identity。

`merchant-agent` 表示商戶系統透過 API key 代理商戶發出 request。

Stage 1 可以先 mock merchant identity values。後續 stage 會把 mock source 替換成 API key validation，並附上 whitelist、audit、debug flows 需要的 API key reference。

### 修改 REST contract identity support

Contract REST API identity enum 需要新增 `merchant-agent` value，讓 external API definitions 不會偽裝成 merchant portal APIs。

### 修改 REST RPC identity support

Service-kit REST RPC identity type 需要新增 `merchant-agent` variant。

Identity 至少要承載 merchant identity information；API key validation 落地後，也應承載 API key identity information。

如果 type 已要求 API key reference，Stage 1 可以先使用 mock API key reference。

## API Gateway Files

### 新增 `apps/esing-pay-api-gateway/src/rest/external/external.module.ts`

註冊 gateway external REST controllers 與 proxies。

匯入既有 REST RPC client module。

### 新增 `apps/esing-pay-api-gateway/src/rest/external/controller/withdrawal-intent.controller.ts`

負責 external withdrawal intent HTTP endpoint：

```text
GET /external/v1/withdrawal-intents
```

職責：

- 綁定 external withdrawal intent API base path。
- 使用 external query DTO parse query params。
- 建立 `merchant-agent` identity。
- 呼叫 external withdrawal intent proxy。
- 將 API key guard 與 IP whitelist guard 保留為明確的 future insertion points。

這個 controller 必須是獨立檔案。不要併進 generic external controller，也不要放在 `src/rest/fund`。

### 新增 `apps/esing-pay-api-gateway/src/rest/external/proxy/withdrawal-intent.proxy.ts`

使用 external withdrawal intent API contract 建立 RestRpc topic map。

職責：

- 接收 gateway controller request。
- 將 typed RestRpc request 轉送到 cradle。
- 回傳 cradle response，不做 business mapping。

### 新增 `apps/esing-pay-api-gateway/src/rest/external/dto/ExternalSearchWithdrawalIntentParamsQuery.ts`

定義 query params 的 Nest validation DTO。

它應與 external contract search params 對齊。

### 新增 `apps/esing-pay-api-gateway/src/rest/external/identity/create-merchant-agent-identity.ts`

建立 RestRpc request 使用的 `merchant-agent` identity。

Stage 1 可以先 mock merchant identity 與 API key identity。後續 stage 會改成從 API key guard output 建立 identity。

### 修改 `apps/esing-pay-api-gateway/src/rest/rest.module.ts`

匯入 new external REST module。

這會讓 gateway app 能接到 `/external/v1/withdrawal-intents`。

## Cradle Files

### 新增 `apps/esingpay-cradle/src/external/app.module.ts`

Cradle external boundary 的 root module。

職責：

- 匯入 external RPC server module。
- 匯入 Stage 1 需要的 external feature modules。
- 讓 external adapter wiring 與 fund / wallet root adapters 保持獨立。

### 新增 `apps/esingpay-cradle/src/external/rpc/server/server.module.ts`

聚合 cradle external RPC server adapters。

Stage 1 匯入 external withdrawal intent RPC server module。

### 新增 `apps/esingpay-cradle/src/external/rpc/server/withdrawal-intent/withdrawal-intent.module.ts`

註冊 external withdrawal intent RPC controller、service、mapper。

匯入 existing fund withdrawal intent context module，讓 external adapter 呼叫既有 fund capability，而不是重寫能力。

### 新增 `apps/esingpay-cradle/src/external/rpc/server/withdrawal-intent/withdrawal-intent.controller.ts`

接收 external withdrawal intent list 的 RestRpc message。

職責：

- 綁定 external withdrawal intent API RestRpc topic。
- 接收 typed RestRpc request。
- 委派給 external withdrawal intent service。

### 新增 `apps/esingpay-cradle/src/external/rpc/server/withdrawal-intent/withdrawal-intent.service.ts`

協調 external list request。

職責：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取出 merchant scope。
- 將 external search params 轉成既有 merchant-scoped withdrawal intent query。
- 呼叫既有 merchant withdrawal intent search capability。
- 將 use-case errors 轉成 external REST result codes。
- 回傳 external contract response envelope。

### 新增 `apps/esingpay-cradle/src/external/rpc/server/withdrawal-intent/withdrawal-intent.mapper.ts`

負責 external contract DTOs 與既有 fund application data 之間的 mapping。

職責：

- 解碼並 normalize external search params。
- 將 internal withdrawal intent composed/result data map 成 `ExternalWithdrawalIntentDto`。
- 隱藏 internal-only fields，例如 actor details、full status history、wallet allocation id、fee payer wallet strategy、internal network endpoint lifecycle。

### 修改 `apps/esingpay-cradle/src/app.module.ts`

匯入 new cradle external app module。

這會讓 cradle 啟動時註冊 external RPC server。

## 第一個 Request Walkthrough

Stage 1 request path 應如下：

```text
GET /external/v1/withdrawal-intents
  -> gateway external withdrawal-intent.controller
  -> gateway external withdrawal-intent.proxy
  -> cradle external withdrawal-intent.controller
  -> cradle external withdrawal-intent.service
  -> existing merchant-scoped withdrawal intent search capability
  -> cradle external withdrawal-intent.mapper
  -> ExternalWithdrawalIntentDto list response
```

## 重用策略

Stage 1 應重用：

- Existing RestRpcClient 與 RestRpc topic mapping infrastructure。
- Existing merchant-scoped withdrawal intent search capability。
- Existing fund withdrawal intent context module。
- Existing paging result pattern。

Stage 1 不應重用：

- Portal fund REST controller。
- Portal fund REST DTO 作為 external response DTO。
- Merchant user identity 作為 API key identity。
- Fund REST module 作為 external route owner。

## 驗證目標

Stage 1 完成時，應能證明：

```text
HTTP GET /external/v1/withdrawal-intents
  -> gateway external controller
  -> gateway external proxy
  -> cradle external RPC adapter
  -> existing fund withdrawal intent capability
  -> external DTO response
```

這會建立後續 stages 加上所有 read APIs、withdrawal create、API key、IP whitelist、Console、docs 時可沿用的結構。

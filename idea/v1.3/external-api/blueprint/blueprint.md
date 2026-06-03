---
status: draft
updated_at: 2026-06-03
updated_by: Codex
---

# External API — 藍圖

## 背景

External API 是一條新的商戶系統整合邊界，不是既有商戶後台 REST adapter 的延伸。

本藍圖對應 External API design：

- `GET /external/v1/deposits`
- `GET /external/v1/deposits/{id}`
- `GET /external/v1/withdrawal-intents`
- `GET /external/v1/withdrawal-intents/{id}`
- `POST /external/v1/withdrawal-intents`
- `GET /external/v1/wallets`
- `GET /external/v1/wallets/{id}`

External API 面向商戶系統整合，應隱藏內部後台流程、actor 細節、狀態歷程、wallet allocation、fee payer strategy、network endpoint lifecycle 等內部結構。

## 階段拆分

### Stage 1：External 邊界骨架

建立 External API 邊界的基本骨架：

- External REST contract rest/ DTO 撰寫。
- Gateway `src/rest/external` module。
- Cradle `src/external` module。
- `merchant-agent` identity 概念。
- 先接通一條 resource path，證明 gateway-to-cradle 流程可運作。

第一條驗證路徑使用：

- `GET /external/v1/withdrawal-intents`

### Stage 2：External 查詢 APIs

完成 read-only External APIs：

- Deposits 列表 / 詳情。
- Withdrawal intents 列表 / 詳情。
- Wallets 列表 / 詳情。
- Merchant scoping。
- External DTO mapping。

### Stage 3：External Withdrawal 建立

完成：

- `POST /external/v1/withdrawal-intents`。
- External create DTO 驗證。
- External request 到 merchant submit flow 的 mapping。
- External response DTO mapping。

第一版不做 idempotency。

### Stage 4：API Key

完成 API key 基礎能力：

- API key 建立。
- API key 列表。
- API key 刪除。
- API key 對應 merchant 解析。

第一版不做：

- API key scope。

### Stage 5：IP 白名單

完成 External API 來源限制：

- API key level IP whitelist 設定。
- IP whitelist 檢查。
- Client IP 解析規則。
- API Gateway 解析來源 IP 後，傳給 cradle verify RPC。
- Cradle 必須先確認 request 來源是可信 API Gateway，才可使用 gateway 傳入的來源 IP。
- 拒絕錯誤 mapping。

### Stage 6：Console

完成：

- 串接 external API key API。
- 建立 Console routing / 頁面。
- 測試 Console API key 建立、列表、刪除流程。

### Stage 7：文件網站

完成：

- 上線新的 external API 文件網站。
- 建立新的 docs domain。
- 使用 Markdown 產出網頁的文件框架工具。
- 撰寫 External API auth / IP whitelist / endpoint examples。

## 端到端流程

External request 的整體流程：

```text
Merchant system
  -> esing-pay-api-gateway /external/v1/*
  -> API key guard
  -> IP whitelist guard
  -> 由 API key 建立 merchant-agent identity
  -> RestRpcClient request
  -> esingpay-cradle src/external RPC server
  -> external adapter service
  -> fund / wallet 既有 use cases 或 query services
  -> external mapper
  -> external contract-rest DTO response
```

Gateway 是 HTTP / API key 邊界。Cradle external module 是 external contract adapter。Fund / wallet modules 持續擁有既有業務行為。

---

## 工程估時

估時以前述 blueprint、既有 codebase 複用程度，以及 AI agent 協作開發方式為前提。

| Stage | 估時 | 依據 |
| --- | ---: | --- |
| Stage 1 | 3 天 | 第一次建立 external boundary、contract-rest、gateway-to-cradle、identity 傳遞，樣板建立成本最高。 |
| Stage 2 | 3 天 | Read APIs 可大量沿用 Stage 1 樣板與既有 use case / query service，主要工作是 mapper、ownership、DTO。 |
| Stage 3 | 2 天 | 抽 shared submitter 與 external submit use case，改動集中，方向已明確。 |
| Stage 4 | 5 天 | API key 新資料模型、hash、create / list / delete、verify RPC、gateway guard、credential owner 決策，未知較高。 |
| Stage 5 | 7 天 | IP whitelist、Client IP 信任鏈、gateway header / IP 解析、cradle verify 整合、安全邊界確認，未知最高。 |
| Stage 6 | 2 天 | Console routing、API 串接、基本測試，功能面清楚。 |
| Stage 7 | 5 天 | Docs framework、domain、Markdown 文件，偏獨立，可在 review window 做。 |

| 範圍 | 估時 | 備註 |
| --- | ---: | --- |
| Stage 1-5 | 20 天 | Backend 主線。 |
| Stage 6-7 | 7 天 | 可放 PR review window 並行。 |
| 整體 calendar | 約 4 週 | 若 review feedback 較少，Stage 6 / 7 可吸收在等待窗口中。 |

等待 PR 時，會實作可同時並行的任務，例如 Stage 6 / Stage 7 會安排在 PR reviewing 的等待窗口。若手邊沒有其他可並行任務，等待時間超過一天，則會直接合併代碼繼續做下去。若 PR 有意見反饋，則會告知該 stage 需延長，並附上 PR 連結。

# Wallet Cashflow And Ledger Export CSV Implementation Plan EP-1

## 背景

本計畫要為以下四支既有 wallet 查詢 API，各自新增一支 `export to csv` API：

1. `plat-ledger`
2. `plat-cashflow`
3. `merch-ledger`
4. `merch-cashflow`

本階段只做同步 MVP：

- `data-file-service` 負責撈資料、組 CSV、產生暫存靜態檔
- `data-file-service` 上傳 Azure Blob 並取得可下載 URL
- API Gateway 對外提供 export API，回傳 URL 給呼叫端
- 呼叫端拿到的是單次 long request 的 `downloadUrl`，不是 job id
- 暫不處理 job queue

整體切法仍需保留未來演進成 job-based export 或其他雲端檔案儲存模式的空間。

## Current Context

目前四支查詢 API 已存在於 API Gateway：

- `GET /wallet/plat/ledger/items`
- `GET /wallet/plat/cashflow/items`
- `GET /wallet/merch/ledger/items`
- `GET /wallet/merch/cashflow/items`

其查詢鏈路為：

```text
API Gateway controller -> wallet proxy -> rest-rpc -> upstream service rest module
```

`data-file-service` 目前幾乎是空殼，`feature/wallet` 只有 `wallet.module.ts`，因此這一波可以直接依照既有 guide 建立乾淨的 feature / context / rest module 結構。

## MVP Key Decisions

### Export API 不併入既有 query contract

雖然四支查詢 API 的 filter 幾乎可以重用，但 export 的回應型別不是 paging list，而是 file URL result，因此應建立獨立 export contract，由 API Gateway 對接 `data-file-service`。

### 跨服務回傳 Azure Blob URL，而不是 inline file content

若 `data-file-service` 與 API Gateway 為不同 process / container，`data-file-service` 產生的本地路徑無法直接讓 API Gateway 使用。因此同步 MVP 雖然仍會先產生本地暫存檔，但跨服務回傳應是 blob URL，而不是檔案內容或 local path。

## 文件拆分

本主題已拆成以下文件：

- [`contract-and-public-api.md`](./contract-and-public-api.md)：export contract、DTO、public API path 與 response shape
- [`data-file-service.md`](./data-file-service.md)：feature structure、provider 責任、flow、batch fetch、timezone 與 temp file 策略
- [`api-gateway.md`](./api-gateway.md)：controller、proxy、query DTO、module wiring 與 permission 對接
- [`csv-column-and-mapping.md`](./csv-column-and-mapping.md)：row DTO、CSV 欄位規劃、row mapper 邊界與前端顯示語意對齊
- [`delivery-and-evolution.md`](./delivery-and-evolution.md)：error handling、檔案變更範圍、驗證清單與後續演進切點

CSV 欄位數值解析的既有依據文件：

- [`../deposit-withdrawal-console-export-field-mapping.md`](../deposit-withdrawal-console-export-field-mapping.md)
- [`../transaction-ledger-console-export-field-mapping.md`](../transaction-ledger-console-export-field-mapping.md)

## 建議閱讀順序

1. 先讀 [`contract-and-public-api.md`](./contract-and-public-api.md) 確認對外 contract。
2. 再讀 [`data-file-service.md`](./data-file-service.md) 與 [`api-gateway.md`](./api-gateway.md) 確認 provider 邊界與 transport wiring。
3. 實作 CSV 包裝前，讀 [`csv-column-and-mapping.md`](./csv-column-and-mapping.md) 與兩份 mapping baseline 文件。
4. 最後用 [`delivery-and-evolution.md`](./delivery-and-evolution.md) 作為實作與驗證清單。
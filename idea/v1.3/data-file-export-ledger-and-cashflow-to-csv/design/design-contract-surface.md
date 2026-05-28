---
status: draft
updated_at: 2026-05-28
updated_by: Copilot
---

# Design Contract Surface

## Context

本文件定義匯出能力的對外語意邊界，避免 query 與 export 在 contract 層混責。

## Capability Scope

本 topic 定義四支 export capability：

- platform ledger export
- platform cashflow export
- merchant ledger export
- merchant cashflow export

## Endpoint Semantics

四支 endpoint 路徑語意統一為既有資源路徑加上 export 子路徑，並以 `:format` 參數化：

- `/wallet/plat/ledger/items/export/:format`
- `/wallet/plat/cashflow/items/export/:format`
- `/wallet/merch/ledger/items/export/:format`
- `/wallet/merch/cashflow/items/export/:format`

第一版 `format` 僅支援 `csv`。

## Request Semantics

### Filter-only

export request 只承載條件篩選，不承載 `page/size/limit`。

原因：

- export 預設語意是「匯出符合條件的完整資料」
- 分頁屬 provider 內部執行策略
- 對外若暴露分頁，會破壞 export 語意一致性

### Platform vs Merchant Filter Boundary

- platform scope 可接受 `merchantIdIn`
- merchant scope 不接受 `merchantIdIn`

### Time Range Naming Convention

時間條件鍵統一採：

- `{fieldName}Gte`
- `{fieldName}Lt`

此命名在 query/export 需保持一致，避免前後端條件鍵漂移。

## Response Semantics

對外 response 為可下載 URL 資訊，不回 inline binary content。

最小語意欄位：

- `fileName`
- `contentType`
- `downloadUrl`
- `expiresAt`

此語意可支撐同步 MVP，也保留後續擴充到 `mode: job` 的演進空間。

## Rejected Alternatives

- 把 export 併入既有 query contract
- 對外回傳 local file path
- 對外直接回傳 CSV 內容

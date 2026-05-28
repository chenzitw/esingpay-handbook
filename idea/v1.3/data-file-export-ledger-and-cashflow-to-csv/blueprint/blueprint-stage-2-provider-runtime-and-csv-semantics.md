---
status: draft
updated_at: 2026-05-28
updated_by: Copilot
---

# Blueprint Stage 2 Provider Runtime And CSV Semantics

## Stage Goal

在不改動 Stage 1 contract 的前提下，確立同步 export runtime 的能力分層與 CSV 語意。

## Stage Scope

- source 批次拉取策略
- row mapper 角色與輸出責任
- csv writer 與 file delivery 的責任切分
- merchant 時區輸出語意
- 暫存檔與上傳刪檔語意

## Runtime Sequencing (Concept Level)

1. 收到 export request
2. 依條件批次抓取來源資料
3. 透過 mapper 轉成 row DTO
4. 輸出 CSV 實體檔
5. 上傳檔案並產生下載 URL
6. 最佳努力刪除暫存檔
7. 回傳 URL metadata

## Critical Decisions In This Stage

- row mapper 為正式流程，不在 writer 內做臨時轉換
- merchant 時間欄位用 merchant timezone，無值 fallback UTC
- 檔名僅用建立時間，不帶查詢條件或識別資訊

## Stage Exit Criteria

- 四支 export flow 共享同一套 runtime 責任分層
- ledger/cashflow CSV 輸出語意已對齊 references 基線
- runtime 不依賴 queue，仍保留演進切點

## Risks

- 大資料量同步匯出造成請求時間與記憶體壓力
- CSV 顯示語意與 console 畫面漂移

## Mitigation

- 保留 batch fetch 與 file delivery 抽象分層
- 以 references mapping 作為 row mapper 的語意基線

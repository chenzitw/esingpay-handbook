---
status: draft
updated_at: 2026-05-28
updated_by: Copilot
---

# Design Export Runtime Semantics

## Runtime Goal

在單次請求內完成 export，回傳可下載 URL，不引入 job queue。

## Runtime Capability Split

匯出流程分四個正式能力：

- source fetching
- row mapping
- csv writing
- file delivery

流程編排責任在 use-case，I/O 能力責任在 service。此分離是為了後續替換儲存媒介或演進為 job 模式。

## Source Fetching Semantics

- 對外不接受分頁參數
- 內部使用固定批次分頁拉取完整資料
- 每批次大小上限採固定值
- 批次屬 implementation detail，不進 public contract

## Timezone Semantics

- merchant 系列 export 的時間欄位，輸出時以 merchant 登記時區為準
- 若無可用時區，第一版 fallback 為 UTC
- platform 系列沿用既有標準時間語意

## Temp File and Upload Semantics

- 先在 provider 內建立本地暫存 CSV
- 完成上傳後再回傳 URL
- 上傳後做最佳努力刪除暫存檔

檔名規則：

- 格式 `wallet-{YYYYMMDDHHmmss}.csv`
- 不夾帶查詢條件或商戶識別資訊

## Error Semantics

第一版錯誤語意區分：

- identity mismatch
- params invalid
- upstream query failure
- file processing failure

原則：

- 不穿透上游內部錯誤細節
- 對外僅保留可行動或可判讀的錯誤型別

## Permission Semantics

- platform export 沿用 platform wallet 權限邊界
- merchant export 沿用 merchant wallet 權限邊界

第一版若無 export 專用 scope，可先沿用 list 類 scope。

## Evolution Hooks

第一版同步 MVP 需保留三個演進切點：

- job-based export
- cloud storage policy extension
- stream-based large dataset export

原則：

- 不改 public path
- 不改 filter contract
- 盡量只替換內部 orchestration 與 delivery strategy

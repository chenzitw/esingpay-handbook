---
status: draft
updated_at: 2026-05-28
updated_by: Copilot
---

# Wallet Cashflow And Ledger Export CSV — Design

## Purpose

此檔為 design 主索引，彙整本 topic 的概念層設計，並把內容拆分到 `design/` 子文件，降低單檔複雜度。

## Design Document Set

- `design/design-contract-surface.md`
- `design/design-export-runtime-semantics.md`
- `design/design-csv-output-semantics.md`

## Coverage Map Against EP-1 Draft

下列功能構想已納入 design 階段文件：

- 四支匯出能力範圍與同步 MVP 定位
- export 與既有 query contract 分離
- 回傳 blob URL 而非 inline 檔案內容
- 對外 endpoint 與 `:format` 語意
- export request filter-only 語意與 `Gte/Lt` 命名慣例
- file-url response 語意與後續 `url/job` 演進保留
- row mapper 作為正式流程，不直接 dump source DTO
- merchant 時區輸出語意與 UTC fallback
- source 批次拉取策略與內部分頁屬 implementation detail
- 暫存檔命名原則與上傳後刪檔原則
- 錯誤語意分層（identity / params / upstream / file processing）
- permission 對齊原有 platform/merchant 邊界
- future hooks（job、cloud storage、streaming）

## Boundaries

- 本 topic 僅停留在 Design 與 Blueprint。
- 不展開 Plan 檔案，不展開 codebase 路徑與實作步驟。
- CSV 欄位顯示語意以 `references/` 內既有 mapping 文件作為基線。

## Open Points

- 時間區間第一版實際開放欄位
- `expiresAt` 生成策略與 SLA
- 內部錯誤碼是否明確區分 system failure
- 同步模式容量上限與 timeout 預設值

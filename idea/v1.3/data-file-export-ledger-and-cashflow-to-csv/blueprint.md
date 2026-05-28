---
status: draft
updated_at: 2026-05-28
updated_by: Copilot
---

# Wallet Cashflow And Ledger Export CSV — Blueprint

## Purpose

此檔為 blueprint 主索引，提供分階段交付圖與文件導覽。內容僅停留在方向層，不展開 plan 細節。

## Blueprint Document Set

- `blueprint/blueprint-stage-1-contract-and-external-surface.md`
- `blueprint/blueprint-stage-2-provider-runtime-and-csv-semantics.md`
- `blueprint/blueprint-stage-3-verification-and-release-guardrails.md`

## Stage Overview

- Stage 1: export contract 與對外介面語意凍結
- Stage 2: provider runtime 與 CSV 語意對齊
- Stage 3: 驗證策略、風險護欄與交付門檻

## Coverage Map Against EP-1 Draft

下列草稿功能構想已納入 blueprint 階段：

- 四支 export endpoint 目標範圍與同步 MVP 交付策略
- `format` 參數化與 `csv` 首版限制
- filter-only request 與時間欄位命名慣例
- file-url response 為對外主語意
- source 批次拉取、mapper、csv、file delivery 的能力分層
- merchant 時區策略與 platform/merchant 邊界
- 暫存檔命名與刪檔原則
- 錯誤與 permission 的方向性規則
- 驗證情境（成功、空資料、大筆數、時區）
- 未來演進切點（job、blob、streaming）

## Boundaries

- 本文件與子文件僅描述 Blueprint 層的階段、依賴與決策，不產生 Plan artifacts。
- 不使用 codebase 檔案路徑作為決策依據。

## Open Points

- 各 stage 的完成判準是否需綁定版本里程碑
- sync 容量上限是否在 stage 2 即先定義保守值
- `expiresAt` 與 URL lifecycle 是否在 stage 1 先凍結一版最小規格

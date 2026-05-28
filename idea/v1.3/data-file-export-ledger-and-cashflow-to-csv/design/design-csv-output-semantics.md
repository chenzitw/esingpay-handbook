---
status: draft
updated_at: 2026-05-28
updated_by: Copilot
---

# Design CSV Output Semantics

## Baseline

本文件的欄位顯示語意以 `references/` 下既有 mapping 文件為基線，不直接修改該基線文件。

## Core Principle

CSV 輸出是「使用者閱讀語意」而不是「原始資料 dump」。

因此必須保留 row mapper 階段，處理：

- 欄位顯示轉換
- 枚舉值顯示文字
- 數值格式化
- 時間字串格式化

## Ledger Export Semantics

第一版 ledger CSV 以穩定一階欄位為主，涵蓋：

- 交易識別與商戶資訊
- 錢包資訊與幣別
- kind/realization/side 等交易語意
- fund case 與 network transaction 摘要資訊
- 主要時間欄位

規則：

- 顯示值優先，不以 raw 值為唯一輸出準據
- merchant 系列時間欄位做時區轉換
- 缺值欄位採一致 fallback 顯示策略

## Cashflow Export Semantics

第一版 cashflow CSV 以穩定一階欄位為主，涵蓋：

- 交易識別與商戶資訊
- 金額欄位（gross/net/fee/delta）
- correlation 與 subject 摘要欄位
- 主要時間欄位

規則：

- 避免第一版展開多層陣列結構
- 複雜欄位留待後續增量版本
- 顯示格式與 console 可讀性語意對齊

## Row DTO Role

row DTO 是 CSV writer 的唯一輸入語意，不讓 writer 直接依賴來源 item schema。

好處：

- mapping 規則集中可維護
- CSV writer 職責單一
- 方便後續替換 source 或擴充欄位

## Open Points

- 空資料集輸出是否固定保留 header row
- 欄位排序與欄位命名是否需嚴格對齊 UI 欄順
- 是否需要同時輸出原始值與顯示值雙欄位

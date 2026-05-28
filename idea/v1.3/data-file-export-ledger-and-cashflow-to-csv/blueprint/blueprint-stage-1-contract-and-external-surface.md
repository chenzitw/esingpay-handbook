---
status: draft
updated_at: 2026-05-28
updated_by: Copilot
---

# Blueprint Stage 1 Contract And External Surface

## Stage Goal

先凍結對外語意，避免後續 runtime 實作反向牽動 public contract。

## Stage Scope

- 四支 export endpoint 能力邊界
- `format` 參數化與首版 `csv` 限定
- filter-only request 語意
- response file-url 語意
- 時間條件命名慣例（`Gte/Lt`）

## Critical Decisions In This Stage

- export contract 與 query contract 分離
- 不暴露 paging 參數
- 對外結果是 URL metadata，不是 binary stream

## Stage Exit Criteria

- 四支 endpoint 語意一致且已定稿
- request/response 的語意欄位已定稿
- platform/merchant 的參數邊界與權限邊界已定稿

## Risks

- 若 contract 語意未先凍結，Stage 2 會出現實作驅動設計漂移

## Mitigation

- 以 stage 1 產物作為後續 stage 的凍結輸入

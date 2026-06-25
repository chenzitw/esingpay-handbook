---
status: active
updated_at: 2026-06-26
updated_by: Tim
---

# v1.3

## Scope

- KYT 風控規則與流程
- integration webhook 設計與實作
- integration api 設計與實作

## Backlog

| Item                                                                       | Feature                  | Status         | Notes                                                                                                                                                                                                   |
| -------------------------------------------------------------------------- | ------------------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RPC 調用 ergonomics（`ResultDto` 收斂 / 逐 code 錯誤處理 / caller helper） | architecture-improvement | 候選（待探討） | 由 [superjson-wire-codec proposal](architecture-improvement/superjson-wire-codec-proposal.md) 劃出的另案：該案解序列化稅，但 caller 端 result 判別 + error 翻譯樣板未收斂；是否值得另立 proposal 待判斷 |

## Spec persistence

| Feature                             | Target spec                                                                                        | Status                                         |
| ----------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| resource-id-codec 統一 + Sqids 改善 | [proposal](architecture-improvement/resource-id-codec-unification-proposal.md) → blueprint(待啟動) | proposal 原則接受(2026-06-26);待啟動 blueprint |

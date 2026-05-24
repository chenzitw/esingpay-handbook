---
status: draft
updated_at: 2026-05-24
updated_by: Claude
---

# Workflow — Closing

> 本文件是 `convention/workflow.md` 的 Closing 撰寫指引。先讀主檔可獲得整體框架。

## 用途

Closing 是版本生命周期的**收尾段**——從 implementation 完成到版本真正歸檔之間的整理工作。

類比餐廳：營業結束（implementation 完成）不等於收工；還有清潔、結帳、補貨（hygiene、backlog 收束、spec distillation）才能真正下班（closed）。

Closing 存在的價值：

- 版本內浮出的未決事項不會被遺忘
- 從 plan / code 學到的 pattern 被蒸餾進 `spec/` / `guide/`，未來複用
- 跨版本的決策軌跡可追溯

Closing 與前面四個 tier 的形狀不同——它不是「撰寫一份文件」的指引，而是「管理一個版本生命周期」的流程，搭配兩份輔助 artifacts：per-version `README.md` 與 handbook root 的 `VERSIONS.md`。

## 版本生命周期

每個 vX.Y 有三個狀態：

| 狀態          | 意義                              | 進入條件                                          |
| ------------- | --------------------------------- | ------------------------------------------------- |
| **in flight** | 開發活躍中                        | 版本啟動時                                        |
| **completed** | Implementation 完成、hygiene 尚未 | 主要 feature 都 merge、進入收尾階段               |
| **closed**    | Hygiene 也都做完                  | 該版本 README 的 backlog 與 spec persistence 歸零 |

狀態追蹤在 handbook root 的 `VERSIONS.md`。轉換是手動的，cadence informal——通常由 Tim 主導，不嚴格綁特定時間點。

**新版本開啟的觸發**：累積足夠新 feature scope、或對齊 system development milestone。沒有固定週期。

## Per-version README

每個 `idea/vX.Y/` 內必有 `README.md`，是該版本的**自包含狀態板**。三個區塊：

- **Scope**：本版本要處理的 feature 清單。
- **Backlog**：開發過程浮出、未在本版本解決的項目，含狀態（`open` / `cancelled` / `merged into vN.M` / `done`）。
- **Spec persistence**：本版本完成後該蒸餾進 `spec/` / `guide/` 的內容，含狀態（`pending` / `done`）。

Per-version README 是 closing 的主要工作介面——hygiene 與 distillation 都靠它列出待處理事項。

**Template**：

```markdown
---
status: draft
updated_at: YYYY-MM-DD
updated_by: <agent>
---

# vX.Y

## Scope

- [feature 清單]

## Backlog

| Item | Feature | Status | Notes |
| ---- | ------- | ------ | ----- |
|      |         |        |       |

## Spec persistence

| Feature | Target spec | Status |
| ------- | ----------- | ------ |
|         |             |        |
```

新版本啟動時用此 template 開檔，之後隨版本進展持續更新 Backlog 與 Spec persistence 兩表。

## VERSIONS.md

Handbook root 的 `VERSIONS.md` 是跨版本狀態總覽：

```markdown
# Versions

| Version | Status    | Notes      |
| ------- | --------- | ---------- |
| v1.0    | closed    | ...        |
| v2.0    | completed | hygiene 中 |
| v3.0    | in flight | ...        |
```

Status flip 由負責人手動更新。VERSIONS.md **不重複** per-version README 的細節——它只回答「哪個版本到哪個 milestone」。

（VERSIONS.md 在 handbook root、不在 `idea/` 內，不套用 frontmatter。）

## Hygiene flow

當版本進入 `completed` 但 README 還有 open backlog 或 pending spec persistence 時，需要做 hygiene round。

**步驟**：

1. 走過 README 的 **Backlog**：每項做 close / cancel / merge into 下個版本之一
2. 走過 README 的 **Spec persistence**：每項蒸餾進對應 `spec/*.md` 或 `guide/*.md`、標記 done
3. 兩表都歸零 → flip VERSIONS.md 該版本為 `closed`

**觸發時機**：通常是「要開新版本前」或「backlog 多到難管」。Cadence informal、不必排程。

### Cross-version backlog protocol

當 backlog 項目從 vX.Y merged into vN.M 時：

- **Origin 版本**（vX.Y）的 README backlog：標記 `merged into vN.M`
- **Target 版本**（vN.M）：**不重複記錄**

避免兩邊維護同一條目。Reader trace：vX.Y backlog → 看到 `merged into vN.M` → 翻 vN.M 的 plan → plan 的 Context preamble 接住。

## Distillation

Distillation 把 `idea/vX.Y/` 內已穩定的內容往兩個方向蒸餾：

- **架構性、跨版本仍會被引用的決策** → handbook 的 `spec/`
- **codebase 寫作 pattern、coding rules** → 該 codebase 的 `guide/`

**蒸餾不等於搬移**——`idea/vX.Y/` 內檔案永不搬走、永遠留在原版本作為歷史記錄。Distillation 是把「資訊精華」濃縮到 spec / guide 作為當前 source of truth。

**判斷該不該蒸餾**：

- 跨版本仍會被引用 → 蒸餾
- 純粹本版本特有、之後不再用 → 不蒸餾
- 可能還會變動 → 等穩定再蒸餾

蒸餾出的內容會在原 idea 文件保留（歷史記錄）+ 在 spec / guide 出現（current truth）。兩處內容自然分歧時，**spec / guide 為準**——idea 是歷史，spec / guide 是現況。

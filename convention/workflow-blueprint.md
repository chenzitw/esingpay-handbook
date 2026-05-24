---
status: draft
updated_at: 2026-05-24
updated_by: Claude
---

# Workflow — Blueprint

> 本文件是 `convention/workflow.md` 的 Blueprint tier 撰寫指引。先讀主檔可獲得整體框架。

## 用途

Blueprint 銜接 design 與 plan——把概念層的「做什麼」轉成方向層的「怎麼推」。它做三件事：

1. 把 design 切成可序列化的 deliverable（stage）
2. 定下要先決的關鍵結構決策（schema shape、contract surface、infra choices）
3. 排定 stage 間與外部依賴的順序

Blueprint 不直接驅動 code 改動；它讓 plan 作者能直接動手——把該先決的決定全先決掉，plan 階段只剩落地。

## 何時需要、何時可略

Blueprint 是 **optional**。要與不要看下游 plan 是否能無痛展開。

**需要 blueprint 的訊號**：

- Design 涵蓋多個可獨立交付的階段，stage 切點影響工程進度
- 有跨多 plan 共用的結構決策（schema 形狀、contract envelope、event topic 命名規則）
- 引入新 infra 或新 pattern，需在 plan 之前決定關鍵選型
- 跨 codebase / 跨 team 的協調

**可略 blueprint 的訊號**：

- 單一 plan 即可完成
- 循既有 pattern、無新結構決策
- 改動小、序列化判斷顯然

判斷準則：寫 blueprint 是不是只為了「儀式上經過 blueprint 階段」？若是，跳過。

## 應涵蓋什麼

- **Stage 切分**：若多階段，列出 stage 及各自交付物。
- **關鍵結構決策**：schema 形狀（單表 / 多表 / 嵌入式...）、contract surface（envelope 結構、命名）、infra 選型（queue 形式、序列化方式）。
- **依賴與順序**：stage 間依賴、與外部 feature 的依賴、必須先完成的前置條件。
- **本次 delivery 的範圍**：哪些 stage 屬本次、哪些遞延後續版本。
- **承擔風險的決策**：值得提前定案、避免 plan 階段才發現的選擇。
- **Open points**：留給 plan 解決的具體細節（schema 欄位型別、命名等）。

## 不該包含什麼

- 具體 SQL 欄位型別、索引定義
- File path、class / function 名稱
- Step-by-step 實作步驟
- Codebase 現況查驗（既有檔案在哪、長怎樣）
- 大段 code snippet

可包含的「結構決策」與不該包含的「實作細節」差別在抽象度：blueprint 講「schema 走單表 + role discriminator」，plan 講「`wallet_allocation_line` 表加 `role varchar(50) NOT NULL` 欄位」。前者是方向，後者是落地。

注意：「不該包含」指的是 blueprint 文件的**內容**，不是禁止 blueprint 作者查 codebase。需驗證既有 schema、pattern、infra 假設時，可請 CLI 端做 codebase survey，把事實當 blueprint 的 input。

## Self-test

**Sequencing 自測**：stage 是否真能獨立執行？stage N+1 能否在 stage N 完成後立刻開工，不需要回頭補 stage N 缺漏？

**Decision sufficiency 自測**：plan 作者能否依此 blueprint 開始寫 plan，不需要回頭追問結構決策？

兩項都過 → blueprint 完備。

## 起手結構示意

Blueprint 的結構視 topic 形狀差異大，沒有單一固定樣板。下方是常見起手骨架，**按 topic 增減重塑**：

```markdown
---
status: draft
updated_at: YYYY-MM-DD
updated_by: <agent>
---

# <Topic> — Blueprint

## Context

[link to design、本次 delivery 的目的]

## Scope

[本次 blueprint 涵蓋的 stage、遞延的部分]

## Critical decisions

[schema 形狀、contract、infra 選型、命名規則]

## Stage breakdown

- Stage 1: ...
- Stage 2: ...

## Dependencies / sequencing

[stage 間、與外部 feature]

## Open points

- ...
```

依 topic 取捨：

- **單一 stage blueprint** — 砍掉 `Stage breakdown`、簡化 `Dependencies`，主軸是 `Critical decisions`。
- **跨 codebase 協調 blueprint** — 加 `Coordination matrix`，列出哪個 codebase 負責哪段。
- **純決策型 blueprint**（例：選 message queue 形式）— 主結構是 `Decision matrix`（options × criteria × choice），其餘節從簡。
- **多 stage 拆分到獨立檔案**（`<topic>-blueprint-stage-N.md`）— master file 留 `Stage breakdown` + `Dependencies`，各 stage 細節進獨立檔。適用於 stage 內含獨立決策、各 stage 有不同 reviewer 時。

**Agent note**：變體為參考、非必填；按 topic 取捨，回 self-test 判斷。

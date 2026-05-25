---
status: draft
updated_at: 2026-05-24
updated_by: Claude
---

# Workflow — Blueprint

> 本文件是 `convention/workflow.md` 的 Blueprint tier 撰寫指引。先讀主檔可獲得整體框架。

## 用途

Blueprint 銜接 design 與 plan——把概念層的「做什麼」轉成方向層的「怎麼推」。它做四件事：

1. 把 design 切成可序列化的 deliverable（stage）+ 列出各 stage 內的 plan inventory
2. 定下要先決的關鍵結構決策（schema shape、contract surface、infra choices）+ 跨 plan 共用詞彙
3. 排定依賴、parallelism、delivery cadence
4. 明示 landed facts、open points、pattern gap

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

**Scope & inventory**：

- **Stage 切分**：若多階段，列出 stage 及各自交付物。
- **Plan inventory per stage**：每個 plan 用 named concept 寫 scope（entity、use case、layer），不到 file / method 細節；標註 plan 間依賴。
- **本次 delivery 的範圍**：哪些 stage 屬本次、哪些遞延後續版本。

**Cross-plan structure**：

- **關鍵結構決策**：schema 形狀（單表 / 多表 / 嵌入式...）、contract surface（envelope 結構、命名）、infra 選型（queue 形式、序列化方式）。
- **跨 plan 共用詞彙與命名規則**：shared error enum scope、新引入 helper 的命名 convention、跨 plan 一致的 input/output 形式（如 short-form vs long-form RPC input）。Blueprint 鎖定避免 plan 間 drift。
- **新 pattern flagging**：當需要的 pattern 在 `guide/` 不存在，blueprint 點名「establishing new pattern X」並給大致形狀（mirror 哪個既有實踐、採什麼結構），讓 plan 不需憑空判斷。日後 pattern 穩定再蒸餾進 guide。

**Orchestration**：

- **依賴與順序**：stage 間依賴、與外部 feature 的依賴、必須先完成的前置條件。
- **Parallelism opportunities**：哪些 plan 可並行起作、哪些必須序列。基於邏輯依賴推導；具體 race condition 留 plan。
- **Delivery cadence**：staging deploy 節奏（例：「stage 1 全部完成 → staging deploy」）、release 策略。

**Context & open points**：

- **Landed facts assumed**：起草前已成立的事實——既有 entity 形態、既有 pattern、跨 feature 已 land 的東西。引用 codebase survey 結論，不重述全貌。
- **承擔風險的決策**：值得提前定案、避免 plan 階段才發現的選擇。
- **Open points**：留給 plan 解決的具體細節（schema 欄位型別、命名等）。

## 不該包含什麼

- 新 class / function 命名（留給 plan + guide convention）
- File path、目錄結構
- Method 簽名、parameter 型別、回傳型別
- DB 欄位型別、index 定義、migration SQL
- Step-by-step 實作順序
- 具體 test case 列表
- 大段 code snippet

**可以提及的 named thing**：既有被改動的 class（point at existing things）、design 已定義的概念（如 `feePayerWalletId` 是 design 決的欄位概念）、subsystem / layer 名（「submit use cases」「REST adapter」）。Blueprint 引用、不創造新名。

**為什麼 blueprint 不命名 class / 不寫 file path**：class 分層與命名品質由 `guide/`（持久 convention）+ plan（個案實例化）共同維持。Blueprint 介入這層會（1）與 plan 重複職責、（2）一旦 plan 階段發現更好的 precedent，整份 blueprint 都要 sync、（3）偷走 plan 的 abstraction 層次。Blueprint 對品質的責任在「pattern gap flagging」與「cross-plan vocabulary anchoring」兩處（見「應涵蓋什麼」），不該再往下伸。

注意：「不該包含」指的是 blueprint 文件的**內容**，不是禁止 blueprint 作者查 codebase。需驗證既有 schema、pattern、infra 假設時，可請 CLI 端做 codebase survey，把事實當 blueprint 的 input。

## Self-test

**Sequencing 自測**：stage 是否真能獨立執行？stage N+1 能否在 stage N 完成後立刻開工，不需要回頭補 stage N 缺漏？

**Decision sufficiency 自測**：plan 作者能否依此 blueprint 開始寫 plan，不需要回頭追問結構決策？

**Granularity 自測**：這條決策若改變，後面 plan 還能不能正確展開？

- 不能 → 是結構性、blueprint 該寫
- 能 → 是 plan 落地細節、別搶

三項都過 → blueprint 完備。

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

- In scope: ...
- Out of scope: ...

## Landed facts assumed

[依賴的既有狀態、references to codebase survey]

## Plan inventory

- Plan-1: [scope at named-concept level, dependency]
- Plan-2: [scope, dependency]
- ...

## Critical decisions

[schema 形狀、contract surface、infra 選型]
[跨 plan 共用詞彙與命名規則]
[Establishing new pattern X（若 guide 未涵蓋）]

## Sequencing

- Dependencies: ...
- Parallelism: ...
- Delivery cadence: ...

## Open points

- ...
```

依 topic 取捨：

- **單一 stage blueprint** — 砍掉 `Plan inventory` / `Sequencing` 內的多 plan 內容、簡化 `Stage breakdown`、主軸是 `Critical decisions`。
- **跨 codebase 協調 blueprint** — 加 `Coordination matrix`，列出哪個 codebase 負責哪段。
- **純決策型 blueprint**（例：選 message queue 形式）— 主結構是 `Decision matrix`（options × criteria × choice），其餘節從簡。
- **多 stage 拆分到獨立檔案**（`<topic>-blueprint-stage-N.md`）— master file 留 `Stage breakdown` + `Sequencing`，各 stage 細節進獨立檔。適用於 stage 內含獨立決策、各 stage 有不同 reviewer 時。

**Agent note**：變體為參考、非必填；按 topic 取捨，回 self-test 判斷。

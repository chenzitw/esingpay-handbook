---
status: draft
updated_at: 2026-05-24
updated_by: Claude
---

# Workflow — Design

> 本文件是 `convention/workflow.md` 的 Design tier 撰寫指引。先讀主檔可獲得整體框架。

## 用途

Design 捕捉一個變動的**概念層樣貌**——做什麼、為什麼、與系統其他部分的關係——獨立於任何特定的實作方式。

Design 是 cascade 的起點，其輸出對 blueprint 與 plan 凍結；因此 design 的清晰度與穩定度直接決定下游品質。一份 design 寫一次、被多次閱讀（blueprint 階段、plan 階段、未來引用者、跨 AI peer review），值得多花時間把概念框準確。

## 應涵蓋什麼

- **範圍與目的**：這次改動要達成什麼、為何在現在處理、適用對象。
- **領域實體與行為**：新增或變動的 entity / aggregate / process，它們的職責與互動。「實體」依 topic 取概念對應——infra 引入可能是 queue / topic / consumer 等；protocol design 可能是 message 結構與 routing。
- **系統層約束**：相關的不變量、邊界條件、依賴假設。例：必須在單一 transaction 內完成、不允許 cross-BC 同步呼叫。
- **In-scope / out-of-scope**：明確切出本次不處理的部分。防止 scope creep、也提示 downstream「這個面向待後續」。
- **被排除的替代方案**：考慮過但未選的方向 + 理由。避免未來讀者問「為什麼不直接 X？」、也讓 cross-AI review 看得到判斷過程。
- **Open points**：留給 blueprint 或 plan 解決的未決問題。明示給下游、不藏在文中。

## 不該包含什麼

- 具體檔案路徑、class / function 名稱
- Step-by-step 實作順序
- DB 欄位型別、具體 schema 定義
- 大段 code snippet（必要 illustration 用、極少量）

這些屬於 plan 層。出現在 design 是訊號：要嘛該往 plan 移、要嘛 design 越界了。

注意：「不該包含」指的是 design 文件的**內容**，不是禁止 design 作者查 codebase。需驗證假設（既有 entity 怎樣建模、某 pattern 是否已存在等）時，可請 CLI 端做 codebase survey，把事實當 design 的 input。Codebase 是 design 的養分來源，不是 design 的 substance。

## Self-test

兩個角度檢查：

**抽象層自測**：兩位能力相當的 implementer 能否依此 design 寫出**結構不同但都滿足設計**的程式碼？

- **能** → 文件停在 design 層，正確。
- **不能**（只有一種實作方式可行）→ 已跌入 plan 領域，把細節移到 plan、design 拉回概念層。

**完整度自測**：blueprint 作者能否依此 design 開始切階段，不需要回頭追問核心概念？

- **能** → design 完備。
- **不能**（仍有「這裡到底是什麼意思」「沒講某個 entity」之類的洞）→ design 還沒寫夠。

## 起手結構示意

Design 不強制固定章節結構——feature、refactor、infra 引入、protocol 設計、concept introduction 各有重心。下方是 feature design 常見的起手骨架，**作為起點而非樣板**，按 topic 增減重塑：

```markdown
---
status: draft
updated_at: YYYY-MM-DD
updated_by: <agent>
---

# <Topic> — Design

## Context

[為何此改動、其在系統中的位置]

## Scope

[in / out of scope]

## Domain model

[新增或變動的 entity、行為、關係]

## Constraints

[系統層約束、不變量]

## Rejected alternatives

- ...

## Open points

- ...
```

不同 topic 重心會偏移，舉例：

- **Refactor design** — 主軸轉為 `Current state` / `Target state` / `Migration strategy` / `Backward compatibility`；Domain model 可能無變動。
- **Infra 引入**（如 Kafka 採用）— 加 `Capacity / scaling` / `Operational concerns` / `Failure mode`。
- **Protocol design** — 重心在 `Contract surface` / `Versioning` / `Compatibility matrix`。
- **Concept introduction**（如 Fund Case）— 三節即可：`Context` / `Concept` / `Implications across the system`。

**Agent note**：變體為參考、非必填；按 topic 取捨，回 self-test 判斷。

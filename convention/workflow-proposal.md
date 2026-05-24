---
status: draft
updated_at: 2026-05-24
updated_by: Claude
---

# Workflow — Proposal

> 本文件是 `convention/workflow.md` 的 Proposal tier 撰寫指引。先讀主檔可獲得整體框架。

## 用途

Proposal 在 cascade 最前端，回答「**該不該做**」——在投入 design / blueprint / plan 之前，先論證一個方向是否值得採納。

Proposal 與其他 tier 有三個結構性差異：

- **輸出不是凍結 input**：proposal 的「acceptance」本身才是觸發 design 的起點。Proposal 未通過時，下游不啟動。
- **有 decision lifecycle**：proposal 寫完不代表完成；要走過審議、做出決議。Design / blueprint / plan 寫完即完成。
- **下游可變**：通過後可能啟動 design / blueprint / plan，也可能只更新 `spec/` / `guide/` / convention，或兩者皆有。

寫 proposal 的關鍵心態：你在論證一個方向、不是宣告事實。Reader 必須能依文件取得自己的判斷依據，包括反對的理由。

## 何時需要、何時可略

Proposal 是 **optional**。多數 feature 共識清楚、直接從 design 起點。

**需要 proposal 的訊號**：

- **Infrastructure 引入**——導入新 queue / cache / serialization library / observability stack 等
- **Convention 變更**——調整既有 RPC envelope、event 命名、coding pattern、跨服務協定
- **跨 feature 的概念引入**——引入會被多個 feature 共用的概念或抽象（如某種架構級的 entity 模型）
- **方向不明確或團隊有分歧**——當「該往哪走」本身需要被論證，而非「怎麼做」

**可略 proposal 的訊號**：

- 方向已在過去討論中對齊
- 影響範圍鎖在單一 feature 內
- 沒有引入新概念、新 infra、或變動既有 convention

判斷準則：「這個變動的方向需要被說服嗎？」需要 → 寫 proposal。不需要 → 直接 design。

## 應涵蓋什麼

- **Context**：當前狀態、問題或機會、為何現在處理。Reader 要在這裡建立背景。
- **Proposal 主張**：具體在提議什麼。一句話可講清楚、整篇文件論證它。
- **Motivation**：為何選此方向。包括問題嚴重性、機會大小、不做的後果。
- **Trade-offs**：採納的好處、付出的代價、不確定性、風險。**誠實列出 cons**，proposal 不是推銷。
- **Rejected alternatives**：考慮過但未選的方向 + 為何不選。包括「不做任何事」這個 alternative。
- **Scope of impact**：通過後會觸發什麼下游——啟動哪些 design / blueprint、更新哪些 spec / guide / convention。讓 reviewer 看得到 ripple effect。
- **Decision request**：明示在請求什麼決議。多數時候是「accept this direction」，但有時是「在 A / B 兩個 alternative 之間選一個」。

## 不該包含什麼

- 具體 file path、class / function 名稱
- Step-by-step 實作步驟
- DB schema 欄位定義
- 假設已通過的語氣（「我們會做 X」），應為「提議做 X」

這些屬於 design 與 plan 層。Proposal 越界到實作細節是常見失誤——讓 reviewer 焦點被「怎麼做」吸走、忽略「該不該做」的核心問題。

**必要 illustration 例外**：當特定 schema 形狀、step 順序、欄位類型是論證的核心舉例時，可以提及——但僅作為說明、篇幅次於論證本身。判準：若移除這些細節後 proposal 仍能論證、reviewer 仍能判斷，這些細節是越界；若不能，是必要 illustration。

注意：與 design / blueprint 同——「不該包含」指 proposal 文件的**內容**，不是禁止 proposal 作者查 codebase。需驗證現況或了解既有 pattern 時，可請 CLI 端做 codebase survey 取得事實。

## Self-test

**判斷依據自測**：reader（特別是決策者）能否從文件本身做出 accept / reject 判斷，不需私下追問？

- **能** → proposal 完備。
- **不能**（reader 需問「實際代價多少」「會影響哪些 feature」之類）→ 某個 should-cover section 寫得不夠。

**Trade-off 誠實度自測**：列出的 cons 是否真實有重量？

- **是**（至少一條 con 可能讓人改變立場）→ 誠實。
- **否**（全是 weak cons、看不出採納成本）→ proposal 在推銷而非論證；重寫 trade-offs 與 rejected alternatives。

## Decision

Proposal 完成審議後，在文末加 `## Decision` 段，記錄結果。**內容格式不規範**——可以是一段話、一個小表、bullet 清單，依議題複雜度決定。常見可寫的元素：

- **判決**：accepted / rejected / superseded by ...
- **日期**
- **決策者 / 參與審議者**
- **理由**：為何 accept 或 reject，特別是當理由與 proposal 原本論證有出入時
- **後續處置**：accept 後啟動哪些下游（design 名單、spec / guide 修改項）

Decision 完成後：

- **Accepted** → 將後續處置追蹤到 per-version README 的 Spec persistence 表
- **Rejected** → proposal 保留原位，不另外移動或追蹤；它本身就是「這個方向被討論過、不採納」的歷史記錄

Frontmatter `status` 反映文件本身的編輯狀態（draft / 穩定後拿掉），與 Decision 段的 accepted / rejected **是不同維度**：proposal 已完成審議、文件穩定，但 Decision 可能是 rejected——這種情況 frontmatter 不需要 `status: draft`，文件穩定即可。

## 起手結構示意

Proposal 結構視 trigger 型態差異大，沒有單一固定樣板。下方是通用骨架；**作為起點而非樣板**，按 topic 增減重塑：

```markdown
---
status: draft
updated_at: YYYY-MM-DD
updated_by: <agent>
---

# <Topic> — Proposal

## Context

[當前狀態、問題或機會]

## Proposal

[一句話講清楚提議什麼]

## Motivation

[為何此方向、不做的後果]

## Trade-offs

- Pros: ...
- Cons: ...
- Unknowns: ...

## Rejected alternatives

- <方案 A>：[為何不選]
- 不做任何事：[為何不選]

## Scope of impact

[通過後觸發哪些下游]

## Decision request

[請求什麼決議]
```

以下舉幾個常見 trigger 為例，列出該議題下通常被加重論證的面向。實際 topic 可能跨多型態或不在下列之內，**按需取捨**。

| Trigger 範例         | 該議題下通常加重的面向                                                     |
| -------------------- | -------------------------------------------------------------------------- |
| Infra 引入           | Selection analysis（候選方案比較）、Operational impact、Capacity / scaling |
| Convention 變更      | Migration cost、Compatibility / coexistence、Affected surface              |
| Concept introduction | Concept definition、Relation to existing model、Where it applies           |

未來遇到其他類型 trigger，可直接補上 row。

**Agent note**：變體為參考、非必填；按 topic 取捨，回 self-test 判斷。

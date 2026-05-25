---
status: draft
updated_at: 2026-05-24
updated_by: Claude
---

# Workflow — Plan

> 本文件是 `convention/workflow.md` 的 Plan tier 撰寫指引。先讀主檔可獲得整體框架。

## 用途

Plan 是 cascade 終點——把 design（+ blueprint）的概念與方向，轉成可機械執行的步驟。Plan 假設上游凍結，自己負責落地完成度。

Plan 的角色跟前兩 tier 不同：

- **與 codebase 強耦合**：寫具體 file path、schema 欄位、既有 pattern 引用
- **唯一允許實作細節的層**：design / blueprint 禁止的，plan 反而必須具體寫出
- **對下游（implementation）不再凍結**：implementer 拿到 plan 後直接執行，plan 是「給 code 之前的最後一站」
- **是 `guide/` convention 的個案實例化**：plan 不重新發明命名、分層、error pattern、transaction 使用等規範，而是按 guide 既有 convention 落地。需 deviate 時顯式記錄 rationale。

寫 plan 的最佳環境是 CLI（直接 codebase 存取）。若不得已在 web 寫，需要先有 codebase survey 文件補足現況查驗。

## 應涵蓋什麼

**Context & preconditions**：

- **Context preamble**：簡述驅動本 plan 的 design / blueprint 來源與 scope，讓 plan 對未來讀者自包含。
- **Landed Facts Assumed**：本 plan 起草前已成立的事實——已 land 的相關 plan、既有 entity 形態、依賴的 cross-feature 既成事實。讓未來讀 plan 的 agent 知道「起點假設」、不必重新 verify。
- **Codebase verification**：相關既有檔案、pattern、命名慣例的現狀。後續 Steps 才能精確指向。

**Specifications**：

- **逐檔變動規格**：每個被觸及的 file，新增 / 修改 / 移除什麼。
- **Schema / contract 具體形狀**：欄位名、型別、constraint、index、enum 值等。
- **Frozen Decisions**：本 plan 收斂的決議列表（D1 / D2 / D3...），含 rationale。對 implementer 的價值是「這些點不必重新思考、已鎖定」。
- **Steps**：機械可執行的順序，內含 `Step 1`、`Step 2`...。每個 Step 可被 implementer 直接照做。
- **Test plan**：哪些 case 要覆蓋、新增還是補既有測試、unit / integration 邊界。

**Boundaries & exceptions**：

- **Open points / known compromises**：少數實作層仍未決的選擇（library X vs Y 之類），或上游 escalation 後接受的妥協，**顯式列出**。
- **Adjacent Doc Drift**：已知與其他 doc（roadmap、其他 plan、guide）不一致、但本 plan 刻意不處理的 drift。寫出來避免別的 agent「順手幫忙修」其實是刻意保留的不一致。
- **Escalation resolutions（如有）**：本 plan 過程中向上 escalate 過的議題、最終決議與後續處置（見下方 Escalation 節）。

## 不該包含什麼

- 重新 redesign upstream 決策（走 escalation，見下節）
- 與 design / blueprint 矛盾的方向（同上）
- 含糊或要 implementer 自己決定概念的步驟（plan 還沒寫夠 / 上游有缺）
- 開放性的方向問題（這屬於 design / blueprint 的 Open points）
- 重述 `guide/` 既有 convention（plan 是 instantiation，不是 reproduction）

Plan 寫到一半若想 redesign，停下、走 escalation；不要在 plan body 偷偷改方向。

## Self-test

**Mechanical 自測**：一個 implementer 能否依此 plan 機械執行，不需要做概念決定？

- **能** → plan 完備。
- **不能**（遇到需要判斷「這該怎麼做」的點）→ 不是 plan 寫不夠細，就是上游有缺。釐清屬哪一邊：plan 細節缺就補；上游缺就走 escalation。

**Closure 自測**：所有 upstream-frozen 的點都被處理？所有開放點都有明確標記（escalation 結論 / known compromise / adjacent drift）？

兩項都過 → plan 可進 implementation。

## Escalation：識別與處理

Plan 階段最常遇到的失敗模式：發現衝突、不確定屬於 design 漏寫還是 plan 自己寫得不對。**判斷準則**：

| 訊號                                                                  | 屬性         | 處置                                        |
| --------------------------------------------------------------------- | ------------ | ------------------------------------------- |
| 同一個 design 假設、兩種同樣合理的實作方式，需要選一條                | Plan 層      | plan 內決定、記錄理由（→ Frozen Decisions） |
| Design 假設了 X，但 codebase 實況 / 新需求否定 X                      | Design 層    | **escalation**                              |
| Blueprint 定的 schema 形狀，落到 plan 才發現缺一個欄位類型支撐某 case | Blueprint 層 | **escalation**                              |
| Design 沒講某 entity 在某 edge case 的行為，plan 需要才能繼續         | Design 層    | **escalation**                              |

**Escalation 流程**：

1. **暫停 plan-writing**——不要在 plan body 偷偷改方向
2. **顯式描述衝突**：哪個 design / blueprint 假設、與什麼衝突、影響哪些 plan 段落
3. **等決策者指示**：可能是回去改 upstream，或接受為 plan 內的 known compromise
4. **記錄結論**：plan 內加一段 `Escalation resolutions` 記下衝突、判決、後續處置

這個流程保護 cascade 完整性，也讓未來讀 plan 的人看得到「為什麼這裡跟 design 不太一樣」。

## 起手結構示意

Plan 結構視 plan 規模（單檔 vs 多 phase）與 topic 差異大。下方是單檔 plan 常見骨架，**按需取捨**：

```markdown
---
status: draft
updated_at: YYYY-MM-DD
updated_by: <agent>
---

# <Topic> — Plan

## Context preamble

[link to design / blueprint、本 plan scope]

## Landed Facts Assumed

- [已 land 的 plan / 既有 entity 形態 / 跨 feature 既成事實]

## Codebase verification

[既有相關 file、pattern、命名慣例]

## Frozen Decisions

- **D1**: [決議] — [rationale]
- **D2**: [決議] — [rationale]

## Changes overview

[逐檔變動摘要]

## Schema / contract

[具體欄位、型別、constraint]

## Steps

### Step 1: <action>

[具體做法]

### Step 2: <action>

[具體做法]

## Test plan

[case 覆蓋、邊界]

## Open points / known compromises

- ...

## Adjacent Doc Drift

- [已知 drift、本 plan 不處理]

## Escalation resolutions（如有）

- ...
```

依規模與 topic 取捨：

- **Multi-phase plan（`<topic>-plan-phase-M.md`）** — 各 phase 獨立檔，master plan（若有）僅留 `Phase breakdown` + `Dependencies`，各 phase 細節進獨立檔。
- **Stage + phase plan（`<topic>-plan-stage-N-phase-M.md`）** — stage master 列出 phase，各 phase 檔有完整 Steps。
- **純 schema 變更 plan** — 砍 `Steps`、用 `Migration order` 取代；強化 `Schema / contract` 與 `Backward compatibility`。
- **純 refactor plan** — 加 `Before / after`，Steps 著重「動什麼、不動什麼」。
- **無 schema 變更的 plan** — 砍 `Schema / contract`、Steps 為主。

**Agent note**：變體為參考、非必填；按 topic 取捨，回 self-test 判斷。

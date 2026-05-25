---
status: draft
updated_at: 2026-05-24
updated_by: Claude
---

# Workflow

本文件描述本專案的開發工作流：**Progressive Spec-Driven Workflow (PSDD)**。PSDD 是團隊在現階段共識下的務實做法——以文件為主軸、漸進地把概念落地為可執行的實作。隨著專案成熟，目標是逐步演進到正式的 Spec-Driven Development（結構化 spec 格式、由 spec 生成 code 與 test、自動化 spec-code 一致性驗證)。

本文件只談工作流的**整體形貌**。各環節的細節(怎麼寫 design、怎麼推 plan、怎麼收尾)放在 `convention/workflow-*.md` 各自的子文件。

## Cascade tiers

工作從方向到落地實作分四層，每層的主軸、性質、所在 repo 不同：

| Tier                         | 主軸        | 性質                             | Repo     |
| ---------------------------- | ----------- | -------------------------------- | -------- |
| **Proposal** — 方向提案      | SHOULD      | 方向主張與決策請求               | handbook |
| **Design** — 概念研究設計    | WHAT        | 概念模型理論                     | handbook |
| **Blueprint** — 開發規劃藍圖 | HOW（方向） | 基於概念理論與開發規範的執行綱要 | handbook |
| **Plan** — 落地實作計劃      | HOW（具體） | 基於 codebase 的具體執行步驟     | codebase |

Proposal 與 Blueprint 為 **optional**；Design 與 Plan 必寫。

四層是**類型**，不編號（口語講「design 階段」可以；正式 `Phase N` 用在下方檔級序列）。每一層的輸出對下游凍結，挑戰上游必須回到該層處理——不可在原地 redesign（見下方 Plan 段的 escalation 規則）。

逐層說明見下面四節。

## Proposal

Proposal 回答「**該不該做**」——在 design 之前先論證一個方向值不值得採納。

Proposal 是 **optional**——多數 feature 直接從 design 起點。觸發場景例：infrastructure 引入（Kafka）、convention 變更（SuperJSON）、跨 feature 的概念引入（Fund Case）。

Proposal 通過後的下游不固定：

- 啟動 design / blueprint（feature 性質的提案）
- 只更新 `spec/` / `guide/` / convention（純規範變更）
- 兩者皆有（例：Kafka 採用觸發 `guide/` 更新，並影響後續 feature design）

建議 AI 共筆環境：**web**。權衡分析與方向論證需要 deliberation，不該被 codebase 細節打斷思考。

詳見 `workflow-proposal.md`。

## Design

Design 回答「**做什麼**」——範圍、新增或變動的領域實體、它們的行為與關係、明確排除的、為何選此路。Design 必寫；每個 feature、refactor、infra 引入都從 design 起點。

建議 AI 共筆環境：**web**。Design 是抽象框定，不從 code 反推產生；若需驗證假設或了解既有狀態，可請 CLI 端做 codebase survey 取得事實。

詳見 `workflow-design.md`。

## Blueprint

Blueprint 回答「**怎麼推**」——把 design 切階段、定關鍵結構決策（schema、contract、infra）、排依賴。

Blueprint **可選**：當 implementation 從 design 已可直接推導，寫 blueprint 反而是 over-engineering。

建議 AI 共筆環境：**web**。Blueprint 仍屬策略思考，不需直接 codebase 存取；若需查驗既有 schema、pattern、infra 假設，可請 CLI 端做 codebase survey。

詳見 `workflow-blueprint.md`。

## Plan

Plan 回答「**怎麼改 code**」——檔案路徑、schema 欄位與型別、step-by-step 順序、測試計劃、codebase 現況查驗。Plan 必寫，code 之前。

**Escalation**：plan 階段若發現衝突真在 design 層級（不是 plan 寫得不對、是 design 漏了），暫停、回報、等指示。**不可在 plan 內就地 redesign**——design 變更必須回 design 層處理，這是 cascade 完整性的關鍵。

建議 AI 共筆環境：**CLI**。Plan 需要驗證實際檔案路徑、schema、既有 pattern，倚賴直接 codebase 存取。

詳見 `workflow-plan.md`。

## Closing

版本完成 implementation 不等於工作完成——還有 hygiene（backlog 收尾）、distillation（資訊蒸餾進 spec / guide）等收尾工作。這段過程稱為 **closing**（類比餐廳營業結束後的清潔收尾）。

Closing 機制（版本周期 vX.Y、VERSIONS.md 狀態、hygiene flow、distillation）詳見 `workflow-closing.md`。

## File naming

Tier 之下還有更細的拆解：

- **Stage** — 大塊分組，可選；橫跨多獨立 deliverable 時用。出現在檔名。
- **Phase** — stage 內的檔級子序列；若無 stage，phase 即主序列。出現在檔名。
- **Step** — plan 內的原子任務（document section，如 `Step 1`）。**不**出現在檔名。

命名原則：**本體在前、序列在後**——文件類型先寫，`stage-N` / `phase-M` 後接。

| 文件                         | Pattern                           |
| ---------------------------- | --------------------------------- |
| Proposal                     | `<topic>-proposal.md`             |
| Design                       | `<topic>-design.md`               |
| Blueprint（單檔）            | `<topic>-blueprint.md`            |
| Blueprint（per-stage）       | `<topic>-blueprint-stage-N.md`    |
| Plan（單檔）                 | `<topic>-plan.md`                 |
| Plan（phase 序列、無 stage） | `<topic>-plan-phase-M.md`         |
| Plan（stage + phase）        | `<topic>-plan-stage-N-phase-M.md` |

Qualifier 只在需要時加。同 topic 只有單一 plan 就停在 `<topic>-plan.md`；新增第二份時再一起改為 `-phase-M`。

例（`wallet-allocation`）：

```
wallet-allocation-design.md
wallet-allocation-blueprint.md
wallet-allocation-plan-stage-1-phase-1.md
wallet-allocation-plan-stage-1-phase-2.md
```

### Topic folder（optional）

當 topic 內 artifact 數量增多時，可改用 folder 組織。Folder 路徑形式：

```
idea/<version>/<topic>/        # handbook 側
plan/<version>/<topic>/        # codebase 側（plans）
```

**何時用 folder**（任一觸發即可）：

- Topic 有 3+ 個檔案
- Topic 有 extension docs（見下）
- Topic 有 multi-stage 或 multi-phase plans

Folder 與 flat 在同一 topic 內保持一致——選定後不混用。

**Folder 內命名**（`<topic>-` 前綴已由路徑表達、檔內不重述）：

| Flat                              | Folder 內                 |
| --------------------------------- | ------------------------- |
| `<topic>-proposal.md`             | `proposal.md`             |
| `<topic>-design.md`               | `design.md`               |
| `<topic>-blueprint.md`            | `blueprint.md`            |
| `<topic>-blueprint-stage-N.md`    | `blueprint-stage-N.md`    |
| `<topic>-plan-stage-N-phase-M.md` | `plan-stage-N-phase-M.md` |

**Extension docs**：tier 主檔之外的延伸補充，可視情況撰寫；命名以 tier 名為前綴——`design-xxx.md` / `blueprint-xxx.md` / `plan-xxx.md`（例：`design-api-draft.md`）。

**Codebase survey 不 commit**：survey 是討論過程的暫存資料，內容常很長且容易 outdated。Survey 結果作為 design / blueprint 的 input、不 commit 進 handbook——只 commit 蒸餾後的正式 artifact。

## Repo split

兩個 repo 配合：

| Repo     | 內容                                                                         |
| -------- | ---------------------------------------------------------------------------- |
| Handbook | `convention/`、`idea/vX.Y/`、`spec/`、`README.md`、`INDEX.md`、`VERSIONS.md` |
| Codebase | code、`guide/`、`plan/vX.Y/`、`CLAUDE.md`、`AGENTS.md`                       |

**為何這樣切**：frontend / backend 各自獨立 codebase。Design 與 blueprint 為概念性、不綁特定 code，放 handbook 讓兩 codebase 共用同源。Plan 與 `guide/` 強綁 codebase（plan 指具體檔案、guide 規範 code 寫法），放 codebase 旁。

預期 layout 為 sibling folder：

```
<workspace>/
  <backend-codebase>/
  <frontend-codebase>/
  <handbook>/
```

Plan 以相對路徑引用 design / blueprint（如 `../<handbook>/idea/v2.0/<topic>-design.md`）。CLI agent 在 codebase 工作時以 `--add-dir <handbook>` 帶入 handbook。

## Trajectory

PSDD 是當前共識，適合尚無 DSL spec / generation 工具的早期專案。隨專案成熟，目標演進至：

- 結構化 spec 格式（DSL / schema-based）
- 由 spec 生成 code 與 test
- 自動化 spec-code 一致性驗證

**OpenSpec** 的結構與 PSDD 高度對齊，列為觀察與評估對象。

現在 markdown 練的紀律——cascade、frozen frame、distill 進 spec——是未來自動化的鋪墊。

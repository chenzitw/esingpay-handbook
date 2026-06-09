---
status: draft
updated_at: 2026-06-09
updated_by: Claude
---

# Webhook 交易事件推播 — Blueprint 缺漏與待確認清單

本文件整理 blueprint suite 審查後發現的缺漏與待確認事項，作為 blueprint 修訂與 plan 啟動前的對照依據。

分三類處理：

- **A 類**：決策尚未做出，需先討論決定，才能更新 blueprint
- **B 類**：決策方向已足夠，可直接補進現有 blueprint 文件
- **C 類**：已列為 open point，但優先級標記不足，需升為阻斷項

---

## A 類 — 需先決策

### A1. Dispatcher 觸發模式

所在文件：[`blueprint/blueprint-infra-queue.md`](./blueprint/blueprint-infra-queue.md)

現況：以條件句處理——「如果 codebase 採 polling scheduler 而非 queue 觸發 dispatcher，`webhook.outbox.dispatch` 可不建立」。條件未決，Phase 3 plan 作者無法確認 queue topology。

已知補充：queue provider 第一版走 Azure Service Bus。這能鎖定 delivery execution 的 queue provider，但 dispatcher 仍需決定是 polling scheduler 觸發，或由 Azure Service Bus job 觸發。

需要的決策：Dispatcher 是否使用 polling scheduler 觸發，或由 queue job 驅動。需先做 codebase survey，取得既有 scheduler/queue 模式，再回 `blueprint-infra-queue.md` 鎖定。

阻斷：Phase 3 plan

---

### A2. Outbox event 與交易 transaction 的一致性策略

所在文件：[`blueprint/blueprint-phase-2.md`](./blueprint/blueprint-phase-2.md)（open point）

現況：「Event production failure 是否影響交易狀態變更 commit」列為 open point，推給 plan 決定。

需要的決策：Outbox event 寫入是否與交易狀態更新在同一 DB transaction 內（同 transaction vs. at-least-once outbox vs. two-phase）。這影響整個 Phase 2 的 production flow 形狀，屬 blueprint 層決策。

阻斷：Phase 2 plan

---

### A3. Signing 演算法與 header 命名

所在文件：[`blueprint/blueprint-payload-contract.md`](./blueprint/blueprint-payload-contract.md)、[`blueprint/blueprint-phase-4.md`](./blueprint/blueprint-phase-4.md)

現況：兩份文件均明確說「不在本文件定案」，但 Phase 4 scope 包含 signing 實作，無演算法 plan 無法展開。

已知補充：`signing_secret` 第一版先由服務環境變數提供預設值並寫入 subscription。這解掉 secret 初始來源，但尚未解掉簽章演算法、簽章 input 與 header 命名。

需要的決策：
- 演算法（例：HMAC-SHA256）
- 簽章 input 組成方式（body 字串、是否含 timestamp）
- Signature header 名稱

阻斷：Phase 4 plan

---

### A4. `deposit.blocked` 是否納入第一版 seed

所在文件：[`blueprint/blueprint-data-model.md`](./blueprint/blueprint-data-model.md)、[`blueprint/blueprint-phase-1.md`](./blueprint/blueprint-phase-1.md)、[`design.md`](./design.md)

現況：多份文件反覆標注為 open point，至今無決定。

需要的決策：是否在 Phase 1 migration seed 中正式寫入 `deposit.blocked`。

阻斷：Phase 1 plan（migration seed 計畫需此決定）

---

### A5. `api_version` 格式與外部交易 ID 格式

所在文件：[`blueprint/blueprint-payload-contract.md`](./blueprint/blueprint-payload-contract.md)

現況：
- `api_version` 是日期字串（`2026-06-01`）還是 semantic version（`v1`）未決
- `withdrawal_id` / `deposit_id` 是 bigint 字串、UUID 還是 external id 未決

需要的決策：兩項均需在 Phase 2 payload builder 實作前決定。

阻斷：Phase 2 plan（payload builder 需鎖定格式）

---

## B 類 — 可直接補進文件

### B1. `webhook_outbox_event.status = FAILED` 語意未說明

所在文件：[`blueprint/blueprint-data-model.md`](./blueprint/blueprint-data-model.md)

現況：status inventory 列出 `FAILED`，但無任何說明觸發條件，與 `NO_SUBSCRIBERS` 的差異不清楚。Phase 2、Phase 3 的 status 轉換邏輯都可能誤用。

修改方式：在 `blueprint-data-model.md` 的 `webhook_outbox_event` 狀態說明中補充 `FAILED` 的觸發條件（例如：production 寫入失敗？dispatcher 嘗試但全部 delivery 失敗？）。

---

### B2. `signing_secret` 是否在 POST response 中回傳

所在文件：[`blueprint/blueprint-api-surface.md`](./blueprint/blueprint-api-surface.md)

現況：POST /webhook-subscriptions 的概念 request 有說明，但未說明 response 是否包含 `signing_secret`。這是商戶取得 secret 的唯一時機，若遺漏則商戶無法驗簽。

修改方式：在 `blueprint-api-surface.md` 的 POST 段落補上 response 語意說明（是否回傳 secret、是否僅回傳一次）。

---

### B3. Dispatcher 去重策略方向

所在文件：[`blueprint/blueprint-phase-3.md`](./blueprint/blueprint-phase-3.md)

現況：validation target 提到「Dispatcher 可重跑而不造成重複 delivery，或 plan 明確定義去重策略」，策略方向推給 plan 決定。

修改方式：在 `blueprint-phase-3.md` 的 Critical Decisions 補建議方向，例如：「outbox event 轉為 DISPATCHED 即視為鎖定，後續重跑時跳過已 DISPATCHED 的 outbox event」。Plan 可沿用或提出替代方案並說明原因。

---

### B4. Master blueprint 缺少 Delivery cadence

所在文件：[`blueprint/blueprint.md`](./blueprint/blueprint.md)

現況：`workflow-blueprint.md` convention 要求說明 staging deploy 節奏，但 master blueprint 的 Sequencing 節只談依賴與並行，未說明各 phase 完成後的交付節奏。

修改方式：在 `blueprint.md` Sequencing 節補 Delivery cadence 段落，說明哪幾個 phase 合併一次 staging deploy，或每個 phase 獨立部署。

---

### B5. Master blueprint 缺少多文件讀取導引

所在文件：[`blueprint/blueprint.md`](./blueprint/blueprint.md)

現況：Blueprint suite 共 10 份文件，master 的 Critical Decisions 節直接把讀者導向子文件，但沒有說明「開始寫 plan 前最低必讀文件集合」，也沒有說明主題 blueprint 與 phase blueprint 的閱讀關係。

修改方式：在 `blueprint.md` Context 節或 Critical Decisions 節前補一段「Plan 作者閱讀建議」，標示各 phase 對應的最低必讀子 blueprint。

---

## C 類 — Open point 需升為阻斷項

### C1. `signing_secret` 儲存與讀取方式

所在文件：[`blueprint/blueprint-data-model.md`](./blueprint/blueprint-data-model.md)（Signing Secret 節）

現況：只說「不能只存不可逆 hash」，具體儲存方式（資料庫加密欄位、應用層 AES 加密、environment key）未決，推給 plan。

已知補充：第一版 secret 值來源先採服務環境變數預設值，建立 subscription 時寫入。仍需確認寫入 DB 後的保存方式是否加密，以及 worker 讀取 secret 的邊界。

需要的處置：升為 Phase 1 plan 前必決的阻斷條件。Phase 1 建立 subscription 時即需產生並儲存 secret；若到 Phase 4 才發現儲存方式不符需求，整個 signing flow 需翻修。建議在 `blueprint-data-model.md` 中標記此項為「Phase 1 plan 啟動前需查驗 codebase secret storage convention 並補回」。

---

### C2. Webhook 模組掛載位置

所在文件：[`blueprint/blueprint-service-architecture.md`](./blueprint/blueprint-service-architecture.md)（open points）

現況：「是否成為獨立 top-level domain/module，或先依現有 merchant/notification boundary 掛載」列為 open point，所有 phase 的 plan 都需要知道才能放置 class 與 module。

需要的處置：升為 Phase 1 plan 前必決的阻斷條件，並在 `blueprint-service-architecture.md` 中標記。建議 Phase 1 plan 啟動前先做 codebase module survey，決定後補回 `blueprint-service-architecture.md`。

---

### C3. Event type read model 形式

所在文件：[`blueprint/blueprint-api-surface.md`](./blueprint/blueprint-api-surface.md)、[`blueprint/blueprint-phase-1.md`](./blueprint/blueprint-phase-1.md)

現況：兩份文件均列為 open point（獨立 endpoint vs. 併入 subscription detail response）。Phase 1 validation target 已包含「list event type options」，若到 plan 時才決定，Phase 1 plan 需同時準備兩種架構。

需要的處置：升為 Phase 1 plan 啟動前必決的阻斷條件。建議在 `blueprint-phase-1.md` 的 Open Points 中標記此項為 Phase 1 plan 不可啟動的前提。

---

## 快速對照表

| # | 類型 | 事項 | 影響 phase | 對應文件 | 狀態 |
|---|------|------|-----------|---------|------|
| A1 | 決策 | Dispatcher 觸發模式 | 3 | blueprint-infra-queue.md | 未決 |
| A2 | 決策 | Outbox-transaction 一致性策略 | 2 | blueprint-phase-2.md | 未決 |
| A3 | 決策 | Signing 演算法與 header | 4 | blueprint-payload-contract.md, blueprint-phase-4.md | 未決 |
| A4 | 決策 | `deposit.blocked` 是否納入 seed | 1 | blueprint-data-model.md, blueprint-phase-1.md | 未決 |
| A5 | 決策 | `api_version` 與外部 ID 格式 | 2 | blueprint-payload-contract.md | 未決 |
| B1 | 補文件 | `outbox.status = FAILED` 語意 | 2, 3 | blueprint-data-model.md | 待補 |
| B2 | 補文件 | `signing_secret` 回傳語意 | 1 | blueprint-api-surface.md | 待補 |
| B3 | 補文件 | Dispatcher 去重策略方向 | 3 | blueprint-phase-3.md | 待補 |
| B4 | 補文件 | Delivery cadence | 全 | blueprint.md | 待補 |
| B5 | 補文件 | 多文件讀取導引 | 全 | blueprint.md | 待補 |
| C1 | 升阻斷 | Signing secret 儲存方式 | 1, 4 | blueprint-data-model.md | 待升級 |
| C2 | 升阻斷 | Webhook 模組掛載位置 | 全 | blueprint-service-architecture.md | 待升級 |
| C3 | 升阻斷 | Event type read model 形式 | 1 | blueprint-api-surface.md, blueprint-phase-1.md | 待升級 |

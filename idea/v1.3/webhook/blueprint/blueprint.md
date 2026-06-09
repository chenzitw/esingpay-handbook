---
status: draft
updated_at: 2026-06-09
updated_by: Codex
---

# Webhook 交易事件推播 — Blueprint

## Context

本 blueprint 承接 [`../design.md`](../design.md)，把 webhook 交易事件推播拆成可落地的 phase 序列，並鎖定 plan 前需要一致的 API surface、data model、RPC surface、service architecture 與 infra queue 邊界。

Webhook 是從零建立的新服務能力，不只是既有 feature 的單點延伸。因此本 topic 的 blueprint 拆成 master file 與多份專題 blueprint：

- [`blueprint-api-surface.md`](./blueprint-api-surface.md)：商戶後台管理 API 與事件目錄 read surface。
- [`blueprint-data-model.md`](./blueprint-data-model.md)：subscription、event type、outbox event、delivery 的 conceptual schema shape。
- [`blueprint-rpc-surface.md`](./blueprint-rpc-surface.md)：internal RPC 能力、producer/dispatcher/worker 需要的服務邊界。
- [`blueprint-service-architecture.md`](./blueprint-service-architecture.md)：domain 命名、module 切分、ownership 與服務職責。
- [`blueprint-infra-queue.md`](./blueprint-infra-queue.md)：queue topic、producer/consumer、dispatcher/worker/recovery topology。
- [`blueprint-payload-contract.md`](./blueprint-payload-contract.md)：POST 到商戶 endpoint 的 payload envelope 與 event-specific data shape。

第一版目標是同時支援：

- 商戶後台管理 webhook subscription。
- 系統將 withdrawal / deposit 交易事件轉換為 webhook outbox event。
- 系統依 subscription 建立 webhook delivery 並追蹤派送結果。
- 系統以非同步 worker 執行 webhook delivery，並能補償 pending 或 timeout 任務。

## Scope

In scope：

- Webhook subscription 管理 API。
- Webhook event type seed 與查詢來源。
- Webhook subscription 與 event type 的多對多關係。
- Webhook outbox event 與 webhook delivery 的拆表模型。
- Dispatcher / worker / recovery scheduler 的方向性流程。
- Internal RPC surface 與服務 ownership。
- Queue topic 與 consumer topology。
- Webhook POST payload 第一版 envelope。
- Phase 拆分與初步開發估時。

Out of scope：

- Migration SQL、ORM entity class 與 repository 檔案位置。
- Request / response DTO 的最終型別定義。
- Webhook payload external contract 的完整文件與所有事件欄位細節。
- Signature 演算法與驗簽 header 的最終規格。
- 重試次數、backoff、人工重送與 delivery history UI。
- Endpoint ownership verification flow。

## Landed Facts Assumed

- Merchant ID 沿用既有 UUID 字串設計，故 webhook schema 內的 `merchant_id` 保持字串語意。
- 其他資料表自身主鍵與 FK 使用 bigint 系統主鍵策略。
- 交易來源追蹤命名沿用既有服務慣例：`correlation_type` / `correlation_identifier`。
- 資料表命名採單數。
- Webhook event type 不是商戶可管理資料，由 migration seed 預先寫入。
- Webhook 派送不得阻塞 withdrawal / deposit 的主交易流程。

## Critical Decisions

跨 phase 共同決策集中於專題 blueprint：

- API route 與 event catalog read model 見 [`blueprint-api-surface.md`](./blueprint-api-surface.md)。
- 資料表命名、ID/time strategy、status 與欄位 inventory 見 [`blueprint-data-model.md`](./blueprint-data-model.md)。
- Internal capability 邊界與 RPC surface 見 [`blueprint-rpc-surface.md`](./blueprint-rpc-surface.md)。
- Domain/service/module 命名與 ownership 見 [`blueprint-service-architecture.md`](./blueprint-service-architecture.md)。
- Queue topic、worker topology 與 recovery 方向見 [`blueprint-infra-queue.md`](./blueprint-infra-queue.md)。
- POST payload envelope 與 event-specific data shape 見 [`blueprint-payload-contract.md`](./blueprint-payload-contract.md)。

## Phase Breakdown

### Phase 1：Subscription Management And Event Catalog

建立 webhook subscription、event type catalog、subscription-event relation 的 persistence 與商戶後台管理 API。

詳細 blueprint：[`blueprint-phase-1.md`](./blueprint-phase-1.md)

依賴：

- `deposit.blocked` 是否納入正式 seed 需要在此 phase 前確認。

完成後應能支援：

- 商戶建立、查詢、修改、刪除 webhook subscription。
- 後端以 `webhook_event_type` catalog 作為 UI checkbox 與訂閱校驗來源。

### Phase 2：Outbox Event Production

讓 withdrawal / deposit 相關交易狀態變更寫入 webhook outbox event，不直接呼叫商戶 endpoint。

詳細 blueprint：[`blueprint-phase-2.md`](./blueprint-phase-2.md)

依賴：

- Event type catalog 已存在，outbox event 能對應正式 event ID。
- Payload envelope 依 [`blueprint-payload-contract.md`](./blueprint-payload-contract.md) 落地；具體欄位來源由 plan 查驗 codebase 後定案。

完成後應能支援：

- Withdrawal / deposit 狀態變更時建立 outbox event。
- 交易主流程只寫入事件，不同步派送 webhook。

### Phase 3：Dispatcher And Delivery Creation

Dispatcher 輪詢 pending outbox event，依 merchant 與 event 找出 subscription 並建立 webhook delivery。

詳細 blueprint：[`blueprint-phase-3.md`](./blueprint-phase-3.md)

依賴：

- Subscription management 已落地。
- Outbox event production 已可產生 pending event。
- Queue / scheduler pattern 已完成 codebase survey。

完成後應能支援：

- 無訂閱者時 outbox event 轉為 `NO_SUBSCRIBERS`。
- 有訂閱者時為每個 subscription 建立 delivery。
- Delivery job 可被發送到 worker queue。

### Phase 4：Delivery Worker, Recovery And Signing

Worker 執行 endpoint POST、更新 delivery 結果；recovery scheduler 補償 pending 或 timeout delivery。

詳細 blueprint：[`blueprint-phase-4.md`](./blueprint-phase-4.md)

依賴：

- Dispatcher 已能建立 delivery。
- Signing secret 保存與產生方式已由 plan/codebase survey 確認。
- Delivery payload snapshot 已符合 [`blueprint-payload-contract.md`](./blueprint-payload-contract.md)。
- Timeout 與 retry 第一版策略已決定。

完成後應能支援：

- Worker 鎖定 delivery 並執行 HTTP POST。
- Delivery 成功 / 失敗狀態可追蹤。
- Recovery scheduler 能補償卡住的 delivery。

## End-To-End Flow

```text
withdrawal / deposit status transition
  -> webhook event producer
  -> webhook_outbox_event
  -> dispatcher polls pending outbox events
  -> query active webhook_subscription by merchant_id + event_id
  -> create webhook_delivery per matching subscription
  -> publish delivery job
  -> delivery worker locks delivery
  -> sign payload
  -> POST endpoint_url
  -> update delivery status
  -> recovery scheduler requeues stuck delivery when needed
```

## Sequencing

Dependencies：

- Phase 1 必須先確定 event type seed，否則 UI checkbox 與 subscription validation 無法落地。
- Phase 2 依賴 event type catalog，因 outbox event 需要對應正式 event ID。
- Phase 3 依賴 subscription relation 與 outbox event。
- Phase 4 依賴 delivery 建立流程。

Parallelism：

- Phase 1 的 API route 與 data model plan 可一起規劃。
- Phase 2 可在 Phase 1 data model 穩定後與 UI 細節並行。
- Phase 3 的 dispatcher plan 可在 queue pattern survey 完成後與 Phase 2 後段並行準備，但 implementation 應等 outbox event production 可用。
- Phase 4 應在 delivery schema、queue topic 與狀態轉換規則穩定後展開。

## Engineering Estimate

估時以前述 blueprint、既有 codebase 複用程度，以及 AI agent 協作開發方式為前提。

| Phase | 估時 | 依據 |
| --- | ---: | --- |
| Phase 1 | 4 天 | 新資料模型、event seed、subscription CRUD、subscription-event relation 與商戶 scope 驗證。 |
| Phase 2 | 3 天 | 需要接 withdrawal / deposit 狀態變更點與 payload envelope；若既有 event/hook pattern 不足，估時可能增加。 |
| Phase 3 | 4 天 | Dispatcher polling、subscription matching、delivery creation、queue publishing 與 outbox 狀態轉換需一起驗證。 |
| Phase 4 | 5 天 | Worker lock、HTTP POST、signature secret、timeout recovery 與失敗狀態處理未知較高。 |

| 範圍 | 估時 | 備註 |
| --- | ---: | --- |
| Phase 1-2 | 7 天 | 先建立可管理 subscription 且能產生 outbox event 的基礎。 |
| Phase 3-4 | 9 天 | 完成完整非同步派送鏈路。 |
| 整體 calendar | 約 3 週 | 若 queue / scheduler / secret handling pattern 已存在，可壓縮；若需建立新 infra convention，需額外時間。 |

## Pattern Gaps

- Outbox dispatcher 與 delivery worker 若 codebase 尚無既有 pattern，plan 需先 survey queue / scheduler / worker 既有實踐。
- Signing secret 的生成、保存與輪替若 codebase 尚無 convention，plan 需明確列出採用方式，後續穩定後蒸餾進 guide。
- Webhook payload 第一版 envelope 已在 [`blueprint-payload-contract.md`](./blueprint-payload-contract.md) 定案；Phase 2 plan 仍需查驗 withdrawal / deposit 欄位來源。
- Queue topic 命名若尚無 guide convention，Phase 3 plan 需明確記錄採用理由。

## Open Points

- `deposit.blocked` 是否正式納入第一版 seed。
- 是否提供獨立 event type read endpoint，或併入 subscription detail response。
- Outbox event 轉為 `DISPATCHED` 的精確時機。
- Delivery 失敗後是否立刻 `FAILED`，或需要 `RETRYING` / retry count 等額外狀態。
- Delivery timeout 門檻與 recovery scheduler cadence。
- Webhook signature 演算法、header 命名與驗簽文件。
- Webhook payload 欄位來源、failure reason 與版本策略細節。

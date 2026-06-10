---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Blueprint

## Context

本 blueprint 承接 [`../design/design.md`](../design/design.md)，只負責把 webhook 交易事件推播拆成可落地的 stage 序列、phase、依賴與估時。跨 stage 的 API、RPC、persistence、queue、payload、service boundary 等 contract 由 design 文件維護。

主要設計來源：

- [`../design/design-rest.md`](../design/design-rest.md)：merchant-facing REST contract。
- [`../design/design-rpc.md`](../design/design-rpc.md)：webhook service RPC surface。
- [`../design/design-persistence-model.md`](../design/design-persistence-model.md)：conceptual table / index model。
- [`../design/design-queue-topology.md`](../design/design-queue-topology.md)：queue topics、payloads、dispatcher / worker / recovery flow。
- [`../design/design-payload-contract.md`](../design/design-payload-contract.md)：outbound webhook payload contract。
- [`../design/design-service-boundary.md`](../design/design-service-boundary.md)：ownership 與 service boundary。

第一版目標是同時支援：

- 商戶後台管理 webhook subscription。
- 系統將 withdrawal / deposit 交易事件轉換為 webhook outbox event。
- 系統依 subscription 建立 webhook delivery 並追蹤派送結果。
- 系統以非同步 worker 執行 webhook delivery，並能補償 pending 或 timeout 任務。

## Scope

In scope：

- Webhook subscription 管理 API。
- Code-defined webhook event type catalog 與查詢來源。
- Webhook subscription 與 event type key 的多對多關係。
- Webhook outbox event 與 webhook delivery 的拆表模型。
- Dispatcher / worker / recovery scheduler 的方向性流程。
- Internal RPC surface 與服務 ownership。
- Queue topic 與 consumer topology。
- Webhook POST payload 第一版 envelope。
- Stage / phase 拆分與初步開發估時。

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
- 交易來源追蹤命名使用：`resource_type` / `resource_identifier`。
- 資料表命名採單數。
- Webhook event type 不是商戶可管理資料，由 TypeScript code-defined catalog 提供，不建 DB table。
- Webhook 派送不得阻塞 withdrawal / deposit 的主交易流程。

## Critical Decisions

跨 stage 共同決策集中於 design 文件：

- API route 與 event catalog read model 見 [`../design/design-rest.md`](../design/design-rest.md)。
- 資料表命名、ID/time strategy、status 與欄位 inventory 見 [`../design/design-persistence-model.md`](../design/design-persistence-model.md)。
- Internal capability 邊界與 RPC surface 見 [`../design/design-rpc.md`](../design/design-rpc.md)。
- Domain/service/module 命名與 ownership 見 [`../design/design-service-boundary.md`](../design/design-service-boundary.md)。
- Queue topic、worker topology 與 recovery 方向見 [`../design/design-queue-topology.md`](../design/design-queue-topology.md)。
- POST payload envelope 與 event-specific data shape 見 [`../design/design-payload-contract.md`](../design/design-payload-contract.md)。

## Stage Breakdown

### Stage 1：Subscription Management And Event Catalog

建立 webhook subscription、code-defined event type catalog、subscription-event relation 的 persistence 與商戶後台管理 API。

詳細 blueprint：[`blueprint-stage-1.md`](./blueprint-stage-1.md)

Stage 1 已拆成多個 phase：

- Phase 1：New service boundary and implementation convention。
- Phase 2：Persistence schema and event catalog。
- Phase 3：Event type read and validation capability。
- Phase 4：Subscription read APIs and merchant scope baseline。
- Phase 5：Subscription write APIs and transaction boundaries。
- Phase 6：Merchant console frontend integration。
- Phase 7：Stage 1 verification and tests。

完成後應能支援：

- 商戶建立、查詢、修改、刪除 webhook subscription。
- 後端以 code-defined event type catalog 作為 UI checkbox 與訂閱校驗來源。

### Stage 2：Outbox Event Production

讓 withdrawal / deposit 相關交易狀態變更寫入 webhook outbox event，不直接呼叫商戶 endpoint。

詳細 blueprint：[`blueprint-stage-2.md`](./blueprint-stage-2.md)

依賴：

- Code-defined event type catalog 已存在，outbox event 能保存正式 event key。
- Payload envelope 依 [`../design/design-payload-contract.md`](../design/design-payload-contract.md) 落地；具體欄位來源由 plan 查驗 codebase 後定案。

完成後應能支援：

- Withdrawal / deposit 狀態變更時建立 outbox event。
- 交易主流程只寫入事件，不同步派送 webhook。

### Stage 3：Dispatcher And Delivery Creation

Dispatcher 輪詢 pending outbox event，依 merchant 與 event 找出 subscription 並建立 webhook delivery。

詳細 blueprint：[`blueprint-stage-3.md`](./blueprint-stage-3.md)

依賴：

- Subscription management 已落地。
- Outbox event production 已可產生 pending event。
- Queue / scheduler pattern 已完成 codebase survey。

完成後應能支援：

- 無訂閱者時不建立 delivery，outbox event 仍轉為 `DISPATCHED` 表示 dispatcher 已處理。
- 有訂閱者時為每個 subscription 建立 delivery。
- Delivery job 可被發送到 worker queue。

### Stage 4：Delivery Worker, Recovery And Signing

Worker 執行 endpoint POST、更新 delivery 結果；recovery scheduler 補償 pending 或 timeout delivery。

詳細 blueprint：[`blueprint-stage-4.md`](./blueprint-stage-4.md)

依賴：

- Dispatcher 已能建立 delivery。
- Signing secret 來源、讀取方式與 header 命名已由 Stage 4 plan/codebase survey 確認。
- Delivery payload snapshot 已符合 [`../design/design-payload-contract.md`](../design/design-payload-contract.md)。
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
  -> query eligible webhook_subscription by merchant_id + event_type
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

- Stage 1 必須先確定 code-defined event type catalog，否則 UI checkbox 與 subscription validation 無法落地。
- Stage 2 依賴 event type catalog，因 outbox event 需要保存正式 event key。
- Stage 3 依賴 subscription relation 與 outbox event。
- Stage 4 依賴 delivery 建立流程。

Parallelism：

- Stage 1 內部 phase 應依 [`blueprint-stage-1.md`](./blueprint-stage-1.md) 的 sequencing 推進。
- Stage 2 可在 Stage 1 data model 穩定後與 UI 細節並行。
- Stage 3 的 dispatcher plan 可在 queue pattern survey 完成後與 Stage 2 後段並行準備，但 implementation 應等 outbox event production 可用。
- Stage 4 應在 delivery schema、queue topic 與狀態轉換規則穩定後展開。

## Engineering Estimate

估時以前述 blueprint、既有 codebase 複用程度，以及 AI agent 協作開發方式為前提。

| Stage | 估時 | 依據 |
| --- | ---: | --- |
| Stage 1 | 6 天 | 新資料模型、code-defined event catalog、event type read endpoint、subscription CRUD、merchant console 串接、subscription-event relation 與商戶 scope 驗證。 |
| Stage 2 | 3 天 | 需要接 withdrawal / deposit 狀態變更點與 payload envelope；若既有 event/hook pattern 不足，估時可能增加。 |
| Stage 3 | 4 天 | Dispatcher polling、subscription matching、delivery creation、queue publishing 與 outbox 狀態轉換需一起驗證。 |
| Stage 4 | 5 天 | Worker lock、HTTP POST、signature secret、timeout recovery 與失敗狀態處理未知較高。 |

| 範圍 | 估時 | 備註 |
| --- | ---: | --- |
| Stage 1-2 | 9 天 | 先建立可管理 subscription 且能產生 outbox event 的基礎。 |
| Stage 3-4 | 9 天 | 完成完整非同步派送鏈路。 |
| 整體 calendar | 約 3.5-4 週 | 若 queue / scheduler / secret handling pattern 已存在，可壓縮；若需建立新 infra convention，需額外時間。 |

## Pattern Gaps

- Outbox dispatcher 與 delivery worker 若 codebase 尚無既有 pattern，plan 需先 survey queue / scheduler / worker 既有實踐。
- Signing secret 的保存、讀取與輪替若 codebase 尚無 convention，plan 需明確列出採用方式，後續穩定後蒸餾進 guide。
- Webhook payload 第一版 envelope 已在 [`../design/design-payload-contract.md`](../design/design-payload-contract.md) 定案；Stage 2 plan 仍需查驗 withdrawal / deposit 欄位來源。
- Queue provider 第一版採 Azure Service Bus；queue/topic 命名若尚無 guide convention，Stage 3 plan 需明確記錄採用理由。

## Open Points

- Outbox event 轉為 `DISPATCHED` 的精確時機。
- Delivery 失敗後是否立刻 `FAILED`，或需要 `RETRYING` / retry count 等額外狀態。
- Delivery timeout 門檻與 recovery scheduler cadence。
- Webhook signature 演算法、header 命名與驗簽文件。
- Webhook payload 欄位來源、failure reason 與版本策略細節。

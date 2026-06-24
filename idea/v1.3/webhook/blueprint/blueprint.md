---
status: draft
updated_at: 2026-06-23
updated_by: Codex
---

# Webhook 交易事件推播 — Blueprint

## Context

本 blueprint 承接 [`../design/design.md`](../design/design.md)，只負責把 Webhook 交易事件推播拆成可落地的 stage 序列、plan inventory、依賴與估時。跨 stage 的 management API、persistence、inbound event、private delivery job、payload 與 service boundary contract 由 design 文件維護。

主要設計來源：

- [`../design/design-rest.md`](../design/design-rest.md)：merchant / platform management REST contract semantics。
- [`../design/design-rpc.md`](../design/design-rpc.md)：webhook service management transport boundary。
- [`../design/design-persistence-model.md`](../design/design-persistence-model.md)：direct-to-delivery persistence 與 deferred inbox target。
- [`../design/design-queue-topology.md`](../design/design-queue-topology.md)：inbound domain event、private delivery job、post-commit publisher / worker / recovery flow。
- [`../design/design-payload-contract.md`](../design/design-payload-contract.md)：outbound webhook payload contract。
- [`../design/design-service-boundary.md`](../design/design-service-boundary.md)：ownership 與 service boundary。

第一版 Webhook scope 目標：

- 商戶後台管理 webhook subscription。
- Webhook 消費既有 Fund transaction event，依 subscription 直接建立 deliveries。
- Webhook 追蹤每個 endpoint 的 delivery 結果。
- Webhook 以 private queue job 非同步執行 delivery，並補償 publish failure 或 stuck execution。

Fund producer、交易狀態 hook 與 producer outbox 不屬於本 blueprint。

## Scope

In scope：

- Webhook subscription management API。
- Code-defined webhook event type catalog 與 subscription-event relation。
- Webhook inbound domain event consumption boundary。
- Direct-to-delivery persistence、source/resource traceability 與 composite idempotency。
- Subscription matching and delivery snapshot creation。
- Post-commit delivery job publishing and stale pending publish recovery。
- Delivery worker、signing 與 stuck execution recovery。
- Stage / plan inventory、dependency、pattern gap 與初步估時。

Out of scope：

- Fund event publisher、transaction status hooks、producer outbox 與 Fund code changes。
- Webhook inbox persistence in the first version。
- Migration SQL、ORM class、method signature 與實際檔案 placement。
- Signature 演算法與驗簽 header 的最終規格。
- 完整 retry/backoff product policy、人工重送與 delivery history UI。
- Endpoint ownership verification flow。
- 新 broker infrastructure 建置；若既有 transport 無法承載 inbound event，需另行決策與估算。

## Landed Facts Assumed

- Stage 1 已完成 subscription management、code-defined event catalog 與 subscription-event binding。
- Webhook capability 已落在 `esingpay-cradle` 的獨立 `webhook` service boundary 與 DB schema。
- Merchant ID 使用既有 UUID string semantics；Webhook persistence record id 使用既有 bigint strategy。
- Webhook event catalog 第一版包含 withdrawal 四種與 deposit 四種事件。
- `source`、`eventKey`、`resourceType`、`resourceIdentifier` 是 inbound traceability 的共用詞彙；第一版 `source = fund`。
- Codebase guide 將跨 capability asynchronous fact 視為 Event，將 delivery execution 視為 Webhook service-private Queue job。
- Fund service 已有 BullMQ dispatcher / worker 與 scheduler / bootstrap backfill precedent，可作為 private delivery job 的方向性 baseline。
- Codebase 目前沒有 Fund domain event definition，也未找到 Azure Service Bus adapter precedent。

## Critical Decisions

- 採方案 A，保留 Stage 1–4 編號：Stage 2 建立 delivery、Stage 3 發布 delivery job、Stage 4 執行 delivery。
- Stage 2 從 Webhook inbound boundary 開始；Fund producer implementation 是 external dependency，不屬於任何 Webhook stage。
- 第一版不建立 webhook outbox / inbox；inbound consumer 直接 matching subscriptions 並建立 delivery snapshots。
- Delivery 第一版以 `source + eventKey + resourceType + resourceIdentifier + subscription` 防止重複建立。
- 沒有 matching subscription 時不建立 receipt；此限制明確由 direct-to-delivery MVP 承擔。
- Inbound domain event 與 Webhook private delivery job 是不同 contract surface，不共用 queue job vocabulary。
- Stage 3 owns post-commit delivery job publication and stale pending publish recovery；Stage 4 owns execution and stuck execution recovery。
- Deferred inbox 是已確認的 reliability target，但不進入第一版 stage estimate。

## Stage Breakdown

### Stage 1：Subscription Management And Event Catalog

建立 subscription、event catalog、subscription-event relation 與 merchant / platform management API。

詳細 blueprint：[`blueprint-stage-1.md`](./blueprint-stage-1.md)

Status：Stage 1 implementation complete；migration must be applied before end-to-end verification。

完成能力：

- Merchant / platform subscription CRUD。
- Code-defined event type lookup and validation。
- Active subscription ownership、soft delete 與 binding semantics。

### Stage 2：Inbound Event Consumption And Delivery Creation

Webhook 消費既有 Fund transaction event，matching active subscriptions，並直接建立 pending delivery snapshots。

詳細 blueprint：[`blueprint-stage-2.md`](./blueprint-stage-2.md)

依賴：

- Stage 1 subscription relation and event catalog 已落地。
- 既有 transport 能將包含必要 source / event / resource / payload semantics 的 event 送達 Webhook boundary。

完成能力：

- Valid event 為每個 matching subscription 建立一筆 delivery。
- Duplicate redelivery 不重複建立同一 subscription delivery。
- No-subscriber event 按 MVP semantics 完成而不留 persistence record。

### Stage 3：Delivery Job Publishing

Delivery DB commit 後立即發布 Webhook private execution job；若暫時 publish failure，delivery 保持 `PENDING`，由 recovery cron 補發。

詳細 blueprint：[`blueprint-stage-3.md`](./blueprint-stage-3.md)

依賴：

- Stage 2 已建立 pending delivery persistence and snapshots。
- Private queue dispatcher、post-commit publish integration 與 recovery trigger pattern 已由 plan 依 codebase baseline 定案。

完成能力：

- Newly committed delivery 可立即發布為 `webhook.delivery.execute` job。
- Publish failure 後 delivery 仍可由 stale pending recovery 補發。

### Stage 4：Delivery Worker, Recovery And Signing

Worker 執行 endpoint POST、更新 delivery 結果；recovery 補償 stuck execution。

詳細 blueprint：[`blueprint-stage-4.md`](./blueprint-stage-4.md)

依賴：

- Stage 3 已能發布 delivery execution job。
- Signing secret 來源、header 與 timeout / retry 第一版策略已決定。
- Delivery payload snapshot 符合 [`../design/design-payload-contract.md`](../design/design-payload-contract.md)。

完成能力：

- Worker 鎖定 delivery、簽章並 POST endpoint。
- Delivery success / failure 可追蹤。
- Stuck execution 可被 recovery flow 重新投入 worker queue。

## End-To-End Flow

```text
existing Fund transaction event transport
  -> Webhook inbound event consumer
  -> validate source + event type + resource + payload
  -> match active subscriptions by merchantId + eventKey
  -> create PENDING webhook_delivery per subscription
  -> commit delivery rows
  -> delivery publisher immediately emits private execution job
  -> delivery worker locks delivery
  -> sign payload
  -> POST endpoint_url
  -> update delivery status
  -> recovery requeues stuck execution when needed
```

## Deferred Inbox Target

Producer 提供穩定 `eventId` 後，後續 blueprint 應導入 `webhook_inbox_event`：

- Consumer 先保存 inbox event，再 complete inbound message。
- Inbox 以 `source + event_id` 去重並保存 no-subscriber receipt。
- Dispatcher 從 pending inbox event 建立 deliveries。
- Delivery 以 `inbox_event_id + subscription_id` 去重。

此 target 不改變本次 Stage 2–4 scope，也不納入本次 estimate。

## Sequencing

Dependencies：

- Stage 1 已完成，是 Stage 2 subscription matching 的前置條件。
- Stage 2 必須先建立 delivery persistence 與 snapshot semantics，Stage 3 才能接上 post-commit publication。
- Stage 3 必須先提供 execution job，Stage 4 worker 才能完整驗證。
- Fund producer implementation 可由外部工作流獨立進行；Webhook blueprint 不安排其 sequencing。

Parallelism：

- Stage 2 plan 可先完成 delivery persistence / matching inventory，再接入已確認的 inbound adapter。
- Stage 3 plan 可在 Stage 2 後段準備 private queue、post-commit publisher 與 recovery scheduler pattern，但 implementation verification 依賴 Stage 2 delivery rows。
- Stage 4 的 signing / HTTP client research 可與 Stage 3 並行，完整 worker flow 需等待 execution job 與 delivery state 穩定。

Delivery cadence：

- Stage 2 完成後先驗證 inbound event 到 pending delivery，不要求 HTTP delivery。
- Stage 3 完成後驗證 newly committed delivery 到 private execution job，以及 stale pending recovery。
- Stage 4 完成後再做完整 endpoint delivery staging verification。

## Engineering Estimate

估時只涵蓋 Webhook scope，不包含 Fund producer 或新 broker infrastructure。

| Stage | 估時 | 依據 |
| --- | ---: | --- |
| Stage 1 | 6 天（已完成） | Subscription persistence、catalog、management API、scope 與 backend verification。 |
| Stage 2 | 4 天 | Inbound adapter、delivery persistence、matching、snapshot、composite idempotency 與 targeted verification。 |
| Stage 3 | 2 天 | Post-commit delivery job publisher、private queue job 與 stale pending publish recovery。 |
| Stage 4 | 5 天 | Worker lock、HTTP POST、signing、failure mapping 與 stuck execution recovery。 |

| 範圍 | 估時 | 備註 |
| --- | ---: | --- |
| Remaining Stage 2–4 | 11 天 | Webhook implementation only。 |
| Whole Stage 1–4 | 17 天 | Stage 1 已完成；不含 Fund / broker infrastructure。 |
| Remaining calendar | 約 2.5–3 週 | 取決於 inbound transport availability 與 signing decisions。 |

## Pattern Gaps

- Fund domain event definition 尚不存在。Stage 2 plan 需確認 Webhook consumer-side shared contract landing；不得擴張為 Fund publisher implementation。
- Azure Service Bus adapter precedent 未找到。若 inbound transport 必須新建 provider integration，應先形成 infrastructure decision 並額外估算。
- Post-commit DB-to-private-queue publication 與 stale pending recovery 是既有 BullMQ dispatcher 與 scheduler precedent 的新組合；Stage 3 plan 必須鎖定 queued marker、deterministic job id 或 pure pending recovery strategy。
- Signing secret handling 與 webhook signature contract 尚未定案，Stage 4 plan 前需解決。

## Open Points

- Webhook consumer 實際接入的既有 topic / subscription、transport envelope 與 broker completion semantics。
- Stage 3 publication state：queued marker、deterministic job id、stale pending threshold 或其他 recovery model。
- Delivery timeout、failure / retry status 與 recovery cadence。
- Webhook signature algorithm、header naming 與 verification documentation。
- Payload optional failure / blocked / cancelled reason 的實際來源。
- Deferred inbox migration timing and stable producer `eventId` contract。

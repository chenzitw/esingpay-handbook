---
status: draft
updated_at: 2026-06-29
updated_by: Codex
---

# Webhook 交易事件推播 — Queue Topology Design

## Purpose

本文定義 webhook 第一版會用到的 asynchronous surface topology 與 Webhook service-private delivery execution job。Fund -> Webhook broadcast event 的 topic、session、transport envelope 與 application payload 已拆至 [`design-inbound-event-contract.md`](./design-inbound-event-contract.md)。

這不是最終 infra provisioning 設定，也不定義 dead letter、部署資源名稱或實際 TypeScript DTO 檔案位置。那些細節留給 blueprint / plan 依服務實作與 infra convention 定案。

## Context

交易狀態變更不是由 Fund service 同步呼叫 webhook service。Fund service 在處理交易狀態變更時向 Azure Service Bus Topic 發出 domain event；Webhook service 以自己的 Subscription 消費該 event，收到事件後直接 matching subscriptions 並建立 `webhook_delivery`。

Webhook 第一版不保存 inbound event 的 outbox 或 inbox record。Inbound consumer 建立 delivery 並 commit 後，Stage 3 會立即發布 BullMQ delivery execution job；若 publish 失敗，delivery 保持 `PENDING`，由 recovery cron 補發 job。

因此第一版有兩個非同步平面：

- Domain event broadcast：Azure Service Bus Topic / Subscription。Fund service 發布交易事件，Webhook service 消費並直接建立 matching deliveries；contract 見 [`design-inbound-event-contract.md`](./design-inbound-event-contract.md)。
- Webhook delivery execution queue：BullMQ service-private job。Webhook inbound consumer 的 post-commit publisher 或 recovery scheduler 發布 delivery job，Webhook worker 消費並執行 endpoint POST。

Provider 決策：

- Cross-service event broadcast 採 Azure Service Bus Topic / Subscription。
- Webhook delivery execution queue 保留 BullMQ，屬 Webhook service-private job，不進入 Azure SB event topic 命名規則。
- Azure SB inbound event subscription 需啟用 session，Webhook consumer 必須使用 session-aware receiver，以 `sessionId` 確保同一 ordering key 內 FIFO。

正式 BullMQ queue name、consumer 設定、dead letter、資源命名與部署設定留給 plan 依 infra convention 定案。Fund inbound Azure SB topic / subscription naming 見 [`design-inbound-event-contract.md`](./design-inbound-event-contract.md)。

## Queue Inventory

| Stage | Transport | Entity / name | Producer | Consumer | Purpose |
| --- | --- | --- | --- | --- | --- |
| Domain event broadcast | Azure SB Topic / session-enabled Subscription | `fund.deposit.status-changed / srv-webhook`; `fund.withdrawal-intent.status-changed / srv-webhook` | Fund service | Webhook inbound event consumer | 通知 webhook 某筆交易狀態已變更，可 matching subscriptions 並建立 deliveries。 |
| Delivery execution | BullMQ private queue | job name `webhook.delivery.execute` | Webhook inbound consumer post-commit publisher / recovery scheduler | Webhook delivery worker | 要求 worker 執行某一筆 delivery。 |

`webhook.delivery.execute` 不是 Azure SB topic。它若保留，僅作為 BullMQ private job name。

## Webhook Delivery Execution Payload

Delivery execution payload 是 webhook 內部 BullMQ job payload，只需要讓 worker 找到要執行的 delivery。

Conceptual envelope：

```json
{
  "deliveryId": "12345"
}
```

Rules：

- Worker 收到 `deliveryId` 後，必須從 persistence 讀取 delivery snapshot。
- Worker 使用 delivery 上保存的 endpoint snapshot 與 `payload`，不重新查 subscription current endpoint 或交易現況。
- Worker 必須透過 delivery 狀態轉換鎖定任務，避免同一 delivery 被多個 worker 重複處理。
- Stage 3 recovery scheduler 可為 stale `PENDING` delivery 重新發布同一 `deliveryId`，由 worker lock 機制決定是否可執行。
- Stage 4 stuck `DELIVERING` recovery 不重新發布 execution job；第一版 stuck execution 會被 guarded transition 標記為 terminal `FAILED`。

## Inbound Event Flow

```text
inbound event consumer
  -> receive Fund domain event from Azure SB session-enabled subscription
  -> validate envelope, subject and change
  -> map to webhook source, event key, resource and merchant scope
  -> begin DB transaction
  -> query active, non-deleted subscriptions by merchantId + eventKey
  -> if none: commit and complete inbound message without persistence
  -> if matches exist:
       create PENDING webhook_delivery per subscription
       ignore duplicate composite delivery identity safely
       commit DB transaction
       publish BullMQ webhook.delivery.execute job per new delivery
       publish failure: leave delivery PENDING for recovery
       complete inbound message
```

If DB persistence fails, the inbound message must not be completed and remains eligible for redelivery or DLQ handling. If DB commit succeeds but delivery job publish fails, the inbound message can still be completed because stale `PENDING` delivery recovery republishes the job. If DB commit succeeds but inbound message completion fails, redelivery relies on delivery composite uniqueness and worker lock safety.

## Deferred Inbox Flow

當 Fund 或其他 producer 提供穩定 `eventId` 後，target flow 改為：

```text
inbound event consumer
  -> validate event id, envelope, subject and change
  -> map to webhook source, event key, resource and merchant scope
  -> begin DB transaction
  -> insert webhook_inbox_event PENDING
       UNIQUE source + event_id
  -> duplicate: commit and complete inbound message
  -> new event: commit and complete inbound message

inbox dispatcher
  -> fetch PENDING inbox events
  -> match subscriptions
  -> create deliveries per subscription
       UNIQUE inbox_event_id + subscription_id
  -> mark inbox event DISPATCHED
```

此 deferred flow 取代 direct-to-delivery 的複合事件去重，並補足 no-subscriber receipt、audit 與 replay。Producer outbox 是否存在不由 Webhook ownership 決定，但 producer 必須在 event contract 提供穩定 `eventId`。

## Delivery Publishing Flow

```text
inbound event consumer
  -> commit newly created webhook_delivery rows
  -> call delivery job publisher with created delivery ids
  -> publish BullMQ webhook.delivery.execute job per delivery id
  -> publish success: complete inbound message
  -> publish failure: keep delivery PENDING and complete inbound message

recovery scheduler
  -> find stale PENDING deliveries whose jobs may not have been published
  -> republish BullMQ webhook.delivery.execute job per delivery id
```

Delivery job publishing is outside the delivery creation DB transaction. The normal path publishes immediately after commit; polling exists only as recovery, not as the primary delivery latency path.

## Delivery Worker Flow

```text
delivery worker
  -> receive delivery id
  -> lock PENDING delivery as DELIVERING
  -> load payload snapshot and endpoint snapshot
  -> load signing secret
  -> sign payload
  -> POST endpoint with 10 second request timeout
  -> mark SUCCESS on 2xx
  -> mark FAILED on non-2xx / timeout / transport error
```

Worker must lock delivery via state transition or equivalent atomic mechanism so the same delivery is not processed concurrently by multiple workers.

First-version delivery execution does not retry. `FAILED` is terminal. HTTP non-2xx response, HTTP request timeout after 10 seconds, and transport error all mark the delivery `FAILED`. The first version does not add `RETRYING`, retry count, next retry time, or delivery attempt fields.

## Recovery Flow

```text
recovery scheduler
  -> find stale PENDING delivery whose job may not have been published
  -> find DELIVERING delivery stuck beyond timeout
  -> republish BullMQ webhook.delivery.execute jobs only for stale PENDING deliveries
  -> mark stuck DELIVERING deliveries FAILED with guarded state transition
```

First-version recovery only guarantees unpublished pending jobs can be requeued and stuck execution can be terminally closed. Retry count, backoff and manual resend are outside the first-version product scope.

## Failure Boundaries

- Merchant endpoint timeout or 5xx must not affect the transaction domain flow.
- Webhook inbound consumer failure must not drop the domain event notification silently.
- Delivery creation must be atomic for all subscriptions matched during one consumer attempt.
- Delivery job publish failure after DB commit must not make pending deliveries permanently unprocessable.
- Delivery worker failure must not lose the delivery record.
- BullMQ publish failure compensation must be explicit in Stage 3; stuck execution terminal failure compensation belongs to Stage 4.

## Flow

```text
fund / transaction domain service
  -> publish Azure SB domain event topic fund.deposit.status-changed or fund.withdrawal-intent.status-changed with sessionId
  -> webhook inbound event consumer on srv-webhook subscription
  -> validate envelope + subject + change
  -> map to webhook source + eventKey + resource + merchant scope
  -> match subscription by merchantId + eventKey
  -> create webhook_delivery PENDING per subscription
  -> commit delivery rows
  -> immediately publish BullMQ webhook.delivery.execute job with deliveryId
  -> delivery worker locks delivery
  -> POST endpoint
  -> update delivery status
```

## Open Points

- Delivery execution queue 是否需要額外 job metadata，例如 trace id、publishedAt、source。
- Delivery publish recovery cadence and publish-state representation, if a queued/published marker is needed.
- Fund broadcast event 的 topic / subscription naming、payload data strategy 與 `occurredAt` contract 見 [`design-inbound-event-contract.md`](./design-inbound-event-contract.md)。

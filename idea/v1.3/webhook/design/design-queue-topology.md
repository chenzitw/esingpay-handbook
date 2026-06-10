---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Queue Topology Design

## Purpose

本文先定義 webhook 第一版會用到的 queue 階段、topic 命名方向與 payload envelope，作為後續 blueprint / plan 討論基礎。

這不是最終 queue provider 設定，也不定義 consumer group、retry、dead letter、部署資源名稱或實際 TypeScript DTO 檔案位置。那些細節留給 blueprint / plan 依服務實作與 infra convention 定案。

## Context

交易狀態變更不是由 fund / withdrawal / deposit 服務同步呼叫 webhook service。其他服務在處理交易狀態變更時，會向 queue 發出一筆 domain event notification。Webhook service 訂閱該 queue，收到事件後建立 `webhook_outbox_event`。

Webhook 後續仍有自己的派送 queue：dispatcher 將 outbox event match subscription 後建立 `webhook_delivery`，再把 delivery job 放入 queue，等待 delivery worker 執行 HTTP POST 並更新 delivery status。

因此第一版至少有兩段 queue：

- Domain event notification queue：fund 或其他交易服務發布交易事件狀態變更，webhook service 消費並建立 outbox event。
- Webhook delivery execution queue：webhook dispatcher 發布 delivery job，webhook worker 消費並執行 endpoint POST。

Queue provider 第一版暫定採 Azure Service Bus。正式 queue/topic 名稱、consumer 設定、dead letter、retry/backoff、資源命名與部署設定留給 plan 依 infra convention 定案。

## Topic Naming

Topic 命名先採以下概念格式：

```text
{domain}.{feature}.{action}
```

命名原則：

- `domain` 表示事件 owner，例如 `fund`。
- `feature` 表示業務資源或能力，例如 `deposit`、`withdrawal`。
- `action` 表示此 topic 代表的事件或工作，例如 `status-changed`、`wallet-sync`、`delivery-execute`。
- Topic 名稱應描述 producer 發出的事實或工作，不應描述 webhook 內部如何處理。

範例：

- `fund.deposit.wallet-sync`
- `fund.deposit.status-changed`
- `fund.withdrawal.status-changed`
- `webhook.delivery.execute`

實際 topic 是否拆成 deposit / withdrawal 各自一條，或 fund transaction 共用一條，留給 blueprint / plan 依 producer 邊界、事件量與 consumer filtering 能力決定。

## Queue Inventory

| Queue stage | Conceptual topic | Producer | Consumer | Purpose |
| --- | --- | --- | --- | --- |
| Domain event notification | `fund.deposit.status-changed` 或同級命名 | Fund / transaction domain service | Webhook inbound event consumer | 通知 webhook 某筆交易狀態已變更，可建立 outbox event。 |
| Domain event notification | `fund.withdrawal.status-changed` 或同級命名 | Fund / transaction domain service | Webhook inbound event consumer | 通知 webhook 某筆 withdrawal 狀態已變更，可建立 outbox event。 |
| Outbox dispatch trigger | `webhook.outbox.dispatch` 或 polling scheduler | Scheduler / dispatcher trigger | Webhook dispatcher | 觸發 pending outbox event dispatch；若採 polling scheduler 可不建此 topic。 |
| Delivery execution | `webhook.delivery.execute` | Webhook dispatcher / recovery scheduler | Webhook delivery worker | 要求 worker 執行某一筆 delivery。 |

## Domain Event Notification Payload

Domain event notification payload 是 webhook outbox event 的上游 input。它應提供 webhook 判斷事件類型、來源資源與 payload builder 所需的最小資料。

Conceptual envelope：

```json
{
  "eventKey": "deposit.completed",
  "resourceType": "deposit",
  "resourceIdentifier": "67890",
  "merchantId": "merchant_uuid",
  "occurredAt": "2026-06-10T02:00:00.000Z",
  "data": {}
}
```

欄位語意：

| Field | Meaning |
| --- | --- |
| `eventKey` | Webhook code-defined event catalog 內的穩定 event key，例如 `deposit.completed`。 |
| `resourceType` | 來源資源類型，例如 `deposit` 或 `withdrawal`。 |
| `resourceIdentifier` | 來源資源識別值。 |
| `merchantId` | 事件所屬商戶。 |
| `occurredAt` | 交易狀態變更發生時間；若 producer 無精確狀態變更時間，需在 blueprint / plan 決定 fallback。 |
| `data` | 交易 domain raw。Withdrawal / deposit raw 欄位見 [`design-type-contract.md`](./design-type-contract.md)。 |

Domain event notification rules：

- Webhook consumer 只接受 code-defined event catalog 內的 `eventKey`。
- `resourceType` / `resourceIdentifier` 會寫入 `webhook_outbox_event`。
- Webhook consumer 建立 outbox event 後，不同步呼叫商戶 endpoint。
- Producer 不應送出 webhook delivery id、signature、retry metadata 或商戶 endpoint 資訊。

## Webhook Delivery Execution Payload

Delivery execution payload 是 webhook 內部工作訊息，只需要讓 worker 找到要執行的 delivery。

Conceptual envelope：

```json
{
  "deliveryId": "12345"
}
```

Rules：

- Worker 收到 `deliveryId` 後，必須從 persistence 讀取 delivery snapshot。
- Worker 使用 delivery 上保存的 `endpoint_url` 與 `payload`，不重新查 subscription current endpoint 或交易現況。
- Worker 必須透過 delivery 狀態轉換鎖定任務，避免同一 delivery 被多個 worker 重複處理。
- Recovery scheduler 可重新發布同一 `deliveryId`，由 worker lock 機制決定是否可執行。

## Dispatcher Flow

```text
dispatcher
  -> fetch pending webhook_outbox_event batch
  -> for each outbox event:
       query active, non-deleted subscriptions by merchant_id + event_type
       if none:
         mark outbox event DISPATCHED
       else:
         create webhook_delivery per subscription
         publish webhook.delivery.execute job per delivery
         mark outbox event DISPATCHED
```

Outbox event status only tracks whether dispatcher has processed the event. If there are no subscribers, no delivery is created and the outbox event still becomes `DISPATCHED`.

Open decision for later planning: when subscribers exist, `DISPATCHED` can be marked after delivery rows are created and jobs are published, or after delivery rows are created with recovery compensating missing queue publishes.

## Delivery Worker Flow

```text
delivery worker
  -> receive delivery id
  -> lock PENDING delivery as DELIVERING
  -> load payload snapshot and endpoint_url snapshot
  -> load signing secret
  -> sign payload
  -> POST endpoint_url
  -> mark SUCCESS on 2xx
  -> mark FAILED on non-2xx / timeout / transport error
```

Worker must lock delivery via state transition or equivalent atomic mechanism so the same delivery is not processed concurrently by multiple workers.

## Recovery Flow

```text
recovery scheduler
  -> find PENDING delivery older than threshold
  -> find DELIVERING delivery stuck beyond timeout
  -> publish webhook.delivery.execute jobs
```

First-version recovery only guarantees stuck jobs can be requeued. Retry count, backoff and manual resend are outside the first-version product scope unless Stage 4 plan decides minimal fields are required.

## Failure Boundaries

- Merchant endpoint timeout or 5xx must not affect the transaction domain flow.
- Webhook inbound consumer failure must not drop the domain event notification silently.
- Dispatcher failure must not make pending outbox events permanently unprocessable.
- Delivery worker failure must not lose the delivery record.
- Queue publish failure compensation must be explicit in Stage 3 / Stage 4 plans.

## Flow

```text
fund / transaction domain service
  -> publish domain event notification
  -> webhook inbound event consumer
  -> validate eventKey
  -> create webhook_outbox_event PENDING
  -> dispatcher matches subscription by merchant_id + event_type
  -> create webhook_delivery
  -> publish webhook.delivery.execute
  -> delivery worker locks delivery
  -> POST endpoint_url
  -> update delivery status
```

## Open Points

- Fund 端實際 topic 要拆成 `fund.deposit.status-changed` / `fund.withdrawal.status-changed`，或合併為單一 transaction topic。
- `fund.deposit.wallet-sync` 是否是實際 webhook event 上游，或只是 fund 內部同步事件。
- Domain event notification payload 的 `data` 是否放完整 domain raw，或只放 resource id 讓 webhook consumer 查詢來源資料。
- `occurredAt` 的權威來源是交易狀態變更時間、交易 `updatedAt`，還是 producer 發送時間。
- Delivery execution queue 是否需要額外 job metadata，例如 trace id、publishedAt、source。
- Azure Service Bus queue/topic naming convention and consumer setting.
- Dispatcher queue trigger vs polling scheduler.

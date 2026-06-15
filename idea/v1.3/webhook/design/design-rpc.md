---
status: draft
updated_at: 2026-06-15
updated_by: Codex
---

# Webhook 交易事件推播 — RPC Design

## Purpose

本文件鎖定 webhook 第一版 internal service-to-service capability，以及 api-gateway REST RPC proxy 對 webhook management RPC 的 route key 命名。Merchant / platform 後台 REST API 見 [`design-rest.md`](./design-rest.md)。

具體 contract-rpc 檔案位置、method name、input/output DTO 型別與 error mapping 留給 plan 依 codebase convention 決定。

## Capability Boundary

Webhook 是交易事件推播能力的 owner。Withdrawal / deposit 服務不應直接建立 delivery 或呼叫商戶 endpoint；它們只需要向 webhook capability 表達「某交易事件已發生」。

Merchant console / platform console 不直接呼叫 webhook service。後台管理 flow 由 api-gateway 暴露 REST route，再透過 REST RPC proxy 呼叫 webhook service management RPC：

```text
merchant console / platform console
  -> api-gateway REST
  -> REST RPC proxy route
  -> webhook service management RPC
  -> webhook subscription persistence
```

第一版 management RPC 統一信任 api-gateway 已完成登入驗證，並信任 api-gateway 傳入的 `identity.merchantId` 作為 merchant scope 來源。Webhook service 不重複驗證 JWT；非 api-gateway caller 的 access control 依 codebase 既有 internal RPC convention 控制。

概念邊界：

```text
withdrawal / deposit service
  -> webhook event production capability
  -> webhook_outbox_event
```

Dispatcher / worker / recovery 是 webhook 服務內部能力：

```text
webhook dispatcher
  -> subscription matching
  -> delivery creation
  -> publish delivery job

webhook delivery worker
  -> delivery lock
  -> signing
  -> HTTP POST
  -> delivery status update
```

## Conceptual RPC Capabilities

Stage 1 subscription management RPC 的 subscription summary / detail output 是 management read model。`eventTypeCount` 與 `eventTypes` 由 `webhook_subscription_event_type` binding 與 code-defined event catalog 組裝，不是 `WebhookSubscription` 本體欄位。

## Stage 1 Management RPC Method Inventory

以下 method name 是 design-level naming anchor，最終 contract-rpc service name 與檔案位置留給 plan 依 codebase convention 定案。

| Conceptual method | Purpose |
| --- | --- |
| `WebhookManagement.ListEventTypes` | 查詢 code-defined event type options。 |
| `WebhookManagement.ListSubscriptions` | 查詢目前 merchant 的 subscription list。 |
| `WebhookManagement.CreateSubscription` | 建立 subscription 與 event bindings。 |
| `WebhookManagement.GetSubscription` | 查詢目前 merchant 的單一 subscription detail。 |
| `WebhookManagement.UpdateSubscription` | 覆蓋 endpoint URL 與 event bindings。 |
| `WebhookManagement.DeleteSubscription` | 軟刪除 subscription。 |

## Stage 1 REST RPC Route Keys

以下 route key 是 api-gateway REST RPC proxy 與 webhook service REST RPC adapter 之間的 naming anchor。它們不同於 conceptual `WebhookManagement.*` method name；plan 可依 codebase convention 決定實際檔案與 typed API 定義位置，但 route key 語意需保持穩定。

命名規則：

- Prefix 固定為 `rest.webhook`。
- Merchant account surface 使用 `merch-*` segment，對應 `/webhook/merch/...` REST route。
- Platform account surface 使用 `plat-*` segment，對應 `/webhook/plat/...` REST route。
- `plat-subscription` 表示 platform account surface 管理 merchant webhook subscriptions；不代表 platform 自己擁有 webhook subscription。

| Account surface | REST RPC route key | REST route | Conceptual method |
| --- | --- | --- | --- |
| Merchant | `rest.webhook.merch-event-type.list` | `GET /webhook/merch/event-types` | `WebhookManagement.ListEventTypes` |
| Platform | `rest.webhook.plat-event-type.list` | `GET /webhook/plat/event-types` | `WebhookManagement.ListEventTypes` |
| Merchant | `rest.webhook.merch-subscription.list` | `GET /webhook/merch/subscriptions` | `WebhookManagement.ListSubscriptions` |
| Merchant | `rest.webhook.merch-subscription.view` | `GET /webhook/merch/subscriptions/{subscriptionId}` | `WebhookManagement.GetSubscription` |
| Merchant | `rest.webhook.merch-subscription.create` | `POST /webhook/merch/subscriptions` | `WebhookManagement.CreateSubscription` |
| Merchant | `rest.webhook.merch-subscription.update` | `PUT /webhook/merch/subscriptions/{subscriptionId}` | `WebhookManagement.UpdateSubscription` |
| Merchant | `rest.webhook.merch-subscription.delete` | `DELETE /webhook/merch/subscriptions/{subscriptionId}` | `WebhookManagement.DeleteSubscription` |
| Platform | `rest.webhook.plat-subscription.list` | `GET /webhook/plat/subscriptions` | `WebhookManagement.ListSubscriptions` |
| Platform | `rest.webhook.plat-subscription.view` | `GET /webhook/plat/subscriptions/{subscriptionId}` | `WebhookManagement.GetSubscription` |
| Platform | `rest.webhook.plat-subscription.create` | `POST /webhook/plat/subscriptions` | `WebhookManagement.CreateSubscription` |
| Platform | `rest.webhook.plat-subscription.update` | `PUT /webhook/plat/subscriptions/{subscriptionId}` | `WebhookManagement.UpdateSubscription` |
| Platform | `rest.webhook.plat-subscription.delete` | `DELETE /webhook/plat/subscriptions/{subscriptionId}` | `WebhookManagement.DeleteSubscription` |

Merchant / platform route keys may map to the same webhook service management RPC method. The account surface split exists so api-gateway can apply different REST route paths, permission decorators, identity extraction, and target merchant scope resolution before invoking the shared webhook capability.

### List Event Types

用途：讓 api-gateway 查詢可訂閱 event type options，供 merchant console 產生 checkbox。

概念 input：

- `identity.merchantId`

概念 output：

- event type option list

決策：

- Event type options 來源是 webhook service 的 code-defined catalog。
- Output 欄位應對齊 REST `WebhookEventTypeOptionDto`。

### List Subscriptions

用途：查詢目前商戶名下未軟刪除 webhook subscriptions。

概念 input：

- `identity.merchantId`
- paging input

概念 output：

- subscription summary list
- paging metadata

### Create Subscription

用途：建立目前商戶名下 webhook subscription。

概念 input：

- `identity.merchantId`
- `endpointUrl`
- `eventKeys`

概念 output：

- subscription detail

決策：

- `eventKeys` 必須存在於 code-defined catalog。
- `eventKeys` 可為空陣列；空陣列表示建立 subscription endpoint 但不建立 event bindings。
- 同一 merchant 可建立多筆相同 endpoint URL 的 subscription。

### Get Subscription

用途：查詢目前商戶名下單一 webhook subscription detail。

概念 input：

- `identity.merchantId`
- `subscriptionId`

概念 output：

- subscription detail

決策：

- Lookup 必須使用 `subscriptionId + identity.merchantId`，不能只用 `subscriptionId` 查詢。

### Update Subscription

用途：覆蓋目前商戶名下 webhook subscription 的 endpoint URL 與訂閱事件集合。

概念 input：

- `identity.merchantId`
- `subscriptionId`
- `endpointUrl`
- `eventKeys`

概念 output：

- subscription detail

決策：

- Update 採完整覆蓋語意，`eventKeys` 取代既有 relation 集合。
- `eventKeys` 可為空陣列；空陣列表示刪除所有 event bindings 但保留 subscription。
- Lookup / update 必須使用 `subscriptionId + identity.merchantId`，不能只用 `subscriptionId` 更新。
- Update 需更新 `webhook_subscription.endpoint_url`，並以 delete existing bindings + insert request event keys 的方式重建 `webhook_subscription_event_type`。
- Endpoint URL 更新與 binding replacement 應在同一 transaction 或等效一致性邊界內完成。

### Delete Subscription

用途：軟刪除目前商戶名下 webhook subscription。

概念 input：

- `identity.merchantId`
- `subscriptionId`

概念 output：

- deleted result

決策：

- Delete 必須使用 `subscriptionId + identity.merchantId`，不能只用 `subscriptionId` 軟刪除。

### Produce Webhook Event

用途：讓交易 domain 在狀態變更時產生 webhook outbox event。

概念 input：

- `merchant_id`
- `event_key`
- `resource_type`
- `resource_identifier`
- `payload`

概念 output：

- outbox event id
- accepted / ignored result

決策：

- 交易主流程只需確認 outbox event 已寫入或被明確忽略，不等待 delivery。
- 若 event key 不存在於 code-defined catalog 或未啟用，錯誤語意由 plan 決定；但不應退化成同步派送。

### Match Subscriptions For Event

用途：dispatcher 依 outbox event 找出 active 且未刪除、且訂閱該 event type 的 subscriptions。

概念 input：

- `merchant_id`
- `event_type`

概念 output：

- matching subscription list

這個 capability 可是 service method，不一定需要 contract-rpc。若 dispatcher 與 subscription management 未來會拆成不同服務，plan 可將它提升成 RPC。

### Create Delivery

用途：為 outbox event 與 subscription 建立 delivery。

概念 input：

- outbox event reference
- subscription reference
- endpoint snapshot
- payload snapshot

概念 output：

- delivery id

### Execute Delivery

用途：worker 鎖定 delivery 並執行 HTTP POST。

概念 input：

- delivery id

概念 output：

- success / failed result
- delivery status after execution

這個 capability 通常由 queue consumer 觸發，不一定暴露為 RPC。

### Recover Stuck Deliveries

用途：recovery scheduler 找出 pending 或 timeout 的 delivery 並重新投入 delivery queue。

概念 input：

- timeout cutoff
- batch size

概念 output：

- requeued count

## Route Ownership

- 商戶 / 平台後台 REST routes 屬 api-gateway management surface。
- REST RPC route keys 屬 api-gateway REST RPC proxy contract；merchant route keys 使用 `rest.webhook.merch-*`，platform route keys 使用 `rest.webhook.plat-*`。
- Subscription management RPC 屬 webhook service，供 api-gateway REST RPC proxy 呼叫。
- 交易服務產生 outbox event 屬 internal webhook production capability。
- Dispatcher、worker、recovery 屬 webhook service internal orchestration。
- 若需要 gateway 或其他服務呼叫 webhook internal capability，應走 contract-rpc，而不是直接 import webhook module internals。

## Stage Relationship

- Stage 1 需要 subscription management RPC，供 api-gateway REST RPC proxy 呼叫；不需要 producer RPC。
- Stage 2 決定並落地 produce webhook event capability。
- Stage 3 需要 dispatcher 使用 subscription matching 與 delivery creation capability。
- Stage 4 需要 delivery execution 與 recovery capability。

## Open Points

- Produce webhook event 使用 true contract-rpc，或在同一 codebase 內先採 service/facade boundary。
- Event production failure 是否影響交易狀態變更 commit。
- Dispatcher / worker 是否需要獨立 service identity 或 job metadata。

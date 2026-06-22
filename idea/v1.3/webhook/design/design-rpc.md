---
status: draft
updated_at: 2026-06-22
updated_by: Codex
---

# Webhook 交易事件推播 — Management Transport Design

## Purpose

本文件鎖定 webhook 第一版 internal service-to-service capability，以及 api-gateway 透過 REST RPC proxy 呼叫 webhook management capability 的邊界。Merchant / platform 後台 REST API 見 [`design-rest.md`](./design-rest.md)。

Stage 1 codebase plan 採 `contract-rest + RestRpc` 作為 gateway-to-cradle management transport，不新增 public `contract-rpc/webhook` namespace。具體檔案位置、method name、input/output DTO 型別與 error mapping 留給 plan 依 codebase convention 決定。

## Capability Boundary

Webhook 是交易事件推播能力的 owner。Fund service 不應直接建立 webhook delivery 或呼叫商戶 endpoint；它只需向 domain event notification queue 發布「某交易事件已發生」。

Merchant console / platform console 不直接呼叫 webhook service。後台管理 flow 由 api-gateway 暴露 REST route，再透過 REST RPC proxy 呼叫 webhook service management capability：

```text
merchant console / platform console
  -> api-gateway REST
  -> REST RPC proxy route
  -> webhook service management capability
  -> webhook subscription persistence
```

第一版 management capability 統一信任 api-gateway 已完成登入驗證，並信任 api-gateway 傳入的 account identity 與 target merchant scope。Webhook service 不重複驗證 JWT；非 api-gateway caller 的 access control 依 codebase 既有 internal convention 控制。

概念邊界：

```text
fund service
  -> domain event notification queue
  -> webhook inbound event consumer
  -> subscription matching
  -> webhook_delivery
```

Inbound consumer / publisher / worker / recovery 是 webhook 服務內部能力：

```text
webhook delivery publisher
  -> pending delivery lookup
  -> publish delivery job

webhook delivery worker
  -> delivery lock
  -> signing
  -> HTTP POST
  -> delivery status update
```

## Conceptual Management Capabilities

Stage 1 subscription management capability 的 subscription summary / detail output 是 management read model。`eventTypeCount` 與 `eventTypes` 由 subscription-event binding 與 code-defined event catalog 組裝，不是 `WebhookSubscription` 本體欄位。

## Stage 1 Management Capability Inventory

以下 capability names 是 design-level naming anchors，最終 service method、RestRpc method 與檔案位置留給 plan 依 codebase convention 定案。

| Conceptual capability | Purpose |
| --- | --- |
| `WebhookManagement.ListEventTypes` | 查詢 code-defined event type options。 |
| `WebhookManagement.ListSubscriptions` | 查詢目前 merchant 的 subscription list。 |
| `WebhookManagement.CreateSubscription` | 建立 subscription 與 event bindings。 |
| `WebhookManagement.GetSubscription` | 查詢目前 merchant 的單一 subscription detail。 |
| `WebhookManagement.UpdateSubscription` | 覆蓋 endpoint URL 與 event bindings。 |
| `WebhookManagement.DeleteSubscription` | 軟刪除 subscription。 |

## Stage 1 REST RPC Route Keys

以下 route key 是 api-gateway REST RPC proxy 與 webhook service REST RPC adapter 之間的 naming anchor。它們不同於 conceptual `WebhookManagement.*` capability name；plan 可依 codebase convention 決定實際檔案與 typed API 定義位置，但 route key 語意需保持穩定。

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
- Output 欄位應對齊 REST event type option read model。

### List Subscriptions

用途：查詢目前商戶名下未軟刪除 webhook subscriptions。

概念 input：

- `identity.merchantId`
- paging input
- platform surface 可帶 target merchant filters

概念 output：

- subscription summary list
- paging metadata

### Create Subscription

用途：建立 webhook subscription。Merchant surface 使用目前登入 merchant；platform surface 需提供 target merchant。

概念 input：

- `identity.merchantId`
- target merchant scope（platform surface）
- `endpointUrl`
- `eventKeys`

概念 output：

- subscription detail

決策：

- `eventKeys` 必須存在於 code-defined catalog。
- `eventKeys` 可為空陣列；空陣列表示建立 subscription endpoint 但不建立 event bindings。
- 同一 merchant 可建立多筆相同 endpoint URL 的 subscription。

### Get Subscription

用途：查詢單一 webhook subscription detail。Merchant surface 必須限制在目前 merchant；platform surface 可依 active subscription id 查詢。

概念 input：

- `identity.merchantId`
- `subscriptionId`

概念 output：

- subscription detail

決策：

- Lookup 必須使用 `subscriptionId + identity.merchantId`，不能只用 `subscriptionId` 查詢。
- Platform surface 不使用登入 merchant ownership 作為限制，但仍只能查 active subscription。

### Update Subscription

用途：覆蓋 webhook subscription 的 endpoint URL 與訂閱事件集合。Merchant surface 必須限制在目前 merchant；platform surface 可依 active subscription id 更新。

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
- Platform surface 不使用登入 merchant ownership 作為限制，但仍只能更新 active subscription。
- Update 需更新 subscription endpoint URL，並以 replace bindings 的方式使 request event keys 成為完整訂閱集合。
- Endpoint URL 更新與 binding replacement 應在同一 transaction 或等效一致性邊界內完成。

### Delete Subscription

用途：軟刪除 webhook subscription。Merchant surface 必須限制在目前 merchant；platform surface 可依 active subscription id 刪除。

概念 input：

- `identity.merchantId`
- `subscriptionId`

概念 output：

- deleted result

決策：

- Delete 必須使用 `subscriptionId + identity.merchantId`，不能只用 `subscriptionId` 軟刪除。
- Platform surface 不使用登入 merchant ownership 作為限制，但仍只能刪除 active subscription。

### Consume Domain Event

用途：讓 webhook inbound consumer 接收 Fund domain event，matching subscriptions 並直接建立 deliveries。這是 queue-triggered internal capability，不暴露為 management RPC。

概念 input：

- `source`
- `merchant_id`
- `event_type`
- `resource_type`
- `resource_identifier`
- `occurred_at`
- domain raw payload

概念 output：

- created delivery ids
- no-subscriber / duplicate result

決策：

- 第一版 `source` 固定為 `fund`；event type 必須存在於 code-defined catalog。
- Matching subscriptions 與建立 deliveries 應在同一 DB transaction 內完成，commit 後才 complete inbound queue message。
- 沒有 matching subscription 時不建立 persistence record。
- 重複訊息由 delivery 的來源／資源／subscription 複合唯一性安全忽略。

### Match Subscriptions For Event

用途：inbound event consumer 找出 active 且未刪除、且訂閱該 event type 的 subscriptions。

概念 input：

- `merchant_id`
- `event_type`

概念 output：

- matching subscription list

這個 capability 是 webhook service method，不需要 contract-rpc；Fund service 只透過 queue 發布事件。

### Create Delivery

用途：為 inbound domain event 與 subscription 建立 delivery。

概念 input：

- source、event type 與 resource reference
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
- Subscription management capability 屬 webhook service，供 api-gateway REST RPC proxy 呼叫。
- Fund service 發布 domain event notification；webhook inbound consumption 不使用 contract-rpc。
- Inbound consumer、delivery publisher、worker、recovery 屬 webhook service internal orchestration。

## Stage Relationship

- Stage 1 需要 subscription management capability，供 api-gateway REST RPC proxy 呼叫。
- Stage 2 落地 queue-triggered inbound event consumption、subscription matching 與 delivery creation capability。
- Stage 3 落地 pending delivery publishing 與 publish recovery capability。
- Stage 4 需要 delivery execution 與 recovery capability。

## Open Points

- Inbound event consumer / worker 是否需要獨立 service identity 或 job metadata。

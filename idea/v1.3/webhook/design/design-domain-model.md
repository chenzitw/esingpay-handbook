---
status: draft
updated_at: 2026-06-29
updated_by: Codex
---

# Webhook 交易事件推播 — Domain Model Design

## Purpose

本文先定義 webhook 服務在 domain 層需要管理的核心物件，作為後續 data model blueprint、domain raw、event contract、DTO 設計的上游語意來源。

Domain object 不等同於 DB table，也不等同於 API DTO。本文只描述系統需要理解與操作的業務概念、識別資訊、關聯與狀態；具體欄位型別、索引、migration 與 ORM mapping 留給 blueprint 或 plan。

## Object Inventory

| Domain object | 代表概念 | Persistence backing |
| --- | --- | --- |
| `WebhookSubscription` | 商戶登記的一個 callback endpoint 本體。 | `webhook_subscription` |
| `WebhookSubscriptionEventBinding` | 某個 subscription 訂閱了某個 event key 的關係。 | `webhook_subscription_event_type` |
| `WebhookEventType` | 系統正式支援且可被訂閱的 webhook event catalog item。 | TypeScript code-defined catalog |
| `WebhookDelivery` | 某一筆 inbound domain event 對某個 subscription endpoint 的實際派送任務。 | `webhook_delivery` |

## Webhook Subscription

Webhook subscription 表示商戶登記的一個 callback endpoint 本體。

Subscription 是商戶維度的資料。商戶只能看到與管理自己的 subscription。被軟刪除的 subscription 不出現在 UI 清單，也不參與後續事件派送。

第一版 subscription 的可編輯範圍以 UI 需求為準：商戶可建立 endpoint、修改 endpoint，並透過 binding 調整訂閱事件。

Conceptual properties：

| Property | 代表意義 |
| --- | --- |
| `id` | Subscription 識別碼。 |
| `merchantId` | 擁有此 subscription 的商戶識別碼。 |
| `endpointUrl` | 商戶接收 webhook POST 的 callback URL。 |
| `createdAt` | Subscription 建立時間。 |
| `updatedAt` | Subscription 最後更新時間。 |
| `deletedAt` | 軟刪除時間；有值時不出現在 UI，也不參與派送。 |

Domain rules：

- 同一 merchant 可建立多筆相同 endpoint URL 的 subscription；第一版不以 merchant + endpoint URL 做唯一性限制。
- Subscription 是否可參與派送由「未刪除且訂閱目標事件」判斷。
- 修改 endpoint 或訂閱事件只影響後續派送，不應改寫既有 delivery snapshot。
- `eventTypes` 不屬於 `WebhookSubscription` 本體欄位；它是由 binding 與 event catalog 組裝出的 management read model。

### Webhook Subscription Event Binding

Webhook subscription event binding 表示某個 subscription 訂閱了某個 event type。

這個概念主要用來表達 subscription 與 event type 的多對多關係。它不是第一版 UI 會獨立管理的資源，也不應暴露成商戶可直接操作的獨立物件。

Conceptual properties：

| Property | 代表意義 |
| --- | --- |
| `subscriptionId` | 訂閱關係所屬的 webhook subscription。 |
| `eventType` | 被訂閱的 webhook event key，例如 `withdrawal.completed`。 |
| `createdAt` | 訂閱關係建立時間。 |

Domain rules：

- 同一 subscription 不可重複訂閱同一 event type。
- 建立或更新 subscription 時，後端需確認 event type 存在於 code-defined event catalog。

## Subscription Management Read Models

Subscription management API 可以回傳為 UI 方便組裝的 read model，但 read model 不等同於 domain object。

`WebhookSubscriptionSummary` 用於列表，概念欄位包含：

| Property | 代表意義 |
| --- | --- |
| `id` | Subscription 識別碼。 |
| `endpointUrl` | Callback URL。 |
| `eventTypeCount` | 此 subscription 目前訂閱事件數量，由 binding 聚合得出。 |
| `createdAt` | Subscription 建立時間。 |
| `updatedAt` | Subscription 最後更新時間。 |

`WebhookSubscriptionDetail` 用於建立、查詢與更新後的 response，概念欄位包含：

| Property | 代表意義 |
| --- | --- |
| `id` | Subscription 識別碼。 |
| `endpointUrl` | Callback URL。 |
| `eventTypes` | 已訂閱 event options，由 binding join code-defined event catalog 組裝。 |
| `createdAt` | Subscription 建立時間。 |
| `updatedAt` | Subscription 最後更新時間。 |

Read model rules：

- Write model 操作 `WebhookSubscription` 與 `WebhookSubscriptionEventBinding`。
- Query model 可以為 REST / RPC response 組裝 `eventTypeCount` 或 `eventTypes`。
- Inbound event consumer matching 不依賴 subscription 物件內含 event list，而是查詢 `webhook_subscription_event_type` relation。

## Webhook Event Type

Webhook event type 是系統支援的事件類型目錄，例如 withdrawal completed 或 deposit failed。

這份目錄用於：

- 後台 UI 顯示事件 checkbox。
- 後端驗證商戶送出的訂閱事件是否有效。
- Inbound event consumer 判斷 domain event 對應哪些 subscription。

Webhook event type 目前不開放 UI CRUD，也不建 DB table。第一版由 TypeScript 檔案定義固定 catalog，服務啟動後視為系統設定資料。

Conceptual properties：

| Property | 代表意義 |
| --- | --- |
| `eventKey` | 穩定事件 key，例如 `withdrawal.completed`。 |
| `displayName` | 後台 UI 顯示名稱，例如 `Withdrawal completed`。 |
| `sortOrder` | UI 顯示或 catalog 排序用的順序值。 |

Domain rules：

- `eventKey` 必須唯一。
- `displayName` 由 code-defined catalog 提供，供第一版 UI 顯示；它不是 validation 或 consumer matching 的識別值。
- Event type 由系統控制，商戶不能自行建立或修改。
- Producer、inbound consumer、management API 都應以同一份 event type catalog 作為事件合法性來源。

目前事件清單草稿：

| Event key | Display name | Sort order |
| --- | --- | ---: |
| `withdrawal.created` | Withdrawal created | 10 |
| `withdrawal.cancelled` | Withdrawal cancelled | 20 |
| `withdrawal.failed` | Withdrawal failed | 30 |
| `withdrawal.completed` | Withdrawal completed | 40 |
| `deposit.created` | Deposit created | 50 |
| `deposit.failed` | Deposit failed | 60 |
| `deposit.completed` | Deposit completed | 70 |
| `deposit.blocked` | Deposit blocked | 80 |

`deposit.blocked` 第一版正式支援，需納入 code-defined catalog 與訂閱選項。

## Deferred Webhook Inbox Event

`WebhookInboxEvent` 是已確認但延後落地的可靠性目標，不屬於第一版 current object inventory。第一版 inbound consumer 直接建立 delivery；當 producer 能提供穩定 event id 時，Webhook 應引入 inbox event，作為 domain event 已被接收與等待 dispatch 的持久化紀錄。

Conceptual properties：

| Property | 代表意義 |
| --- | --- |
| `id` | Webhook 內部 inbox event 識別碼。 |
| `sourceEventId` | Producer 提供的穩定事件識別碼。 |
| `source` | 事件來源服務或 bounded context。 |
| `eventType` | 穩定 webhook event key。 |
| `resourceType` | 業務資源類型。 |
| `resourceIdentifier` | 業務資源識別值。 |
| `merchantId` | 事件所屬商戶。 |
| `occurredAt` | 上游事件發生時間。 |
| `payload` | Webhook 驗證與正規化後的事件 payload snapshot。 |
| `status` | `PENDING` 或 `DISPATCHED`。 |
| `receivedAt` | Webhook 成功保存事件的時間。 |
| `dispatchedAt` | Subscription matching 與 delivery creation 完成時間。 |

Target rules：

- `source + sourceEventId` 必須唯一，用於 queue redelivery 去重。
- Consumer 必須先 commit inbox event，再 complete inbound event message。
- Dispatcher 從 pending inbox event 建立 deliveries；沒有 matching subscription 時仍保留 inbox event 並標記 `DISPATCHED`。
- Inbox event 支援 inbound audit、no-subscriber receipt 與後續 replay；它不代表 webhook 已送達商戶 endpoint。
- Delivery 應以 `inboxEventId + subscriptionId` 防止同一 inbox event 對同一 subscription 重複建立。

## Webhook Delivery

Webhook delivery 是某一筆 inbound domain event 對某一個 subscription endpoint 的實際派送任務與結果。第一版不在 webhook service 保存 outbox 或 inbox event；consumer 收到事件後直接 matching subscriptions 並建立 delivery。

Delivery 的重點是「這筆事件有沒有成功送到某個 URL」。一筆 inbound event 可以產生零筆、多筆 delivery：

- 沒有 subscription 訂閱該事件時，不建立 delivery。
- 一個 endpoint 訂閱該事件時，建立一筆 delivery。
- 多個 endpoint 訂閱同一事件時，建立多筆 delivery，各自追蹤成功、失敗或後續補償狀態。

Delivery 應保存事件來源、業務資源、派送當下的 endpoint 與 payload snapshot，避免 subscription 後續修改或刪除造成歷史派送紀錄失真。

Conceptual properties：

| Property | 代表意義 |
| --- | --- |
| `id` | Delivery 識別碼。 |
| `subscriptionId` | 此 delivery 目標的 webhook subscription。 |
| `merchantId` | 此 delivery 所屬商戶。 |
| `source` | 事件來源服務或 bounded context；第一版固定為 `fund`。 |
| `eventType` | 穩定 webhook event key，例如 `deposit.completed`。 |
| `resourceType` | 上游業務資源類型，例如 `deposit` 或 `withdrawal-intent`。 |
| `resourceIdentifier` | 業務資源識別值。 |
| `occurredAt` | 上游交易事件發生時間。 |
| `endpointUrl` | 派送當下的 callback URL snapshot。 |
| `payload` | 派送當下的 payload snapshot。 |
| `status` | Delivery worker 對此派送任務的處理狀態。 |
| `createdAt` | Delivery 建立時間。 |
| `updatedAt` | Delivery 最後更新時間。 |
| `deliveredAt` | 成功送達時間；尚未成功時可為空。 |

Status：

- `PENDING`：delivery 已建立，尚未開始派送。
- `DELIVERING`：worker 已鎖定並正在派送。
- `SUCCESS`：商戶 endpoint 回傳成功狀態。
- `FAILED`：商戶 endpoint 回傳失敗、timeout 或 worker 發生派送錯誤。

Domain rules：

- Delivery 是以 endpoint 為單位追蹤派送結果的任務。
- 同一 inbound event 可以為多個 matching subscriptions 建立 delivery，各自獨立成功或失敗。
- Delivery 的 `resourceType` / `resourceIdentifier` 追蹤上游 Fund subject；withdrawal 類 webhook event 第一版使用 `resourceType = withdrawal-intent` 與 `WithdrawalIntent.id`，對外 webhook `eventType` 與 payload 命名仍使用 `withdrawal`。
- 第一版以 `source + eventType + resourceType + resourceIdentifier + subscriptionId` 識別重複 delivery。
- 此複合識別假設同一業務資源的同一 event type 只會發生一次，只適用於 direct-to-delivery MVP。
- 引入 deferred inbox model 後，應改由 `inboxEventId + subscriptionId` 去重，並保留來源／資源欄位供追蹤。
- 第一版 direct-to-delivery 在沒有 matching subscription 時不建立 delivery，也不保留 inbound event receipt；deferred inbox 導入後則保留 receipt。
- Worker 必須透過狀態轉換或等效鎖定機制避免同一筆 delivery 被重複處理。
- Delivery snapshot 不應因 subscription 或來源交易後續修改而改變。

## Persistence Concepts

資料表命名採單數。概念層資料表如下：

| 資料表 | 用途 |
| ------ | ---- |
| `webhook_subscription` | 記錄商戶登記的 webhook endpoint。 |
| `webhook_subscription_event_type` | 記錄某個 webhook subscription 訂閱了哪些 event key。 |
| `webhook_delivery` | 記錄某一筆 inbound domain event 對某一個 subscription endpoint 的實際推播任務與結果。 |

Persistence 設計原則：

- 資料表自身主鍵與關聯鍵使用系統現行主鍵策略。
- `merchant_id` 維持既有 UUID 字串設計。
- 時間欄位使用具時區語意的 timestamp。
- 軟刪除 subscription 後，不應阻擋商戶用相同 endpoint 重新建立新的 subscription。
- Subscription endpoint 在同一 merchant 內允許重複。
- Delivery 需能在不依賴 subscription 或來源交易現況的情況下還原派送當下資訊。

具體欄位型別、索引、FK 與 migration 細節留給 blueprint 或 plan。

## Rejected Alternatives

- 單表同時保存事件與派送結果。
  - 不採用。單一事件可能要派送到多個 endpoint，若不拆 delivery，無法獨立追蹤每個 endpoint 的成功、失敗與重試狀態。

- Delivery 只透過 subscription join 取得 endpoint。
  - 不採用。Subscription 可能在 delivery 後被修改或刪除，delivery 需要保存派送當下的 endpoint snapshot。

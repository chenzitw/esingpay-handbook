---
status: draft
updated_at: 2026-06-10
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
| `WebhookOutboxEvent` | 交易服務已產生、等待 webhook 系統處理的一筆事件紀錄。 | `webhook_outbox_event` |
| `WebhookDelivery` | 某一筆 outbox event 對某個 subscription endpoint 的實際派送任務。 | `webhook_delivery` |

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
- Dispatcher matching 不依賴 subscription 物件內含 event list，而是查詢 `webhook_subscription_event_type` relation。

## Webhook Event Type

Webhook event type 是系統支援的事件類型目錄，例如 withdrawal completed 或 deposit failed。

這份目錄用於：

- 後台 UI 顯示事件 checkbox。
- 後端驗證商戶送出的訂閱事件是否有效。
- Dispatcher 判斷 outbox event 對應哪些 subscription。

Webhook event type 目前不開放 UI CRUD，也不建 DB table。第一版由 TypeScript 檔案定義固定 catalog，服務啟動後視為系統設定資料。

Conceptual properties：

| Property | 代表意義 |
| --- | --- |
| `eventKey` | 穩定事件 key，例如 `withdrawal.completed`。 |
| `displayName` | 後台 UI 顯示名稱，例如 `Withdrawal completed`。 |
| `sortOrder` | UI 顯示或 catalog 排序用的順序值。 |

Domain rules：

- `eventKey` 必須唯一。
- `displayName` 由 code-defined catalog 提供，供第一版 UI 顯示；它不是 validation 或 dispatcher matching 的識別值。
- Event type 由系統控制，商戶不能自行建立或修改。
- Producer、dispatcher、management API 都應以同一份 event type catalog 作為事件合法性來源。

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

## Webhook Outbox Event

Webhook outbox event 是 webhook 系統的事件時間序紀錄。它表示系統內部在某個時間點發生了一筆需要對外通知的交易事件。

Outbox event 的重點是「發生了什麼事件」，例如某筆 withdrawal completed。它保存事件本身、來源關聯與處理狀態，讓 webhook dispatcher 可以非同步處理，不阻塞原本的交易流程。

Outbox event 使用 resource 命名來追蹤來源資料：

- `resource_type` 表示來源類型，例如 deposit 或 withdrawal。
- `resource_identifier` 表示來源資料的識別值。

Outbox event 的事件類型保存為 event key 字串。若沒有任何 subscription 訂閱該事件，outbox event 仍保留紀錄；dispatcher 完成處理後以 `DISPATCHED` 表示已處理，不再另外使用沒有訂閱者狀態。

Conceptual properties：

| Property | 代表意義 |
| --- | --- |
| `id` | Outbox event 識別碼。 |
| `eventType` | 此事件對應的 webhook event key。 |
| `resourceType` | 來源交易資料類型，例如 deposit 或 withdrawal。 |
| `resourceIdentifier` | 來源交易資料識別值。 |
| `merchantId` | 此事件所屬商戶。 |
| `payload` | 待派送事件內容。第一版具體 external contract 另行定義。 |
| `status` | Dispatcher 對此 outbox event 的處理狀態。 |
| `createdAt` | Outbox event 建立時間。 |
| `dispatchedAt` | Dispatcher 完成處理時間；尚未完成時可為空。 |

Status：

- `PENDING`：事件已建立，尚未被 dispatcher 完成處理。
- `DISPATCHED`：事件已完成 dispatcher 處理；可能已建立 delivery，也可能因沒有訂閱者而未建立 delivery。

Domain rules：

- Outbox event 代表交易事件已發生，不代表 webhook 已送達商戶 endpoint。
- Outbox event 不直接呼叫商戶 endpoint，只作為 dispatcher 的處理來源。
- Dispatcher 未完成或處理失敗時，outbox event 維持 `PENDING`，由下一輪 dispatcher 重試。
- 沒有訂閱者時仍保留 outbox event，dispatcher 完成 matching 後標記為 `DISPATCHED`。

## Webhook Delivery

Webhook delivery 是某一筆 outbox event 對某一個 subscription endpoint 的實際派送任務與結果。

Delivery 的重點是「這筆事件有沒有成功送到某個 URL」。一筆 outbox event 可以產生零筆、多筆 delivery：

- 沒有 subscription 訂閱該事件時，不建立 delivery。
- 一個 endpoint 訂閱該事件時，建立一筆 delivery。
- 多個 endpoint 訂閱同一事件時，建立多筆 delivery，各自追蹤成功、失敗或後續補償狀態。

Delivery 應保存派送當下的 endpoint 與 payload snapshot。即使這些資料可透過 subscription 或 outbox event 關聯取得，snapshot 仍能避免 subscription 後續修改或刪除造成歷史派送紀錄失真。

Conceptual properties：

| Property | 代表意義 |
| --- | --- |
| `id` | Delivery 識別碼。 |
| `outboxEventId` | 此 delivery 來源的 outbox event。 |
| `subscriptionId` | 此 delivery 目標的 webhook subscription。 |
| `merchantId` | 此 delivery 所屬商戶。 |
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
- 同一 outbox event 可以建立多筆 delivery，各自獨立成功或失敗。
- Worker 必須透過狀態轉換或等效鎖定機制避免同一筆 delivery 被重複處理。
- Delivery snapshot 不應因 subscription 或 outbox event 後續修改而改變。

## Persistence Concepts

資料表命名採單數。概念層資料表如下：

| 資料表 | 用途 |
| ------ | ---- |
| `webhook_subscription` | 記錄商戶登記的 webhook endpoint。 |
| `webhook_subscription_event_type` | 記錄某個 webhook subscription 訂閱了哪些 event key。 |
| `webhook_outbox_event` | 記錄業務服務已產生、等待 webhook dispatcher 處理的一筆交易事件。 |
| `webhook_delivery` | 記錄某一筆 outbox event 對某一個 subscription endpoint 的實際推播任務與結果。 |

Persistence 設計原則：

- 資料表自身主鍵與關聯鍵使用系統現行主鍵策略。
- `merchant_id` 維持既有 UUID 字串設計。
- 時間欄位使用具時區語意的 timestamp。
- 軟刪除 subscription 後，不應阻擋商戶用相同 endpoint 重新建立新的 subscription。
- Subscription endpoint 在同一 merchant 內允許重複。
- Delivery 需能在不依賴 subscription 現況的情況下還原派送當下資訊。

具體欄位型別、索引、FK 與 migration 細節留給 blueprint 或 plan。

## Rejected Alternatives

- 單表同時保存事件與派送結果。
  - 不採用。單一事件可能要派送到多個 endpoint，若不拆 delivery，無法獨立追蹤每個 endpoint 的成功、失敗與重試狀態。

- Delivery 只透過 subscription join 取得 endpoint。
  - 不採用。Subscription 可能在 delivery 後被修改或刪除，delivery 需要保存派送當下的 endpoint snapshot。

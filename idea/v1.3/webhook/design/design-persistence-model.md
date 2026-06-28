---
status: draft
updated_at: 2026-06-29
updated_by: Codex
---

# Webhook 交易事件推播 — Persistence Model Design

## Purpose

本文件鎖定 webhook 第一版跨 phase 共用的 conceptual persistence shape。這不是 final DB schema；具體 SQL、ORM mapping、index name、migration 檔案、欄位型別、PK/FK 寫法與實際 index 定義留給 plan。

## Naming

- 資料表採單數：`webhook_subscription`、`webhook_subscription_event_type`、`webhook_delivery`。
- 商戶登記的 webhook endpoint 稱為 `webhook_subscription`。
- 推播目標 URL 使用 `endpoint_url`，UI 可顯示為 Callback URL。
- 可訂閱事件的穩定 key 使用 `event_type` 欄位保存，例如 `withdrawal.completed`。
- Webhook event type catalog 不建 DB table；第一版由 TypeScript 檔案定義固定事件清單與 `sortOrder`。
- Delivery 使用 `event_type` 保存 event key 字串，不透過 DB catalog id 關聯事件類型。
- 事件來源使用 `source`；來源交易資料追蹤使用 `resource_type` / `resource_identifier`。
- 實際推播任務稱為 `webhook_delivery`。

## Identifier And Time Semantics

- Webhook persistence record 需要有穩定內部識別值，對外 management API 使用平台 short id string representation。
- `merchant_id` 對齊既有 merchant identity 語意，作為 subscription ownership 與 inbound consumer matching 的主要 scope。
- `resource_identifier` 表示 withdrawal intent / deposit 等來源交易主體識別值，實際 primitive 由對應交易模型與 plan 決定。
- 時間欄位需保留具時區語意，讓 list sorting、delivery snapshot 與 recovery 判斷可穩定運作。

## Subscription State

- 軟刪除使用 `deleted_at`，被軟刪除的 subscription 不出現在 UI 清單，也不參與派送。

## Event Type Catalog

- Webhook event type catalog 是程式碼內的系統設定資料，不建 DB table，也不開放 UI CRUD。
- 第一版以 TypeScript 檔案定義 `eventKey`、`displayName` 與 `sortOrder`。
- UI checkbox 與後端訂閱校驗都以同一份 code-defined catalog 為準。
- `deposit.blocked` 第一版正式支援，需納入 code-defined catalog。

Catalog 草稿事件：

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

## Direct-To-Delivery Persistence

- 第一版不在 webhook service 建立 outbox 或 inbox event table。
- Webhook inbound event consumer 直接 matching subscriptions，為每個 matching subscription 建立一筆 `webhook_delivery`。
- Delivery 保存事件來源、業務資源、`endpoint_url` 與 `payload` snapshot。
- 若沒有 matching subscription，不建立 persistence record；第一版不提供 inbound event receipt、replay 或 no-subscriber audit。
- 目前 Fund event 沒有穩定 `eventId`，因此第一版以來源與資源複合語意防止重複 delivery。Producer 提供穩定 `eventId` 是導入 deferred inbox 的前置條件。

## Deferred Inbox Persistence

`webhook_inbox_event` 是已確認但延後落地的可靠性目標。第一版不建立此 table；當 producer 提供穩定 `eventId` 後，應以 migration 與後續 blueprint 導入。

目標用途：

- 保存 Webhook 已接收的 domain event，不依賴是否存在 matching subscription。
- 以 producer event id 處理 broker redelivery 去重。
- 支援 no-subscriber receipt、inbound audit 與後續 replay。
- 將 inbound event consumption 與 subscription matching / delivery creation 分成可恢復的兩個階段。

目標概念欄位：

- `id`
- `event_id`
- `source`
- `event_type`
- `resource_type`
- `resource_identifier`
- `merchant_id`
- `occurred_at`
- `payload`
- `status`
- `received_at`
- `dispatched_at`

目標約束：

- `source + event_id` 唯一。
- `status` 第一版目標只需 `PENDING` / `DISPATCHED`。
- Consumer commit inbox event 後才 complete inbound event message；duplicate event 可安全 complete。
- Dispatcher 完成 matching 與 delivery creation 後標記 `DISPATCHED`；沒有 matching subscription 時同樣標記 `DISPATCHED`。
- Delivery 導入 `inbox_event_id` reference，並以 `inbox_event_id + webhook_subscription_id` 防止重複建立。
- 導入 inbox 後，第一版 direct-to-delivery 複合唯一鍵可保留為追蹤 index 或降級防線，但不再是主要事件冪等識別。

## Conceptual Persistence Inventory

### `webhook_subscription`

用途：記錄商戶登記的 webhook endpoint。

概念欄位：

- `id`
- `merchant_id`
- `endpoint_url`
- `created_at`
- `updated_at`
- `deleted_at`

約束：

- 同一 merchant 可建立多筆相同 `endpoint_url` 的 subscription；第一版不建立 merchant + endpoint URL 唯一性約束。
- 查詢可派送 subscription 時必須排除 deleted subscription。

### `webhook_subscription_event_type`

用途：記錄某個 webhook subscription 訂閱了哪些事件類型。

概念欄位：

- `webhook_subscription_id`
- `event_type`（保存 event key 字串）
- `created_at`

約束：

- 同一 subscription 不可重複訂閱同一 event type；實際以 composite key、unique constraint 或等效 repository 保護由 plan 定案。
- `event_type` 必須存在於 code-defined webhook event type catalog；DB 層不建立 event type FK。
- Relation 指向 `webhook_subscription`。

### `webhook_delivery`

用途：記錄某一筆 inbound domain event 對某一個 subscription endpoint 的實際推播任務與結果。

概念欄位：

- `id`
- `webhook_subscription_id`
- `merchant_id`
- `source`
- `event_type`
- `resource_type`
- `resource_identifier`
- `occurred_at`
- `endpoint_url`
- `payload`
- `status`
- `created_at`
- `updated_at`
- `delivered_at`

狀態：

- `PENDING`
- `DELIVERING`
- `SUCCESS`
- `FAILED`

狀態語意：

- 第一版不做 retry/backoff；`FAILED` 是 terminal status。
- HTTP non-2xx、10 秒 request timeout、transport error 或 stuck `DELIVERING` recovery 都會讓 delivery 進入 `FAILED`。
- 第一版不需要 `RETRYING`、retry count、next retry time 或 attempt 欄位；若未來導入 retry policy，需回到本文件更新 conceptual inventory。

約束：

- `webhook_subscription_id` 對應派送建立當下的 subscription。
- `source` 表示事件來源服務或 bounded context；第一版固定為 `fund`。
- `event_type` 保存 code-defined webhook event key。
- `resource_type` 第一版為 `deposit` 或 `withdrawal-intent`，用於追蹤上游 Fund subject；`event_type` 仍保存對外 webhook event key，例如 `withdrawal.completed`。
- `resource_identifier` 保存對應交易識別值；withdrawal 事件保存 `WithdrawalIntent.id`，deposit 事件保存 `Deposit.id`。
- `endpoint_url` 保存派送當下的 URL snapshot。
- `payload` 保存派送當下的 payload snapshot。
- 第一版需以 `source + event_type + resource_type + resource_identifier + webhook_subscription_id` 建立 unique constraint 或等效一致性保護。
- 上述複合唯一性假設同一資源的同一 event type 只會發生一次，只適用於 direct-to-delivery MVP；導入 deferred inbox 後改由 inbox reference 去重。
- Worker 必須透過狀態轉換鎖定 delivery，避免多 worker 重複處理同一任務。

## Query Support Requirements

Plan 需讓 persistence 支援以下查詢與一致性需求；實際 index、constraint、migration 語法與命名由 codebase plan 定案。

- 依 merchant scope 查詢未刪除 subscription。
- 依 subscription 找出其訂閱的 event type keys。
- 防止同一 subscription 重複綁定同一 event key。
- 依 event type 反向找出訂閱者，供 inbound event consumer matching 使用。
- 防止同一來源事件對同一 subscription 重複建立 delivery。
- 依 delivery status 找出尚未發布或需要 recovery 的 delivery。
- 支援同一 merchant 建立多筆相同 endpoint URL 的 subscription；不得用唯一性約束阻止此行為。

## Stage Relationship

- Stage 1 建立 `webhook_subscription`、`webhook_subscription_event_type`，並建立 code-defined event type catalog。
- Stage 2 建立 `webhook_delivery`，並完成 inbound event matching、payload snapshot 與複合冪等保護。
- Stage 3 在 delivery commit 後發布 execution job，並補償 stale pending delivery 的 queue publish failure。
- Stage 4 補足 delivery worker、recovery 與 signing；第一版不新增 retry count 或 next retry time。

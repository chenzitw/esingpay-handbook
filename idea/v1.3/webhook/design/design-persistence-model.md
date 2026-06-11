---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Persistence Model Design

## Purpose

本文件鎖定 webhook 第一版跨 phase 共用的 conceptual schema shape。這不是 final DB schema；具體 SQL、ORM mapping、index name、migration 檔案與欄位型別細節留給 plan。

## Naming

- 資料表採單數：`webhook_subscription`、`webhook_subscription_event_type`、`webhook_outbox_event`、`webhook_delivery`。
- 商戶登記的 webhook endpoint 稱為 `webhook_subscription`。
- 推播目標 URL 使用 `endpoint_url`，UI 可顯示為 Callback URL。
- 可訂閱事件的穩定 key 使用 `event_type` 欄位保存，例如 `withdrawal.completed`。
- Webhook event type catalog 不建 DB table；第一版由 TypeScript 檔案定義固定事件清單與 `sortOrder`。
- Outbox event 使用 `event_type` 保存 event key 字串，不再透過 DB catalog id 關聯事件類型。
- 來源交易資料追蹤使用 `resource_type` / `resource_identifier`。
- 實際推播任務稱為 `webhook_delivery`。

## ID And Time Strategy

- 資料表自身 `id` 與 FK 欄位使用 bigint。
- `merchant_id` 保持 string，對齊既有 merchant UUID 設計。
- `resource_identifier` 若對應 withdrawal / deposit 主表 ID，使用 bigint。
- 時間欄位使用 PostgreSQL `timestamptz`。

## Subscription State

- 軟刪除使用 `deleted_at`，被軟刪除的 subscription 不出現在 UI 清單，也不參與派送。

## Event Type Catalog

- Webhook event type catalog 是程式碼內的系統設定資料，不建 DB table，也不開放 UI CRUD。
- 第一版以 TypeScript 檔案定義 `eventKey` 與 `sortOrder`。
- UI checkbox 與後端訂閱校驗都以同一份 code-defined catalog 為準。
- `deposit.blocked` 第一版正式支援，需納入 code-defined catalog。

Catalog 草稿事件：

- `withdrawal.created`
- `withdrawal.cancelled`
- `withdrawal.failed`
- `withdrawal.completed`
- `deposit.created`
- `deposit.failed`
- `deposit.completed`
- `deposit.blocked`

## Outbox And Delivery Split

- `webhook_outbox_event` 是事件時間序紀錄，表示系統內部發生了一筆需要對外通知的交易事件。
- `webhook_delivery` 是該事件對某個 subscription endpoint 的派送任務與結果。
- 一筆 outbox event 可對應零筆、多筆 delivery。
- 若沒有訂閱者，dispatcher 不建立 delivery，並將 outbox event 標記為 `DISPATCHED`，表示該事件已完成 dispatcher 處理。
- Delivery 保存派送當下的 `endpoint_url` 與 `payload` snapshot，避免 subscription 或 outbox 後續變更造成歷史紀錄失真。

## Table Inventory

### `webhook_subscription`

用途：記錄商戶登記的 webhook endpoint。

欄位：

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

欄位：

- `webhook_subscription_id`（PK 一部分）
- `event_type`（PK 一部分，保存 event key 字串）
- `created_at`

主鍵：composite PK `(webhook_subscription_id, event_type)`，不另設 own `id` 欄位。

約束：

- 同一 subscription 不可重複訂閱同一 event type（由 composite PK 保證）。
- `event_type` 必須存在於 code-defined webhook event type catalog；DB 層不建立 event type FK。
- FK 指向 `webhook_subscription`。

### `webhook_outbox_event`

用途：記錄業務服務已產生、等待 webhook dispatcher 處理的一筆交易事件。

欄位：

- `id`
- `event_type`
- `resource_type`
- `resource_identifier`
- `merchant_id`
- `payload`
- `status`
- `created_at`
- `dispatched_at`

狀態：

- `PENDING`
- `DISPATCHED`

約束：

- `event_type` 保存 code-defined event key 字串。
- `payload` 保存事件內容本身。
- `status` 只表示 dispatcher 是否處理過該 outbox event。
- Dispatcher 未完成處理或處理失敗時，outbox event 維持 `PENDING`，讓下一輪 dispatcher 可重新拾取。
- Dispatcher 完成 subscription matching 後即標記 `DISPATCHED`；沒有訂閱者時不建立 delivery，但仍標記 `DISPATCHED`。

### `webhook_delivery`

用途：記錄某一筆 outbox event 對某一個 subscription endpoint 的實際推播任務與結果。

欄位：

- `id`
- `outbox_event_id`
- `webhook_subscription_id`
- `merchant_id`
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

約束：

- `outbox_event_id` 對應 `webhook_outbox_event.id`。
- `webhook_subscription_id` 對應 `webhook_subscription.id`。
- `endpoint_url` 保存派送當下的 URL snapshot。
- `payload` 保存派送當下的 payload snapshot。
- Worker 必須透過狀態轉換鎖定 delivery，避免多 worker 重複處理同一任務。

## Index Inventory (Stage 1)

以下為 Stage 1 migration 所需最小 index 集合。具體 index name 與 migration 語法留給 plan。

### `webhook_subscription`

| Index | Type | 用途 |
| --- | --- | --- |
| `(merchant_id, deleted_at)` | B-tree | Merchant scope query：快速篩出特定商戶的有效 subscription。 |
| `(merchant_id, endpoint_url)` | B-tree | Optional query support；不是唯一性約束，同商戶可重複建立相同 endpoint。 |

### `webhook_subscription_event_type`

| Index | Type | 用途 |
| --- | --- | --- |
| `(webhook_subscription_id, event_type)` | Composite PK | 重複訂閱防護，兼作 subscription → event type 正向查詢 index。 |
| `(event_type)` | B-tree | 反向查詢：依 event type 找出訂閱中的 subscription；Stage 3 dispatcher 使用。 |

## Stage Relationship

- Stage 1 建立 `webhook_subscription`、`webhook_subscription_event_type`，並建立 code-defined event type catalog。
- Stage 2 建立 `webhook_outbox_event` 與 producer 所需狀態。
- Stage 3 建立或補足 `webhook_delivery` 與 dispatcher 所需狀態轉換。
- Stage 4 補足 delivery worker、recovery 與 signing 所需欄位；若需要 retry count 或 next retry time，需回到本文件更新 conceptual inventory。

---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Data Model Blueprint

## Purpose

本文件鎖定 webhook 第一版跨 phase 共用的 conceptual schema shape。這不是 final DB schema；具體 SQL、ORM mapping、index name、migration 檔案與欄位型別細節留給 plan。

## Naming

- 資料表採單數：`webhook_subscription`、`webhook_event_type`、`webhook_subscription_event_type`、`webhook_outbox_event`、`webhook_delivery`。
- 商戶登記的 webhook endpoint 稱為 `webhook_subscription`。
- 推播目標 URL 使用 `endpoint_url`，UI 可顯示為 Callback URL。
- 可訂閱事件的穩定 key 使用 `event_key`，例如 `withdrawal.completed`。
- Outbox event 使用 `event_id` 對應 `webhook_event_type.id`，不另存 `event_type` 字串。
- 來源交易資料追蹤使用 `correlation_type` / `correlation_identifier`。
- 實際推播任務稱為 `webhook_delivery`。

## ID And Time Strategy

- 資料表自身 `id` 與 FK 欄位使用 bigint。
- `merchant_id` 保持 string，對齊既有 merchant UUID 設計。
- `correlation_identifier` 若對應 withdrawal / deposit 主表 ID，使用 bigint。
- 時間欄位使用 PostgreSQL `timestamptz`。

## Subscription State

- 軟刪除使用 `deleted_at`，被軟刪除的 subscription 不出現在 UI 清單，也不參與派送。

## Event Type Catalog

- `webhook_event_type` 是系統 catalog，不開放 UI CRUD。
- Event type 資料由 migration seed 寫入。
- UI checkbox 與後端訂閱校驗都以這份 catalog 為準。
- `deposit.blocked` 第一版正式支援，需納入 migration seed。

Seed 草稿事件：

- `withdrawal.created`
- `withdrawal.canceled`
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
- 若沒有訂閱者，outbox event 標記為 `NO_SUBSCRIBERS`，不建立 delivery。
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

- 同一 merchant 的未刪除 endpoint 不可重複。
- 查詢可派送 subscription 時必須排除 deleted subscription。

### `webhook_event_type`

用途：記錄系統支援的 webhook 事件類型，供 UI checkbox 顯示與後端事件校驗使用。

欄位：

- `id`
- `event_key`
- `sort_order`
- `created_at`
- `updated_at`

約束：

- `event_key` 必須唯一。

### `webhook_subscription_event_type`

用途：記錄某個 webhook subscription 訂閱了哪些事件類型。

欄位：

- `webhook_subscription_id`
- `webhook_event_type_id`
- `created_at`

約束：

- 同一 subscription 不可重複訂閱同一 event type。
- FK 指向 `webhook_subscription` 與 `webhook_event_type`。

### `webhook_outbox_event`

用途：記錄業務服務已產生、等待 webhook dispatcher 處理的一筆交易事件。

欄位：

- `id`
- `event_id`
- `correlation_type`
- `correlation_identifier`
- `merchant_id`
- `payload`
- `status`
- `created_at`
- `dispatched_at`

狀態：

- `PENDING`
- `DISPATCHED`
- `FAILED`
- `NO_SUBSCRIBERS`

約束：

- `event_id` 對應 `webhook_event_type.id`。
- `payload` 保存事件內容本身。
- `status` 表示 dispatcher 對 outbox event 的處理狀態。
- `NO_SUBSCRIBERS` 表示事件存在，但沒有任何 subscription 需要接收。

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

## Stage Ownership

- Stage 1 建立 `webhook_subscription`、`webhook_event_type`、`webhook_subscription_event_type`。
- Stage 2 建立 `webhook_outbox_event` 與 producer 所需狀態。
- Stage 3 建立或補足 `webhook_delivery` 與 dispatcher 所需狀態轉換。
- Stage 4 補足 delivery worker、recovery 與 signing 所需欄位；若需要 retry count 或 next retry time，需回到本文件更新 conceptual inventory。

---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — API Surface Blueprint

## Purpose

本文件鎖定 webhook 第一版的商戶後台 API surface。它只定義 API route 與概念欄位，最終 DTO 型別、validation decorator、controller 檔案位置與 error mapping 留給 plan。

REST request / response 草案見 [`../design/design-rest.md`](../design/design-rest.md)。

## Subscription Management

### `GET /webhook/merch/subscriptions`

查詢目前商戶的 webhook subscription 清單。

概念行為：

- 支援 paging。
- 清單資料需能呈現 endpoint URL、訂閱事件數量、建立時間與更新時間。
- 僅回傳目前商戶名下且未軟刪除的 subscription。

### `POST /webhook/merch/subscriptions`

建立新的 webhook subscription。

概念 request：

- `endpointUrl`
- 訂閱的 event type keys

概念行為：

- 後端需驗證 event type keys 都存在於 event type catalog。
- 同一 merchant 的未刪除 endpoint 不可重複。

### `GET /webhook/merch/subscriptions/{subscriptionId}`

查詢單一 webhook subscription。

概念行為：

- 回傳 endpoint URL 與已訂閱事件清單。
- 僅允許查詢目前商戶名下且未軟刪除的 subscription。

### `PUT /webhook/merch/subscriptions/{subscriptionId}`

覆蓋 endpoint URL 與訂閱事件集合。

概念行為：

- 需重新檢查同商戶有效 endpoint 唯一性。
- 更新訂閱事件時，以 request 送入的事件集合覆蓋目前關聯集合。

### `DELETE /webhook/merch/subscriptions/{subscriptionId}`

軟刪除 webhook subscription。

概念行為：

- 刪除後不再接收新事件派送。
- 歷史 delivery 紀錄仍應保留派送當下 endpoint snapshot。
- 重複刪除或刪除不存在資料的錯誤語意由 plan 依既有 API pattern 決定。

## Event Type Read Model

### `GET /webhook/merch/event-types`

查詢可訂閱事件清單，以供 UI 產生 checkbox options。

概念行為：

- 事件選項來源必須是 `webhook_event_type` catalog，而不是前端 hard-code。
- 第一版支援事件包含 `deposit.blocked`。

## API Boundaries

- Webhook event type 不開放 UI CRUD。
- 第一版不提供 delivery history 查詢 API。
- 第一版不提供人工重送 delivery API。
- 第一版不提供 endpoint ownership verification API。
- 第一版不提供 secret rotation API。

## Stage Ownership

- Stage 1 落地 subscription management API 與 event type read model。
- Stage 2-4 不應改變 subscription management API 的外部語意；若 worker/delivery 需要額外欄位，應先確認是否屬 data model 或 API extension。

## Open Points

- Subscription list 是否需要回傳最新 delivery 狀態不在第一版範圍，若 UI 需要應另開 design extension。

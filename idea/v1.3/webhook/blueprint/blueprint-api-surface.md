---
status: draft
updated_at: 2026-06-09
updated_by: Codex
---

# Webhook 交易事件推播 — API Surface Blueprint

## Purpose

本文件鎖定 webhook 第一版的商戶後台 API surface。它只定義 API route 與概念欄位，最終 DTO 型別、validation decorator、controller 檔案位置與 error mapping 留給 plan。

## Subscription Management

### `GET /webhook-subscriptions`

查詢目前商戶的 webhook subscription 清單。

概念行為：

- 支援 paging。
- 清單資料需能呈現 endpoint URL、訂閱事件數量、active 狀態、建立時間與更新時間。
- 僅回傳目前商戶名下且未軟刪除的 subscription。

### `POST /webhook-subscriptions`

建立新的 webhook subscription。

概念 request：

- `endpoint_url`
- 訂閱的 event type keys

概念行為：

- 系統建立時同步寫入 `signing_secret` 與預設 `active = true`。
- 後端需驗證 event type keys 都存在且啟用。
- 同一 merchant 的未刪除 endpoint 不可重複。

### `GET /webhook-subscriptions/{subscription_id}`

查詢單一 webhook subscription。

概念行為：

- 回傳 endpoint URL、active 狀態與已訂閱事件清單。
- 僅允許查詢目前商戶名下且未軟刪除的 subscription。

### `PATCH /webhook-subscriptions/{subscription_id}`

修改 endpoint URL 或訂閱事件。

概念行為：

- 不允許透過此 API 修改 `signing_secret` 或 `active`。
- 若變更 endpoint URL，需重新檢查同商戶有效 endpoint 唯一性。
- 更新訂閱事件時，以 request 送入的事件集合覆蓋目前關聯集合。

### `DELETE /webhook-subscriptions/{subscription_id}`

軟刪除 webhook subscription。

概念行為：

- 刪除後不再接收新事件派送。
- 歷史 delivery 紀錄仍應保留派送當下 endpoint snapshot。
- 重複刪除或刪除不存在資料的錯誤語意由 plan 依既有 API pattern 決定。

## Event Type Read Model

UI 需要取得可訂閱事件清單以產生 checkbox。此能力有兩個可接受方向：

- 併入 subscription detail response。
- 提供獨立 read endpoint。

不論採用哪一種 API 形式，事件選項來源都必須是 `webhook_event_type` catalog，而不是前端 hard-code。

Plan 1 需依既有後台 API pattern 決定具體 route。若後台已有 catalog/read-options 類 route convention，優先沿用。

## API Boundaries

- Webhook event type 不開放 UI CRUD。
- 第一版不提供 delivery history 查詢 API。
- 第一版不提供人工重送 delivery API。
- 第一版不提供 endpoint ownership verification API。
- 第一版不提供 signing secret rotate API。

## Phase Ownership

- Phase 1 落地 subscription management API 與 event type read model。
- Phase 2-4 不應改變 subscription management API 的外部語意；若 worker/delivery 需要額外欄位，應先確認是否屬 data model 或 API extension。

## Open Points

- Event type read model 採獨立 endpoint 或併入 subscription detail。
- Subscription list 是否需要回傳最新 delivery 狀態不在第一版範圍，若 UI 需要應另開 design extension。

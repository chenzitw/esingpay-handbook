---
status: draft
updated_at: 2026-06-08
updated_by: Codex
---

# Webhook 交易事件推播 — Blueprint

## Context

本 blueprint 承接 [`design.md`](./design.md)，把 webhook 交易事件推播拆成可落地的 delivery slice，並鎖定 plan 前需要一致的 API surface、schema shape、命名與流程邊界。

第一版目標是同時支援：

- 商戶後台管理 webhook subscription。
- 系統將 withdrawal / deposit 交易事件轉換為 webhook outbox event。
- 系統依 subscription 建立 webhook delivery 並追蹤派送結果。

## Scope

In scope：

- Webhook subscription 管理 API。
- Webhook event type seed 與查詢來源。
- Webhook subscription 與 event type 的多對多關係。
- Webhook outbox event 與 webhook delivery 的拆表模型。
- Dispatcher / worker / recovery scheduler 的方向性流程。
- DB 欄位 inventory 與跨 plan 命名決策。

Out of scope：

- Migration SQL、ORM entity class 與 repository 檔案位置。
- Request / response DTO 的最終型別定義。
- Webhook payload external contract。
- Signature 演算法與驗簽 header。
- 重試次數、backoff、人工重送與 delivery history UI。
- Endpoint ownership verification flow。

## Landed Facts Assumed

- Merchant ID 沿用既有 UUID 字串設計，故 webhook schema 內的 `merchant_id` 保持字串語意。
- 其他資料表自身主鍵與 FK 使用 bigint 系統主鍵策略。
- 交易來源追蹤命名沿用既有服務慣例：`correlation_type` / `correlation_identifier`。
- 資料表命名採單數。
- Webhook event type 不是商戶可管理資料，由 migration seed 預先寫入。

## Plan Inventory

- Plan 1：Subscription management
  - 建立 webhook subscription、event type catalog、subscription-event relation 的 persistence 與商戶後台管理 API。
  - 依賴：event type seed 決策。

- Plan 2：Outbox event production
  - 讓 withdrawal / deposit 相關交易狀態變更寫入 webhook outbox event。
  - 依賴：event type catalog 已可查找正式事件。

- Plan 3：Dispatcher and delivery creation
  - Dispatcher 輪詢 pending outbox event，依 merchant 與 event 建立 webhook delivery。
  - 依賴：subscription management 與 outbox event production。

- Plan 4：Delivery worker and recovery
  - Worker 執行 endpoint POST、更新 delivery 結果；recovery scheduler 補償 pending 或 timeout delivery。
  - 依賴：dispatcher 已能建立 delivery。

第一版若需要壓縮交付範圍，可先完成 Plan 1 作為 UI 管理能力，再接 Plan 2-4 完成完整推播鏈路。

## Critical Decisions

### Naming

- 資料表採單數：`webhook_subscription`、`webhook_event_type`、`webhook_subscription_event_type`、`webhook_outbox_event`、`webhook_delivery`。
- 商戶登記的 webhook endpoint 稱為 `webhook_subscription`。
- 推播目標 URL 使用 `endpoint_url`，UI 可顯示為 Callback URL。
- 可訂閱事件的穩定 key 使用 `event_key`，例如 `withdrawal.completed`。
- Outbox event 使用 `event_id` 對應 `webhook_event_type.id`，不另存 `event_type` 字串。
- 來源交易資料追蹤使用 `correlation_type` / `correlation_identifier`。
- 實際推播任務稱為 `webhook_delivery`。

### ID And Time Strategy

- 資料表自身 `id` 與 FK 欄位使用 bigint。
- `merchant_id` 保持 string，對齊既有 merchant UUID 設計。
- `correlation_identifier` 若對應 withdrawal / deposit 主表 ID，使用 bigint。
- 時間欄位使用 PostgreSQL `timestamptz`。

### Subscription State

- Subscription 使用 `active` boolean，而非 enum status。
- `active = true` 表示可被 dispatcher 查詢並建立 delivery。
- `active = false` 表示資料保留，但不參與事件派送。
- `active` 第一版不開放 UI 調整，建立時預設為 true。
- 軟刪除使用 `deleted_at`，被軟刪除的 subscription 不出現在 UI 清單，也不參與派送。

### Signing Secret

- `signing_secret` 保存於 subscription。
- 第一版不開放 UI 調整。
- 建立 subscription 時由服務產生，或依服務環境變數套用預設值。
- 實作 plan 需確認 secret 是否加密保存；因系統需要用它產生簽章，不能只存不可逆 hash。

### Event Type Catalog

- `webhook_event_type` 是系統 catalog，不開放 UI CRUD。
- Event type 資料由 migration seed 寫入。
- UI checkbox 與後端訂閱校驗都以這份 catalog 為準。
- `deposit.blocked` 是否納入正式 seed 需在 Plan 1 前確認。

### Outbox And Delivery Split

- `webhook_outbox_event` 是事件時間序紀錄，表示系統內部發生了一筆需要對外通知的交易事件。
- `webhook_delivery` 是該事件對某個 subscription endpoint 的派送任務與結果。
- 一筆 outbox event 可對應零筆、多筆 delivery。
- 若沒有訂閱者，outbox event 標記為 `NO_SUBSCRIBERS`，不建立 delivery。
- Delivery 保存派送當下的 `endpoint_url` 與 `payload` snapshot，避免 subscription 或 outbox 後續變更造成歷史紀錄失真。

## API Surface

### Subscription Management

`GET /webhook-subscriptions`

- 查詢目前商戶的 webhook subscription 清單。
- 支援 paging。
- 清單資料需能呈現 endpoint URL、訂閱事件數量、active 狀態、建立時間與更新時間。
- 僅回傳目前商戶名下且未軟刪除的 subscription。

`POST /webhook-subscriptions`

- 建立新的 webhook subscription。
- Request 概念欄位包含 `endpoint_url` 與訂閱的 event type keys。
- 系統建立時同步寫入 `signing_secret` 與預設 `active = true`。
- 後端需驗證 event type keys 都存在且啟用。

`GET /webhook-subscriptions/{subscription_id}`

- 查詢單一 webhook subscription。
- 回傳 endpoint URL、active 狀態與已訂閱事件清單。
- 僅允許查詢目前商戶名下且未軟刪除的 subscription。

`PATCH /webhook-subscriptions/{subscription_id}`

- 修改 endpoint URL 或訂閱事件。
- 不允許透過此 API 修改 `signing_secret` 或 `active`。
- 若變更 endpoint URL，需重新檢查同商戶有效 endpoint 唯一性。
- 更新訂閱事件時，以 request 送入的事件集合覆蓋目前關聯集合。

`DELETE /webhook-subscriptions/{subscription_id}`

- 軟刪除 webhook subscription。
- 刪除後不再接收新事件派送。
- 歷史 delivery 紀錄仍應可保留派送當下 endpoint snapshot。

### Event Type Read Model

UI 需要取得可訂閱事件清單以產生 checkbox。此能力可併入 subscription detail response，或提供獨立 read endpoint；Plan 1 需依既有後台 API pattern 決定。

不論採用哪一種 API 形式，事件選項來源都必須是 `webhook_event_type` catalog，而不是前端 hard-code。

## Schema Shape

本節列出 plan 必須落地的欄位 inventory。具體 SQL、ORM mapping、index name 與 migration 檔案留給 plan。

### `webhook_subscription`

用途：記錄商戶登記的 webhook endpoint，以及該 endpoint 是否可用於接收事件推播。

欄位：

- `id`
- `merchant_id`
- `endpoint_url`
- `signing_secret`
- `active`
- `created_at`
- `updated_at`
- `deleted_at`

約束：

- 同一 merchant 的未刪除 endpoint 不可重複。
- 查詢 active subscription 時必須同時排除 deleted subscription。

### `webhook_event_type`

用途：記錄系統支援的 webhook 事件類型，供 UI checkbox 顯示與後端事件校驗使用。

欄位：

- `id`
- `event_key`
- `display_name`
- `category`
- `description`
- `is_active`
- `sort_order`
- `created_at`
- `updated_at`

約束：

- `event_key` 必須唯一。
- 第一版 category 至少涵蓋 withdrawal 與 deposit。
- Seed 草稿事件：
  - `withdrawal.created`
  - `withdrawal.canceled`
  - `withdrawal.failed`
  - `withdrawal.completed`
  - `deposit.created`
  - `deposit.failed`
  - `deposit.completed`
  - `deposit.blocked`

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

## Dispatch Orchestration

### Event Production

Withdrawal / deposit 服務在交易狀態變更時建立 webhook outbox event。交易主流程只負責寫入事件，不直接呼叫商戶 endpoint。

### Dispatching

Dispatcher 輪詢 pending outbox event，依 `merchant_id` 與 `event_id` 查找 active 且未刪除的 subscription。查詢需透過 subscription-event relation 判斷該 subscription 是否訂閱該 event type。

若找不到 subscription，outbox event 轉為 `NO_SUBSCRIBERS`。若找到 subscription，針對每一筆 subscription 建立 delivery，並將 delivery job 發布給 delivery worker。

### Delivery Execution

Delivery worker 消費 delivery job 後，先將 delivery 由 `PENDING` 鎖定為 `DELIVERING`。只有鎖定成功的 worker 能繼續產生簽章並 POST 到 endpoint。

HTTP response 為 2xx 時，delivery 轉為 `SUCCESS` 並記錄 delivered time。非 2xx、timeout 或其他錯誤時，delivery 轉為 `FAILED`。

### Recovery

Recovery scheduler 查詢仍為 `PENDING` 或 timeout 停留在 `DELIVERING` 的 delivery，重新投入 delivery queue。具體 timeout 與重試策略留給 plan 或後續 design extension。

## Sequencing

Dependencies：

- Plan 1 必須先確定 event type seed，否則 UI checkbox 與 subscription validation 無法落地。
- Plan 2 依賴 event type catalog，因 outbox event 需要對應正式 event ID。
- Plan 3 依賴 subscription relation 與 outbox event。
- Plan 4 依賴 delivery 建立流程。

Parallelism：

- UI 管理 API 與 event type seed 可先一起規劃。
- Outbox event production 可在 subscription schema 決策穩定後與 UI 細節並行。
- Delivery worker 與 recovery 應在 delivery schema 與狀態轉換規則穩定後再展開。

Delivery cadence：

- Stage 1：Webhook subscription 管理與 event type catalog。
- Stage 2：Outbox event production 與 dispatcher delivery 建立。
- Stage 3：Delivery worker、recovery 與派送結果追蹤。

## Pattern Gaps

- Outbox dispatcher 與 delivery worker 若 codebase 尚無既有 pattern，plan 需先 survey queue / scheduler / worker 既有實踐。
- Signing secret 的生成、保存與輪替若 codebase 尚無 convention，plan 需明確列出採用方式，後續穩定後蒸餾進 guide。
- Webhook payload external contract 尚未定案，Plan 2 前需決定 payload envelope 是否獨立成 design extension。

## Open Points

- `deposit.blocked` 是否正式納入第一版 seed。
- 是否提供獨立 event type read endpoint，或併入 subscription detail response。
- Outbox event 轉為 `DISPATCHED` 的精確時機。
- Delivery 失敗後是否立刻 `FAILED`，或需要 `RETRYING` / retry count 等額外狀態。
- Delivery timeout 門檻與 recovery scheduler cadence。
- Webhook signature 演算法、header 命名與驗簽文件。
- Webhook payload external contract 與版本策略。

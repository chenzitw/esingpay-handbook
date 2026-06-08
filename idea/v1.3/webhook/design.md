---
status: draft
updated_at: 2026-06-08
updated_by: Codex
---

# Webhook 交易事件推播 — Design

## Context

Webhook 讓商戶可以在後台登記 callback endpoint，接收系統主動推送的交易事件。第一版聚焦 withdrawal / deposit 相關事件，讓商戶不需要輪詢查詢 API，也能在交易狀態變更時更新自己的系統。

此設計同時覆蓋兩個面向：

- 商戶後台對 webhook endpoint 的觀看、登記、修改與刪除。
- 系統內部把交易事件轉換為對外 webhook delivery 的派送流程。

Webhook 派送不應綁在交易主流程內同步呼叫商戶 endpoint。交易服務只負責產生事件，後續由 webhook dispatcher、delivery worker 與 recovery scheduler 非同步處理。

## Scope

In scope：

- 商戶可管理自己名下的 webhook subscription。
- 商戶可為每個 webhook endpoint 選擇要接收的事件類型。
- 系統可根據交易事件建立 webhook outbox event。
- 系統可根據 outbox event 與 subscription 建立 delivery 任務。
- 系統可追蹤每個 endpoint 的派送結果。
- 系統可補償 pending 或 timeout 的 delivery 任務。

Out of scope：

- 事件 payload 的完整欄位 contract。
- 簽章演算法與 HTTP header 命名。
- delivery 重試次數、退避策略與最終失敗規則。
- 商戶手動重送 delivery 的後台功能。
- webhook endpoint 驗證流程。
- webhook event type 的 UI CRUD。

## Domain Model

### Webhook Subscription

Webhook subscription 表示商戶登記的一個 callback endpoint，以及該 endpoint 訂閱哪些 webhook event type。

Subscription 是商戶維度的資料。商戶只能看到與管理自己的 subscription。被軟刪除的 subscription 不出現在 UI 清單，也不參與後續事件派送。

第一版 subscription 的可編輯範圍以 UI 需求為準：商戶可建立 endpoint、修改 endpoint 與調整訂閱事件。`signing_secret` 與 `active` 不開放 UI 調整，由服務建立時套用預設值或環境設定。

### Webhook Event Type

Webhook event type 是系統支援的事件類型目錄，例如 withdrawal completed 或 deposit failed。

這份目錄用於：

- 後台 UI 顯示事件 checkbox。
- 後端驗證商戶送出的訂閱事件是否有效。
- Dispatcher 判斷 outbox event 對應哪些 subscription。

Webhook event type 目前不開放 UI CRUD。資料由 migration seed 預先寫入，服務啟動後視為系統設定資料。

目前事件清單草稿：

- `withdrawal.created`
- `withdrawal.canceled`
- `withdrawal.failed`
- `withdrawal.completed`
- `deposit.created`
- `deposit.failed`
- `deposit.completed`
- `deposit.blocked`

`deposit.blocked` 已出現在 UI 草稿，但原 Mermaid 流程尚未列入，需要在 blueprint 前確認是否正式支援。

### Webhook Outbox Event

Webhook outbox event 是 webhook 系統的事件時間序紀錄。它表示系統內部在某個時間點發生了一筆需要對外通知的交易事件。

Outbox event 的重點是「發生了什麼事件」，例如某筆 withdrawal completed。它保存事件本身、來源關聯與處理狀態，讓 webhook dispatcher 可以非同步處理，不阻塞原本的交易流程。

Outbox event 使用既有服務慣例的 correlation 命名來追蹤來源資料：

- `correlation_type` 表示來源類型，例如 deposit 或 withdrawal。
- `correlation_identifier` 表示來源資料的識別值。

Outbox event 的事件類型對應 webhook event type。若沒有任何 subscription 訂閱該事件，outbox event 仍保留紀錄，並標示為沒有訂閱者。

### Webhook Delivery

Webhook delivery 是某一筆 outbox event 對某一個 subscription endpoint 的實際派送任務與結果。

Delivery 的重點是「這筆事件有沒有成功送到某個 URL」。一筆 outbox event 可以產生零筆、多筆 delivery：

- 沒有 subscription 訂閱該事件時，不建立 delivery。
- 一個 endpoint 訂閱該事件時，建立一筆 delivery。
- 多個 endpoint 訂閱同一事件時，建立多筆 delivery，各自追蹤成功、失敗或後續補償狀態。

Delivery 應保存派送當下的 endpoint 與 payload snapshot。即使這些資料可透過 subscription 或 outbox event 關聯取得，snapshot 仍能避免 subscription 後續修改或刪除造成歷史派送紀錄失真。

## Management Surface

商戶後台第一版提供 webhook subscription 管理能力：

- 查看 subscription 清單，包含 callback URL、訂閱事件數量與建立/更新時間。
- 建立新的 subscription，輸入 callback URL 並選擇事件類型。
- 查看單一 subscription，顯示 callback URL 與事件 checkbox。
- 修改 subscription 的 callback URL 與訂閱事件。
- 軟刪除 subscription。

後台不管理 webhook event type。事件選項由系統 seed 的 event type 目錄提供。

## Persistence Concepts

資料表命名採單數。概念層資料表如下：

| 資料表 | 用途 |
| ------ | ---- |
| `webhook_subscription` | 記錄商戶登記的 webhook endpoint，以及該 endpoint 是否可用於接收事件推播。 |
| `webhook_event_type` | 記錄系統支援的 webhook 事件類型，供 UI checkbox 顯示與後端事件校驗使用。 |
| `webhook_subscription_event_type` | 記錄某個 webhook subscription 訂閱了哪些事件類型。 |
| `webhook_outbox_event` | 記錄業務服務已產生、等待 webhook dispatcher 處理的一筆交易事件。 |
| `webhook_delivery` | 記錄某一筆 outbox event 對某一個 subscription endpoint 的實際推播任務與結果。 |

Persistence 設計原則：

- 資料表自身主鍵與關聯鍵使用系統現行主鍵策略。
- `merchant_id` 維持既有 UUID 字串設計。
- 時間欄位使用具時區語意的 timestamp。
- 軟刪除 subscription 後，不應阻擋商戶用相同 endpoint 重新建立新的 subscription。
- Subscription endpoint 在同一 merchant 的有效資料內需保持唯一。
- Delivery 需能在不依賴 subscription 現況的情況下還原派送當下資訊。

具體欄位型別、索引、FK 與 migration 細節留給 blueprint 或 plan。

## Dispatch Flow

Webhook dispatch 流程如下：

1. Withdrawal 或 deposit 服務在交易狀態變更時產生 webhook event。
2. 業務服務寫入 webhook outbox event，不直接呼叫商戶 endpoint。
3. Dispatcher 輪詢 pending outbox event。
4. Dispatcher 根據 merchant 與 event type 找出 active subscription。
5. 若沒有 subscription，outbox event 標示為沒有訂閱者。
6. 若有 subscription，系統為每個 subscription 建立 delivery。
7. Delivery worker 鎖定 pending delivery，產生簽章並 POST 到 endpoint。
8. 商戶 endpoint 回傳成功狀態時，delivery 標記成功。
9. 商戶 endpoint 回傳失敗、timeout 或其他錯誤時，delivery 標記失敗。
10. Recovery scheduler 補償 pending 或 timeout 的 delivery，避免任務永久卡住。

## Constraints

- Webhook 派送不得阻塞 withdrawal / deposit 的主交易流程。
- Dispatcher 只能針對 active 且未刪除的 subscription 建立 delivery。
- Delivery worker 必須以鎖定機制避免同一筆 delivery 被多個 worker 重複處理。
- 商戶 endpoint 的修改不應改寫歷史 delivery 的實際派送目標。
- Event type 目錄應由系統控制，避免商戶建立不存在或未支援的事件類型。
- Signing secret 不開放 UI 編輯，避免商戶誤操作導致既有接收端驗簽失敗。

## Rejected Alternatives

- 單表同時保存事件與派送結果。
  - 不採用。單一事件可能要派送到多個 endpoint，若不拆 delivery，無法獨立追蹤每個 endpoint 的成功、失敗與重試狀態。

- Delivery 只透過 subscription join 取得 endpoint。
  - 不採用。Subscription 可能在 delivery 後被修改或刪除，delivery 需要保存派送當下的 endpoint snapshot。

- 在交易服務內同步呼叫商戶 endpoint。
  - 不採用。外部 endpoint 的 timeout 或失敗不應影響內部交易狀態更新。

- 開放 webhook event type UI CRUD。
  - 不採用。事件類型代表系統正式支援的 contract，第一版由 migration seed 控制。

## Open Points

- `deposit.blocked` 是否正式納入第一版支援事件。
- Webhook payload 的 external contract。
- Signature 演算法、header 命名與驗簽文件。
- Delivery 失敗後的重試策略。
- Outbox event 何時標記為完成處理。
- 是否需要提供商戶查詢 delivery history。
- 是否需要提供管理端人工重送 delivery。

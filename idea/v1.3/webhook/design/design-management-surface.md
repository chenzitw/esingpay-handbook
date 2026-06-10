---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Management Surface Design

## Merchant Console

商戶後台第一版提供 webhook subscription 管理能力：

- 查看 subscription 清單，包含 callback URL、訂閱事件數量與建立/更新時間。
- 建立新的 subscription，輸入 callback URL 並選擇事件類型。
- 查看單一 subscription，顯示 callback URL 與事件 checkbox。
- 修改 subscription 的 callback URL 與訂閱事件。
- 軟刪除 subscription。

後台不管理 webhook event type。事件選項由後端 TypeScript code-defined event type catalog 提供。

## Expected States

Subscription list：

- Loading：首次載入與刪除後刷新時需呈現載入狀態。
- Empty：商戶尚未建立 subscription 時，清單顯示空狀態並提供建立入口。
- Error：查詢失敗時顯示可理解錯誤，並保留重試入口。
- Pagination：若資料超過一頁，依 API 回傳 paging metadata 顯示分頁控制。

Create / edit form：

- Event checkbox options 由 `GET /webhook/merch/event-types` 載入，不在前端 hard-code。
- Event options 依 `sortOrder` 顯示，顯示文字第一版直接使用 `eventKey`。
- Submit 時送出 trim 後的 `endpointUrl` 與完整 `eventKeys` 集合。
- 允許不勾選任何事件；送出空 `eventKeys` 陣列時，subscription endpoint 保留但不訂閱任何事件。
- Validation error 需能顯示 endpoint URL 錯誤、event selection 錯誤與 merchant scope / not found 錯誤。

Delete：

- Delete 前需有 confirmation。
- Delete 成功後刷新 list。
- Delete 失敗時保留原清單狀態並顯示錯誤。

## Constraints

- 商戶只能看到與管理自己名下的 webhook subscription。
- 被軟刪除的 subscription 不出現在 UI 清單。
- Event type 目錄應由系統控制，避免商戶建立不存在或未支援的事件類型。

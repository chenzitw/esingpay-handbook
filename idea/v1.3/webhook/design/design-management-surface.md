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

後台不管理 webhook event type。事件選項由系統 seed 的 event type 目錄提供。

## Constraints

- 商戶只能看到與管理自己名下的 webhook subscription。
- 被軟刪除的 subscription 不出現在 UI 清單。
- Event type 目錄應由系統控制，避免商戶建立不存在或未支援的事件類型。

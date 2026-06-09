---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Dispatch Flow Design

## Dispatch Flow

Webhook dispatch 流程如下：

1. Withdrawal 或 deposit 服務在交易狀態變更時產生 webhook event。
2. 業務服務寫入 webhook outbox event，不直接呼叫商戶 endpoint。
3. Dispatcher 輪詢 pending outbox event。
4. Dispatcher 根據 merchant 與 event type 找出未刪除且已訂閱該事件的 subscription。
5. 若沒有 subscription，outbox event 標示為沒有訂閱者。
6. 若有 subscription，系統為每個 subscription 建立 delivery。
7. Delivery worker 鎖定 pending delivery，POST 到 endpoint。
8. 商戶 endpoint 回傳成功狀態時，delivery 標記成功。
9. 商戶 endpoint 回傳失敗、timeout 或其他錯誤時，delivery 標記失敗。
10. Recovery scheduler 補償 pending 或 timeout 的 delivery，避免任務永久卡住。

## Constraints

- Webhook 派送不得阻塞 withdrawal / deposit 的主交易流程。
- Dispatcher 只能針對未刪除且已訂閱目標事件的 subscription 建立 delivery。
- Delivery worker 必須以鎖定機制避免同一筆 delivery 被多個 worker 重複處理。
- 商戶 endpoint 的修改不應改寫歷史 delivery 的實際派送目標。

## Rejected Alternatives

- 在交易服務內同步呼叫商戶 endpoint。
  - 不採用。外部 endpoint 的 timeout 或失敗不應影響內部交易狀態更新。

---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Dispatch Flow Design

## Dispatch Flow

Webhook dispatch 流程如下：

1. Fund / transaction domain service 在 withdrawal 或 deposit 狀態變更時發布 domain event notification queue message。
2. Webhook inbound event consumer 訂閱 domain event notification queue，驗證 event key 並建立 webhook outbox event。
3. Dispatcher 輪詢 pending outbox event。
4. Dispatcher 根據 merchant 與 event type 找出未刪除且已訂閱該事件的 subscription。
5. 若沒有 subscription，不建立 delivery，outbox event 標示為 `DISPATCHED`。
6. 若有 subscription，系統為每個 subscription 建立 delivery，並發布 delivery execution queue message。
7. Delivery worker 鎖定 pending delivery，POST 到 endpoint。
8. 商戶 endpoint 回傳成功狀態時，delivery 標記成功。
9. 商戶 endpoint 回傳失敗、timeout 或其他錯誤時，delivery 標記失敗。
10. Recovery scheduler 補償 pending 或 timeout 的 delivery，避免任務永久卡住。

## Constraints

- Webhook 派送不得阻塞 withdrawal / deposit 的主交易流程。
- Fund / transaction domain service 不直接寫入 webhook outbox event，也不直接呼叫商戶 endpoint。
- Dispatcher 只能針對未刪除且已訂閱目標事件的 subscription 建立 delivery。
- Delivery worker 必須以鎖定機制避免同一筆 delivery 被多個 worker 重複處理。
- 商戶 endpoint 的修改不應改寫歷史 delivery 的實際派送目標。
- Outbox event 的 `status` 只表示 dispatcher 是否已處理；沒有訂閱者不是獨立 outbox status。
- Queue topic 與 payload envelope 的討論基礎見 [`design-queue-topology.md`](./design-queue-topology.md)。

## Rejected Alternatives

- 在交易服務內同步呼叫商戶 endpoint。
  - 不採用。外部 endpoint 的 timeout 或失敗不應影響內部交易狀態更新。

- 由 fund / transaction domain service 直接寫入 webhook outbox table。
  - 不採用。Webhook service 應 own outbox persistence；交易服務只發布 domain event notification。

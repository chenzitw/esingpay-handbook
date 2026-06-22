---
status: draft
updated_at: 2026-06-22
updated_by: Codex
---

# Webhook 交易事件推播 — Dispatch Flow Design

## Dispatch Flow

Webhook dispatch 流程如下：

1. Fund service 在 withdrawal 或 deposit 狀態變更時發布 domain event notification queue message。
2. Webhook inbound event consumer 驗證 `source`、event type、resource 與 payload。
3. Consumer 開始 DB transaction，根據 merchant 與 event type 找出未刪除且已訂閱該事件的 subscriptions。
4. 若沒有 subscription，不建立 persistence record，commit 後完成 queue message。
5. 若有 subscriptions，consumer 在該 transaction 內為每個 subscription 建立 pending delivery；每筆 delivery 保存來源、資源、endpoint 與 payload snapshot。
6. Transaction commit 後才完成 inbound queue message；重送時由 delivery 複合唯一性避免重複建立。
7. Delivery publisher 掃描 pending deliveries，發布 delivery execution queue messages；publish failure 由後續輪詢補償。
8. Delivery worker 鎖定 pending delivery，POST 到 endpoint。
9. 商戶 endpoint 回傳成功狀態時，delivery 標記成功。
10. 商戶 endpoint 回傳失敗、timeout 或其他錯誤時，delivery 標記失敗。
11. Recovery scheduler 補償 pending 或 timeout 的 delivery，避免任務永久卡住。

## Constraints

- Webhook 派送不得阻塞 withdrawal / deposit 的主交易流程。
- Fund service 不直接寫入 webhook persistence，也不直接呼叫商戶 endpoint。
- Webhook 第一版不保存 outbox 或 inbox event；consumer 只能針對未刪除且已訂閱目標事件的 subscription 建立 delivery。
- Delivery 必須保存 `source`、`event_type`、`resource_type` 與 `resource_identifier`。
- 第一版以 `source + event_type + resource_type + resource_identifier + subscription` 去重，並接受同一資源的同一 event type 只能發生一次的限制。
- 沒有訂閱者的事件不留 receipt；若 queue complete 失敗後 subscription 發生變動，redelivery 會依當下 subscription 重新 matching。
- Delivery worker 必須以鎖定機制避免同一筆 delivery 被多個 worker 重複處理。
- 商戶 endpoint 的修改不應改寫歷史 delivery 的實際派送目標。
- Queue topic 與 payload envelope 的討論基礎見 [`design-queue-topology.md`](./design-queue-topology.md)。

## Rejected Alternatives

- 在交易服務內同步呼叫商戶 endpoint。
  - 不採用。外部 endpoint 的 timeout 或失敗不應影響內部交易狀態更新。

- Webhook service 為 inbound Fund event 建立自己的 outbox 或 inbox table。
  - 第一版不採用。時程優先下直接建立 delivery；缺少 inbound audit、replay 與 no-subscriber receipt 是已接受限制。

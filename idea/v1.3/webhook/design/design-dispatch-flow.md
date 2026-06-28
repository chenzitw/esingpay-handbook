---
status: draft
updated_at: 2026-06-29
updated_by: Codex
---

# Webhook 交易事件推播 — Dispatch Flow Design

## Dispatch Flow

Webhook dispatch 流程如下：

1. Fund service 在 withdrawal 或 deposit 狀態變更時發布 Azure Service Bus domain event，topic 命名 follow `fund.<resource>.status-changed`。
2. Webhook inbound event consumer 驗證 `source`、event key、resource 與 payload。
3. Consumer 開始 DB transaction，根據 merchant 與 event key 找出未刪除且已訂閱該事件的 subscriptions。
4. 若沒有 subscription，不建立 persistence record，commit 後完成 inbound event message。
5. 若有 subscriptions，consumer 在該 transaction 內為每個 subscription 建立 pending delivery；每筆 delivery 保存來源、資源、endpoint 與 payload snapshot。
6. Transaction commit 後，consumer 立即嘗試為新建 delivery 發布 BullMQ delivery execution job，job payload 只需包含 `deliveryId`。
7. 若 delivery job publish 成功，consumer 完成 inbound event message；若 publish 失敗，delivery 保持 `PENDING`，consumer 仍可完成 inbound event message，後續由 recovery cron 補發 delivery job。
8. Inbound message redelivery 時，由 delivery 複合唯一性避免重複建立；若同一 delivery job 被重複發布，worker lock 負責防止重複 POST。
9. Delivery worker 消費 BullMQ delivery execution queue，鎖定 pending delivery，簽章並以 10 秒 request timeout POST 到 endpoint。
10. 商戶 endpoint 回傳成功狀態時，delivery 標記成功。
11. 商戶 endpoint 回傳失敗、10 秒 timeout 或其他 transport error 時，delivery 標記 terminal `FAILED`；第一版不 retry。
12. Recovery scheduler 補償 stale pending delivery 的 job publish failure；timeout `DELIVERING` delivery 以 guarded transition 標記為 terminal `FAILED`，避免任務永久卡住。

## Constraints

- Webhook 派送不得阻塞 withdrawal / deposit 的主交易流程。
- Fund service 不直接寫入 webhook persistence，也不直接呼叫商戶 endpoint。
- Webhook 第一版不保存 outbox 或 inbox event；consumer 只能針對未刪除且已訂閱目標事件的 subscription 建立 delivery。
- Delivery 必須保存 `source`、`eventKey`、`resourceType` 與 `resourceIdentifier`。
- 第一版以 `source + eventKey + resourceType + resourceIdentifier + subscription` 去重，並接受同一資源的同一 event key 只能發生一次的限制。
- 沒有訂閱者的事件不留 receipt；若 inbound event complete 失敗後 subscription 發生變動，redelivery 會依當下 subscription 重新 matching。
- Delivery job publish 不與 delivery DB commit 共用 transaction；publish failure 必須由 stale `PENDING` recovery 補償。
- Delivery worker 必須以鎖定機制避免同一筆 delivery 被多個 worker 重複處理。
- 第一版 delivery execution 不做 retry/backoff；HTTP non-2xx、10 秒 timeout 或 transport error 直接標記 terminal `FAILED`。
- 商戶 endpoint 的修改不應改寫歷史 delivery 的實際派送目標。
- Azure SB inbound event contract 見 [`design-inbound-event-contract.md`](./design-inbound-event-contract.md)；BullMQ job payload 與 execution queue 見 [`design-queue-topology.md`](./design-queue-topology.md)。

## Rejected Alternatives

- 在交易服務內同步呼叫商戶 endpoint。
  - 不採用。外部 endpoint 的 timeout 或失敗不應影響內部交易狀態更新。

- Webhook service 為 inbound Fund event 建立自己的 outbox 或 inbox table。
  - 第一版不採用。時程優先下直接建立 delivery；缺少 inbound audit、replay 與 no-subscriber receipt 是已接受限制。

- 以 cron polling publisher 作為正常 delivery job 發布主路徑。
  - 不採用。正常路徑在 delivery commit 後立即 publish job；cron 只補償 publish failure 或卡住的 delivery。

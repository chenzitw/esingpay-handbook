---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Design

## Context

Webhook 讓商戶可以在後台登記 callback endpoint，接收系統主動推送的交易事件。第一版聚焦 withdrawal / deposit 相關事件，讓商戶不需要輪詢查詢 API，也能在交易狀態變更時更新自己的系統。

此設計同時覆蓋兩個面向：

- 商戶後台對 webhook endpoint 的觀看、登記、修改與刪除。
- 系統內部把交易事件轉換為對外 webhook delivery 的派送流程。

Webhook 派送不應綁在交易主流程內同步呼叫商戶 endpoint。交易服務只負責向 domain event notification queue 發布交易狀態變更事件，後續由 webhook inbound event consumer、dispatcher、delivery worker 與 recovery scheduler 非同步處理。

## Scope

In scope：

- 商戶可管理自己名下的 webhook subscription。
- 商戶可為每個 webhook endpoint 選擇要接收的事件類型。
- 系統可根據交易 domain event notification 建立 webhook outbox event。
- 系統可根據 outbox event 與 subscription 建立 delivery 任務。
- 系統可追蹤每個 endpoint 的派送結果。
- 系統可補償 pending 或 timeout 的 delivery 任務。

Out of scope：

- 事件 payload 的完整欄位 contract。
- delivery 重試次數、退避策略與最終失敗規則。
- 商戶手動重送 delivery 的後台功能。
- webhook endpoint 驗證流程。
- webhook event type 的 UI CRUD。

## Design Map

- [`design-domain-model.md`](./design-domain-model.md)：webhook subscription、event type、outbox event、delivery 的概念模型與 persistence principles。
- [`design-persistence-model.md`](./design-persistence-model.md)：webhook 第一版資料表、欄位、index 與 stage relationship 的 conceptual schema shape。
- [`design-service-boundary.md`](./design-service-boundary.md)：webhook capability ownership、service boundary、anti-patterns。
- [`design-management-surface.md`](./design-management-surface.md)：商戶後台第一版 webhook subscription 管理能力。
- [`design-rest.md`](./design-rest.md)：merchant console 前端與後端交握的 REST request / response 草案。
- [`design-rpc.md`](./design-rpc.md)：api-gateway REST RPC proxy 與 webhook service internal capability 的 RPC surface。
- [`design-error-contract.md`](./design-error-contract.md)：subscription management 的錯誤語意、穩定 error code 與 REST/RPC mapping 邊界。
- [`design-dispatch-flow.md`](./design-dispatch-flow.md)：交易事件轉 outbox、dispatcher 建立 delivery、worker 派送與 recovery 的流程約束。
- [`design-queue-topology.md`](./design-queue-topology.md)：domain event notification queue 與 webhook delivery execution queue 的 topic / payload 討論基礎。
- [`design-type-contract.md`](./design-type-contract.md)：後續 domain raw、event contract、request / response DTO 型別定義的設計入口。
- [`design-payload-contract.md`](./design-payload-contract.md)：POST 到商戶 endpoint 的 webhook payload envelope 與 event-specific data shape。

## Open Points

- Webhook payload 的 external contract。
- Delivery 失敗後的重試策略。
- Outbox event 何時標記為完成處理。
- Fund / transaction domain event notification topic 的實際拆分與命名。
- REST/RPC error envelope 的最終格式。
- 是否需要提供商戶查詢 delivery history。
- 是否需要提供管理端人工重送 delivery。

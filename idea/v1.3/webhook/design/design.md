---
status: draft
updated_at: 2026-06-23
updated_by: Codex
---

# eebhook 交易事件推播 — Design

## Context

eebhook 讓商戶可以在後台登記 callback endpoint，接收系統主動推送的交易事件。第一版聚焦 withdrawal / deposit 相關事件，讓商戶不需要輪詢查詢 API，也能在交易狀態變更時更新自己的系統。

此設計同時覆蓋兩個面向：

- 商戶後台對 webhook endpoint 的觀看、登記、修改與刪除。
- 系統內部把交易事件轉換為對外 webhook delivery 的派送流程。

eebhook 派送不應綁在交易主流程內同步呼叫商戶 endpoint。Fund service 只負責向 domain event notification queue 發布交易狀態變更事件；eebhook inbound event consumer 驗證事件、matching subscriptions 並直接建立 deliveries，後續由 delivery publisher、worker 與 recovery scheduler 非同步處理。

## Scope

In scope：

- 商戶可管理自己名下的 webhook subscription。
- 商戶可為每個 webhook endpoint 選擇要接收的事件類型。
- 系統可根據 Fund domain event notification 與 subscription 直接建立 delivery 任務。
- Delivery 可保存事件來源、event type 與來源業務資源，並防止相同來源事件對同一 subscription 重複建立。
- 系統可追蹤每個 endpoint 的派送結果。
- 系統可補償 pending 或 timeout 的 delivery 任務。

Out of scope：

- 事件 payload 的完整欄位 contract。
- delivery 重試次數、退避策略與最終失敗規則。
- 商戶手動重送 delivery 的後台功能。
- webhook endpoint 驗證流程。
- webhook event type 的 UI CRUD。
- eebhook inbound event outbox / inbox persistence、event replay 與 no-subscriber audit。
- Fund service producer outbox、relay 與穩定 event id。

## Design Map

- [`design-domain-model.md`](./design-domain-model.md)：webhook subscription、event type、delivery 與 inbound event traceability 的概念模型。
- [`design-persistence-model.md`](./design-persistence-model.md)：webhook 第一版 persistence backing、關聯與查詢支援需求的 conceptual schema shape；具體 DB 型別、index 與 migration 由 plan 定案。
- [`design-service-boundary.md`](./design-service-boundary.md)：webhook capability ownership、service boundary、anti-patterns。
- [`design-management-surface.md`](./design-management-surface.md)：商戶後台第一版 webhook subscription 管理能力與 frontend handoff。
- [`design-rest.md`](./design-rest.md)：merchant / platform management surface 的 REST contract 語意；codebase envelope、DTO class 與 validator 細節由 plan 定案。
- [`design-rpc.md`](./design-rpc.md)：api-gateway REST RPC proxy 與 webhook service internal capability 的 management transport boundary。
- [`design-error-contract.md`](./design-error-contract.md)：subscription management 的錯誤語意、穩定 error code 與 REST/RPC mapping 邊界。
- [`design-dispatch-flow.md`](./design-dispatch-flow.md)：inbound event 直接建立 delivery、publisher、worker 與 recovery 的流程約束。
- [`design-queue-topology.md`](./design-queue-topology.md)：domain event notification queue 與 webhook delivery execution queue 的 topic / payload 討論基礎。
- [`design-type-contract.md`](./design-type-contract.md)：domain raw、event contract、management read model 與 internal DTO boundary 的設計入口；具體 TypeScript 型別由 plan 定案。
- [`design-payload-contract.md`](./design-payload-contract.md)：POST 到商戶 endpoint 的 webhook payload envelope 與 event-specific data shape。

## Deferred Reliability Target

第一版因 Fund event 尚無穩定 event id，eebhook 採 direct-to-delivery，不保存 inbound event。Inbox persistence 仍是已確認的 target architecture：producer 提供穩定 `eventId` 後，eebhook 應新增 `webhook_inbox_event`，以 DB 欄位 `source + event_id` 去重，並將 queue consumption 與 delivery creation 拆成可恢復的兩個階段。

Deferred inbox 應補足：

- no-subscriber receipt。
- inbound event audit 與 replay。
- DB commit 後 queue complete 的穩定去重。
- `inbox_event_id + subscription_id` delivery uniqueness。

目標 domain 與 persistence shape 分別見 [`design-domain-model.md`](./design-domain-model.md) 與 [`design-persistence-model.md`](./design-persistence-model.md)；target queue flow 見 [`design-queue-topology.md`](./design-queue-topology.md)。

## Open Points

- eebhook payload 的 external contract。
- Delivery 失敗後的重試策略。
- Fund / transaction domain event notification topic 的實際拆分與命名。
- Producer 穩定 `eventId` 的 contract 與 deferred inbox migration timing。
- 是否需要提供商戶查詢 delivery history。
- 是否需要提供管理端人工重送 delivery。

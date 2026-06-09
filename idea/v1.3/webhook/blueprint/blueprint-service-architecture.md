---
status: draft
updated_at: 2026-06-09
updated_by: Codex
---

# Webhook 交易事件推播 — Service Architecture Blueprint

## Purpose

本文件鎖定 webhook 第一版的 domain 命名、module 切分與 ownership。具體 class 名稱、目錄路徑與 provider wiring 留給 plan 依 codebase guide 決定。

## Ownership

Webhook capability owns：

- Webhook subscription lifecycle。
- Webhook event type catalog。
- Webhook outbox event persistence。
- Webhook delivery persistence and state transitions。
- Dispatcher / worker / recovery orchestration。
- Signing secret usage for outbound delivery。

Withdrawal / deposit capability owns：

- 自身交易狀態。
- 何時產生可對外通知的 domain event。
- 對 webhook producer capability 提供必要 event payload。

商戶後台 owns：

- UI route、form、table、checkbox display。
- 呼叫 webhook management API。

## Domain Vocabulary

- `webhook_subscription`：商戶登記的一個 callback endpoint 與訂閱事件集合。
- `webhook_event_type`：系統支援的 webhook event catalog item。
- `webhook_outbox_event`：內部交易事件轉換後、等待 webhook dispatcher 處理的 outbox record。
- `webhook_delivery`：某 outbox event 對某 subscription endpoint 的派送任務與結果。
- `dispatcher`：處理 outbox event，建立 delivery。
- `delivery worker`：執行 delivery HTTP POST。
- `recovery scheduler`：補償 pending 或 timeout delivery。

## Suggested Service Boundaries

概念上可切成以下 capability：

- Subscription management：create / list / get / update / delete subscription。
- Event type catalog：seeded catalog query and validation。
- Event production：withdrawal / deposit 狀態變更時建立 outbox event。
- Dispatching：pending outbox event matching subscriptions and creating deliveries。
- Delivery execution：locking delivery, signing payload, POST endpoint, updating result。
- Recovery：finding stuck delivery and requeueing.

Plan 可依 codebase 現有 module convention 決定是否拆成多個 module。重點是 capability responsibility 不混用。

## Boundary Rules

- Trading domain 不直接呼叫 merchant endpoint。
- Dispatcher 不重新組 event payload；它使用 outbox event payload snapshot。
- Delivery worker 不重新查 subscription 的 current endpoint 作為發送目標；它使用 delivery endpoint snapshot。
- Subscription management 不處理 delivery execution。
- Event type catalog 是系統設定資料，不由商戶 UI CRUD。
- Signing secret 可由 subscription management 建立，但 delivery worker 使用它簽章；若 secret storage 需解密，plan 需定義讀取邊界。

## Anti-Patterns

不得採用：

```text
withdrawal / deposit service
  -> HTTP POST merchant endpoint
```

不得採用：

```text
delivery worker
  -> current subscription endpoint
  -> send
```

不得採用：

```text
merchant UI
  -> hard-code event type options
```

不得採用：

```text
dispatcher
  -> create delivery without subscription-event relation check
```

## Phase Ownership

- Phase 1 owns subscription management and event type catalog.
- Phase 2 owns event production and outbox persistence.
- Phase 3 owns dispatcher and delivery creation.
- Phase 4 owns delivery worker, signing and recovery.

## Open Points

- Webhook 是否在 codebase 中成為獨立 top-level domain/module，或先依現有 merchant/notification boundary 掛載。
- Signing secret storage/read boundary 與 rotation 是否需要獨立 service。
- Payload builder 是否屬 event production capability，或拆成獨立 mapper/builder capability。

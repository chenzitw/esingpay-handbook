---
status: draft
updated_at: 2026-06-23
updated_by: Codex
---

# Webhook 交易事件推播 — Service Boundary Design

## Purpose

本文件鎖定 webhook 第一版的 domain 命名、module 切分與 ownership。具體 class 名稱、目錄路徑與 provider wiring 留給 plan 依 codebase guide 決定。

## Ownership

Webhook capability owns：

- Webhook subscription lifecycle。
- Code-defined webhook event type catalog。
- Domain event queue consumption、validation 與 subscription matching。
- Webhook delivery persistence and state transitions。
- Inbound consumer / delivery publisher / worker / recovery orchestration。
- Signing secret usage for outbound delivery。

Fund / transaction capability owns：

- 自身交易狀態。
- 何時產生可對外通知的 domain event。
- 向 domain event notification queue 發布必要 event payload。
- 未來若導入 producer outbox，其 persistence、relay 與 event id 由 producer service 擁有。

商戶後台 owns：

- UI route、form、table、checkbox display。
- 呼叫 webhook management API。

## Domain Vocabulary

- `webhook_subscription`：商戶登記的一個 callback endpoint 本體。
- `webhook_subscription_event_type`：subscription 訂閱 event key 的 binding。
- `webhook event type catalog`：系統支援的 webhook event key 清單，由 TypeScript 檔案定義，不建 DB table。
- `domain event notification`：上游服務發布、供 webhook 消費的業務事件訊息。
- `webhook_delivery`：某 inbound event 對某 subscription endpoint 的派送任務與結果，並保存來源與資源追蹤資訊。
- `inbound event consumer`：驗證 domain event、matching subscriptions 並直接建立 delivery。
- `delivery publisher`：在 delivery commit 後發布 execution job；recovery cron 僅補發 stale pending delivery job。
- `delivery worker`：執行 delivery HTTP POST。
- `recovery scheduler`：補償 stale pending publish failure 或 timeout delivery。

## Suggested Service Boundaries

概念上可切成以下 capability：

- Subscription management：create / list / get / update / delete subscription。
- Event type catalog：code-defined catalog query and validation。
- Inbound event consumption：驗證 queue event、mapping payload、matching subscriptions and creating deliveries。
- Delivery publishing：publishing execution jobs after delivery commit and republishing stale pending deliveries for recovery。
- Delivery execution：locking delivery, signing payload, POST endpoint, updating result。
- Recovery：finding stuck delivery and requeueing.

Plan 可依 codebase 現有 module convention 決定是否拆成多個 module。重點是 capability responsibility 不混用。

## Boundary Rules

- Trading domain 不直接呼叫 merchant endpoint。
- Webhook service 不擁有 producer outbox。第一版不為 inbound event 建立 inbox record；deferred inbox 導入後，inbox persistence 與 dispatch state 由 Webhook service 擁有。
- Inbound event consumer 建立 delivery 時即保存 payload snapshot；post-commit publisher 與 worker 不重新查交易現況組 payload。
- Delivery worker 不重新查 subscription 的 current endpoint 作為發送目標；它使用 delivery endpoint snapshot。
- Subscription management 不處理 delivery execution。
- Event type catalog 是系統設定資料，由 TypeScript 檔案定義，不由商戶 UI CRUD，也不建 DB table。
- 第一版 signing secret 由環境變數提供統一預設值，不屬於 subscription management；delivery worker 使用該 secret 簽章。

## Anti-Patterns

不得採用：

```text
webhook service
  -> persist webhook-owned outbox event for an inbound fund event
```

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
inbound event consumer
  -> create delivery without subscription-event relation check
```

## Stage Relationship

- Stage 1 owns subscription management and event type catalog.
- Stage 2 owns inbound event consumption, subscription matching and delivery creation.
- Stage 3 owns post-commit delivery job publishing and stale pending publish recovery.
- Stage 4 owns delivery worker, signing and recovery.

## Open Points

- Webhook 是否在 codebase 中成為獨立 top-level domain/module，或先依現有 merchant/notification boundary 掛載。
- Signing secret storage/read boundary 與 rotation 是否需要獨立 service。
- Payload builder 是否內嵌於 inbound event consumer，或拆成獨立 mapper/builder capability。

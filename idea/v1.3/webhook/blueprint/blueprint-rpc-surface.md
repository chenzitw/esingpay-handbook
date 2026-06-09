---
status: draft
updated_at: 2026-06-09
updated_by: Codex
---

# Webhook 交易事件推播 — RPC Surface Blueprint

## Purpose

本文件鎖定 webhook 第一版 internal service-to-service capability。它不定義 public merchant-facing API；商戶後台 API 見 [`blueprint-api-surface.md`](./blueprint-api-surface.md)。

具體 contract-rpc 檔案位置、method name、input/output DTO 型別與 error mapping 留給 plan 依 codebase convention 決定。

## Capability Boundary

Webhook 是交易事件推播能力的 owner。Withdrawal / deposit 服務不應直接建立 delivery 或呼叫商戶 endpoint；它們只需要向 webhook capability 表達「某交易事件已發生」。

概念邊界：

```text
withdrawal / deposit service
  -> webhook event production capability
  -> webhook_outbox_event
```

Dispatcher / worker / recovery 是 webhook 服務內部能力：

```text
webhook dispatcher
  -> subscription matching
  -> delivery creation
  -> publish delivery job

webhook delivery worker
  -> delivery lock
  -> signing
  -> HTTP POST
  -> delivery status update
```

## Conceptual RPC Capabilities

### Produce Webhook Event

用途：讓交易 domain 在狀態變更時產生 webhook outbox event。

概念 input：

- `merchant_id`
- `event_key` 或 event type reference
- `correlation_type`
- `correlation_identifier`
- `payload`

概念 output：

- outbox event id
- accepted / ignored result

決策：

- 交易主流程只需確認 outbox event 已寫入或被明確忽略，不等待 delivery。
- 若 event type 不存在或未啟用，錯誤語意由 plan 決定；但不應退化成同步派送。

### Match Subscriptions For Event

用途：dispatcher 依 outbox event 找出 active 且未刪除、且訂閱該 event type 的 subscriptions。

概念 input：

- `merchant_id`
- `event_id`

概念 output：

- matching subscription list

這個 capability 可是 service method，不一定需要 contract-rpc。若 dispatcher 與 subscription management 未來會拆成不同服務，plan 可將它提升成 RPC。

### Create Delivery

用途：為 outbox event 與 subscription 建立 delivery。

概念 input：

- outbox event reference
- subscription reference
- endpoint snapshot
- payload snapshot

概念 output：

- delivery id

### Execute Delivery

用途：worker 鎖定 delivery 並執行 HTTP POST。

概念 input：

- delivery id

概念 output：

- success / failed result
- delivery status after execution

這個 capability 通常由 queue consumer 觸發，不一定暴露為 RPC。

### Recover Stuck Deliveries

用途：recovery scheduler 找出 pending 或 timeout 的 delivery 並重新投入 delivery queue。

概念 input：

- timeout cutoff
- batch size

概念 output：

- requeued count

## Route Ownership

- 商戶後台 REST routes 屬 management surface。
- 交易服務產生 outbox event 屬 internal webhook production capability。
- Dispatcher、worker、recovery 屬 webhook service internal orchestration。
- 若需要 gateway 或其他服務呼叫 webhook internal capability，應走 contract-rpc，而不是直接 import webhook module internals。

## Phase Ownership

- Phase 1 不需要 producer RPC；主要是 subscription management。
- Phase 2 決定並落地 produce webhook event capability。
- Phase 3 需要 dispatcher 使用 subscription matching 與 delivery creation capability。
- Phase 4 需要 delivery execution 與 recovery capability。

## Open Points

- Produce webhook event 使用 true contract-rpc，或在同一 codebase 內先採 service/facade boundary。
- Event production failure 是否影響交易狀態變更 commit。
- Dispatcher / worker 是否需要獨立 service identity 或 job metadata。

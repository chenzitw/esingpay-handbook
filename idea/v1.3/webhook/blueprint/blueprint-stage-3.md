---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Stage 3 Blueprint

## Goal

Stage 3 完成 dispatcher 與 delivery creation。Dispatcher 從 pending outbox event 找出 matching subscriptions，建立 webhook delivery，並發布 delivery execution job。

## Scope

In scope：

- Pending outbox event polling / dispatch trigger。
- Eligible subscription matching。
- `NO_SUBSCRIBERS` outbox status。
- Delivery creation。
- Delivery job publishing。
- Outbox event dispatch status transition。

Out of scope：

- 實際 HTTP POST endpoint。
- Signing secret 使用。
- Delivery timeout recovery。
- Retry/backoff policy。

## Inputs

- Data model：[`blueprint-data-model.md`](./blueprint-data-model.md)。
- RPC surface：[`blueprint-rpc-surface.md`](./blueprint-rpc-surface.md)。
- Infra queue：[`blueprint-infra-queue.md`](./blueprint-infra-queue.md)。

## Dispatch Flow

```text
dispatcher
  -> fetch pending webhook_outbox_event batch
  -> query active, non-deleted subscription by merchant_id + event_id
  -> if none:
       mark outbox event NO_SUBSCRIBERS
     else:
       create webhook_delivery per subscription
       publish webhook.delivery.execute job per delivery
       mark outbox event DISPATCHED
```

## Critical Decisions

- Dispatcher 查詢 subscription 時必須排除已軟刪除資料，並確認 subscription 已訂閱目標事件。
- Matching 必須透過 `webhook_subscription_event_type` relation，不可只用 merchant scope。
- Delivery 需保存 endpoint 與 payload snapshot。
- Queue publish failure 的補償方式需在 plan 明確處理。

## Validation Target

Stage 3 完成時應能證明：

```text
pending outbox event
  -> no matching subscription
  -> outbox event NO_SUBSCRIBERS
```

以及：

```text
pending outbox event
  -> matching subscriptions
  -> webhook_delivery rows created
  -> delivery execution jobs published
  -> outbox event DISPATCHED
```

驗證重點：

- 多 subscription 訂閱同一 event 時建立多筆 delivery。
- 已刪除或未訂閱目標事件的 subscription 不建立 delivery。
- Dispatcher 可重跑而不造成重複 delivery，或 plan 明確定義去重策略。

## Estimate

| Item | 估時 |
| --- | ---: |
| Dispatcher polling / scheduling | 1 天 |
| Subscription matching and delivery creation | 1.5 天 |
| Queue publishing and dispatch status handling | 1.5 天 |

總估時：4 天。

## Open Points

- Outbox event 轉為 `DISPATCHED` 的精確時機。
- Delivery 去重策略。
- Dispatcher batch size 與 lock strategy。

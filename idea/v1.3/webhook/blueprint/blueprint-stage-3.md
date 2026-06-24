---
status: draft
updated_at: 2026-06-23
updated_by: Codex
---

# Webhook 交易事件推播 — Stage 3 Blueprint

## Goal

Stage 3 完成 delivery job publisher capability、inbound consumer post-commit publish integration 與 stale pending publish recovery。Stage 2 已建立的 deliveries 在 DB commit 後可立即投入 Webhook service-private execution queue；若 publish 暫時失敗，delivery 保持 `PENDING` 並由 recovery cron 補發，不讓 delivery 永久遺失。

## Scope

In scope：

- Delivery job publisher capability。
- Post-commit delivery job publish integration in the inbound consumer flow。
- `webhook.delivery.execute` private job publishing。
- Delivery job payload and deduplication direction。
- Stale `PENDING` delivery publish recovery。
- Recovery scheduling / triggering direction。

Out of scope：

- Fund publisher implementation。
- Subscription matching and delivery snapshot creation semantics。
- HTTP POST、signing、execution timeout recovery。
- Retry/backoff product policy。
- Deferred inbox / outbox persistence。

## Inputs

- Persistence model：[`../design/design-persistence-model.md`](../design/design-persistence-model.md)。
- Queue topology：[`../design/design-queue-topology.md`](../design/design-queue-topology.md)。
- Service boundary：[`../design/design-service-boundary.md`](../design/design-service-boundary.md)。
- Stage 2 pending delivery persistence and snapshot semantics。

## Publishing Flow

```text
webhook inbound event consumer
  -> create webhook_delivery rows in DB transaction
  -> commit DB transaction
  -> call delivery job publisher with created delivery ids
  -> publish webhook.delivery.execute with delivery id
  -> publish success: complete inbound message
  -> publish failure: keep delivery PENDING and complete inbound message

recovery cron
  -> find stale PENDING deliveries whose jobs may not have been published
  -> republish webhook.delivery.execute with delivery id
  -> leave duplicate job safety to Stage 4 worker lock
```

Stage 3 changes the normal path from polling-first publication to immediate post-commit publication. The cron path is recovery only; it should not be the normal source of delivery latency.

## Critical Decisions

- Delivery execution queue 是 Webhook service-private job mechanism，不是 Fund / Webhook 跨 domain event contract。
- Job payload 只需穩定 delivery identifier；worker 從 persistence 讀取 endpoint 與 payload snapshot。
- Queue publish 與 DB state 無法視為單一 transaction；publish failure 必須可由 stale `PENDING` recovery 補償。
- Inbound message may be completed after delivery DB commit even if delivery job publish fails, because the delivery record is recoverable。
- Duplicate execution jobs are allowed only when Stage 4 worker state locking makes them safe。
- Publisher 不重新 matching subscriptions，也不修改 delivery payload / endpoint snapshot。
- Stage 3 owns post-commit delivery job publication and stale pending publish recovery；Stage 4 owns execution and stuck `DELIVERING` recovery。
- Queued marker、deterministic job id 或 pure pending recovery 的實際組合由 plan 依現有 queue pattern 與 concurrency requirements 定案。

## Plan Inventory

- Delivery job publisher：發布 delivery execution job，input 為 delivery ids。
- Inbound consumer post-commit integration：DB commit 後呼叫 publisher，publish failure 不 rollback delivery creation。
- Stale pending recovery query：找出可能尚未成功發布 job 的 `PENDING` deliveries。
- Recovery scheduler：週期性補發 stale pending delivery jobs。
- Targeted verification：驗證 immediate publish、publish failure recovery、duplicate job safety boundary。

## Validation Direction

Stage 3 完成時應能證明：

```text
newly committed PENDING delivery
  -> post-commit publisher emits webhook.delivery.execute
  -> job contains delivery identifier
  -> inbound message can be completed
```

以及：

```text
post-commit queue publish failure
  -> delivery remains PENDING
  -> inbound message can be completed
  -> recovery cron later publishes the job
```

驗證重點：

- Publisher 不建立第二筆 delivery。
- Publisher 不重新查 subscription 或來源交易資料。
- 多次 publish 或 recovery cycle 不會造成不可控的 concurrent execution。
- Stage 3 不執行 HTTP POST；worker lock and final delivery result belong to Stage 4。

## Estimate

| Item | 估時 |
| --- | ---: |
| Delivery job publisher capability | 0.75 天 |
| Post-commit publish integration | 0.5 天 |
| Stale pending recovery and targeted verification | 0.75 天 |

總估時：2 天。

## Pattern Gaps

- Fund BullMQ dispatcher / worker 與 scheduler / bootstrap backfill 可作為方向性 baseline，但 post-commit DB-to-queue publication 與 stale pending recovery 仍是新組合，plan 必須明確決定 queued marker、deterministic job id 或 pure pending recovery strategy。

## Open Points

- 是否需要獨立 queued/published marker，或以 deterministic job id + stale pending query 支援補償。
- Stale pending threshold、recovery cadence 與 batch size。
- Delivery identifier 在 private job payload 中的 primitive representation。

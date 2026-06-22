---
status: draft
updated_at: 2026-06-22
updated_by: Codex
---

# Webhook 交易事件推播 — Stage 3 Blueprint

## Goal

Stage 3 完成 pending delivery job publishing 與 publish recovery。Stage 2 已建立的 deliveries 可被投入 Webhook service-private execution queue，且暫時 publish failure 不會讓 delivery 永久遺失。

## Scope

In scope：

- Pending delivery lookup and claim direction。
- `webhook.delivery.execute` private job publishing。
- Delivery job deduplication direction。
- Queue publish failure recovery。
- Publisher scheduling / triggering direction。

Out of scope：

- Inbound domain event consumption。
- Subscription matching and delivery creation。
- Fund publisher implementation。
- HTTP POST、signing、execution timeout recovery。
- Retry/backoff product policy。

## Inputs

- Persistence model：[`../design/design-persistence-model.md`](../design/design-persistence-model.md)。
- Queue topology：[`../design/design-queue-topology.md`](../design/design-queue-topology.md)。
- Service boundary：[`../design/design-service-boundary.md`](../design/design-service-boundary.md)。
- Stage 2 pending delivery persistence and snapshot semantics。

## Publishing Flow

```text
delivery publisher
  -> find eligible PENDING deliveries
  -> claim or identify publishable batch
  -> publish webhook.delivery.execute with delivery id
  -> record publication outcome if required by the selected recovery model

later publisher cycle
  -> find deliveries whose jobs were not published successfully
  -> publish again
```

## Critical Decisions

- Delivery execution queue 是 Webhook service-private job mechanism，不是 Fund / Webhook 跨 domain event contract。
- Job payload 只需穩定 delivery identifier；worker 從 persistence 讀取 endpoint 與 payload snapshot。
- Queue publish 與 DB state 無法視為單一 transaction；publish failure 必須可由後續 publisher cycle 補償。
- Duplicate execution jobs are allowed only when Stage 4 worker state locking makes them safe。
- Publisher 不重新 matching subscriptions，也不修改 delivery payload / endpoint snapshot。
- Stage 3 owns unpublished / publish-failed pending delivery recovery；Stage 4 owns execution and stuck execution recovery。
- Publication claim、queued marker 或 deterministic job id 的實際組合由 plan 依現有 BullMQ pattern與 concurrency requirements 定案。

## Plan Inventory

- Delivery publication query：找出可發布或需補發的 pending deliveries。
- Private queue dispatcher：發布 delivery execution job。
- Publisher trigger：以 scheduler、bootstrap backfill 或等效既有 pattern 觸發 publication cycles。
- Targeted verification：驗證 successful publish、publish failure recovery、duplicate job safety boundary。

## Validation Direction

Stage 3 完成時應能證明：

```text
PENDING delivery
  -> publisher emits webhook.delivery.execute
  -> job contains delivery identifier
```

以及：

```text
queue publish failure
  -> delivery remains recoverable
  -> later publisher cycle can publish the job
```

驗證重點：

- Publisher 不建立第二筆 delivery。
- Publisher 不重新查 subscription 或來源交易資料。
- 多個 publisher cycle 不會造成不可控的 concurrent execution。

## Estimate

| Item | 估時 |
| --- | ---: |
| Pending delivery publication capability | 1 天 |
| Publish recovery and targeted verification | 1 天 |

總估時：2 天。

## Pattern Gaps

- Fund BullMQ dispatcher / worker 與 scheduler / bootstrap backfill 可作為方向性 baseline，但 delivery publication 的 DB-to-queue recovery 仍是新組合，plan 必須明確決定 publication state 與 concurrent publisher claim strategy。

## Open Points

- 是否需要獨立 queued/published marker，或以 deterministic job id + pending query 支援補償。
- Publisher batch size、cadence 與 concurrency claim strategy。
- Delivery identifier 在 private job payload 中的 primitive representation。

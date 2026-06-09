---
status: draft
updated_at: 2026-06-09
updated_by: Codex
---

# Webhook 交易事件推播 — Infra Queue Blueprint

## Purpose

本文件鎖定 webhook 第一版的 queue / scheduler / worker topology。Queue provider 採 Azure Service Bus；具體 queue/topic name、consumer 設定、資源命名與 retry/backoff implementation 留給 plan 依 codebase infra convention 決定。

## Reference Baseline

- [`../references/webhook-dispatch-flow.md`](../references/webhook-dispatch-flow.md) preserves an early Mermaid baseline for webhook dispatcher / worker / recovery flow. Current blueprint decisions override naming and schema details in that reference.

## Topology

Webhook 第一版需要兩類非同步處理：

- Dispatcher：從 `webhook_outbox_event` 找出 pending event，建立 delivery。
- Delivery worker：消費 delivery job，實際 POST 到 merchant endpoint。

Recovery scheduler 補償卡住的 delivery：

```text
recovery scheduler
  -> query PENDING or timeout DELIVERING delivery
  -> requeue delivery job
```

## Conceptual Topics

第一版 queue provider 採 Azure Service Bus。Plan 仍需依 codebase Azure Service Bus convention 決定正式 queue/topic 名稱、consumer 設定與部署資源命名。概念 topic 如下：

| Conceptual topic | Producer | Consumer | Purpose |
| --- | --- | --- | --- |
| `webhook.outbox.dispatch` | scheduler or dispatcher trigger | dispatcher | 觸發 pending outbox event dispatch。 |
| `webhook.delivery.execute` | dispatcher / recovery scheduler | delivery worker | 執行某一筆 webhook delivery。 |

如果 codebase 採 polling scheduler 而非 queue 觸發 dispatcher，`webhook.outbox.dispatch` 可不建立；但 delivery execution 應維持 Azure Service Bus queue-based，避免 dispatcher 同步呼叫外部 endpoint。

## Dispatcher Flow

```text
dispatcher
  -> fetch pending webhook_outbox_event batch
  -> for each outbox event:
       query active, non-deleted subscriptions by merchant_id + event_id
       if none:
         mark outbox event NO_SUBSCRIBERS
       else:
         create webhook_delivery per subscription
         publish webhook.delivery.execute job per delivery
         mark outbox event DISPATCHED
```

Open point：`DISPATCHED` 的精確時機需要 plan 決定。可選方向：

- Delivery 建立完成並成功 publish jobs 後標記。
- Delivery 建立完成即標記，由 recovery 補償未 publish 的 job。

## Delivery Worker Flow

```text
delivery worker
  -> receive delivery id
  -> lock PENDING delivery as DELIVERING
  -> load payload snapshot and endpoint_url snapshot
  -> load signing secret
  -> sign payload
  -> POST endpoint_url
  -> mark SUCCESS on 2xx
  -> mark FAILED on non-2xx / timeout / transport error
```

Worker 必須透過狀態轉換鎖定 delivery，避免多 worker 重複處理同一任務。

## Recovery Flow

```text
recovery scheduler
  -> find PENDING delivery older than threshold
  -> find DELIVERING delivery stuck beyond timeout
  -> publish webhook.delivery.execute jobs
```

第一版 recovery 只保證卡住任務可重新投入。重試次數、backoff 與人工重送不在第一版 scope，除非 Stage 4 plan 決定需要最低限度欄位支援。

## Failure Boundaries

- Merchant endpoint timeout 或 5xx 不影響交易主流程。
- Delivery worker failure 不應遺失 delivery record。
- Dispatcher failure 不應造成 outbox event 永久無法處理；pending outbox event 必須可被下一輪 dispatcher 拾取。
- Queue publish failure 的補償方式需在 Stage 3 plan 明確處理。

## Stage Ownership

- Stage 2 可先不建立 delivery queue，但 outbox event production 必須可供 dispatcher 後續處理。
- Stage 3 建立 dispatcher 與 delivery job publishing。
- Stage 4 建立 delivery worker 與 recovery scheduler。

## Pattern Gaps

- 若 codebase 尚無 scheduler pattern，Stage 3 plan 需先 survey。
- 若 codebase 尚無 Azure Service Bus queue/topic naming convention，Stage 3 plan 需定義命名並記錄 rationale。
- 若 codebase 尚無 worker lock pattern，Stage 4 plan 需定義 delivery 狀態轉換與 atomic update 方式。

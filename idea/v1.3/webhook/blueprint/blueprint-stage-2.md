---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Stage 2 Blueprint

## Goal

Stage 2 完成 withdrawal / deposit 狀態變更到 webhook outbox event 的 production flow。交易主流程只寫入 outbox event，不直接呼叫商戶 endpoint。

## Scope

In scope：

- `webhook_outbox_event` persistence。
- Produce webhook event capability。
- Withdrawal / deposit 狀態變更點接入。
- Event key 到 code-defined event type catalog 的解析。
- Payload snapshot 寫入 outbox event。

Out of scope：

- Dispatcher matching subscription。
- Delivery creation。
- HTTP POST 到 merchant endpoint。
- Retry / recovery。
- Payload external contract 的完整版本策略。

## Inputs

- Data model：[`../design/design-persistence-model.md`](../design/design-persistence-model.md)。
- Management transport / capability boundary：[`../design/design-rpc.md`](../design/design-rpc.md)。
- Service boundary：[`../design/design-service-boundary.md`](../design/design-service-boundary.md)。
- Payload contract：[`../design/design-payload-contract.md`](../design/design-payload-contract.md)。

## Event Production

Withdrawal / deposit 服務在交易狀態變更時建立 webhook outbox event：

```text
withdrawal / deposit status transition
  -> produce webhook event
  -> validate event_key against code-defined event type catalog
  -> persist webhook_outbox_event PENDING
```

交易主流程不等待 dispatcher 或 worker。

## Critical Decisions

- Outbox event 使用 `event_type` 保存 event key 字串。
- Outbox event 保存 `merchant_id`、`resource_type`、`resource_identifier` 與 `payload`。
- Event production failure 是否影響交易狀態變更 commit 需在 plan 明確決定。
- Outbox payload 必須使用 [`../design/design-payload-contract.md`](../design/design-payload-contract.md) 定義的固定 envelope，`data` 依 event type 填入 withdrawal / deposit 內容。
- Stage 2 plan 需查驗 withdrawal / deposit 欄位來源，決定 optional 欄位是否可提供。

## Validation Target

Stage 2 完成時應能證明：

```text
withdrawal / deposit status transition
  -> webhook_outbox_event created with PENDING status
  -> event_type stores a code-defined event key
  -> payload snapshot is persisted
```

驗證重點：

- 交易狀態變更不直接呼叫 merchant endpoint。
- 不支援或未啟用 event key 的處理語意明確。
- Outbox event 可被 Stage 3 dispatcher 查詢。

## Estimate

| Item | 估時 |
| --- | ---: |
| Outbox data model and service | 1 天 |
| Withdrawal/deposit event hook integration | 1 天 |
| Payload builder and validation | 1 天 |

總估時：3 天。

## Open Points

- Withdrawal / deposit payload optional 欄位的實際來源。
- Event production 與交易 transaction 的一致性策略。
- Withdrawal / deposit 各狀態對應哪些 webhook event keys。

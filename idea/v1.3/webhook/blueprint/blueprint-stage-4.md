---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Stage 4 Blueprint

## Goal

Stage 4 完成 delivery worker、signing 與 recovery scheduler。完成後 webhook 系統能實際 POST 到 merchant endpoint，追蹤成功/失敗，並補償卡住的 delivery。

## Scope

In scope：

- Delivery worker queue consumer。
- Delivery lock from `PENDING` to `DELIVERING`。
- Signing payload。
- HTTP POST endpoint。
- Delivery success / failure status update。
- Pending / timeout delivery recovery scheduler。

Out of scope：

- 人工重送 delivery。
- Delivery history UI。
- Retry/backoff 完整產品化策略。
- Signing secret rotation UI/API。
- Endpoint ownership verification flow。

## Inputs

- Data model：[`../design/design-persistence-model.md`](../design/design-persistence-model.md)。
- Service boundary：[`../design/design-service-boundary.md`](../design/design-service-boundary.md)。
- Queue topology：[`../design/design-queue-topology.md`](../design/design-queue-topology.md)。
- Payload contract：[`../design/design-payload-contract.md`](../design/design-payload-contract.md)。

## Worker Flow

```text
delivery worker
  -> receive delivery id
  -> lock PENDING delivery as DELIVERING
  -> load endpoint_url snapshot
  -> load payload snapshot
  -> load signing secret
  -> sign payload
  -> POST endpoint_url
  -> mark SUCCESS on 2xx
  -> mark FAILED on non-2xx / timeout / transport error
```

## Recovery Flow

```text
recovery scheduler
  -> find old PENDING delivery
  -> find timeout DELIVERING delivery
  -> publish delivery execution job
```

## Critical Decisions

- Worker 必須透過狀態轉換鎖定 delivery，避免多 worker 重複處理同一任務。
- HTTP 2xx 視為 success；非 2xx、timeout 或 transport error 視為 failed。
- 第一版若未導入 retry count，recovery 應只補償卡住任務，不做無限重試語意。
- Signing secret 不能只存不可逆 hash，因 worker 需要用它產生簽章。
- Worker POST 的 JSON body 必須使用 delivery payload snapshot，且符合 [`../design/design-payload-contract.md`](../design/design-payload-contract.md)。
- 簽章 input 應以實際送出的 JSON body 為準，避免簽章 payload 與 POST body 不一致。

## Validation Target

Stage 4 完成時應能證明：

```text
delivery job
  -> delivery worker locks delivery
  -> POST endpoint
  -> 2xx response
  -> delivery SUCCESS
```

以及：

```text
delivery job
  -> POST endpoint timeout or non-2xx
  -> delivery FAILED
```

以及：

```text
stuck PENDING or DELIVERING delivery
  -> recovery scheduler
  -> delivery execution job requeued
```

## Estimate

| Item | 估時 |
| --- | ---: |
| Worker lock and execution flow | 1.5 天 |
| HTTP POST and signing | 1.5 天 |
| Failure mapping and status updates | 1 天 |
| Recovery scheduler | 1 天 |

總估時：5 天。

## Open Points

- Delivery 失敗後是否立刻 `FAILED`，或需要 `RETRYING` / retry count 等額外狀態。
- Delivery timeout 門檻與 recovery scheduler cadence。
- Webhook signature 演算法、header 命名與驗簽文件。

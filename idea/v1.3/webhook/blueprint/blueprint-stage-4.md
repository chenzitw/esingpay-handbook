---
status: draft
updated_at: 2026-06-29
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
- HTTP POST endpoint with 10 second request timeout。
- Delivery success / failure status update。
- Stuck execution recovery scheduler。

Out of scope：

- 人工重送 delivery。
- Delivery history UI。
- Retry/backoff；第一版 `FAILED` 是 terminal status。
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
  -> serialize JSON body once
  -> sign raw JSON body with HMAC-SHA256
  -> POST endpoint_url with 10 second timeout
  -> mark SUCCESS on 2xx
  -> mark FAILED on non-2xx / timeout / transport error
```

Request signing headers：

```text
X-ESingPay-Timestamp: <unix timestamp seconds>
X-ESingPay-Signature: sha256=<lowercase hex HMAC-SHA256 digest>
X-ESingPay-Delivery-Id: <delivery id>
X-ESingPay-Event-Key: <event key>
```

Signature input：

```text
<timestamp> + "." + <raw JSON request body bytes interpreted as UTF-8>
```

## Recovery Flow

```text
recovery scheduler
  -> find timeout DELIVERING delivery
  -> atomically mark it FAILED with guarded state transition
  -> do not requeue execution job
```

## Critical Decisions

- Worker 必須透過狀態轉換鎖定 delivery，避免多 worker 重複處理同一任務。
- Stage 3 owns post-commit delivery job publication and stale pending publish recovery；Stage 4 不建立第二套 pending publisher。
- HTTP request timeout 第一版固定為 10 秒。
- HTTP 2xx 視為 success；非 2xx、10 秒 timeout 或 transport error 視為 failed。
- 第一版不做 retry/backoff，不新增 `RETRYING`、retry count、next retry time 或 attempt 欄位；`FAILED` 是 terminal status。
- Recovery 必須以 guarded state transition 將 timeout `DELIVERING` 標記為 `FAILED`；不得重新發 execution job 形成 retry 語意。
- Signing secret 不能只存不可逆 hash，因 worker 需要用它產生簽章。
- 第一版簽章演算法為 HMAC-SHA256，簽章值以 `sha256=<lowercase hex digest>` 放入 `X-ESingPay-Signature`。
- 簽章 input 為 `<timestamp> + "." + <raw JSON request body bytes interpreted as UTF-8>`。
- Worker 需帶 `X-ESingPay-Timestamp`、`X-ESingPay-Signature`、`X-ESingPay-Delivery-Id` 與 `X-ESingPay-Event-Key` headers。
- 驗簽文件應要求 merchant 使用 constant-time compare，並建議拒絕 timestamp 與接收時間相差超過 5 分鐘的 request。
- Worker POST 的 JSON body 必須使用 delivery payload snapshot，且符合 [`../design/design-payload-contract.md`](../design/design-payload-contract.md)。
- 簽章 input 應以實際送出的 JSON body bytes 為準；worker 必須序列化一次，並以同一份 raw JSON bytes 完成簽章與 HTTP POST，避免簽章 payload 與 POST body 不一致。

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
  -> POST endpoint 10 second timeout, non-2xx or transport error
  -> delivery FAILED
```

以及：

```text
stuck DELIVERING delivery
  -> recovery scheduler
  -> guarded transition to FAILED
  -> no execution job requeue
```

## Estimate

| Item | 估時 |
| --- | ---: |
| Worker lock and execution flow | 1.5 天 |
| HTTP POST and signing | 1.5 天 |
| Failure mapping and status updates | 1 天 |
| Stuck execution recovery scheduler | 1 天 |

總估時：5 天。

## Open Points

- Stuck `DELIVERING` recovery scheduler cadence。

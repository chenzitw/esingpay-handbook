---
status: draft
updated_at: 2026-06-23
updated_by: Codex
---

# Webhook 交易事件推播 — Payload Contract Design

## Purpose

本文件鎖定 webhook 第一版送到商戶登記 URL 的 POST payload 格式。所有 webhook delivery 預設使用 HTTP `POST`，外層 envelope 固定，事件內容放在 event-specific `data` 內。

這是第一版 conceptual contract。最終 DTO 型別、序列化細節、簽章 header 與版本化文件留給 plan 或後續 docs。

## HTTP Method

- Webhook delivery 一律使用 HTTP `POST`。
- Request body 使用 JSON。
- `Content-Type` 使用 `application/json`。
- 第一版不支援商戶自訂 method、header template 或 query parameter template。

## Envelope Shape

所有 webhook payload 使用相同 envelope：

```json
{
  "id": "webhookDeliveryId",
  "eventKey": "withdrawal.completed",
  "occurredAt": "2026-06-09T12:34:56.000Z",
  "deliveredAt": "2026-06-09T12:35:01.000Z",
  "merchantId": "merchantUuid",
  "apiVersion": "2026-06-01",
  "data": {}
}
```

欄位語意：

| Field | Meaning |
| --- | --- |
| `id` | 本次 delivery 的識別值。商戶可用於 log correlation。 |
| `eventKey` | 穩定事件 key，例如 `withdrawal.completed`。 |
| `occurredAt` | 上游交易事件發生時間。 |
| `deliveredAt` | worker 發送本次 HTTP request 的時間。 |
| `merchantId` | 事件所屬 merchant。 |
| `apiVersion` | webhook payload contract version。 |
| `data` | 依 event 類型不同的事件內容。 |

## Data Shape Principle

`data` 應提供商戶處理事件所需的最小穩定欄位，不暴露內部流程細節。

第一版原則：

- 包含交易主體 id。
- 包含交易狀態。
- 包含金額與幣別/資產資訊。
- 包含商戶側可對帳的 reference 或 correlation 欄位，若該交易類型已有此資料。
- 不包含 actor details、完整 status history、wallet allocation、fee payer strategy、internal network endpoint lifecycle。

## Withdrawal Event Data

適用事件：

- `withdrawal.created`
- `withdrawal.cancelled`
- `withdrawal.failed`
- `withdrawal.completed`

Conceptual payload：

```json
{
  "id": "webhookDeliveryId",
  "eventKey": "withdrawal.completed",
  "occurredAt": "2026-06-09T12:34:56.000Z",
  "deliveredAt": "2026-06-09T12:35:01.000Z",
  "merchantId": "merchantUuid",
  "apiVersion": "2026-06-01",
  "data": {
    "withdrawalId": "12345",
    "status": "completed",
    "amount": "100.00000000",
    "asset": "USDT",
    "network": "TRON",
    "toAddress": "Txxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "merchantReference": "merchant-side-reference",
    "createdAt": "2026-06-09T12:30:00.000Z",
    "updatedAt": "2026-06-09T12:34:56.000Z"
  }
}
```

Notes：

- `withdrawalId` 的實際命名需與 external/public API 對外命名一致。
- `merchantReference` 若 withdrawal flow 沒有對應欄位，可省略。
- Failure reason 是否放入第一版 payload 由 Stage 2 plan 依既有交易模型決定；若加入，應使用商戶可理解的穩定 code，不暴露 internal error。

## Deposit Event Data

適用事件：

- `deposit.created`
- `deposit.failed`
- `deposit.completed`
- `deposit.blocked`

Conceptual payload：

```json
{
  "id": "webhookDeliveryId",
  "eventKey": "deposit.completed",
  "occurredAt": "2026-06-09T12:34:56.000Z",
  "deliveredAt": "2026-06-09T12:35:01.000Z",
  "merchantId": "merchantUuid",
  "apiVersion": "2026-06-01",
  "data": {
    "depositId": "67890",
    "status": "completed",
    "amount": "250.00000000",
    "asset": "USDT",
    "network": "TRON",
    "fromAddress": "Tyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy",
    "toAddress": "Txxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "transactionHash": "0x...",
    "merchantReference": "merchant-side-reference",
    "createdAt": "2026-06-09T12:30:00.000Z",
    "updatedAt": "2026-06-09T12:34:56.000Z"
  }
}
```

Notes：

- `transactionHash` 只在鏈上資料已存在時提供。
- `merchantReference` 若 deposit flow 沒有對應欄位，可省略。
- `deposit.blocked` 第一版正式支援；blocked reason 是否提供需另行決定，第一版可先只提供 status。

## Versioning

- 第一版 payload 使用 `apiVersion` 固定字串。
- `apiVersion` 表示 payload contract version，不代表 endpoint route version。
- 新增 optional field 可在同一版本內演進。
- 移除欄位、改名、改變語意或改變型別應視為 breaking change，需新版本策略。

## Snapshot Rules

- Inbound event consumer 在建立 delivery 時保存實際送出的 payload snapshot，表示事件消費當下的內容。
- Delivery payload snapshot 不受 subscription 或來源交易後續變更影響。
- Worker 發送與簽章時使用 delivery payload snapshot。

## Signing Relationship

簽章演算法與 header 名稱不在本文件定案，但簽章 input 應以實際送出的 JSON body 為準。

Plan 需避免以下情況：

- 簽章時使用一份 payload，實際 POST 時又重新序列化成不同內容。
- Worker 發送前重新查交易現況組 payload，造成 delivery payload 與 inbound event 發生時間不一致。

## Stage Relationship

- Stage 2：inbound consumer 使用 payload builder 產生符合本 envelope 的 payload，並直接保存為 delivery snapshot。
- Stage 3：delivery publisher 在 delivery commit 後發布 execution job；publish recovery 補發 stale pending delivery，不重建 payload。
- Stage 4：worker 使用 delivery payload snapshot 簽章並 POST。

## Open Points

- `apiVersion` 第一版字串是否採日期格式或 semantic version。
- Withdrawal / deposit 對外 id 是否使用 bigint 字串、UUID 或 external id。
- Failure / blocked reason 是否納入第一版 payload。
- `merchantReference` 在 withdrawal / deposit 中的實際來源欄位。
- 是否需要在 envelope 中加入 `attempt` 或 `deliveryAttempt`；第一版若不做 retry count 可先不加。
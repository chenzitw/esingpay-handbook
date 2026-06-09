---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — REST API Design

## Purpose

本文定義 merchant console 前端與後端交握時需要的 webhook management REST request / response 草案。

REST DTO 是前端交握 contract，不等同於 domain object，也不等同於 DB schema。Domain 語意來源見 [`design-domain-model.md`](./design-domain-model.md)；具體 controller、validation decorator、error mapping 與 route prefix 仍留給 plan 依 codebase pattern 定案。

## Naming

- JSON 欄位使用 camelCase，對齊前端 TypeScript 與 domain design 命名。
- DB 欄位若使用 snake_case，由後端 mapping layer 轉換，不直接外露給前端。
- Subscription ID 與 event type ID 在 response 中以 `id` 表示；request 建立或更新訂閱事件時使用穩定的 `eventKey`。

## Endpoint Inventory

| Method | Path | 用途 |
| --- | --- | --- |
| `GET` | `/webhook/merch/subscriptions` | 查詢目前商戶的 webhook subscription 清單。 |
| `POST` | `/webhook/merch/subscriptions` | 建立 webhook subscription。 |
| `GET` | `/webhook/merch/subscriptions/{subscriptionId}` | 查詢單一 webhook subscription detail。 |
| `PUT` | `/webhook/merch/subscriptions/{subscriptionId}` | 覆蓋 callback URL 與訂閱事件集合。 |
| `DELETE` | `/webhook/merch/subscriptions/{subscriptionId}` | 軟刪除 webhook subscription。 |
| `GET` | `/webhook/merch/event-types` | 查詢可訂閱的 webhook event type options。 |

## Shared DTOs

### WebhookEventTypeOptionDto

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `id` | `string` | yes | Event type 識別碼。 |
| `eventKey` | `string` | yes | 穩定事件 key，例如 `deposit.completed`。 |
| `sortOrder` | `number` | yes | 前端顯示排序。 |

範例：

```json
{
  "id": "8",
  "eventKey": "deposit.blocked",
  "sortOrder": 80
}
```

### WebhookSubscriptionSummaryDto

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `id` | `string` | yes | Subscription 識別碼。 |
| `endpointUrl` | `string` | yes | Callback URL。 |
| `eventTypeCount` | `number` | yes | 已訂閱事件數量。 |
| `createdAt` | `IsoDateTimeUtc` | yes | 建立時間。 |
| `updatedAt` | `IsoDateTimeUtc` | yes | 最後更新時間。 |

### WebhookSubscriptionDetailDto

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `id` | `string` | yes | Subscription 識別碼。 |
| `endpointUrl` | `string` | yes | Callback URL。 |
| `eventTypes` | `WebhookEventTypeOptionDto[]` | yes | 已訂閱事件清單。 |
| `createdAt` | `IsoDateTimeUtc` | yes | 建立時間。 |
| `updatedAt` | `IsoDateTimeUtc` | yes | 最後更新時間。 |

## GET `/webhook/merch/event-types`

查詢系統支援且可供商戶訂閱的 event type options。前端建立與編輯 subscription 時都應使用此 endpoint 取得 checkbox options，不應 hard-code event list。

Response：

```json
{
  "items": [
    {
      "id": "1",
      "eventKey": "withdrawal.created",
      "sortOrder": 10
    },
    {
      "id": "8",
      "eventKey": "deposit.blocked",
      "sortOrder": 80
    }
  ]
}
```

第一版支援事件：

- `withdrawal.created`
- `withdrawal.canceled`
- `withdrawal.failed`
- `withdrawal.completed`
- `deposit.created`
- `deposit.failed`
- `deposit.completed`
- `deposit.blocked`

## GET `/webhook/merch/subscriptions`

查詢目前商戶名下未刪除的 webhook subscriptions。

Query：

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `page` | `number` | no | 頁碼；是否從 1 開始依 codebase paging convention。 |
| `pageSize` | `number` | no | 每頁筆數。 |

Response：

```json
{
  "items": [
    {
      "id": "101",
      "endpointUrl": "https://merchant.example.com/webhooks/esingpay",
      "eventTypeCount": 4,
      "createdAt": "2026-06-10T02:00:00.000Z",
      "updatedAt": "2026-06-10T02:10:00.000Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

## POST `/webhook/merch/subscriptions`

建立新的 webhook subscription。

Request：

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `endpointUrl` | `string` | yes | 商戶接收 webhook POST 的 callback URL。 |
| `eventKeys` | `string[]` | yes | 要訂閱的 event type keys。 |

範例：

```json
{
  "endpointUrl": "https://merchant.example.com/webhooks/esingpay",
  "eventKeys": [
    "withdrawal.completed",
    "deposit.completed",
    "deposit.blocked"
  ]
}
```

Response：`WebhookSubscriptionDetailDto`

```json
{
  "id": "101",
  "endpointUrl": "https://merchant.example.com/webhooks/esingpay",
  "eventTypes": [
    {
      "id": "4",
      "eventKey": "withdrawal.completed",
      "sortOrder": 40
    },
    {
      "id": "7",
      "eventKey": "deposit.completed",
      "sortOrder": 70
    },
    {
      "id": "8",
      "eventKey": "deposit.blocked",
      "sortOrder": 80
    }
  ],
  "createdAt": "2026-06-10T02:00:00.000Z",
  "updatedAt": "2026-06-10T02:00:00.000Z"
}
```

Validation：

- `endpointUrl` 必須是可接受的 HTTP/HTTPS URL；是否允許 HTTP 由 plan 依環境與安全規則決定。
- `eventKeys` 不可為空。
- `eventKeys` 內每個 key 都必須存在於 webhook event type catalog。
- 同一 merchant 的未刪除 subscription endpoint 不可重複。

## GET `/webhook/merch/subscriptions/{subscriptionId}`

查詢單一 webhook subscription detail。

Response：`WebhookSubscriptionDetailDto`

```json
{
  "id": "101",
  "endpointUrl": "https://merchant.example.com/webhooks/esingpay",
  "eventTypes": [
    {
      "id": "4",
      "eventKey": "withdrawal.completed",
      "sortOrder": 40
    }
  ],
  "createdAt": "2026-06-10T02:00:00.000Z",
  "updatedAt": "2026-06-10T02:10:00.000Z"
}
```

## PUT `/webhook/merch/subscriptions/{subscriptionId}`

覆蓋 subscription 的可編輯內容。前端送出目前表單上的 `endpointUrl` 與完整 `eventKeys` 集合，後端以 request 內的事件集合覆蓋目前 subscription 的訂閱關係。

Request：

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `endpointUrl` | `string` | yes | Callback URL。 |
| `eventKeys` | `string[]` | yes | 完整訂閱 event key 集合。 |

範例：

```json
{
  "endpointUrl": "https://merchant.example.com/webhooks/esingpay-v2",
  "eventKeys": [
    "withdrawal.failed",
    "withdrawal.completed",
    "deposit.failed",
    "deposit.completed",
    "deposit.blocked"
  ]
}
```

Response：`WebhookSubscriptionDetailDto`

Validation：

- `endpointUrl` 需套用與 create 相同的 URL 格式與 endpoint 唯一性檢查。
- `eventKeys` 不可為空，且每個 key 都必須存在於 webhook event type catalog。
- 修改只影響後續 delivery，不改寫既有 delivery snapshot。

## DELETE `/webhook/merch/subscriptions/{subscriptionId}`

軟刪除 webhook subscription。刪除後不出現在前端清單，也不再參與後續事件派送。

Response：

```json
{
  "deleted": true
}
```

重複刪除、刪除不存在 subscription、或刪除非目前商戶資料時的 HTTP status 與 error body 依 codebase 既有 API error convention 定案。

## Non-Goals

- 第一版不提供 delivery history 查詢 API。
- 第一版不提供人工重送 delivery API。
- 第一版不提供 event type CRUD API。
- 第一版不提供 endpoint ownership verification API。
- 第一版不提供 secret rotation API。

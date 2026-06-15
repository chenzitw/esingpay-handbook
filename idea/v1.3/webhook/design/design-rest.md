---
status: draft
updated_at: 2026-06-15
updated_by: Codex
---

# Webhook 交易事件推播 — REST API Design

## Purpose

本文定義 merchant console / platform console 與 api-gateway 交握時需要的 webhook management REST contract 語意。

REST contract 是 account surface 的語意邊界，不等同於 domain object、DB schema 或 codebase DTO class。Domain 語意來源見 [`design-domain-model.md`](./design-domain-model.md)；錯誤語意見 [`design-error-contract.md`](./design-error-contract.md)。具體 controller、validation decorator、error mapping、response envelope、paging DTO 與 REST RPC proxy mapping 由 plan 依 codebase pattern 定案。

## Gateway And RPC Boundary

Merchant console / platform console 不直接呼叫 webhook service。第一版管理流程是：

```text
merchant console / platform console
  -> api-gateway REST endpoint
  -> REST RPC proxy
  -> webhook service management capability
```

REST endpoint 是前端可見 contract；webhook service 內部落地對應的 management capability。Stage 1 codebase plan 採 `contract-rest + RestRpc`，不新增 public `contract-rpc/webhook` namespace。Api-gateway 負責既有登入驗證與 REST error envelope mapping，webhook service 依 api-gateway 傳入的 identity 與 target merchant scope 做 ownership 判斷。

第一版 management capability 統一信任 api-gateway 的登入驗證與 merchant/platform identity 傳遞；webhook service 不重複驗證 JWT。非 api-gateway caller 是否可呼叫 management capability 依 codebase 既有 internal access convention 控制。

本文描述 response payload 語意。實際 HTTP response envelope、paging wrapper、error code primitive 與 field-level validation metadata 由 codebase plan 依既有 REST convention 定案。

REST request / response 欄位應盡量與 webhook management capability input / output 對齊，避免 gateway 做不必要的語意轉換。

Account scope rules：

- List 只回傳 `merchant_id = identity.merchantId` 且未軟刪除的 subscription。
- Get / update / delete 必須同時以 `subscriptionId` 與 `merchant_id = identity.merchantId` 查詢。
- 其他 merchant 即使得知 `subscriptionId`，也不能透過 API 讀取、修改或刪除該 subscription。
- Platform account surface 代表平台管理 merchant webhook subscriptions；platform list 可依目標 merchant filters 查詢，platform create 需明確帶入 target merchant，platform view / update / delete 依 active subscription id 操作。

## Naming

- JSON 欄位使用 camelCase，對齊前端 TypeScript 與 domain design 命名。
- DB 欄位若使用 snake_case，由後端 mapping layer 轉換，不直接外露給前端。
- Subscription ID 在 response 中以 `id` 表示；event type option 第一版不提供 DB id，建立或更新訂閱事件時使用穩定的 `eventKey`。
- Subscription `id` 與 `{subscriptionId}` path parameter 使用平台 short id string representation；persistence id 仍為 bigint。Api-gateway 或 webhook management boundary 需先驗證並解析 short id，再查詢 persistence；格式不合法或無法解析時回傳 `WEBHOOK_SUBSCRIPTION_ID_INVALID`。

## Request Validation And Normalization

Validation rules：

- `endpointUrl` 必填，trim 後不可為空。
- `endpointUrl` 必須是有效 HTTPS URL；第一版不接受 `http`。
- 後端保存與回傳的 `endpointUrl` 使用 trim 後的值；不做會改變語意的 URL normalization，例如自動補 slash、改 host casing 以外的重寫。
- `eventKeys` 必填且必須是陣列；允許空陣列。
- `eventKeys` 不可包含重複 key；重複 key 應回傳 validation error，不由後端靜默去重。
- `eventKeys` 每個 key 都必須存在於 backend code-defined event catalog。
- 空 `eventKeys` 代表保留 subscription endpoint，但取消所有事件訂閱。
- Request 中 `eventKeys` 順序沒有業務語意；response 中 `eventTypes` 依 catalog `sortOrder` 排序。
- 同一 merchant 可建立多筆相同 endpoint URL 的 subscription；重複 endpoint 不應被視為 validation error。

Pagination rules：

- `page` 預設為 `1`，採 1-based。
- Page size 需有預設值與最大值；公開 query parameter 名稱與 codebase paging DTO 對齊。
- Paging input 必須是正整數；超過最大值或格式不合法時回傳 pagination validation error。

List sorting：

- Subscription list 預設依 `createdAt` descending 排序。
- 相同 `createdAt` 時以 `id` descending 作為穩定次排序。

## Endpoint Inventory

| Method | Merchant path | Platform path | 用途 |
| --- | --- | --- | --- |
| `GET` | `/webhook/merch/subscriptions` | `/webhook/plat/subscriptions` | 查詢 webhook subscription 清單。 |
| `POST` | `/webhook/merch/subscriptions` | `/webhook/plat/subscriptions` | 建立 webhook subscription。 |
| `GET` | `/webhook/merch/subscriptions/{subscriptionId}` | `/webhook/plat/subscriptions/{subscriptionId}` | 查詢單一 webhook subscription detail。 |
| `PUT` | `/webhook/merch/subscriptions/{subscriptionId}` | `/webhook/plat/subscriptions/{subscriptionId}` | 覆蓋 callback URL 與訂閱事件集合。 |
| `DELETE` | `/webhook/merch/subscriptions/{subscriptionId}` | `/webhook/plat/subscriptions/{subscriptionId}` | 軟刪除 webhook subscription。 |
| `GET` | `/webhook/merch/event-types` | `/webhook/plat/event-types` | 查詢可訂閱的 webhook event type options。 |

下方 endpoint 詳細章節以 merchant path 作為章節標題；同列 platform path 使用相同 request / response 語意、validation 與 error semantics。Platform surface 的 merchant scope 來源不同：list 可帶 merchant filter，create 需帶 target merchant，view / update / delete 以 subscription id 操作 active subscription。

## Permission Semantics

Stage 1 subscription CRUD API 需依 account surface 掛上對應 permission code。Merchant account surface 使用 `/webhook/merch/...`，操作目前登入商戶自己的 webhook subscriptions；platform account surface 使用 `/webhook/plat/...`，操作 merchant webhook subscriptions。

| Operation | Merchant route + permission | Platform route + permission |
| --- | --- | --- |
| List subscriptions | `GET /webhook/merch/subscriptions` + `merch:webhooks:list` | `GET /webhook/plat/subscriptions` + `plat:merch_webhooks:list` |
| View subscription | `GET /webhook/merch/subscriptions/{subscriptionId}` + `merch:webhooks:view` | `GET /webhook/plat/subscriptions/{subscriptionId}` + `plat:merch_webhooks:view` |
| Create subscription | `POST /webhook/merch/subscriptions` + `merch:webhooks:create` | `POST /webhook/plat/subscriptions` + `plat:merch_webhooks:create` |
| Update subscription | `PUT /webhook/merch/subscriptions/{subscriptionId}` + `merch:webhooks:update` | `PUT /webhook/plat/subscriptions/{subscriptionId}` + `plat:merch_webhooks:update` |
| Delete subscription | `DELETE /webhook/merch/subscriptions/{subscriptionId}` + `merch:webhooks:delete` | `DELETE /webhook/plat/subscriptions/{subscriptionId}` + `plat:merch_webhooks:delete` |

Stage 1 subscription update uses dedicated update permission codes: merchant route uses `merch:webhooks:update`; platform route uses `plat:merch_webhooks:update`.

`GET /webhook/merch/event-types` 與 `GET /webhook/plat/event-types` 是 subscription form 的輔助讀取 endpoint，不在 subscription CRUD permission code 清單內。若 codebase route convention 要求所有後台 endpoint 皆掛 permission decorator，plan 應沿用該 account surface 的 list 類 permission。

## Management Read Models

Subscription summary / detail 是 management read model，不是 `WebhookSubscription` domain object 本體。`eventTypeCount` 與 `eventTypes` 都由 subscription-event binding 與 code-defined event catalog 組裝。具體 DTO class 名稱、decorator、serialization 與 response envelope 由 plan 定案。

### Event Type Option

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `eventKey` | `string` | yes | 穩定事件 key，例如 `deposit.completed`。 |
| `displayName` | `string` | yes | 後台 UI 顯示名稱，例如 `Deposit completed`。 |
| `sortOrder` | `number` | yes | 前端顯示排序。 |

範例：

```json
{
  "eventKey": "deposit.blocked",
  "displayName": "Deposit blocked",
  "sortOrder": 80
}
```

### Subscription Summary

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `id` | `string` | yes | Subscription 識別碼。 |
| `endpointUrl` | `string` | yes | Callback URL。 |
| `eventTypeCount` | `number` | yes | 已訂閱事件數量；由 binding 聚合得出。 |
| `createdAt` | `IsoDateTimeUtc` | yes | 建立時間。 |
| `updatedAt` | `IsoDateTimeUtc` | yes | 最後更新時間。 |

### Subscription Detail

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `id` | `string` | yes | Subscription 識別碼。 |
| `endpointUrl` | `string` | yes | Callback URL。 |
| `eventTypes` | event type option list | yes | 已訂閱事件清單；由 binding join code-defined catalog 組裝。 |
| `createdAt` | `IsoDateTimeUtc` | yes | 建立時間。 |
| `updatedAt` | `IsoDateTimeUtc` | yes | 最後更新時間。 |

## GET `/webhook/merch/event-types`

查詢系統支援且可供商戶訂閱的 event type options。後端從 TypeScript code-defined catalog 回傳資料；前端建立與編輯 subscription 時都應使用此 endpoint 取得 checkbox options，不應在前端 hard-code event list。

Response payload contains event type options. Actual REST envelope follows codebase convention.

```json
{
  "items": [
    {
      "eventKey": "withdrawal.created",
      "displayName": "Withdrawal created",
      "sortOrder": 10
    },
    {
      "eventKey": "deposit.blocked",
      "displayName": "Deposit blocked",
      "sortOrder": 80
    }
  ]
}
```

第一版支援事件：

| Event key | Display name | Sort order |
| --- | --- | ---: |
| `withdrawal.created` | Withdrawal created | 10 |
| `withdrawal.cancelled` | Withdrawal cancelled | 20 |
| `withdrawal.failed` | Withdrawal failed | 30 |
| `withdrawal.completed` | Withdrawal completed | 40 |
| `deposit.created` | Deposit created | 50 |
| `deposit.failed` | Deposit failed | 60 |
| `deposit.completed` | Deposit completed | 70 |
| `deposit.blocked` | Deposit blocked | 80 |

## GET `/webhook/merch/subscriptions`

查詢目前商戶名下未刪除的 webhook subscriptions。

Query：

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `page` | `number` | no | 1-based 頁碼；預設 `1`。 |
| page size parameter | `number` | no | 每頁筆數；具體 parameter 名稱、預設值與 alias 由 codebase plan 對齊既有 paging convention。 |

Sorting：

- 預設排序為 `createdAt desc, id desc`。
- 第一版不提供自訂排序參數。

Response payload contains subscription summaries plus paging metadata. Actual list envelope follows codebase plan and existing REST paging convention.

## POST `/webhook/merch/subscriptions`

建立新的 webhook subscription。

Operation semantics：

- 驗證並 trim `endpointUrl`。
- 驗證 `eventKeys` 為陣列、無重複、全部存在於 code-defined event catalog；允許空陣列。
- 建立 webhook subscription。
- 依 request `eventKeys` 建立 subscription-event bindings；若 `eventKeys` 為空陣列，則不建立 bindings。
- Subscription 建立與 binding insert 應在同一 transaction 或等效一致性邊界內完成。
- Response 回傳 subscription detail read model，其中 `eventTypes` 由 binding join code-defined catalog 組裝，並依 `sortOrder` 排序。

Request：

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `endpointUrl` | `string` | yes | 商戶接收 webhook POST 的 callback URL。 |
| `eventKeys` | `string[]` | yes | 要訂閱的 event type keys；可為空陣列，表示暫不訂閱任何事件。 |

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

Response：subscription detail read model

```json
{
  "id": "101",
  "endpointUrl": "https://merchant.example.com/webhooks/esingpay",
  "eventTypes": [
    {
      "eventKey": "withdrawal.completed",
      "displayName": "Withdrawal completed",
      "sortOrder": 40
    },
    {
      "eventKey": "deposit.completed",
      "displayName": "Deposit completed",
      "sortOrder": 70
    },
    {
      "eventKey": "deposit.blocked",
      "displayName": "Deposit blocked",
      "sortOrder": 80
    }
  ],
  "createdAt": "2026-06-10T02:00:00.000Z",
  "updatedAt": "2026-06-10T02:00:00.000Z"
}
```

Validation：

- `endpointUrl` 必須是可接受的 HTTPS URL；第一版不接受 HTTP。
- `eventKeys` 必須是陣列，可為空陣列，且不可重複。
- `eventKeys` 內每個 key 都必須存在於 backend code-defined webhook event type catalog。
- 同一 merchant 可建立多筆相同 endpoint URL 的 subscription；第一版不拒絕重複 endpoint。

## GET `/webhook/merch/subscriptions/{subscriptionId}`

查詢單一 webhook subscription detail。

後端查詢必須同時套用目前 identity 的 merchant id scope；不得只用 `subscriptionId` 查詢。

Response：subscription detail read model

```json
{
  "id": "101",
  "endpointUrl": "https://merchant.example.com/webhooks/esingpay",
  "eventTypes": [
    {
      "eventKey": "withdrawal.completed",
      "displayName": "Withdrawal completed",
      "sortOrder": 40
    }
  ],
  "createdAt": "2026-06-10T02:00:00.000Z",
  "updatedAt": "2026-06-10T02:10:00.000Z"
}
```

## PUT `/webhook/merch/subscriptions/{subscriptionId}`

覆蓋 subscription 的可編輯內容。前端送出目前表單上的 `endpointUrl` 與完整 `eventKeys` 集合，後端以 request 內的事件集合覆蓋目前 subscription 的訂閱關係。

後端更新必須同時套用目前 identity 的 merchant id scope；不得只用 `subscriptionId` 更新。

Operation semantics：

- 更新 subscription endpoint URL。
- 刪除該 subscription 既有 event bindings。
- 依 request `eventKeys` 重新建立 event bindings；若 `eventKeys` 為空陣列，則不建立 bindings，代表取消所有事件訂閱但保留 subscription。
- URL 更新與 binding replacement 應在同一個 transaction 或等效一致性邊界內完成，避免 endpoint 已改但 event binding 未完成，或反向不一致。

Request：

| Field | Type | Required | 說明 |
| --- | --- | --- | --- |
| `endpointUrl` | `string` | yes | Callback URL。 |
| `eventKeys` | `string[]` | yes | 完整訂閱 event key 集合；可為空陣列，表示取消所有事件訂閱。 |

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

Response：subscription detail read model

Validation：

- `endpointUrl` 需套用與 create 相同的 URL 格式檢查。
- `eventKeys` 必須是陣列，可為空陣列，不可重複；每個 key 都必須存在於 backend code-defined webhook event type catalog。
- 修改只影響後續 delivery，不改寫既有 delivery snapshot。

## DELETE `/webhook/merch/subscriptions/{subscriptionId}`

軟刪除 webhook subscription。刪除後不出現在前端清單，也不再參與後續事件派送。

後端刪除必須同時套用目前 identity 的 merchant id scope；不得只用 `subscriptionId` 軟刪除。

Operation semantics：

- 以 `subscriptionId + identity.merchantId` 查詢未刪除 subscription。
- 找不到、已刪除或不屬於目前 merchant 時，錯誤語意使用 not found。
- Soft delete 同步更新 deletion marker 與 update time。
- Soft delete 不刪除 subscription-event bindings；dispatcher 查詢必須排除 deleted subscription。

Response：

```json
{
  "deleted": true
}
```

重複刪除、刪除不存在 subscription、或刪除非目前商戶資料時，錯誤語意使用 `WEBHOOK_SUBSCRIPTION_NOT_FOUND`。REST error envelope 形狀依 api-gateway 既有 convention 定案。

## Non-Goals

- 第一版不提供 delivery history 查詢 API。
- 第一版不提供人工重送 delivery API。
- 第一版不提供 event type CRUD API。
- 第一版不提供 endpoint ownership verification API。
- 第一版不提供 secret rotation API。

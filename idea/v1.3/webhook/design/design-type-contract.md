---
status: draft
updated_at: 2026-06-29
updated_by: Codex
---

# Webhook 交易事件推播 — Type Contract Design

## Purpose

本文作為 webhook 型別邊界設計入口。後續可在此拆分或補充 domain raw、event contract、management read model 與 delivery internal DTO 等型別設計。

本文先定義 withdrawal / deposit 交易事件在 webhook 服務內部需要接收的 Fund contract raw subject，以及第一版 event key 對應關係。Fund contract raw 是 payload builder 的輸入來源，不等同於最終 DB schema、ORM entity、REST DTO 或對外 webhook payload。

具體 TypeScript 型別、class / interface 名稱、validation decorator、序列化格式與檔案位置留給 plan 依新服務實作慣例定案。

## Type Layers

- Domain raw：Fund broadcast event `payload.subject` 內的 contract raw object。
- Event contract：POST 到商戶 endpoint 的 external-facing payload envelope 與 event-specific data。
- Management read model：商戶後台管理 webhook subscription 與查詢 event type catalog 的 request / response 語意，REST contract 見 [`design-rest.md`](./design-rest.md)。
- Delivery internal DTO：publisher、worker、recovery 在服務內部傳遞 delivery 任務時使用的型別。

## Domain Raw Principles

Domain raw 表示 Fund service 在交易狀態變更時放入 domain event notification 的 contract raw object。Webhook inbound consumer 使用 domain raw 建立 delivery payload snapshot；delivery publisher 與 worker 不應在後續重新查交易現況組 payload。

第一版不為 8 個 event key 各自定義完整獨立 inbound raw DTO，而是直接消費 `contract-base` 已有的 Fund raw object：

- Deposit event 的 `payload.subject` 使用 `fund/raw/deposit.raw.ts` 的 `Deposit`。
- Withdrawal event 的 `payload.subject` 使用 `fund/raw/withdrawal-intent.raw.ts` 的 `WithdrawalIntent`。

Webhook payload builder 可在服務內部將 Fund raw object 映射為 withdrawal / deposit 兩組 builder input view。採 builder view 的原因是同一交易類型的事件共享大部分來源欄位，事件差異主要由 `eventKey`、交易狀態、發生時間與 optional reason 表達。若後續某個 event 需要明顯不同的欄位集合，再拆成 event-specific builder view。

Fund raw object 不包含以下 webhook 系統生成欄位：

- webhook delivery id。
- webhook payload `apiVersion`。
- webhook payload `deliveredAt`。
- signing header 或 signature。
- retry / attempt metadata。

Fund raw object 可包含 Webhook 對外 payload 不應暴露的內部資料。Payload builder 必須只取商戶處理事件所需的穩定欄位，不外露 actor details、完整 status history、wallet allocation、fee payer strategy、internal network endpoint lifecycle。

## Withdrawal Payload Builder View

Withdrawal payload builder view 由 `WithdrawalIntent` contract raw 映射而來，表示 withdrawal 狀態變更時 payload builder 可使用的來源資料。

Conceptual fields：

| Field | Required | 說明 |
| --- | --- | --- |
| `merchantId` | yes | 事件所屬商戶，來自 `WithdrawalIntent.senderMerchantId`。 |
| `withdrawalId` | yes | Withdrawal 識別值；對外 payload 命名為 `withdrawalId`。 |
| `status` | yes | Withdrawal 在事件發生後的狀態。 |
| `amount` | yes | Withdrawal 金額；對外 payload 應使用穩定 decimal string。 |
| `asset` | yes | 資產代號，例如 `USDT`。 |
| `network` | yes | 區塊鏈網路或交易網路代號。 |
| `toAddress` | yes | 出金目標地址。 |
| `merchantReference` | no | 商戶側對帳 reference；若 withdrawal flow 沒有對應欄位可省略。 |
| `createdAt` | yes | Withdrawal 建立時間。 |
| `updatedAt` | yes | Withdrawal 最後更新時間；可放入對外 payload data，但不作為 Webhook delivery `occurredAt` 的推導來源。 |
| `failureCode` | no | 穩定、可對外理解的失敗代碼；僅 failed 類事件可能提供。 |
| `failureReason` | no | 可對外揭露的失敗描述；是否納入第一版 payload 待定。 |
| `cancelledReason` | no | 可對外揭露的取消描述；是否納入第一版 payload 待定。 |

第一版 withdrawal event keys：

| Event key | Contract raw source | Expected status meaning | `resourceType` |
| --- | --- | --- | --- |
| `withdrawal.created` | `WithdrawalIntent` | withdrawal 已建立。 | `withdrawal-intent` |
| `withdrawal.cancelled` | `WithdrawalIntent` | withdrawal 已取消。 | `withdrawal-intent` |
| `withdrawal.failed` | `WithdrawalIntent` | withdrawal 已失敗。 | `withdrawal-intent` |
| `withdrawal.completed` | `WithdrawalIntent` | withdrawal 已完成。 | `withdrawal-intent` |

## Deposit Payload Builder View

Deposit payload builder view 由 `Deposit` contract raw 映射而來，表示 deposit 狀態變更時 payload builder 可使用的來源資料。

Conceptual fields：

| Field | Required | 說明 |
| --- | --- | --- |
| `merchantId` | yes | 事件所屬商戶，來自 `Deposit.recipientMerchantId`。 |
| `depositId` | yes | Deposit 識別值；對外 payload 命名為 `depositId`。 |
| `status` | yes | Deposit 在事件發生後的狀態。 |
| `amount` | yes | Deposit 金額；對外 payload 應使用穩定 decimal string。 |
| `asset` | yes | 資產代號，例如 `USDT`。 |
| `network` | yes | 區塊鏈網路或交易網路代號。 |
| `fromAddress` | no | 入金來源地址；鏈上資料尚未完整時可省略。 |
| `toAddress` | yes | 入金目標地址。 |
| `transactionHash` | no | 鏈上交易 hash；資料尚未存在時可省略。 |
| `merchantReference` | no | 商戶側對帳 reference；若 deposit flow 沒有對應欄位可省略。 |
| `createdAt` | yes | Deposit 建立時間。 |
| `updatedAt` | yes | Deposit 最後更新時間；可放入對外 payload data，但不作為 Webhook delivery `occurredAt` 的推導來源。 |
| `failureCode` | no | 穩定、可對外理解的失敗代碼；僅 failed 類事件可能提供。 |
| `failureReason` | no | 可對外揭露的失敗描述；是否納入第一版 payload 待定。 |
| `blockedCode` | no | 穩定、可對外理解的 blocked 代碼；僅 blocked 類事件可能提供。 |
| `blockedReason` | no | 可對外揭露的 blocked 描述；是否納入第一版 payload 待定。 |

第一版 deposit event keys：

| Event key | Contract raw source | Expected status meaning | `resourceType` |
| --- | --- | --- | --- |
| `deposit.created` | `Deposit` | deposit 已建立。 | `deposit` |
| `deposit.failed` | `Deposit` | deposit 已失敗。 | `deposit` |
| `deposit.completed` | `Deposit` | deposit 已完成。 | `deposit` |
| `deposit.blocked` | `Deposit` | deposit 已被系統阻擋或凍結處理。 | `deposit` |

## Inbound Event Mapping

Webhook inbound consumer 接收 domain raw 時，必須同時取得 source 與 event key。Event key 是 webhook code-defined event type catalog 的穩定識別，consumer 不接受 catalog 以外的 key。

Mapping rules：

- `source` 第一版固定為 `fund`，用於標示 producer service / bounded context。
- `eventKey` 必須存在於 backend TypeScript code-defined event type catalog。
- `resourceType` 追蹤上游 subject token：withdrawal 事件使用 `withdrawal-intent`，deposit 事件使用 `deposit`。
- `resourceIdentifier` 使用該交易主體 id：withdrawal 事件使用 `WithdrawalIntent.id`，deposit 事件使用 `Deposit.id`。對外 payload 仍可命名為 `withdrawalId` / `depositId`。
- `merchantId` 來自 domain raw 的 `merchantId`。
- `occurredAt` 由 inbound event `payload.change.occurredAt` 取得；Webhook 不從 raw `updatedAt` 或其他 raw 欄位推導。
- Payload builder 將 domain raw 轉換為 [`design-payload-contract.md`](./design-payload-contract.md) 的 external envelope + `data` shape，並在建立 delivery 時保存 snapshot。
- 第一版以 `source + eventKey + resourceType + resourceIdentifier + subscriptionId` 識別重複 delivery；此語意假設同一資源的同一 event key 只發生一次。

## Event Contract Boundary

External webhook payload 使用固定 envelope，`data` 內只放商戶處理事件所需的穩定欄位。Domain raw 可含有 payload builder 需要判斷的欄位，但不代表每個欄位都會外露。

第一版外部 payload 原則：

- `withdrawalId` / `depositId` 使用交易識別值。
- `status` 使用商戶可理解的穩定狀態文字。
- `amount` 使用 decimal string，避免 JSON number 精度問題。
- Optional reason 欄位只有在產品決定對外揭露穩定 code / reason 時才加入。
- `id`、`apiVersion`、`deliveredAt` 由 webhook 系統補齊，不從 domain raw 輸入。

## Management Read Model Boundary

Merchant console 的 subscription management read model 與交易 domain raw 無直接耦合。

Subscription summary / detail 是 management read model，不是 `WebhookSubscription` domain object 本體。`eventTypeCount` 與 `eventTypes` 都由 subscription-event binding 與 code-defined event catalog 組裝，不應反推為 subscription 本體欄位。

Subscription `id` 與 path parameter `subscriptionId` 使用平台 short id string representation；persistence id 的實際 primitive、codec 使用與 class 命名由 plan 依 codebase pattern 決定。時間欄位需以穩定、可序列化且具 UTC 語意的格式對外呈現。

Management read model conceptual fields：

| Model | Fields |
| --- | --- |
| Event type option | `eventKey`, `displayName`, `sortOrder` |
| Event type list | event type option collection |
| Subscription summary | `id`, `endpointUrl`, `eventTypeCount`, `createdAt`, `updatedAt` |
| Subscription detail | `id`, `endpointUrl`, `eventTypes`, `createdAt`, `updatedAt` |
| Subscription list | subscription summary collection with paging metadata; concrete envelope follows codebase plan |
| Create subscription input | `endpointUrl`, `eventKeys` |
| Update subscription input | `endpointUrl`, `eventKeys` |
| Delete subscription result | deleted success marker |

Create and update inputs have the same conceptual field set. Plan may implement them with one shared class or separate classes according to codebase convention.

UI 顯示文字第一版使用 catalog 提供的 `displayName`；`eventKey` 仍是 request validation、subscription binding 與 inbound consumer matching 的穩定識別。`sortOrder` 只供前端排序，不代表事件優先權或處理順序。

## Signing Boundary

第一版 signing secret 不屬於 subscription-level type contract。Webhook delivery 使用環境變數提供的統一預設 signing secret。

因此 Stage 1 subscription management read model、domain raw 與 management API 不包含 secret 欄位。

第一版 delivery signing 使用 HMAC-SHA256。Worker 以 delivery payload snapshot 序列化出的 raw JSON request body bytes 建立簽章，並帶出 `X-ESingPay-Timestamp`、`X-ESingPay-Signature`、`X-ESingPay-Delivery-Id` 與 `X-ESingPay-Event-Key` headers。簽章 input 與驗簽規則見 [`design-payload-contract.md`](./design-payload-contract.md)。

## Open Points

- External event contract 是否需要與 external API contract 共用命名與 primitive type。
- Management read model 是否只服務 merchant console，或也會被 external API 文件引用。
- Delivery internal DTO 是否需要獨立於 persistence entity 定義。
- Failure / blocked / cancelled reason 是否納入第一版 external payload。

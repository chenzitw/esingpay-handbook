---
status: draft
updated_at: 2026-06-22
updated_by: Codex
---

# Webhook 交易事件推播 — Type Contract Design

## Purpose

本文作為 webhook 型別邊界設計入口。後續可在此拆分或補充 domain raw、event contract、management read model 與 delivery internal DTO 等型別設計。

本文先定義 withdrawal / deposit 交易事件在 webhook 服務內部需要接收的 domain raw family，以及第一版 event key 對應關係。Domain raw 是 payload builder 的輸入，不等同於最終 DB schema、ORM entity、REST DTO 或對外 webhook payload。

具體 TypeScript 型別、class / interface 名稱、validation decorator、序列化格式與檔案位置留給 plan 依新服務實作慣例定案。

## Type Layers

- Domain raw：內部交易服務產生 webhook event 前的原始 domain input。
- Event contract：POST 到商戶 endpoint 的 external-facing payload envelope 與 event-specific data。
- Management read model：商戶後台管理 webhook subscription 與查詢 event type catalog 的 request / response 語意，REST contract 見 [`design-rest.md`](./design-rest.md)。
- Delivery internal DTO：publisher、worker、recovery 在服務內部傳遞 delivery 任務時使用的型別。

## Domain Raw Principles

Domain raw 表示 Fund service 在交易狀態變更時放入 domain event notification 的最小業務事實。Webhook inbound consumer 使用 domain raw 建立 delivery payload snapshot；delivery publisher 與 worker 不應在後續重新查交易現況組 payload。

第一版不為 8 個 event key 各自定義完整獨立 raw DTO，而是採兩組 raw family：

- `WithdrawalWebhookDomainRaw`
- `DepositWebhookDomainRaw`

採 family 的原因是同一交易類型的事件共享大部分來源欄位，事件差異主要由 `eventKey`、交易狀態、發生時間與 optional reason 表達。若後續某個 event 需要明顯不同的欄位集合，再拆成 event-specific raw extension。

Domain raw 不包含以下 webhook 系統生成欄位：

- webhook delivery id。
- webhook payload `api_version`。
- webhook payload `delivered_at`。
- signing header 或 signature。
- retry / attempt metadata。

Domain raw 也不應暴露內部流程細節，例如 actor details、完整 status history、wallet allocation、fee payer strategy、internal network endpoint lifecycle。

## Withdrawal Domain Raw

`WithdrawalWebhookDomainRaw` 表示 withdrawal 狀態變更時，payload builder 可使用的來源資料。

Conceptual fields：

| Field | Required | 說明 |
| --- | --- | --- |
| `merchantId` | yes | 事件所屬商戶。 |
| `withdrawalId` | yes | Withdrawal 識別值；對外 payload 命名為 `withdrawal_id`。 |
| `status` | yes | Withdrawal 在事件發生後的狀態。 |
| `amount` | yes | Withdrawal 金額；對外 payload 應使用穩定 decimal string。 |
| `asset` | yes | 資產代號，例如 `USDT`。 |
| `network` | yes | 區塊鏈網路或交易網路代號。 |
| `toAddress` | yes | 出金目標地址。 |
| `merchantReference` | no | 商戶側對帳 reference；若 withdrawal flow 沒有對應欄位可省略。 |
| `createdAt` | yes | Withdrawal 建立時間。 |
| `updatedAt` | yes | Withdrawal 最後更新時間；通常作為 event occurred time 的候選來源。 |
| `failureCode` | no | 穩定、可對外理解的失敗代碼；僅 failed 類事件可能提供。 |
| `failureReason` | no | 可對外揭露的失敗描述；是否納入第一版 payload 待定。 |
| `cancelledReason` | no | 可對外揭露的取消描述；是否納入第一版 payload 待定。 |

第一版 withdrawal event keys：

| Event key | Domain raw family | Expected status meaning | `resource_type` |
| --- | --- | --- | --- |
| `withdrawal.created` | `WithdrawalWebhookDomainRaw` | withdrawal 已建立。 | `withdrawal` |
| `withdrawal.cancelled` | `WithdrawalWebhookDomainRaw` | withdrawal 已取消。 | `withdrawal` |
| `withdrawal.failed` | `WithdrawalWebhookDomainRaw` | withdrawal 已失敗。 | `withdrawal` |
| `withdrawal.completed` | `WithdrawalWebhookDomainRaw` | withdrawal 已完成。 | `withdrawal` |

## Deposit Domain Raw

`DepositWebhookDomainRaw` 表示 deposit 狀態變更時，payload builder 可使用的來源資料。

Conceptual fields：

| Field | Required | 說明 |
| --- | --- | --- |
| `merchantId` | yes | 事件所屬商戶。 |
| `depositId` | yes | Deposit 識別值；對外 payload 命名為 `deposit_id`。 |
| `status` | yes | Deposit 在事件發生後的狀態。 |
| `amount` | yes | Deposit 金額；對外 payload 應使用穩定 decimal string。 |
| `asset` | yes | 資產代號，例如 `USDT`。 |
| `network` | yes | 區塊鏈網路或交易網路代號。 |
| `fromAddress` | no | 入金來源地址；鏈上資料尚未完整時可省略。 |
| `toAddress` | yes | 入金目標地址。 |
| `transactionHash` | no | 鏈上交易 hash；資料尚未存在時可省略。 |
| `merchantReference` | no | 商戶側對帳 reference；若 deposit flow 沒有對應欄位可省略。 |
| `createdAt` | yes | Deposit 建立時間。 |
| `updatedAt` | yes | Deposit 最後更新時間；通常作為 event occurred time 的候選來源。 |
| `failureCode` | no | 穩定、可對外理解的失敗代碼；僅 failed 類事件可能提供。 |
| `failureReason` | no | 可對外揭露的失敗描述；是否納入第一版 payload 待定。 |
| `blockedCode` | no | 穩定、可對外理解的 blocked 代碼；僅 blocked 類事件可能提供。 |
| `blockedReason` | no | 可對外揭露的 blocked 描述；是否納入第一版 payload 待定。 |

第一版 deposit event keys：

| Event key | Domain raw family | Expected status meaning | `resource_type` |
| --- | --- | --- | --- |
| `deposit.created` | `DepositWebhookDomainRaw` | deposit 已建立。 | `deposit` |
| `deposit.failed` | `DepositWebhookDomainRaw` | deposit 已失敗。 | `deposit` |
| `deposit.completed` | `DepositWebhookDomainRaw` | deposit 已完成。 | `deposit` |
| `deposit.blocked` | `DepositWebhookDomainRaw` | deposit 已被系統阻擋或凍結處理。 | `deposit` |

## Inbound Event Mapping

Webhook inbound consumer 接收 domain raw 時，必須同時取得 source 與 event key。Event key 是 webhook code-defined event type catalog 的穩定識別，consumer 不接受 catalog 以外的 key。

Mapping rules：

- `source` 第一版固定為 `fund`，用於標示 producer service / bounded context。
- `eventKey` 必須存在於 backend TypeScript code-defined event type catalog。
- `resource_type` 由交易類型決定：withdrawal 事件使用 `withdrawal`，deposit 事件使用 `deposit`。
- `resource_identifier` 使用該交易主體 id：withdrawal 使用 `withdrawalId`，deposit 使用 `depositId`。
- `merchant_id` 來自 domain raw 的 `merchantId`。
- `occurred_at` 預設由 raw 的狀態變更時間或 `updatedAt` 映射；具體來源由 Stage 2 plan 依交易模型決定。
- Payload builder 將 domain raw 轉換為 [`design-payload-contract.md`](./design-payload-contract.md) 的 external envelope + `data` shape，並在建立 delivery 時保存 snapshot。
- 第一版以 `source + eventKey + resourceType + resourceIdentifier + subscriptionId` 識別重複 delivery；此語意假設同一資源的同一 event key 只發生一次。

## Event Contract Boundary

External webhook payload 使用固定 envelope，`data` 內只放商戶處理事件所需的穩定欄位。Domain raw 可含有 payload builder 需要判斷的欄位，但不代表每個欄位都會外露。

第一版外部 payload 原則：

- `withdrawal_id` / `deposit_id` 使用交易識別值。
- `status` 使用商戶可理解的穩定狀態文字。
- `amount` 使用 decimal string，避免 JSON number 精度問題。
- Optional reason 欄位只有在產品決定對外揭露穩定 code / reason 時才加入。
- `id`、`api_version`、`delivered_at` 由 webhook 系統補齊，不從 domain raw 輸入。

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

因此 Stage 1 subscription management read model、domain raw 與 management API 不包含 secret 欄位。Signing header、signature algorithm 與驗簽文件留給 Stage 4 設計或 plan 補齊。

## Open Points

- External event contract 是否需要與 external API contract 共用命名與 primitive type。
- Management read model 是否只服務 merchant console，或也會被 external API 文件引用。
- Delivery internal DTO 是否需要獨立於 persistence entity 定義。
- Failure / blocked / cancelled reason 是否納入第一版 external payload。
- `occurred_at` 應使用交易狀態變更時間、交易 `updatedAt`，或 producer 發送時間。

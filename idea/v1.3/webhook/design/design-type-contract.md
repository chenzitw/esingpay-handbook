---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Type Contract Design

## Purpose

本文作為 webhook 型別定義設計入口。後續可在此拆分或補充 domain raw、event contract、request / response DTO 等型別設計。

本文先定義 withdrawal / deposit 交易事件在 webhook 服務內部需要接收的 domain raw family，以及第一版 event key 對應關係。Domain raw 是 payload builder 的輸入，不等同於最終 DB schema、ORM entity、REST DTO 或對外 webhook payload。

具體 TypeScript 型別、validation decorator、序列化格式與檔案位置留給 blueprint / plan 依新服務實作慣例定案。

## Type Layers

- Domain raw：內部交易服務產生 webhook event 前的原始 domain input。
- Event contract：POST 到商戶 endpoint 的 external-facing payload envelope 與 event-specific data。
- Management DTO：商戶後台管理 webhook subscription 與查詢 event type catalog 的 request / response DTO，REST 草案見 [`design-rest.md`](./design-rest.md)。
- Delivery internal DTO：dispatcher、worker、recovery 在服務內部傳遞 delivery 任務時使用的型別。

## Domain Raw Principles

Domain raw 表示交易 domain 在狀態變更當下交給 webhook event producer 的最小業務事實。Webhook 服務使用 domain raw 建立 outbox payload snapshot；delivery worker 不應在送出前重新查交易現況組 payload。

第一版不為 8 個 event key 各自定義完整獨立 raw DTO，而是採兩組 raw family：

- `WithdrawalWebhookDomainRaw`
- `DepositWebhookDomainRaw`

採 family 的原因是同一交易類型的事件共享大部分來源欄位，事件差異主要由 `eventKey`、交易狀態、發生時間與 optional reason 表達。若後續某個 event 需要明顯不同的欄位集合，再拆成 event-specific raw extension。

Domain raw 不包含以下 webhook 系統生成欄位：

- webhook delivery id。
- webhook outbox event id。
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

## Event Production Mapping

Webhook event producer 接收 domain raw 時，必須同時確定 event key。Event key 是 webhook code-defined event type catalog 的穩定識別，producer 不應自行產生 catalog 以外的 key。

Mapping rules：

- `eventKey` 必須存在於 backend TypeScript code-defined event type catalog。
- `resource_type` 由交易類型決定：withdrawal 事件使用 `withdrawal`，deposit 事件使用 `deposit`。
- `resource_identifier` 使用該交易主體 id：withdrawal 使用 `withdrawalId`，deposit 使用 `depositId`。
- `merchant_id` 來自 domain raw 的 `merchantId`。
- `occurred_at` 預設由 raw 的狀態變更時間或 `updatedAt` 映射；具體來源由 Stage 2 plan 依交易模型決定。
- Payload builder 將 domain raw 轉換為 [`design-payload-contract.md`](./design-payload-contract.md) 的 external envelope + `data` shape。

## Event Contract Boundary

External webhook payload 使用固定 envelope，`data` 內只放商戶處理事件所需的穩定欄位。Domain raw 可含有 payload builder 需要判斷的欄位，但不代表每個欄位都會外露。

第一版外部 payload 原則：

- `withdrawal_id` / `deposit_id` 使用交易識別值。
- `status` 使用商戶可理解的穩定狀態文字。
- `amount` 使用 decimal string，避免 JSON number 精度問題。
- Optional reason 欄位只有在產品決定對外揭露穩定 code / reason 時才加入。
- `event_id`、`id`、`api_version`、`delivered_at` 由 webhook 系統補齊，不從 domain raw 輸入。

## Management DTO Boundary

Merchant console 的 subscription management DTO 與交易 domain raw 無直接耦合。

Subscription summary / detail DTO 是 management read model，不是 `WebhookSubscription` domain object 本體。`eventTypeCount` 與 `eventTypes` 都由 subscription-event binding 與 code-defined event catalog 組裝，不應反推為 subscription 本體欄位。

`id` 欄位以 `string` 表示，對應後端 bigint 序列化結果。時間欄位以 `IsoDateTimeUtc` 表示，值為 UTC ISO 8601 字串。具體 TypeScript class、validation decorator 與序列化設定留給 plan 依 codebase pattern 決定。

```typescript
type IsoDateTimeUtc = string;

interface WebhookEventTypeOptionDto {
  eventKey: string;
  sortOrder: number;
}

interface WebhookSubscriptionSummaryDto {
  id: string;
  endpointUrl: string;
  eventTypeCount: number;
  createdAt: IsoDateTimeUtc;
  updatedAt: IsoDateTimeUtc;
}

interface WebhookSubscriptionDetailDto {
  id: string;
  endpointUrl: string;
  eventTypes: WebhookEventTypeOptionDto[];
  createdAt: IsoDateTimeUtc;
  updatedAt: IsoDateTimeUtc;
}

interface WebhookSubscriptionListResponseDto {
  items: WebhookSubscriptionSummaryDto[];
  page: number;
  pageSize: number;
  total: number;
}

interface CreateWebhookSubscriptionRequestDto {
  endpointUrl: string;
  eventKeys: string[];
}

interface UpdateWebhookSubscriptionRequestDto {
  endpointUrl: string;
  eventKeys: string[];
}

interface DeleteWebhookSubscriptionResponseDto {
  deleted: true;
}
```

`CreateWebhookSubscriptionRequestDto` 與 `UpdateWebhookSubscriptionRequestDto` 欄位集合相同；plan 可根據 codebase 慣例決定使用同一個 class 或分開定義。

UI 顯示文字第一版直接使用 `eventKey`；`sortOrder` 只供前端排序，不代表事件優先權或處理順序。

## Signing Boundary

第一版 signing secret 不屬於 subscription-level type contract。Webhook delivery 使用環境變數提供的統一預設 signing secret。

因此 Stage 1 subscription DTO、domain raw 與 management API 不包含 secret 欄位。Signing header、signature algorithm 與驗簽文件留給 Stage 4 設計或 plan 補齊。

## Open Points

- External event contract 是否需要與 external API DTO 共用命名與 primitive type。
- Management DTO 是否只服務 merchant console，或也會被 external API 文件引用。
- Delivery internal DTO 是否需要獨立於 persistence entity 定義。
- Failure / blocked / cancelled reason 是否納入第一版 external payload。
- `occurred_at` 應使用交易狀態變更時間、交易 `updatedAt`，或 outbox event 建立時間。

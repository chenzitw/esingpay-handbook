---
status: draft
updated_at: 2026-06-29
updated_by: Codex
---

# Webhook 交易事件推播 — Inbound Event Contract Design

## Purpose

本文定義 Webhook 如何透過 Azure Service Bus subscription 監聽 Fund 廣播的 transaction event，並將事件帶入 Webhook boundary。Fund event broadcast 不是為 Webhook 專設；Fund 發布交易事件，有需要的服務自行建立 subscription 收聽，目前只有 Webhook 需要消費這批事件。

本文只處理 Webhook inbound 使用到的 Azure Service Bus event topic / subscription、transport envelope、application payload、session 與 unsupported event handling。

Webhook 內部的 delivery execution job 屬 service-private BullMQ queue，不使用本文的 event topic naming 或 event envelope。Delivery job contract 見 [`design-queue-topology.md`](./design-queue-topology.md)。

## Transport Boundary

Fund transaction status change 是 cross-service asynchronous fact，因此走 Azure Service Bus Topic / Subscription。Webhook 以自己的 durable subscription 消費 Fund event，收到後 matching subscriptions 並建立 `webhook_delivery`。

Provider 決策：

- Cross-service event broadcast 採 Azure Service Bus Topic / Subscription。
- Subscription 使用 consuming service role 命名，Webhook 第一版使用 `srv-webhook`。
- Webhook inbound subscription 需啟用 session，consumer 必須使用 session-aware receiver。
- 第一版 Azure SB topics / subscriptions 由 infrastructure 或人工流程預先建立，不由 Webhook Stage 2 建置；本文只定義 naming 與 Webhook runtime dependency。
- Delivery execution job 不走 Azure SB event topic；Stage 3 以 BullMQ private job 發布 `webhook.delivery.execute`。

## Topic Naming

Fund event topic follow [`../../architecture-improvement/messaging-transport-design-event-naming-rule.md`](../../architecture-improvement/messaging-transport-design-event-naming-rule.md) 的三段結構：

```text
<namespace>.<resource>.<predicate>
```

第一版採 resource lifecycle stream，不採 topic-per-status。

命名原則：

- `namespace` 表示事件 owner / bounded context；第一版 Fund transaction event 使用 `fund`。
- `resource` 表示 producer 領域資源，對齊 Fund contract-event resource vocabulary；第一版使用 `deposit`、`withdrawal-intent`。
- `predicate` 第一版統一使用 `status-changed`，表示該 resource 的狀態已變更。
- `.` 只分隔 identity 欄位；段內多詞使用 `-`。
- Topic 名稱描述 producer 發出的領域事實，不描述 webhook 內部如何處理。

第一版候選 topic：

| Webhook event keys | Azure SB event topic | Azure SB subscription |
| --- | --- | --- |
| `withdrawal.created`, `withdrawal.cancelled`, `withdrawal.failed`, `withdrawal.completed` | `fund.withdrawal-intent.status-changed` | `srv-webhook` |
| `deposit.created`, `deposit.failed`, `deposit.completed`, `deposit.blocked` | `fund.deposit.status-changed` | `srv-webhook` |

Azure SB subscription name is scoped by topic, so the same consumer role name `srv-webhook` is intentionally reused under each topic:

```text
fund.deposit.status-changed / srv-webhook
fund.withdrawal-intent.status-changed / srv-webhook
```

Withdrawal 在 Azure SB topic / event envelope、payload subject 與 Webhook delivery `resourceType` 中使用 Fund domain token `withdrawal-intent`；只有 Webhook catalog event key 與對外 payload 命名使用商戶語彙 `withdrawal`。Topic 形狀維持三段 `namespace.resource.predicate`。

## Topic Granularity

第一版不採 `fund.deposit.completed`、`fund.deposit.failed` 這種一個 status 一個 topic 的粒度。

理由：

- Azure SB subscription 掛在單一 topic 下；若每個 status 各自一個 topic，同一 resource 的多個狀態變更會分散到不同 subscription / receiver flow。
- `sessionId` 的 FIFO 只在各自 topic 內成立；topic-per-status 無法表達同一 resource lifecycle 的整體順序。
- `status-changed` topic 讓同一 deposit / withdrawal intent 的所有狀態變更進入同一 topic + session。
- 具體狀態語意由 envelope `eventType = status-changed` 搭配 payload `change.status` 表達；Webhook subscription matching 使用 consumer 由 subject / change 映射出的 catalog `eventKey`。

Webhook 第一版不使用 Azure SB subscription filter 表示具體 status。Webhook consumer 接收 resource lifecycle stream 後自行依映射出的 `eventKey` 判斷是否為 Webhook 支援事件。

## Session Requirement

Webhook inbound Fund event consumption 需要同一業務資源內 FIFO。Azure SB subscription 因此需以 session-enabled entity provisioning，且 producer 必須為每則 event 帶 `sessionId`。

Session 規則：

- `sessionId` 預設使用 `namespace + ':' + subjectType + ':' + subjectId`，例如 `fund:deposit:67890` 或 `fund:withdrawal-intent:12345`。
- FIFO 保證只要求同一 `sessionId` 內成立；不同 deposit / withdrawal intent 之間不保證全域順序。
- Webhook consumer 必須使用 session receiver，且同一 session 內一次只處理一則 message。
- 若 plan 決定要以 merchant 為 ordering key，必須另行記錄吞吐量與 head-of-line blocking trade-off；預設不採 merchant-wide session。

Stage 2 runtime dependency:

| Topic | Subscription | Requirement |
| --- | --- | --- |
| `fund.deposit.status-changed` | `srv-webhook` | Subscription must be session-enabled. |
| `fund.withdrawal-intent.status-changed` | `srv-webhook` | Subscription must be session-enabled. |

LockDuration、MaxDeliveryCount、DLQ policy 與其他 Azure SB operational settings 依環境既有 convention 或 infra 手動設定，不在本文定案。

## Transport Envelope

Architecture-improvement 的 event envelope 採 plain envelope + payload 子樹。Fund status-changed event 第一版將事件 identity、routing 與 time 放在第一層，domain subject 與本次變更內容留在 `payload` 內：

```json
{
  "id": "event-id",
  "occurredAt": "2026-06-10T02:00:00.000Z",
  "namespace": "fund",
  "subjectType": "deposit",
  "subjectId": "67890",
  "eventType": "status-changed",
  "payload": {
    "subject": {},
    "change": {
      "status": "completed",
      "occurredAt": "2026-06-10T02:00:00.000Z",
      "reason": null
    }
  }
}
```

Envelope rules：

- `id`、`occurredAt`、`namespace`、`subjectType`、`subjectId` 與 `eventType` 屬 transport / event scaffolding，必須讓 broker routing、DLQ triage 與 tracing 可不經 payload codec 讀取。
- `eventType` 表示 topic predicate，例如 `status-changed`；它不是 Webhook subscription catalog key。
- `payload` 是 domain data 子樹；若 event runtime 導入 SuperJSON codec，codec 只作用於 `payload`。
- `payload.subject` 保存狀態變更後的 Fund contract raw object，而不是只放 resource id。Deposit event 使用 `contract-base` 的 `fund/raw/deposit.raw.ts` `Deposit`；withdrawal event 使用 `contract-base` 的 `fund/raw/withdrawal-intent.raw.ts` `WithdrawalIntent`。
- `payload.change` 保存本次變更摘要，例如狀態、Fund 指定的狀態發生時間與 reason。
- 第一版 direct-to-delivery idempotency 不依賴 producer-stable `id`；若 producer 尚未提供可作 domain idempotency anchor 的 stable event id，Webhook 仍以 delivery composite identity 去重。
- `payload.change.occurredAt` 是 Fund producer 提供的狀態變更發生時間。Webhook 不推導此時間、不 fallback 到 subject `updatedAt` 或 envelope `occurredAt`；delivery `occurredAt` 與 outbound webhook payload `occurredAt` 必須完全 follow `payload.change.occurredAt`。
- Envelope `occurredAt` 表示 event record / publish time，只供 tracing、broker triage 或 audit 使用，不作為交易狀態發生時間。

## Application Payload

Webhook consumer 需要的 conceptual `payload`：

```json
{
  "subject": {},
  "change": {
    "status": "completed",
    "occurredAt": "2026-06-10T02:00:00.000Z",
    "reason": null
  }
}
```

欄位語意：

| Field | Meaning |
| --- | --- |
| `subject` | 狀態變更後的 Fund contract raw object。Deposit 使用 `Deposit`；withdrawal 使用 `WithdrawalIntent`。 |
| `change.status` | 本次狀態變更後的狀態。 |
| `change.occurredAt` | 本次狀態變更發生時間，由 Fund producer 代入並負責語意。Webhook 必須照值使用，不自行推導。 |
| `change.reason` | 狀態變更 reason；第一版只作內部 payload builder input，不直接承諾為外部 webhook payload 的 stable code。 |

Producer 不應在 payload 內放入 webhook delivery id、signature、retry metadata、商戶 endpoint 或 webhook worker 所需的內部欄位。

`payload.subject` 必須足以讓 Webhook 建立 delivery payload snapshot；Webhook consumer 不應只收到 resource id 後回查 Fund current state。Deposit / withdrawal 的外部 webhook payload 仍由 Webhook payload builder 從 contract raw object 映射而來，不代表 contract raw 的所有欄位都會外露給商戶。

Merchant scope 不上提為 envelope metadata。Webhook consumer 第一版需由 `payload.subject` 依 subject type 映射出 merchant scope：Deposit 使用 `recipientMerchantId`，WithdrawalIntent 使用 `senderMerchantId`。若 subject 不屬於 merchant，consumer 不應建立 merchant webhook delivery；具體 discard / violation 行為由 Stage 2 plan 依 Fund event semantics 定案。

## Webhook Mapping

Webhook consumer 需將通用 Fund event 映射為 Webhook application input：

| Webhook input | Mapping |
| --- | --- |
| `source` | envelope `namespace`；第一版為 `fund`。 |
| `eventKey` | 由 `subjectType + change.status` 映射到 webhook catalog，例如 `deposit` + `completed` -> `deposit.completed`；`withdrawal-intent` + `completed` -> `withdrawal.completed`。 |
| `resourceType` | 追蹤上游 subject 的 resource token，第一版直接使用 envelope `subjectType`；`deposit` -> `deposit`，`withdrawal-intent` -> `withdrawal-intent`。 |
| `resourceIdentifier` | envelope `subjectId`。 |
| `merchantId` | 從 `payload.subject` 的 merchant ownership 欄位映射；不屬於 envelope 第一層欄位。 |
| `occurredAt` | 使用 `payload.change.occurredAt`。 |
| domain raw | `payload.subject`。 |

## Consumer Handling Rules

- Webhook consumer 只為可映射到 code-defined event catalog 的 `eventKey` 建立 delivery。
- `subjectType + change.status` 無法映射到 Webhook catalog 時，consumer 應視為 non-webhook event：不建立 delivery、不查 subscription，記錄必要 metric / debug 後 complete message。
- Unsupported derived `eventKey` 不應因 Webhook 不支援而進入 retry 或 DLQ；Fund lifecycle stream 可包含 Webhook 第一版不處理的事件。
- `namespace`、`subjectType`、`subjectId`、`eventType`、`payload.subject`、`payload.change` 或 `payload.change.occurredAt` 缺失或 malformed 時，才屬 contract violation；plan 需依 broker adapter convention 決定 retry / DLQ / alert 行為。
- Azure SB topic 必須可依 naming rule 對應到 `namespace + subjectType + eventType`；第一版 `eventType` 為 `status-changed`。
- `source`、derived `eventKey`、`resourceType` 與 `resourceIdentifier` 會映射並寫入每一筆 matching `webhook_delivery`。
- Consumer matching subscriptions 與建立 deliveries 應在同一 DB transaction 內完成；commit 後才 complete inbound event message。
- 重送時以 `source + eventKey + resourceType + resourceIdentifier + subscription` 防止重複 delivery。
- 沒有 matching subscription 時不建立 persistence record，consumer 可直接 complete message。
- Webhook consumer 建立 delivery 後，不同步呼叫商戶 endpoint。

Handling matrix：

| Condition | Webhook action | Message outcome |
| --- | --- | --- |
| Valid event + matching active subscriptions | 建立一筆 `PENDING webhook_delivery` per matching subscription，保存 source/resource/endpoint/payload snapshot。 | DB commit 後 complete。 |
| Valid event + no matching subscription | 不建立 persistence record。 | Complete。 |
| Duplicate redelivery for an already-created delivery | 透過 `source + eventKey + resourceType + resourceIdentifier + subscription` composite identity 忽略重複建立。 | Complete after idempotent handling。 |
| `subjectType + change.status` 無法映射到 Webhook catalog | 視為 non-webhook lifecycle event；不查 subscription、不建立 delivery，記錄必要 metric / debug。 | Complete。 |
| `payload.subject` 不屬於 merchant scope，例如 merchant ownership 欄位為 null | 不建立 merchant webhook delivery，記錄必要 metric / debug。 | Complete。 |
| Required envelope / payload field missing or malformed | 視為 inbound contract violation；不建立 delivery。 | 依 broker adapter convention retry / DLQ / alert，不應靜默 complete。 |
| Delivery persistence transaction failure | 不 complete message，保留 broker redelivery 機會。 | Abandon / retry according to adapter convention。 |

## Stage Relationship

- Stage 2 owns inbound event consumption, validation, subscription matching and delivery creation。
- Stage 3 owns private delivery job publishing；不改寫 inbound event contract。
- Stage 4 owns delivery worker execution；不重新查 Fund source data。

## Open Points

- 實際 transport envelope type 應落在 `contract-event` shared contract layer；目前既有 `EventEnvelope<TPayload>` 為 `{ id, occurredAt, payload }`，需在 Stage 2 plan 決定是否新增 / 擴充 broadcast envelope 以承載 `namespace`、`subjectType`、`subjectId` 與 `eventType`。
- Azure SB SDK adapter、Nest module、session receiver lifecycle 與 message settlement policy 應落在 `service-kit` runtime layer；Webhook service 只接 handler / mapping / delivery creation，不各自初始化 Azure SDK。

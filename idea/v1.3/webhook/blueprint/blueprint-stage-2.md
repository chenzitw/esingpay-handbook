---
status: draft
updated_at: 2026-06-29
updated_by: Codex
---

# Webhook 交易事件推播 — Stage 2 Blueprint

## Goal

Stage 2 完成 Webhook inbound domain event consumption、subscription matching 與 delivery creation。Webhook 從 Azure Service Bus event topic / subscription 接收 Fund transaction event，直接建立 matching deliveries；第一版不建立 webhook outbox 或 inbox。

## Scope

In scope：

- Webhook inbound domain event consumer。
- Consumer-side event envelope validation and mapping。
- Event key 到 code-defined event type catalog 的解析。
- Active subscription matching。
- `webhook_delivery` persistence foundation。
- Event source、resource、endpoint 與 payload snapshot。
- Direct-to-delivery composite idempotency。
- Inbound message completion boundary。

Out of scope：

- Fund event publisher、transaction status hooks、producer outbox 與 Fund code changes。
- Webhook outbox / inbox persistence。
- Delivery execution job publishing and publish recovery；Stage 3 會把 post-commit publisher 接回本 consumer flow。
- HTTP POST、signing 與 delivery retry。
- Deferred inbox migration。

## Inputs

- Persistence model：[`../design/design-persistence-model.md`](../design/design-persistence-model.md)。
- Inbound event contract：[`../design/design-inbound-event-contract.md`](../design/design-inbound-event-contract.md)。
- Queue topology boundary with Stage 3：[`../design/design-queue-topology.md`](../design/design-queue-topology.md)。
- Service boundary：[`../design/design-service-boundary.md`](../design/design-service-boundary.md)。
- Type boundary：[`../design/design-type-contract.md`](../design/design-type-contract.md)。
- Payload contract：[`../design/design-payload-contract.md`](../design/design-payload-contract.md)。
- Stage 1 landed subscription persistence、binding relation 與 code-defined event catalog。
- Azure SDK session receiver sample：`nest-service-bus/src/event/azure-sdk-event.service.ts`。此 external sample 可作 Stage 2 plan 的 adapter survey input，不是 Webhook codebase placement 或 production policy 的 design authority。

## External Dependency

Stage 2 假設 Fund transaction domain event 已可由 Azure Service Bus Topic / session-enabled Subscription 送達 Webhook。Blueprint 只鎖定 Webhook consumer 所需的 input semantics，不規劃或估算 producer implementation。

Webhook 端依賴的 transport semantics：

- Topic 命名 follow architecture-improvement naming rule 的三段結構，但 webhook 第一版採 resource lifecycle stream。
- Stage 2 runtime 依賴以下既存 Azure SB entities：`fund.deposit.status-changed / srv-webhook`、`fund.withdrawal-intent.status-changed / srv-webhook`。
- `srv-webhook` 是每個 Fund event topic 底下的 Webhook consumer subscription name；Azure SB subscription name scoped by topic，因此可在不同 topic 下重複使用。
- Webhook inbound subscription 必須啟用 session，producer 必須帶 `sessionId`，預設 ordering key 為 `namespace + ':' + subjectType + ':' + subjectId`。
- Consumer 必須使用 session-aware receiver；FIFO 只要求同一 `sessionId` 內成立。
- Azure SB topic / subscription 由 infrastructure 或人工流程預先建立，不由 Webhook Stage 2 建置；LockDuration、MaxDeliveryCount、DLQ policy 依環境 operational convention。
- Delivery execution job 不走 Azure SB event topic；Stage 3 仍以 BullMQ private queue 發布 `webhook.delivery.execute` job。

Consumer 必須從 inbound event envelope 取得：

- `id` 與 `occurredAt`。
- `namespace`，第一版為 `fund`。
- `subjectType`，第一版為 `deposit` 或 `withdrawal-intent`。
- `subjectId`。
- `eventType`，第一版為 `status-changed`。
- `payload.subject`，作為 payload builder 所需 Fund contract raw object；deposit 使用 `Deposit`，withdrawal 使用 `WithdrawalIntent`。
- `payload.change`，作為本次狀態變更摘要。

Consumer 必須在 application mapping 階段推導：

- `source`，由 `namespace` 映射，第一版為 `fund`。
- `eventKey`，由 `subjectType + change.status` 映射到 code-defined webhook event catalog；`withdrawal-intent` 狀態映射到 `withdrawal.*` event keys。
- `merchantId`，由 `payload.subject` 依 subject type 映射；不要求 producer 上提為 envelope metadata。
- `resourceType`，追蹤上游 subject 的 resource token，第一版直接使用 `subjectType`；`deposit` -> `deposit`，`withdrawal-intent` -> `withdrawal-intent`。
- `resourceIdentifier`，由 `subjectId` 映射。
- `occurredAt`，直接使用 `payload.change.occurredAt`；Webhook 不 fallback 到 envelope `occurredAt` 或 subject `updatedAt`。

實際 transport envelope type、shared contract landing 與 Webhook codebase broker adapter wiring 由 Stage 2 codebase plan 依 event adapter convention 定案，但不得將 Fund publisher implementation 或 Azure SB entity provisioning 納入 Webhook plan。

## Consumption Flow

```text
webhook inbound event consumer
  -> validate envelope + subject + change
  -> map namespace + subjectType + subjectId + change to webhook source + eventKey + resource + merchant scope
  -> begin webhook DB transaction
  -> query active subscriptions by merchantId + eventKey
  -> no matches:
       commit without persistence
       complete inbound message
  -> matches:
       build endpoint and payload snapshots
       create PENDING webhook_delivery per subscription
       commit
       complete inbound message
```

Persistence failure must leave the inbound message eligible for broker redelivery. If persistence commits but message completion fails, redelivery relies on delivery composite uniqueness.

## Critical Decisions

- `source` 由 event envelope `namespace` 映射，表示 producer service / bounded context，不與 `eventKey` 或 `resourceType` 混用。
- Delivery 保存 `source`、`eventKey`、`resourceType`、`resourceIdentifier`、`merchantId`、`occurredAt`、endpoint 與 payload snapshot。
- Delivery `occurredAt` 與 outbound payload `occurredAt` 必須完全 follow `payload.change.occurredAt`；缺失或 malformed 屬 inbound contract violation。
- Matching 必須透過 `webhook_subscription_event_type` relation，並排除 soft-deleted subscriptions。
- `subjectType + change.status` 無法映射到 Webhook catalog event key 時視為 non-webhook event：不建立 delivery、不查 subscription，記錄必要 metric / debug 後 complete message。
- 第一版以 `source + eventKey + resourceType + resourceIdentifier + subscription` 防止重複 delivery。
- 複合冪等假設同一 resource 的同一 event key 只發生一次。
- 沒有 matching subscription 時不建立 receipt；redelivery 會依當下 subscription 重新 matching。
- Payload snapshot 在 delivery creation 時完成；Stage 3 post-commit publisher 與 Stage 4 worker 不重新查交易現況組 payload。
- Stable producer `eventId`、deferred inbox、delivery job publish integration 都不屬於本 stage。

## Plan Inventory

- Inbound adapter and contract mapping：接收與驗證 domain event，映射為 Webhook application input。
- Delivery persistence and repository capability：建立 delivery backing、來源／資源追蹤與複合唯一性。
- Event handling use case：在同一 consistency boundary 內 matching subscriptions、建立 delivery snapshots。
- Targeted verification：驗證 matching、no-subscriber、duplicate redelivery、rollback 與 snapshot semantics。

## Validation Direction

Stage 2 完成時應能證明：

```text
valid inbound event + matching subscriptions
  -> one PENDING delivery per subscription
  -> source/resource/payload/endpoint snapshots persisted
  -> inbound message completed after commit
```

以及：

```text
duplicate inbound event
  -> no duplicate delivery for the same subscription
```

以及：

```text
unsupported inbound subject/change
  -> no persistence record
  -> inbound message completed
```

以及：

```text
inbound event + no matching subscription
  -> no persistence record
  -> inbound message completed
```

## Estimate

| Item | 估時 |
| --- | ---: |
| Webhook inbound adapter and mapping | 1 天 |
| Delivery persistence and composite idempotency | 1 天 |
| Subscription matching and snapshot creation | 1 天 |
| Targeted verification and broker-boundary checks | 1 天 |

總估時：4 天。不包含 Fund producer、新 broker infrastructure 或 Stage 3 delivery job publish integration 的建置。

## Contract And Runtime Placement

Stage 2 的 inbound event contract 與 Azure SB runtime 不應落在同一層。Blueprint 採以下 placement direction：

| Concern | Landing | Rationale |
| --- | --- | --- |
| Fund / Webhook shared event contract | `contract-event` | Cross-service event envelope、event declaration 與 payload contract 必須由 producer / consumer 共同引用。 |
| Fund domain raw subject | `contract-base` | `Deposit` 與 `WithdrawalIntent` 已是 Fund raw object；event payload `subject` 直接引用這些 raw types。 |
| Azure SB SDK adapter / Nest module / session receiver runtime | `service-kit` | Runtime transport、publisher / consumer module、session receiver lifecycle、settlement policy、logging / metrics 需集中治理，避免各 app 各自初始化 Azure SDK。 |
| Webhook application mapping and delivery creation | Webhook service code | `subjectType + change.status -> eventKey`、merchant matching、delivery snapshot creation 屬 Webhook capability。 |

`contract-event` 目前既有 `EventEnvelope<TPayload>` shape 是 `{ id, occurredAt, payload }`。Webhook inbound contract 需要 transport-visible `namespace`、`subjectType`、`subjectId` 與 `eventType`。Stage 2 plan 需先決定是擴充 / 新增 broadcast envelope type，或調整 topic-derived metadata 與 payload shape 的分層；不得把這些 shared contract fields 定義在 Webhook service local code。

Azure SB 引入方向是 service-kit package-level capability：Webhook app 應 import service-kit 提供的 Azure SB module / consumer abstraction，宣告 topic、subscription、session requirement 與 handler，不應在各 app 內重複 `ServiceBusClient`、`acceptNextSession`、`completeMessage`、`abandonMessage` 與 shutdown lifecycle。

## Pattern Gaps

- Codebase 尚無 Fund domain event definition；Webhook plan 需確認 consumer-side shared contract landing，但不得擴張到 Fund publisher implementation。
- Webhook target codebase 尚未確認 Azure Service Bus precedent；外部 sample 已示範 Azure SDK session receiver、manual completion、handler success complete / failure abandon 的 baseline，但正式 adapter landing、provider wiring、session concurrency、shutdown lifecycle、logging / metrics 與 retry / DLQ policy 仍由 Stage 2 plan 定案。
- 若 service-kit 尚未具備 session-aware Azure SB consumer abstraction，broker adapter establishment 需另行估算並先取得 infrastructure decision。

## Open Points

- `contract-event` broadcast envelope type shape and shared contract landing。
- `service-kit` Azure SB module / consumer abstraction and Webhook broker adapter wiring。

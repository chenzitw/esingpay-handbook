---
status: draft
updated_at: 2026-06-22
updated_by: Codex
---

# Webhook 交易事件推播 — Stage 2 Blueprint

## Goal

Stage 2 完成 Webhook inbound domain event consumption、subscription matching 與 delivery creation。Webhook 從既有 event transport 接收 Fund transaction event，直接建立 matching deliveries；第一版不建立 webhook outbox 或 inbox。

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
- Delivery execution job publishing。
- HTTP POST、signing 與 delivery retry。
- Deferred inbox migration。

## Inputs

- Persistence model：[`../design/design-persistence-model.md`](../design/design-persistence-model.md)。
- Queue / event topology：[`../design/design-queue-topology.md`](../design/design-queue-topology.md)。
- Service boundary：[`../design/design-service-boundary.md`](../design/design-service-boundary.md)。
- Type boundary：[`../design/design-type-contract.md`](../design/design-type-contract.md)。
- Payload contract：[`../design/design-payload-contract.md`](../design/design-payload-contract.md)。
- Stage 1 landed subscription persistence、binding relation 與 code-defined event catalog。

## External Dependency

Stage 2 假設 Fund transaction domain event 已可由既有 transport 送達 Webhook。Blueprint 只鎖定 Webhook consumer 所需的 input semantics，不規劃或估算 producer implementation。

Consumer 必須取得：

- `source`，第一版為 `fund`。
- `event_type`。
- `merchant_id`。
- `resource_type`，第一版為 `deposit` 或 `withdrawal`。
- `resource_identifier`。
- `occurred_at`。
- Payload builder 所需 domain raw。

實際 transport envelope、shared contract landing 與 broker wiring 由 Stage 2 codebase plan 依 event adapter convention 定案，但不得將 Fund publisher implementation 納入 Webhook plan。

## Consumption Flow

```text
webhook inbound event consumer
  -> validate source + event type + resource + payload
  -> begin webhook DB transaction
  -> query active subscriptions by merchant_id + event_type
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

- `source` 表示 producer service / bounded context，不與 `event_type` 或 `resource_type` 混用。
- Delivery 保存 `source`、`event_type`、`resource_type`、`resource_identifier`、`merchant_id`、`occurred_at`、endpoint 與 payload snapshot。
- Matching 必須透過 `webhook_subscription_event_type` relation，並排除 soft-deleted subscriptions。
- 第一版以 `source + event_type + resource_type + resource_identifier + subscription` 防止重複 delivery。
- 複合冪等假設同一 resource 的同一 event type 只發生一次。
- 沒有 matching subscription 時不建立 receipt；redelivery 會依當下 subscription 重新 matching。
- Payload snapshot 在 delivery creation 時完成；Stage 3 publisher 與 Stage 4 worker 不重新查交易現況組 payload。
- Stable producer `event_id` 與 deferred inbox 都不屬於本 stage。

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

總估時：4 天。不包含 Fund producer 或新 broker infrastructure 的建置。

## Pattern Gaps

- Codebase 尚無 Fund domain event definition；Webhook plan 需確認 consumer-side shared contract landing，但不得擴張到 Fund publisher implementation。
- Codebase 尚未找到 Azure Service Bus precedent；若既有 transport 無法直接承載此 consumer，broker adapter establishment 需另行估算並先取得 infrastructure decision。

## Open Points

- Webhook consumer 實際接入的既有 topic / subscription 與 transport envelope。
- Domain raw 是完整放入 event，或由已存在的 consumer contract 提供足夠 payload building data。
- `occurred_at` 的 authoritative field。

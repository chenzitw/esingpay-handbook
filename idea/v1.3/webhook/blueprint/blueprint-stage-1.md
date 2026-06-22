---
status: draft
updated_at: 2026-06-22
updated_by: Codex
---

# Webhook 交易事件推播 — Stage 1 Blueprint

## Goal

Stage 1 建立 webhook subscription management 與 code-defined webhook event type catalog 的第一版 backend 基礎。完成後，merchant / platform management surface 可透過 api-gateway 管理 merchant webhook subscriptions，後續 Stage 2-4 可依同一份 event catalog 與 subscription relation 消費 inbound event、match subscription 並建立 delivery。

本 blueprint 只定義 Stage 1 的 phase 拆分、跨 phase 決策、依賴與驗收方向。所有 codebase-specific landing details 由 codebase plan set 負責。

## Inputs

- REST contract semantics：[`../design/design-rest.md`](../design/design-rest.md)。
- Error semantics：[`../design/design-error-contract.md`](../design/design-error-contract.md)。
- Management surface：[`../design/design-management-surface.md`](../design/design-management-surface.md)。
- Management transport boundary：[`../design/design-rpc.md`](../design/design-rpc.md)。
- Persistence model：[`../design/design-persistence-model.md`](../design/design-persistence-model.md)。
- Service boundary：[`../design/design-service-boundary.md`](../design/design-service-boundary.md)。
- Type boundary：[`../design/design-type-contract.md`](../design/design-type-contract.md)。

## Codebase Plan Authority

Stage 1 的落地權威在 `<codebase>/plan/v1.3/webhook`：

- `context-progress.md` owns current implementation status, landed facts and frozen decisions.
- `plan-stage-1-phase-1.md` through `plan-stage-1-phase-7.md` own concrete implementation specifications, verification details and handoff checklist.

若 handbook design / blueprint 與 codebase plan 出現 implementation wording drift，以 plan 記錄的 codebase convention 為準；若 drift 反映的是概念或 scope 變更，回到 design / blueprint 層同步。

## Scope

In scope：

- Webhook subscription persistence backing and subscription-event binding relation.
- Code-defined webhook event type catalog；不建 event type DB table，不開放 UI CRUD。
- Merchant / platform account surfaces for event type lookup and subscription list / view / create / update / delete.
- Gateway-to-webhook management transport using the existing REST RPC proxy pattern.
- Merchant ownership rules and platform management scope rules.
- Event key validation, endpoint URL validation, soft delete and binding replacement semantics.
- Backend verification and frontend handoff notes.

Out of scope：

- Inbound event consumption and delivery creation.
- Delivery publisher / delivery worker / recovery scheduler.
- Delivery history and manual resend.
- Signature generation / verification docs.
- Endpoint ownership verification.
- Secret rotation.
- Frontend implementation; this requires a separate frontend plan after the target frontend repo is confirmed.

## Stage Decisions

- Event type catalog is code-defined and shared by event type read, subscription validation and later inbound consumer matching.
- `deposit.blocked` is part of the first supported catalog.
- Subscription can exist with zero event bindings; empty `eventKeys` means keep the endpoint but subscribe to no events.
- Duplicate endpoint URL is allowed for the same merchant in Stage 1.
- Subscription delete is soft delete. Deleted subscriptions do not appear in management reads and do not participate in later inbound consumer matching.
- Subscription management read models compose `eventTypeCount` and `eventTypes` from binding + catalog; these are not subscription entity fields.
- Gateway owns account authentication and account-surface permission enforcement. Webhook management capability trusts the gateway-provided identity and target merchant scope.
- Merchant surface operations are scoped to the authenticated merchant.
- Platform surface manages merchant webhook subscriptions: list can accept target merchant filters, create requires a target merchant, and view / update / delete operate on active subscription id.
- Management transport is `contract-rest + RestRpc` for Stage 1. Do not introduce a public `contract-rpc/webhook` namespace for subscription management.
- REST RPC route keys remain split by account surface using `rest.webhook.merch-*` and `rest.webhook.plat-*` naming anchors.
- Permission concepts are split by account surface: merchant uses `merch:webhooks:*`; platform uses `plat:merch_webhooks:*`.
- Subscription API ids use the platform short id string representation. Persistence id primitive and codec use are codebase plan concerns.
- REST list envelope, paging query parameter names and error enum primitive follow codebase REST convention; design only fixes their semantics.

## Phase Breakdown

### Phase 1: Boundary And Codebase Convention

Purpose：settle how Stage 1 lands in the codebase before schema, contract and adapters are split across later plans.

Blueprint-level outputs：

- Confirm the new webhook capability boundary and account-surface management transport.
- Confirm that Stage 1 uses `contract-rest + RestRpc`, not a separate public contract-rpc namespace.
- Confirm account-surface route-key vocabulary, permission vocabulary, merchant/platform scope rules and frontend scope boundary.
- Record codebase convention decisions in the plan set so later phases do not rediscover them.

Plan-owned details：concrete codebase landing and convention instantiation.

### Phase 2: Persistence Foundation And Event Catalog

Purpose：establish the Stage 1 persistence backing and catalog source needed by read/write APIs and later inbound consumer matching.

Blueprint-level outputs：

- Subscription persistence backing exists conceptually.
- Subscription-event binding relation exists conceptually and prevents duplicate binding of the same event key for one subscription.
- Event type catalog is code-defined and includes the first-version event keys.
- Query support requirements from [`../design/design-persistence-model.md`](../design/design-persistence-model.md) are assigned to the plan.

Plan-owned details：concrete persistence implementation and data-access surface.

### Phase 3: Event Type Read And Validation Capability

Purpose：make the event catalog usable by management UI options and subscription write validation.

Blueprint-level outputs：

- Merchant and platform surfaces can read the supported event type options.
- Subscription write validation can reject unsupported event keys.
- Event type options expose stable key, display label and ordering semantics from the catalog.

Plan-owned details：concrete REST contract implementation, validation placement and targeted tests.

### Phase 4: Subscription Read Surface And Scope Baseline

Purpose：establish read-side management behavior before write-side mutation and transaction boundaries.

Blueprint-level outputs：

- Merchant list / detail reads enforce authenticated merchant ownership.
- Platform list / detail reads follow the approved platform scope semantics.
- Subscription summaries and details are composed read models, not direct persistence records.
- Short id parsing failure, not-found, paging failure and merchant scope failure map to stable error semantics.

Plan-owned details：concrete read-side implementation, paging convention and response envelope.

### Phase 5: Subscription Write Surface And Consistency Boundary

Purpose：complete subscription create / update / delete behavior after catalog validation and read-side scope are stable.

Blueprint-level outputs：

- Create validates endpoint URL and event keys, creates subscription and event bindings in one consistency boundary.
- Update replaces mutable endpoint URL and full event binding set in one consistency boundary.
- Delete soft-deletes subscription and leaves historical binding data available unless a later cleanup requirement is approved.
- Merchant write operations enforce authenticated merchant ownership.
- Platform write operations follow the approved platform target merchant / active subscription semantics.

Plan-owned details：concrete write-side implementation, transaction mechanism and exact error mapping.

### Phase 6: Backend Verification

Purpose：verify Stage 1 backend behavior before it becomes the foundation for inbound consumption、delivery creation and execution stages.

Blueprint-level outputs：

- Event catalog, validation, read/write behavior, merchant/platform scope, soft delete and duplicate endpoint semantics are verified.
- DB-dependent checks follow the codebase convention for integration test availability.
- Frontend implementation remains out of scope for this backend workspace.

Plan-owned details：test file names, unit vs integration split, smoke commands, self-skip mechanics and exact case list.

### Phase 7: Handoff And Design Sync

Purpose：close Stage 1 planning and make implementation / follow-up work resumable.

Blueprint-level outputs：

- Consolidate approval gates and non-goals.
- Record any design sync items created by codebase convention discoveries.
- Provide frontend handoff notes for a later frontend plan.
- Keep Stage 2-4 prerequisites visible without expanding their implementation scope.

Plan-owned details：current implementation snapshot, final checklist, approval gate table and codebase-specific drift notes.

## Sequencing

- Phase 1 comes first because it freezes codebase convention decisions that affect every later phase.
- Phase 2 precedes Phase 3 and Phase 4 because event catalog and subscription persistence backing are shared prerequisites.
- Phase 3 and Phase 4 can proceed after Phase 2 is stable; Phase 3 supplies validation capability required by write-side work, while Phase 4 proves read-side scope and transport behavior.
- Phase 5 depends on Phase 3 event key validation and Phase 4 scope/error baselines.
- Phase 6 verifies the complete Stage 1 backend surface after read/write behavior is available.
- Phase 7 closes planning and records handoff/design-sync items. It should not add runtime behavior.

Parallelism：

- Frontend planning can start once backend contract semantics are stable, but frontend implementation is not part of this backend Stage 1 plan set.
- Stage 2 planning can begin after event catalog and subscription persistence semantics are stable, but implementation should wait for Stage 1 backend behavior that Stage 2 depends on.

## Validation Direction

Stage 1 is ready to support Stage 2 when:

- Event type catalog is code-defined and readable through both account surfaces.
- Merchant and platform subscription management APIs expose list / view / create / update / delete semantics.
- Merchant ownership and platform scope behavior match the Stage Decisions.
- Subscription read models include event type summary/detail data derived from bindings and catalog.
- Create / update / delete honor validation, duplicate endpoint, empty event keys and soft delete semantics.
- Codebase plan records any envelope, paging, error enum and permission implementation differences from handbook wording.

## Open Points

- Frontend implementation target is not identified in the current backend workspace and requires a separate plan.
- Subscription list does not include latest delivery state in Stage 1; adding it requires a later design extension.

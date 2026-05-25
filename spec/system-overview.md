---
status: active
updated_at: 2026-04-28
updated_by: Tim
remark: 本文件未依照 spec 規範撰寫，暫供內容參考，等待後續重構改寫。
---

# System Overview

## 1. Purpose

This document defines the high-level system model for the Fund, Network, and Wallet domains.

The core boundary is:

> **Fund owns business orchestration. Network owns external transaction facts. Wallet owns accounting truth. Ledger/Cashflow read models own user-facing reflection.**

The system separates:

- **business intent** from **external transaction fact**
- **accounting truth** from **user-facing display truth**
- **fund orchestration** from **wallet execution**
- **posted balance effects** from **processing / failed / cancelled display traces**

This overview focuses on domain responsibility, causality, and canonical flow behavior. It intentionally avoids table-level schema, DTO field completeness, and projector implementation details.

---

## 2. Domain Responsibility Map

## 2.1 Fund

Fund owns business aggregates and user/business-facing lifecycle decisions.

Examples:

- `Deposit`
- `WithdrawalIntent`
- `TransferIntent`

Fund is responsible for:

- creating and validating business intent
- deciding business status transitions
- coordinating Wallet allocation lifecycle calls
- coordinating Network instruction or recognition
- managing multi-step or multi-leg business workflows
- handling business failure, rejection, cancellation, or retry semantics

Fund does **not** own balance mutation.

Fund does **not** directly write wallet accounting facts.

---

## 2.2 Network

Network owns external transaction facts.

Examples:

- blockchain transaction
- gateway transaction
- bank transfer record
- provider-side transaction status

Network is responsible for:

- recognizing external inbound facts
- instructing outbound network transactions
- tracking provider/network status
- emitting milestone facts to Fund

Network does **not** decide business accounting meaning.

Network does **not** mutate Wallet balances.

---

## 2.3 Wallet

Wallet owns accounting truth and balance invariants.

Wallet is responsible for:

- `WalletAllocation` lifecycle
- `WalletEntry` / `WalletEntryLine` creation
- `WalletAssetBalance` mutation
- atomic balance check and mutation
- reservation / settlement / reversal accounting facts
- protecting balance invariants under concurrency

Wallet does **not** own business workflow meaning beyond the allocation contract.

Wallet does **not** decide user-facing display lifecycle directly inside the allocation transaction.

---

## 3. Wallet Model Layers

Wallet contains several distinct layers. These layers must not be collapsed.

## 3.1 WalletAllocation: allocation case record

`WalletAllocation` is the Wallet-side allocation case record for a fund-driven accounting plan.

It records:

- flow family
- lifecycle status
- fund case reference
- allocation type
- plan lines (intended allocation snapshot)
- outcome lines (final result snapshot)
- reservation deadline when applicable

Allocation uses two groups of lines to describe the full case lifecycle:

- **planLines**: the intended allocation snapshot submitted at reserve / propose time. Serves as the envelope boundary for settle-time validation. Null for direct-settle (born-realized, no plan stage).
- **outcomeLines**: the final result snapshot recorded when the allocation reaches a terminal state via settle. Includes successful settlement, partial realized cost (e.g. network fee consumed but principal failed), or full-amount settlement. Null for released / expired allocations (projection derives "all zero" from absence of outcome).

These form a three-layer data chain:

```text
planLines → outcomeLines → entryLines
```

- `planLines` record intended economic effects (plan / envelope). They should not include zero-amount placeholder lines; absence of a line kind means there is no intended economic effect for that kind.
- `outcomeLines` record the final economic outcome of the case
- `WalletEntryLine` records the accounting facts derived from outcome lines

`outcomeLines` are not a duplicate of `WalletEntryLine`. The former is the upstream outcome snapshot (what the case resolved to); the latter is the downstream accounting effect (what Wallet posted as accounting facts, including bucket mapping and reversal splits).

`WalletAllocation` is the coordination object for creating accounting facts and triggering downstream read-model sideflows.

---

## 3.2 WalletEntry / WalletEntryLine: accounting truth

`WalletEntry` and `WalletEntryLine` are posted-only accounting facts.

They are the only Wallet records that directly drive `WalletAssetBalance`.

Key rules:

- `WalletEntry` is insert-once immutable.
- `WalletEntryLine` is insert-once immutable.
- Entry creation immediately affects balance in the same transaction.
- There is no provisional entry status.
- There is no void entry status.
- There is no entry update path in normal operation.
- Failed / cancelled / processing user-facing traces are not represented by fake entry lines.

`WalletEntryLine` is the source of truth for balance mutation, audit, fact ordering, and balance rebuild.

---

## 3.3 WalletAssetBalance: operational balance projection

`WalletAssetBalance` is the operational materialized balance for a wallet asset.

It is driven by posted `WalletEntryLine` records.

Normal write path rule:

> `WalletAllocation`, `WalletEntry` / `WalletEntryLine`, and `WalletAssetBalance` mutation are handled in the same database transaction.

`lastAppliedEntryLineId` is a reconciliation / rebuild watermark. It is not an eventual-consistency catch-up mechanism in the normal command path.

Wallet uses ordered locking on affected balance rows to preserve atomic balance check and mutation.

---

## 3.4 WalletLedgerItem: user-facing ledger detail

`WalletLedgerItem` is the user-facing ledger detail read model.

It represents **business-aware bucket effects** for users: each row is anchored in wallet bucket movement, but may also carry business context such as external source/destination or network transaction references when relevant.

Its basic row grain is:

> one business-aware bucket effect for a specific allocation / fund case, wallet, bucket, currency, and kind.

A single allocation or fund case may produce multiple ledger rows. Ledger rows do not promise a 1:1 mapping with `WalletEntryLine` or `WalletAllocationLine`.

It exists because the user-facing ledger is not identical to the internal accounting ledger:

- it shows `available` / `held` user-facing movement
- it hides `locked` / `held_locked` reservation transitions
- it can show processing / failed / invalid rows
- it can represent user-facing outcomes that do not directly affect balance

`WalletLedgerItem` is the source for ledger list / pagination.

It is not directly mapped from `WalletEntryLine`.

It is driven by allocation lifecycle and downstream sideflow projection.

Ledger row visibility is enforced by query/use-case policy. For example, merchant users may only see merchant-visible buckets, while platform users may see platform treasury buckets and platform-owned held buckets. This visibility rule is not part of the accounting fact model.

---

## 3.5 WalletCashflowItem: user-facing asset movement summary

`WalletCashflowItem` is the user-facing cashflow read model.

It is closer to a bank passbook concept than to a bucket ledger.

It summarizes asset movement by fund case, allocation, wallet, and currency.

Typical behavior:

- principal and same-currency service fee are combined into one primary cashflow item
- different-currency network fee becomes a separate network-fee cashflow item
- the item exposes `direction`, `amountGross`, `amountNet`, and `amountFee`
- it does not need a generic signed `amountDelta`, because passbook semantics are better represented by direction plus gross/net/fee breakdown
- the item can be valid or invalid independently of fund case status

`WalletCashflowItem` is grouped by `(allocationId, walletId, currencyCode, category)` in current canonical flows. This is a current-flow invariant, not a universal claim for all future transfer variants.

Cashflow items primarily follow fund case and allocation, but may include compact network transaction context, including transaction identifier, source, and destination, when needed for display or traceability.

A cashflow item may use a category such as `primary` or `network_fee`:

- `primary`: principal plus same-currency service fee folded into one passbook item
- `network_fee`: different-currency network fee represented as an independent passbook item

`WalletCashflowItem` is also a post-commit sideflow resource, not part of the Wallet Allocation transaction.

---

## 4. Allocation Flow Families

Wallet Allocation supports two flow families.

## 4.1 Reservation flow

Reservation flow is used when internal funds must be controlled before the fund case becomes fully effective.

Typical case:

- `WithdrawalIntent`

Lifecycle:

```text
reserve -> bind -> settle
        -> release
        -> expire
```

Important semantics:

- `reserve` locks internal funds.
- `reserve` does not create user-facing ledger/cashflow rows.
- `bind` attaches a real fund case identifier.
- user-facing sideflow begins only after the allocation is bound.
- `settle` creates posted settlement / reversal accounting facts. Settle is used when the business has completed its process — including failure cases where partial costs (e.g. network fee) were consumed.
- `release` actively terminates a known allocation that should not continue. Release is used only when the business entity was never successfully created (e.g. Fund failed to persist the business aggregate after reserve).
- `expire` passively cleans up unbound reserved allocations after `reservedUntil`.

`reserved` allocations have no final fund case identity and may expire.

`bound` allocations have a real fund case identity and do not expire as ghost reservations.

---

## 4.2 Direct flow

Direct flow is used when the fund case already exists before Wallet allocation begins, and no prior reservation control is needed.

Typical case:

- `Deposit`

Two variants exist.

### Direct pending

Used when the external fact is known but not final.

Lifecycle:

```text
propose -> settle
        -> release
```

Semantics:

- `propose` creates a bound allocation.
- `propose` does not create `WalletEntry`.
- `propose` can trigger post-commit ledger/cashflow sideflow.
- `settle` creates posted accounting facts and updates balance.
- `release` finalizes the allocation as not continuing, without posted entry if no accounting effect occurred.

### Direct immediate

Used when the external fact is already final.

Lifecycle:

```text
direct-settle
```

Semantics:

- allocation is created settled
- posted settlement entry is created immediately
- balance is updated in the same transaction
- ledger/cashflow sideflow is triggered after commit

---

## 5. Accounting Fact Model

## 5.1 Entry phases

`WalletEntry` uses allocation phase to describe the accounting role of an entry.

Typical phases:

- `reservation`
- `settlement`
- `reversal`

`allocationPhase` belongs to the accounting fact. It is not a user-facing display status.

---

## 5.2 Reservation accounting

Reservation entries move funds from liquid buckets into locked buckets.

Examples:

- `available -> locked`
- `held -> held_locked`

Reservation entries are posted accounting facts and affect balance.

They are internal accounting control facts.

They are normally hidden from user-facing ledger/cashflow views.

---

## 5.3 Settlement accounting

Settlement entries represent final effective economic effects.

Examples:

- deposit credit into `available`
- service fee movement into `held`
- withdrawal principal leaving `available`
- network fee leaving fee payer wallet

Settlement entries affect balance immediately.

---

## 5.4 Reversal accounting

Reversal entries undo prior reservation control when funds are released back.

Examples:

- `locked -> available`
- `held_locked -> held`

Reversal entries are posted accounting facts.

They are internal accounting facts and may be hidden from user-facing ledger views depending on display rules.

---

## 5.5 Same-entry self-funding

A single posted entry may contain lines whose net effect within the same wallet/bucket/currency is computed together.

Validation is based on per-entry net effect:

> For each `(walletId, bucket, currencyCode)`, pre-existing balance plus net entry effect must not become negative.

This supports display-driven accounting facts such as:

```text
available +500
available -10
held +10
```

without requiring fake intermediate entries or internal execution ordering.

---

## 6. User-Facing Read Model Principles

## 6.1 Post-commit sideflow boundary

`WalletLedgerItem` and `WalletCashflowItem` are post-commit sideflow resources.

Wallet Allocation transaction handles only:

- allocation state
- entry / entry lines
- asset balance

Ledger and cashflow read models are updated by downstream sideflow.

Sideflow failure must not roll back Wallet Allocation transaction.

Sideflow projectors must be retryable and idempotent.

---

## 6.2 Row lifecycle

`WalletLedgerItem` and `WalletCashflowItem` use their own read-model row lifecycle.

Each row has a persisted state axis:

```text
state: active | superseded
```

`state` describes whether the row is the current visible version.

- `active`: included in default user-facing list
- `superseded`: retained for audit/debug, excluded from default list

A row becomes `superseded` only when a newer row from the same source is created to replace it. Typically, plan-based rows become superseded when outcome-based rows are projected.

Read model rows record plan amounts and outcome amounts (when available) as relatively stable facts. They do not persist a `validity` field.

**Validity is derived at the DTO layer.** The REST DTO mapper computes validity by comparing plan and outcome amounts per kind:

- outcome amount > 0 for a given kind → valid
- outcome amount = 0 or outcome lines absent → invalid
- fund case failure does not imply all wallet-facing items are invalid; validity is per item / per kind

This separation keeps read model rows stable while allowing UI presentation logic to evolve freely.

---

## 6.3 Versioned row replacement

Read model rows are append-oriented.

When a newer row from the same source should replace an older visible row:

- the old row becomes `superseded`
- the new row becomes `active`
- the new row appears according to its own ordering timestamp
- old rows remain for audit/debug

For current canonical flows, Ledger item source identity can be treated as:

```text
allocationId + walletId + bucket + currencyCode + kind
```

For Cashflow items, source identity can be treated as:

```text
allocationId + walletId + currencyCode + category
```

These identities define which active row should be superseded when a newer row from the same source is projected.

`updatedAt` on read-model rows can be used to track supersession time.

Default list ordering may use natural increasing row identity, typically `id DESC`, because read-model rows are append-oriented and newer projected rows should appear later / above older rows.

`effectiveAt` should still be kept as the semantic time of the displayed effect. For processing rows, it follows the allocation lifecycle time that produced the display version, such as bind or propose time. For final rows, it follows the allocation lifecycle time that produced the final display version, such as settle or release time.

The business payload of a read-model row is immutable after creation.

---

## 6.4 Relation to fund case and allocation

Read-model rows should preserve enough relation information to support business display and future transfer expansion.

At minimum:

- fund case type
- fund case identifier
- allocation id
- wallet id
- bucket
- currency code
- `kind` for Ledger items, or `category` for Cashflow items

Ledger DTO / composed rows may also carry display context such as merchant / wallet display information, `space`, derived `ownerType`, bucket, source, destination, and a compact network transaction reference when the row is directly related to an external network effect. These fields belong to the user-facing ledger read model or composed DTO shape; the overview does not require all of them to be stored directly on the persistence row.

This supports future `TransferIntent` patterns where one fund case may map to multiple allocations.

A separate leg identifier is not required in the initial model.

---

## 7. Canonical Flow: Deposit

## 7.1 Deposit pending

Scenario:

- Network recognizes inbound transaction.
- Transaction is not final yet.
- Fund creates `Deposit` in processing/transacting state.
- Fund asks Wallet to `propose` direct allocation.

Wallet behavior:

- create bound direct allocation
- create no WalletEntry yet
- trigger post-commit ledger/cashflow sideflow

Read-model reflection:

- ledger/cashflow rows may show active valid expected movement
- no balance has changed yet
- no WalletEntryLine exists yet

---

## 7.2 Deposit settled

Scenario:

- Network transaction becomes final.
- Fund marks deposit completed.
- Fund asks Wallet to settle allocation.

Wallet behavior:

- create posted settlement entry
- create entry lines
- update balance in same transaction
- trigger post-commit ledger/cashflow sideflow

Example accounting effect for 500 USDT deposit with 10 USDT service fee:

```text
available +500
available -10
held +10
```

Read-model reflection:

- previous plan-based expected rows become superseded
- new active outcome-based rows are appended
- cashflow item summarizes deposit amount by allocation, wallet, and currency

---

## 7.3 Deposit failed

Scenario:

- Network transaction fails or is rejected.
- Fund marks deposit failed.
- The business process has completed (with a failure result).
- Fund asks Wallet to settle allocation with outcome lines reflecting the failure: all amounts are zero.

Wallet behavior:

- settle allocation with outcomeLines where all amounts are zero
- no posted entry is created (no accounting effect — propose did not lock funds)
- balance remains unchanged
- trigger post-commit ledger/cashflow sideflow

Read-model reflection:

- previous plan-based expected rows become superseded
- new active outcome-based rows may be appended to show failed expected movement (validity derived at DTO layer)

---

## 8. Canonical Flow: Withdrawal Intent

## 8.1 Withdrawal reservation

Scenario:

- User or system creates withdrawal intent request.
- Fund must ensure funds can be reserved before fund case becomes effective.

Wallet behavior:

- `reserve` allocation
- create posted reservation entry
- update balance in same transaction
- no user-facing ledger/cashflow rows yet

Reason:

- allocation is not yet bound to a real fund case identifier
- locked bucket transitions are internal accounting control facts
- user-facing ledger hides locked / held_locked transitions

---

## 8.2 Withdrawal bind

Scenario:

- Fund successfully persists `WithdrawalIntent`.
- Fund binds allocation to the real fund case identifier.

Wallet behavior:

- allocation moves from `reserved` to `bound`
- no WalletEntry is created
- trigger post-commit ledger/cashflow sideflow

Read-model reflection:

- active plan-based expected rows may be appended
- user-facing ledger begins at bind, not reserve

---

## 8.3 Withdrawal settled

Scenario:

- Network transaction succeeds.
- Fund finalizes withdrawal as completed.
- Fund asks Wallet to settle allocation.

Wallet behavior:

- create posted settlement entry for actual effects
- create posted reversal entry if reserved remainder must be released
- update balance in same transaction
- trigger post-commit ledger/cashflow sideflow

Read-model reflection:

- previous plan-based expected rows become superseded
- active outcome-based final rows are appended
- ledger principal row may include external destination and network transaction reference when tied to an external transfer
- network-fee row follows the actual fee payer wallet and may include network transaction reference when the fee is consumed by the external network
- cashflow rows summarize by allocation, wallet, and currency, and may include network transaction context when useful for display or traceability

---

## 8.4 Withdrawal failed without consumed network fee

Scenario:

- Withdrawal is rejected, cancelled, or network transaction fails before any network fee is consumed.
- The business process has completed (with a failure result).
- Fund asks Wallet to settle allocation with outcome lines reflecting the failure: all amounts are zero.

Wallet behavior:

- settle allocation with outcomeLines where principal=0, service_fee=0, network_fee=0
- create posted reversal entry to unlock all previously reserved funds
- update balance in same transaction
- trigger post-commit ledger/cashflow sideflow

Read-model reflection:

- plan-based expected rows become superseded
- new active outcome-based rows are appended with zero outcome amounts (validity derived at DTO layer — all items invalid)

---

## 8.5 Withdrawal failed with consumed network fee

This is a canonical stress case.

Scenario:

- Withdrawal fund case fails overall.
- Principal transfer does not complete.
- Service fee should not be retained.
- Network fee was already consumed on-chain.
- The business process has completed (with a failure result that includes partial cost).
- Fund asks Wallet to settle allocation with outcome lines reflecting the partial result.

Wallet behavior:

- settle allocation with outcomeLines where principal=0, service_fee=0, network_fee=actual consumed amount
- create posted settlement entry for consumed network fee
- create posted reversal entry for released remainder (principal + service_fee locked amounts fully reversed, network_fee remainder reversed)
- update balance in same transaction
- trigger post-commit ledger/cashflow sideflow

Read-model reflection:

A single failed fund case may produce mixed item validity (derived at DTO layer):

```text
USDT principal: invalid (plan -980 → outcome 0)
USDT service_fee: invalid (plan -20 → outcome 0)
TRX network_fee: valid (plan -30 → outcome -25)
```

Principal and network-fee ledger rows may reference a network transaction when they are directly related to external network effects. Service-fee rows usually do not, because service fee is an internal allocation / bucket movement rather than an external network effect.

This proves:

- item validity is per item / per kind, derived from plan vs outcome comparison
- fund case failure does not imply all wallet-facing rows are invalid
- network-fee rows follow actual economic effect, not fund-case-level status alone
- Ledger and Cashflow read models follow the same validity derivation principle
- settle is the correct terminal operation for completed business processes, including failures with partial costs

---

## 9. Canonical Flow: Transfer Intent

Transfer intent may involve:

- multiple source wallets to one destination wallet
- one source wallet to multiple destination wallets

Wallet does not need to become a multi-leg workflow engine.

Fund owns transfer orchestration.

Recommended model:

- `TransferIntent` owns transfer legs / subtasks
- each leg can create its own WalletAllocation
- one fund case may map to multiple allocations
- Wallet continues to execute one allocation at a time

If one reserve fails, Fund compensates previously reserved allocations by releasing them.

Wallet remains focused on a single allocation execution unit.

Ledger/Cashflow read models should preserve `fundCaseId + allocationId` so future transfer views can group or distinguish per allocation without forcing Wallet core to understand the whole transfer graph.

---

## 10. Design Boundaries

## 10.1 Wallet does not own business orchestration

Wallet should not know the full workflow of a transfer, withdrawal approval process, or deposit recognition process.

Wallet executes allocation lifecycle operations and enforces accounting invariants.

Business workflow belongs to Fund.

---

## 10.2 Entry does not own user-facing display lifecycle

`WalletEntry` and `WalletEntryLine` are accounting facts.

They do not represent:

- pending display state
- failed display state
- cancelled display state
- superseded display versions

These belong to read models.

---

## 10.3 Ledger/Cashflow do not own accounting truth

`WalletLedgerItem` and `WalletCashflowItem` are user-facing read models.

They may display invalid or failed items.

They do not drive balance.

They do not replace `WalletEntryLine` as accounting truth.

---

## 10.4 Allocation transaction does not include sideflow read models

The allocation unit of work includes:

- allocation state mutation
- entry creation
- balance mutation

It does not include persistence of ledger/cashflow read models.

Sideflows are triggered after commit.

Sideflow failure is retried separately and does not roll back accounting truth.

---

## 10.5 Settle vs release vs expire semantic boundary

The choice of terminal operation depends on business process completion, not on success/failure:

- **settle**: the business process has completed — whether successfully, partially, or with failure. Fund submits outcome lines reflecting the final result (including zero amounts for failed components). Settle is the correct terminal operation for any allocation whose fund case has run through its lifecycle.
- **release**: the business entity was never successfully created. For example, Fund reserved funds but failed to persist the business aggregate. Release undoes the reservation without recording an outcome.
- **expire**: the allocation was never bound to a fund case and exceeded its reservation deadline. Expire is a passive cleanup mechanism.

This means a "failed withdrawal" still walks through settle, not release — because the business process ran and produced a result (even if that result is "principal=0, network_fee=25").

---

## 11. Summary

The system separates three truths:

1. **Business truth**: owned by Fund
2. **External transaction truth**: owned by Network
3. **Accounting truth**: owned by Wallet Entry / Entry Line

User-facing truth is represented by read models:

- `WalletLedgerItem` for business-aware bucket-effect details
- `WalletCashflowItem` for asset-movement summaries

`WalletAllocation` connects business intent to Wallet accounting execution as an allocation case record, tracking both the intended plan and the final outcome.

Wallet Allocation is the orchestration boundary for accounting facts and sideflow triggers, but it does not collapse accounting truth and display truth into one model.

---
status: draft
updated_at: 2026-05-10
updated_by: Tim
---

# Wallet Basic Design

version: 2026-05-10

## Scope

This document defines the **basic Wallet resource model and REST surface**.

Included:

- Wallet
- WalletAsset
- WalletAssetBalance
- Network balance exposure on WalletAsset
- Wallet basic APIs

Excluded:

- Wallet Allocation
- Wallet Entry
- Wallet Entry Line

For ledger-driving flow and accounting semantics, see **Wallet Allocation Design**.

---

## Core Model

### Wallet

Represents an accounting wallet.

Key points:

- `Wallet` is the root container.
- `Wallet` binds exactly one `networkEndpointId`.
- Current design assumes one endpoint = one address = one chain context.
- Therefore a wallet only tracks assets valid under that endpoint context.

```ts
interface Wallet {
  id: bigint;
  type: WalletType;
  merchantId: Uuid | null;
  networkEndpointId: bigint;
  createdAt: Date;
  updatedAt: Date;
}
```

### WalletAsset

Represents one asset under one wallet, identified externally by `walletId + currencyCode`.

Key points:

- Raw model keeps surrogate `id`.
- API identity uses `walletId + currencyCode`.
- `merchantId` is intentionally duplicated for query/indexing needs.
- `nickname` is non-null and always has a default value.

```ts
interface WalletAsset {
  id: bigint;
  merchantId: Uuid | null;
  walletId: bigint;
  currencyCode: CurrencyCode;
  nickname: string;
  createdAt: Date;
  updatedAt: Date;
}
```

### WalletAssetBalance

Represents the **current balance projection** for one wallet asset.

Key points:

- One row per `walletId + currencyCode`.
- Derived from wallet entry lines, with `initialAmount` as the opening point for the available bucket.
- Uses bucket-based accounting:
  - `availableAmount`
  - `lockedAmount`
  - `heldAmount`
  - `heldLockedAmount`

- `initialAmount` is the opening balance for the available bucket - see "Opening Balance" section below. Server-internal audit field; not exposed in API.
- `lastAppliedEntryLineId` is the projection watermark; nullable when no entry line has been applied yet.
- `computedAt` is the projection refresh time.
- Current projection uses `walletId + currencyCode`, not `walletAssetId`.
- `WalletAssetBalance` is a mandatory companion row of `WalletAsset`. Any path that creates a `WalletAsset` must create the matching balance row in the same atomic boundary.

```ts
interface WalletAssetBalance {
  id: bigint;
  walletId: bigint;
  currencyCode: CurrencyCode;
  initialAmount: NumericString;
  availableAmount: NumericString;
  lockedAmount: NumericString;
  heldAmount: NumericString;
  heldLockedAmount: NumericString;
  lastAppliedEntryLineId: bigint | null;
  computedAt: Date;
  createdAt: Date;
  updatedAt: Date;
}
```

---

### Opening Balance

`WalletAssetBalance.initialAmount` records the opening point of the available bucket at the time of balance row creation.

Key points:

- Only `availableAmount` has an opening counterpart; `lockedAmount` / `heldAmount` / `heldLockedAmount` always start at `0` because they are produced by the reservation system, which has no state to inherit at creation time.
- `initialAmount` is **create-once immutable**. After the row is persisted, no path may mutate it.
- `initialAmount` is a **server-internal audit field** and is **not exposed in the API**. The DTO only surfaces the live bucket amounts plus `computedAt`.
- The audit identity holds at all times:

  ```text
  availableAmount == initialAmount + sum(entry-line available-bucket delta applied to this row)
  ```

This makes balance reconciliation explainable when entry-line totals do not match `availableAmount` directly - the residual is the opening contribution. Reconciliation tooling reads `initialAmount` directly from the database, not from the API.

Initialization sources:

- **Standard creation path** (forward fix `create-wallet-asset.use-case.ts`): `initialAmount = '0'`. New wallet assets start clean.
- **Backfill path** (admin migration use case): `initialAmount = networkBalance.amount` if a corresponding `NetworkBalance` row exists in the network domain at backfill time; `initialAmount = '0'` if it does not. Missing network-side data is tolerated as zero (workaround posture; see implementation plan for rationale).

Why initialAmount is needed:

Without this field, the reconciliation `sum(entry-line delta) == availableAmount` would fail silently for wallet assets whose ledger started above zero (backfilled assets). Recording the opening point makes the residual auditable instead of unexplained.

---

## Network Relationship

### NetworkEndpoint

`Wallet` binds one `NetworkEndpoint`.

Important consequence:

- `WalletAsset` does **not** need its own asset-id layer for chain distinction.
- Because endpoint already fixes the chain/address context, `currencyCode` is sufficient inside wallet domain for current design.
- Example: one wallet under a TRON endpoint tracks TRON-side USDT only.

### Network balance on WalletAsset

`WalletAssetDto` includes a `networkBalance` block.

This is allowed because:

- `WalletAssetDto` is a REST representation, not a raw entity shape.
- Top-level exposure of `networkEndpoint` and `networkBalance` is an intentional projection optimization.
- This does not change the raw ownership model (`Wallet -> NetworkEndpoint`).

### Non-null network balance invariant

`networkBalance` is non-null in API.

Assumption:

- successful `NetworkEndpoint` creation already guarantees at least one provider balance fetch
- if initial balance fetch fails, endpoint creation fails or emits explicit error

Therefore API does not model "uninitialized network balance" as nullable.

---

## DTO Rules

### WalletDto

Used when the primary resource is `Wallet`.

`WalletDto.assets` uses `WalletAssetItemDto[]` because assets are embedded in the wallet representation.

### WalletAssetItemDto

Use only when embedded inside `WalletDto.assets`.

### WalletAssetDto

Use whenever the API resource itself is `WalletAsset`, including:

- search across wallets
- list under `/wallets/:walletId/assets`
- get one asset under `/wallets/:walletId/assets/:currencyCode`
- update one asset
- refresh network balance

Path ancestry under `/wallets/:walletId/...` does **not** imply embedded DTO usage.
If the endpoint target is a wallet-asset resource/collection, return `WalletAssetDto`.

`WalletAssetDto` exposes two distinct balance blocks, each representing a different concept and source:

- `balance` - the wallet domain's own ledger projection. Bucket-based (4 buckets + `computedAt`), sourced from `WalletAssetBalance`. This is what the wallet ledger believes the balance to be. Non-null. The opening point (`initialAmount`) is intentionally not exposed.
- `networkBalance` - the network domain's mirror of on-chain state. Single-amount projection, sourced from `NetworkEndpointCurrencyBalance`. This is what the chain (as observed by the network domain's last sync) shows. Non-null.

The two are independent projections and may diverge - divergence is signal, not error. Do not derive one from the other in DTO mapping.

### WalletAssetBalanceDto

Expose bucket balances (`availableAmount`, `lockedAmount`, `heldAmount`, `heldLockedAmount`) and `computedAt`.
Do not expose `id` / `walletId` / `currencyCode` / `initialAmount` / `lastAppliedEntryLineId` / `createdAt` / `updatedAt` in API.

`initialAmount` is intentionally hidden from API consumers - it is a server-side reconciliation audit field, not a user-facing concept.

---

## REST Surface

### Wallet API

#### Get wallet

- `GET /wallet/plat/wallets/:walletId`
- returns `WalletDto`
- embeds `assets: WalletAssetItemDto[]`

### WalletAsset Search API

#### Search wallet assets across wallets

- `GET /wallet/plat/wallet-assets`
- paging search
- returns `OffsetPagingResultDto<CommonCode.Ok, WalletAssetDto[]>`
- endpoint name uses `searchXxx` by project convention

### WalletAsset API

#### List wallet assets under a wallet

- `GET /wallet/plat/wallets/:walletId/assets`
- returns `ResultDto<CommonCode.Ok, WalletAssetDto[]>`
- endpoint name uses `listXxx` by project convention (non-paging)

#### Get one wallet asset

- `GET /wallet/plat/wallets/:walletId/assets/:currencyCode`
- returns `ResultDto<CommonCode.Ok, WalletAssetDto>`

#### Update one wallet asset

- `PATCH /wallet/plat/wallets/:walletId/assets/:currencyCode`
- currently used for asset metadata such as nickname
- returns `ResultDto<CommonCode.Ok, WalletAssetDto>`

#### Refresh network balance

- `POST /wallet/plat/wallets/:walletId/assets/:currencyCode/$refresh-network-balance`
- triggers network-side balance refresh for the asset’s endpoint/currency
- returns `ResultDto<CommonCode.Ok, WalletAssetDto>` if refresh is synchronous and response contains refreshed representation
- if refresh later becomes async, this contract must be revisited

---

## Naming Conventions

Project-specific conventions used here:

- `searchXxx` = paging search
- `listXxx` = non-paging list
- `getXxxByYyyAndZzz` = object-target naming; `Yyy/Zzz` do not need explicit primary-key wording

Examples:

- `searchWalletAssets`
- `listWalletAssets`
- `getWalletAssetByWalletAndCurrency`
- `updateWalletAssetByWalletAndCurrency`
- `refreshWalletAssetNetworkBalanceByWalletAndCurrency`

---

## Current Decisions

- `WalletAsset` is the correct domain term; do not use `WalletCurrency`.
- Wallet domain uses `currencyCode` as asset discriminator under current wallet-endpoint model.
- `WalletAsset` keeps surrogate `id`, but API identity is `walletId + currencyCode`.
- `WalletAssetBalance` tracks current projection only.
- `WalletAssetDto` may elevate `networkEndpoint` to top-level representation.
- `WalletAssetDto.networkBalance` is non-null under endpoint-initialization invariant.
- `WalletAssetItemDto` exists only for embedded wallet view.
- `WalletAssetBalance` is a mandatory companion row of `WalletAsset`. Standard creation path creates both atomically; backfill path repairs missing rows for legacy assets.
- `WalletAssetBalance.initialAmount` is the opening point of the available bucket. Create-once immutable. Server-internal audit field; not exposed in API. Source depends on creation path.
- `WalletAssetDto.balance` and `WalletAssetDto.networkBalance` are independent projections from different domains; neither derives from the other.
- `WalletAssetItemDto` carries the same `balance` block as `WalletAssetDto` to keep embedded-view consistent with standalone-view.

---

## Future Plan

### WalletAssetBalanceSnapshot

Planned for future implementation.

Purpose:

- periodic weekly/monthly snapshot
- reporting / history / audit-style queries

Not part of current basic wallet implementation.

Current `WalletAssetBalance` remains the live current projection.

---

## Non-Goals

This document does not define:

- wallet allocation lifecycle
- reservation / settlement semantics
- wallet entry / line schema
- cashflow item / passbook projection
- ledger posting rules

See **Wallet Allocation Design** for those topics.

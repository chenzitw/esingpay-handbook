# Fund Case Expansion REST Contract Design

version: 2026-05-13

## Scope

本 spec 範圍：對應 `fund-case-expansion-design-draft.md` 落地的 REST contract 表面——request / response DTO 形狀、endpoint URL、error code surface。前端可以以此 spec 作為對齊 target，與後端實作平行進行。

涵蓋：

- `WithdrawalIntent` 既有 REST contract 因 category 引入而異動之處
- `TransferIntent` 全新 REST contract
- 兩個 fund case 的批次 submit endpoint contract
- 跨契約共用的 batch envelope 形狀

不涵蓋：

- Domain 層細節（raw / entity / 狀態機）——見 `fund-case-expansion-design-draft.md`
- Wallet RPC contract——本期不動
- Implementation 細節（controller / mapper / module wiring）——見對應 impl plan

---

## General Conventions

### Type mapping cheatsheet

Raw → REST DTO：

| Raw 型                               | REST DTO 型                  |
| ------------------------------------ | ---------------------------- |
| `bigint`                             | `ShortId`                    |
| `Date`                               | `IsoDateTimeUtc`             |
| `NumericString`                      | `NumericString`              |
| `Uuid`                               | `Uuid`                       |
| `CurrencyCode` enum                  | `CurrencyCode`               |
| Embedded reference (`xxxId: bigint`) | Expanded `Related*Dto` shape |

### Envelope

REST API 沿用既有 `ResultDto<Code, Data>` 與 `OffsetPagingResultDto<Code, Item[]>` envelope。本 spec 內 endpoint signature 沿用既有 fund REST 表面（見 `libs/contract-rest/src/lib/fund/api/*.api.ts`）的型態。

### Identity

新增 endpoint 全部走 `ApiIdentity.PlatformUser`。本期 transfer 不對 merchant API 開放主動發起，merchant API 僅有 withdrawal 既有面。

### 命名

- DTO 後綴 `Dto`（所有 REST shape type）
- 批次相關 DTO 後綴 `XxxBatchDto`，shape 指示插在動詞位（如 `PlatformSubmitWithdrawalIntentGatherBatchDto`、`PlatformSubmitTransferIntentDistributeBatchDto`）
- Batch item 型別後綴 `ItemDto` / `ResultItemDto`（如 `WithdrawalIntentGatheringItemDto`、`TransferIntentDistributionResultItemDto`）——動詞變名詞形式（Gather → Gathering、Distribute → Distribution）
- Enum 與 IssueCode 為值集合，不視為 shape type，不加 `Dto` 後綴（與既有 `WithdrawalIntentStatus`、`WithdrawalIntentQuoteIssueCode` 對齊）
- Enum 後綴 `Category` / `Status` / `IssueCode` 依用途

---

## Shared Batch Contract Design

### Batch shapes

本期 batch endpoint 採兩種 shape：

| Shape          | 結構                                     | Batch verb   | 範例 UI 流程                                  |
| -------------- | ---------------------------------------- | ------------ | --------------------------------------------- |
| **Gather**     | 多 sender 收斂到 1 recipient/destination | `gather`     | merchant 多 wallet 歸集到 1 platform treasury |
| **Distribute** | 1 sender 散發到多 recipient              | `distribute` | platform 1 treasury 注入多 merchant wallet    |

各 fund case 採用：

- **WithdrawalIntent**：僅 `gather`（本期 PM 無 1-to-N withdrawal 需求）
- **TransferIntent**：兩者皆有

`Sweep` / `Disburse` 等業界 PSP 詞彙刻意保留給未來真正的業務語意操作，**不用於 batch shape 命名**。

### DTO 結構模板

Request：

```ts
type XxxYyyBatchDto = {
  // 1-side 共用欄位（recipient 或 sender，依 shape 而定）
  <sharedSideFields>;
  currencyCode: SupportedCurrencyCode;
  // N-side per-item 陣列
  items: XxxYyyItemDto[];
};

type XxxYyyItemDto = {
  // 對應 N-side 的 wallet ref
  <varyingSideWalletIdField>: ShortId;
  amount: NumericString;
};
```

Response：

```ts
type XxxYyyBatchResultDto = {
  code: CommonCode.Ok; // 永遠 Ok，partial 成敗在 data 內承擔
  data: XxxYyyResultItemDto[]; // 與 input items 同序、同長度
};

// Result item 透過 intersection 延伸 request item，顯式表達「結果 = 輸入 + (createdId, issue)」
type XxxYyyResultItemDto = XxxYyyItemDto & {
  createdId: ShortId | null;
  issue: XxxYyySubmitIssueCode | null;
};
```

設計依據：

- **Envelope 維持 `{ code, data }` 標準 ResultDto 形狀**：partial item 失敗不上升為 envelope error
- **Result item 採 intersection 延伸 request item**：明確表達 response = request + (createdId, issue) 的延伸關係，避免重複欄位定義
- **`createdId` / `issue` 互斥不變式**：兩者恰好一個非 null
  - success → `createdId` 非 null、`issue` null
  - failure → `createdId` null、`issue` 非 null
- **Result items 與 input items 同序、同長度**：前端依 index 對位
- **無 `message` 欄位**：與既有 REST envelope 一致；錯誤語意由 issue code 承擔
- **Item 採命名型別而非 anonymous inline**：對齊 codebase 既有 `WithdrawalIntentStatusHistoryItemDto` pattern；型別可 import 給 validator / mapper 重用

### 外層 endpoint 可能的非 Ok 回應

當整批請求結構上無法處理（envelope-level 失敗）時：

- `data_invalid`：input shape 錯誤、items 陣列為空、超出容量限制
- `forbidden`：caller 無權呼叫此 endpoint

注意這些是**整批未進入處理的失敗**，與 per-item issue 不是同一層概念。

### Batch 容量限制

當前 default：**每批次 50 筆**。

- 超出回 `data_invalid`（外層 endpoint error）
- 容量決策依據：避免單一 request 觸發過多 wallet RPC + DB ops（每筆 reserve + persist + bind + dispatch 約 6 個 DB ops + 多次 RPC），50 筆對應約 300+ DB ops 與 50 次 wallet RPC，仍在合理 gateway timeout 範圍內

此 default 可依實際運維狀況調整，由 server-side 配置承載，不在 contract 表面暴露具體上限值。前端**不應假設特定上限**；若需動態查詢上限，留 future capability endpoint。

### Atomicity

明確的：**per-item 獨立成敗，無跨 item rollback**。

- 一筆 item 失敗不影響其他 items
- 不存在「整批 atomic」變體
- 部分成功為合法 batch outcome

### Concurrency

伺服器端對 batch 內 items 的處理順序與並發策略**為實作細節**，不在 contract 表面承諾：

- 不保證並發處理
- 不保證序列處理
- 不保證任何特定執行順序

前端不應依賴 item 間執行順序作為業務邏輯——每筆 item 的成敗只能由其自身 result 判斷。

---

## WithdrawalIntent Contracts

### Enums

**新增 `WithdrawalIntentCategory`**（建議放 `libs/contract-base/src/lib/fund/enum/`）：

```ts
export enum WithdrawalIntentCategory {
  FromMerchant = 'from_merchant',
  FromPlatform = 'from_platform',
}
```

**`WithdrawalIntentStatus` 既有 enum 不動**。

### `WithdrawalIntentDto` 更新

新增兩個欄位：`category`（mandatory）與 `feePayerWallet`（mandatory，承 impl ep-1 plan-1 first part 的 feePayer 持久化）：

```ts
export type WithdrawalIntentDto = {
  id: ShortId;
  category: WithdrawalIntentCategory; // NEW
  senderMerchant: RelatedMerchantDto | null;
  senderWallet: RelatedWalletDto;
  senderNetworkEndpoint: RelatedNetworkEndpointDto;
  feePayerWallet: RelatedWalletDto; // NEW
  targetNetworkTransaction: CorrelatedNetworkTransactionDto | null;
  destination: NetworkDestinationDto;
  currencyCode: CurrencyCode;
  amount: NumericString;
  amountNet: NumericString;
  amountFee: NumericString;
  status: WithdrawalIntentStatus;
  submittedBy: RelatedActorDto;
  submittedAt: IsoDateTimeUtc;
  cancellableUntil: IsoDateTimeUtc;
  processedAt: IsoDateTimeUtc | null;
  instructedAt: IsoDateTimeUtc | null;
  finalizedAt: IsoDateTimeUtc | null;
  statusHistory: WithdrawalIntentStatusHistoryItemDto[];
  createdAt: IsoDateTimeUtc;
  updatedAt: IsoDateTimeUtc;
};
```

### Submit DTO（不變動；category 由 server 推導，不在 client 端契約表面）

兩個 submit DTO 形狀**與既有版本完全一致**，本期不加 `category` 欄位：

```ts
export type MerchantSubmitWithdrawalIntentDto = {
  senderWalletId: ShortId;
  destination: NetworkDestinationDto;
  currencyCode: SupportedCurrencyCode;
  amount: NumericString;
};

export type PlatformSubmitWithdrawalIntentDto = {
  senderWalletId: ShortId;
  destination: NetworkDestinationDto;
  currencyCode: SupportedCurrencyCode;
  amount: NumericString;
};
```

Server 端在 submit use case 內依 sender wallet type 推導 category：

```text
senderWallet.type = platform_treasury → from_platform
senderWallet.type = merchant_wallet   → from_merchant
```

Caller 端不感知 category；category 為 server-side derived attribute，僅在 response DTO 與 search filter 表面出現。

### Search 參數更新

`PlatformSearchWithdrawalIntentParamsDto` 與 `MerchantSearchWithdrawalIntentParamsDto` 新增 `categoryIn`：

```ts
export type PlatformSearchWithdrawalIntentParamsDto = OffsetPagingParamsDto & {
  categoryIn?: WithdrawalIntentCategory[]; // NEW
  senderMerchantIdIn?: Uuid[];
  senderWalletIdIn?: ShortId[];
  statusIn?: WithdrawalIntentStatus[];
};

export type MerchantSearchWithdrawalIntentParamsDto = OffsetPagingParamsDto & {
  categoryIn?: WithdrawalIntentCategory[]; // NEW; merchant 端實際只會傳 ['from_merchant']
  senderWalletIdIn?: ShortId[];
  statusIn?: WithdrawalIntentStatus[];
};
```

### Quote DTO 更新

`WithdrawalIntentQuoteDto` 新增 `category` 反映 server 推導結果：

```ts
export type WithdrawalIntentQuoteDto = {
  category: WithdrawalIntentCategory; // NEW
  senderMerchant: RelatedMerchantDto | null;
  senderWallet: RelatedWalletDto;
  senderNetworkEndpoint: RelatedNetworkEndpointDto;
  destination: NetworkDestinationDto;
  currencyCode: SupportedCurrencyCode;
  amount: NumericString;
  amountNet: NumericString;
  amountFee: NumericString;
  isSubmittable: boolean;
  issues: WithdrawalIntentQuoteIssueCode[];
};
```

### NEW: Platform Gather 批次 submit

WithdrawalIntent 本期僅引入 `gather` shape（多 sender → 1 external destination）：

```ts
// Per-item types
export type WithdrawalIntentGatheringItemDto = {
  senderWalletId: ShortId;
  amount: NumericString;
};

export type WithdrawalIntentGatheringResultItemDto = WithdrawalIntentGatheringItemDto & {
  createdId: ShortId | null;
  issue: WithdrawalIntentSubmitIssueCode | null;
};

// Batch request: shared destination + currencyCode + items
export type PlatformSubmitWithdrawalIntentGatherBatchDto = {
  destination: NetworkDestinationDto;
  currencyCode: SupportedCurrencyCode;
  items: WithdrawalIntentGatheringItemDto[];
};

// Batch response: envelope 永遠 Ok，partial 結果在 data 內
export type PlatformSubmitWithdrawalIntentGatherBatchResultDto = {
  code: CommonCode.Ok;
  data: WithdrawalIntentGatheringResultItemDto[];
};

// Per-item issue code（enum；type name 承擔 scope，enum value bare）
export enum WithdrawalIntentSubmitIssueCode {
  ConfigIncomplete = 'config_incomplete',
  DestinationInvalid = 'destination_invalid',
  DestinationInternal = 'destination_internal',
  BalanceInsufficient = 'balance_insufficient',
  DailyLimitExceeded = 'daily_limit_exceeded',
  MonthlyLimitExceeded = 'monthly_limit_exceeded',
  DataInvalid = 'data_invalid',
}
```

注意：

- `destination` 與 `currencyCode` 為整批 shared，items 內僅變動 `senderWalletId` 與 `amount`
- Per-item `issue` code surface 鏡像 single submit 錯誤碼，enum value 為 bare 形式（無 `withdrawal_intent.` prefix，type name 已承擔 scope）
- `data` items 順序與 input `items` 對位
- `createdId` 與 `issue` 兩者恰好一個非 null（success → createdId 非 null；failure → issue 非 null）
- 批次上限 50 筆（envelope-level `data_invalid` 拒收超量）
- Items 內 category 必為 `from_platform` 或 `from_merchant`，由 server 依 sender wallet type 自動推導（與 single submit 相同邏輯）

### Endpoint summary（platform API）

`basePath: '/fund/plat/withdrawal-intents'`

| Endpoint                                     | Method/path                    | 變更                                 | Input                                                    | Output                                                                                     |
| -------------------------------------------- | ------------------------------ | ------------------------------------ | -------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `searchWithdrawalIntents`                    | `GET ''`                       | params 加 `categoryIn`               | `PlatformSearchWithdrawalIntentParamsDto`                | `OffsetPagingResultDto<Ok, WithdrawalIntentDto[]>` 或 `params_invalid`                     |
| `getWithdrawalIntentById`                    | `GET '/{id}'`                  | DTO 加 `category` + `feePayerWallet` | `{ id: ShortId }`                                        | `ResultDto<Ok, WithdrawalIntentDto>` 或 `not_found`                                        |
| `quoteWithdrawalIntent`                      | `POST '/$quote'`               | quote DTO 加 `category`              | `{ data: PlatformSubmitWithdrawalIntentDto }`            | `ResultDto<Ok, WithdrawalIntentQuoteDto>` 或 `forbidden \| data_invalid`                   |
| `submitWithdrawalIntent`                     | `POST ''`                      | submit DTO 加 optional `category`    | `{ data: PlatformSubmitWithdrawalIntentDto }`            | `ResultDto<Ok, WithdrawalIntentDto>` 或 `forbidden \| data_invalid \| withdrawal_intent.*` |
| **NEW**: `submitWithdrawalIntentGatherBatch` | `POST '/$submit-gather-batch'` | 新增                                 | `{ data: PlatformSubmitWithdrawalIntentGatherBatchDto }` | `PlatformSubmitWithdrawalIntentGatherBatchResultDto` 或 `forbidden \| data_invalid`        |
| `cancelWithdrawalIntentById`                 | `POST '/{id}/$cancel'`         | DTO 改動                             | `{ id: ShortId }`                                        | `ResultDto<Ok, WithdrawalIntentDto>` 或 `not_found \| conflict`                            |
| `rejectWithdrawalIntentById`                 | `POST '/{id}/$reject'`         | DTO 改動                             | `{ id: ShortId; reason: string }`                        | `ResultDto<Ok, WithdrawalIntentDto>` 或 `not_found \| conflict`                            |
| `releaseWithdrawalIntentById`                | `POST '/{id}/$release'`        | DTO 改動                             | `{ id: ShortId }`                                        | `ResultDto<Ok, WithdrawalIntentDto>` 或 `not_found \| conflict`                            |

merchant API 同步異動 search params / submit DTO / quote DTO，但**不**新增 batch endpoint（merchant 端本期無 batch 需求）。

### Error code 表面（withdrawal）

| Error code                                 | HTTP status | 觸發條件                                                                                                     |
| ------------------------------------------ | ----------- | ------------------------------------------------------------------------------------------------------------ |
| `data_invalid`                             | 400         | input shape 不合法；batch items 超出容量                                                                     |
| `withdrawal_intent.config_incomplete`      | 400         | fee policy / limit / delegation / asset / wallet config 缺失                                                 |
| `withdrawal_intent.destination_invalid`    | 400         | destination 形狀錯誤、identifier 格式不符網路規格等通用格式問題                                              |
| `withdrawal_intent.destination_internal`   | 400         | destination identifier 命中已知 internal network endpoint（方向不變式違反——caller 應改走 transfer endpoint） |
| `withdrawal_intent.balance_insufficient`   | 400         | wallet allocation reserve 不足                                                                               |
| `withdrawal_intent.daily_limit_exceeded`   | 400         | merchant 日限額                                                                                              |
| `withdrawal_intent.monthly_limit_exceeded` | 400         | merchant 月限額                                                                                              |

本期新增 `withdrawal_intent.destination_internal` 一個 withdrawal-specific error code，承載方向不變式檢查結果（destination 不能命中已知 internal endpoint）。`destination_invalid` 維持承載通用格式問題，與新 code 形成清楚分工。

**Batch per-item issue codes**（與上表 envelope-level error 不同層）：見 `WithdrawalIntentSubmitIssueCode`（上節「NEW: Platform Gather 批次 submit」）。Issue code 為 bare 形式（無 `withdrawal_intent.` prefix，由 type name 承擔 scope），語意一一對應上表 envelope-level error 觸發條件，但承載於批次 result item 內、不影響 envelope code。

---

## TransferIntent Contracts

### Enums

```ts
// libs/contract-base/src/lib/fund/enum/transfer-intent-category.enum.ts
export enum TransferIntentCategory {
  PlatformToPlatform = 'platform_to_platform',
  PlatformToMerchant = 'platform_to_merchant',
  MerchantToPlatform = 'merchant_to_platform',
}

// libs/contract-base/src/lib/fund/enum/transfer-intent-status.enum.ts
export enum TransferIntentStatus {
  Preparing = 'preparing',
  Reviewing = 'reviewing',
  Processing = 'processing',
  Processed = 'processed',
  Transacting = 'transacting',
  Completed = 'completed',
  Cancelled = 'cancelled',
  Declined = 'declined',
  Rejected = 'rejected',
  Failed = 'failed',
}

// Quote issue codes
export enum TransferIntentQuoteIssueCode {
  NetworkNotMatch = 'network_not_match',
  WalletNotMatch = 'wallet_not_match',
  WalletCircular = 'wallet_circular',
  ConfigIncomplete = 'config_incomplete',
  BalanceInsufficient = 'balance_insufficient',
}
```

`TransferIntentStatus` 值 verbatim 鏡像 `WithdrawalIntentStatus`，但兩 enum 各自獨立、不共用 type（fund case 各擁自家狀態機）。

### `TransferIntentDto`

```ts
// libs/contract-rest/src/lib/fund/dto/transfer-intent.dto.ts
export type TransferIntentDto = {
  id: ShortId;
  category: TransferIntentCategory;

  senderMerchant: RelatedMerchantDto | null;
  senderWallet: RelatedWalletDto;
  senderNetworkEndpoint: RelatedNetworkEndpointDto;
  senderBucket: WalletBucket.Available | WalletBucket.Held;

  recipientMerchant: RelatedMerchantDto | null;
  recipientWallet: RelatedWalletDto;
  recipientNetworkEndpoint: RelatedNetworkEndpointDto;
  recipientBucket: WalletBucket.Available | WalletBucket.Held;

  feePayerWallet: RelatedWalletDto;

  targetNetworkTransaction: CorrelatedNetworkTransactionDto | null;
  currencyCode: CurrencyCode;
  amount: NumericString;

  status: TransferIntentStatus;
  submittedBy: RelatedActorDto;
  submittedAt: IsoDateTimeUtc;
  processedAt: IsoDateTimeUtc | null;
  instructedAt: IsoDateTimeUtc | null;
  finalizedAt: IsoDateTimeUtc | null;

  statusHistory: TransferIntentStatusHistoryItemDto[];

  createdAt: IsoDateTimeUtc;
  updatedAt: IsoDateTimeUtc;
};

export type TransferIntentStatusHistoryItemDto = {
  sequence: Integer;
  status: TransferIntentStatus;
  occurredAt: IsoDateTimeUtc;
  actor: RelatedActorDto;
  reason: string | null;
};
```

設計依據：

- `senderMerchant` / `recipientMerchant` 對稱可空——merchant sender / recipient 時非 null
- `senderBucket` / `recipientBucket` 顯式攜帶——前端展示「資金從哪個桶到哪個桶」需要這個資訊
- `feePayerWallet` mandatory——transfer 一律有 network fee，feePayer 必存在
- 無 `destination` 欄位（recipient 是 internal wallet，無 external descriptor）
- 無 `amountNet` / `amountFee`（不收 service fee）
- 無 `cancellableUntil`（cancel 政策純由 status + actor 權限承擔）

### Submit DTO（platform）

```ts
export type PlatformSubmitTransferIntentDto = {
  senderWalletId: ShortId;
  recipientWalletId: ShortId;
  currencyCode: SupportedCurrencyCode;
  amount: NumericString;
};
```

Server 端在 submit use case 內依 (sender wallet type, recipient wallet type) pair 推導 category：

```text
(senderWallet.type, recipientWallet.type)
  (platform_treasury, platform_treasury) → platform_to_platform
  (platform_treasury, merchant_wallet)   → platform_to_merchant
  (merchant_wallet,   platform_treasury) → merchant_to_platform
```

不在上述三組之內 → 命中 `wallet_not_match` issue 或對應 envelope-level error。

Bucket 不在 submit DTO 暴露——由 server 依 category 套用 canonical invariant（見 design draft 「Category × bucket × fee payer 不變式」段）。前端只需挑選 wallet，無需理解 bucket 結構。

Caller 端不感知 category；category 為 server-side derived attribute，僅在 response DTO 與 search filter 表面出現。

### Search 參數

```ts
export type PlatformSearchTransferIntentParamsDto = OffsetPagingParamsDto & {
  categoryIn?: TransferIntentCategory[];
  senderMerchantIdIn?: Uuid[];
  senderWalletIdIn?: ShortId[];
  recipientMerchantIdIn?: Uuid[];
  recipientWalletIdIn?: ShortId[];
  statusIn?: TransferIntentStatus[];
};
```

無 merchant 端 search（merchant API 本期不開放 transfer）。

### Quote DTO

```ts
export type TransferIntentQuoteDto = {
  category: TransferIntentCategory;

  senderMerchant: RelatedMerchantDto | null;
  senderWallet: RelatedWalletDto;
  senderNetworkEndpoint: RelatedNetworkEndpointDto;
  senderBucket: WalletBucket.Available | WalletBucket.Held;

  recipientMerchant: RelatedMerchantDto | null;
  recipientWallet: RelatedWalletDto;
  recipientNetworkEndpoint: RelatedNetworkEndpointDto;
  recipientBucket: WalletBucket.Available | WalletBucket.Held;

  feePayerWallet: RelatedWalletDto;

  currencyCode: SupportedCurrencyCode;
  amount: NumericString;
  estimatedNetworkFee: NumericString;

  isSubmittable: boolean;
  issues: TransferIntentQuoteIssueCode[];
};
```

`estimatedNetworkFee` 為 onchain network fee 預估值（與 withdrawal 對齊；幣別由 endpoint 推導，display side 需另查或假設網路 native token）。

### NEW: Platform Gather / Distribute 批次 submit

TransferIntent 提供兩個 batch shape：

**Gather shape**（多 sender → 1 recipient）：

```ts
export type TransferIntentGatheringItemDto = {
  senderWalletId: ShortId;
  amount: NumericString;
};

export type TransferIntentGatheringResultItemDto = TransferIntentGatheringItemDto & {
  createdId: ShortId | null;
  issue: TransferIntentSubmitIssueCode | null;
};

export type PlatformSubmitTransferIntentGatherBatchDto = {
  recipientWalletId: ShortId;
  currencyCode: SupportedCurrencyCode;
  items: TransferIntentGatheringItemDto[];
};

export type PlatformSubmitTransferIntentGatherBatchResultDto = {
  code: CommonCode.Ok;
  data: TransferIntentGatheringResultItemDto[];
};
```

**Distribute shape**（1 sender → 多 recipient）：

```ts
export type TransferIntentDistributionItemDto = {
  recipientWalletId: ShortId;
  amount: NumericString;
};

export type TransferIntentDistributionResultItemDto = TransferIntentDistributionItemDto & {
  createdId: ShortId | null;
  issue: TransferIntentSubmitIssueCode | null;
};

export type PlatformSubmitTransferIntentDistributeBatchDto = {
  senderWalletId: ShortId;
  currencyCode: SupportedCurrencyCode;
  items: TransferIntentDistributionItemDto[];
};

export type PlatformSubmitTransferIntentDistributeBatchResultDto = {
  code: CommonCode.Ok;
  data: TransferIntentDistributionResultItemDto[];
};
```

**Per-item Issue code**（enum；type name 承擔 scope，enum value bare）：

```ts
export enum TransferIntentSubmitIssueCode {
  ConfigIncomplete = 'config_incomplete',
  NetworkNotMatch = 'network_not_match',
  WalletNotMatch = 'wallet_not_match',
  WalletCircular = 'wallet_circular',
  BalanceInsufficient = 'balance_insufficient',
  DataInvalid = 'data_invalid',
}
```

注意：

- Gather 與 Distribute 為兩個獨立 endpoint，前端依 UI 流程選用對應 endpoint
- 同一批次內所有 items 共用同 batch shape，無法混合 gather + distribute
- Items 內 category 由 server 依 sender + recipient wallet type pair 對自動推導（與 single submit 相同邏輯）
- `data` items 順序與 input `items` 對位
- `createdId` 與 `issue` 兩者恰好一個非 null
- 兩個 batch endpoint 共用上限 50 筆

### Endpoint summary（platform API）

`basePath: '/fund/plat/transfer-intents'`

| Endpoint                              | Method/path                        | Input                                                      | Output                                                                                 |
| ------------------------------------- | ---------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `searchTransferIntents`               | `GET ''`                           | `PlatformSearchTransferIntentParamsDto`                    | `OffsetPagingResultDto<Ok, TransferIntentDto[]>` 或 `params_invalid`                   |
| `getTransferIntentById`               | `GET '/{id}'`                      | `{ id: ShortId }`                                          | `ResultDto<Ok, TransferIntentDto>` 或 `not_found`                                      |
| `quoteTransferIntent`                 | `POST '/$quote'`                   | `{ data: PlatformSubmitTransferIntentDto }`                | `ResultDto<Ok, TransferIntentQuoteDto>` 或 `forbidden \| data_invalid`                 |
| `submitTransferIntent`                | `POST ''`                          | `{ data: PlatformSubmitTransferIntentDto }`                | `ResultDto<Ok, TransferIntentDto>` 或 `forbidden \| data_invalid \| transfer_intent.*` |
| `submitTransferIntentGatherBatch`     | `POST '/$submit-gather-batch'`     | `{ data: PlatformSubmitTransferIntentGatherBatchDto }`     | `PlatformSubmitTransferIntentGatherBatchResultDto` 或 `forbidden \| data_invalid`      |
| `submitTransferIntentDistributeBatch` | `POST '/$submit-distribute-batch'` | `{ data: PlatformSubmitTransferIntentDistributeBatchDto }` | `PlatformSubmitTransferIntentDistributeBatchResultDto` 或 `forbidden \| data_invalid`  |
| `cancelTransferIntentById`            | `POST '/{id}/$cancel'`             | `{ id: ShortId }`                                          | `ResultDto<Ok, TransferIntentDto>` 或 `not_found \| conflict`                          |
| `rejectTransferIntentById`            | `POST '/{id}/$reject'`             | `{ id: ShortId; reason: string }`                          | `ResultDto<Ok, TransferIntentDto>` 或 `not_found \| conflict`                          |
| `releaseTransferIntentById`           | `POST '/{id}/$release'`            | `{ id: ShortId }`                                          | `ResultDto<Ok, TransferIntentDto>` 或 `not_found \| conflict`                          |

### Error code 表面（transfer）

| Error code                             | HTTP status | 觸發條件                                                                                                        |
| -------------------------------------- | ----------- | --------------------------------------------------------------------------------------------------------------- |
| `data_invalid`                         | 400         | input shape 不合法；batch items 超出容量                                                                        |
| `transfer_intent.config_incomplete`    | 400         | fee delegation 缺失（merchant_to_platform）、fee policy 缺失、wallet asset 缺失等                               |
| `transfer_intent.network_not_match`    | 400         | sender 與 recipient 兩端 networkEndpoint 推導出的 `(type, provider)` pair 不相等（same-network invariant 違反） |
| `transfer_intent.wallet_not_match`     | 400         | sender + recipient wallet type 組合不在三 category 任一支援列                                                   |
| `transfer_intent.wallet_circular`      | 400         | sender 與 recipient 為同一個 wallet（語義無效轉帳；self-loop / circular reference）                             |
| `transfer_intent.balance_insufficient` | 400         | wallet allocation reserve 不足                                                                                  |

**Batch per-item issue codes**（與上表 envelope-level error 不同層）：見 `TransferIntentSubmitIssueCode`（上節「NEW: Platform Gather / Distribute 批次 submit」）。Issue code 為 bare 形式（無 `transfer_intent.` prefix，由 type name 承擔 scope），語意一一對應上表 envelope-level error 觸發條件，但承載於批次 result item 內、不影響 envelope code。

---

## Frontend Consumption Notes

### 三個批次 endpoint 對位邏輯

三個 batch endpoint（withdrawal gather、transfer gather、transfer distribute）採同一結構模板：

- input：`{ <sharedFields>, items: BatchItem[] }`
- output：`{ code: CommonCode.Ok, data: BatchResultItem[] }`（partial 失敗仍是 `code: Ok`）
- `data` items 順序 = input `items` 順序、長度相等

前端對位 pseudo-code：

```ts
for (let i = 0; i < input.items.length; i++) {
  const inputItem = input.items[i];
  const resultItem = response.data[i];

  // inputItem 的欄位（senderWalletId/recipientWalletId、amount）會原樣
  // 出現在 resultItem 上（intersection 結構）
  if (resultItem.createdId !== null) {
    // success
    //   resultItem.createdId: 新建 fund case 的 ShortId
    //   resultItem.issue: null
    //   前端可導向 detail / highlight
  } else {
    // failure
    //   resultItem.createdId: null
    //   resultItem.issue: WithdrawalIntentSubmitIssueCode | TransferIntentSubmitIssueCode
    //   依 issue 顯示對應使用者訊息
  }
}
```

不變式：每 `data` item 上 `createdId` 與 `issue` 恰好一個非 null。

外層 `code !== Ok`（如 `data_invalid` / `forbidden`）代表**整批未進入處理**，與 per-item issue 不同層；此時 `data` 不存在。

### Category 不暴露於 submit 契約

Submit DTO（withdrawal merchant / withdrawal platform / transfer platform）皆**不**接受 `category` 欄位。Category 完全由 server 在 submit use case 內依 wallet type 推導：

- WithdrawalIntent：依 sender wallet type 推導 `from_merchant` / `from_platform`
- TransferIntent：依 (sender wallet type, recipient wallet type) pair 推導三個 category 之一；不在三組之內命中 `wallet_not_match` 失敗

前端**不需要**理解 category 結構，亦不需在提交前判斷類別。Category 僅在 response DTO（顯示用）與 search filter（`categoryIn`）表面出現。

### Bucket 不在 submit DTO

Transfer 三 category 對應的 sender / recipient bucket 由 server 自動決定，前端**無需挑 bucket**。前端只需挑選兩個 wallet + currency + amount。

如需在 UI 顯示「資金從哪個 bucket 到哪個 bucket」（例如 `merchant_to_platform` 是從 held 收回），可在 quote 或 submit 回傳後讀取 `senderBucket` / `recipientBucket` 顯示。

### Quote 在批次場景的使用

Quote endpoint 為 single-item 設計，不提供批次 quote。前端若需顯示批次預估摘要（如「總計 N 筆，預估總 network fee Y」），需自行 client-side aggregate N 個 quote 呼叫結果。

是否需要 batch quote endpoint 留 future consideration。

---

## Open Points

下列項目於 `spec/open-points.md` 追蹤：

1. **Batch capacity 50 筆** 為當前 default，待實際運維壓測後可能調整。前端不應 hardcode 此數字作為 UI 限制依據。
2. **Batch quote endpoint** 目前不提供；前端若有強需求可 client-side 聚合 single quote，或未來提供 batch quote。
3. **`estimatedNetworkFee` 幣別** 隱含由 endpoint network 推導；REST DTO 未顯式攜帶 fee currency。若前端需要顯示 fee currency 名稱，可由 wallet 或 endpoint reference 推導。是否要在 quote DTO 直接攜帶 fee currency 為待議。
4. **Merchant API 對 transfer 開放與否** 目前不開放；若 PM 未來規劃 merchant 主動發起 transfer，需新增 merchant API endpoint 與對應 use case，本期 spec 未涵蓋。
5. **Status filter 與 category filter 的組合預期** 在 search endpoint 上採 AND 語意（傳兩個 filter 取交集）；無特別 invariant。
6. **`withdrawal_intent.destination_internal` 偵測實作** 為晚期 plan 項目。Enum value 與 error code 已於本期 contract spec 確立，但實際的「destination identifier 命中已知 internal endpoint」偵測邏輯尚未實作——需要 NetworkEndpoint repository 支援依 identifier 的 reverse-lookup 能力，這份能力本身可能也是新增開發。早期 plan 先讓 enum 落定，server 端偵測落地遞延到該能力具備之後的 plan。

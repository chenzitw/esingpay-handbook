# Wallet Export CSV Column And Mapping Plan

本文整理 wallet ledger / cashflow export CSV 的 row DTO、欄位規劃與資料包裝規則。

## Mapping 原則

- export params 只決定 filter semantics，不決定最終 CSV 顯示值。
- CSV output 不應直接 dump upstream DTO 原始欄位。
- 匯出流程必須保留獨立 row mapper，先把 query item 轉成「CSV row view model」，再交給 csv writer。
- 已存在於前端 list view 的顯示轉換規則，CSV 應沿用同一組語意，避免畫面與匯出檔案不一致。

目前可直接依循的欄位值解析文件：

- [deposit-withdrawal-console-export-field-mapping.md](../deposit-withdrawal-console-export-field-mapping.md)
- [transaction-ledger-console-export-field-mapping.md](../transaction-ledger-console-export-field-mapping.md)

## Row DTO Contract

CSV 欄位規劃應先落成獨立 row DTO，避免 mapper / csv writer 直接依賴 upstream item schema。

```ts
export type WalletLedgerItemSheetRowDto = {
  id: string;
  merchantId: string;
  merchantName: string;
  walletId: string;
  walletType: string;
  walletMerchantId: string;
  networkEndpointId: string;
  currencyCode: string;
  bucket: string;
  kind: string;
  ownerType: string;
  planAmountDelta: string;
  outcomeAmountDelta: string;
  realization: string;
  side: string;
  sourceSpace: string;
  destinationSpace: string;
  fundCaseType: string;
  fundCaseIdentifier: string;
  fundCaseStatus: string;
  networkTxId: string;
  networkTxIdentifier: string;
  networkTxStatus: string;
  effectiveAt: string;
  createdAt: string;
  updatedAt: string;
};
```

```ts
export type WalletCashflowItemSheetRowDto = {
  id: string;
  merchantId: string;
  merchantName: string;
  walletId: string;
  walletType: string;
  walletMerchantId: string;
  networkEndpointId: string;
  direction: string;
  kind: string;
  currencyCode: string;
  amountDelta: string;
  amountGross: string;
  amountNet: string;
  amountFee: string;
  status: string;
  correlationType: string;
  correlationIdentifier: string;
  subjectId: string;
  subjectStatus: string;
  subjectCurrencyCode: string;
  subjectAmount: string;
  subjectAmountNet: string;
  subjectAmountFee: string;
  subjectFinalizedAt: string;
  effectiveAt: string;
  createdAt: string;
  updatedAt: string;
};
```

## Ledger CSV Columns

第一版建議輸出：

| Column | Source |
| --- | --- |
| `id` | `item.id` |
| `merchantId` | `item.merchant?.id ?? ''` |
| `merchantName` | `item.merchant?.name ?? ''` |
| `walletId` | `item.wallet.id` |
| `walletType` | `item.wallet.type` |
| `walletMerchantId` | `item.wallet.merchantId ?? ''` |
| `networkEndpointId` | `item.wallet.networkEndpointId` |
| `currencyCode` | `item.currencyCode` |
| `bucket` | `item.bucket` |
| `kind` | `item.kind` |
| `ownerType` | `item.ownerType` |
| `planAmountDelta` | `item.planAmountDelta ?? ''` |
| `outcomeAmountDelta` | `item.outcomeAmountDelta ?? ''` |
| `realization` | `item.realization` |
| `side` | `item.side` |
| `sourceSpace` | `item.source.space` |
| `destinationSpace` | `item.destination.space` |
| `fundCaseType` | `item.fundCase.type` |
| `fundCaseIdentifier` | `item.fundCase.identifier` |
| `fundCaseStatus` | `item.fundCase.subject.status` |
| `networkTxId` | `item.networkTransaction?.id ?? ''` |
| `networkTxIdentifier` | `item.networkTransaction?.identifier ?? ''` |
| `networkTxStatus` | `item.networkTransaction?.status ?? ''` |
| `effectiveAt` | `item.effectiveAt` |
| `createdAt` | `item.createdAt` |
| `updatedAt` | `item.updatedAt` |

但實際輸出值不應直接等於原始欄位值，而是由 `WalletLedgerExportRowMapper` 統一處理：

- 前端已有顯示轉換邏輯的欄位，在 CSV 也應沿用同一語意
- merchant 系列的 `effectiveAt` / `createdAt` / `updatedAt` 應先轉成 merchant 登記時區再輸出
- 若需要在原始值與顯示值間擇一，應優先以 CSV 顯示值為主

platform ledger 目前可直接參照：

- [transaction-ledger-console-export-field-mapping.md](../transaction-ledger-console-export-field-mapping.md)

## Cashflow CSV Columns

第一版建議輸出：

| Column | Source |
| --- | --- |
| `id` | `item.id` |
| `merchantId` | `item.merchant?.id ?? ''` |
| `merchantName` | `item.merchant?.name ?? ''` |
| `walletId` | `item.wallet.id` |
| `walletType` | `item.wallet.type` |
| `walletMerchantId` | `item.wallet.merchantId ?? ''` |
| `networkEndpointId` | `item.wallet.networkEndpointId` |
| `direction` | `item.direction` |
| `kind` | `item.kind` |
| `currencyCode` | `item.currencyCode` |
| `amountDelta` | `item.amountDelta` |
| `amountGross` | `item.amountGross` |
| `amountNet` | `item.amountNet` |
| `amountFee` | `item.amountFee` |
| `status` | `item.status` |
| `correlationType` | `item.correlation.type` |
| `correlationIdentifier` | `item.correlation.identifier` |
| `subjectId` | `item.correlation.subject.id` |
| `subjectStatus` | `item.correlation.subject.status` |
| `subjectCurrencyCode` | `item.correlation.subject.currencyCode` |
| `subjectAmount` | `item.correlation.subject.amount` |
| `subjectAmountNet` | `item.correlation.subject.amountNet` |
| `subjectAmountFee` | `item.correlation.subject.amountFee` |
| `subjectFinalizedAt` | `item.correlation.subject.finalizedAt ?? ''` |
| `effectiveAt` | `item.effectiveAt` |
| `createdAt` | `item.createdAt` |
| `updatedAt` | `item.updatedAt` |

第一版先輸出穩定的一階欄位，不把 `statusHistory`、`source`、`destination` 等多層陣列 / 物件全面攤平。若後續需要，再以 JSON 欄位或額外平坦欄位擴充。

cashflow mapper 應負責：

- 套用前端既有數值 / 狀態顯示轉換規則
- 處理 merchant 系列 export 的時區轉換後時間字串
- 確保 CSV 欄位語意面向使用者閱讀，而不是資料庫欄位 dump

merchant cashflow 目前可直接參照：

- [deposit-withdrawal-console-export-field-mapping.md](../deposit-withdrawal-console-export-field-mapping.md)

## Row Mapping Strategy

建議明確保留以下責任切分：

1. source service 只負責拿原始 item
2. row mapper 負責欄位轉換、枚舉顯示值轉換、數值顯示格式調整、時間字串轉換
3. use-case 負責串接 mapper 與 csv / file service
4. csv service 只接收已完成轉換的 row model 寫檔

這樣未來若前端顯示邏輯更新，可以集中比對 mapper 規則，而不需要在 controller 或 csv writer 追查散落邏輯。

## Mapping Coverage Note

目前已提供的兩份 mapping 文件可直接支撐：

- platform transaction ledger 的欄位顯示語意
- merchant deposit / withdrawal console 的 cashflow 類欄位顯示語意

若 `plat-cashflow` 或 `merch-ledger` 在實作時存在與這兩份 baseline 不同的畫面欄位語意，應先補充對應 mapping 文件，再實作 row mapper，避免 CSV 語意在不同入口間漂移。
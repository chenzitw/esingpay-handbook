# Transaction Ledger 欄位對應與轉換邏輯

此文件整理 platform portal transaction ledger 目前顯示中的欄位與 `WalletLedgerItemDto` 的對應關係。

參考程式碼：

console-esing repo 中：

- `apps/esing-pay-platform-portal/src/pages/companies/assetOverview/transactionLedgerSection/transactionLedgerSection.tsx`
- `apps/esing-pay-platform-portal/src/pages/companies/assetOverview/transactionLedgerSection/utils.ts`

資料來源 DTO：

- `WalletLedgerItemDto`

## 欄位總表

| 顯示欄位           | column id           | DTO 來源                                               | 最後顯示文字 / 值                                                  |
| ------------------ | ------------------- | ------------------------------------------------------ | ------------------------------------------------------------------ |
| Created At         | `createdAt`         | `fundCase.subject.createdAt`                           | 格式化後的日期時間文字                                             |
| Updated At         | `updatedAt`         | `updatedAt`                                            | 格式化後的日期時間文字                                             |
| Serial Number      | `serialNumber`      | `fundCase.identifier`                                  | `fundCase.identifier`                                              |
| Asset Type         | `assetType`         | `ownerType`                                            | `ownerType`；無值顯示 `-`                                          |
| Transaction Type   | `transactionType`   | `fundCase.type`                                        | `fundCase.type`；無值顯示 `-`                                      |
| Transaction Reason | `transactionReason` | `kind`                                                 | `kind`；無值顯示 `-`                                               |
| Status             | `status`            | `fundCase.subject.status`                              | `fundCase.subject.status`                                          |
| Tx Owner           | `txOwner`           | `ownerType`, `merchant`                                | 依 `ownerType` 與 `merchant` 組出文字，規則見下方                  |
| Amount             | `amount`            | `realization`, `planAmountDelta`, `outcomeAmountDelta` | 依 `realization` 選擇金額欄位；無值顯示 `-`                        |
| From               | `fromName`          | `networkTransaction`, `source`                         | 有 `networkTransaction` 時依 `source` 組出文字；否則顯示 `-`       |
| From Address       | `fromAddress`       | `networkTransaction.source.identifier`                 | `networkTransaction.source.identifier`；無值顯示 `-`               |
| To                 | `toName`            | `networkTransaction`, `destination`                    | 有 `networkTransaction` 時依 `destination` 組出文字；否則顯示 `-`  |
| To Address         | `toAddress`         | `networkTransaction.destination.identifier`            | `networkTransaction.destination.identifier`；無值顯示 `-`          |
| Action             | `actions`           | `networkTransaction.identifier`                        | TronScan tx hash 網址；無值顯示 `-`，測試鏈 / 正式鏈依環境變數決定 |


## Created At

DTO 對應：

```ts
row.fundCase.subject.createdAt;
```

顯示邏輯：將 UTC datetime 依商戶時區 / 日期格式設定格式化成日期時間文字。

範例：

| 轉換前 DTO                                           | 轉換後顯示                      |
| ---------------------------------------------------- | ------------------------------- |
| `fundCase.subject.createdAt: "2026-03-01T10:20:00Z"` | MMM D, YYYY HH:mm:ss |

## Updated At

DTO 對應：

```ts
row.updatedAt;
```

顯示邏輯：將 UTC datetime 依商戶時區 / 日期格式設定格式化成日期時間文字。

範例：

| 轉換前 DTO                          | 轉換後顯示                      |
| ----------------------------------- | ------------------------------- |
| `updatedAt: "2026-03-01T10:30:00Z"` | MMM D, YYYY HH:mm:ss |

## Serial Number

DTO 對應：

```ts
row.fundCase.identifier;
```

顯示邏輯：直接顯示 DTO 值。

範例：

| 轉換前 DTO                            | 轉換後顯示     |
| ------------------------------------- | -------------- |
| `fundCase.identifier: "DP00BO6EW2NM"` | `DP00BO6EW2NM` |

## Asset Type

DTO 對應：

```ts
row.ownerType;
```

顯示邏輯：有值時顯示 `ownerType` 原始值；無值顯示 `-`。

範例：

| 轉換前 DTO              | 轉換後顯示 |
| ----------------------- | ---------- |
| `ownerType: "merchant"` | `Merchant` |
| `ownerType: "platform"` | `Provider` |
| `ownerType` 無值        | `-`        |

## Transaction Type

DTO 對應：

```ts
row.fundCase.type;
```

顯示邏輯：有值時顯示 `fundCase.type` 原始值；無值顯示 `-`。

範例：

| 轉換前 DTO                           | 轉換後顯示          |
| ------------------------------------ | ------------------- |
| `fundCase.type: "deposit"`           | `Deposit`           |
| `fundCase.type: "withdrawal_intent"` | `Withdrawal` |

## Transaction Reason

DTO 對應：

```ts
row.kind;
```

顯示邏輯：有值時顯示 `kind` 原始值；無值顯示 `-`。

範例：

| 轉換前 DTO            | 轉換後顯示    |
| --------------------- | ------------- |
| `kind: "principal"`   | `Principal`   |
| `kind: "service_fee"` | `Service Fee` |
| `kind: "network_fee"` | `Network Fee` |

## Status

DTO 對應：

```ts
row.fundCase.subject.status;
```

顯示邏輯：直接顯示 `fundCase.subject.status` 原始值。

範例：

| 轉換前 DTO                             | 轉換後顯示  |
| -------------------------------------- | ----------- |
| `fundCase.subject.status: "completed"` | `completed` |
| `fundCase.subject.status: "failed"`    | `failed`    |

## Tx Owner

DTO 對應：

```ts
row.ownerType;
row.merchant;
```

顯示邏輯：

```ts
const renderTxOwner = (
  ownerType: WalletLedgerItemDto["ownerType"],
  merchant: WalletLedgerItemDto["merchant"],
) => {
  return merchant
    ? `${merchant.name}${ownerType === WalletLedgerItemOwnerType.Platform ? " (Provider)" : ""}`
    : "Provider";
};
```

轉換規則：

| 條件                                         | 顯示                         |
| -------------------------------------------- | ---------------------------- |
| `merchant` 有值，且 `ownerType = "merchant"` | `{merchant.name}`            |
| `merchant` 有值，且 `ownerType = "platform"` | `{merchant.name} (Provider)` |
| `merchant` 為 `null`                         | `Provider`                   |

範例：

| 轉換前 DTO                                       | 轉換後顯示        |
| ------------------------------------------------ | ----------------- |
| `ownerType: "merchant"`, `merchant.name: "Momo"` | `Momo`            |
| `ownerType: "platform"`, `merchant.name: "Momo"` | `Momo (Provider)` |
| `ownerType: "platform"`, `merchant: null`        | `Provider`        |

## Amount

DTO 對應：

```ts
row.realization;
row.planAmountDelta;
row.outcomeAmountDelta;
```

Amount 欄位不是固定取同一個 DTO 欄位，而是依照 `realization` 判斷要顯示預期金額或實際金額。

顯示邏輯：

```ts
const OUTCOME_REALIZATION_LIST = ["partial", "realized"] as const;

const getDisplayAmount = ({
  outcomeAmountDelta,
  planAmountDelta,
  realization,
}: WalletLedgerItemDto) =>
  isOutcomeRealization(realization) ? outcomeAmountDelta : planAmountDelta;
```

轉換規則：

| `realization` | 使用欄位             | 顯示含義                         |
| ------------- | -------------------- | -------------------------------- |
| `pending`     | `planAmountDelta`    | 預期會發生，但尚未實現           |
| `cancelled`   | `planAmountDelta`    | 已取消或失敗，仍顯示原本預期金額 |
| `partial`     | `outcomeAmountDelta` | 部分實現，顯示實際發生金額       |
| `realized`    | `outcomeAmountDelta` | 已實現，顯示實際發生金額         |

選出金額後，若金額為 `null` 則顯示 `-`；若有值則顯示格式化後的數字文字。

範例：

| 轉換前 DTO                                                                         | 轉換後顯示 |
| ---------------------------------------------------------------------------------- | ---------- |
| `realization: "pending"`, `planAmountDelta: "1000"`, `outcomeAmountDelta: null`    | `1,000`    |
| `realization: "cancelled"`, `planAmountDelta: "-500"`, `outcomeAmountDelta: "0"`   | `-500`     |
| `realization: "partial"`, `planAmountDelta: "-500"`, `outcomeAmountDelta: "-300"`  | `-300`     |
| `realization: "realized"`, `planAmountDelta: "1000"`, `outcomeAmountDelta: "1000"` | `1,000`    |
| `realization: "realized"`, `planAmountDelta: null`, `outcomeAmountDelta: "-1.5"`   | `-1.5`     |

## From

DTO 對應：

```ts
row.networkTransaction;
row.source;
```

欄位顯示前先判斷是否有 `networkTransaction`：

```ts
networkTransaction ? renderFromTo(source) : "-";
```

`renderFromTo` 轉換規則見下方「From / To 共用轉換」。

範例：

| 轉換前 DTO                                                                                                                     | 轉換後顯示           |
| ------------------------------------------------------------------------------------------------------------------------------ | -------------------- |
| `networkTransaction` 有值，`source.space: "external"`                                                                          | `[external address]` |
| `networkTransaction` 有值，`source.space: "internal"`, `source.wallet.type: "merchant_wallet"`, `source.merchant.name: "Momo"` | `Momo's Wallet`      |
| `networkTransaction` 有值，`source.space: "internal"`, `source.wallet.type: "platform_treasury"`                               | `Provider's Wallet`  |
| `networkTransaction: null`                                                                                                     | `-`                  |

## From Address

DTO 對應：

```ts
row.networkTransaction?.source?.identifier;
```

顯示邏輯：有值時顯示 `networkTransaction.source.identifier`；無值顯示 `-`。

範例：

| 轉換前 DTO                                                  | 轉換後顯示          |
| ----------------------------------------------------------- | ------------------- |
| `networkTransaction.source.identifier: "TX_SOURCE_ADDRESS"` | `TX_SOURCE_ADDRESS` |
| `networkTransaction: null`                                  | `-`                 |
| `networkTransaction.source.identifier` 無值                 | `-`                 |

## To

DTO 對應：

```ts
row.networkTransaction;
row.destination;
```

欄位顯示前先判斷是否有 `networkTransaction`：

```ts
networkTransaction ? renderFromTo(destination) : "-";
```

`renderFromTo` 轉換規則見下方「From / To 共用轉換」。

範例：

| 轉換前 DTO                                                                                                                                    | 轉換後顯示           |
| --------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| `networkTransaction` 有值，`destination.space: "external"`                                                                                    | `[external address]` |
| `networkTransaction` 有值，`destination.space: "internal"`, `destination.wallet.type: "merchant_wallet"`, `destination.merchant.name: "Momo"` | `Momo's Wallet`      |
| `networkTransaction` 有值，`destination.space: "internal"`, `destination.wallet.type: "platform_treasury"`                                    | `Provider's Wallet`  |
| `networkTransaction: null`                                                                                                                    | `-`                  |

## To Address

DTO 對應：

```ts
row.networkTransaction?.destination?.identifier;
```

顯示邏輯：有值時顯示 `networkTransaction.destination.identifier`；無值顯示 `-`。

範例：

| 轉換前 DTO                                                            | 轉換後顯示               |
| --------------------------------------------------------------------- | ------------------------ |
| `networkTransaction.destination.identifier: "TX_DESTINATION_ADDRESS"` | `TX_DESTINATION_ADDRESS` |
| `networkTransaction: null`                                            | `-`                      |
| `networkTransaction.destination.identifier` 無值                      | `-`                      |

## Action

export 改為 tx url 連結

DTO 對應：

```ts
row.networkTransaction?.identifier;
```

顯示邏輯：有 `networkTransaction.identifier` 時，將 tx hash 組成 TronScan transaction URL；沒有值時顯示 `-`。

URL 格式：

```txt
{VITE_TRON_SCAN_URL}/#/transaction/{networkTransaction.identifier}
```

`VITE_TRON_SCAN_URL` 依部署環境設定，因此會對應到測試鏈或正式鏈的 TronScan 網址。

測試鏈：https://nile.tronscan.org
正式鏈：https://tronscan.org

範例：

| 轉換前 DTO                               | 轉換後顯示                                 |
| ---------------------------------------- | ------------------------------------------ |
| `networkTransaction.identifier: "0xabc"` | `{VITE_TRON_SCAN_URL}/#/transaction/0xabc` |
| `networkTransaction: null`               | `-`                                        |
| `networkTransaction.identifier` 無值     | `-`                                        |

## From / To 共用轉換

`From` 與 `To` 都使用同一個 `renderFromTo` 函式，差別只在傳入 `source` 或 `destination`。

DTO node 來源：

```ts
row.source;
row.destination;
```

顯示邏輯：

```ts
const renderFromTo = (node: WalletLedgerItemDto["source"]) => {
  if (node.space === "external") {
    return "[external address]";
  } else if (node.space === "internal") {
    switch (node.wallet.type) {
      case WalletType.MerchantWallet:
        return node.merchant ? `${node.merchant.name}'s Wallet` : "-";
      case WalletType.PlatformTreasury:
        return "Provider's Wallet";
      default:
        return "-";
    }
  }

  return "-";
};
```

轉換規則：

| 條件                                                                                              | 顯示                            |
| ------------------------------------------------------------------------------------------------- | ------------------------------- |
| `node.space = "external"`                                                                         | `[external address]`            |
| `node.space = "internal"` 且 `node.wallet.type = "merchant_wallet"`，並且 `node.merchant` 有值    | `{node.merchant.name}'s Wallet` |
| `node.space = "internal"` 且 `node.wallet.type = "merchant_wallet"`，但 `node.merchant` 為 `null` | `-`                             |
| `node.space = "internal"` 且 `node.wallet.type = "platform_treasury"`                             | `Provider's Wallet`             |
| 其他未處理的 `wallet.type`                                                                        | `-`                             |

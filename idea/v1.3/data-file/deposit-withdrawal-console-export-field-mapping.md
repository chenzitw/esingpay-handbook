# Deposit / Withdrawal Console Export 欄位對應

此文件整理 merchant portal Deposit / Withdrawal list 目前顯示欄位與 `WalletCashflowItemDto` 的對應關係。Deposit 與 Withdrawal 目前使用相同欄位，因此共用同一份 mapping。

參考程式碼：

console-esing repo 中：

- `apps/esing-pay-merchant-portal/src/pages/deposit/depositTable.tsx`
- `apps/esing-pay-merchant-portal/src/pages/withdrawal/withdrawalTable.tsx`

資料來源 DTO：

- Deposit: `MerchantDepositListItemDto = WalletCashflowItemDto`
- Withdrawal: `MerchantWithdrawalListItemDto = WalletCashflowItemDto`

## 欄位總表

| 顯示欄位            | column id               | DTO 來源                                     | 最後顯示文字 / 值                          |
| ------------------- | ----------------------- | -------------------------------------------- | ------------------------------------------ |
| Merchant Name       | `merchantName`          | `merchant.name`                              | `merchant.name`；無值顯示 `-`              |
| Serial Number       | `serialNumber`          | `correlation.identifier`                     | `correlation.identifier`                   |
| Status              | `status`                | `correlation.subject.status`                 | 依 status 值轉成使用者顯示文字，規則見下方 |
| From Address        | `sourceIdentifier`      | `correlation.subject.source.identifier`      | 有值顯示 address；無值顯示 `-`             |
| To Address          | `destinationIdentifier` | `correlation.subject.destination.identifier` | 有值顯示 address；無值顯示 `-`             |
| Chain               | `sourceRail`            | `correlation.subject.source.rail`            | 依 rail 值轉成鏈名稱，規則見下方           |
| Amount (USDT)       | `amountGross`           | `amountGross`                                | 數字文字，固定顯示到小數點第二位           |
| Handling Fee (USDT) | `amountFee`             | `amountFee`                                  | 數字文字，固定顯示到小數點第二位           |
| Net Amount (USDT)   | `amountNet`             | `amountNet`                                  | 數字文字，固定顯示到小數點第二位           |
| Created At          | `createdAt`             | `createdAt`                                  | 格式化後的日期時間文字                     |

註：table 中的 view action 只用於 console 內開啟詳情，不屬於使用者面向 export 欄位，因此不列入此 mapping。

## Merchant Name

DTO 對應：

```ts
row.merchant.name;
```

顯示邏輯：有值時顯示 `merchant.name`；無值顯示 `-`。

範例：

| 轉換前 DTO                   | 轉換後顯示  |
| ---------------------------- | ----------- |
| `merchant.name: "Momo"`      | `Momo`      |
| `merchant.name: "Demo Shop"` | `Demo Shop` |
| `merchant` 無值              | `-`         |

## Serial Number

DTO 對應：

```ts
row.correlation?.identifier;
```

顯示邏輯：直接顯示 `correlation.identifier`。

範例：

| 轉換前 DTO                               | 轉換後顯示     |
| ---------------------------------------- | -------------- |
| `correlation.identifier: "DP00BO6EW2NM"` | `DP00BO6EW2NM` |
| `correlation.identifier` 無值            | `-`            |

## Status

DTO 對應：

```ts
row.correlation?.subject.status;
```

顯示邏輯：將 `correlation.subject.status` 轉成使用者可讀的狀態文字。

轉換規則：

| DTO status   | 最後顯示文字 |
| ------------ | ------------ |
| `preparing`  | `Preparing`  |
| `cancelled`  | `Cancelled`  |
| `reviewing`  | `Reviewing`  |
| `processing` | `Processing` |
| `processed`  | `Processed`  |
| `declined`   | `Declined`   |
| `failed`     | `Failed`     |
| `completed`  | `Completed`  |
| 無值         | `-`          |

範例：

| 轉換前 DTO                                | 轉換後顯示  |
| ----------------------------------------- | ----------- |
| `correlation.subject.status: "completed"` | `Completed` |
| `correlation.subject.status: "failed"`    | `Failed`    |
| `correlation.subject.status` 無值         | `-`         |

## From Address

DTO 對應：

```ts
row.correlation?.subject.source?.identifier;
```

顯示邏輯：有值時顯示 `correlation.subject.source.identifier`；無值顯示 `-`。

範例：

| 轉換前 DTO                                                   | 轉換後顯示          |
| ------------------------------------------------------------ | ------------------- |
| `correlation.subject.source.identifier: "TX_SOURCE_ADDRESS"` | `TX_SOURCE_ADDRESS` |
| `correlation.subject.source: null`                           | `-`                 |

## To Address

DTO 對應：

```ts
row.correlation?.subject.destination?.identifier;
```

顯示邏輯：有值時顯示 `correlation.subject.destination.identifier`；無值顯示 `-`。

範例：

| 轉換前 DTO                                                             | 轉換後顯示               |
| ---------------------------------------------------------------------- | ------------------------ |
| `correlation.subject.destination.identifier: "TX_DESTINATION_ADDRESS"` | `TX_DESTINATION_ADDRESS` |
| `correlation.subject.destination: null`                                | `-`                      |

## Chain

DTO 對應：

```ts
row.correlation?.subject.source?.rail;
```

顯示邏輯：將 `correlation.subject.source.rail` 轉成使用者可讀的鏈名稱。

目前支援：

| DTO rail | 最後顯示文字 |
| -------- | ------------ |
| `tron`   | `TRON`       |
| 無值     | `-`          |

範例：

| 轉換前 DTO                                | 轉換後顯示 |
| ----------------------------------------- | ---------- |
| `correlation.subject.source.rail: "tron"` | `TRON`     |
| `correlation.subject.source: null`        | `-`        |

## Amount (USDT)

DTO 對應：

```ts
row.amountGross;
```

顯示邏輯：顯示 `amountGross` 的數字文字，固定顯示到小數點第二位。

範例：

| 轉換前 DTO              | 轉換後顯示 |
| ----------------------- | ---------- |
| `amountGross: "10"`     | `10.00`    |
| `amountGross: "10.5"`   | `10.50`    |
| `amountGross: "10.567"` | `10.57`    |
| `amountGross` 無值      | `-`        |

## Handling Fee (USDT)

DTO 對應：

```ts
row.amountFee;
```

顯示邏輯：顯示 `amountFee` 的數字文字，固定顯示到小數點第二位。

範例：

| 轉換前 DTO           | 轉換後顯示 |
| -------------------- | ---------- |
| `amountFee: "0"`     | `0.00`     |
| `amountFee: "0.2"`   | `0.20`     |
| `amountFee: "0.235"` | `0.24`     |
| `amountFee` 無值     | `-`        |

## Net Amount (USDT)

DTO 對應：

```ts
row.amountNet;
```

顯示邏輯：顯示 `amountNet` 的數字文字，固定顯示到小數點第二位。

範例：

| 轉換前 DTO           | 轉換後顯示 |
| -------------------- | ---------- |
| `amountNet: "9.8"`   | `9.80`     |
| `amountNet: "1000"`  | `1,000.00` |
| `amountNet: "9.876"` | `9.88`     |
| `amountNet` 無值     | `-`        |

## Created At

DTO 對應：

```ts
row.createdAt;
```

顯示邏輯：將 UTC datetime 依商戶時區格式化成日期時間文字。預設格式為：

```txt
MMM D, YYYY HH:mm:ss
```

範例：

| 轉換前 DTO                          | 轉換後顯示                                   |
| ----------------------------------- | -------------------------------------------- |
| `createdAt: "2026-03-01T10:20:00Z"` | 依商戶時區顯示為 `MMM D, YYYY HH:mm:ss` 格式 |
| `createdAt` 無值                    | `-`                                          |

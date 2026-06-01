# WalletDto 欄位對照

本文整理目前對內 merchant wallet API 使用的 `WalletDto` 與相關 DTO 欄位，供後續設計 external-facing wallet DTO 時取捨參考。

目前對內 merchant wallet API 來源：

```text
GET /wallet/merch/wallets
GET /wallet/merch/wallets/{id}
```

## 目前對內 WalletDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `id` | `ShortId` | Wallet ID。 |  |
| `type` | `WalletType` | Wallet 類型，例如 merchant wallet、platform treasury。 |  |
| `merchant` | `RelatedMerchantDto \| null` | Wallet 所屬商戶摘要；非商戶 wallet 時可能為 `null`。 |  |
| `networkEndpoint` | `NetworkEndpointDto` | Wallet 對應的 network endpoint。 |  |
| `createdAt` | `IsoDateTimeUtc` | Wallet record 建立時間。 |  |
| `updatedAt` | `IsoDateTimeUtc` | Wallet record 最後更新時間。 |  |
| `assets` | `WalletAssetItemDto[]` | Wallet 持有的資產列表與餘額摘要。 |  |

## 目前對內 WalletAssetItemDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `currencyCode` | `CurrencyCode` | 資產幣別，例如 `USDT`、`TRX`。 |  |
| `nickname` | `string` | Wallet asset 顯示名稱或暱稱。 |  |
| `balance` | `WalletAssetBalanceDto` | 此資產在 wallet 中的操作型餘額。 |  |
| `createdAt` | `IsoDateTimeUtc` | Wallet asset record 建立時間。 |  |
| `updatedAt` | `IsoDateTimeUtc` | Wallet asset record 最後更新時間。 |  |

## 目前對內 WalletAssetBalanceDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `availableAmount` | `NumericString` | 可用餘額。 |  |
| `lockedAmount` | `NumericString` | 鎖定餘額。 |  |
| `heldAmount` | `NumericString` | 保留餘額。 |  |
| `heldLockedAmount` | `NumericString` | 已保留且鎖定的餘額。 |  |
| `computedAt` | `IsoDateTimeUtc` | 系統計算此餘額的時間。 |  |

## 目前對內 NetworkEndpointDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `id` | `ShortId` | Network endpoint ID。 |  |
| `type` | `NetworkType` | Network 類型，例如 blockchain。 |  |
| `provider` | `NetworkProvider` | Network provider，例如 tron。 |  |
| `identifier` | `string` | Provider 上的 endpoint identifier，例如鏈上地址。 |  |
| `status` | `NetworkEndpointStatus` | Network endpoint 狀態。 |  |
| `blockchainDetail` | `NetworkEndpointBlockchainDetailDto \| null` | Blockchain-specific endpoint detail；非 blockchain endpoint 時為 `null`。 |  |

## 目前對內 NetworkEndpointBlockchainDetailDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `chainId` | `BlockchainId` | Blockchain chain identifier，例如 `tron:mainnet`。 |  |
| `address` | `string` | 鏈上地址。 |  |
| `path` | `string` | HD wallet derivation path。 |  |
| `account` | `Integer` | HD wallet account index。 |  |
| `index` | `Integer` | HD wallet address index。 |  |

## 目前對內 MerchantSearchWalletParamsDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `idIn` | `string[] \| undefined` | 依多個 wallet ID 篩選。 |  |
| `currencyCode` | `CurrencyCode \| undefined` | 依資產幣別篩選 wallet。 |  |

此 DTO 繼承 `OffsetPagingParamsDto`，因此也會帶分頁參數。

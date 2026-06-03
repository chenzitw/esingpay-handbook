# ExternalWalletDto 欄位設計

本文定義 external `GET /external/v1/wallets` 與 `GET /external/v1/wallets/{id}` 可使用的 Wallet DTO 草案。

設計目標：

- 以目前 internal `WalletDto` 欄位為基礎做刪減，不新增 external-only 欄位，例如 `rail`、`kind`。
- 不暴露 top-level `type`、`merchant`。
- 不暴露 `networkEndpoint.status`，避免商戶依賴 internal endpoint lifecycle。
- `blockchainDetail` 僅在此 wallet 對應 crypto / blockchain endpoint 時出現；非 blockchain endpoint 為 `null`。
- `bankDetail` 作為未來 bank wallet 的擴充點；非 bank endpoint 為 `null`。
- `assets.balance` 僅保留商戶 API 整合需要的可用與鎖定餘額，不暴露 provider / internal settlement 使用的 held 欄位。

## ExternalWalletDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `id` | `string` | Wallet ID。商戶建立 withdrawal intent 時可使用此 ID 作為 `walletId`。 |
| `networkEndpoint` | `ExternalWalletNetworkEndpointDto` | Wallet 對應的 network endpoint。 |
| `createdAt` | `IsoDateTimeUtc` | Wallet record 建立時間。 |
| `updatedAt` | `IsoDateTimeUtc` | Wallet record 最後更新時間。 |
| `assets` | `ExternalWalletAssetDto[]` | Wallet 持有的資產列表與餘額摘要。 |

## ExternalWalletNetworkEndpointDto

此 DTO 沿用 internal `NetworkEndpointDto` 的主要欄位，但拿掉 `status`。

| Key | Type | 代表意義 |
| --- | --- | --- |
| `type` | `string` | Network endpoint 類型，例如 `blockchain`。 |
| `provider` | `string` | Network provider，例如 `tron`。 |
| `identifier` | `string` | Provider 上的 endpoint identifier。Crypto wallet 通常是鏈上地址；bank wallet 可是 masked bank account 或 provider account reference。 |
| `blockchainDetail` | `ExternalWalletNetworkEndpointBlockchainDetailDto \| null` | Blockchain-specific endpoint detail；非 blockchain endpoint 時為 `null`。 |
| `bankDetail` | `ExternalWalletNetworkEndpointBankDetailDto \| null` | Bank-specific endpoint detail；非 bank endpoint 時為 `null`。目前是未來 bank wallet 擴充欄位。 |

## ExternalWalletNetworkEndpointBlockchainDetailDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `chainId` | `string` | Blockchain chain identifier，例如 `tron:mainnet`。 |
| `address` | `string` | 鏈上地址。 |

Internal `NetworkEndpointBlockchainDetailDto` 內的 `path`、`account`、`index` 屬於 HD wallet / key derivation 實作資訊，external DTO 不暴露。

## ExternalWalletNetworkEndpointBankDetailDto (future)

`bankDetail` 對應未來 bank wallet 的 bank-specific endpoint 資訊。此 DTO 先定義 external response shape，實際 mapping 需要等 internal `NetworkEndpointDto` 或 bank endpoint model 補上對應欄位。

| Key | Type | 代表意義 |
| --- | --- | --- |
| `countryCode` | `string \| null` | 銀行帳戶國別，例如 `US`、`HK`。 |
| `routingCode` | `string \| null` | 銀行路由碼，例如 SWIFT/BIC、ABA、bank code。 |
| `accountNumber` | `string` | 銀行帳號或 masked account number。 |
| `accountName` | `string \| null` | 銀行帳戶名稱。 |

擴充方式：

- Crypto wallet：`type = blockchain`，使用 `blockchainDetail`，`bankDetail = null`。
- Bank wallet：`type = bank`，使用 `bankDetail`，`blockchainDetail = null`。
- 若未來有不同 bank rail，例如 `swift`、`ach`、`sepa`，先由 `provider` 或 internal bank endpoint model 決定對外值，不在 wallet DTO 額外新增 `rail`。
- 若未來 bank endpoint 需要更多欄位，例如 `branchCode`、`iban`、`beneficiaryAddress`，優先加在 `bankDetail` 內，避免污染 top-level `networkEndpoint`。

## ExternalWalletAssetDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `currencyCode` | `string` | 資產幣別，例如 `USDT`、`TRX`、`USD`。 |
| `nickname` | `string` | Wallet asset 顯示名稱或暱稱。 |
| `balance` | `ExternalWalletAssetBalanceDto` | 此資產在 wallet 中的操作型餘額。 |
| `createdAt` | `IsoDateTimeUtc` | Wallet asset record 建立時間。 |
| `updatedAt` | `IsoDateTimeUtc` | Wallet asset record 最後更新時間。 |

## ExternalWalletAssetBalanceDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `availableAmount` | `NumericString` | 可用餘額。 |
| `lockedAmount` | `NumericString` | 已鎖定餘額，例如已建立但尚未完成的 withdrawal intent 佔用金額。 |

Internal `WalletAssetBalanceDto` 內的 `heldAmount`、`heldLockedAmount` 偏向 provider / internal settlement 使用資料，external DTO 先不暴露。

## Crypto Wallet Example

```json
{
  "id": "BO6EW2NM",
  "networkEndpoint": {
    "type": "blockchain",
    "provider": "tron",
    "identifier": "TWKDTBuJDqT5H5WhcMRHXvHj9rTsTp4jBp",
    "blockchainDetail": {
      "chainId": "tron:mainnet",
      "address": "TWKDTBuJDqT5H5WhcMRHXvHj9rTsTp4jBp"
    },
    "bankDetail": null
  },
  "createdAt": "2026-05-01T00:00:00.000Z",
  "updatedAt": "2026-05-25T03:00:00.000Z",
  "assets": [
    {
      "currencyCode": "USDT",
      "nickname": "USDT TRON",
      "balance": {
        "availableAmount": "1000.000000",
        "lockedAmount": "100.000000",
      },
      "createdAt": "2026-05-01T00:00:00.000Z",
      "updatedAt": "2026-05-25T03:00:00.000Z"
    }
  ]
}
```

## Bank Wallet Example

Bank wallet 沿用同一組 `networkEndpoint` 欄位，並透過 `bankDetail` 放置 bank-specific 資訊。這樣未來支援 fiat wallet 時，不需要改動 wallet top-level shape。

```json
{
  "id": "WA00USD001",
  "networkEndpoint": {,
    "type": "bank",
    "provider": "swift",
    "identifier": "****1234",
    "blockchainDetail": null,
    "bankDetail": {
      "countryCode": "US",
      "routingCode": "CHASUS33",
      "accountNumber": "****1234",
      "accountName": "ACME Trading Inc."
    }
  },
  "createdAt": "2026-05-01T00:00:00.000Z",
  "updatedAt": "2026-05-25T03:00:00.000Z",
  "assets": [
    {
      "currencyCode": "USD",
      "nickname": "USD SWIFT",
      "balance": {
        "availableAmount": "10000.00",
        "lockedAmount": "500.00",
      },
      "createdAt": "2026-05-01T00:00:00.000Z",
      "updatedAt": "2026-05-25T03:00:00.000Z"
    }
  ]
}
```

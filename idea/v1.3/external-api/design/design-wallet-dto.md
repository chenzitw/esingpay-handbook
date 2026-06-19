# ExternalWalletDto 欄位設計

本文定義 external `GET /external/v1/wallets` 與 `GET /external/v1/wallets/{id}` 可使用的 Wallet DTO 草案。

設計目標：

- 以目前 internal `WalletDto` 欄位為基礎做刪減，不新增 external-only 欄位，例如 `rail`、`kind`。
- 不暴露 top-level `type`、`merchant`。
- 不暴露 `networkEndpoint.status`，避免商戶依賴 internal endpoint lifecycle。
- `blockchainDetail` 僅在此 wallet 對應 crypto / blockchain endpoint 時出現；非 blockchain endpoint 為 `null`。
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

## ExternalWalletNetworkEndpointBlockchainDetailDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `chainId` | `string` | Blockchain chain identifier，例如 `tron:mainnet`。 |
| `address` | `string` | 鏈上地址。 |

Internal `NetworkEndpointBlockchainDetailDto` 內的 `path`、`account`、`index` 屬於 HD wallet / key derivation 實作資訊，external DTO 不暴露。

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
    }
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

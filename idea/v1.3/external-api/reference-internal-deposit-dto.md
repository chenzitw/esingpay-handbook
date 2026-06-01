# DepositDto 欄位對照

本文整理目前對內 `DepositDto` 欄位，供後續設計 external-facing DTO 時取捨參考。

## 目前對內 DepositDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `id` | `ShortId` | Deposit 業務單 ID，也就是這筆入金紀錄的識別碼。 |  |
| `recipientMerchant` | `RelatedMerchantDto \| null` | 入金接收方若是商戶，這裡會帶商戶摘要資料。 |  |
| `recipientWallet` | `RelatedWalletDto` | 入金接收 wallet。 |  |
| `recipientNetworkEndpoint` | `RelatedNetworkEndpointDto` | 入金接收 wallet 對應的 network endpoint，例如鏈上地址或其他 rail endpoint。 |  |
| `originNetworkTransaction` | `CorrelatedNetworkTransactionDto` | 觸發這筆入金辨識的外部 network transaction。 |  |
| `source` | `NetworkSourceDto` | 入金來源，例如鏈上來源地址、未來 bank source 或其他 payment rail source。 |  |
| `currencyCode` | `CurrencyCode` | 入金資產幣別，例如 `USDT`、`TRX`。 |  |
| `amount` | `NumericString` | 入金原始金額。 |  |
| `amountNet` | `NumericString` | 扣除手續費後的入金淨額。 |  |
| `amountFee` | `NumericString` | 此筆入金收取的手續費。 |  |
| `status` | `DepositStatus` | Deposit 業務狀態。 |  |
| `recognizedAt` | `IsoDateTimeUtc` | 系統辨識到此筆入金並建立 Deposit 的時間。 |  |
| `finalizedAt` | `IsoDateTimeUtc \| null` | Deposit 進入最終狀態的時間；尚未最終化時為 `null`。 |  |
| `statusHistory` | `DepositStatusHistoryItemDto[]` | Deposit 狀態變更歷史。 |  |
| `createdAt` | `IsoDateTimeUtc` | Deposit record 建立時間。 |  |
| `updatedAt` | `IsoDateTimeUtc` | Deposit record 最後更新時間。 |  |

## 目前對內 DepositStatusHistoryItemDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `sequence` | `Integer` | 第幾次狀態變更，供排序與審計使用。 |  |
| `status` | `DepositStatus` | 變更後的 Deposit 狀態。 |  |
| `occurredAt` | `IsoDateTimeUtc` | 狀態變更發生時間。 |  |
| `actor` | `RelatedActorDto` | 造成這次狀態變更的人或系統。 |  |
| `reason` | `string \| null` | 狀態變更原因；沒有則為 `null`。 |  |

## RelatedMerchantDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `id` | `Uuid` | 商戶 ID。 |  |
| `name` | `string` | 商戶名稱。 |  |

## RelatedWalletDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `id` | `ShortId` | Wallet ID。 |  |
| `type` | `WalletType` | Wallet 類型，例如 merchant wallet、platform treasury。 |  |
| `merchantId` | `Uuid \| null` | Wallet 所屬商戶 ID；非商戶 wallet 時可能為 `null`。 |  |
| `networkEndpointId` | `ShortId` | Wallet 對應的 network endpoint ID。 |  |
| `createdAt` | `IsoDateTimeUtc` | Wallet 建立時間。 |  |

## RelatedNetworkEndpointDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `id` | `ShortId` | Network endpoint ID。 |  |
| `type` | `NetworkType` | Network 類型，例如 blockchain。 |  |
| `provider` | `NetworkProvider` | Network provider，例如 tron。 |  |
| `identifier` | `string` | Provider 上的 endpoint identifier，例如鏈上地址。 |  |
| `status` | `NetworkEndpointStatus` | Network endpoint 狀態。 |  |

## NetworkSourceDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `type` | `NetworkSourceType` | 金流來源類型，例如 crypto。 |  |
| `rail` | `NetworkSourceRail` | 金流來源管道，例如 tron、swift。 |  |
| `identifier` | `string` | 來源識別值，例如鏈上地址或 bank account reference。 |  |

## NetworkDestinationDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `type` | `NetworkDestinationType` | 金流目的地類型，例如 crypto。 |  |
| `rail` | `NetworkDestinationRail` | 金流目的地管道，例如 tron、swift。 |  |
| `routingCode` | `string` | 目的地路由識別，例如 chain id、bank routing code、SWIFT/BIC。 |  |
| `identifier` | `string` | 目的地識別值，例如鏈上地址或銀行帳號。 |  |

## CorrelatedNetworkTransactionDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `id` | `ShortId` | Network transaction ID。 |  |
| `type` | `NetworkType` | Network 類型，例如 blockchain。 |  |
| `provider` | `NetworkProvider` | Network provider，例如 tron。 |  |
| `identifier` | `string` | Provider 上的交易識別，例如 transaction hash 或 provider reference。 |  |
| `status` | `NetworkTransactionStatus` | Network transaction 狀態。 |  |
| `direction` | `NetworkTransactionDirection` | Network transaction 方向，例如 inbound、outbound。 |  |
| `occurredAt` | `IsoDateTimeUtc` | 外部交易發生時間。 |  |
| `finalizedAt` | `IsoDateTimeUtc \| null` | 外部交易最終化時間；尚未最終化時為 `null`。 |  |
| `senderEndpoint` | `RelatedNetworkEndpointDto \| null` | 系統已知的 sender endpoint；未知或外部端點時可能為 `null`。 |  |
| `recipientEndpoint` | `RelatedNetworkEndpointDto \| null` | 系統已知的 recipient endpoint；未知或外部端點時可能為 `null`。 |  |
| `feePayerEndpoint` | `RelatedNetworkEndpointDto \| null` | 支付 network fee 的 endpoint；不適用或未知時可能為 `null`。 |  |
| `source` | `NetworkSourceDto \| null` | Network transaction 來源資訊。 |  |
| `destination` | `NetworkDestinationDto \| null` | Network transaction 目的地資訊。 |  |
| `blockchainDetail` | `NetworkTransactionBlockchainDetailDto \| null` | Blockchain-specific transaction detail；非 blockchain transaction 時為 `null`。 |  |
| `legs` | `NetworkTransactionLegDto[]` | Transaction legs，描述 principal、network fee 等不同資產金額。 |  |
| `observedAt` | `IsoDateTimeUtc` | 系統觀測到此交易的時間。 |  |
| `syncedAt` | `IsoDateTimeUtc` | 系統同步此交易狀態的時間。 |  |
| `createdAt` | `IsoDateTimeUtc` | Network transaction record 建立時間。 |  |
| `updatedAt` | `IsoDateTimeUtc` | Network transaction record 最後更新時間。 |  |

## NetworkTransactionBlockchainDetailDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `chainId` | `BlockchainId` | Blockchain chain identifier，例如 `tron:mainnet`。 |  |
| `txHash` | `string` | Blockchain transaction hash。 |  |
| `fromAddress` | `string` | 鏈上來源地址。 |  |
| `toAddress` | `string` | 鏈上目的地址。 |  |
| `feePayerAddress` | `string` | 鏈上支付 network fee 的地址。 |  |
| `gasAssetId` | `BlockchainAssetId` | 支付 gas / network fee 的鏈上資產識別。 |  |
| `gasPrice` | `NumericString` | Gas price。 |  |
| `gasLimit` | `NumericString` | Gas limit。 |  |
| `gasUsed` | `NumericString` | 實際使用 gas。 |  |

## NetworkTransactionLegDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `type` | `NetworkTransactionLegType` | Transaction leg 類型，例如 principal、network fee。 |  |
| `currencyCode` | `CurrencyCode` | 此 leg 的資產幣別。 |  |
| `amount` | `NumericString` | 此 leg 的金額。 |  |

## RelatedActorDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `type` | `ActorType` | Actor 類型，例如 system、platform user、merchant user。 |  |
| `identifier` | `string \| null` | Actor identifier；system 或無使用者識別時可能為 `null`。 |  |
| `name` | `string` | Actor 顯示名稱。 |  |
| `email` | `Email \| null` | Actor email；system 或無 email 時可能為 `null`。 |  |

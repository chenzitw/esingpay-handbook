# ExternalDepositDto 欄位設計

本文定義 external `GET /external/v1/deposits` 與 `GET /external/v1/deposits/{id}` 可使用的 Deposit DTO 草案。

設計目標：

- 支援多幣種與不同 payment rail，例如 crypto、bank。
- 結構比 internal DTO 簡化，但保留對帳與交易追蹤需要的細節。
- 不直接暴露 internal `DepositDto`、`CorrelatedNetworkdepositTransactionDto`、endpoint lifecycle、actor、status history。

## List Query Params

`GET /external/v1/deposits` 第一版查詢參數：

| Key | Type | 代表意義 |
| --- | --- | --- |
| `page` | `number` | Offset paging page。 |
| `size` | `number` | Offset paging size。 |
| `statusIn` | `ExternalDepositStatus[]` | 以對外 Deposit 狀態過濾。 |

本版先不提供 wallet id filter。若後續新增 wallet filter，應明確定義它對應 `ExternalDepositDto.wallet.id`，而不是 internal wallet numeric id。

## ExternalDepositDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `id` | `string` | Deposit 業務單 ID，也就是這筆入金紀錄的識別碼。 |
| `wallet` | `WalletDto` | 入金接收 wallet 的 external 摘要。 |
| `transaction` | `DepositTransactionDto` | 本筆 deposit 使用的 payment / network / transaction 摘要。 |
| `source` | `ExternalDepositSourceDto` | 入金來源。這是外部付款方，只保留觀測到的基本資訊。 |
| `status` | `ExternalDepositStatus` | 對外 Deposit 狀態。 |
| `currencyCode` | `CurrencyCode` | 出金資產幣別，例如 `USDT`、`TRX`。 |  |
| `amount` | `NumericString` | 入金原始金額。 |
| `amountFee` | `NumericString` | 此筆入金收取的手續費。 |
| `amountNet` | `NumericString` | 扣除手續費後的入金淨額。 |
| `finalizedAt` | `IsoDateTimeUtc \| null` | Deposit 進入最終狀態的時間；尚未最終化時為 `null`。 |
| `createdAt` | `IsoDateTimeUtc` | Deposit record 建立時間。 |
| `updatedAt` | `IsoDateTimeUtc` | Deposit record 最後更新時間。 |

## WalletDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `id` | `string` | Wallet ID。 |
| `type` | `string` | Wallet 類型。 |

## DepositTransactionDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `type` | `'blockchain' \| 'bank'` | Payment rail 類型。 |
| `provider` | `string` | Network provider，例如 `tron`、`solana`、`swift`、`stripe`。 |
| `blockchain` | `TransactionBlockchainDetailDto \| null` | Blockchain-specific transaction detail。 |
| `bank` | `null` | 第一版不提供 bank-specific transfer detail；bank detail 之後另開文件補規格。 |
| `legs` | `ExternalLegsDto[]` | 此交易實際造成的資產流動明細，例如 principal、network fee。 |


### TransactionBlockchainDetailDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `chainId` | `string` | Blockchain chain identifier，例如 `tron:mainnet`。 |
| `hash` | `string` | Blockchain transaction hash。 |
| `fromAddress` | `string` | 鏈上來源地址。 |
| `toAddress` | `string` | 鏈上目的地址。 |

### Bank Detail

第一版 `DepositTransactionDto.bank` 固定為 `null`，不定義 `TransactionBankDetailDto`。

後續若要支援 bank rail detail，另開設計文件定義欄位；在此之前 implementation doc 可省略 bank detail DTO。

### ExternalLegsDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `type` | `'principal' \| 'network_fee'` | 此交易造成的資產流動類型。 |
| `currencyCode` | `CurrencyCode` | 此 leg 金額所屬幣種。 |
| `amount` | `NumericString` | 此 legs 的金額。 |

## ExternalDepositSourceDto

`source` 是外部付款來源

| Key | Type | 代表意義 |
| --- | --- | --- |
| `type` | `string` | 金流來源類型，例如 `crypto`、`bank`。 |
| `rail` | `string` | 金流來源管道，例如 `tron`、`solana`。 |
| `identifier` | `string` | 來源識別值，例如鏈上 from address、bank account。 |

## ExternalDepositStatus

| Value | 代表意義 |
| --- | --- |
| `pending` | Deposit 已建立但尚未完成。 |
| `completed` | Deposit 已完成。 |
| `failed` | Deposit 失敗或無效。 |

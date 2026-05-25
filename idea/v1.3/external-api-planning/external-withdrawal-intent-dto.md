# ExternalWithdrawalIntentDto 欄位設計

本文定義 external `GET /external/v1/withdrawal-intents` 與 `GET /external/v1/withdrawal-intents/{id}` 可使用的 Withdrawal Intent DTO 草案。

設計目標：

- 與 `ExternalDepositDto` 保持接近的 response shape。
- 支援多幣種與不同 payment rail，例如 crypto、bank。
- 保留對帳與交易追蹤需要的 transaction detail / legs
- 不直接暴露 internal `WithdrawalIntentDto`、完整 `CorrelatedNetworkTransactionDto`、actor、status history、fee payer strategy。

## ExternalWithdrawalIntentDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `id` | `string` | Withdrawal Intent 業務單 ID，也就是這筆出金申請的識別碼。 |
| `wallet` | `WalletDto` | 出金扣款來源 wallet 的 external 摘要。 |
| `transaction` | `WithdrawalTransactionDto` | 本筆 withdrawal intent 使用的 payment rail / network / transaction 摘要。 |
| `destination` | `ExternalWithdrawalDestinationDto` | 出金收款方 / 目的地。
| `status` | `WithdrawalIntentStatus` | 對外 Withdrawal Intent 狀態。 |
| `amount` | `NumericString` | 出金申請原始金額。 |
| `amountFee` | `NumericString` | 此筆出金收取的手續費。 |
| `amountNet` | `NumericString` | 扣除手續費後實際送往目的地的淨額。 |
| `submittedAt` | `IsoDateTimeUtc` | 出金申請提交時間。 |
| `finalizedAt` | `IsoDateTimeUtc \| null` | Withdrawal Intent 進入最終狀態的時間；尚未最終化時為 `null`。 |
| `createdAt` | `IsoDateTimeUtc` | Withdrawal Intent record 建立時間。 |
| `updatedAt` | `IsoDateTimeUtc` | Withdrawal Intent record 最後更新時間。 |

## WalletDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `id` | `string` | Wallet ID。 |
| `type` | `string` | Wallet 類型。 |


## WithdrawalTransactionDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `type` | `'blockchain' \| 'bank'` | Payment rail 類型。 |
| `provider` | `string` | Rail provider，例如 `tron`、`solana`、`swift`、`stripe`。 |
| `blockchain` | `BlockchainTransactionDetailDto \| null` | Blockchain-specific transaction detail。 |
| `bank` | `BankTransferDetailDto \| null` | Bank-specific transfer detail。 |
| `legs` | `ExternalLegsDto[]` | 此交易實際造成的資產流動明細，例如 principal、network fee。 |

### BlockchainTransactionDetailDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `chainId` | `string` | Blockchain chain identifier，例如 `tron:mainnet`。 |
| `hash` | `string` | Blockchain transaction hash。 |
| `fromAddress` | `string` | 鏈上來源地址。 |
| `toAddress` | `string` | 鏈上目的地址。 |

### BankTransferDetailDto (future)

| Key | Type | 代表意義 |
| --- | --- | --- |
| `rail` | `string` | Bank transfer rail，例如 `swift`、`ach`、`sepa`。 |
| `providerReferenceId` | `string \| null` | Provider reference id。 |
| `bankReferenceId` | `string \| null` | Bank-side reference id。 |
| `senderBankCode` | `string \| null` | Sender bank code。 |
| `recipientBankCode` | `string \| null` | Recipient bank code。 |

### ExternalLegsDto

| Key | Type | 代表意義 |
| --- | --- | --- |
| `type` | `'principal' \| 'network_fee' \| 'service_fee' \| 'adjustment'` | 此交易造成的資產流動類型。 |
| `currencyCode` | `string` | 此 effect 對應的資產代碼。 |
| `rail` | `string \| null` | 此 effect 對應的 rail。 |
| `network` | `string \| null` | 此 effect 對應的 network。 |
| `amount` | `NumericString` | 此 effect 的金額。 |


## ExternalWithdrawalDestinationDto

`destination` 是商戶指定或外部收款方

| Key | Type | 代表意義 |
| --- | --- | --- |
| `type` | `string` | 金流終端類型，例如 `crypto`、`bank`。 |
| `rail` | `string` | 金流終端管道，例如 `tron`、`swift`。 |
| `routingCode` | `string` | 路由識別值，例如 chain id `tron:mainnet` 或 bank routing code。 |
| `identifier` | `string` | 目的地識別值，例如鏈上 to address、bank account reference。 |

## WithdrawalIntentStatus

| Value | 代表意義 |
| --- | --- |
| `pending` | Withdrawal intent 已建立，仍在內部審核、處理或等待送出。 |
| `completed` | 出金完成。 |
| `failed` | 出金失敗。 |
| `cancelled` | 出金已取消。 |

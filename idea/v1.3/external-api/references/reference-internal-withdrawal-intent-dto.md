# WithdrawalIntentDto 欄位對照

本文整理目前對內 `WithdrawalIntentDto` 欄位，供後續設計 external-facing DTO 時取捨參考。

## 目前對內 WithdrawalIntentDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `id` | `ShortId` | Withdrawal Intent 業務單 ID，也就是這筆出金申請的識別碼。 |  |
| `category` | `WithdrawalIntentCategory` | 出金類型，例如 merchant wallet 出金或 platform wallet 出金。 |  |
| `senderMerchant` | `RelatedMerchantDto \| null` | 出金來源若是商戶，這裡會帶商戶摘要資料。 |  |
| `senderWallet` | `RelatedWalletDto` | 出金扣款來源 wallet。 |  |
| `senderNetworkEndpoint` | `RelatedNetworkEndpointDto` | 出金來源 wallet 對應的 network endpoint。 |  |
| `feePayerWallet` | `RelatedWalletDto` | 支付 network fee 的 wallet。 |  |
| `targetNetworkTransaction` | `CorrelatedNetworkTransactionDto \| null` | 實際送到外部 network 後產生的 network transaction；尚未建立時為 `null`。 |  |
| `destination` | `NetworkDestinationDto` | 出金目的地，例如鏈上地址、未來 bank account 或其他 payment rail destination。 |  |
| `currencyCode` | `CurrencyCode` | 出金資產幣別，例如 `USDT`、`TRX`。 |  |
| `amount` | `NumericString` | 出金申請原始金額。 |  |
| `amountNet` | `NumericString` | 扣除手續費後實際送往目的地的淨額。 |  |
| `amountFee` | `NumericString` | 此筆出金收取的手續費。 |  |
| `status` | `WithdrawalIntentStatus` | Withdrawal Intent 業務狀態。 |  |
| `submittedBy` | `RelatedActorDto` | 建立這筆 withdrawal intent 的 actor。 |  |
| `submittedAt` | `IsoDateTimeUtc` | 出金申請提交時間。 |  |
| `cancellableUntil` | `IsoDateTimeUtc` | 可取消期限。 |  |
| `processedAt` | `IsoDateTimeUtc \| null` | 內部處理完成時間；尚未處理完成時為 `null`。 |  |
| `instructedAt` | `IsoDateTimeUtc \| null` | 已向外部 network 下達交易指令的時間；尚未下達時為 `null`。 |  |
| `finalizedAt` | `IsoDateTimeUtc \| null` | Withdrawal Intent 進入最終狀態的時間；尚未最終化時為 `null`。 |  |
| `statusHistory` | `WithdrawalIntentStatusHistoryItemDto[]` | Withdrawal Intent 狀態變更歷史。 |  |
| `createdAt` | `IsoDateTimeUtc` | Withdrawal Intent record 建立時間。 |  |
| `updatedAt` | `IsoDateTimeUtc` | Withdrawal Intent record 最後更新時間。 |  |

## 目前對內 WithdrawalIntentStatusHistoryItemDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `sequence` | `Integer` | 第幾次狀態變更，供排序與審計使用。 |  |
| `status` | `WithdrawalIntentStatus` | 變更後的 Withdrawal Intent 狀態。 |  |
| `occurredAt` | `IsoDateTimeUtc` | 狀態變更發生時間。 |  |
| `actor` | `RelatedActorDto` | 造成這次狀態變更的人或系統。 |  |
| `reason` | `string \| null` | 狀態變更原因；沒有則為 `null`。 |  |

## Nested DTO

Nested DTO 欄位可先沿用 [`internal-deposit-dto.md`](./internal-deposit-dto.md) 的共用表格：

- `RelatedMerchantDto`
- `RelatedWalletDto`
- `RelatedNetworkEndpointDto`
- `NetworkDestinationDto`
- `CorrelatedNetworkTransactionDto`
- `RelatedActorDto`

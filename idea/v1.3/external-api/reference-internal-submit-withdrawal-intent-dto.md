# POST Internal Withdrawal Intents DTO 欄位對照

本文整理目前對內建立 withdrawal intent 使用的 DTO，供後續設計 external `POST /external/v1/withdrawal-intents` request DTO 時取捨參考。

目前對內 DTO：

- `MerchantSubmitWithdrawalIntentDto`

兩者目前欄位相同。

## MerchantSubmitWithdrawalIntentDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `senderWalletId` | `ShortId` | 出金扣款來源 wallet ID。 |  |
| `destination` | `NetworkDestinationDto` | 出金目的地，例如鏈上地址、未來 bank account 或其他 payment rail destination。 |  |
| `currencyCode` | `SupportedCurrencyCode` | 出金資產幣別，目前限定在系統支援的 currency code。 |  |
| `amount` | `NumericString` | 出金申請原始金額。 |  |

## NetworkDestinationDto 欄位對照

| Key | Type | 代表意義 | Notes |
| --- | --- | --- | --- |
| `type` | `NetworkDestinationType` | 金流目的地類型，例如 crypto。 |  |
| `rail` | `NetworkDestinationRail` | 金流目的地管道，例如 tron、swift。 |  |
| `routingCode` | `string` | 目的地路由識別，例如 chain id、bank routing code、SWIFT/BIC。 |  |
| `identifier` | `string` | 目的地識別值，例如鏈上地址或銀行帳號。 |  |

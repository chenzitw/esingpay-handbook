# External Transaction Leg CurrencyCode Amendment

本文記錄 external transaction legs 在 v1.3 design 已對齊後，因 implementation review 發現需要補充的欄位調整。

## Background

External Deposit / Withdrawal Intent response 都會透過 `transaction.legs` 表達 network transaction 實際造成的資產流動明細。

原本 Withdrawal Intent 的 `ExternalLegsDto` 只包含：

```ts
type: 'principal' | 'network_fee';
amount: NumericString;
```

這會讓 response 使用者無法判斷每一筆 leg 的金額所屬幣種。例如 `principal` 和 `network_fee` 可能不是同一個 currency，若只回 `amount`，對帳與顯示都缺少必要語意。

Deposit design 原本已有 `currencyCode`，但型別使用 `string`。為了讓所有 external transaction legs 語意一致，Deposit / Withdrawal Intent 的 leg currency 都統一使用 `CurrencyCode`。

## Decision

所有 external transaction leg DTO 都應包含：

```ts
currencyCode: CurrencyCode;
```

更新後 shape：

```ts
type: 'principal' | 'network_fee';
currencyCode: CurrencyCode;
amount: NumericString;
```

## Rationale

`legs` 表示 network transaction 實際造成的資產流動明細。每一筆 leg 的 `amount` 必須和 `currencyCode` 一起出現，否則金額無法獨立解讀。

這也與 internal `NetworkTransactionLegDto` 的欄位語意一致。

## Scope

本 amendment 適用於 external response DTO 中的 transaction legs：

- `ExternalDepositDto.transaction.legs`
- `ExternalWithdrawalIntentDto.transaction.legs`

不改 `DepositDto.amount` / `amountFee` / `amountNet` 的既有語意。

不改 `WithdrawalIntentDto.amount` / `amountFee` / `amountNet` 的既有語意。

不新增 bank transfer detail。

不改 API endpoint surface。

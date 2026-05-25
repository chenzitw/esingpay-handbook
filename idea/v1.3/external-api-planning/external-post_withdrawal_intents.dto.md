# POST External Withdrawal Intents DTO 欄位設計

本文定義 external `POST /external/v1/withdrawal-intents` request DTO 草案。

## ExternalCreateWithdrawalIntentDto

```ts
export type ExternalCreateWithdrawalIntentDto = {
  walletId: string;
  currencyCode: string;
  amount: NumericString;
  destination: ExternalWithdrawalDestinationDto;
  referenceId?: string;
};
```

| Key | Type | 代表意義 |
| --- | --- | --- |
| `walletId` | `string` | 出金扣款來源 wallet ID。 |
| `currencyCode` | `string` | 出金資產幣別，例如 `USDT`、`TRX`、`USD`。 |
| `amount` | `NumericString` | 出金申請原始金額。 |
| `destination` | `ExternalWithdrawalDestinationDto` | 出金目的地。依付款方式使用不同 union variant。 |

## ExternalWithdrawalDestinationDto

```ts
export type ExternalWithdrawalDestinationDto =
  | ExternalBlockchainWithdrawalDestinationDto
  | ExternalBankWithdrawalDestinationDto;
```

## ExternalBlockchainWithdrawalDestinationDto

```ts
export type ExternalBlockchainWithdrawalDestinationDto = {
  type: 'blockchain_address';
  rail: string;
  address: string;
};
```

| Key | Type | 代表意義 |
| --- | --- | --- |
| `type` | `'blockchain_address'` | 表示目的地是 blockchain address。 |
| `rail` | `string` | Blockchain rail，例如 `tron`。實際 mainnet / testnet 由系統環境決定。 |
| `address` | `string` | 鏈上收款地址。 |

## 目前支援範圍

第一版 external withdrawal intent create 先支援：

| Currency | Rail | Production Network | Test Network |
| --- | --- | --- | --- |
| `USDT` | `tron` | `tron:mainnet` | `tron:nile` 或 `tron:shasta` |

Request 不提供 `network` 欄位。正式環境固定解析到正式鏈，測試環境固定解析到測試鏈。

## ExternalBankWithdrawalDestinationDto

```ts
export type ExternalBankWithdrawalDestinationDto = {
  type: 'bank_account';
  rail: string;
  countryCode: string;
  routingCode: string | null;
  accountNumber: string;
  accountName: string;
};
```

| Key | Type | 代表意義 |
| --- | --- | --- |
| `type` | `'bank_account'` | 表示目的地是 bank account。 |
| `rail` | `string` | Bank rail，例如 `swift`、`ach`、`sepa`、`local_bank`。 |
| `countryCode` | `string` | 銀行帳戶國別。 |
| `routingCode` | `string \| null` | 銀行路由碼，例如 SWIFT/BIC、ABA routing number、bank code。 |
| `accountNumber` | `string` | 銀行帳號。 |
| `accountName` | `string` | 銀行帳戶名稱。 |

## Tron USDT Example

```json
{
  "walletId": "BO6EW2NM",
  "currencyCode": "USDT",
  "amount": "100",
  "destination": {
    "type": "blockchain_address",
    "rail": "tron",
    "address": "TXxxxx..."
  },
  "referenceId": "WD-20260520-0001"
}
```

## Internal Mapping Notes

第一版 mapping 到目前對內 `MerchantSubmitWithdrawalIntentDto` 時：

```ts
{
  senderWalletId: dto.walletId,
  currencyCode: dto.currencyCode,
  amount: dto.amount,
  destination: {
    type: 'crypto',
    rail: dto.destination.rail,
    routingCode: resolveRoutingCodeByEnvironment(dto.destination.rail),
    identifier: dto.destination.address,
  },
}
```

第一版可先只支援：

- `destination.type = 'blockchain_address'`
- `destination.rail = 'tron'`
- `currencyCode = 'USDT'`

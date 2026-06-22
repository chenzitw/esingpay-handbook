# POST External Withdrawal Intents DTO 欄位設計

本文定義 external `POST /external/v1/withdrawal-intents` request DTO 草案。

## CreateExternalWithdrawalIntentDto

```ts
import type { NumericString, ShortId, SupportedCurrencyCode } from '@esingpay/contract-base/common';

export type CreateExternalWithdrawalIntentDto = {
  senderWalletId: ShortId;
  currencyCode: SupportedCurrencyCode;
  amount: NumericString;
  destination: CreateExternalWithdrawalDestinationDto;
};
```

| Key | Type | 代表意義 |
| --- | --- | --- |
| `senderWalletId` | `ShortId` | 出金扣款來源 wallet ID。此值來自 external wallet list/detail response 的 wallet id。 |
| `currencyCode` | `SupportedCurrencyCode` | 出金資產幣別。第一版只支援 `USDT`。 |
| `amount` | `NumericString` | 出金申請原始金額。 |
| `destination` | `CreateExternalWithdrawalDestinationDto` | 出金目的地。第一版只支援 crypto Tron address。 |

## CreateExternalWithdrawalDestinationDto

External create request 的 destination 採接近 internal `NetworkDestinationDto` 的 shape，但不接收 `routingCode`。

```ts
import type { NetworkDestinationRail, NetworkDestinationType } from '@esingpay/contract-base/network';

export type CreateExternalWithdrawalDestinationDto = {
  type: NetworkDestinationType;
  rail: NetworkDestinationRail;
  identifier: string;
};
```

| Key | Type | 代表意義 |
| --- | --- | --- |
| `type` | `NetworkDestinationType` | 金流終端類型。第一版只支援 `crypto`。 |
| `rail` | `NetworkDestinationRail` | 金流終端管道。第一版只支援 `tron`。 |
| `identifier` | `string` | 目的地識別值。Tron rail 下為鏈上收款地址。 |

## 不接收 routingCode

Request 不提供 `routingCode`。

當 `destination.rail = tron` 時，service implementation 依執行環境解析 internal `routingCode`：

- production：Tron mainnet。
- test / non-production：Tron test network。

實際 routing code resolution 屬 implementation concern，不是 external request contract 的一部分。

## 目前支援範圍

第一版 external withdrawal intent create 先支援：

| Currency | Type | Rail | Destination identifier |
| --- | --- | --- | --- |
| `USDT` | `crypto` | `tron` | Tron address |

第一版不支援：

- `referenceId`
- `Idempotency-Key`
- `bank_account`
- request-level network selection

## POST Error Response Codes

`POST /external/v1/withdrawal-intents` 的 error response code 對齊 `externalWithdrawalIntentApi.createWithdrawalIntent` contract。

| Code | HTTP status | 代表情境 |
| --- | --- | --- |
| `forbidden` | 403 | Merchant agent 無權建立此出金申請。 |
| `data_invalid` | 400 | Request body shape 或欄位值不符合 contract。 |
| `withdrawal_intent.config_incomplete` | 400 | 出金設定尚未完成，無法建立出金申請。 |
| `withdrawal_intent.destination_invalid` | 400 | 出金目的地不符合目前支援範圍或驗證失敗。 |
| `withdrawal_intent.destination_internal` | 400 | 出金目的地指向系統內部端點，不可作為 external withdrawal destination。 |
| `withdrawal_intent.balance_insufficient` | 400 | 來源 wallet 可用餘額不足。 |
| `withdrawal_intent.daily_limit_exceeded` | 400 | 申請金額超過每日出金限制。 |
| `withdrawal_intent.monthly_limit_exceeded` | 400 | 申請金額超過每月出金限制。 |

## Tron USDT Example

```json
{
  "senderWalletId": "BO6EW2NM",
  "currencyCode": "USDT",
  "amount": "100",
  "destination": {
    "type": "crypto",
    "rail": "tron",
    "identifier": "TXxxxx..."
  }
}
```

## Internal Mapping Notes

第一版 mapping 到 `SubmitWithdrawalIntentInput` 時：

```ts
{
  merchantId: identity.merchantId,
  senderWalletId: decodeShortId(dto.senderWalletId),
  currencyCode: dto.currencyCode,
  amount: dto.amount,
  destination: {
    type: dto.destination.type,
    rail: dto.destination.rail,
    routingCode: resolveRoutingCodeByEnvironment(dto.destination.rail),
    identifier: dto.destination.identifier,
  },
}
```

第一版應只接受：

- `destination.type = NetworkDestinationType.Crypto`
- `destination.rail = NetworkDestinationRail.Tron`
- `currencyCode = SupportedCurrencyCode.USDT`

不符合上述條件者，應回 external API 的 invalid request / data invalid 類錯誤。

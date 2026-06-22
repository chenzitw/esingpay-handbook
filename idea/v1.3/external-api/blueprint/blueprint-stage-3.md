---
status: draft
updated_at: 2026-06-19
updated_by: Codex
---

# External API — Stage 3 Blueprint

## 目標

Stage 3 完成 external withdrawal intent create：

- `POST /external/v1/withdrawal-intents`

這個 stage 的重點不是 copy 一份 `MerchantSubmitWithdrawalIntentUseCase`，而是把 withdrawal intent submit 的核心流程抽成共用 `WithdrawalIntentSubmitterService`。

共用 service 不決定 caller 是 merchant portal 還是 external API，也不建立 actor。它只接受上層 use case 已經決定好的 `submittedBy`。

## Boundary

Stage 3 延續 Stage 1 / Stage 2 已落地的 external API boundary：

```text
Gateway -> External = contract-rest + rest-rpc transport bridge
External -> Fund = cradle-internal facade bridge
Fund internal = use case / service / repository / composer
```

因此 external adapter 不直接 DI fund use case，也不在本 stage 新增 fund submit `contract-rpc`。

這是刻意延續前面 stages 的實作慣例：

- Stage 1 / Stage 2 external read path 目前透過 external client 呼叫 fund facade。
- Stage 3 第一版 create path 也先沿用相同 pattern，降低唯一 write API 接入時的 boundary 變更風險。
- 新的 fund submit `contract-rpc` 屬於未來 service-to-service boundary migration，不是本 stage 的必要條件。

完整 create flow 應為：

```text
POST /external/v1/withdrawal-intents
  -> apps/esing-pay-api-gateway/src/rest/external/controller/withdrawal-intent.controller.ts
  -> apps/esing-pay-api-gateway/src/rest/external/proxy/withdrawal-intent.proxy.ts
  -> rest-rpc transport bridge
  -> apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent/withdrawal-intent.controller.ts
  -> apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent/withdrawal-intent.service.ts
  -> apps/esingpay-cradle/src/external/client/withdrawal-intent.client.ts
  -> apps/esingpay-cradle/src/fund/facade/withdrawal-intent.facade.ts
  -> ExternalSubmitWithdrawalIntentUseCase
  -> WithdrawalIntentSubmitterService
  -> raw WithdrawalIntent
  -> ExternalWithdrawalIntentComposer
  -> ExternalWithdrawalIntentDto
```

Merchant portal submit 仍維持：

```text
MerchantSubmitWithdrawalIntentUseCase
  -> createMerchantUserActor(command.merchantUserId)
  -> WithdrawalIntentSubmitterService
```

兩條路徑共用 submit mechanics，但 actor strategy 各自明確。

## Actor Strategy

External API 使用 `merchant-agent` identity。它代表商戶系統透過 API key 呼叫，不是 merchant portal user。

Stage 3 第一版採用：

```ts
submittedBy: createSystemActor()
```

理由：

- 不新增 `ActorType`。
- 不新增 actor persistence 欄位。
- 不需要 migration。
- 不把 `merchantId` 或 API key id 塞進 `merchantUserId`。
- composer 不會因為 `merchantUserId` 而去 account management 查 merchant user refs。

限制：

- withdrawal intent 的 `submittedBy` 只會顯示 System。
- 無法從 actor 欄位直接知道是哪把 API key 建立。
- 若正式 audit 需要追 API key，應另外設計 API key audit / request log，不應塞進 actor。

## 共用 Submitter Service

新增 fund 內部共用 service：

```text
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/service/withdrawal-intent-submitter.service.ts
```

它的 command 應明確接受 `submittedBy`：

```ts
import type { Actor } from '@esingpay/contract-base/account';
import type { NumericString, SupportedCurrencyCode, Uuid } from '@esingpay/contract-base/common';
import type { NetworkDestination } from '@esingpay/contract-base/network';

export type SubmitWithdrawalIntentCommand = {
  merchantId: Uuid;
  senderWalletId: bigint;
  destination: NetworkDestination;
  currencyCode: SupportedCurrencyCode;
  amount: NumericString;
  submittedBy: Actor;
};
```

Service interface 概念：

```ts
@Injectable()
export class WithdrawalIntentSubmitterService {
  async submit(
    command: SubmitWithdrawalIntentCommand,
  ): Promise<Either.Either<WithdrawalIntent, SubmitWithdrawalIntentError>> {
    // 原本 MerchantSubmitWithdrawalIntentUseCase.execute() 的核心流程移到這裡。
  }
}
```

這個 service 會承接目前 `MerchantSubmitWithdrawalIntentUseCase.execute()` 的大段 submit mechanics：

- sender wallet ownership check。
- amount normalize。
- sender network endpoint lookup。
- withdrawal category derive。
- fee payer resolve。
- network fee estimate。
- withdrawal intent evaluation。
- merchant withdrawal limit lookup。
- wallet allocation reserve。
- withdrawal intent persist。
- wallet allocation bind。
- acceptance job dispatch。
- submit error convergence。

關鍵差異有兩個：

- entity 建立時不再自己建立 merchant user actor。
- submitter 成功時回傳 raw `WithdrawalIntent`，不做 `WithdrawalIntentComposed` composition。

原本：

```ts
const withdrawalIntentEntity = WithdrawalIntentEntity.createSubmitted({
  // ...
  submittedBy: createMerchantUserActor(command.merchantUserId),
  submittedAt,
  cancellableUntil,
});
```

改成：

```ts
const withdrawalIntentEntity = WithdrawalIntentEntity.createSubmitted({
  // ...
  submittedBy: command.submittedBy,
  submittedAt,
  cancellableUntil,
});
```

`WithdrawalIntentSubmitterService` 不應 import：

```ts
createMerchantUserActor
createSystemActor
```

它只接收 actor，不決定 actor。

`WithdrawalIntentSubmitterService` 也不應 inject 或呼叫 `WithdrawalIntentComposer`。composition 由 caller-specific wrapper / adapter 決定。

## Merchant Portal Use Case

既有 `MerchantSubmitWithdrawalIntentUseCase` 保留 merchant portal 語意，但變成薄 wrapper。

路徑不變：

```text
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/use-case/merchant-submit-withdrawal-intent.use-case.ts
```

Command 保留 `merchantUserId`：

```ts
export type MerchantSubmitWithdrawalIntentCommand = {
  merchantId: Uuid;
  merchantUserId: Uuid;
  senderWalletId: bigint;
  destination: NetworkDestination;
  currencyCode: SupportedCurrencyCode;
  amount: NumericString;
};
```

Use case 改為：

```ts
@Injectable()
export class MerchantSubmitWithdrawalIntentUseCase {
  constructor(
    @Inject(WithdrawalIntentSubmitterService)
    private readonly withdrawalIntentSubmitterService: WithdrawalIntentSubmitterService,
    @Inject(WithdrawalIntentComposer)
    private readonly withdrawalIntentComposer: WithdrawalIntentComposer,
  ) {}

  async execute(
    command: MerchantSubmitWithdrawalIntentCommand,
  ): Promise<Either.Either<WithdrawalIntentComposed, MerchantSubmitWithdrawalIntentError>> {
    const result = await this.withdrawalIntentSubmitterService.submit({
      merchantId: command.merchantId,
      senderWalletId: command.senderWalletId,
      destination: command.destination,
      currencyCode: command.currencyCode,
      amount: command.amount,
      submittedBy: createMerchantUserActor(command.merchantUserId),
    });

    if (Either.isLeft(result)) {
      return Either.left(result.left);
    }

    return Either.right(await this.withdrawalIntentComposer.composeOne(result.right));
  }
}
```

這代表 merchant portal actor 邏輯仍在 portal use case，不會散到共用 submitter。

## Fund External Submit Use Case

新增 fund 端給 external facade bridge 使用的 use case：

```text
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/use-case/external-submit-withdrawal-intent.use-case.ts
```

它接收 external submit command，不接收 `merchantUserId`：

```ts
export type ExternalSubmitWithdrawalIntentCommand = {
  merchantId: Uuid;
  senderWalletId: bigint;
  destination: NetworkDestination;
  currencyCode: SupportedCurrencyCode;
  amount: NumericString;
};
```

Use case 概念：

```ts
@Injectable()
export class ExternalSubmitWithdrawalIntentUseCase {
  constructor(
    @Inject(WithdrawalIntentSubmitterService)
    private readonly withdrawalIntentSubmitterService: WithdrawalIntentSubmitterService,
  ) {}

  async execute(
    command: ExternalSubmitWithdrawalIntentCommand,
  ): Promise<Either.Either<WithdrawalIntent, ExternalSubmitWithdrawalIntentError>> {
    return this.withdrawalIntentSubmitterService.submit({
      merchantId: command.merchantId,
      senderWalletId: command.senderWalletId,
      destination: command.destination,
      currencyCode: command.currencyCode,
      amount: command.amount,
      submittedBy: createSystemActor(),
    });
  }
}
```

這裡可以建立 `createSystemActor()`，因為這個 use case 已經代表 external caller strategy。

## Fund Facade Bridge

Stage 3 第一版延續 Stage 1 / Stage 2 的 facade pattern，在 fund facade 補上 external submit bridge。

建議新增或擴充：

```text
apps/esingpay-cradle/src/fund/facade/withdrawal-intent.facade.ts
```

概念：

```ts
@Injectable()
export class WithdrawalIntentFacade {
  constructor(
    @Inject(ExternalSubmitWithdrawalIntentUseCase)
    private readonly externalSubmitWithdrawalIntentUseCase: ExternalSubmitWithdrawalIntentUseCase,
  ) {}

  async submitWithdrawalIntent(
    command: ExternalSubmitWithdrawalIntentCommand,
  ): Promise<Either.Either<WithdrawalIntent, ExternalSubmitWithdrawalIntentError>> {
    return this.externalSubmitWithdrawalIntentUseCase.execute(command);
  }
}
```

Fund facade 不建立 portal actor，也不接收 `merchantUserId`。它只作為 cradle 內 external client 到 fund use case 的受控 bridge。

## External Withdrawal Intent Client

Cradle external client 延續 Stage 1 / Stage 2 pattern，呼叫 fund facade，不直接 DI fund use case。

建議新增或擴充：

```text
apps/esingpay-cradle/src/external/client/withdrawal-intent.client.ts
```

概念：

```ts
@Injectable()
export class ExternalWithdrawalIntentClient {
  constructor(
    @Inject(WITHDRAWAL_INTENT_FACADE)
    private readonly withdrawalIntentFacade: WithdrawalIntentFacade,
  ) {}

  async submitWithdrawalIntent(
    command: ExternalSubmitWithdrawalIntentCommand,
  ): Promise<Either.Either<WithdrawalIntent, ExternalSubmitWithdrawalIntentError>> {
    return this.withdrawalIntentFacade.submitWithdrawalIntent(command);
  }
}
```

這個 client 回傳 raw `WithdrawalIntent` 的 `Either`，不回傳 fund internal `WithdrawalIntentComposed`。

## External REST Adapter

Cradle external adapter 仍放在 Stage 1 建立的 external REST adapter 路徑：

```text
apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent/
```

Stage 3 在這裡新增 `POST` handling。

責任：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取得 `merchantId`。
- 將 external create DTO 轉成 `ExternalSubmitWithdrawalIntentCommand`。
- 呼叫 `ExternalWithdrawalIntentClient.submitWithdrawalIntent(...)`。
- 將 `Either.left` map 成 external REST result code。
- 將 `Either.right` 的 raw `WithdrawalIntent` 交給 `ExternalWithdrawalIntentComposer`。
- 將 composed result map 成 `ExternalWithdrawalIntentDto`。

概念：

```ts
async createWithdrawalIntent(
  request: RestRpcRequest<RpcApi['createWithdrawalIntent']['Input']>,
): Promise<RpcApi['createWithdrawalIntent']['Output']> {
  if (request.identity.type !== RestIdentityType.MerchantAgent) {
    throw createCommonRestException({ code: CommonCode.Forbidden });
  }

  const command = this.mapper.toExternalSubmitWithdrawalIntentCommand({
    merchantId: request.identity.merchantId,
    body: request.input.body,
  });

  if (command === null) {
    throw createCommonRestException({ code: CommonCode.DataInvalid });
  }

  const result = await this.externalWithdrawalIntentClient.submitWithdrawalIntent(command);

  if (Either.isLeft(result)) {
    throw this.mapper.toExternalRestException(result.left);
  }

  const composed = await this.externalWithdrawalIntentComposer.composeOne(result.right);

  return RestResultFactory.ok({
    data: this.mapper.toExternalWithdrawalIntentDto(composed),
  });
}
```

External adapter 不直接呼叫：

```ts
MerchantSubmitWithdrawalIntentUseCase
WithdrawalIntentSubmitterService
```

## Contract REST

Stage 3 需要在 external withdrawal intent API contract 補上：

- `POST /external/v1/withdrawal-intents`

路徑：

```text
libs/contract-rest/src/lib/external/api/withdrawal-intent.api.ts
```

概念：

```ts
export const externalWithdrawalIntentApi = defineApi({
  namespace: NAMESPACE,
  id: 'external-withdrawal-intent' as const,
  identity: ApiIdentity.MerchantAgent,
  basePath: '/external/v1/withdrawal-intents',
  endpoints: {
    searchWithdrawalIntents(...) { ... },

    createWithdrawalIntent(arg: {
      body: CreateExternalWithdrawalIntentDto;
    }): ApiRequest<ResultDto<CommonCode.Ok, ExternalWithdrawalIntentDto> | ResultDto<CommonCode.DataInvalid | CommonCode.Forbidden>> {
      return {
        method: HttpMethod.Post,
        path: '',
        body: arg.body,
      };
    },
  },
});
```

External create request DTO 依 design 文件定義，第一版支援範圍以 Tron USDT blockchain address 為主。

第一版不做：

- `Idempotency-Key`。
- API key scope。
- API key prefix。

## Module Wiring

`WithdrawalIntentSubmitterService` 要加入 withdrawal intent context module：

```text
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/withdrawal-intent-context.module.ts
```

概念：

```ts
const services = [
  WithdrawalIntentSubmitterService,
  WithdrawalIntentFeeCalculationService,
  WithdrawalIntentEvaluationService,
  // ...
];

const useCases = [
  MerchantSubmitWithdrawalIntentUseCase,
  ExternalSubmitWithdrawalIntentUseCase,
  // ...
];
```

因為 merchant portal use case、external submit use case 與 fund facade bridge 都需要使用 submitter，所以 submitter 應在 context module exports 中可被使用。

## Reuse Rules

Stage 3 應重用：

- 現有 withdrawal intent submit mechanics。
- 現有 wallet ownership / fee / evaluation / allocation / queue dispatch dependencies。
- Merchant portal response 繼續重用 fund internal withdrawal intent composer。
- External response 使用 external feature 的 `ExternalWithdrawalIntentComposer`。
- 現有 submit error type 與 result mapping 可用部分。

Stage 3 不應重用：

- `merchantUserId` 作為 external API caller identity。
- Merchant portal REST DTO 作為 external request / response DTO。
- Merchant portal submit use case 作為 external adapter 的直接 dependency。
- Fund internal `WithdrawalIntentComposed` 作為 external boundary payload。
- Fund submit `contract-rpc` 作為本 stage 的必要 boundary。
- Copy-pasted submit implementation。

## Validation Target

Stage 3 完成時，應能證明 external submit 路徑：

```text
POST /external/v1/withdrawal-intents
  -> gateway external controller
  -> gateway external proxy
  -> cradle external REST adapter
  -> external withdrawal intent client
  -> fund withdrawal intent facade
  -> ExternalSubmitWithdrawalIntentUseCase
  -> WithdrawalIntentSubmitterService
  -> ExternalWithdrawalIntentComposer
  -> ExternalWithdrawalIntentDto response
```

同時，merchant portal submit 仍應走：

```text
MerchantSubmitWithdrawalIntentUseCase
  -> createMerchantUserActor(merchantUserId)
  -> WithdrawalIntentSubmitterService
```

驗證重點：

- external submit 不傳 `merchantUserId`。
- external submit 不建立 `MerchantUser` actor。
- merchant portal submit 仍建立 `MerchantUser` actor。
- `WithdrawalIntentSubmitterService` 沒有 caller 判斷。
- `WithdrawalIntentSubmitterService` 沒有 actor 建立。
- `WithdrawalIntentSubmitterService` 不做 `WithdrawalIntentComposed` composition。
- External adapter 透過 `ExternalWithdrawalIntentComposer` 組成 external response。
- submit mechanics 只有一份。

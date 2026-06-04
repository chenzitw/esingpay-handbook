---
status: draft
updated_at: 2026-06-04
updated_by: Codex
---

# External API — Stage 3 Blueprint

## 目標

Stage 3 完成 external withdrawal intent create：

- `POST /external/v1/withdrawal-intents`

這個 stage 的重點不是 copy 一份 `MerchantSubmitWithdrawalIntentUseCase`，而是把 withdrawal intent submit 的核心流程抽成共用 `WithdrawalIntentSubmitterService`。

共用 service 不決定 caller 是 merchant portal 還是 external API，也不建立 actor。它只接受上層 use case 已經決定好的 `submittedBy`。

## Boundary

Stage 3 需符合目前 boundary：

```text
Gateway -> External = contract-rest + rest-rpc transport bridge
External -> Fund = contract-rpc service-to-service call
Fund internal = use case / service / repository / composer
```

因此 external adapter 不直接 DI fund use case。

完整 create flow 應為：

```text
POST /external/v1/withdrawal-intents
  -> apps/esing-pay-api-gateway/src/rest/external/controller/withdrawal-intent.controller.ts
  -> apps/esing-pay-api-gateway/src/rest/external/proxy/withdrawal-intent.proxy.ts
  -> rest-rpc transport bridge
  -> apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent/withdrawal-intent.controller.ts
  -> apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent/withdrawal-intent.service.ts
  -> external fund rpc client
  -> fund rpc server withdrawal-intent controller
  -> ExternalSubmitWithdrawalIntentUseCase
  -> WithdrawalIntentSubmitterService
  -> WithdrawalIntentComposed
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
  ): Promise<Either.Either<WithdrawalIntentComposed, SubmitWithdrawalIntentError>> {
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
- result compose。
- submit error convergence。

關鍵差異是 entity 建立時不再自己建立 merchant user actor。

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
  ) {}

  async execute(
    command: MerchantSubmitWithdrawalIntentCommand,
  ): Promise<Either.Either<WithdrawalIntentComposed, MerchantSubmitWithdrawalIntentError>> {
    return await this.withdrawalIntentSubmitterService.submit({
      merchantId: command.merchantId,
      senderWalletId: command.senderWalletId,
      destination: command.destination,
      currencyCode: command.currencyCode,
      amount: command.amount,
      submittedBy: createMerchantUserActor(command.merchantUserId),
    });
  }
}
```

這代表 merchant portal actor 邏輯仍在 portal use case，不會散到共用 submitter。

## Fund External Submit Use Case

新增 fund 端給 external RPC server 使用的 use case：

```text
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/use-case/external-submit-withdrawal-intent.use-case.ts
```

它接收 external service-to-service command，不接收 `merchantUserId`：

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
  ): Promise<Either.Either<WithdrawalIntentComposed, ExternalSubmitWithdrawalIntentError>> {
    return await this.withdrawalIntentSubmitterService.submit({
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

## Fund RPC Contract

External service 與 Fund service 溝通要走 `contract-rpc`。

Stage 3 需要新增或擴充 withdrawal intent RPC contract，提供 external submit method。概念上：

```ts
export const withdrawalIntentRpc = defineRpc({
  namespace: 'fund',
  id: 'withdrawal-intent',
  methods: {
    externalSubmitWithdrawalIntent(
      input: ExternalSubmitWithdrawalIntentRpcInputDto,
    ): Promise<ExternalSubmitWithdrawalIntentRpcResultDto> {
      // contract only
    },
  },
});
```

RPC input 應承載 fund submit 必要欄位：

```ts
export type ExternalSubmitWithdrawalIntentRpcInputDto = {
  merchantId: string;
  senderWalletId: string;
  destination: NetworkDestinationDto;
  currencyCode: SupportedCurrencyCode;
  amount: NumericString;
};
```

RPC output 可以回傳 composed withdrawal intent DTO，或回傳足夠讓 external adapter map 成 `ExternalWithdrawalIntentDto` 的資料。

RPC error code 需能表達：

- forbidden。
- data invalid。
- config incomplete。
- destination invalid。
- destination internal。
- limit exceeded。
- balance insufficient。
- allocation failure。

## Fund RPC Server

Fund RPC server 負責接 `contract-rpc` method，轉成 fund use case command。

建議新增或擴充：

```text
apps/esingpay-cradle/src/fund/rpc/server/withdrawal-intent/withdrawal-intent.controller.ts
```

概念：

```ts
@MessagePattern(rpcTopic.externalSubmitWithdrawalIntent)
async externalSubmitWithdrawalIntent(
  input: ExternalSubmitWithdrawalIntentRpcInputDto,
): Promise<ExternalSubmitWithdrawalIntentRpcResultDto> {
  const command = this.mapper.toExternalSubmitCommand(input);

  if (command === null) {
    return { code: CommonCode.DataInvalid, data: null };
  }

  const result = await this.externalSubmitWithdrawalIntentUseCase.execute(command);

  if (Either.isLeft(result)) {
    return this.mapper.toExternalSubmitErrorResult(result.left);
  }

  return this.mapper.toExternalSubmitOkResult(result.right);
}
```

Fund RPC server 不建立 portal actor，也不接收 `merchantUserId`。

## External REST Adapter

Cradle external adapter 仍放在 Stage 1 建立的 external REST adapter 路徑：

```text
apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent/
```

Stage 3 在這裡新增 `POST` handling。

責任：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取得 `merchantId`。
- 將 external create DTO 轉成 fund RPC input。
- 呼叫 external fund RPC client。
- 將 fund RPC result map 成 `ExternalWithdrawalIntentDto`。
- 將 fund RPC error map 成 external REST result code。

概念：

```ts
async createWithdrawalIntent(
  request: RestRpcRequest<RpcApi['createWithdrawalIntent']['Input']>,
): Promise<RpcApi['createWithdrawalIntent']['Output']> {
  if (request.identity.type !== RestIdentityType.MerchantAgent) {
    throw createCommonRestException({ code: CommonCode.Forbidden });
  }

  const rpcInput = this.mapper.toExternalSubmitWithdrawalIntentRpcInput({
    merchantId: request.identity.merchantId,
    body: request.input.body,
  });

  if (rpcInput === null) {
    throw createCommonRestException({ code: CommonCode.DataInvalid });
  }

  const rpcResult = await this.fundWithdrawalIntentRpcClient.externalSubmitWithdrawalIntent(rpcInput);

  if (rpcResult.code !== RpcCommonCode.Ok) {
    throw this.mapper.toExternalRestException(rpcResult);
  }

  return RestResultFactory.ok({
    data: this.mapper.toExternalWithdrawalIntentDto(rpcResult.data),
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

因為 merchant portal use case、fund RPC server use case 都需要使用 submitter，所以 submitter 應在 context module exports 中可被使用。

## Reuse Rules

Stage 3 應重用：

- 現有 withdrawal intent submit mechanics。
- 現有 wallet ownership / fee / evaluation / allocation / queue dispatch dependencies。
- 現有 withdrawal intent composer。
- 現有 submit error type 與 result mapping 可用部分。

Stage 3 不應重用：

- `merchantUserId` 作為 external API caller identity。
- Merchant portal REST DTO 作為 external request / response DTO。
- Merchant portal submit use case 作為 external adapter 的直接 dependency。
- Copy-pasted submit implementation。

## Validation Target

Stage 3 完成時，應能證明 external submit 路徑：

```text
POST /external/v1/withdrawal-intents
  -> gateway external controller
  -> gateway external proxy
  -> cradle external REST adapter
  -> fund contract-rpc client
  -> fund rpc server
  -> ExternalSubmitWithdrawalIntentUseCase
  -> WithdrawalIntentSubmitterService
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
- submit mechanics 只有一份。

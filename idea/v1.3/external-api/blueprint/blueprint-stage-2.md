---
status: draft
updated_at: 2026-06-07
updated_by: Codex
---

# External API — Stage 2 Blueprint

## 目標

Stage 2 完成 external read APIs：

- `GET /external/v1/deposits`
- `GET /external/v1/deposits/{id}`
- `GET /external/v1/withdrawal-intents`
- `GET /external/v1/withdrawal-intents/{id}`
- `GET /external/v1/wallets`
- `GET /external/v1/wallets/{id}`

Stage 2 延續 Stage 1 建立的 boundary：

```text
esing-pay-api-gateway src/rest/external
  -> RestRpcClient
  -> esingpay-cradle src/external
  -> external client
  -> fund / wallet facade
  -> fund / wallet query capability
  -> external DTO response
```

External adapter 不直接 import 或 inject fund / wallet internal context。

## Scope

範圍內：

- 補齊 external deposit list / detail。
- 補齊 external withdrawal intent list / detail。
- 補齊 external wallet list / detail。
- 實作 external read DTO mapping。
- 在 cradle external adapter 內做 merchant scoping。
- 建立 facade / client 暫時邊界，避免 external adapter 直接依賴 fund / wallet internal implementation。
- 補足 fund / wallet read capability 需要的 query service method。
- 確保 external response 不直接輸出 portal DTO。

範圍外：

- `POST /external/v1/withdrawal-intents`。
- 真 `contract-rpc` migration。
- API key persistence and validation。
- IP whitelist enforcement。
- Console API key management。
- Docusaurus docs site。
- `Idempotency-Key`。
- API key scope。

## Service Boundary Decision

長期正解是：

```text
external adapter
  -> fund / wallet contract-rpc client
  -> fund / wallet rpc server
  -> fund / wallet query capability
```

但目前 `contract-rpc` serialization 尚未完整支援 domain raw object，因此 Stage 2 暫時採用 facade / client pattern：

```text
external adapter
  -> external client
  -> fund / wallet facade
  -> fund / wallet query service / composer
```

這是 intra-cradle temporary boundary。它利用目前 fund / wallet / external 暫時同在 `esingpay-cradle` instance 內的事實，用 function call 先完成 service-to-service capability。

這個暫時解法的限制：

- facade / client 是為了保留 microservice boundary，不是正式 RPC。
- 未來 `contract-rpc` + serialization 完成後，client/facade 應抽換成 RPC client/server。
- external adapter 的主要流程不應因抽換 RPC 而大改。

禁止做法：

```text
external adapter
  -> fund context module
  -> MerchantSearchDepositsUseCase
```

```text
external adapter
  -> wallet context module
  -> WalletQueryService / WalletComposer
```

External adapter 不應直接 import 或 inject 隔壁 microservice 的 internal context、use case、query service、composer。

## Facade / Client Pattern

Facade 用途與簡化例子另見：

- [`reference-facade-pattern.md`](../references/reference-facade-pattern.md)

對位參考：

```text
本體 facade:
apps/esingpay-cradle/src/wallet/facade/wallet.facade.ts

客體 client:
apps/esingpay-cradle/src/fund/client/wallet.client.ts
```

Stage 2 應使用相同概念。

Capability owner 定義 facade：

```text
fund/facade/deposit.facade.ts
fund/facade/withdrawal-intent.facade.ts
wallet/facade/wallet.facade.ts
```

Caller 定義 client：

```text
external/client/deposit.client.ts
external/client/withdrawal-intent.client.ts
external/client/wallet.client.ts
```

External adapter 只依賴 external client。External client 暫時透過 `ModuleRef.get(..., { strict: false })` 取得對應 facade。

Facade 回傳 domain raw object 或 composed domain result，不回傳 REST DTO。

## Capability Shape

Stage 2 read capability 應以 domain result 傳遞：

```ts
export type ExternalServiceSearchResult<TItem> = {
  items: TItem[];
  paging: {
    page: number;
    size: number;
    count: number;
  };
};
```

Facade / client method 可先使用 capability-specific type，不必強行共用泛型名稱；但概念上要包含：

- merchant scope input。
- list paging input。
- list paging output。
- detail by id input。
- not found / invalid paging 的可表達結果。

External mapper 才負責把 domain/composed result 轉成 external contract DTO。

## Fund Deposit Capability

### APIs

- `GET /external/v1/deposits`
- `GET /external/v1/deposits/{id}`

### Query Service

Deposit 需要補 query service，避免 external 直接使用 merchant portal use case。

建議新增：

```text
apps/esingpay-cradle/src/fund/feature/deposit/service/deposit-query.service.ts
```

參考：

```text
apps/esingpay-cradle/src/wallet/feature/wallet/service/wallet-asset-query.service.ts
```

Deposit query service 應提供 merchant-scoped read method：

```ts
export type SearchDepositsByMerchantIdQuery = {
  merchantId: Uuid;
  recipientWalletIdIn?: bigint[];
  statusIn?: DepositStatus[];
  paging?: PartialOffsetPaging;
};

@Injectable()
export class DepositQueryService {
  async searchByMerchantId(
    query: SearchDepositsByMerchantIdQuery,
  ): Promise<Either.Either<SearchDepositsResult, DepositQueryError>> {
    // DepositRepository.search
    // DepositComposer.composeMany
  }

  async getOneByMerchantId(input: {
    merchantId: Uuid;
    depositId: bigint;
  }): Promise<DepositComposed | null> {
    // DepositRepository.getOneById
    // merchant ownership check
    // DepositComposer.composeOne
  }
}
```

`MerchantSearchDepositsUseCase` 與 `MerchantGetDepositUseCase` 可以後續改為包這個 query service，避免 read logic 分叉。

### Facade

新增：

```text
apps/esingpay-cradle/src/fund/facade/deposit.facade.ts
```

Facade 負責呼叫 `DepositQueryService`：

```ts
@Injectable()
export class DepositFacade {
  constructor(
    @Inject(DepositQueryService)
    private readonly depositQueryService: DepositQueryService,
  ) {}

  async searchDeposits(input: SearchDepositsByMerchantIdQuery) {
    return this.depositQueryService.searchByMerchantId(input);
  }

  async getDeposit(input: { merchantId: Uuid; depositId: bigint }) {
    return this.depositQueryService.getOneByMerchantId(input);
  }
}
```

### External Client

新增：

```text
apps/esingpay-cradle/src/external/client/deposit.client.ts
```

External client 暫時 call facade：

```ts
@Injectable()
export class ExternalDepositClient {
  constructor(
    @Inject(ModuleRef)
    private readonly moduleRef: ModuleRef,
  ) {}

  async searchDeposits(input: SearchDepositsByMerchantIdQuery) {
    return this.getDepositFacade().searchDeposits(input);
  }

  private getDepositFacade(): DepositFacade {
    return this.moduleRef.get(DepositFacade, { strict: false });
  }
}
```

未來改真 RPC 時，替換 `ExternalDepositClient` 的 implementation 即可。

### External Adapter Responsibility

Cradle external deposit adapter 負責：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取得 `merchantId`。
- 將 external query params 轉成 `ExternalDepositClient` input。
- 呼叫 `ExternalDepositClient`。
- 將 `DepositComposed` map 成 `ExternalDepositDto`。
- 將 client result / error map 成 external REST result code。

External deposit adapter 不直接 inject：

- `DepositQueryService`
- `DepositComposer`
- `MerchantSearchDepositsUseCase`
- `MerchantGetDepositUseCase`
- `DepositRepository`

## Fund Withdrawal Intent Capability

### APIs

- `GET /external/v1/withdrawal-intents`
- `GET /external/v1/withdrawal-intents/{id}`

### Query Service

Withdrawal intent 需要提供 merchant-scoped query method。

若既有 `WithdrawalIntentQueryService` 已可支援 external read，則補足 method；若不足，依 wallet asset query service pattern 補齊：

```text
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/service/withdrawal-intent-query.service.ts
```

建議 method：

```ts
export type SearchWithdrawalIntentsByMerchantIdQuery = {
  merchantId: Uuid;
  senderWalletIdIn?: bigint[];
  statusIn?: WithdrawalIntentStatus[];
  paging?: PartialOffsetPaging;
};

@Injectable()
export class WithdrawalIntentQueryService {
  async searchByMerchantId(
    query: SearchWithdrawalIntentsByMerchantIdQuery,
  ): Promise<Either.Either<SearchWithdrawalIntentsResult, WithdrawalIntentQueryError>> {
    // WithdrawalIntentRepository.search
    // WithdrawalIntentComposer.composeMany
  }

  async getOneByMerchantId(input: {
    merchantId: Uuid;
    withdrawalIntentId: bigint;
  }): Promise<WithdrawalIntentComposed | null> {
    // WithdrawalIntentRepository.getOneById
    // merchant ownership check
    // WithdrawalIntentComposer.composeOne
  }
}
```

`MerchantSearchWithdrawalIntentsUseCase` 與 `MerchantGetWithdrawalIntentUseCase` 可以後續改為包這個 query service，避免 read logic 分叉。

### Facade

新增：

```text
apps/esingpay-cradle/src/fund/facade/withdrawal-intent.facade.ts
```

Facade 負責呼叫 `WithdrawalIntentQueryService`：

```ts
@Injectable()
export class WithdrawalIntentFacade {
  constructor(
    @Inject(WithdrawalIntentQueryService)
    private readonly withdrawalIntentQueryService: WithdrawalIntentQueryService,
  ) {}

  async searchWithdrawalIntents(input: SearchWithdrawalIntentsByMerchantIdQuery) {
    return this.withdrawalIntentQueryService.searchByMerchantId(input);
  }

  async getWithdrawalIntent(input: {
    merchantId: Uuid;
    withdrawalIntentId: bigint;
  }) {
    return this.withdrawalIntentQueryService.getOneByMerchantId(input);
  }
}
```

### External Client

新增：

```text
apps/esingpay-cradle/src/external/client/withdrawal-intent.client.ts
```

External client 暫時 call facade：

```ts
@Injectable()
export class ExternalWithdrawalIntentClient {
  constructor(
    @Inject(ModuleRef)
    private readonly moduleRef: ModuleRef,
  ) {}

  async searchWithdrawalIntents(input: SearchWithdrawalIntentsByMerchantIdQuery) {
    return this.getWithdrawalIntentFacade().searchWithdrawalIntents(input);
  }

  private getWithdrawalIntentFacade(): WithdrawalIntentFacade {
    return this.moduleRef.get(WithdrawalIntentFacade, { strict: false });
  }
}
```

未來改真 RPC 時，替換 `ExternalWithdrawalIntentClient` 的 implementation 即可。

### External Adapter Responsibility

Cradle external withdrawal intent adapter 負責：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取得 `merchantId`。
- 將 external query params 轉成 `ExternalWithdrawalIntentClient` input。
- 呼叫 `ExternalWithdrawalIntentClient`。
- 將 `WithdrawalIntentComposed` map 成 `ExternalWithdrawalIntentDto`。
- 將 client result / error map 成 external REST result code。

External withdrawal intent adapter 不直接 inject：

- `WithdrawalIntentQueryService`
- `WithdrawalIntentComposer`
- `MerchantSearchWithdrawalIntentsUseCase`
- `MerchantGetWithdrawalIntentUseCase`
- `WithdrawalIntentRepository`

## Wallet Capability

### APIs

- `GET /external/v1/wallets`
- `GET /external/v1/wallets/{id}`

### Query Service

Wallet read 目前已有 `WalletQueryService` 與 composer capability。Stage 2 不新增只包一層的 wallet use case。

但 external adapter 不直接 inject `WalletQueryService` 或 `WalletComposer`，而是透過 external wallet client 呼叫 wallet facade。

### Facade

既有：

```text
apps/esingpay-cradle/src/wallet/facade/wallet.facade.ts
```

若現有 facade method 不足，Stage 2 應擴充 wallet facade，例如：

```ts
@Injectable()
export class WalletFacade {
  async searchWalletsByMerchantId(input: SearchWalletsByMerchantIdQuery) {
    // WalletQueryService search/list
    // WalletComposer composeMany
  }

  async getWalletByMerchantId(input: {
    merchantId: Uuid;
    walletId: bigint;
  }) {
    // WalletQueryService.getOneById
    // merchant ownership check
    // WalletComposer.composeOne
  }
}
```

### External Client

新增：

```text
apps/esingpay-cradle/src/external/client/wallet.client.ts
```

External client 暫時 call wallet facade：

```ts
@Injectable()
export class ExternalWalletClient {
  constructor(
    @Inject(ModuleRef)
    private readonly moduleRef: ModuleRef,
  ) {}

  async searchWallets(input: SearchWalletsByMerchantIdQuery) {
    return this.getWalletFacade().searchWalletsByMerchantId(input);
  }

  private getWalletFacade(): WalletFacade {
    return this.moduleRef.get(WalletFacade, { strict: false });
  }
}
```

未來改真 RPC 時，替換 `ExternalWalletClient` 的 implementation 即可。

### External Adapter Responsibility

Cradle external wallet adapter 負責：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取得 `merchantId`。
- 將 external query params 轉成 `ExternalWalletClient` input。
- 呼叫 `ExternalWalletClient`。
- Detail 必須驗證 wallet 的 `merchantId` 等於 identity merchantId；此檢查應在 wallet facade/query capability 或 client result contract 中明確保證。
- 將 composed wallet map 成 `ExternalWalletDto`。
- 將 not found / invalid params map 成 external REST result code。

External wallet adapter 不直接 inject：

- `WalletQueryService`
- `WalletComposer`
- `WalletRepository`

## External DTO Mapping

External response 不應直接重用 portal DTO。

Deposit mapper 應隱藏：

- actor details。
- full status history。
- internal correlation details。
- network endpoint lifecycle。

Withdrawal intent mapper 應隱藏：

- actor details。
- full status history。
- wallet allocation id。
- fee payer wallet strategy。
- internal network endpoint lifecycle。

Wallet mapper 應隱藏：

- top-level `merchant`。
- `networkEndpoint.status`。
- HD wallet path / account / index。
- held / heldLocked balance。

## Gateway

Stage 2 gateway external module 應延伸 Stage 1 結構，新增或補齊：

- external deposit controller / proxy / query DTO。
- external withdrawal intent detail method。
- external wallet controller / proxy / query DTO。

Gateway external controllers 應依 resource 拆檔，例如：

- `controller/deposit.controller.ts`
- `controller/withdrawal-intent.controller.ts`
- `controller/wallet.controller.ts`

Gateway 不做 business mapping，也不直接呼叫 fund / wallet portal controllers。

## Cradle External

Stage 2 cradle external module 應延伸 Stage 1 結構，新增或補齊：

- external deposit REST adapter。
- external withdrawal intent REST adapter detail method。
- external wallet REST adapter。
- external read mappers。
- external deposit / withdrawal intent / wallet clients。

Cradle external adapters 不呼叫 fund / wallet REST adapters。

正確方向：

```text
external adapter
  -> external client
  -> fund / wallet facade
  -> fund / wallet query service / composer
  -> external mapper
```

錯誤方向：

```text
external adapter
  -> fund or wallet REST adapter
```

```text
external adapter
  -> fund / wallet internal context
  -> use case / query service / composer
```

## Future RPC Migration

Stage 2 的 facade / client 是暫時解。

未來 `contract-rpc` serialization 完成後，遷移方向是：

```text
external client
  -> fund / wallet contract-rpc client
  -> fund / wallet rpc server
  -> existing facade or query capability
```

遷移時 external REST adapter 不應重寫主要流程，只替換 client implementation 與對應 RPC contract/server。

## Validation Target

Stage 2 完成時，應能證明：

```text
GET /external/v1/deposits
GET /external/v1/deposits/{id}
GET /external/v1/withdrawal-intents
GET /external/v1/withdrawal-intents/{id}
GET /external/v1/wallets
GET /external/v1/wallets/{id}
```

都能走：

```text
gateway src/rest/external
  -> cradle src/external REST adapter
  -> external client
  -> fund / wallet facade
  -> fund / wallet query capability
  -> external mapper
  -> external DTO response
```

驗證重點：

- external adapter 沒有 import fund / wallet context module。
- external adapter 沒有 inject fund / wallet use case、query service、composer、repository。
- facade 回傳 domain raw object 或 composed domain result。
- external mapper 才輸出 external DTO。
- client / facade boundary 可在未來抽換成真 RPC。

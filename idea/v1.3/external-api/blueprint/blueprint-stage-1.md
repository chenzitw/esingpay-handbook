---
status: draft
updated_at: 2026-06-07
updated_by: Codex
---

# External API — Stage 1 Blueprint

## 目標

Stage 1 建立 External API boundary 的基本骨架。

第一條驗證路徑使用：

- `GET /external/v1/withdrawal-intents`

這條路徑用來證明 external API boundary 可接到真實 withdrawal intent list data：

```text
esing-pay-api-gateway src/rest/external
  -> contract-rest external API
  -> rest-rpc transport bridge
  -> esingpay-cradle src/external/feature/fund/rest
  -> ExternalWithdrawalIntentClient
  -> WithdrawalIntentFacade
  -> WithdrawalIntentQueryService.searchByMerchantId
  -> WithdrawalIntentRepository.search
  -> WithdrawalIntentComposer.composeMany
  -> external mapper
  -> ExternalWithdrawalIntentDto list response
```

Stage 1 不完成所有 External API resource，但 `GET /external/v1/withdrawal-intents` 必須接到真實資料，不以 mock response 作為完成目標。

## 邊界原則

External API 有兩種不同的通訊邊界，不能混用資料夾語意。

### Gateway To External

API Gateway 到 cradle external 的這段是：

```text
HTTP request
  -> contract-rest
  -> RestRpcClient
  -> cradle external REST adapter
```

這段雖然物理上使用 `@MessagePattern` / RPC transport 承載 request，但它的 contract 語意是 REST API proxy。

因此 cradle 端實作應放在：

```text
apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent
```

不得放在：

```text
apps/esingpay-cradle/src/external/rpc/server
```

`external/rpc/server` 會誤導為 `contract-rpc` server implementation，不符合這段 gateway-to-external 的語意。

### External To Service

External module 往 Fund / Wallet / Network 等 service 取得能力時，才是 service-to-service method call。

長期正解應使用：

```text
contract-rpc
```

例如：

```text
cradle external REST adapter
  -> fund contract-rpc client
  -> fund/rpc/server
  -> fund query service / use case
```

未來即使 `fund`、`wallet`、`network` 從 `esingpay-cradle` 拆成獨立微服務，External module 也可維持相同 `contract-rpc` client boundary。

但在 `contract-rpc` serialization 尚未完整支援 domain raw object 前，cradle 內跨 microservice 的暫時解法是 facade / client pattern：

```text
cradle external REST adapter
  -> external client
  -> fund / wallet facade
  -> fund / wallet query service / composer
```

這個暫時解法只允許透過 client / facade 控制邊界，不允許 external adapter 直接 import 或 inject fund / wallet internal context、use case、query service、composer。

## 程式碼放置

External API Gateway 程式碼放在：

```text
apps/esing-pay-api-gateway/src/rest/external
```

External cradle adapter 程式碼放在：

```text
apps/esingpay-cradle/src/external
```

Stage 1 不得把 external controllers 或 adapters 放在：

```text
apps/esing-pay-api-gateway/src/rest/fund
apps/esing-pay-api-gateway/src/rest/wallet
apps/esingpay-cradle/src/fund/feature
apps/esingpay-cradle/src/wallet/feature
apps/esingpay-cradle/src/external/rpc/server
```

Fund 和 wallet 仍是 business capability owners。External 是 integration adapter boundary。

## Stage 1 Scope

範圍內：

- 定義 external withdrawal intent list REST contract。
- 新增 gateway external REST module。
- 新增 gateway external withdrawal intent controller，並以獨立 controller file 表示。
- 新增 gateway external withdrawal intent proxy。
- 新增 cradle external module。
- 新增 cradle external fund feature module。
- 新增 cradle external withdrawal intent REST adapter。
- 建立 boundary 使用的 `merchant-agent` identity。
- 將 `GET /external/v1/withdrawal-intents` 從 gateway 串到 cradle external REST adapter。
- 以 facade / client pattern 接到真實 withdrawal intent list data。
- 補足 `WithdrawalIntentQueryService.searchByMerchantId`。
- 新增 `WithdrawalIntentFacade`。
- 新增 `ExternalWithdrawalIntentClient`。
- 將 composed withdrawal intent list map 成 `ExternalWithdrawalIntentDto` list response。

Stage 1 需接真實 withdrawal intent list data。External 不直接 DI fund internal use case。長期應透過 fund `contract-rpc` capability 取得資料；但目前 fund withdrawal intent RPC 尚無 merchant-scoped search capability，且 RPC serialization 尚未完成，因此 Stage 1 暫時透過 external client -> fund facade 取得資料。

範圍外：

- 完整 API key persistence 與 validation。
- 完整 IP whitelist enforcement。
- Console API key management。
- Docs website。
- `Idempotency-Key`。
- API key scope。
- API key prefix。
- 其他 external read resources。
- Withdrawal intent create。

## Contract REST

Stage 1 應在 external namespace 下建立 external REST contract，不放在 fund 或 wallet portal API definitions 底下。

### 新增 `libs/contract-rest/src/lib/external/namespace.ts`

定義 external REST contract namespace。

### 新增 `libs/contract-rest/src/lib/external/api/withdrawal-intent.api.ts`

定義 public external withdrawal intent API：

```text
GET /external/v1/withdrawal-intents
```

API identity 應使用 `merchant-agent`，不是 merchant user。

### 新增 `libs/contract-rest/src/lib/external/dto/withdrawal-intent.dto.ts`

對應 `idea/v1.3/external-api/design/design-withdrawal-intent-dto.md`。

### 新增 `libs/contract-rest/src/lib/external/dto/search-withdrawal-intent-params.dto.ts`

定義 `GET /external/v1/withdrawal-intents` 的 query params。

Stage 1 可以先保持 params 最小化。除非 plan 發現既有 pattern 有必要最低欄位，否則 pagination 足以證明 boundary。

### 新增 `libs/contract-rest/src/lib/external/api/index.ts`

匯出 external API definitions。

### 新增 `libs/contract-rest/src/lib/external/dto/index.ts`

匯出 external DTOs。

### 新增 `libs/contract-rest/src/lib/external/index.ts`

匯出 external namespace、APIs、DTOs。

## Contract RPC

Gateway 到 External 不新增 `contract-rpc`。

只有 External 需要向 Fund / Wallet / Network 取得 service capability 時，才新增或重用 `contract-rpc`。

Stage 1 需要把 withdrawal intent list 接到真實資料。長期應先確認 fund 是否已有足夠 RPC capability。

目前既有 fund RPC：

```text
libs/contract-rpc/src/lib/fund/rpc/withdrawal-intent.rpc.ts
```

目前既有 method 只有 `getById` / `listByIds`，但 External list 需要 merchant-scoped search。長期應在 fund RPC contract 補上對應 read capability，再由：

```text
apps/esingpay-cradle/src/fund/rpc/server/withdrawal-intent
```

實作。

但在 RPC serialization 尚未完成前，Stage 1 不補真 RPC。Stage 1 暫時解法是：

```text
external REST adapter
  -> external withdrawal-intent client
  -> fund withdrawal-intent facade
  -> WithdrawalIntentQueryService.searchByMerchantId
  -> WithdrawalIntentRepository.search
  -> WithdrawalIntentComposer.composeMany
```

External REST adapter 只能呼叫 external client，不應直接引用 fund internal use case、query service、composer 或 context module 作為跨 service boundary。

## Facade / Query Service 實作要求

Stage 1 以 `GET /external/v1/withdrawal-intents` 接到真實資料為目標，因此需要先補足 fund read capability 的 facade / query service 積木。

對位參考：

```text
apps/esingpay-cradle/src/wallet/feature/wallet/service/wallet-query.service.ts
```

參考重點：

- query service 接收 paging input。
- query service 轉成 repository search query。
- repository 回傳 `items` 與 `count`。
- query service 回傳 list items 與 paging output。
- facade 只包 capability owner 的 query service method，不直接做 REST DTO mapping。

### Withdrawal Intent Query Service

目前 `WithdrawalIntentQueryService` 只有 `getById` / `listByIds`，Stage 1 需要新增 merchant-scoped search method：

```text
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/service/withdrawal-intent-query.service.ts
```

建議 shape：

```ts
export type SearchWithdrawalIntentsByMerchantIdQuery = {
  merchantId: Uuid;
  senderWalletIdIn?: bigint[];
  statusIn?: WithdrawalIntentStatus[];
  paging?: PartialOffsetPaging;
};

export type SearchWithdrawalIntentsByMerchantIdResult = {
  items: WithdrawalIntentComposed[];
  paging: OffsetPagingResult;
};

@Injectable()
export class WithdrawalIntentQueryService {
  async searchByMerchantId(
    query: SearchWithdrawalIntentsByMerchantIdQuery,
  ): Promise<Either.Either<SearchWithdrawalIntentsByMerchantIdResult, BasicError<QueryErrorType.PagingInvalid>>> {
    // resolve paging
    // WithdrawalIntentRepository.search
    // WithdrawalIntentComposer.composeMany
  }
}
```

`searchByMerchantId` 可參考 `MerchantSearchWithdrawalIntentsUseCase` 既有查詢條件，但不應讓 external adapter 直接呼叫該 use case。

### Withdrawal Intent Facade

新增：

```text
apps/esingpay-cradle/src/fund/facade/withdrawal-intent.facade.ts
```

Facade method：

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
}
```

Stage 1 external adapter 透過 `ExternalWithdrawalIntentClient` 呼叫 `WithdrawalIntentFacade.searchWithdrawalIntents`。

## Identity

Stage 1 應建立 external API calls 使用 `merchant-agent` identity，而不是 merchant user identity。

`merchant-agent` 表示商戶系統透過 API key 代理商戶發出 request。

Stage 1 可以先 mock merchant identity values。後續 stage 會把 mock source 替換成 API key validation，並附上 whitelist、audit、debug flows 需要的 API key reference。

### 修改 REST contract identity support

Contract REST API identity enum 需要新增 `merchant-agent` value，讓 external API definitions 不會偽裝成 merchant portal APIs。

### 修改 REST RPC identity support

Service-kit REST RPC identity type 需要新增 `merchant-agent` variant。

Identity 至少要承載 merchant identity information；API key validation 落地後，也應承載 API key identity information。

如果 type 已要求 API key reference，Stage 1 可以先使用 mock API key reference。

## API Gateway Files

### 新增 `apps/esing-pay-api-gateway/src/rest/external/external.module.ts`

註冊 gateway external REST controllers 與 proxies。

匯入既有 REST RPC client module。

### 新增 `apps/esing-pay-api-gateway/src/rest/external/controller/withdrawal-intent.controller.ts`

負責 external withdrawal intent HTTP endpoint：

```text
GET /external/v1/withdrawal-intents
```

職責：

- 綁定 external withdrawal intent API base path。
- 使用 external query DTO parse query params。
- 建立 `merchant-agent` identity。
- 呼叫 external withdrawal intent proxy。
- 將 API key guard 與 IP whitelist guard 保留為明確的 future insertion points。

這個 controller 必須是獨立檔案。不要併進 generic external controller，也不要放在 `src/rest/fund`。

### 新增 `apps/esing-pay-api-gateway/src/rest/external/proxy/withdrawal-intent.proxy.ts`

使用 external withdrawal intent API contract 建立 RestRpc topic map。

職責：

- 接收 gateway controller request。
- 將 typed RestRpc request 轉送到 cradle external REST adapter。
- 回傳 cradle response，不做 business mapping。

### 新增 `apps/esing-pay-api-gateway/src/rest/external/dto/ExternalSearchWithdrawalIntentParamsQuery.ts`

定義 query params 的 Nest validation DTO。

它應與 external contract search params 對齊。

### 新增 `apps/esing-pay-api-gateway/src/rest/external/identity/create-merchant-agent-identity.ts`

建立 RestRpc request 使用的 `merchant-agent` identity。

Stage 1 可以先 mock merchant identity 與 API key identity。後續 stage 會改成從 API key guard output 建立 identity。

### 修改 `apps/esing-pay-api-gateway/src/rest/rest.module.ts`

匯入 new external REST module。

這會讓 gateway app 能接到 `/external/v1/withdrawal-intents`。

## Cradle Files

### 新增 `apps/esingpay-cradle/src/external/app.module.ts`

Cradle external boundary 的 root module。

職責：

- 匯入 external feature module。
- 讓 external adapter wiring 與 fund / wallet root adapters 保持獨立。

### 新增 `apps/esingpay-cradle/src/external/feature/feature.module.ts`

聚合 external feature modules。

Stage 1 匯入 external fund feature module。

### 新增 `apps/esingpay-cradle/src/external/feature/fund/fund.module.ts`

聚合 external fund adapters。

Stage 1 匯入 external fund REST module。

### 新增 `apps/esingpay-cradle/src/external/feature/fund/rest/rest.module.ts`

聚合 external fund REST adapters。

Stage 1 匯入 external withdrawal intent REST module。

### 新增 `apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent/withdrawal-intent.module.ts`

註冊 external withdrawal intent REST controller、service、mapper。

這個 module 應匯入 external client module。client 目前可暫時透過 facade 呼叫 fund capability；未來再抽換為 fund `contract-rpc` client。

這個 module 不應匯入 fund internal feature context module。

### 新增 `apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent/withdrawal-intent.controller.ts`

接收 external withdrawal intent list 的 RestRpc message。

職責：

- 綁定 external withdrawal intent API RestRpc topic。
- 接收 typed RestRpc request。
- 委派給 external withdrawal intent service。

雖然 controller 使用 `@MessagePattern`，它仍是 REST adapter，因為它對應的是 `contract-rest/external`。

### 新增 `apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent/withdrawal-intent.service.ts`

協調 external list request。

職責：

- 驗證 request identity 是 `merchant-agent`。
- 從 identity 取出 merchant scope。
- 將 external search params 轉成 external client search input。
- 呼叫 external client 取得 withdrawal intent data。
- 將 client result / errors 轉成 external REST result codes。
- 回傳 external contract response envelope。

### 新增 `apps/esingpay-cradle/src/external/feature/fund/rest/withdrawal-intent/withdrawal-intent.mapper.ts`

負責 external contract DTOs 與 fund capability result 之間的 mapping。

職責：

- 解碼並 normalize external search params。
- 將 fund capability result map 成 `ExternalWithdrawalIntentDto`。
- 隱藏 internal-only fields，例如 actor details、full status history、wallet allocation id、fee payer wallet strategy、internal network endpoint lifecycle。

### 修改 `apps/esingpay-cradle/src/app.module.ts`

匯入 new cradle external app module。

這會讓 cradle 啟動時註冊 external REST adapter。

## 第一個 Request Walkthrough

Stage 1 request path 應如下：

```text
GET /external/v1/withdrawal-intents
  -> gateway external withdrawal-intent.controller
  -> gateway external withdrawal-intent.proxy
  -> cradle external feature/fund/rest withdrawal-intent.controller
  -> cradle external feature/fund/rest withdrawal-intent.service
  -> ExternalWithdrawalIntentClient
  -> WithdrawalIntentFacade
  -> WithdrawalIntentQueryService.searchByMerchantId
  -> WithdrawalIntentRepository.search
  -> WithdrawalIntentComposer.composeMany
  -> cradle external withdrawal-intent.mapper
  -> ExternalWithdrawalIntentDto list response
```

Stage 1 不以 mock response 作為完成目標。`cradle external feature/fund/rest withdrawal-intent.service` 必須透過 `ExternalWithdrawalIntentClient` 接到真實 withdrawal intent list data。

未來 `contract-rpc` serialization 完成後，上述 client / facade 位置可抽換成：

```text
external withdrawal-intent client
  -> fund contract-rpc client
  -> fund/rpc/server withdrawal-intent controller
```

## 重用策略

Stage 1 應重用：

- Existing RestRpcClient 與 RestRpc topic mapping infrastructure。
- External `contract-rest` API definition。
- Temporary facade / client boundary，或未來 Fund `contract-rpc` capability。
- Existing paging result pattern。

Stage 1 不應重用：

- Portal fund REST controller。
- Portal fund REST DTO 作為 external response DTO。
- Merchant user identity 作為 API key identity。
- Fund REST module 作為 external route owner。
- Fund internal use case 作為 external-to-fund service boundary。
- Fund / wallet internal context 作為 external adapter dependency。

## 驗證目標

Stage 1 完成時，應能證明：

```text
HTTP GET /external/v1/withdrawal-intents
  -> gateway external controller
  -> gateway external proxy
  -> cradle external REST adapter
  -> ExternalWithdrawalIntentClient
  -> WithdrawalIntentFacade
  -> WithdrawalIntentQueryService.searchByMerchantId
  -> WithdrawalIntentRepository.search
  -> WithdrawalIntentComposer.composeMany
  -> external mapper
  -> external DTO response
```

這會建立後續 stages 加上所有 read APIs、withdrawal create、API key、IP whitelist、Console、docs 時可沿用的結構。

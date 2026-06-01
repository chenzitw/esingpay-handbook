# API Gateway 實作 Plan：External Withdrawal Intent

## 目標

新增對外 API Gateway 入口：

```http
GET /external/v1/withdrawal-intents
```

但檔案放在現有 Fund REST 結構下：

```text
apps/esing-pay-api-gateway/src/rest/fund/
```

## 新增 External Controller

### 新增檔案

```text
apps/esing-pay-api-gateway/src/rest/fund/controller/external-withdrawal-intent.controller.ts
```

第一版先不加：

- `ApiKeyGuard`
- `IpWhitelistGuard`

先 mock `merchantId`。

### 概念

```ts
@Controller(externalWithdrawalIntentApi.basePath)
@UseFilters(RestRpcHttpExceptionFilter)
@UsePipes(new ValidationPipe({ transform: true, whitelist: true }))
export class ExternalWithdrawalIntentController {
  constructor(private readonly proxy: ExternalWithdrawalIntentProxy) {}

  @Get('')
  async searchWithdrawalIntents(
    @Query() params: ExternalSearchWithdrawalIntentParamsQuery,
  ) {
    return await this.proxy.searchWithdrawalIntents({
      identity: createExternalUserIdentity(),
      input: { params },
    });
  }
}
```

## 新增 External Proxy

### 新增檔案

```text
apps/esing-pay-api-gateway/src/rest/fund/proxy/external-withdrawal-intent.proxy.ts
```

使用新的 external contract：

```ts
const rpcTopic = createRestRpcTopicMap({
  api: externalWithdrawalIntentApi,
});
```

### 概念

```ts
@Injectable()
export class ExternalWithdrawalIntentProxy {
  constructor(private readonly rpcClient: RestRpcClient) {}

  async searchWithdrawalIntents(
    request: RestRpcRequest<RpcApi['searchWithdrawalIntents']['Input']>,
  ): Promise<RpcApi['searchWithdrawalIntents']['Output']> {
    return (await this.rpcClient.request(
      rpcTopic.searchWithdrawalIntents,
      request,
    )) as RpcApi['searchWithdrawalIntents']['Output'];
  }
}
```

## 新增 External Query DTO

### 新增檔案

```text
apps/esing-pay-api-gateway/src/rest/fund/dto/ExternalSearchWithdrawalIntentParamsQuery.ts
```

### 作用

接 external API 的 query params。

它可以先對齊 external contract DTO：

```ts
export class ExternalSearchWithdrawalIntentParamsQuery
  implements ExternalSearchWithdrawalIntentParamsDto
{
  page?: number;
  size?: number;
  statusIn?: WithdrawalIntentStatus[];
}
```

如果第一版外部 API 不支援太多 filter，就先只開：

- `page`
- `size`

之後再補：

- `createdAtFrom`
- `createdAtTo`

## 新增 createExternalUserIdentity

### 建議新增檔案

目前 `createMerchantUserIdentity` 是從這裡 import：

```text
../../../core/rest-rpc
```

所以建議放在既有 rest-rpc core 旁邊。


### 第一版 mock

```ts
const MOCK_EXTERNAL_MERCHANT_ID = '...';

export function createExternalUserIdentity() {
  return {
    type: 'external_merchant' as const,
    merchantId: MOCK_EXTERNAL_MERCHANT_ID,
  };
}
```

### 正式版

之後正式版改成：

```ts
export function createExternalUserIdentity(apiKey: ApiKeyRecord) {
  return {
    type: 'external_merchant' as const,
    merchantId: apiKey.merchantId,
    apiKeyId: apiKey.id,
  };
}
```

## 修改 Fund REST Module

需要把新的 controller / proxy 註冊進去。

可能修改檔案：

```text
apps/esing-pay-api-gateway/src/rest/fund/fund-rest.module.ts
```

或其他 Fund REST aggregator module。

加入：

```ts
controllers: [
  MerchWithdrawalIntentController,
  ExternalWithdrawalIntentController,
];

providers: [
  MerchWithdrawalIntentProxy,
  ExternalWithdrawalIntentProxy,
];
```

## API Gateway 新增 / 修改清單

### 新增

- `apps/esing-pay-api-gateway/src/rest/fund/controller/external-withdrawal-intent.controller.ts`
- `apps/esing-pay-api-gateway/src/rest/fund/proxy/external-withdrawal-intent.proxy.ts`
- `apps/esing-pay-api-gateway/src/rest/fund/dto/ExternalSearchWithdrawalIntentParamsQuery.ts`
- `apps/esing-pay-api-gateway/src/core/rest-rpc/create-external-user-identity.ts`

### 修改

- `apps/esing-pay-api-gateway/src/core/rest-rpc/index.ts`
- `apps/esing-pay-api-gateway/src/rest/fund/<fund-rest-module>.ts`

### 會重用既有的

- `RestRpcClient`
- `createRestRpcTopicMap`
- `RestRpcHttpExceptionFilter`
- `ValidationPipe`
- Fund REST module 結構
- 現有 controller / proxy pattern

## 目前不做

- `ApiKeyGuard`
- `IpWhitelistGuard`
- API key DB lookup
- API key permission scope
- Rate limit
- Audit log

但架構上先留好位置，之後只會把：

```ts
createExternalUserIdentity()
```

從 mock 改成根據 API key 建立 identity，不需要改 controller → proxy → topic 的整體流程。

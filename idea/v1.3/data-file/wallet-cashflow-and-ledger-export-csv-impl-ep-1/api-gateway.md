# Wallet Export API Gateway Plan

本文整理 API Gateway 在 wallet cashflow / ledger export CSV EP-1 需要新增的 controller、proxy、DTO 與 module wiring。

## Controller Changes

在以下四個既有 controller 檔案各新增一支 export endpoint：

- `plat-ledger.controller.ts`
- `merch-ledger.controller.ts`
- `plat-cashflow.controller.ts`
- `merch-cashflow.controller.ts`

建議 method name：

- ledger：`XxxExportWalletLedgerItems`
- cashflow：`XxxExportWalletCashflowItems`

path：

```ts
@Get('/items/export/:format')
```

## Proxy Changes

在 API Gateway `rest/wallet/proxy/` 新增四支 export proxy：

- `plat-ledger-export.proxy.ts`
- `merch-ledger-export.proxy.ts`
- `plat-cashflow-export.proxy.ts`
- `merch-cashflow-export.proxy.ts`

這四支 proxy 對應新的 export contract，而不是重用既有 query proxy。

proxy 範例：

```ts
@Injectable()
export class PlatCashflowExportProxy {
  constructor(private readonly rpcClient: RestRpcClient) {}

  async platExportWalletCashflowItems(
    request: RestRpcRequest<RpcApi['platExportWalletCashflowItems']['Input']>,
  ): Promise<RpcApi['platExportWalletCashflowItems']['Output']> {
    return (await this.rpcClient.request(
      rpcTopic.platExportWalletCashflowItems,
      request,
    )) as RpcApi['platExportWalletCashflowItems']['Output'];
  }
}
```

ledger proxy 同理，只需替換 method name 與對應 contract。

## Controller Shape

controller 端沿用既有 wallet controller 風格，在原有 controller 上補 method 即可。

```ts
@Controller(platCashflowExportApi.basePath)
@UseFilters(RestRpcHttpExceptionFilter)
@UseGuards(PlatformAccessGuard, ActiveGuard, PermissionGuard)
@UsePipes(new ValidationPipe({ transform: true, whitelist: true }))
export class PlatCashflowController {
  constructor(private readonly exportProxy: PlatCashflowExportProxy) {}

  @Get('/items/export/:format')
  @Permission(PermissionScope.plat.merchantTransactionLedger.list)
  async platExportWalletCashflowItems(
    @User() user: PlatformAccount,
    @Param('format') format: ExportFileFormat,
    @Query() params: PlatformExportCashflowItemsCsvParamsQuery,
  ) {
    const result = await this.exportProxy.platExportWalletCashflowItems({
      identity: createPlatformUserIdentity(user),
      input: {
        pathParams: { format },
        params,
      },
    });

    return result;
  }
}
```

成功情境下，controller 直接回傳 `ResultDto<CommonCode.Ok, WalletExportFileUrlDto>`。

## Export Query DTO

API Gateway 建議新增四組 query class，專門綁 export API：

- `PlatformExportWalletLedgerItemsCsvParamsQuery`
- `MerchantExportWalletLedgerItemsCsvParamsQuery`
- `PlatformExportCashflowItemsCsvParamsQuery`
- `MerchantExportCashflowItemsCsvParamsQuery`

這四組 class-validator 規則可直接複用既有 search query DTO 的 filter 欄位，只移除：

- `page`
- `size`
- `limit`

另外需預留時間區間欄位命名：

- `{fieldName}Gte`
- `{fieldName}Lt`

gateway query class、contract DTO 與 controller `@Query()` key 必須完全一致，避免 transport 層轉名。

## Gateway URL Handling

API Gateway controller 在拿到 `WalletExportFileUrlDto` 後應：

1. 驗證 `format` 是否在 `ExportFileFormat` 範圍。
2. 呼叫對應 export proxy。
3. 直接回傳包含 `downloadUrl` 的 result 給前端。

這樣可以把檔案產生、上傳與 blob URL 管理責任收斂在 `data-file-service`，讓 API Gateway 維持 transport adapter 角色。

## Module Wiring

`apps/esing-pay-api-gateway/src/rest/wallet/wallet.module.ts` 需同步補：

- 新增四個 export proxy provider
- 確保既有四個 controller 的 constructor injection 可拿到對應 proxy

若不希望擴大既有 controller 檔案，也可拆成四個獨立 export controller；但以目前 repo 風格，直接掛回既有 `plat-ledger.controller.ts` / `plat-cashflow.controller.ts` / `merch-ledger.controller.ts` / `merch-cashflow.controller.ts` 會較一致。

## Permission Strategy

建議原則：

- platform export 沿用 platform wallet 對應 scope
- merchant export 沿用 merchant wallet 對應 scope

若 permission matrix 尚未有 export 專用 scope，MVP 可先沿用現有 list scope；若已存在 export scope，則優先使用 export scope，而不是 list scope。
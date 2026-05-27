# Wallet Export data-file-service Plan

本文整理 `data-file-service` 在 wallet cashflow / ledger export CSV EP-1 的 feature 結構、模組責任與執行流程。

## Feature Structure

建議把 `apps/data-file-service/src/feature/wallet/` 擴成：

```text
feature/wallet/
├── wallet.module.ts
├── wallet-context.module.ts
├── service/
│   ├── wallet-export-source.service.ts
│   ├── wallet-export-merchant-timezone.service.ts
│   ├── wallet-export-file.service.ts
│   └── wallet-export-csv.service.ts
├── use-case/
│   ├── export-platform-ledger-csv.use-case.ts
│   ├── export-merchant-ledger-csv.use-case.ts
│   ├── export-platform-cashflow-csv.use-case.ts
│   └── export-merchant-cashflow-csv.use-case.ts
├── schema/
│   ├── wallet-export-file.ts
│   └── wallet-export-batch-query.ts
├── rest/
│   ├── plat-ledger-export/
│   │   ├── plat-ledger-export.module.ts
│   │   ├── plat-ledger-export.controller.ts
│   │   ├── plat-ledger-export.service.ts
│   │   └── plat-ledger-export.mapper.ts
│   ├── merch-ledger-export/
│   ├── plat-cashflow-export/
│   └── merch-cashflow-export/
└── mapper/
    ├── wallet-ledger-export-row.mapper.ts
    └── wallet-cashflow-export-row.mapper.ts
```

## Module Responsibility

- `wallet.module.ts`
  匯總四個 inbound export rest modules 與 `wallet-context.module.ts`
- `wallet-context.module.ts`
  集中提供四個 export flow 共用的 provider
- `wallet-export-source.service.ts`
  負責呼叫既有 wallet query APIs，把 source paging 拉完整
- `wallet-export-merchant-timezone.service.ts`
  負責 merchant export flow 查詢 merchant 登記時區，提供時間顯示轉換所需的時區資訊
- `wallet-export-csv.service.ts`
  負責接收 row model 並執行 csv-writer 寫檔，不負責流程編排
- `wallet-export-file.service.ts`
  負責建立暫存路徑、Azure Blob upload、組 URL DTO、刪檔等 I/O 操作，不負責商業流程判斷
- `use-case/*.use-case.ts`
  負責 export 流程編排：查資料、套 mapper、寫 CSV、上傳 Blob、回傳 URL DTO
- `mapper/*.mapper.ts`
  負責把原始 query item 轉成 CSV row view model，承接前端既有顯示轉換規則與時間格式轉換
- `rest/<slice>/<slice>.service.ts`
  負責身份判斷、組 use-case input、呼叫 use-case、回 REST result

## Why `wallet-context.module.ts`

此 feature 同時有四個 inbound adapter，而且 source fetch / csv write / temp file handling 都會重用。依目前 guide 的 service structure 規則，應建立 `wallet-context.module.ts`，而不是把共用 provider 分散在各 transport module。

## rest-rpc Provider 實作方式

`data-file-service` 端作為 export contract provider，實作風格對齊現有 rest-rpc controller / service 模式。

controller 範例：

```ts
@UseFilters(HttpExceptionToRpcFilter)
@Controller()
export class PlatCashflowExportRestController {
  constructor(
    @Inject(PlatCashflowExportRestService)
    private readonly platCashflowExportRestService: PlatCashflowExportRestService,
  ) {}

  @MessagePattern(rpcTopic.platExportWalletCashflowItems)
  async platExportWalletCashflowItems(
    request: RestRpcRequest<RpcApi['platExportWalletCashflowItems']['Input']>,
  ): Promise<RpcApi['platExportWalletCashflowItems']['Output']> {
    return await this.platCashflowExportRestService.platExportWalletCashflowItems(request);
  }
}
```

service 範例：

```ts
@Injectable()
export class PlatCashflowExportRestService {
  constructor(
    @Inject(PlatCashflowExportMapper)
    private readonly mapper: PlatCashflowExportMapper,
    @Inject(ExportPlatformCashflowCsvUseCase)
    private readonly exportPlatformCashflowCsvUseCase: ExportPlatformCashflowCsvUseCase,
  ) {}

  async platExportWalletCashflowItems(
    request: RestRpcRequest<RpcApi['platExportWalletCashflowItems']['Input']>,
  ): Promise<RpcApi['platExportWalletCashflowItems']['Output']> {
    if (request.identity.type !== 'platform_user') {
      throw createWalletRestException({
        code: CommonCode.Forbidden,
        message: 'Platform identity is required for plat cashflow export',
      });
    }

    const query = this.mapper.buildExportQuery({
      params: request.input.params,
    });

    if (query === null) {
      throw createWalletRestException({ code: CommonCode.ParamsInvalid });
    }

    const fileDto = await this.exportPlatformCashflowCsvUseCase.execute({
      query,
      format: request.input.pathParams.format,
    });

    return RestResultFactory.ok({ data: fileDto });
  }
}
```

## Use-Case Orchestration

每一支 export flow 各自有一個 use-case，維持流程編排與底層 I/O service 分離。

```ts
@Injectable()
export class ExportPlatformCashflowCsvUseCase {
  constructor(
    @Inject(WalletExportSourceService)
    private readonly walletExportSourceService: WalletExportSourceService,
    @Inject(WalletExportCsvService)
    private readonly walletExportCsvService: WalletExportCsvService,
    @Inject(WalletExportFileService)
    private readonly walletExportFileService: WalletExportFileService,
  ) {}

  async execute(arg: {
    query: PlatformCashflowExportBatchQuery;
    format: ExportFileFormat;
  }) {
    const { query, format } = arg;
    const items = await this.walletExportSourceService.searchPlatformCashflowItemsInBatches(query);
    const csvFile = await this.walletExportCsvService.writePlatformCashflowCsv({ items });
    return await this.walletExportFileService.uploadAndCreateFileUrlDto({
      localFile: csvFile,
      format,
    });
  }
}
```

## Context Module Wiring

```ts
@Module({
  providers: [
    WalletExportSourceService,
    WalletExportMerchantTimezoneService,
    WalletExportCsvService,
    WalletExportFileService,
    ExportPlatformLedgerCsvUseCase,
    ExportMerchantLedgerCsvUseCase,
    ExportPlatformCashflowCsvUseCase,
    ExportMerchantCashflowCsvUseCase,
  ],
  exports: [
    WalletExportSourceService,
    WalletExportMerchantTimezoneService,
    WalletExportCsvService,
    WalletExportFileService,
    ExportPlatformLedgerCsvUseCase,
    ExportMerchantLedgerCsvUseCase,
    ExportPlatformCashflowCsvUseCase,
    ExportMerchantCashflowCsvUseCase,
  ],
})
export class WalletContextModule {}
```

## Export Flow

platform ledger export flow：

```text
API Gateway HTTP Controller
-> PlatLedgerExportProxy
-> rest-rpc topic
-> data-file-service PlatLedgerExportRestController
-> PlatLedgerExportRestService
-> ExportPlatformLedgerCsvUseCase.execute()
-> WalletExportSourceService.searchPlatformLedgerItemsInBatches()
-> WalletLedgerExportRowMapper
-> WalletExportCsvService.writeLedgerCsv()
-> WalletExportFileService.uploadToBlobAndDeleteLocalFile()
-> return WalletExportFileUrlDto
-> API Gateway 回傳 downloadUrl 給前端
```

其餘三支 export flow 同理。

## Source Fetching Strategy

upstream query API 目前是 offset paging，且 DTO 上限是 `1000`，MVP 採固定批次拉取：

1. 以 export filters 組成第一頁查詢。
2. 強制 internal paging 為 `page = 1`, `size = 1000`。
3. 取回第一頁後讀 `paging.totalPages`。
4. 逐頁迭代到最後一頁。
5. 聚合成完整 item list。
6. 再交給 row mapper 做欄位轉換，之後才進 CSV writer。

注意：

- export API 對外不接受 paging params
- internal batch query 若有時間區間欄位，需原樣攜帶 `{fieldName}Gte` / `{fieldName}Lt`
- 後續若資料量變大，這裡會是切成 stream / job 的主要演進點

## Merchant Timezone Conversion

merchant 系列 export 在時間欄位處理上，不能直接固定輸出資料庫 UTC 值。

建議流程：

1. 先依 merchant 身分或 export item 關聯 merchant，查出 merchant 登記時區。
2. 對所有需要顯示的時間欄位，從資料庫 UTC 值轉成 merchant 登記時區後再輸出。
3. 若 merchant 時區不存在，第一版 fallback 為 `UTC`，並保留記錄點方便後續補強。

這個編排責任應放在 merchant 專用 use-case，而不是 controller 或純 I/O service。

## Temp File Strategy

`data-file-service` 內部先把 CSV 寫到 service-local temp 位置，例如：

```text
./uploads/data-file-service/wallet/
```

檔名格式固定為：

```text
wallet-{YYYYMMDDHHmmss}.csv
```

例如：

- `wallet-20260525153000.csv`
- `wallet-20260526104530.csv`

命名原則：

- 檔名只附帶建立時間
- 不附帶 query 條件、filter 值、merchantId、walletId 或其他查詢語意資訊

CSV 完成後立刻上傳 Azure Blob，取得 URL 後刪除本地暫存檔，避免長期堆積 service local file。
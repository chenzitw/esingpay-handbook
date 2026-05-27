# Wallet Export Contract And Public API

本文整理 wallet cashflow / ledger export CSV 在 contract layer 與 public API 的規劃，供 API Gateway 與 `data-file-service` 對接時使用。

## 設計原則

- export API 與既有 query API 共用 filter semantics，但不共用 response contract。
- export response 不回 inline file content，而是回傳可下載 URL。
- public API path 維持既有 wallet resource path，僅在 endpoint 上新增 `/items/export/:format`。
- 本階段只支援同步 CSV export，但 contract shape 必須保留未來演進成 job-based export 的空間。

## User-Facing HTTP Endpoints

| Scope | Resource | Method | Path |
| --- | --- | --- | --- |
| Platform | Ledger | `GET` | `/wallet/plat/ledger/items/export/:format` |
| Platform | Cashflow | `GET` | `/wallet/plat/cashflow/items/export/:format` |
| Merchant | Ledger | `GET` | `/wallet/merch/ledger/items/export/:format` |
| Merchant | Cashflow | `GET` | `/wallet/merch/cashflow/items/export/:format` |

目前 `format` 由 `ExportFileFormat` enum 管理，第一版只支援 `Csv = 'csv'`。

## Export Request Params

export API 應重用查詢 API 的 filter semantics，但不暴露 paging semantics。

原因：

- export 的預設語意是匯出符合條件的全部資料。
- 若保留 `page` / `size` / `limit`，容易讓「匯出目前頁」與「匯出全部」的語意混淆。
- `data-file-service` 內部仍可用固定批次分頁抓 source data，但那是 implementation detail，不應進入 public contract。

建議新增四組 export params DTO：

- `PlatformExportWalletLedgerItemsCsvParamsDto`
- `MerchantExportWalletLedgerItemsCsvParamsDto`
- `PlatformExportCashflowItemsCsvParamsDto`
- `MerchantExportCashflowItemsCsvParamsDto`

欄位來源：

- ledger export：沿用原本 ledger query 的 `merchantIdIn` / `walletIdIn` / `currencyCodeIn` / `kindIn`
- cashflow export：沿用原本 cashflow query 的 `merchantIdIn` / `directionIn` / `currencyCodeIn`
- merchant scope 不接受 `merchantIdIn`

時間區間 filter 命名應統一採 `{fieldName}Gte` / `{fieldName}Lt`，例如：

- `effectiveAtGte` / `effectiveAtLt`
- `createdAtGte` / `createdAtLt`

若後續既有 query API 補上時間篩選，export DTO 應沿用相同 key，避免前後端 filter key drift。

## Export Response

MVP 採單一 URL response DTO：

```ts
type WalletExportFileUrlDto = {
  fileName: string;
  contentType: 'text/csv; charset=utf-8';
  downloadUrl: string;
  expiresAt: string | null;
};
```

流程語意：

1. `data-file-service` 產出本地暫存 CSV。
2. `data-file-service` 上傳 Azure Blob 並取得 `downloadUrl`。
3. `data-file-service` 刪除本地暫存檔。
4. `data-file-service` 透過 rest-rpc 回 `WalletExportFileUrlDto` 給 API Gateway。
5. API Gateway 直接回傳 URL result 給前端。

未來若要演進成 job mode，可擴充為：

```ts
type WalletExportFileResultDto =
  | {
      mode: 'url';
      fileName: string;
      contentType: string;
      downloadUrl: string;
      expiresAt: string | null;
    }
  | {
      mode: 'job';
      jobId: string;
      status: 'queued';
    };
```

本波先不加入 `mode`，以最小 `WalletExportFileUrlDto` 為主。

## Contract Namespace Plan

本波新增的 `API Gateway <-> data-file-service` contract，top-level folder 直接命名為 `data-file/`：

```text
libs/contract-rest/src/lib/data-file/
```

這樣能讓 provider 邊界與實際維運責任一致，降低 gateway 與 `data-file-service` 對接時的語意落差。

## New REST API Contracts

建議在 `libs/contract-rest/src/lib/data-file/api/` 新增：

- `plat-ledger-export.api.ts`
- `merch-ledger-export.api.ts`
- `plat-cashflow-export.api.ts`
- `merch-cashflow-export.api.ts`

命名原則：

- namespace 使用 `data-file`
- `id` 使用獨立 resource id，例如 `plat-ledger-export`
- `basePath` 維持原 resource path，例如 `/wallet/plat/ledger`
- endpoint path 固定用 `/items/export/:format`

範例：

```ts
export const platLedgerExportApi = defineApi({
  namespace: NAMESPACE,
  id: 'plat-ledger-export' as const,
  identity: ApiIdentity.PlatformUser,
  basePath: '/wallet/plat/ledger',
  endpoints: {
    platExportWalletLedgerItems(arg: {
      pathParams: { format: ExportFileFormat };
      params: PlatformExportWalletLedgerItemsCsvParamsDto;
    }): ApiRequest<
      | ResultDto<CommonCode.Ok, WalletExportFileUrlDto>
      | ResultDto<CommonCode.Forbidden | CommonCode.ParamsInvalid>
    > {
      return {
        method: HttpMethod.Get,
        path: '/items/export/:format',
        pathParams: arg.pathParams,
        params: arg.params,
      };
    },
  },
});
```

## DTO Contracts

建議新增：

- `wallet-export-file-url.dto.ts`
- `export-file-format.enum.ts`
- `platform-export-wallet-ledger-items-csv-params.dto.ts`
- `merchant-export-wallet-ledger-items-csv-params.dto.ts`
- `platform-export-cashflow-items-csv-params.dto.ts`
- `merchant-export-cashflow-items-csv-params.dto.ts`

第一版 DTO shape 建議：

```ts
export enum ExportFileFormat {
  Csv = 'csv',
}
```

```ts
export type WalletExportFileUrlDto = {
  fileName: string;
  contentType: 'text/csv; charset=utf-8';
  downloadUrl: string;
  expiresAt: string | null;
};
```

```ts
export type PlatformExportCashflowItemsCsvParamsDto = {
  merchantIdIn?: string[];
  directionIn?: WalletCashflowItemDirection[];
  currencyCodeIn?: SupportedCurrencyCode[];
  effectiveAtGte?: string;
  effectiveAtLt?: string;
};
```

```ts
export type MerchantExportCashflowItemsCsvParamsDto = {
  directionIn?: WalletCashflowItemDirection[];
  currencyCodeIn?: SupportedCurrencyCode[];
  effectiveAtGte?: string;
  effectiveAtLt?: string;
};
```

ledger export params DTO 則採：

- platform: `merchantIdIn` / `walletIdIn` / `currencyCodeIn` / `kindIn` / `{timeField}Gte` / `{timeField}Lt`
- merchant: `walletIdIn` / `currencyCodeIn` / `kindIn` / `{timeField}Gte` / `{timeField}Lt`

## `defineApi` 寫法原則

本波不修改既有：

- `platLedgerApi`
- `merchLedgerApi`
- `platCashflowApi`
- `merchCashflowApi`

而是各自新增一個 export api。原因：

- 查詢 API 的 output 是 paging result
- export API 的 output 是 file payload
- 兩者雖然共用 base path，但 capability 不同

cashflow 範例如下：

```ts
export const platCashflowExportApi = defineApi({
  namespace: NAMESPACE,
  id: 'plat-cashflow-export' as const,
  identity: ApiIdentity.PlatformUser,
  basePath: '/wallet/plat/cashflow',
  endpoints: {
    platExportWalletCashflowItems(arg: {
      pathParams: { format: ExportFileFormat };
      params: PlatformExportCashflowItemsCsvParamsDto;
    }): ApiRequest<
      | ResultDto<CommonCode.Ok, WalletExportFileUrlDto>
      | ResultDto<CommonCode.Forbidden | CommonCode.ParamsInvalid>
    > {
      const { pathParams, params } = arg;
      return {
        method: HttpMethod.Get,
        path: '/items/export/:format',
        pathParams,
        params,
      };
    },
  },
});
```

merchant cashflow、platform ledger、merchant ledger 只需對應替換：

- `basePath`
- params DTO
- endpoint method name
- identity

## Barrel Export

barrel export 需一併補齊：

- `libs/contract-rest/src/lib/data-file/api/index.ts`
- `libs/contract-rest/src/lib/data-file/dto/index.ts`
- `libs/contract-rest/src/lib/data-file/index.ts`
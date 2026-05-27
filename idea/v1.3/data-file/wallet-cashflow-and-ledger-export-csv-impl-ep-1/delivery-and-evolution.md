# Wallet Export Delivery And Evolution Notes

本文整理 wallet cashflow / ledger export CSV EP-1 的錯誤處理、檔案變更範圍、驗證項目與後續演進切點。

## Error Handling

MVP 建議：

- identity type 不符：`CommonCode.Forbidden`
- params invalid：`CommonCode.ParamsInvalid`
- upstream query failure：映射成 `CommonCode.BadRequest` 或 `CommonCode.InternalError`
- csv write / temp file / blob upload 失敗：`CommonCode.InternalError`

若目前 `contract-rest/common` 尚未在此路徑慣用 `InternalError`，可先用現有錯誤碼落地；但正式實作仍應區分系統失敗與輸入錯誤。

## File Change Plan

### Contract Layer

- `libs/contract-rest/src/lib/data-file/api/*-export.api.ts`
- `libs/contract-rest/src/lib/data-file/dto/*export*.dto.ts`
- `libs/contract-rest/src/lib/data-file/index.ts`

### API Gateway

- `apps/esing-pay-api-gateway/src/rest/wallet/controller/*.controller.ts`
- `apps/esing-pay-api-gateway/src/rest/wallet/proxy/*-export.proxy.ts`
- `apps/esing-pay-api-gateway/src/rest/wallet/dto/*Export*.ts`
- `apps/esing-pay-api-gateway/src/rest/wallet/wallet.module.ts`

### data-file-service

- `apps/data-file-service/src/feature/wallet/wallet.module.ts`
- `apps/data-file-service/src/feature/wallet/wallet-context.module.ts`
- `apps/data-file-service/src/feature/wallet/service/*.service.ts`
- `apps/data-file-service/src/feature/wallet/use-case/*.use-case.ts`
- `apps/data-file-service/src/feature/wallet/mapper/*.mapper.ts`
- `apps/data-file-service/src/feature/wallet/rest/**`
- `apps/data-file-service/src/feature/feature.module.ts`

## Recommended MVP Scope

第一波只做：

1. 四支 export contract
2. API Gateway 四支 export endpoint
3. `data-file-service` 四支對應 rest modules
4. `data-file-service` 內部同步整批抓取 + CSV 產檔 + Azure Blob upload
5. API Gateway 回傳 `downloadUrl` 給前端

先不做：

1. queue / scheduler / job table
2. pre-signed URL lifecycle management 進階策略
3. export history list
4. retry / resume

## Verification Checklist

### Contract And DTO

- [ ] 新增 4 支 wallet export API contract
- [ ] 新增 `ExportFileFormat` enum
- [ ] 新增 `WalletExportFileUrlDto`
- [ ] 新增 4 組 export params DTO，且不帶 paging 欄位
- [ ] 補齊 `data-file` contract barrel exports

### API Gateway

- [ ] 四個既有 wallet controller 補上 `GET /items/export/:format`
- [ ] 新增四支 export proxy 並完成 rest-rpc topic 對接
- [ ] 新增四組 export query DTO
- [ ] controller 完成 `format` 驗證與 URL result 回傳
- [ ] `wallet.module.ts` 補齊 export proxy provider wiring

### data-file-service Structure

- [ ] 建立四支 export use-case
- [ ] 建立 source / merchant-timezone / csv / file service
- [ ] 建立 ledger / cashflow row mapper
- [ ] 建立四組 rest slice module / controller / service / mapper
- [ ] `wallet-context.module.ts` 匯入與匯出必要 provider / use-case

### Export Flow And Mapping

- [ ] source service 以固定批次 `page=1..N, size=1000` 拉完整資料
- [ ] row mapper 套用既有前端顯示語意
- [ ] merchant export 時間欄位套用 merchant 註冊時區轉換
- [ ] csv service 僅接收 row model 並產出 csv 實體檔
- [ ] file service 完成 blob upload、URL DTO 組裝與刪除暫存檔

### Verification

- [ ] 四支 export endpoint 都能成功回傳可下載 URL
- [ ] 空資料集時仍可下載 CSV 且標頭正確
- [ ] merchant 時區轉換結果正確
- [ ] 大筆數分頁拉取可完整匯出且不漏資料
- [ ] 補齊必要交付文件

## Future Extension Hooks

### Job-Based Export

把「撈資料 -> 轉 row -> 產 csv -> 上傳 blob 並回 URL」封裝在 feature-shared service，之後可改成：

- `create export job`
- queue worker 執行 csv generation
- job status query API

這樣不需要重寫 row mapper 與 csv writer。

### Cloud Storage

第一階段即採用 Azure Blob：

- local temp file -> azure blob upload
- 回 `downloadUrl` / `expiresAt`
- 刪除 local temp file

### Streaming

當資料量變大時，可把目前 `collect all rows in memory` 改成：

- paging fetch
- append csv chunk
- stream write

這不應影響 public export API path 與 controller 結構。

## Flexibility Guardrails

- 保持 `WalletExportFileUrlDto` 與內部流程分層，避免 controller 綁死 local-file 實作
- use-case 與 row mapper 不依賴儲存媒介，方便替換成其他 upload target
- file service 介面預留 `upload-and-return-url` 類型的擴充點
- 預留 job 模式資料模型欄位：`jobId` / `status` / `submittedAt` / `finishedAt` / `errorCode` / `errorMessage`
- 避免在 public API path 與 filter contract 引入只適用同步模式的耦合
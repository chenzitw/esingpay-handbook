---
status: draft
updated_at: 2026-06-04
updated_by: Claude
---

# Data File Export — Ledger and Cashflow to CSV 藍圖

## 背景

Data file export 是 v1.3 新增的資料輸出能力。平台後台與商戶後台可將
wallet ledger item 與 cashflow item 匯出為 CSV，取得一個有時效的簽名下載連結。

Export 是獨立操作邊界：不延伸既有 wallet query API，而是透過獨立的
`data-file` contract 命名空間，由 data-file-service provider 負責取資料、
生成 CSV、上傳雲端儲存，並回傳下載 URL。

---

## 交付範圍總覽

| Phase | 主題 | 前置 |
| --- | --- | --- |
| Phase 1 | Contract surface | — |
| Phase 2 | Gateway transport | Phase 1 |
| Phase 3 | Provider service | Phase 1 |
| Phase 4 | CSV 語意與檔案儲存 | Phase 3 |
| Phase 5 | 驗證與交付把關 | Phase 2, 4 |

---

## Phase 1：Contract surface

### 目標

凍結對外 contract surface，使 gateway 與 provider 可依此獨立實作，
不因後續決策漂移。

### 範圍

In scope：
- contract-rest 新增 `data-file` namespace
- 兩支 scope-level export API 定義
- 四個 request params DTO
- 一個 response DTO 與一個 format enum
- Barrel exports

Out of scope：
- Gateway controller / proxy 實作
- data-file-service REST server / use-case 實作
- CSV row DTO / mapper
- Queue / job / history / retry

### 凍結決策

**D1. Route namespace**
Export 路由掛在 `/data-file/...`，不延伸 `/wallet/...`。
Export 與 query 是不同操作語意，共用 basePath 會混責。

**D2. API 數量與 scope 結構**
兩支 API：`platWalletExportApi`（platform scope）與
`merchWalletExportApi`（merchant scope）。
每支 API 各含兩個 endpoint，不拆成四支獨立 API。

**D3. Endpoint key 命名**
`exportLedgerItems` 與 `exportCashflowItems`，兩個 scope 一致。

**D4. Response shape**
固定回傳 `WalletExportFileUrlDto`：
`fileName`、`contentType`、`downloadUrl`、`expiresAt`。
不回 inline binary，不回 job ID。

**D5. Timezone 輸入欄位**
Request DTO 只接受 `timezoneOffsetMinutes`（integer）。
`timezoneOffsetLabel` 不進 contract；顯示用 label 在 provider 層由
`timezoneOffsetMinutes` 派生，格式固定為 `UTC{十進位時差}`。

**D6. Currency filter 基數**
Request DTO 只接受單值 `currencyCode`，不接受 `currencyCodeIn` 或多值陣列。
Provider 內部呼叫 upstream wallet RPC 時，自行轉為 `currencyCodeIn: [currencyCode]`。

**D7. Request DTO 為 filter-only**
不含 `page`、`size`、`limit` 等 pagination 參數。
Export 批次拉取由 provider 自行控制，不由 caller 決定。

**D8. 不就地擴充 wallet query contract**
Export contract 落在 `libs/contract-rest/src/lib/data-file/`，
不修改既有 wallet query contract 檔案。

### 結束條件

- contract-rest data-file namespace 建立，barrel exports 完整
- `platWalletExportApi` 與 `merchWalletExportApi` 可 type-safe 推導
  四個 endpoint 的 Input / Output
- Request params DTO 不含 `timezoneOffsetLabel`、`currencyCodeIn`、
  pagination 參數
- Merchant DTO 不含 `merchantIdIn`
- Wallet query contract 檔案未被修改

---

## Phase 2：Gateway transport

### 目標

在 API Gateway 建立 data-file export 的 HTTP 接收與 RPC 轉發層，
完成端到端路由接通。

### 範圍

In scope：
- API Gateway data-file export controller
- API Gateway data-file export proxy（RestRpcClient 轉發至 data-file-service）
- Identity wiring
- Route 與 module 註冊

Out of scope：
- data-file-service provider 實作
- CSV 生成邏輯
- Azure storage 串接

### 凍結決策

**D1. Identity pass-through**
Gateway 從 request context 取得 identity 後，原樣穿傳至 RPC request。
不替換為固定 system user ID（如 SYSTEM_PLATFORM_USER_ID）。
Identity 替換會導致 provider 端無法正確判斷操作者 scope。

**D2. Gateway module 與 wallet query 分離**
Export controller / proxy 建立獨立的 data-file gateway module，
不放入既有 wallet query module，不共用 proxy 或 controller 檔案。

**D3. Controller 無業務邏輯**
Controller 只做 transport：接收 HTTP request、呼叫 proxy、回傳結果。
不含業務判斷或資料轉換，所有邏輯在 provider 層處理。

**D4. Format path param 在 gateway 層驗證**
Path param `format` 以 `ExportFileFormat` enum 驗證。
非 enum 值回 `ParamsInvalid`，不穿透至 provider。

### 結束條件

- 四個 export endpoint 在 gateway 可接收 HTTP 請求並轉發 RPC
- Identity 從 request context 穿傳，無固定 system user ID
- Export gateway module 與既有 wallet query module 無交叉依賴
- Format 非合法 enum 值回 `ParamsInvalid`

---

## Phase 3：Provider service

### 目標

在 data-file-service 建立 export 的 REST server 接收層與 use-case 編排層，
完成 RPC 接通至業務流程入口。

### 範圍

In scope：
- 2 組 scope-level REST slice（plat-wallet-export / merch-wallet-export）：controller / service / mapper
- 4 支 use-case（export-platform-ledger / export-merchant-ledger /
  export-platform-cashflow / export-merchant-cashflow）
- WalletContextModule（shared service 集中提供）
- Provider 端 identity 驗證
- Module 註冊

Out of scope：
- CSV 生成實作（Phase 4）
- Azure Blob 上傳（Phase 4）
- Source fetch 實作細節（Phase 4）

### 凍結決策

**D1. 每個 export operation 對應一支 use-case**
四種 export（plat/merch × ledger/cashflow）各自一支 use-case，
不共用泛型 use-case。流程差異（merchant timezone 轉換、identity 類型）
在 use-case 層顯式處理，避免 flag/switch 混入 service。

**D2. WalletContextModule 集中提供 shared services**
Source fetch、CSV 生成、檔案上傳、merchant timezone 等跨 slice 共用的 service，
統一由 `wallet-context.module.ts` 提供與匯出，
REST slice module 僅 import WalletContextModule，不自行宣告共用 provider。

**D3. REST slice 為 scope-level 結構**
REST slice 以 scope 為單位建立，共 2 組：
- `rest/plat-wallet-export/`：承載 platform ledger 與 cashflow export
- `rest/merch-wallet-export/`：承載 merchant ledger 與 cashflow export

每組 slice 各自有 module / controller / service / mapper，
對齊 `platWalletExportApi` / `merchWalletExportApi` 的 scope-level contract 結構。

**D4. Provider 端再次驗證 identity 類型**
REST service 層在執行業務前，驗證 RPC request 的 identity type
與 scope 相符（platform slice 要求 `platform_user`，merchant slice 要求 `merchant_user`）。
不符者回 `Forbidden`，不依賴 gateway 已驗證的假設。

**D5. REST service 無業務邏輯**
REST service 只負責：identity 驗證、組裝 use-case input、呼叫 use-case、
回傳 REST result。Source fetch、CSV 生成、上傳邏輯不在此層。

### 結束條件

- 四組 REST slice 可接收 RPC request 並轉入 use-case
- WalletContextModule 正確提供並匯出所有 shared service
- Identity type 不符時回 `Forbidden`
- Use-case 呼叫介面已定義（實作可 stub，Phase 4 補齊）
- Module 註冊完整，data-file-service 可啟動不報錯

---

## Phase 4：CSV 語意與檔案儲存

### 目標

實作 source 批次拉取、row 轉換、CSV 生成、雲端上傳與簽名 URL 回傳，
完成 use-case 的完整執行能力。

### 範圍

In scope：
- Source 批次拉取（WalletExportSourceService）
- Ledger / cashflow row mapper
- CSV 生成（WalletExportCsvService）
- 暫存檔建立與 best-effort 刪除
- Azure Blob 上傳（FileStoragePort 抽象）
- 簽名下載 URL 組裝（WalletExportFileUrlDto）
- Merchant timezone label 派生

Out of scope：
- Job queue / 非同步模式
- Export history / retry / resume
- Stream-based CSV writer 重構

### 凍結決策

**D1. 批次大小固定為 100**
Source 批次拉取每頁固定 `size=100`，不由 caller 控制。
此值為保守估計，留有後續依容量觀測調整的空間。

**D2. currencyCode 在 source 層轉為 currencyCodeIn**
Export request 接受單值 `currencyCode`；
WalletExportSourceService 呼叫 upstream wallet RPC 時，
自行轉為 `currencyCodeIn: [currencyCode]`，不修改 contract DTO。

**D3. CSV header 使用 explicit display title**
CSV 欄位標頭使用人類可讀的顯示名稱，不使用 DTO field key。
動態 token（`{Currency Code}`、`{UTC label}`）在輸出前 resolve，
不得以原始 token 字串出現在輸出檔案中。

**D4. Timezone label 由 timezoneOffsetMinutes 派生**
Merchant timezone 顯示 label 格式固定為 `UTC{十進位時差}`，
例如 `480 → UTC+8`、`330 → UTC+5.5`、`0 → UTC+0`。
不接受外部傳入 label 字串。

**D5. 儲存層透過 FileStoragePort 抽象**
use-case / CSV service / REST controller 均只依賴 `FileStoragePort` 介面，
不直接引用 Azure SDK。
僅 adapter 層（`FileStoragePort` 的實作）可依賴 Azure SDK。

**D6. 暫存檔 best-effort 刪除**
CSV 上傳 blob 成功後，立即執行 best-effort 本地暫存檔刪除。
刪除失敗不影響 export response，但須記錄 log。

**D7. ledger row 不含 actions 欄位**
`WalletLedgerItemSheetRowDto` 不包含 `actions` 欄位，
避免將後台操作入口語意混入資料匯出。

### 結束條件

- Source 批次拉取以 size=100 完整取回所有符合條件的資料
- Row mapper 輸出欄位與 CSV header display title 一致
- CSV header 無未 resolve 的動態 token
- Ledger row 不含 `actions` 欄位
- 上傳完成後取得可下載的 `downloadUrl` 與 `expiresAt`
- 非 adapter 層無直接 Azure SDK import
- 暫存檔於上傳成功後執行刪除

---

## Phase 5：驗證與交付把關

### 目標

確認四個 export endpoint 的功能、輸出、邊界、語意與錯誤行為均符合預期，
建立可機械判讀的交付門檻。

### 範圍

In scope：
- 五軸驗證矩陣
- Release guardrails 確認
- Evolution hooks 保留確認

Out of scope：
- Job queue / 非同步模式驗證
- Live Azure signed URL TTL 端到端驗證（列為 deferred risk）
- Stream writer 重構

### 凍結決策

**D1. 驗證分五軸**
- 功能完整性：四個 endpoint 各自成功回傳可下載 URL
- 輸出完整性：CSV header 使用 display title、動態 token 已 resolve、ledger 不含 `actions`
- 邊界完整性：空資料集、大筆數分頁、signed URL TTL
- 語意完整性：merchant timezone 轉換、scope filter 邊界、storage 抽象邊界、
  endpoint key 對齊、timezone label 派生、單值 currencyCode 行為
- 錯誤與權限：identity 不符、params 無效、上游失敗、CSV 處理失敗

**D2. Release guardrails**
本版交付不得突破以下邊界，否則視為 scope 膨脹需重新評估：
- 不擴充為 job / queue / 非同步模式
- 不修改 public contract path 或 filter 欄位
- Provider 內部執行細節不穿透至 response

**D3. Evolution hooks 交付時必須保留**
以下三個切點須確保在實作中存在，不得因實作收斂而消除：
- Job mode 擴充切點（use-case 可替換為 enqueue 行為）
- Cloud storage policy 擴充切點（FileStoragePort 可替換實作）
- Stream-based export 擴充切點（CSV writer 可替換為 stream 模式）

### 驗證軸向說明

| 軸向 | 覆蓋範圍 |
| --- | --- |
| 功能完整性 | 四個 endpoint 各自成功回傳，signed URL 可存取 |
| 輸出完整性 | Header display title、動態 token resolve、欄位包含與排除 |
| 邊界完整性 | 空資料集、大筆數、signed URL TTL 邊界 |
| 語意完整性 | Timezone 轉換與 fallback、scope filter、storage 抽象、contract 對齊 |
| 錯誤與權限 | Forbidden、ParamsInvalid、上游失敗、CSV 失敗、無 job cache 依賴 |

### 結束條件

- 五軸各有可重現 input 與單一預期結果的驗證案例
- Release guardrails 三條均通過或明確標記為 deferred
- 三個 evolution hooks 切點確認存在
- 交付結論可由 checklist 機械產出

---

## 全局邊界宣告

本版交付為同步 MVP，以下能力明確不在範圍內：

- Queue / scheduler / job table
- Export history / status query / retry / resume
- Stream writer 重構
- Public contract 二次改版
- Signed URL lifecycle 管理進階策略（revoke、refresh）

## 風險與緩解

| 風險 | 緩解 |
| --- | --- |
| 大資料量造成長 request 與記憶體壓力 | 批次拉取 size=100、保留 job mode 切點、容量觀測後再決定門檻 |
| CSV 顯示語意與 console 漂移 | Row mapper 對齊 console 既有顯示轉換規則，以 golden sample 驗收 |
| Signed URL TTL 或權限設定錯誤 | 驗證軸向邊界完整性涵蓋 TTL 案例；live Azure 驗證列為 deferred |
| Storage 抽象漏洞導致層間耦合 | Phase 5 驗證語意完整性軸向含 FileStoragePort override 測試 |
| Identity 未穿傳導致 scope 錯誤 | Phase 2 / 3 均列為凍結決策，Phase 5 語意完整性軸向含 scope filter 驗證 |

## Open Points

- 時間區間第一版實際開放欄位（`effectiveAtGte/Lt` 或 `createdAtGte/Lt`）
- `expiresAt` 在 MVP 是否可固定為 `null`（待 live Azure 環境確認）
- Signed URL TTL 值（目前 plan 層設定為 1 小時，是否需在 blueprint 層凍結）

---
status: draft
updated_at: 2026-06-09
updated_by: Codex
---

# REST API — Stage 7 文件網站

## 本版要做的事情

Stage 7 目標是建立 REST API 對外文件網站的第一版，讓商戶可以依文件完成基本 API 串接。

本版已確認：

- 文件網站 framework 使用 Docusaurus。
- Docusaurus 官方文件：https://docusaurus.io/docs
- 文件以 Markdown / MDX 撰寫。
- 先建立 REST API 文件網站的基本站台與文件目錄結構。
- 先以 Stage 1 的 `GET /external/v1/withdrawal-intents` 作為第一個 endpoint 文件範例。
- 建立 endpoint 文件固定格式，後續 deposits、wallets、withdrawal create 可沿用。
- 建立 common docs，例如 authentication、pagination、error response。
- 建立 models docs，例如 withdrawal intent response DTO。
- 建立新的 docs domain 上線路徑。
- 第一版部署優先使用 GitHub Pages + GitHub Actions + custom domain。
- 第一版只建立 production 環境。
- Production domain 暫定 `docs.esing.io`。

本版暫不做：

- SDK generation。
- 多語系文件。
- 文件版本切換。
- IP whitelist 文件；Stage 5 本版本不做。

第一版先用手寫 MDX 確立文件格式與維護方式。

---

## 階段拆分

### Phase 1：Docusaurus 站台基礎設定

確認或建立 REST API docs site 的 Docusaurus 專案。

Framework reference：

- [Docusaurus Documentation](https://docusaurus.io/docs)

工作內容：

- 設定 site title / navbar / footer。
- 移除 Docusaurus demo tutorial / blog 內容。
- 設定 docs sidebar。
- 設定 production URL / base URL。
- 確認 build output 可部署為靜態網站。
- 確認 docs domain 指向方式。

第一版不需要客製化太多 UI。重點是先讓文件能穩定產出、部署、閱讀。

### Phase 2：部署策略

第一版部署優先採用：

```text
GitHub Pages + GitHub Actions + custom domain
```

Production domain 暫定：

```text
docs.esing.io
```

本版只建立 production 環境，不另外建立 staging docs site。

GitHub Actions workflow：

- PR 時執行 Docusaurus build check，不部署。
- merge / push 到 main 後 build Docusaurus site。
- 將 Docusaurus `build` output 部署到 GitHub Pages。

GitHub Pages 設定：

- 使用 custom domain `docs.esing.io`。
- 啟用 HTTPS。
- Docusaurus `url` 對應 `https://docs.esing.io`。
- Docusaurus `baseUrl` 使用 `/`。

Azure Static Web Apps 暫不作為第一版部署目標。

後續若有 Azure governance、preview environment、auth / authorization、serverless API 或 corporate cloud policy 需求，再評估 Azure Static Web Apps。

### Phase 3：文件目錄結構

建議文件放在：

```text
docs/
  rest-api/
    _category_.json
    v1/
      _category_.json
      overview.mdx
      quickstart.mdx
      authentication.mdx
      pagination.mdx
      errors.mdx
      changelog.mdx

      endpoints/
        _category_.json

        withdrawal-intents/
          _category_.json
          list.mdx
          retrieve.mdx
          create.mdx

        deposits/
          _category_.json
          list.mdx
          retrieve.mdx

        wallets/
          _category_.json
          list.mdx
          retrieve.mdx

      models/
        _category_.json
        withdrawal-intent.mdx
        deposit.mdx
        wallet.mdx
        common.mdx
```

`rest-api/v1` 對應 API path `/external/v1/*`。第一版先用路徑表示 API version，不啟用 Docusaurus 內建 versioning。

### Phase 4：共用串接文件

先建立商戶串接會先讀的基礎文件：

- `overview.mdx`
- `quickstart.mdx`
- `authentication.mdx`
- `pagination.mdx`
- `errors.mdx`

文件內容應聚焦 external client 需要知道的資訊，不描述 internal gateway / cradle / fund / wallet 實作細節。

`authentication.mdx` 說明：

- API key 放在哪個 header。
- API key 如何由 Console 建立。
- API key 遺失或刪除後的處理方式。

`pagination.mdx` 說明：

- `page`
- `size`
- response paging shape

`errors.mdx` 說明：

- common error response shape
- authentication failed
- params invalid
- not found

### Phase 5：第一個 endpoint 文件範例

第一個 endpoint 文件使用 Stage 1 已落地的：

```http
GET /external/v1/withdrawal-intents
```

文件位置：

```text
docs/rest-api/v1/endpoints/withdrawal-intents/list.mdx
```

此文件需建立後續 endpoint 可沿用的固定格式：

```text
Title
Method / Path
Description
Authentication
Query Parameters
Response Body
Response Example
Error Codes
Related Models
```

第一版 query params 至少包含：

- `page`
- `size`

若 Stage 1 Phase 4 已完成 external status mapping，則補：

- `statusIn`

Response example 應使用 `ExternalWithdrawalIntentDto`，不得暴露 internal `WithdrawalIntentDto`、actor、status history、raw bigint、Map 或 internal network endpoint lifecycle。

### Phase 6：Models 文件

建立 withdrawal intent response DTO 文件：

```text
docs/rest-api/v1/models/withdrawal-intent.mdx
```

內容包含：

- `ExternalWithdrawalIntentDto`
- `ExternalWithdrawalWalletDto`
- `ExternalWithdrawalTransactionDto`
- `ExternalBlockchainTransactionDetailDto`
- `ExternalWithdrawalLegDto`
- `ExternalWithdrawalDestinationDto`
- `ExternalWithdrawalIntentStatus`

Models 文件只描述 external contract，不引用 internal DTO 名稱作為正式 schema。

若需要說明狀態語意，應使用 external status：

```text
pending
processing
completed
failed
cancelled
```

不要在對外文件列出 internal statuses，例如：

```text
preparing
reviewing
processed
transacting
declined
rejected
```

### Phase 7：後續 endpoint 擴充

等第一個 withdrawal intent list 文件格式確認後，再依同一套格式擴充：

- `GET /external/v1/withdrawal-intents/{id}`
- `POST /external/v1/withdrawal-intents`
- `GET /external/v1/deposits`
- `GET /external/v1/deposits/{id}`
- `GET /external/v1/wallets`
- `GET /external/v1/wallets/{id}`

每個 endpoint 文件都應連到對應 models 文件，避免在 endpoint 頁重複完整 DTO schema。

---

## 文件撰寫原則

### 面向商戶系統整合

文件讀者是商戶工程師，不是內部開發者。

文件應描述：

- 如何取得 API key。
- 如何呼叫 API。
- request 需要帶什麼。
- response 會回什麼。
- 錯誤如何處理。
- 狀態如何理解。

文件不應描述：

- cradle / fund / wallet module 內部責任。
- facade / query service / repository。
- internal DTO。
- actor / status history。
- wallet allocation。
- fee payer strategy。

### Endpoint 文件固定格式

每個 endpoint 文件應使用相同順序：

```text
# <Action Name>

<HTTP_METHOD> <PATH>

## Description

## Authentication

## Request

## Response

## Example

## Error Codes

## Related Models
```

### Example 必須可直接使用

範例 request / response 應貼近真實 API。

不要使用：

- internal id
- raw bigint
- mock id
- internal status
- hidden field

### Models 文件作為 schema source

Endpoint 頁不重複貼完整 nested DTO。

Endpoint 頁只貼常用 example，完整欄位以 models 文件為準。

---

## 驗收標準

- Docusaurus site 可 build。
- GitHub Actions PR build check 可執行。
- main branch 可自動部署到 GitHub Pages。
- production custom domain 暫定 `docs.esing.io`。
- docs sidebar 可正確顯示 REST API / v1。
- `GET /external/v1/withdrawal-intents` 文件已完成。
- `ExternalWithdrawalIntentDto` model 文件已完成。
- authentication / pagination / errors 文件已完成第一版。
- 文件不暴露 internal DTO 或 internal lifecycle。
- 文件中的 endpoint path 與 contract-rest external API 對齊。
- 文件中的 response example 與 external mapper output 對齊。

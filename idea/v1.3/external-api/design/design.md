# 對外商戶 API 設計筆記：Deposit、Withdrawal Intent 與 Wallet

## 背景

目前內部已存在 Merchant Portal 使用的 Deposit / Withdrawal 相關 API，例如：

- Deposit detail / list：由 Fund `DepositDto` 對應
- Create withdrawal intent：`POST /fund/merch/withdrawal-intents`
- Wallet detail / list：由 Wallet DTO 對應

但對外 API 的目標不同。External API 面向的是商戶系統整合，而不是內部 portal / back office，因此應避免直接暴露內部 domain workflow、actor、projection、correlation、network endpoint lifecycle 等結構。

---

## 1. 設計方向：以業務流 DTO 做 external mapping

### 1.1 External DTO Mapping

External merchant 串接 API 時，主要關心的是可直接整合的業務資源：

- 我要查入金紀錄
- 我要查單筆入金狀態
- 我要建立出金申請
- 我要查出金申請狀態
- 我要查可用 wallet

External API 以既有業務流 DTO 作為資料來源，再透過 mapping layer 轉換成 external-facing DTO：

```text
DepositDto
  -> ExternalDepositDto

WithdrawalIntentDto
  -> ExternalWithdrawalIntentDto

WalletDto / RelatedWalletDto
  -> ExternalWalletDto
```

External API 可以重用內部業務單資料，但要重新定義 external-facing DTO，不直接輸出內部 REST DTO。

### 1.2 對外 API 應隱藏內部實作與 portal workflow

External DTO 不應直接暴露以下內部概念：

- correlation
- wallet allocation id
- network endpoint lifecycle
- fee payer wallet strategy
- submittedBy / statusHistory actor details

這些都屬於內部實作或 portal workflow，不應成為 public API contract。

### 1.3 目前開放 API Surface

#### Deposit

```http
GET /external/v1/deposits
GET /external/v1/deposits/{id}
```

#### Withdrawal Intent

```http
GET  /external/v1/withdrawal-intents
GET  /external/v1/withdrawal-intents/{id}
POST /external/v1/withdrawal-intents
```

#### Wallet

```http
GET /external/v1/wallets
GET /external/v1/wallets/{id}
```

---

## 2. 對外 Deposit API

### Endpoints

```http
GET /external/v1/deposits
GET /external/v1/deposits/{id}
```

目前對內 `DepositDto` 欄位對照另見：

- [`reference-internal-deposit-dto.md`](../references/reference-internal-deposit-dto.md)

External Deposit DTO 草案另見：

- [`design-deposit-dto.md`](./design-deposit-dto.md)

---

## 3. 對外 Withdrawal Intent API

### GET Withdrawal Intent

```http
GET  /external/v1/withdrawal-intents
GET  /external/v1/withdrawal-intents/{id}
```

> Note：目前對外命名先沿用 business resource `withdrawal-intents`，避免把內部仍存在的 intent lifecycle 偽裝成已完成 withdrawal。

目前對內 GET response `WithdrawalIntentDto` 欄位對照另見：

- [`reference-internal-withdrawal-intent-dto.md`](../references/reference-internal-withdrawal-intent-dto.md)

External GET response DTO 草案另見：

- [`design-withdrawal-intent-dto.md`](./design-withdrawal-intent-dto.md)

Transaction legs currencyCode amendment 留底另見：

- [`amendment-external-transaction-leg-currency-code.md`](../amendments/amendment-external-transaction-leg-currency-code.md)

### POST Withdrawal Intent

```http
POST /external/v1/withdrawal-intents
```

目前對內 POST request DTO 欄位對照另見：

- [`reference-internal-submit-withdrawal-intent-dto.md`](../references/reference-internal-submit-withdrawal-intent-dto.md)

External POST request DTO 草案另見：

- [`design-submit-withdrawal-intent-dto.md`](./design-submit-withdrawal-intent-dto.md)

### 外部 API 參考

以下 API 與 eSing Pay withdrawal intent / payout 類功能相近，可作為 external DTO 與 endpoint 設計參考：

| Provider | Reference | 對應概念 |
| --- | --- | --- |
| Uniwire | [Payouts](https://docs.uniwire.com/api/payouts) | Payout list / get / create、recipient、reference id、network fee。 |
| Stripe | [Create a payout](https://docs.stripe.com/api/payouts/create) | 建立 payout、amount / currency / destination / metadata、payout status。 |
| UniPayment | [Payment Object](https://unipayment.readme.io/reference/payment-object) | 泛用 payment object、transfer method、network、destination object，支援 blockchain / bank / internal account。 |

---

## 4. 對外 Wallet API

### Endpoints

```http
GET /external/v1/wallets
GET /external/v1/wallets/{id}
```

目前對內 merchant wallet API / DTO 欄位對照另見：

- [`reference-internal-wallet-dto.md`](../references/reference-internal-wallet-dto.md)

External Wallet DTO 草案另見：

- [`design-wallet-dto.md`](./design-wallet-dto.md)

### Wallet Resolution

External client 可以透過 `GET /external/v1/wallets` 取得可用 wallet，再於 `POST /external/v1/withdrawal-intents` 帶入 `senderWalletId`。

後端仍需以 API key 對應的 merchant 驗證 wallet 是否屬於該 merchant：

```text
API key -> merchantId
merchantId + senderWalletId -> internal senderWalletId
```

### 為什麼不直接暴露完整 wallet model

External merchant 可以理解「從哪個 wallet 出金」，但不需要理解內部 wallet 結構。

External DTO 應避免暴露：

- `merchantId`
- `networkEndpointId`
- wallet allocation strategy
- fee payer wallet strategy
- 未來 wallet selection logic 可能會改變
- 多鏈 / 多資產支援應由系統 config 決定

---

## 5. Reference ID

TODO：確認 external withdrawal intent request / response 是否正式加入 `referenceId`。

`referenceId` 是商戶自己系統裡的參考 ID。

它用來把 eSing Pay withdrawal intent record 對應回商戶自己系統裡的出金單。

例如：

```text
referenceId = 商戶自己的出金單號
id = eSing Pay withdrawal intent id
```

### 範例

商戶建立 withdrawal intent：

```json
{
  "senderWalletId": "BO6EW2NM",
  "asset": {
    "currencyCode": "USDT",
    "chainId": "tron:mainnet"
  },
  "amount": "100",
  "destination": {
    "address": "TXxxxx..."
  },
  "referenceId": "WD-20260520-0001"
}
```

eSing Pay response：

```json
{
  "id": "WI00BO6EW2NM",
  "referenceId": "WD-20260520-0001",
  "status": "pending"
}
```

這可以讓商戶知道：

```text
eSing Pay withdrawal intent WI00BO6EW2NM
= merchant withdrawal order WD-20260520-0001
```

### 使用情境 1：對帳

當商戶匯出或比對 records 時，可以使用：

```text
referenceId
```

將 eSing Pay records 與商戶內部訂單對應起來。

### 使用情境 2：Webhook 識別

未來 webhook payload：

```json
{
  "eventType": "withdrawal_intent.completed",
  "data": {
    "id": "WI00BO6EW2NM",
    "referenceId": "WD-20260520-0001",
    "status": "completed"
  }
}
```

商戶收到 webhook 後，可以直接更新自己系統裡對應的 withdrawal order，不需要額外維護 mapping table。

---

## 6. API Key 與 Console 管理

External API 使用 API key 作為商戶系統呼叫對外 API 的認證方式。

第一版 API key 管理由 Console 提供，讓商戶可以自行新增 API key。

### 第一版範圍

第一版只支援：

- 新增 API key。
- 查看既有 API key 列表。
- 停用 API key。
- 設定 API key 對應的 IP whitelist。

第一版不支援：

- 編輯既有 API key。
- API key scope。
- API key prefix。
- `Idempotency-Key`。

API key 建立後若需要調整權限或重新配置，商戶應停用既有 key 並新增一把新的 API key。

### API Scope

第一版不處理 API key scope。

只要使用有效且未停用的 API key，即可呼叫目前開放的 external API。

未來若需要分離查詢與寫入權限，再新增 scope 設計，例如：

- read deposits
- read wallets
- create withdrawal intents

---


## 7. IP Whitelist

IP whitelist 設計另見：

- [`design-ip-whitelist.md`](./design-ip-whitelist.md)


---

## 8. 文件網站

External API 文件預計以 Markdown / MDX 撰寫，後續會研究適合的文件網站 framework。

候選方向：

- Docusaurus：支援 Markdown / MDX、版本化文件、側邊欄、搜尋與靜態站台輸出，適合 API 文件網站。
- 其他 Markdown / MDX-based docs framework：可再依部署方式、搜尋需求、版本管理與維護成本比較。

Demo / reference：

- [Docusaurus OpenAPI Docs Demo](https://docusaurus-openapi.netlify.app/docs/tutorial-basics/create-a-page)

TODO：確認文件網站是否需要 API reference 自動生成、OpenAPI 整合、版本切換、範例程式碼 tabs、搜尋服務與部署方式。

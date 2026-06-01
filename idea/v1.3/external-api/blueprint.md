**external response shape 的設計草案**，包含：

```txt
GET /external/v1/withdrawal-intents
GET /external/v1/withdrawal-intents/{id}
```

而且明確說 external 不應直接暴露 internal `WithdrawalIntentDto`、完整 `CorrelatedNetworkTransactionDto`、actor、status history、fee payer strategy。

下面給你一版 **high-level 新增這個 API 的檔案流程**，只講新增/修改什麼，不展開細節程式碼。

---

# External Withdrawal Intent API 新增流程

## Phase 1：Contract REST

先把 handbook 的 DTO 草案落成正式 `contract-rest`。

### 新增

```txt
libs/contract-rest/src/lib/fund/api/external-withdrawal-intent.api.ts
```

定義：

```txt
GET /external/v1/withdrawal-intents
GET /external/v1/withdrawal-intents/:id
```

---

### 新增

```txt
libs/contract-rest/src/lib/fund/dto/external-withdrawal-intent.dto.ts
```

放 handbook 定義的：

```txt
ExternalWithdrawalIntentDto
WalletDto
WithdrawalTransactionDto
BlockchainTransactionDetailDto
BankTransferDetailDto
ExternalLegsDto
ExternalWithdrawalDestinationDto
ExternalWithdrawalIntentStatus
```

這些欄位來源就是 handbook 裡的 external DTO 設計。

---

### 新增

```txt
libs/contract-rest/src/lib/fund/dto/external-search-withdrawal-intent-params.dto.ts
```

定義 list API query params，例如：

```txt
page
size
statusIn
```

第一版可以先簡單，不一定要把所有 filter 都開出來。

---

### 修改

```txt
libs/contract-rest/src/lib/fund/api/index.ts
libs/contract-rest/src/lib/fund/dto/index.ts
```

把新的 external api / dto export 出去。

---

## Phase 2：API Gateway

在 gateway 新增 external controller / proxy，但放在既有 `rest/fund` 結構下。

### 新增

```txt
apps/esing-pay-api-gateway/src/rest/fund/controller/external-withdrawal-intent.controller.ts
```

作用：

```txt
接 GET /external/v1/withdrawal-intents
接 GET /external/v1/withdrawal-intents/:id
mock external merchant identity
呼叫 ExternalWithdrawalIntentProxy
```

---

### 新增

```txt
apps/esing-pay-api-gateway/src/rest/fund/proxy/external-withdrawal-intent.proxy.ts
```

作用：

```txt
使用 externalWithdrawalIntentApi 建立 RestRpc topic
透過 RestRpcClient 打到 cradle
```

---

### 新增

```txt
apps/esing-pay-api-gateway/src/rest/fund/dto/ExternalSearchWithdrawalIntentParamsQuery.ts
```

作用：

```txt
接收 external list API 的 query params
```

---

### 新增

```txt
apps/esing-pay-api-gateway/src/core/rest-rpc/create-external-user-identity.ts
```

第一版先 mock：

```txt
type = external_merchant
merchantId = MOCK_MERCHANT_ID
```

之後 API key 做好，再改成從 API key record 取 merchantId。

---

### 修改

```txt
apps/esing-pay-api-gateway/src/core/rest-rpc/index.ts
```

export：

```txt
createExternalUserIdentity
```

---

### 修改

```txt
apps/esing-pay-api-gateway/src/rest/fund/<fund-rest-module>.ts
```

註冊：

```txt
ExternalWithdrawalIntentController
ExternalWithdrawalIntentProxy
```

---

## Phase 3：Cradle external rest adapter

在 cradle 新增 external adapter，但不要重寫業務流程。

### 新增資料夾

```txt
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/rest/external-withdrawal-intent/
```

---

### 新增

```txt
external-withdrawal-intent.module.ts
external-withdrawal-intent.controller.ts
external-withdrawal-intent.service.ts
external-withdrawal-intent.mapper.ts
```

作用分別是：

```txt
module      註冊 external adapter providers
controller  接 externalWithdrawalIntentApi 對應的 MessagePattern
service     做 identity check、use-case 呼叫、error mapping、result wrapping
mapper      external params -> use-case query
            WithdrawalIntentComposed -> ExternalWithdrawalIntentDto
```

---

## Phase 4：Cradle module wiring

### 修改

```txt
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/<withdrawal-intent-module>.ts
```

或實際聚合 rest modules 的 module。

加入：

```txt
ExternalWithdrawalIntentRestModule
```

讓新的 `@MessagePattern` controller 生效。

---

# 會重用的既有檔案

這些不用新增 external 版本：

```txt
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/use-case/merchant-search-withdrawal-intents.use-case.ts
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/use-case/merchant-get-withdrawal-intent.use-case.ts
apps/esingpay-cradle/src/fund/repository/withdrawal-intent.repository.ts
apps/esingpay-cradle/src/fund/feature/withdrawal-intent/composer/withdrawal-intent.composer.ts
```

原因：

```txt
external API 本質上也是 merchant scope 查詢 withdrawal intent
差異只在入口身份與 response DTO
```

所以重用現有 use-case / repository / composer，新增 external adapter / mapper 即可。

---

# 不直接重用的既有檔案

不建議直接重用：

```txt
merchWithdrawalIntentApi
WithdrawalIntentDto
WithdrawalIntentRestMapper.toWithdrawalIntentDto()
```

因為 handbook 已經明確定義 external DTO 不要暴露 internal `WithdrawalIntentDto`、actor、status history、fee payer strategy 等內部資訊。

所以 external 需要自己的：

```txt
externalWithdrawalIntentApi
ExternalWithdrawalIntentDto
ExternalWithdrawalIntentMapper
```

---

# 最終 high-level 流程

```txt
1. handbook DTO 草案
   ↓
2. 落成 contract-rest DTO + externalWithdrawalIntentApi
   ↓
3. API Gateway 新增 external controller / proxy / mock identity
   ↓
4. Cradle 新增 external rest adapter
   ↓
5. Adapter 重用既有 search/get use-case
   ↓
6. Repository / Composer 照舊 (fund)
   ↓
7. External mapper 輸出 ExternalWithdrawalIntentDto
```

一句話：

```txt
新增 external contract 與 adapter，重用既有 business flow。
```

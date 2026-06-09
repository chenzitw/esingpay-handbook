
# KYT TRON 黑名單檢查 — Design

## Context

TRON 鏈上的 USDT（TRC20）合約持有一份黑名單（Blacklist），由 Tether Limited 維護。被列入黑名單的地址將無法接收或轉出 USDT。

系統需要在處理入金認定或出金授權前，具備查詢目標地址黑名單狀態的能力，以滿足基本的 KYT（Know Your Transaction）合規要求，避免資金與受凍結地址互動。

此能力屬於 **Network domain** 的職責（向外部網路查詢地址合規狀態），由 `libs/services/tron` 提供，**不侵入 Fund domain**，由 Fund 的 use case caller 自行決定業務處置邏輯。

## Scope

**In scope：**
- 查詢單一 TRON Base58 地址是否被 USDT TRC20 合約列入黑名單
- 錯誤情境下的透明 re-throw，保留完整 error context 讓 caller 決策

**Out of scope：**
- 黑名單查詢結果的持久化或快取
- 其他鏈（Ethereum、BSC 等）的 USDT 黑名單
- 主動監聽黑名單異動事件（event streaming）
- 自動封鎖業務流程的規則（屬 Fund domain 職責）

## Domain Model

### 外部事實提供者

USDT TRC20 智能合約（鏈上地址：`TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t`）持有黑名單狀態，透過 `getBlackListStatus(address: address) returns (bool)` view function 公開查詢。

此事實由合約方（Tether Limited）單方面維護，系統只讀，不擁有。

### 黑名單查詢服務

系統引入一個新的 Tron domain service，其職責單一：接受一個 TRON 地址作為輸入，回傳一個布林值表示該地址是否在黑名單中。

- 輸入：TRON 錢包地址（Base58 格式，`T` 開頭）
- 輸出：`true`（在黑名單）/ `false`（未在黑名單）
- 異常：任何無法取得確定結果的情況皆以異常形式回報

此服務為純查詢（read-only）行為，不產生任何鏈上交易，不消耗 Energy 或 Bandwidth。

## Constraints

- 查詢必須透過節點 RPC 進行，結果的即時性取決於 TRON 節點的區塊同步狀態
- 未掛載 API Key 時，TronGrid 公共節點限制 3 RPS；超量請求將收到 HTTP 429，此為外部依賴的速率限制，需在部署設定層面處理
- 地址格式必須為 TRON Base58（以 `T` 開頭、34 字元），非法格式將在 SDK ABI 編碼階段即被拒絕
- 此服務不具備自我重試能力；呼叫方負責決定重試策略

## Rejected Alternatives

**替代方案 A：透過 REST API 間接查詢（TronScan / TronGrid API）**
- 優點：不依賴 SDK ABI 呼叫，介面較直觀
- 拒絕理由：增加額外的 HTTP 外部服務依賴；TronScan API 不保證即時性；合約直接呼叫才是事實來源

**替代方案 B：在服務內部快取結果（e.g. in-memory Map + TTL）**
- 優點：降低 Rate Limit 風險，加快重複查詢速度
- 拒絕理由：v1.3 查詢頻率未知，過早引入快取層增加複雜性；快取 TTL 設定不當可能導致使用過期的黑名單資訊，業務風險高於效能收益

**替代方案 C：掛載到既有 `TronApiService` Facade**
- 拒絕理由：`TronApiService` 是向後相容用的 Facade，設計上凍結不擴張；新能力遵循 domain service 模式，由 caller 直接注入所需 service，不透過 Facade 轉發

## Open Points

- 後續應評估以 Redis TTL 快取查詢結果（建議 TTL 5 分鐘），以應對高頻查詢場景
- TRON_PRO_API_KEY 的取得與環境變數設定規範需確認（目前佔位符為 `'tronGridApiKey'`）

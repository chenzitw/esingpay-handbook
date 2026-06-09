
# KYT TRON 黑名單檢查 — Blueprint

## Context

本 Feature 屬於 v1.3 KYT 風控規則與流程。業務目的是讓 Fund domain 在處理出入金時，能向 TRON 主網查詢目標錢包地址是否已被 USDT (TRC20) 合約方（Tether Limited）凍結。

本 Feature 為單一 plan、循既有 `libs/services/tron` domain service pattern 落地，屬可略 blueprint 的訊號案例。此文件補後、作為業務決策備查。

## Scope

- **In scope**：查詢指定 TRON Base58 地址是否被 USDT TRC20 合約列入黑名單
- **Out of scope**：
  - 黑名單結果的快取（v1.3 不引入；後續視查詢頻率決策）
  - 其他鏈（ETH、BSC）的 USDT 黑名單查詢
  - 自動封鎖入金 / 拒絕提幣的業務規則（由 Fund domain 的 caller 自行決定）
  - 批次地址掃描
  - 黑名單異動事件監聽

## Landed Facts Assumed

- `libs/services/tron` 內已有完整的 dual-injection 模式（Interface + Symbol token + Concrete Class），以 `TronBalanceService` 為 canonical precedent
- `TronApiModule.forRoot()` 提供共用的 `TronWeb` 實例，`contractAddress` 由呼叫方透過環境變數注入
- USDT TRC20 合約（主網：`TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t`）公開了 `getBlackListStatus(address)` view function
- 現行主網不掛 API Key 時，TronGrid 限制 3 RPS，超量回傳 HTTP 429

## Critical Decisions

### 外部合約呼叫策略
採用 `tronweb` 的 `contract().at(contractAddress).getBlackListStatus(address).call()` 路徑，執行 read-only（constant）call，不廣播交易、不消耗能量。

### Dummy Default Address 初始化
`tronweb` 在呼叫 `contract().at()` 前必須有 `defaultAddress`，否則 SDK 內部拋出 `Invalid address provided`。決策：建構時若 `defaultAddress` 未設定，將 USDT 合約地址本身設定為 dummy。此 dummy 不影響 view call 的呼叫結果，因 `getBlackListStatus` 為純 constant query。

### API Key 注入策略
API Key 透過 `TronApiOptions.apiKey` 在 `TronApiModule.forRoot()` 設定時統一注入，各 domain service 不獨立管理 key，維持單一設定入口。

### 錯誤處理策略
所有異常（網路逾時、HTTP 429、地址格式錯誤）皆向上 re-throw，錯誤訊息包含原始 error message 以利 caller 決定業務行為（拒絕？重試？）。Service 本身不吸收、不靜默 fallback。

## Open Points

- TronGrid API Key 申請後應透過環境變數注入，目前預設值為 `'tronGridApiKey'`（佔位符），需在部署設定補齊
- 後續若查詢頻率高，需評估是否引入短效快取（e.g. TTL 5 分鐘）避免 Rate Limit 與增加延遲

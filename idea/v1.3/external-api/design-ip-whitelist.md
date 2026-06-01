# External API IP Whitelist 設計筆記

## 目的

IP whitelist 用來限制 external API 只接受來自商戶允許來源 IP 的 request，降低 API key 外洩後被任意來源濫用的風險。

## 設定層級

白名單先走 API key level。

商戶可以在後台設定 API key 對應的 allowed IPs。未來若支援多把 API key，不同 key 可以有不同白名單：

```text
merchant_api_keys
- id
- merchant_id
- scopes
- allowed_ips
```

例如：

- Key A：只查 deposits，來自報表系統 IP。
- Key B：建立 withdrawal intents，來自出金服務 IP。

## API Scope

TODO：api key scopes 如何設計。

初步可以先簡化為：只要使用有效 API key，即可呼叫所有 external API。

## Withdrawal Intent Create

高風險操作如 `POST /external/v1/withdrawal-intents` 是否強制啟用 IP whitelist，需要再決定。

可選方向：

- 未設定 whitelist 時禁止建立 withdrawal intent。
- 查詢 API 可先不強制 whitelist。

## Client IP 判斷

若未來 request 會經過 Cloudflare / LB / proxy，需要先確認 request 來自可信任 proxy，再讀取來源 IP header：

- `CF-Connecting-IP`
- `X-Forwarded-For`

不能直接信任 client 自帶的 forwarding header。

## IP 格式

第一版先支援 IPv4。

後續再評估：

- IPv6
- CIDR
- CIDR 範圍限制，避免過大的白名單範圍

## 商戶責任

若商戶 client IP 浮動造成 request 被擋，商戶需要處理固定出口 IP，或在未來 CIDR 支援後提供合理範圍。

## Logging

TODO：Log 要記錄完整，可先不做。

未來若要支援問題排查，至少需要能回答：

- request 使用哪把 API key
- 判定出的 client IP 是什麼
- 命中的 whitelist rule 是什麼
- 被拒絕的原因是什麼

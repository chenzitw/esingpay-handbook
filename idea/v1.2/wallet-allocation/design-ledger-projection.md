# Wallet Ledger Projection — Network Transaction 歸屬

> 本文是 `design.md` §5 (Output Layer：Ledger Item / Cashflow Item) 的延伸,
> 收斂 ledger projection 中 `networkTransactionId` 的歸屬 invariant。
>
> **範圍說明**:本輪僅定義 NT 歸屬規則。Projection 的其餘設計層內容
> (projection context、line source、grouping、read model 粒度)目前散落在
> `blueprint-stage-2.md`,屬層級錯位,待後續另案從 blueprint 上提至本 design 層。
> 本文不展開該重構。

## Network Transaction 歸屬 invariant

一筆 ledger row 是否攜帶 `networkTransactionId`,取決於它的 `source` / `destination`
是否構成一次**跨 wallet 的資金移動(cross-wallet movement)**。

- **跨 wallet movement → 掛 fund case 的 networkTransactionId**

  當 source 與 destination **不是同一個 internal wallet** 時——任一端為 external,
  或兩端皆 internal 但 `walletId` 不同——此 row 對應一筆鏈上資金移動,掛上 fund case
  snapshot 的 `networkTransactionId`。該值可能為 null,表示該 fund case 此時尚無
  鏈上交易(例:尚未 instruct)。

- **同 wallet 內 bucket reclassify → 不掛(null)**

  當 source 與 destination 為同一個 internal wallet、僅 bucket 不同時(例:service_fee
  的 available → held 截留),此 row 是 wallet 內部歸屬調整,不對應鏈上交易,
  `networkTransactionId` 恆為 null。

此規則是**純結構性 invariant**:僅依賴 endpoint 的 `space` / `walletId`,
不依賴 fund case type,亦不需為任何特定 fund case 開特例。

## 三種 fund case 對照

| Fund case  | kind        | source → destination               | cross-wallet?   | networkTransactionId |
| ---------- | ----------- | ---------------------------------- | --------------- | -------------------- |
| Deposit    | principal   | external → merchant                | 是(external 端) | 掛(origin NT)        |
| Deposit    | service_fee | merchant.available → merchant.held | 否(同 wallet)   | null                 |
| Withdrawal | principal   | sender → external                  | 是              | 掛(target NT)        |
| Withdrawal | service_fee | sender.available → sender.held     | 否              | null                 |
| Withdrawal | network_fee | feePayer → external                | 是              | 掛(target NT)        |
| Transfer   | principal   | sender → recipient                 | 是(不同 wallet) | 掛(target NT)        |
| Transfer   | network_fee | feePayer → external                | 是              | 掛(target NT)        |

## 設計意涵

此 invariant 取代「僅當有 external endpoint 才掛」的早期實作行為。

- 早期行為對 deposit / withdrawal **正確**:其 internal→internal pair 僅 service_fee
  reclassify,本就不該掛 NT。
- 早期行為對 **transfer principal 錯誤**:sender → recipient 是跨 wallet 移動但兩端皆
  internal(無 external 端),早期規則會漏掛 NT,使 principal ledger row 無法回溯鏈上交易。

改採 cross-wallet 表述後,三種 fund case 一致,deposit / withdrawal 行為零變化,
transfer principal 正確掛上 target NT,且不需 fund-case-specific 特例。

## 時序保證(transfer principal)

Transfer 的 principal ledger row 唯一寫入點是 settle 時的 `projectOutcome`。
NT 在 instruct 階段綁定,早於 settle,故 projectOutcome 寫 principal 時 NT 已就緒,
**不需回填**。唯一殘餘 null 情形是 instruct 失敗的 zero-outcome 路徑——該情形本就無
鏈上交易,principal 掛 null 為正確語意。

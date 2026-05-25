---
status: draft
updated_at: 2026-04-28
updated_by: Tim
remark: 本文件不作為規格內容，為 system-overview 之中文翻譯版本，且可能過時，僅供閱讀參考。
language: zh-TW
source: ./system-overview.md
---

# 系統總覽

## 1. 目的

本文定義 Fund、Network、Wallet 三個 domain 的高階系統模型。

核心邊界如下：

> **Fund 負責業務編排。Network 負責外部交易事實。Wallet 負責帳務真相。Ledger/Cashflow 讀模型負責面向使用者的呈現映射。**

本系統分離：

- **業務意圖** 與 **外部交易事實**
- **帳務真相** 與 **使用者可見的顯示真相**
- **Fund 編排** 與 **Wallet 執行**
- **posted 餘額效果** 與 **processing / failed / cancelled 顯示軌跡**

本總覽聚焦於 domain 責任、因果關係與標準流程行為。它刻意不涵蓋 table-level schema、DTO 欄位完整性與 projector 實作細節。

---

## 2. Domain 責任地圖

## 2.1 Fund

Fund 負責業務 aggregates，以及面向使用者／業務的生命週期決策。

範例：

- `Deposit`
- `WithdrawalIntent`
- `TransferIntent`

Fund 負責：

- 建立並驗證業務意圖
- 決定業務狀態轉換
- 協調 Wallet allocation 生命週期呼叫
- 協調 Network instruction 或 recognition
- 管理多步驟或多 leg 的業務工作流
- 處理業務失敗、拒絕、取消或重試語義

Fund **不**負責餘額 mutation。

Fund **不**直接寫入 Wallet 帳務事實。

---

## 2.2 Network

Network 負責外部交易事實。

範例：

- blockchain transaction
- gateway transaction
- bank transfer record
- provider-side transaction status

Network 負責：

- 識別外部 inbound 事實
- 下達 outbound network transaction 指令
- 追蹤 provider / network status
- 向 Fund 發出 milestone facts

Network **不**決定業務帳務意義。

Network **不**異動 Wallet 餘額。

---

## 2.3 Wallet

Wallet 負責帳務真相與餘額 invariants。

Wallet 負責：

- `WalletAllocation` 生命週期
- `WalletEntry` / `WalletEntryLine` 建立
- `WalletAssetBalance` mutation
- atomic balance check and mutation
- reservation / settlement / reversal 帳務事實
- 在 concurrency 下保護餘額 invariants

Wallet 除 allocation contract 之外，**不**負責業務工作流語義。

Wallet **不**在 allocation transaction 內直接決定使用者可見的顯示生命週期。

---

## 3. Wallet 模型層次

Wallet 包含多個明確分離的層次。這些層次不可合併。

## 3.1 WalletAllocation：allocation case record

`WalletAllocation` 是 Fund 驅動之帳務計畫在 Wallet 端的 allocation case record。

它記錄：

- flow family
- lifecycle status
- fund case reference
- allocation type
- plan lines（intended allocation snapshot）
- outcome lines（final result snapshot）
- reservation deadline（適用時）

Allocation 使用兩組 lines 描述完整 case lifecycle：

- **planLines**：在 reserve / propose time 提交的 intended allocation snapshot。它作為 settle-time validation 的 envelope boundary。direct-settle 時為 null，因為 direct-settle 是 born-realized，沒有 plan stage。
- **outcomeLines**：allocation 經由 settle 抵達 terminal state 時記錄的 final result snapshot。它包含 successful settlement、partial realized cost（例如 network fee 已消耗但 principal 失敗），或 full-amount settlement。released / expired allocations 為 null，projection 會從 absence of outcome 推導出「全部為零」。

它們形成三層資料鏈：

```text
planLines → outcomeLines → entryLines
```

- `planLines` 記錄 intended economic effects（plan / envelope）。不應包含 0 金額 placeholder line；某 kind 的 line 不存在，表示該 kind 沒有預期經濟效果。
- `outcomeLines` 記錄 case 的 final economic outcome
- `WalletEntryLine` 記錄從 outcome lines 推導而來的 accounting facts

`outcomeLines` 不是 `WalletEntryLine` 的重複資料。前者是 upstream outcome snapshot（case 最終解析成什麼結果）；後者是 downstream accounting effect（Wallet 實際 posted 的帳務事實，包含 bucket mapping 與 reversal splits）。

`WalletAllocation` 是用來建立帳務事實，並觸發下游 read-model sideflows 的協調物件。

---

## 3.2 WalletEntry / WalletEntryLine：帳務真相

`WalletEntry` 與 `WalletEntryLine` 是 posted-only 的帳務事實。

它們是唯一會直接驅動 `WalletAssetBalance` 的 Wallet records。

關鍵規則：

- `WalletEntry` 是 insert-once immutable。
- `WalletEntryLine` 是 insert-once immutable。
- Entry 建立後，會在同一 transaction 中立即影響 balance。
- 不存在 provisional entry status。
- 不存在 void entry status。
- 一般操作中不存在 entry update path。
- failed / cancelled / processing 等使用者可見軌跡，不以假 entry lines 表示。

`WalletEntryLine` 是 balance mutation、audit、fact ordering 與 balance rebuild 的真相來源。

---

## 3.3 WalletAssetBalance：操作型餘額投影

`WalletAssetBalance` 是 wallet asset 的操作型 materialized balance。

它由 posted `WalletEntryLine` records 驅動。

一般寫入路徑規則：

> `WalletAllocation`、`WalletEntry` / `WalletEntryLine` 與 `WalletAssetBalance` mutation 必須在同一 database transaction 中處理。

`lastAppliedEntryLineId` 是 reconciliation / rebuild watermark。它不是一般 command path 中的 eventual-consistency catch-up 機制。

Wallet 會對受影響的 balance rows 使用 ordered locking，以維持 atomic balance check and mutation。

---

## 3.4 WalletLedgerItem：使用者可見 ledger 明細

`WalletLedgerItem` 是使用者可見的 ledger detail read model。

它為使用者呈現 **business-aware bucket effects**：每一 row 以 wallet bucket movement 為錨點，但在相關時也可攜帶 external source/destination 或 network transaction reference 等業務脈絡。

它的基本 row 粒度是：

> 特定 allocation / fund case、wallet、bucket、currency、kind 下的一個 business-aware bucket effect。

單一 allocation 或 fund case 可能產生多筆 ledger rows。Ledger rows 不承諾與 `WalletEntryLine` 或 `WalletAllocationLine` 具有 1:1 對應。

它存在的原因是：使用者可見 ledger 不等同於內部 accounting ledger：

- 它呈現 `available` / `held` 的使用者可見 movement
- 它隱藏 `locked` / `held_locked` reservation transitions
- 它可以呈現 processing / failed / invalid rows
- 它可以表示不直接影響 balance 的使用者可見結果

`WalletLedgerItem` 是 ledger list / pagination 的來源。

它不是直接從 `WalletEntryLine` 映射而來。

它由 allocation lifecycle 與下游 sideflow projection 驅動。

Ledger row visibility 由 query/use-case policy 強制執行。例如，merchant users 只能看到 merchant-visible buckets，而 platform users 可以看到 platform treasury buckets 與 platform-owned held buckets。這個 visibility rule 不屬於 accounting fact model。

---

## 3.5 WalletCashflowItem：使用者可見資產流動摘要

`WalletCashflowItem` 是使用者可見的 cashflow read model。

它比較接近 bank passbook 概念，而不是 bucket ledger。

它依 fund case、allocation、wallet、currency 彙總 asset movement。

典型行為：

- principal 與同幣別 service fee 合併成一個 primary cashflow item
- 不同幣別的 network fee 成為一個獨立 network-fee cashflow item
- item 暴露 `direction`、`amountGross`、`amountNet`、`amountFee`
- 它不需要 generic signed `amountDelta`，因為 passbook 語義更適合用 direction 加 gross/net/fee breakdown 表示
- item 可獨立於 fund case status 而為 valid 或 invalid

在目前標準流程中，`WalletCashflowItem` 依 `(allocationId, walletId, currencyCode, category)` 分組。這是 current-flow invariant，不是對所有未來 transfer variants 的普遍宣告。

Cashflow items 主要跟隨 fund case 與 allocation，但在顯示或可追蹤性需要時，也可包含 compact network transaction context，包括 transaction identifier、source、destination。

Cashflow item 可使用 `primary` 或 `network_fee` 等 category：

- `primary`：principal 加上同幣別 service fee，折疊成一個 passbook item
- `network_fee`：不同幣別 network fee，以獨立 passbook item 表示

`WalletCashflowItem` 也是 post-commit sideflow resource，不屬於 Wallet Allocation transaction。

---

## 4. Allocation 流程族

Wallet Allocation 支援兩種 flow family。

## 4.1 Reservation flow

Reservation flow 用於 fund case 完全生效前，就必須先控制 internal funds 的場景。

典型案例：

- `WithdrawalIntent`

生命週期：

```text
reserve -> bind -> settle
        -> release
        -> expire
```

重要語義：

- `reserve` 會鎖定 internal funds。
- `reserve` 不建立使用者可見的 ledger/cashflow rows。
- `bind` 綁定真實 fund case identifier。
- user-facing sideflow 只會在 allocation bound 之後開始。
- `settle` 建立 posted settlement / reversal accounting facts。settle 用於 business 已完成其流程的情況，也包含 partial costs（例如 network fee）已消耗的 failure cases。
- `release` 主動終止一筆已知不該繼續的 allocation。release 只用於 business entity 從未成功建立的情況，例如 Fund 在 reserve 之後 persist business aggregate 失敗。
- `expire` 在 `reservedUntil` 之後，被動清理 unbound reserved allocations。

`reserved` allocations 沒有最終 fund case identity，且可能 expire。

`bound` allocations 具備真實 fund case identity，不會作為 ghost reservations expire。

---

## 4.2 Direct flow

Direct flow 用於 Wallet allocation 開始前，fund case 已存在，且不需要事前 reservation control 的場景。

典型案例：

- `Deposit`

存在兩種 variants。

### Direct pending

用於外部事實已知但尚未 final 的情況。

生命週期：

```text
propose -> settle
        -> release
```

語義：

- `propose` 建立 bound allocation。
- `propose` 不建立 `WalletEntry`。
- `propose` 可觸發 post-commit ledger/cashflow sideflow。
- `settle` 建立 posted accounting facts 並更新 balance。
- 若尚未發生 accounting effect，`release` 會在不建立 posted entry 的情況下，將 allocation 收尾為不再繼續。

### Direct immediate

用於外部事實已經 final 的情況。

生命週期：

```text
direct-settle
```

語義：

- allocation 建立即為 settled
- posted settlement entry 立即建立
- balance 在同一 transaction 中更新
- ledger/cashflow sideflow 在 commit 後觸發

---

## 5. 帳務事實模型

## 5.1 Entry phases

`WalletEntry` 使用 allocation phase 描述一筆 entry 的帳務角色。

典型 phases：

- `reservation`
- `settlement`
- `reversal`

`allocationPhase` 屬於 accounting fact。它不是使用者可見的 display status。

---

## 5.2 Reservation accounting

Reservation entries 將資金從 liquid buckets 移入 locked buckets。

範例：

- `available -> locked`
- `held -> held_locked`

Reservation entries 是 posted accounting facts，且會影響 balance。

它們是 internal accounting control facts。

它們通常會被使用者可見的 ledger/cashflow views 隱藏。

---

## 5.3 Settlement accounting

Settlement entries 表示最終有效的經濟效果。

範例：

- deposit credit into `available`
- service fee movement into `held`
- withdrawal principal leaving `available`
- network fee leaving fee payer wallet

Settlement entries 會立即影響 balance。

---

## 5.4 Reversal accounting

Reversal entries 在 funds 被釋放回來時，用來撤銷先前的 reservation control。

範例：

- `locked -> available`
- `held_locked -> held`

Reversal entries 是 posted accounting facts。

它們是 internal accounting facts，並且可依 display rules 決定是否從使用者可見 ledger views 中隱藏。

---

## 5.5 Same-entry self-funding

單一 posted entry 可以包含多條 lines，這些 lines 在同一 wallet/bucket/currency 內的 net effect 會一起計算。

驗證基於 per-entry net effect：

> 對每個 `(walletId, bucket, currencyCode)`，pre-existing balance 加上 net entry effect 不得為負。

這支援 display-driven accounting facts，例如：

```text
available +500
available -10
held +10
```

不需要建立假的中介 entries，也不需要內部執行順序。

---

## 6. 使用者可見 Read Model 原則

## 6.1 Post-commit sideflow 邊界

`WalletLedgerItem` 與 `WalletCashflowItem` 是 post-commit sideflow resources。

Wallet Allocation transaction 只處理：

- allocation state
- entry / entry lines
- asset balance

Ledger 與 cashflow read models 由下游 sideflow 更新。

Sideflow failure 不得 rollback Wallet Allocation transaction。

Sideflow projectors 必須 retryable 且 idempotent。

---

## 6.2 Row lifecycle

`WalletLedgerItem` 與 `WalletCashflowItem` 使用自己的 read-model row lifecycle。

每一 row 有一個 persisted state 軸線：

```text
state: active | superseded
```

`state` 描述該 row 是否為目前可見版本。

- `active`：包含於預設 user-facing list
- `superseded`：為 audit/debug 保留，但從預設 list 排除

一筆 row 只有在同 source 的新版 row 被建立並取代它時，才會成為 `superseded`。通常，plan-based rows 會在 outcome-based rows 被 projected 時變成 superseded。

Read model rows 會記錄 plan amounts 與 outcome amounts（若可用），作為相對穩定的 facts。它們不 persist `validity` 欄位。

**Validity 在 DTO layer 推導。** REST DTO mapper 會依 kind 比較 plan 與 outcome amounts 來計算 validity：

- outcome amount > 0 for a given kind → valid
- outcome amount = 0 或 outcome lines absent → invalid
- fund case failure 不代表所有 wallet-facing items 都 invalid；validity 是 per item / per kind

這個分離讓 read model rows 維持穩定，同時允許 UI presentation logic 自由演進。

---

## 6.3 Versioned row replacement

Read model rows 採 append-oriented。

當同 source 的新版 row 應取代舊的 visible row 時：

- old row 變成 `superseded`
- new row 變成 `active`
- new row 依自己的 ordering timestamp 出現
- old rows 保留作為 audit/debug

對目前標準流程而言，Ledger item source identity 可視為：

```text
allocationId + walletId + bucket + currencyCode + kind
```

對 Cashflow items 而言，source identity 可視為：

```text
allocationId + walletId + currencyCode + category
```

這些 identities 定義當同 source 的新版 row 被 project 出來時，哪一筆 active row 應被 supersede。

Read-model rows 上的 `updatedAt` 可用來追蹤 supersession time。

Default list ordering 可以使用自然遞增 row identity，通常是 `id DESC`，因為 read-model rows 是 append-oriented，而較新的 projected rows 應較晚出現／排在舊 rows 上方。

`effectiveAt` 仍應保留為所顯示效果的語義時間。對 processing rows 而言，它跟隨產生該 display version 的 allocation lifecycle time，例如 bind 或 propose time。對 final rows 而言，它跟隨產生該 final display version 的 allocation lifecycle time，例如 settle 或 release time。

Read-model row 的 business payload 建立後不可變。

---

## 6.4 與 fund case 和 allocation 的關係

Read-model rows 應保留足夠的關聯資訊，以支援業務顯示與未來 transfer 擴充。

至少包含：

- fund case type
- fund case identifier
- allocation id
- wallet id
- bucket
- currency code
- Ledger items 的 `kind`，或 Cashflow items 的 `category`

Ledger DTO / composed rows 也可攜帶 display context，例如 merchant / wallet display information、`space`、derived `ownerType`、bucket、source、destination，以及當 row 直接與外部 network effect 相關時的 compact network transaction reference。這些欄位屬於 user-facing ledger read model 或 composed DTO shape；本 overview 不要求它們全部直接儲存在 persistence row 上。

這支援未來 `TransferIntent` patterns：同一 fund case 可對應多個 allocations。

初始模型不需要獨立的 leg identifier。

---

## 7. 標準流程：Deposit

## 7.1 Deposit pending

場景：

- Network 識別 inbound transaction。
- Transaction 尚未 final。
- Fund 建立 processing/transacting 狀態的 `Deposit`。
- Fund 要求 Wallet `propose` direct allocation。

Wallet 行為：

- 建立 bound direct allocation
- 尚不建立 WalletEntry
- 觸發 post-commit ledger/cashflow sideflow

Read-model 呈現：

- ledger/cashflow rows 可以呈現 active valid expected movement
- balance 尚未變更
- WalletEntryLine 尚不存在

---

## 7.2 Deposit settled

場景：

- Network transaction 變成 final。
- Fund 將 deposit 標記為 completed。
- Fund 要求 Wallet settle allocation。

Wallet 行為：

- 建立 posted settlement entry
- 建立 entry lines
- 在同一 transaction 中更新 balance
- 觸發 post-commit ledger/cashflow sideflow

500 USDT deposit 且有 10 USDT service fee 的 accounting effect 範例：

```text
available +500
available -10
held +10
```

Read-model 呈現：

- 先前 plan-based expected rows 變成 superseded
- 新的 active outcome-based rows 被 append
- cashflow item 依 allocation、wallet、currency 彙總 deposit amount

---

## 7.3 Deposit failed

場景：

- Network transaction fails 或 rejected。
- Fund 將 deposit 標記為 failed。
- business process 已完成（帶有 failure result）。
- Fund 要求 Wallet 使用反映 failure 的 outcome lines settle allocation：所有 amounts 都是 zero。

Wallet 行為：

- 使用所有 amounts 都為 zero 的 outcomeLines settle allocation
- 不建立 posted entry（沒有 accounting effect，因為 propose 未 lock funds）
- balance 維持不變
- 觸發 post-commit ledger/cashflow sideflow

Read-model 呈現：

- 先前 plan-based expected rows 變成 superseded
- 可以 append 新的 active outcome-based rows，以呈現 failed expected movement（validity 在 DTO layer 推導）

---

## 8. 標準流程：Withdrawal Intent

## 8.1 Withdrawal reservation

場景：

- 使用者或系統建立 withdrawal intent request。
- Fund 必須在 fund case 生效前，先確保 funds 可以被 reserved。

Wallet 行為：

- `reserve` allocation
- 建立 posted reservation entry
- 在同一 transaction 中更新 balance
- 尚不建立 user-facing ledger/cashflow rows

原因：

- allocation 尚未綁定真實 fund case identifier
- locked bucket transitions 是 internal accounting control facts
- user-facing ledger 隱藏 locked / held_locked transitions

---

## 8.2 Withdrawal bind

場景：

- Fund 成功 persist `WithdrawalIntent`。
- Fund 將 allocation 綁定到真實 fund case identifier。

Wallet 行為：

- allocation 從 `reserved` 轉為 `bound`
- 不建立 WalletEntry
- 觸發 post-commit ledger/cashflow sideflow

Read-model 呈現：

- 可以 append active plan-based expected rows
- user-facing ledger 從 bind 開始，而不是 reserve

---

## 8.3 Withdrawal settled

場景：

- Network transaction succeeds。
- Fund 將 withdrawal finalize 為 completed。
- Fund 要求 Wallet settle allocation。

Wallet 行為：

- 建立 actual effects 的 posted settlement entry
- 若 reserved remainder 必須釋放，建立 posted reversal entry
- 在同一 transaction 中更新 balance
- 觸發 post-commit ledger/cashflow sideflow

Read-model 呈現：

- 先前 plan-based expected rows 變成 superseded
- active outcome-based final rows 被 append
- ledger principal row 在綁定外部 transfer 時，可包含 external destination 與 network transaction reference
- network-fee row 跟隨實際 fee payer wallet，且 fee 被 external network 消耗時可包含 network transaction reference
- cashflow rows 依 allocation、wallet、currency 彙總，且在顯示或可追蹤性有用時可包含 network transaction context

---

## 8.4 Withdrawal failed without consumed network fee

場景：

- Withdrawal 被 rejected、cancelled，或 network transaction 在任何 network fee 被消耗前失敗。
- business process 已完成（帶有 failure result）。
- Fund 要求 Wallet 使用反映 failure 的 outcome lines settle allocation：所有 amounts 都是 zero。

Wallet 行為：

- 使用 principal=0、service_fee=0、network_fee=0 的 outcomeLines settle allocation
- 建立 posted reversal entry，以 unlock 所有先前 reserved funds
- 在同一 transaction 中更新 balance
- 觸發 post-commit ledger/cashflow sideflow

Read-model 呈現：

- plan-based expected rows 變成 superseded
- 新的 active outcome-based rows 以 zero outcome amounts 被 append（validity 在 DTO layer 推導，所有 items 都 invalid）

---

## 8.5 Withdrawal failed with consumed network fee

這是標準 stress case。

場景：

- Withdrawal fund case 整體失敗。
- Principal transfer 未完成。
- Service fee 不應被保留。
- Network fee 已經在鏈上被消耗。
- business process 已完成（帶有包含 partial cost 的 failure result）。
- Fund 要求 Wallet 使用反映 partial result 的 outcome lines settle allocation。

Wallet 行為：

- 使用 principal=0、service_fee=0、network_fee=actual consumed amount 的 outcomeLines settle allocation
- 為 consumed network fee 建立 posted settlement entry
- 為 released remainder 建立 posted reversal entry（principal + service_fee locked amounts 全部 reversed，network_fee remainder reversed）
- 在同一 transaction 中更新 balance
- 觸發 post-commit ledger/cashflow sideflow

Read-model 呈現：

單一 failed fund case 可以產生混合 item validity（在 DTO layer 推導）：

```text
USDT principal: invalid (plan -980 → outcome 0)
USDT service_fee: invalid (plan -20 → outcome 0)
TRX network_fee: valid (plan -30 → outcome -25)
```

Principal 與 network-fee ledger rows 在直接關聯外部 network effects 時，可以 reference network transaction。Service-fee rows 通常不會，因為 service fee 是 internal allocation / bucket movement，而不是 external network effect。

這證明：

- item validity 是 per item / per kind，從 plan vs outcome comparison 推導
- fund case failure 不代表所有 wallet-facing rows 都 invalid
- network-fee rows 跟隨實際 economic effect，而不是只跟隨 fund-case-level status
- Ledger 與 Cashflow read models 遵循相同 validity derivation principle
- settle 是 completed business processes 的正確 terminal operation，包含帶有 partial costs 的 failures

---

## 9. 標準流程：Transfer Intent

Transfer intent 可能包含：

- 多個 source wallets 到一個 destination wallet
- 一個 source wallet 到多個 destination wallets

Wallet 不需要變成 multi-leg workflow engine。

Fund 負責 transfer orchestration。

建議模型：

- `TransferIntent` 負責 transfer legs / subtasks
- 每個 leg 可以建立自己的 WalletAllocation
- 一個 fund case 可對應多個 allocations
- Wallet 持續一次執行一個 allocation

如果某個 reserve 失敗，Fund 會透過 release 補償先前已 reserved 的 allocations。

Wallet 保持聚焦於單一 allocation execution unit。

Ledger/Cashflow read models 應保留 `fundCaseId + allocationId`，讓未來 transfer views 可以依 allocation 分組或區分，而不強迫 Wallet core 理解整個 transfer graph。

---

## 10. 設計邊界

## 10.1 Wallet 不負責業務編排

Wallet 不應知道 transfer、withdrawal approval process 或 deposit recognition process 的完整 workflow。

Wallet 執行 allocation lifecycle operations，並強制 accounting invariants。

Business workflow 屬於 Fund。

---

## 10.2 Entry 不負責使用者可見的顯示生命週期

`WalletEntry` 與 `WalletEntryLine` 是 accounting facts。

它們不表示：

- pending display state
- failed display state
- cancelled display state
- superseded display versions

這些屬於 read models。

---

## 10.3 Ledger/Cashflow 不負責帳務真相

`WalletLedgerItem` 與 `WalletCashflowItem` 是使用者可見的 read models。

它們可以顯示 invalid 或 failed items。

它們不驅動 balance。

它們不取代 `WalletEntryLine` 作為 accounting truth。

---

## 10.4 Allocation transaction 不包含 sideflow read models

Allocation unit of work 包含：

- allocation state mutation
- entry creation
- balance mutation

它不包含 ledger/cashflow read models 的 persistence。

Sideflows 在 commit 後觸發。

Sideflow failure 會分開 retry，不 rollback accounting truth。

---

## 10.5 Settle vs release vs expire 語義邊界

terminal operation 的選擇取決於 business process completion，而不是 success/failure：

- **settle**：business process 已完成，不論是成功、部分成功或失敗。Fund 提交反映 final result 的 outcome lines（包含 failed components 的 zero amounts）。settle 是任何已經跑完整個 lifecycle 的 allocation 的正確 terminal operation。
- **release**：business entity 從未成功建立。例如 Fund reserved funds 但 persist business aggregate 失敗。release 會撤銷 reservation，但不記錄 outcome。
- **expire**：allocation 從未 bound 到 fund case，且超過 reservation deadline。expire 是 passive cleanup mechanism。

這代表「failed withdrawal」仍然走 settle，而不是 release，因為 business process 已經執行並產生結果，即使該結果是「principal=0、network_fee=25」。

---

## 11. 摘要

本系統分離三種真相：

1. **Business truth**：由 Fund 負責
2. **External transaction truth**：由 Network 負責
3. **Accounting truth**：由 Wallet Entry / Entry Line 負責

User-facing truth 由 read models 表示：

- `WalletLedgerItem`：business-aware bucket-effect details
- `WalletCashflowItem`：asset-movement summaries

`WalletAllocation` 作為 allocation case record，連接 business intent 與 Wallet accounting execution，並同時追蹤 intended plan 與 final outcome。

Wallet Allocation 是 accounting facts 與 sideflow triggers 的 orchestration boundary，但它不會把 accounting truth 與 display truth 合併成同一個模型。

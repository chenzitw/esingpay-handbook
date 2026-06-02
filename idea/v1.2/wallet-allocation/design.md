---
status: draft
updated_at: 2026-04-28
updated_by: Tim
remark: 本文件未依照 design 規範撰寫，暫供內容參考，等待後續重構改寫。
---

# Wallet Allocation Design

## 1. 定位

`WalletAllocation` 是 Wallet domain 的**資金配置案例記錄（allocation case record）**。

它記錄一筆資金配置從計畫到結算的完整生命週期，包含配置計畫（planLines）、最終結果快照（outcomeLines）與狀態流轉。

### 四層語義模型

| 層級                    | 資源                              | 語義                                                                  |
| ----------------------- | --------------------------------- | --------------------------------------------------------------------- |
| **Case Record**         | `WalletAllocation`                | 資金配置案例（配置計畫 + 最終結果快照 + 生命週期）                    |
| **Accounting Fact**     | `WalletEntry` + `WalletEntryLine` | Insert-once immutable posted-only 帳務事實（影響 balance 的唯一來源） |
| **User-facing Detail**  | `WalletLedgerItem`                | 面向使用者的帳本明細（含 processing / failed / completed）            |
| **User-facing Summary** | `WalletCashflowItem`              | 面向使用者的業務彙總明細（一件業務 × 幣別）                           |

### 三層資料鏈

```text
planLines → outcomeLines → entryLines
```

| 層級                | 資源              | 語義                                                             |
| ------------------- | ----------------- | ---------------------------------------------------------------- |
| **Plan**            | `planLines`       | caller 在 reserve / propose / direct-settle 時提交的配置計畫     |
| **Outcome**         | `outcomeLines`    | allocation case 的最終結果快照（含成功結算、部分消耗、全部歸零） |
| **Accounting Fact** | `WalletEntryLine` | Wallet 依 outcomeLines 展開後產生的帳務事實                      |

`outcomeLines` 不是 `WalletEntryLine` 的重複——前者是 upstream outcome snapshot（case 的最終結果是什麼），後者是 downstream accounting effect（Wallet 展開了什麼帳務事實，含 bucket 映射、reversal 拆分等內部邏輯）。

### 核心分離原則

- `WalletAllocation` + `WalletAllocationLine`：描述配置計畫與最終結果快照
- `WalletEntry` + `WalletEntryLine`：描述帳務事實（insert-once immutable、posted-only，影響 balance）
- `WalletLedgerItem` / `WalletCashflowItem`：描述使用者可見的過程軌跡（含失敗、處理中、已完成）
- **帳務真相與顯示真相分離**：Entry 只記錄影響 balance 的事實；使用者可見但不影響 balance 的狀態留在 Ledger/Cashflow layer
- `WalletAllocation` 驅動三類輸出：entry（帳務事實）、ledger item（過程明細）、cashflow item（業務彙總）

### 實作範圍

**本次實作（Wallet Allocation feature）**：WalletAllocation、WalletEntry / WalletEntryLine、WalletAssetBalance、Fund 端 Deposit / WithdrawalIntent 流程銜接。

**未來另案（Projection scope）**：WalletLedgerItem、WalletCashflowItem。Projection context 原則、line source 規則、grouping 原則與 read model 粒度輪廓已初步收斂（見 §5），完整 schema 待定案。本次 allocation use cases 預留 sideflow hook，但不在同一 transaction 內執行 projection。

---

## 2. 空間維度：Space × Side

資產移動的完整座標系由兩個維度構成：

### Space（空間）— 資產所在的邏輯區域

| 值         | 說明                             | walletId |
| ---------- | -------------------------------- | -------- |
| `internal` | 系統內錢包空間                   | **必填** |
| `external` | 系統外空間（鏈上地址、外部銀行） | **null** |

### Side（定位）— 資產在事務中的角色

| 值            | 說明                     |
| ------------- | ------------------------ |
| `source`      | 事務的提供方（資產起點） |
| `destination` | 事務的接收方（資產終點） |

### 四象限窮舉

| Space × Side           | 範例                                       |
| ---------------------- | ------------------------------------------ |
| internal × source      | merchant wallet available USDT 出金        |
| internal × destination | merchant wallet held USDT 收取 service fee |
| external × source      | 鏈上入帳（deposit 來源）                   |
| external × destination | 鏈上出金（withdrawal 本金離開系統）        |

### 關鍵規則

- **External lines 不參與餘額檢查、不更新 WalletAssetBalance、不產生 WalletEntryLine**
- External lines 是 allocation 層級的意圖 metadata，用於審計完整性
- **Bucket 對外僅暴露 `available` / `held`**（Fund 視角）
- `locked` / `held_locked` 是 Wallet 內部的 two-phase 實作細節，不對外暴露

---

## 3. Aggregate 欄位

### WalletAllocation（Aggregate Root）

| 欄位                 | 型別                             | 說明                                                                                      |
| -------------------- | -------------------------------- | ----------------------------------------------------------------------------------------- |
| `id`                 | `bigint` (auto-increment)        | Wallet 內部主鍵                                                                           |
| `flow`               | `WalletAllocationFlow`           | `reservation` \| `direct`（immutable）                                                    |
| `status`             | `WalletAllocationStatus`         | `reserved` \| `bound` \| `settled` \| `released` \| `expired`                             |
| `type`               | `WalletAllocationType`           | `deposit` \| `withdrawal` \| `transfer` \| `adjustment`（caller 宣告，Wallet 不推斷）     |
| `fundCaseType`       | `FundCaseType`                   | fund case 類型（e.g. `WithdrawalIntent`）                                                 |
| `fundCaseIdentifier` | `bigint \| null`                 | fund case ID（reservation flow: reserve 時 null，bind 時回填；direct flow: 建立時即帶入） |
| `reservedUntil`      | `DateTime \| null`               | reservation 有效截止點（僅 reservation flow，進入 bound 後失去語義效力）                  |
| `planLines`          | `WalletAllocationLine[] \| null` | 配置計畫（reserve / propose / direct-settle 時 set-once）                                 |
| `outcomeLines`       | `WalletAllocationLine[] \| null` | 最終結果快照（settle / direct-settle 時 set-once）；released / expired 時為 null          |
| `createdAt`          | `DateTime`                       |                                                                                           |
| `updatedAt`          | `DateTime`                       |                                                                                           |

### planLines / outcomeLines Flow Invariant

| Flow / Status                    | planLines | outcomeLines |
| -------------------------------- | --------: | -----------: |
| reservation / reserved           |  required |         null |
| reservation / bound              |  required |         null |
| reservation / settled            |  required |     required |
| reservation / released           |  required |         null |
| reservation / expired            |  required |         null |
| direct (propose) / bound         |  required |         null |
| direct (propose) / settled       |  required |     required |
| direct (propose) / released      |  required |         null |
| direct (direct-settle) / settled |  required |     required |

### WalletAllocationLine（Embedded Value Object, 1:N）

Aggregate raw 中的 embedded child。不含 infra row identity。

| 欄位           | 型別                       | 說明                                                                  |
| -------------- | -------------------------- | --------------------------------------------------------------------- |
| `space`        | `WalletAllocationSpace`    | `internal` \| `external`                                              |
| `side`         | `WalletAllocationSide`     | `source` \| `destination`                                             |
| `walletId`     | `bigint \| null`           | internal 時必填，external 時 null                                     |
| `bucket`       | `WalletBucket \| null`     | internal 時必填（`available` \| `held`，Fund 視角）；external 時 null |
| `currencyCode` | `string`                   | 幣別                                                                  |
| `amount`       | `NumericString`            | 金額（非負數；zero outcome 場景可為 0）                               |
| `kind`         | `WalletAllocationLineKind` | `principal` \| `network_fee` \| `service_fee` \| `adjustment`         |

Plan lines 表達預期經濟效果，不應 emit 0 金額 placeholder line。若某 kind 在 planLines 中缺席，表示該 kind 沒有預期經濟效果；例如 withdrawal service fee 為 0 時，不建立 service_fee plan line，也不產生對應 ledger projection row。Outcome lines 可在失敗、釋放或部分實現場景使用 0 金額表達 terminal result。

注意：DB table row 仍有 surrogate `id`、`allocationId` FK、以及 `role` column，但這些僅存在於 persistence model 層，不進 aggregate raw。

### Persistence：role column

DB 使用同一張 `wallet_allocation_line` table 儲存 planLines 與 outcomeLines，以 `role` column 區分：

```ts
enum WalletAllocationLineRole {
  Plan = 'plan',
  Outcome = 'outcome',
}
```

Mapper 負責按 `role` filter 組裝為 aggregate raw 的 `planLines` / `outcomeLines`。

`role` 是 persistence-level discriminator，不進 aggregate raw。它描述的是「這筆 line 在 allocation case record 中的紀錄用途」——配置計畫 or 最終結果紀錄。

### 維度概念對照

| 概念     | 型別名                     | 所屬資源                      | 值域                                | 語義                  |
| -------- | -------------------------- | ----------------------------- | ----------------------------------- | --------------------- |
| 狀態     | `WalletAllocationStatus`   | Allocation                    | reserved / bound / settled / ...    | 生命週期位置          |
| 紀錄用途 | `WalletAllocationLineRole` | AllocationLine（persistence） | plan / outcome                      | 這組 lines 的紀錄用途 |
| 帳務作用 | `WalletAllocationPhase`    | Entry                         | reservation / settlement / reversal | 這筆 entry 的會計性質 |

---

## 4. WalletEntry：Posted-only Accounting Fact

### 設計原則

`WalletEntry` 是 **posted-only 的帳務事實紀錄**，是影響 `WalletAssetBalance` 的唯一來源。

- Entry 從建立那一刻起就是 posted，就影響 balance
- `WalletEntry` / `WalletEntryLine` 都是 insert-once immutable accounting fact
- **不存在 provisional 或 void lifecycle**——使用者可見的過程狀態（processing / failed）留在 `WalletLedgerItem`
- 一個 allocation × 一個 allocationPhase 可產生一筆或多筆 entry（例如 settle 時同時產生 settlement + reversal）
- Entry 不包含 external lines（external 不影響系統內餘額）
- Entry 不強制複式平衡（非總帳模型，line amountDelta 加總不要求為零）
- **Same-entry self-funding**：同一筆 entry 內，某條 line 的 debit 可由同一筆 entry 內另一條 line 的 credit 供給。餘額驗證以 per-entry net effect 為準

### WalletEntry 欄位

| 欄位              | 型別                      | 說明                                        |
| ----------------- | ------------------------- | ------------------------------------------- |
| `id`              | `bigint` (auto-increment) |                                             |
| `allocationId`    | `bigint`                  | FK → WalletAllocation                       |
| `allocationPhase` | `WalletAllocationPhase`   | `reservation` \| `settlement` \| `reversal` |
| `createdAt`       | `DateTime`                |                                             |

注意：

- 不再有 `status` 欄位。所有 entry 皆為 posted
- 不再有 `updatedAt` 欄位。Entry 為 insert-once immutable accounting fact，無 update path
- 不含 `allocationType` 欄位。過往為 user-facing ledger read model 而保留的 denormalized snapshot 已隨 `WalletLedgerItem` 獨立而移除；若未來 read path 明確需要再加回

### WalletEntryLine 欄位

| 欄位           | 型別                      | 說明                                                                                                        |
| -------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `id`           | `bigint` (auto-increment) | Watermark 引用、fact row ordering、raw line query pagination / audit                                        |
| `entryId`      | `bigint`                  | FK → WalletEntry                                                                                            |
| `walletId`     | `bigint`                  | 目標錢包（永遠 internal）                                                                                   |
| `merchantId`   | `Uuid \| null`            | **Fact-time denormalization**：由 `walletId` 導出，建 line 時同步寫入，寫入後不回頭追隨 wallet 關聯變化修正 |
| `bucket`       | `WalletBucket`            | `available` \| `locked` \| `held` \| `held_locked`（物理 bucket）                                           |
| `currencyCode` | `string`                  |                                                                                                             |
| `amountDelta`  | `NumericString`           | 變動量（有正負）                                                                                            |
| `kind`         | `WalletEntryLineKind`     | `reservation` \| `principal` \| `network_fee` \| `service_fee` \| `adjustment`                              |
| `createdAt`    | `DateTime`                |                                                                                                             |

注意：不再有 `updatedAt` 欄位。EntryLine 為 insert-once immutable fact atom，無 update path。

### AllocationLine vs EntryLine 對照

| 面向        | AllocationLine（planLines / outcomeLines）                                             | EntryLine                                                                                         |
| ----------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Bucket 值域 | `available` \| `held`（Fund 視角）；external 時 null                                   | `available` \| `locked` \| `held` \| `held_locked`（物理）                                        |
| Space       | 有（internal / external）                                                              | 無（永遠 internal）                                                                               |
| Side        | 有（source / destination）                                                             | 無（以 amountDelta 正負表達）                                                                     |
| 金額        | 非負數 + side 決定方向                                                                 | 有正負（amountDelta）                                                                             |
| Kind 值域   | `WalletAllocationLineKind`：`principal` / `service_fee` / `network_fee` / `adjustment` | `WalletEntryLineKind`：`reservation` / `principal` / `service_fee` / `network_fee` / `adjustment` |
| Kind 語義   | 業務歸因                                                                               | 帳務歸因（reservation family 統一用 `reservation`，settlement family 才拆分具體 kind）            |
| Identity    | Aggregate raw 中不含 id / FK / role（infra concern）                                   | 保留 id / entryId / merchantId（ledger fact unit）                                                |
| 角色區分    | 由 persistence `role` column 區分為 planLines / outcomeLines                           | 由 `WalletEntry.allocationPhase` 區分帳務作用                                                     |

### Repository / Schema Implication

- `WalletEntry` raw 為獨立資源，不強制內嵌 `lines`
- `WalletEntryLine` 是獨立的重要帳務個體；service implementation 另外以 `WalletEntryPack { entry, lines }` / `WalletEntryPackDraft` 承接聚合 create/read
- `WalletEntryRepository` 收斂為 **create / read only**
- 一般情況下不存在 `WalletEntry update(...)` path；entry / line 建立即 posted，建立即影響 balance

---

## 5. Output Layer：Ledger Item / Cashflow Item

> **本 section 描述的資源不在本次實作範圍。** 本次不定義 WalletLedgerItem / WalletCashflowItem 的 raw、table、repository 細節。但 projection context 原則、line source 規則、grouping 原則與 read model 粒度輪廓已初步收斂，作為未來實作的設計約束。

> **NetworkTransaction 歸屬**：ledger row 的 `networkTransactionId` 歸屬 invariant（cross-wallet movement 才掛）獨立收斂於 [design-ledger-projection.md](./design-ledger-projection.md)。

### WalletLedgerItem（獨立資源，未來實作）

面向使用者的帳本明細。由 `WalletAllocation` 操作驅動建立與更新。

- 呈現使用者可見的過程軌跡：processing / completed / failed / cancelled
- 它才是 ledger list / pagination source，不是直接由 `WalletEntryLine` 分頁
- **不影響 balance**
- 可呈現失敗項目（人工拒絕、network transaction failed）
- 獨立資源，有自己的生命週期
- 支援 pagination / sorting / querying

### WalletCashflowItem（獨立資源，未來實作）

面向使用者的業務彙總明細。以「一件業務 × 幣別」為粒度。由 `WalletAllocation` 驅動存滅。

- 它是獨立的 cashflow read model，不是 `WalletEntry` / `WalletEntryLine` 本體

### 驅動規則

| 輸出層                            | 驅動時機                              | 影響 balance | 本次實作        |
| --------------------------------- | ------------------------------------- | ------------ | --------------- |
| `WalletEntry` / `WalletEntryLine` | 僅在產生 posted 帳務事實時            | **是**       | **是**          |
| `WalletLedgerItem`                | allocation 進入使用者可見狀態後的操作 | 否           | 否（預留 hook） |
| `WalletCashflowItem`              | allocation 進入使用者可見狀態後的操作 | 否           | 否（預留 hook） |

**Sideflow 觸發門檻**：進入 `bound`，或 direct-settle born `settled` 的 allocation，才驅動使用者可見的 sideflow。純 `reserved → expired` 的 ghost cleanup 不產生 ledger / cashflow item——使用者從未看過這筆 allocation。

### Projection Service

本次預留 `AllocationProjectionService` interface，定義兩個 projection 關鍵點，與 `WalletAllocationLineRole`（plan / outcome）對稱：

| Method           | 觸發操作                                       | Read-model 效果                                                                    |
| ---------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------- |
| `projectPlan`    | bind / propose                                 | 建立 plan-based LedgerItem（processing）+ CashflowItem                             |
| `projectOutcome` | settle / direct-settle / release（from bound） | 比較 planLines vs outcomeLines，supersede plan-based rows，建立 outcome-based rows |

- `projectPlan`：allocation 進入使用者可見的活躍狀態時觸發。讀 planLines 建立 processing rows
- `projectOutcome`：allocation 到達終態時觸發。讀 planLines + outcomeLines 比較差異，推導每筆 ledger item 的最終結算金額：
  - **正常 settle**：outcomeLines ≤ planLines，建立 outcome-based rows，supersede plan-based rows
  - **Withdrawal failed（network fee consumed）**：outcomeLines 中 principal=0, service_fee=0, network_fee=25 → plan vs outcome 比較後推導出哪些 kind 有實際結算
  - **Release（from bound）**：outcomeLines = null → 語義上全部歸零，等價於 outcome 全為 0
  - **Direct-settle**：planLines 與 outcomeLines 在同 call 內同時 set-once；因未經 projectPlan，無既有 plan-based rows 需 supersede，直接用 outcomeLines 建立 completed rows
- expire 和 reserve 不觸發 projection（使用者未見）
- release from reserved（未曾 bound）不觸發 projection（使用者未見）
- Phase 4 先註冊 no-op implementation
- Projection hook 在 `TransactionRunner.run()` 完成後由 use-case 直接呼叫，不需特殊 post-commit callback 機制

### Transaction 邊界

- **Wallet Allocation Tx / unit-of-work 只處理 allocation + entry + balance**
- WalletLedgerItem / WalletCashflowItem 屬於 **post-commit sideflow**，不在同一 transaction 內
- 不可把 WalletLedgerItem / WalletCashflowItem mutation 放進 allocation transaction unit-of-work
- projector / sideflow handler failure 不可回滾 Wallet Allocation Tx；後續 sideflow handler 應可 retry / idempotent
- hook trigger 必須在 allocation unit-of-work commit 之後才執行
- 無論未來走 queue / outbox / post-commit orchestrator，此約束不可違反

### Projection Context 原則

Projection 使用明確的 function name（`projectPlan` / `projectOutcome`）作為入口，與 `WalletAllocationLineRole`（plan / outcome）對稱，不需要 generic transition metadata。

最小 context shape：

```ts
type WalletAllocationProjectionContext = {
  allocation: WalletAllocation; // 含 planLines / outcomeLines / status 等完整 case record
  fundCase: WalletProjectionFundCase;
  effectiveAt: Date;
};
```

設計決策：

- **不帶 `trigger` / `operation`**：已由 function name 表達
- **不帶 `toStatus`**：已存在於 `allocation.status`（projection 在 status transition 完成後呼叫）
- **不帶 `fromStatus`**：目前同步 post-commit sideflow 不需要。若未來改為 outbox / event-based replay pipeline，再把 `fromStatus` / `toStatus` 放進 event envelope
- **不包含 `WalletEntryPack`**——projector 用 allocation lines（planLines / outcomeLines）推導使用者面向的 read model，不穿透到 accounting layer
- `fundCase` 攜帶 fund case 的最小展示欄位（狀態、金額、fee、source/destination 等），由 use-case 在呼叫 projection hook 前組裝。Projection service 不自行查詢 Fund aggregate
- Network transaction 資訊屬於 fund case snapshot 的一部分，不獨立於 context 頂層

### Projection Line Source 規則

| Hook             | 讀取的 lines                 | 推導邏輯                                                |
| ---------------- | ---------------------------- | ------------------------------------------------------- |
| `projectPlan`    | `planLines`                  | 直接用 planLines 建立 processing rows                   |
| `projectOutcome` | `planLines` + `outcomeLines` | 比較 plan vs outcome 差異，推導每筆 kind 的最終結算金額 |

- `projectPlan`：planLines 是唯一 source（此時 outcomeLines 尚不存在）
- `projectOutcome`：兩組 lines 都參與。Per kind 比較 plan 金額與 outcome 金額，推導出哪些 kind 有實際結算、金額是否變動。Release 時 outcomeLines = null，語義等價於全部歸零

Stress case 驗證——withdrawal failed, network fee consumed：

```
projectPlan 建立的 processing ledger items：
  principal:   plan -980
  service_fee: plan -20
  network_fee: plan -30

projectOutcome 比較後：
  principal:   plan -980 → outcome 0
  service_fee: plan -20  → outcome 0
  network_fee: plan -30  → outcome -25
```

Ledger item 記錄 plan 金額與 outcome 金額作為相對恆定的事實。Validity 推導（哪些 kind 有效、哪些失效）留給 REST DTO 轉換層——DTO mapper 根據 plan / outcome 金額差異推斷 validity，套用 UI 期望的呈現邏輯。未來 DTO 格式可自由變換，不影響 ledger item read model。

### Projection Grouping 原則

Allocation lines 維持攤平（flat half-edges）。Source-destination pairing / semantic grouping 是 `AllocationProjectionService` 的責任，不回滲到 allocation line shape。

- Projection 按 `allocation.type + flow + trigger` 選擇 grouping strategy
- `kind` 是 primary semantic discriminator，但不假設 `kind` alone 是 globally sufficient grouping key——strategy 可視需求納入 `currencyCode`、`walletId`、`bucket` 等維度
- 不同 allocation type 可以有不同的 grouping strategy（withdrawal 的 grouping 邏輯和 deposit 不同）
- 若未來 strategy-based deterministic grouping 不足以區分同一 allocation 內的 lines，優先在 allocation line 上擴充 `groupKey?: string`，而非 `pairId`。`groupKey` 支援 1:N / N:1 / N:N 的 semantic leg 分組，不預設 edge graph

### Read Model 粒度輪廓（preliminary）

> 以下為初步粒度方向，待 LedgerItem / CashflowItem 完整 schema 定案時確認。

**WalletLedgerItem 粒度方向**：

基本粒度為 `allocationId + walletId + bucket + currencyCode + kind`。它是 business-aware bucket effect row，不是 entry line proxy。Bucket 使用 Fund-facing 值域（available / held），不暴露物理 bucket（locked / held_locked）。

Ledger item 記錄相對恆定的事實：plan 金額、outcome 金額（nullable）、state。不記錄 validity——validity 是 UI 呈現關注點，由 REST DTO 轉換層根據 plan / outcome 金額推導。

**WalletCashflowItem 粒度方向**：

基本粒度為 `allocationId + walletId + currencyCode + category`。其中 `category` 是 strategy-derived 的業務分類（初步為 `primary` / `network_fee`）。`primary` 可合併 principal + same-currency service fee；`network_fee` 獨立一列。

**State 管理**：

Read model row 使用 `state` 維度：

| 值域         | 語義                                                              |
| ------------ | ----------------------------------------------------------------- |
| `active`     | 當前有效版本                                                      |
| `superseded` | 已被同源新版 row 取代（plan-based row 被 outcome-based row 取代） |

- `projectPlan` 建立 `active` rows
- `projectOutcome` 建立新的 `active` rows，同時將先前 plan-based rows 標為 `superseded`

**Validity 不在 read model 層**：

業務結果是否有效（例如 withdrawal failed 但 network_fee 仍有效）由 REST DTO 轉換層推導。DTO mapper 根據 plan / outcome 金額差異推斷 validity，套用 UI 期望的呈現邏輯。這樣 read model 保持穩定，UI 呈現邏輯可自由演進。

---

## 6. 狀態機

```
  Reservation family:

  ┌──────────┐   bind()   ┌──────────┐  settle()  ┌──────────┐
  │ reserved ├───────────►│  bound   ├───────────►│ settled  │
  └──┬───┬───┘            └────┬─────┘            └──────────┘
     │   │                     │
     │   │ release()           │ release()  ┌──────────┐
     │   ├─────────────────────┼───────────►│ released │
     │   │                     │            └──────────┘
     │   │  expire()   ┌──────────┐
     │   └────────────►│ expired  │
     │                 └──────────┘
     └─── reserved: unbound, expirable
          bound: fund-case-bound, non-expirable

  Direct flow — propose (born bound):

  ┌──────────┐  settle()  ┌──────────┐
  │  bound   ├───────────►│ settled  │
  │ (born)   │            └──────────┘
  └────┬─────┘
       │ release()  ┌──────────┐
       └───────────►│ released │
                    └──────────┘

  Direct flow — direct-settle (born settled):

  ┌──────────┐
  │ settled  │  (born settled)
  └──────────┘
```

### 合法轉換

| Flow                     | 狀態             | 可轉換至                         |
| ------------------------ | ---------------- | -------------------------------- |
| `reservation`            | `reserved`       | `bound` / `released` / `expired` |
| `reservation`            | `bound`          | `settled` / `released`           |
| `direct` (propose)       | `bound` (born)   | `settled` / `released`           |
| `direct` (direct-settle) | `settled` (born) | （terminal）                     |

- `settled`、`released`、`expired` 皆為 terminal state
- `reserved` → `expired`：僅限 TTL 逾時，由 scheduler 觸發（ghost cleanup）
- `reserved` → `released`：Fund 主動放棄（已知失敗的顯式補償）
- `bound` → `released`：業務主動取消 / pending deposit 失敗
- `bound` 不可 → `expired`：已綁定真實 fund case，不受 ghost cleanup 管轄
- `reservation` flow 必須設定 `reservedUntil`
- `direct` flow 的 `reservedUntil` 為 null

### Release vs Expire 語義邊界

| 面向        | `release`             | `expire`                 |
| ----------- | --------------------- | ------------------------ |
| 觸發方      | Fund 顯式呼叫         | TTL scheduler 被動觸發   |
| 適用 status | `reserved` 或 `bound` | 僅 `reserved`            |
| 語義        | 已知不需要，主動放棄  | 逾時未綁定，判定為 ghost |
| 定位        | 主要補償路徑          | Fallback cleanup         |

---

## 7. 操作與 Entry 產生

### 7.1 reserve()

- 建立 `WalletAllocation`（flow=reservation, status=reserved）
- 持久化 `planLines`（role=plan）
- 僅對 **internal source** lines 執行餘額檢查與鎖定
- 產生 `WalletEntry`（allocationPhase=reservation）：
  - EntryLine：internal source 的 available→locked（或 held→held_locked）
  - EntryLine kind 統一為 `reservation`
- 同步更新 `WalletAssetBalance`
- **Atomic**：所有 internal source lines 的餘額檢查與鎖定必須 all-or-nothing
- **Insufficient balance**：caller-actionable business error（Either），fail-fast first，攜帶 `(walletId, currencyCode, required, available)`。Caller 可調整金額或補錢後重試
- 回傳 `walletAllocationId`
- 🔌 Sideflow hook：無（fundCaseIdentifier 尚未存在）

### 7.2 bind()

- 前提：`status = reserved`
- 更新 `WalletAllocation`：status → `bound`，回填 `fundCaseIdentifier`
- **不產生 WalletEntry**（無 balance 變動）
- 重複 bind（allocation 已非 reserved）一律回傳錯誤
- 典型時序：
  1. Fund 呼叫 reserve() → 取得 walletAllocationId
  2. Fund persist WithdrawalIntent → 取得 withdrawalIntentId
  3. Fund 呼叫 bind(walletAllocationId, fundCaseIdentifier=withdrawalIntentId)
  4. 若步驟 2 失敗 → Fund 呼叫 release() 或等待 TTL expire
- 🔌 Sideflow hook：`projectPlan`（未來驅動 LedgerItem / CashflowItem 建立）

### 7.3 settle()（reservation flow）

- 前提：`status = bound`
- **Single-shot settlement**：settle 為終結性操作，一次完成
- 更新 `WalletAllocation`：status → `settled`
- Fund 提交 outcomeLines
- 持久化 `outcomeLines`（role=outcome，set-once）
- **Envelope Principle**：outcomeLines 不得超過 planLines 的邊界（internal + external 都參與 envelope check）；超出則 fail loudly（allocation 維持 `bound`，不寫入 outcomeLines）
- 在同一事務中：
  1. 產生 `WalletEntry`（allocationPhase=settlement）：最終帳務事實
     - EntryLine kind 使用具體業務 kind（principal / service_fee / network_fee / adjustment）
     - **Reservation flow settle 的 entry bucket 映射**：internal source 走 locked mapping（`available → locked`、`held → held_locked`）；internal destination 走原 bucket
  2. 若 settle 金額 < reserve 金額，自動產生 `WalletEntry`（allocationPhase=reversal）：反轉剩餘鎖定額度
     - Reversal 精細粒度：per `(walletId, currencyCode, lockedBucket)` 計算差額
     - `locked` 差額 → `locked -delta, available +delta`；`held_locked` 差額 → `held_locked -delta, held +delta`
     - EntryLine kind = `reservation`
  3. 同步更新 `WalletAssetBalance`
- `allocationPhase` 描述 entry 在 allocation lifecycle 中所對應的帳務作用
- **Net-effect validation 失敗為 invariant violation**：envelope check 已保證 outcome 各 signature ≤ plan，settlement 從 `locked` 出帳數學上必夠；validation 失敗 → locked 餘額被未追蹤途徑改動，throw 而非 Either
- 🔌 Sideflow hook：`projectOutcome`（未來驅動 LedgerItem → completed、CashflowItem 更新）

### 7.4 propose()（direct flow, born bound）

- 適用於：discovered 但尚未 final 的 deposit
- 建立 `WalletAllocation`（flow=direct, status=**bound**，born bound）
- 同時帶入 `fundCaseIdentifier`（deposit aggregate 在呼叫前已存在）
- 持久化 `planLines`（role=plan）
- **不產生 WalletEntry**（尚未確認，不影響 balance）——propose 不鎖定資金
- 後續走 settle()（確認完成）或 release()（確認失敗）
- 🔌 Sideflow hook：`projectPlan`（未來驅動 LedgerItem / CashflowItem 建立，status=processing）

### 7.5 settle()（direct flow, proposed → settled）

- 前提：`status = bound`（由 propose 建立）
- 更新 `WalletAllocation`：status → `settled`
- Fund 提交 outcomeLines
- 持久化 `outcomeLines`（role=outcome，set-once）
- 若 outcomeLines 非全 zero，產生 `WalletEntry`（allocationPhase=settlement）：最終帳務事實
  - 支援 same-entry self-funding（例如 deposit gross credit + fee debit）
  - **Direct flow settle 的 entry bucket 映射**：所有 internal lines 走原 bucket（不經 locked），因為 propose 不鎖定資金
- 若 outcomeLines 非全 zero，同步更新 `WalletAssetBalance`
- Envelope Principle 同樣適用（outcomeLines 不得超出 planLines 邊界，internal + external 都參與）
- **Net-effect validation 對所有 flow 統一執行，不放寬**
- **Net-effect validation 失敗為 invariant violation**：outcomeLines 受 envelope 鎖死（不可超過 propose 時的 planLines），planLines 為 propose 已定不可變更——caller 無法調整 outcomeLines / planLines；唯一恢復路徑是 release 此 allocation 後重新 propose，已是業務決策非 retry 行為，無 caller-actionable space。throw 而非 Either
- **Zero outcome 例外（direct flow only）**：若 outcomeLines 全為 zero，allocation 仍 transition 至 `settled`、outcomeLines 仍寫入；但**不產生** WalletEntry、不更新 WalletAssetBalance。Reservation flow 不適用此例外（仍走 reversal pack 釋放 locked 資金）
- 🔌 Sideflow hook：`projectOutcome`（未來驅動 LedgerItem → completed）

### 7.6 direct-settle()（direct flow, born settled）

- 適用於：discovered 時已 final 的 deposit
- 建立 `WalletAllocation`（flow=direct, status=**settled**，born settled）
- 同時帶入 `fundCaseIdentifier`（已知）
- 持久化 `planLines` 與 `outcomeLines`（同 call 內同時 set-once）
- 若 outcomeLines 非全 zero，產生 `WalletEntry`（allocationPhase=settlement）
  - Entry bucket 映射：與 propose→settle 共用 direct flow 映射（直接原 bucket，不經 locked）
- 若 outcomeLines 非全 zero，同步更新 `WalletAssetBalance`
- 不經過 propose 階段
- **Direct-settle 不做 envelope 比較**：planLines 與 outcomeLines 在同 call 內同時 set，envelope 由 caller 在同 call 內保證（Wallet 不複驗）
- **Insufficient balance**：caller-actionable business error（Either），fail-fast first。Caller 可調整 planLines / outcomeLines / fee 後重新提交
- **Zero outcome 例外**：若 outcomeLines 全為 zero（典型情境：failed deposit），allocation 仍 born `settled`、planLines + outcomeLines 仍寫入；但**不產生** WalletEntry、不更新 WalletAssetBalance。planLines 仍寫入是業務需求——declared amount 對 ledger / audit / reconciliation 仍有實質價值（PM 視角：「我們預期 X 但失敗了」）
- 🔌 Sideflow hook：`projectOutcome`（未來驅動 LedgerItem / CashflowItem 建立，status=completed）

### 7.7 release()

- 前提：`status = reserved` 或 `status = bound`
- 更新 `WalletAllocation`：status → `released`
- 若有 locked 資金（reservation flow），產生 `WalletEntry`（allocationPhase=reversal）：
  - 鎖定的完整反轉（locked→available / held_locked→held）
  - EntryLine kind = `reservation`
  - 同步更新 `WalletAssetBalance`
- 若無 locked 資金（direct flow, proposed），不產生 entry（無 balance 變動）
- 語義統一：「已知不該繼續，主動收尾」
- **Reversal 流程 net-effect validation 失敗為 invariant violation**：reversal 是把 locked 退回 available（增加 available、減少 locked），理論上不會 fail；若 fail 表示 locked 被未追蹤途徑改動，throw 而非 Either
- 🔌 Sideflow hook：若曾 bound → `projectOutcome`（未來驅動 LedgerItem → cancelled/failed）；若從 reserved 直接 release → 無（使用者未見）

### 7.8 expire()

- 前提：`status = reserved` 且 `reservedUntil` 已過（**僅限 reserved，不適用 bound**）
- 由 TTL scheduler 觸發（scheduler = pure trigger，邏輯在 use case）
- use-case 自主查詢需要 expire 的 allocations，逐筆 expire
- 產生 `WalletEntry`（allocationPhase=reversal）：反轉鎖定
- 同步更新 `WalletAssetBalance`
- 定位：ghost reservation 的 fallback cleanup
- 反轉原因追溯：`Entry.allocationId → Allocation.status`（released vs expired）
- **Reversal 流程 net-effect validation 失敗為 invariant violation**：同 release 推理，throw 而非 Either
- 🔌 Sideflow hook：無（ghost reservation 對使用者不可見）

---

## 8. Domain Invariants

1. **餘額驗證（per-entry net-effect）**：任何 entry 產生時，計算該 entry 所有 lines 的 net effect per (walletId, bucket, currencyCode)。若任何組合的 net effect 為負，則 pre-existing balance + net effect 必須 ≥ 0。支援 same-entry self-funding。**對所有 flow 統一執行，不放寬**
2. **Atomic all-or-nothing**：跨 wallet 的 lines 必須在同一 atomic 操作中完成
3. **Flow immutability**：建立後不可變更
4. **Status transition legality**：僅允許上述狀態機定義的轉換路徑
5. **Fund Case 完整性**：bind 時必須提供 fundCaseIdentifier；reserve 時必須為 null；settle 的前提為 `status = bound`
6. **ReservedUntil 一致性**：reservation flow 必須有 reservedUntil；direct flow 必須為 null。進入 bound 後失去語義效力
7. **Expire 限定**：expire 僅適用於 `status = reserved` 且 `reservedUntil` 已過
8. **重複操作一律回傳錯誤**：bind / settle / release / expire 對不符合前提 status 的 allocation 一律回傳錯誤（use-case 層以 Either 封裝）
9. **Envelope Principle（hard cap, kind-inclusive）**：settle 時以 `(side, space, walletId?, bucket?, currencyCode, kind)` 為 signature 分組，比較 outcomeLines vs planLines 的 amount 總和，**internal + external lines 都參與**。超出則 reject，allocation 維持 `bound`，不寫入 outcomeLines。Direct-settle（born settled）不做 envelope 比較；planLines 與 outcomeLines 在同 call 內同時 set，envelope 由 caller 在同 call 內保證（Wallet 不複驗）
10. **Single-shot settlement**：settle 為終結性操作，不支援多次 settle；settle 同時自動反轉剩餘保留額度
11. **Concurrency control**：Phase 4 先使用 `AllocationMutexService`（基於 KeyedMutexService 的 in-memory mutex，全域單一 key `'wallet-allocation'`）確保 allocation 操作序列化。**Mutex outer, transaction inner**。目標設計為 per `(walletId, currencyCode)` ordered locking，待吞吐量需求出現後再升級
12. **Space-walletId 一致性**：space=internal 時 walletId 必填；space=external 時 walletId 必須為 null
13. **Space-bucket 一致性**：space=internal 時 bucket 必填（`available` \| `held`）；space=external 時 bucket 必須為 null
14. **External exclusion**：external lines 不參與餘額檢查、不更新 balance、不產生 WalletEntryLine
15. **Kind 值域分離**：AllocationLine.kind 不含 `reservation`；EntryLine.kind 含 `reservation`，僅用於 reservation / reversal allocationPhase
16. **Reserve retry policy**：reserve 重送直接新建新 allocation，舊的自然 expired；接受短期雙重圈存
17. **Asset 前提**：凡 allocation lines 涉及 internal `(walletId, currencyCode)`，對應的 `WalletAsset` 必須已存在。不存在即 reject
18. **Balance 前提**：`WalletAssetBalance` 是 `WalletAsset` 的 mandatory companion row。BalanceMutationService 不做 auto-create
19. **Watermark row-scoped**：`WalletAssetBalance.lastAppliedEntryLineId`（nullable）按個別 balance row 推進至該 row 本次實際被套用的 entry lines 之 max(id)
20. **Entry posted-only**：WalletEntry 不存在 `status` 欄位，也不存在 provisional / void lifecycle。所有 entry 建立即 posted，建立即影響 balance
21. **Entry builder flow-aware mapping**：EntryBuilderService 根據 `allocation.flow` 區分映射規則——reservation flow settle 的 internal source 走 locked mapping；direct flow settle 的所有 internal lines 走原 bucket（不經 locked）。Direct-settle 與 propose→settle 共用 direct flow mapping
22. **Reversal 精細粒度**：per `(walletId, currencyCode, lockedBucket)` 計算差額。`locked ↔ available`、`held_locked ↔ held`，不交叉
23. **planLines immutability**：planLines 為 create-time set-once。Reserve / propose / direct-settle 時一次寫入，之後不可修改、不可覆蓋
24. **outcomeLines set-once**：outcomeLines 在 allocation 到達終態時寫入一次，不可覆蓋。Settle 時 outcomeLines 的寫入、entries 的建立（若有）、balance mutation（若有）、status → settled 必須在同一 transaction 中原子完成。Release from bound 時 outcomeLines 為 null（全部歸零語義由 projection 推導，不顯式寫入空 lines）
25. **planLines / outcomeLines flow consistency**：reserve / propose / direct-settle 必須有 planLines（non-null）。Direct-settle 時 planLines 與 outcomeLines 在同 call 內同時 set-once。outcomeLines 僅出現在 settled status；released / expired 的 allocation 其 outcomeLines 為 null

---

## 9. API Input Shape

API input 原則上使用 `lines` 作為欄位名稱。Caller 不需要知道 persistence `role` 概念——`role` 是 Wallet 內部的 discriminator，由操作語義自動決定。Direct-settle 因同 call 同時帶 plan 與 outcome，是例外：input 需明確區分 `planLines` / `outcomeLines`。

- **reserve / propose**：request 中的 `lines` 將作為 `planLines`（role=plan）持久化
- **settle**：request 中的 `lines` 將作為 `outcomeLines`（role=outcome）持久化
- **direct-settle**：request 中的 `planLines`（role=plan）與 `outcomeLines`（role=outcome）會在同 call 內同時持久化

### Reserve Request

```ts
interface ReserveRequest {
  fundCaseType: FundCaseType;
  type: WalletAllocationType;
  reservedUntil: DateTime;
  lines: AllocationLineInput[];
}

interface AllocationLineInput {
  space: 'internal' | 'external';
  side: 'source' | 'destination';
  walletId?: bigint; // internal 時必填，external 時 null
  bucket?: 'available' | 'held'; // internal 時必填（Fund 視角），external 時 null
  currencyCode: string;
  amount: NumericString; // 非負數；zero outcome 場景可為 0
  kind: WalletAllocationLineKind;
}
```

### Bind Request

```ts
interface BindRequest {
  walletAllocationId: bigint;
  fundCaseIdentifier: bigint;
}
```

### Propose Request（direct flow, born bound）

```ts
interface ProposeRequest {
  fundCaseType: FundCaseType;
  fundCaseIdentifier: bigint;
  type: WalletAllocationType;
  lines: AllocationLineInput[];
}
```

### Settle Request

```ts
interface SettleRequest {
  walletAllocationId: bigint;
  lines: AllocationLineInput[];
}
```

### Direct-Settle Request（direct flow, born settled）

```ts
interface DirectSettleRequest {
  fundCaseType: FundCaseType;
  fundCaseIdentifier: bigint;
  type: WalletAllocationType;
  planLines: AllocationLineInput[];
  outcomeLines: AllocationLineInput[];
}
```

### Release Request

```ts
interface ReleaseRequest {
  walletAllocationId: bigint;
}
```

---

## 10. 命名策略（已收斂）

### 資源命名

| 層級                 | 名稱                   | 說明                                                   |
| -------------------- | ---------------------- | ------------------------------------------------------ |
| Case Record          | `WalletAllocation`     | 資金配置案例記錄（配置計畫 + 最終結果快照 + 生命週期） |
| Case Record Line     | `WalletAllocationLine` | 配置明細（含 space/side；由 role 區分 plan / outcome） |
| Accounting Fact      | `WalletEntry`          | Insert-once immutable posted-only 帳務事實             |
| Accounting Fact Line | `WalletEntryLine`      | 資產變動明細（永遠 internal）                          |
| User Detail          | `WalletLedgerItem`     | 使用者帳本明細（含 processing / failed）               |
| User Summary         | `WalletCashflowItem`   | 使用者業務彙總明細                                     |

### Fund Case 命名

- 內核模型：`fundCaseType` / `fundCaseIdentifier`
- 查詢 DTO：`fundCase { type, identifier, subject }`
- `FundCaseType` enum 來源：`@esingpay/contract-base`
- Legacy：既有 `correlation: {}`、`subject: {}` 定義可未來安排遷移重構

### API 欄位命名

- Request body 原則上使用 `lines`（即 `AllocationLineInput[]`）；direct-settle 使用 `planLines` + `outcomeLines`
- Caller 不需要知道 persistence `role`；direct-settle caller 需明確區分 plan / outcome shape
- 操作語義自動決定 persistence role：reserve/propose → plan；settle → outcome；direct-settle → plan + outcome

### Raw 與 Service-layer Schema 分層規則

`contract-base` 的 raw 維持資源級別（每個資源獨立 raw）。需要 parent + children 聚合輸入輸出時，由 service implementation 另外定義 pack / composed schema（例如 `WalletEntryPack`、`WalletEntryLineComposed`），不回滲到 base raw。

具體而言：

- `WalletAllocation` raw 內嵌 `planLines: WalletAllocationLine[] | null` + `outcomeLines: WalletAllocationLine[] | null`（allocation line 是 embedded value object）
- `WalletEntry` / `WalletEntryLine` raw 各自獨立（entry raw 不強制內嵌 lines；entry line 是 ledger fact unit）
- Service 層使用 `WalletEntryPack { entry, lines }` 作為聚合操作的 input/output schema

---

## 11. Canonical Paths

### 11.1 Withdrawal Intent（reservation family, platform 代付 network fee）

業務條件：gross=1000 USDT, principal=980 USDT, service_fee=20 USDT（進 merchant held），network_fee 預留 30 TRX（platform 代付），實際消耗 25 TRX。

#### Path 1：Success（reserved → bound → settled）

**Step 1. Fund → Wallet.reserve()**

Allocation planLines：

| side        | space    | wallet       | bucket    | currency | amount | kind        |
| ----------- | -------- | ------------ | --------- | -------- | ------ | ----------- |
| source      | internal | merchant     | available | USDT     | 980    | principal   |
| source      | internal | merchant     | available | USDT     | 20     | service_fee |
| source      | internal | **platform** | available | TRX      | 30     | network_fee |
| destination | external | —            | —         | USDT     | 980    | principal   |
| destination | internal | merchant     | held      | USDT     | 20     | service_fee |
| destination | external | —            | —         | TRX      | 30     | network_fee |

**Step 2. Wallet reserve side effects**

Allocation：status=reserved, fundCaseIdentifier=null

Entry（allocationPhase=reservation, all lines kind=`reservation`）：

| wallet   | bucket    | currency | amountDelta |
| -------- | --------- | -------- | ----------- |
| merchant | available | USDT     | -1000       |
| merchant | locked    | USDT     | +1000       |
| platform | available | TRX      | -30         |
| platform | locked    | TRX      | +30         |

Balance 同步更新。不產生 ledger item。

**Step 3. Fund persist WithdrawalIntent → bind()**

Allocation：reserved → bound，回填 fundCaseIdentifier。

不產生 entry。🔌 Sideflow hook: `projectPlan`。

**Step 4. Fund → Wallet.settle(outcomeLines)**

Settlement lines（network fee 僅 25 TRX）：same structure, TRX amounts = 25。

Envelope check：按 `(side, space, walletId?, bucket?, currencyCode, kind)` signature 分組，outcomeLines vs planLines ✓（含 external lines）。

**Step 5. Wallet settle side effects**

Entry 1（allocationPhase=settlement）：

| wallet   | bucket | currency | amountDelta | kind        |
| -------- | ------ | -------- | ----------- | ----------- |
| merchant | locked | USDT     | -980        | principal   |
| merchant | locked | USDT     | -20         | service_fee |
| merchant | held   | USDT     | +20         | service_fee |
| platform | locked | TRX      | -25         | network_fee |

注意：reservation flow settle 的 internal source 走 locked mapping（`available → locked`）。

Entry 2（allocationPhase=reversal, kind=`reservation`）：

| wallet   | bucket    | currency | amountDelta |
| -------- | --------- | -------- | ----------- |
| platform | locked    | TRX      | -5          |
| platform | available | TRX      | +5          |

Reversal 按 `(platform, TRX, locked)` 精細粒度計算：原始鎖定 30 - 實際消耗 25 = 差額 5。

Allocation：bound → settled。Balance 同步更新。🔌 Sideflow hook: `projectOutcome`。

**Final balance effect**：merchant USDT available -1000, held +20；platform TRX available -25。

---

#### Path 2：Persist Fail（reserved → released）

前提：reserve 成功，但 Fund persist WithdrawalIntent 失敗。

**Step 1. Fund 呼叫 release(allocationId)**

Allocation：reserved → released

**Step 2. Wallet release side effects**

Entry（allocationPhase=reversal, kind=`reservation`）：反轉全部鎖定。Balance 同步更新。不產生 ledger item（未曾 bind，使用者未見）。

---

#### Path 3：Over-envelope Settle（bound, settle rejected）

前提：allocation 已 bound，settle outcomeLines 中 network fee = 35 TRX > planLines 中 reserved 30 TRX。

Wallet：直接回 error（Either error side）。不產生 entry，不寫入 outcomeLines，allocation 維持 `bound`。

Envelope check 涵蓋 internal + external lines。

---

#### Variant：Merchant 自付 network fee

唯一差異：network_fee source 從 platform 改為 merchant。

---

### 11.2 Deposit Canonical Paths（direct family）

業務條件：inbound deposit, gross=500 USDT, service_fee=2%=10 USDT, net=490 USDT。

#### Path 4：Direct-pending Success（propose → settle）

**Step 1. Fund 建立 deposit(transacting) → Wallet.propose()**

Allocation planLines：

| side        | space    | wallet   | bucket    | currency | amount | kind        |
| ----------- | -------- | -------- | --------- | -------- | ------ | ----------- |
| source      | external | —        | —         | USDT     | 500    | principal   |
| destination | internal | merchant | available | USDT     | 500    | principal   |
| source      | internal | merchant | available | USDT     | 10     | service_fee |
| destination | internal | merchant | held      | USDT     | 10     | service_fee |

Allocation born bound（fundCaseIdentifier = depositId），flow=direct。

**Step 2. Wallet propose side effects**

不產生 entry（尚未確認，不影響 balance——propose 不鎖定資金）。🔌 Sideflow hook: `projectPlan`。

**Step 3. Network transaction settled → Fund → Wallet.settle(outcomeLines)**

Persist outcomeLines（role=outcome）。

Entry（allocationPhase=settlement, same-entry self-funding）：

| wallet   | bucket    | currency | amountDelta | kind        |
| -------- | --------- | -------- | ----------- | ----------- |
| merchant | available | USDT     | +500        | principal   |
| merchant | available | USDT     | -10         | service_fee |
| merchant | held      | USDT     | +10         | service_fee |

注意：direct flow settle 走原 bucket（不經 locked）。

Net effect check：merchant available net = +490 ≥ 0 ✓

Allocation：bound → settled。Balance 同步更新。🔌 Sideflow hook: `projectOutcome`。

**Final balance effect**：merchant USDT available +490, held +10。

---

#### Path 5：Direct-pending Failure（propose → release）

前提：propose 成功，但 network transaction 最終失敗。

Allocation：bound → released。不產生 entry（無 locked 資金）。🔌 Sideflow hook: `projectOutcome`。

---

#### Path 6：Direct-immediate（direct-settle）

Allocation born settled。planLines 與 outcomeLines 在同 call 內同時 set-once。若 outcomeLines 非全 zero，直接產生 settlement entry + balance update（走 direct flow mapping，原 bucket）；若 outcomeLines 全為 zero，仍持久化 planLines + outcomeLines，但不產生 entry、不更新 balance。不做 envelope 比較（same-call 由 caller 保證）。🔌 Sideflow hook: `projectOutcome`。

---

## 12. Operation Verb Summary

| 動詞          | flow                 | 前提 status       | 目標 status    | planLines | outcomeLines | Entry                                                             | Projection hook                | Envelope |
| ------------- | -------------------- | ----------------- | -------------- | --------- | ------------ | ----------------------------------------------------------------- | ------------------------------ | -------- |
| reserve       | reservation          | —                 | reserved       | create    | —            | reservation                                                       | 無                             | N/A      |
| bind          | reservation          | reserved          | bound          | —         | —            | 不產生                                                            | `projectPlan`                  | N/A      |
| propose       | direct               | —                 | bound (born)   | create    | —            | 不產生                                                            | `projectPlan`                  | N/A      |
| settle        | reservation / direct | bound             | settled        | —         | create       | settlement + reversal（if remainder）；direct zero outcome 不產生 | `projectOutcome`               | ✓        |
| direct-settle | direct               | —                 | settled (born) | create    | create       | settlement（non-zero outcome）                                    | `projectOutcome`               | N/A      |
| release       | 全部                 | reserved 或 bound | released       | —         | —            | reversal（若有 locked）                                           | `projectOutcome`（僅曾 bound） | N/A      |
| expire        | reservation          | reserved          | expired        | —         | —            | reversal                                                          | 無                             | N/A      |

---

## 13. Open Points（待後續討論）

1. **`direct-settle` 命名**：暫用，待進一步優化。Phase 4 use-case impl plan 已沿用此命名（`DirectSettleUseCase`），命名收斂前不阻擋實作
2. **WalletLedgerItem 完整 schema**：粒度方向與 state/validity model 已初步收斂（見 §5 Read Model 粒度輪廓），完整欄位定義待定案
3. **WalletCashflowItem 完整 schema**：粒度方向與 category model 已初步收斂（見 §5 Read Model 粒度輪廓），完整欄位定義待定案

### 已解決

| 原 Open Point                         | 決議                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mutex 範圍                            | Phase 4 先用 AllocationMutexService 全域 key `'wallet-allocation'`；mutex outer, tx inner；目標設計為 per (walletId, currencyCode) ordered locking，待吞吐量需求出現後升級                                                                                                                                                                                                                                  |
| Settle 與 Reserve 對應性              | Envelope Principle——不需一一對應，但不得超出保留邊界                                                                                                                                                                                                                                                                                                                                                        |
| Partial settle                        | 不支援多次 settle；金額差異透過 single-shot settlement + 自動反轉處理                                                                                                                                                                                                                                                                                                                                       |
| Lines grouping 策略                   | Fund 提交扁平 allocation lines。Wallet 依 operation semantics 產生必要 entries（reserve→reservation, settle→settlement, remainder→reversal），不做 effect-family decomposition                                                                                                                                                                                                                              |
| Entry 與 Wallet 的綁定                | Entry 升格為 allocationPhase 原子事件，不綁定單一 wallet                                                                                                                                                                                                                                                                                                                                                    |
| External 是否產生 entry line          | 不產生——external lines 是 allocation 層級的意圖 metadata                                                                                                                                                                                                                                                                                                                                                    |
| Source/Destination 結構               | 扁平化——以 `side` 欄位區分                                                                                                                                                                                                                                                                                                                                                                                  |
| Bucket 對外暴露範圍                   | Fund 視角僅 `available` / `held`；`locked` / `held_locked` 為 Wallet 內部                                                                                                                                                                                                                                                                                                                                   |
| movementItems 命名                    | 統一為 `WalletAllocationLine` / `lines`                                                                                                                                                                                                                                                                                                                                                                     |
| Bound status                          | `reserved`（unbound, expirable）→ `bound`（fund-case-bound, non-expirable）                                                                                                                                                                                                                                                                                                                                 |
| Release vs Expire 語義                | release = 顯式補償；expire = ghost cleanup                                                                                                                                                                                                                                                                                                                                                                  |
| Reservation entry line kind           | 統一為 `reservation`；具體業務 kind 僅用於 settlement family                                                                                                                                                                                                                                                                                                                                                |
| Remainder release phase               | `reversal`——phase 描述會計性質                                                                                                                                                                                                                                                                                                                                                                              |
| Reserve retry policy                  | 新建新 allocation，舊的自然 expired                                                                                                                                                                                                                                                                                                                                                                         |
| Envelope policy                       | Hard cap；超出 = error；internal + external 都參與 envelope check                                                                                                                                                                                                                                                                                                                                           |
| Over-envelope settle                  | Error（Either error side），allocation 維持 `bound`                                                                                                                                                                                                                                                                                                                                                         |
| Deposit family                        | Direct flow；pending 用 propose，immediate 用 direct-settle                                                                                                                                                                                                                                                                                                                                                 |
| Deposit failure path                  | propose → release；業務失敗語義在 deposit aggregate                                                                                                                                                                                                                                                                                                                                                         |
| Same-entry self-funding               | 採用；per-entry net-effect validation                                                                                                                                                                                                                                                                                                                                                                       |
| Execution planner / effect family     | 不採用；改以 self-funding 取代                                                                                                                                                                                                                                                                                                                                                                              |
| `project` 命名                        | 改為 `propose`                                                                                                                                                                                                                                                                                                                                                                                              |
| 冪等策略                              | 不符合前提 status 一律回傳錯誤（Either）；requestKey 已移除（Phase 4 前）                                                                                                                                                                                                                                                                                                                                   |
| Envelope 驗證粒度                     | Kind-inclusive signature grouping；internal + external 都參與                                                                                                                                                                                                                                                                                                                                               |
| `expiresAt` → `reservedUntil`         | 僅對 reserved 有效                                                                                                                                                                                                                                                                                                                                                                                          |
| `entryType` → `WalletAllocation.type` | Canonical field 在 allocation。Entry 曾短暫保留 `allocationType` 作為 denormalized snapshot，後因 `WalletLedgerItem` 抽離為獨立 read model 而移除；未來 read path 有明確需求再加回                                                                                                                                                                                                                          |
| `phase` → `allocationPhase`           | 描述 entry 在 allocation lifecycle 中的帳務作用                                                                                                                                                                                                                                                                                                                                                             |
| Entry provisional/void                | **已移除**——Entry 收斂為 posted-only immutable fact；使用者可見的過程狀態留在 `WalletLedgerItem`                                                                                                                                                                                                                                                                                                            |
| WalletLedgerItem 定位                 | 獨立資源，由 allocation 驅動；它才是 ledger list / pagination source，不由 entry line 映射                                                                                                                                                                                                                                                                                                                  |
| WalletCashflowItem 定位               | 獨立資源，由 allocation 驅動的 cashflow read model                                                                                                                                                                                                                                                                                                                                                          |
| 金額型別                              | amount 用 NumericString / Decimal / DB decimal(32,16)；id 用 bigint                                                                                                                                                                                                                                                                                                                                         |
| Watermark nullable                    | `lastAppliedEntryLineId` 允許 null                                                                                                                                                                                                                                                                                                                                                                          |
| Asset/Balance 前提                    | asset 不存在 = 業務錯；balance 不存在 = 系統錯                                                                                                                                                                                                                                                                                                                                                              |
| requestKey                            | **已移除**——Phase 4 前移除欄位，避免誤用。未來有冪等需求時另案加回                                                                                                                                                                                                                                                                                                                                          |
| External line bucket                  | external 時 bucket 為 null（非 required）；新增 space-bucket 一致性 invariant                                                                                                                                                                                                                                                                                                                               |
| Error flow                            | use-case boundary 以 Either 封裝 caller-actionable business error；判定原則為「caller 有無可採取的合理恢復動作」。System invariant violation（例如 balance row 不存在、reversal 流程 balance 不足、reserve 後 wallet/asset 消失）一律 throw 不入 Either。詳見各 use-case 對應 §7.x 與 plan Appendix Error 分類原則                                                                                          |
| Projection service                    | `AllocationProjectionService` 定義兩個關鍵點：`projectPlan` / `projectOutcome`；Phase 4 為 no-op impl；hook 在 `TransactionRunner.run()` 後由 use-case 直接呼叫                                                                                                                                                                                                                                             |
| Entry builder flow-aware mapping      | Reservation flow settle: internal source 走 locked mapping；Direct flow settle: 直接原 bucket。Builder 按 `allocation.flow` 區分。Direct-settle 與 propose→settle 共用 direct flow mapping                                                                                                                                                                                                                  |
| Reversal 精細粒度                     | per `(walletId, currencyCode, lockedBucket)` 計算差額；`locked ↔ available`、`held_locked ↔ held`                                                                                                                                                                                                                                                                                                           |
| Direct flow propose 不鎖定            | propose 不產生 reservation entry、不鎖定資金。settle 時直接從原 bucket 扣                                                                                                                                                                                                                                                                                                                                   |
| Net-effect validation scope           | 對所有 flow 統一執行，不放寬                                                                                                                                                                                                                                                                                                                                                                                |
| Direct-settle envelope                | Direct-settle（born settled）不做 envelope 比較；planLines 與 outcomeLines 在同 call 內同時 set，envelope 由 caller 保證，Wallet 不複驗                                                                                                                                                                                                                                                                     |
| Insufficient balance                  | 依 caller actionable space 判定：reserve / direct-settle 為 caller-actionable business error（Either），fail-fast first，攜帶 `(walletId, currencyCode, required, available)`；settle / release / expire 為 invariant violation（envelope-bounded 或 reversal-bounded，caller 無 actionable space），throw                                                                                                  |
| Expire use-case 粒度                  | use-case 自主查詢 + 逐筆 expire（per-allocation single）                                                                                                                                                                                                                                                                                                                                                    |
| planLines / outcomeLines 設計         | `WalletAllocation` 從 create-time intent 提升為 allocation case record。`lines` 拆為 `planLines`（reserve / propose / direct-settle 時 set-once）+ `outcomeLines`（終態時 set-once）。DB 同一張 `wallet_allocation_line` table，以 `role` column（`WalletAllocationLineRole`：`plan` / `outcome`）區分。Direct-settle 時 planLines 與 outcomeLines 在同 call 內同時 set-once。API input 由操作語義決定 role |

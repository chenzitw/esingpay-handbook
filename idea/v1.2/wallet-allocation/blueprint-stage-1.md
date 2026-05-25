---
status: draft
updated_at: 2026-04-28
updated_by: Tim
remark: 本文件未依照 blueprint 規範撰寫，暫供內容參考，等待後續重構改寫。
---

# Wallet Allocation Blueprint Stage 1

本文件基於 `./design.md` spec，為 AI agent 提供實作指引。

---

## 1. 實作範圍總覽

本次實作目標：**建立 Wallet Allocation feature**——allocation-driven ledger core，以支撐 deposit 與 withdrawal intent 業務流程。

### 現有基礎

- **Wallet 本體**：已完整實作（entity + model + repository + migration），不在本次範圍內
- **WalletAsset / WalletAssetBalance / WalletEntry / WalletEntryLine**：僅有 raw（domain type）定義，無 entity / model / repository / migration
- **WalletAllocation / WalletAllocationLine**：全新資源，從零開始

### 本次交付物

1. **WalletAsset**：model + repository + migration（raw 已存在，不另建 entity）
2. **WalletAssetBalance**：model + repository + migration（含 4-bucket balance + watermark；raw 已存在，不另建 entity）
3. **WalletEntry（posted-only）**：model + repository + migration（immutable fact，不需 entity）
4. **WalletEntryLine**：model + repository + migration（ledger fact unit）
5. **WalletAllocation**：entity + model + repository + migration（DraftableEntityBase）
6. **WalletAllocationLine**：embedded model + migration（含 space / side / role；不獨立設 repository）
7. **Allocation use cases**：reserve / bind / propose / settle / direct-settle / release / expire
8. **Balance mutation service**：per-entry net-effect validation + mutex-based concurrency control + 同步更新
9. **Entry builder service**：Fund-bucket → physical-bucket mapping + entry/line construction（pure computation，flow-aware）
10. **Allocation projection service**：post-commit sideflow hook interface + no-op implementation
11. **RPC contract**：`walletAllocationRpc`，供 Fund service 呼叫
12. **Fund service 整合**：withdrawal intent / deposit 的 allocation 呼叫

### 明確不在本次範圍

- **WalletLedgerItem**：使用者帳本明細 read model，亦是未來 ledger list / pagination source。Projection context 原則、line source 規則、grouping 原則與 read model 粒度輪廓已初步收斂（見 design spec §5），完整 schema 待定案
- **WalletCashflowItem**：獨立 cashflow read model，同上
- 兩者的 raw schema 本次不定義
- 本次 allocation use cases 預留 sideflow hook / projection extension point，但不實作 projection 邏輯
- Ledger / Cashflow mutation 不進 allocation transaction unit of work

### 命名邊界

- **`walletAllocationRpc`**：操作 allocation lifecycle / entry generation / balance mutation
- **`walletLedger...`**：保留給未來 WalletLedgerItem 的 read model / list & pagination 範疇
- **Wallet Allocation**：作為整個 feature umbrella name，涵蓋 allocation + entry/lines + balance

---

## 2. 建議實作順序

### Phase 1：基礎資源落地（WalletAsset + WalletAssetBalance）

目標：讓 Wallet 具備資產與餘額的持久化能力。

工作項：

1. **WalletAsset model + repository**
   - 參考 `Wallet_Asset_Design` 的 WalletAsset raw
   - 直接以 contract-base raw 作為 repository public boundary，不另外建立 entity
   - 欄位：id, merchantId, walletId, currencyCode, nickname, createdAt, updatedAt
   - API identity 為 `walletId + currencyCode`（surrogate id 內部使用）

2. **WalletAssetBalance model + repository**
   - 參考 `Wallet_Asset_Design` 的 WalletAssetBalance raw
   - 直接以 contract-base raw 作為 repository public boundary，不另外建立 entity
   - 欄位：id, walletId, currencyCode, availableAmount, lockedAmount, heldAmount, heldLockedAmount, lastAppliedEntryLineId, computedAt, createdAt, updatedAt
   - 金額欄位型別：`NumericString`（raw / persistence）/ `Decimal`（domain 計算時）
   - DB decimal 精度：`precision: 32, scale: 16`
   - `lastAppliedEntryLineId`：**nullable**（新建 balance 時為 null，表示尚無 watermark）
   - key 為 `(walletId, currencyCode)`

3. **DB migration**
   - 新建 wallet_asset 表
   - 新建 wallet_asset_balance 表
   - wallet_asset_balance 需 unique index on `(walletId, currencyCode)`

### Phase 2：帳務事實層（WalletEntry + WalletEntryLine）

目標：實作 posted-only 的 WalletEntry 設計。

工作項：

1. **WalletEntry raw（contract-base）**
   - 獨立 raw，不內嵌 lines
   - 欄位：id, allocationId, allocationPhase, createdAt
   - allocationPhase：`reservation | settlement | reversal`（`WalletAllocationPhase`）
   - **不再有 status 欄位**——所有 entry 建立即 posted，建立即影響 balance
   - **不再有 updatedAt 欄位**——Entry 為 insert-once immutable accounting fact
   - **不含 allocationType**——過往為 user-facing ledger read model 保留的 denormalized snapshot，已隨 `WalletLedgerItem` 獨立而移除；若未來 read path 明確需要再加回

2. **WalletEntryLine raw（contract-base）**
   - 獨立 raw，作為重要帳務個體存在（ledger fact unit）
   - **Insert-once immutable**：所有欄位在寫入後不可修改
   - 必要欄位（不可省略）：id, entryId, walletId, merchantId, bucket, currencyCode, amountDelta, kind, createdAt
   - **不含 updatedAt**——EntryLine 為 insert-once immutable fact atom
   - `id`：保留——watermark (`lastAppliedEntryLineId`) 直接引用、fact row ordering、raw line query pagination / sorting / audit
   - `entryId`：保留——line 需要被獨立查詢時的 join key
   - `merchantId`：**fact-time denormalization**——由 walletId 導出，建 line 時同步寫入，寫入後不回頭追隨 wallet 關聯變化修正。避免 list / search / composed query 退化成過多 join（admin query / audit trace 必需）
   - bucket：`available | locked | held | held_locked`（物理 bucket，4 值）
   - kind：`reservation | principal | service_fee | network_fee | adjustment`（`WalletEntryLineKind`）
   - amountDelta：`NumericString`（有正負）
   - walletId 需要 index

3. **Aggregate schema（service implementation 層）**
   - `WalletEntryPack`：aggregate read output — `{ entry: WalletEntry; lines: WalletEntryLine[] }`
   - `WalletEntryPackDraft`：aggregate create input — `{ entry: Omit<WalletEntry, 'id'>; lines: WalletEntryLineDraft[] }`
   - `WalletEntryLineDraft`：`Omit<WalletEntryLine, 'id' | 'entryId'>`
   - 放在 service-local `schema/`

4. **WalletEntry construction helper（非 aggregate entity）**
   - Entry 是 immutable fact——建立後不可修改。因此 **不需要 DraftableEntityBase**，也不需要 dedicated aggregate entity / transition methods
   - 可簡化為 factory function 或輕量 helper，用來組出 `WalletEntryPackDraft`
   - 不再有 void / post transition

5. **WalletEntryRepository**（create / read only）
   - `createPackInTx(packDraft, tx)`：aggregate create，一次寫入 entry + lines
   - `getPackById(id)`：回傳完整 `WalletEntryPack`
   - `getPacksByAllocationId(allocationId)`：回傳所有 phase 的 packs
   - **無 update method**——entry / line 皆為 insert-once immutable，無一般 mutable aggregate repository 的 update path

6. **DB migration**
   - 新建 wallet_entry 表（含 allocation_id FK, allocation_phase 欄位，**無 status、無 updated_at、無 allocation_type**）
   - 新建 wallet_entry_line 表（含 wallet_id, merchant_id, bucket, amount_delta, created_at，**無 updated_at**）
   - wallet_entry_line.wallet_id 需要 index
   - wallet_entry_line.merchant_id 需要 index

### Phase 3：案例記錄層（WalletAllocation + WalletAllocationLine）

目標：實作 WalletAllocation aggregate（allocation case record）及其完整狀態機。

工作項：

1. **WalletAllocation entity**
   - 參考 spec Section 3
   - 欄位：id, flow, status, type, fundCaseType, fundCaseIdentifier, reservedUntil, planLines, outcomeLines, createdAt, updatedAt
   - flow：`reservation | direct`（immutable）
   - status：`reserved | bound | settled | released | expired`
   - type：`WalletAllocationType`（`deposit | withdrawal | transfer | adjustment`），caller 宣告，Wallet 不推斷
   - fundCaseIdentifier：nullable（reserve 時 null，bind 時回填）
   - reservedUntil：僅 reservation flow，進入 bound 後失去語義效力
   - `planLines: WalletAllocationLine[] | null`：配置計畫（reserve / propose / direct-settle 時 set-once）
   - `outcomeLines: WalletAllocationLine[] | null`：最終結果快照（allocation 到達終態時 set-once）；未到達終態時為 null
   - 狀態轉換邏輯封裝在 entity（factory methods + transition methods）

2. **WalletAllocationLine embedded value object（aggregate raw）**
   - aggregate raw 欄位：space, side, walletId, bucket, currencyCode, amount, kind
   - raw **不含** `id` / `allocationId` / `role`
   - `role` 是 persistence-level discriminator，僅存在於 DB table / model，不進 aggregate raw / contract shape
   - space：`internal | external`
   - side：`source | destination`
   - bucket：internal 時 `available | held`（Fund 視角，僅 2 值）；external 時 null
   - kind：`principal | network_fee | service_fee | adjustment`（`WalletAllocationLineKind`，不含 `reservation`）
   - amount：非負數；zero outcome 場景可為 0

3. **Persistence：role column 與 mapper**
   - DB 使用同一張 `wallet_allocation_line` table 儲存 planLines 與 outcomeLines
   - `role` column：`WalletAllocationLineRole`（`plan | outcome`），persistence-only
   - Mapper 負責按 `role` filter 組裝為 aggregate raw 的 `planLines` / `outcomeLines`
   - 寫入時由 repository method 決定 role 值（見下方 repository 說明）

4. **Repository**
   - WalletAllocationRepository：CRUD + 按 status 查詢 + 按 fundCaseType/fundCaseIdentifier 查詢
   - 不設 `WalletAllocationLineRepository`；line row 由 allocation aggregate repository 一併持久化
   - **Create-time**：`createInTx(draft, tx)` 建立 allocation root + draft 中有值的 lines（repository 內部按 draft 的 planLines / outcomeLines 有值與否決定 role 寫入）。不拆分為兩個 create method——單一 `createInTx` 同時適用 reservation/propose（planLines 有值）與 direct-settle（planLines / outcomeLines 同 call 同時有值）
   - **Settle-time**：`recordOutcomeLinesInTx(allocationId, lines, tx)` 寫入 outcomeLines（role=outcome），set-once。若已存在 outcome lines 則拋錯
   - **Update**：`updateByIdInTx(id, patch, tx)` 只改 parent mutable fields（status, fundCaseIdentifier, updatedAt），patch type **排除 lines**
   - planLines create-time immutable；outcomeLines settle-time set-once；兩者都不走 update patch

5. **DB migration**
   - 新建 wallet_allocation 表（**不含 request_key**）
   - 新建 wallet_allocation_line 表（含 `role` column：`plan | outcome`）

### Phase 4：Ledger Gateway + Balance Mutation

目標：實作所有 ledger 操作邏輯。

約束：

- **Wallet Allocation Tx / unit-of-work 只處理 allocation + entry (+ lines) + balance**
- `WalletLedgerItem` / `WalletCashflowItem` 為 **post-commit sideflow**
- 不可把 Ledger / Cashflow mutation 放進 allocation transaction unit of work
- projector / sideflow handler failure 不可回滾 Wallet Allocation Tx；後續實作需可 retry / idempotent
- 本次只保留 sideflow hook / projection extension point

工作項：

0. **Asset 前提驗證**
   - 凡 allocation lines 涉及 internal `(walletId, currencyCode)`，對應的 `WalletAsset` 必須已存在
   - 不存在時在接收 allocation request 階段即 reject（business error，以 Either 回傳）
   - `WalletAssetBalance` 是 `WalletAsset` 的 mandatory companion row（建立 asset 時同 transaction 建立 zero balance row）
   - BalanceMutationService 一律假設 balance row 存在；不存在視為 system invariant violation

1. **Concurrency control**
   - 建立 `AllocationMutexService`（基於 `KeyedMutexService`，最薄包裝）
   - Phase 4 使用固定 key `'wallet-allocation'` 序列化所有 allocation 操作
   - **Mutex outer, transaction inner**：`mutex.runExclusive(() => runner.run(tx => ...))`
   - 目標設計：未來升級為 per `(walletId, currencyCode)` ordered locking，待吞吐量需求出現後再升級
   - `// TODO: upgrade to per-(walletId, currencyCode) ordered locking`

2. **Balance mutation 機制**（`BalanceMutationService`，放在 `feature/allocation/service/`）
   - 同一 transaction 內同步完成：entry line 寫入 + WalletAssetBalance 更新 + lastAppliedEntryLineId 推進
   - `lastAppliedEntryLineId` 定位為 reconciliation / rebuild watermark，不是 eventual catch-up 的 gap indicator
   - **Watermark row-scoped**：每個被 touched 的 balance row 各自推進至該 row 本次實際被套用的 entry lines 之 max(id)，不使用整批 lines 的全域 max
   - 正常交易路徑中 balance 永遠與最新 entry lines 同步
   - `computedAt` 設為 `new Date()`（每次 balance mutation 時取當下時間）
   - Balance mutation 涵蓋本次操作**所有 internal lines** 涉及的 `(walletId, currencyCode)` 組合（不分 source / destination）

3. **Per-entry net-effect validation**
   - 計算整筆 entry 所有 lines 的 net effect per (walletId, bucket, currencyCode)
   - 若任何組合 net effect 為負，驗證 pre-existing balance + net effect ≥ 0
   - 支援 same-entry self-funding
   - **對所有 flow 統一執行，不放寬**
   - Insufficient balance 依 caller actionable space 判定：reserve / direct-settle 為 caller-actionable business error（Either），**fail-fast first**，攜帶 `(walletId, currencyCode, required, available)`；settle / release / expire 為 invariant violation，throw

4. **Entry builder service**（`EntryBuilderService`，放在 `feature/allocation/service/`）
   - **Pure computation**：不碰 DB、不需要 `tx`、可完全 unit test
   - 責任：allocation lines → entry line drafts 的映射（Fund-bucket → physical-bucket）
   - **Flow-aware mapping**：Builder 根據 `allocation.flow` 區分兩套映射規則：
     - **Reservation flow**：
       - Reserve 映射：per internal source allocation line `(walletId, bucket=available, amount)` → `[(available, -amount), (locked, +amount)]`，kind=`reservation`；`bucket=held` → `[(held, -amount), (held_locked, +amount)]`
       - Settlement 映射：internal source 走 locked mapping（`available → locked`、`held → held_locked`）；internal destination 走原 bucket
       - Reversal 映射：per `(walletId, currencyCode, lockedBucket)` 計算差額。`locked` 差額 → `locked -delta, available +delta`；`held_locked` 差額 → `held_locked -delta, held +delta`
     - **Direct flow**：
       - Settlement 映射：所有 internal lines 走原 bucket（不經 locked），因為 propose 不鎖定資金
       - Direct-settle 與 propose→settle 共用同一套 direct flow mapping
       - 無 reservation / reversal mapping（direct flow 無 locked 資金）
   - `merchantId`：由 use-case 預先 resolve（walletId → merchantId mapping），餵給 builder。Builder 不注入 repository。若 wallet type 為 merchant_wallet 卻 resolve 不到 merchantId，應拋錯

5. **Envelope check**
   - 按 `(side, space, walletId?, bucket?, currencyCode, kind)` 為 signature 分組比較 amount 總和
   - **Internal + external lines 都參與** envelope check
   - Settle（reservation / direct flow）：outcomeLines vs planLines，按 signature 分組比較
   - **Direct-settle 不做 envelope 比較**（planLines 與 outcomeLines 在同 call 內同時 set，envelope 由 caller 保證，Wallet 不複驗）

6. **AllocationProjectionService**（放在 `feature/allocation/service/`）
   - Interface 定義兩個 projection 關鍵點，與 `WalletAllocationLineRole`（plan / outcome）對稱：
     - `projectPlan(context)`：bind / propose 後觸發
     - `projectOutcome(context)`：settle / direct-settle / release（from bound）後觸發
   - **Projection context** 最小為 `{ allocation, fundCase, effectiveAt }`。不帶 trigger / fromStatus / toStatus（trigger 由 function name 表達，toStatus 已在 allocation.status）。不包含 entries——projector 用 allocation lines 推導，不穿透 accounting layer
   - **Line source 規則**：projectPlan 讀 planLines；projectOutcome 讀 planLines + outcomeLines 比較差異（release 時 outcomeLines = null → 全部歸零）
   - Phase 4 註冊 no-op implementation
   - Projection hook 在 `TransactionRunner.run()` 完成後由 use-case 直接呼叫，不需特殊 post-commit callback 機制
   - expire / reserve 不觸發 projection
   - release from reserved（未曾 bound）不觸發 projection

7. **Ledger gateway use cases**
   - 每個操作各一個 use case，放在 `feature/allocation/use-case/`
   - Use-case 定義自己的 `Command` schema（e.g. `ReserveCommand`）；若有主體性形狀或共用價值可放 `feature/allocation/schema/`，否則放 use-case 同檔
   - Use-case 以 `Either<Result, Error>` 封裝 **caller-actionable** business error（status 前提不符、envelope 違反、caller-input wallet/asset 不存在、reserve / direct-settle 的 insufficient balance）
   - **System invariant violation 一律 throw 不入 Either**：balance row 不存在（asset 存在則 balance 必存在的 invariant 破壞）、reversal 流程 balance insufficient（locked 數學保證、settle envelope-bounded）、DB lines 中的 wallet 在 reserve 後消失等
   - 判定原則：「caller 有無可採取的合理恢復動作」，不是「典型發生機率」
   - 錯誤型別放在 `feature/allocation/error/`（feature-level shared error）；use-case-private error 留在 use-case 同檔
   - Use-case 頂層先 `const now = new Date()` 產生時間戳，傳入後續邏輯
   - 使用 `TransactionRunner` 管理 transaction lifecycle

   **reserve()**：
   - 建立 allocation（flow=reservation, status=reserved）+ planLines（role=plan）
   - 產生 reservation entry（all lines kind=reservation，bucket: available→locked / held→held_locked）
   - 同步更新 WalletAssetBalance
   - Insufficient balance = caller-actionable business error (Either)，fail-fast first；caller 可調整金額或補錢後重試
   - 🔌 Projection: 無（fundCaseIdentifier 尚未存在）
   - 回傳 allocationId

   **bind()**：
   - reserved → bound，回填 fundCaseIdentifier
   - 不產生 entry（無 balance 變動）
   - status 前提不符一律回 Either error
   - 🔌 Projection: `projectPlan`

   **propose()**：
   - 建立 allocation（flow=direct, born bound）+ planLines（role=plan）
   - 不產生 entry（尚未確認，不影響 balance——propose 不鎖定資金）
   - 支援後續 settle 時的 same-entry self-funding
   - 🔌 Projection: `projectPlan`

   **settle()**（reservation / direct flow）：
   - bound → settled
   - 持久化 outcomeLines（role=outcome，set-once）
   - Envelope check（kind-inclusive，internal + external）：按 signature 分組比較 outcomeLines vs planLines；超出為 caller-actionable error（Either）
   - 產生 settlement entry
   - Reservation flow: internal source 走 locked mapping
   - Direct flow: 直接原 bucket
   - 若 remainder > 0（reservation flow only）：產生 reversal entry（kind=reservation，精細粒度 per lockedBucket）
   - 同步更新 WalletAssetBalance
   - **outcomeLines 寫入、entries 建立、balance mutation、status → settled 必須同 transaction**
   - **Net-effect validation 失敗為 invariant violation**（throw 不入 Either）：reservation flow 受 envelope + locked 數學保證；direct flow 受 envelope 鎖死，caller 無法調整 outcomeLines / planLines（唯一恢復路徑是 release 後重新 propose，屬業務決策非 retry）
   - 🔌 Projection: `projectOutcome`

   **direct-settle()**：
   - 建立 allocation（flow=direct, born settled）+ planLines（role=plan）+ outcomeLines（role=outcome），同 call 同時 set-once
   - 若 outcomeLines 非全 zero，直接產生 settlement entry（direct flow mapping，原 bucket）
   - 不做 envelope 比較（same-call 由 caller 保證）
   - 若 outcomeLines 非全 zero，同步更新 WalletAssetBalance
   - outcomeLines 全 zero 時，allocation 仍 born settled 且 planLines + outcomeLines 仍寫入；但不產 entry、不更新 balance
   - Insufficient balance = caller-actionable business error (Either)，fail-fast first；caller 可調整 planLines / outcomeLines / fee 後重新提交此次 directSettle
   - 🔌 Projection: `projectOutcome`

   **release()**：
   - reserved/bound → released
   - 若有 locked 資金（reservation flow）：產生 reversal entry（kind=reservation），同步更新 WalletAssetBalance
   - 若無 locked 資金（direct flow, proposed）：不產生 entry
   - **Reversal 流程 net-effect validation 失敗為 invariant violation**（throw）：reversal 是把 locked 退回 available，數學上不會 fail；若 fail 表示 locked 被未追蹤途徑改動
   - DB lines 的 wallet/asset missing 為 invariant（reserve 時已驗過），throw
   - 🔌 Projection: 若曾 bound → `projectOutcome`（outcomeLines = null → 全部歸零）；若從 reserved 直接 release → 無

   **expire()**：
   - 僅限 reserved
   - use-case 自主查詢需要 expire 的 allocations，逐筆 expire
   - 產生 reversal entry，同步更新 WalletAssetBalance
   - **Reversal 流程 net-effect validation 失敗為 invariant violation**（throw）：同 release 推理
   - DB lines 的 wallet/asset missing 為 invariant（reserve 時已驗過），throw
   - 🔌 Projection: 無（ghost reservation 對使用者不可見）

### Phase 5：RPC Contract + Fund Integration

目標：暴露 Wallet Allocation 操作給 Fund service，完成 deposit / withdrawal intent 端到端流程。

工作項：

1. **RPC 定義**（在 `@esingpay/contract-rpc` 中）
   - Namespace：`wallet`
   - 定義 `walletAllocationRpc`
   - Methods：reserve / bind / propose / settle / directSettle / release
   - Input/output DTO 參考 spec Section 9

2. **RPC server（provider side）**
   - 在 Wallet service `rpc/server/` 目錄
   - 各 method delegate 到對應的 use case
   - RPC controller 負責 DTO → Command mapping + Either → result code mapping

3. **RPC client**
   - 在 Fund service `rpc/client/` 目錄
   - 供 Fund 的 withdrawal / deposit use case 呼叫

4. **Fund service 整合**
   - WithdrawalIntent create flow：reserve → persist → bind
   - WithdrawalIntent settle flow：settle（with actual lines）
   - Deposit discovered pending：propose
   - Deposit settled：settle（existing allocation）
   - Deposit discovered already settled：directSettle

---

## 3. 不在本次範圍的項目

- Wallet 本體的改動（已完整實作）
- WalletAsset REST API（參考 Wallet_Asset_Design，但本次聚焦帳務層）
- **WalletLedgerItem**（使用者帳本明細 read model；未來 ledger list / pagination source，另案定義 schema 與實作）
- **WalletCashflowItem**（獨立 cashflow read model，另案定義 schema 與實作）
- Transfer intent 的 allocation 整合（模型已支援，實作排後）
- expire scheduler（已有 scheduler 模式可套用，排後）
- `direct-settle` 命名最終化（暫用）
- Ledger / Cashflow REST API（本次 RPC 先行，read model 排後）
- WalletAssetBalanceSnapshot（未來規劃，不在本次）

---

## 4. 架構對齊提醒

以下是 AI agent 實作時需注意的既有架構規範：

### Entity Pattern

- **WalletAllocation**：使用 `DraftableEntityBase`。有複雜生命週期行為（reserve → bind → settle / release / expire）
- **Entity transition methods**：`bind() / settle() / release() / expire()` 應直接 mutate entity state + 呼叫 `touch()`，回傳 `void`
- **WalletAllocation entity**：內部持有 allocation raw（內嵌 planLines + outcomeLines）。`toRaw()` 輸出完整 aggregate raw
- **Settlement lines 寫入**：settle transition 時 entity 接收 outcomeLines 並設入 raw（set-once，已有 outcomeLines 則拋錯）
- **WalletEntry**：**不需要 DraftableEntityBase**——entry 是 immutable fact，建立即 posted，建立後不可修改。可簡化為 factory function 或輕量 helper，不需要 dedicated entity file
- **預期使用方式**：
  - Allocation create：`entity = createDraft(...)` → `await repo.createInTx(entity.toDraft(), tx)`
  - Allocation settle：`entity.settle(outcomeLines)` → `await repo.recordOutcomeLinesInTx(...)` + `await repo.updateByIdInTx(...)`
  - Entry：由 entry-builder 建構 PackDraft → `await repo.createPackInTx(packDraft, tx)`（無 update）
- **structuredClone for bigint**：entity 內部 clone 使用 structuredClone，不用 lodash

### Draft / Patch / Raw 定義

- **`WalletAllocation` raw 包含 lines**：`planLines: WalletAllocationLine[] | null` + `outcomeLines: WalletAllocationLine[] | null` 是 aggregate raw 的一部分，不是外掛的 pack children。WalletAllocationLine 是 embedded value object，不具獨立 identity
- **`WalletAllocationDraft`**：`Omit<WalletAllocation, 'id'>`。單一 draft type，不拆分為兩種。Draft 中 planLines / outcomeLines 的 null 或有值由 entity factory 根據 flow 決定：
  - reservation / propose：`planLines` 有值，`outcomeLines` 為 null
  - direct-settle：`planLines` 有值，`outcomeLines` 有值（planLines 與 outcomeLines 同 call 同時 set-once）
- **`WalletAllocationPatch`**：`Partial<Pick<WalletAllocation, 'status' | 'fundCaseIdentifier' | 'updatedAt'>>`。明確排除 lines——planLines 為 create-time immutable，outcomeLines 為 settle-time set-once，兩者都不經由 patch 修改
- **Repository method 與 schema 對應**：
  - `createInTx(draft: WalletAllocationDraft, tx)` — 建立 aggregate root + draft 中有值的 lines
  - `updateByIdInTx(id, patch: WalletAllocationPatch, tx)` — 只改 parent mutable fields
  - `recordOutcomeLinesInTx(allocationId, lines: WalletAllocationLine[], tx)` — set-once insert outcomeLines
  - read methods 回傳完整 `WalletAllocation`（含 lines）

### Raw 與 Aggregate Schema 分層

#### Contract-base raw：分離

- `WalletEntry` 和 `WalletEntryLine` 在 contract-base 層維持 **獨立 raw**，不內嵌
- `WalletAllocation` 在 contract-base 層的 raw 內嵌 `planLines: WalletAllocationLine[] | null` + `outcomeLines: WalletAllocationLine[] | null`（allocation line 是 embedded value object，無獨立語義）
- **分層規則**：`contract-base` 的 raw 維持資源級別。需要 parent + children 聚合輸入輸出時，由 service implementation 另外定義 pack / composed schema，不回滲到 base raw

#### Service implementation 層：Pack schema

- `WalletEntryPack`：aggregate read output — `{ entry: WalletEntry; lines: WalletEntryLine[] }`
- `WalletEntryPackDraft`：aggregate create input — `{ entry: Omit<WalletEntry, 'id'>; lines: WalletEntryLineDraft[] }`
- 放在 service-local `schema/` 目錄
- 未來的 composed read schema（例如 `WalletEntryLineComposed { line, entry, wallet, merchant? }`）獨立定義，不影響 pack 形狀

#### AllocationLine vs EntryLine：不同的簡化規則

Allocation line 是 **embedded value object**（配置計畫 / 最終結果快照明細）；Entry line 是 **ledger fact unit**。兩者不套用同一套規則。

**WalletAllocationLine**（embedded value object）：

- Aggregate raw 中不包含 surrogate `id`、parent FK `allocationId`、以及 `role`——infra concern，留在 model 層
- `role`（`plan | outcome`）是 persistence-level discriminator，mapper 按 role filter 組裝為 planLines / outcomeLines
- 不被外部獨立引用、不被分頁、不作為 watermark target

**WalletEntryLine**（ledger fact unit）：

- Raw 中**保留 `id`**——具實質語義用途：watermark 引用、fact row ordering、raw line query pagination / sorting / audit
- **保留 `entryId`**——line 需要被獨立查詢時的 join key
- **保留 `merchantId`**——fact-time denormalization：由 walletId 導出、建 line 時寫入、不追隨 wallet 關聯變化
- **保留 `createdAt`**；**不含 `updatedAt`**（insert-once immutable）

### Set-once Lines

- `planLines`：create-time set-once。Reserve / propose / direct-settle 時一次寫入，之後不可修改、不可覆蓋
- `outcomeLines`：terminal-time set-once。Settle 時一次寫入，之後不可修改、不可覆蓋。Settle 時寫入必須與 entries / balance / status 同 transaction。Release from bound 時 outcomeLines 不寫入（為 null，全部歸零語義由 projection 推導）
- `WalletEntryPack.lines`：create-time immutable（entry 本身即 insert-once）
- `WalletAllocationRepository.updateByIdInTx(...)` 只改 parent mutable fields（`WalletAllocationPatch`），不做 child diff / child rewrite；patch type **排除 lines**
- `WalletAllocationRepository.recordOutcomeLinesInTx(...)` 為 outcomeLines 的唯一寫入路徑（set-once，已存在則拋錯）
- `WalletEntryRepository` 無 update path

### Repository Pattern

- **Repository boundary**：repository public methods 不接受/回傳 entity instances；hydration 在 use case 層
- **Transaction context**：使用 `TransactionRunner` 管理 transaction lifecycle。Mutation repository methods 顯式吃 `tx: TransactionContext`。純讀 methods 不吃 `tx`；需要 row-level lock 的讀操作（如 `SELECT FOR UPDATE`）視為 tx-bound，顯式加 `InTx` suffix 並吃 `tx`。不採 optional `tx?` 混用
- **Explicit mapping**：mapper 不使用 object spread，兩側顯式列出欄位
- **WalletAllocation repository naming**：`WalletAllocation` raw 內嵌 lines（embedded value object），因此 create 命名為 `createInTx(draft, tx)`（標準 primary raw resource create，不用 `createPackInTx`）。`updateByIdInTx(id, patch, tx)` 只改 parent。`recordOutcomeLinesInTx(allocationId, lines, tx)` 為 set-once child insertion
- **WalletEntry repository naming**：aggregate create 命名為 `createPackInTx(packDraft, tx)`，明示非單 row create（entry 與 entry line 為兩個獨立 raw，需要 pack 語意）；`InTx` suffix 對齊 mutation method 帶 tx 的 naming convention。Read 命名為 `getPackById(id)` / `getPacksByAllocationId(allocationId)`，read methods 不帶 tx 故無 `InTx` suffix。**不提供 `update(...)`**，因為 entry / line 為 create-once immutable fact

### Typing & Naming

- **Domain enum preference**：`WalletAllocationStatus` / `WalletAllocationFlow` / `WalletAllocationType` / `WalletAllocationPhase` / `WalletAllocationLineRole` / `WalletEntryLineKind` 這類業務欄位，預設使用 TypeScript string enum
- **RPC method result code**：若有 RPC method 需要定義 result code / error code，使用一般 TypeScript string enum；不要用 `const enum`

### Error & Validation

- **Error flow**：use-case boundary 以 `Either<Result, Error>` 封裝 caller-actionable business error。涵蓋：status 前提不符、envelope 違反、caller-input wallet/asset 不存在、reserve / direct-settle 的 insufficient balance。System invariant violation 一律 throw，不入 Either
- **Insufficient balance**：依 caller actionable space 判定：reserve / direct-settle 為 business error (Either)，**fail-fast first**，攜帶 `(walletId, currencyCode, required, available)`；settle / release / expire 為 invariant violation，throw
- **錯誤型別**：feature-level shared error 放在 `feature/allocation/error/`；use-case 專屬錯誤可留在 use-case 鄰近
- **Asset 前提**：建立 allocation 前驗證所有 internal lines 的 `(walletId, currencyCode)` 對應的 WalletAsset 存在。不存在 = 業務錯誤，以 Either 回傳
- **Balance 前提**：BalanceMutationService 一律假設 balance row 存在。不存在 = system invariant violation，throw，不以 Either 回傳

### Persistence & Numeric

- **金額型別原則**：`amount` 用 `NumericString`（raw / contract）/ `Decimal`（domain 計算）/ DB `decimal(32,16)`。`bigint` 僅用於 identity / FK 欄位
- **DB decimal 精度**：monetary columns 一律 `type: 'decimal', precision: 32, scale: 16`
- **Numeric calculation**：`NumericString` → `Decimal`（decimal.js-light）做計算，直接 `new Decimal(numericString)`，不需額外 converter helper
- **Timestamp precision**：DB timestamp columns 使用 `timestamptz, precision: 3`
- **Balance mutation 同步**：同一 transaction 內同步寫 entry lines + 更新 WalletAssetBalance + 推進 watermark（row-scoped，nullable）
- **Shallow directory structure**：max 兩層

### Clock

- Use-case 頂層先 `const now = new Date()` 產生時間戳，傳入 `perform` 方法
- 若 use-case 拆分為 `execute` 和 `perform` 兩階段，由 `execute` 產生 `now` 傳入 `perform`
- `ClockService` 注入列為未來 TODO，之後再統一

---

## 5. 建議的目錄結構

```
wallet-service/
  entity/
    wallet-allocation.entity.ts               # 新增（DraftableEntityBase，內嵌 planLines + outcomeLines）
  model/
    wallet-asset.model.ts                     # 新增
    wallet-asset-balance.model.ts             # 新增
    wallet-allocation.model.ts                # 新增
    wallet-allocation-line.model.ts           # 新增
    wallet-entry.model.ts                     # 新增
    wallet-entry-line.model.ts                # 新增
  mapper/
    wallet-asset-model.mapper.ts              # 新增
    wallet-asset-balance-model.mapper.ts      # 新增
    wallet-allocation-model.mapper.ts         # 新增（含 role filter → planLines / outcomeLines 組裝）
    wallet-entry-model.mapper.ts              # 新增（含 lines pack/unpack）
  repository/
    wallet-asset.repository.ts                # 新增
    wallet-asset-balance.repository.ts        # 新增
    wallet-allocation.repository.ts           # 新增（createInTx / updateByIdInTx / recordOutcomeLinesInTx）
    wallet-entry.repository.ts                # 新增（createPackInTx / getPackById / getPacksByAllocationId）
  schema/
    wallet-allocation-draft.ts                # 新增（WalletAllocationDraft / WalletAllocationPatch）
    wallet-entry-pack.ts                      # 新增（WalletEntryPack / WalletEntryPackDraft）
  feature/
    allocation/                               # 新增 feature
      allocation.module.ts
      allocation-context.module.ts
      error/
        allocation.error.ts                   # feature-level shared error types
      use-case/
        reserve.use-case.ts
        bind.use-case.ts
        propose.use-case.ts
        settle.use-case.ts
        direct-settle.use-case.ts
        release.use-case.ts
        expire.use-case.ts
      service/
        allocation-mutex.service.ts           # KeyedMutexService wrapper，固定 key 'wallet-allocation'
        balance-mutation.service.ts           # 餘額驗證 + 同步更新（tx-bound）
        entry-builder.service.ts              # entry + entry line 建構（pure computation，flow-aware）
        allocation-projection.service.ts      # projection interface + no-op impl
  rpc/
    server/
      server.module.ts
      wallet-allocation/
        wallet-allocation.module.ts
        wallet-allocation.controller.ts
```

注意：

- entity / model / mapper / repository 放在 service root 層級（aggregate-core placement）
- `schema/wallet-allocation-draft.ts` 放在 service root `schema/`（`WalletAllocationDraft` / `WalletAllocationPatch` 由 repository create / update method 使用，跨多個 feature use case 共用）
- `schema/wallet-entry-pack.ts` 放在 service root `schema/`（Pack 型別會被多個 feature 共用）
- WalletEntry 不需要 entity 檔——posted-only immutable fact，由 entry-builder 建構 PackDraft
- `WalletAsset` / `WalletAssetBalance` 走 raw 直接 persist，不需要 entity 檔
- RPC 命名為 `walletAllocationRpc`（`ledger` 保留給未來 LedgerItem read model）
- `feature/allocation/error/` 放置 feature-level shared error types
- `feature/allocation/` 依賴 RepositoryModule + TransactionRunnerModule + KeyedMutexService 所在 module；經 `feature.module.ts` 註冊

---

## 6. 關鍵型別值域對照

| 概念                   | 型別名                     | 值域                                                                           |
| ---------------------- | -------------------------- | ------------------------------------------------------------------------------ |
| Allocation flow        | `WalletAllocationFlow`     | `reservation` \| `direct`                                                      |
| Allocation status      | `WalletAllocationStatus`   | `reserved` \| `bound` \| `settled` \| `released` \| `expired`                  |
| Allocation space       | `WalletAllocationSpace`    | `internal` \| `external`                                                       |
| Allocation side        | `WalletAllocationSide`     | `source` \| `destination`                                                      |
| Allocation line kind   | `WalletAllocationLineKind` | `principal` \| `network_fee` \| `service_fee` \| `adjustment`                  |
| Allocation line bucket | （Fund 視角）              | `available` \| `held`（internal 時）；null（external 時）                      |
| Allocation line role   | `WalletAllocationLineRole` | `plan` \| `outcome`（persistence-only，不進 aggregate raw）                    |
| Allocation type        | `WalletAllocationType`     | `deposit` \| `withdrawal` \| `transfer` \| `adjustment`                        |
| Entry allocation phase | `WalletAllocationPhase`    | `reservation` \| `settlement` \| `reversal`                                    |
| Entry line kind        | `WalletEntryLineKind`      | `reservation` \| `principal` \| `network_fee` \| `service_fee` \| `adjustment` |
| Entry line bucket      | （物理 bucket）            | `available` \| `locked` \| `held` \| `held_locked`                             |

### 維度概念對照

| 概念     | 型別名                     | 所屬資源                      | 語義                  |
| -------- | -------------------------- | ----------------------------- | --------------------- |
| 狀態     | `WalletAllocationStatus`   | Allocation                    | 生命週期位置          |
| 紀錄用途 | `WalletAllocationLineRole` | AllocationLine（persistence） | 這組 lines 的紀錄用途 |
| 帳務作用 | `WalletAllocationPhase`    | Entry                         | 這筆 entry 的會計性質 |

注意：WalletEntry 不再有 status 欄位（posted-only）。
注意：WalletEntry / WalletEntryLine 皆不含 `updatedAt`；WalletEntry 亦不含 `allocationType`。

---

## 7. 參考文件

- `wallet-allocation-design.md`：WalletAllocation 完整 spec
- `wallet-basic-design.md`：Wallet / WalletAsset / WalletAssetBalance 基礎資源規格
- `guide/contract-structure.md`：contract package 組織規範
- `guide/service-structure.md`：service 目錄結構規範
- `guide/implementation-rules.md`：repository IO shape / mapper / entity / DI rules

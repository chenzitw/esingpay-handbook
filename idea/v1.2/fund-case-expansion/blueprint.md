---
status: draft
updated_at: 2026-05-13
updated_by: Tim
remark: 本文件未依照 blueprint 規範撰寫，暫供內容參考，等待後續重構改寫。
---

# Fund Case Expansion Blueprint

## Scope

本 roadmap 對應 `./design.md` 與 `./design-rest.md` 的實作落地節奏。涵蓋：

- WithdrawalIntent 擴展（category 引入、platform sender 支援、gather batch、destination_internal 偵測）
- TransferIntent 新建（全新 fund case，含 fast-forward 政策、gather/distribute batch、ledger projection 整合、instruct flow）

不涵蓋：

- 設計層細節——見 design draft / REST contract spec
- 各 plan 內部 phase 切分與 touchpoint 列表——留 plan 起草階段
- Wallet allocation 對 transfer 的 path 涵蓋設計（屬 wallet-allocation ep-2 範疇，本 roadmap 不展開）
- Cashflow projection（本期不實作）

---

## Strategy

依 fund case 邊界切分為 ep-1（WithdrawalIntent 擴展）與 ep-2（TransferIntent 新建），兩 ep 之間**完全解耦**、可並行起作。

Staging 上線節奏：依 ep 整波上 staging。

- **Stage 1**: ep-1 plan-1 ~ plan-4 全部完成 → staging deploy（platform withdrawal intent 完整可用）
- **Stage 2**: ep-2 plan-1 ~ plan-8 全部完成 → staging deploy（transfer intent 完整可用）

---

## Plan Inventory

### ep-1: WithdrawalIntent 擴展

#### ep-1 plan-1: feePayer 持久化 + Category 整改

合併 cross-cutting 改造（feePayer 持久化）與 WithdrawalIntent category 引入。兩者高度耦合 schema migration 路徑，分開 plan 反而增加 migration 複雜度。

**範圍**：

- 持久化層：raw / entity / model / migration 加 `feePayerWalletId` 與 `category`
- DTO / REST mapper：`WithdrawalIntentDto` 加 `category` + `feePayerWallet`；search params 加 `categoryIn`；quote DTO 加 `category`
- Submit use cases（merchant / platform）寫入新欄位，category 由 sender wallet type 推導
- `InstructWithdrawalIntentUseCase` 改讀 entity `feePayerWalletId`，不再呼叫 resolver
- `WithdrawalIntentFeePayerResolverService` 角色收斂為 submit-time-only
- `WithdrawalIntentEvaluationService` 既有 `senderWallet.merchantId !== null` 分支 → `category === FromMerchant`
- `WithdrawalIntentAutoProgressService.canAutoRelease` 既有 `senderMerchantId === null` 分支 → `category === FromPlatform`
- Side-effect 封裝：抽出 `WithdrawalIntentInstructionDispatcherService`（語義中性 dispatch wrapper），既有 sideEffectService 內呼叫；platform submit path 在 fast-forward 結束後也呼叫
- `cancellableUntil` per-category 填值政策：`from_merchant` 沿用既有規則、`from_platform` 設為 `submittedAt`（trivially-past，讓 acceptance queue 立即可受理）

**依賴**：無（ep-1 起點）

**Phase 切分方向**（具體留 plan 起草階段細調）：

- Phase 1：持久化骨架（raw / entity / model / migration / repository）
- Phase 2：DTO + REST mapper
- Phase 3：Submit use cases 寫入新欄位
- Phase 4：Instruct use case 改讀 entity + resolver 角色收斂
- Phase 5：Evaluation / Auto-progress service `senderMerchantId === null` 整改 + Instruction dispatcher 抽出 + platform fast-forward path

#### ep-1 plan-2: Allocation Line Builder Category-Aware

**範圍**：

- `WithdrawalIntentAllocationLineBuilderService` 依 category 變形 plan lines：
  - `from_merchant`：principal + networkFee pairs；`amountFee > 0` 時另含 serviceFee pair（共 6 lines），`amountFee = 0` 時不 emit serviceFee placeholder（共 4 lines）
  - `from_platform`：4 lines（principal + networkFee × source/destination；無 serviceFee）
- 對應 settlement service 的 outcome builders（`buildCompletedOutcomeLines` 與 `buildNetworkFeeOnlyOutcomeLines`）處理 4-line shape
- Service fee 政策確立：**只有 merchant withdrawal 收 service fee**；platform withdrawal 無此名目

**依賴**：ep-1 plan-1（需 entity 內 `category` 已持久化）

#### ep-1 plan-3: Gather Batch Endpoint

**範圍**：

- 新增 REST endpoint：`POST /fund/plat/withdrawal-intents/$submit-gather-batch`
- 新增 use case：`PlatformSubmitWithdrawalIntentGatherBatchUseCase`，內部 fan-out N 次 single submit use case
- 既有 single submit use case 不變動——batch use case 僅作 orchestration
- Per-item result 對位（與 input items 同序、同長度）+ issue code mapping
- Per-item issue code enum：`WithdrawalIntentSubmitIssueCode`
- 容量上限 50 筆（超出回 envelope-level `data_invalid`）
- 採既有 codebase fan-out pattern：sequential `for...of` + per-item try/catch（mirror `ExpireReservedAllocationsUseCase`）

**依賴**：ep-1 plan-1 + ep-1 plan-2（單筆 platform submit 須完整可用）

#### ep-1 plan-4: destination_internal 偵測

**範圍**：

- `NetworkEndpointRpc` 擴充：新增 `findByIdentifier(...)` 或 `listByIdentifierIn(...)`（含 repository query filter、facade、RPC controller）
- DB 已有 `identifier` unique constraint，repository 層加 query filter 即可
- `WithdrawalIntentEvaluationService.evaluate(...)` 內呼叫此查找，命中即 issue `destination_internal`
- `WithdrawalIntentQuoteIssueCode.DestinationInternal` 已於 spec 確立，本 plan 落地實裝
- Envelope-level error code `withdrawal_intent.destination_internal` 暴露

**依賴**：ep-1 plan-1（需 evaluation service 已 category-aware）

**並行特性**：與 plan-2 / plan-3 解耦深（屬 cross-BC NetworkEndpoint 改動），可與 plan-2 並行起作，不影響 plan-2 / plan-3 路徑。

### ep-2: TransferIntent 新建

#### ep-2 plan-1: 持久化骨架

**範圍**：

- 新建 `TransferIntent` raw / entity / model / migration / repository
- 新建 `TransferIntentStatus` enum（10 值，cancelled / declined / rejected 註解標示「不可用」，過渡狀態註解標示「略過」）
- 新建 `TransferIntentCategory` enum
- Entity transition map（mirror Deposit `recognizedTransitionPath` 模式）：`preparing → reviewing → processing → processed → transacting → completed/failed`
- Entity factory `createSubmitted(...)` 建立 `preparing` draft + structural invariant 校驗（`(category, bucket, network, feePayer)` 四項）；**不**執行 fast-forward
- `submittedTransitionPath` 常數於 entity 檔暴露（作為後續 plan AutoProgressService 的 transition 序列契約來源）
- Entity action methods：`instruct(...)` / `handleSettled(...)` / `handleFailed(...)`（不含 cancel / reject / release / accept / approve / decline）
- 新建 `transferIntentRpc.getById`（mirror `withdrawalIntentRpc` 結構，採 `ResultDto<CommonCode.Ok, TransferIntentDto | null>` 形狀）

**依賴**：無（ep-2 起點，與 ep-1 完全解耦）

#### ep-2 plan-2: Foundation — Services + Contract-rest types + Queue infra

**範圍**：

- contract-rest 端 type 落地（submit/quote 範圍）：
  - `PlatformSubmitTransferIntentDto`、`TransferIntentQuoteDto`、`TransferIntentDto`（rich shape）
  - `TransferIntentQuoteIssueCode` enum、`TransferIntentSubmitIssueCode` enum（submit issue code 由 plan-5 batch 端首次使用、本 plan 先 surface 對位）
  - `TransferIntentCode` REST envelope error code enum + 對應 `.message.ts` / `.status.ts`
  - `plat-transfer-intent.api.ts` 內含 `submitTransferIntent` + `quoteTransferIntent` 兩 endpoint definition（search / getById 屬 plan-4）
- Application services：
  - `TransferIntentEvaluationService`（quote 邏輯 + same-network 不變式 + wallet_circular 檢查 + balance / config 檢查；balance 經由 `walletBalanceRpc.isBucketSufficient`）
  - `TransferIntentAllocationLineBuilderService`（依 category 構造 plan lines，含 `merchant_to_platform` 的 delegated feePayerWallet）
  - `TransferIntentFeePayerResolverService`（命名對稱 withdrawal；TODO 條目記錄「考慮與 `WithdrawalIntentFeePayerResolverService` 抽共用」）
  - `TransferIntentAutoProgressService`（單一同步 `progress({ entity, occurredAt })` method、unconditional fast-forward 至 `processed`；對 entity 依 `submittedTransitionPath` 依序 transitions、append statusHistory、設 `processedAt`）
  - **沿用** 既有 `WithdrawalNetworkFeeEstimationService`——該 service 的 "Withdrawal" 為 behavior scope（任何 sender 對 external network 發 transaction），非 fund case scope；transfer sender 同樣在做 withdrawal 之事，直接注入既有 service、不建 transfer-specific estimation service
- Queue infra：
  - `TransferIntentInstructionDispatcherService`（命名對稱 withdrawal dispatcher）
  - `TransferIntentInstructionQueueDispatcher` + queue definition（mirror withdrawal 既有 queue pattern；worker 屬 plan-7）
- Module wiring 擴張（providers 增 services；export 限制依既有 feature module 政策對位）

**依賴**：ep-2 plan-1

**Cross-BC 依賴**：本 plan 引入對 `walletBalanceRpc.isBucketSufficient({ walletId, currencyCode, amount, bucket })` 的依賴，由 **ep-2 plan-8** 承接落地（mirror 既有 `walletBalanceRpc.isWithdrawalSufficient` 結構）。Phase 2 evaluation service 直接照預期 signature 呼叫、caller 位置留 TODO 註解。Build-time 對位：plan-8 contract-rpc method signature 必須於本 plan phase 2 起作前 surface（否則 evaluation service build fail）。

#### ep-2 plan-3: Submit/Quote wiring

**範圍**：

- `PlatformSubmitTransferIntentUseCase`（單一 use case 涵蓋三 category）
  - Category 自動推導：依 (sender wallet type, recipient wallet type) pair
  - Submit flow：evaluate → `walletAllocationRpc.reserve` → entity factory `createSubmitted` (preparing) → `autoProgressService.progress` (in-memory fast-forward to processed) → `repository.saveInTx` (transaction) → `walletAllocationRpc.bind` → `instructionDispatcher.dispatch` (best-effort)
  - Compensation：reserve 後 persist 失敗 → reserve release best-effort；reserve + persist 後 bind 失敗 → delete preparing intent + log orphan
- `PlatformQuoteTransferIntentUseCase`（純 read-side、無 allocation 副作用、不持久化）
- `TransferIntentComposer`（mirror `WithdrawalIntentComposer`；批次查詢 sender/recipient merchant + sender/recipient/feePayer wallet + sender/recipient endpoint + target network transaction + actors）
- `TransferIntentRestMapper`（`toTransferIntentDto` / `toTransferIntentQuoteDto`，依 composed refs 組裝 rich DTO）
- REST controller：`PlatTransferIntentController` 內 `submitTransferIntent` + `quoteTransferIntent` 兩 method
- Repository 擴張：`saveInTx(input, tx)` + `deletePreparingInTx(id, tx)`
- Module wiring 擴張（providers 增兩 use case + composer + REST mapper；controller 註冊）

**依賴**：ep-2 plan-2、ep-2 plan-8（plan-8 wallet-side `isBucketSufficient` server-side impl 必須於本 plan 起作前 landed，否則 submit/quote use case 第一次呼叫 evaluation 即 RPC 失敗）

#### ep-2 plan-4: Read path — Search + GetById

**範圍**：

- contract-rest 端擴張：`plat-transfer-intent.api.ts` 加 `searchTransferIntents` + `getTransferIntentById` 兩 endpoint definition；`PlatformSearchTransferIntentParamsDto`
- `PlatformSearchTransferIntentsUseCase`、`PlatformGetTransferIntentUseCase`
- Repository 擴張：`search(query)`、`listByIdIn(ids)`（composer 批次 hydrate）
- Query service 擴張：`search(input)`、`listRefsByIdIn(input)`
- REST controller 擴張：`searchTransferIntents` + `getTransferIntentById` 兩 method（reuse plan-3 composer + REST mapper）

**依賴**：ep-2 plan-3

#### ep-2 plan-5: Gather + Distribute Batch

**範圍**：

- 兩個 batch endpoint：
  - `POST /fund/plat/transfer-intents/$submit-gather-batch`
  - `POST /fund/plat/transfer-intents/$submit-distribute-batch`
- 兩個 use case：`PlatformSubmitTransferIntentGatherBatchUseCase` / `PlatformSubmitTransferIntentDistributeBatchUseCase`，內部 fan-out N 次 single submit
- 容量上限 50 筆（與 withdrawal 共用相同 default）
- 採與 ep-1 plan-3 相同的 fan-out pattern
- Per-item issue code mapping：use case 內 private method，將 single submit error 翻譯為 `TransferIntentSubmitIssueCode`（enum 由 plan-2 落地）

**依賴**：ep-2 plan-3

#### ep-2 plan-6: Ledger Projection 整合

**範圍**：

- `WalletLedgerProjectionService` 注入 `transferIntentRpc`
- `fetchFundCaseNetworkTransactionId` 內 `FundCaseType.TransferIntent` switch case 從 TODO 改為實作
- 新增 `parseTransferIntentNetworkTransactionId` helper（mirror `parseDepositNetworkTransactionId` / `parseWithdrawalIntentNetworkTransactionId`）
- 確認既有 `WalletLedgerProjectionService` 的 plan / outcome line 處理對 transfer 三 category（含 `merchant_to_platform` 的 delegated feePayer line）正確 cover——projection 跨 fund case agnostic，預期無額外改動

**依賴**：ep-2 plan-1（`transferIntentRpc.getById` 已存在）

**並行特性**：與 ep-2 plan-2 / plan-3 / plan-4 / plan-5 / plan-7 解耦深（屬 wallet-side ledger 改動），可於 plan-1 phase 2 完成後即起作，不影響其他 ep-2 plan 路徑。

#### ep-2 plan-7: Instruct + handleSettled + handleFailed + Worker + Backfill

**範圍**：

- `InstructTransferIntentUseCase`（mirror `InstructWithdrawalIntentUseCase`；直接讀 entity `feePayerWalletId`，無 resolver path）
- `HandleNetworkTransactionSettledUseCase` for transfer（觸發 `entity.handleSettled(...)` + settle allocation completed outcome）
- `HandleNetworkTransactionFailedUseCase` for transfer（觸發 `entity.handleFailed(...)` + settle allocation network-fee-only outcome）
- `TransferIntentAllocationSettlementService`（zero / completed / networkFeeOnly outcome，mirror withdrawal 既有 service）
- Queue worker：`TransferIntentInstructionQueueWorker`（mirror withdrawal 既有 worker；queue definition + dispatcher 已於 plan-2 落地）
- `BackfillDueTransferIntentInstructionUseCase`（mirror `backfill-due-withdrawal-intent-instruction.use-case.ts`，補救 instruction dispatch 失敗）

**依賴**：ep-2 plan-3（worker 需 submit use case 建立的 row 來處理；end-to-end 驗證需要寫入路徑存在）

#### ep-2 plan-8: Wallet-side `isBucketSufficient` RPC

**範圍**：

- contract-rpc 端：`walletBalanceRpc.isBucketSufficient({ walletId, currencyCode, amount, bucket }): ResultDto<CommonCode.Ok, { sufficient: boolean }>` method signature（mirror 既有 `walletBalanceRpc.isWithdrawalSufficient` 結構）
- wallet cradle 端：`WalletBalanceRpcController.isBucketSufficient(...)` server-side impl；依 `bucket` 參數查 wallet 對應 bucket（`available` / `held`）的 balance、判斷是否足以支付 `amount`
- Mirror 既有 `isWithdrawalSufficient` query 路徑與 model lookup；Tim 評估實作不難
- 落地時**同步移除** fund 端 ep-2 plan-2 `TransferIntentEvaluationService` 內 `walletBalanceRpcClient.isBucketSufficient(...)` 兩處 call point 上方的 `// TODO (ep-2 plan-8): walletBalanceRpc.isBucketSufficient pending wallet-side impl` 註解(call point 銜接)

**Cross-BC 性質**：本 plan 工作項落於 **wallet bounded context**（非 fund / transfer-intent feature），但收於 fund ep-2 roadmap 編號內、作為 transfer-intent integration 的 cross-BC dependency tracker。Codex 落地時於 wallet feature module / RPC server module 對應路徑展開（`apps/esingpay-cradle/src/wallet/...` —— 具體 path 由 Codex 對位 wallet feature 既有結構）。

**依賴**：無（與 ep-2 plan-1 / plan-2 / plan-6 全部解耦，可從 ep-2 起點即並行起作）

**對位 deadline**：

- Build-time：ep-2 plan-2 phase 2 起作前 contract-rpc method signature 必須 surface（否則 fund 端 `TransferIntentEvaluationService` build fail）
- Caller-time：ep-2 plan-3 起作前 wallet cradle server-side impl 必須 landed（否則 plan-3 submit/quote use case 第一次呼叫 evaluation 即 RPC 失敗）
- 實務上 contract-rpc signature 與 server impl 通常同 PR、實質單一 deadline = **plan-3 起作前**

**Phase 切分方向**（具體留 plan 起草階段）：

- 規模小（mirror 既有 `isWithdrawalSufficient`、單一 method）；預期 1 phase 落地
- contract-rpc + wallet cradle server controller + balance query 同 phase 完成

---

## Dependency Graph

```text
ep-1:
  plan-1 ─┬─→ plan-2 ──→ plan-3
          └─→ plan-4

ep-2:
  plan-1 ─┬─→ plan-2 ─┐
          │           ├─→ plan-3 ─┬─→ plan-4
          │   plan-8 ─┘           ├─→ plan-5
          │                       └─→ plan-7
          └─→ plan-6

ep-1 與 ep-2 之間：無 cross-ep dependency，完全並行
ep-2 plan-8 為 cross-BC（wallet bounded context）；與 plan-1 / plan-2 / plan-6 全部解耦，從 ep-2 起點即可並行起作；為 plan-3 hard prerequisite
```

---

## Parallelism

### Ep 之間

ep-1 與 ep-2 完全並行——TransferIntent 是全新 fund case，schema / entity / use case 與 WithdrawalIntent 改動在檔案層不衝突。Cross-cutting 的 `feePayerWalletId` 概念在 ep-2 plan-1 直接以新欄位形式內建於 TransferIntent schema，與 ep-1 plan-1 對 WithdrawalIntent 的 schema 改動互不依賴。

### Ep 內並行

ep-1：

- plan-1 是 critical path 起點
- plan-1 完成後，plan-2 與 plan-4 可並行起作
- plan-3 等 plan-2 完成

ep-2：

- plan-1 是 critical path 起點
- plan-1 完成後，plan-2 與 plan-6 可並行起作
- plan-8 為 cross-BC、與 plan-1 / plan-2 / plan-6 全部解耦，可從 ep-2 起點即並行起作
- plan-2 完成**且** plan-8 已 landed → plan-3 起作
- plan-3 完成後，plan-4、plan-5 與 plan-7 可並行起作

### Claude Code 任務分配候選

- ep-1：plan-1 → (plan-2 + plan-4) parallel → plan-3
- ep-2：plan-8 為 cross-BC、與 ep-2 主線 plan-1 起點即可並行；主線 plan-1 → (plan-2 + plan-6) parallel；(plan-2 + plan-8) 兩者皆 landed 後 → plan-3 → (plan-4 + plan-5 + plan-7) parallel

實際 Claude Code 工作粒度由各 plan 內 phase / touchpoint 拆分決定，roadmap 不再展開。

---

## Landed Facts Assumed

下列事實為 roadmap 起草前已確認，後續 plan 起草直接套用：

- `WalletAllocationType` 已包含 `Deposit | Withdrawal | Transfer | Adjustment`（Tim 確認 2026-05-13）
- 既有 ledger projection 覆蓋：deposit (platform + merchant) + withdrawal intent (merchant)
- 既有未覆蓋：withdrawal intent (platform) 與 transfer intent
- Cashflow projection 本期不實作
- `feePayerWalletId` 在 WithdrawalIntent 尚未持久化（屬 ep-1 plan-1 範圍）
- TransferIntent 完全不存在於 codebase（屬 ep-2 plan-1 範圍）
- 既有 `WithdrawalIntentFeePayerResolverService` 推導鏈：`senderWalletId → senderWalletRef → networkFeeDelegation → feePayerWallet`，依賴 `NetworkClient.getNetworkFeeDelegationBySenderEndpointId`（TODO 標記的暫時 bridge）
- 既有 batch endpoint pattern 在 codebase 內**不存在**——ep-1 plan-3 / ep-2 plan-5 須 establish 此 pattern（採 sequential fan-out + per-item try/catch，mirror `ExpireReservedAllocationsUseCase`）
- 既有 `WalletLedgerProjectionService` 已對 `FundCaseType.TransferIntent` 預留 switch case（目前丟 Error）——ep-2 plan-6 是「實作既有 placeholder」，非新建 projection 流

---

## Open Points Referenced

下列項目於對應 spec / draft 的 Open Points 追蹤，不在 roadmap 重述：

- `spec/fund/fund-case-expansion-rest-contract-spec.md` Open Points 1~7
- `spec/fund/fund-case-expansion-design-draft.md` Open Points 1~6

---

## Out of Scope

- Wallet allocation 對 transfer 的 path 涵蓋設計（屬 wallet-allocation ep-2 範疇）
- Cashflow projection
- Merchant API 對 transfer 開放（spec open point 4）
- Batch quote endpoint（spec open point 2）
- 跨 fund case 共用 instruction dispatcher abstract interface（封裝 stays per-fund-case，命名對稱、結構同形、不共用程式碼）

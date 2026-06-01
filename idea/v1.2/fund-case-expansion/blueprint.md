---
status: draft
updated_at: 2026-05-13
updated_by: Tim
remark: 本文件未依照 blueprint 規範撰寫，暫供內容參考，等待後續重構改寫。
---

# Fund Case Expansion Blueprint

## Scope

本 blueprint 對應 `./design.md` 與 `./design-rest.md` 的實作落地節奏。涵蓋：

- WithdrawalIntent 擴展（category 引入、platform sender 支援、gather batch、destination_internal 偵測）
- TransferIntent 新建（全新 fund case，含 fast-forward 政策、gather/distribute batch、ledger projection 整合、instruct flow）
- TransferIntent platform endpoint 的 API gateway expose（cross-BC，Stage 2 Phase 9）

不涵蓋：

- 設計層細節——見 `./design.md` / `./design-rest.md`
- 各 stage/phase plan 內部切分與 touchpoint 列表——留 plan 起草階段
- Wallet allocation 對 transfer 的 path 涵蓋設計（屬 wallet-allocation Stage 2 範疇，本 blueprint 不展開）
- Cashflow projection（本期不實作）

---

## Strategy

依 fund case 邊界切分為 Stage 1（WithdrawalIntent 擴展）與 Stage 2（TransferIntent 新建），兩個 stage 之間**完全解耦**、可並行起作。

命名對位：Stage 1 = legacy ep-1、Stage 2 = legacy ep-2；Phase N = legacy plan-N。

Staging 上線節奏：依 stage 整波上 staging。

- **Stage 1**: Phase 1 ~ Phase 4 全部完成 → staging deploy（platform withdrawal intent 完整可用）
- **Stage 2**: Phase 1 ~ Phase 8 全部完成 → staging deploy（transfer intent 完整可用）

---

## Stage / Phase Inventory

### Stage 1: WithdrawalIntent 擴展

#### Stage 1 Phase 1: feePayer 持久化 + Category 整改

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

**依賴**：無（Stage 1 起點）

**Phase 切分方向**（具體留 plan 起草階段細調）：

- Phase 1：持久化骨架（raw / entity / model / migration / repository）
- Phase 2：DTO + REST mapper
- Phase 3：Submit use cases 寫入新欄位
- Phase 4：Instruct use case 改讀 entity + resolver 角色收斂
- Phase 5：Evaluation / Auto-progress service `senderMerchantId === null` 整改 + Instruction dispatcher 抽出 + platform fast-forward path

#### Stage 1 Phase 2: Allocation Line Builder Category-Aware

**範圍**：

- `WithdrawalIntentAllocationLineBuilderService` 依 category 變形 plan lines：
  - `from_merchant`：principal + networkFee pairs；`amountFee > 0` 時另含 serviceFee pair（共 6 lines），`amountFee = 0` 時不 emit serviceFee placeholder（共 4 lines）
  - `from_platform`：4 lines（principal + networkFee × source/destination；無 serviceFee）
- 對應 settlement service 的 outcome builders（`buildCompletedOutcomeLines` 與 `buildNetworkFeeOnlyOutcomeLines`）處理 4-line shape
- Service fee 政策確立：**只有 merchant withdrawal 收 service fee**；platform withdrawal 無此名目

**依賴**：Stage 1 Phase 1（需 entity 內 `category` 已持久化）

#### Stage 1 Phase 3: Gather Batch Endpoint

**範圍**：

- 新增 REST endpoint：`POST /fund/plat/withdrawal-intents/$submit-gather-batch`
- 新增 use case：`PlatformSubmitWithdrawalIntentGatherBatchUseCase`，內部 fan-out N 次 single submit use case
- 既有 single submit use case 不變動——batch use case 僅作 orchestration
- Per-item result 對位（與 input items 同序、同長度）+ issue code mapping
- Per-item issue code enum：`WithdrawalIntentSubmitIssueCode`
- 容量上限 50 筆（超出回 envelope-level `data_invalid`）
- 採既有 codebase fan-out pattern：sequential `for...of` + per-item try/catch（mirror `ExpireReservedAllocationsUseCase`）

**依賴**：Stage 1 Phase 1 + Stage 1 Phase 2（單筆 platform submit 須完整可用）

#### Stage 1 Phase 4: destination_internal 偵測

**範圍**：

- `NetworkEndpointRpc` 擴充：新增 `findByIdentifier(...)` 或 `listByIdentifierIn(...)`（含 repository query filter、facade、RPC controller）
- DB 已有 `identifier` unique constraint，repository 層加 query filter 即可
- `WithdrawalIntentEvaluationService.evaluate(...)` 內呼叫此查找，命中即 issue `destination_internal`
- `WithdrawalIntentQuoteIssueCode.DestinationInternal` 已於 `./design-rest.md` 確立，本 phase 落地實裝
- Envelope-level error code `withdrawal_intent.destination_internal` 暴露

**依賴**：Stage 1 Phase 1（需 evaluation service 已 category-aware）

**並行特性**：與 Phase 2 / Phase 3 解耦深（屬 cross-BC NetworkEndpoint 改動），可與 Phase 2 並行起作，不影響 Phase 2 / Phase 3 路徑。

### Stage 2: TransferIntent 新建

#### Stage 2 Phase 1: 持久化骨架

**範圍**：

- 新建 `TransferIntent` raw / entity / model / migration / repository
- 新建 `TransferIntentStatus` enum（10 值，cancelled / declined / rejected 註解標示「不可用」，過渡狀態註解標示「略過」）
- 新建 `TransferIntentCategory` enum
- Entity transition map（mirror Deposit `recognizedTransitionPath` 模式）：`preparing → reviewing → processing → processed → transacting → completed/failed`
- Entity factory `createSubmitted(...)` 建立 `preparing` draft + structural invariant 校驗（`(category, bucket, network, feePayer)` 四項）；**不**執行 fast-forward
- `submittedTransitionPath` 常數於 entity 檔暴露（作為後續 phase AutoProgressService 的 transition 序列契約來源）
- Entity action methods：`instruct(...)` / `handleSettled(...)` / `handleFailed(...)`（不含 cancel / reject / release / accept / approve / decline）
- 新建 `transferIntentRpc.getById`（mirror `withdrawalIntentRpc` 結構，採 `ResultDto<CommonCode.Ok, TransferIntentDto | null>` 形狀）

**依賴**：無（Stage 2 起點，與 Stage 1 完全解耦）

#### Stage 2 Phase 2: Foundation — Services + Contract-rest types + Queue infra

**範圍**：

- contract-rest 端 type 落地（submit/quote 範圍）：
  - `PlatformSubmitTransferIntentDto`、`TransferIntentQuoteDto`、`TransferIntentDto`（rich shape）
  - `TransferIntentQuoteIssueCode` enum、`TransferIntentSubmitIssueCode` enum（submit issue code 由 Phase 5 batch 端首次使用、本 phase 先 surface 對位）
  - `TransferIntentCode` REST envelope error code enum + 對應 `.message.ts` / `.status.ts`
  - `plat-transfer-intent.api.ts` 內含 `submitTransferIntent` + `quoteTransferIntent` 兩 endpoint definition（search / getById 屬 Phase 4）
- Application services：
  - `TransferIntentEvaluationService`（quote 邏輯 + same-network 不變式 + wallet_circular 檢查 + balance / config 檢查；balance 經由 `walletBalanceRpc.isBucketSufficient`）
  - `TransferIntentAllocationLineBuilderService`（依 category 構造 plan lines，含 `merchant_to_platform` 的 delegated feePayerWallet）
  - `TransferIntentFeePayerResolverService`（命名對稱 withdrawal；TODO 條目記錄「考慮與 `WithdrawalIntentFeePayerResolverService` 抽共用」）
  - `TransferIntentAutoProgressService`（單一同步 `progress({ entity, occurredAt })` method、unconditional fast-forward 至 `processed`；對 entity 依 `submittedTransitionPath` 依序 transitions、append statusHistory、設 `processedAt`）
  - **沿用** 既有 `WithdrawalNetworkFeeEstimationService`——該 service 的 "Withdrawal" 為 behavior scope（任何 sender 對 external network 發 transaction），非 fund case scope；transfer sender 同樣在做 withdrawal 之事，直接注入既有 service、不建 transfer-specific estimation service
- Queue infra：
  - `TransferIntentInstructionDispatcherService`（命名對稱 withdrawal dispatcher）
  - `TransferIntentInstructionQueueDispatcher` + queue definition（mirror withdrawal 既有 queue pattern；worker 屬 Phase 7）
- Module wiring 擴張（providers 增 services；export 限制依既有 feature module 政策對位）

**依賴**：Stage 2 Phase 1

**Cross-BC 依賴**：本 phase 引入對 `walletBalanceRpc.isBucketSufficient({ walletId, currencyCode, amount, bucket })` 的依賴，由 **Stage 2 Phase 8** 承接落地（mirror 既有 `walletBalanceRpc.isWithdrawalSufficient` 結構）。Phase 2 evaluation service 直接照預期 signature 呼叫、caller 位置留 TODO 註解。Build-time 對位：Phase 8 contract-rpc method signature 必須於 Stage 2 Phase 2 plan 內 evaluation service 起作前 surface（否則 evaluation service build fail）。

#### Stage 2 Phase 3: Submit/Quote wiring

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

**依賴**：Stage 2 Phase 2、Stage 2 Phase 8（Phase 8 wallet-side `isBucketSufficient` server-side impl 必須於 Stage 2 Phase 3 起作前 landed，否則 submit/quote use case 第一次呼叫 evaluation 即 RPC 失敗）

#### Stage 2 Phase 4: Read path — Search + GetById

**範圍**：

- contract-rest 端擴張：`plat-transfer-intent.api.ts` 加 `searchTransferIntents` + `getTransferIntentById` 兩 endpoint definition；`PlatformSearchTransferIntentParamsDto`
- `PlatformSearchTransferIntentsUseCase`、`PlatformGetTransferIntentUseCase`
- Repository 擴張：`search(query)`、`listByIdIn(ids)`（composer 批次 hydrate）
- Query service 擴張：`search(input)`、`listRefsByIdIn(input)`
- REST controller 擴張：`searchTransferIntents` + `getTransferIntentById` 兩 method（reuse Phase 3 composer + REST mapper）

**依賴**：Stage 2 Phase 3

#### Stage 2 Phase 5: Gather + Distribute Batch

**範圍**：

- 兩個 batch endpoint：
  - `POST /fund/plat/transfer-intents/$submit-gather-batch`
  - `POST /fund/plat/transfer-intents/$submit-distribute-batch`
- 兩個 use case：`PlatformSubmitTransferIntentGatherBatchUseCase` / `PlatformSubmitTransferIntentDistributeBatchUseCase`，內部 fan-out N 次 single submit
- 容量上限 50 筆（與 withdrawal 共用相同 default）
- 採與 Stage 1 Phase 3 相同的 fan-out pattern
- Per-item issue code mapping：use case 內 private method，將 single submit error 翻譯為 `TransferIntentSubmitIssueCode`（enum 由 Phase 2 落地）

**依賴**：Stage 2 Phase 3

#### Stage 2 Phase 6: Ledger Projection 整合

**範圍**：

- `WalletLedgerProjectionService` 注入 `transferIntentRpc`
- `fetchFundCaseNetworkTransactionId` 內 `FundCaseType.TransferIntent` switch case 從 TODO 改為實作
- 新增 `parseTransferIntentNetworkTransactionId` helper（mirror `parseDepositNetworkTransactionId` / `parseWithdrawalIntentNetworkTransactionId`）
- 確認既有 `WalletLedgerProjectionService` 的 plan / outcome line 處理對 transfer 三 category（含 `merchant_to_platform` 的 delegated feePayer line）正確 cover——projection 跨 fund case agnostic，預期無額外改動

**依賴**：Stage 2 Phase 1（`transferIntentRpc.getById` 已存在）

**並行特性**：與 Stage 2 Phase 2 / Phase 3 / Phase 4 / Phase 5 / Phase 7 解耦深（屬 wallet-side ledger 改動），可於 Stage 2 Phase 1 plan 內部第 2 階段完成後即起作，不影響其他 Stage 2 phase 路徑。

#### Stage 2 Phase 7: Instruct + handleSettled + handleFailed + Worker + Backfill

**範圍**：

- `InstructTransferIntentUseCase`（mirror `InstructWithdrawalIntentUseCase`；直接讀 entity `feePayerWalletId`，無 resolver path）
- `HandleNetworkTransactionSettledUseCase` for transfer（觸發 `entity.handleSettled(...)` + settle allocation completed outcome）
- `HandleNetworkTransactionFailedUseCase` for transfer（觸發 `entity.handleFailed(...)` + settle allocation network-fee-only outcome）
- `TransferIntentAllocationSettlementService`（zero / completed / networkFeeOnly outcome，mirror withdrawal 既有 service）
- Queue worker：`TransferIntentInstructionQueueWorker`（mirror withdrawal 既有 worker；queue definition + dispatcher 已於 Phase 2 落地）
- `BackfillDueTransferIntentInstructionUseCase`（mirror `backfill-due-withdrawal-intent-instruction.use-case.ts`，補救 instruction dispatch 失敗）

**依賴**：Stage 2 Phase 3（worker 需 submit use case 建立的 row 來處理；end-to-end 驗證需要寫入路徑存在）

#### Stage 2 Phase 8: Wallet-side `isBucketSufficient` RPC

**範圍**：

- contract-rpc 端：`walletBalanceRpc.isBucketSufficient({ walletId, currencyCode, amount, bucket }): ResultDto<CommonCode.Ok, { sufficient: boolean; bucketBalance: NumericString }>` method signature（mirror 既有 `walletBalanceRpc.isWithdrawalSufficient` 的 lookup intent，但採 RPC envelope）
- wallet cradle 端：`WalletBalanceRpcController.isBucketSufficient(...)` server-side impl；依 `bucket` 參數查 wallet 對應 bucket（`available` / `locked` / `held` / `held_locked`）的 balance、判斷是否足以支付 `amount`，並回傳該 bucket 當前 balance
- Mirror 既有 `isWithdrawalSufficient` query 路徑與 model lookup；return shape 依 Stage 2 Phase 8 plan 文件收斂為 envelope + `bucketBalance`
- 落地時**同步移除** fund 端 Stage 2 Phase 2 `TransferIntentEvaluationService` 內 `walletBalanceRpcClient.isBucketSufficient(...)` 兩處 call point 上方的 `// TODO (Stage 2 Phase 8): walletBalanceRpc.isBucketSufficient pending wallet-side impl` 註解(call point 銜接)

**Cross-BC 性質**：本 phase 工作項落於 **wallet bounded context**（非 fund / transfer-intent feature），但收於 fund Stage 2 blueprint 編號內、作為 transfer-intent integration 的 cross-BC dependency tracker。Codex 落地時於 wallet feature module / RPC server module 對應路徑展開（`apps/esingpay-cradle/src/wallet/...` —— 具體 path 由 Codex 對位 wallet feature 既有結構）。

**依賴**：無（與 Stage 2 Phase 1 / Phase 2 / Phase 6 全部解耦，可從 Stage 2 起點即並行起作）

**對位 deadline**：

- Build-time：Stage 2 Phase 2 plan 內 evaluation service 起作前 contract-rpc method signature 必須 surface（否則 fund 端 `TransferIntentEvaluationService` build fail）
- Caller-time：Stage 2 Phase 3 起作前 wallet cradle server-side impl 必須 landed（否則 Phase 3 submit/quote use case 第一次呼叫 evaluation 即 RPC 失敗）
- 實務上 contract-rpc signature 與 server impl 通常同 PR、實質單一 deadline = **Phase 3 起作前**

**Phase 切分方向**（具體留 plan 起草階段）：

- 規模小（mirror 既有 `isWithdrawalSufficient`、單一 method）；預期 1 phase 落地
- contract-rpc + wallet cradle server controller + balance query 同 phase 完成

#### Stage 2 Phase 9: Gateway REST Expose（platform）

**範圍**：

- 於 API gateway 對齊 `platTransferIntentApi` 的 **6 個 platform endpoint**——`submitTransferIntent` / `quoteTransferIntent` / `searchTransferIntents` / `getTransferIntentById` / `submitTransferIntentGatherBatch` / `submitTransferIntentDistributeBatch`
- gateway 端新建：`PlatTransferIntentProxy`（包 `platTransferIntentApi` 6 method 的 rest-rpc 轉發）、`PlatTransferIntentController`（HTTP route → proxy，含 permission decorator）、對應 request body / query DTO
- `FundRestModule` wiring 擴張（註冊新 proxy + controller）
- 全部 mirror gateway 既有 `PlatWithdrawalIntentProxy` / `PlatWithdrawalIntentController` pattern，純機械對位、薄轉發層、不引入新 precedent

**不涵蓋**：

- merchant 端 gateway expose——呼應 `./design-rest.md` Open Point 4「Merchant API 對 transfer 開放與否目前不開放」；本期 transfer 不對 merchant API 開放主動發起，gateway 亦不需 merchant transfer proxy
- `cancelTransferIntentById` / `rejectTransferIntentById` / `releaseTransferIntentById`——cradle 端本就未實作（Stage 2 Phase 1 entity action methods 明確排除 cancel / reject / release），故 gateway 不對齊此三 endpoint（`./design-rest.md` Endpoint summary 雖列出，屬未來範疇）

**Cross-BC 性質**：本 phase 工作項落於 **api-gateway bounded context**（`apps/esing-pay-api-gateway/src/rest/fund/...`），非 fund / transfer-intent cradle feature；與 Stage 2 Phase 8（wallet bounded context）並列為 cross-BC tracker，收於 fund Stage 2 blueprint 編號內、作為 transfer-intent 對外可用的最後一塊。

**依賴**：Stage 2 Phase 3（submit / quote contract 定型）+ Phase 4（search / getById）+ Phase 5（gather / distribute batch）。gateway proxy 依賴全部 6 個 contract endpoint 皆已定型，結構上必落於所有 contract phase 之後——非並行 phase。

**Phase 切分方向**（具體留 plan 起草階段）：

- 規模小（mirror 既有 gateway withdrawal proxy / controller、薄轉發層）；預期 1 phase 落地
- proxy + controller + request DTO + module wiring 同 phase 完成

---

## Delivery Conventions

下列交付慣例為跨 phase 通則，使既有產物（smoke test 等）的「因」可溯，避免後續誤判為遺漏：

### Smoke test 隨 feature inline 交付

每個 cradle fund case feature 落地時，submit-path happy 的 smoke test **隨該 feature 的 submit / wiring phase inline 交付，不另立 phase**。Smoke test 附著於 submit/wiring phase（具備 submit 寫入路徑後即可驗證 happy path），不具獨立 phase 的並行/依賴邊界性質，故不在 stage/phase inventory 中單列編號。

- TransferIntent 對應 **Stage 2 Phase 3**（Submit/Quote wiring），覆蓋已實作 category 的 submit-path happy path（`platform_to_platform`、`merchant_to_platform`）
- WithdrawalIntent 對應 Stage 1 Phase 3（platform submit path）同理
- gateway 薄轉發層（Stage 2 Phase 9）依既有 gateway 慣例不配 cradle 式 submit-path smoke

---

## Dependency Graph

```text
Stage 1:
  Phase 1 ─┬─→ Phase 2 ──→ Phase 3
          └─→ Phase 4

Stage 2:
  Phase 1 ─┬─→ Phase 2 ─┐
          │           ├─→ Phase 3 ─┬─→ Phase 4 ─┐
          │   Phase 8 ─┘           ├─→ Phase 5 ─┼─→ Phase 9
          │                       └─→ Phase 7   │
          └─→ Phase 6                           （Phase 9 依賴 Phase 3 + 4 + 5）

Stage 1 與 Stage 2 之間：無 cross-stage dependency，完全並行
Stage 2 Phase 8 為 cross-BC（wallet bounded context）；與 Phase 1 / Phase 2 / Phase 6 全部解耦，從 Stage 2 起點即可並行起作；為 Phase 3 hard prerequisite
Stage 2 Phase 9 為 cross-BC（api-gateway bounded context）；依賴 Phase 3 + 4 + 5 全部 landed（6 endpoint contract 全定型），結構上落於所有 contract phase 之後，非並行 phase
```

---

## Parallelism

### Stage 之間

Stage 1 與 Stage 2 完全並行——TransferIntent 是全新 fund case，schema / entity / use case 與 WithdrawalIntent 改動在檔案層不衝突。Cross-cutting 的 `feePayerWalletId` 概念在 Stage 2 Phase 1 直接以新欄位形式內建於 TransferIntent schema，與 Stage 1 Phase 1 對 WithdrawalIntent 的 schema 改動互不依賴。

### Stage 內並行

Stage 1：

- Phase 1 是 critical path 起點
- Phase 1 完成後，Phase 2 與 Phase 4 可並行起作
- Phase 3 等 Phase 2 完成

Stage 2：

- Phase 1 是 critical path 起點
- Phase 1 完成後，Phase 2 與 Phase 6 可並行起作
- Phase 8 為 cross-BC、與 Phase 1 / Phase 2 / Phase 6 全部解耦，可從 Stage 2 起點即並行起作
- Phase 2 完成**且** Phase 8 已 landed → Phase 3 起作
- Phase 3 完成後，Phase 4、Phase 5 與 Phase 7 可並行起作
- Phase 9 為 cross-BC（api-gateway）、非並行：依賴 Phase 3 + 4 + 5 全部 landed（6 endpoint contract 全定型），落於所有 contract phase 之後

### Claude Code 任務分配候選

- Stage 1：Phase 1 → (Phase 2 + Phase 4) parallel → Phase 3
- Stage 2：Phase 8 為 cross-BC、與 Stage 2 主線 Phase 1 起點即可並行；主線 Phase 1 → (Phase 2 + Phase 6) parallel；(Phase 2 + Phase 8) 兩者皆 landed 後 → Phase 3 → (Phase 4 + Phase 5 + Phase 7) parallel

實際 Claude Code 工作粒度由各 stage/phase plan 內部切分與 touchpoint 決定，blueprint 不再展開。

---

## Landed Facts Assumed

下列事實為 blueprint 起草前已確認，後續 plan 起草直接套用：

- `WalletAllocationType` 已包含 `Deposit | Withdrawal | Transfer | Adjustment`（Tim 確認 2026-05-13）
- 既有 ledger projection 覆蓋：deposit (platform + merchant) + withdrawal intent (merchant)
- 既有未覆蓋：withdrawal intent (platform) 與 transfer intent
- Cashflow projection 本期不實作
- `feePayerWalletId` 在 WithdrawalIntent 尚未持久化（屬 Stage 1 Phase 1 範圍）
- TransferIntent 完全不存在於 codebase（屬 Stage 2 Phase 1 範圍）
- 既有 `WithdrawalIntentFeePayerResolverService` 推導鏈：`senderWalletId → senderWalletRef → networkFeeDelegation → feePayerWallet`，依賴 `NetworkClient.getNetworkFeeDelegationBySenderEndpointId`（TODO 標記的暫時 bridge）
- 既有 batch endpoint pattern 在 codebase 內**不存在**——Stage 1 Phase 3 / Stage 2 Phase 5 須 establish 此 pattern（採 sequential fan-out + per-item try/catch，mirror `ExpireReservedAllocationsUseCase`）
- 既有 `WalletLedgerProjectionService` 已對 `FundCaseType.TransferIntent` 預留 switch case（目前丟 Error）——Stage 2 Phase 6 是「實作既有 placeholder」，非新建 projection 流

---

## Open Points Referenced

下列項目於 `./design-rest.md` / `./design.md` 的 Open Points 追蹤，不在 blueprint 重述：

- `./design-rest.md` Open Points 1~7
- `./design.md` Open Points 1~6

---

## Out of Scope

- Wallet allocation 對 transfer 的 path 涵蓋設計（屬 wallet-allocation Stage 2 範疇）
- Cashflow projection
- Merchant API 對 transfer 開放（`./design-rest.md` Open Point 4）——含 merchant 端 gateway expose
- TransferIntent `cancel` / `reject` / `release` endpoint 的 gateway expose（cradle 端 Stage 2 Phase 1 已排除此三 entity action，gateway 亦不對齊）
- Batch quote endpoint（`./design-rest.md` Open Point 2）
- 跨 fund case 共用 instruction dispatcher abstract interface（封裝 stays per-fund-case，命名對稱、結構同形、不共用程式碼）

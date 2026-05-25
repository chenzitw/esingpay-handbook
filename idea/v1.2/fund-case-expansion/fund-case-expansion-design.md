---
status: draft
updated_at: 2026-05-13
updated_by: Tim
---

# Fund Case Expansion Design Draft

## Scope

本 spec 範圍：

- 既有 `WithdrawalIntent` 擴展支援 platform treasury sender，引入 `category` 區分子業務
- 新建 `TransferIntent` fund case，含三 category
- 跨 fund case 的方向不變式 (directional invariants)
- 對應的 evaluation / validation service 整改

不在本 spec 範圍：

- Implementation phasing / migration ordering（留 impl plan）
- Cross-cutting `feePayerWalletId` 欄位的持久化改造（先於本 spec 主線，排在 impl ep-1 plan-1 第一部分）
- Wallet-side 改動——零改動。`WalletAllocationType.Transfer` 已存在；`WalletAllocationLine` 的 `Internal | External` discriminated union 已 accommodate transfer 場景
- Ledger / cashflow read model 調整——既有 `wallet-allocation-design` 已涵蓋
- 批次 UI 的後台對帳 / 顯示分組——批次不入 domain，無 `batchCorrelationId` 持久化

---

## Fund Case 方向不變式

esingpay 三大 fund case 依資金流向結構性區隔：

| Fund case          | sender 空間 | recipient 空間 |
| ------------------ | ----------- | -------------- |
| `Deposit`          | external    | internal       |
| `WithdrawalIntent` | internal    | external       |
| `TransferIntent`   | internal    | internal       |

**方向不變式** 是 entity 邊界對方向性偽裝的拒絕：fund case 必須由結構性事實證明自己屬於哪一型，不能用命名掩護方向錯置。具體三條：

| Fund case          | 方向不變式                                                                                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Deposit`          | recipient 必須是已知 internal wallet endpoint（由 deposit recognition 既有機制保證）                                                                         |
| `WithdrawalIntent` | destination identifier 必須 **不** 對應任何已知 internal network endpoint                                                                                    |
| `TransferIntent`   | sender 與 recipient 都必須是已知 internal wallet（by `walletId` 引用，by construction）；且兩端 network endpoint 必須屬於同網路（見「Same-network 不變式」） |

違反時 entity 拒絕建立，或於 evaluation service 命中 issue。

執行位置：

- `Deposit`：既有 recognition service / `Wallet` lookup 已 cover
- `WithdrawalIntent`：`WithdrawalIntentEvaluationService.evaluate(...)` 內新增 destination identifier 對 `NetworkEndpoint` 的反查，命中即 issue `destination_internal`（新增 issue code，三層碼詳見「Destination internal-endpoint 拒收」段）。Lookup 性能優化（identifier index 等）屬實作細節，不在本 spec 範圍
- `TransferIntent`：entity factory `createSubmitted(...)`、wallet 引用解析、與同網路 invariant 各自校驗

此原則的意義：避免「業務上是內部移動但會計上記為 withdrawal」這類語義鴻溝，保證 ledger 與業務事實一致。

---

## TransferIntent

### Raw shape

```ts
export interface TransferIntent {
  id: bigint;
  walletAllocationId: bigint;
  category: TransferIntentCategory;

  /** present if the sender is a merchant */
  senderMerchantId: Uuid | null;
  senderWalletId: bigint;
  senderNetworkEndpointId: bigint;
  senderBucket: WalletBucket.Available | WalletBucket.Held;

  /** present if the recipient is a merchant */
  recipientMerchantId: Uuid | null;
  recipientWalletId: bigint;
  recipientNetworkEndpointId: bigint;
  recipientBucket: WalletBucket.Available | WalletBucket.Held;

  /** wallet that pays network fee; equals senderWalletId for sender-self-paid, or platform delegated wallet */
  feePayerWalletId: bigint;

  /** present after a real network transaction is created */
  targetNetworkTransactionId: bigint | null;

  currencyCode: CurrencyCode;
  amount: NumericString;

  status: TransferIntentStatus;
  submittedBy: Actor;
  submittedAt: Date;

  processedAt: Date | null;
  instructedAt: Date | null;
  finalizedAt: Date | null;

  statusHistory: TransferIntentStatusHistoryItem[];

  createdAt: Date;
  updatedAt: Date;
}
```

Per-field 對 `WithdrawalIntent` raw 對照：

| 欄位                                           | 對 Withdrawal 的關係                                         | 設計依據                                                                 |
| ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------ |
| `id`                                           | 同                                                           | 同型                                                                     |
| `walletAllocationId`                           | 同                                                           | 一個 transfer intent 對應一個 wallet allocation                          |
| `category`                                     | **新增**                                                     | `TransferIntentCategory` 三值 enum                                       |
| `senderMerchantId`                             | 同                                                           | merchant sender 時非 null；audit / query marker                          |
| `senderWalletId`                               | 同                                                           |                                                                          |
| `senderNetworkEndpointId`                      | 同                                                           |                                                                          |
| `senderBucket`                                 | **新增**                                                     | transfer 兩端 bucket 都顯式（withdrawal 隱含 sender bucket = available） |
| `recipientMerchantId`                          | **新增**，sender 對稱                                        |                                                                          |
| `recipientWalletId`                            | **取代** withdrawal 的 `destination`                         | recipient 是內部 wallet，不需 external descriptor                        |
| `recipientNetworkEndpointId`                   | **新增**，sender 對稱                                        | 兩端 endpoint 都儲存供 audit 與 same-network invariant 用                |
| `recipientBucket`                              | **新增**，sender 對稱                                        |                                                                          |
| `feePayerWalletId`                             | **新增**（withdrawal 也應補上，見「WithdrawalIntent 對齊」） | 由 category 政策決定；解決 submit-time 與 instruct-time 不一致問題       |
| `targetNetworkTransactionId`                   | 同                                                           | transfer 是 onchain 動作                                                 |
| `destination`                                  | **不存在**                                                   | transfer 不對外，無 external descriptor                                  |
| `currencyCode`                                 | 同                                                           | principal 幣別                                                           |
| `amount`                                       | 同                                                           | principal 金額                                                           |
| `amountNet` / `amountFee`                      | **不存在**                                                   | transfer 不收 service fee（見 Open Points）                              |
| `status`                                       | 同型不同 enum                                                | `TransferIntentStatus` 自有 enum，不共用                                 |
| `submittedBy`                                  | 同                                                           |                                                                          |
| `submittedAt`                                  | 同                                                           |                                                                          |
| `cancellableUntil`                             | **不存在**                                                   | transfer 不引入此機制；cancel 政策純靠 status + actor 權限               |
| `processedAt` / `instructedAt` / `finalizedAt` | 同                                                           |                                                                          |
| `statusHistory`                                | 同型                                                         | item 型別為 `TransferIntentStatusHistoryItem`                            |
| `createdAt` / `updatedAt`                      | 同                                                           |                                                                          |

### Category enum

```ts
export enum TransferIntentCategory {
  PlatformToPlatform = 'platform_to_platform',
  PlatformToMerchant = 'platform_to_merchant',
  MerchantToPlatform = 'merchant_to_platform',
}
```

採 `<sender>_to_<recipient>` 命名。subject-first 排序——alphabetical sort 後 sender 即排序錨點。

「`platform`」於本 spec 命名脈絡下指 `platform_treasury` wallet type（platform 擁有的 wallet 目前只有此型），與 `merchant` 對稱不寫全名。若未來引入新 platform-owned wallet 類型，再評估精確化命名。

### Category × bucket × fee payer 不變式

Entity factory `createSubmitted(...)` 強制校驗：

| Category               | senderBucket | recipientBucket | feePayer 政策                                                           |
| ---------------------- | ------------ | --------------- | ----------------------------------------------------------------------- |
| `platform_to_platform` | available    | available       | feePayer = sender wallet                                                |
| `platform_to_merchant` | available    | held            | feePayer = sender wallet                                                |
| `merchant_to_platform` | held         | available       | feePayer = delegated platform wallet（透過 wallet fee delegation 查詢） |

違反任一條 throw entity error（具體 error type 命名於 entity 落地時收齊）。

此 invariant table 為當前 PM 規格快照；未來規格演化可能擴張 category enum、放寬組合、或引入 sub-category——以 `spec/open-points.md` 對應條目追蹤。

### Same-network 不變式

兩端 wallet 各自的 `networkEndpoint` 推導出來的 `(networkType, networkProvider)` pair 必須**完全相等**：

```text
sender.networkEndpoint    → (networkType_s, networkProvider_s)
recipient.networkEndpoint → (networkType_r, networkProvider_r)

invariant:  networkType_s === networkType_r
         && networkProvider_s === networkProvider_r
```

判斷準據以 `NetworkTransaction` 上的 `type` 與 `provider` 兩欄為對齊基礎——也就是「若以此 endpoint 發 transaction，其產生的 `NetworkTransaction.type` 與 `.provider` 為何」。

違反時 entity 拒絕建立。

實作層面：從 `NetworkEndpoint` 取得 `(type, provider)` 的具體 derivation path（直接欄位 / reference 查表 / 其他）為實作細節，依 `NetworkEndpoint` 內部結構承接。本 spec 不規約 derivation 的程式表面。

### Status machine

本期 TransferIntent 採 **fast-forward 政策**：submit 即一次性快進至 `processed`，無 review / cancel / reject 流程。`cancelled` / `declined` / `rejected` 三狀態於 enum 保留（維持與其他 fund case 結構對稱，沿用 Deposit 註解標示模式），但**不可達**——entity transition map 不收錄這些 branch。

狀態機 mirror `WithdrawalIntent` 全集，dead branches 以 `[dead]` 標示：

```text
preparing → reviewing → processing → processed → transacting → completed
         ↘ cancelled  ↘ cancelled / declined ↘ rejected      ↘ failed
           [dead]       [dead]    [dead]       [dead]
```

`failed` 為 `transacting` 的合法終態（onchain 失敗），不屬於 dead branches。

Entity transition map（mirror Deposit `recognizedTransitionPath` 模式）：

```ts
const statusTransitionMap: Record<TransferIntentStatus, TransferIntentStatus[]> = {
  [TransferIntentStatus.Preparing]: [TransferIntentStatus.Reviewing],
  [TransferIntentStatus.Cancelled]: [],
  [TransferIntentStatus.Reviewing]: [TransferIntentStatus.Processing],
  [TransferIntentStatus.Declined]: [],
  [TransferIntentStatus.Processing]: [TransferIntentStatus.Processed],
  [TransferIntentStatus.Rejected]: [],
  [TransferIntentStatus.Processed]: [TransferIntentStatus.Transacting],
  [TransferIntentStatus.Failed]: [],
  [TransferIntentStatus.Transacting]: [TransferIntentStatus.Completed, TransferIntentStatus.Failed],
  [TransferIntentStatus.Completed]: [],
};

const finalizedStatuses = [TransferIntentStatus.Completed, TransferIntentStatus.Failed];

const submittedTransitionPath: TransferIntentStatus[] = [
  TransferIntentStatus.Reviewing,
  TransferIntentStatus.Processing,
  TransferIntentStatus.Processed,
];
```

`submittedTransitionPath` 為 submit 一次性快進路徑（mirror Deposit `recognizedTransitionPath`）。Submit 階段 entity 行為分兩步：

1. **Entity factory `createSubmitted(...)`** 建立 draft，status = `preparing`、statusHistory 含 1 筆 `preparing` 紀錄（actor 為 submitter）。Factory 僅執行 structural invariant 校驗，**不**執行 fast-forward。
2. **`TransferIntentAutoProgressService.progress(...)`**（同步、in-memory）依 `submittedTransitionPath` 對 entity 依序呼叫 `transitionTo(...)`，append 三筆 system actor statusHistory，最後設 `processedAt = submittedAt`。

Submit use case orchestration 為：`createSubmitted(...)` → `autoProgressService.progress(...)` → persist。Service 公開 surface 為單一同步方法，不持久化、不發 RPC。

完整 submit 後 statusHistory 累積四筆：

| sequence | status       | actor                      | 由誰寫入                              | 註記                                                    |
| -------- | ------------ | -------------------------- | ------------------------------------- | ------------------------------------------------------- |
| 0        | `preparing`  | submitter（platform user） | entity factory `createSubmitted(...)` | draft 初始狀態                                          |
| 1        | `reviewing`  | system                     | `autoProgressService.progress(...)`   | fast-forward 過渡                                       |
| 2        | `processing` | system                     | `autoProgressService.progress(...)`   | fast-forward 過渡                                       |
| 3        | `processed`  | system                     | `autoProgressService.progress(...)`   | fast-forward 終止；`processedAt` 設為等於 `submittedAt` |

可達性：三 category 共用同一條快進路徑——可達狀態為 `{preparing, reviewing, processing, processed, transacting, completed, failed}`；不可達狀態為 `{cancelled, declined, rejected}`。

Entity action methods 範圍亦對應收斂：

| Method                                                             | 存在於 entity | 註記                                                                                                                                |
| ------------------------------------------------------------------ | ------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `static createSubmitted(...)`                                      | ✓             | factory，建立 `preparing` draft + structural invariant 校驗；fast-forward 由 `TransferIntentAutoProgressService.progress(...)` 承擔 |
| `instruct(...)`                                                    | ✓             | `processed → transacting`，由 instruct use case 觸發                                                                                |
| `handleSettled(...)`                                               | ✓             | `transacting → completed`，由 network handler 觸發                                                                                  |
| `handleFailed(...)`                                                | ✓             | `transacting → failed`，由 network handler 觸發                                                                                     |
| `accept` / `approve` / `release` / `cancel` / `reject` / `decline` | **不存在**    | 對應 status path 不可達，entity 不暴露這些 action                                                                                   |

未來若 PM 規格引入合規審核、人工撤銷或拒絕需求，需重新引入相應 status path、entity action methods、與 lifecycle endpoint，並重新評估 `TransferIntentAutoProgressService` 政策表（見 Open Points）。

### Allocation plan lines per category

每筆 `TransferIntent` 對應一個 `WalletAllocation`，`type = WalletAllocationType.Transfer`。

Plan lines 表達 logical「sender bucket → recipient bucket」流向；reservation flow 中實際 bucket 物理轉換（如 `held → held_locked`、`held_locked → external`）由 wallet 端 allocation lifecycle 處理，不在 plan lines 表面呈現。

以下表格 column 表面（side / space / wallet / bucket / currency / amount / kind）沿用 `wallet-allocation-design` §11 約定，本 spec 不重複解釋。

**`platform_to_platform`**

業務條件：sender platform treasury 對 recipient platform treasury 跨 endpoint 傳輸 X amount 主幣（如 USDT），sender 自付 network fee Y（如 TRX）。

| side        | space    | wallet    | bucket    | currency | amount | kind        |
| ----------- | -------- | --------- | --------- | -------- | ------ | ----------- |
| source      | internal | sender    | available | USDT     | X      | principal   |
| destination | internal | recipient | available | USDT     | X      | principal   |
| source      | internal | sender    | available | TRX      | Y      | network_fee |
| destination | external | —         | —         | TRX      | Y      | network_fee |

**`platform_to_merchant`**

業務條件：sender platform treasury 對 recipient merchant wallet held 注入 X amount 主幣（典型用途：TRX network fee 預存），sender 自付 network fee Y。

| side        | space    | wallet    | bucket    | currency | amount | kind        |
| ----------- | -------- | --------- | --------- | -------- | ------ | ----------- |
| source      | internal | sender    | available | USDT     | X      | principal   |
| destination | internal | recipient | held      | USDT     | X      | principal   |
| source      | internal | sender    | available | TRX      | Y      | network_fee |
| destination | external | —         | —         | TRX      | Y      | network_fee |

**`merchant_to_platform`**

業務條件：sender merchant held 對 recipient platform treasury available 歸集 X amount（典型用途：service fee 歸集），network fee Y 由 delegated platform wallet 支付。

| side        | space    | wallet         | bucket    | currency | amount | kind        |
| ----------- | -------- | -------------- | --------- | -------- | ------ | ----------- |
| source      | internal | sender         | held      | USDT     | X      | principal   |
| destination | internal | recipient      | available | USDT     | X      | principal   |
| source      | internal | feePayerWallet | available | TRX      | Y      | network_fee |
| destination | external | —              | —         | TRX      | Y      | network_fee |

設計觀察：

- 三 category 的 principal 都是 internal-to-internal（與 withdrawal 的 internal-to-external 不同）
- network fee 端統一是 internal source → external destination（onchain 消耗 gas，destination 為鏈本身）
- `merchant_to_platform` 唯一引入 feePayerWallet（非 sender wallet）作為 fee 來源
- `service_fee` kind 不出現於任何 transfer plan lines

### Use case 初始範圍

對應 `WithdrawalIntent` 既有 use case 結構，但因 fast-forward 政策（見 Status machine 段），review / cancel / reject / acceptance queue 相關 use case 全部不存在：

| Use case                                 | 角色                                                          | 對應 WithdrawalIntent            | 備註                                                                                                                                                                             |
| ---------------------------------------- | ------------------------------------------------------------- | -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PlatformSubmitTransferIntentUseCase`    | platform user 主動發起                                        | `PlatformSubmitWithdrawalIntent` | 三 category 共用此 use case；category 由 sender + recipient wallet type 自動推導。submit orchestration 呼叫 `TransferIntentAutoProgressService.progress(...)` 快進至 `processed` |
| `InstructTransferIntentUseCase`          | 把 `processed` 推進到 `transacting`，發出 network transaction | `InstructWithdrawalIntent`       |                                                                                                                                                                                  |
| `HandleNetworkTransactionSettledUseCase` | 處理 onchain 成功                                             | 同名                             | transfer 版本                                                                                                                                                                    |
| `HandleNetworkTransactionFailedUseCase`  | 處理 onchain 失敗                                             | 同名                             | transfer 版本                                                                                                                                                                    |

對應 `WithdrawalIntent` 但本期 **不存在** 的 use cases：

| 對應 WithdrawalIntent use case      | 不存在原因                                                                |
| ----------------------------------- | ------------------------------------------------------------------------- |
| `MerchantSubmitWithdrawalIntent`    | transfer 不對 merchant API 開放主動發起                                   |
| `PlatformCancelWithdrawalIntent`    | 無 cancel endpoint；`cancelled` status 不可達                             |
| `PlatformRejectWithdrawalIntent`    | 無 reject endpoint；`rejected` status 不可達                              |
| `PlatformReleaseWithdrawalIntent`   | 無 release endpoint；`processed` 由 submit fast-forward 直達              |
| `AcceptWithdrawalIntent`            | 無 review flow；`reviewing` 為 fast-forward 過渡狀態，不暴露給外部 caller |
| `ApproveWithdrawalIntent`           | 同上；`processing` 為 fast-forward 過渡狀態                               |
| `ProcessWithdrawalIntentAcceptance` | 無 acceptance queue 流程                                                  |

注意事項：

- `PlatformSubmitTransferIntentUseCase` 涵蓋三 category，REST endpoint 設計留實作 plan 決定
- Category 自動推導規則：

  ```text
  (senderWalletType, recipientWalletType)
    (platform_treasury, platform_treasury) → platform_to_platform
    (platform_treasury, merchant_wallet)   → platform_to_merchant
    (merchant_wallet,   platform_treasury) → merchant_to_platform
  ```

  不符合任一組合 entity 拒絕建立。

- Fast-forward 過渡狀態（`reviewing` / `processing`）於 statusHistory 留有紀錄供前端顯示，但不對應任何 caller 可觸發的 use case——這些狀態純粹是 `TransferIntentAutoProgressService.progress(...)` 內部 transition path 的中介步驟。

### TransferIntentAutoProgressService

對應 `WithdrawalIntentAutoProgressService`，但本期採 unconditional fast-forward——三 category default 全部自動快進至 `processed`：

| Category               | 自動 accept (preparing→reviewing) | 自動 approve (reviewing→processing) | 自動 release (processing→processed) |
| ---------------------- | --------------------------------- | ----------------------------------- | ----------------------------------- |
| `platform_to_platform` | ✓                                 | ✓                                   | ✓（無金額門檻）                     |
| `platform_to_merchant` | ✓                                 | ✓                                   | ✓（無金額門檻）                     |
| `merchant_to_platform` | ✓                                 | ✓                                   | ✓（無金額門檻）                     |

實作收斂為單一 fast-forward 方法，**不**拆分為 `canAutoAccept` / `canAutoApprove` / `canAutoRelease` 三段條件決策：

```ts
class TransferIntentAutoProgressService {
  // unconditional fast-forward: preparing → reviewing → processing → processed
  progress(input: { entity: TransferIntentEntity<'draft'>; occurredAt: Date }): void;
}
```

落地考量：

- Fast-forward 邏輯由本 service 承擔——caller（submit use case）拿到 entity factory 回的 `preparing` draft entity 後呼叫 `progress(...)`，service 依 `submittedTransitionPath` 對 entity 依序呼叫 `transitionTo(...)`，statusHistory append 三筆 system actor 紀錄，最後設 `entity.state.processedAt = occurredAt`
- Service 為**同步、純 in-memory operation**：不持久化、不發 RPC、不接 transaction。Caller 負責於後續步驟 persist
- 與 `WithdrawalIntentAutoProgressService` 命名對稱、但內部行為不同：withdrawal 為條件式 progress（依 category + 限額），transfer 為 unconditional fast-forward
- 未來若 PM 引入條件式 progress（例如「大額轉帳需審核」），由該時期的 spec 重新評估 service shape 與 method 拆分；本期不引入 `canAuto*` 子方法

未來若 PM 規格引入「大額平台轉帳需審核」之類，可改為依 category + 金額 + 限額政策決定（與 withdrawal 限額機制平行），同時重新引入相應 use case 與 lifecycle endpoint（見 Open Points）。

---

## WithdrawalIntent 對齊

`TransferIntent` 引入後，`WithdrawalIntent` 同步整理：

### Category 引入

新增欄位 `category: WithdrawalIntentCategory`：

```ts
export enum WithdrawalIntentCategory {
  FromMerchant = 'from_merchant',
  FromPlatform = 'from_platform',
}
```

採 `from_<sender>` 命名。Withdrawal 的 recipient 永遠是 external，category 只需編碼 sender 變動的那一邊。

Schema migration 規則：

- 既有 row backfill：`senderMerchantId IS NOT NULL → 'from_merchant'`、`senderMerchantId IS NULL → 'from_platform'`
- 新欄位 NOT NULL；backfill 期間以 expression default 達成，migration 結束後 drop default

Submit DTO 新增 optional `category` 欄位：caller 可顯式提供，後端依 sender wallet type 推導並校驗一致：

```text
senderWalletType = platform_treasury → from_platform
senderWalletType = merchant_wallet   → from_merchant
```

不符或顯式 category 與推導值衝突，回 `data_invalid`。

### `senderMerchantId === null` 反模式整改

以下位置目前依賴隱式 `senderMerchantId === null` 分流，全部改成讀 `category`：

| 位置                                                 | 當前判斷                           | 整改後判斷                  |
| ---------------------------------------------------- | ---------------------------------- | --------------------------- |
| `WithdrawalIntentAutoProgressService.canAutoRelease` | `senderMerchantId === null`        | `category === FromPlatform` |
| `WithdrawalIntentEvaluationService` merchant 分支    | `senderWallet.merchantId !== null` | `category === FromMerchant` |
| 限額 / fee policy 查詢分支                           | 同上                               | 同上                        |

`senderMerchantId` 欄位本身保留——它是 merchant ownership 的 audit / query marker，不再參與業務分流判斷。

### `cancellableUntil` 填值政策

欄位保留（不動 schema），per-category 填值政策：

| Category        | `cancellableUntil` 填值             | 行為                      |
| --------------- | ----------------------------------- | ------------------------- |
| `from_merchant` | 提交時間 + 政策窗口（沿用既有規則） | 時間到 auto-accept        |
| `from_platform` | `submittedAt`（trivially-past）     | acceptance job 立即可受理 |

Acceptance worker 完全不需要知道 category——只依 `cancellableUntil` 是否到期決定。Category 差異被填值政策吸收，下游 worker 邏輯零變動。

### Destination internal-endpoint 拒收

承「Fund Case 方向不變式」第二條：`WithdrawalIntentEvaluationService.evaluate(...)` 內新增 destination identifier 對 `NetworkEndpoint` 的反查，命中即 issue `destination_internal`（quote 表面為 `WithdrawalIntentQuoteIssueCode.DestinationInternal`、submit batch 表面為 `WithdrawalIntentSubmitIssueCode.DestinationInternal`、envelope-level error code 為 `withdrawal_intent.destination_internal`）。`destination_invalid` 維持承載通用格式問題，與 `destination_internal` 形成清楚分工：形狀錯為前者、形狀合法但命中已知 internal endpoint 為後者。

具體實作由 `networkEndpointRpc.findByProviderAndIdentifier({ provider, identifier })` 承接；性能 / 索引優化屬實作層，本 spec 不規約。

### Platform / Merchant submit endpoint 分工

| Endpoint                                | 接受的 category    | sender 限制                                                                                         |
| --------------------------------------- | ------------------ | --------------------------------------------------------------------------------------------------- |
| `/fund/merch/withdrawal-intents` submit | 僅 `from_merchant` | sender 必須屬於 request merchant                                                                    |
| `/fund/plat/withdrawal-intents` submit  | 兩者皆可           | `from_merchant` 代提交：sender 為任一 merchant wallet；`from_platform`：sender 為 platform treasury |

「submitter actor」與「sender ownership」是兩個獨立維度，由 `submittedBy` 與 `senderMerchantId` 分別承擔：

- merchant user 代 merchant 提交：`submittedBy.type = merchant_user`, `senderMerchantId = <merchant>`, `category = from_merchant`
- platform user 代 merchant 提交：`submittedBy.type = platform_user`, `senderMerchantId = <merchant>`, `category = from_merchant`
- platform user 為平台自家提交：`submittedBy.type = platform_user`, `senderMerchantId = null`, `category = from_platform`

`category` 由 sender wallet type 推導，與 `submittedBy` 完全解耦。

### Withdrawal service fee allocation lines

Withdrawal service fee line is effect-driven rather than category-driven:

- `amountFee > 0`：planLines include one `service_fee` source line from sender wallet `available` and one `service_fee` destination line into sender wallet `held`
- `amountFee = 0`：planLines omit the `service_fee` pair entirely; no zero-amount placeholder line is emitted, and ledger projection does not create a service-fee row
- `from_platform` currently has no service fee policy, so it naturally produces no `service_fee` pair
- `from_merchant` may also omit the `service_fee` pair when the configured policy calculates fee 0

---

## 批次 API 介面（不入 domain）

PM UI 的批次操作（如「選 N 個 platform treasury 各送 amount 到 1 個 destination」）封裝在 API layer 與 use case orchestration layer，**不入 entity / fund case domain**。

落地形態：

- 批次 REST endpoint：input 為 N 筆 sub-request 陣列
- 後端 fan-out 為 N 次 single-item submit use case 呼叫
- 每筆獨立 reserve / persist / bind / dispatch，獨立成敗
- API response 為 N 筆對應結果（success + fund case id，或 failure + error code）

不引入 `batchCorrelationId` 持久化。每筆 fund case 完全獨立 lifecycle——audit、ledger、對帳全部 per-fund-case。

具體 batch endpoint 形狀、並發策略、部分失敗回傳格式，留實作 plan 收。

---

## Open Points (本設計待議)

下列項目於 `spec/open-points.md` 追蹤（spec/open-points.md 目前不存在，本 spec 落地時一併新建）：

1. **TransferIntent category × bucket invariant 的擴張時機**
   當前三組 bucket 組合為 PM 規格快照。未來若需要新組合（例如 `platform_to_merchant` 同時支援 available 與 held recipient），策略尚未決：擴張 category enum、放寬 invariant、或引入 sub-category。

2. **TransferIntent service fee 政策**
   當前所有 category 不收 service fee。架構上易於擴展——未來若特定 category 引入 fee，補 raw 欄位（`amountFee`、`amountNet`）並調整 allocation plan lines 即可。

3. **`WithdrawalIntent.decline()` 與對應 `TransferIntent.decline()` dead method**
   Mirror 對稱保留，目前無 use-case caller。整體 fund case dead method cleanup 時可一併考慮。

4. **`feePayerWalletId` 持久化跨 fund case 改造**
   本 spec 假設 `feePayerWalletId` 欄位已存在於 TransferIntent 與 WithdrawalIntent。實際落地排程為 impl ep-1 plan-1 第一部分（cross-cutting，先於 transfer 主線實作）。

5. **`NetworkEndpoint → (networkType, networkProvider)` derivation path**
   Same-network invariant 的 derivation 實作待 Codex 依 `NetworkEndpoint` 內部結構承接。本 spec 不規約 derivation 程式表面。

6. **TransferIntent 無 review / cancel / reject flow（fast-forward 政策）**
   本期 TransferIntent submit 即快進至 `processed`，`cancelled` / `declined` / `rejected` 三 status 不可達，亦無對應 entity action / use case / lifecycle endpoint。`TransferIntentStatus` enum 與 `statusHistory` shape 保留全 10 狀態以維持與其他 fund case 結構對稱（沿用 Deposit 註解標示模式）。未來 PM 若規劃合規審核、主動撤銷、或 processing 階段拒絕需求，需重新引入：
   - 相應 status path 與 entity action methods（`accept` / `approve` / `release` / `cancel` / `reject`）
   - 對應 use cases（`AcceptTransferIntent` / `ApproveTransferIntent` / `PlatformReleaseTransferIntent` / `PlatformCancelTransferIntent` / `PlatformRejectTransferIntent` / `ProcessTransferIntentAcceptance`）
   - REST lifecycle endpoints（`cancelTransferIntentById` / `rejectTransferIntentById` / `releaseTransferIntentById`）
   - 重新評估 `TransferIntentAutoProgressService` 政策表（依 category + 金額 + 政策做條件判斷）

   此項與 `fund-case-expansion-rest-contract-spec.md` Open Points 第 7 條對應。

---

## 對 `spec/system-overview.md` 的回流建議

本 spec 落地後，建議 `spec/system-overview.md` 同步 amend（留另案 PR）：

1. **§9 Canonical Flow: Transfer Intent**
   當前段落描述「multiple source wallets to one destination wallet」與「atomic compensation」對應 multi-leg + atomic 的 transfer 模型。實際落地為 single-leg + non-atomic（批次拆分為多 fund case），§9 應重寫對應實際模型。

2. **新增章節「Fund Case 方向不變式」**
   把本 spec 「Fund Case 方向不變式」段落提升到 system-overview 層級——這是跨 fund case 的根本結構性原則，比單一 fund case 細節更高層，自然屬於 overview。

3. **Fund Case lifecycle 對齊**
   §6 等對 fund case 一般性描述的章節，可補一句「fund case 內部可有 category 子分類，承擔 mechanical 共用但業務語義差異的場景」（對應 `fund-case-concept-proposal` 的延伸實踐）。

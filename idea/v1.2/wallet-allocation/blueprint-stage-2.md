---
status: draft
updated_at: 2026-04-28
updated_by: Tim
remark: 本文件未依照 blueprint 規範撰寫，暫供內容參考，等待後續重構改寫。
---

# Wallet Allocation Blueprint Stage 2

> **Status**: 議題 1-8 完整收斂；Phase 5 設計收斂完成
> **目標**：deposit / withdrawal intent 銜接 wallet allocation，並產出 ledger item 供前端顯示
> **Cashflow item**：本輪略過

---

## 議程進度

- [x] 1. WalletLedgerItem Schema + Query API（FundCase 對齊版）
  - [x] 1.1 粒度與唯一鍵
  - [x] 1.2 欄位構成（planAmount / outcomeAmount / realization / revision / source-destination / 時戳）
  - [x] 1.3 Persistence shape
  - [x] 1.4 Query API 形狀（含 PM ledger view 對齊）
- [x] 2. RPC Contract
  - [x] 2.1 RPC 資源切分 + namespace/id + 檔案 placement
  - [x] 2.2 RPC Code Enum 收斂
  - [x] 2.3 共用 Wire DTO
  - [x] 2.4 Per-method Spec
- [x] 3. Withdrawal 銜接黑洞
  - [x] 3.1 WithdrawalIntent lifecycle 與 wallet RPC 對應點
  - [x] 3.4 ID 綁定時機 + submit use-case 編排序列
  - [x] 3.3 Allocation lines 構造責任
  - [x] 3.2 失敗補償拓撲
- [x] 4. Deposit 銜接黑洞
  - [x] 4.1 NT event × deposit state → wallet RPC 對應拓撲（含 ID 綁定/編排）
  - [x] 4.2 失敗補償拓撲
  - [x] 4.3 Allocation lines 構造（含 fee 拓撲）
- [x] 5. FundCase Context Shape
  - [x] 5.1 Generic vs Per-type RPC endpoint 大方向 — 選 per-type
  - [x] 5.2 per-entity RPC 暴露的 DTO shape（合併原 5.6）
    - [x] 5.2a fund 端 RPC DTO 風格 — 採 full `Serialized<Entity>`
    - [x] 5.2b wallet 端 `WalletLedgerItemFundCaseDto` shape — REST DTO 已手刻 land
  - [x] 5.4 `withdrawalIntentRpc` contract 落地（含 envelope 規範對齊）
  - [x] 5.3 Cross-BC dependency wiring（wallet ledger composer 注入多個 fund RPC client）
  - [x] 5.5 對偶性檢查 minimal closing（含跨議題原則 13 commit）
- [x] 6. Projection 實作邏輯
  - [x] 6.1 Projection mapping framing — 單 row 持 plan + outcome 雙態
  - [x] 6.2 Path-by-path projection 行為（含 directSettle 一次寫 + release-from-bound + audit trail 對等性）
  - [x] 6.3 Cross-BC client 注入 + WalletLedgerItemComposer constructor + module wiring delta
  - [x] 6.4 Error handling pattern 沿用 + composer throw + reconciliation 推議題 7
- [x] 7. Cron / Scheduler
  - [x] 7.1 Phase 5 範圍與 framing — 全部推 future 實作
  - [x] 7.2 Withdrawal 側失敗點 audit + comment 模板
  - [x] 7.3 跨議題原則 14 commit（事後追認用 queue catch-up 為 default）
  - [x] 7.4 推 future TODO 條目分類
- [x] 8. Smoke Test 落地
  - [x] 8.1 Scope — Layer 1 happy paths only
  - [x] 8.2 Test infrastructure — B.1 use-case spec layer
  - [x] 8.3 Layer 1 happy path scenarios codify

---

## 1. WalletLedgerItem Schema + Query API

### 1.1 粒度與唯一鍵 ✅

**決議**：unique key = `(allocationId, walletId, bucket, currencyCode, kind)`

- `bucket` 採 Fund 視角值域：`available | held`，不暴露物理 bucket（`locked` / `held_locked`）
- 不引入 `side`——source/destination 由 amount 正負號隱含
- 同 (wallet, currency, kind) 跨 bucket 不合併（UI 需要區分可用 vs 鎖定）

**金額命名**：`plan` / `outcome`（捨棄 `expected` / `actual`，與 `WalletAllocationLineRole` 對齊）

**範例**（withdrawal 1000 USDT principal + 20 USDT service fee + 30 TRX network fee）：

| allocationId | walletId       | bucket    | currencyCode | kind        | plan | outcome |
| ------------ | -------------- | --------- | ------------ | ----------- | ---- | ------- |
| A            | merchantWallet | available | USDT         | principal   | -980 | -980    |
| A            | merchantWallet | held      | USDT         | service_fee | +20  | +20     |
| A            | platformWallet | available | TRX          | network_fee | -30  | -25     |

📌 延伸觀察（未展開）：同 (wallet, currency, kind) 在 source/destination 各一條時的 UI 合併顯示是 DTO mapper 職責，不影響 read model schema。

### 1.2 欄位構成（進行中）

#### 1.2a 金額命名 ✅

`plan` / `outcome`（amountDelta，帶正負號）。捨棄 `expected` / `actual`——那組詞暗示「預測 vs 修正」，但 wallet allocation 的 plan 是已鎖定的會計承諾、outcome 是兌現結果，「承諾 vs 兌現」語意更準。

#### 1.2b 結果有效性 ✅

**Field**: `realization`
**值域**: `pending | realized | partial | cancelled`
**性質**: DTO derived field，**不存 row**（純 (plan, outcome) 函數）

判定規則：

| plan                         | outcome           | realization |
| ---------------------------- | ----------------- | ----------- |
| 任何                         | null              | `pending`   |
| > 0                          | 0                 | `cancelled` |
| > 0                          | 絕對值 = plan     | `realized`  |
| > 0                          | 0 < 絕對值 < plan | `partial`   |
| null（legacy/backfill only） | 任何非 0          | `realized`  |

> 比較用絕對值 + sign match。`realization` 詞根呼應 `realized`，且為會計標準術語（realized gain / unrealized loss）。

📌 延伸觀察（未展開）：deposit propose 階段的 `pending` 與 withdrawal reserve→bound 的 `pending` 在 UI 是否需區分；可能在 from/to 處理後一起想。記下。

📌 延伸觀察（未展開）：未來若需 `excessive`（outcome > plan，例如 deposit 實收多於預期），`overrealized` 是會計可接受詞彙。記下。

#### 1.2c 版本管理 ✅

**Field**: `revision`
**值域**: `current | superseded`
**性質**: Row column（持久化，不可從 plan/outcome derive）
**寫入規則**: **Immutable append-only**——同 unique key 重新計算結果時，insert 新 row（revision=current）+ mark 既有 row 為 superseded。`plan` / `outcome` 既值不可改寫。

> 為何不用 `state`/`status`/`lifecycle`：避免與 allocation status (reserved/bound/settled) 在語意層級混淆。`revision` 與「版本族」心智模型自然搭配，`current/superseded` 比 `isCurrent: boolean` 保留「有後繼」的精準度。

#### 1.2d from/to endpoint 結構 ✅

**分層架構**（三個獨立概念）：

| 層           | 概念                                                                                             | 性質                                        |
| ------------ | ------------------------------------------------------------------------------------------------ | ------------------------------------------- |
| Wallet 雙端  | `source` / `destination`：space + (wallet, bucket) 或 external                                   | Row column                                  |
| Network 雙端 | `networkTransactionId` → 撈 `NetworkTransactionDto` → 投影為 `WalletLedgerNetworkTransactionDto` | Row 只存 id；DTO 拼裝                       |
| Row 自身位置 | `side: 'source' \| 'destination'` 指向兩端之一                                                   | Row column；DTO 也用 side（捨棄 direction） |

**Endpoint shape（discriminated union）**：

```ts
type LedgerEndpoint =
  | { space: 'internal'; walletId: bigint; merchantId: Uuid; bucket: 'available' | 'held' }
  | { space: 'external' };
```

External 變體無 wallet/merchant/bucket 欄位——細節靠 `networkTransactionId` 撈 NetworkTransaction 取得。

**命名收斂**：

- Wallet 層用 `source` / `destination`，與 `WalletAllocationSide` 詞彙對齊
- 不用 `from` / `to`（避免與 network 層詞彙混用）
- DTO 也用 `side` 不用 `direction`（前者是 row 欄位 pointer，閉環自洽）

**Network 攜帶方式**：

- Persistence 層只存 `networkTransactionId: bigint | null`
- API read path：composer 透過 RPC 取得 NetworkTransactionDto，投影為 `WalletLedgerNetworkTransactionDto`，拼進 `WalletLedgerItemDto`
- 與 fund `WithdrawalIntentComposed` 同 pattern
- 不用 `Ref` 命名（Ref 是 raw 內 inline embed 的 pattern；本場景 row 上只有 id，靠 RPC 補資料）

📌 延伸觀察（記下不展開）：

- `WalletLedgerNetworkTransactionDto` 具體節錄欄位（type / identifier / status / confirmedAt / ...）待 1.4 query API + composer 設計時細化。
- 同 (wallet, currency, kind) 在 source/destination 各一條時的 UI 合併顯示是 DTO mapper 職責。

#### 1.2e 時戳欄位 ✅

```ts
effectiveAt: Date; // 業務發生時間（from use-case occurredAt），UI 排序基準
createdAt: Date; // row insert 時刻，技術 audit
updatedAt: Date; // 充當 supersededAt 用途；current revision 等同 createdAt
```

捨棄獨立 `supersededAt`：immutable append-only 下 row 唯一可能 update 就是 revision: current → superseded，updatedAt 自然映射這個時刻、不展示，不另開欄位。

捨棄 `recordedAt` / `postedAt` / `recognizedAt`：與 `updatedAt` 標準配對 + 跨 entity 一致 > 為金流場景另開專屬詞。

📌 延伸觀察（記下不展開）：`effectiveAt` 與 `createdAt` 在 happy path 通常相等，但 retry / projection backfill 場景會分離，分離保留 audit 能力。

---

### 1.2 完整 Schema（定稿）

```ts
// Row 內 endpoint shape（discriminated union）
type LedgerEndpoint =
  | { space: 'internal'; walletId: bigint; merchantId: Uuid; bucket: 'available' | 'held' }
  | { space: 'external' };

type WalletLedgerItem = {
  // 識別
  id: bigint;
  allocationId: bigint;
  merchantId: Uuid;          // for query
  walletId: bigint;
  currencyCode: SupportedCurrencyCode;
  bucket: 'available' | 'held';
  kind: WalletAllocationLineKind;

  // 金額（amountDelta，帶正負號）
  plan: NumericString | null;
  outcome: NumericString | null;

  // 雙端
  source: LedgerEndpoint;
  destination: LedgerEndpoint;
  side: 'source' | 'destination';   // 指向 row 自己對應的端

  // Network reference（persistence 只存 id）
  networkTransactionId: bigint | null;

  // 版本（immutable append-only）
  revision: 'current' | 'superseded';

  // Subject context
  subjectType: WalletAllocationSubjectType;
  subjectIdentifier: bigint;

  // 時戳
  effectiveAt: Date;
  createdAt: Date;
  updatedAt: Date;
};

// API 層 DTO（composer 拼裝）
type WalletLedgerNetworkTransactionDto = {
  // 從 NetworkTransactionDto 投影；具體欄位待 1.4 定
};

type WalletLedgerItemDto = {
  ...                                                              // raw 投影（不含 networkTransactionId）
  realization: 'pending' | 'realized' | 'partial' | 'cancelled';   // derived from (plan, outcome)
  networkTransaction: WalletLedgerNetworkTransactionDto | null;    // RPC 補資料
};
```

---

### 1.4 Query API 形狀（進行中）

#### 1.4a PM Ledger View 可行性對齊 ✅

逐欄對照 PM CSV vs schema，**100% 對齊**。Composer 拼裝模式：

| PM 欄位            | 來源                                                    | 拼裝路徑                                     |
| ------------------ | ------------------------------------------------------- | -------------------------------------------- |
| Case               | Fund subject snapshot                                   | RPC by subjectType + subjectIdentifier       |
| Create/Update Time | createdAt / updatedAt                                   | row direct                                   |
| Serial Number      | subjectIdentifier + subjectType                         | DTO format `#WD_xxx` / `#DE_xxx` / `#TR_xxx` |
| Asset Type         | wallet owner type                                       | RPC by walletId                              |
| Transaction Type   | subjectType                                             | row direct                                   |
| Transaction Reason | kind                                                    | row direct                                   |
| Status             | subject 業務狀態                                        | RPC by subjectType + subjectIdentifier       |
| Tx Owner           | wallet display                                          | RPC by walletId                              |
| Currency           | currencyCode                                            | row direct                                   |
| 預計金額 Amount    | plan / outcome                                          | row direct                                   |
| 是否發生           | realization                                             | DTO derived                                  |
| From / To name     | source / destination endpoint                           | row direct + wallet RPC for display          |
| From/To Address    | wallet address (internal) / network endpoint (external) | wallet RPC + networkTransaction RPC          |
| Link               | network explorer URL                                    | networkTransaction RPC                       |

**設計原則**：Subject 與 wallet 兩邊生命週期不同，wallet ledger 不 snapshot subject 業務狀態——透過 (subjectType, subjectIdentifier) 即時 RPC。

PM CSV 中右側「Provider/Company/Balance」三欄與 Case 描述欄位實為 PM 給開發團隊的 annotation（標注 projection 呈現時 balance 預期狀態），非列表正文。Schema 不需處理。

#### 1.4b Query criteria + paging + endpoint ✅

**HTTP endpoint**：merch + plat 並列

- `GET /wallet/merch/ledger/items` — 商戶後台（merchantId 從 auth context 取）
- `GET /wallet/plat/ledger/items` — 平台後台（merchantIdIn 為 query 參數）

**Query criteria（先從簡，本輪只支援三個）**：

```ts
type WalletLedgerItemCriteria = {
  merchantIdIn?: Uuid[];
  walletIdIn?: bigint[];
  currencyCodeIn?: SupportedCurrencyCode[];
};
```

> 其他維度（kindIn / subjectTypeIn / effectiveAt range 等）未來逐步擴充。

**Paging**：本輪採 offset-based（沿用既有工具 `SearchOffsetQuery` / `SearchOffsetResult`）。

**Sort**：default `effectiveAt DESC, id DESC`（最新業務時間在前），不暴露 sort 自訂。

📌 延伸觀察（記下不展開）：ledger 紀錄會日益增多，未來應遷移至 cursor-based paging。等 cursor-based 工具 ready 後再切換。

#### 1.4c DTO Composer 流程

`WalletLedgerItemComposer`（與 fund `WithdrawalIntentComposer` 同 pattern）：

1. Repository 撈 row（含 raw schema 欄位）
2. Batch 取 wallet RPC（unique walletIds）
3. Batch 取 subject RPC（unique (subjectType, subjectIdentifier) pairs）
4. Batch 取 networkTransaction RPC（unique networkTransactionIds, nullable）
5. 拼成 DTO list

📌 延伸觀察（記下不展開）：Composer 對 fund subject 的 RPC 是 cross-bounded-context dependency——需要 fund 提供 `walletAllocationSubjectRpc.getById(subjectType, subjectIdentifier)` 之類 generic 介面，或各 subject type 各自 endpoint。設計責任在 fund 不在 wallet。若本輪 fund 沒這 RPC，DTO 的 subject 欄位本輪先 null。

---

## 議題 1 完整結束 ✅

---

## Followup List

- 議題 1 之外的延伸觀察都記在各小節 📌 標記中
- Cursor-based paging 遷移：等工具 ready 後切換 ledger query
- `WalletLedgerNetworkTransactionDto` 具體節錄欄位：等 NetworkTransactionDto 形狀 + composer 設計時細化
- Fund 提供 subject RPC 的接口設計：subject 業務狀態 + Case 描述

（暫無）

---

### 1.3 Persistence Shape ✅

**1.3a Endpoint 攤平**：

- `source_space, source_wallet_id, source_merchant_id, source_bucket`
- `destination_space, destination_wallet_id, destination_merchant_id, destination_bucket`
- External 變體下 wallet/merchant/bucket 為 null
- Mapper 從攤平 column 拼回 discriminated union

**1.3b 不加 DB-level unique constraint**：

- Ledger 非 source of truth，問題隨時可從 allocation 重新修正
- 不引入 partial unique index `WHERE revision = 'current'`
- Application 層保證寫入順序即可

**1.3c Migration scope**：

- 本輪建立 `wallet_ledger_item` 表 + 基本 query indexes
- 不做 `wallet_cashflow_item`

📌 延伸觀察（記下不展開）：未加 DB unique constraint 的決策代表「ledger 是 derived view，可重建」這個強原則。Repository / write path 仍應遵守 immutable append-only 順序，但破壞時不會炸 DB。

---

## 2. RPC Contract（進行中）

### 2.1 RPC 資源切分 + namespace/id + 檔案 placement ✅

**決議**：單一 `walletAllocationRpc` 包 6 個 method（reserve / bind / propose / settle / directSettle / release）。

| 項目          | 值                                                  |
| ------------- | --------------------------------------------------- |
| namespace     | `wallet`                                            |
| id            | `wallet-allocation`                                 |
| 檔案          | `contract-rpc/wallet/rpc/wallet-allocation.rpc.ts`  |
| 變數名        | `walletAllocationRpc`                               |
| Runtime topic | `rpc.<env>.wallet.wallet-allocation.<method-kebab>` |

**理由**：

- 單一 RPC：`settle` / `release` 在 Phase 4 凍結為兩條 flow（reservation / direct）共用——按 flow 拆會被迫重複定義或硬塞，破壞 use-case 的 1:1 mapping
- id 採真名 `wallet-allocation`：`WalletAllocation` 是 ubiquitous language，contract identity 直接對應領域資源；不為避免 namespace 重複詞而切短真名
- topic 段重複 `wallet.wallet-allocation` 是 routing key 的副產物，不影響語意正確性

**`expire` 不入 RPC**：scheduler-driven 的內部批次作業，由 wallet 自家 cron 觸發 use-case，不過 RPC 邊界（議題 7 處理 scheduler 配置）。

📌 延伸觀察（未展開）：variable name 與 id 不對稱（`walletAllocationRpc` vs id=`wallet-allocation`）並非 anti-pattern——variable name 是 caller import 視角的識別字、id 是 namespace 內 routing key，職責不同所以詞長度可不同。記下。

### 2.2 RPC Code Enum 收斂 ✅

**來源**：Phase 4 凍結事實——7 個 use-case error union 攤平去重後，RPC 邊界要承載 7 個 code（`Ok` / `NotFound` / `Conflict` / `WalletNotFound` / `AssetNotFound` / `BalanceInsufficient` / `OutcomeExcessive`）。

**切分**：

```ts
// contract-rpc/common/code/common.code.ts
export enum CommonCode {
  Ok = 'ok',
  NotFound = 'not_found',
  Conflict = 'conflict',
  // ... 其他既有 common code
}

// contract-rpc/wallet/code/wallet-allocation.code.ts
export enum WalletAllocationCode {
  WalletNotFound = 'wallet_not_found',
  AssetNotFound = 'asset_not_found',
  BalanceInsufficient = 'balance_insufficient',
  OutcomeExcessive = 'outcome_excessive',
}
```

**判準**：

- **CommonCode**（`Ok` / `NotFound` / `Conflict`）：對 RPC 邊界為通用 404 / 409 語意，不指明 subject。allocation 自身的「不存在」走 `CommonCode.NotFound`，與 Phase 4 use-case 用 `CommonErrorType.NotFound` 一致
- **WalletAllocationCode**（domain）：在 wallet allocation context 內才有意義的 code。`Wallet` / `Asset` / `Balance` / `Outcome` 各自為獨立可指認的主體，全套採 **subject-first** 命名（與 contract-structure §B 範例 `StatusInvalid` / `EnvelopeExceeded` 的 subject-implicit 不同；本次採 subject-first 全套以避免日後判讀成本）

**`OutcomeExcessive` 提升為 RPC public code**：

- ep-2-plan-1 review 已收斂：`outcome_excessive` 不再維持 file-local literal，而是提升為 shared `WalletAllocationErrorType.OutcomeExcessive`
- RPC contract 直接沿用 shared vocabulary；RpcMapper 做 mechanical translation（`WalletAllocationErrorType.OutcomeExcessive` → `WalletAllocationCode.OutcomeExcessive`），不再依賴 magic string 判斷
- Caller actionable space 不同：`Conflict`（status 不符）通常無 retry、`OutcomeExcessive` 可調整 outcomeLines 重試——不該被 generic `Conflict` 吞掉

**檔案 placement**：`contract-rpc/wallet/code/wallet-allocation.code.ts`（與 `wallet-allocation.rpc.ts` 同 stem）。

📌 延伸觀察（未展開）：若日後 envelope check 抽成 `OutcomeValidationService`（Phase 4 plan §4.4 已預留 zero-refactor signature），shared `WalletAllocationErrorType.OutcomeExcessive` 可直接沿用；不需再做第二次命名遷移。

### 2.3 共用 Wire DTO ✅

#### 2.3a `WalletAllocationLineDto`

```ts
// contract-rpc/wallet/dto/wallet-allocation-line.dto.ts
import type { Serialized } from '@esingpay/contract-base/common';
import type { WalletAllocationLine } from '@esingpay/contract-base/wallet';

export type WalletAllocationLineDto = Serialized<WalletAllocationLine>;
```

**理由**：raw 欄位（`space` / `side` / `walletId: bigint | null` / `bucket` / `currencyCode` / `amount: NumericString` / `kind`）僅 `walletId` 為 bigint，由 `Serialized<T>` 工具型別自動 `bigint → string` 化，alias 路線零維護成本。Codebase 已落實 `Serialized<T>`（Tim 確認）。

#### 2.3b `WalletAllocationDto`（success data 統一形狀）

所有 6 個 method 的 success path 統一回完整 allocation。理由：

- Use-case 已建構好完整 entity，return path 上資料在手，不回傳是浪費
- Caller 多源 verify 方便（fund 收到 allocation 後可直接 stash 在自己的 case 上）
- 形狀統一降低 mapper 複雜度——6 個 method 同一條 success path mapping
- Wire 成本低（allocation + lines 通常 < 10 lines）
- ack 形狀其實是 caller subset：caller 不需要可丟欄位，但 contract 不該預先剝奪取得完整資訊的能力

```ts
// contract-rpc/wallet/dto/wallet-allocation.dto.ts
export type WalletAllocationDto = {
  id: string; // bigint as string
  flow: WalletAllocationFlow; // 'reservation' | 'direct'
  status: WalletAllocationStatus; // reserved | bound | settled | released | expired
  type: WalletAllocationType;
  fundCaseType: FundCaseType;
  fundCaseIdentifier: string | null; // bigint as string; reserve 時 null
  reservedUntil: string | null; // ISO datetime; direct flow null
  planLines: WalletAllocationLineDto[] | null; // reserve / propose / direct-settle 時有值
  outcomeLines: WalletAllocationLineDto[] | null; // 未到終態時 null
  createdAt: string;
  updatedAt: string;
};
```

> 等同 `Serialized<WalletAllocation>`（若 raw 已內嵌 lines）；落實時若發現 raw 與 dto 1:1 對應，可改用 alias。本次先列展開形以利 method spec 直接引用閱讀。

#### 2.3c FundCase 在 RPC 採攤平形狀（不用 wrapper）

| 層              | FundCase 形狀                                          | identifier 形式        |
| --------------- | ------------------------------------------------------ | ---------------------- |
| Persistence     | 攤平 `fundCaseType` + `fundCaseIdentifier: bigint`     | raw bigint             |
| **RPC（本層）** | **攤平 `fundCaseType` + `fundCaseIdentifier: string`** | **bigint as string**   |
| REST DTO        | wrapper `fundCase: { type, identifier, subject }`      | codec encoded short id |

理由：RPC 是內部信任服務間的會計操作 contract，`{ type, identifier }` 攤平更直接；subject hydration 不在 wallet RPC 邊界提供（fund 自己有 case 資料）。Wrapper 是 REST 層形狀，承載 subject partial hydration 與 identifier 短碼。

#### 例外：listByFundCase input 用短名

`walletAllocationRpc.listByFundCase` 的 input 採短名 `{ type, identifier }`，不沿 RPC 層 long form：

```ts
// 例外（短名）
listByFundCase({ type: FundCaseType, identifier: IntegerString })

// 其他 method（long form 維持不變）
reserve({ fundCaseType, ... })
bind({ allocationId, fundCaseIdentifier })
propose({ fundCaseType, fundCaseIdentifier, ... })
directSettle({ fundCaseType, fundCaseIdentifier, ... })
```

**理由**：`listByFundCase` method name 已點明 `FundCase` scope，input field 加 `fundCase` prefix 是 method-name redundant prefix。其他 method method name 不含 `FundCase`，input 加 prefix 才能定位資源；繼續用 `fundCaseType` / `fundCaseIdentifier`，convention 主軸不變。

**Repository / Persistence 層不受影響**：repository signature 仍 long form `listByFundCase({ fundCaseType, fundCaseIdentifier })`；本例外限於 RPC contract input 層。

**WalletAllocationDto wire shape 不受影響**：DTO field 仍 `fundCaseType` / `fundCaseIdentifier`（DTO 是 entity-level shape，無 method-name context）。

#### 2.3d 跨層 wire 規則逐欄落實

| 欄位類別                                                          | RPC wire 形式                                         | 來源規則                                            |
| ----------------------------------------------------------------- | ----------------------------------------------------- | --------------------------------------------------- |
| `id` / `allocationId` / `walletId`                                | `string`（bigint 字面量）                             | contract-structure §D：bigint 不直接出現在 RPC wire |
| `fundCaseIdentifier`                                              | `string`（bigint 字面量；NOT codec encoded short id） | 議題 1 三層識別字區分                               |
| `createdAt` / `updatedAt` / `reservedUntil`                       | `string`（ISO datetime）                              | contract-structure §D：Date 不直接出現在 RPC wire   |
| `amount`                                                          | `NumericString`                                       | 既有慣例                                            |
| `bucket` / `space` / `side` / `kind` / `flow` / `status` / `type` | string enum value                                     | 既有慣例                                            |

📌 延伸觀察（未展開）：FundCase wrapper 在 REST、攤平在 RPC、攤平在 Persistence——這代表 wrapper 是「需要承載 subject hydration + identifier codec」的 outbound 表現層形狀，inbound trusted 層用攤平更精簡。記下不展開。

### 2.4 Per-method Spec ✅

#### 2.4a Type alias 全面採用

依 `@esingpay/contract-base/common/type/basic.ts` 提供的 alias，本節 spec 全面套用：

| 欄位類別         | Alias            | 取代        |
| ---------------- | ---------------- | ----------- |
| bigint as string | `IntegerString`  | 裸 `string` |
| ISO datetime     | `IsoDateTimeUtc` | 裸 `string` |
| 金額             | `NumericString`  | 既有        |
| 幣別             | `CurrencyCode`   | 既有        |
| Uuid             | `Uuid`           | 既有        |

#### 2.4b 整體骨架

```ts
// contract-rpc/wallet/rpc/wallet-allocation.rpc.ts
import { defineRpc, defineRpcMethod } from '@esingpay/contract-rpc/core';
import type { ResultDto } from '@esingpay/contract-rpc/common';
import { CommonCode } from '@esingpay/contract-rpc/common';
import { WalletAllocationCode } from '../code/wallet-allocation.code';
import type { WalletAllocationDto } from '../dto/wallet-allocation.dto';
import type { WalletAllocationLineDto } from '../dto/wallet-allocation-line.dto';
import { NAMESPACE } from '../namespace';

export const walletAllocationRpc = defineRpc({
  namespace: NAMESPACE, // 'wallet'
  id: 'wallet-allocation',
  methods: {
    reserve: defineRpcMethod<ReserveInput, ReserveOutput>(),
    bind: defineRpcMethod<BindInput, BindOutput>(),
    propose: defineRpcMethod<ProposeInput, ProposeOutput>(),
    settle: defineRpcMethod<SettleInput, SettleOutput>(),
    directSettle: defineRpcMethod<DirectSettleInput, DirectSettleOutput>(),
    release: defineRpcMethod<ReleaseInput, ReleaseOutput>(),
  },
});
```

> Method key 採 camelCase（`directSettle`），runtime topic helper 自動 kebab 化為 `direct-settle`。對齊 contract-structure §B 的 `getById` / `listRefsByIdIn` precedent。
>
> Input DTO 採 inline 在同檔（不獨立 `*.dto.ts`）：input 形狀緊耦合 method spec、獨立檔案會碎成 12 個檔；output success data + error code 才 export 為共用 DTO。

#### 2.4c WalletAllocationDto（success data）

```ts
// contract-rpc/wallet/dto/wallet-allocation.dto.ts
import type { IntegerString, IsoDateTimeUtc } from '@esingpay/contract-base/common';

export type WalletAllocationDto = {
  id: IntegerString;
  flow: WalletAllocationFlow; // 'reservation' | 'direct'
  status: WalletAllocationStatus; // reserved | bound | settled | released | expired
  type: WalletAllocationType;
  fundCaseType: FundCaseType;
  fundCaseIdentifier: IntegerString | null; // reserve 時 null
  reservedUntil: IsoDateTimeUtc | null; // direct flow null
  planLines: WalletAllocationLineDto[] | null; // reserve / propose / direct-settle 時有值
  outcomeLines: WalletAllocationLineDto[] | null; // 未到終態時 null
  createdAt: IsoDateTimeUtc;
  updatedAt: IsoDateTimeUtc;
};
```

#### 2.4d 6 個 method spec

##### `reserve`

```ts
type ReserveInput = {
  fundCaseType: FundCaseType;
  type: WalletAllocationType;
  reservedUntil: IsoDateTimeUtc;
  lines: WalletAllocationLineDto[];
};

type ReserveOutput =
  | ResultDto<CommonCode.Ok, WalletAllocationDto>
  | ResultDto<
      | WalletAllocationCode.WalletNotFound
      | WalletAllocationCode.AssetNotFound
      | WalletAllocationCode.BalanceInsufficient
    >;
```

> 不含 `NotFound` / `Conflict`：reserve 是 create 操作，不假設 allocation 存在。
> `reservedUntil` 由 caller (Fund) 指定——TTL 業務語意是 fund 端流程的容忍視窗，wallet 不假設業務節奏。

##### `bind`

```ts
type BindInput = {
  allocationId: IntegerString;
  fundCaseIdentifier: IntegerString;
};

type BindOutput =
  | ResultDto<CommonCode.Ok, WalletAllocationDto>
  | ResultDto<CommonCode.NotFound | CommonCode.Conflict>;
```

> action by id，遵 contract-structure §E：含 `NotFound` 變體。

##### `propose`

```ts
type ProposeInput = {
  fundCaseType: FundCaseType;
  fundCaseIdentifier: IntegerString;
  type: WalletAllocationType;
  lines: WalletAllocationLineDto[];
};

type ProposeOutput =
  | ResultDto<CommonCode.Ok, WalletAllocationDto>
  | ResultDto<WalletAllocationCode.WalletNotFound | WalletAllocationCode.AssetNotFound>;
```

> 不含 `BalanceInsufficient`：propose 不動 balance（Phase 4 plan §3.2 既有決策）。
> 不含 `NotFound` / `Conflict`：create 操作，不假設既有 allocation。

##### `settle`

```ts
type SettleInput = {
  allocationId: IntegerString;
  outcomeLines: WalletAllocationLineDto[];
};

type SettleOutput =
  | ResultDto<CommonCode.Ok, WalletAllocationDto>
  | ResultDto<
      | CommonCode.NotFound
      | CommonCode.Conflict
      | WalletAllocationCode.WalletNotFound
      | WalletAllocationCode.AssetNotFound
      | WalletAllocationCode.OutcomeExcessive
    >;
```

> `OutcomeExcessive` 由 shared `WalletAllocationErrorType.OutcomeExcessive` 經 mapper 機械對應到 RPC code。
> 不含 `BalanceInsufficient`：Phase 4 plan §4.2 settle 兩 flow 在 settle 階段都無 caller-actionable recovery，視為 invariant throw。

##### `directSettle`

```ts
type DirectSettleInput = {
  fundCaseType: FundCaseType;
  fundCaseIdentifier: IntegerString;
  type: WalletAllocationType;
  lines: WalletAllocationLineDto[];
};

type DirectSettleOutput =
  | ResultDto<CommonCode.Ok, WalletAllocationDto>
  | ResultDto<
      | WalletAllocationCode.WalletNotFound
      | WalletAllocationCode.AssetNotFound
      | WalletAllocationCode.BalanceInsufficient
    >;
```

> 含 `BalanceInsufficient`：Phase 4 plan §5.2 既有決策（caller 可調整 outcomeLines / fee 重試）。
> 不含 `OutcomeExcessive`：directSettle 不做 Wallet envelope check；use-case side 已在 ep-1-plan-3 收斂為 same-call planLines + outcomeLines 由 caller/RPC mapper 保證。RPC `DirectSettleInput` 加 planLines 仍屬 ep-2-plan-1 pending。

##### `release`

```ts
type ReleaseInput = {
  allocationId: IntegerString;
};

type ReleaseOutput =
  | ResultDto<CommonCode.Ok, WalletAllocationDto>
  | ResultDto<CommonCode.NotFound | CommonCode.Conflict>;
```

> 與 bind 對稱：action by id 含 `NotFound`，status 前提不符走 `Conflict`。

#### 2.4e 跨 method 規律對照表

| Method       | 操作性質     | NotFound/Conflict | WalletNotFound/AssetNotFound | BalanceInsufficient | OutcomeExcessive |
| ------------ | ------------ | ----------------- | ---------------------------- | ------------------- | ---------------- |
| reserve      | create       | —                 | ✓                            | ✓                   | —                |
| bind         | action by id | ✓                 | —                            | —                   | —                |
| propose      | create       | —                 | ✓                            | —                   | —                |
| settle       | action by id | ✓                 | ✓                            | —                   | ✓                |
| directSettle | create       | —                 | ✓                            | ✓                   | —                |
| release      | action by id | ✓                 | —                            | —                   | —                |

讀法：

- **create**（reserve / propose / directSettle）：不含 NotFound/Conflict（不假設 allocation 存在），但因 lines 由 caller 提供故含 WalletNotFound/AssetNotFound
- **action by id**（bind / settle / release）：含 NotFound/Conflict；只有 settle 因 caller 提供 outcomeLines 故額外含 WalletNotFound/AssetNotFound + envelope 的 OutcomeExcessive
- **BalanceInsufficient** 僅在 caller-actionable 時出現：reserve（caller 可調整鎖定金額）、directSettle（caller 可調整 outcome）

#### 2.4f 檔案佈局總覽

```
libs/contract-rpc/wallet/
├── code/
│   └── wallet-allocation.code.ts           # WalletAllocationCode enum
├── dto/
│   ├── wallet-allocation.dto.ts            # WalletAllocationDto
│   └── wallet-allocation-line.dto.ts       # WalletAllocationLineDto = Serialized<WalletAllocationLine>
├── rpc/
│   └── wallet-allocation.rpc.ts            # walletAllocationRpc + 6 inline input types
└── namespace.ts                            # NAMESPACE = 'wallet'
```

📌 延伸觀察（記下不展開）：議題 1 的 `WalletLedgerItemDto`（contract-rest）也應套同一份 alias，由 Claude Code 落實 `wallet-ledger-item-impl-plan-draft.md` 時順手用 `IntegerString` / `IsoDateTimeUtc`。不另開議題。

📌 延伸觀察（記下不展開）：未來若 ops / 監管需要統一 reservation TTL 上限，可在 wallet 端加 **TTL ceiling**（caller 提供值不可超過 wallet 強制 max）——但這是 wallet 側的防護機制，不取代 fund 端決定 TTL 的語意主導權。本輪不做。

---

## 議題 2 完整結束 ✅

---

## 3. Withdrawal 銜接黑洞（進行中）

### 3.1 WithdrawalIntent Lifecycle 與 wallet RPC 對應點 ✅

#### 3.1a 核心原則

> **Fund case 建立失敗才 release；一旦建立，無論業務成功或失敗都走 settle。**

語意：

| Wallet terminal | 含意                                                                         |
| --------------- | ---------------------------------------------------------------------------- |
| `released`      | allocation 從未綁定真實業務（鎖定退回、不留紀錄）                            |
| `settled`       | allocation 綁定了真實業務，這是業務的真實資金結局（無論 outcome 是不是全 0） |

「業務成功失敗」交給 fund 端解讀，wallet 不解讀——把**會計事實**（資金移動了沒）與**業務判讀**（status）解耦。

對齊 spec §11.1 Path 2 既有設計（"Persist Fail（reserved → released）"）；對齊議題 1 projection 邏輯（release 永不需要 projection）。

#### 3.1b Withdrawal 採 reservation flow

Withdrawal 因 ID generation = DB auto-increment bigint，必須走 **reservation flow**（reserve 取 allocationId → persist 取 withdrawalIntentId → bind）。Direct flow（propose / direct-settle）入參需 `fundCaseIdentifier`，於 withdrawal 場景該值不存在，路線不可行。

#### 3.1c Lifecycle ↔ Wallet RPC 對應表

| #   | Transition              | Allocation 狀態 | Wallet RPC                   | Outcome shape                                                             |
| --- | ----------------------- | --------------- | ---------------------------- | ------------------------------------------------------------------------- |
| 1   | (init) → preparing      | (尚未存在)      | **reserve → persist → bind** | planLines reserved                                                        |
| 2   | preparing → cancelled   | bound           | **settle**                   | outcomeLines 全 0                                                         |
| 3   | preparing → reviewing   | bound           | (none)                       | —                                                                         |
| 4   | reviewing → cancelled   | bound           | **settle**                   | outcomeLines 全 0                                                         |
| 5   | reviewing → declined    | bound           | **settle**                   | outcomeLines 全 0                                                         |
| 6   | reviewing → processing  | bound           | (none)                       | —                                                                         |
| 7   | processing → rejected   | bound           | **settle**                   | outcomeLines 全 0                                                         |
| 8   | processing → processed  | bound           | (none)                       | fund.release ≠ wallet.release                                             |
| 9   | processed → failed      | bound           | **settle**                   | outcomeLines 全 0（instruction 未上鏈）                                   |
| 10  | processed → transacting | bound           | (none)                       | —                                                                         |
| 11  | transacting → failed    | bound           | **settle**                   | outcomeLines = [principal=0, service_fee=0, network_fee=actual]           |
| 12  | transacting → completed | bound           | **settle**                   | outcomeLines = [principal=actual, service_fee=actual, network_fee=actual] |
| F   | submit 內 persist 失敗  | reserved        | **release**                  | (no projection)                                                           |

**Release 在 withdrawal flow 下只發生在 Path F**——reserve 成功、persist 失敗的補償視窗。

#### 3.1d 命名衝突處理：fund.release vs wallet.release

兩者剛好同名，完全不同層次概念，不更名。

- fund: `processing → processed`（財務授權通過）
- wallet: 撤銷鎖定（allocation reserved/bound → released）

實作層透過 `walletAllocationRpcClient.release(...)` 呼叫，已含 client 前綴避免 shadowing。

#### 3.1e Outcome shape 各 Path 驗證

**Path A**（preparing/reviewing/processing → cancelled/rejected/declined）：

```
planLines (reserved):  principal=1000, service_fee=20, network_fee=30
outcomeLines:          0 / 0 / 0
envelope check:        0 ≤ planLines ✓
reversal entry:        locked 1050 → available 1050（全退）
ledger:                寫入一筆 withdrawal allocation，outcome 全 0
```

**Path D**（transacting → failed，已上鏈但 revert，network fee 實扣）：

```
planLines:             principal=1000, service_fee=20, network_fee=30
NetworkTransaction.legs[NetworkFee]: amount=25
outcomeLines:          principal=0, service_fee=0, network_fee=25
envelope check:        ✓
reversal:              principal +1000、service_fee +20、network_fee +5
ledger:                principal/service_fee outcome=0；network_fee outcome=25 — 真實成本可追溯
```

**Path D'**（transacting → failed，已上鏈但 revert，鏈上實扣 network fee = 0）：

```
planLines:             principal=1000, service_fee=20, network_fee=30
NetworkTransaction.legs[NetworkFee]: amount=0
outcomeLines:          principal=0, service_fee=0, network_fee=0
envelope check:        ✓
reversal:              全退 1050（同 Path A）
ledger:                全 0 — 退化為 zero outcome
```

> Path D' 退化為 Path A 形狀。Caller 在事實層 dispatch：看到 actual fee = 0 改走 `buildZeroOutcomeLines(...)`，**不**呼叫 `buildNetworkFeeOnlyOutcomeLines(..., '0')`——後者語意是「fee 實扣 + principal/service_fee 不實扣」，零值 fee 不符該語意。

**Path E**（transacting → completed，network fee 實扣 > 0）：

```
planLines:             principal=1000, service_fee=20, network_fee=30
NetworkTransaction.legs[NetworkFee]: amount=25
outcomeLines:          principal=1000, service_fee=20, network_fee=25
reversal:              network_fee 差額 5 退回
```

**Path E'**（transacting → completed，鏈上實扣 network fee = 0，free-tx allowance / staked resource 抵扣）：

```
planLines:             principal=1000, service_fee=20, network_fee=30
NetworkTransaction.legs[NetworkFee]: amount=0
outcomeLines:          principal=1000, service_fee=20, network_fee=0
envelope check:        ✓
reversal:              network_fee 全退 30（fee payer wallet 的 locked 30 全部釋回 available）
ledger:                principal/service_fee outcome=per plan；network_fee outcome=0 — 鏈上實扣為 0 的合法結果
```

> Path E' 不是異常路徑。Caller 直接 `buildCompletedOutcomeLines(allocation, '0')`，settle 照常呼叫；不 log critical。詳見 §3.3a「Actual fee = 0 的設計立場」。

**Path F**（reserve OK + persist 失敗）：

```
allocation status:     reserved → released
projection:            無
ledger:                無紀錄（user 視圖從未感知到這筆 withdrawal intent）
```

#### 3.1f Corner case 警告（留 3.2 處理）

**reserve OK + persist OK + bind 失敗**：原則上 fund case 已建立應 settle，但 allocation 仍 `reserved` 無法直接 settle。Trusted RPC 下 bind 失敗實質為 system invariant violation（剛從 reserve 拿到的 allocationId 不可能 NotFound 或 Conflict）。處理策略：throw + alert，3.2 失敗補償拓撲議題正式定案。

📌 延伸觀察（記下不展開）：

- `bound → released` transition 在 withdrawal flow 下永不發生。Wallet allocation entity 仍保留此 transition 以支援其他 fund case type；`walletAllocationRpc.release` method 是否需區分 `releaseFromReserved` / `releaseFromBound`？目前不必動，一個 method 接受兩種起點是可接受的，contract 不因 caller 限制而拆分。
- 議題 4 Deposit 銜接也適用此原則：deposit propose（direct flow, born bound）→ 一旦 bound、所有終態走 settle、release 永不出現。Direct flow 的 propose 失敗則 fund case 從未建立、無 allocation 需要清理。等議題 4 直接套用。

### 3.4 ID 綁定時機 + Submit Use-case 編排序列 ✅

#### 3.4a ID generation 既成事實

| 端                  | ID generation                            |
| ------------------- | ---------------------------------------- |
| WithdrawalIntent.id | DB BIGSERIAL auto-increment（survey §1） |
| WalletAllocation.id | DB auto-increment bigint（Phase 4 既有） |

兩端皆 DB 自增 → propose 路線不可行（propose 入參需 `fundCaseIdentifier`，submit 階段 withdrawalIntentId 不存在）→ withdrawal 必走 reservation flow。

順序固定：**reserve → persist → bind**。

#### 3.4b walletAllocationId 欄位新增策略

新增到 `WithdrawalIntent` raw / draft / DB column / ORM model：

```ts
walletAllocationId: bigint; // NOT NULL
```

**Legacy data 處理**：

- Migration 將既存 row 的 `wallet_allocation_id` 填 0
- 應用層所有 wallet allocation 互動點加守衛：`if (intent.walletAllocationId !== 0n) { ... }`
- 守衛位置加註解：

```ts
// TEMPORARY: Legacy withdrawal intents have walletAllocationId=0 and skip allocation interaction.
// Remove this guard when legacy data is fully backfilled.
if (intent.walletAllocationId !== 0n) {
  await walletAllocationRpcClient.release(...);
}
```

#### 3.4c 最終 Submit 序列

```
1. Validate wallet ref               (walletClient.getWalletRefById)
2. Validate network endpoint         (networkEndpointRpcClient.listRefsByIdIn)
3. Evaluate fee/limit/balance        (WithdrawalIntentEvaluationService)
4. wallet.reserve(...)
   ├ 失敗 → submit 失敗（throw）
   └ 成功 → 取得 allocationId
5. Build draft entity                (WithdrawalIntentEntity.createSubmitted, 含 walletAllocationId)
6. Persist (DB tx)                   (withdrawalIntentRepository.create)
   ├ 失敗 → wallet.release(allocationId) [best-effort]; submit 失敗（throw）
   └ 成功 → 取得 withdrawalIntentId
7. wallet.bind(allocationId, intent.id)
   ├ 失敗 → withdrawalIntentRepository.delete(intent.id); submit 失敗（throw）
   │        Wallet 端 bound 孤兒由 reconciliation cron 清掃（議題 7）
   └ 成功 → allocation 進入 bound
8. Dispatch acceptance queue         (best-effort, dispatch 失敗只 log)
9. Compose output DTO
```

#### 3.4d Bind 失敗處理：方案 B（不做 best-effort release）

**核心論證**（Tim）：bind 失敗的根因高機率是 wallet 服務不可用，best-effort release 同源也會失敗——徒增複雜度而無收益。

執行邏輯：

- bind 失敗 → `repository.delete(intent.id)`（intent 從未對 user 公開、無業務發生過、刪除等同 transaction rollback 精神）
- 不主動 release allocation
- Wallet 端 bound 孤兒由未來 wallet reconciliation cron 清掃

#### 3.4e 「夭折」vs「幽靈」（allocation 孤兒）術語

| 名稱                | 定義                                                                  | Allocation 端表現                                        | 清理機制                   |
| ------------------- | --------------------------------------------------------------------- | -------------------------------------------------------- | -------------------------- |
| **夭折（aborted）** | Fund case insert db 後立刻被刪除（如 bind 失敗 → intent delete）      | **bound 孤兒**（fundCaseIdentifier 指向不存在的 record） | wallet reconciliation cron |
| **幽靈（ghost）**   | Fund case 從未 insert db（如 persist 失敗，allocation 停在 reserved） | **reserved 孤兒**                                        | wallet TTL auto-expire     |

**Allocation 孤兒** 是兩者的統稱（從 wallet 視角看「失去 fund case 對應」的 allocation）。

#### 3.4f Path F vs Bind 失敗的補償差異對照

| Path                  | 失敗根因典型                        | Wallet 服務狀態 | 兜底機制                  | Best-effort release      |
| --------------------- | ----------------------------------- | --------------- | ------------------------- | ------------------------ |
| **F**（persist 失敗） | DB 連線 / constraint 違反           | 通常正常        | reserved TTL auto-expire  | **做**（加速回收）       |
| **Bind 失敗**         | Wallet 服務不可用 / partial failure | 通常異常        | bound reconciliation cron | **不做**（同源失敗無效） |

關鍵差異：reserved 狀態有 TTL expire 兜底；bound 狀態沒有 TTL 兜底、必須由 reconciliation 接手。

📌 延伸觀察（記下不展開）：reconciliation cron 「偵測 wallet 端 bound 但 fundCaseIdentifier 對應資料不存在」的清掃邏輯是議題 7 的工作項目。本議題建立 dependency hook，落實留 7。

### 3.3 Allocation Lines 構造責任（進行中）

#### 3.3a Network Fee 來源與責任歸屬 ✅

| 項目                         | 決議                                                                                     |
| ---------------------------- | ---------------------------------------------------------------------------------------- |
| Service 命名                 | `WithdrawalNetworkFeeEstimationService`（fund/withdrawal 範疇 service）                  |
| 本輪實作範圍                 | TRON provider + USDT 場景，hardcode 常數 `n TRX`                                         |
| 估算時機                     | quote 階段就要算（balance check 需要）                                                   |
| Reserve / Settle 取值        | reserve = estimate（保守上限）；settle = actual（從 NetworkTransaction 取）              |
| Actual 取值來源              | NetworkTransaction `legs[type=NetworkFee]` 的 `{ currencyCode, amount }`；見下節         |
| WithdrawalIntent schema 變動 | **不**重命名 amountFee；**不**新增 amountNetworkFee 欄位（不同幣別、非 amount 構成元素） |
| Fee 代付方                   | 本輪 platform 代付為主，需查 sender endpoint 的 fee delegation 關係                      |
| Merchant 配置                | 不引入，列 TODO                                                                          |

**Actual 取值來源詳述**：

- 來源：`networkTransaction.legs.find(leg => leg.type === NetworkTransactionLegType.NetworkFee)`，讀其 `{ currencyCode, amount }`。
- 正規化責任：Network BC 負責把 chain raw 的實扣 fee 正規化到該 leg；Fund 不從 `blockchainDetail.gasUsed` 或其他 provider-specific raw 欄位自行推導。
  - 反例：Tron 的 `gasUsed`（= `energy_used`）描述 energy 用量，不是實扣 fee。staked energy 抵扣後 `energy_used > 0` 但 fee = 0 是常態；fee 應從 chain raw 的 `fee` 欄位（單位 SUN，= `energy_fee + net_fee`，已扣 stake 抵扣）取。Fund 若自行從 `gasUsed` 推導會產生虛假 fee。
- 不變式：對於有 network fee 概念的鏈，settled / failed NetworkTransaction 的 `legs` **必須**包含一條 `type = NetworkFee` 的 leg；缺漏視為 Network BC 違約，Fund 不為其做 fallback 推導（避免錯帳）。

**Actual fee = 0 的設計立場**：

`networkFee` leg 的 `amount = "0"` 是**鏈上合法事實**，不是觀測缺失。原因：

- 部分鏈有 free-tx allowance（例：Tron 每地址每日 free bandwidth，足以 cover 簡單 TRX transfer）。
- 部分鏈支援 staked resource 抵扣（例：Tron staked energy / bandwidth 抵扣 TRC20 fee 至 0）。
- 上述情境下，鏈回報的實扣 fee 本來就是 0；Network BC 必須照實在 `networkFee` leg 上填 `"0"`，不可改填 estimate、不可填 null。

語意對照：

| `networkFee` leg 狀態                       | 語意                                | Fund 處理                                       |
| ------------------------------------------- | ----------------------------------- | ----------------------------------------------- |
| 缺漏 / `legs` 中無 `type = NetworkFee` 條目 | 觀測未到位 / 上游 schema 違反不變式 | extractor 回 `null`，settle skip + critical log |
| `amount = "0"`                              | 鏈上實扣為 0（合法）                | settle 照走，outcome 帶 fee = 0                 |
| `amount > "0"`                              | 鏈上實扣 > 0                        | settle 照走，outcome 帶 actual fee              |

> Service 命名理由：以「withdrawal」這件事為範疇而非以 entity 為範疇——estimation 是 use-case-level 服務，幫 withdrawal flow 做特定情境估算，不是 WithdrawalIntent entity 的方法。

#### 3.3b PlanLines / OutcomeLines 結構 ✅

**Lines 結構**：positive service fee withdrawal 為 6 條（source / destination 對稱表達；ledger projection 兩端都要記）。若 service fee 為 0，省略 service_fee pair，withdrawal planLines 為 principal + network_fee 兩組 pair。

**Withdrawal reserve planLines**（platform 代付 network fee）：

| #   | walletId                     | bucket    | side        | space    | kind        | currency |
| --- | ---------------------------- | --------- | ----------- | -------- | ----------- | -------- |
| 1   | sender_wallet_id             | available | source      | internal | principal   | USDT     |
| 2   | null（external chain）       | null      | destination | external | principal   | USDT     |
| 3   | sender_wallet_id             | available | source      | internal | service_fee | USDT     |
| 4   | sender_wallet_id             | held      | destination | internal | service_fee | USDT     |
| 5   | platform_fee_payer_wallet_id | available | source      | internal | network_fee | TRX      |
| 6   | null（external chain）       | null      | destination | external | network_fee | TRX      |

**設計亮點**：

- **Service fee destination 為 sender_wallet_id 的 held bucket**：service fee 是 merchant wallet 內部移動（available → held），merchant 失去動用權但資金仍在自家 wallet 上；未來再以另一個 allocation 歸集到 platform treasury
- **Space 是 line 個別屬性**（不是 allocation 整體）：每條 line 描述 endpoint 是 internal 還是 external，不描述 allocation 方向
- **Platform 代付 network fee**：source 是 platform_fee_payer_wallet（不是 sender），destination 是 external chain

**OutcomeLines 構造對稱**：每條 plan line 對應一條 outcome line（同 walletId/bucket/side/space/kind/currency），僅 amount 變動。

#### 3.3c Platform Fee Payer Wallet 取值機制

Fund 透過 wallet 暴露的 RPC 查 platform fee payer wallet id（入參 `networkEndpointId`、出參 walletId）。

Wallet 內部實作暫時 hack（wallet 偷偷查 network 取得 fee delegation 配置），未來這個機制全部由 wallet 自己管理。

> RPC method 命名與位置（`walletBasicRpc.getNetworkFeePayerWalletId`、新建 RPC 或其他）留實作階段決定，不在本設計議題收斂——本議題只標記「fund 對 wallet 的查詢依賴」需求。已加入 TODO 章節。

#### 3.3d PathA Outcome 構造（helper 設計）

cancel/reject 等 Path A 場景的全 0 outcomeLines 由 caller 顯式構造（不引入 contract magic null sentinel）。

Fund 端封裝為 helper functions（不用 composer 字樣——`composer` 在 esingpay 慣例專門用於 aggregate）：

```ts
// fund/feature/withdrawal-intent/helper/withdrawal-allocation-line.helper.ts
export function buildZeroOutcomeLines(allocation: WalletAllocationDto): WalletAllocationLineDto[] {
  return allocation.planLines.map((line) => ({ ...line, amount: '0' }));
}

export function buildCompletedOutcomeLines(
  allocation: WalletAllocationDto,
  actualNetworkFee: NumericString,
): WalletAllocationLineDto[] {
  return allocation.planLines.map((line) =>
    line.kind === 'network_fee' ? { ...line, amount: actualNetworkFee } : line,
  );
}

export function buildNetworkFeeOnlyOutcomeLines(
  allocation: WalletAllocationDto,
  actualNetworkFee: NumericString,
): WalletAllocationLineDto[] {
  return allocation.planLines.map((line) =>
    line.kind === 'network_fee' ? { ...line, amount: actualNetworkFee } : { ...line, amount: '0' },
  );
}
```

三個 helper 採 **amount-effect naming**（描述 outcome 結構），不綁定 fund 側 status：

- `buildZeroOutcomeLines`：所有 line amount 全 0（cancel / decline / reject / pre-chain failed / failed-with-zero-fee 場景）
- `buildCompletedOutcomeLines`：principal / service_fee 照 plan、network_fee 用實扣（completed 場景；實扣可為 0）
- `buildNetworkFeeOnlyOutcomeLines`：principal / service_fee 為 0、network_fee 用實扣（failed-after-network-cost 場景；實扣必 > 0）

caller 依事實層選擇 helper（network 是否消耗了 fee），不依 fund status 字面選擇。

各 cancel/reject use-case → `buildZeroOutcomeLines(allocation)` → `wallet.settle(allocationId, lines)`。

**`actualNetworkFee` 零值語意**：

- `buildCompletedOutcomeLines(allocation, '0')` **是合法呼叫**：代表 completed 但鏈上實扣 fee = 0（free-tx allowance / staked resource 抵扣，見 §3.3a）。outcome 的 network_fee line amount = `'0'`，settle 時 plan 30 − outcome 0 = 30 全退。
- `buildNetworkFeeOnlyOutcomeLines` **不應收到 `'0'`**：fee = 0 的 failed 場景應 dispatch 到 `buildZeroOutcomeLines`（caller 在事實層判斷後選 helper，不傳 `'0'` 給此 helper）。對應 §3.1e Path D'。
- helper 不檢查 `actualNetworkFee` 是否為 0；caller 端 dispatch 已涵蓋語意。

---

## 議題 3.3 完整結束 ✅

### 3.2 失敗補償拓撲 ✅

#### 3.2a 策略選擇：B with stub

**選擇 B（decoupled）但 retry 機制尚未落實，失敗端先 log + TODO**。

| 失敗點                                    | Fund transition                        | Wallet RPC 行為                          |
| ----------------------------------------- | -------------------------------------- | ---------------------------------------- |
| Submit 內 reserve / persist / bind 失敗   | submit 失敗（throw）                   | 3.4 既決議（不重議）                     |
| Submit 之後所有 transition 的 settle 失敗 | **transition 已 commit**，照常返回成功 | catch + log + `// TODO: implement retry` |

**核心 invariant**：settle 失敗 = 多鎖 + 帳本落後，不會錯帳、不會丟失資金。金融層面安全，可日後補。

最壞影響：

- Wallet 端 allocation 卡在 bound、planLines 持續占用 locked
- Ledger projection 落後（缺 outcome rows）
- Fund intent status 已 transition 完成，user 視角無感
- 事後 retry 即可恢復

#### 3.2b Settle Idempotency

留待以後完善（未查 wallet 端既有設計、未定 idempotency token 機制）。在 retry 機制設計時一併處理——retry 重複呼叫的安全性是 retry 的前置條件。列入 TODO。

#### 3.2c Settle 失敗的可見性

**只用 log，不為 fund intent 加 settlement status 欄位**。

統一記錄欄位：

```ts
logger.error('wallet settle failed, intent transition kept; allocation is stuck', {
  intentId,
  allocationId,
  outcomeLines,
  err,
});
```

未來 retry 機制需要的資訊（allocationId / outcomeLines）都在 log，可用 log query 手動 reconcile，無需污染 fund schema。

#### 3.2d 編排順序與守衛模式

每個 transition use-case 內統一模式：

```ts
intent.cancel();                              // entity transition
await this.repository.update(intent);         // commit fund DB

if (intent.walletAllocationId !== 0n) {       // legacy guard (見 3.4b)
  try {
    await walletAllocationRpcClient.settle(
      intent.walletAllocationId,
      buildZeroOutcomeLines(allocation),      // helper from 3.3d
    );
  } catch (err) {
    // TODO: implement async retry mechanism for stuck settle
    logger.error('wallet settle failed, ...', { ... });
  }
}
```

**順序固定**：`transition commit → settle try`。Settle 失敗不反向 rollback transition（user 已收到結果、business 已完成判讀）。

#### Implementation note：抽 service 化

上述 inline pattern 為設計層示意。實作層當多個 transition use-case 共用同一個 readback / 守衛 / catch+log 行為時，可抽為 application service（如 `WithdrawalIntentAllocationSettlementService.settleZeroOutcomeBestEffort(...)` / `settleCompletedOutcomeBestEffort(...)` / `settleNetworkFeeOnlyOutcomeBestEffort(...)`），集中：

- legacy `walletAllocationId === 0n` 守衛
- `walletAllocationRpc.listByFundCase(...)` readback
- cardinality assertion（length 0 / 1 / >1 三分支）
- outcome line 構造（呼上述 helper）
- `walletAllocationRpc.settle(...)` 呼叫 + catch / log（議題 7.2b Class 2 模板）

Service 化後 transition use-case 仍保留「transition commit → 呼 `settleXxxOutcomeBestEffort(...)`」順序紀律。Service 規則：

- 不開 fund DB transaction（settle RPC 在 transaction 外）
- 不 mutate intent
- 不決定 status transition
- 不呼 `walletAllocationRpc.release(...)`
- 回 void，transition use-case 保留自身 business return shape

#### 3.2e 與議題 7 reconciliation 的銜接

議題 7 reconciliation cron 新增職責：

> 偵測「fund intent 已 transition 到 terminal status（cancelled / declined / rejected / failed / completed），但 wallet allocation 仍 bound」的 stuck case，自動 retry settle。

這就是「未來補全的 retry 機制」本體。本議題建立 dependency hook、實作落實在議題 7。

📌 延伸觀察（記下不展開）：選 B 的代價是 transition + settle 不在同一 transaction 邊界。NetworkTransaction event handler 收到事件 → fund commit transition → settle 失敗的場景下，event 已 ack 無法 redeliver——這是接受的取捨（Tim 已判讀金融風險可控）。未來實作 outbox pattern 時可一併解決 event redelivery 與 settle retry 兩個問題。

---

## 議題 3 完整結束 ✅

---

## 4. Deposit 銜接黑洞

### 4 議程結構說明

議題 3 是 4 個子議題（lifecycle / submit 編排 / lines 構造 / 失敗補償）。Deposit 走 direct flow，沒有 reserve+bind 兩階段、ID 在 propose/directSettle 時 caller 已持有 deposit aggregate（NT event 驅動下，deposit 在 wallet RPC 之前就已被 createInTx），議題 3.4 「ID 綁定時機 + submit 編排」對 deposit 沒有獨立問題空間，併進 4.1。

調整後：

- 4.1 NT event × deposit state → wallet RPC 對應拓撲（含 ID 寫入 / 編排序列）
- 4.2 失敗補償拓撲
- 4.3 Allocation lines 構造（含 fee 拓撲）

---

### 4.1 NT event × Deposit State → Wallet RPC 對應拓撲 ✅

#### 4.1a Deposit codebase 既成事實（凍結，from `fund-deposit-codebase-survey.md`）

- 只有 `Deposit`，無 `DepositCase` / `DepositIntent`
- ID = DB-side `BIGSERIAL` auto-increment（與 `WithdrawalIntent` 一致）
- Wallet ref = `recipientWalletId: bigint`（單一欄位，被動接收方）
- Network ref = `originNetworkTransactionId`（無 target）
- Amount fields = `amount` / `amountNet` / `amountFee`（無 networkFee 欄位）
- Status 路徑：`createRecognized` 內部一次走完 `preparing → reviewing → processing → processed → transacting`；`Cancelled` / `Declined` / `Rejected` 在 enum 內但 transition map 全空（dead code）
- Finalized = `Completed | Failed`
- 3 個 NT event handler（discovered / settled / failed）骨架一致：mutex → recognizableCheck → existing? → createDeposit / updateExistingDeposit → handleSideEffects
- Persistence 採 `TransactionRunner.run + createInTx/replaceInTx`（比 withdrawal 進步，可在同 tx 寫 `walletAllocationId`）
- `walletClient.handleDepositCreated` / `handleDepositFinalized` 為 TODO no-op（待重構為直呼 `walletAllocationRpcClient`）

#### 4.1b Deposit 採 direct flow

依 spec §4.2 與 wallet allocation design §7.4-7.6，deposit 採 direct flow：

- 未 final 階段 → `propose`（born bound）
- 已 final + 第一次見到 → `directSettle`（born settled）
- 已 final + 從 bound 推進 → `settle`

**spec §7.6 字面定義 directSettle 的觸發條件就是「discovered 時已 final 的 deposit」**——C 路徑（settled + missing）是字面對應；E 路徑（failed + missing）是同一語意空間下「outcome 為 zero」的對稱情形。Deposit 不走兩階段補一個 fake bound 階段。

#### 4.1c Wallet 同步策略：Queue 模式（取代 inline RPC）

Deposit use case 不直接呼叫 `walletAllocationRpcClient`。Wallet 同步職責推到 fund-internal queue (`depositWalletSyncQueue`)，由 worker 執行：

- **Use case**：persist deposit + dispatch sync message
- **Worker**：拉 message → read latest deposit → 從 status 判斷該 sync 操作 → 呼叫 wallet → 回填 allocationId

這個切分的核心理由（凍結為原則）：

1. **Deposit 業務流程不被 wallet 同步阻擋**：deposit 是被動接收外部資金，業務流程只關心 NT event 的 recognize / transition；wallet 同步是另一個 concern
2. **失敗補償統一為 queue retry**：不需要 path-by-path 寫 throw + log + cron retry 三套機制
3. **Worker-from-DB 不變式**：worker 不信任 message payload（除 `depositId`），永遠從 DB 拿 single source of truth；message 過期或亂序不影響正確性
4. **In-process queue 性質**：dispatch 對 use case 等同 function call；dispatch 失敗 = system bug 而非 application 補償範圍

#### 4.1d walletAllocationId Schema 訊號設計

`Deposit.walletAllocationId: bigint | null`（**NULLABLE**）。

| 值            | 含義                      | Worker 行為    |
| ------------- | ------------------------- | -------------- |
| `null`        | new deposit, sync pending | 進入 sync 流程 |
| `0n`          | legacy data backfill      | skip / ack     |
| `bigint > 0n` | already synced            | skip / ack     |

對 worker 來說，三種 case 的判斷收斂為一行：

```ts
if (deposit.walletAllocationId !== null) return; // ack
// 走到這裡 deposit.walletAllocationId === null，無條件 sync
```

**Migration**：

- DB column: `wallet_allocation_id BIGINT NULL`（無 default）
- Migration script 對既存 row 執行 `UPDATE ... SET wallet_allocation_id = 0`
- 新建 row 由 use case path A/C/E 顯式寫 `null`

**與 Withdrawal schema 的不對稱**：

| Fund case  | Schema            | Legacy sentinel | Transient pending      |
| ---------- | ----------------- | --------------- | ---------------------- |
| Withdrawal | `bigint NOT NULL` | `0n`            | (不存在 — inline sync) |
| Deposit    | `bigint NULL`     | `0n`            | `null`                 |

不對稱反映業務本質（reservation+inline vs direct+queue），不是 inconsistency。

#### 4.1e 6 Paths Use Case 編排（凍結）

**Path A — discovered + missing**

```text
1. mutex (by networkTransactionId)
2. recognizableCheck
3. getByOriginNetworkTransactionId → null
4. resolveWallet (walletClient.getWalletByNetworkEndpointId)
5. build deposit draft (walletAllocationId = null)
6. transactionRunner.run → depositRepository.createInTx(draft, tx)
7. depositWalletSyncQueue.dispatch({ depositId: deposit.id })
```

deposit 落地 status = `transacting`，walletAllocationId = `null`。Worker 拉到 → 見 null + transacting → propose。

**Path B — discovered + existing tx (no transition)**

```text
1. mutex
2. recognizableCheck
3. getByOriginNetworkTransactionId → existing (transacting)
4. (no transition — already in correct status)
5. (no persist)
6. (no dispatch)
```

唯一不 dispatch 的 path。理由：NT 重投無新事實，Path A 那次 dispatch 是 single source of truth；in-process queue 性質下 Path A 的 dispatch 視為已成功進 queue，後續由 queue 自身機制（retry policy + dead letter）保證 sync 完成。

**Path C — settled + missing**

```text
1. mutex
2. recognizableCheck
3. getByOriginNetworkTransactionId → null
4. resolveWallet
5. build deposit draft (walletAllocationId = null) + entity.handleSettled(...)
6. transactionRunner.run → depositRepository.createInTx(finalizedDraft, tx)
7. depositWalletSyncQueue.dispatch({ depositId: deposit.id })
```

deposit 落地 status = `completed`，walletAllocationId = `null`。Worker 拉到 → 見 null + completed → directSettle (actual outcomeLines)。

**Path D — settled + existing tx (transacting → completed)**

```text
1. mutex
2. recognizableCheck
3. getByOriginNetworkTransactionId → existing (transacting)
4. depositEntity = fromRaw(existing); depositEntity.handleSettled(...)
5. transactionRunner.run → depositRepository.replaceInTx(entity.toRaw(), tx)
6. depositWalletSyncQueue.dispatch({ depositId: existing.id })
```

walletAllocationId 維持 `replaceInTx` 的原值（可能 null / 0n / bigint）。Worker 自己處理。

**Path E — failed + missing**

```text
1. mutex
2. recognizableCheck
3. getByOriginNetworkTransactionId → null
4. resolveWallet
5. build deposit draft (walletAllocationId = null) + entity.handleFailed(...)
6. transactionRunner.run → depositRepository.createInTx(failedDraft, tx)
7. depositWalletSyncQueue.dispatch({ depositId: deposit.id })
```

deposit 落地 status = `failed`，walletAllocationId = `null`。Worker 拉到 → 見 null + failed → directSettle (zero outcomeLines)。

**Path F — failed + existing tx (transacting → failed)**

```text
1. mutex
2. recognizableCheck
3. getByOriginNetworkTransactionId → existing (transacting)
4. depositEntity = fromRaw(existing); depositEntity.handleFailed(...)
5. transactionRunner.run → depositRepository.replaceInTx(entity.toRaw(), tx)
6. depositWalletSyncQueue.dispatch({ depositId: existing.id })
```

#### 4.1f Wallet Client Hook 重構方向

廢除 `walletClient.handleDepositCreated` / `walletClient.handleDepositFinalized` 兩個 TODO no-op hook。Use case 不再注入 `walletAllocationRpcClient`，改為注入 `depositWalletSyncQueue`，在每個 path 結尾 dispatch（除 Path B）。

`WalletClient` 仍保留為 `getWalletByNetworkEndpointId` 等 wallet read facade，本質不變。

#### 4.1g depositWalletSyncQueue 定義

##### 4.1g.1 設計憲法（不變式）

四條原則。任何後續修改都應以此為衡量：

1. **In-process function call 性質**：dispatch 對 use case 等同 function call；失敗 = system bug，application 不寫補償邏輯
2. **Worker 不信任 message payload**：除 `depositId` 外，message 內容對 worker 邏輯無影響；DB 是唯一真相來源
3. **Use case 不 inspect walletAllocationId**：dispatch 決策不依賴 allocation 同步狀態；schema 訊號（null / 0n / bigint）由 worker 端解讀
4. **Sync 失敗的責任完全在 queue 自身機制**：retry policy + dead letter；不靠 NT event 重投救（Path B 不 dispatch 是這個原則的具體體現）

##### 4.1g.2 Message Shape

```ts
type DepositWalletSyncMessage = {
  depositId: bigint;
};
```

只攜帶 `depositId`。其他資訊（status / 應做 propose 還是 directSettle / outcomeLines 構造）由 worker 從 DB 重讀決定。

##### 4.1g.3 Worker 流程（6 步）

> **[Amended after plan-3 land]** 此段落於 plan-3 落地後 amend，反映 step 3.5 fence、9-cell 矩陣、與 step 5 conditional writeback 設計演進。原 5-step 流程的 step 2 broad skip（`if (deposit.walletAllocationId !== null) return`）在 Path D / F transacting → completed / failed 場景下會錯誤跳過 settle 路徑，已棄用；改採 6-step 流程，`walletAllocationId` 僅在 step 2 legacy guard 與 step 3.5 fence 兩處被 inspect，step 4 dispatch 純依 `(deposit.status, existing)` 矩陣決策。

Worker 流程（6 步）：

**Step 1 — Read latest deposit**

```ts
const deposit = await this.depositRepository.getById(msg.depositId);
if (deposit === null) {
  // Dispatch 前一定已 persist；null 視為 invariant violation
  throw new Error(`deposit ${msg.depositId} not found`);
}
```

**Step 2 — Legacy guard**（唯一將 `walletAllocationId` 作為直接決策依據的點之一）

```ts
// Legacy backfill 列永久跳過。0n sentinel 由 phase 10 migration 寫入。
if (deposit.walletAllocationId === 0n) return;
```

Step 2 是「跨議題原則 8 不適用範圍」的具體體現。worker 必須 inspect `walletAllocationId` 才能識別 legacy 列。原則 8 適用於 NT event handler use cases；worker 屬於另一範疇。

**Step 3 — `listByFundCase` catch-up**（length 0 / 1 / >1 三分支）

```ts
const result = await this.walletAllocationRpcClient.listByFundCase({
  type: FundCaseType.Deposit,
  identifier: deposit.id.toString(),
});
const list = unwrapOk(result);
let existing: WalletAllocationDto | null;
if (list.length === 0) {
  existing = null;
} else if (list.length === 1) {
  existing = list[0];
} else {
  // length > 1: cross-instance race 留下的多筆 allocation（議題 4.2e narrow risk）
  this.logger.error(`multiple wallet allocations found for deposit ${deposit.id}`, {
    depositId: deposit.id.toString(),
    allocationIds: list.map((allocation) => allocation.id),
  });
  existing = list[0]; // 取 [0]，多餘 allocation 交議題 7 reconciliation 清理
}
```

**Step 3.5 — Local-remote consistency fence**（fail-loud；新增 layer）

Step 3 後，只有兩種 `walletAllocationId` 狀態 legal 進入 step 4：

- **A.** `deposit.walletAllocationId === null`（catch-up mode；step 5 將 writeback `existing.id` 若 binding 已存在）
- **B.** `deposit.walletAllocationId === existing?.id`（已 bound；step 5 跳過 writeback）

任何其他組合視為上游程序錯誤。Worker **不**自動修正：log critical + skip + ack。對齊跨議題原則 8 與議題 4.2e fail-loud 精神延伸：所有異常都不 silently auto-correct，surface 給議題 7 reconciliation 處理。

```ts
if (deposit.walletAllocationId !== null) {
  if (existing === null) {
    // Orphan local id：local 認為有 allocation，wallet 端卻沒對應 fund case 的 allocation
    this.logger.error(
      `local walletAllocationId set but wallet has no allocation for deposit ${deposit.id}`,
      {
        depositId: deposit.id.toString(),
        localAllocationId: deposit.walletAllocationId.toString(),
      },
    );
    return; // skip + ack；不 auto-correct
  }
  if (deposit.walletAllocationId !== existing.id) {
    // ID mismatch：local id 與 fund case lookup 拿到的 wallet allocation id 不同
    this.logger.error(
      `local walletAllocationId mismatches wallet allocation id for deposit ${deposit.id}`,
      {
        depositId: deposit.id.toString(),
        localAllocationId: deposit.walletAllocationId.toString(),
        walletAllocationId: existing.id.toString(),
      },
    );
    return; // skip + ack；不 auto-correct
  }
}
// 進入 step 4：deposit.walletAllocationId === null OR (existing !== null AND deposit.walletAllocationId === existing.id)
```

**Step 4 — 9-cell matrix dispatch**（純依 `(deposit.status, existing)` 決策）

| `deposit.status` | `existing` | Action                                                                                 |
| ---------------- | ---------- | -------------------------------------------------------------------------------------- |
| `transacting`    | `null`     | `propose` with full plan lines                                                         |
| `transacting`    | `bound`    | skip（讓下一個 NT event 驅動 settle）                                                  |
| `transacting`    | `settled`  | skip + log critical（anomaly：wallet finalized before deposit transitioned；per OP14） |
| `completed`      | `null`     | `directSettle` with full plan + full outcome                                           |
| `completed`      | `bound`    | `settle` with full outcome（best-effort：Conflict → silent skip + warn log）           |
| `completed`      | `settled`  | skip（已同步）                                                                         |
| `failed`         | `null`     | `directSettle` with full plan + zero outcome                                           |
| `failed`         | `bound`    | `settle` with zero outcome（best-effort）                                              |
| `failed`         | `settled`  | skip（已同步）                                                                         |

不支援的 deposit status（如 `processing`）throw invariant violation。

`(_, bound)` settle 路徑的 `outcomeLines` 從 `existing.planLines`（plan-1 first-class field readback）建構，不從 deposit 端 reconstruct。

**Step 5 — Conditional writeback**（三條件）

| 條件                                                                                                             | Writeback target          |
| ---------------------------------------------------------------------------------------------------------------- | ------------------------- |
| Step 4 建了新 allocation（`propose` / `directSettle`）                                                           | `newAllocation.id`        |
| Step 4 對 existing allocation 動作或跳過（`settle` / skip-on-existing）AND `deposit.walletAllocationId === null` | `existing.id`（catch-up） |
| 其他（`deposit.walletAllocationId === existing.id`，已對齊）                                                     | no writeback              |

Catch-up writeback 適用所有 `existing !== null` 且 local 為 `null` 的 cell，包括 `(_, bound)` settle cell、`(_, settled)` skip cell、`(transacting, settled)` anomaly cell。只要 binding 已存在而 local 未寫入，writeback 就把 local 對齊 wallet truth。Mismatch case 已於 step 3.5 過濾，此處不會撞到。

Writeback 採 narrow patch method `updateWalletAllocationIdByIdInTx(id, walletAllocationId, tx)`，於 `transactionRunner.run(...)` callback 內執行；callback 內**禁止**任何 wallet RPC / queue dispatch / 外部 client 呼叫。

任何 step 1-5 失敗 → throw → queue 自動 retry。

##### 4.1g.4 議題 2 RPC Contract Amend 需求

Worker step 3 需要 `walletAllocationRpc.listByFundCase`：

```ts
type ListByFundCaseInput = {
  type: FundCaseType;
  identifier: IntegerString;
};

type ListByFundCaseOutput = ResultDto<CommonCode.Ok, WalletAllocationDto[]>;
```

Read-only 添加，不破壞議題 2 既凍結 spec。具體 contract 落地在 `walletAllocationRpc` 第 7 個 method（與其他 6 個 method 並列）。

Cardinality 設計：collection-shape 而非 single-shape。理由：

- 議題 4.2e 接受 cross-instance race narrow risk 留下多筆 allocation，設計上接受 1-to-many narrow case
- 議題 7 機制 #4 reconciliation 偵測「同 fund case 對應多筆 allocation」需 list shape 才能 surface 多筆
- Single-shape RPC 在 worker step 3 catch-up 拿到第一筆即返回，會 silently 漏看第二筆，反而增加 reconciliation 偵測成本

Caller-side 約束：

- Worker step 3 catch-up：caller 預期 max-1，`length > 1` 時 log critical，並取 `[0]` 繼續流程；多餘 allocation 交議題 7 reconciliation 清理
- 議題 7 reconciliation cron：caller 預期可見多筆，直接用 list 結果走清理邏輯
- Repository `WalletAllocationRepository.listByFundCase(...)` 返回 deterministic `ORDER BY id ASC`；不在 repository / RPC 邊界 assert max-1

##### 4.1g.5 Worker 實作位置

> **[Amended after plan-3 land]** 此段落於 plan-3 落地後 amend：worker placement 從單檔 `feature/deposit/worker/deposit-wallet-sync.worker.ts` 改為「queue adapter（三檔分離）+ business orchestration use-case（feature 內）」雙層拆分，對齊 codebase 既有 `withdrawal-intent-acceptance` / `withdrawal-intent-instruction` queue convention。Adapter 只負責 job decode / mutex / error handling / ack；business 決策（step 1-5 流程、step 3.5 fence、9-cell 矩陣、conditional writeback）全部在 use-case 內。

Worker placement（雙層拆分）：

**Queue adapter（三檔分離；service-root 下）：**

```text
apps/esingpay-cradle/src/fund/queue/deposit-wallet-sync/
├── definition.ts
├── deposit-wallet-sync.queue-dispatcher.ts
└── deposit-wallet-sync.queue-worker.ts
```

**Business orchestration（deposit feature 下）：**

```text
apps/esingpay-cradle/src/fund/feature/deposit/use-case/
└── process-deposit-wallet-sync.use-case.ts
```

Worker adapter 觸發即：

```ts
await this.processDepositWalletSyncUseCase.execute({
  depositId: BigInt(job.data.depositId),
});
```

Module wiring：plain `@Injectable()` providers（**不**用 `@nestjs/bull` / `BullModule`），手動 instantiate BullMQ `Queue` / `Worker`。對齊既有 `queue-dispatcher.module.ts` / `queue-worker.module.ts` / `queue.module.ts` 結構。

DDD 邊界：queue adapter ≠ business orchestration。Adapter 屬於 infrastructure layer 的 inbound message handler；use-case 屬於 application layer 的 business orchestration unit。雙層拆分讓 unit test 可只 mock use-case，不需 spawn BullMQ infrastructure。

##### 4.1g.6 Cutoff 狀況：worker 看到三種 walletAllocationId 的時機

> **[Amended after plan-3 land]** 此 cutoff table 於 plan-3 落地後 amend：`bigint > 0` 行為不再是 unconditional skip。Worker 對 `bigint > 0` 的處理依「step 3.5 fence + (deposit.status, existing) 9-cell 矩陣」決定。fence 通過後進入矩陣分流，可能走 settle / skip / log critical 不同分支。詳見 §4.1g.3 amend 後的完整 6-step 流程。

| Worker 看到  | 來源情境                                     | 進入流程的 stage                                                                           | 頻率   |
| ------------ | -------------------------------------------- | ------------------------------------------------------------------------------------------ | ------ |
| `null`       | path A/C/E 剛建好的 new deposit              | sync（step 4 矩陣）                                                                        | 主路徑 |
| `null`       | 上一輪 sync 失敗、queue 自身 retry 中        | sync / catch-up（step 4）                                                                  | 失敗時 |
| `null`       | path D/F 對 existing pending（罕見）         | sync（step 4 矩陣）                                                                        | 極罕見 |
| `0n`         | legacy data backfill                         | step 2 legacy guard skip                                                                   | 罕見   |
| `bigint > 0` | local-remote 一致（`existing.id === local`） | step 3.5 fence 通過 → step 4 矩陣分流（settle / skip 視 `(status, existing.status)` 而定） | 偶發   |
| `bigint > 0` | orphan local id（`existing === null`）       | step 3.5 fence reject → log critical + skip + ack                                          | 罕見   |
| `bigint > 0` | id mismatch（`existing.id !== local`）       | step 3.5 fence reject → log critical + skip + ack                                          | 罕見   |

關鍵觀察（amended）：

- `null` 永遠來自 use case path A/C/E 顯式寫入；使用者主動建立的「sync pending」訊號
- `0n` 永遠來自 migration backfill；legacy data 的 immortal sentinel
- `bigint > 0` 不必然代表 already-synced，可能是 path D/F 已 bound 但尚未 settle 的場景；fence + 矩陣決定實際行為
- 三種 schema 訊號（`null` / `0n` / `bigint > 0`）由 worker 端解讀；application code 其他位置無 inspect 負擔

##### 4.1g.7 不在本議題範圍

- Worker 內部 deposit allocation line builder 細節（amount-effect dispatch / service placement）→ 議題 4.3
- Reconciliation cron（dead letter 清理 / cross-instance race 孤兒偵測）→ 議題 7
- DB unique constraint on `(fundCaseType, fundCaseIdentifier)` 作為 hard guard → 未來 TODO

---

## 議題 4.1 完整結束 ✅

---

## 4.2 失敗補償拓撲

由於 4.1 將 wallet 同步推到 queue，失敗模式統一為 queue retry，4.2 範圍大幅縮小。本節確認失敗點分類、補償責任歸屬、以及與議題 3 補償拓撲的對比。

### 4.2a 失敗點分類

deposit ↔ wallet sync 的失敗點分為兩類：

**Class 1：Use case path 內部失敗**

- mutex / recognizableCheck / resolveWallet / persist 失敗
- 處理：throw，NT event 沒 ack，自然重投
- Wallet 端無動作、無孤兒
- **不屬於議題 4.2 範圍**——是 deposit feature 既有失敗模式

**Class 2：Worker step 1-5 內部失敗**

- DB read fail / mutex 撞 deadlock / wallet RPC fail / DB write fail
- 處理：worker throw → queue 自動 retry
- 重試後可能撞到的 wallet 端狀態：
  - 完全沒動作（step 3 read fail / step 4 RPC fail before reach wallet）
  - 已建 allocation 但 worker 還沒 writeback（step 4 RPC partial / step 5 DB write fail）
- **本議題範圍核心**

### 4.2b Class 2 失敗的 Catch-up 機制

> **[Amended after plan-3 land]** 此段落於 plan-3 落地後 amend：catch-up 不再是「step 3 拿到 existing → 寫回 allocationId 後 return；step 4-5 跳過」單一行為。Path D / F transacting → completed / failed 且已 bound 的場景下，catch-up 後仍需走 step 4 矩陣的 settle 分支（`(completed, bound)` / `(failed, bound)`），並由 step 5 conditional writeback 補回 local id。Catch-up 與「正常進入 step 4」不再是互斥分支，而是**同一條 6-step pipeline 的不同 cell**。

Worker step 4 RPC 成功 + step 5 DB writeback 失敗的場景下，wallet 端有 allocation 但 deposit 端 `walletAllocationId` 仍 `null`。Queue retry 拉到同一 message：

```text
Step 1: read deposit → walletAllocationId 仍 null
Step 2: legacy guard skip 條件不滿足（null !== 0n）
Step 3: listByFundCase → 拿到 existing list（前一輪 RPC 留下的）
  → length > 1 時 log critical + 取 [0]；length === 1 時 existing = list[0]
Step 3.5: fence 通過（local null + existing 任意）
Step 4: 依 (deposit.status, existing.status) 矩陣分流：
  - (transacting, bound)：skip（讓下一個 NT event 驅動 settle）
  - (completed, bound)：settle with full outcome（從 existing.planLines 建構）
  - (failed, bound)：settle with zero outcome
  - (_, settled)：skip（已同步）
  - (transacting, settled)：skip + log critical（anomaly cell）
Step 5: conditional writeback existing.id（catch-up 條件命中）
```

Partial commit 的恢復納入 6-step 流程的自然分支，不需要額外機制。Worker 流程本身已涵蓋 catch-up 與 forward sync 兩種 mode。

### 4.2c 失敗補償的責任歸屬

| 補償類型                                     | 責任歸屬                    | 落地時機     |
| -------------------------------------------- | --------------------------- | ------------ |
| 單次 worker 失敗的 retry                     | Queue retry policy          | **本次實作** |
| Retry exhaustion 的 dead letter              | Queue dead letter mechanism | **本次實作** |
| Dead letter 的人工 / cron 清理               | 議題 7 reconciliation       | 議題 7       |
| Cross-instance race 留下的多筆 allocation    | 議題 7 reconciliation       | 議題 7       |
| DB unique constraint on `(type, identifier)` | Wallet phase 4 amend        | 未來 TODO    |

註：worker step 3 若 `listByFundCase` 結果 `length > 1`，會 log critical surface 給議題 7 reconciliation。

### 4.2d Queue Retry Policy（建議大方向，實作時細化）

- 短期 transient（DB hiccup / RPC timeout）：exponential backoff（建議 1s / 5s / 30s / ...）
- 最大 retry 次數：N 次後送 dead letter（具體 N 待 codebase 既有 acceptance queue retry policy 參考）
- Dead letter：log + alert，不自動丟棄

具體 retry policy 數字在實作 phase plan 時依 codebase 既有 queue infrastructure 慣例決定，不在本議題凍結。

### 4.2e Cross-instance Race 接受聲明

> **[Amended after plan-3 land]** 此段落於 plan-3 落地後 amend：除原本「`length > 1`（duplicate allocations）」一種異常範式外，補完 plan-3 step 3.5 fence 處理的另外兩種異常（orphan local id + id mismatch）。三種異常統一採「fail-loud surface + 議題 7 reconciliation 兜底」設計線。所有異常都不 silently auto-correct，而是 surface 給 reconciliation 處理。

#### 三種 cross-instance / 程序異常處理範式

| 異常類型                          | 觸發條件                                                                            | Worker 行為                                                                                                               | Surface 機制               |
| --------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | -------------------------- |
| **Duplicate allocations**（既有） | `listByFundCase` 返回 `length > 1`                                                  | log critical + 取 `[0]` + continue                                                                                        | 議題 7 reconciliation cron |
| **Orphan local id**               | local `walletAllocationId > 0n` AND `existing === null`                             | log critical（含 `depositId, localAllocationId`）+ skip + ack；**不**呼叫 wallet mutation；**不** writeback；**不** throw | 議題 7 reconciliation cron |
| **ID mismatch**                   | local `walletAllocationId > 0n` AND `existing !== null` AND `local !== existing.id` | log critical（含 `depositId, localAllocationId, walletAllocationId`）+ skip + ack；同上                                   | 議題 7 reconciliation cron |

接受 narrow risk（amended）：

- 在 in-process queue + single-consumer dispatch 機制下，三類異常發生機率均極低
- 後果：多餘 / 失聯的 allocation（不破壞會計事實）
- 偵測 + 清理：議題 7 reconciliation 從 critical log surface 偵測，後續清理 / 修復
- 長期 hard guard：DB partial unique index on `(fund_case_type, fund_case_identifier) WHERE fund_case_identifier IS NOT NULL` 已列未來 TODO；migration 後 duplicate allocations 異常將被 wallet-side schema 阻擋於建立階段

#### 設計統一線

「**fail-loud surface + 議題 7 reconciliation 兜底**」是這三種異常的共同設計線。Worker 看到任一異常時：

1. 不 throw（throw + retry 在無 forward-progress path 下，會於 `JOB_ATTEMPTS` 耗盡後 silently 失敗）
2. 不 auto-correct（mismatch / orphan 都被視為上游程序錯誤；自動修正會掩蓋根因）
3. log critical + skip + ack（job 從 queue 移除、停止 retry；critical log 是 reconciliation 的唯一入口）

此一設計線同時對齊跨議題原則 8（「不依 allocation 同步狀態決策」）的精神延伸。worker 雖必須 inspect `walletAllocationId` 做 fence 決策，但不允許據此 auto-correct，僅允許 fail-loud surface 後等待議題 7 處理。

### 4.2f 議題 3 vs 議題 4 補償拓撲對比

| 維度                        | 議題 3（withdrawal, inline sync）                                 | 議題 4（deposit, queue sync）                 |
| --------------------------- | ----------------------------------------------------------------- | --------------------------------------------- |
| 同步機制                    | Inline RPC 序列                                                   | Queue + worker                                |
| 失敗類型                    | 「夭折」bound 孤兒 / 「settle stuck」                             | Worker retry / Catch-up / Cross-instance race |
| 補償複雜度                  | 高：path-by-path throw + cron retry + 孤兒清掃 + 策略 B with stub | 低：queue retry + catch-up，自然涵蓋大部分    |
| 為什麼複雜度不對稱          | reservation flow + caller need allocationId before persist        | direct flow + 業務不阻擋設計                  |
| 兩端是否需要 contract amend | 已凍結，無 amend                                                  | 議題 2 amend 加 `listByFundCase`              |

複雜度不對稱反映業務本質的不對稱，不是設計疏失。withdrawal 的補償複雜度由 wallet allocation spec 的 reservation flow 決定，無法簡化；deposit 的補償複雜度因為 queue 切分而集中可控。

---

## 議題 4.2 完整結束 ✅

---

## 4.3 Allocation Lines 構造（含 fee 拓撲）

Worker step 4 在三種 deposit status 下構造 lines 的設計細節。本議題範圍：line 結構（含 fee 拓撲）/ plan vs outcome 對應關係 / failed 場景處理 / builder 簽名與位置。

### 4.3a Lines 基本結構（4 條 gross-record lines）

採 **gross recording** 會計慣例：external 進來的 amount 全額先記入 available bucket，再從 available 分流 fee 到 held bucket。對齊 PM ledger view 對「先收全額、再截留 fee」的會計事實序列認知。

```text
L1  external source                  — principal,    amount = deposit.amount
L2  internal destination (available) — principal,    amount = deposit.amount
L3  internal source      (available) — service_fee,  amount = deposit.amountFee
L4  internal destination (held)      — service_fee,  amount = deposit.amountFee
```

帳務恆等：

- principal envelope: external in (amount) == internal destination (amount) ✓
- service_fee envelope: internal source (amountFee) == internal destination (amountFee) ✓
- 商戶 wallet net delta: available + amount - amountFee = +amountNet；held + amountFee
- 整體 net: amount = amountNet + amountFee（survey §5 既存事實）

#### 為什麼採 gross recording 而非 net recording

| 維度               | Gross recording（採用）        | Net recording（不採用）                     |
| ------------------ | ------------------------------ | ------------------------------------------- |
| Lines 條數         | 4                              | 3（external in / available net / held fee） |
| 業務事件序列       | 完整保留（收全額 → 截留 fee）  | 跳過 fee movement，直接記淨額               |
| Audit / tax 友好度 | 高（fee 截留作為獨立事件可查） | 低（fee 動作隱藏）                          |
| Projection 邏輯    | Trivial pair-up                | 需要 fabricate 不存在的 movement            |
| 會計慣例           | 大多數正規系統採用             | 簡化系統採用                                |

採 gross recording 是會計記帳的選擇，不是「為了 PM view 而屈就」。

### 4.3b Service Fee 拓撲（併入 4.3a 結構）

Service fee（`amountFee`）進**商戶 wallet 的 held bucket**，不進獨立 platform service fee wallet。

理由：

- spec §5.3 字面：「service fee movement into `held`」明確指 held bucket
- Wallet 內部「同 wallet 不同 bucket 表示資金歸屬」是既有設計（available = 商戶可用 / held = 暫扣歸 platform）
- 不需要新增「platform service fee accumulation wallet」概念，與議題 5 FundCase Context Shape 解耦
- 不需要新增 `getServiceFeeRecipientWalletId` RPC

Held bucket 的會計語意：「位於商戶 wallet table 但不可動用，未來會被 platform settlement 機制結算」。具體 settlement 機制不在本議題範圍。

### 4.3c Plan vs Outcome 對應關係

| 場景                           | Plan (declared) | Outcome (realized) | Plan == Outcome ? |
| ------------------------------ | --------------- | ------------------ | ----------------- |
| Path A → D（success）          | deposit.amount  | deposit.amount     | ✅                |
| Path C（success born settled） | deposit.amount  | deposit.amount     | ✅                |
| Path F（fail in transit）      | deposit.amount  | 0                  | ❌                |
| Path E（fail born failed）     | deposit.amount  | 0                  | ❌                |

#### 術語精確化：declared vs realized

- **Plan = declared amount**：來自 deposit.amount，NT broadcast 即固定的鏈上宣告金額。Failed 不影響此值（事實仍然發生過：「鏈上宣告會來 X」）。
- **Outcome = realized amount**：實際進入系統的資金。Success 場景 = deposit.amount；failed 場景 = 0（資金未進系統）。

「Actual」一詞只適用於 outcome 一邊；plan 用 declared 描述更精確。

#### 為什麼 failed 場景下 plan ≠ outcome 是業務需求

PM ledger view 對 failed deposit 的呈現需求：「我們預期 X 但失敗了」。這個資訊對 audit / accounting / reconciliation 有實質價值。

若 path E 不寫 plan（spec 字面），ledger 看不到 declared amount，僅顯示 outcome zero——PM 看到 zero rows 但不知道原本預期金額。Path F 與 Path E 同樣是 failed，但 path F 因經過 propose 階段有 plan record、path E 沒有，**ledger 顯示能力不對等**。

設計修正：path C/E 走 directSettle 時也寫 planLines，讓所有 deposit path 在 ledger 都能呈現 declared vs realized 對比。

#### 對 spec §7.6 directSettle 的 amend 需求

| Spec layer        | Before                                               | After                                                           |
| ----------------- | ---------------------------------------------------- | --------------------------------------------------------------- |
| §7.6 directSettle | allocation born settled, only outcomeLines persisted | allocation born settled, planLines + outcomeLines 同時 set-once |
| §3 planLines      | reserve / propose 時 set-once                        | reserve / propose / directSettle 時 set-once                    |

#### Plan 在 direct flow 是「宣告」性質（不違反 spec 設計意圖）

| Plan 類型                                 | Lock 資金 | 業務語意                 | 時間性     |
| ----------------------------------------- | --------- | ------------------------ | ---------- |
| Reservation flow plan                     | ✅        | 「預期將要」             | 強未來時態 |
| Direct flow propose plan（path A）        | ❌        | 「預期接收」（進行中）   | 中度時態   |
| Direct flow directSettle plan（amend 後） | ❌        | 「宣告接收」（已知事實） | 弱時間性   |

Direct flow 的 plan 沒有 lock 約束，純粹是宣告；事後宣告與事前宣告差別只是時間順序，事實本質一致（皆基於 deposit.amount）。**這不是 fake history，是弱時間性下的 reordering**。

### 4.3d Failed Outcome 處理：Zero outcomeLines 不產 Entry

#### 設計

Failed deposit 的 directSettle / settle 下：

- `WalletAllocation.planLines` 寫入 4 條 declared amount lines（基於 deposit.amount / amountFee）
- `WalletAllocation.outcomeLines` 寫入 4 條 zero amount lines
- `WalletAllocation.status` → `settled`
- **不產** `WalletEntry` / `WalletEntryLine`
- `WalletAssetBalance` 不變動（無 watermark 推進，因無 entry）
- Allocation envelope check 仍執行（0 ≤ declared，永遠 pass）

#### 設計原則：Allocation 與 Entry 解耦

兩個資源的職責分明：

| 資源             | 角色                                | 是否寫入 zero                    |
| ---------------- | ----------------------------------- | -------------------------------- |
| Allocation lines | Business accounting fact            | 寫入（zero 是事實）              |
| Entry lines      | Balance mutation log（append-only） | 不寫入（zero 不 mutate balance） |

「Failed deposit 仍寫 plan + zero outcome 到 allocation」與「不產 zero entry」並不衝突——前者是業務事實記錄，後者是 balance mutation log；解耦後各自遵循自己的 invariant。

#### Spec amend 補完

Spec §7.5 / §7.6 既有描述「立即建立 entry」是針對典型場景（有 movement）。Zero outcome 是 spec 沒明確涵蓋的 corner case，補規則：

- 若 outcomeLines 全為 zero，allocation 仍 settled、outcomeLines 仍寫入，但**不產 entry / 不更新 balance**
- 若 outcomeLines 有任何非 zero 條目（含部分 zero），entry 正常產生

### 4.3e Deposit Builder Pattern

> **[Amended after plan-3 land]** 此段落於 plan-3 落地後 amend：deposit builder pattern 從 entity-in 雙 method（`buildDepositPlanLines(deposit)` / `buildDepositOutcomeLines(deposit)`）改為 amount-effect dispatch 三 method + scalar input + planLines-based outcome 範式，對齊 plan-2 phase 7 `WithdrawalIntentAllocationLineBuilderService`（per guide G1 patch + 跨議題原則 12 amend）。Caller 依「事實層」（`deposit.status === Completed` → `buildFullOutcomeLines`；`deposit.status === Failed` → `buildZeroOutcomeLines`）選 method，不在 builder 內 inspect status。

#### 三 method amount-effect dispatch

```ts
buildPlanLines(input: BuildDepositPlanLinesInput): WalletAllocationLine[];
buildFullOutcomeLines(input: { planLines: WalletAllocationLine[] }): WalletAllocationLine[];
buildZeroOutcomeLines(input: { planLines: WalletAllocationLine[] }): WalletAllocationLine[];
```

Method-name semantics（amount-effect dispatch）：

- `buildFullOutcomeLines`：outcome amounts equal plan amounts（success outcome — completed deposit）
- `buildZeroOutcomeLines`：outcome amounts 全為 `'0'`（no-realization outcome — failed deposit）

#### Plan builder input shape（scalar，不 take `Deposit` entity）

```ts
type BuildDepositPlanLinesInput = {
  recipientWalletId: bigint;
  currencyCode: CurrencyCode;
  amount: NumericString;
  amountFee: NumericString;
};
```

Input 規則：

- 直接使用 Deposit raw 欄位名（`recipientWalletId` / `currencyCode` / `amount` / `amountFee`），不在 builder 內 alias
- **不**包含 `Deposit` 物件本身；caller extract scalar facts
- **不**包含 `amountNet`（gross recording — 4 lines 的 lines 2 + 3 net effect 自然產出 `+amount - amountFee`）
- **不**包含 `networkTransactionId`（builder 不負責 projection / network reference）

#### Outcome builder input shape（planLines-only）

```ts
type BuildFullDepositOutcomeLinesInput = { planLines: WalletAllocationLine[] };
type BuildZeroDepositOutcomeLinesInput = { planLines: WalletAllocationLine[] };
```

Outcome shape 完全由 plan-line identity preservation + all-zero-vs-preserve 決定，不需其他 deposit fact。Caller 依事實層 dispatch：

- `(completed, null)` directSettle：caller `buildPlanLines(...)` → 立即 `buildFullOutcomeLines({ planLines })`
- `(failed, null)` directSettle：caller `buildPlanLines(...)` → 立即 `buildZeroOutcomeLines({ planLines })`
- `(completed, bound)` settle：caller 從 `existing.planLines`（wallet readback；plan-1 first-class field）→ `buildFullOutcomeLines({ planLines: existing.planLines })`
- `(failed, bound)` settle：同上 → `buildZeroOutcomeLines({ planLines: existing.planLines })`

#### Plan lines（4 條 gross-record lines；shape 不變）

| #   | side        | space    | walletId            | bucket    | currencyCode   | amount      | kind        |
| --- | ----------- | -------- | ------------------- | --------- | -------------- | ----------- | ----------- |
| 1   | source      | external | `null`              | `null`    | `currencyCode` | `amount`    | principal   |
| 2   | destination | internal | `recipientWalletId` | available | `currencyCode` | `amount`    | principal   |
| 3   | source      | internal | `recipientWalletId` | available | `currencyCode` | `amountFee` | service_fee |
| 4   | destination | internal | `recipientWalletId` | held      | `currencyCode` | `amountFee` | service_fee |

規則：

- Gross recording：principal 用 `amount`（不用 `amountNet`），available net effect 由 lines 2 + 3 自然 emerge
- Service fee 用 `amountFee` 並 reclassify `available → held`（lines 3 + 4，merchant 自有 wallet 的 held bucket，不另開 platform service-fee wallet）
- External lines（line 1）`walletId = null`、`bucket = null`
- Internal lines 只用 `available` / `held`；不用 `locked` / `held_locked`（後者屬 reservation flow，deposit 是 direct flow）
- Lines emit 順序固定（依上表），不動態 sort

#### Worker step 4 完整 call sites（amend 後對齊 plan-3 §12.6）

```ts
// (transacting, null) → propose
const planLines = depositAllocationLineBuilder.buildPlanLines({
  recipientWalletId: deposit.recipientWalletId,
  currencyCode: deposit.currencyCode,
  amount: deposit.amount,
  amountFee: deposit.amountFee,
});
walletAllocationRpc.propose({
  fundCaseType: FundCaseType.Deposit,
  fundCaseIdentifier: deposit.id.toString(),
  type: WalletAllocationType.Deposit,
  planLines,
});

// (completed, null) → directSettle full
const fullPlanLines = depositAllocationLineBuilder.buildPlanLines({
  recipientWalletId: deposit.recipientWalletId,
  currencyCode: deposit.currencyCode,
  amount: deposit.amount,
  amountFee: deposit.amountFee,
});
const fullOutcomeLines = depositAllocationLineBuilder.buildFullOutcomeLines({
  planLines: fullPlanLines,
});
walletAllocationRpc.directSettle({
  fundCaseType: FundCaseType.Deposit,
  fundCaseIdentifier: deposit.id.toString(),
  type: WalletAllocationType.Deposit,
  planLines: fullPlanLines,
  outcomeLines: fullOutcomeLines,
});

// (failed, null) → directSettle zero
const zeroPlanLines = depositAllocationLineBuilder.buildPlanLines({
  recipientWalletId: deposit.recipientWalletId,
  currencyCode: deposit.currencyCode,
  amount: deposit.amount,
  amountFee: deposit.amountFee,
});
const zeroOutcomeLines = depositAllocationLineBuilder.buildZeroOutcomeLines({
  planLines: zeroPlanLines,
});
walletAllocationRpc.directSettle({
  fundCaseType: FundCaseType.Deposit,
  fundCaseIdentifier: deposit.id.toString(),
  type: WalletAllocationType.Deposit,
  planLines: zeroPlanLines,
  outcomeLines: zeroOutcomeLines,
});

// (completed, bound) → settle full (best-effort)
const settledFullOutcomeLines = depositAllocationLineBuilder.buildFullOutcomeLines({
  planLines: existing.planLines,
});
walletAllocationRpc.settle({ allocationId: existing.id, outcomeLines: settledFullOutcomeLines });

// (failed, bound) → settle zero (best-effort)
const settledZeroOutcomeLines = depositAllocationLineBuilder.buildZeroOutcomeLines({
  planLines: existing.planLines,
});
walletAllocationRpc.settle({ allocationId: existing.id, outcomeLines: settledZeroOutcomeLines });
```

#### Builder placement

> **[Amended after plan-3 land]** builder placement 從 `worker/build-deposit-lines.ts` pure function 模組改為 injectable application service，對齊 plan-2 phase 7 `WithdrawalIntentAllocationLineBuilderService` placement convention。Service 仍然 pure relative to inputs（無 repository / RPC client / queue / transaction / clock / logger / 任何 side effect），DI pattern 為 future fee-variant injection / unit test mocking 提供 surface。

Builder 落點：

```text
apps/esingpay-cradle/src/fund/feature/deposit/service/deposit-allocation-line-builder.service.ts
```

Service rules：

- `@Injectable()` provider；deterministic + pure relative to inputs
- 無 repository / RPC client / queue / transaction / clock / logger / side effect
- 返回 `WalletAllocationLine[]` from `@esingpay/contract-base`（不建 Fund-local DTO duplicate）
- 用 `NumericString` 處理金額；不引入 bigint amount math
- 用 contract-base enums（`WalletAllocationSide` / `WalletAllocationSpace` / `WalletBucket` / `WalletAllocationLineKind`）；不建 Fund-local duplicate
- Lines emit 順序固定（依 §4.3e plan lines 表），不動態 sort

Service 註冊位置：deposit feature module（與 `DepositFeeCalculationService` / `DepositRecognitionService` 同 module，按既有 application service convention）。

Module wiring 不引入 `@nestjs/bull` / `BullModule`；plain `@Injectable()` provider 即可。

---

## 議題 4.3 完整結束 ✅

---

## 議題 4 完整結束 ✅

---

## 5. FundCase Context Shape

範圍：fund 對 wallet 暴露 fund case context 的接口設計，讓 wallet ledger composer 能 hydrate 出 PM ledger view 所需的「Case 描述」/「業務狀態」/「Serial Number 片段」等欄位（議題 1.4a 已凍結需求）。

---

### 5.1 Generic vs Per-type RPC Endpoint 大方向 ✅

#### 5.1a 決議

Fund 對 wallet 暴露 fund case context 採 **per-type RPC** 路線：

- 重用既有 `depositRpc.getById`（已存在於 `libs/contract-rpc/src/lib/fund/rpc/deposit.rpc.ts`）
- 新建 `withdrawalIntentRpc.getById`（fund 端目前完全沒有此 RPC contract / DTO file，本輪一併建立）
- 未來新 fund case type（payment / refund 等）各自加 `<resource>Rpc`

#### 5.1b 決策依據

兩個 frame 的拉扯：

- **Frame 1（ubiquitous language 第一性）**：wallet composer 用 fund case 級別意圖思考，被迫呼叫 type-specific RPC 是 first-class 表達缺席；偏好 generic `fundCaseRpc.getByRef`
- **Frame 2（誠實反映 + paradigm consistency）**：fund 內部就是 per-aggregate；hydration 必然 type-specific（DTO 欄位不同）；偏好 per-entity RPC

關鍵 anchor 推向 Frame 2：

1. **Per-entity RPC 必然會擴展 get 以外的 method**——PSP 系統每個 entity 對外都會擴自家專屬 operations / queries。選 generic 路線等於 fund 對外 API surface 同時並存「unified hydration RPC（`fundCaseRpc`）」+「per-entity operation RPCs（`depositRpc` / `withdrawalIntentRpc` / ...）」，形成 paradigm split。Per-type 路線則 paradigm consistent——每個 entity 的 hydration + operations 都在同一個 RPC contract 下。

2. **Hydration 的 type narrowing 不可避免**：generic RPC 回傳必然是 polymorphic envelope（discriminated union of `DepositDto | WithdrawalIntentDto | ...`），wallet composer 拿到後仍要按 discriminator narrow 才能拼 ledger DTO 的 type-specific fields。「generic 省下分支」的便宜並不存在。

3. **Ubiquitous language 已多處 first-class 具體化**：`FundCaseType` enum、reference shape `{ type, identifier }`、REST wrapper `fundCase: { type, identifier, subject? }`（議題 2.3c）、wallet 端 `listByFundCase`（議題 4.1g.4）。Hydration RPC 走 per-entity **不會讓 ubiquitous language 缺乏表達**——只是 hydration 不在 RPC layer 額外拉抽象。

4. **Fund 端目前是「半完成品」**：既有 `depositRpc` 採 `DepositDto | null` 風格，無 envelope、無 fund-specific code enum；`withdrawalIntentRpc` 完全不存在；`guide/contract-structure.md` 規範要求 envelope。本輪正是 fund 端 RPC 完整性對齊規範的時機，但對齊的 paradigm 應該是 per-entity（沿用既有結構），不強行 generic。

#### 5.1c Wallet Composer 處理模式

Wallet ledger composer 從 page result 收到 `WalletLedgerItem[]` 後：

1. 按 `subjectType` 分組（`deposit` group / `withdrawal_intent` group / 未來新 type group）
2. 各 group 用對應的 fund RPC client batch 取 DTO（`depositRpcClient.getById` / `withdrawalIntentRpcClient.getById`）
3. 各別投影到 wallet-side fund case subject DTO（具體 shape 5.2 / 5.6 處理）
4. 拼進最終 `WalletLedgerItemDto`

「按 subjectType 分支」並非 anti-pattern——hydration 場景 wallet 必須解讀 type 才能填 ledger row 的 type-specific fields，這是 hydration scenario 的本然，不是「替 fund 維護字典」（fund-case-proposal 警告的反模式）。

#### 5.1d 接口不對稱的定位

Wallet 端 `listByFundCase`（議題 4.1g.4）是 generic 接 `fundCaseType + fundCaseIdentifier`；fund 端 fund case context hydration 是 per-entity——**這個接口層不對稱是有意設計**：

- Wallet 端 generic 反映 **wallet allocation table 的 polymorphic structure**：一個 allocation table 對應多 fund case type，從業務 identifier 反查 allocation 是 wallet 內生的 polymorphic operation
- Fund 端 per-entity 反映 **fund 各 aggregate 獨立業務結構**：Deposit / WithdrawalIntent 各自獨立 service / repository / model（FundCase ≠ write-side aggregate 紅線），每個 entity 對外擴展自己的 operations / queries

兩端形式不對稱，但**各自誠實反映自家內部結構**。「接口形式對稱」不是設計目標——**「分清楚 > 形式對稱」**。本子議題提出此原則作跨議題候選 #13，待後續議題（5.x 或議題 6）收斂時評估正式 commit。

#### 5.1e 不在本子議題範圍

- 各 RPC 暴露的 DTO shape（Full `Serialized<Entity>` 重用 vs 設計 partial subject DTO）→ 5.2 / 5.6 合併議題
- `withdrawalIntentRpc` contract 落地（file placement / method signature / envelope 風格 — 依 `guide/contract-structure.md` 規範補完）→ 5.4
- Cross-BC dependency wiring（wallet ledger composer 注入多個 fund RPC client）→ 5.3
- Fund 端既有 `depositRpc` 是否同步 envelope 化（半完成品對齊規範）→ 5.4 一併處理

📌 延伸觀察（記下不展開）：`fund-case-concept-proposal` 將「`fundCaseRpc.getByRef` 統一介面 vs per-aggregate RPC + consumer dispatch」明確列為兩個 unresolved 方向。本子議題 5.1 的決議實際上是這個 proposal 留下的決策點之一，可在 fund-case-proposal 後續 amend 時 reference 本決議作為 RPC 層 closure。

---

## 議題 5.1 完整結束 ✅

---

### 5.2 per-entity RPC 暴露的 DTO shape（合併原 5.6）

5.6（「Deposit / withdrawal 對 wallet 暴露 fields 的具體選擇」）的設計問題與 5.2（「FundCase Context DTO shape — wallet 從 fund 拉什麼資訊」）是同一件事的兩個視角，合併處理。本子議題分兩個 sub-decisions：

- **5.2a（已收斂）**：fund 端 RPC DTO 風格 — 採 full `Serialized<Entity>`
- **5.2b（pending）**：wallet 端 `WalletLedgerFundCaseSubjectDto`（contract-rest layer）shape — wallet composer 從 fund DTO 投影的目標 partial shape

#### 5.2a Fund 端 RPC DTO 風格 ✅

**決議**：fund 端 per-entity RPC 採 **full `Serialized<Entity>` DTO** 風格，對齊 codebase-wide DTO convention。

**具體**：

- 既有 `DepositDto = Serialized<Deposit>` 維持不變
- 新建 `WithdrawalIntentDto = Serialized<WithdrawalIntent>`（檔案 placement 細節 5.4 落實）
- 未來新 fund case type 各自 `<Entity>Dto = Serialized<Entity>`

**決策依據**：

1. **Codebase-wide DTO convention 是 full `Serialized<Entity>`**——既有 fund `DepositDto`、wallet `WalletAllocationDto`（議題 2.3b）都這樣。選 full 對齊全域慣例。
2. **`fund-case-concept-proposal` 「subject 形狀變形由各 consumer 自決」**——fund 端對外保持 generic full DTO，consumer 投影成自己需要的 partial。Fund contract 不為單一 consumer wallet-aware。
3. **Over-fetch 是 codebase 整體 trade-off**——`statusHistory[]` 等 fields 在 ledger list view 確實 over-fetch，但這是 codebase 全 RPC 共有的取捨，不是本議題該翻盤的。
4. **YAGNI**：未來若 over-fetch 真成痛點，可在 fund 端 codebase-wide 引入 `<Entity>SummaryDto` 變體（不只 wallet 用，其他 list caller 都受惠）；現在不超前優化。

**對 wallet 端的影響**：

Wallet composer 拿到 full `DepositDto` / `WithdrawalIntentDto` 後，**自行投影成 wallet 端 partial subject DTO**（具體 shape 5.2b 處理）。投影邏輯與輸入 shape 解耦——未來若 fund 端 DTO 變動或引入 SummaryDto variant，只動 wallet composer 的 mapper 一處。

📌 延伸觀察（記下不展開）：未來引入 `<Entity>SummaryDto` 的 trigger condition 可參考——
（1）wallet ledger list view 的 wire size measurable regression
（2）多個 list-scenario caller 出現相似 over-fetch 情境
（3）`statusHistory[]` 等陣列欄位在 batch hydration 場景成為 measurable cost

#### 5.2b Wallet 端 `WalletLedgerItemFundCaseDto` shape ✅

**狀態**：DTO + REST API spec 已手刻 land 在 `libs/contract-rest/src/lib/wallet/`。本子議題追認既成事實並 codify 設計依據。

**整體 `WalletLedgerItemDto` 結構**（contract-rest layer 已實作）：

```ts
// libs/contract-rest/src/lib/wallet/dto/wallet-ledger-item.dto.ts

export type WalletLedgerItemDto = {
  id: ShortId;
  merchant: RelatedMerchantDto | null;
  wallet: RelatedWalletDto;
  currencyCode: SupportedCurrencyCode;
  bucket: WalletBucket.Available | WalletBucket.Held;
  kind: WalletLedgerItemKind;
  ownerType: WalletLedgerItemOwnerType;

  planAmountDelta: NumericString | null;
  outcomeAmountDelta: NumericString | null;
  realization: WalletLedgerItemRealization;

  side: WalletLedgerItemSide;
  source: WalletLedgerNodeDto;
  destination: WalletLedgerNodeDto;

  fundCase: WalletLedgerItemFundCaseDto;
  networkTransaction: WalletLedgerNetworkTransactionDto | null;

  effectiveAt: IsoDateTimeUtc;
  createdAt: IsoDateTimeUtc;
  updatedAt: IsoDateTimeUtc;
};
```

**Fund case wrapper + subject DTO**：

```ts
// libs/contract-rest/src/lib/wallet/dto/wallet-ledger-fund-case.dto.ts

export type WalletLedgerItemFundCaseDto = {
  type: FundCaseType;
  identifier: ShortId;
  subject: WalletLedgerItemFundCaseSubjectDto;
};

export type WalletLedgerItemFundCaseSubjectDto = {
  id: ShortId;
  status: WalletLedgerFundCaseSubjectStatus;
  finalizedAt: IsoDateTimeUtc | null;
  createdAt: IsoDateTimeUtc;
  updatedAt: IsoDateTimeUtc;
};
```

**Row-level node DTO**（row 自己的 source / destination 用，**非** fund case subject 的 source / destination）：

```ts
export type WalletLedgerNodeDto = WalletLedgerInternalNodeDto | WalletLedgerExternalNodeDto;

export type WalletLedgerInternalNodeDto = {
  space: WalletLedgerItemSpace.Internal;
  wallet: RelatedWalletDto;
  merchant: RelatedMerchantDto | null;
  bucket: WalletBucket.Available | WalletBucket.Held;
};

export type WalletLedgerExternalNodeDto = {
  space: WalletLedgerItemSpace.External;
};
```

**`WalletLedgerNetworkTransactionDto`**（一併補完原議題 1 延伸觀察「具體節錄欄位待 1.4 query API + composer 設計時細化」）：

```ts
// libs/contract-rest/src/lib/wallet/dto/wallet-ledger-network-transaction.dto.ts

export type WalletLedgerNetworkTransactionDto = {
  id: ShortId;
  type: NetworkType;
  provider: NetworkProvider;
  /** identifier on the provider */
  identifier: string;
  status: NetworkTransactionStatus;
  direction: NetworkTransactionDirection;
  source: NetworkSourceDto | null;
  destination: NetworkDestinationDto;
  createdAt: IsoDateTimeUtc;
  updatedAt: IsoDateTimeUtc;
};
```

**設計依據**：

1. **對齊 `fund-case-concept-proposal` 的 `{ type, identifier, subject }` wrapper 形狀**——proposal 建議的跨 domain 標準 reference shape
2. **`fundCase: not nullable`** 反映 wallet allocation invariant——wallet ledger item 既然存在，對應 fund case 必然存在（allocation 必由 fund case 驅動）
3. **Subject 採 uniform shape**（不分 fund case type discriminated union）——wallet ledger 對 fund case 的需求是 **reference + 必要 metadata**（id / status / 時戳）；業務細節（from/to / 流向 / 業務分類）由 row level `source` / `destination` / `networkTransaction` / `kind` / `side` 投影完成，subject 不需 per-type 差異化
4. **`WalletLedgerFundCaseSubjectStatus` 大集合 enum**——直接 union 各 fund case type 的 status string literal values（與 deposit / withdrawal_intent status values 完全一致），verbatim 透傳。wallet contract-rest 端 owning enum，與 fund-side enum 解耦（不直接 import fund 端類型）。Caller 配合 `fundCase.type` 仍能精準對 status 子集判斷
5. **`identifier: ShortId`** 對齊三層識別字第三層（議題 2.3c 凍結：REST codec encoded short id）
6. **`networkTransaction: ... | null`** 反映「某些 ledger item 不對應 network tx」（純內部移動 / fee 累積），nullable 容忍

**Trade-off 容忍**：

- Fund 端加新 status value → wallet `WalletLedgerFundCaseSubjectStatus` 同步更新（infrequent；codified union 比 normalization 邏輯透明）
- Subject 不 carry per-type 細節 fields（wallet ledger 不需要，row level 已 cover）

**檔案 placement**：

- `libs/contract-rest/src/lib/wallet/dto/wallet-ledger-item.dto.ts`
- `libs/contract-rest/src/lib/wallet/dto/wallet-ledger-fund-case.dto.ts`
- `libs/contract-rest/src/lib/wallet/dto/wallet-ledger-network-transaction.dto.ts`

📌 延伸觀察（記下不展開）：

- Fund case 未來加新 type（payment / refund）影響 `WalletLedgerFundCaseSubjectStatus` 大集合 enum——需要在 wallet 端 enum 同步加 fund-side status values。Plan 撰寫時 freshman 要 visible 這個 ripple
- `wallet-ledger-item-impl-plan-draft.md` Phase 1 範圍實質已完成（contract-rest DTO + API definition 已 land）；draft 文件待 phase 5 後 amend 反映此狀態，或在 ep-2-plan-2 中重新評估 Phase 1 之後（Phase 2+）的拆分

---

## 議題 5.2 完整結束 ✅

---

### 5.4 Fund 端 RPC Contract 落地（含 envelope 規範對齊）✅

本子議題在 fund 端落實 5.1 / 5.2a 結論的具體 RPC contract，並順手把既有 `depositRpc` envelope 化（議題 5.1e 已 commit「5.4 一併處理」）。

#### 5.4a 三個 sub-decisions

1. **`withdrawalIntentRpc` methods 範圍**：只 `getById`（YAGNI；fund 內部目前無「從 network transaction 反查 withdrawal intent」需求；對稱性不是設計目標，依跨議題原則 13「分清楚 > 形式對稱」）
2. **`depositRpc` 同步 envelope 化**：Yes 本輪做。理由：
   - 議題 5.1e 已 commit
   - Caller 範圍可控（fund-rpc-survey §4 確認 wallet 端零依賴；fund 內部 server impl 改 mechanical）
   - 不做的話 fund 端 RPC 風格混雜（新 RPC envelope vs 舊 RPC nullable union）
3. **不建 fund-specific code enum**：只用 `CommonCode`（`Ok`）。Fund RPC 目前都是 read-side lookup，無 typed error space；未來 fund RPC 擴展 operation methods（cancel / approve）若需業務特定 codes 再加

#### 5.4b 新建 `withdrawalIntentRpc`

```ts
// libs/contract-rpc/src/lib/fund/rpc/withdrawal-intent.rpc.ts

export const withdrawalIntentRpc = defineRpc({
  namespace: NAMESPACE,
  id: 'withdrawal-intent',
  methods: {
    getById: defineRpcMethod<
      { id: IntegerString },
      ResultDto<CommonCode.Ok, WithdrawalIntentDto | null>
    >(),
  },
});
```

lookup miss 在這裡是 expected outcome，不升成 distinct error envelope。Caller 若把 `null` 視為 invariant violation，應由 caller context 自行 throw / log，而不是在 RPC contract 層 codify `NotFound`。

對應新建 DTO：

```ts
// libs/contract-rpc/src/lib/fund/dto/withdrawal-intent.dto.ts

export type WithdrawalIntentDto = Serialized<WithdrawalIntent>;
```

#### 5.4c `depositRpc` envelope 化

兩個 method 改 envelope return shape：

```ts
// libs/contract-rpc/src/lib/fund/rpc/deposit.rpc.ts

export const depositRpc = defineRpc({
  namespace: NAMESPACE,
  id: 'deposit',
  methods: {
    getById: defineRpcMethod<{ id: IntegerString }, ResultDto<CommonCode.Ok, DepositDto | null>>(),
    getByNetworkTransactionId: defineRpcMethod<
      { networkTransactionId: IntegerString },
      ResultDto<CommonCode.Ok, DepositDto | null>
    >(),
  },
});
```

`DepositDto` 不變（5.2a 已 commit `Serialized<Deposit>` 保持）。

對應 server impl 調整（從 nullable return 改 envelope return）：

```ts
// 從：
return deposit ? toDepositDto(deposit) : null;

// 改為：
return ok({ data: deposit ? toDepositDto(deposit) : null });
```

lookup-style `get*` method 採 nullable result，與 action-by-id 的 `NotFound` envelope 切分。`getById` / `getByNetworkTransactionId` miss 都屬 expected lookup absence，不需要額外 error code。

`getByNetworkTransactionId` 同樣調整。具體 wiring 與 server controller 修改細節留 ep-2-plan-1 階段（Codex 實作）。

#### 5.4d 檔案 placement summary

新建：

- `libs/contract-rpc/src/lib/fund/rpc/withdrawal-intent.rpc.ts`
- `libs/contract-rpc/src/lib/fund/dto/withdrawal-intent.dto.ts`
- `apps/esingpay-cradle/src/fund/rpc/server/withdrawal-intent/withdrawal-intent.controller.ts`（server impl，新建）

修改：

- `libs/contract-rpc/src/lib/fund/rpc/deposit.rpc.ts`（envelope return 替換 nullable union）
- `apps/esingpay-cradle/src/fund/rpc/server/deposit/deposit.controller.ts`（server impl 改 envelope return）

無需新建：

- Fund-side code enum file（不建，只用 `CommonCode`）

#### 5.4e 不在本子議題範圍

- Wallet ledger composer 注入 `depositRpc` / `withdrawalIntentRpc` client → 5.3
- ep-2-plan-1 phase 拆分與 acceptance criteria → ep-2-plan-1 撰寫範圍

📌 延伸觀察（記下不展開）：

- 既有 `depositRpc` input 從 `{ id: string }` 改 `{ id: IntegerString }` 是 type-level alias 改善，非 wire-level breaking change
- Fund-side `FundCode` / per-resource code enum 留在 fund operation RPC 擴展時（cancel / approve / submit 等）再建
- Fund 端 deposit query service 已存在 codebase（fund-rpc-survey §2.1 server impl 看到）；新建 `withdrawalIntentQueryService.getById` 是新增工作（具體細節留 ep-2-plan-1）

---

## 議題 5.4 完整結束 ✅

---

### 5.3 Cross-BC Dependency Wiring ✅

本子議題 commit「composer 設計憲法」——wallet ledger composer 注入 fund RPC client 的 module wiring + 處理 pattern。具體 file placement / mapper 命名 / helper signature 留 ep-2-plan-2 階段。

#### 5.3a Sub-decisions

1. **Module wiring placement**：service-root `RpcClientModule` 註冊（與既有 `networkEndpointRpc` / `networkTransactionRpc` 同級 definitions list）；非 feature-local 註冊。對齊 fund-rpc-survey §4.3 既有 wallet `RpcClientModule` pattern
2. **5.3 範圍**：只 commit fund RPC client 注入（`depositRpc` + `withdrawalIntentRpc`）；其他 cross-BC client（`networkTransactionRpc` for ledger row 的 `WalletLedgerNetworkTransactionDto`、merchant / account-management for `RelatedMerchantDto`、wallet self for `RelatedWalletDto`）的 wallet ledger composer 注入屬議題 6 projection 範圍

#### 5.3b Module wiring

```ts
// apps/esingpay-cradle/src/wallet/rpc/client/client.module.ts

import { networkEndpointRpc, networkTransactionRpc } from '@esingpay/contract-rpc/network';
import { depositRpc, withdrawalIntentRpc } from '@esingpay/contract-rpc/fund';

@Module({
  imports: [
    BaseRpcClientModule.forRoot({
      environment: ENVIRONMENT,
      services: {
        [ServiceName.Network]: { transport: ..., options: ... },
        [ServiceName.Fund]: { transport: ..., options: ... },  // 新增（若尚未有）
      },
      definitions: [
        networkEndpointRpc,
        networkTransactionRpc,
        depositRpc,             // 新增
        withdrawalIntentRpc,    // 新增
      ],
    }),
  ],
  exports: [BaseRpcClientModule],
})
export class RpcClientModule {}
```

`ServiceName.Fund` services config entry 是否已存在 codebase 待 ep-2-plan-1 階段 confirm；若未有，一併補入（mechanical addition）。

#### 5.3c Composer 注入 pattern

```ts
@Injectable()
export class WalletLedgerItemComposer {
  constructor(
    @InjectRpcClient(depositRpc)
    private readonly depositRpcClient: RpcClientOf<typeof depositRpc>,

    @InjectRpcClient(withdrawalIntentRpc)
    private readonly withdrawalIntentRpcClient: RpcClientOf<typeof withdrawalIntentRpc>,

    // 其他 wallet self / network / merchant client 注入由議題 6 處理
  ) {}
}
```

#### 5.3d Batch hydration dispatch pattern

Composer 處理流程：按 `subjectType` 分組 collect、parallel hydrate per type、map 拼裝。

```ts
async compose(items: WalletLedgerItem[]): Promise<WalletLedgerItemDto[]> {
  // 1. 按 subjectType 分組 collect identifiers
  const depositIds = items
    .filter(i => i.subjectType === FundCaseType.Deposit)
    .map(i => i.subjectIdentifier.toString());
  const withdrawalIntentIds = items
    .filter(i => i.subjectType === FundCaseType.WithdrawalIntent)
    .map(i => i.subjectIdentifier.toString());

  // 2. Batch hydrate per type (parallel)
  const [depositResults, withdrawalIntentResults] = await Promise.all([
    Promise.all(depositIds.map(id => this.depositRpcClient.getById({ id }))),
    Promise.all(withdrawalIntentIds.map(id => this.withdrawalIntentRpcClient.getById({ id }))),
  ]);

  // 3. Build map (id → DTO) + 拼裝 per item（具體 build mapper 細節留 ep-2-plan-2）
  return items.map(item => buildWalletLedgerItemDto(item, /* ... */));
}
```

Mapper 命名遵循 codebase mapper convention（`build` for multi-source composition；具體 helper signature / file placement 留 ep-2-plan-2）。

#### 5.3e 不在本子議題範圍

- Composer / mapper file 的具體 placement / 命名 / signature → ep-2-plan-2
- 其他 cross-BC client（`networkTransactionRpc` / merchant / wallet self）的 wallet ledger composer 注入 → 議題 6 projection
- Composer error handling 細節（含 fund-side `getById` 回 NotFound 的 invariant violation 處理）→ 議題 6
- Batch endpoint 優化（`getByIdIn`）→ YAGNI；未來 fund RPC 加 batch endpoint 時再評估

📌 延伸觀察（記下不展開）：

- 一頁 N 個 deposit + M 個 withdrawal_intent 觸發 N + M 個 RPC call。未來 fund RPC 加 batch endpoint（`getByIdIn({ ids: [...] })`）可降到 2 個 RPC call
- Fund-side `getById` 回 `CommonCode.NotFound` 在合理 invariant 下不該發生（議題 5.2b 設計依據第 2 條：wallet allocation 必由 fund case 驅動）；若實際出現是 invariant violation——composer 該 throw / log critical

---

## 議題 5.3 完整結束 ✅

---

### 5.5 對偶性檢查 Minimal Closing ✅

5.1d 已 commit「接口不對稱定位」。本子議題做 (1) 整條 stack 對偶情況 documentation + (2) 候選原則 13 正式 commit 為跨議題原則。

#### 5.5a 整條 Stack 對偶情況

| Layer                | Wallet 端                                                                            | Fund 端                                                         | 對偶情況                                                                            |
| -------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Persistence          | wallet allocation table polymorphic（`subject_type` / `subject_identifier` columns） | per-aggregate 獨立 tables（`deposit` / `withdrawal_intent`）    | 不對稱（誠實反映業務結構）                                                          |
| RPC reference lookup | `walletAllocationRpc.listByFundCase` generic（collection-shape）                     | N/A（fund 內部不需要 reference 反查 wallet allocation）         | 單向 lookup                                                                         |
| RPC hydration        | N/A（不是 wallet 業務）                                                              | `depositRpc.getById` / `withdrawalIntentRpc.getById` per-entity | 單向 hydration                                                                      |
| Composer 處理        | 按 `subjectType` 分組 dispatch                                                       | per-aggregate query 各自處理                                    | 各自處理 polymorphic / per-aggregate 邏輯                                           |
| REST DTO（wallet）   | `WalletLedgerItemFundCaseDto = { type, identifier, subject }`；subject uniform shape | N/A                                                             | wrapper + uniform subject（fund-case-proposal 對齊 + 不為 per-type 形式對稱付代價） |

整條 stack 的「**分清楚**」原則貫穿：

- **Reference / lookup**：wallet generic（polymorphic table 結構需要）、fund 不需 reference 反查
- **Hydration**：fund per-entity（per-aggregate 結構誠實反映）、wallet 不需提供 hydration
- **Wire shape**：wallet REST DTO 用 wrapper + uniform subject

#### 5.5b 跨議題原則 13 正式 commit

從議題 5.1d「候選原則 13」提升為正式跨議題原則 13。Resume Point「跨議題原則」清單從 12 條擴為 13 條。

**原則內容**：

> **「分清楚 > 形式對稱」**：對稱性是 emergent property、不是 design goal。當底層業務結構真的不對稱時，強行對稱反而不誠實，並會帶來實際代價（paradigm split / 抽象層複雜化 / wiring 複雜化）。

**議題 5 內 Evidence**：

- **5.1**：fund 端不為「與 wallet 端 `listByFundCase` 對稱」而引入 generic dispatcher；fund per-entity 反映 fund 內部 per-aggregate 結構
- **5.2b**：wallet REST subject 採 uniform shape，不為「per-type 形式對稱」付出 wallet composer / 前端 type narrow 複雜化
- **5.4 sub-decision 1**：`withdrawalIntentRpc` 不加 `getByNetworkTransactionId` 對齊 `depositRpc`；fund 內部目前沒此需求

**適用範圍**：

- 跨 BC 接口設計（RPC / REST contract shape）
- 抽象層引入決策（dispatcher / generic wrapper）
- Method 數量擴張決策

**不適用**：

- Codebase convention 一致性（既有命名 / 風格不該被當違反 convention 的藉口）
- True symmetry（業務結構本就對稱時，shape 對稱是正確設計）

📌 延伸觀察（記下不展開）：原則 13 落地後，未來 Phase 6+ design 收斂時（payment / refund 等新 fund case type 加入）可作為「不為新舊 type 形式對稱付出代價」的 anchor。

---

## 議題 5.5 完整結束 ✅

---

## 議題 5 完整結束 ✅

---

## 6. Projection 實作邏輯

### 子議題地圖（reframe 後）

| 子議題 | 範圍                                                | 狀態    |
| ------ | --------------------------------------------------- | ------- |
| 6.1    | Projection mapping framing — row 寫入模型           | ✅      |
| 6.2    | Path-by-path projection 行為                        | ✅      |
| 6.3    | Cross-BC client 注入 + networkTransaction hydration | pending |
| 6.4    | Error handling + reconciliation 兜底                | pending |

備註：handoff 原議題 6.1（trigger 升級）已收斂為「沿用 Phase 4 inline best-effort hook，不重議 trigger」，未列為獨立子議題；handoff 原議題 6.3（realization derivation）已併入 6.2 path-by-path table（議題 1.2b derivation 表已 codify，6.2 已套用至每個 path）。

### 6.1 Projection Mapping Framing ✅

**模型 A — 單 row 持 plan + outcome 雙態，append-only revision**。

**寫入規則**：

- `projectPlan`（bind / propose）：insert N 條 rows，每 row `plan = declared / outcome = null / revision = current`
- `projectOutcome`（settle / direct-settle / release-from-bound）：找出同 unique key 的 current rows → mark superseded → insert 新 rows，每 row `plan = declared`（同前）/ `outcome = realized`（或 0）/ `revision = current`
- **Direct-settle 場景**（path C/E）：直接走 `projectOutcome`（不先走 `projectPlan`），ctx 內 allocation 同時含 planLines + outcomeLines，**一次寫** plan + outcome 同時持值的 row，不留中間 plan-only rows

**Read API 預設 filter**：`revision = 'current'`（議題 1.4b 已 commit），superseded plan rows 不暴露給 caller。

**Trigger 點**：沿用 Phase 4 inline best-effort hook（use-case 在 transaction 外呼叫 `projectionService.projectPlan` / `projectOutcome`，`try/catch` + `log warn` 不阻斷主流程）。失敗後 ledger 缺 rows 風險由議題 7 reconciliation cron 兜底。

**設計依據**：

1. 對齊議題 1.2a schema（`plan` / `outcome` 同 row）+ 1.2c append-only revision
2. Resume Point 既有 TODO「path A→D plan rows redundant」自動解：outcome 抵達時 plan row supersede + 預設 view filter current 不暴露兩組
3. Realization derived field（議題 1.2b）在單 row 上是純函數，不需 cross-row aggregate
4. Path C/E directSettle 一次寫不留中間 plan-only rows，避免 transient 浪費 + 對齊 ep-1-plan-1 §3.3 觸發時機表
5. 議題 4.3c「declared vs realized 對比」能力滿足：所有 path 的 outcome row 同時持 plan + outcome

---

## 議題 6.1 完整結束 ✅

---

### 6.2 Path-by-Path Projection 行為 ✅

| Path                                                             | Project method | row plan | row outcome                                | revision 演進                        | realization         |
| ---------------------------------------------------------------- | -------------- | -------- | ------------------------------------------ | ------------------------------------ | ------------------- |
| Withdrawal reserve                                               | （不觸發）     | —        | —                                          | —                                    | —                   |
| Withdrawal bind                                                  | projectPlan    | declared | null                                       | insert current                       | pending             |
| Withdrawal settle（success）                                     | projectOutcome | declared | realized                                   | supersede + insert new               | realized            |
| Withdrawal settle（failed without consumed fee）                 | projectOutcome | declared | 0                                          | supersede + insert new               | cancelled           |
| Withdrawal settle（failed with consumed network_fee, spec §8.5） | projectOutcome | declared | partial（network_fee row）/ 0（principal） | supersede + insert new               | partial / cancelled |
| Withdrawal release-from-bound                                    | projectOutcome | declared | 0                                          | supersede + insert new               | cancelled           |
| Deposit propose（path A→D）                                      | projectPlan    | declared | null                                       | insert current                       | pending             |
| Deposit settle（path A→D success）                               | projectOutcome | declared | declared                                   | supersede + insert new               | realized            |
| Deposit settle（path F fail in transit）                         | projectOutcome | declared | 0                                          | supersede + insert new               | cancelled           |
| Deposit directSettle（path C success）                           | projectOutcome | declared | declared                                   | insert current（不留中間 plan-only） | realized            |
| Deposit directSettle（path E fail）                              | projectOutcome | declared | 0                                          | insert current（不留中間 plan-only） | cancelled           |

**6.2a directSettle 一次寫 ✅**：

Path C/E 一次寫 plan+outcome row，不留中間 plan-only rows。

設計依據：

1. 對齊 ep-1-plan-1 §3.3 觸發時機表（directSettle → projectOutcome only）
2. directSettle 業務本質是 single event（born settled），不是「propose→settle 壓縮」
3. Path A→D 與 path C/E 的 audit trail 不對等是 feature——反映業務 event 真實數量（2 個 vs 1 個）
4. 中間 plan-only row 從不可見（瞬間 supersede），留著是 wasted writes

**6.2b release-from-bound 寫法 ✅**：

按 spec「release 時 outcomeLines = null → 全部歸零」+ 模型 A：projection 從 `ctx.allocation.planLines` 取 `(walletId, bucket, currencyCode, kind)` tuples，對每個 tuple supersede 既有 current row + insert 新 row（`plan = 同前 declared, outcome = 0, realization = cancelled`）。

**6.2c failed 場景的 audit trail 對等性 ✅**：

Deposit path F 與 path E 最終 current row 結果相同（plan = declared, outcome = 0），但 audit trail 不同：

- Path F 留 superseded plan-only row 的歷史 trace（propose 階段已寫）
- Path E 不留（一階段 directSettle）

兩 path 真實的業務 event sequence 不同（path F 經過 propose→settle 兩階段；path E 一階段 directSettle），audit 對等性差異是業務事實真實映射，不是設計缺陷。

---

## 議題 6.2 完整結束 ✅

---

### 6.3 Cross-BC Client 注入 + WalletLedgerItemComposer Constructor ✅

5.3 已 cover fund subject 的 `depositRpc` / `withdrawalIntentRpc` 注入；6.3 補 composer 對其他三個 source 的 hydration（networkTransaction / merchant / wallet self），以及整體 composer constructor + module wiring delta 概要。

#### 6.3a `networkTransactionRpc` 注入 ✅

`@InjectRpcClient(networkTransactionRpc)` 直接注入 `WalletLedgerItemComposer`，對齊：

- `NetworkClient.getNetworkTransactionById` deprecated comment「use direct `@InjectRpcClient(networkTransactionRpc)`」
- 議題 5.3 fund cross-BC wiring pattern（直接 `@InjectRpcClient`）

Module wiring：5.3 已 cover（`networkTransactionRpc` 已 registered in `RpcClientModule` definitions list）—— **不需新增**。

投影邏輯：1:1 `NetworkTransactionDto` → `WalletLedgerNetworkTransactionDto`（議題 5.2b shape 已 codify）；具體 mapper 重用 vs 新建留 ep-2-plan-X 階段。

#### 6.3b `RelatedMerchantDto` Hydration — 沿用 `AccountManagementClient` ✅

**選 A：沿用 `AccountManagementClient`**（既有 wallet-asset.composer.ts precedent）

設計依據：

1. **AccountManagementClient 未 deprecated**：fund-rpc-survey §4.3 確認，與 `NetworkClient`（已標 deprecated）對比明顯——既有 codebase pattern 沒 push「直接 RPC injection」
2. **跨議題原則 13 適用**：composer 內 fund 用直接 RPC injection vs merchant 用 facade 是 acceptable——fund 端 RPC contract 是本輪新建（5.4 commit），merchant facade 是既有 pattern；硬要對稱反而引入新 contract 設計負擔
3. **避免 unknown territory 風險**：`@esingpay/contract-rpc/account-management` 既有覆蓋未 cover；直接 RPC injection 案需先 survey + 可能要新建 RPC contract，本輪 scope 擴大
4. **Future migration 成本可控**：若 AccountManagementClient 未來 deprecated（按 NetworkClient 演進），WalletAssetComposer 與 WalletLedgerItemComposer 同步 migrate；不形成 lock-in

📌 延伸觀察（記下不展開）：未來若 `AccountManagementClient` 與 `NetworkClient` 同步 deprecate，兩 composer 一起 migrate；可在那時的議題加 outboundwave decision 一次處理。

#### 6.3c `RelatedWalletDto` Hydration — 路徑 X（直接 inject `WalletQueryService`）✅

**Survey anchor 結論**（`wallet-allocation-impl-codebase-survey-4.md`）：

- `WalletQueryService.listRefsByIdIn({ walletIdIn: bigint[] })` → `Promise<WalletRef[]>`：wallet-side batch wallet ref read 既有 anchor（class 名是 `WalletQueryService`，**不是** `WalletService`；method 命名 `listRefsByIdIn`，**不是** `findByIdIn`）
- `WalletRefMapper.toRelatedWalletDto(input: WalletRef): RelatedWalletDto`：1:1 ref → DTO mapper，存於 `apps/esingpay-cradle/src/shared/mapper/wallet-ref.mapper.ts`，由 `SharedMapperModule` export，fund-side deposit / withdrawal-intent / cashflow-demo composer 已使用——直接重用
- `WalletAssetComposer` **不是** wallet hydration helper：input 帶 wallet raw 進來，不暴露 `composeWalletsByIdIn` 等 helper——ledger composer **不重用** asset composer
- 無 batch mapper：caller `array.map(ref => mapper.toRelatedWalletDto(ref))` 是慣例（fund-side 三個 mapper 已 precedent）

**X 路徑 finalize**：

- Inject `WalletQueryService`（service layer，對齊 application-domain rules — composer invokes service, not repository）
- Inject `WalletRefMapper`（shared mapper，重用既有；不新建 batch mapper）

**捨 Y 路徑**（重用 `WalletAssetComposer` helper）：survey confirm WalletAssetComposer 不暴露可重用 wallet hydration helper，Y 路徑不成立。

#### 6.3d Composer Constructor + Module Wiring ✅

**Composer dependencies**（5 個 cross-BC source + 重用 mapper）：

```ts
@Injectable()
export class WalletLedgerItemComposer {
  constructor(
    // Fund subject RPC（議題 5.3）
    @InjectRpcClient(depositRpc)
    private readonly depositRpcClient: RpcClientOf<typeof depositRpc>,
    @InjectRpcClient(withdrawalIntentRpc)
    private readonly withdrawalIntentRpcClient: RpcClientOf<typeof withdrawalIntentRpc>,

    // NetworkTransaction RPC（議題 6.3a）
    @InjectRpcClient(networkTransactionRpc)
    private readonly networkTransactionRpcClient: RpcClientOf<typeof networkTransactionRpc>,

    // Merchant hydration（議題 6.3b — 沿用 AccountManagementClient）
    @Inject(AccountManagementClient)
    private readonly accountManagementClient: AccountManagementClient,

    // Wallet self hydration（議題 6.3c — 走 service）
    @Inject(WalletQueryService)
    private readonly walletQueryService: WalletQueryService,

    // Mapper（shared，重用既有）
    @Inject(WalletRefMapper)
    private readonly walletRefMapper: WalletRefMapper,
    // RelatedMerchantDto / WalletLedgerNetworkTransactionDto 投影 mapper
    // （MerchantRefMapper / NetworkTransactionRpcMapper 對齊狀態待 ep-2-plan-X 階段 confirm）
  ) {}
}
```

**Module wiring delta**：

`allocation-context.module.ts`（既有）目前 import `RepositoryModule` + `SharedServiceModule`，**不 import** `RpcClientModule`。WalletLedgerItem feature module 落地需補：

| Import                                | 來源                                                                                         | 狀態          |
| ------------------------------------- | -------------------------------------------------------------------------------------------- | ------------- |
| `RpcClientModule`                     | 既有 `apps/.../wallet/rpc/client/client.module.ts`（議題 5.3 已 commit 加 fund definitions） | 6.3 補 import |
| `WalletQueryService` 所在 module      | 既有 `feature/wallet/service`                                                                | 6.3 補 import |
| `SharedMapperModule`                  | 既有，已 export `WalletRefMapper` 等                                                         | 6.3 補 import |
| `AccountManagementClient` 所在 module | 既有（wallet-asset composer 已 import）                                                      | 6.3 補 import |

#### 留 ep-2-plan-X 階段的細節

- `WalletLedgerItemComposer` file placement：放 `feature/allocation/composer/` 還是新建 `feature/ledger/`（與未來 `WalletCashflowItem` 一併決定 file placement convention）
- `MerchantRefMapper` / `NetworkTransactionRpcMapper` 直接重用 vs 新建 wallet-ledger 專用 mapper（shape 對齊狀態 confirm）
- `AccountManagementClient.getMerchantInfo` 返回 detail vs ref；對 `RelatedMerchantDto` 投影路徑

---

## 議題 6.3 完整結束 ✅

---

### 6.4 Error Handling + Reconciliation 兜底 ✅

#### 6.4a Inline Hook Error Handling — 沿用 ep-1-plan-2 best-effort pattern ✅

ep-1-plan-2 已 land 的 best-effort pattern 直接沿用：

```ts
try {
  await this.projectionService.projectOutcome({
    allocation: ...,
    fundCase: {},
    effectiveAt: now,
  });
} catch (error) {
  this.logger.warn(
    `projectOutcome failed for allocation ${...id}`,
    error instanceof Error ? error.stack : error,
  );
}
```

議題 6.1 已 commit「沿用 Phase 4 inline best-effort hook」。本子議題 confirm，不重議。

#### 6.4b Composer 內部 Cross-BC 失敗 Propagation — 整個 throw ✅

`WalletLedgerItemComposer` batch hydrate 4 個 cross-BC source（fund subject ×2 / networkTransaction / merchant / wallet self）；**任一 source 失敗 → composer 整個 throw**，由上層 use-case caller 的 try/catch（6.4a pattern）接住 + log warn。

設計依據：

1. **對齊 5.2b `fundCase not nullable` invariant**：partial rows fallback 違反 invariant
2. **延伸 5.3e 設計方向**「fund-side `getById` 回 NotFound 是 invariant violation，composer 該 throw」——本決議延伸到所有 cross-BC source
3. **Best-effort「fail loud, recover later」哲學**：失敗時不寫 partial rows，cron rerun 時 idempotency 友善（atomic write — 要麼全寫，要麼全不寫）
4. **Composer 邏輯最簡**：不需 per-source try/catch + fallback 邏輯
5. **Telemetry 升級可控**：log warn + stack 在初期 sufficient；metric / alert 是 ep-2-plan-X 之後的 enhancement，不在 phase 5 範圍

捨棄選項：

- **B per-source fallback null**：違反 5.2b invariant + cron 重跑邏輯複雜
- **C structured throw**：log warn pattern 透過 `error.cause` / stack 已能 carry detail，C 案複雜度上升 marginal benefit

#### 6.4c Reconciliation Cron 與 Wallet-Side 介面 — Use-case 公開方向 ✅（實作推議題 7 / Future）

**設計方向 commit**：議題 7 reconciliation cron 不直接 invoke `projectionService.projectXxx`；wallet-side 公開 `BackfillProjectionUseCase`（命名暫定），cron 走 use-case 介面。

**Phase 5 落地不實作**：

最低主線運作下 BackfillProjectionUseCase 不是必要——

1. Projection 99% 場景成功；罕見失敗時 inline best-effort hook log warn，ledger 缺 rows
2. 缺 rows 不阻斷主流程（allocation + entry + balance 是 ground truth；議題 1.3b「ledger 非 source of truth，問題隨時可從 allocation 重新修正」）
3. 缺 rows 的修復：phase 5 階段 manual operation（DBA / dev 直接重新 trigger projection or 補 SQL insert）即可
4. 自動 backfill 是「成熟運維」需求，phase 5 是初期設計階段——YAGNI

**設計方向選 use-case 而非直接 invoke service** 的依據：

1. **介面 boundary 清晰**：cron 與 projection service 解耦——cron 不需懂 projection method 選擇邏輯（projectPlan vs projectOutcome based on allocation status）
2. **Use-case 是 application layer 既有 boundary pattern**：對齊 codebase use-case 命名 / DI / error handling 慣例
3. **Future projection 重構耐久**：若 projection method 重構（增 method / 改 ctx shape），cron 介面不變

具體 backfill use-case 的 input shape / behavior / cron triggering frequency / 偵測 ledger 缺 rows 的 SQL pattern 留議題 7 設計（含 use-case 實作時機）。

---

## 議題 6.4 完整結束 ✅

---

## 議題 6 完整結束 ✅

---

## 7. Cron / Scheduler

### 子議題地圖

| 子議題 | 範圍                                        | 狀態 |
| ------ | ------------------------------------------- | ---- |
| 7.1    | Phase 5 範圍與 framing — 全部推 future 實作 | ✅   |
| 7.2    | Withdrawal 側失敗點 audit + comment 模板    | ✅   |
| 7.3    | 跨議題原則 14 commit                        | ✅   |
| 7.4    | 推 future TODO 條目分類                     | ✅   |

備註：handoff 原預估子議題（7.1 整體架構 / 7.2 孤兒類偵測 / 7.3 settle stuck / 7.4 dead letter）依本輪 framing 修正後 fold 為「全部推 future 實作 + phase 5 限縮為 TODO comment + queue 大方向 commit」。原子議題不個別展開。

### 7.1 Phase 5 範圍與 Framing ✅

**核心 framing 修正**：handoff 預設「reconciliation cron」基於 cron-based detection 為 default。對齊議題 4.1g 既有 `depositWalletSyncQueue` pattern，本 codebase「事後追認」範式應為 **queue-based catch-up**——失敗點 dispatch 到 queue、worker 消化補 sync。Cron-based 只在無 dispatch point（純 wallet-side anomaly）時 fallback。

**機制 sweep**：

| #   | 機制                      | 性質                                       | Phase 5 結論                   |
| --- | ------------------------- | ------------------------------------------ | ------------------------------ |
| 1   | Settle stuck retry        | Queue-based 可行                           | 推 future（withdrawal queue）  |
| 2   | Settle idempotency        | 與 #1 配對                                 | 推 future（與 queue 一起設計） |
| 3   | Wallet bound 孤兒清掃     | 純 wallet-side anomaly                     | 推 future（不限定方向）        |
| 4   | Cross-instance race 偵測  | 純 wallet-side anomaly                     | 推 future（不限定方向）        |
| 5   | Dead letter 清理          | DLQ 自身機制 + ops monitor                 | 不需新設計                     |
| 6   | BackfillProjectionUseCase | Queue-based 可行（介面方向已 6.4c commit） | 推 future                      |

**Phase 5 必要性 anchor**（套 6.4c lesson）：每個機制答「主線 caller 是誰 / 缺乏業務後果」皆「無 / 不影響核心業務正確性」。**全部 phase 5 不落地實作**。

**Phase 5 唯一動作**：

1. Withdrawal 側失敗點留 TODO comment（見 7.2）
2. Withdrawal 補 sync 大方向 commit：mirror `depositWalletSyncQueue`，未來建 `withdrawalIntentWalletSyncQueue`（命名暫定）。**未在 phase 5 落地**

捨棄選項：

- **介面藍圖 commit**（六個機制各自介面 phase 5 commit）：未來實作時可能 cron / queue / 其他混合，phase 5 commit 介面過早；queue 大方向 commit 已足
- **framework 抽象**（reconciliation cron 共用 framework）：YAGNI，phase 5 主線運作不需要

📌 延伸觀察（記下不展開）：「Phase 5 主線 caller 必要性」這條從 6.4c 起就在誕生為跨議題原則候選。本議題全程套用無例外，但獨立性不足以升原則——若議題 8 / 未來 phase 反覆套用可考慮升原則。

### 7.2 Withdrawal 側失敗點 Audit + Comment 模板 ✅

議題 3 / 6 各 use-case 內目前處於「failed sync 留 TODO」狀態。本子議題 codify Codex 實作時統一遵循的失敗點清單與 comment 模板。

#### 7.2a 失敗點清單

**Class 1：Submit 內失敗 → wallet 端孤兒（無 intent.id 可 dispatch）**

來源：議題 3.4b/c

| #   | Use-case                                                                          | 失敗點                                                                                             |
| --- | --------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| 1   | `MerchantSubmitWithdrawalIntentUseCase` / `PlatformSubmitWithdrawalIntentUseCase` | Step 6 persist 失敗 → `wallet.release(allocationId)` best-effort 也失敗 → wallet 端 reserved 孤兒  |
| 2   | `MerchantSubmitWithdrawalIntentUseCase` / `PlatformSubmitWithdrawalIntentUseCase` | Step 7 `wallet.bind` 失敗 → `withdrawalIntentRepository.delete(intent.id)` 後 wallet 端 bound 孤兒 |

→ 屬議題 7 #3 / #4 wallet-side reconciliation 範疇；fund 端無 dispatch point（intent 從未 / 已 deleted）。

**Class 2：Transition commit 後 settle 失敗 → allocation stuck bound**

來源：議題 3.2d

| #   | Use-case                                                                                                | Settle 內容                                                |
| --- | ------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| 3   | `PlatformCancelWithdrawalIntentUseCase`                                                                 | `settle(zero outcome)`                                     |
| 4   | `PlatformRejectWithdrawalIntentUseCase`                                                                 | `settle(zero outcome)`                                     |
| 5   | `PlatformReleaseWithdrawalIntentUseCase`                                                                | `settle(actual outcome)`                                   |
| 6   | `WithdrawalIntentAutoProgressService`(auto-release path)                                                | `settle(actual outcome)`                                   |
| 7   | `HandleNetworkTransactionSettledUseCase`(withdrawal-intent feature)                                     | `settle(actual outcome)`                                   |
| 8   | `HandleNetworkTransactionFailedUseCase`(withdrawal-intent feature；fail without consumed fee 分支)      | `settle(zero outcome)`                                     |
| 9   | `HandleNetworkTransactionFailedUseCase`(withdrawal-intent feature；fail with consumed network_fee 分支) | `settle(partial: network_fee row + 0 principal，議題 6.2)` |

備註：所有 entries 對應 use-case 命名以 `wallet-allocation-impl-codebase-survey-6.md` audit confirm（NestJS `UseCase` suffix convention）。#8 / #9 對應同一 `HandleNetworkTransactionFailedUseCase`，內部分支 outcome-line builder 區分 zero vs partial。Status 全 pending — TODO comment 隨議題 3 / ep-2-plan-X implementation 同期插入；codebase 各 use-case file path / catch 點 line number 見 codebase-survey-6 Section 1。

→ 未來走 `withdrawalIntentWalletSyncQueue`（mirror `depositWalletSyncQueue`）。

#### 7.2b Comment 模板

**Class 1（孤兒，#1 / #2）**：

```ts
// TODO: wallet-side orphan; resolution by wallet-side reconciliation (見議題 7 #3).
// Until reconciliation lands, manual ops cleans orphans.
logger.error('wallet release/bind failed, orphan allocation', { ... });
```

**Class 2（stuck，#3-#9）**：

```ts
// TODO: dispatch to withdrawalIntentWalletSyncQueue for catch-up sync.
// Mirror depositWalletSyncQueue pattern (見議題 4.1g).
// Until queue lands, manual ops resolves stuck allocations.
logger.error('wallet settle failed, allocation stuck', { ... });
```

### 7.3 跨議題原則 14：事後追認用 Queue Catch-up 為 Default ✅

**原則內容**：

> **事後追認用 queue catch-up 為 default；cron 只在無 dispatch point 時 fallback**。當 fund-side use-case 對 wallet 的 sync 失敗時，預設由失敗點 dispatch 到 queue，worker 消化補 sync（mirror 議題 4.1g `depositWalletSyncQueue`）。Cron-based 只在純 wallet-side anomaly（無 dispatch point）時使用。

**議題 7 內 Evidence**：

- 失敗點 audit Class 2（#3-#9）：transition commit 後 settle 失敗，每個都有 fund-side dispatch point → queue-based
- 失敗點 audit Class 1（#1 / #2）：submit 內 wallet release / bind 失敗，intent 從未 / 已 deleted，無 dispatch point → cron-based fallback
- 議題 7 機制 #3 / #4（wallet bound 孤兒 / cross-instance race）：純 wallet-side anomaly → cron-based
- 議題 7 機制 #6（projection backfill）：projection 失敗點在 use-case 內，可 dispatch → queue-based 範式

**適用範圍**：

- 跨 BC sync 補償機制設計
- Worker / dispatcher / scheduler 範式選擇

**不適用**：

- Fund-side internal 補償（如 transaction rollback / saga）
- Wallet-side internal 補償

📌 延伸觀察（記下不展開）：本原則一旦其他議題 / 未來 phase 套用一致，可作為「補償機制範式」決策 anchor，引導未來新 BC 接入時直接套 queue catch-up 而非 cron-based。

### 7.4 推 Future TODO 條目分類 ✅

議題 7 範圍六個機制全部推 future。對應 TODO 章節既有條目（無需新增）：

| 機制                         | 主文件 TODO 章節對應                                                 |
| ---------------------------- | -------------------------------------------------------------------- |
| #1 Settle stuck retry        | 來自議題 3.2 — Settle stuck retry 機制                               |
| #2 Settle idempotency        | 來自議題 3.2 — 同 #1 entry（含 idempotency）                         |
| #3 Wallet bound 孤兒清掃     | 來自議題 3 wallet reconciliation                                     |
| #4 Cross-instance race 偵測  | 來自議題 4.2 — Cross-instance race 偵測 cron                         |
| #5 DLQ 清理（不需新設計）    | 來自議題 4.2 — Dead letter handling（縮減為 ops monitor，不需 cron） |
| #6 BackfillProjectionUseCase | 來自議題 6.4 — BackfillProjectionUseCase 實作                        |

**議題 7 內新生 TODO 條目**：

- **`withdrawalIntentWalletSyncQueue` 大方向 commit**（mirror `depositWalletSyncQueue`；承載 7.2 Class 2 失敗點 #3-#9 的 catch-up sync）

該條目落入新建 TODO sub-section「### 來自議題 7」（見 TODO 章節）。

---

## 議題 7 完整結束 ✅

---

## 8. Smoke Test 落地

### 子議題地圖

| 子議題 | 範圍                                    | 狀態 |
| ------ | --------------------------------------- | ---- |
| 8.1    | Scope — Layer 1 happy paths only        | ✅   |
| 8.2    | Test infrastructure — B.1 use-case spec | ✅   |
| 8.3    | Layer 1 happy path scenarios codify     | ✅   |

### 8.1 Smoke Test Scope ✅

採 **Layer 1 happy paths only**。Failure paths（NT failed / system errors / cross-instance race / projection 失敗）+ wallet 自治 cron 機制（如 reservation TTL expire）皆推 future。

A.2 happy 涵蓋粒度：**純成功 path + user-initiated 正常分支（cancel）**。Cancel 不算「failure」（system error / RPC fail / NT fail 才是失敗），是 user 視角業務正常 abort 分支；phase 5 上線後 cancel 是高頻 user action，不測等於沒 cover。

捨棄選項：

- **A.1 純成功 only**：縮太窄，cancel 不測等於沒 cover
- **含 failure paths**：違反「最小可驗證集」精神；推 future 餘力時加碼

### 8.2 Test Infrastructure — Layer B.1 ✅

採 **use-case-level Jest spec**（對齊 codebase 既有 `*.use-case.spec.ts` pattern）。

#### 8.2a Mock 範圍

統一所有外部 dependency mock：

- Wallet RPC client / fund RPC client
- Repository
- Queue dispatcher
- Network endpoint RPC client

#### 8.2b Verify 範圍

- Entity transition correctness（intent / allocation / deposit status 對齊預期）
- 外部 call sequence + payload shape：
  - Wallet RPC method 命名 + lines payload
  - Repository method（create / update / delete）
  - Queue dispatcher payload

#### 8.2c 不 Cover（推 future）

以下屬 B.2 / B.3 範疇，**Layer 1 不 cover**：

- 跨 BC RPC contract drift
- 真實 BullMQ infrastructure（dispatch / worker 整合）
- 真實 DB persistence
- HTTP / cross-service e2e

#### 8.2d Jest Runner

依 codebase 既有 `temporary-rules.md` Temporary Jest Runner Rule：

```sh
npm run test:one -- --project-root apps/esingpay-cradle -- <spec paths>
```

新建 spec files 直接套用，無 infrastructure 新投資。

### 8.3 Layer 1 Happy Path Scenarios ✅

#### 8.3a Path 表

| #   | Scenario                    | 業務 Trigger                  | 預期 Wallet RPC sequence（spec verify 重點）                               |
| --- | --------------------------- | ----------------------------- | -------------------------------------------------------------------------- |
| 1   | Withdrawal happy end-to-end | Submit → release → NT settled | `wallet.reserve` + `wallet.bind` → `wallet.settle(actual)`                 |
| 2   | Withdrawal user cancel      | Submit → cancel               | `wallet.reserve` + `wallet.bind` → `wallet.settle(zero)`                   |
| 3   | Deposit happy end-to-end    | NT settled → recognize + sync | `depositWalletSyncQueue.dispatch` → worker → `wallet.directSettle(actual)` |

#### 8.3b Path → Use-case Spec Mapping（Anchor，不 codify count）

每 path 對應的 use-case spec list — Codex 實作時依議題 3 / 4 final use-case set 自行 enumerate；下方為 anchor，**主文件不 codify spec 數量 / file path / shape**：

**Path 1（Withdrawal happy end-to-end）**：

- `PlatformSubmitWithdrawalIntentUseCase` / `MerchantSubmitWithdrawalIntentUseCase`：reserve + bind happy
- `PlatformReleaseWithdrawalIntentUseCase` / `WithdrawalIntentAutoProgressService`（auto-release path）：release → `settle(actual)`
- `HandleNetworkTransactionSettledUseCase`（withdrawal-intent feature）：NT settled → `settle(actual)`

**Path 2（Withdrawal user cancel）**：

- `PlatformSubmitWithdrawalIntentUseCase`：reserve + bind happy
- `PlatformCancelWithdrawalIntentUseCase`：cancel → `settle(zero)`

**Path 3（Deposit happy end-to-end）**：

- Deposit feature 的 NT-settled 觸發 use-case（議題 4 設計 ep-2-plan-X 落地時 confirm；可能對應 `Handle...DiscoveredUseCase` / `Handle...ChangedUseCase`）：建 deposit + dispatch queue
- `DepositWalletSyncQueueWorker`：worker step 1-5 → `directSettle(actual)`

#### 8.3c Spec Implementation 推遲

主文件 codify 範圍：**3 path scope + B.1 mocking 邊界**。不 codify spec count / file path / shape。

理由：

1. 議題 3 / 4 implementation 尚未 land（per `wallet-allocation-impl-codebase-survey-6.md` Gap 1-7），spec 落地時 use-case set 才 finalize
2. 一個 path 對應「N 個 use-case specs」，N 數會隨 ep-2-plan-X 落地時 vary
3. Layer 1 anchor 是「path 完整覆蓋」，不是「固定 N 個 specs」

#### 8.3d 推 Future（非議題 8 範圍，落入 TODO 章節「來自議題 8」）

對齊 8.1 framing，下列推 future：

- **Failure paths**：withdrawal NT failed (without / with consumed network_fee) / reject / system errors / cross-instance race / projection 失敗
- **Wallet self TTL expire smoke**：wallet `ExpireReservedAllocationsUseCase`
- **B.2 service integration**：spawn fund + wallet 兩 instance / 真實 BullMQ / DB fixture / RPC roundtrip
- **B.3 app e2e**：HTTP-level full stack

---

## 議題 8 完整結束 ✅

---

## 未來 TODO 與可改善項目

整理本次設計過程中提到的暫時方案、未來改善項目。每條附觸發條件。

### 來自議題 1

- **WalletLedgerItemDto 套 IntegerString / IsoDateTimeUtc alias**
  - 議題 2.4 觀察：ledger DTO 應用 contract-base alias
  - **觸發**：Claude Code 落實 `wallet-ledger-item-impl-plan-draft.md` 時順手套用

### 來自議題 2

- **Reservation TTL ceiling 防護機制**
  - 在 wallet 端加 max（caller 提供值不可超過強制上限）
  - **觸發**：ops / 監管要求出現

### 來自議題 3.4

- **Legacy `withdrawal_intent.wallet_allocation_id` backfill 完成後移除守衛**
  - Migration 將既存 row 填 0；應用層加 `if (intent.walletAllocationId !== 0n)` 守衛並標 TEMPORARY 註解
  - **觸發**：legacy data backfill 完成

### 來自議題 3.3

- **Wallet 暴露 RPC：query platform fee payer wallet id**（議題 3.3 衍生；ep-2-plan-2 Phase 5 採暫代方案，未回正規方向）
  - **長期方向（議題 3.3c 凍結）**：Fund 透過 wallet 暴露的 RPC 查 `networkEndpointId → platformFeePayerWalletId`，wallet 內部自管 fee delegation（含對 network 的查詢）
  - **Phase 5 暫代方案（ep-2-plan-2 §7.6）**：Fund 端建 `WithdrawalIntentFeePayerResolverService` 吸收 endpoint plumbing：sender wallet → networkEndpointId（既有 `WalletClient.getWalletRefById`）→ feePayerEndpointId（新增 narrow `NetworkClient.getNetworkFeeDelegationBySenderEndpointId` bridge）→ feePayerWalletId（既有 `WalletClient.getWalletByNetworkEndpointId`）。messy plumbing 隔離在 Fund 一個 application service + 一個 narrow NetworkClient method
  - **取捨理由**：Phase 5 不在 wallet 端 grow 正規 fee delegation 能力；plan-2 不該凍結 wallet 端 RPC 命名與 shape
  - **回正規方向觸發**：未來 wallet 端落地正規 fee delegation 能力時，swap 出 Fund-side resolver 內部即可；allocation line builder + submit 編排 + WithdrawalIntent schema 都不需動
  - **不要做的事**：不要把 Fund-side resolver 模式延伸到其他場景；它是 phase 5 局部取捨，不是新範式

- **NetworkEndpoint RPC 提供 estimate gas fee**
  - 取代 `WithdrawalNetworkFeeEstimationService` 內 hardcoded TRX 常數
  - **觸發**：networkEndpoint 端開發 estimateGasFee RPC 後

- **Merchant fee delegation 配置擴充**
  - 當前 platform 代付為主；未來擴充 merchant 自付 / 代付方案
  - **觸發**：商業需求出現

- **WithdrawalIntent 新增 amountNetworkFee 欄位（如需要）**
  - 目前僅在 reserve 階段使用估算值，不持久化到 intent 上
  - **觸發**：若有對帳 / 報表需求需 query intent 的 network fee

- **NetworkTransaction.networkFee 欄位正規化**
  - 目前從 `blockchainDetail.gasUsed` 取得；未來應正規化為 NetworkTransaction 的一級欄位
  - **觸發**：networkTransaction 重構時

### 來自議題 3 wallet reconciliation

- **Wallet bound 孤兒清掃 cron**（議題 7 已 sweep）
  - 偵測 wallet 端 bound 但 fundCaseIdentifier 對應 fund case 不存在的孤兒 allocation（對齊「夭折」概念清理）
  - **議題 7 已 sweep**：phase 5 推 future 實作；範式不限定（純 wallet-side detection，無 dispatch point — 跨議題原則 14 cron-based fallback 範式）

### 來自議題 3.2

- **Settle stuck retry 機制**（議題 7 已 sweep）
  - 偵測「fund intent 已 transition 到 terminal status，但 wallet allocation 仍 bound」的 stuck case，自動 retry settle
  - 同時包含 settle idempotency 機制定案（caller side token 或 wallet side natural idempotency）
  - **議題 7 已 sweep**：phase 5 推 future 實作；範式 queue-based — 由失敗點 dispatch 到 `withdrawalIntentWalletSyncQueue`（mirror `depositWalletSyncQueue`）；與議題 7.2 Class 2 失敗點 #3-#9 一起承載

- **Outbox pattern（事件保序 + retry 一併解決）**
  - 解決 NetworkTransaction event handler 內 transition commit 後 settle 失敗造成 event 已 ack 無法 redeliver 的取捨
  - 同時涵蓋 settle retry
  - **觸發**：未來業務需要強保證最終一致性、且 reconciliation cron 機制已驗證不足以覆蓋時

### 來自議題 4.1（queue 模式）

- **Wallet allocation `(fundCaseType, fundCaseIdentifier)` partial unique index**
  - DB-level hard guard，防止 cross-instance race 留下多筆同 fund case 的 allocation
  - Partial: `WHERE fund_case_identifier IS NOT NULL`（reservation flow 期間 `fundCaseIdentifier` 可為 null）
  - 引入後 `propose` 失敗會回 typed conflict error，caller 收到後重新 read `listByFundCase` 即可
  - **觸發**：議題 7 reconciliation 偵測到 race 真的高頻發生時

- **Deposit `walletAllocationId` legacy backfill 完成後**
  - Migration 將所有 `wallet_allocation_id = 0` 的 row 視為已處理完畢
  - 可在 worker step 2 將 `0n` 從 schema 訊號移除（保留 NULL 與 bigint 二態）
  - **觸發**：legacy data 全數對應到真實 allocation 或顯式宣告「不再 sync」之後

### 來自議題 4.2

- **Queue retry policy 細節**
  - exponential backoff 參數、max retry 次數、dead letter 機制
  - **觸發**：實作 phase plan 時參考 codebase 既有 acceptance queue retry policy 慣例

- **Cross-instance race 偵測 cron**（議題 7 已 sweep）
  - 偵測「同 fund case 對應多筆 allocation」的 race 結果並清理
  - **議題 7 已 sweep**：phase 5 推 future 實作；範式不限定（純 wallet-side detection — 跨議題原則 14 cron-based fallback 範式）

- **Dead letter handling**（議題 7 已 sweep — 縮減為 ops monitor）
  - 持續 retry exhaust 後送 dead letter，需要人工或 cron 介入
  - **議題 7 已 sweep**：DLQ 自身機制 + ops monitor 已足，**不需新設計 cron**；DLQ 累積為 ops alert 觸發手動介入

### 來自議題 4.3

- ✅ **Spec amend §7.6 directSettle 入參加 planLines**（已 land via `ep-2-plan-1`）
  - spec / use-case side shape 加 `planLines: WalletAllocationLineDto[]`（required）
  - 配合 spec §3 planLines timing 描述擴展為「reserve / propose / directSettle 時 set-once」
  - **狀態**：原由 ep-1-plan-3 sidecar 標記為後續 amend，現已於 ep-2-plan-1 落地

- ✅ **Spec amend §7.5 / §7.6 補 zero outcome 例外條款**（已 land via `ep-1-plan-3` sidecar spec patch）
  - 「若 outcomeLines 全為 zero，allocation 仍 settled、outcomeLines 仍寫入 allocation table，但不產 entry / 不更新 balance」
  - **狀態**：與 ep-1-plan-3 同期 land

- ✅ **議題 2 amend：DirectSettleInput 加 planLines field**（已 land via `ep-2-plan-1`）
  - 凍結 spec 局部 unfreeze（input shape 變動）已完成，caller / wallet RPC server mapper 已同步對齊
  - **狀態**：ep-1-plan-3 §0.2 amend 點 1 原標記為「不在本文件」，後續已於 ep-2-plan-1 落地

- ✅ **Wallet directSettle / settle use case 內部加 zero check**（已 land via `ep-1-plan-3` Phase 2 + Phase 3）
  - 在 entry 產生前判斷 outcomeLines 是否全 zero，若是則 skip entry / balance mutation
  - **狀態**：
    - directSettle use case zero-outcome guard land 於 ep-1-plan-3 Phase 2（amend 點 3）
    - settle use case direct flow zero-outcome guard land 於 ep-1-plan-3 Phase 3（amend 點 4）
    - WalletAllocation entity `createSettledDraft` 接 `planLines` land 於 ep-1-plan-3 Phase 1（amend 點 5）
    - directSettle use case 寫入 `planLines`（含 `lines` → `outcomeLines` 命名正名）land 於 ep-1-plan-3 Phase 2（amend 點 2）
    - `DirectSettleUseCase` 對 `planLines ∪ outcomeLines` 同步做 wallet / asset reference validation land 於 ep-2-plan-1（amend 點 7）
    - settle `outcome_excessive` 提升為 `WalletAllocationErrorType.OutcomeExcessive` land 於 ep-2-plan-1（amend 點 8）

- **Held bucket 結算機制**
  - 商戶 wallet held bucket 累積的 service fee 如何結算給 platform（service fee accumulation wallet）
  - 不在 wallet allocation 系統範圍，是 settlement run / treasury 機制設計
  - **觸發**：未來 platform fee settlement 業務需求成熟時

### 來自議題 7

- **`withdrawalIntentWalletSyncQueue`（命名暫定）大方向**
  - Mirror `depositWalletSyncQueue` pattern（議題 4.1g 不變式 1-4 直接套用）
  - 承載議題 7.2 Class 2 失敗點 #3-#9 的 catch-up sync（withdrawal cancel / reject / release / auto-release / NT event handler 觸發的 complete / fail 系列）
  - phase 5 不實作；落地時應參考議題 4.1g 不變式
  - **觸發**：未來業務需要自動補 sync（取代 manual ops）時實作

### 來自議題 8

- **Failure paths smoke**
  - withdrawal NT failed (without / with consumed network_fee) / reject / system errors / cross-instance race / projection 失敗
  - **觸發**：議題 8 餘力 / phase 6+ ops 成熟度評估

- **Wallet self TTL expire smoke**
  - Wallet `ExpireReservedAllocationsUseCase`
  - **觸發**：wallet 端獨立驗證需求成熟時

- **B.2 service integration smoke 升級**
  - spawn fund + wallet 兩 instance / 真實 BullMQ / DB fixture / RPC roundtrip
  - **觸發**：phase 5 上線後驗證需求 + ops 成熟度評估

- **B.3 app e2e smoke 升級**
  - HTTP-level full stack
  - **觸發**：phase 5 上線 + B.2 已驗證 + 業務 / 監管要求 e2e 信心

---

## Resume Point（Handoff 點）

**已收斂**：議題 1-8（完整）。Phase 5 設計收斂完成。

**下一站**：ep-2-plan-X implementation phases（落地議題 3 / 4 / 5 / 6 / 7 / 8 設計到 codebase）；`wallet-allocation-impl-codebase-survey-6.md` Gap 1-7 應作為 implementation 起點 anchor。

---

### 議題 4 進度（完整收斂）

| 子議題                                             | 狀態      | 備註                                                                           |
| -------------------------------------------------- | --------- | ------------------------------------------------------------------------------ |
| 4.1 NT event × deposit state → wallet RPC 對應拓撲 | ✅ 已收斂 | Queue 模式 + worker-from-DB + step 3.5 fence + conditional writeback           |
| 4.2 失敗補償拓撲                                   | ✅ 已收斂 | 失敗模式統一為 queue retry + catch-up                                          |
| 4.3 Allocation lines 構造（含 fee 拓撲）           | ✅ 已收斂 | 4 條 gross-record lines / declared vs realized / amount-effect builder pattern |

### 已交付給下游的事實（凍結，不重議）

- **ep-1-plan-3 已 land**（議題 4.3c / 4.3d 的 wallet 端 amend 已落地）：
  - WalletAllocation entity `createSettledDraft` 接 `planLines` 參數（Phase 1，amend 點 5）
  - `directSettle` use case 寫入 `planLines` + `lines` → `outcomeLines` 正名（Phase 2，amend 點 2）
  - `directSettle` use case zero-outcome explicit check（Phase 2，amend 點 3）
  - `settle` use case direct flow zero-outcome explicit check（Phase 3，amend 點 4）
  - Spec §7.6 / §3 字面 amend（sidecar spec patch，amend 點 6）
  - `DirectSettleInput` shape 加 `planLines` required field（amend 點 1，後續已於 ep-2-plan-1 land）
  - `DirectSettleUseCase` 對 `planLines ∪ outcomeLines` 同步做 wallet / asset reference validation（amend 點 7，已於 ep-2-plan-1 land）
  - settle `outcome_excessive` 提升為 `WalletAllocationErrorType.OutcomeExcessive`（amend 點 8，已於 ep-2-plan-1 land）

- 議題 1：ledger item schema 與 query API（已產 `wallet-ledger-item-impl-plan-draft.md` 給 Claude Code）
- 議題 2：`walletAllocationRpc` 6 method spec / `WalletAllocationCode` / `WalletAllocationDto` / `WalletAllocationLineDto`
- 議題 2 amend（已 land，主文件本次對齊）：新增 `listByFundCase` read RPC（worker step 3 catch-up + reconciliation 用）；`DirectSettleInput` 加 `planLines` field
- 議題 3：withdrawal 銜接 7 步序列 / 6 條 line 結構（含 platform 代付 network fee 模式）/ 失敗補償策略 B with stub
- 議題 4.1：deposit 採 direct flow + queue sync / `walletAllocationId: bigint | null` schema / 6 paths use case 編排（5 dispatch + Path B no dispatch）/ `depositWalletSyncQueue` 4 不變式憲法 / worker 6-step 流程（含 step 3.5 fence / 9-cell matrix / conditional writeback）
- 議題 4.2：失敗模式收斂為 queue retry + catch-up；cross-instance race 接受 narrow risk + 議題 7 兜底；DB unique constraint 推未來 TODO
- 議題 4.3：4 條 gross-record lines / amountFee 進商戶 wallet held bucket / plan vs outcome 在 success 一致 / failed 場景 plan = declared, outcome = 0 / amount-effect 三方法 builder pattern（`buildPlanLines` / `buildFullOutcomeLines` / `buildZeroOutcomeLines`）+ injectable service placement / zero outcome 不產 entry 但寫入 allocation table
- 議題 5.1：fund 對 wallet 採 per-type RPC 路線（重用 `depositRpc.getById` + 新建 `withdrawalIntentRpc.getById`）；wallet composer 按 `subjectType` 分組 batch；接口不對稱（wallet generic + fund per-entity）為有意設計，反映各自業務結構
- 議題 5.2a：fund 端 per-entity RPC 採 full `Serialized<Entity>` DTO（既有 `DepositDto` 維持 + 新建 `WithdrawalIntentDto = Serialized<WithdrawalIntent>`）；wallet 端在 composer 自行投影成 partial subject DTO
- 議題 5.2b：wallet contract-rest layer DTO 已手刻 land（`WalletLedgerItemDto` / `WalletLedgerItemFundCaseDto` / `WalletLedgerItemFundCaseSubjectDto` / `WalletLedgerNetworkTransactionDto` / `WalletLedgerNodeDto` 等）；fundCase 採 `{ type, identifier, subject }` wrapper，subject 為 uniform shape（不分 type）含 `WalletLedgerFundCaseSubjectStatus` 大集合 enum；fundCase not nullable 反映 wallet allocation invariant；ShortId 對齊三層識別字第三層
- 議題 5.4：fund 端 RPC contract 落地——新建 `withdrawalIntentRpc.getById` lookup envelope（只 getById；input `IntegerString`；return `ResultDto<CommonCode.Ok, WithdrawalIntentDto | null>`）+ 新建 `WithdrawalIntentDto = Serialized<WithdrawalIntent>`；既有 `depositRpc` 兩 method 同步 envelope 化為 `ResultDto<CommonCode.Ok, DepositDto | null>`；不建 fund-specific code enum（lookup 無 typed error space）
- 議題 5.3：wallet ledger composer cross-BC wiring——service-root `RpcClientModule` 註冊 `depositRpc` / `withdrawalIntentRpc` definitions 與 `ServiceName.Fund` services entry；composer 直接 `@InjectRpcClient` 注入兩 fund RPC client；處理 pattern 為按 `subjectType` 分組 collect → parallel batch hydrate per type → map 拼裝；其他 cross-BC client（network / merchant / wallet self）注入留議題 6
- 議題 5.5：整條 stack 對偶情況 documentation（persistence / RPC reference / RPC hydration / composer / REST DTO 五層）；候選原則 13「分清楚 > 形式對稱」正式提升為跨議題原則 13
- 議題 6.1：Projection mapping framing 採模型 A（單 row 持 plan + outcome 雙態 + append-only revision）；trigger 沿用 Phase 4 inline best-effort hook（use-case 內 try/catch + log warn）；失敗後 ledger 缺 rows 由 manual operation 修復；自動補償已於議題 7 sweep 後推 future
- 議題 6.2：path-by-path projection 行為 table 完整 codify（withdrawal reserve/bind/settle/release-from-bound + deposit propose/settle/directSettle 各 path）；directSettle 一次寫（不模擬兩階段）；release-from-bound outcomeLines=null 視為全 0；deposit path F vs path E audit trail 對等性差異視為業務事實真實映射
- 議題 6.3：WalletLedgerItemComposer constructor 與 cross-BC wiring 完整收斂——5 個 cross-BC source dependencies（fund subject RPC × 2 / networkTransaction RPC / merchant via AccountManagementClient 沿用 / wallet self via WalletQueryService.listRefsByIdIn）+ shared mapper 重用（WalletRefMapper.toRelatedWalletDto from SharedMapperModule，不新建 batch mapper）；module wiring delta 4 條 import；具體 composer file placement / 其他 mapper shape 對齊細節留 ep-2-plan-X
- 議題 6.4：Error handling 三層收斂——（1）inline hook 沿用 ep-1-plan-2 best-effort pattern（議題 6.1 已 commit）；（2）composer 內部任一 cross-BC source 失敗 → 整個 throw（不 fallback partial rows，避免違反 5.2b fundCase not nullable invariant，cron rerun idempotency 友善）；（3）backfill 介面方向 commit 走 use-case（BackfillProjectionUseCase 命名暫定）但 phase 5 不實作——主線運作不需要 backfill，缺 rows 由 manual operation 修復；BackfillProjectionUseCase 已於議題 7 sweep：phase 5 推 future，範式 queue-based

### 跨議題原則（已確立）

1. **Fund case 建立失敗才 release；一旦建立、無論業務成功失敗都 settle**（議題 3.1）
2. **夭折 vs 幽靈**（allocation 孤兒分類）—— deposit 在 queue 模式下無 inline 孤兒，但 cross-instance race 留下的多筆 allocation 屬於新孤兒類型
3. **三層識別字**：Persistence bigint / RPC bigint as string (`IntegerString`) / REST codec encoded
4. **FundCase 在 RPC 攤平、在 REST 用 wrapper**
5. **PM ledger view 對齊**（議題 1.4a）
6. **In-process queue dispatch 等同 function call**（議題 4.1g 不變式 1）
7. **Worker 不信任 message payload**（議題 4.1g 不變式 2）
8. **Use case 不 inspect walletAllocationId**（議題 4.1g 不變式 3；amended after plan-3 land）

   **適用範圍**：NT event handler use cases（`HandleNetworkTransactionDiscoveredUseCase` / `HandleNetworkTransactionSettledUseCase` / `HandleNetworkTransactionFailedUseCase` 對應 Path A-F dispatch 邏輯）。NT handler dispatch 決策純依 NT event 事實 + deposit 既有狀態，**不**依 allocation 同步狀態；schema 訊號（`null` / `0n` / `bigint > 0`）由 worker 端解讀。

   **不適用範圍**：deposit wallet sync worker 的 orchestration use-case（`ProcessDepositWalletSyncUseCase`）。Worker 必須 inspect `walletAllocationId` 在以下兩處（恰好兩處，非更多）：

- **Step 2 legacy guard**：`walletAllocationId === 0n` 識別 legacy backfill 列並 skip
- **Step 3.5 local-remote consistency fence**：`walletAllocationId > 0n` 時對照 `existing?.id`，做 fail-loud 決策

Step 4 矩陣 dispatch 純依 `(deposit.status, existing)`；step 5 writeback 是 conditional 但只用 `walletAllocationId === null` 作為 catch-up 條件，不作為新決策。

**Rationale**：原則 8 的核心目的是讓 NT handler「always dispatch + 不依 allocation 同步狀態決策」，符合議題 4.1g 不變式 4「Sync 失敗的責任完全在 queue 自身機制」精神。Worker 是 sync 機制本身，自然必須 inspect 自身負責的 schema 訊號。原則 8 與 worker inspection 並存，不衝突。9. **Sync 失敗的責任完全在 queue 自身機制**（議題 4.1g 不變式 4）10. **Allocation = business fact（含 zero）vs Entry = balance mutation log（不含 zero）**（議題 4.3d）11. **Gross recording over net recording**（議題 4.3a）—— allocation lines 採 gross 記帳，反映完整業務事件序列 12. **Builder 角色決定 attribute inspection 模式**（議題 4.3e；amended after plan-3 land）—— amount-effect naming pattern：plan builder 從 declaration source（amount fields）建構（`buildPlanLines(scalarInput)`）；outcome builder 從 plan-line identity preservation + amount effect 建構（`buildFullOutcomeLines({ planLines })` / `buildZeroOutcomeLines({ planLines })` / 視 fund case 增加 `buildNetworkFeeOnlyOutcomeLines` 等變形）。各 method fixed behavior，**caller 依事實層 dispatch（如 `deposit.status === Completed` → full / `=== Failed` → zero），不傳 status flag 給 builder**；builder 內部不 inspect status。命名描述 amount-effect（不描述 fund-side status literal），未來新 fund case type 套用同 pattern。13. **分清楚 > 形式對稱**（議題 5.5）—— 對稱性是 emergent property、不是 design goal。當底層業務結構真的不對稱時，強行對稱反而不誠實，並會帶來實際代價（paradigm split / 抽象層複雜化 / wiring 複雜化）。適用：跨 BC 接口設計、抽象層引入決策、method 數量擴張決策。不適用：codebase convention 一致性 / true symmetry 場景 14. **事後追認用 queue catch-up 為 default**（議題 7）—— 跨 BC sync 失敗時，預設由失敗點 dispatch 到 queue 補 sync（mirror 議題 4.1g `depositWalletSyncQueue`）；cron-based 只在純 wallet-side anomaly（無 dispatch point）時 fallback。適用：跨 BC sync 補償機制、worker / scheduler 範式選擇。不適用：fund/wallet internal 補償 15. **`get*` lookup nullable / `*ById` action NotFound**（本次 amend 對齊）—— lookup-style method miss 屬 expected outcome，採 nullable / empty collection；action-by-id target 預期存在，miss 才升成 `NotFound` envelope。適用：RPC / repository / query service 的 public return shape；不適用：已凍結 contract 的 backward-compat 變更，或帶其他 typed error union 的 action method

### 議題 5 推進進度 ✅

- [x] **5.1** Generic vs Per-type RPC endpoint 大方向 — 選 per-type
- [x] **5.2** per-entity RPC 暴露的 DTO shape（合併原 5.6）
  - [x] **5.2a** fund 端 RPC DTO 風格 — 採 full `Serialized<Entity>`
  - [x] **5.2b** wallet 端 `WalletLedgerItemFundCaseDto` shape — REST DTO 已手刻 land
- [x] **5.4** fund 端 RPC contract 落地 — 新建 `withdrawalIntentRpc` envelope + `depositRpc` 同步 envelope 化
- [x] **5.3** Cross-BC dependency wiring — service-root module wiring + composer @InjectRpcClient + batch dispatch
- [x] **5.5** 對偶性檢查 minimal closing — stack 對偶 documentation + 跨議題原則 13 commit

議題 5 完整收斂。本對話串接下來進入 final output 產出階段（議題 6 handoff + 副對話 ep-2-plan-1 prompt）。

### 待議題 5-8 處理的 TODO（從本輪累積）

詳見「未來 TODO 與可改善項目」章節。其中與議題 5 直接相關的：

- ~~Fund 暴露 generic / per-type 介面選擇~~ — 5.1 已收斂為 per-type；具體 contract 落地等 5.4
- ~~FundCase Context shape 對 wallet projection 的影響~~ — 5.2 已 commit
- ~~Fund 端 envelope 規範對齊（`depositRpc` envelope 化 + 新 `withdrawalIntentRpc` 直接走 envelope）~~ — 5.4 已 commit design；implementation 留 ep-2-plan-1

與議題 7 直接相關的（議題 7 已 sweep，phase 5 全推 future）：

- ~~Settle stuck retry / outbox~~ — 議題 7.1 sweep 推 future；範式 queue-based（`withdrawalIntentWalletSyncQueue`），見議題 7.4 + 跨議題原則 14
- ~~Wallet bound 孤兒清掃 cron~~ — 議題 7.1 sweep 推 future；範式不限定（pure wallet-side detection）
- ~~Cross-instance race 偵測 cron~~ — 議題 7.1 sweep 推 future；範式不限定（pure wallet-side detection）
- ~~Dead letter handling~~ — 議題 7.1 sweep 結論：DLQ 自身機制 + ops monitor 已足，不需新設計

與議題 6 直接相關的：

- ~~Deposit 在 path A→D 的 plan rows 是否在 ledger 顯示~~ — 6.1 已解（模型 A append-only supersede + filter current revision，自動不暴露 plan-only superseded rows）
- ~~Failed 場景 plan vs outcome deviation 的 ledger projection 規則~~ — 6.2 已解（path-by-path table codify failed 場景 plan = declared / outcome = 0 行為）
- ~~Realization field 語意實作~~ — 6.2 已解（path-by-path table 直接套用議題 1.2b derivation 表）
- ~~6.3 Cross-BC client 注入（補 5.3 之外：networkTransaction / merchant / wallet self）+ networkTransaction hydration~~ — 已 commit composer constructor（5 個 dependencies + WalletRefMapper 重用）+ module wiring delta 4 條 import；ep-2-plan-X 階段補 file placement / mapper shape 對齊細節
- ~~6.4 Error handling 細節 + 與議題 7 reconciliation cron 介面定義~~ — 已 commit inline hook 沿用 + composer throw + backfill 介面方向走 use-case；BackfillProjectionUseCase 已於議題 7 sweep：phase 5 推 future，範式 queue-based

與議題 8 相關的（議題 8 已收斂，scope 限縮為 Layer 1 happy）：

- ~~Smoke test path scope~~ — 議題 8.1 commit Layer 1 happy paths only；failure paths / wallet 自治 cron / B.2 / B.3 升級全推 future（見 TODO 章節「來自議題 8」）
- ~~Test infrastructure layer~~ — 議題 8.2 commit B.1 use-case spec（對齊既有 `*.use-case.spec.ts` pattern）
- ~~Layer 1 happy path scenarios codify~~ — 議題 8.3 commit 3 path scope（withdrawal happy / cancel / deposit happy）+ spec count 推 Codex enumeration

### 來自議題 6.4

- **`BackfillProjectionUseCase` 實作**（議題 7 已 sweep）
  - Wallet-side 公開 backfill use-case；介面方向命名暫定 `BackfillProjectionUseCase`（cron / queue worker 走 use-case 不直接 invoke service）
  - **議題 7 已 sweep**：phase 5 推 future 實作；範式 queue-based（projection 失敗點在 use-case 內，可 dispatch — 跨議題原則 14 default 範式）；具體是否與 `withdrawalIntentWalletSyncQueue` 共用 / 或新建 queue 留實作時評估
  - Phase 5 主線運作不需要——projection 99% 成功，缺 rows 由 manual operation 修復；自動 backfill 屬「成熟運維」需求，YAGNI 適用

未來 phase amend：

- ✅ Wallet 端 `directSettle` / `settle` use case 加 zero check（已於 ep-1-plan-3 Phase 2 / 3 完成）
- ✅ `DirectSettleInput` 加 `planLines` required field（已於 ep-2-plan-1 land）
- ✅ `DirectSettleUseCase` 對 `planLines ∪ outcomeLines` 同步做 wallet / asset reference validation（已於 ep-2-plan-1 land）
- ✅ settle `outcome_excessive` 提升為 `WalletAllocationErrorType.OutcomeExcessive`（已於 ep-2-plan-1 land）
- Wallet allocation `(fundCaseType, fundCaseIdentifier)` partial unique index（未來 phase 4 amend）
- ✅ Spec §7.5/§7.6 amend（zero outcome 例外已由 ep-1-plan-3 sidecar patch 完成；directSettle planLines input 已於 ep-2-plan-1 land）

---

### 本輪同時產出的獨立交付文件

- `wallet-ledger-item-impl-plan-draft.md`：Phase 1 only，可立即交給 Claude Code 實作 contract-rest dto / api 定義給前端 preview。Phase 2 之後留給下一輪設計收斂後再產 plan。

### 下一輪對話的啟動建議

新對話的 system prompt 應包含本文件 + `fund-withdrawal-intent-codebase-survey.md`（議題 3 input）+ `fund-deposit-codebase-survey.md`（議題 4 input）。議題 5 可能需要新的 codebase survey（fund case context / projection 相關），啟動時評估是否先做 survey。

---

### Context 與 Handoff 評估（從議題 4 末對話）

- 本輪累計 context 預估 50-55%
- 議題 5 進行中可能達 65-70%
- **建議在議題 5 結束後 handoff**：5 與 4 概念連續性強（fund case 與 wallet 銜接），同 context 推進更順；6/7/8 是新主題（projection / scheduling / testing），自然斷點

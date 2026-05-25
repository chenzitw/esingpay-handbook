---
status: draft
updated_at: 2026-05-04
updated_by: Tim
remark: 本文件未依照 proposal 規範撰寫，暫供內容參考，等待後續重構改寫。
---

# 提案：正視 Fund Case 概念

**Status**: ✅ Approved
**Audience**: esingpay 架構成員、其他 AI 對話實例、ChatGPT
**核心決議**: 在 `@esingpay/contract-base` 定義 `FundCaseType` enum，承認 Fund Case 為跨 domain ubiquitous language

---

## 概述

esingpay Fund domain 持有多個業務 aggregate(`Deposit` / `WithdrawalIntent` / `TransferIntent`，以及未來可能但**未確定**會發生的其他類型)，需要一個對它們的共同稱呼，作為跨 domain 引用、API 契約、與潛在 read model grouping 的 first-class 詞彙。

本提案：

- **承認 "Fund Case" 為 ubiquitous language**: 屬 Fund domain 主權，作為跨 domain 引用的標準形狀
- **不**將 FundCase 實作為 write-side aggregate;既有業務 aggregate 維持獨立
- **立即落實項目(範圍刻意收緊)**: 在 `contract-base` 定義 `FundCaseType` enum
- **適用範圍**: 以新功能為主;既有命名(`subject*`、`correlation*`)不強制遷移

---

## 主要理由: 完善 Fund 內外的邊界定義

esingpay 採 service-per-domain 結構，Fund / Wallet / Network 各自獨立。當 wallet 與 network 都需要儲存「這筆資料對應 fund 的哪一筆業務流程個體」時，缺少 umbrella concept 會在邊界上產生兩種隱性錯位。

### Fund 自身擴展自由

Fund 加新業務 aggregate 時，引用方的 schema、enum、validation rule 都必須跟著調整。Fund 的內政變動會穿透到別的 domain。這違反 service-per-domain 的初衷——Fund 應該能在自己的邊界內自由演化。

引入 FundCase 概念後：

> Fund 擁有 `FundCaseType` 字典與 case 識別權。Fund 加新 case，只動 fund 自家 contract。引用方無需參與 fund 的演化決策。

### 引用方引用惰性

引用方(wallet / network / 未來其他 domain)若要描述自己的 invariant(例如「allocation 必由恰好一筆 fund-side 業務個體驅動」)，現況下找不到合適的詞彙，只能維護一份本地的 `subjectType` enum 字典。

這份字典本質上是**引用方在替 fund 維護 fund 的字典**——typing layer 上做的事，是 anti-corruption layer 的工作，但 fund / wallet 之間本來不需要這層翻譯。

引入 FundCase 概念後：

> 引用方不解讀 type 內容、不維護字典;只要透過 `(type, identifier)` 引用即可。Fund 加新 case 時，引用方的程式可零修改地透傳。

### 邊界形狀的標準型

這個邊界形狀對應 DDD 的 customer-supplier relationship：

- **Fund** 為 supplier，定義 `FundCaseType` 字典與對外契約
- **Wallet / Network 等引用方** 為 customer，惰性消費契約

**這是本提案的主要架構價值**，比任何具體的下游應用(read model、後台流水簿等)都更根本。下游應用是這個邊界形狀的副產物，不是它的主要驅動力。

### 對既有顧慮的處理

引入 umbrella concept 過去曾擔心會誘發 Fund domain 出現 abstract write-side aggregate / base class，壓平 Deposit / WithdrawalIntent / TransferIntent 真正的差異。

本提案明確將 **「FundCase 不是 write-side aggregate」** 列為設計紅線。Fund domain 內部如何實作 aggregate，是 fund 自家的設計選擇——而**引用方惰性引用這條邊界，反而保證**了 fund 內部結構自由:不論 fund 選擇保持各自獨立、或選擇某種共用基礎，都不影響引用方。

---

## FundCase 是什麼，不是什麼

### FundCase **是**

- Fund domain 的命名約定，指稱 Fund-owned 業務流程個體之一
- 跨 domain 引用的標準 reference 形狀:`{ type, identifier }`
- 跨資源潛在的 grouping key
- 後台 / customer service / ops 場景的 first-class 業務詞彙(如未來相關需求出現)

### FundCase **不是**

- ❌ Write-side aggregate
- ❌ Abstract base class / interface
- ❌ 共用 lifecycle state machine
- ❌ 共用 invariant 或 validation rule
- ❌ Fund domain 內部的 orchestration entity

**這條紅線守得住，提案就成立;守不住，提案就應被質疑或撤回。**

---

## 適用範圍

**以新功能為主**：

- 新建立的 schema、entity、DTO 採用 `fundCase` 命名
- 新的 RPC / REST contract 採用 `fundCase` wrapper(具體形狀見下節)
- 新加入的 Fund-owned 業務個體(若有)加入 `FundCaseType`

**既有命名不強制遷移**：

- Legacy `correlation: {}`、`subject: {}` 定義可未來安排遷移重構(不在本提案承諾範圍內)

---

## 立即落實項目

> **在 `@esingpay/contract-base` 定義 `FundCaseType` enum**

Initial values 至少包含當前 fund 已存在的業務個體類型：

```ts
// contract-base
export enum FundCaseType {
  Deposit = 'deposit',
  WithdrawalIntent = 'withdrawal_intent',
  TransferIntent = 'transfer_intent',
}
```

**這是本提案承諾的全部立即動作。** 其餘下游採納(wallet / network / read model 是否使用 `fundCase` wrapper、phase 4 是否同步 rename、fund domain 內部是否寫專屬 spec 章節等)皆為**獨立決議**，不在本提案承諾範圍內。

---

## 跨領域(Domain / Service)概念採納建議

當 wallet、network、或其他 domain 的 schema / DTO 需要引用 Fund-owned 業務流程個體時，**建議**採用以下形狀：

```ts
{
  fundCase: {
    type: FundCaseType;
    identifier: string;
    subject: ...;        // hydrated 主體(變形如下)
  }
}
```

### Subject 形狀的變形

`subject` 欄位的形狀**不強制統一**——由各 consumer 的需求決定：

- **Full subject**:完整 hydrated 業務個體 DTO(適合 detail view)
- **Partial subject**:只攜帶 list view 所需欄位(適合 list / search 結果)
- **No subject**:僅攜帶 `type` + `identifier`，不 hydrate(適合純 reference 場景)

各 consumer 可依自身需求選擇變形。**重點是 `fundCase: { type, identifier }` 這個 reference 部分維持一致**，subject hydration 程度可變。

### 與 Wallet / Network domain 的關係

- Wallet domain 對 allocation / ledger / cashflow 等資源**建議**採用此 wrapper 或其變形
- Network domain 對 transaction 與業務歸屬欄位**建議**採用此 wrapper 或其變形(可 nullable，因未識別的 inbound transaction 不對應 case)
- 不說死一定要等價形狀;以 reference 部分(`{ type, identifier }`)一致為核心

---

## 建議的可能衍生方向(非承諾)

以下方向皆為可能落實途徑，**不構成本提案的承諾項目**。實際是否執行、執行順序、執行範圍由後續各自決議。

### Domain Shape 層

可能方向：

- Fund spec / system overview 增寫 "Fund Case" 概念章節
- Fund domain 內部各 aggregate 章節維持獨立
- 若 Fund 未來引入新類型(charge / refund / 其他)，加入 `FundCaseType` enum

不建議的方向：

- 在 Fund 內部建立 `FundCaseEntityBase` / abstract aggregate(破壞紅線)
- 將既有 aggregate 合併重構為 FundCase 的 subtype

### RPC 層

可能方向：

- Fund 提供 `fundCaseRpc.getByRef(type, identifier)` 統一 hydration 介面
- 或：Fund 各 aggregate 維持各自 RPC，consumer 自行 dispatch by type
- Composer batch 取 case 的接口設計依實際使用情境調整

### REST 層

可能方向：

- 新 wallet / network endpoint 採用 `fundCase: { type, identifier, subject }` 形狀或其變形
- 既有 endpoint 不強制遷移
- Subject 形狀(full / partial / none)由各 endpoint 自選

### Read Model 層(純可能性)

可能方向：

- 若未來真有跨 case grouping 的 read model 需求(例如業務活動流水簿、跨類型後台查詢等)，可考慮由 Fund domain 擁有的 read model(如 `FundCaseActivity` / `FundCaseEvent`)

**注意**:這類需求目前**僅為舉例**，並非已存在的真實需求，不構成本提案的論證基礎。本提案的論證重心在邊界定義(前述「主要理由」)，不依賴下游 read model 應用是否實現。

---

## Open Points

1. **`FundCaseType` enum 的初始 value 列舉**
   建議至少 `deposit` / `withdrawal_intent` / `transfer_intent`。其他類型(charge / refund 或其他)皆為**待定**，加入時間點未定，可能不發生。

2. **`subject` 內層欄位名是否最佳**
   候選包含 `subject`(業務詞)、`summary`(技術詞)、`detail`(暗示完整投影)。傾向 `subject`，但不在本提案承諾範圍內，由各 consumer 自決。

3. **`FundCaseType` 在 `contract-base` vs 未來 `contract-fund` 的歸屬**
   立即落實放在 `contract-base`，避免立刻引入 fund-specific contract package 的依賴。未來若 fund domain contract 獨立成 package，可考慮遷移。

4. **Wallet allocation phase 4 是否同步採用**
   Phase 4 impl plan 與已寫的 entity / model / mapper 是否同步 rename `subjectType` → `fundCaseType`，為**獨立決議**，不在本提案承諾內。時機評估顯示窗口未關閉前成本較低，但是否動作由 phase 4 主導者決議。

5. **既有 `correlation: {}`、`subject: {}` 的長期遷移**
   兩者皆為 legacy 命名形狀,可未來安排遷移重構至 `fundCase: {}`。具體時機與範圍另議,本提案不承諾。

6. **TransferIntent 多腿情境的 grouping**
   一個 fundCase 可能對應多個 wallet allocation。Wrapper 形狀沒問題，但跨 read model 的 group by 設計待後續釐清。

7. **「FundCase」英文詞本身**
   "Case" 在客服 / Salesforce / ops 語境很常見。金融 PSP 業界是否有更貼合的標準詞，可繼續討論。若有更佳替代，本提案可被該替代取代。

---

## 附錄：論證的元層次說明

本提案在較早期討論中曾被 Claude(本提案實際撰寫者之一)反對，主要顧慮是「會誘發 abstract aggregate」與「不是 ubiquitous language」。立場轉變的關鍵：

1. **「Case」不必然是 aggregate**——Salesforce / Zendesk 領域的 "case" 是 lifecycle wrapper 與命名約定，不是 entity hierarchy。命名約定不會自動誘發 abstract write-side base class。

2. **真正的論證重心應落在邊界形狀**——本提案的主要理由(Fund 自身擴展自由 + 引用方引用惰性)是架構性的。早期論證過度依賴下游應用作為理由(業務活動流水簿、charge / refund 等)，反而讓提案的架構價值被掩蓋。最終版本將下游應用降級為舉例與可能性。

3. **「不是 ubiquitous language」此論的反方向**——若 Fund domain 不給出 umbrella term，引用方就只能本地維護 type 字典，這是**更糟**的非 ubiquitous 結果。引入 FundCase 反而是讓 ubiquitous language 浮現，而不是壓抑它。

紅線(FundCase ≠ write-side aggregate)是先前顧慮的延續，不是消解。守住紅線就守住先前的擔憂。

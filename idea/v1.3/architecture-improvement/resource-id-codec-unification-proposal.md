---
updated_at: 2026-06-26
updated_by: Tim
---

# Resource ID Codec 統一與 Sqids 用法改善 — Proposal

> 記錄已收斂方向(D1–D12):codec 統一、Sqids canonical 修復、id 去關聯(`typeCode`)、拆碼封裝、id 安全定位(純識別碼 + 授權把關,非安全邊界)。Sqids 用法改善已收斂;餘為 blueprint 細節。

## Context

目前對「面向外部的資源 id」由兩個彼此獨立的工具協作產生:

- 一個**短碼 codec**(Sqids 基底),負責 `bigint ↔ 不透明短碼字串` 的混淆與還原。它是 DI 注入的 component。
- 一個**資源前綴工具**,在短碼外層套上型別前綴(deposit / transfer-intent / withdrawal-intent 各有自己的前綴),以一組 resource-type 列舉為鍵。它是 static 的常數物件。

實務上兩者**幾乎總是被串起來**:出站先把 bigint 編成短碼、再套前綴;入站先剝前綴、再把短碼解回 bigint。但這個串接沒有被收斂成單一操作——它被**手寫複製在十多個出站 mapper 與數個入站 mapper**,各自重複同一套組合與 null 處理。同時,短碼層的 Sqids 用法本身也存在待改善之處(含同事回報的 bug)。

為何現在處理:external-api 正在持續擴張對外 endpoint,每新增一條對外資源都會再複製一次這套手寫串接。趁對外 surface 還小、複製點還可控時收斂,成本遠低於日後再回頭整理。

## Proposal

把「對外資源 id 的編解碼」收斂為**單一 codec**,由它擁有 `bigint ↔ 對外 id` 的**完整往返作為單一操作**,呼叫端不再手寫兩層串接;前綴改為由**集中的資源型別目錄**驅動,並一併改善 Sqids 用法。

一句話:**一個 codec、一個集中的資源型別目錄、一次往返;前綴由目錄決定,呼叫端只表達「這是哪種資源」。**

### 已收斂決策(本版留底)

| #   | 決策                 | 內容                                                                                                                                                                                                                                                                                                                                                                              |
| --- | -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | 收斂形態             | 兩個工具合為**單一 injectable service**(短碼混淆層 + 前綴層同一個家、同一種取用方式)                                                                                                                                                                                                                                                                                              |
| D2  | API 形狀             | 單一入口 `encode/decode(id, type)`;**type 必填**;`encode` 與 `decode` 雙向皆驗證型別(decode 期望型別不符即拒絕,防型別混淆)                                                                                                                                                                                                                                                        |
| D3  | 失敗語意             | decode 失敗回 **nullable / total**,不細分失敗原因;REST 例外仍由 **adapter 層** 以既有 factory 轉換,codec 維持 transport-agnostic                                                                                                                                                                                                                                                  |
| D4  | 對外 id 型別         | 在 contract-base 新增 **doc-only `type ResourceId = string`**(配註解說明「裸短碼或帶前綴」),非真 brand,沿用既有 `ShortId` 的 alias 慣例                                                                                                                                                                                                                                           |
| D5  | 放置與真實來源       | codec 機制 + `ResourceType` 目錄 + registry 全置於 **service-kit**(跨服務共享);registry 值由「只放前綴字串」升級為 **`ResourceIdScheme { prefix: string \| null; typeCode: number \| null }`**,完整 `Record<ResourceType, ResourceIdScheme>`(prefix / typeCode 各自可空,彈性最大)                                                                                                 |
| D6  | `General` 用途       | **保留** `General` 為預設 scheme `{ prefix: null, typeCode: null }`(→ 裸碼 `[high,low]`);不需客製的資源一律歸 `General`,需客製前綴/去關聯者再**自行擴充 `ResourceType`** 並定義其 scheme                                                                                                                                                                                          |
| D8  | `ResourceId` 放置    | 置於 **contract-base**;contract-rest 的 DTO 欄位引用之(依賴方向 service-kit → contract-base,contract 套件不依賴 service-kit)                                                                                                                                                                                                                                                      |
| D9  | Sqids canonical 驗證 | decode 末端**重新 encode 並比對輸入**(round-trip),不符即回 `null`;**不依賴 length 檢查**(已證實無效)。修復**併入統一 codec**(排序採乙),不另做即刻小修;由統一後的 codec 保證此性質                                                                                                                                                                                                 |
| D10 | 去關聯(encode 因子)  | scheme `typeCode` 非 null → encode `[typeCode, high, low]`(同數字不同資源 → 尾碼不同,消除肉眼關聯);null → `[high, low]`。逐資源 opt-in,可零遷移維持現況(`typeCode: null`),要去關聯時再設值(該資源此時遷移)。`typeCode` 人工指派、**不強制唯一**(設相同 = 該組刻意不去關聯)。decode 走 D2 per-type 分支 + D9 canonical,新舊 scheme 不歧義。**僅擋肉眼、非安全**(decode 者仍可反解) |
| D11 | 拆碼封裝             | high/low 拆碼為 codec 私有實作,公開 API 只進出 `bigint`;維持現行 **2×32**(byte-stability,涵蓋至 2^64)                                                                                                                                                                                                                                                                             |
| D12 | id 安全定位          | **純識別碼**;存取一律由**伺服器端授權**把關(猜到 id 也讀不到資料);文件明載「**id 非安全邊界**」。**不**做帶金鑰加密、**不**導入自訂字母表(在此定位下為 obfuscation theater、可被持有字母表者還原)。可逆/枚舉造成的 metadata 外洩視為可接受                                                                                                                                        |

> (D7 為 migration 範圍/分階段,屬 blueprint「怎麼推」altitude;其 API 形狀部分已由 D2 決定,blast radius 見 Scope of impact,phasing 留 blueprint。)

### Before → After(示意)

> 僅為說明統一前後的呼叫差異,非實作規格;完整 call pattern 留待 blueprint。以 deposit id 為例。

**出站(encode)**

```ts
// Before:兩層手動串接
id: ResourceIdCodec.encode(
  ResourceType.Deposit,
  this.shortIdCodec.encodeBigint(input.deposit.id),
),

// After:單一往返
id: this.resourceIdCodec.encode(input.deposit.id, ResourceType.Deposit),
```

**入站(decode)**

```ts
// Before:剝前綴 → null check → 解短碼(中間值需 as 轉型、兩段 null)
const encodedId = ResourceIdCodec.decode(ResourceType.Deposit, input.depositId);
if (encodedId === null) {
  return null;
}
return this.shortIdCodec.decodeToBigint(encodedId as ShortId);

// After:單一往返,失敗單一 null
return this.resourceIdCodec.decode(input.depositId, ResourceType.Deposit);
```

無前綴資源(如 wallet)同樣經統一入口:`this.resourceIdCodec.encode(input.id, ResourceType.General)`。

## Motivation

- **重複的手寫串接**:出站與入站各自把「短碼層 + 前綴層」的組合與 null 處理重寫在十多處;任一層語意調整需逐點修改、易漏。
- **放置與生命週期不一致**:同一個「對外 id 表示法」被拆成一個 injectable component 與一個 static 常數,呼叫端被迫「注入一個、import 另一個」。
- **型別與失敗語意散落**:剝前綴後的中間值需強制轉型才能進下一層;解碼失敗被各呼叫點各自詮釋。
- **前綴真實來源分散**:前綴對照與型別列舉分離,且無「哪些資源該帶前綴」的成文依據,易隨手新增而漂移。
- **缺乏跨服務的資源感知**:資源型別目錄綁在單一 app,真微服務化後其他服務無法一致辨識/驗證資源 id。
- **Sqids decode 過寬(canonical bug,live)**:`decodeToBigint` 只檢查 payload 長度、未驗 canonical,導致非 canonical 字串(尾碼垃圾)解出相同數字仍被接受——例如 `WI_74dBD7` / `WI_74dBD7X` / `WI_74dBD7X2` 都查到同一筆資料。實測確認,影響所有走 decode 的入站 id(deposit / transfer-intent / withdrawal-intent / wallet / ledger query)。此為正確性/越權查詢風險,需在統一中一併修。

不做的後果:上述每一條都會隨 external-api 擴張被線性放大——複製點更多、分歧更深、日後收斂成本更高。

## Trade-offs

- **Pros**
  - 對外 id 編解碼成為單一、可測試、可演進的操作;新增對外資源時不再複製串接。
  - 中央資源型別目錄使跨服務對「系統有哪些資源」有共同感知。
  - 單一取用方式、單一前綴真實來源;`null` 前綴在完整 `Record` 下仍受編譯器強制,避免靜默漏前綴。
  - 失敗語意集中一處定義;`type` 必填使「為某資源編 id」一定帶意圖。
  - decode 帶型別,讓未來向下相容擴充能**封裝在各型別內**(per-type 相容分支),不受其他型別干擾。
- **Cons**
  - **Migration blast radius 大**:若貫徹單一公開入口,目前約 59 處 encode 與多處 decode(含 wallet/network 等裸碼站點)都需改走統一入口,屬廣度大、單點淺的遷移。
  - 遷移期間必須確保**對外 id 輸出位元級不變**(既有對外 id 不可改變編碼結果),為硬約束、需驗證。
  - `General` 可能變成偷懶預設:該標特定型別者圖方便填 `General`。型別必填擋得住「漏填」,擋不住「填錯成 General」,只能靠命名語意 + review 緩解。
- **Unknowns**
  - Migration 的分階段策略與「輸出不變/輸出變更」的驗證方式,以及哪些既有資源實際設定 `typeCode`(屬 blueprint)。

## Rejected alternatives

- **維持現狀(不做)**:問題隨 external-api 擴張被放大,複製點與分歧只增不減。
- **只加 facade、保留兩個獨立工具**:消除呼叫端重複,但放置不一致、真實來源分散、跨服務無感知等根因仍在,屬半套。
- **decode 自動偵測型別(不帶 type)**:此處無「收混合資源 id 的多型入口」,自動偵測用不到,卻丟失型別混淆驗證、且在格式演進下反查易歧義。
- **branded `ResourceId`(真 brand)**:`as` cast 痛點已由 D1 統一消除;真 brand 跨 package ripple 大、與 `General` 無前綴形式有型別衝突,邊際效益不足。改採 doc-only alias。
- **前綴表用 `Partial<Record<…>>`**:放棄編譯器強制「每型別宣告前綴決定」,等於把靜默漏前綴的 footgun 請回。改用完整 `Record` + `null`。
- **強制所有資源都帶前綴**:多數資源不需要,徒增對外契約變更面與遷移成本而無收益。改採 `General` 預設無前綴。
- **per-resource 不同字母表**:前綴 + type 驗證已區分資源,per-resource 字母表無收益,且把 domain-agnostic 短碼層重新耦合資源分類。改採全系統單一字母表。
- **把 type/salt 塞進 Sqids payload 當保密**:Sqids 無金鑰,decode 原封吐回,salt 形同公開、無法達成不可猜。去關聯改用 `typeCode`(D10,僅擋肉眼);真保密須帶金鑰加密。
- **以 length 檢查取代 canonical 驗證**:已實測證實無效——解出的陣列長度恆為 2(尾碼垃圾照樣通過);字串長度浮動(8~9 碼),固定長度檢查會誤殺合法值且擋不住同長度非 canonical 字串。改採 re-encode round-trip(D9)。
- **canonical bug 獨立即刻小修(甲案)**:統一解方即將實現,獨立小修為即將被丟棄的重工。改採併入統一(乙案,D9)。

## Scope of impact

通過後預期觸發:

- **service-kit**:新增統一 codec、`ResourceType` 目錄、`ResourceIdScheme` registry。
- **contract-base**:新增 `ResourceId` doc-only 型別。
- **cradle adapter / mapper**:遷移 id 編解碼站點至統一入口(blast radius 如 Cons 所述)。
- **Guide 更新**:可能需在 contract / naming 相關 guide 補「對外 id 表示法」原則(哪些資源帶前綴、由誰決定、放在哪一層),並載明「**id 非安全邊界**」之系統定位(D12)。
- **下游**:啟動 blueprint(codec API、目錄與前綴真實來源、失敗語意、Sqids 改善、遷移分階段與輸出不變驗證),再進 plan。

## Sqids 用法改善

- **canonical bug — 已裁定**:見 D9。修法 = decode 末端 re-encode round-trip 比對,併入統一 codec(乙)。同事的 `plan/v1.3/external-api/plan-bug-short-id-codec-canonical.md` 為**事前研究參考**,其分析與本 proposal 一致,定位為**由本決策取代**(不另行獨立實作)。
- **去關聯 — 已裁定(D10)**:以 scheme `typeCode` 混入 encode 值達成,逐資源 opt-in;見 D10。
- **拆碼封裝 — 已裁定(D11)**:見 D11。
- **id 安全定位 — 已裁定(D12)**:選「純識別碼 + 伺服器授權把關、id 非安全邊界」。據此**不**做帶金鑰加密、**不**導入自訂字母表(Sqids 無金鑰,payload 塞 type/salt 都會被 decode 原封吐回,obfuscation 無實質價值)。
- **字母表**:全系統單一共用,不做 per-resource;且不導入自訂字母表(見 D12)。

Sqids 用法改善至此收斂(D9/D10/D11/D12);無待裁定項。

## Decision request

請求裁定:**接受本版已收斂方向(D1–D12)作為後續 blueprint 的基礎**。Sqids 用法改善已收斂,proposal 主張完整;餘為 blueprint 細節。

## Decision

- 本版收斂決策(D1–D12)經主管(Tim,系統架構主導者)於 2026-06-26 裁定**原則接受**,作為 blueprint 基礎。
- canonical bug 修復排序裁定為**乙(併入統一)**;`plan-bug-short-id-codec-canonical.md` 為研究參考、由本決策取代。
- id 安全定位裁定為**純識別碼 + 伺服器授權把關(id 非安全邊界)**;不做帶金鑰加密、不導入自訂字母表。
- Sqids 用法改善已收斂(D9/D10/D11/D12),proposal 主張完整;餘項(migration phasing、各資源 `typeCode` 指派)屬 blueprint。
- frontmatter `status: draft` 已移除(文件編輯穩定、本輪決議完成)。

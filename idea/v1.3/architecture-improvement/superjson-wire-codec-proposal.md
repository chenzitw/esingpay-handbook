---
status: draft
updated_at: 2026-06-26
updated_by: Claude
---

# SuperJSON Wire Codec — Proposal

> **聯合提案**:與 [messaging-transport-proposal.md](messaging-transport-proposal.md) 聯合審議——本提案定 wire codec(RPC + event 兩 channel 的序列化與分層),該案定 event transport(Azure SB)與 envelope 結構。兩案引用互相對齊。

## Context

兩條 typed transport channel——RPC(同步請求-回應)與 event(非同步事實)——都要把 domain 物件送過 wire。目前兩端都沒有共用的序列化邊界:wire 上只能走 JSON-safe 值,`bigint` / `Date` 等非基礎型別必須在契約層先降階。

RPC 端現況:`contract-rpc` 以 `Serialized<T>` 把 `bigint` → `IntegerString`、`Date` → `IsoDateTimeUtc`,caller 收到後再 `BigInt(...)` / `new Date(...)` 還原。轉換散在 server mapper、caller use-case、shared mapper、tests。典型例:`WalletAllocationDto = Serialized<WalletAllocation>`,input 的 `identifier: IntegerString`、`reservedUntil: IsoDateTimeUtc`。

值得釐清:`Serialized<T>` 不只是「字串化」——它同時把 `Map` / `Set` / function / symbol 映成 `never`,在型別層**強制** wire payload 結構安全。它承擔了兩個角色:JSON-safe 降階,與結構守門。

Event 端現況:`service-kit` 的 event runtime(`createEventPublisher` / `NestEventTransport`)目前直接把 envelope emit,沒有 codec;`contract-event` 除 debug bus 外幾乎是空的。聯合提案 `messaging-transport-proposal.md` 定 event 的 envelope 結構與「SuperJSON 僅作用於 `payload` 子樹」;wire codec 的正式定調(含 event 層採用)由本提案擁有,兩案聯合審議。

既有的 `proposal-rpc-superjson.md`(v1.2 草稿,自承未依規範)提出 RPC 採 SuperJSON,但範圍只限 RPC、把 SuperJSON 當終局、且主張大規模移除 `Serialized<T>`。本提案重寫並取代它,改以**向下相容的新路**定調,並把兩條 channel 的 wire 序列化統一收斂。

## Proposal

在 `service-kit` runtime 的 transport 邊界,為 **RPC 與 event 兩條 typed transport channel 開一條 SuperJSON wire codec 的新路**:契約層可選擇宣告 rich domain 型別(`bigint` / `Date`),由 codec 在 wire 邊界序列化,業務端操作 rich domain object、對 codec 無感。

此路**向下相容、零強制遷移**:既有 `Serialized<T>` 契約一行不改,與新路永久並存。SuperJSON 為 interim 編碼,終局指向 schema-first 中性格式。本提案擁有兩條 channel 的 codec 與分層決策。

**範圍邊界(非目標)**:本提案只解「序列化 / domain object 保真」這一條軸。RPC 的調用 ergonomics(`ResultDto` 收斂、逐 code 錯誤處理、client 注入樣板)**不在此案**,屬另案。現階段即時 payoff 在 **RPC**;event 端 `contract-event` 尚空、consume 併入 SB runtime design,屬「對齊方向、隨 SB 生效」,非當下見效。

## 開新路,不是遷移

關鍵基調:這是**加一條 lane**,不是拆掉舊的。

- **`Serialized<T>` 永久續用**。既有契約維持 `Serialized<T>` + caller 手動還原,**不改、不棄、不視為待清理**。它也仍是 REST / legacy 等 JSON-safe 邊界的型別。
- **rich lane 為 opt-in**。新契約(或日後自願遷移者)才宣告 rich `bigint` / `Date`,由新的 `WireSafe<T>` 守門、codec 承載。
- **兩 lane 並存,per-contract 自選**。`Serialized<T>`(JSON-safe lane)與 `WireSafe<T>`(rich lane)是兩種型別約定,各契約挑一條。

具體形狀(illustration;確切型別實作留 design):

```ts
// Before — JSON-safe lane:Serialized 降階,caller 手動還原
type Output = ResultDto<CommonCode.Ok, Serialized<WalletAllocation>[]>;
const id = BigInt(result.data[0].id);                        // 手動還原

// After — rich lane(opt-in;既有契約留 Serialized lane、不動):
//   WireSafe 顯式包住「codec 空間子樹」,契約直接引用 domain 型別、不另立 alias
type Input  = WireSafe<{ type: FundCaseType; identifier: bigint }>;    // request 無封套 → 整包守
type Output = ResultDto<CommonCode.Ok, WireSafe<WalletAllocation[]>>;  // response → 只守 data 子樹
listByFundCase: defineRpcMethod<Input, Output>(),
const { id, reservedUntil } = result.data[0];                // 直接 bigint / Date,無還原
```

**向下相容性質**:既有 `Serialized<T>` 值本已 JSON-safe,過 SuperJSON codec round-trip 後結果不變——因此「打開 codec」是一次原子部署、不改變任何既有契約的可觀察行為。這把「打開 codec」與「逐契約遷移到 rich type」**解耦**:codec 一次到位,契約各自挑時間遷移、或永遠留在 `Serialized<T>`。

(系統部署為協調式、兩端一起更新,不存在「一端有 codec、一端沒有」的脫鉤狀態;因此 codec 不需為混版相容設計 marker / fallback——此屬下游 design,非本提案議題。)

## Layering — envelope plain,payload SuperJSON

對齊 `messaging-transport-proposal.md` 的 event 決策,兩條 channel 同一原則:**routing / 語意 scaffolding 維持 plain 中立,domain data 子樹才進 codec**。

- **Event**:`EventEnvelope = { id, occurredAt, payload }`。`id` / `occurredAt` plain,使 broker routing、DLQ triage、tracing 能不過 codec 讀取;`payload` 子樹是 codec 空間。
- **RPC response**:`ResultDto = { code } | { code, data }`。`code`(status / error union)是語意 routing 層,維持 plain;`data` 子樹是 codec 空間。
- **RPC request**:無 code 封套(routing 由 RPC topic 表達),input 物件整體即 payload、屬 codec 空間。

因此 codec 作用的是 `data` / `payload` / input 子樹,**不**碰 `code`、`id`、`occurredAt` 這類語意 / 中立 scaffolding。分層把序列化的 blast radius 鎖在 payload:遷往 schema-first 時 envelope 已中立、只需替換 payload codec。

## Positioning — interim,非終局

SuperJSON 是當前 wire 編碼,**不是終局**。終局指向 schema-first 中性格式(protobuf / Avro)。採 SuperJSON 而非現在就寫 schema-directed codec 的理由:終局屆時逐 contract 的 schema 本來就要寫,先寫一份 runtime schema 是會被丟棄的中間投資;現階段以 SuperJSON 自反解撐過、跳過中間態。

對價是 **rich lane 的型別自律**,由 `WireSafe<T>` 在型別層落實:

- **准**:基礎型別、branded 基礎型別(`NumericString` 等)、`bigint`、`Date`、plain nested object、array。
- **禁**:`Map` / `Set` / function / symbol。

判準是「**是不是合法的 domain object**」:domain object 本來就不拿 `Map` / `Set` 當欄位型別,所以此自律是把既有 domain modeling 紀律編碼進型別,非外加的稅。

`WireSafe<T>` 是 **identity-or-never 守門**:合規則等於原型別,含禁用結構則為 `never`——不轉換、不字串化,這是它與 `Serialized<T>` 的本質差異(後者把 `bigint` / `Date` 字串化降階)。

它**顯式寫在契約 define 處**(`defineRpcMethod` / `defineEvent`),包住「codec 空間子樹」:request 整個 input、`ResultDto.data`、`EventEnvelope.payload`;plain 的 `code` / `id` / `occurredAt` 不套。因此 **guard 範圍 = codec 範圍**,落在同一子樹。in / out 的不對稱(request 整包守、response 只守 `data`)正對映 envelope 結構,非隨意。**不藏進 envelope 型別、不另立 alias**——守門在宣告處一眼可見(代價見 Trade-offs)。

此自律是日後機械化遷移到中性格式的對價;RPC 與 event 兩端共守。`Serialized<T>` 與 `WireSafe<T>` 因此分工明確:前者守 JSON-safe lane(降階 + 守門),後者守 rich lane(只守門、放行 `bigint` / `Date` 交 codec)。

## Motivation

**讓契約直接說 domain object。** 標準就一條——契約宣告**合法的 domain object**,其自身型別(`bigint` ID、`Date` 時間戳、`NumericString` 金額等 branded 型別)原樣過 wire,免去 `Serialized<T>` shadow DTO 與散落各層的手動降階 / 還原。(`NumericString` 是 domain 對小數金額的正解、非 wire workaround,故原樣保留;rich 化的是被 `Serialized` 降階過的 `bigint` / `Date`。)既有契約零動,churn 最小。

**單一序列化故事。** RPC 與 event 共用同一 codec 邊界、分層原則與型別自律,監控 / 除錯 / 遷移面收斂為一套——對位 `messaging-transport-proposal.md` 選 Azure SB 時「單一失敗模型」的同一精神。

**宣告層零衝擊;runtime churn 收斂。** `contract-rpc` / `contract-event` 宣告層不動,不新造 whole-bus abstraction。codec 落 `service-kit`:client / publish 端有現成 seam(`RpcClient.call` / `createEventPublisher`);server / consume 端今天**沒有** service-kit seam(controller 直掛 `@MessagePattern`),須由 service-kit 提供 NestJS serializer / deserializer、在各 app 的 microservice bootstrap 接線——屬 design scope、非 drop-in。churn 收斂在 `service-kit` + bootstrap,不散入 controller / 業務碼。

**不做的後果。** event 端已把「codec 存在」當既成假設(`messaging-transport-proposal.md` 的後續 feature 依賴它);RPC 端新契約仍被迫沿用手動降階;兩條 channel 序列化策略各行其是,日後遷往中性格式時成本翻倍且難對齊。

## Trade-offs

**Pros:**

- 既有契約零改、向下相容;打開 codec 與遷移契約解耦,風險與 churn 最小。
- 新契約業務端操作 rich domain object,免手動降階 / 還原。
- 兩 channel 單一 codec 邊界 + 型別自律,日後遷中性格式可機械化。
- 宣告層零衝擊;codec churn 收斂在 `service-kit` 與各 app bootstrap,不散入 controller / 業務碼。

**Cons:**

- interim 投資會被丟棄。SuperJSON codec 終將被 schema-first codec 取代,現在寫的是過渡件。
- 長期維護兩種型別約定。`Serialized<T>`(JSON-safe lane)與 `WireSafe<T>`(rich lane)並存,契約作者須理解何時用哪條;兩 lane 不收斂為終局狀態。
- 顯式守門靠紀律。`WireSafe<T>` 須由作者在每個 define 處手寫,漏寫即無守護,須 review / lint 補。換得的是守門在宣告處一眼可見、零 magic、零 alias 增生。
- 存量 wire 數據遺害。rich lane 的 payload 若落入 SB dead-letter、event log 或下游存檔,是 SuperJSON 格式,遷移時屬額外存量處理,不在「換 codec」範圍內。
- server 側 codec 無現成 seam。client 有 `RpcClient`,但 server controller 直掛 `@MessagePattern`、無 service-kit 攔截點;codec 須新增 serializer / deserializer 並在各 app microservice bootstrap 接線(非純 service-kit 內改)。

**Unknowns:**

- SuperJSON 對大型 payload 的序列化開銷;若有 throughput 敏感 path,須在 plan 階段量測。
- 自願遷移到 rich lane 的契約數與節奏,取決於各 feature 需求,非本提案規劃。

## Rejected alternatives

- **大規模移除 `Serialized<T>`、強制遷移(v1.2 路線)**:不選。churn 大、風險高,且 `Serialized<T>` 在 REST / legacy 仍有用;沒有理由為了 rich type 把既有契約一次翻掉。本提案改為並存 + opt-in。
- **現在就寫 schema-first codec(protobuf / Avro)**:不選為現階段。終局方向,但逐 contract 寫 schema 是大投資;在跨語言 / 異構 consumer 尚未成硬需求前,先寫一份會被丟棄的 runtime schema 不划算(對位 `messaging-transport-proposal.md` 同一判斷)。
- **RPC 與 event 各自獨立序列化方案**:不選。違背統一 codec 初衷;兩套 codec、兩套失敗模型,監控與遷移雙倍成本。
- **envelope 全包 SuperJSON(不分層)**:不選。broker routing / DLQ / tracing 須能不過 codec 讀 envelope;全包使 metadata 對 infra 不透明,且遷 schema-first 時 envelope 無法獨立中立化。
- **擴大到 rest-rpc / API gateway / public REST / legacy client / queue**:不選(沿用 v1.2 邊界)。rest-rpc 走 `contract-rest`、不共用 `contract-rpc` 的 `ResultDto` / DTO,排除它不在共用邊界製造不一致;那些是對外 / legacy 的 JSON-safe 邊界,納入會污染 public contract 與相容性。
- **不做任何事**:不選。event 端已把 codec 當既成假設;RPC 新契約持續被迫手動降階;方向不定則兩 channel 各自臨時拼裝、不一致。

## Scope of impact

本提案 accept 後觸發以下下游(下游可變,此處為預期 ripple):

- **聯合提案,event-layer 決策歸本提案**:本提案與 `messaging-transport-proposal.md` 為聯合提案、聯合審議——該案定 event transport(SB)與 envelope 結構,本提案定 wire codec 機制與「SuperJSON 正式納入 event 層」。兩案引用已於撰寫時對齊(該案 open point 同步指向本提案)。
- **Supersede v1.2 proposal**:`proposal-rpc-superjson.md` 由本提案取代,保留原位作為歷史記錄。
- **`guide/contract-structure.md` 準則更新**(獨立於 design,作為 design frozen input):現行「RPC payload 必須 JSON-safe、`Date` / `bigint` 不直接出現在 wire DTO」(L106-108)與對應的 event 規則(L174-175)改為——typed transport(RPC + event)契約**可經 `service-kit` codec 宣告 rich `bigint` / `Date`(rich lane,`WireSafe<T>` 守門);`Serialized<T>`(JSON-safe lane)仍有效並存**;public REST / legacy / queue 另依各自 guide,不受此變更影響。應早於或並行於 design,不延至 plan 尾。
- **service-kit wire codec design**:RPC + event 共用的 codec 邊界、envelope / payload 分層落點、`WireSafe<T>` 守門型別、codec 機制細節(版本化等;因原子部署,**無**混版 marker / fallback 需求);含 `contract-rpc` / `contract-event` 整合(rich lane 契約宣告形狀、`ResultDto.data` 子樹邊界)。codec 以 **NestJS custom serializer / deserializer** 落地——NestJS 自帶 packet 雙層 envelope,codec 在其 `data` / `response` 子層動、非 raw bytes。
- **server seam + 各 app bootstrap 接線**(codebase survey 發現,非預期內的 drop-in):client 走 `RpcClient` 有現成 seam,但 server controller 直掛 `@MessagePattern`、**無 service-kit 攔截點**;codec server 側須新增 serializer / deserializer,並在各 app 的 `createMicroservice(...)` bootstrap 接線。event consume 端同理。此為 design 與 plan 的明確工項,勿當 service-kit 內部小改估算。
- **event codec 落點 = SB runtime design**:event 端 codec 併入 `messaging-transport-proposal.md` 下游的 Azure SB event runtime design,**不**插入現行 NestJS event transport(該 transport 將被 SB 取代,插了白做)。
- **遷移單元 = contract + server handler + 所有 caller 同動**:rich lane 化一個 method,須同時拔掉 server mapper 的 `BigInt()` / `new Date()` 與 caller 端還原邏輯(`BigInt(dto.x)` 等);非「只改契約」。屬各 feature opt-in,本提案不規劃。
- **下游 plan**(design 之後,不在本提案展開):codec 接線、`WireSafe<T>` 落地、targeted tests;各契約遷往 rich lane 屬各自 feature 的 opt-in,非本提案規劃。
- 後續處置追蹤進 per-version README 的 Spec persistence 表(`idea/v1.3/README.md`)。

**邊界澄清**:可複用的 wire codec 機制落在 `service-kit`(`rpc` + `event` runtime)。明確不觸及:`rest-rpc`、API gateway `RestRpcClient`、legacy microservice client、queue payload、public REST DTO——這些維持各自的 JSON-safe 邊界。

## Decision request

請求 accept 此 wire codec 方向:

- 為 RPC + event 兩條 typed transport channel 開 SuperJSON wire codec 新路,落在 `service-kit` runtime。
- 向下相容、零強制遷移:`Serialized<T>`(JSON-safe lane)永久續用,`WireSafe<T>`(rich lane)opt-in,兩者並存。
- 分層:envelope / `code` 維持 plain 中立,payload / `data` 子樹為 codec 空間。
- 定位:interim,終局 schema-first 中性格式;rich lane 型別自律(准 `bigint` / `Date`,禁 `Map` / `Set`)。
- 範圍排除 rest-rpc / REST / legacy / queue,及調用 ergonomics(`ResultDto` 收斂 / 錯誤處理,屬另案);本提案擁有兩 channel 的 codec 與分層決策。

accept 後啟動 Scope of impact 所列下游。

## Decision

> 審議當天填寫:勾選下列項目、補上判決與日期。若 accept 條件與上方論證有出入,在「理由 / 偏離」記一筆。

- **判決**:[ ] Accepted　[ ] Rejected　[ ] Superseded
- **日期**:`____`
- **決策者**:`____`(預設 Tim)
- **參與審議者**(如有):`____`

**主張逐段確認**(一併 accept,或標出保留):

- [ ] 為 RPC + event 開 SuperJSON wire codec 新路,落 `service-kit` runtime
- [ ] 向下相容:`Serialized<T>` 永久續用,`WireSafe<T>` rich lane opt-in,並存
- [ ] 分層:envelope / `code` plain,payload / `data` 子樹為 codec 空間
- [ ] 定位 interim,終局 schema-first;rich lane 型別自律(准 `bigint` / `Date`,禁 `Map` / `Set`)
- [ ] 範圍排除 rest-rpc / REST / legacy / queue 及調用 ergonomics(另案);codec 與分層決策歸本提案

**理由 / 偏離**(僅在與原論證有出入時填寫,否則留空):

- `____`

**後續處置**(accept 後啟動):

- [ ] 更新 `guide/contract-structure.md`:typed transport 可經 codec 用 rich type、`Serialized<T>` lane 仍有效並存(獨立於 design,作為 design frozen input)
- [x] 同步 `messaging-transport-proposal.md` event-layer open point 的引用（已於聯合提案撰寫時對齊）
- [x] 標記 `proposal-rpc-superjson.md` 為 superseded（已於撰寫時標註；direction 之審議在本文進行）
- [ ] 啟動 service-kit wire codec design(含 `WireSafe<T>` 守門、contract-rpc / contract-event 整合;先 survey `service-kit/rpc`、`service-kit/event` 現狀)
- [ ] 後續處置追蹤進 per-version README 的 Spec persistence 表(`idea/v1.3/README.md`)

**文件 hygiene**(accept 後):

- [ ] 移除 frontmatter 的 `status: draft`(文件審議完成、穩定)

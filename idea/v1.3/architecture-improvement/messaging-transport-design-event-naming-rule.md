---
status: draft
updated_at: 2026-06-27
updated_by: Claude
---

# Event Messaging Naming Rule — Draft

> **定位**:本文是 [messaging-transport-proposal.md](messaging-transport-proposal.md) 觸發的「service-kit SB event runtime design」之 **design-tier extension doc**,亦即 proposal「guide 更新」ripple 的候選內容 —— 一份 Azure Service Bus 版的 event messaging 命名規範,供 design 收編、日後 distill 落入 `<codebase>/guide/`。依 leaf-before-parent,本葉先於 design 主檔(`messaging-transport-design.md`)產出。
>
> **狀態**:`status: draft`(maturity,非 tier)。最終定稿須等 design 階段 survey `contract-event` / `service-kit` 現狀做衝突校驗(見文末 Open dependencies);本文允許 design 階段微調。
>
> **範疇**:只管 **SB event-broadcast 平面**(Topic / Subscription)的命名。service-private job(BullMQ / 未來 SB queue)與 RPC 不在此文。
>
> **參考姿態**:[Kafka reference](messaging-transport-reference-event-topic-naming-conventions.md) 是擇優採納的原料,不是移植藍本(逐條對照見文末)。

## Frozen inputs(來自 proposal,本文在其上立規,不修改)

- broadcast 走 Azure Service Bus Topic / Subscription。
- Subscription = consumer group;同一 service 的 scaling instance 以 group 為單位只收一次。
- Subscription 命名綁 service 身分,不綁 instance 身分。
- event 端預設不開 session(consumer 靠 idempotency + version-stamping order-tolerant)。
- event identity = `namespace + id + event type`。
- envelope `{ id, occurredAt, payload }` 維持中性;SuperJSON 僅作用 `payload` 子樹。

---

## Topic 命名

### 骨架

```
<namespace>.<resource>.<predicate>
```

- 三段**恆在**:即使 `resource` 與 `namespace` 同名,也照列、不 collapse、不 dedupe。
  - **理由**:搜索歷史 event 紀錄時,制式統一比省一段更重要。
- 全小寫 ASCII。
- 範例:`fund` + `withdrawal-intent` + `completed` → **`fund.withdrawal-intent.completed`**。

### `.` 與 `-` 的分工(結構不變式)

- **`.` = identity 欄位分隔符**,結構性,恆切出**剛好三個欄位**:`namespace` / `resource` / `predicate`。
- **`-` = 段內字詞分隔符**(kebab-case),不產生新欄位。
- 故 `fund.withdrawal-intent.partially-settled` **永遠是三欄位**:`resource = withdrawal-intent`、`predicate = partially-settled`。
- 此不變式保證 topic 名永遠可機械解析回 identity 三元組。

### 粒度:一 event type 一 topic

- 每個 `namespace.resource.predicate` 自成一個 topic(topic-per-event)。
- **不**採「一 namespace 一 topic + subscription filter」的合併模型 —— 那會把 identity 拆進訊息屬性,破壞「整串名字即可檢索」。
- 配額:本專案 domain event 量級為數十,遠低於 SB topic 數上限,topic-per-event 安全(實際 tier / 配額屬 plan-time verify)。

### 分層備註(避免 drift)

topic 規則本身只做一件事:機械投影 `namespace + '.' + eventType`,其中 `eventType = resource.predicate` 的文法由 [Event type](#event-type-命名) 節擁有。「resource 恆在」這條約束落在 event-type 文法上,topic 規則只把它表面化成三段點分。`contract-event` 仍是事件名的唯一權威。

---

## Namespace / Bounded context 劃分

### 術語消歧（重要）

> 本文 topic 名裡的 `namespace` 段是**邏輯 bounded-context 標籤**;Azure SB 自身也有一個叫 **Namespace** 的資源(`*.servicebus.windows.net` 訊息容器)。**兩者不是同一物。**

且兩者正交:命名層的 `namespace` 段**恆在 topic 字串裡**,與「實體上開幾個 Azure SB Namespace 資源」無關。物理拓樸(共用一個 SB Namespace 或每 context 一個)是 plan 的事;日後物理整併 / 拆分不動命名。

### 四條劃分原則

1. **綁領域,不綁服務**:`namespace` = 擁有該 resource 的 bounded context(`fund`),不是跑它的 deployable service / team / app。服務改名、拆分、合併都不該動 topic 名。
2. **顆粒度 = 一致性 / 擁有權邊界**:一個 namespace 對一個 bounded context,擁有一組 aggregate(`fund` 擁有 `deposit`、`withdrawal-intent`、`transfer-intent` 等)。比 resource 粗、比「整間公司」細。**不與 microservice 1:1**。
3. **封閉字彙,受治理**:`namespace` 值只能取自已登記集合(權威落在 `contract-event` 宣告),不可逐 event 臨時發明。
4. **Emitter-owned**:某 namespace 下的 event **只由擁有它的 context 發送**;別的 context 可跨 namespace 訂閱,但**絕不往別人的 namespace 發 event**。namespace 因此是可靠的來源 / 擁有權訊號。
   - 推論(反模式):跨 context 的「回報」需求由 RPC 或「客體發自身 event + 主體訂閱」承接,**不以 callback event 寫入主體 namespace**(詳見 [反模式](#反模式)）。

---

## Event type 命名

`event type = resource.predicate`(subject-first)。

### 形式規則(本 guide 擁有）

| 維度           | 規則                                                          |
| -------------- | ------------------------------------------------------------- |
| 順序           | subject-first:`resource` 在前、`predicate` 在後               |
| predicate 時態 | **過去式 = 已發生的事實**(`completed` / `created` / `failed`) |
| 大小寫         | 全小寫 ASCII,段內 kebab-case                                  |
| resource 形式  | 單數名詞(aggregate 或衍生抽象資源),kebab-case                 |

### 字彙規則(領域擁有,本 guide 不立法)

- **predicate 的字彙來自 resource 自身的 DDD 狀態更迭與動作**(`settled`、`reversed`、`refunded`），由領域 ubiquitous language 決定。本 guide **不編造**一套中央事件語言。
- **治理**:有效 `(resource, predicate)` 組合枚舉登記在 `contract-event`(治理「有沒有登記」),內容來源是領域模型(領域管「叫什麼」)。
- **通用轉換的拼寫一致性**(非強制、非本 guide 立法):當 resource 的 lifecycle 含通用的「誕生 / 變更」轉換時,慣用拼寫為 `created` / `updated`(避免 `made` / `added` / `changed` 等同義詞分歧)。主詞仍是領域,本 guide 只做 consistency 建議。
- `updated` 是 anemic 的:**有具體領域轉換時優先用具體的**(`limit-raised`、`address-corrected`),`updated` 僅作「真的只是通用變更」的退路。

---

## Subscription 命名

Subscription 結構上永遠屬於某個 topic（路徑 `topic/subscriptions/<name>`）。命名綁**消費端身分**(與 topic 綁 producer 身分相對:各實體用自己的擁有者命名）。

### 標準（durable 消費者）

```
srv-<consuming-service>
```

- `<consuming-service>` = 消費端的**穩定邏輯 role**,kebab-case、小寫。跨 deploy / scaling / instance / region 不變。
- **禁**綁會變的身分:instance、pod、replica、team、owner。
- **身分單位 = 一個 deployable consumer(microservice)**,因為那是 competing-consumer 去重邊界(同 deployable 的多副本須共用一條 subscription 才能「以 group 為單位只收一次」)。
- **數量**:per-topic。一個 service 訂 N 個 topic = N 條**同名** `srv-<service>` subscription,各掛在各自 topic 下。
- **基底無關**:serverless function、durable CLI batch、microservice,只要是長駐消費者,一律 `srv-<role>`。`srv-` 標的是「**長駐、受認可**」這個 lifecycle class,**不是** compute kind。
- 跨多個 namespace 的 consumer:只增加它的 subscription 數量,不切碎它的身分(namespace 是 producer 的事,不滲進 consumer 命名）。

### 臨時（ephemeral 消費者）

```
tmp-<owner>-<purpose>
```

- ad-hoc CLI、debug、臨時 tail 事件用。
- 必設 **auto-delete-on-idle**,用完即走。
- **不得**占用標準 `srv-` subscription（原因見 [反模式](#反模式)：durable subscription 會在無 receiver 時累積 backlog 直到 DLQ 爆）。

### 前綴體系 self-classifying

任何 subscription 不是 `srv-` 就是 `tmp-`;**兩者皆非 = off-convention,可被 lint / 清理**。掃前綴即可分出「長駐 / 臨時 / 違規」三堆。

### 約束

- subscription 名上限 **50 字元**,允許 letters / numbers / `.` / `-` / `_` / `/`。role 名勿過長。
- subscription 由 consumer 端 provision(topic 由 producer 端);屬 service-local 配置,不外溢進 `service-kit` 共用契約。

---

## 版本化策略

1. **Topic 名 version-free**:不帶任何 `vN`(理由:topic-per-event 下版本維度會乘上一層、撞配額;SB topic 不可改名;版本化 topic 每次 bump 都逼 consumer re-subscribe)。
2. **相容 = additive 演進,且須 transitive**:
   - 鄰近版本恆向後相容(expand / contract 隔代:加新的 additive,隔一代確認無人用舊的,才移除)。
   - **forward-compat 窗口對齊「最老可重消費事件」**,而非僅相鄰版本 —— replay / DLQ 重處理 / 存量存檔可能比一代更老,須 transitive-additive(現行 schema 容忍全歷史 payload)。
3. **破壞性變更 = 鑄新 event type,不是掛 `vN` 後綴**。新 resource / predicate 命名新的領域事實,過渡期雙跑、退役舊的。此機制與版本化**正交**:領域事實變了才用。
4. **wire 上不帶 schema 版本欄位**(Path A）:
   - live routing 不需要它（type 已分流、additive 由 consumer 容忍）。
   - 對齊未來 protobuf 終局（結構性相容、訊息不帶 schema 版本),不製造 SuperJSON 期會被丟棄的中間製品。
   - envelope 維持凍結 `{ id, occurredAt, payload }`,**不新增欄位**。
5. **治理 = `contract-event` review**(SB 無內建 registry 的替代):相容性在宣告審查時把關,非 runtime registry。

> **Named deferred — upcasting**(trigger:出現無法以 additive 表達、又非「新事實」的 payload reshape 需求):屆時才引入 event-sourcing 式 upcaster 鏈,並就地帶版本;不現在預建。

---

## 反模式

> 本節把規則背後的觀念釘住,防後人誤用。

### Callback 不寫入主體 namespace

「客體 microservice 往主體 topic 發事件回報」是 Kafka 期的 callback 模式,本專案**棄用**:

- 違反 emitter-owned(B 往 A 的 namespace 寫 = 破壞 namespace 的 provenance）。
- 把 request-response 誤塞進 Event 平面(本專案三語意切分:回報屬 RPC 或 Queue,不屬 Event）。
- **fan-in(多客體 → 一主體)是 Queue 的形狀,不是 Topic（fan-out 一 → 多）。**

主體要的「回報、更新 flag」用以下承接,皆不需 callback topic:

| 需求                           | 承接                                                                                    |
| ------------------------------ | --------------------------------------------------------------------------------------- |
| 同步要答案                     | RPC                                                                                     |
| 客體完成某領域事實、主體想知道 | 客體發**自身 namespace 的 event**,主體訂閱、用 envelope `id` / payload correlation 對上 |
| 定向回報(主體更新自身 flag)    | principal-owned **reply queue** + `ReplyTo` / `CorrelationId`(fan-in,Queue 平面)        |
| 發起的工作要「補償到成功」     | **DB intent record + backfill**(保證 anchor 在 DB,不在 broker）                         |

### Topic-per-event,不用 bundle + filter

合併 topic + subscription filter 會把 identity 拆進訊息屬性、養一套 filter rule 維護,破壞「整串名字即可檢索」。topic-per-event 讓每 topic 契約單一、DLQ / metrics 隔離。

### Durable vs ephemeral subscription

SB subscription 是 **durable** 的:不管有沒有 receiver 連著,訊息照堆。CLI / 臨時工具若開標準 `srv-` subscription,人不在跑時會默默累積 backlog 直到 DLQ 爆。臨時消費**必須**走 `tmp-*` + auto-delete-on-idle。

### 三種「version」別混

本專案 wire 上**刻意不帶 schema 版本欄位**。與之相關但**不同層**的版本概念:

| 概念             | 是什麼                               | 住哪                          |
| ---------------- | ------------------------------------ | ----------------------------- |
| schema 版本      | payload 契約版本                     | **刻意不帶**(見版本化 Path A) |
| version-stamp    | 實體 / 事件版本戳,order-tolerance 用 | **payload** 內,領域欄位       |
| `SequenceNumber` | broker 指派的投遞序號                | SB system property            |

若日後要加 schema 版本欄位,須重開版本化決策(Path A → B),不可隨手加。

---

## Kafka reference 逐條對照

| Kafka 條目                                                 | 裁決                                                                                                        | 分類       |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ---------- |
| `.` 分隔、全小寫                                           | 全採                                                                                                        | 直接可用   |
| topic 結構 `<scope>.<message type>.<domain>.<domain data>` | 砍 `scope` / `message type`(專屬 event bus 上退化成常數);`domain` → `resource`、`domain data` → `predicate` | 需翻譯     |
| `scope`(public/private)                                    | event 平面本質全 public broadcast;private 在此不存在                                                        | 不適用該棄 |
| `message type`(queuing/state/…)                            | 語意由 channel 承載,不重編進名字                                                                            | 不適用該棄 |
| 「prefer domain name,not application name」                | namespace 綁領域、不綁 app/service                                                                          | 直接可用   |
| 「禁綁會變欄位(producer/consumer/team/owner)」             | namespace / subscription 皆守                                                                               | 直接可用   |
| 「disable auto-create,manual only」                        | 翻成 namespace / event type 取自 `contract-event` 註冊集                                                    | 需翻譯     |
| 「version number 致 topic 暴量」                           | 版本不進 topic 名(topic-per-event 下更嚴重）                                                                | 直接可用   |
| schema registry 管版本                                     | SB 無 registry → 以 `contract-event` review + transitive-additive 替代                                      | 無對應要補 |
| consumer-group 命名                                        | Kafka rule 未寫 → 本專案新增 `srv-<role>`                                                                   | 無對應要補 |
| partition key / SessionId                                  | event 端不開 session,無 key                                                                                 | 不適用該棄 |
| domain 範例(asset-iot/tag/site)                            | 前專案 domain,僅借維度                                                                                      | 不適用該棄 |

---

## Open dependencies / design-survey 校驗點

- **webhook eventKey vs 內部 topic**:既有 webhook eventKey 是兩段 `withdrawal.completed`(對外目錄層);內部 SB topic 是三段 `fund.withdrawal-intent.completed`。兩者是否同物、要否對齊,待 `contract-event` survey 校驗。
- **namespace 清單錨定**:具體 namespace 列舉錨定既有 codebase domain 模組邊界,清單留 design survey 定。
- **`-service` 後綴**:`srv-` 後的 identity token 以 codebase canonical service identifier 為準;本 guide 傾向裸 role(去冗餘 `-service`),codebase 慣例若帶則以 codebase 為準。
- **upcasting deferred trigger**:見版本化節。
- **最終衝突校驗**:design 階段 survey `contract-event` / `service-kit`(SB adapter precedent、event runtime placement、consumer-group 落點)後定稿。

---

## 回灌 parent proposal 的反饋

本 subtask 浮現、應回報給 [messaging-transport-proposal.md](messaging-transport-proposal.md)（+ 聯合提案 superjson-wire-codec）調整者:

1. **【envelope 結構 — 結論:不動】** 一度考慮為 handler routing / replay 在 envelope 加 schema 版本欄位;經 Google / Cloudflare / Confluent / event-sourcing 參考類別校驗後**否決**(live routing 不需要、protobuf 終局不帶、additive transitive 已足)。envelope 維持凍結 `{ id, occurredAt, payload }`。**此項無需 parent 改動,僅記錄結論。**
2. **【版本化定調】** 建議回填 proposal「Open points」:topic version-free + transitive-additive(隔代 expand/contract）+ 破壞性 = 新 event type（與版本正交)+ `contract-event` review 當 registry 替代 + upcasting 為 named deferred。
3. **【replay 窗口 caveat】** proposal 的 deferred「存量 SuperJSON 數據」與版本化交互:forward-compat 窗口須對齊「最老可重消費事件」(transitive),非僅相鄰版本。建議在 proposal 標明此依存。
4. **【event identity 細化】** proposal「event identity = namespace + id + event type」可細化:topic = `namespace.resource.predicate`,即 `event type` 內部 = `resource.predicate`。建議回填使 identity 描述與命名層一致。
5. **【callback / 反模式（概念回灌）】** fan-in 回報走 reply queue + correlation,不寫主體 namespace;emitter-owned 維持。可補進 proposal 或留本 guide。

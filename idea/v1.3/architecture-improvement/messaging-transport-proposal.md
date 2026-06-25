---
status: draft
updated_at: 2026-06-26
updated_by: Claude
---

# Messaging Transport — Proposal

> **聯合提案**:與 [superjson-wire-codec-proposal.md](superjson-wire-codec-proposal.md) 聯合審議——本提案定 event transport(Azure SB)與 envelope 結構,該案定 wire codec(RPC + event 兩 channel 的序列化與分層)。兩案引用互相對齊。

## Context

系統目前缺少一個可靠的 cross-service event broadcast 機制。

NestJS 內建的 event 能力都不勝任這個角色:

- `@nestjs/event-emitter` 是純 in-process、單機記憶體的 EventEmitter,無跨程序、無持久化,連跨 instance 都做不到。
- NestJS microservices 的 Redis transport 底層是 Redis Pub/Sub,語意是 fire-and-forget / at-most-once:無 subscriber 在線即丟棄、不重送、無 consumer group。它無法滿足「同一 service 的 scaling instance 以 group 為單位只收一次」,也撐不起 `deposit completed` 這類不可遺失的領域事件。

Service-private job 端則已有既成事實:fund-case 既有的 instruction / settlement pipeline 跑在 BullMQ(Redis)上,具備可用的 retry / backoff / delayed job。

兩個需求性質不同,對應 codebase 既有的三語意切分(`Queue = Internal Job`、`Event = Asynchronous Fact`、`REST/RPC = Synchronous Request-Response`,見 `guide/rules/application-adapter.md`):

- **領域事件廣播**(Event):跨 service fan-out,且同一 consuming service 的多個 instance 只承接一次。
- **service-private job**(Queue):服務專用的執行機制,某些情況下需要在 key 組合下維持 FIFO。

現在處理的理由:webhook 等後續 feature 已假設存在一個 domain event notification 的上游,但該上游機制尚未確立。在更多 feature 把「有可靠 broadcast」當成既成假設之前,先把 transport 方向定下來。

## Proposal

採用 Azure Service Bus 承載 cross-service event broadcast;service-private job 暫留 BullMQ 作為 interim,完成保證 anchor 在 DB intent record + backfill,而非任何 broker 的 redelivery 語意。

## Motivation

**broadcast 是 Azure SB 的剛需,無既有替代。** SB 的 Topic + Subscription 原生提供本提案要的兩層語意:Topic 對多個 Subscription fan-out(不同 service 各收一份),單一 Subscription 內的多個 receiver 是 competing consumers(同一事件只交給其中一個 instance)。因此「Subscription = consumer group、instance 數 = group 內平行度」直接成立,且具備 at-least-once 投遞與 dead-letter queue。這是 Redis Pub/Sub 給不了的可靠度。

**job 端的完成保證已不依賴 broker,所以 broker 對 job 降格為 trigger。** 「失敗補償到成功」這個保證,不押在 BullMQ 或 SB 任一邊的 redelivery 上,而是 anchor 在 DB 的 job/intent 狀態機(`PENDING → DONE`)加上 catch-up backfill——對位 codebase 既有的 webhook pending-delivery recovery 與「queue catch-up over cron」原則。broker(BullMQ 或 SB)只是 happy-path 的執行觸發;即使某則訊息遺失,backfill loop 仍保證最終完成。保證來自 DB row,不是 broker。

**SB 的引入不衝擊 contract 宣告層。** `contract-event` 既有 convention 已把宣告與 runtime 分離(`contract-event` 只放宣告;event runtime helper 與 DI 組裝留在 `service-kit`)。SB adapter 是往 `service-kit` 既有的 runtime 空位填,不新造 whole-bus abstraction、不動 `contract-event` 的宣告層。event identity(`namespace + id + event type`)可直接推導 SB 的 topic 命名與 subscription routing。

## Trade-offs

**Pros:**

- 單一 broadcast 失敗模型。所有 cross-service event 共用一套 at-least-once + DLQ 語意,監控與除錯面收斂。
- consumer-group 語意原生。「同 service scaling instance 收一次」由 Subscription 直接給,無須自建 dedup。
- contract 宣告層零衝擊。SB runtime 落在 `service-kit` 既有空位,churn 收斂在 runtime 層。
- 既有 BullMQ job 一行不動。本提案不要求立即遷移 job 端,interim 維持現狀,churn 最小。

**Cons:**

- SB 無內建 exponential backoff。BullMQ 開箱即用的 `attempts` / backoff / delayed job,在 SB 上需以 scheduled re-enqueue 自行構造。
- long handling 須處理 lock renewal。SB 是 per-message PeekLock(LockDuration 預設約 30 秒),超時的長 job 須週期 RenewLock,否則處理中被誤判為失效而 redeliver。
- 多一個 infra 依賴。SB 是新 broker,需 provisioning、命名規範、ops 參數與部署設定(屬下游 plan)。
- 既有 BullMQ job 終須遷移。interim 不是終局;未來 job 端遷往可靠 queue 服務時,既有 fund-case worker 須改寫。
- 維運期兩個 broker 並存。interim 期間 broadcast(SB)與 job(BullMQ)是兩套機制、兩種失敗模型,須各自理解。

**Unknowns:**

- SB tier 的 feature / quota 邊界以官方計費頁為準(各家雲端細項常改版),落地前須實查。本提案的架構判斷是「broadcast fan-out 需 Topic,故最低 Standard;Basic 無 Topic、出局」,但具體 tier 選定屬下游 plan。
- event 量級與 subscription 數對 throughput / 延遲的影響,須在 plan 階段以實際負載評估是否需 Premium。

## Rejected alternatives

- **Azure SB 立即接管 job(SB-for-both)**:不選。churn 高(既有 BullMQ job 須一次性遷走),且「補償到成功」的保證既已 anchor 在 DB + backfill,SB 在 job 端的 durability 大部分是冗餘的。job 端改用 SB 應在 persistence-based retry practice 成熟後,作為具名 trigger 下的後續遷移,而非本提案範圍。
- **NestJS Redis Pub/Sub 當 broadcast**:不選。at-most-once、無持久化、無 consumer group,無法滿足「不遺失 + 同 group 收一次」。(Redis Streams + consumer group 在語意上接近,但那不是 NestJS 內建 transport 提供的,且須自建;在已選 SB 的前提下無必要。)
- **混用 BullMQ 長期當 job + SB 當 event(永久 hybrid)**:不選為終局。job 端的「補償到成功」會橫跨「BullMQ 重試 → 放棄 → backfill 接手」三段,且 job 端 BullMQ redelivery 與 event 端 SB redelivery 是兩套冪等與 DLQ 語意要對齊。作為 interim 可接受,作為終局則徒增一套要永久維護的失敗模型。
- **不做任何事**:不選。後續 feature 已假設存在可靠的 domain event broadcast 上游;不確立 transport,這些 feature 無法落地,或被迫各自臨時拼裝,造成不一致。

## Scope of impact

本提案 accept 後觸發以下下游(下游可變,此處為預期 ripple):

- **`guide/rules/application-adapter.md` 準則更新**(獨立於 design):補上三語意的 broker 對應——broadcast → Azure SB Topic/Subscription;service-private job → BullMQ(interim);完成保證 → DB intent + backfill。此準則為 design 的 frozen input,應早於或並行於 design 落地,**不延至 plan 尾**。(注意:此與 closing 階段的 distillation 不同;distillation 是實作後浮現的慣例回流,屬 plan/closing 尾段。)
- **service-kit SB event runtime design**:topic / subscription / envelope mapping、consumer-group 語意、idempotency 要求、failure model(DLQ、PeekLock/Complete、RenewLock)。此 design 須含 `contract-event` 整合(topic 命名、envelope ↔ SB message 映射、event serialization codec 落點)。建議由 CLI 端 agent 先 survey `contract-event` / `service-kit` 現狀(SB adapter precedent、event runtime placement、consumer-group 落點)再框。
- **下游 plan**(design 之後,不在本提案展開):service-kit 建置 + service-local 的 topic / subscription 配置與 SB ops 參數(tier、LockDuration、MaxDeliveryCount 等)。

**邊界澄清**:可複用的 SB event runtime 機制落在 `service-kit`;各 service 的 topic / subscription 配置屬 service-local(對位 application-adapter「queue / event 的 service-private 決策不外溢成共用契約」)。design 階段不應把命名 / 訂閱配置誤集中進 service-kit。

## Open points

以下留給 design / plan 解決,或為具名 deferred:

- **Event serialization**:envelope(`{ id, occurredAt, payload }`)維持固定 plain 結構;SuperJSON serialize **僅作用於 `payload` 子樹**,由 `service-kit` event codec 在 payload 邊界處理。envelope 層因此始終語言中立(`id` / `occurredAt` 為 plain),rich 型別(`bigint` / `Date`)僅出現於 payload、由 codec 還原為 rich domain object 交給業務端;業務端對 codec 無感。此切分把「envelope 必須中性」從隱性約定升級為 codec 層顯式契約,並提早分清 envelope / payload 邊界,利於日後遷移。codec 僅需知道「`payload` field 整個交給 SuperJSON」,顆粒度為單一 field、**無逐 event schema 成本**(與 schema-directed codec 不同)。Payload 型態自律於中性 schema(protobuf)可直接表達的型別集(`bigint` / `Date` / `numeric` / `string` / `bool` / plain nested object / array),**不使用** `Map` / `Set` 等 SuperJSON 專屬結構——此自律是日後遷移可機械化的對價。
  - 採 SuperJSON(而非現在就寫 schema-directed codec)的理由:終局指向 schema-first 中性格式,屆時逐 event 的 schema 本來就要寫;先寫一份 runtime schema 是會被丟棄的中間投資,故現階段以 SuperJSON 自反解撐過、跳過中間態。
- **SuperJSON 與 contract-event 的關係**:SuperJSON 作為 typed transport wire codec 的正式定調,由聯合提案 [`superjson-wire-codec-proposal.md`](superjson-wire-codec-proposal.md) 擁有(取代 v1.2 草稿 `proposal-rpc-superjson.md`)。該案統一定調 RPC + event 兩 channel 的 codec 與分層(envelope plain / `payload` 子樹 SuperJSON),event 層的 SuperJSON 採用即在該案裁決。兩案**聯合審議**:本提案定 event transport(SB)與上述 envelope 結構,該案定 wire codec 機制與型別自律;本段「`payload` 子樹 SuperJSON」即與該案一致的 event 側落點。
- **Deferred — 中性 wire 格式遷移**(具名 trigger:event 出現非 TS / 異構 consumer,或 event 契約的跨語言 / 強兼容成為硬需求):遷移至 schema-first 中性格式(protobuf / Avro)。屆時翻寫 `service-kit` event codec + 新增 schema(`.proto` 等);業務端因操作 rich domain object 而不受影響。**須額外處理存量 wire 數據**——若 event 曾落入 SB dead-letter、event log 或下游存檔,那些是 SuperJSON 格式,不在「翻寫 codec / mapper」範圍內,屬額外存量處理,日後勿低估為「只是換 codec」。
- **Deferred — job 端遷移**(具名 trigger:persistence-based retry practice 成熟):service-private job 由 BullMQ 遷往可靠 queue 服務(Azure SB)。interim 期間 BullMQ 的 `attempts` 設小(僅吸收 transient 抖動、不擔保完成),擔保留給 DB + backfill;`attempts` 不設過大,以免遮蔽 backfill 破口、磨不出 backfill practice。
- **FIFO / session 落點**:job 端若採 SB,FIFO 由 Queue with Sessions 提供(`SessionId` = causal key)。`requiresSession` 為 provisioning-time 屬性,故為 per-queue 決策:預設建 plain queue,僅具名的 per-key 因果序需求才建 sessioned queue。event 端預設不開 session(consumer 以 idempotency + version-stamping 維持 order-tolerance)。此屬 design / plan,列此備忘。

## Decision request

請求 accept 此 transport 方向:

- broadcast → Azure Service Bus Topic/Subscription
- service-private job → BullMQ(interim,具名 trigger 下後續遷移)
- 完成保證 → DB intent record + backfill(broker 為 trigger,非保證層)

accept 後啟動 Scope of impact 所列下游。

## Decision

> 審議當天填寫:勾選下列項目、補上判決與日期。若 accept 條件與上方論證有出入,在「理由 / 偏離」記一筆。

- **判決**:[ ] Accepted　[ ] Rejected　[ ] Superseded
- **日期**:`____`
- **決策者**:`____`(預設 Tim)
- **參與審議者**(如有):`____`

**主張逐段確認**(三段一併 accept,或標出保留):

- [ ] broadcast → Azure Service Bus Topic/Subscription
- [ ] service-private job → BullMQ(interim,具名 trigger 下後續遷移)
- [ ] 完成保證 → DB intent record + backfill(broker 為 trigger,非保證層)

**理由 / 偏離**(僅在與原論證有出入時填寫,否則留空):

- `____`

**後續處置**(accept 後啟動):

- [ ] 更新 `guide/rules/application-adapter.md`:補三語意 broker 對應(獨立於 design,作為 design frozen input)
- [ ] 啟動 service-kit SB event runtime design(含 contract-event 整合;CLI 端先 survey `contract-event` / `service-kit` 現狀)
- [ ] 後續處置追蹤進 per-version README 的 Spec persistence 表(`idea/v1.3/README.md`)

**文件 hygiene**(accept 後):

- [ ] 移除 frontmatter 的 `status: draft`(文件審議完成、穩定)

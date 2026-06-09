---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Stage 1 Blueprint

## Goal

Stage 1 建立 webhook subscription management 與 webhook event type catalog 的第一版基礎。完成後商戶後台可以管理 webhook endpoint，後端可用 event catalog 驗證訂閱事件。

Stage 1 是原 Phase 1 的細拆版本。原本同時包含 API route、DB schema、seed、service implementation 與 merchant scoping，粒度過粗；本文件將它拆成多個 phase，讓 plan 與實作可以逐步驗證。

## Scope

In scope：

- `webhook_subscription` persistence。
- `webhook_event_type` seed 與 read endpoint。
- `webhook_subscription_event_type` relation。
- Subscription create / list / get / update / delete REST API。
- Event type keys validation。
- Merchant scoping。

Out of scope：

- Outbox event production。
- Dispatcher / delivery worker。
- Delivery history。
- Signature verification docs。
- Endpoint ownership verification。
- Secret rotation。

## Inputs

- Design REST contract：[`../design/design-rest.md`](../design/design-rest.md)。
- API surface：[`blueprint-api-surface.md`](./blueprint-api-surface.md)。
- Data model：[`blueprint-data-model.md`](./blueprint-data-model.md)。
- Service architecture：[`blueprint-service-architecture.md`](./blueprint-service-architecture.md)。

## Stage Decisions

- Event type catalog 由 migration seed 寫入，不開放 UI CRUD。
- `deposit.blocked` 第一版正式納入 seed 與可訂閱選項。
- Event type read model 採獨立 endpoint：`GET /webhook/merch/event-types`。
- Subscription delete 採 soft delete。
- Subscription 是否可參與後續派送由「未刪除且訂閱目標事件」判斷。
- REST request / response 欄位使用 camelCase；DB 欄位命名由 mapping layer 轉換。

## Phase Breakdown

### Phase 1：Codebase Survey And Implementation Boundary

確認 Stage 1 要落在哪個 codebase、module、controller pattern 與 migration pattern。

工作內容：

- Survey merchant console / backend 既有 REST controller、service、repository、DTO、validation、pagination、error response pattern。
- Survey migration / seed 的既有做法，確認 event type catalog seed 要放在哪裡。
- 確認 merchant identity / merchant scoping 的取得方式。
- 確認 soft delete query convention。
- 確認 route prefix 與 path parameter 命名是否沿用 [`design-rest.md`](../design/design-rest.md)。

輸出：

- Stage 1 plan 可使用的檔案位置、module boundary 與 implementation pattern。
- 若 codebase pattern 與本 blueprint 不一致，plan 需回寫差異與採用理由。

Validation target：

- Plan 作者能指出 controller、service、repository、migration、seed、DTO 應落在哪些既有目錄。

### Phase 2：Persistence Schema And Event Seed

建立 Stage 1 所需資料結構與 event type catalog seed。

工作內容：

- 建立 `webhook_subscription`。
- 建立 `webhook_event_type`。
- 建立 `webhook_subscription_event_type`。
- 寫入第一版 event type seed：
  - `withdrawal.created`
  - `withdrawal.canceled`
  - `withdrawal.failed`
  - `withdrawal.completed`
  - `deposit.created`
  - `deposit.failed`
  - `deposit.completed`
  - `deposit.blocked`
- 建立同 merchant 未刪除 endpoint 不可重複的約束或等效檢查。
- 建立 subscription-event relation 不可重複的約束或等效檢查。

Validation target：

```text
migration applied
  -> event type seed exists
  -> duplicate effective endpoint rejected
  -> duplicate subscription-event relation rejected
  -> soft-deleted subscription does not block endpoint reuse
```

### Phase 3：Event Type Read And Validation Capability

建立前端 checkbox options 與 subscription API 共用的 event type 讀取 / 驗證能力。

工作內容：

- 實作 `GET /webhook/merch/event-types`。
- 回傳 `WebhookEventTypeOptionDto[]` 所需欄位。
- 依 `sortOrder` 排序。
- 建立 event keys validation helper / service capability。
- 確保 subscription create / update 不接受不存在的 event key。

Validation target：

```text
merchant user
  -> GET /webhook/merch/event-types
  -> receives all first-version event keys including deposit.blocked
  -> invalid event key is rejected by subscription validation
```

### Phase 4：Subscription REST API And Merchant Scope

建立商戶可操作的 webhook subscription management API。

工作內容：

- 實作 `GET /webhook/merch/subscriptions`。
- 實作 `POST /webhook/merch/subscriptions`。
- 實作 `GET /webhook/merch/subscriptions/{subscriptionId}`。
- 實作 `PUT /webhook/merch/subscriptions/{subscriptionId}`。
- 實作 `DELETE /webhook/merch/subscriptions/{subscriptionId}`。
- 實作 merchant scoping，商戶只能查詢與修改自己名下 subscription。
- Create / update endpoint URL 時檢查同 merchant 有效 endpoint 唯一性。
- Update event keys 時以 request 事件集合覆蓋目前 subscription-event relation。
- Delete 時採 soft delete，後續 list / get 不回傳已刪除資料。

Validation target：

```text
merchant user
  -> create webhook subscription with endpointUrl + eventKeys
  -> list subscriptions
  -> get subscription detail
  -> update endpointUrl and eventKeys
  -> soft delete subscription
  -> deleted subscription disappears from list
```

驗證重點：

- Merchant A 不能讀寫 Merchant B 的 subscription。
- Event keys 必須來自 `webhook_event_type`。
- Deleted subscription 不出現在 UI list，也不會被後續 dispatcher 使用。

## Sequencing

- Phase 1 先完成，避免 schema、route、module 位置在 plan 期間反覆改動。
- Phase 2 是 Phase 3 / Phase 4 的前提，因 read endpoint 與 subscription validation 都依賴 event type catalog。
- Phase 3 可在 Phase 2 migration plan 穩定後開始。
- Phase 4 可與 Phase 3 後段並行，但 create / update validation 需共用 Phase 3 的 event key validation capability。

## Estimate

| Phase | 估時 | 依據 |
| --- | ---: | --- |
| Phase 1 | 0.5 天 | Survey 既有 controller、migration、merchant scoping 與 soft delete pattern。 |
| Phase 2 | 1 天 | 三張表、event seed、唯一性與 relation 約束。 |
| Phase 3 | 0.75 天 | Event type read endpoint 與 event key validation capability。 |
| Phase 4 | 1.75 天 | Subscription CRUD、relation replacement、merchant scope、soft delete 與 error handling。 |

總估時：4 天。

## Open Points

- Subscription list 是否需要回傳最新 delivery 狀態不在 Stage 1 範圍；若 UI 需要應另開 design extension。

---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Stage 1 Blueprint

## Goal

Stage 1 建立 webhook subscription management 與 code-defined webhook event type catalog 的第一版基礎。完成後商戶後台可以管理 webhook endpoint，後端可用 event catalog 驗證訂閱事件。

Stage 1 是原 Phase 1 的細拆版本。原本同時包含 API route、DB schema、event catalog、service implementation 與 merchant scoping，粒度過粗；本文件將它拆成多個 phase，讓 plan 與實作可以逐步驗證。

## Scope

In scope：

- `webhook_subscription` persistence。
- `webhook_subscription_event_type` relation。
- Code-defined webhook event type catalog 與 read endpoint。
- Api-gateway subscription create / list / get / update / delete REST API。
- Webhook service subscription management RPC。
- Event type keys validation。
- Merchant scoping。
- Merchant console subscription management API 串接。
- Stage 1 backend / frontend verification。

Out of scope：

- Outbox event production。
- Dispatcher / delivery worker。
- Delivery history。
- Signature verification docs。
- Endpoint ownership verification。
- Secret rotation。
- Webhook management UI 的完整視覺重設。

## Inputs

- Design REST contract：[`../design/design-rest.md`](../design/design-rest.md)。
- Error contract：[`../design/design-error-contract.md`](../design/design-error-contract.md)。
- Management surface design：[`../design/design-management-surface.md`](../design/design-management-surface.md)。
- RPC surface：[`../design/design-rpc.md`](../design/design-rpc.md)。
- Data model：[`../design/design-persistence-model.md`](../design/design-persistence-model.md)。
- Service boundary：[`../design/design-service-boundary.md`](../design/design-service-boundary.md)。

## Stage Decisions

- Event type catalog 由 TypeScript 檔案定義，不建 DB table，也不開放 UI CRUD。
- `deposit.blocked` 第一版正式納入 code-defined catalog 與可訂閱選項。
- Event type read model 採獨立 endpoint：`GET /webhook/merch/event-types`。
- Subscription delete 採 soft delete。
- Subscription 是否可參與後續派送由「未刪除且訂閱目標事件」判斷。
- Subscription 可沒有任何 event binding；空 `eventKeys` 代表保留 endpoint 但取消所有事件訂閱。
- `WebhookSubscription` 本體不包含 `eventTypes`；`eventTypes` / `eventTypeCount` 是 management read model 組裝結果。
- 同一 merchant 可建立多筆相同 endpoint URL 的 subscription；第一版不做 merchant + endpoint URL 唯一性限制。
- REST request / response 欄位使用 camelCase；DB 欄位命名由 mapping layer 轉換。
- Merchant console 經由 api-gateway REST route 存取 webhook management；api-gateway 使用 REST RPC proxy 呼叫 webhook service management RPC。
- Webhook service 是 RPC 內部服務，不重複驗證 JWT。商戶身分來源為 RPC 呼叫方傳入的 `identity.merchantId`；merchant scope query 使用此值作為 `merchant_id` 篩選條件。
- Validation、normalization、pagination 與 sorting 依 [`../design/design-rest.md`](../design/design-rest.md)。
- Stable error code 與 merchant scope failure 語意依 [`../design/design-error-contract.md`](../design/design-error-contract.md)。

## Phase Breakdown

### Phase 1：New Service Boundary And Implementation Convention

確認 Stage 1 在新 webhook service 與 merchant console frontend 的 implementation boundary、module 切分與工程慣例。

工作內容：

- 決定 api-gateway REST controller / proxy mapping，以及 webhook service RPC handler、service、repository、DTO、validation、pagination、error response pattern。
- 決定 migration 的做法，並確認 code-defined event type catalog 要放在哪個 TypeScript module。
- 決定 merchant console frontend 的 API client、route、page/form state 與錯誤顯示慣例。
- 確認 webhook service 讀取 RPC 傳入 `identity.merchantId` 的實作慣例，確保與 codebase 其他 RPC 服務一致。
- 確認 soft delete query convention。
- 確認 route prefix 與 path parameter 命名是否沿用 [`design-rest.md`](../design/design-rest.md)。

輸出：

- Stage 1 plan 可使用的 service boundary、module boundary 與 implementation convention。
- 若新服務需沿用平台既有 convention，plan 需記錄來源與採用理由。

Validation target：

- Plan 作者能指出 api-gateway REST proxy、webhook RPC handler、service、repository、migration、event catalog module、DTO 與 frontend API/page/form 應如何切分。
- Plan 作者能指出 `identity.merchantId` 在 webhook service 層如何取得並用於 merchant scope query。

### Phase 2：Persistence Schema And Event Catalog

建立 Stage 1 所需資料結構與 code-defined event type catalog。

工作內容：

- 建立 `webhook_subscription`。
- 建立 `webhook_subscription_event_type`。
- 在 TypeScript 檔案定義第一版 event type catalog：
  - `withdrawal.created`
  - `withdrawal.canceled`
  - `withdrawal.failed`
  - `withdrawal.completed`
  - `deposit.created`
  - `deposit.failed`
  - `deposit.completed`
  - `deposit.blocked`
- `webhook_subscription_event_type` 以 `event_type` 保存 event key 字串。
- 建立 subscription-event relation 不可重複的約束或等效檢查。
- 不建立 merchant + endpoint URL 唯一性約束。

Validation target：

```text
migration applied
  -> webhook_subscription and webhook_subscription_event_type exist
  -> code-defined event type catalog exists
  -> duplicate subscription-event relation rejected
  -> same merchant can create duplicate endpoint_url subscriptions
```

### Phase 3：Event Type Read And Validation Capability

建立前端 checkbox options 與 subscription API 共用的 event type 讀取 / 驗證能力。

工作內容：

- 實作 `GET /webhook/merch/event-types`。
- 回傳 `WebhookEventTypeOptionDto[]` 所需欄位。
- 依 `sortOrder` 排序。
- 建立 event keys validation helper / service capability。
- 確保 subscription create / update 不接受 code-defined catalog 以外的 event key。

Validation target：

```text
merchant user
  -> GET /webhook/merch/event-types
  -> receives all first-version event keys including deposit.blocked
  -> invalid event key is rejected by subscription validation
```

### Phase 4：Subscription REST / RPC API And Merchant Scope

建立商戶可操作的 api-gateway REST API 與 webhook service management RPC。

工作內容：

- 實作 api-gateway `GET /webhook/merch/subscriptions` 並 proxy 到 webhook list subscriptions RPC。
- 實作 api-gateway `POST /webhook/merch/subscriptions` 並 proxy 到 webhook create subscription RPC。
- 實作 api-gateway `GET /webhook/merch/subscriptions/{subscriptionId}` 並 proxy 到 webhook get subscription RPC。
- 實作 api-gateway `PUT /webhook/merch/subscriptions/{subscriptionId}` 並 proxy 到 webhook update subscription RPC。
- 實作 api-gateway `DELETE /webhook/merch/subscriptions/{subscriptionId}` 並 proxy 到 webhook delete subscription RPC。
- 實作 webhook service merchant scoping，商戶只能查詢與修改自己名下 subscription。
- List 查詢必須以 `merchant_id = identity.merchantId` 篩選。
- Get / update / delete 查詢必須同時帶入 `subscriptionId` 與 `merchant_id = identity.merchantId`，避免其他 merchant 得知 subscription id 後透過 API 讀寫資料。
- 確認 REST request / response 與 RPC input / output 欄位對齊；gateway 僅做必要 envelope / identity 轉換。
- 實作 validation / normalization：endpoint URL trim、eventKeys 必須是陣列但可為空且不可重複、pagination 正整數與 pageSize 上限。
- 實作 error mapping：webhook RPC stable error code 對應 api-gateway REST status / error envelope。
- 實作 list pagination 與預設排序 `created_at desc, id desc`。
- Create subscription 時在同一 transaction 或等效一致性邊界內建立 `webhook_subscription` 與 `webhook_subscription_event_type` bindings。
- Write model 應分別操作 `webhook_subscription` 本體與 `webhook_subscription_event_type` binding。
- Query model 可為 list/detail 組裝 `eventTypeCount` 與 `eventTypes`，但不得將它們視為 subscription row 欄位。
- Update subscription 時更新 `webhook_subscription.endpoint_url`。
- Update event keys 時以 request 事件集合覆蓋目前 subscription-event relation：刪除既有 `webhook_subscription_event_type` rows，再 insert 新勾選的 event keys；若 request 為空陣列，刪除後不 insert。
- Update 的 endpoint URL 更新與 binding replacement 應在同一 transaction 或等效一致性邊界內完成。
- Delete 時採 soft delete，後續 list / get 不回傳已刪除資料。
- Delete 不刪除 binding rows；dispatcher 需透過 subscription deleted state 排除已刪除資料。

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
- 已知其他 merchant 的 `subscriptionId` 仍不能 get / update / delete 該 subscription。
- Event keys 必須來自 backend code-defined catalog。
- 同一 merchant 可建立多筆相同 endpoint URL 的 subscription。
- Deleted subscription 不出現在 UI list，也不會被後續 dispatcher 使用。

### Phase 5：Merchant Console Frontend Integration

串接 merchant console 的 webhook subscription management UI。此 phase 只負責第一版功能串接，不展開完整視覺設計或 delivery history。

工作內容：

- 建立或掛載 webhook subscription 管理入口。
- 串接 `GET /webhook/merch/event-types` 作為 event checkbox options 來源。
- 串接 subscription list / create / detail / update / delete API。
- Create / update form 使用 `endpointUrl` 與完整 `eventKeys` 集合送出。
- Event type 顯示文字第一版直接使用 `eventKey`，並依 `sortOrder` 排序。
- Delete 後刷新清單，確保軟刪除資料不再顯示。
- 顯示 backend validation error，例如 event key 無效、merchant scope 拒絕。

Validation target：

```text
merchant console user
  -> opens webhook management page
  -> sees event type checkboxes from API
  -> creates subscription
  -> edits endpointUrl and eventKeys
  -> deletes subscription
  -> list reflects latest backend state
```

### Phase 6：Stage 1 Verification And Tests

補齊 Stage 1 backend / frontend 驗證，確保 subscription management 能作為 Stage 2-4 的穩定基礎。

工作內容：

- Backend 測試 code-defined event type catalog、event type read endpoint、event key validation。
- Backend 測試 subscription create / list / get / update / delete。
- Backend 測試 merchant scoping，Merchant A 不能讀寫 Merchant B 的 subscription。
- Backend 測試以 Merchant B 的 token / identity 操作 Merchant A 的 `subscriptionId` 會被拒絕或回傳 not found，具體錯誤語意依 error convention 定案。
- Backend 測試 soft delete 後 list / get 排除已刪除資料。
- Backend 測試同 merchant 可建立重複 endpoint URL。
- Backend 測試 duplicate subscription-event relation。
- Backend 測試 update 會刪除未再勾選的 event bindings、insert 新勾選的 event bindings，並保留仍勾選的事件語意。
- Frontend 測試或 smoke test webhook management 主要操作路徑。
- 整合驗證前端送出的 `eventKeys` 與後端回傳的 `eventTypes` contract 一致。

Validation target：

```text
stage 1 verification
  -> backend API tests pass
  -> frontend subscription management smoke path passes
  -> code-defined event type catalog includes all first-version event keys
  -> merchant scoping and soft delete behavior verified
```

## Test Matrix

| Area | Coverage |
| --- | --- |
| Event catalog | `GET /event-types` returns all first-version event keys, sorted by `sortOrder`, without DB event type id. |
| Create subscription | Creates `webhook_subscription` and binding rows in one consistency boundary; returns detail read model with sorted `eventTypes`. |
| Create validation | Rejects blank / invalid endpoint URL, duplicated eventKeys, unsupported eventKeys, malformed body; allows empty eventKeys array. |
| Empty event subscription | Create with empty `eventKeys` creates subscription without binding rows; update with empty `eventKeys` removes all binding rows and keeps subscription. |
| Duplicate endpoint | Allows same merchant to create multiple subscriptions with the same endpoint URL. |
| List subscriptions | Filters by `identity.merchantId`, excludes deleted rows, returns pagination metadata, sorts by `created_at desc, id desc`. |
| Get subscription | Looks up by `subscriptionId + identity.merchantId`, returns detail read model, rejects deleted or cross-merchant access as not found. |
| Update subscription | Updates endpoint URL, replaces binding rows by delete + insert, validates eventKeys, keeps operation atomic. |
| Delete subscription | Soft deletes by `subscriptionId + identity.merchantId`, excludes deleted row from list/get, does not hard delete binding rows. |
| Gateway proxy | Maps REST requests to webhook management RPC inputs, passes `identity.merchantId`, maps RPC errors to REST status/envelope. |
| Frontend smoke | Loads options, creates, edits, deletes, handles empty/loading/error/delete confirmation states. |

## Sequencing

- Phase 1 先完成，避免 schema、route、module 與 frontend 串接邊界在 plan 期間反覆改動。
- Phase 2 是 Phase 3 / Phase 4 的前提，因 read endpoint 與 subscription validation 都依賴 code-defined event type catalog。
- Phase 3 可在 Phase 2 migration plan 穩定後開始。
- Phase 4 可與 Phase 3 後段並行，但 create / update validation 需共用 Phase 3 的 event key validation capability。
- Phase 5 依賴 Phase 3 event type endpoint 與 Phase 4 subscription REST / RPC contract；若 DTO 已穩定，可先以 mock API 或 stub data 開始 UI 串接。
- Phase 6 在 Phase 4 / Phase 5 完成後收斂，也可於各 phase 實作期間同步補測試，最後集中驗收 Stage 1 行為。

## Estimate

| Phase | 估時 | 依據 |
| --- | ---: | --- |
| Phase 1 | 0.5 天 | 決定新服務 backend/frontend boundary、migration、event catalog module、merchant scoping 與 soft delete convention。 |
| Phase 2 | 0.75 天 | 兩張表、code-defined event catalog、relation 約束；不建立 event type table 與 endpoint unique constraint。 |
| Phase 3 | 0.75 天 | Event type read endpoint 與 event key validation capability。 |
| Phase 4 | 2 天 | Api-gateway REST proxy、webhook management RPC、relation replacement、merchant scope、soft delete 與 error handling。 |
| Phase 5 | 1 天 | Merchant console API client、list/detail/form/checkbox 串接與基本錯誤顯示。 |
| Phase 6 | 1 天 | Backend API/integration tests、frontend smoke path 與 Stage 1 驗收。 |

總估時：6 天。

## Open Points

- Subscription list 是否需要回傳最新 delivery 狀態不在 Stage 1 範圍；若 UI 需要應另開 design extension。

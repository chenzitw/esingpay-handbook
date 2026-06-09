---
status: draft
updated_at: 2026-06-09
updated_by: Codex
---

# Webhook 交易事件推播 — Phase 1 Blueprint

## Goal

Phase 1 完成 webhook subscription management 與 webhook event type catalog。完成後商戶後台可以管理 webhook endpoint，後端可用 event catalog 驗證訂閱事件。

## Scope

In scope：

- `webhook_subscription` persistence。
- `webhook_event_type` seed 與 read model。
- `webhook_subscription_event_type` relation。
- Subscription create / list / get / update / delete API。
- Event type keys validation。
- Merchant scoping。

Out of scope：

- Outbox event production。
- Dispatcher / delivery worker。
- Delivery history。
- Signature verification docs。
- Endpoint ownership verification。
- Signing secret rotate。

## Inputs

- API surface：[`blueprint-api-surface.md`](./blueprint-api-surface.md)。
- Data model：[`blueprint-data-model.md`](./blueprint-data-model.md)。
- Service architecture：[`blueprint-service-architecture.md`](./blueprint-service-architecture.md)。

## Critical Decisions

- Event type catalog 由 migration seed 寫入，不開放 UI CRUD。
- Subscription 建立時預設 `active = true`。
- 第一版不開放 UI 修改 `active` 與 `signing_secret`。
- Subscription 建立時第一版先使用服務環境變數中的預設 signing secret 寫入。
- Subscription delete 採 soft delete。
- Event type read model 可由 plan 依既有後台 API pattern 決定是獨立 endpoint 或併入 subscription detail。

## Validation Target

Phase 1 完成時應能證明：

```text
merchant user
  -> list event type options
  -> create webhook subscription with endpoint_url + event keys
  -> list subscriptions
  -> get subscription detail
  -> patch endpoint_url or event keys
  -> soft delete subscription
```

驗證重點：

- Merchant 只能管理自己名下 subscription。
- Event keys 必須來自 `webhook_event_type`。
- Deleted subscription 不出現在 UI list，也不會被後續 dispatcher 使用。

## Estimate

| Item | 估時 |
| --- | ---: |
| Data model and migration plan | 1 天 |
| Subscription API and service | 2 天 |
| Event catalog seed/read/validation | 1 天 |

總估時：4 天。

## Open Points

- `deposit.blocked` 是否正式納入第一版 seed。
- Event type read model 採獨立 endpoint 或併入 subscription detail。
- `signing_secret` 是否需要加密保存仍需依 codebase secret storage convention 確認。

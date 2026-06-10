---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Error Contract Design

## Purpose

本文定義 webhook Stage 1 subscription management 的錯誤語意，供 api-gateway REST route、REST RPC proxy、webhook service management RPC 與 merchant console 共用。

本文只定義錯誤場景、穩定 error code、建議 HTTP status 與使用者可理解訊息。不定義 exception class、RPC enum 檔案位置、REST error envelope 形狀或 i18n 實作方式。

## Error Boundary

```text
merchant console
  -> api-gateway REST
  -> REST RPC proxy
  -> webhook service management RPC
```

Webhook service owns domain validation and merchant scope checks. Api-gateway owns REST status / envelope mapping.

REST 與 RPC 的 error code 應保持同一組穩定 code，避免 gateway 做語意轉換。Gateway 可以依既有 convention 包裝 HTTP status、trace id 或 field errors。

## Stage 1 Error Matrix

| Scenario | Stable code | Suggested HTTP status | User-facing message |
| --- | --- | ---: | --- |
| `endpointUrl` missing or blank | `WEBHOOK_ENDPOINT_URL_REQUIRED` | 400 | Callback URL is required. |
| `endpointUrl` format invalid | `WEBHOOK_ENDPOINT_URL_INVALID` | 400 | Callback URL must be a valid HTTP or HTTPS URL. |
| `eventKeys` missing or not an array | `WEBHOOK_EVENT_KEYS_REQUIRED` | 400 | Event type selection is required. |
| `eventKeys` contains duplicate key | `WEBHOOK_EVENT_KEYS_DUPLICATED` | 400 | Event type selection contains duplicates. |
| `eventKeys` contains unsupported key | `WEBHOOK_EVENT_KEY_UNSUPPORTED` | 400 | One or more event types are not supported. |
| `subscriptionId` malformed | `WEBHOOK_SUBSCRIPTION_ID_INVALID` | 400 | Subscription id is invalid. |
| Subscription not found, deleted, or not owned by merchant | `WEBHOOK_SUBSCRIPTION_NOT_FOUND` | 404 | Webhook subscription was not found. |
| Pagination input malformed | `WEBHOOK_PAGINATION_INVALID` | 400 | Pagination parameters are invalid. |
| `pageSize` exceeds maximum | `WEBHOOK_PAGE_SIZE_EXCEEDED` | 400 | Page size exceeds the maximum allowed value. |
| Unexpected persistence or transaction failure | `WEBHOOK_SUBSCRIPTION_OPERATION_FAILED` | 500 | Webhook subscription operation failed. |

## Merchant Scope Failure

When a merchant attempts to get, update, or delete another merchant's subscription, the API should use `WEBHOOK_SUBSCRIPTION_NOT_FOUND` rather than exposing ownership details.

Rationale:

- Subscription id is not a permission boundary.
- Other merchants may learn or guess a subscription id.
- Returning not found avoids confirming whether the subscription exists.

## Validation Notes

- Duplicate `endpointUrl` for the same merchant is allowed and must not produce an error.
- Empty `eventKeys` array is allowed. It means the subscription remains active as an endpoint record but is not subscribed to any event.
- Unsupported event keys are evaluated against the backend code-defined event catalog.
- Duplicate event keys in request should be rejected, not silently deduplicated, so frontend and backend state drift is visible.
- `eventKeys` response ordering should follow catalog `sortOrder`; request ordering has no business meaning.

## Open Points

- Whether error messages are returned directly from backend or mapped to locale-specific frontend copy.
- Whether validation errors include field-level metadata such as `field: "eventKeys"`.
- Final REST error envelope shape and RPC error enum location.

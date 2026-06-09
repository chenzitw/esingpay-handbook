---
status: draft
updated_at: 2026-06-10
updated_by: Codex
---

# Webhook 交易事件推播 — Type Contract Design

## Purpose

本文作為 webhook 型別定義設計入口。後續可在此拆分或補充 domain raw、event contract、request / response DTO 等型別設計。

目前不在本文定義最終欄位 contract；具體欄位需要等 webhook 服務應用端的實際使用範圍確定後再收斂。

## Type Layers

- Domain raw：內部交易服務產生 webhook event 前的原始 domain input。
- Event contract：POST 到商戶 endpoint 的 external-facing payload envelope 與 event-specific data。
- Management DTO：商戶後台管理 webhook subscription 與查詢 event type catalog 的 request / response DTO，REST 草案見 [`design-rest.md`](./design-rest.md)。
- Delivery internal DTO：dispatcher、worker、recovery 在服務內部傳遞 delivery 任務時使用的型別。

## Open Points

- 哪些 withdrawal / deposit domain raw 欄位會成為 webhook event 的來源。
- External event contract 是否需要與 external API DTO 共用命名與 primitive type。
- Management DTO 是否只服務 merchant console，或也會被 external API 文件引用。
- Delivery internal DTO 是否需要獨立於 persistence entity 定義。

# eSingPay Handbook

本 handbook 記錄專案在 AI 協作開發實踐下的 workflow 與 conventions（採用 Progressive Spec-Driven Workflow, PSDD）。Codebase 以 sibling folder 形式與本 handbook 並排存放。

## 關於 PSDD

**Progressive Spec-Driven Workflow (PSDD)** 是本專案的開發工作流。核心理念：以文件為主軸、漸進地把概念落地為可執行的實作，避免「先寫 code、再補文件」或「文件寫完即拋」兩種極端。

PSDD 將工作切成 cascade 四層——proposal（該不該做）、design（做什麼）、blueprint（怎麼推）、plan（怎麼改 code）——每層輸出對下游凍結，需要挑戰上游時透過 escalation 回到該層處理。版本完成 implementation 後另有 closing 流程，做 backlog 收尾與 spec distillation。

PSDD 是現階段務實做法。隨專案成熟，目標逐步演進至正式的 Spec-Driven Development（結構化 spec 格式、由 spec 生成 code 與 test、自動化 spec-code 一致性驗證）。

完整框架見 [`convention/workflow.md`](./convention/workflow.md)。

## 結構

```
<workspace>/
  <handbook>/        ← you are here
  <backend-codebase>/
  <frontend-codebase>/
```

Handbook 內容：

- `AGENTS.md` — AI agent 入口：working norms 與 navigation
- `convention/` — Workflow 與撰寫規範
- `idea/` — 各版本的 proposal、design、blueprint、plan 文件
- `spec/` — 已穩定、跨版本適用的架構 specification
- `VERSIONS.md` — 版本狀態總覽

## 快速入口

- **AI agent**：先讀 [`AGENTS.md`](./AGENTS.md)
- **Workflow 框架**：[`convention/workflow.md`](./convention/workflow.md)
- **版本狀態**：[`VERSIONS.md`](./VERSIONS.md)
- **當前版本**：`idea/vX.Y/README.md`（X.Y 取 `VERSIONS.md` 內最新項目）

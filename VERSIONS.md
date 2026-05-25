# Versions

本檔追蹤 handbook 各版本的開發狀態。詳細的 scope、backlog、spec persistence 進度於各版本的 `idea/vX.Y/README.md`。

## 狀態定義

| 狀態        | 意義                                  |
| ----------- | ------------------------------------- |
| **pending** | 規劃中、尚未啟動開發                  |
| **active**  | 開發活躍中                            |
| **closing** | Implementation 完成、hygiene 進行中   |
| **closed**  | Hygiene 與 spec distillation 也都做完 |

狀態轉換流程詳見 [`convention/workflow-closing.md`](./convention/workflow-closing.md)。

## 版本

| Version                     | Status | Notes                                  |
| --------------------------- | ------ | -------------------------------------- |
| [v1.0](idea/v1.0/README.md) | closed | PSDD 導入、初始 convention 與 workflow |
| [v1.2](idea/v1.2/README.md) | active | 新版金流帳務系統基本功能               |

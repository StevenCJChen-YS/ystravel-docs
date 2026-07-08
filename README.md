# Docs Index

這個 `docs/` 已完成第一階段整理：先依文件角色分資料夾，讓正式架構、操作資訊、參考資料、流程文件、分析稿與歷程輸出分開。

## Development Process（從這裡開始）

每個新功能要走的完整流程（PRD → SA/SD → BDD → 開發 → 測試 → Code Review → Release），以及對應的模板：

- [development-process/00-overview.md](./development-process/00-overview.md) — 先讀這份，說明角色分工與整體流程
- development-process/01-prd-template.md ～ 09-release-checklist.md — 各步驟的模板

每個功能的實際文件放在 [features/](./features/README.md)，一個功能一個資料夾。

## Current Structure

```text
docs/
  README.md
  development-process/
  features/
  architecture/
  analysis/
  operations/
  process/
  reference/
  archive/
    handoffs/
    legacy/
```

## Architecture

長期有效、作為團隊設計共識的文件：

- [architecture/CRM_AUTH_SERVICE_SA.md](./architecture/CRM_AUTH_SERVICE_SA.md)
- [architecture/CRM_AUTH_SERVICE_SD.md](./architecture/CRM_AUTH_SERVICE_SD.md)
- `architecture/YSTRAVEL_AUTH_PERMISSION_MODEL.md.docx`
- `architecture/YSTRAVEL_FRONTEND_ARCHITECTURE.md.docx`
- `architecture/YSTRAVEL_UI_DESIGN_SYSTEM.md.docx`

## Reference

參考資料、遷移規則、對照表：

- [reference/AUTHPORTAL_NUXTUI_COMPONENT_SCALE.md](./reference/AUTHPORTAL_NUXTUI_COMPONENT_SCALE.md) — AuthPortal Nuxt UI 元件尺寸客製化（Button 已完成，Input/Select 待遇到再調）
- `reference/EIP_AUTH_PERMISSION_MIGRATION_PLAN.md.docx`
- `reference/EIP_PERMISSION_MAPPING.md.docx`
- [reference/EIP_PERMISSION_MAPPING.xlsx](./reference/EIP_PERMISSION_MAPPING.xlsx)

## Operations

會隨專案進度變動、日常查閱頻率高的文件：

- `operations/YSTRAVEL_AUTH_CRM_PROGRESS.md.docx`

## Process

資安基線，開發流程本體已搬到上面的 [Development Process](#development-process從這裡開始) 並拆成模板：

- [process/SECURITY_AND_ISO27001_BASELINE.md](./process/SECURITY_AND_ISO27001_BASELINE.md)

## Analysis

特定主題分析文件：

- `analysis/YSTRAVEL_AUTHPORTAL_LOGIN_UI_ANALYSIS.md.docx`
- [analysis/CUSTOMER_EXCEL_FIRST_PASS_ANALYSIS.md](./analysis/CUSTOMER_EXCEL_FIRST_PASS_ANALYSIS.md)
- [analysis/CUSTOMER_FILTER_RULES_V1_ANALYSIS.md](./analysis/CUSTOMER_FILTER_RULES_V1_ANALYSIS.md)
- [analysis/INDUSTRY_CLASSIFICATION_MATCHING_STRATEGY.md](./analysis/INDUSTRY_CLASSIFICATION_MATCHING_STRATEGY.md)

## Archive

不再視為主文件，但保留供追溯：

- `archive/handoffs/2026-07-04-004100-auth-crm-permission-architecture.md.docx`
- `archive/handoffs/2026-07-06-004101-ai-bdd-development-process.md.docx`
- `archive/legacy/軟體開發.docx`
- `archive/legacy/AI_BDD_DEVELOPMENT_PROCESS_PLAN.md.docx` — 內容已拆解、整理進 `development-process/`，這份保留供追溯

## Rules Of Thumb

- `architecture/` 放長期維護的主文件。
- `reference/` 放來源資料與遷移對照。
- `operations/` 放會過期的環境、進度、啟動資訊。
- `archive/` 放 handoff、草稿、舊資料，不當成 source of truth。

## Known Issues

- 多數 `.docx` 看起來是同一批由 `python-docx` 輸出的 artifact，不像原始編輯檔。
- 部分中文 `.docx` 有亂碼風險，暫時不建議把它們當唯一主文件來源。
- 目前仍有幾份核心內容只有 `.docx`，後續建議補回可維護的 Markdown source。

## Suggested Next Cleanup

1. 把核心 `.docx` 補回 Markdown 原始版。
2. 為 `architecture/` 和 `reference/` 文件補狀態欄位與互相引用。
3. 視需要把 `.docx` 明確降為 export，而不是主編輯格式。

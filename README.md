# Docs Index

> 🧭 **先看這張「主題導覽」再搜尋。** 想知道某主題該開哪個檔，直接查表命中，不要全域搜尋整個 `docs/`。

## 主題導覽（Topic → 權威檔案）

| 我想知道… | 直接開這個 |
|---|---|
| **用什麼技術 / 技術棧 / 框架** | [architecture/CRM_AUTH_SERVICE_SD.md](./architecture/CRM_AUTH_SERVICE_SD.md)（開頭「建議技術」表） |
| **資料庫 / MySQL / PostgreSQL / 換 DB / 資料遷移** | [architecture/DATABASE.md](./architecture/DATABASE.md) |
| **開發流程 / 要走哪些步驟 / PRD·SA·SD·BDD** | [development-process/00-overview.md](./development-process/00-overview.md) |
| **某個功能的規格文件** | [features/](./features/README.md) → 一個功能一個資料夾 |
| **UI / 設計系統 / 顏色 / 元件 / RWD / 深色模式** | [design-system/foundation.md](./design-system/foundation.md) |
| **Nuxt UI 元件尺寸客製** | [reference/AUTHPORTAL_NUXTUI_COMPONENT_SCALE.md](./reference/AUTHPORTAL_NUXTUI_COMPONENT_SCALE.md) |
| **權限模型 / 角色體系** | [architecture/CRM_AUTH_SERVICE_SA.md](./architecture/CRM_AUTH_SERVICE_SA.md) ＋ [features/auth-role-management/](./features/auth-role-management/prd.md) |
| **登入 / auth 現況** | [features/auth-native-login-v2/README.md](./features/auth-native-login-v2/README.md) |
| **資安 / ISO 27001** | [process/SECURITY_AND_ISO27001_BASELINE.md](./process/SECURITY_AND_ISO27001_BASELINE.md) |
| **密碼政策** | [process/PASSWORD_POLICY.md](./process/PASSWORD_POLICY.md) |
| **客戶匯入 / 資料清洗 / 產業分類** | [features/crm-import-cleaning/](./features/crm-import-cleaning/prd.md) ＋ [analysis/](./analysis/CUSTOMER_EXCEL_FIRST_PASS_ANALYSIS.md) |

> 表裡沒有的主題，再往下看「Current Structure」按資料夾找；找到新的常用主題請補進這張表。

---

`docs/` 依「文件角色」分資料夾：正式架構、操作資訊、參考資料、流程文件、分析稿與封存歷程各自分開。

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
- 標 `.docx` 的檔案是舊 export（多為 `python-docx` 產出、部分中文有亂碼風險），僅供追溯，別當唯一主文件來源。

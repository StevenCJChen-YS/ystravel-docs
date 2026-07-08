# SD(系統設計)模板

> 複製到 `docs/features/<feature-slug>/sd.md`。**這份完全由 AI 草擬與負責技術正確性**,Steven 不需要看懂每一行技術細節,但要確認第 6 節(安全與稽核)有沒有漏洞感、第 9 節(測試策略)有沒有涵蓋到你在意的情境。

```markdown
# SD: <功能名稱>

| 項目 | 內容 |
|---|---|
| 對應 SA | ./sa.md |
| 狀態 | 草稿 / 已確認 |
| 建立日期 | YYYY-MM-DD |

## 0. 白話摘要

3-5 句話：這份要蓋什麼、關鍵技術選擇、最大的風險或取捨。讓不讀技術細節的人看完這段就掌握重點。下面各節是專業展開。

## 1. 系統架構

哪些模組/服務會被新增或修改?(例如 NestJS module、Vue view、Prisma model)

## 2. API 設計

| Method | Path | 說明 | 權限 |
|---|---|---|---|
| | | | |

每支 API 的 request/response 型別另外寫在 `docs/features/<feature-slug>/api-spec.md`(如果有新增或異動 API)。

## 3. DB Schema 異動

Prisma model 的新增/修改欄位,附上 migration 名稱規劃。

## 4. Service / Module 分層

依照既有 pattern(參照 `Ystravel-CRM-Backend/src/<existing-module>/`),不要引入新的分層風格。

## 5. 認證與授權方式

- 沿用哪個 guard(`JwtAuthGuard`、`PermissionsGuard`)?
- 新增了哪些 permission code?格式參照既有 `CRM.<MODULE>.<ACTION>`。

## 6. 安全與稽核（必填 — 依 SECURITY_AND_ISO27001_BASELINE.md §6.3）

| 欄位 | 內容 |
|---|---|
| 敏感欄位保護 | 是否需要 hash / mask,哪些欄位 |
| Audit log | 這個功能的哪些操作要寫 audit log |
| 備份 / 還原 | 是否影響備份策略 |
| 錯誤處理與告警 | |
| 測試資料策略 | 測試/開發環境不可用真實個資,說明如何造測試資料 |

## 7. 效能與非功能需求

（預期資料量、分頁、逾時、併發等,若無特殊需求可寫「沿用既有標準」)

## 8. 錯誤碼

| 情境 | HTTP Status | 說明 |
|---|---|---|

## 9. 測試策略

依 [07-testing-strategy.md](./07-testing-strategy.md) 分工,列出這個功能實際會寫哪些測試。

## 10. 待決策事項

列出 SD 階段發現、需要 Steven 決定的架構或取捨問題(附上選項與 AI 建議)。
```

## 使用方式

1. AI 根據已確認的 SA 草擬 SD,並在第 10 節列出需要 Steven 拍板的架構取捨(例如「要不要引入新套件」「要不要拆新模組」),每個選項附上取捨說明。
2. Steven 只需要對第 10 節的問題做選擇,不需要自己想技術方案。
3. 確認後,依 [04-bdd-guide.md](./04-bdd-guide.md) 寫 Gherkin,再進入開發。

# SD: CRM Excel 匯入與清洗

| 項目 | 內容 |
|---|---|
| 對應 SA | ./sa.md |
| 狀態 | 已確認 |
| 建立日期 | 2026-07-08 |

## 1. 系統架構

新增到既有 `src/imports/` 模組,不新增獨立模組:

- `import-cleaning.ts` — 純函式,套用 5 條排除規則到單一列資料,回傳 `{ excluded, exclusionReasons }`。不依賴 NestJS DI,方便單獨測試。
- `import-cleaning.feature` / `import-cleaning.steps.ts` — BDD 測試(jest-cucumber),覆蓋 `import-cleaning.ts` 的規則邏輯。
- `imports.service.ts` — 新增 `createBatch()` 方法,呼叫 `import-cleaning.ts`,寫入 `ImportBatch` 與 `RawOrderRow`。內部用 ExcelJS 的 streaming reader(`WorkbookReader`)逐列讀取,每 1000 列分批 `createMany` 一次(見第 7 節,v1 版一次性讀取+寫入曾在真實約 10 萬列的檔案上讓 process 記憶體爆掉,已修正)。
- `imports.controller.ts` — 新增 `POST /imports/batches`,用 `FileInterceptor` 接收上傳檔案。

## 2. API 設計

| Method | Path | 說明 | 權限 |
|---|---|---|---|
| POST | `/imports/batches` | 上傳 Excel,建立批次並清洗 | `CRM.IMPORT.MANAGE` |

Request:`multipart/form-data`,欄位 `file`(單一 `.xlsx` 檔案)。

Response(`201`):

```json
{
  "data": {
    "id": "uuid",
    "sourceFileName": "客戶分析報表.xlsx",
    "status": "NORMALIZED",
    "rowCount": 108535,
    "excludedRowCount": 12508
  }
}
```

錯誤(`400`):非 Excel 檔案、無法解析、缺少必要欄位時回傳 `{ "message": "..." }`。

## 3. DB Schema 異動

不需要 migration。既有 `ImportBatch`、`RawOrderRow` 欄位已足夠:

- `RawOrderRow.rawJson` — 存原始列(逐欄位 key-value)。
- `RawOrderRow.normalizedJson` — 存 `{ excluded: boolean, exclusionReasons: string[] }`。
- `ImportBatch.status` — `PENDING` → `NORMALIZED` / `FAILED`。
- `ImportBatch.rowCount` — 這次上傳的總列數。

## 4. Service / Module 分層

沿用既有 pattern:`ImportsController` 呼叫 `ImportsService`,`ImportsService` 呼叫 `PrismaService`。`import-cleaning.ts` 是不經過 DI 的純函式,被 `ImportsService` import 使用,方便 BDD 測試不需要啟動 Nest 應用程式。

## 5. 認證與授權方式

沿用 `JwtAuthGuard` + `PermissionsGuard` + `@RequirePermissions('CRM.IMPORT.MANAGE')`,與既有 `findBatches` 等方法相同,不新增權限模型。

## 6. 安全與稽核

| 欄位 | 內容 |
|---|---|
| 敏感欄位保護 | `rawJson` 內含身份證字號、護照號碼等 Restricted 資料,依賴既有 `CRM.IMPORT.MANAGE` 存取控管,這次不額外加密(與既有 `RawOrderRow` 讀取 API 一致的保護層級) |
| Audit log | 這次範圍不新增 audit log 表(專案尚未有通用 audit log 機制);上傳批次本身的 `createdAt`/`importedById` 已足以追溯是誰在何時上傳 |
| 備份 / 還原 | 無新增備份需求,沿用既有資料庫備份策略 |
| 錯誤處理與告警 | 解析失敗時批次標記 `FAILED` 並記錄 `RawOrderRow` 為空,不告警(內部工具,人工會發現) |
| 測試資料策略 | BDD/單元測試一律使用虛構身份證字號、護照號碼、姓名,不使用 `docs/analysis/` 分析中出現的任何真實資料樣本 |

## 7. 效能與非功能需求

單檔最多約 10 萬列(依現有 Excel 規模)。**v1 版曾用「一次讀進記憶體 + 一次性 `createMany`」處理,2026-07-08 拿 Steven 的真實檔案上傳時讓 backend process 記憶體爆掉、崩潰重啟**(前端會卡在 loading,因為對應的 HTTP 連線變孤兒,不會自己跳錯)。

已改為:

- ExcelJS streaming reader(`ExcelJS.stream.xlsx.WorkbookReader`)逐列讀取,不一次性持有整份檔案在記憶體。
- 每 1000 列 `createMany` 分批寫入一次,避免單一 SQL 語句超過 MySQL `max_allowed_packet`,也縮短單次交易鎖定時間。

**已實測驗證**:用合成的 108,535 列檔案(欄位結構比照真實 Excel)直接呼叫 `ImportsService.createBatch()`(繞過 HTTP/認證層的一次性腳本),約 21.5 秒完成,process 沒有崩潰。之後若資料量再明顯放大(例如百萬列等級)或上傳常態性逾時,再考慮背景工作佇列與非同步處理(目前範圍不需要)。

## 8. 錯誤碼

| 情境 | HTTP Status | 說明 |
|---|---|---|
| 未上傳檔案 | 400 | `file is required` |
| 檔案格式不是 Excel | 400 | `invalid file format` |
| Excel 解析失敗 | 400 | `failed to parse excel file` |
| 無權限 | 403 | 沿用既有 `PermissionsGuard` |

## 9. 測試策略

依 [07-testing-strategy.md](../../development-process/07-testing-strategy.md):

- **BDD(jest-cucumber)**:`import-cleaning.feature` 覆蓋 5 條排除規則 + 邊界情況(多規則命中、正常列)。
- **Unit test**:`imports.service.spec.ts`(如需要)可另外覆蓋 Excel 解析與寫入邏輯的邊界情況(例如空檔案)。
- **手動 QA**:Steven 上傳一份真實結構、假資料內容的 Excel,確認批次列表與清洗結果符合預期。

## 10. 待決策事項

無。

# SA: CRM Excel 匯入與清洗

| 項目 | 內容 |
|---|---|
| 對應 PRD | ./prd.md |
| 狀態 | 已確認 |
| 建立日期 | 2026-07-08 |

## 1. 系統邊界

只涉及 `Ystravel-CRM-Backend`(新增上傳/清洗邏輯)與 `Ystravel-CRM-Frontend`(接上既有「匯入 Excel」按鈕)。不涉及 AuthService、EIP。既有系統邊界與權限模型參照 [CRM_AUTH_SERVICE_SA.md](../../architecture/CRM_AUTH_SERVICE_SA.md) §4.4、§8、§9,本文件不重複描述。

## 2. 角色與權限

沿用既有 `CRM.IMPORT.MANAGE` permission code(已用在 `imports.controller.ts` 的讀取 API 上)。

| 角色 | 可以做什麼 |
|---|---|
| 具備 `CRM.IMPORT.MANAGE` 的使用者 | 上傳 Excel、查看批次與清洗結果 |
| 不具備此權限的使用者 | 無法呼叫上傳 API(沿用既有 `PermissionsGuard`,不需新邏輯) |

## 3. 操作流程(正常路徑)

```text
使用者在 ImportsView 點擊「匯入 Excel」
  -> 選擇檔案
  -> 前端呼叫 POST /imports/batches(multipart/form-data)
  -> 後端建立 ImportBatch(status = PENDING)
  -> 後端解析 Excel,逐列寫入 RawOrderRow(rawJson = 原始列)
  -> 後端對每一列套用 5 條排除規則,寫入 normalizedJson(excluded, exclusionReasons)
  -> 全部處理完後,ImportBatch.status 改為 NORMALIZED
  -> 前端刷新批次列表,顯示新批次與狀態
```

## 4. 狀態轉換

`ImportBatchStatus`(既有 enum,不新增值):

```text
PENDING -> NORMALIZED   # 這次範圍會用到的轉換
PENDING -> FAILED       # 解析失敗時
```

`REVIEWING`、`PUBLISHED` 是下一個功能(候選比對/人工審核)會用到的狀態,這次不處理。

## 5. 例外流程

- 上傳非 Excel 檔案 / 檔案損毀 → 回傳 400,批次標記為 `FAILED`,不寫入任何 `RawOrderRow`。
- 上傳過程中連線中斷 → 依 NestJS 預設行為處理,不做額外的斷點續傳(超出這次範圍)。
- 沒有權限的使用者呼叫 API → 沿用既有 `PermissionsGuard`,回傳 403,不需要新邏輯。

## 6. 資料流與敏感資料

- Excel 檔案本身**不落地保存**(只在記憶體中解析,處理完即釋放),降低檔案外流風險 —— 對應 PRD §7 的風險摘要。
- 原始列資料(`rawJson`)含 Restricted 等級個資(身份證字號、護照號碼、聯絡方式等),永久保存在 `raw_order_rows`,只有具備 `CRM.IMPORT.MANAGE` 的角色能查詢(沿用既有讀取 API 的權限控管)。
- `normalizedJson` 這次只記錄排除標記,不含額外個資衍生欄位。

## 7. 與外部系統的互動

無(純內部處理,不呼叫外部系統)。

## 8. 風險點與人工審核點

- 排除規則寫錯的風險最高 —— 已用 `docs/analysis/CUSTOMER_FILTER_RULES_V1_ANALYSIS.md` 中已經對 108,535 筆真實資料驗證過的規則定義,不是憑空設計,降低此風險。
- 這次不做人工審核介面(排除標記只是記錄,不會擋住任何操作),人工審核點留給候選比對功能設計。

## 9. 待決策事項

- 無(範圍內的問題已在 PRD §9、Example Mapping 中處理完畢)。

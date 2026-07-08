# CRM Import And Analysis MVP

## 1. 目前共識

今天的優先順序不是先把整個 CRM 做滿，而是先建立一個可用來分析旅客與訂單資料的第一版能力。

主軸如下：

1. `AuthService` 繼續作為 SSO 與權限中心。
2. CRM 第一版先聚焦在 `Import + Analysis Workbench`。
3. EIP 整合維持分階段進行，不在目前資料分析階段強耦合。
4. AI 暫時放在第二階段，先把資料治理、資料分層、清洗規則、人工審核流程建立好。

## 2. 第一版真正要做的不是完整 CRM，而是資料分析工作台

第一版建議目標：

- Excel upload
- Raw preview
- Validation report
- Normalized orders
- Customer candidates
- Manual review queue
- Analysis dashboard

這一版的商業價值不是「客戶 360 完整管理」，而是：

- 盡快看清楚現有訂單資料品質
- 盡快知道哪些欄位最亂
- 盡快建立旅客/訂單的標準欄位規範
- 盡快反饋給業務，避免之後資料越積越亂

## 3. 資料分層

### 3.1 Raw layer

用途：

- 原始 Excel 檔保存
- 每列資料完整保存
- 保留追溯能力

原則：

- 不直接進正式 customers
- 權限最嚴格
- 僅少數匯入管理者可看

### 3.2 Normalized layer

用途：

- 日期、金額、狀態、通路、國家、角色等欄位標準化
- 欄位映射與格式修正
- 記錄每列清洗結果與錯誤原因

### 3.3 Curated layer

用途：

- 正式 customer / order / participant 分析
- 給 CRM 查詢、篩選、統計
- 給後續報表與人工審核使用

## 4. 第一版資料流程

```text
Excel 匯入
  -> import batch
  -> raw rows 保存
  -> 欄位標準化
  -> 驗證與錯誤分類
  -> 建立 order candidates / traveler candidates
  -> 高信心自動連結
  -> 低信心進人工審核
  -> 再寫入正式 customers / orders / order_participants
```

## 5. 第一版最重要的資料表

- `import_batches`
- `raw_order_rows`
- `customers`
- `customer_identities`
- `orders`
- `order_participants`
- `customer_merge_candidates`

第一階段至少先把這些表的角色定清楚，不一定要一次把所有功能 UI 做滿。

## 6. 資安原則

### 6.1 敏感資料處理

- 身分證、護照、台胞證等資料不要在正式分析畫面明碼大量曝光。
- 正式身份表優先保存 `hash` 與 `masked value`。
- 原始個資只留在 raw layer 與必要的受控處理流程。

補充：

- 若要正確抽旅客與訂單資料，原始 Excel 在分析階段不一定能完全去識別化。
- 這是可接受的，但必須在受控環境中進行。
- 去識別化不是第一目標，防外洩、可追溯、最小存取才是第一目標。

### 6.2 權限分級

建議至少分成：

- `CRM_IMPORT_ADMIN`
- `CRM_DATA_ADMIN`
- `CRM_ANALYST`
- `CRM_SALES`
- `CRM_MANAGER`

### 6.3 AI 使用原則

第一階段原則：

- 不把 raw Excel 個資直接丟外部 AI
- 不讓 AI 直接做正式客戶合併
- 不讓 AI 判定唯一真實身份

若未來要導入 AI：

- 優先考慮本地或受控內網模型
- 先從去識別化統計、分類建議、髒資料標註開始
- 所有 AI 建議都必須保留人工覆核點

第二階段才考慮：

- 地址解析
- 國家/路線分類
- 髒資料偵測
- 疑似重複客戶建議
- 客戶偏好摘要

## 7. 關於 LLM / LLaMA / RAG 的判斷

目前這個問題的第一核心不是 RAG，而是：

- 欄位治理
- 資料清洗
- 身份比對
- 訂單與旅客關聯建模

所以第一階段不建議以 `LLaMA + RAG` 作為主方案。

較合理的順序：

1. 先建立 deterministic ETL / validation pipeline
2. 先建立人工審核流程
3. 先建立欄位規範與資料品質報告
4. 再評估本機或內網 LLM 做低風險輔助分析

## 8. 你之後給 Excel 時，我們怎麼合作

你不需要現在一次把所有篩選條件講完。

建議流程：

1. 你先把 Excel 給我
2. 我先幫你盤點欄位、列數、重複情況、空值、格式問題
3. 我先提出「目前這份資料可抽什麼、不能穩定抽什麼」
4. 你再告訴我你真正想要的輸出名單或分析口徑
5. 我們一起收斂成正式篩選規則與欄位規範

也就是說：

- 可以先給檔案
- 篩選規則可以在看到資料後再一起定
- 不需要一開始就把所有條件講得很完整

處理原則：

- 先做資料結構盤點
- 先做欄位與品質分析
- 先判斷哪些欄位真能穩定抽取
- 再收斂成正式規則

也就是先看資料，再定標準，不是先憑想像定死規則

## 8.1 資料安全邊界

處理真實 Excel 時，原則上：

- 只為目前任務分析使用
- 不做無關用途延伸
- 不把 raw data 當成一般文件散佈
- 不把真實個資寫進 repo 或測試假資料

若後續要落地系統，也應以 [../process/SECURITY_AND_ISO27001_BASELINE.md](../process/SECURITY_AND_ISO27001_BASELINE.md) 為基線

## 9. 可直接提供的檔案格式

你可以直接給我：

- `.xlsx`
- `.xls`
- `.csv`

如果原始檔很敏感，建議你可以先提供：

- 範例檔
- 局部工作表
- 去識別化版本

但如果你希望我直接依真實資料做欄位分析，也可以直接提供原始 Excel，我會先以資料結構盤點與清洗規則分析為主。

## 10. 我收到 Excel 後第一輪會幫你做的事

1. 欄位盤點
2. 資料筆數與訂單筆數分析
3. 旅客 vs 訂單關聯判斷
4. 哪些欄位可直接當 key
5. 哪些欄位格式最亂
6. 哪些資料需要人工判斷
7. 初版篩選規則建議
8. 初版欄位治理建議

## 11. 下一步

收到 Excel 後，優先不要急著做最終客戶主檔匯入。

先做三件事：

1. 產出資料剖析報告
2. 定義第一版欄位規範
3. 定義第一版匯入與人工審核流程

等這三件事站穩後，再往正式 CRM 客戶/訂單模型推進。

## 12. 已確認的 V1 排除規則

目前已與行銷主管討論出第一版排除規則，並已有實際分析結果：

- [../analysis/CUSTOMER_FILTER_RULES_V1_ANALYSIS.md](../analysis/CUSTOMER_FILTER_RULES_V1_ANALYSIS.md)

目前 V1 規則：

1. `身份證字號` 為空者先排除
2. `身分` 為 `導遊` 或 `領隊` 者排除
3. `警示訊息` 有值者排除
4. `來源帳號` 為 `永信旅行社` 或 `永信福委會` 者排除
5. `護照號碼` 為 `未建旅客檔` 者排除

後續待補：

- 員工身分證黑名單排除
- 同業 / 直客 分類規則
- 正式寫入資料庫欄位映射與轉換邏輯

## 13. 已落地的前端工作台方向

目前 CRM Frontend 已可先朝 `本機 Excel 處理工作台` 方向落地，目的如下：

1. 先讓使用者上傳 Excel 後直接在瀏覽器端做欄位對應與規則篩選
2. 先產出 `保留名單 / 排除名單 / 無身分證清單`
3. 先讓行銷、業務、主管快速確認規則影響
4. 先避免 raw Excel 在第一階段就直接進正式資料庫或外部服務

此方向的好處：

- 較符合目前資料敏感度與資安邊界
- 不必等正式 ETL pipeline 全部完成才能開始用
- 可先加快欄位治理與業務規範收斂

第一版前端工作台建議最少具備：

- Excel upload
- Sheet selection
- Field mapping
- Rule toggles / editable rule lists
- Summary metrics
- Downloadable Excel outputs

第二階段再往下接：

- 正式 raw layer 匯入
- Audit log
- 同業 / 直客 自動判定
- 人工覆核佇列
- 正式寫入 customers / orders / participants

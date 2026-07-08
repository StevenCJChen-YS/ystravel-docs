# Example Mapping: CRM Excel 匯入與清洗

## Rule 1:身份證字號為空的列要標記排除

Example:
- 身份證字號欄位是空白 → 標記排除,原因 `NO_ID_NUMBER`
- 身份證字號欄位有值(即使格式看起來不對)→ 不標記排除(格式驗證不在這次範圍)

## Rule 2:身分為「導遊」或「領隊」的列要標記排除

Example:
- 身分欄位 = `導遊` → 標記排除,原因 `GUIDE_OR_LEADER`
- 身分欄位 = `領隊` → 標記排除,原因 `GUIDE_OR_LEADER`
- 身分欄位 = `旅客` → 不排除

Question:已解決 —— 只比對這兩個字串,不做模糊比對或大小寫轉換(來源資料是固定選項,不會有變形)。

## Rule 3:警示訊息有值的列要標記排除

Example:
- 警示訊息欄位有任何非空白內容 → 標記排除,原因 `WARNING_MESSAGE_PRESENT`
- 警示訊息欄位是空白或只有空白字元 → 不排除

## Rule 4:來源帳號為特定內部帳號的列要標記排除

Example:
- 來源帳號 = `永信旅行社` → 標記排除,原因 `INTERNAL_SOURCE_ACCOUNT`
- 來源帳號 = `永信旅行社股份有限公司` → 標記排除,原因 `INTERNAL_SOURCE_ACCOUNT`
- 來源帳號 = `永信福委會` → 標記排除,原因 `INTERNAL_SOURCE_ACCOUNT`
- 來源帳號 = 其他值(例如同業或直客帳號)→ 不排除

## Rule 5:護照號碼為「未建旅客檔」的列要標記排除

Example:
- 護照號碼 = `未建旅客檔` → 標記排除,原因 `PASSPORT_PLACEHOLDER`
- 護照號碼為實際護照號碼或空白 → 不排除(空白護照號碼本身不是排除條件,只有這個特定字串才是)

## Rule 6(邊界情況):一列可能同時符合多條規則

Example:
- 身分為「導遊」且身份證字號也是空的 → 標記排除,排除原因要**同時記錄** `GUIDE_OR_LEADER` 和 `NO_ID_NUMBER`(不是只記第一個命中的規則)

Question:已解決 —— 依 `CUSTOMER_FILTER_RULES_V1_ANALYSIS.md` 的分析,約 11,177 列會命中 2 條以上規則,必須完整記錄才能之後追溯。

## Rule 7:不符合任何排除規則的列正常寫入,不做額外清洗

這次範圍不做日期/金額/國家名稱格式清洗(那些留給候選比對或後續功能),`normalizedJson` 這次只記錄「是否排除」與「排除原因」。

Example:
- 一列沒有命中任何排除規則 → `normalizedJson.excluded = false`,`normalizedJson.exclusionReasons = []`

## Question(未解決,留待下一個功能處理)

- 訂單層級要不要因為某一列被排除就整張訂單排除?→ 這次不決定,只在列層級標記。
- 員工身分證黑名單、同業名單比對規則 → 尚未提供資料,列為後續功能。

## 最終確認的 Scenario 清單(對應 Gherkin)

1. 正常列,不命中任何規則 → 不排除
2. 身份證字號為空 → 排除(NO_ID_NUMBER)
3. 身分為導遊 → 排除(GUIDE_OR_LEADER)
4. 身分為領隊 → 排除(GUIDE_OR_LEADER)
5. 警示訊息有值 → 排除(WARNING_MESSAGE_PRESENT)
6. 來源帳號為永信旅行社股份有限公司 → 排除(INTERNAL_SOURCE_ACCOUNT)
7. 護照號碼為未建旅客檔 → 排除(PASSPORT_PLACEHOLDER)
8. 同時命中多條規則 → 排除原因同時記錄多個
9. 沒有匯入權限的使用者嘗試上傳 → 系統拒絕
10. 上傳非 Excel 檔案 → 系統回報清楚錯誤,不寫入任何資料

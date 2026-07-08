# BDD 指南:Example Mapping 與 Gherkin

## 為什麼要用 BDD

Steven 不是資訊背景,平常很難看懂技術規格,但**看得懂具體例子**(「A 可以做 X,B 不行」)。BDD 的價值就是把需求變成 Steven 看得懂、AI 也能拿來當測試依據的「例子」,兩邊用同一份文件溝通。

## 步驟 1:Example Mapping(先用例子對齊,不要急著寫 Gherkin)

在 PRD 確認後、寫 SA 之前,先做一次 Example Mapping。可以直接在對話中進行,不需要額外工具:

- **Rule(規則)**:這個功能的一條業務規則
- **Example(例子)**:符合這條規則的具體情境,包含正向、負向、邊界情況
- **Question(問題)**:目前答不出來、需要 Steven 決定的問題

範例:

```text
Rule: 只有具備停用權限的後台使用者可以停用會員
Example:
  - 超級管理員可以停用一般會員
  - 沒有權限的客服不能停用會員
  - 管理員不能停用自己的帳號
Question:
  - 停用是否需要填寫原因?
  - 停用後既有登入 token 是否立即失效?
```

**這一步 AI 負責提出規則草稿與例子草稿,Steven 負責:**
- 確認哪些規則是真的需要的
- 刪掉低價值或重複的情境
- 回答或指派人回答 Question

把結果存到 `docs/features/<feature-slug>/prd.md` 或另開 `example-mapping.md`,兩者都可以,重點是不要遺失。

## 步驟 2:把 Rule / Example 寫成 Gherkin

Gherkin 檔案放在對應 repo 的模組資料夾內,跟程式碼放一起,例如:

```text
Ystravel-CRM-Backend/src/imports/imports-cleaning.feature
Ystravel-CRM-Backend/src/imports/imports-cleaning.steps.ts
```

格式:

```gherkin
Feature: 會員停用
  Rule: 只有具備停用權限的後台使用者可以停用會員

    Scenario: 有權限的管理員停用啟用中的會員
      Given 後台使用者 "Amy" 具備 "MEMBER_DISABLE" 權限
      And 會員 "王小明" 目前狀態為啟用
      When "Amy" 停用會員 "王小明"
      Then 會員 "王小明" 的狀態應為停用
      And 系統應記錄操作者 "Amy" 的停用紀錄

    Scenario: 沒有權限的使用者嘗試停用會員
      Given 後台使用者 "Bob" 不具備 "MEMBER_DISABLE" 權限
      When "Bob" 嘗試停用會員 "王小明"
      Then 系統應拒絕此操作
```

撰寫原則:

- 每個 Scenario 描述**使用者或外部系統可觀察的行為**,不要寫資料庫欄位或內部實作細節。
- 每個 Scenario 控制在 3-6 個 step。
- 每個功能至少涵蓋:1 個正向情境、1 個權限/例外情境、1 個邊界情境。
- 同一個 domain 的用詞要一致(例如「停用」不要有時又寫「停權」)。
- 不要一次追求 100% 情境覆蓋,先寫核心行為,之後迭代補充。
- 用繁體中文撰寫,Steven 才能直接看懂並確認。

## 步驟 3:誰負責什麼

| 工作 | 負責人 |
|---|---|
| 提出 Rule/Example 草稿 | AI |
| 刪減、確認、回答 Question | Steven |
| 把確認過的 Rule/Example 轉成 Gherkin | AI |
| 確認 Gherkin 場景是否涵蓋自己在意的情況 | Steven |
| 撰寫 step definition、讓測試從紅燈變綠燈 | AI |

## 測試框架

後端使用 `jest-cucumber`(沿用既有 Jest,不另外裝 Cucumber CLI)。執行方式與既有 `npm test` 相同,`.feature` 檔會被自動抓到。詳見 [07-testing-strategy.md](./07-testing-strategy.md)。

## 常見錯誤

| 錯誤 | 風險 | 避免方式 |
|---|---|---|
| 跳過 Example Mapping 直接寫 Gherkin | 场景遗漏、AI 猜業務規則 | 先做 Example Mapping,人類刪減後再轉 Gherkin |
| Scenario 寫得太技術化 | Steven 看不懂,無法確認 | 只描述可觀察行為 |
| 把所有測試都塞進 BDD | 測試變慢、難維護 | 依 [07-testing-strategy.md](./07-testing-strategy.md) 分層 |
| 每個 Scenario 都建立新的 step | step definition 難維護 | 建立共用 domain language,重複使用 step |

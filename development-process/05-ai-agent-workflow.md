# AI Agent 開發交付方式

## 核心原則

AI(Claude Code)最適合接收**明確、可驗收**的任務。這份文件定義「什麼是好任務」以及 AI 在動手做之前、做的時候、做完之後分別要遵守什麼邊界。

## 不好的任務 vs 好的任務

不好的任務(太模糊):

> 幫我做會員停用功能。

好的任務(有明確依據與完成條件):

> 請根據 `src/members/member-disable.feature` 實作會員停用功能。
>
> 完成條件:
> - 所有 `member-disable.feature` 的 scenario 都要通過
> - 沿用既有 service / repository / guard pattern
> - 使用既有 permission guard
> - 停用操作必須寫入 audit log
> - 補必要的 unit test
> - 不改動無關功能
> - 完成後回報修改了哪些檔案、測試結果

## AI Agent 開發前必須做的事

1. 先讀對應的 `sa.md` / `sd.md`,不要跳過直接看 `.feature` 就開始寫。
2. 讀既有相同 domain 的實作(例如要加新的 CRM module,先讀 `customers` 模組怎麼寫的),沿用既有 pattern,不引入新風格。
3. 確認 `.feature` 檔案內容與 SD 的 API 設計一致,如果不一致要先跟 Steven 確認,不要自己猜。

## AI Agent 的安全邊界

- 必須先讀現有程式架構,沿用既有 pattern。
- 必須讓 BDD / unit / integration 測試通過才算完成。
- 必須開 PR 給 Steven review,不直接推到 `main`。
- 不應直接改 production 資料或連線 production 資料庫。
- 不應接觸或輸出 secrets、`.env` 內容、真實個資。
- 測試資料一律使用假資料,不使用真實旅客/客戶資料(依 `SECURITY_AND_ISO27001_BASELINE.md` §4.3)。
- 不新增不必要的套件依賴。
- 不改動與任務無關的檔案。

## 完成後必須回報的內容

- 修改了哪些檔案
- 做了哪些設計取捨(如果 SD 沒寫清楚、AI 自己臨時決定的部分,一定要點出來讓 Steven 知道)
- 測試結果(哪些通過、哪些還沒寫)
- 是否有偏離 SD 的地方,以及原因

## Prompt 範本

### 需求拆解 Prompt(PRD 之後、Example Mapping 用)

```text
以下是需求草稿:
[貼上需求]

請用 BDD 的角度幫我整理:
1. 使用者角色
2. 業務規則
3. 正向情境
4. 負向情境
5. 邊界情境
6. 待釐清問題
7. 建議的 Gherkin scenarios

請先聚焦使用者可觀察行為,不要直接進入技術實作。
```

### Gherkin 產生 Prompt

```text
請根據以下規則產生 Gherkin feature file。

要求:
- 使用繁體中文
- 用 Feature / Rule / Scenario 結構
- 每個 Scenario 控制在 3 到 6 個 steps
- Then 描述使用者或外部系統可觀察結果
- 避免描述資料庫內部欄位
- 補上至少 1 個正向情境、1 個權限失敗情境、1 個邊界情境
```

### AI Agent 實作 Prompt

```text
請根據 [feature 檔案路徑] 實作功能。
請先閱讀既有架構與相同 domain 的實作 pattern。

限制:
- 不新增不必要套件
- 不改無關檔案
- 沿用既有 service / repository / guard pattern
- 讓 BDD 測試通過
- 補必要 unit / integration test
- 完成後回報修改檔案、設計取捨、測試結果
```

### PR Review Prompt(自我審查用,見 [08-code-review-checklist.md](./08-code-review-checklist.md))

```text
請 review 這個 PR 是否符合 [feature 檔案] 的內容。

請檢查:
- 是否完整覆蓋 scenario
- 是否漏掉權限或例外情境
- 是否有硬寫測試資料
- 是否破壞既有流程
- 是否引入安全風險
- 測試是否只測 happy path
- 是否需要補文件或 migration 說明
```

## 常見錯誤與避免方式

| 錯誤 | 風險 | 避免方式 |
|---|---|---|
| 直接叫 AI 寫 code,沒有先寫 examples | AI 會猜業務規則,猜錯要重工 | 先做 Example Mapping、寫 acceptance criteria |
| AI 產太多低價值情境 | 測試維護成本爆炸 | Steven 刪減,只留核心情境 |
| 沒有 CI/測試把關 | AI 產的碼風險無法控管 | PR 前必跑 lint/test/build |
| 沒有人類 review 就合併 | 錯誤可能看起來很合理但其實錯 | 即使一人團隊,也要用 [08-code-review-checklist.md](./08-code-review-checklist.md) 走一次自我 review |

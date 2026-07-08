# Code Review Checklist

用在每一個 PR,即使目前只有 Steven 一人 + AI,也要照這份清單走一次(可以請 AI 用「PR Review Prompt」對自己的 PR 做一次獨立檢查,見 [05-ai-agent-workflow.md](./05-ai-agent-workflow.md))。

## 檢查項目

- [ ] 是否完整覆蓋對應 `.feature` 的所有 scenario?
- [ ] 是否漏掉權限檢查或例外情境(例如沒權限的使用者、資料不存在)?
- [ ] 是否有硬寫死的測試資料或魔術數字,應該用設定或常數?
- [ ] 是否破壞了既有功能或流程(有沒有跑過既有測試)?
- [ ] 是否引入安全風險(SQL injection、未授權存取、敏感資料外洩、log 洩漏 secrets)?
- [ ] 測試是否只測 happy path,漏了失敗情境?
- [ ] 是否需要補文件(`docs/features/<slug>/` 是否同步更新)或資料庫 migration 說明?
- [ ] 是否新增了不必要的套件依賴?
- [ ] 是否符合 `SECURITY_AND_ISO27001_BASELINE.md` 的資料分級與稽核要求(如果這個功能涉及敏感資料)?
- [ ] PR 描述是否清楚說明「改了什麼、為什麼、怎麼驗證」?

## 常見錯誤對照表

| 錯誤 | 風險 | 避免方式 |
|---|---|---|
| 直接叫 AI 寫 code,沒有先寫 examples | AI 會猜業務規則 | 先寫 examples 和 acceptance criteria |
| BDD scenario 太技術化 | Steven 看不懂,無法確認對不對 | 用使用者可觀察行為描述 |
| AI 產太多低價值情境 | 測試維護成本爆炸 | 人類刪減,只保留核心情境 |
| 每個 scenario 都產新 step | Step definition 難維護 | 建立共用 domain language |
| 把所有測試都塞進 BDD | 測試變慢、難 debug | Unit/Integration/BDD/E2E 分層(見 07) |
| 沒有 CI gate | AI 產碼風險無法控管 | PR 必跑 lint/test/build |
| 沒有人類 review | 錯誤可能看起來很合理 | 即使一人團隊,也要用這份 checklist 自我審查 |

## 誰簽核

Steven 最終要看過 PR 描述與 checklist 結果,確認同意才合併。AI 可以先跑過 checklist 並回報結果,但簽核責任在 Steven。

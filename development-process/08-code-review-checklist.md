# Code Review Checklist

用在每一個 PR,即使目前只有 Steven 一人 + AI,也要照這份清單走一次(可以請 AI 用「PR Review Prompt」對自己的 PR 做一次獨立檢查,見 [05-ai-agent-workflow.md](./05-ai-agent-workflow.md))。

---

## 兩軸原則(2026-07-23 改版)

Review 分成**兩個獨立的軸**,分別跑、**結果不合併、不重新排序**:

| 軸 | 問的問題 | 判準來源 |
|---|---|---|
| **Spec** | 有沒有**忠實做到**當初講好的事? | `.feature` + `prd.md` |
| **Standards** | 寫得**好不好**、合不合規矩? | 鐵則 + 設計系統 + 資安 + 壞味道 |

**為什麼要分開跑**:一份改動可以一軸過、另一軸掛——

- 規範全部遵守,但做的是錯的東西 -> Spec 掛、Standards 過
- 完全照 issue 做,但破壞了專案慣例 -> Spec 過、Standards 掛

混在同一份清單裡,模型會自己權衡出一個「整體還行」的結論,**其中一軸會遮蔽另一軸**。
分開報告就不會。

> 用 AI 跑的話:開**兩個平行的 sub-agent**,各給各的判準來源,兩份報告並排呈現,不要合併。

---

## Spec 軸

- [ ] 是否完整覆蓋對應 `.feature` 的**所有** scenario?
- [ ] `prd.md` 的 Acceptance Criteria 有沒有漏掉的?
- [ ] 有沒有 **scope creep**——做了 PRD/票沒有要求的東西?
- [ ] 有沒有「看起來做了但其實做錯」的需求?
- [ ] 是否漏掉權限檢查或例外情境(沒權限的使用者、資料不存在、資料已刪除)?
- [ ] 測試是否只測 happy path,漏了失敗情境?

每一條發現都要**引用出處**(哪一條 scenario、PRD 第幾點)。

---

## Standards 軸

### A. 專案鐵則(平台專案必查)

`ystravel-platform/CLAUDE.md` 的五條擋件標準,逐條對照:

- [ ] **模組邊界**——業務模組之間有沒有直接 import 對方的 service/repository/prisma 查詢?
      只准用對方 NestJS module `exports` 的介面。
- [ ] **資料主權**——組織與人事資料(Company/Department/職稱)有沒有被 `core` 寫入?
      那些歸 `hr` 擁有,`core` 只能唯讀。(見 `docs/adr/0001-data-sovereignty-sor.md`)
- [ ] **防腐層**——科威的資料格式有沒有洩漏到 `integration/cowell` 之外?出了這層一律轉內部 DTO。
- [ ] **前端分層**——見下面 B 段。
- [ ] **沒有為了「未來可能需要」引入**微服務、message queue、額外資料庫等基礎設施。

### B. UI / 設計系統

判準權威是 [design-system/design.md](../design-system/design.md) **§13 Code Review 擋件清單**,重點:

- [ ] 有沒有各頁自己發明顏色/尺寸,而不是用 design token?
- [ ] 業務頁有沒有用 `!important` 或深層選擇器硬蓋元件樣式?(視覺調校一律走全域 theme)
- [ ] 重複的 UI 樣式有沒有抽成 `shared/ui/` 共用元件,而不是各頁重造?
- [ ] 新元件的歸屬對不對?(跨模組 -> `shared/ui/<類型>`;單一模組 -> `modules/<模組>/components/`)
- [ ] 有沒有做出「純換名字、對全域 theme 無加值」的薄 `App*` 包裝?(那種不做)
- [ ] 四種狀態都有嗎?(loading / 空 / 錯誤 / 正常,見 [10-uiux-guide.md](./10-uiux-guide.md) §4)
- [ ] 有 dark mode 的畫面,light / dark 都看過了嗎?

### C. 資安與稽核

- [ ] 是否引入安全風險(SQL injection、未授權存取、敏感資料外洩、log 洩漏 secrets)?
- [ ] 是否符合 [SECURITY_AND_ISO27001_BASELINE.md](../process/SECURITY_AND_ISO27001_BASELINE.md) 的資料分級與稽核要求?
- [ ] 測試/fixture 有沒有用到真實個資?(一律假資料,見 [07](./07-testing-strategy.md))

### D. 測試品質

- [ ] 有沒有**同義反覆**的測試——期望值是用跟程式碼同一套邏輯算出來的?(見 [07](./07-testing-strategy.md) 反模式 1)
- [ ] 有沒有**與實作綁死**——mock 內部協作者、測 private method、直接查 DB 驗證?
- [ ] `.feature` 有沒有寫進呈現方式(toast/inline)?那些屬 UI/UX,不進 Gherkin。

### E. 程式壞味道(判斷題,不是硬性違規)

以下取自 Fowler《Refactoring》ch.3。**專案既有規範永遠壓過這份 baseline**,
而且**工具(lint/type-check)已經擋掉的一律略過**:

| 味道 | 是什麼 | 怎麼修 |
|---|---|---|
| Mysterious Name | 名字看不出這東西做什麼/裝什麼 | 改名;想不出誠實的名字,代表設計本身模糊 |
| Duplicated Code | 同一段邏輯出現在改動的多處 | 抽出來,兩邊都呼叫 |
| Feature Envy | 某個 method 一直伸手拿別人的資料,超過拿自己的 | 把 method 搬到它羨慕的資料旁邊 |
| Data Clumps | 同幾個欄位/參數老是一起出現 | 包成一個型別一起傳 |
| Primitive Obsession | 用字串/數字代替一個該有型別的領域概念 | 給那個概念自己的小型別 |
| Repeated Switches | 同一個 switch/if 串在多處重複 | 換成多型,或一張兩邊共用的 map |
| Shotgun Surgery | 一個邏輯改動要動很多檔案 | 把一起變的東西收進同一個模組 |
| Divergent Change | 同一個檔案為了好幾種不相干的理由被改 | 拆開,讓每個模組只為一種理由改 |
| Speculative Generality | 為了「未來可能需要」加的抽象/參數/hook | 刪掉,inline 回去,等真的有需求再說 |
| Message Chains | `a.b().c().d()` 這種長串導航 | 藏到第一個物件的一個 method 後面 |
| Middle Man | 一個類別/函式幾乎只是在轉手 | 砍掉,直接呼叫真正的目標 |
| Refused Bequest | 子類別/實作忽略或覆寫掉大部分繼承來的東西 | 別繼承,改用組合 |

### F. 其他

- [ ] 是否破壞了既有功能或流程(有沒有跑過既有測試)?
- [ ] 是否有硬寫死的測試資料或魔術數字,應該用設定或常數?
- [ ] 是否新增了不必要的套件依賴?
- [ ] 是否需要補文件(`docs/features/<slug>/` 是否同步更新)或 migration 說明?
- [ ] Migration 是否向後相容(expand-contract:先加後刪,新舊程式可並存)?
- [ ] 新出現的領域術語有沒有進 `CONTEXT.md`?
- [ ] PR 描述是否清楚說明「改了什麼、為什麼、怎麼驗證」?

---

## 報告格式

兩軸各自成段,**不要合併也不要跨軸排名**:

```markdown
## Spec
（發現 N 項,每項引用 scenario / PRD 出處）

## Standards
（發現 M 項,標明哪些是硬性違規、哪些是判斷題）
```

最後一行只寫:各軸幾項、各軸**自己內部**最嚴重的是哪一項。不要挑「整體最嚴重」——
那正是分軸要防的重新排序。

---

## 常見錯誤對照表

| 錯誤 | 風險 | 避免方式 |
|---|---|---|
| 直接叫 AI 寫 code,沒有先寫 examples | AI 會猜業務規則 | 先寫 examples 和 acceptance criteria |
| BDD scenario 太技術化 | Steven 看不懂,無法確認對不對 | 用使用者可觀察行為描述 |
| AI 產太多低價值情境 | 測試維護成本爆炸 | 人類刪減,只保留核心情境 |
| 每個 scenario 都產新 step | Step definition 難維護 | 建立共用 domain language(見 `CONTEXT.md`) |
| 把所有測試都塞進 BDD | 測試變慢、難 debug | Unit/Integration/BDD/E2E 分層(見 07) |
| 兩軸混在一起 review | 一軸的問題被另一軸蓋掉 | 分開跑、分開報告,不合併 |
| 一個 session 做完整個功能 | context 髒掉,開始改到不相關的檔案 | 每張票一個全新 session(見 00 §3) |
| 沒有 CI gate | AI 產碼風險無法控管 | PR 必跑 lint/test/build |
| 沒有人類 review | 錯誤可能看起來很合理 | 即使一人團隊,也要用這份 checklist 自我審查 |

---

## 誰簽核

Steven 最終要看過 PR 描述與兩軸的 review 結果,確認同意才合併。
AI 可以先跑過兩軸並回報,但簽核責任在 Steven。

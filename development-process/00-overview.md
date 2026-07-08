# 開發流程總覽

| 項目 | 內容 |
|---|---|
| 文件目的 | 定義「一個新功能從想法到上線」該走過的最小必要步驟,讓每個功能都留下同一組文件,而不是各寫各的 |
| 適用範圍 | Ystravel-CRM、Ystravel-AuthService/AuthPortal、未來的 EIP 重建、以及日後任何新專案 |
| 文件狀態 | 取代 `docs/archive/legacy/AI_BDD_DEVELOPMENT_PROCESS_PLAN.md.docx`,為目前的 source of truth |
| 前置文件 | [SECURITY_AND_ISO27001_BASELINE.md](../process/SECURITY_AND_ISO27001_BASELINE.md)(資安必填欄位的來源) |

---

## 1. 為什麼需要這份流程

目前的開發方式是「看畫面為主、想到功能就做、後端幾乎交給 AI」。這在專案小的時候沒問題,但這次要做全新 CRM + SSO,規模與風險都會提高:

- 沒有人(包含 AI)記得住「為什麼這樣設計」,只留下程式碼,沒留下決策理由。
- 後端交給 AI 寫,如果沒有先講清楚規則,AI 會用猜的,猜錯了要花更多時間除錯。
- SSO、權限、客戶個資這幾塊出錯的代價很高,不能靠「做完看感覺對不對」。
- 未來會有第 2、3 個人加入,沒有共同流程,大家做法會分歧。

## 2. 角色分工(這是這份流程最重要的前提)

目前團隊是「Steven 一人 + AI」,不是傳統的 PM/SA/PG/QA 分開的團隊。所以這份流程把角色拆成兩邊,而不是拆成很多人:

| 角色 | 負責人 | 負責什麼 |
|---|---|---|
| PM / PO / UI-UX | **Steven** | 決定要不要做、做給誰用、畫面長怎樣、驗收標準對不對、風險是否可接受 |
| SA / SD / PG / 第一線 QA | **AI(Claude Code)** | 依 Steven 的決定草擬 PRD 細節、系統分析、系統設計、寫程式、寫測試、跑測試、自我 review |
| 最終決策與簽核 | **Steven** | 每份文件、每個 PR,最後都要 Steven 看過同意才算數 |

白話原則:**Steven 永遠不需要自己寫 SA/SD 或懂後端架構細節,但每一份文件、每一支 PR,Steven 都要看得懂重點、能表達同意或不同意。** 如果哪一份文件 Steven 看不懂,代表那份文件寫太技術化,要請 AI 重寫成白話版。

未來加入第 2、3 人時,只要把上面「AI 負責」的部分,拆出一部分給真人負責即可,文件格式不用改。

## 3. 整體流程

```text
Discovery(要不要做)
  -> PRD(做什麼、給誰用、風險是什麼)
  -> Example Mapping(用例子對齊行為)
  -> Gherkin(把行為寫成可執行的規格)
  -> SA(系統分析:流程、角色、資料流、風險點)
  -> SD(系統設計:架構、API、DB、安全機制)
  -> 任務拆解
  -> AI 協助開發(照 05 的邊界進行)
  -> 測試(Unit / Integration / BDD / 手動 QA)
  -> Code Review(照 08 的 checklist)
  -> 走 GitHub Flow 開 PR、合併
  -> Release(照 09 的 checklist)
  -> 上線後觀察與迭代
```

每個步驟對應的模板:

| 步驟 | 模板 |
|---|---|
| PRD | [01-prd-template.md](./01-prd-template.md) |
| SA | [02-sa-template.md](./02-sa-template.md) |
| SD | [03-sd-template.md](./03-sd-template.md) |
| Example Mapping / Gherkin | [04-bdd-guide.md](./04-bdd-guide.md) |
| AI 開發交付 | [05-ai-agent-workflow.md](./05-ai-agent-workflow.md) |
| Git / 分支 / 發版 | [06-git-and-release-flow.md](./06-git-and-release-flow.md) |
| 測試分工 | [07-testing-strategy.md](./07-testing-strategy.md) |
| Code Review | [08-code-review-checklist.md](./08-code-review-checklist.md) |
| Release | [09-release-checklist.md](./09-release-checklist.md) |
| UI/UX 設計流程 | [10-uiux-guide.md](./10-uiux-guide.md) |
| 共用設計系統(顏色/字體/主色) | [../design-system/foundation.md](../design-system/foundation.md) |

## 4. 每個功能的文件放哪

```text
docs/features/<feature-slug>/
  prd.md
  sa.md
  sd.md
  api-spec.md      # 有新增/異動 API 才需要
  test-plan.md
features/<feature-slug>.feature   # 在對應的 repo 內,跟程式碼放一起,例如 Ystravel-CRM-Backend/src/imports/imports-cleaning.feature
```

`<feature-slug>` 用小寫、連字號,例如 `crm-import-cleaning`。

## 5. 什麼情況可以跳過哪些步驟

不是每個改動都要走完整流程。判斷原則:

- **純 UI 調整**(顏色、間距、文字):不需要 PRD/SA/SD,直接改、直接看畫面確認即可。這是 Steven 最擅長的部分,保持原本的工作方式。
- **小 bug 修正**:不需要完整流程,但如果修的是權限或資料正確性相關的 bug,要在 PR 描述寫清楚「為什麼會錯、怎麼修的、怎麼確認修好了」。
- **新功能 / 新 API / 改資料結構 / 改權限規則**:走完整流程,從 PRD 開始。
- **涉及個資、金流、權限、SSO 的任何改動**:一定要走完整流程,且 SA/SD 必須補資安欄位(見 01/02/03 模板)。

## 6. 什麼時候要重新讀這份文件

- 每次要開始一個「新功能」而不是單純改畫面時。
- 新人加入團隊時,作為第一份要讀的文件。
- 每季或每次流程卡住時,回來檢討這份流程本身要不要調整。

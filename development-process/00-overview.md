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

> **2026-07-23 改版**:流程本身沒變,但明確劃出 **session 邊界**,並在前後各補一個步驟
> (`/grill-with-docs` 逼問、`/to-tickets` 切票)。改版理由見本節末「為什麼要分 session」。

### 先決定尺寸(大多數改動不必走全套)

| 尺寸 | 判準 | 走什麼 |
|---|---|---|
| **微改** | UI/CSS 調整、文案、一行 bug | **直接做**,不走流程(見 §5) |
| **中等** | 一個功能,1–3 天,單一模組 | Session 1 + 2,**跳過 to-tickets**,一次 implement |
| **大** | 跨模組、多天、要做好幾天 | **完整流程**(下圖) |
| **迷霧** | 連「要做什麼」都講不清楚 | 先 `/wayfinder` 撥霧,之後才進下圖 |

「中等」是目前絕大多數的工作。`to-tickets` 只在「一張票塞不進一個 context window」時才值得。

### 完整流程

```text
┌─ Session 1 ── 想清楚(Steven 全程在場)─────────────────┐
│  /grill-with-docs  一次問一題、每題附推薦答案            │
│      │  碰到術語 -> 當場更新 CONTEXT.md                  │
│      │  碰到難逆轉的決策 -> 開 ADR                       │
│      v                                                 │
│  prd.md -> example-mapping.md -> <slug>.feature         │
│  結束條件:Steven 確認 .feature 的 scenario 就是他要的    │
└────────────────────────────────────────────────────────┘
                    │ 產物是文件,文件本身就是交接
                    v
┌─ Session 2 ── 技術設計(AI 主導,Steven 只拍板)─────────┐
│  讀 prd.md + .feature + CONTEXT.md(不需要對話歷史)      │
│      v                                                 │
│  sa.md -> sd.md                                        │
│      │  Steven 只回答 sa §9 / sd §10 的「待決策事項」     │
│      v                                                 │
│  /to-tickets -> tickets/01-*.md ...(含 blocking edges)  │
└────────────────────────────────────────────────────────┘
                    │ 每張票 = 一個乾淨 context
                    v
┌─ Session 3..N ── 每張票一個 session ───────────────────┐
│  /implement 讀 tickets/NN-*.md                          │
│      ├─ 外圈 BDD:.feature scenario 一條條變綠            │
│      └─ 內圈 TDD:複雜純邏輯用單元測試(見 07)            │
│      v                                                 │
│  兩軸 code review(見 08)-> commit                      │
└────────────────────────────────────────────────────────┘
                    v
        PR -> main(見 06)-> Release(見 09)
```

### 為什麼要分 session

AI 的 context window 有限,而且接近上限時推理品質會下降(約 120K token 之後)。
一路做到底的 session,到後半段會開始改到不相關的檔案、忘記前面講過的決策。

**這套流程剛好把 session 邊界放在「文件寫完」的地方**——PRD、SA、SD 本身就是交接文件,
新 session 讀檔案就能接上,不需要對話歷史。這是走重流程換來的好處:文件即 context。

**唯一不能省的紀律:每張票開一個全新 session。**

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
| 共用設計系統(顏色/字體/主色/元件) | [../design-system/design.md](../design-system/design.md)（唯一權威規範書） |

### 各步驟用哪個 skill

| Session | 指令 | 做什麼 |
|---|---|---|
| 1 | `/grill-with-docs` | 一次問一題逼問到對齊,同時維護 `CONTEXT.md` 與 ADR |
| 1 | `/feature-kickoff` | 接著產 PRD / example mapping / .feature |
| 2 | `/feature-kickoff` | 產 SA / SD |
| 2 | `/to-tickets` | 切成垂直切片票(只有「大」尺寸才需要) |
| 3..N | `/implement` | 照票實作,內部驅動 BDD/TDD |
| 3..N | `/code-review` | 兩軸 review(見 08) |
| 視需要 | `/prototype` | 版面或狀態機講不清時,做丟棄式多版本比較(見 10) |
| 視需要 | `/wayfinder` | 連要做什麼都講不清的大工程,先撥霧 |

skill 本體在 `my-agent/000_Agent/skills/`(`~/.claude/skills` 是指過去的 symlink)。

## 4. 每個功能的文件放哪

**現行專案(ystravel-platform)**——文件跟程式碼同一個 branch、同一個 PR:

```text
ystravel-platform/
  CONTEXT.md                          # 專案詞彙表(全域,不屬於單一功能)
  docs/adr/NNNN-<slug>.md             # 難逆轉的架構決策
  docs/features/<feature-slug>/
    prd.md
    example-mapping.md
    sa.md
    sd.md
    api-spec.md                       # 有新增/異動 API 才需要
    tickets/                          # 只有「大」尺寸功能才會有
      01-<slug>.md
      02-<slug>.md
  apps/api/src/<模組>/<slug>.feature   # .feature 跟程式碼放一起,不放 docs/
```

**歷史專案**(auth-*、crm-import-cleaning,2026-07-16 以前)在本 repo 的 `features/`,保留追溯不搬。

`<feature-slug>` 用小寫、連字號,例如 `crm-import-cleaning`。同一個 slug 同時用在:
功能文件資料夾、`feature/<slug>` git 分支、`.feature` 檔名。

**檔名慣例**:一律 kebab-case。只有「必須照順序讀」的東西才編號——
本資料夾的 `00~10`、ADR 的 `NNNN`、tickets 的 `01~NN`。功能文件內部固定就那幾個名字,**不編號**。

## 5. 什麼情況可以跳過哪些步驟

> 尺寸速查表在 §3 開頭,本節是各尺寸的細則。兩邊有出入時以 §3 為準。

不是每個改動都要走完整流程。判斷原則:

- **純 UI 調整**(顏色、間距、文字):不需要 PRD/SA/SD,直接改、直接看畫面確認即可。這是 Steven 最擅長的部分,保持原本的工作方式。
- **小 bug 修正**:不需要完整流程,但如果修的是權限或資料正確性相關的 bug,要在 PR 描述寫清楚「為什麼會錯、怎麼修的、怎麼確認修好了」。
- **新功能 / 新 API / 改資料結構 / 改權限規則**:走完整流程,從 PRD 開始。
- **涉及個資、金流、權限、SSO 的任何改動**:一定要走完整流程,且 SA/SD 必須補資安欄位(見 01/02/03 模板)。

## 6. 什麼時候要重新讀這份文件

- 每次要開始一個「新功能」而不是單純改畫面時。
- 新人加入團隊時,作為第一份要讀的文件。
- 每季或每次流程卡住時,回來檢討這份流程本身要不要調整。

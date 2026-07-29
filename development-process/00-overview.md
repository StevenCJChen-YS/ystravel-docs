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
>
> **2026-07-29 補**:尺寸表補上「**機制**」一格與 `/brainstorm`。原本四格漏掉
> 「比微改大、但沒有領域規則要定」這一類(側邊欄搜尋、全站介面縮放),於是每次遇到都靠
> 當場判斷、下一場再被重問一次——**做法只活在實作裡而沒寫進文件,等於沒有這條規則。**

### 先決定尺寸(大多數改動不必走全套)

| 尺寸 | 判準 | 走什麼 |
|---|---|---|
| **微改** | UI/CSS 調整、文案、一行 bug | **直接做**,不走流程(見 §5) |
| **機制** | 比微改大,但**沒有領域規則要定**(三條判準見下) | `/brainstorm` 產計劃書 -> 直接做,**不寫 PRD/SA/SD** |
| **中等** | 一個功能,1–3 天,單一模組 | Session 1 + 2,**跳過 to-tickets**,一次 implement |
| **大** | 跨模組、多天、要做好幾天 | **完整流程**(下圖) |
| **迷霧** | 連「要做什麼」都講不清楚 | 先 `/wayfinder` 撥霧,之後才進下圖 |

「中等」是目前絕大多數的工作。`to-tickets` 只在「一張票塞不進一個 context window」時才值得。

**這張表不是單一軸**:微改/中等/大是尺寸、迷霧是清晰度、機制是性質。它真正在回答的是
「**走什麼流程**」,左欄只是標籤——判的時候三個軸都要過一遍,不要只問「這個大不大」。

#### 「機制」怎麼判(2026-07-29 新增)

這格卡在微改與中等之間,**改的是「東西怎麼運作」,不是「業務規則是什麼」**。
已完成的兩個實例:全站介面縮放(字級怎麼算)、側邊欄搜尋(頁面怎麼找)。

**三條判準要同時成立**,少一條就回去走中等、從 PRD 開始:

1. **沒有新的領域概念**——不需要往 `CONTEXT.md` 加詞。
2. **沒有「誰在什麼情況下可以做什麼」的規則**——試著寫一條 Gherkin scenario,
   寫不出來、或寫出來只是在描述畫面該長怎樣,才算。
3. **不新增要存起來、要有人維護的資料**——沒有新資料表,也沒有新的維護介面。

**第 3 條是防濫用的閘,不能省。** 前兩條是「不必寫規格」的理由,第 3 條是「它不會長大」的
保證;少了它,這格會變成「我覺得我這個沒規則」的逃生門,任何工作都能拿它跳過 PRD。

**走完長什麼樣**:`/brainstorm` -> 計劃書存 my-agent `100_Todo/projects/YYYY-MM-DD_主題.md`
-> 依計劃直接實作。**不產 `prd.md`/`sa.md`/`sd.md`,也不產 `.feature`。**
硬跑完整 Session 1 只會得到一份沒有 scenario 可寫的 PRD 和一個空的 `.feature`
——那不是流程嚴謹,是流程空轉。

> **活教材(2026-07-29)**:「常用連結」原本被推薦跑 `/brainstorm`,也就是這一格
> (看起來只是把幾條外部網址放進側欄),是 Steven 一句「**現在當然只有一條,但未來很可能
> 有多條**」把它打回中等的——會長就要有資料模型與維護介面,**第 3 條當場不成立**。
> 這格的邊界就在這:**一旦東西要存起來、要有人維護,它就不是這格了。**

> ⚠️ **計劃書放哪是待決事項**(2026-07-29 刻意先不動)。放 my-agent 是**現況**
> (`/brainstorm` skill 的預設路徑),但計劃書是專案專屬的,照知識三層分工的口訣
> 「換一個專案這條還成立嗎」該進 platform `docs/features/`,才會跟程式碼同一個 PR、
> 未來加入的人才看得到。改位置要連帶搬既有兩份計劃書、改 skill 預設路徑、改 roadmap 連結,
> 不在本次範圍。

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

### 多 session 並行(2026-07-25 定案)

上面講的是「一條線的 session 怎麼切」。實務上還有另一個問題:**能不能同時開兩個
session 做不同的事**?能,但形狀要固定。

**形狀 = 一前台 + 一後台,上限兩場。**

| 場 | 吃什麼 | 典型工作 |
|---|---|---|
| **前台** | Steven 的注意力 | `/grill-with-docs`、UI 逐項調、驗收 |
| **後台** | 自己跑,只在做完時回報 | 照票 `/implement`、寫測試、寫文件 |

**三場以上不要開。** 那會讓 Steven 變成「在視窗之間切換的人」,每場都只給得出片段的
注意力,品質反而比一場一場做還差。

**五條規則:**

1. **開工先宣告領土**——會動哪些檔、**明確不動哪些**,寫進 commit message 或 PR 描述。
   「明確不動」比「會動」更重要:它才是另一場判斷有無衝突的依據。
2. **各自 feature 分支,PR 逐一合**,不要兩場共用一支分支。
3. **共用文件寫前先 pull,後合的負責解衝突**。共用文件 = `CONTEXT.md`、
   `design-system/design.md`、`my-agent/MEMORY.md`、當日 `daily/`。
4. **PR 合併後,要主動跟還開著的另一場說「main 更新了,pull」**。
   AI 不會自己知道 main 前進了——它只在開場 pull 過一次。
5. **daily 各寫各的章節**,標明是哪一條線,不改別場寫的段落。

> **活教材(2026-07-24)**:「平台命名收斂」與「主鈕對比度」兩場並行,沒撞到靠的正是
> 前四條。命名場的 commit 訊息寫了「明確不動 `HomePage`、`accessible-systems.ts`」,
> 對比場才敢直接判定無衝突、不必停下來問。若當時只寫了「動了哪些」,對比場就只能猜。

**git worktree 現階段不引入**。成本(多一份工作目錄、node_modules、環境變數)大於效益;
等真的出現「兩場非得同時改同一 repo 的重疊區」再說。目前靠分支 + 領土宣告就夠。

### 每個步驟對應的模板

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
| **機制**尺寸 | `/brainstorm` | **取代整套 Session 1–2**:對齊目標、比方案、產計劃書,確認後直接實作。判準見上 |
| 1 | `/grill-with-docs` | 一次問一題逼問到對齊,同時維護 `CONTEXT.md` 與 ADR。**唯一「問」的地方**(見下) |
| 1 | `/feature-kickoff` | 把 grill 的結論寫成 PRD / example mapping / .feature,**不重跑 discovery** |
| 2 | `/feature-kickoff` | 產 SA / SD |
| 2 | `/to-tickets` | 切成垂直切片票(只有「大」尺寸才需要) |
| 3..N | `/implement` | 照票實作,內部驅動 BDD/TDD |
| 3..N | `/code-review` | 兩軸 review(見 08) |
| 3..N | `/tdd` | `implement` 內部驅動的 red-green 迴圈 |
| 視需要 | `/prototype` | 版面或狀態機講不清時,做丟棄式多版本比較(見 10) |

`/code-review` 是 **Claude Code 內建指令,不是 skill**,不用裝(怎麼餵它判準見 [08](./08-code-review-checklist.md) §怎麼實際跑)。

### Session 1 兩支 skill 的邊界:grill 問、kickoff 寫(2026-07-25 定案)

Session 1 有兩支 skill,**分工是「誰負責問」與「誰負責寫」,不是流程的前後兩半**:

| skill | 職責 | 產出 |
|---|---|---|
| `/grill-with-docs` | **唯一「問」的地方** | 對齊後的理解、`CONTEXT.md` 詞彙、ADR |
| `/feature-kickoff` | **「寫」**:把 grill 的結論落成文件 | `prd.md`、`example-mapping.md`、`<slug>.feature` |

**問題是這兩支本來會重複問一輪。** `feature-kickoff` Step 1 明文要「conversationally 問
discovery questions」——解決什麼問題、誰用、完成長怎樣、UI/UX 期待,而這四題 grill 一定
已經逼問過。Steven 被同一組問題問兩次,第二次還會覺得「剛剛不是講過了」。

**約定:**

1. **grill 要問完 PRD 需要的全部輸入**,不只問架構取捨。包含 `01-prd-template.md` 的
   使用者、完成定義、驗收標準,以及 **§7 資安與資料分級**(這欄最容易被 grill 漏掉,
   因為它不像「決策」而像「表格」——但它不是選填)。
2. **kickoff 不重跑 discovery**。同一個 session 內已經跑過 grill 的話,Step 1 直接把
   grill 的結論寫成 PRD 草稿給 Steven 看。
3. **缺口要明說是缺口**。grill 沒覆蓋到的欄位,kickoff 補問時要講「這題 grill 沒問到」,
   不要混在一堆問題裡重問一遍——Steven 才分得出「你沒在聽」與「這題真的還沒談」。
4. **反過來也成立**:grill 是**通用**逼問工具(也用於純架構決策、非功能討論),
   不綁定功能開發。所以不把 PRD 產出塞進 grill 本身,`prd.md` 一律由 kickoff 寫。

> ⚠️ **`/wayfinder` 尚未安裝**(給「連要做什麼都講不清」的大工程用,例如 W3 SP 團系統)。
> 它重度依賴 issue tracker,要用之前得先適配,屆時再說。

skill 本體在 `my-agent/000_Agent/skills/`(`~/.claude/skills` 是指過去的 symlink)。
**換一台機器要先確認那條 symlink 存在**,否則 pull 到檔案也不會生效。

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
- **機制型改動**(改「東西怎麼運作」而非「業務規則是什麼」,例如全站字級刻度、頁面搜尋):
  不寫 PRD/SA/SD,但**要先跑 `/brainstorm` 產計劃書**——它比純 UI 調整大,直接動手很容易
  做出「能動但方向錯」的東西(介面縮放會動到 `design.md` §3 的全站字級刻度就是例子)。
  三條判準見 §3。⚠️ **一旦要新增資料表或維護介面,就掉回下一條「新功能」,從 PRD 開始。**
- **新功能 / 新 API / 改資料結構 / 改權限規則**:走完整流程,從 PRD 開始。
- **涉及個資、金流、權限、SSO 的任何改動**:一定要走完整流程,且 SA/SD 必須補資安欄位(見 01/02/03 模板)。

## 6. 什麼時候要重新讀這份文件

- 每次要開始一個「新功能」而不是單純改畫面時。
- 新人加入團隊時,作為第一份要讀的文件。
- 每季或每次流程卡住時,回來檢討這份流程本身要不要調整。

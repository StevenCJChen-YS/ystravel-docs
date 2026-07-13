# Ystravel 設計系統 — Foundation（全公司共用）

| 項目 | 內容 |
|---|---|
| 文件性質 | 跨系統設計系統來源（source of truth） |
| 適用範圍 | Ystravel-AuthPortal、Ystravel-CRM-Frontend、未來 EIP 前端 |
| 技術 | NuxtUI v4 + Tailwind CSS v4 |
| 狀態 | 草稿（2026-07-08；2026-07-12 對齊 07-11 拍板：主色 Auth=Blue、灰階 gray/zinc 混搭、圓角尺寸階梯、深色表頭） |
| 前身 | 抽自 `Ystravel-AuthPortal/docs/AUTHPORTAL_UI_FOUNDATION.md`（2026-07-06，已相當成熟），升級為全公司共用版並加上「各系統主題層」 |

---

## 0. 這份文件是什麼

你**不是從零開始**——AuthPortal 已經有一套成熟的 UI foundation。這份文件做兩件事：
1. 把那套 foundation **抽成全公司共用的基礎**，讓 CRM、EIP 都沿用同一套底。
2. 加上「**各系統不同主色**」的主題層（你 2026-07-08 的決定）。

**核心觀念：設計 Token 分兩層**
- **Base（共用底層）**：字體、間距、圓角、灰階、語意色（成功/警告/錯誤/資訊）、密度、按鈕階層 → **全公司一致**。
- **Theme（各系統主題層）**：只有 `primary` 主色每個系統覆寫 → **各系統不同**。

> 業界（Shopify、Atlassian、Material）都是這樣：共用一套底，各產品換主色。

---

## 1. 各系統主色（Theme 層，2026-07-10 修訂）

| 系統 | primary 主色 |
|---|---|
| **AuthPortal** | **Blue** |
| **EIP**（未來） | **Teal** |
| **CRM** | **Violet** |

> 2026-07-12 修訂：回到 2026-07-08 原定案 **Auth=Blue / EIP=Teal / CRM=Violet**。07-10 曾因 AuthPortal 實作全採 teal 而「現實勝出」翻成 Auth=Teal，**07-11 Steven 再拍板翻回 Auth=Blue**（後台已實作 `b70a73c`）。⚠️ 登入頁 terminal 調色盤與 AuthService 信件尾板目前仍是 teal，屬待跟進的收尾工作（登入頁＝平台門面，色彩收斂另議）。

- 三色在色環上大致等距，最好辨識「我在哪個系統」。
- 三個主色都**避開了語意色**（見 §3），不會跟成功/警告/資訊訊息撞色。
- 每個系統只覆寫 `primary`，其餘全部沿用 Base。

---

## 2. Base — Typography（共用）

- **字體（2026-07-13 定案，推翻 07-11「自架 Noto 為主」）**：`--font-sans: "Microsoft JhengHei", "微軟正黑體", "PingFang TC", "Noto Sans TC Variable", "Noto Sans TC", "Segoe UI", ui-sans-serif, system-ui, sans-serif`——**全系統字**：Win 微軟正黑體、Mac 蘋方 PingFang TC、非 Win/Mac 自架 Noto 保底，**主字不用自架 webfont**。原因：自架 webfont（Noto／台北黑體／Inter）在 Windows 無 hinting 都糊，系統字有 hinting 最清晰（2026-07-13 完整試錯一輪驗證：微軟正黑體清晰、台北黑體/Inter 皆糊）。演進：靠系統字 → 自架 Noto（07-11）→ **回全系統字（07-13）**。mono：`"JetBrains Mono", "Cascadia Code", "SFMono-Regular", Consolas, monospace`。
- **文字層級**（內部系統以穩定可掃讀為優先，body 不隨斷點頻繁變動）：

| 用途 | 大小 |
|---|---|
| Page title | mobile `text-xl`，`sm+` `text-2xl` |
| Section title | mobile `text-base`，`lg+` `text-lg` |
| Body / Help | 固定 `text-sm` |
| Meta / code / timestamp | 固定 `text-xs` |

### 2.1 響應式縮放原則（2026-07-09 視覺巡檢定案）

- **大字才縮、小字不縮**：大標題（28px+）小螢幕可降一~兩階；body/說明（14~16px）與小字（12px）已在可讀性底線，**不隨斷點縮小**。
- **文字灰階（語意 token → neutral 階；2026-07-12 對齊 gray 實際值）**——一律用 semantic token，下表 light 值供對照：

| 階層 | semantic token | neutral 階 | light 值（gray） | 用途 |
|---|---|---|---|---|
| 主文字 | `text-highlighted` | neutral-900 | `#101828` | 標題 |
| 標籤/內文 | `text-default` | neutral-700 | `#364153` | 欄位標籤、內文 |
| 說明 | `text-toned` | neutral-600 | `#4a5565` | 輔助句 |
| 淡化 | `text-muted` | neutral-500 | `#6a7282` | placeholder、次要連結、提示 |
| 更淡 | `text-dimmed` | neutral-400 | `#99a1af` | 最弱提示、角標 |

- **灰階定案（2026-07-12 對齊，取代先前 slate）＝明暗混搭**：**light 用 gray**（帶微藍調）、**dark 用 zinc**（無彩度）。全案色階統一走 `neutral-*` 別名單一入口——light 家族由 `vite.config` `neutral: 'gray'` 管、dark 家族由 `main.css` 的 `.dark` 重指整組 oklch 管，換家族只動一處。演進：slate（07-09）→ zinc → gray → 明暗混搭定案。上表 light 值為 gray 家族；**dark 由 semantic token 自動翻低階變亮＋改 zinc 家族**（`main.css .dark` 寫死 zinc oklch，`--color-neutral-850` dark=`#1c1c1e`／light=`#172030`），不用另鎖 dark hex。

### 2.2 字重（2026-07-13 定案）

- **系統字只有兩檔粗細**：微軟正黑體 400/700（Light 300 不用於中文後台）、Mac 蘋方 100–600（**無 700**），跨平台實務交集＝**400（細）／700（粗）**。
- **CSS 字重匹配有方向**（關鍵，別誤判）：`font-medium`(500) 往**下**靠 **400**（細）、`font-semibold`(600) 往**上**靠 **700**（粗）。所以 `semibold` 在系統字會匹配到實體 700 而變粗，`medium` 則等同 400。
- **規則**：**標題／需強調 → `font-semibold`；一般內容 → `font-normal`；避免 `font-medium`**（medium 對系統字＝400、視覺無效；若字體堆疊混入西文 webfont〔如 Inter〕，medium 還會造成同排「英文粗、中文細」的斷層）。
- 層次不只靠字重：善用字級（§2 表）＋文字灰階（§2.1 表）＋間距（§4）；後台 400/700 兩檔已足夠（EIP 用微軟正黑體亦僅此兩檔）。

---

## 3. Base — 色彩（共用語意色）

### 3.1 語意色盤（各系統一致，**不覆寫**）
| 語意 | 色 |
|---|---|
| secondary | sky |
| success | emerald |
| info | sky |
| warning | amber |
| error | rose |
| neutral | gray（light）／zinc（dark）— 見 §2.1 |

> ⚠️ 各系統的 `primary` 主色（Blue/Teal/Violet）**不得使用上面任何一個**，否則主色會跟語意訊息撞色。

### 3.2 中性底色分層（不只白底一層，2026-07-10 修訂）

| Token | 用途 |
|---|---|
| `bg-default` | sidebar、header、**內容區的 card/panel（白卡）** |
| `bg-muted` | hover、次層背景 |
| `bg-elevated` | **白卡上的次層帶**（卡片 header 底） |
| `bg-accented` | 選取態、active nav、當前項目 |

> 表格 thead 另採**深色表頭白字**（2026-07-11 拍板，AuthPortal TableCard）：light `neutral-700`、dark `neutral-850`；斑馬紋偶列 `bg-muted`（dark neutral-800）、奇列走卡底 `neutral-900`——不走 `bg-elevated`。

> ⚠️ 2026-07-10 實測發現：NuxtUI light mode 的 `bg-elevated` 與 slate-100 頁面底色**同值**，卡片用 `bg-elevated` 會糊進頁底。定案層次＝**頁面底 muted 灰（shell main）→ 卡片 `bg-default` 白 → 卡內次層帶 `bg-elevated`**；dark mode 同一組 token 自動成立（實測卡片比頁底亮一階）。

### 3.3 文字與邊框語意
- 文字：`text-highlighted`（標題）／`text-default`（內文）／`text-toned`（中層）／`text-muted`（輔助）
- 邊框：`border-muted`（一般分隔/card）／`border-toned`（需更明顯）／`border-accented`（強調/選取）

---

## 4. Base — 其他基礎（共用）

- **圓角（2026-07-12 對齊）**：基準 `--ui-radius: 0.375rem`（sm=6 / md=9 / lg=12px；卡片與 modal 用 lg=12px）。**控件（btn/input/select）另走尺寸階梯**（vite.config size variants）：`xs=2 / sm=4 / md=6 / lg=8 / xl=10px`。演進：0.5rem（肥）→ 0.25（太尖）→ **0.375rem**（現行，Steven 驗收中）＋控件尺寸階梯。
- **斷點**：base `<640` / sm `≥640` / md `≥768` / lg `≥1024` / xl `≥1280` / 2xl `≥1536`
- **Layout tokens**：`--app-sidebar-width: 240px`、`--app-sidebar-collapsed-width: 4rem`、`--app-header-height: 70px`
- **密度**（各系統可依任務調整）：
  - Login：`xl`
  - Auth Admin：mobile/tablet `lg`、desktop `md`、row action `sm`
  - CRM：compact，desktop 表單/filter/table 以 `sm`/`md` 為主，不複製 login 的 `xl`
- **按鈕階層**：
  - Primary page action：`color="primary" variant="solid"`
  - Secondary：`color="neutral" variant="outline"`
  - Toolbar/utility：`color="neutral" variant="ghost"`
  - Filled neutral：`color="neutral" variant="soft"`（`neutral solid` 會近黑，避免當一般灰按鈕）
  - Risky：`warning soft`；Destructive：`error soft`/`outline`
  - 同一 view 只保留一個最強 primary。
- **間距節奏（8 點網格，2026-07-09 定案）**：
  - 所有間距用 4/8 的倍數（8、16、24、32…），不用 10、18、22 這類亂數。
  - 親疏原則：**群組內 8px、群組間 24px**——用距離表達「誰跟誰一組」。
  - 響應式：小螢幕收攏**結構性留白**（頁面外距 24→16、卡片內距 32→20~24）；**節奏性間距（8/24）不縮**，縮了層次就亂。

### 4.1 表單輸入規範（2026-07-09 定案，AuthPortal 已落地於 vite.config input theme）

| 規則 | 值 | 原因 |
|---|---|---|
| 輸入框字級 | **手機（<640px）16px、桌面 14px** | iOS 點擊 <16px 的輸入框會觸發整頁自動放大；桌面 14px 對齊 compact 表單密度 |
| placeholder | 手機縮小為 `text-sm`（14px）顯示 | FB 同款技巧：iOS 縮放只看輸入文字字級，placeholder 可獨立縮小，視覺不笨重 |
| 無標籤欄位 | 必須有欄位外說明文字撐住語意 | 標籤與 placeholder 不可同時缺席 |
| 欄位標籤/說明/錯誤字級 | **固定 14px（text-sm），不隨 input size 放大** | 16px 底線只限輸入控件；NuxtUI formField `xl` 預設會讓標籤繼承容器 16px，已在 vite.config formField theme 鎖回（2026-07-10 登入頁實測發現） |
| 適用範圍 | UInput `md`/`lg`；login 的 `xl` 為品牌特例不套 | 見 §6 |
| 16px 底線範圍 | **只限輸入控件**（input/select/textarea） | 表格、標籤、說明等顯示文字不受限，手機照用 12~14px |

---

## 5. Light / Dark Mode（共用）

- **兩個模式一開始就一起做**，禁止新增只支援 light mode 的硬寫色。
- 顏色偏好共用 storage key：`ystravel.platform.color-mode`（Auth/CRM/EIP 共用）。
- 新頁面優先用 semantic class + NuxtUI color/variant，不硬寫 `bg-white`/`text-slate-*`。
- **主色的 dark mode**：主色用中間調（如 blue/teal/violet 的 500/600），深色模式自動對應到淺一階（如 400），明暗兩模式對比都足夠——這也是當初不選 sky 當主色的原因（sky 太淺，暗色模式會刺眼/糊）。
- **dark mode 浮層規則（2026-07-13 定案）**：dark 下陰影幾乎看不見 → 上層元素（modal／popover／各下拉 content／表格卡）改用 **`ring-1 dark:ring-white/25`（hairline）＋深色明度分層** 製造「在上層」感，**不靠 `shadow`**。明度階梯：頁底 `neutral-950`（最深）< 卡／modal／側欄 `850` < 表格內回 `950`、斑馬偶列 `900`；輸入框 dark 底壓到 `950`。switch thumb dark 白（`dark:bg-white`）。已抽到各 repo `vite.config` 的 modal／select／selectMenu／dropdownMenu theme 的 content slot，新元件自動套用。

---

## 6. 例外：LoginPage 品牌入口頁

- `LoginPage` 是**特殊品牌入口**，可保留較強的品牌視覺與頁面局部 CSS variable（現有 6 色 palette），**不當作內頁模板**。
- Auth Admin、CRM、EIP 的內頁一律走本文件的 semantic token，不沿用 LoginPage 的 palette。

---

## 7. 各 repo 怎麼套用（實作位置）

| 用途 | 位置 |
|---|---|
| 全站 NuxtUI theme 覆寫（含 primary） | 各 repo `vite.config.ts` → `ui({ ui: { colors: { primary: '...' }, ... } })` |
| 全域 CSS（字體、icon 線條、layout token） | 各 repo `src/assets/css/main.css` 的 `@theme` / `:root` |
| 頁面局部 override | `.vue` 的 `size` / `ui` prop / `class` |

- **共用底層**（字體、語意色、密度、按鈕階層…）三個 repo 應保持一致。
- **主題層**只有 `primary` 每個 repo 不同（Auth=Blue / CRM=Violet / EIP=Teal）。
- 目前**不改任何現有 code**；本文件先作為規範，實際套用在各系統做 UI 時逐步落實。

---

## 8. 與既有文件的關係

- 本文件 = 全公司**共用來源**（Base + Theme 決策）。
- `Ystravel-AuthPortal/docs/AUTHPORTAL_UI_FOUNDATION.md` = AuthPortal 的**詳細實作規範**（shell、shared 元件、drawer、dropdown 等細節），持續有效，作為本文件的參考實作。
- `docs/reference/AUTHPORTAL_NUXTUI_COMPONENT_SCALE.md` = `UButton` 等元件尺寸客製的技術背景。
- CRM / EIP 未來各自可有 `docs/<system>-ui.md` 記錄自己的延伸，但**不得牴觸本文件的 Base**。

---

## 9. 待決策 / 待辦
- [ ] 三系統 primary 的**確切色階**（例如 blue-600 或 blue-500 當亮色主色）——做 UI 時實測 light/dark 對比後定。
- [ ] 是否需要把 Base token 做成一個共用 npm 套件 / 共用 CSS 檔，讓三 repo 真正共用而非各自複製（現階段先文件對齊，未來重複痛了再抽套件）。

---

## 10. 資料表格與順序管理範式（2026-07-14，AuthPortal 已落地）

管理後台的資料表格與「顯示順序」維護，抽成一套共用範式，CRM／EIP 沿用（AuthPortal `src/shared/ui`＋`src/shared/composables`）。

### 10.1 資料表格共用
- **表頭外觀抽全域**：深色表頭白字／列高／內距寫在各 repo `vite.config` 的 **`table` theme**（不再逐頁寫 `:ui`）。要改表頭底色只動這一處。
- **共用元件/組合式**：`TableCard`（白卡＋斑馬＋分頁 footer）、`TableLoading`（載入 spinner）、`FormModal`（valibot 行內驗證，關閉延後清狀態避免收場閃爍→`useEditModal`）、`ColumnVisibilityMenu`＋`useResponsiveColumns`（欄位顯示下拉＋響應式自動藏欄）、`DataToolbar`（搜尋＋篩選＋動作）。
- 列表頁一律：`PageHeader → TableCard → DataToolbar → 深色表頭 UTable`；回饋走 toast、危險/切換動作走 `useConfirm`。

### 10.2 順序（order）管理
- **順序是主檔自己的屬性**，在該主檔管理頁維護（業界標準：Salesforce picklist／Odoo sequence／Shopify／WordPress）；存一份、所有 consumer（下拉／報表）讀同一份，**不要每個表單各自定義順序**。
- **⚠️ 名詞**：「排序」中文有兩義——sort（升降冪）vs order（排列序位）。本節指**後者**。
- **三原則**：
  1. `order` 一律「**相對同層兄弟**」（部門＝同公司內、選項＝同類別內；未來多層分類樹同理），不用全域數字。
  2. UI 用**拖曳**（共用 `OrderManager.vue`＝VueUse `useSortable` 握把拖＋「依名稱排序」一鍵＋**批次儲存/取消**，非邊拖邊存）。
  3. 建立/編輯 modal **不放** order 數字欄，新增自動排到最後。
- **數字給人看一律 1-based（`order+1`）**，0-based 只留程式內。
- **後端**：reorder 端點收「排好的 id 陣列」→交易重編 0,1,2…＋audit。
- **未做／未來**：多層分類樹（同原則延伸樹狀拖曳）；持久「自動/手動 sort mode」（現以「依名稱排序」一鍵 bake 字母序代替，Shopify/Salesforce 式）。

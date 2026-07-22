# Ystravel 設計系統 — Foundation（全公司共用）

> # ⛔ 本檔已停止維護（2026-07-22）
>
> 內容已整併進 **[design.md](./design.md)＝設計系統唯一權威規範書**。
> **查規則請看 design.md，不要用本檔。** 本檔保留僅為考古與既有連結不斷，內容可能已過時。
> 決策時間線與踩坑經過見 [ui-conventions.md](./ui-conventions.md)。

| 項目 | 內容 |
|---|---|
| 文件性質 | 跨系統設計系統來源（source of truth） |
| 適用範圍 | `ystravel-platform`（`apps/portal` 及未來各模組前端）；歷史沿用：Ystravel-AuthPortal、Ystravel-CRM-Frontend |
| 技術 | NuxtUI v4 + Tailwind CSS v4 |
| 狀態 | 草稿（2026-07-08；2026-07-14 對齊：主色 **Auth=Teal**、語意色明暗色階 light 深一階(primary/warning 自訂 550、其餘 600)/dark 400、灰階 gray/zinc 混搭、圓角尺寸階梯、深色表頭） |
| 前身 | 抽自 [authportal-ui-foundation.md](./authportal-ui-foundation.md)（2026-07-06，已相當成熟；原生於 AuthPortal，2026-07-16 遷入 docs），升級為全公司共用版並加上「各系統主題層」 |

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
| **AuthPortal** | **Teal** |
| **EIP**（未來） | 待定（原暫定 Teal，與 Auth 撞色，另議） |
| **CRM** | **Violet** |

> **2026-07-14 定案：Auth = Teal**（Steven 拍板，PR #23）。理由：登入頁 terminal 調色盤與 AuthService 信件模板本就是 teal，App 主色改 teal 反而全平台一致（不再有「後台藍、門面 teal」的分裂）。
> 沿革：07-08 原定 Auth=Blue → 07-10 現實勝出翻 Teal → 07-11 翻回 Blue → **07-14 再定回 Teal**（這次連信件/登入頁一起收斂，不再是暫時狀態）。
> ⚠️ EIP 原暫定 Teal 會與 Auth 撞色，未來實作前再定；CRM=Violet 不變。**各系統主色後續各自定，改一行 `vite.config` `primary` 別名即可**（全站走 semantic token）。

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

> ⚠️ 各系統的 `primary` 主色（Auth=Teal／CRM=Violet／EIP 待定）**不得使用上面任何一個語意色**，否則主色會跟語意訊息撞色。

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
- **語意色明暗色階策略（2026-07-14 定案，AuthPortal 落地）**：同一語意色在明暗用**不同階**（Radix/Material 式）——白底對比偏低要**深一階**、深底要**亮一階**才跳得出來。
  - **語意變數 `--ui-*` light 深一階、dark = 400**（Nuxt UI 預設 light=500 白底偏淡）。實作＝`main.css` 未分層覆寫 `:root`（light）＋ `.dark`（補回 -400，抵銷未分層洩漏）。影響：`text-primary`／subtle・soft 標籤／focus ring／**outline 主鈕**。
    - **primary(teal)／warning(amber)：light = 自訂 550 半階**（500 偏亮、600 偏暗，取 oklch 中間值）；success(emerald)／secondary／info(sky)：light = 600；error(rose) 維持預設。
    - **自訂 550 半階做法**（Nuxt UI `--ui-color-*` 別名只內建 50–950）：`@theme` 定 `--color-teal-550`/`--color-amber-550`（oklch 插值）→ 接 `--color-primary-550`/`--color-warning-550`（讓 `bg-primary-550` utility 生成）＋ `:root` 補 `--ui-color-primary-550`/`--ui-color-warning-550`（讓 `--ui-primary` 可引用）。**兩套別名都要接**。
  - **例外（走固定色階、明暗一致，不隨上面變）**：**solid 主鈕**＝`bg-primary-550` hover/active 600/700（明暗都 550）；**switch checked**＝`bg-primary-500`（明暗都 500）。固定色階明暗同值＝明暗一致，寫在 `vite.config` button／switch theme。
  - **表格內資訊標籤用 `info`（sky）不用品牌 `primary`**：主色改 teal 後，表格裡的「系統／appCode」等資訊 tag 若用 primary 會變品牌色搶眼 → 改 `color="info"`（藍＝資訊語意）。品牌 primary 留給主要動作。
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
- **主題層**只有 `primary` 每個 repo 不同（Auth=Teal / CRM=Violet / EIP 待定）。
- 目前**不改任何現有 code**；本文件先作為規範，實際套用在各系統做 UI 時逐步落實。

---

## 8. 與既有文件的關係

- 本文件 = 全公司**共用來源**（Base + Theme 決策）。
- [authportal-ui-foundation.md](./authportal-ui-foundation.md) = **詳細實作規範**（shell、shared 元件、drawer、dropdown 等細節），持續有效，作為本文件的參考實作（原生於 AuthPortal，2026-07-16 遷入 docs 共通層；路徑對應平台 `apps/portal/src/...`）。
- `docs/reference/AUTHPORTAL_NUXTUI_COMPONENT_SCALE.md` = `UButton` 等元件尺寸客製的技術背景。
- CRM / EIP 未來各自可有 `docs/<system>-ui.md` 記錄自己的延伸，但**不得牴觸本文件的 Base**。

---

## 9. 待決策 / 待辦
- [x] ~~三系統 primary 的確切色階~~ → Auth 定案：語意 light-600/dark-400、solid 鈕固定 600、switch 500（見 §5）。CRM/EIP 沿用同策略、各自定 primary 家族。
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
- **UI＝modal**（2026-07-14 二次改版）：工具列「調整順序」鈕開 `OrderManager` **modal**（比照建立/編輯 modal 慣例，不再整頁切換）；清單窄單欄（`max-w-2xl`）、可捲、序號 1-based 藍底小徽章；清單空時顯示空狀態。⚠️`useSortable` 要加 `watchElement: true`（modal body 晚掛載，預設只 mount 初始化一次會撲空、拖曳失效）。
- **未做／未來**：多層分類樹（同原則延伸樹狀拖曳）；持久「自動/手動 sort mode」（現以「依名稱排序」一鍵 bake 字母序代替，Shopify/Salesforce 式）。

---

## 11. 共用 UI 元件庫與 RWD 範式（2026-07-14，AuthPortal 管理中心巡檢定案）

「看到會重複就當下抽」的落地成果（`src/shared/ui`）；CRM／EIP 沿用。

### 11.0 icon-only 按鈕通則（2026-07-14 一天踩兩次，嚴格執行）

RWD 收成 icon-only 的按鈕，藏字之後**問三件事：padding 等距了嗎？置中了嗎？aria-label 還在嗎？**

1. **藏字用 `max-sm:hidden`**＋`aria-label`/`title` 保留全文；**不要**用 `max-md:size-10` 硬縮還留著文字（會擠爆）。
2. **文字藏了 ≠ 正方**：按鈕仍留文字鈕的左右 padding（px-3 等）→ 橫長方形。要補**等距 padding**：md 尺寸鈕＝`max-sm:p-2.5`（10+16+10＝36×36，同全站 square md 規格）。實例：群組頁「加入成員」鈕。
3. **固定尺寸放大觸控目標**（如 `max-md:size-10`）時必須同時 `:ui="{ base: 'justify-center p-0' }"`——UButton 預設非置中，固定尺寸＋原 padding 會讓 icon 偏離中心。實例：`ColumnVisibilityMenu`。

### 11.1 新增共用元件
- **`ToolbarButton`**：工具列／頁首動作鈕（新增 XX、調整順序…）。桌面 icon＋文字，**`<sm` 收成 40×40 icon-only 正方**（照 §11.0 通則）。可換 `color`/`variant`（新增＝primary solid、調整順序＝neutral outline）。
- **`EmptyState`**：空狀態。`UEmpty variant="naked"`＋`border-dashed`（官方 outline/subtle 的框是 `ring`＝box-shadow，**做不了虛線**，naked＋class 疊）。`size` 對應 UEmpty 檔位：`sm`＝標題14/說明12/icon 縮小（窄欄用，內距鎖 24px 不隨螢幕長到 32），`md`＝大區塊。
- **`FilterPanel`**（曾叫 `FilterDrawer`→`FilterModal`）：篩選收合殼，**RWD 分流：`<sm` 底部 `UDrawer`／`≥sm` `UModal`**（`useBreakpoints` 以 sm 為界）。「篩選 (N)」鈕＋footer 重置/取消/套用。兩種鈕模式：預設（鈕只 `<md` 出現，桌面平鋪下拉）、`alwaysVisible`（鈕常駐，搜尋為主頁用）。草稿流程由頁面持有：`@open` 拷真值進草稿、`@apply` 寫回真值。
  - ⚠️ **vaul（UDrawer）踩坑**：抽屜開啟殘留 `translate3d`→帶 transform 自成 GPU 合成層，**Windows 顯示縮放 125/150% 下文字糊**。但那只是「桌面 devtools 模擬手機」的假象、**真機正常**，故 `<sm` 用抽屜可接受。（曾一度全改 modal 避開，Steven 後來要手機回抽屜。別再為此加 `transform:none` 全域 hack。）
  - **dark 浮起 ring**：UDrawer 預設 `ring-default` 在 dark 看不見 → content 加 `dark:ring-white/25`（同 §5 浮層規則）。modal 端由全域 modal theme 自帶。
- **`ListItemButton`**：左側清單項（群組清單／選項類別清單同款）。選中＝側欄同款灰 pill（`bg-elevated`＋深字），非選中 hover 半透明灰；**別用藍底**。內容排版由 slot 自理。
- **`ListPanel`**（2026-07-15）：master-detail **左欄清單殼**（群組頁/選項頁用，CRM 同款頁沿用）。桌面 `lg:sticky top-6`＋`max-h calc(100vh-8rem)` 清單自己捲（不再整頁無限長）；手機不 sticky、清單 max-h 50vh；`filterable` 開頂部過濾框（過濾邏輯由頁面自持，v-model:filter）；px-3 py-2、項目間 space-y-1。量大靠搜尋不靠捲；幾百筆以上才考慮虛擬捲動。
- **`TableSortButton`**：深色表頭的可排序表頭鈕。補 `font-semibold`——**button theme 全域 `font-normal`（中英字重一致）會蓋掉 `table` theme 的 `th` 字重**，可排序欄不補會比純文字表頭細一截。

### 11.2 RWD 範式
- **列表頁篩選收合**：即時篩選頁（使用者/角色）＝桌面平鋪下拉、`<md` 收 `FilterPanel`（草稿、套用才寫回真值）；**搜尋為主＋低頻條件頁**（稽核）＝左模糊搜尋（debounce 即時查）＋右上 `FilterPanel alwaysVisible`（動作/日期收面板、套用才查），**無查詢/清除鈕**。這是「資料量大、欄位多」頁的標準範式（CRM 客戶/訂單沿用）。面板本身 `<sm` 抽屜／`≥sm` modal（見 §11.1）。
- **`DataToolbar` inline 模式**：動作少的頁（組織/部門）小螢幕也單列（搜尋 `flex-1`＋動作貼右）；動作多（含篩選下拉）維持堆疊。
- **側欄自動收合**：`AppShell` 監看 `xl`(1280)，`<xl` 收 icon rail、`≥xl` 展開（仍可手動 toggle，跨斷點回預設）。
- **RWD 藏欄**：走 `useResponsiveColumns`（別逐欄寫死 `hidden md:table-cell`），搭 `ColumnVisibilityMenu` 可手動加回；同一份 `columnVisibility` 綁 UTable。

### 11.3 踩坑
- **grid 子項橫向溢出**：`lg:grid-cols-[Npx_minmax(0,1fr)]` 這類 grid 在 `<lg` 塌成單欄時，子項預設 `min-width:auto`，內含表格等寬內容會撐破欄寬造成整頁橫向捲動 → **grid 子項（section）補 `min-w-0`**。
- **modal 縮窄**：欄位少的建立/編輯 modal 用 `width-class="sm:max-w-md"`（別用預設 512 太寬）。

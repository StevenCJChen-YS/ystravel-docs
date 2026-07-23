# Ystravel 設計系統規範書（Design System Spec）

> **這份是設計系統的單一權威（single source of truth）。** 要查「現在的規則是什麼」，只看這份。
> 建立：2026-07-22（整併自 `foundation.md`＋`ui-conventions.md`＋`authportal-ui-foundation.md`）

| 項目 | 內容 |
|---|---|
| 適用範圍 | `ystravel-platform`（`apps/portal` 與各模組前端）＋**系統寄出的信件**（`apps/api` 的 mail 模板，見 §11）；未來各系統前端沿用同一套 Base |
| 技術 | Vue 3 + Nuxt UI v4 + Tailwind CSS v4（純 Vue 安裝，不用 Nuxt 框架） |
| 狀態 | 生效中，滾動修訂 |

---

## 0. 怎麼用這份文件

### 0.1 三份文件的分工（改版後）

| 文件 | 回答什麼問題 | 什麼時候看 |
|---|---|---|
| **`design.md`（本檔）** | **現在的規則是什麼？** | 寫任何 UI 之前 |
| [`ui-conventions.md`](./ui-conventions.md) | **為什麼是這條規則？當初踩過什麼坑？** | 想推翻某條規則、或遇到詭異現象時 |
| [`authportal-ui-foundation.md`](./authportal-ui-foundation.md) | （已封存）2026-07 初期的實作規範 | 考古用，**不可作為依據** |

**衝突時的順位**：本檔 > `ui-conventions.md` 最新拍板 > 程式碼現況。
發現本檔與程式碼不一致時，**先確認哪邊是對的再改**，並回頭更新本檔——不要默默讓兩邊繼續分岔。

### 0.2 Token 分兩層

- **Base（共用底層）**：字體、間距、圓角、灰階、語意色、密度、按鈕階層 → **全系統一致**。
- **Theme（各系統主題層）**：只有 `primary` 主色各系統覆寫 → 改一行 `vite.config` 即可。

業界（Shopify、Atlassian、Material）都是這個結構：共用一套底，各產品換主色。

### 0.3 動 UI 前的固定動作

1. 讀本檔對應章節
2. 查 §7 共用元件庫——**有現成的就用，不要重造**
3. 需要調整視覺 → 走 §12 的實作位置（theme seam），**不准 `!important` 或深層選擇器硬蓋**
4. 寫任何頁面前先問：**「第二個模組/頁也會用到嗎？」** 會 → 抽 `shared/ui/`；不會 → 放 `modules/<模組>/components/`

---

## 1. 設計原則

1. **後台導向、穩定可掃讀**：資料用表格（表頭＋分頁 footer）、建立/編輯開 modal 而非行內輸入、列操作用 icon 鈕。使用者熟悉的是 EIP/Vuetify 式後台。
2. **白卡浮灰底**：light mode 的卡片一定是白的，浮在灰底頁面上。
3. **規則定在共用層，不靠各頁記得**：任何「每頁都要記得貼」的規則遲早會漏。看到規則性問題就收斂到元件或 theme，不要在單一頁面貼補丁。
4. **Light / Dark 一起做**：禁止新增只支援 light mode 的硬寫色。
5. **能預防就不報錯**：最好的錯誤訊息是根本不會出現的錯誤（見 §8.3）。
6. **一個 view 只有一個最強 primary**。

---

## 2. 色彩

### 2.1 主色（Theme 層）

| 系統 | primary |
|---|---|
| **平台（Auth／入口網／各模組）** | **Teal** |
| CRM（未來若獨立主色） | Violet |
| EIP（未來） | 待定（不得與平台撞色） |

- 主色**不得使用任何語意色**，否則會與成功/警告/資訊訊息撞色。
- 換主色＝改 `vite.config` 的 `primary` 別名一行，全站走 semantic token 自動連動。

### 2.2 語意色盤（各系統一致，不覆寫）

| 語意 | 色 |
|---|---|
| secondary | sky |
| success | emerald |
| info | sky |
| warning | amber |
| error | rose |
| neutral | **light = gray／dark = zinc**（明暗混搭） |

灰階單一入口＝`neutral-*` 別名：light 家族由 `vite.config` 的 `neutral: 'gray'` 管、dark 家族由 `main.css` 的 `.dark` 重指整組 oklch 管。**換家族只動一處。**

### 2.3 明暗色階策略

**同一語意色在明暗用不同階**——白底對比偏低要**深一階**、深底要**亮一階**才跳得出來。

| 對象 | light | dark |
|---|---|---|
| 語意變數 `--ui-*`（`text-primary`／subtle・soft 標籤／focus ring／outline 主鈕） | primary(teal)、warning(amber) ＝**自訂 550 半階**；success／secondary／info ＝ 600；error(rose) 用預設 | **400** |
| **solid 主鈕**（固定色階，明暗一致） | `bg-primary-550`，hover/active → 600/700 | 同 light |
| **switch checked**（固定色階，明暗一致） | `bg-primary-500` | 同 light |

**自訂 550 半階的做法**（Nuxt UI `--ui-color-*` 別名只內建 50–950）——**兩套別名都要接**，缺一不生效：

```css
@theme {
  --color-teal-550: oklch(65.2% 0.129 183.604);   /* oklch 插值取 500/600 中間 */
  --color-amber-550: oklch(71.75% 0.1835 64.199);
  --color-primary-550: var(--color-teal-550);      /* ① Tailwind utility 用（bg-primary-550） */
  --color-warning-550: var(--color-amber-550);
}
:root {
  --ui-color-primary-550: var(--color-teal-550);   /* ② Nuxt UI --ui-primary 引用用 */
  --ui-color-warning-550: var(--color-amber-550);
}
```

> ⚠️ `--ui-*` 未分層寫在 `:root` 會洩漏到 dark，故 `.dark` 要補回 400 抵銷。

### 2.4 中性底色分層

| Token | 用途 |
|---|---|
| `bg-default` | sidebar、header、**內容區的卡片/面板（白卡）** |
| `bg-muted` | hover、次層背景、頁面底 |
| `bg-elevated` | **白卡上的次層帶**（卡片 header 底） |
| `bg-accented` | 選取態、active nav、當前項目 |

**層次＝頁面底 muted 灰 → 卡片 `bg-default` 白 → 卡內次層帶 `bg-elevated`。**

> ⚠️ **卡片不要用 `bg-elevated`**：light mode 的 `bg-elevated` 與頁面底色同值，卡片會糊進頁底（2026-07-10 實測）。dark mode 同一組 token 自動成立。

**表格 thead 另採深色表頭白字**：light `neutral-700`、dark `neutral-850`；斑馬紋偶列 `bg-muted`（dark `neutral-800`）、奇列走卡底（dark `neutral-900`）——不走 `bg-elevated`。

### 2.5 文字灰階

一律用 semantic token，下表 light 值供對照（dark 由 token 自動翻低階變亮）：

| 階層 | token | neutral 階 | light 值 | 用途 |
|---|---|---|---|---|
| 主文字 | `text-highlighted` | 900 | `#101828` | 標題 |
| 標籤/內文 | `text-default` | 700 | `#364153` | 欄位標籤、內文 |
| 說明 | `text-toned` | 600 | `#4a5565` | 輔助句 |
| 淡化 | `text-muted` | 500 | `#6a7282` | placeholder、次要連結、提示 |
| 更淡 | `text-dimmed` | 400 | `#99a1af` | 最弱提示、角標 |

### 2.6 邊框

`border-muted`（一般分隔線/卡片）／`border-toned`（需更明顯）／`border-accented`（強調/選取）。

---

## 3. 字體排印

### 3.1 字體堆疊

```
--font-sans: "Microsoft JhengHei", "微軟正黑體", "PingFang TC",
             "Noto Sans TC Variable", "Noto Sans TC", "Segoe UI",
             ui-sans-serif, system-ui, sans-serif;
--font-mono: "JetBrains Mono", "Cascadia Code", "SFMono-Regular", Consolas, monospace;
```

**全系統字、零自架中文 webfont**：Win 微軟正黑體、Mac 蘋方，其他平台自架 Noto 保底。

> ⚠️ **不要再提議自架中文 webfont 追求多字重**。自架 webfont（Noto／台北黑體／Inter）在 Windows 無 hinting 一律糊，系統字有 hinting 最清晰（2026-07-13 完整試錯驗證）。

### 3.2 字級

| 用途 | 大小 |
|---|---|
| Page title | mobile `text-xl`，`sm+` `text-2xl` |
| Section title | mobile `text-base`，`lg+` `text-lg` |
| 卡片內區塊標題 | `text-[15px] font-semibold`（帳號中心慣例） |
| **Body / Help / 定義列** | **固定 `text-sm`** |
| Meta / code / timestamp | 固定 `text-xs` |

**密集資訊流條目**（通知收件匣等「一格三行」的 feed 型 UI，2026-07-23 新增）：

| 條目層 | 大小 |
|---|---|
| 條目標題（類別，如「人事異動通知」） | `text-sm font-semibold` |
| 條目內文（事實句） | `text-[13px]` |
| 條目時間戳 | `text-xs` |

> 13px 只准用在 feed 條目內文這類「同格內要與 15px 標題拉開兩層」的密集場景，
> 不得拿去縮一般 Body（一般內文仍固定 `text-sm`）。

**響應式縮放原則：大字才縮、小字不縮。** 大標題（28px+）小螢幕可降一~兩階；body/說明（14~16px）與小字（12px）已在可讀性底線，**不隨斷點縮小**。

> ⚠️ 唯一放大到 16px 的地方是**手機的輸入控件**（見 §6.2），那是 iOS 限制不是排版選擇。顯示文字不要跟著放大。

### 3.3 字重

**系統中文字只有兩檔粗細**：微軟正黑體 400/700、蘋方無 700，跨平台交集＝ **400（細）／700（粗）**。

**CSS 字重匹配有方向**（關鍵，別誤判）：`font-medium`(500) 往**下**靠 400、`font-semibold`(600) 往**上**靠 700。

**規則：標題／需強調 → `font-semibold`；一般內容 → `font-normal`；避免 `font-medium`。**
（medium 對系統字＝400 視覺無效；混入西文 webfont 時還會造成同排「英文粗、中文細」的斷層。）

層次不只靠字重——善用字級（§3.2）＋文字灰階（§2.5）＋間距（§4.1）。

---

## 4. 間距、圓角、斷點、密度

### 4.1 間距（8 點網格）

- 所有間距用 **4/8 的倍數**（8、16、24、32…），不用 10、18、22 這類亂數。
- **親疏原則：群組內 8px、群組間 24px**——用距離表達「誰跟誰一組」。
- 響應式：小螢幕收攏**結構性留白**（頁面外距 24→16、卡片內距 32→20~24）；**節奏性間距（8/24）不縮**，縮了層次就亂。

### 4.2 圓角

- 基準 `--ui-radius: 0.375rem`（sm=6／md=9／lg=12px）；**卡片與 modal 用 lg=12px**。
- **控件另走尺寸階梯**（`vite.config` size variants）：`xs=2 / sm=4 / md=6 / lg=8 / xl=10px`。

### 4.3 斷點

`base <640` / `sm ≥640` / `md ≥768` / `lg ≥1024` / `xl ≥1280` / `2xl ≥1536`

### 4.4 Layout tokens

```css
--app-sidebar-width: 240px;
--app-sidebar-collapsed-width: 4rem;
--app-header-height: 70px;
--ui-container: 84rem;
```

### 4.5 密度

| 場景 | 尺寸 |
|---|---|
| 登入頁（品牌特例） | `xl` |
| 管理後台 mobile/tablet | `lg` |
| 管理後台 desktop | `md` |
| 列內操作鈕 | `sm` |
| Badge 預設／代碼型 badge | `sm`／`xs` |

---

## 5. 版面與殼

### 5.1 統一殼（2026-07-17 起唯一結構）

全站住在 **`AppShell`**，沒有第二種殼（`LobbyShell`／`AccountShell`／`AuthAdminShell`／`AuthAccountShell` 皆已退役）。

```
┌──────┬─────────────┬──────────────────────────┐
│ Rail │ 模組側欄      │ 內容區                    │
│（最窄）│（各模組 nav） │（AppPageLayout ＋ 卡片）   │
└──────┴─────────────┴──────────────────────────┘
```

- **`ModuleRail`**（最左窄軌）＝系統切換，依權限顯示。不放首頁鈕（**點 logo 回首頁**）；底部小分隔線後放帳號中心 icon（`railFooter`）。
- **模組側欄**：各模組自帶 `nav.ts`，彙整於 `shared/nav/module-nav.ts`，旗標 `railHidden`／`railFooter`。
- **側欄自動收合**：`<xl`(1280) 收成 icon rail、`≥xl` 展開；仍可手動 toggle，跨斷點回預設。
- **手機＝Discord 式雙欄抽屜**：rail 在 `<lg` 進 slideover，點模組先切右側清單、不跳頁。
- **個人/帳號一律收在頭像選單**（`UserMenu`）——「找自己＝點頭像」全站一致。

### 5.2 頁面骨架

列表頁固定長這樣：

```
PageHeader → TableCard → DataToolbar → 深色表頭 UTable → TablePaginationFooter
```

回饋走 toast；危險／切換類動作走 `useConfirm`。

### 5.3 篩選範式（選錯會改到行為，不只改版面）

**判準是「這頁的篩選是即時查還是套用才查」，不是螢幕寬度。**

| 模式 | 篩選鈕何時出現 | 用在哪 | 前提 |
|---|---|---|---|
| 預設 | `<md` | 條件 ≤4 的即時篩選頁（使用者、角色） | 平鋪版在 `≥md` 排得下 |
| `inline-from="2xl"` | `<2xl` | 條件 5 個的即時篩選頁（員工管理） | 平鋪版要 1536 才排得下 |
| `always-visible` | 所有寬度 | 搜尋為主、**套用才查**的頁（稽核） | 沒有平鋪版 |

- 前兩種是同一件事的兩個斷點（平鋪版存在、只是排不排得下）；`always-visible` 是另一回事。
- **不要把 `inline-from` 套到「套用才查」的頁**——平鋪版沒有套用鈕，等於每動一個下拉就掃一次大表。
- `FilterPanel` 本身 RWD 分流：`<sm` 底部 `UDrawer`／`≥sm` `UModal`。草稿流程由頁面持有：`@open` 拷真值進草稿、`@apply` 才寫回。
- **動作鈕貼著表格，不要放到很遠的頁首**（比照 Vuetify data-table CRUD，而非 Polaris 頁首 primaryAction）。

**條件多到排不下時，先量再決定斷點**——用 `getBoundingClientRect` 量工具列那一列的寬度與 `top`（`top` 出現兩種值＝已經換行了），別憑感覺挑。

### 5.4 表格

- **外觀一律抽 `vite.config` 的 `table` theme**（深色表頭、列高、內距），不逐頁寫 `:ui`。
- **欄寬＝auto 佈局＋百分比 th**（名稱/說明欄不設寬、拿剩餘），**勿加 `table-fixed`**——會鎖死 100%，多開欄位互相硬擠疊字。配 `TableCard` 的 `overflow-x-auto` 自然橫向捲動。
- **RWD 藏欄走 `useResponsiveColumns`**，別逐欄寫死 `hidden md:table-cell`；搭 `ColumnVisibilityMenu` 可手動加回，同一份 `columnVisibility` 綁 UTable。
- Nuxt UI 的 `UTable` **沒有內建 density prop**，要舒適/精簡切換得自綁兩組 padding。
- 可排序的表頭用 `TableSortButton`（button theme 的全域 `font-normal` 會蓋掉 th 字重，不補會細一截）。

### 5.5 順序（order）管理

> ⚠️ **名詞**：「排序」中文兩義——sort（升降冪）vs **order（排列序位）**。本節指後者。

- **順序是主檔自己的屬性**，在該主檔管理頁維護（Salesforce picklist／Odoo sequence 同理）；存一份、所有 consumer 讀同一份，**不要每個表單各自定義順序**。
- 三原則：①`order` 一律「**相對同層兄弟**」②UI 用**拖曳**（`OrderManager`＝握把拖＋「依名稱排序」一鍵＋**批次儲存/取消**，非邊拖邊存）③建立/編輯 modal **不放** order 數字欄，新增自動排最後。
- **數字給人看一律 1-based**（`order + 1`），0-based 只留程式內。
- 後端 reorder 端點收「排好的 id 陣列」→ 交易重編 0,1,2… ＋ audit。
- UI＝**modal**（工具列「調整順序」鈕開），清單窄單欄 `max-w-2xl`、可捲、序號 1-based 徽章。

---

## 6. 元件規範

### 6.1 按鈕階層

| 用途 | 寫法 |
|---|---|
| 頁面主要動作 | `color="primary" variant="solid"` |
| 次要動作 | `color="neutral" variant="outline"` |
| 工具列／utility | `color="neutral" variant="ghost"` |
| 需要灰色填滿感 | `color="neutral" variant="soft"` |
| 有風險但非破壞性 | `color="warning" variant="soft"` |
| 破壞性 | `color="error" variant="soft"` 或 `outline` |

> ⚠️ `neutral + solid` 會近黑，**不要拿來當一般灰按鈕**。

**icon-only 鈕三問**（藏字之後必問，一天踩兩次的教訓）：

1. **padding 等距了嗎？** 文字藏了但仍留文字鈕的左右 padding → 變橫長方形。md 尺寸鈕補 `max-sm:p-2.5`（36×36，同全站 square md 規格）。
2. **置中了嗎？** 固定尺寸（如 `max-md:size-10`）必須同時 `:ui="{ base: 'justify-center p-0' }"`——UButton 預設非置中。
3. **`aria-label` 還在嗎？** 藏字用 `max-sm:hidden` ＋ 保留 `aria-label`/`title` 全文；**不要**用 `max-md:size-10` 硬縮還留著文字。

### 6.2 表單輸入

| 規則 | 值 | 原因 |
|---|---|---|
| 輸入框字級 | **手機（<640px）16px、桌面 14px** | iOS 點擊 <16px 的輸入框會整頁放大 |
| placeholder | 手機縮為 `text-sm` 顯示 | iOS 縮放只看輸入文字字級，placeholder 可獨立縮小 |
| 欄位標籤/說明/錯誤 | **固定 14px，不隨 input size 放大** | 16px 底線只限輸入控件；formField theme 已鎖回 |
| 16px 底線範圍 | **只限輸入控件**（input/select/textarea） | 表格、標籤、說明等顯示文字不受限 |
| 無標籤欄位 | 必須有欄位外說明文字撐住語意 | 標籤與 placeholder 不可同時缺席 |

- **說明/備註欄**：獨占一列 → `<UTextarea :rows="3" class="w-full" />`；跟單行欄位同排 grid → 維持 `AppInput`（textarea 高度突兀）。
- **必填星號只有「同表單有可選欄位可對比」時才標**。整張表單每欄都必填就不要標（整排星號傳達不了資訊）。**標了就要在 schema 裡真的驗。**
- **placeholder 要舉例，不要複述欄位名**：「科威員編」配「例如：Y001」，不是「請輸入科威員編」。（密碼類無法舉例，屬例外。）

### 6.3 狀態徽章的顏色語義

| 顏色 | 語義 | 用在 |
|---|---|---|
| `success` | 正常運作中 | 啟用、在職 |
| `warning` | **待處理——有人要去追** | 邀請中（還沒設密碼）、留職停薪 |
| `neutral` | 非啟用，但一切正常 | 停用、離職 |
| `error` | **真正的錯誤或破壞性結果** | 匯入失敗、驗證不通過 |

**關鍵判準：`error`（紅）不是「負面」，是「出事了」。**
員工離職導致帳號停用是系統照設計正確運作，標紅會把視線拉到一件完全正常的事上；紅色被這樣用久了，真的出事時就沒人看見。

**這組對照必須有單一來源**＝`apps/portal/src/shared/lib/account-status.ts`，新頁面一律從那裡取（含稽核詳情的前後對照，否則同一件事在列表寫「停用」、稽核寫「已停用」像兩件事）。

### 6.4 Tooltip

**一律 `UTooltip`**，唯一例外＝**輸入框內緣的小圖示鈕**（見 §8.1-4）——浮層會蓋住 26px 的小鈕，滑鼠移過去先開浮層就點不到。例外處要**在原地寫下理由**，否則下一輪巡檢又會有人「順手統一」改回去。

官方主題把 tooltip 的 `text` 設成 `truncate`（**提示被截斷等於沒有用**），全域改掉：

```ts
tooltip: {
  slots: {
    content: 'rounded-[2px] h-auto max-w-xs whitespace-normal',
    text: 'whitespace-normal',
  },
}
```

`h-auto` 不能漏——原主題有固定高度，只放開 `whitespace` 第二行會被切掉。

### 6.5 Icon

- **一律 lucide（`i-lucide-*`），禁止混用其他 icon 套件**——不同套件筆觸/比例不一致，混排一眼突兀。
- lucide 找不到想要的先上 lucide.dev 搜（新 icon 常已收錄），真沒有再帶候選來討論，**不要自行混套件**。
- 裝飾用 emoji 一律換成 lucide icon。
- ⚠️ icon 目前執行期從 Iconify API 抓（斷網會開天窗），內網正式部署前要把 lucide collection 打包進本地。

### 6.6 頭像與圓形裁切

**任何 `rounded-full` + `overflow-hidden` 的容器都必須有背景色；有 `ring` 時背景色要與 ring 同色。**

圓形裁切邊緣是反鋸齒的，那圈半透明像素必須跟底下的東西混色；容器沒背景色就混到後面的內容（頭像跨在深色背景照與白卡交界時，只有上半圓看得見細邊）。

**結論：頭像一律用 `UAvatar`**（根元素本來就帶 `bg-elevated`），不要手刻 `<img class="rounded-full">`。

### 6.7 空狀態與其他

- **`EmptyState`**：`UEmpty variant="naked"` ＋ `border-dashed`（官方 outline/subtle 的框是 `ring`＝box-shadow，做不出虛線）。`sm`＝窄欄用、`md`＝大區塊。
- **空值一律 `—`（em dash）**：`?? '—'` 或 `<span class="text-dimmed">—</span>`，不准出現 `-`／`_`。
- **顯示給使用者的資料用中文名，不露 code**：有 code＋name 的實體一律顯示 name（hover 補英文代碼）。
- **modal 縮窄**：欄位少的建立/編輯 modal 用 `width-class="sm:max-w-md"`，別用預設 512 太寬。

---

## 7. 共用元件庫（`apps/portal/src/shared/`）

**改任何頁面 UI 前先看有沒有現成的，別重造。** import 一律走 `@/shared/ui/<類>/<元件>` 絕對別名。

| 資料夾 | 元件 |
|---|---|
| `ui/base/` | `AppInput` `AppSelect` `AppSelectMenu` `DateInput` `EmailInput` `ToolbarButton` `ColorModeToggle` |
| `ui/table/` | `TableCard` `TableFrame` `TableLoading` `TablePaginationFooter` `TableSortButton` `ColumnVisibilityMenu` `DataToolbar` `FilterPanel` `OrderManager` |
| `ui/layout/` | `AppPageLayout` `PageHeader` `SurfaceCard` `AuroraBackdrop` |
| `ui/nav/` | `ModuleRail` `UserMenu` `ListPanel` `ListItemButton` |
| `ui/overlay/` | `FormModal` `ConfirmModal` |
| `ui/card/` | `SystemAppCard` `UserIdentityCard` |
| `ui/feedback/` | `EmptyState` `PasswordStrengthMeter` |
| `ui/options/` | `OptionsWorkspace` `OptionsManager` `OptionTreeRows` |
| `ui/audit/` | `AuditLogTable` |
| `composables/` | `useConfirm` `useEditModal` `useResponsiveColumns` `useAppTheme` `useAccessibleSystems` |
| `lib/` | `datetime` `account-status` `permission-check` `module-labels` `org-options` `format-employee` `company` `color-mode` `accessible-systems` `profile-backgrounds` |

**元件歸屬判準**：跨模組共用 → `shared/ui/`（依**類型**分子資料夾）；單一模組專屬 → `modules/<模組>/components/`。

**`App*` 包裝只在「補官方元件缺的功能」或「統一 app 慣例」時才做**（如 `AppInput` 的 clearable）。純換名字、對全域 theme 無加值的薄包裝**不做**——theme 已足夠且無重複時，原始 `U*` 可直接用。

---

## 8. 互動與狀態規則

### 8.1 控件狀態四條硬規則

1. **停用的控件不准對滑鼠有任何回應**：`hover`/`active` 一律掛 `not-disabled:`。停用還會變邊框，使用者會以為點得動。（已在 `vite.config` 的 button/input/textarea/select/selectMenu theme 補齊。）
2. **警告色只留給「真正需要行動」的事**：狀態資訊用欄位＋篩選呈現，不要常駐警告。天天顯示不是問題的事，只會訓練使用者無視所有警告。
3. **連動下拉：父層沒選就 disabled，並用 placeholder 講下一步**（「請先選擇公司」）。**同一條規則要同時套到 modal 表單與篩選列**，且父層變更/清空時一併重置子層。
4. **輸入框內緣的小圖示鈕用原生 `title` 不用 `UTooltip`**（見 §6.4），並**一律加 `@mousedown.prevent`**——不擋的話按下瞬間輸入框先失焦，表單會用舊值跑驗證而閃出錯誤。

### 8.2 Dark mode 與浮層

- **dark 下不靠 `shadow` 製造浮起感**（陰影幾乎看不見），改用 **`ring-1 dark:ring-white/25` hairline ＋ 深色明度分層**：頁底 `neutral-950`（最深）< 卡／modal／側欄 `850` < 表格內回 `950`、斑馬偶列 `900`；輸入框 dark 底壓到 `950`。已抽到 `vite.config` 各 content slot，新元件自動套用。
- **【鐵律】dark 裝飾效果不准動到 light**：裝飾層用 `hidden dark:block` 只在 dark 掛載；效果 class 一律 `dark:` 前綴。**改完務必切 light 實測 computed** 才算數。

### 8.3 錯誤顯示三層

> 原則：**能預防就不報錯；報錯盡量貼著欄位；toast 是最後手段**。

1. **預防層（UI 直接擋）**：日曆反灰不可選日期、下拉只列合法值、按鈕 disabled。
2. **前端驗證層 → 欄位下方紅字**：走 `FormModal` 的 `:schema`（valibot）→ UForm 行內驗證。schema 管不到的欄外狀態自寫即時計算函式，紅字（`text-xs text-error`）放欄位正下方＋submit 入口再擋一次。
   - **紅字左緣對齊 input 起點，不是容器左緣**：label 在左的行內佈局用 `col-start-N` 對到 input 那欄——錯誤講的是「這個欄位」；靠容器左緣會像整張卡在報錯。
   - **瀏覽器原生驗證氣泡全站禁用**（`FormModal` 的 UForm 已掛 `novalidate`）：位置不可控、各瀏覽器長相不一，且它搶在 schema 驗證前跑＝行內驗證永遠沒機會顯示。新表單不准依賴原生 `required`/`min`/`pattern`。
3. **後端錯誤層**：
   - **對得到欄位的**（email 已存在、名稱重複）→ 盡量映射回該欄位下方紅字；映射不了才 toast。
   - **對不到欄位的**（500／網路／權限／整批失敗）→ toast（error 色＋`i-lucide-circle-alert`），文案走各 api 層的 `ERROR_MESSAGES` 中文對照表。

### 8.4 RWD 補充

- **分頁 footer**：`<sm` 2 欄 grid（分頁置中橫跨上排、下排「共N筆」左＋「每頁」右）；`≥sm` 一列 `justify-between`。
- **`DataToolbar` inline 模式**：動作少的頁小螢幕也單列（搜尋 `flex-1`＋動作貼右）；動作多維持堆疊。
- **grid 子項要補 `min-w-0`**：`lg:grid-cols-[Npx_minmax(0,1fr)]` 在 `<lg` 塌單欄時，子項預設 `min-width:auto`，內含表格會撐破欄寬造成整頁橫向捲動。

---

## 9. 主題系統（深淺色 ×主題）

**架構**：深淺色「模式」之下各自可選「主題」。深色＝**極光（預設）／預設（純黑）**；淺色＝預設 only。存 localStorage（`ystravel.platform.theme.dark/light`，每台裝置各記；色彩模式另存 `ystravel.platform.color-mode`）。

實作＝`useAppTheme.ts`（watch 掛 html `.theme-aurora` class）＋ `main.css` 的
`@custom-variant aurora (&:where(.dark.theme-aurora, .dark.theme-aurora *))`。

### 9.1 兩條鐵律

1. **保證深色「預設」像素級等同原設計、淺色不變**：元件**只「加」`aurora:` 前綴樣式、絕不改 base class**。未來要加新主題＝新 class＋新 `@custom-variant`，同法炮製——**別回頭硬寫某個 shell**。
2. **別用 `dark:` 做「只有某主題才要」的裝飾**：`dark:` ＝ 所有深色（含純黑）都套，會破壞上面的保證。極光專屬效果一律 `aurora:`。

### 9.2 aurora 色值＝單一 token 區塊

所有毛玻璃/半透明值集中在 `main.css` 的 `.dark.theme-aurora` 區塊，元件一律 `aurora:bg-[var(--aurora-panel)]` 引用，**不准散寫 `neutral-900/38`、`white/8` 這類數字**。

| 群組 | token |
|---|---|
| 表面 | `--aurora-panel` `rail` `overlay` `field` `inset` `header` |
| 狀態 | `--aurora-row-alt` `hover` `hover-soft` `active` |
| 邊框 | `--aurora-border` `border-soft` `divider` |

### 9.3 外殼元件

- **`SurfaceCard`**＝毛玻璃卡片外殼（圓角/邊框/底/陰影＋aurora 玻璃）；`as` 控 section/div、`rounded` prop。
- **`TableFrame`**＝表格內縮帶框＋斑馬紋＋列 hover＋`#footer` slot。`TableCard` = `SurfaceCard` + `TableFrame` + 分頁。
- **`AuroraBackdrop`**＝全視窗星空＋極光光暈固定背景層，**掛在 `App.vue` 根組件**（放 `AppShell` 會每次換頁重掛載、動畫歸零）。

**維護心法**：改 aurora 顏色 → 只動 `main.css` token 區塊；新卡片/表格 → 用 `SurfaceCard`/`TableFrame`/`TableCard`，別貼長字串。

---

## 10. 文案與格式

### 10.1 日期與時間

畫面上一律 **`2026/07/17`（斜線、補零）** 與 **24 小時制 `15:04`**；秒數只給稽核日誌。
**唯一出口＝`shared/lib/datetime.ts`，不要在頁面裡自己寫格式化。**

| 用途 | 函式 | 樣子 |
|---|---|---|
| 純日期欄位（到職日） | `formatDate` | `2026/07/17` |
| 時間戳只要日期那面 | `formatTimestampDate` | `2026/07/17` |
| 時間戳（最後登入） | `formatDateTime` | `2026/07/17 15:04` |
| 稽核日誌 | `formatDateTimeWithSeconds` | `2026/07/17 15:04:05` |
| 表單預填今天、匯出檔名 | `todayIsoDate` | `2026-07-17`（ISO，非顯示用） |

理由：**斜線**貼齊同事的 Excel/科威/政府系統；**補零**是為了對齊（顯示日期的欄位一併加 `tabular-nums`，等寬數字才真的對齊）；**24 小時制**避免「上午 12 點」的歧義；**不用民國年**（排序、比對、匯入都會多一層換算）。

**三條硬規則**（都是踩過的坑）：

1. **資料層維持 ISO，只有送到人眼前的最後一哩才轉。** 資料庫、API、`DateInput` 的 `v-model`、匯出檔名一律 `YYYY-MM-DD`。
2. **「純日期」不准經過 `new Date()`。** `new Date('2023-09-23')` 是 UTC 午夜，換算回當地會變前一天。所以 `formatDate` 與 `formatTimestampDate` 是**兩個不同的函式**。
3. **要「今天」就用 `todayIsoDate()`，不要 `toISOString().slice(0,10)`。** 後者是 UTC 的今天，台灣 00:00–08:00 之間會少一天。後端用 `Intl.DateTimeFormat('en-CA', { timeZone: 'Asia/Taipei' })`（不依賴容器時區）。

> **「存的值 ≠ 使用者選的值」時，換算必須是唯一一份。** 角色到期日存「隔天 00:00」，顯示要換算回「最後有效那天」——曾因兩頁各自換算而差一天。

### 10.2 用字

- **第二人稱一律「您」**（後端信件範本本來就是）。程式註解不受此限。
- 回應與文件輸出使用**繁體中文（台灣用語）**。

---

## 11. 信件版型（Email）

> 信件是**另一個媒介**：後端字串模板、沒有 CSS 變數、沒有 dark mode、客戶端會改寫你的 HTML。
> 本節的規則只管信件，前面章節的 token 與元件規範一律用不上（原因見 §11.5）。
> 2026-07-23 定版，實作在 `apps/api/src/common/mail/`。

### 11.1 四級顏色（頂條＋眉標＋按鈕同一色）

**顏色編碼「收件人該有什麼反應」，不是「哪個模組寄的」**——模組資訊已經由眉標文字表達，
顏色再講一次等於浪費一個維度。

| 頂條色 | 語意 | 對使用者的意思 | 例子 |
|---|---|---|---|
| **teal** `#0d9488` | 週知 | 跟你有關的事發生了，**你不用做什麼** | 人事異動、公告 |
| **藍** `#2563eb` | 待辦 | **需要你動手**完成一個動作 | 密碼重設、邀請開通、待審批 |
| **amber** | 提醒 | 有時限、快來不及了，但**還沒出事** | 審批逾期、連結即將過期、額度將滿 |
| **rose** | 警示 | **已經出事**，或涉及帳號安全 | 異常登入、權限遭變更、匯入失敗 |

三條規則：

1. **一封信只有一個顏色**：頂條、眉標、按鈕、內文連結全部同色。
2. **rose 要稀有。** 什麼壞事都用紅色＝紅色不再有意義。判準是「使用者**忽略**這封信會不會有實質損失或安全風險」——不會就降級 amber。
3. **顏色不是唯一線索。** 色盲、以及部分客戶端會改寫色彩，所以**眉標文字必須單獨就講得清楚**，不能靠顏色補語意。

> amber 與 rose 尚無真實信件使用，色值等第一封落地時比照 §2.2 的 warning／error 家族定。

### 11.2 版型骨架

三封信（通知信、密碼重設、邀請開通）共用同一組結構與數值，只有主色不同。

| 區塊 | 規格 |
|---|---|
| 外層 body | `padding:24px`、底色 `#f4f5f7` |
| 卡片 | `max-width:440px`、白底、圓角 12、`overflow:hidden` |
| 頂條 | 高 4px，主色 |
| 卡片內距 | `28px 32px 32px` |
| 眉標 | 14px／700／`letter-spacing:.12em`／主色／下方 16px |
| 標題 h1 | 18px／`line-height:1.5`／`#101828`／`margin:0` |
| 內文 | 14px／`line-height:1.7`／`#4a5565`；第一段距標題 16px，其後段落間距 14px |
| 結語（通知信專有） | 14px／1.7／`#6a7282`／上方 24px |
| 按鈕 | 區塊上方 30px、置中；鈕身 `padding:10px 24px`、圓角 8、14px、白字、主色底 |
| 分隔線 | 1px `#e5e7eb`、`margin:30px 0 20px` |
| 備援連結 | 12px／1.7／`#9aa5b1`／`word-break:break-all` |
| 結尾聲明 | 12px `#9aa5b1` 置中、`max-width:440px`、上方 16px |

### 11.3 眉標／標題／按鈕：三者不重複

| 位置 | 放什麼 | 通知信 | 交易信 |
|---|---|---|---|
| 眉標 | **分類** | 人事通知 | 帳號安全／帳號開通 |
| h1 | **事實**或**要做的事**（完整一句話） | 陳小明的部門已由「資訊部」調整為「行銷企劃部」 | 重設您的密碼 |
| 按鈕 | **動作** | 前往系統查看 | 設定新密碼 |

⚠️ **踩過的坑**：交易信原本 h1 與按鈕共用同一個變數，導致大標寫的是按鈕文字「重設密碼」
而不是一句完整的話，內容反而被擠成小字。**h1 與按鈕文案一定要分開傳。**

### 11.4 品牌 footer（全信件共用）

兩排、整體置中：

- **第一排**：13px「Ystravel Platform」／`letter-spacing:.08em`／`#9aa5b1`／獨立一排／下方 8px
- **第二排**：22px 圓形 logo（右間距 8px）＋ 18px「永信生活旅遊事業」／600／`letter-spacing:.05em`／`#40525a`

⚠️ **第一排要用 `colspan` 橫跨整個 lockup 才置得了中。** 若只佔右欄（左欄留空當縮排），
它會跟著 logo 的寬度往右推、看起來沒對齊（2026-07-23 踩過）。

⚠️ **中文字距 `.05em` 刻意比英文 `.08em` 窄**：方塊字字面積本來就滿，同樣的 em 值看起來會鬆很多。

### 11.5 郵件特有的施工限制（跟 portal 完全不同）

1. **一律 inline style**，不能用 class 或外部 CSS——多數客戶端會剝掉 `<style>`。
2. **排版用 `<table>`，不用 flex／grid**——Outlook 走 Word 排版引擎。
3. **logo 走 CID 附件，不是 data URI**——Gmail 不顯示 `data:` 圖。預覽頁要臨時換成 data URI 才看得到。
4. **設計 token 用不到**：色值必須寫死十六進位。**改平台主色時要回來同步這份規範**，信件不會自動跟著變。
5. **驗收一定要真的寄一封。** 預覽頁驗不到 CID logo，也看不出各客戶端差異。

### 11.6 信件改在哪（前端的 theme seam 對信件無效，見 §12）

| 用途 | 位置 |
|---|---|
| 三封信的版型與文案 | `apps/api/src/common/mail/mail-templates.ts` |
| 品牌 footer 與 logo（base64） | `apps/api/src/common/mail/mail-branding.ts` |

---

## 12. 實作位置（改東西要改在哪）

| 用途 | 位置 |
|---|---|
| 全站 Nuxt UI theme 覆寫（含 primary、各元件 slots） | `apps/portal/vite.config.ts` → `ui({ ui: { ... } })` |
| 全域 CSS（字體、圓角基準、layout token、語意色階、aurora token） | `apps/portal/src/assets/css/main.css` 的 `@theme` / `:root` / `.dark` |
| 頁面局部微調 | `.vue` 的 `size` / `:ui` prop / `class` |

**三條施工紀律**：

1. **`vite.config` 是 build-time，改完要重啟 dev server**（不是 HMR）。
2. **覆寫主題前先讀 `node_modules/.nuxt-ui/ui/<component>.ts` 確認 class 在哪一層**，放同層才贏；覆蓋互動元件底色要連 hover/active/focus-visible 一起蓋。
3. **`vite.config` 的 `ui` 設定裡同名 key 會整個蓋掉，而且是靜默的。** 看到 TypeScript 的 `TS1117`（duplicate key）**不要只改名字閃過去，要把兩個區塊合併**。

---

## 13. Code Review 擋件清單

以下任一出現即退回：

1. `!important` 或深層選擇器硬蓋官方元件樣式（改走 §12 的 theme seam）
2. 硬寫色（`bg-white`／`text-slate-*`／`neutral-900/38`）取代 semantic token 或 aurora token
3. 重複的 UI 樣式各頁重造，沒抽 `shared/ui/`
4. 卡片用 `bg-elevated`（會糊進頁底，見 §2.4）
5. 只做 light mode、沒處理 dark
6. 用 `dark:` 寫極光專屬裝飾（見 §9.1）
7. 空值顯示 `-`／`_` 而非 `—`
8. 頁面內自己寫日期格式化，沒走 `shared/lib/datetime.ts`
9. 手刻 `<img class="rounded-full">` 當頭像（用 `UAvatar`）
10. 混用非 lucide 的 icon 套件
11. `font-medium`（系統中文字無效，見 §3.3）
12. 表格加 `table-fixed`（見 §5.4）
13. 狀態徽章顏色自己定，沒走 `account-status.ts`
14. 依賴瀏覽器原生驗證氣泡（見 §8.3）
15. 信件用 class／外部 CSS，或用 flex／grid 排版（見 §11.5）
16. 信件顏色自己定，沒走 §11.1 的四級；或 h1 與按鈕共用同一份文案（見 §11.3）

---

## 14. 已知落差與待辦

- **`reka-ui` 分段日期輸入顯示不補零**（`2023/9/23`）：來自套件內部寫死的 `defaultPartOptions`，無 prop/locale/theme 可改，要一致就得 patch 相依套件。判斷不划算——那是「編輯中的輸入框」不是「資料呈現」，兩者不會並排出現。已記在 `DateInput.vue` 檔頭，升版時再看。
- **icon 執行期從 Iconify API 抓**：內網部署前要打包 lucide collection 進本地。
- **Base token 是否抽成共用 npm 套件**：現階段先文件對齊，未來多系統重複痛了再抽（技術雷達 `@ystravel/ui`）。
- **全站表單巡檢**（哪些 modal 還沒接 schema、哪些後端欄位錯誤還在走 toast）＝獨立一輪，未排程。
- **vaul（UDrawer）殘留 transform 在 Windows 高 DPI 下文字糊**：只是桌面 devtools 模擬手機的假象、**真機正常**，不要為此加 `transform:none` 全域 hack。
- **交易信的藍 `#2563eb` 不在平台語意色盤**（§2.2 的藍＝sky）。2026-07-23 Steven 決定維持現況——它是信件裡「要你動手」的專用色、不與 app 的 info 混用；日後要對齊 sky 隨時可換，兩封交易信一起改即可。
- **信件 amber／rose 兩級尚未落地**：規則已定（§11.1），色值等第一封警告類／警示類信件出現時比照 §2.2 定。

---

## 附錄：本檔的來源與沿革

本檔 2026-07-22 整併自三份文件，並在整併過程中修正了以下**互相矛盾**之處：

| 爭點 | 舊文件的兩種說法 | 本檔採用 |
|---|---|---|
| 卡片用哪個底色 token | `authportal-ui-foundation` §3.3 說 `bg-elevated`；`foundation` §3.2 說 `bg-default` | **`bg-default`**（07-10 實測 `bg-elevated` 會糊進頁底） |
| `bg-default` 是什麼 | 舊：整頁背景 | **卡片/sidebar/header**；頁面底走 muted 灰 |
| neutral 灰階家族 | 舊：`slate` | **light gray／dark zinc 明暗混搭** |
| 殼元件 | 舊：`AuthAdminShell`／`AuthAccountShell` 雙殼 | **`AppShell` 單一統一殼**（2026-07-17） |
| 身份名片元件 | 舊：`SidebarUserCard`（過渡名） | **`UserIdentityCard`** |

決策的完整時間線與踩坑經過保留在 [`ui-conventions.md`](./ui-conventions.md)。

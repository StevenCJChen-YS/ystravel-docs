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
| **solid 主鈕**（固定色階，**明暗拆階**，2026-07-24 拍板） | `bg-primary-550` 白字（2.98，**刻意接受例外**見 §2.7），hover/active → **650/750** | `bg-primary-450` **深字** `text-neutral-900`（8.37 ✅），hover/active → **550/650** |
| **solid error 鈕**（固定色階，明暗一致，2026-07-24） | `bg-error-600` 白字（4.53 ✅），hover/active → 700/800 | 同 light |
| **switch checked**（固定色階，明暗一致） | `bg-primary-500` | 同 light |

> **solid 主鈕拆階的來龍去脈**（2026-07-24，取代 07-14「明暗一律固定 550」）：550 白字僅 2.98
> 未達 AA，本輪原意是修到達標——但過標的 light 候選 Steven 都嫌深（650＝4.59、700＝5.36 皆
> 看過實畫面後否決），**拍板「好看優先」維持 550、列刻意接受例外**（teal 亮階配白字物理上
> 到不了 4.5，見 §2.7）。dark 側則真正修好：700 明暗通用版對背景分離度腰斬（3.30）淘汰、
> 550 深字版 active 越按對比越掉（3.31）淘汰，定案**亮底深字**（Material／Nuxt UI 官方同路線，
> 官方 dark＝400 深字，我們取自訂 450 半階稍飽和）。最終梯度＝**「半階位整步跳」：
> light 550→650→750、dark 450→550→650，dark 就是 light 整體亮一階**。各組合數字見 §2.7。

**自訂半階的做法**（Nuxt UI `--ui-color-*` 別名只內建 50–950）——**兩套別名都要接**，缺一不生效
（例外：650 只當 utility 用、沒被 `--ui-*` 引用，接 ① 即可）：

```css
@theme {
  --color-teal-450: oklch(74.05% 0.146 182.2);    /* oklch 插值取 400/500 中間（主鈕 dark base） */
  --color-teal-550: oklch(65.2% 0.129 183.604);   /* oklch 插值取 500/600 中間 */
  --color-teal-650: oklch(54.6% 0.105 185.8);     /* ⚠️ 非中點：中點白字 4.41 差 0.09 不過 AA，取「最淺過標點」＝4.59 */
  --color-teal-750: oklch(47.4% 0.087 187.3);     /* oklch 插值取 700/800 中間（主鈕 light active） */
  --color-amber-550: oklch(71.75% 0.1835 64.199);
  --color-primary-450: var(--color-teal-450);      /* ① Tailwind utility 用（bg-primary-450） */
  --color-primary-550: var(--color-teal-550);
  --color-primary-650: var(--color-teal-650);
  --color-primary-750: var(--color-teal-750);
  --color-warning-550: var(--color-amber-550);
}
:root {
  --ui-color-primary-550: var(--color-teal-550);   /* ② Nuxt UI --ui-primary 引用用 */
  --ui-color-warning-550: var(--color-amber-550);
}
```

> ⚠️ `--ui-*` 未分層寫在 `:root` 會洩漏到 dark，故 `.dark` 要補回 400 抵銷。

### 2.4 表面層級（2026-07-24 重訂，取代原「中性底色分層」）

**全站只有三層表面，一律引用 token，禁止各頁再寫 `dark:bg-neutral-8xx`（§13-25 擋件）：**

| Token | 用在哪 | light | dark |
|---|---|---|---|
| `--app-surface-page` | AppShell 內容區底（卡片後面那層） | neutral-100 | neutral-950 |
| `--app-surface-card` | 卡片（`SurfaceCard`）、殼件（header／drawer）、modal | `#fff` | neutral-800 |
| `--app-surface-inset` | 卡片**裡面**的區塊（帳號中心的內卡等） | neutral-50 | neutral-900 |

用法：`class="bg-[var(--app-surface-card)]"`。定義在 platform `apps/portal/src/assets/css/main.css`。

**為什麼重訂**：原本 850／950 被混用在不同語意上（850 同時當卡片、深色表頭、modal；
950 同時當內容區底與欄位），實測結果是**全站其實只有一個表面層級**——
三層容器套疊時色值完全相同，「內卡 vs 外卡」明暗兩邊都是 **1.00（逐位元同色）**，
巢狀讀起來只是邊框在疊；卡片對頁面也只有 light 1.05／dark 1.17。
連帶症狀：輸入框與內容區底同為 neutral-950，卡片上的欄位看起來像被打了一個直通頁面的洞。

**相鄰層分離度（真實元件實測）**：

| | light | dark |
|---|---|---|
| card vs page | 1.05 → **1.10** | 1.17 → **1.34** |
| inset vs card | 1.00 → **1.05** | 1.00 → **1.19** |

> **light 拉不高是物理限制**，近白表面之間擠不出明度差——淺色靠的是邊框與陰影，
> 不要為了追數字把淺色頁面底壓深。淺色的層次感主要來自「內卡不與外卡同色」這個結構修正。

**規則**：

1. **`inset` 不可直接疊在 `inset` 上**（會退回 1.00 的老問題）。需要再深一層＝回來重新檢討層級。
2. **⚠️ token 一律引用 `--ui-color-neutral-*`，不可用 `--color-neutral-*`**——後者明暗都是 gray、
   從未被重新指向，只有前者會在 `.dark` 被換成 zinc。寫錯的話深色會變成帶藍調的 gray
   （實測 `rgb(3,7,18)`），違反 2026-07-11「dark 走 zinc 免得偏 navy」的拍板，
   **而且淺色完全正常、不會被發現**。
3. **井底層（輸入框／`TableFrame`／`OrderManager`）刻意不納入這三個 token**，維持 light 白／dark `neutral-950`：
   ①它與 page 同值可接受——兩者永遠不相鄰（井底一定在卡片內、page 一定在卡片外）
   ②`TableFrame` 的斑馬紋在 dark 正是 `neutral-900`，把框底改成 `inset(900)` 斑馬紋會直接消失。
   原本「欄位像挖了個洞」的觀感，卡片提到 800 後井底對卡片拉到 1.34，就會讀成刻意的凹槽。
4. **表格 thead 是深色標題帶、不是表面層**，維持 light `neutral-700`／dark `neutral-850`；
   斑馬紋偶列 `bg-muted`（dark `neutral-900`，對框底 1.12）。

> Nuxt UI 原生的 `bg-default`／`bg-muted`／`bg-elevated`／`bg-accented` 仍用於**狀態**
>（hover、選取態、active nav、卡內次層帶），但**表面層級一律走上表三個 token**。
> ⚠️ 卡片不要用 `bg-elevated`：light 的 `bg-elevated` 與頁面底同值，卡片會糊進頁底（2026-07-10 實測）。

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

### 2.7 對比度基線（2026-07-24 首度實測定案）

標準＝WCAG AA：**一般文字 4.5、大字與 UI 元件 3.0**。
量測法＝computed style → 畫進 canvas 讀 sRGB → WCAG relative luminance。
**對比是 sRGB 純數學、與螢幕無關**——換螢幕看起來深淺不同是面板體感，數字不會變；
「同一頁兩個螢幕數字不同」＝其中一邊是舊版頁面，先重新整理。

**已驗組合**（新增顏色組合前先對表；表上沒有就量了再用）：

| 組合 | 對比 | 判定 |
|---|---|---|
| solid 主鈕 light：白字 on teal-550（base）／650（hover）／750（active） | **2.98**／4.59／6.39 | base ＝刻意接受例外（見下） |
| solid 主鈕 dark：neutral-900 深字 on teal-450／550／650 | 8.37／5.96／**3.87** | base·hover ✅；active 為按住瞬間的過渡態，接受 |
| solid 主鈕 dark：按鈕 450 對卡片底 `#18181b` | 8.36 | ✅（分離度） |
| solid error 鈕：白字 on rose-600／700／800 | 4.53／6.03／— | ✅ |
| 文字五階 on 白底：highlighted／default／toned／muted／dimmed | 17.75/10.30/7.56/4.84/**2.60** | 前四階 ✅；dimmed 見下 |
| 表格深色表頭白字 | 10.30 | ✅ |
| 白字 on teal-500／550／600 | 2.42／2.98／3.67 | ❌ 不得用於 solid 鈕文字 |
| 白字 on rose-500／amber-550／emerald-600／sky-600 | 3.75／2.61／3.65／4.02 | ❌ 同上 |
| 深字 on amber-550 | 6.8 | ✅（warning 若做 solid 鈕的唯一解） |
| **soft／subtle light**：文字 **700 階** on 同色相 `/10` 淡染底 | primary 4.81／success 4.73／warning 4.55／info 5.13／error 5.21 | ✅ **現行** |
| **soft／subtle dark**：文字 400 階 on 同色相 `/10` 淡染底 | 5.40～8.48 | ✅（本來就達標，**不動**） |
| soft 鈕 hover（底色轉 `/15`）：文字 700 階 | error 4.84 | ✅ |
| ~~soft／subtle light：文字＝語意色本身（550/600）~~ | 2.38～3.56 | ❌ 2026-07-24 已修 |
| soft／subtle light：文字 600 階 | 2.92～3.91 | ❌ 全數未過（success·info 本就是 600＝不動） |
| soft／subtle light：文字 650 半階 | 3.64～4.52 | ❌ 僅 error 過 |

> **為什麼 soft／subtle 非得用到 700 這麼深**：它的底色是**同色相 10% 淡染**＝一塊接近白的底，
> 要在近白底上拿到 4.5，文字就必須是真的深色。這與 solid 鈕（深底白字）是相反的問題，
> 不能拿 solid 的色階直覺套過來。

**刻意接受的例外**：

- **solid 主鈕 light base（白字 on teal-550）2.98**：過標候選 650／700 Steven 看過實畫面後
  均嫌深，拍板好看優先（2026-07-24）。teal 亮階配白字物理上到不了 4.5——這是品牌色的
  已知代價，**不得**引用本例外去開新的不及格組合（新組合仍走規則 1）。
- **`text-dimmed` 2.60**：拉高會破壞 §2.5 五階層次。限最弱提示、角標；不得承載必要資訊。
- **switch checked teal-500 對白底 2.42**：開關狀態由**滑塊位置**表達，顏色是冗餘線索。
- **focus ring（`--ui-primary` 550）對白底 2.98**：差 3.0 線 0.02，現階段接受；日後動 `--ui-primary` 時順手校正。

**規則**：

1. **新做任何「語意色 solid 鈕」前先過上表**——白字達 4.5 才准上；不夠就換深階或改深字，不准硬上。
2. **amber 永不配白字**（2.61）；warning 若需 solid 鈕一律深字。
3. success／info 目前無 solid 鈕；要新增前先量。
4. **`text-primary`（550，對白底 2.98）限強調色／標籤用途，不得當正文連結色**——正文超連結要另尋達 4.5 的方案。
5. **soft／subtle 一律「light 文字 700 階＋dark 維持 400 階」**，規則收在 `vite.config.ts` 的
   badge／alert／button theme（`SOFT_TEXT` 一張表），**不准逐頁貼 class**。新元件若也吃
   `bg-{color}/10 text-{color}` 這組官方預設，補進同一張表即可。
6. **新增自訂半階一律取「最淺過標點」，不是數學中點**——`--color-teal-650` 就是這樣來的
   （中點 L55.55% 白字 4.41 差 0.09 不過，往下取 L54.6%＝4.59）。**同名 token 只能有一套推導邏輯**：
   要補 emerald／amber／sky／rose 的 650 就得照這條規則現場量，不可用插值省事
   （2026-07-24 曾評估補齊四色 650，因只有 error 過標＋推導邏輯會分裂而作罷）。

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

**墨量實測（2026-07-24，微軟正黑體，canvas 數字元覆蓋的像素比例）**——把上面的結論變成數字，
日後再問「那用 medium 呢」直接看表結案：

| | 400 | 500 | 600 | 700 |
|---|---|---|---|---|
| 中文 14px | 1.00 | **1.00** | 1.168 | **1.168** |
| 中文 12px | 1.00 | **1.00** | 1.174 | **1.174** |
| 英文 14px | 1.00 | **1.00** | 1.155 | **1.155** |

- **500 與 400 的墨量完全相同**（比值 1.000）＝ `font-medium` 是 no-op，不是「差一點點」。
- **600 與 700 渲染結果一致**（都吃到同一個 Bold 字面）→ 需要粗體時一律寫 `font-semibold`，不用 `font-bold`。
- **Bold 相對 Regular 只多約 17% 墨**——「400 跳 700」聽起來很大，實際的視覺跳躍比想像小。

> 起因：dark solid 主鈕改成「亮底深字」後 Steven 覺得字看起來變細。
> 那是 **irradiation（光暈效應）**——亮底上的暗字邊緣被光侵蝕、看起來較細，暗底上的亮字則外溢顯粗，
> 所以同一字重在明暗兩種底色下本來就不等重。**2026-07-24 拍板維持 400 不加粗**（Steven 看過並排後
> 認為 Bold 太粗）；對比 8.37 遠超 AA，這純粹是觀感、沒有可讀性問題。

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

### 4.3 斷點與驗收基準寬度

`base <640` / `sm ≥640` / `md ≥768` / `lg ≥1024` / `xl ≥1280` / `2xl ≥1536`

**驗收基準寬度（2026-07-24 Steven 定）**：

| 寬度 | 要求 | 說明 |
|---|---|---|
| **375px** | **完整驗收——要好看、不擠、不裁切** | 手機的驗收基準。iPhone SE／13 mini 的邏輯寬，現行最窄的常見機型 |
| 320px | 只要求**不出現水平捲動**，不要求好看 | `main.css` 的 `body { min-width: 320px }` 是防線 |
| ≥1280px | 桌機驗收 | |

**UI 改動的驗收＝桌機與 375px 各看一次**（2026-07-23 選項樹事故：只看桌機、數字漂亮就當完成，
手機實際是兩層巢狀捲動）。**不要用「把元素寬度鎖成 375px」來模擬**——Tailwind 的 `sm:` 斷點
看的是**視窗寬度**，元素縮小不會觸發單欄，量到的東西全部無效（2026-07-24 踩過，
中文被逐字拆行、兩欄一直開著）。要真的把視窗調到 375。

**375px 要看的兩件事**：

1. **無水平捲動**（`document.documentElement.scrollWidth > innerWidth`）
2. **沒有元素的右緣超出容器內容框**——⚠️ **不能用 `scrollWidth > clientWidth` 驗**，
   元素會自己撐大、兩者永遠相等（見 §5.5.1）

> **「剛好塞得下」＝不合格。** 量到內容寬與可用寬相等（零餘裕）就要當成會壞——
> 不同裝置的字體渲染差一點點就翻車。2026-07-24 個人資料頁的 email 正是
> 「375px 下恰好 197px vs 可用 197px」，我的環境剛好過、Steven 實機被切掉。

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
- **動作鈕分兩層**（2026-07-24 依實作校正原「一律貼著表格」的說法）：
  ①**頁面主要動作（新增／匯入等）→ `PageHeader` 的 `#actions`**（標題同行右邊，用 `ToolbarButton`）——全站一致（使用者、角色、工作日曆皆此擺法）。
  ②**列內／表格層的 CRUD（每列編輯刪除、欄位顯示、調整順序）→ 貼著表格**（`DataToolbar #actions` 或列尾），別把這些堆到頁首。
  （原文「動作鈕不要放頁首」易被讀成「連主要動作都不能放頁首」，與 roles/users 現況不符，故校正。）

**條件多到排不下時，先量再決定斷點**——用 `getBoundingClientRect` 量工具列那一列的寬度與 `top`（`top` 出現兩種值＝已經換行了），別憑感覺挑。

### 5.4 表格

- **外觀一律抽 `vite.config` 的 `table` theme**（深色表頭、列高、內距），不逐頁寫 `:ui`。
- **欄寬＝auto 佈局＋百分比 th**（名稱/說明欄不設寬、拿剩餘），**勿加 `table-fixed`**——會鎖死 100%，多開欄位互相硬擠疊字。配 `TableCard` 的 `overflow-x-auto` 自然橫向捲動。
- **RWD 藏欄走 `useResponsiveColumns`**，別逐欄寫死 `hidden md:table-cell`；搭 `ColumnVisibilityMenu` 可手動加回，同一份 `columnVisibility` 綁 UTable。
- Nuxt UI 的 `UTable` **沒有內建 density prop**，要舒適/精簡切換得自綁兩組 padding。
- 可排序的表頭**一律走 `useTableSort` composable ＋ `sortableHeader()` helper**（見 §5.4.1），不各頁自刻 sortKey/accessors；底層按鈕＝`TableSortButton`（button theme 的全域 `font-normal` 會蓋掉 th 字重，helper 內已補回）。

### 5.4.1 表格欄位排序（sort，2026-07-24 新增）

> ⚠️ 名詞：這裡是 **sort（升降冪）**，跟 §5.5 的 order（人工序位）是兩件事。
> 一張表**兩者擇一**：有拖曳序位的表不給 column sort（否則點欄位排序會蓋掉你排好的序位）。

**哪些欄位可排序＝按型別自動推導，不必逐頁討論**（只標例外）：

| 欄位型別 | 預設 | 理由 |
|---|---|---|
| 名稱／文字識別、日期／時間、數字／量 | ✅ 可排序 | 掃描、找最近/最大很常用 |
| 單一狀態／enum（在職、啟用…） | 🟡 **有專屬篩選就不給**（能篩就不排） | 避免篩選與排序做同一件事 |
| 操作欄、頭像/icon 欄、多值欄（角色/標籤清單） | ❌ 不給 | 無語意順序或無單一排序鍵 |
| 有人工拖曳序位（order）的表 | ❌ 整表不給 column sort | 與序位語意衝突（見上方 ⚠️）|

**機制＝全站一套 UI、資料綁定分兩種**（`apps/portal/src/shared/composables/useTableSort.ts`）：

- **前端分頁的表**：`const sort = useTableSort(() => filtered.value, accessors)`，把 `sort.sorted` 接在**分頁切片之前**（`sort.sorted.value.slice(...)`）；欄位定義寫 `header: sortableHeader('姓名', 'name', sort)`。三段循環：升→降→取消。
- **後端分頁的表**（員工、稽核）：同一個 `sortableHeader` UI，但把 `sortKey`/`sortDir` 餵給 API 的 `orderBy`／`orderDir`，**不做前端排序**（前端只排得到當頁＝錯的）。
- accessor 回傳字串走 `localeCompare('zh-TW')`、數字比大小；空值給 `''` 或 `0`。

### 5.5 順序（order）管理

> ⚠️ **名詞**：「排序」中文兩義——sort（升降冪）vs **order（排列序位）**。本節指後者。

- **順序是主檔自己的屬性**，在該主檔管理頁維護（Salesforce picklist／Odoo sequence 同理）；存一份、所有 consumer 讀同一份，**不要每個表單各自定義順序**。
- 三原則：①`order` 一律「**相對同層兄弟**」②UI 用**拖曳**（`OrderManager`＝握把拖＋「依名稱排序」一鍵＋**批次儲存/取消**，非邊拖邊存）③建立/編輯 modal **不放** order 數字欄，新增自動排最後。
- **數字給人看一律 1-based**（`order + 1`），0-based 只留程式內。
- 後端 reorder 端點收「排好的 id 陣列」→ 交易重編 0,1,2… ＋ audit。
- UI＝**modal**（工具列「調整順序」鈕開），清單窄單欄 `max-w-2xl`、可捲、序號 1-based 徽章。

### 5.5.1 定義列（唯讀資料的 `dl`，2026-07-24 新增）

唯讀資料頁（帳號中心個人資料等）的欄位陳列：

- **單值欄位配成兩欄**（`sm:grid-cols-2`），`sm` 以下自動回單欄。
  8 個欄位一行一個會拉出很長的卡片、右側整片留白；配對後高度直接砍半
  （實測任職資料卡 512px→269px，桌機省 47%、手機 704→448 省 36%）。
- **label 與 value 要拉開層級**：label `text-xs text-muted` 固定寬、value `text-sm font-medium text-highlighted`。
  兩者同字重只差顏色時，掃視會糊成一片、眼睛沒有著陸點。
- **手機維持 label／value 同一行**（不要上下堆疊），比堆疊更矮。
- **⚠️ 長值（email 等無空白字串）要不撐破版面，「列」與「value」都要 `min-w-0`，缺一不可**：
  ①列是 grid item，`min-width:auto` 使其最小尺寸＝內容 min-content，
  grid 欄會被撐到比容器還寬（實測欄 328.7px vs 容器 285px）
  ②value 是 flex item，同樣不會縮到小於內容寬。
  `break-words`（`overflow-wrap:break-word`）**只在排版時允許斷字、不會縮小 min-content**，單靠它救不了。

> **⚠️ 這類溢出用 `scrollWidth > clientWidth` 驗不出來**——元素會自己撐大，兩者永遠相等。
> 正確驗法＝比對「**右緣是否超出容器內容框**」。2026-07-24 因此誤報過一次「沒有溢出」，
> 實機才被 Steven 抓到被切掉。附帶：該 email 在 375px 下恰好 197px、可用寬也恰好 197px，
> **零餘裕**——這種「剛好塞得下」在不同裝置的字體渲染下必然翻車，量到剛好就要當成不合格。

> **⚠️ 多值欄位一律跨整列（`sm:col-span-2`），不參與兩欄配對。**
> 多值＝角色、標籤、分眾這類「數量不固定的徽章清單」（§5.4 已把它列為不給排序的欄型，同一個分類）。
> 理由不只是自己會擠：**CSS grid 同列等高，多值欄一膨脹會把隔壁的單值欄一起撐高**，
> 下方留一大片空白。實測 6 個角色在半欄折成 3 行、把隔壁「公司信箱」撐到 109px；
> 跨整列後 6 個角色一行放得下、列高 49px，**整個區塊反而更矮**。
> 一個帳號可跨模組掛多個角色（`UserRole` 多對多、`Role` 各綁一個模組），這不是極端情境。
> CRM 的客戶標籤／分眾是同一個形狀且數量更多，比照辦理。

### 5.6 容器範式：modal／整頁／drawer（2026-07-23 新增）

> §1 原則 1 只寫「建立/編輯開 modal」，**沒有上限**。本節補上邊界。
> 起因：CRM 的客戶詳情會長成「基本資料＋聯繫紀錄＋消費總表＋標籤＋諮詢 case」，
> 照字面套 §1 會做出一個塞不下的 modal。
>
> ⚠️ **本節是初版，尚未經實際使用驗證**（2026-07-23 訂）。用到覺得不對就回來改，
> 改的時候把「哪個頁面、為什麼不適用」一起寫進來。

**判準不是「資料多少」，是這三問，由上往下問：**

| # | 問題 | 是 → | 否 → |
|---|---|---|---|
| ① | 這東西需要**有自己的網址**嗎？（貼連結給同事、重新整理後還在、瀏覽器上一頁要有意義）| **整頁** | 往下 |
| ② | 使用者需要**一邊看它、一邊參照背後的清單**嗎？ | **drawer / slideover** | 往下 |
| ③ | 做完之後是**回到原地繼續**嗎？ | **modal** | 整頁 |

**第①問是最硬的判準**——能不能分享、能不能加書籤、瀏覽器上一頁會不會壞掉，
這三件事 modal 永遠給不了。有疑慮時先問這一題。

**一條硬上限（凌駕上表）**：欄位超過約 8 個、或需要 tabs 才裝得下 → **一律整頁**。
塞不下的 modal 會逼使用者在小視窗裡捲動填表，是最差的體驗。

### 已知歸屬

| 場景 | 容器 | 依據 |
|---|---|---|
| 新增/編輯使用者、角色、公司、部門 | modal | ①否 ②否 ③是；欄位少 |
| 任職歷程檢視 | modal | 1~3 筆唯讀，蓋一整頁＝過度設計（見 platform `features/hr-employment-history/decision.md`）|
| 調整順序 | modal | 做完回原地（§5.5）|
| 篩選 | `<sm` drawer／`≥sm` modal | 見 §5.3 |
| 稽核紀錄 | 整頁 | 搜尋為主、要能貼連結 |
| **CRM 客戶詳情** | **整頁** | ①是——要能貼連結、內容分多區、會長出 tabs |

**詳情頁重啟條件**（modal 改整頁的訊號，任一出現就重新評估）：
內容長出第二個功能區塊（附件／權限覆寫／子清單）、需要被連結分享、
或觸控裝置使用者反映資訊看不完。

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
- **浮層裡的空狀態（可搜尋下拉、command palette…）＝借 `EmptyState` 的結構語言，但不要套那個元件。** 它是虛線框＋24px 內距，塞進浮層會變成「框裡面又一個框」（同樹狀選項踩過的兩層框）。
  做法：小 icon（`size-5 text-dimmed`）／標題 `text-default`／說明 `text-xs text-dimmed`，**層次靠顏色做**——官方 `empty` 插槽外層已經是 `text-sm text-muted` 置中，標題不升 `text-default` 的話兩行會糊成一坨（2026-07-23 實作時 Steven 當場退回過一次）。
- **選項範圍有收斂時，空狀態要「指路」，不能只說「找不到」。** 清單本身有範圍限制（例：稽核的動作下拉跟著檢視分頁收斂）時，攤開清單使用者看得到範圍、不會誤會；**一旦加了搜尋框，空白就讀起來像「系統沒有這個東西」而不是「你在錯的地方」**。
  要接著講出正確的位置（「切到『登入與安全』分頁再搜尋看看」），且**位置名從資料反查、不要寫死文案**——之後新增選項或分頁才不必回頭改字。
- **空值一律 `—`（em dash）**：`?? '—'` 或 `<span class="text-dimmed">—</span>`，不准出現 `-`／`_`。
- **顯示給使用者的資料用中文名，不露 code**：有 code＋name 的實體一律顯示 name（hover 補英文代碼）。
- **modal 縮窄**：欄位少的建立/編輯 modal 用 `width-class="sm:max-w-md"`，別用預設 512 太寬。
- **有輸入欄位的 modal 一律走 `FormModal`，不要手刻 `UModal`**（2026-07-23 收斂帳號中心三顆）。
  手刻的會少掉晃動提示、footer 樣式與標題凍結，行為跟其他頁不一致。
- **點遮罩能不能關＝看「誤關的代價」，不是看畫面大小**：
  - **鎖住（預設，`dismissible=false`）**：有打字的表單。誤關要重打，代價高 → 點遮罩改成
    **晃一下**（macOS 對話框手勢），明確告訴使用者「這裡關不掉，走取消或 X」。
  - **可關（`dismissible`）**：挑一下就走的輕操作（換頭像、選名片背景）。誤關重開零成本，
    鎖住反而礙事。
  ⚠️ **「有擋但沒有回饋」＝使用者會以為程式壞了**——這正是手刻 UModal 的問題（點遮罩靜默無反應）。
  要嘛給晃動，要嘛就讓它關。

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
   ⚠️ **只要你覆寫了 hover 的底色或邊框，就要自己補 gate——官方的 `disabled:` 保護不到你的覆寫。**
   官方 theme 用 `disabled:bg-*` 把停用態壓回去，那只蓋得住官方自己的 hover 值；我們在同層寫的
   `hover:bg-*`／`aurora:hover:bg-*` 特異性相同、順序在後，照樣贏。
   **踩法（2026-07-23）**：light 看起來正常、dark/aurora 才露餡——因為 light 我們的 hover 值恰好
   與底色同值（巧合，不是設計），aurora 換成半透明值就現形。
   **檢查法**：`vite.config` 裡搜 `hover:`／`active:`，每一條問「這是我們自己寫的嗎」，是就要有 `not-disabled:`；
   驗證要**真滑鼠 hover ＋量 computed**（停用件 hover 前後同值），別只看截圖。
2. **警告色只留給「真正需要行動」的事**：狀態資訊用欄位＋篩選呈現，不要常駐警告。天天顯示不是問題的事，只會訓練使用者無視所有警告。
3. **連動下拉：父層沒選就 disabled，並用 placeholder 講下一步**（「請先選擇公司」）。**同一條規則要同時套到 modal 表單與篩選列**，且父層變更/清空時一併重置子層。
4. **輸入框內緣的小圖示鈕用原生 `title` 不用 `UTooltip`**（見 §6.4），並**一律加 `@mousedown.prevent`**——不擋的話按下瞬間輸入框先失焦，表單會用舊值跑驗證而閃出錯誤。

### 8.2 Dark mode 與浮層

- **dark 下不靠 `shadow` 製造浮起感**（陰影幾乎看不見），改用 **`ring-1 dark:ring-white/25` hairline ＋ 深色明度分層**：頁底 `neutral-950`（最深）< 卡／modal／側欄 `850` < 表格內回 `950`、斑馬偶列 `900`；輸入框 dark 底壓到 `950`。已抽到 `vite.config` 各 content slot，新元件自動套用。
- **【鐵律】dark 裝飾效果不准動到 light**：裝飾層用 `hidden dark:block` 只在 dark 掛載；效果 class 一律 `dark:` 前綴。**改完務必切 light 實測 computed** 才算數。
- **遮罩（scrim）全站同一種深度＝`bg-black/50 dark:bg-black/70`**（2026-07-23 定）。官方預設是
  `bg-elevated/75` 霧白，深色頁面上分離感不足；dark 加深到 70% 對齊 Material dark 慣例（60~70%）。
  ⚠️ **三個元件的層位不同，覆寫要各放各的同層才贏**：`modal` 的底色在 **overlay 變體層**
  （`variants.overlay.true.overlay`）、`drawer`／`slideover` 在 **slots 層**（`slots.overlay`）。
  放錯層＝靜默無效。改遮罩時**三個一起改**，別只改 modal（drawer/slideover 曾漏改半年）。

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

### 8.5 捲動與畫面角落的浮動元件（2026-07-23）

**捲的不是 window。** `AppShell` 是 `fixed inset-0 overflow-hidden`，真正在捲的是 Nuxt UI
dashboard panel 的 body——`window.scrollTo` / `window.scrollY` / `useWindowVirtualizer`
在本站**通通沒作用**。任何跟捲動打交道的功能一律用共用的
`shared/composables/useScrollRoot`，不要自己寫一份（那個判定有 CSS 陷阱，見
`operations/TROUBLESHOOTING.md`）。

**長清單一律跟著整頁捲，不要給它自己的捲動區。** 「頁面主體就是一張表／一棵樹」的管理頁，
標準做法是整頁捲＋底層虛擬化。硬塞一個 `max-h` 內捲區塊會同時製造兩個問題：桌機高螢幕
下方留一片空白、手機變成頁面與清單**兩層巢狀捲動**（2026-07-23 選項樹踩過，Steven 回報）。

**畫面角落只能有一個常駐主角。** 右下角歸**全站回頂部鈕**（`i-lucide-arrow-up-to-line`、
圓角正方、桌機 40×40／手機 36×36、`fixed right-4 bottom-4 z-40`）：

- 捲超過 **400px** 才淡入；**捲到頂整顆移除**（`v-if`，不是變透明）——不佔位就不可能擋到手機內容
- 桌機 toast 也在右下 → **toast 讓位**：viewport 從 `bottom-4` 抬到 `bottom-20`，
  **只在按鈕顯示時抬**（固定抬高會讓頁面頂端的 toast 下方空一段沒有理由的距離），
  並加 `transition-[bottom]`——沒有它，顯示中的 toast 剛好跨越門檻會「跳」一下，讀起來像 bug
- ⚠️ **不要把 toast 改到 `top-right` 來閃避**：實測會蓋住 header 的「內部收件匣」鈴鐺，
  疊 4 則時整個右上被吃掉；通知 toast 蓋住通知鈴鐺說不過去
- 手機 toast 在 `top-center`，與按鈕不同角落，不需處理

**長清單的效能門檻。** 每列的成本要算**元件實例**而不是 DOM 或按鈕數——包裝元件比被包的那顆
還貴（選項樹實測每列 64 個實例，其中約一半來自 5 個 `UTooltip` 的四層包裝）。
可見列數可能破百的清單一律虛擬化（`@tanstack/vue-virtual`，flatten visible nodes + windowing），
不要靠「加個 loading 遮一下」掩飾卡頓。

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

### 9.2 極光下的文字對比（2026-07-24 逐像素掃描後定案）

**極光下文字階整體提亮半階**，規則收在 `main.css` 的 `.dark.theme-aurora`：

```css
--ui-text-muted:  oklch(0.788 0.0105 286.18);  /* zinc 350 半階（原 400）*/
--ui-text-dimmed: oklch(0.6285 0.0155 286);    /* zinc 450 半階（原 500）*/
```

> **⚠️ 是半階不是整階，這點踩過坑。** 第一版提整階（muted → 300），但 **`toned` 本來就是 300**——
> 兩階變同色，五階層次被壓成四階（Steven 一看畫面就發現「沒有層次」，量測證實
> `toned→muted` 的亮度差是 **0**）。改取中點後相鄰階亮度差回到 0.222／0.117／0.172／0.240 的均勻節奏。
>
> **這兩個中點值刻意不照 §2.7 規則 6 的「最淺過標點」推導**——那條規則是為了讓組合過 AA，
> 而這裡的目的是**層次間距**，且極光下本來就過不了。取中點才能讓五階等距。

**不要再逐處貼 `aurora:text-muted`／`aurora:text-toned`**（§13-28 擋件）——
`AppShell` 的 icon／分組小標與 `EmptyState` 的 avatar icon 原本各自貼補丁做同一件事，
2026-07-24 已收進 token 並移除，視覺不變。

**為什麼需要**：極光背景層是 `fixed inset-0`，**亮光帶永遠釘在畫面上方三分之一、不隨捲動離開**，
所有內容都會從它底下經過——那正好是閱讀時視線停留的位置，不是「頁面頂端的一小塊」。
掃描（動畫取峰值、含毛玻璃 24px blur、扣掉 header 的內容區）：

| 文字階 | 改前（玻璃 38%） | **改後（玻璃 45% ＋ 半階）** |
|---|---|---|
| highlighted | 4.21／1.8% | **4.98／0%** |
| default | 3.32／4.4% | 3.93／3.1% |
| toned | 2.85／8.0% | 3.37／4.8% |
| **muted** | **1.61／22.1%** | **2.56／10.3%** |
| dimmed | 1.00／100% | 1.42／28.2% |

**單靠玻璃不透明度救不了**——達 4.5 所需的最低不透明度：white 41%／default 51%／toned 57%／
**muted 80%**／**dimmed 即使 100% 也不夠**（`neutral-500` 對 `neutral-900` 本身只有約 3.0）。
80% 等於看不見星空，主題就白做了。**玻璃的邊際效益也很低**：對 muted 而言
45%→10.3%、55%→6.9%、60%→4.5%。真正把 muted 從 22% 拉下來的是文字階提亮，不是玻璃。

> **刻意接受的例外**：最亮光帶上 `muted` 仍有約 10% 面積、`dimmed` 約 28% 面積低於 AA 4.5。
> 那是半透明玻璃壓在發亮漸層上的固有結果。
> `dimmed` 與 §2.7 既有的 dimmed 例外同一條理由：限最弱提示、**不得承載必要資訊**。
> 日後若覺得亮帶處仍糊，下一個有意義的點是玻璃 **55%**（`toned` 降到 1.8%、`muted` 6.9%），改一個數字即可。
>
> ⚠️ **層次與對比在這裡是直接的取捨**：提整階讓 muted 只剩 4.6% 不過標，但會與 toned 撞色、
> 損失一階層次。2026-07-24 Steven 拍板**要層次**，接受 muted 的 10.3%。

> 附帶：`--aurora-panel` 原本的 38%（2026-07-24 改 45%）**不是為這片極光調的**——2026-07-16 從另一個專案
> my-apps 的 `AppPageLayout glass` 直接沿用。所以它沒有「原本的理由」需要保護。

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

### 10.3 通知內容規範（收件匣＋Email 共用，2026-07-24 定）

> 每種通知的文字都照同一套文法填，未來新事件照套即可。權威＝後端註冊表
> `apps/api/src/core/notifications/notification-registry.ts`（每筆事件一組 `category`／`title`／`body`）。
> 收件匣（`NotificationPanel.vue`）與 Email 眉標（§11.3）**吃同一份**，不各寫一套。

**三個語意欄位：**

| 欄位 | 是什麼 | 規則 |
|---|---|---|
| **類別** | 這是哪種通知 | 短、結尾一律「通知」；**註冊表 `category` 是單一來源**，收件匣第一行＝Email 眉標同字。**與偏好分組脫鉤**（行事曆提醒掛 ACCOUNT 分組收偏好，但類別是「工作日曆通知」）。 |
| **標題** | 一句「發生了什麼」的短事實 | 單行、不塞會爆長的細節。**語態跟收件人走**：通知關於**別人**→第三人稱事實（`王小明 已到職`）；關於**您自己**→第二人稱「您」（§10.2）（`您的帳號角色已變更`）。 |
| **內文**（可選） | 補充：影響／要做什麼／安全提醒 | 標題放不下的細節下放到這；沒有就不給。 |

**踩過的坑（這規範解掉的）：**

1. **類別雙頭**：曾經收件匣用前端字典、Email 用分組名各推一套——行事曆提醒收件匣顯示「管理中心通知」、Email 顯示「帳號與安全通知」，同一封信兩個名字。收斂成註冊表 `category` 單一來源。
2. **標題爆長換行**：`${姓名} ${summary}` 一長就撐到第二行。規範：標題只放短事實，細節進內文（如職務異動：標題 `王小明 職務異動`＋內文 `職稱調整為 資深專員兼專案經理`）。

**現行 6 種通知（範例對照）：**

| 事件 | 類別 | 標題 | 內文 |
|---|---|---|---|
| 到職 | 人事異動通知 | 王小明 已到職（2026-08-01） | 公司／部門 |
| 職務異動 | 人事異動通知 | 王小明 職務異動 | 職稱／調動細節 |
| 離職 | 人事異動通知 | 王小明 已離職（2026-08-01） | — |
| 角色調整 | 帳號安全通知 | 您的帳號角色已變更 | 由管理員調整，可至個人資料頁查看目前權限。 |
| 行事曆提醒 | 工作日曆通知 | 2027 年政府行事曆尚未匯入 | 下載＋匯入指引 |
| 密碼重設 | 帳號安全通知 | 管理員已為您寄出密碼重設連結 | 安全提醒 |

> **敏感細節鐵則（不變）**：`title`／`body` 只准寫「誰、什麼異動、什麼時候」，禁止夾離職原因、薪資、考核等——
> Code Review 擋件（見註冊表檔頭 Rule 8）。離職通知刻意無 `body`＝結構上就沒地方塞敏感欄位。

**新增通知類型的流程（Code Review 擋件，§13-19）：** 任何新增 `notificationRegistry` 事件的 PR，
其 **`category`／`title`／`body` 文案必須先跟 Steven 討論定案**（這類叫什麼、標題怎麼寫），
不可工程師自行命名就合併。理由：這些字是**全體員工會看到的對外文案**，用語一致性與親和度是
Steven 的決策範圍（同「面向全員的文案／命名先拍板」的通則）。流程：開發前把新事件的三格草案
列給 Steven → 拍板 → 才寫進註冊表；PR 描述附上拍板結論供 review 對照。

### 10.4 頁面副標（PageHeader description，2026-07-24）

副標是「這頁大概在幹嘛」的**一句目的提示，不是說明書**。過長的副標在縮小螢幕時會折成 2～3 行、把版面撐歪。

- **一句、單一子句、建議 ≤ 20 全形字**；不要用 `；` 串多個子句。
- **機制／操作說明**（優先順序、停用行為、怎麼查找、匯入步驟）→ 放 **tooltip／內文／docs**，不塞副標。
- **不放 `共 N 筆` 之類計數**——`TablePaginationFooter` 已顯示總數，副標再放一次是重複。
- **版面**（`PageHeader` 已內建，不用逐頁寫）：`<sm` 手機隱藏；`≥sm` 顯示但 `line-clamp-1`——就算某頁寫太長也只佔一行＋省略號，不會折成一坨。
- 好例：「查看、搜尋、建立與編輯使用者」「集中管理所有模組的權限組合」；壞例：「全平台唯一的『哪天上班』真相：每年匯入政府行事曆，颱風假等突發以覆寫處理；出勤等模組依此判斷工作日。」（已於 2026-07-24 全站收斂）。

> 面向全員的文案，改動照「先跟 Steven 拍板」的通則（同 §10.3）。

### 10.5 平台命名（2026-07-24 Steven 拍板）

> 平台對員工有三種寫法，各有各的場合。**定義的權威在 platform repo `CONTEXT.md` 的「產品名」詞條**，
> 本節只管「哪個場合用哪個、什麼不准寫」。

| 寫法 | 用在哪 |
|---|---|
| **Ystravel Portal**（英文字樣） | 登入／忘記密碼／重設密碼頁的視覺區、瀏覽器分頁標題 |
| **永信生活旅遊事業員工入口網**（中文全稱） | 正式落款：登入頁副標、對外公告、未來 Email footer |
| **員工入口網**（中文簡稱） | 日常：內文提及、通知內容、口語 |

- **分頁標題格式 `<頁名>｜Ystravel Portal`**（如 `登入｜Ystravel Portal`）。中文頁名配英文站名是**刻意**的——分頁擠窄時只看得到前段，站名短好辨識（同 `收件匣 | Gmail`）。**這不算「中英混雜」違規，不要來修。**
- **不要在同一塊視覺區給平台第二個類別詞。** 登入頁左側已有「Portal」（主標）與「網站」（副標）；再疊 workspace／gateway／platform 就是三行三個類別詞，讀起來像作者自己也不確定這是什麼。
- **Workspace 不採用**：公司用 Google Workspace（Google OAuth 的 `hd` hint 指 `ystravel.com.tw`），登入頁上就有 Google 登入鈕，同畫面兩個 workspace 會混淆。
- **Platform 只留工程層**（repo `ystravel-platform`、`@ystravel/*`）：它描述「**我們蓋了什麼**」，不是「**員工用什麼**」，不當門面名。
- **登入頁不放沒接資料的假狀態。** 已清掉的實例：`🟢 SSO Live`（假狀態燈）、`Routes 03`（新增模組就過期的寫死數字）、`Roles Synced`（宣稱一個沒在監測的狀態）。裝飾方塊要放就放**真的且不會過期**的事實（現行：`Modules/Unified`、`Roles/Scoped`、`Security/RS256`）。
- 「Portal」**不是** SSO 時代殘留：企業界 Employee Portal 的標準語意就是「員工所有內部功能的統一網站」，不是中轉門；雙軌期（門）與 EIP 退役後（全部功能）兩種形狀都成立。

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
| 眉標 | **分類** | 人事異動通知 | 帳號安全／帳號開通 |
| h1 | **事實**或**要做的事**（完整一句話） | 王小明 職務異動 | 重設您的密碼 |
| 按鈕 | **動作** | 前往系統查看 | 設定新密碼 |

> 通知信的眉標＝§10.3 的「類別」，讀註冊表 `category`（收件匣同字），已含結尾「通知」——
> 模板不再自己補「通知」二字（舊版 `${分組名}通知` 已改）。標題爆長的細節依 §10.3 下放到內文。

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
17. 覆寫了 `hover:`／`active:` 的底色或邊框卻沒補 `not-disabled:`（見 §8.1-1）
18. 有輸入欄位的 modal 手刻 `UModal` 而非用 `FormModal`（見 §6.7）
19. 新增 `notificationRegistry` 事件，其 `category`／`title`／`body` 文案未附 Steven 拍板（見 §10.3）
20. 頁面副標超過一句、塞機制說明、或放 `共 N 筆` 計數（見 §10.4）
21. 表格排序各頁自刻 sortKey/accessors、前端分頁把排序套在切片「之後」、或有拖曳序位的表又給 column sort（見 §5.4.1）
22. 自造平台名稱（在對外文案給平台第二個類別詞：workspace／gateway／platform），或登入頁放沒接資料的假狀態指示（見 §10.5）
23. 新增顏色組合沒過 §2.7 對比度基線（solid 鈕白字未達 4.5、amber 配白字、text-primary 當正文連結色）
24. soft／subtle 的文字色在業務頁自己貼 class，而不是收在 `vite.config` 主題層的 `SOFT_TEXT`；
    或新增自訂半階用插值取中點而非「最淺過標點」（見 §2.7 規則 5、6）
25. 表面底色寫死 `dark:bg-neutral-8xx` 而不走 `--app-surface-*` token；`inset` 直接疊在 `inset` 上；
    或 token 引用 `--color-neutral-*`（明暗都是 gray）而非 `--ui-color-neutral-*`（見 §2.4）
26. 定義列把多值欄位（角色／標籤／分眾）放進兩欄配對，而不是 `sm:col-span-2` 跨整列（見 §5.5.1）
27. UI 改動沒在 **375px** 看過；或用「鎖元素寬度」代替真的調視窗（`sm:` 斷點不會觸發，量測無效）；
    或用 `scrollWidth > clientWidth` 當溢出檢查（元素自己撐大，永遠驗不出來）——見 §4.3
28. 為了「星空底上太淡」逐處貼 `aurora:text-muted`／`aurora:text-toned`——極光的文字提亮
    已收在 `main.css` 的 `.dark.theme-aurora` token，全站自動生效（見 §9.2）

---

## 14. 已知落差與待辦

- **`reka-ui` 分段日期輸入顯示不補零**（`2023/9/23`）：來自套件內部寫死的 `defaultPartOptions`，無 prop/locale/theme 可改，要一致就得 patch 相依套件。判斷不划算——那是「編輯中的輸入框」不是「資料呈現」，兩者不會並排出現。已記在 `DateInput.vue` 檔頭，升版時再看。
- **icon 執行期從 Iconify API 抓**：內網部署前要打包 lucide collection 進本地。
- **Base token 是否抽成共用 npm 套件**：現階段先文件對齊，未來多系統重複痛了再抽（技術雷達 `@ystravel/ui`）。
- **全站表單巡檢**（哪些 modal 還沒接 schema、哪些後端欄位錯誤還在走 toast）＝獨立一輪，未排程。
- **業務頁直接寫 `text-{語意色}` 共 24 處／14 檔，尚未逐一判定**（2026-07-24 掃出，與 §2.7 規則 5
  是同一個病灶但**不能一刀切**）：`text-primary` 8、`text-error` 8、`text-warning` 5、`text-success` 3。
  難處在於**門檻不同**——其中一部分是圖示（`PageHeader`／`ModuleRail`／`TableLoading` 的 spinner，
  UI 元件門檻 3.0），另一部分是真的文字訊息（`ConfirmModal` 警語、`AccountSecurityPage` 錯誤提示、
  `WorkCalendarPage` 警示，門檻 4.5）。要逐處看背景與用途才能決定改哪些、哪些列例外。
  ⚠️ 注意 `text-primary` 對白底是 2.98，**連 UI 元件的 3.0 都差 0.02**（同 focus ring 那條例外）。
- **vaul（UDrawer）殘留 transform 在 Windows 高 DPI 下文字糊**：只是桌面 devtools 模擬手機的假象、**真機正常**，不要為此加 `transform:none` 全域 hack。
- **交易信的藍 `#2563eb` 不在平台語意色盤**（§2.2 的藍＝sky）。2026-07-23 Steven 決定維持現況——它是信件裡「要你動手」的專用色、不與 app 的 info 混用；日後要對齊 sky 隨時可換，兩封交易信一起改即可。
- **信件 amber／rose 兩級尚未落地**：規則已定（§11.1），色值等第一封警告類／警示類信件出現時比照 §2.2 定。
- ~~solid 主鈕白字對比未達 WCAG AA~~ **已結案（2026-07-24）**：light base 維持 550
  （2.98，Steven 看過過標候選後拍板好看優先，列 §2.7 刻意接受例外）、hover/active 改
  650/750；dark 改亮底深字（450，全過標）；error 鈕同輪明暗統一 600（light 原 3.75 → 4.53 ✅）。
  規範落 §2.3＋§2.7，量測過程封存於 platform `prototype/primary-button-contrast` 分支。
- ~~soft／subtle 文字對比未達 WCAG AA~~ **已結案（2026-07-24）**：徽章／Alert／soft 鈕的文字
  在 light 一律壓 700 階（2.38～3.56 → 4.55～5.21），底色與 dark 不動。600 階經實測無效
  （0/5 過標）、650 半階只有 error 過，故直接到 700。規則落 §2.7 規則 5＋§13-24。
  這是比主鈕更嚴重的一區——**徽章承載狀態資訊（在職／邀請中／啟用），不是裝飾**。

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

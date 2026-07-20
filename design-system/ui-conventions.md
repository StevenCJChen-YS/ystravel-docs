# UI 慣例與實作雷點（UI Conventions & Pitfalls）

> **定位**：本檔承接原本記在 AI 分身記憶（my-agent `MEMORY.md`）裡的 UI 範式、拍板決策與踩坑雷點——2026-07-16 知識分層整理時遷移至此。
> **與 [foundation.md](./foundation.md) 的分工**：foundation＝設計系統規範本體（token、色階、元件規格）；本檔＝**範式決策的時間線、理由、與實作層的血淚雷點**。兩檔衝突時以最新拍板為準並回頭更新。
> 📁 路徑基準（2026-07-16 Phase 1 整併完成後更新）：本檔 `src/shared/ui`、`src/shared/composables` 等路徑對應平台的 **`apps/portal/src/...`**（`ystravel-platform`）；早期以 `AuthPortal` 為主詞的敘述皆指同一份已遷入 platform 的程式碼。

---

## 1. Steven 的後台 UI 總偏好（2026-07-10~11 視覺巡檢定調）

習慣 **EIP/Vuetify 式後台**——資料用 `UTable`（表頭＋分頁 footer：共N筆/pagination/每頁筆數）、建立/編輯開 **modal（dialog）** 不是行內輸入、列操作用 icon 鈕；**白卡浮灰底**（light 卡片一定白）；灰階已定案 **gray**（light=gray、dark=zinc 明暗混搭）；**圓角偏小**（基準 `--ui-radius` 0.375rem＝sm6/md9/lg12px；控件 btn/input/select 另走尺寸階梯 xs2/sm4/md6/lg8/xl10px）；**深色表頭白字**、表格內縮帶框斑馬紋。給參考圖時是「類似排版、仍要走 NuxtUI＋semantic token」，不要照抄配色。

## 2. 篩選排版範式（2026-07-12 定調）

後台列表頁的篩選要能隨「篩選數量」縮放，別一套硬套：

- **篩選少（≤4，如 AuthPortal 使用者頁）**＝filter bar 一列、跟動作（新增/欄位）一起貼在**表格正上方**，即時篩選。**⚠️動作鈕別放到離表格很遠的頁首**——Steven 對「新增鈕離表格太遠」明確有顧慮，寧可動作也貼表格（比照 Vuetify data-table CRUD 範例，而非 Shopify Polaris 的頁首 primaryAction）。
- **篩選多（5+，如 EIP 的 `B2CStatisticsManagement`／`EmployeeManagement`／`evaluation-management`）**＝獨立**篩選卡片**（filter card：grid 排多欄、RWD 換行、可收合、右下「查詢／清除」鈕、**手動查詢不即時打 API**——資料量大時即時篩選太重）。EIP 已驗證、Steven 熟悉，新系統沿用其「NuxtUI 現代化版」。
- 兩態用**同一套共用篩選組件**（少→平鋪、多→卡片），別每頁各寫。EIP 參考檔：`GInternational-Frontend/src/pages/B2CStatisticsManagement.vue`（filter card＋`search-label`＋查詢鈕）。
- **「搜尋為主＋低頻條件收 FilterPanel（alwaysVisible）」＝資料量大/欄位多頁的標準範式**（稽核頁已落地，CRM 客戶/訂單沿用）。

## 3. 篩選 RWD 收合範式（2026-07-13 落地，斷點 `<md`）

列表頁篩選**桌面平鋪**（filter bar 貼表格上方、即時篩選），**手機＋平板（`<md`＝768px 以下）收進 `UDrawer` 抽屜**——工具列改放「篩選 (N)」鈕（N＝作用中篩選數 `activeFilterCount`），抽屜內改**草稿值**（`draftStatusFilter` 等），按「套用」才寫回真篩選（`applyMobileFilters`／`resetMobileFilters`）。桌面下拉 `hidden md:flex`、小螢幕鈕 `md:hidden` 切同一套 filter。**斷點 07-13 從 `sm` 上調 `md`**：平板 sm~md 走桌面平鋪會擠。**分頁 footer 同步 RWD**：小螢幕（`<sm`）2 欄 grid＝分頁置中橫跨上排、下排「共N筆」左＋「每頁」右；桌面 `sm:flex justify-between` 一列（`TablePaginationFooter.vue`）。

## 4. UTable 密度（2026-07-12 查證）

NuxtUI `UTable` **沒有內建 density/compact prop**（table theme 只有 sticky 變體）；表頭/列高＝`:ui` 的 `th`/`td` padding（官方預設 th `py-3.5`、td `p-4`）。要「舒適/精簡」切換得自建（綁兩組 padding）。表格外觀要全站一致一律抽 `vite.config` 的 `table` theme，別逐頁散寫 `:ui`。

## 5. 字體與字重（2026-07-13 定案）

- **全系統字**：Win 微軟正黑體、Mac 蘋方 PingFang TC、非 Win/Mac 自架 Noto 保底，**零自架 webfont**（實作在 `main.css --font-sans`）。原因：自架 webfont（Noto／台北黑體／Inter）在 Windows 無 hinting 都糊，系統字最清晰（Steven 07-13 完整試錯驗證）。**別再提議自架中文 webfont 追求「多字重」**——系統中文字只有 400/700，跨平台不齊，EIP（微軟正黑體）證明後台 400/700 就夠。
- **字重匹配有方向**（Steven 實測糾正）：`medium`(500) 往下靠 400（細）、`semibold`(600) 往**上**靠 **700**（粗）——別再說「semibold 落回 400 不會變粗」。UI 字重規則：**標題 `semibold`、一般內容 `normal`、避免 `medium`**（medium 對系統字＝400、對 Inter＝真 500，是唯一造成中英不一致的一階）。

## 6. Dark mode 規則

- **浮層（2026-07-13 定調）**：dark 下陰影幾乎看不見 → 上層元素（modal／popover／下拉 content／表格卡）用 **`ring-1 dark:ring-white/25`（hairline）＋深色明度分層**（頁底 `neutral-950` < 卡/modal/側欄 `850`，表格內回 `950`、斑馬偶列 `900`；輸入框 dark 底 `950`）；switch thumb dark 白。**新 dark UI 一律照這套做浮起感，別靠 `shadow`。**
- **【ALWAYS】dark 裝飾效果不准動到 light（2026-07-15）**：裝飾層元素用 `hidden dark:block` 只在 dark 掛載；效果 class 一律 `dark:` 前綴（light 有 `bg-white` 要 `dark:bg-transparent` 蓋掉再疊漸層）。**改完務必切 light 實測 computed**（bgColor 實心、backdrop `none`、border 還在、裝飾層 `display:none`）才算數。曾犯：大廳毛玻璃兩模式都套，light 白卡被改掉。

## 7. 品牌主色＋明暗色階（2026-07-14 定案，規範本體在 [foundation.md](./foundation.md) §1/§5）

**Auth primary = teal**（推翻先前 blue）；CRM=Violet、EIP 待定；各系統只改 `primary` 別名（semantic token，`vite.config` 一行全連動）。**明暗色階**：語意色 light 深一階、**dark=400**（白底要深、深底要亮）——`main.css` `:root`＋`.dark` 補 -400。light 階：**primary(teal)/warning(amber)＝自訂 550 半階**（oklch 插值；自訂半階要同時接 Tailwind `--color-*-550` 與 Nuxt `--ui-color-*-550` 兩套別名）、success/info＝600。**固定色階例外**：solid 主鈕 `bg-primary-550`、switch checked `bg-primary-500`（明暗一致）。**表格資訊標籤用 `info`(sky) 不用品牌 primary**（否則搶眼）。⚠️`vite.config` 改動要**重啟 dev server**（build-time，非 HMR）。

## 8. 管理後台共用元件＋實作雷點（2026-07-14~15，元件規格本體在 [foundation.md](./foundation.md) §11）

平台 `apps/portal/src/shared/ui/` 共用元件（改任何管理頁 UI 前先看有沒有現成的，別重造），**2026-07-16 PR3 起依類型分子資料夾**：
- `base/`：有加值的原始元件包裝 `AppInput`/`AppSelect`/`DateInput`/`ToolbarButton`/`ColorModeToggle`
- `table/`：`TableCard`/`TableLoading`/`TablePaginationFooter`/`TableSortButton`/`ColumnVisibilityMenu`/`DataToolbar`/`FilterPanel`/`OrderManager`
- `overlay/`：`FormModal`/`ConfirmModal`　`layout/`：`AppPageLayout`/`PageHeader`/**`SurfaceCard`**/**`AuroraBackdrop`**（2026-07-17 統一殼後 **`LobbyShell`/`AccountShell` 退役**，見 §13）
- `nav/`：`UserMenu`/`ListPanel`/`ListItemButton`/**`ModuleRail`**（2026-07-17 **`MySystemsMenu` 退役**，「我的系統」併入統一殼）　`card/`：`SystemAppCard`/`UserIdentityCard`　`feedback/`：`EmptyState`/`PasswordStrengthMeter`　`table/` 另加 **`TableFrame`**（見 §13）
- import 一律走 `@/shared/ui/<類>/<元件>` 絕對別名（`@/` = `src/`）。

**血淚雷點**：

1. **icon-only 鈕三雷（2026-07-14 一天踩兩次，Steven 要求記牢）**：(a) `<sm` 收 icon-only＝文字 `max-sm:hidden` 藏＋aria-label/title 保留，**別用 `max-md:size-10` 硬縮**（文字還在會擠爆）；(b) 文字藏了但按鈕仍留文字鈕左右 padding → 變橫長方形，**要補等距 padding**：md 尺寸鈕＝`max-sm:p-2.5`（36×36，同全站 square md 規格）；(c) 固定尺寸放大觸控目標時**必須同時 `:ui="{ base: 'justify-center p-0' }"`**（UButton 預設非置中）。**口訣：藏字之後問三件事——padding 等距了嗎？置中了嗎？aria-label 還在嗎？**
2. 篩選收合用 `FilterPanel`（RWD 分流：`<sm` 底部 UDrawer／`≥sm` UModal）——vaul UDrawer 殘留 transform 在 Windows 高 DPI 下文字糊，**只是桌面 devtools 模擬手機的假象、真機正常**，別加 `transform:none` 全域 hack（Steven 不要動官方）。
3. 深色表頭「可排序欄」用 `TableSortButton`（button theme 全域 font-normal 會蓋掉 th 的 font-semibold）。
4. `lg:grid-cols-[Npx_minmax(0,1fr)]` 在 `<lg` 塌單欄時子項要補 `min-w-0`（否則表格撐破欄寬、整頁橫向溢出）。
5. `useSortable` 放 modal 內要 `watchElement:true`（body 晚掛載）。
6. **管理表格欄寬＝auto 佈局＋百分比 th（名稱/說明欄不設寬拿剩餘），勿加 `table-fixed`**——會鎖死 100%，多開欄位互相硬擠疊字；auto 佈局配 TableCard 的 `overflow-x-auto` 自然橫向捲動（2026-07-15 三頁套了又拆；例外＝選項頁 4 欄全開仍 <100% 可留 fixed）。
7. 側欄 `<xl`(1280) 自動收合。

## 9. 零星硬規則（Steven 明確要求，全站嚴格執行）

- **空值一律 `—`（em dash）**（2026-07-14）：`?? '—'` 或 `<span class="text-dimmed">—</span>`，別再出現 `-`／`_`。
- **modal 說明/備註欄**（2026-07-14）：獨占一列 → `<UTextarea :rows="3" class="w-full" />`（theme 已全域：圓角階梯/手機16px/`resize-y`）；跟單行欄位同排 grid → 維持 AppInput（textarea 高度突兀，例＝使用者 modal 備註，Steven 補拍板）。
- **顯示給使用者的資料用「中文名」不露 code**（2026-07-15）：有 code＋name 的實體畫面一律顯示 name（Auth payload 已加 `roleNames` 與 `roles` 同序）；emoji 區塊裝飾偏好換 lucide icon。
- **前端分層（2026-07-16 修訂；權威＝平台 CLAUDE.md 鐵則 4）**：分層責任分明、別為包而包——
  - 視覺調校 → `vite.config` component theme（**別包**）；重複 UI → `shared/ui/`（依類型分子夾）；重複狀態邏輯 → `shared/composables/`（useConfirm/useEditModal…）。
  - **`App*` 包裝只在「補官方元件缺的功能」或「統一 app 慣例」時才做**（如 `AppInput`/`AppSelect` 的 clearable）；純換名字、對全域 theme 無加值的薄包裝**不做**，theme 足夠且無重複時原始 `U*` 可直接用。
  - **元件歸屬**：跨模組共用 → `shared/ui/`；單一模組專屬 → `modules/<模組>/components/`。寫任何頁面前先問「第二個模組/頁也會用嗎？」會→shared、不會→模組內。
  - 已落地例：table theme 深表頭一處改全站、`TableLoading.vue`、`useEditModal.ts`（關閉延後清 target，修「對話框收場先變矮」）。
- **Icon 一律用 lucide（`i-lucide-*`），禁止混用其他 icon 套件**（2026-07-17 Steven 明確要求）：不同套件筆觸/比例不一致（tabler 的盾牌比 lucide 寬圓，混排一眼突兀）。lucide 找不到想要的 icon 時先上 lucide.dev 搜（新 icon 常已收錄，如 `shield-cog-corner`），真沒有再帶著候選回來問 Steven，不要自行混套件。⚠️ 附帶：icon 目前是執行期從 Iconify API 抓的（斷網會開天窗），正式部署內網前要把 lucide collection 打包進本地（部署階段待辦）。

## 10. 順序（order）管理範式（2026-07-14 定調，落地公司/部門/選項頁）

「顯示順序」是**主檔自己的屬性、在該主檔管理頁維護**（Salesforce picklist／Odoo sequence 同理），存一份、所有 consumer 讀同一份，**別每個表單各自定義順序**。**⚠️「排序」中文兩義**：sort（升降冪）vs order（序位）——Steven 講「排序欄位/順序怎麼排」是**後者**（2026-07-14 誤解過一次整套 revert）。三原則：①`order` 一律「相對同層兄弟」②UI 用拖曳（`OrderManager.vue`＝useSortable 握把拖＋「依名稱排序」一鍵＋批次儲存/取消，非 autosave）③modal 不放 order 數字欄，新增自動排最後。**給人看 1-based（`order+1`），0-based 只留程式內。** 後端 reorder 收「排好的 id 陣列」→交易重編＋audit。未做：持久 sort mode。

## 11. 資訊架構：「個人/帳號」與「管理」兩世界（2026-07-12 拍板 B 案；07-16 帳號中心殼；**07-17 全併統一殼**）

- **【2026-07-17 統一殼】`LobbyShell`／`AccountShell` 退役、全部住進 `AppShell`**：左最窄軌 `ModuleRail`（系統切換，依權限顯示；平台區塊＝「員工入口網」icon `plane-takeoff`，rail 不放首頁鈕、點 logo 回首頁、rail 底小分隔線後放帳號中心 icon＝`railFooter`）＋單模組側欄（nav 各模組自帶 `nav.ts`，彙整於 `shared/nav/module-nav.ts`，旗標 `railHidden`/`railFooter`）。`/my-apps` 大廳併入 `/home`、「我的系統」選單移除。手機＝**Discord 式雙欄抽屜**（rail `<lg` 進 slideover，點模組先切右側清單不跳頁）。
- **個人／帳號仍收右上/側欄底頭像選單**（`UserMenu`，「找自己＝點頭像」全站一致）。帳號中心三分頁（個人資料/帳號安全/個人化）改由 `AccountLayout`（＝`AppShell`＋`SurfaceCard`）承載、導覽走側欄（`modules/account/nav.ts`），**推翻 07-16 專屬 AccountShell**。分頁名「個人化」（頭貼+背景），「外觀設定」語意已由 §13 主題系統的「介面外觀」承接。
- **SSO 特性**：個人資料多為公司資料唯讀（部門/職稱/角色由管理員/未來 HR 主檔管），本人能改的只有大頭貼＋背景（顯示名稱政策未定案；給的話稽核/官方列表仍用本名＋帳號）。

## 12. 角色體系顯示與代碼慣例（2026-07-15 拍板，決策本體在 [features/auth-role-management/](../features/auth-role-management/prd.md) example-mapping 決策表）

①**系統角色最少化**——只有 `SUPER_ADMIN`（全系統角色不掛 AUTH_ 前綴）是系統角色（不可刪/停用/改權限）；出廠角色（AUTH_ADMIN/AUTH_HR/CRM_*）＝起手範本、可改可刪，自建角色同等地位。②**授權/保護判斷看 permission、別看 role code**（超管 guard bypass 唯一例外）。③appCode=null 的 UI 用詞＝「**全系統**」。④前端超管代碼一律 import `SUPER_ADMIN_ROLE_CODE`（auth-api.ts），別散寫字串。⑤改角色代碼類 DB 遷移走 **seed 前置冪等收斂**，跑完要重新登入（舊 token 帶舊代碼）。

## 13. 主題系統：深淺色下各自可選主題＋aurora 毛玻璃（2026-07-17 系統化，取代舊 LobbyShell 硬寫）

**架構（可擴充、非硬寫）**：深淺色「模式」之下各自可選「主題」。深色＝**極光（預設）／預設（純黑）**、淺色＝預設 only；存 localStorage（`ystravel.platform.theme.dark/light`，每台裝置各記，帳號級同步＝未來 `auth-user-preferences`）。實作＝`useAppTheme.ts`（watch 掛 html `.theme-aurora` class）＋Tailwind 4 `@custom-variant aurora (&:where(.dark.theme-aurora, .dark.theme-aurora *))`（`main.css`）。個人化頁「介面外觀」區出兩個主題 select。

- **【鐵律】保證深色「預設」像素級等同原設計、淺色不變**：元件**只「加」`aurora:` 前綴樣式、絕不改 base class**（`aurora:` 變體本身有 `.dark` gate）。未來要加新主題＝新 class＋新 `@custom-variant`，同法炮製——**別回頭硬寫某個 shell**（LobbyShell 舊做法的反面）。
- **⚠️ 別再用 `dark:` 做「只有某主題才要」的裝飾**：`dark:` = 所有深色（含「預設」純黑）都套 → 破壞上面的保證。極光專屬效果一律 `aurora:`。曾犯：AccountLayout／AppPageLayout 的毛玻璃原用 `dark:`，連純黑也玻璃，2026-07-17 全改 `aurora:`。（呼應 §6 的「dark 裝飾不准動 light」，多一層「aurora 裝飾不准動預設深色」。）

**aurora 色值＝單一 token 區塊（唯一來源，改一處全站生效）**：所有毛玻璃/半透明值集中在 `main.css` `.dark.theme-aurora` 的 13 個語意 token，元件一律 `aurora:bg-[var(--aurora-panel)]` 引用、**不准再散寫 `neutral-900/38`、`white/8` 這類數字**（Steven 2026-07-17 明確痛點：「到處調色未來難維護」）。token 用 `color-mix` + 原本同組 Tailwind 色變數＝與舊 utility 逐位元一致。

| 群組 | token | 用途 |
|---|---|---|
| 表面 | `--aurora-panel` `rail` `overlay` `field` `inset` `header` | 卡片/側欄/導覽列/rail/抽屜popover/input/表格內層/表頭 |
| 狀態 | `--aurora-row-alt` `hover` `hover-soft` `active` | 斑馬列/列hover/連結hover pill/active pill |
| 邊框 | `--aurora-border` `border-soft` `divider` | 卡片框/內層框/分隔線 |

**卡片/表格外殼一律走共用元件（別再各頁貼 chrome 長字串）**：
- **`SurfaceCard.vue`**（`shared/ui/layout/`）＝毛玻璃卡片外殼（圓角/邊框/底/陰影＋aurora 玻璃、預設深色與淺色實色）；`as` 控 section/div、`rounded` prop（大卡 rounded-xl），版面差異用 class 疊加（Vue 自動併入根元素）。
- **`TableFrame.vue`**（`shared/ui/table/`）＝表格內縮帶框＋斑馬紋＋列 hover＋`#footer` slot（分頁選配）。`TableCard` = `SurfaceCard`＋`TableFrame`＋分頁的組裝。
- **`AuroraBackdrop.vue`**（`shared/ui/layout/`）＝全視窗星空＋極光光暈固定背景層，`hidden dark:block` 且只在極光渲染（AppShell 條件掛載）。

**按鈕的 aurora 收斂＝vite.config button theme 一處（別逐頁調）**：neutral **outline**（篩選/調整順序等工具列鈕）底走 `--aurora-field`（與 input/select 同 token 同手感）、hover 只亮邊框、active 疊白；neutral **ghost**（導覽列展開/信箱等）hover 由實心 `bg-elevated` 改 `--aurora-hover` 半透明白。primary solid 彩色鈕不需處理（彩色在星空上本來就對）。

**背景層位置＝App.vue 根組件（勿放 AppShell）**：各頁自包 `<AppShell>`＝換頁整殼重掛載，背景放殼裡動畫每次歸零（2026-07-17 踩過）。`AuroraBackdrop` 固定層放 `App.vue`（app 生命週期只掛載一次、換頁動畫連續），gate＝`darkTheme==='aurora' && route.matched 有 requiresAuth`（登入等公開頁不出現）。

**維護心法（往後照走）**：改 aurora 顏色→只動 `main.css` token 區塊；新卡片/表格→用 `SurfaceCard`/`TableFrame`/`TableCard`，別貼長字串。這正是 CLAUDE.md 鐵則 4（視覺調校走 theme seam、重複 UI 抽共用元件）的延伸。

## 14. 錯誤顯示三層規範（2026-07-19 拍板）

> 全站表單/操作的錯誤呈現只有一套，按「錯誤在哪裡被攔下」分三層。原則：**能預防就不報錯；報錯盡量貼著欄位；toast 是最後手段**。

1. **預防層（UI 直接擋）**：能用介面讓錯誤發生不了就先做——日曆反灰不可選日期（`DateInput` `min-value`）、下拉只列合法值、按鈕 disabled。最好的錯誤訊息是根本不會出現的錯誤。
2. **前端驗證層 → 欄位下方紅字**：必填/格式/範圍等前端就知道的，走 `FormModal` 的 `:schema`（valibot）→ UForm 行內驗證（欄位下方紅字＋框變紅）。schema 管不到的欄外狀態（如 AdminUsersPage 的角色到期日）→ 自寫 `xxxError()` 函式即時算，紅字（`text-xs text-error`）放欄位正下方＋submit handler 入口擋。
   - **紅字左緣對齊 input 起點，不是容器左緣**（2026-07-19 Steven 拍板）：label 在左的行內佈局（如「臨時｜到期日｜input」grid），紅字用 `col-start-N` 對到 input 那欄——錯誤講的是「這個欄位」，貼欄位起點歸屬感才對；靠容器左緣會跟 label 搶同一條線、像整張卡在報錯。UFormField 行內錯誤本來就貼 input，這條讓手寫紅字跟它一致。**全站有類似手寫錯誤訊息的位置都要照此檢查**（併入表單巡檢輪）。
   - **瀏覽器原生驗證氣泡全站禁用**：`FormModal` 的 UForm 已掛 `novalidate`（2026-07-19）。原生氣泡位置不可控、各瀏覽器長相不一、還會錨到分段輸入的隱藏 input 上（Chrome 氣泡指到 modal 外），且它搶在 schema 驗證前跑＝行內驗證永遠沒機會顯示。新表單不准依賴原生 `required`/`min`/`pattern` 提示。
3. **後端錯誤層**：送出後才知道的，分兩種——
   - **對得到欄位的**（email 已存在、名稱重複…）：表單還開著時盡量映射回該欄位下方紅字（使用者眼睛在表單上，toast 在角落容易錯過、也不知道該改哪欄）；映射不了才 toast。
   - **對不到欄位的**（500、網路、權限、整批操作失敗）：toast（error 色＋`i-lucide-circle-alert`），文案走各 api 層的錯誤中文對照表（`auth-api.ts`/`hr-api.ts` `ERROR_MESSAGES`）。

**現況與待辦**：novalidate＋到期日行內驗證已落地（AdminUsersPage 角色到期日＝欄外狀態驗證的參考實作）。**全站表單巡檢**（哪些 modal 還沒接 schema、哪些後端欄位錯誤還在走 toast）＝獨立一輪，未排程。

## 15. 控件狀態與提示的四條硬規則（2026-07-20 Steven 實測回報，逐條驗證後定案）

同一輪回報裡浮出的共同病灶：**規則沒有定在元件/主題層，靠各頁自己記得**——會漏、會不一致。
以下四條都已在共用層落地，新頁面不必也不該再自貼。

1. **停用的控件不准對滑鼠有任何回應**：`hover`/`active` 一律掛 `not-disabled:`。
   停用還會變邊框，使用者會以為點得動。`button` theme 原本就這樣寫，但 `input`/`textarea`/`select`/`selectMenu`
   四個漏了（`vite.config` 已補齊，編譯結果為 `&:not(*:disabled){&:hover{…}}`）。
   ⚠️ `vite.config` 是 build-time，改完要重啟 dev server。

2. **警告色只留給「真正需要行動」的事**：狀態資訊用欄位＋篩選呈現，不要用常駐警告。
   案例＝權限目錄原本常駐黃底「有 N 條權限未掛給任何角色」，但「權限碼還沒被使用」是正常狀態
   （還沒有實習生就沒人需要 `HR.ORG.READ`）。天天顯示不是問題的事，只會訓練使用者無視所有警告。
   降級為「使用中角色」欄位＋工具列「未使用 (N)」篩選鈕＝查詢工具，需要時查得到、平常不吵。

3. **連動下拉：父層沒選就 disabled，並用 placeholder 講下一步**（例：沒選公司時部門顯示「請先選擇公司」）。
   原本是可點開但空無一物，使用者分不出是沒資料還是壞了。
   **同一條規則要同時套到 modal 表單與篩選列**，且父層變更/清空時要一併重置子層——
   否則清掉公司後部門條件還在作用、下拉卻已停用，使用者看到被篩過的清單卻找不到是哪個條件在擋。

4. **輸入框內緣的小圖示鈕用原生 `title`，不要 `UTooltip`**：浮層會蓋住 26px 的小鈕，
   滑鼠移過去先開浮層、再按下就點不到（實測真實滑鼠點擊完全無反應）。無障礙由 `aria-label` 顧。
   另外這類鈕**一律加 `@mousedown.prevent`**：不擋的話按下瞬間輸入框先失焦，
   表單會用「按鈕還沒改到的舊值」跑驗證而閃出錯誤（EmailInput 補網域鈕實例：
   按下去先跳「Email 格式不正確」，要再離開欄位一次才消失）。

> **通則**：共用元件的規則要在元件或主題層定死。`DateInput` 的 `w-full` 就是反例——
> 原本要各頁自己貼，5 個使用點有 2 個貼、3 個漏，漏的就變半截寬；已改為內建
> （同 §8 icon 尺寸「各頁自貼補丁、新頁面漏貼就破功」的教訓）。

## 16. 日期與時間顯示格式（2026-07-20 Steven 拍板）

畫面上一律 **`2026/07/17`（斜線、補零）** 與 **24 小時制 `15:04`**；
秒數只給稽核日誌。唯一出口＝`shared/lib/datetime.ts`，**不要在頁面裡自己寫格式化**。

| 用途 | 函式 | 樣子 |
|---|---|---|
| 純日期欄位（到職日、離職日） | `formatDate` | `2026/07/17` |
| 時間戳只要日期那面（建立日期） | `formatTimestampDate` | `2026/07/17` |
| 時間戳（最後登入、加入時間） | `formatDateTime` | `2026/07/17 15:04` |
| 稽核日誌 | `formatDateTimeWithSeconds` | `2026/07/17 15:04:05` |
| 表單預填今天、匯出檔名 | `todayIsoDate` | `2026-07-17`（ISO，不是顯示用） |

選這組格式的理由：

- **斜線**不是技術考量而是使用者考量——同事的 Excel、科威、政府系統都是斜線，無痛上手優先。
- **補零**是為了對齊：`2026/7/7` 和 `2026/12/17` 一長一短，整欄掃下來會歪。
  顯示日期的欄位**一併加 `tabular-nums`**，等寬數字才真的對齊（只補零不夠）。
- **24 小時制**：`下午3:04` 是 `zh-TW` locale 的預設，但「上午 12 點」有歧義，且欄寬不一。
- **不用民國年**：只有對政府的表單需要，內部系統用民國會讓排序、比對、匯入都多一層換算。

### 三條硬規則（都是踩過的坑）

1. **資料層維持 ISO，只有送到人眼前的最後一哩才轉。**
   資料庫、API、`DateInput` 的 `v-model`、匯出檔名一律 `YYYY-MM-DD`；不要把斜線格式往回灌。

2. **「純日期」不准經過 `new Date()`。**
   `new Date('2023-09-23')` 是 UTC 午夜，在 UTC 以西的時區換算回當地會變前一天。
   到職日這種值沒有「時間點」的概念，用字串換字串。所以 `formatDate` 和 `formatTimestampDate`
   是**兩個不同的函式**，不是同一個加參數。

3. **要「今天」就用 `todayIsoDate()`，不要 `toISOString().slice(0, 10)`。**
   後者是 UTC 的今天，台灣（UTC+8）00:00–08:00 之間會少一天。
   實際踩到：離職生效日預填成昨天、匯出檔名標成昨天（前後端各一處）。
   後端沒有這個共用函式，改用 `Intl.DateTimeFormat('en-CA', { timeZone: 'Asia/Taipei' })`
   ——輸出剛好就是 `YYYY-MM-DD`，且不依賴容器時區（容器預設 UTC）。

### 額外兩個教訓

**「排他邊界」這種存法，換算函式一定要共用。** 角色到期日存的是「隔天 00:00」，
顯示時要換算回「最後有效那天」。原本帳號中心換算、稽核日誌直接印邊界值，
同一筆角色兩頁差一天（選 12/31 會顯示「效期至 2027/1/1」，看起來差一年）。
凡是「存的值 ≠ 使用者選的值」，換算就必須是**唯一一份**，不能靠各頁記得。

**已知落差（查證後決定不修）**：`reka-ui` 2.10.1 的分段日期輸入顯示不補零的 `2023/9/23`。
段落文字來自 `useDateFormatter` 的 `defaultPartOptions`（`month`/`day` 寫死 `'numeric'`），
且 `createContentObj` 呼叫 `formatter.part()` 時不帶 options——沒有任何 prop、locale
或 theme 改得動，要一致就得 patch 相依套件。判斷是不划算：那是「編輯中的輸入框」不是
「資料呈現」，兩者不會並排出現。已記在 `DateInput.vue` 檔頭，升版時再回頭看。

## 17. 文案用字（2026-07-20 拍板）

### 第二人稱一律「您」

後端信件範本（`mail-templates.ts`）本來就全用「您」，前端則是混的——
2026-07-20 把前端 9 處使用者看得到的「你」統一過來。同一頁同時出現兩種稱呼很難看
（帳號安全頁一度上面寫「**你**目前使用 Google 登入」、下面寫「若不是**您**本人的紀錄」）。

**程式註解不受此限**：那是寫給開發者看的，維持「你」比較自然，改了只是製造 diff 雜訊。

### 必填星號：只有「表單裡有可選欄位可以對比」時才標

`UFormField` 的 `required` 是給使用者「這欄非填不可、那欄可以跳過」的區別資訊。
**整張表單每欄都必填時不要標**——整排星號傳達不了任何訊息，只是視覺噪音。
登入頁、重設密碼頁、修改密碼 modal 都屬此類（Google／GitHub／Microsoft 的登入頁也都不標）。

反過來，員工表單那種有「備註（選填）」的表單就該標，而且**標了就要在 schema 裡真的驗**
——2026-07-20 巡檢抓到「聘僱類型」標了 `required` 卻沒進 schema，一直靠預設值掩護。

### placeholder 要舉例，不要複述欄位名

代碼型欄位尤其重要：「科威員編」配「例如：Y001」、「分機號碼」配「例如：2700」，
使用者才知道格式長什麼樣。`placeholder="請輸入科威員編"` 等於什麼都沒說。
（密碼類欄位無法舉例，屬例外。）

## 18. 圓形裁切一定要有底色（2026-07-20，頭像細邊事故）

**任何 `rounded-full` + `overflow-hidden` 的容器都必須有背景色；有 `ring` 時，背景色要與 `ring` 同色。**

現象：名片的圓形頭像，**上半圓內側**浮出一條很淡的深邊；把 `rounded-full` 拿掉就消失。

原因：圓形裁切的邊緣是反鋸齒的，那一圈半透明像素必須跟「底下的東西」混色。
容器沒有背景色時就混到後面的內容——名片頭像跨在背景照片與白色卡片的交界上，
所以**只有上半圓（後面是深色照片）看得見**，下半圓混白卡片看不出來。

`UAvatar` 的根元素本來就帶 `bg-elevated`，所以全站用它的地方從來沒有這個現象。

### 直接的結論：頭像一律用 `UAvatar`，不要手刻 `<img class="rounded-full">`

事故當下全站三個頭像，只有名片是手刻的，也只有它有問題。已改為統一用 `UAvatar`。

### 除錯時的教訓（比這條 CSS 規則更重要）

第一輪誤判成「圖檔底色 `#f4f4f4` 與白色 `ring` 的色差」，講得很肯定卻沒先驗證。
Steven 一句「我把 `rounded-full` 拿掉顏色就正常了」直接推翻——**若是色差，方形版
應該一樣看得到邊**。

真正該問的第一個問題是「**跟正常的那個有什麼不一樣**」：同一頁有兩個沒問題的頭像，
比對它們與出問題的那個，差異一眼就能收斂到「有沒有底色」。從機制去猜繞了三輪，
從對照組去找一輪就到。有可運作的對照組時，先比對，再談機制。

## 19. Tooltip 一律 `UTooltip`，只有一個例外（2026-07-20 全站統一）

全站原本原生 `title` 與 `UTooltip` 混用，同一個功能在不同頁面的提示長相不一樣。**一律用 `UTooltip`。**

**唯一例外＝輸入框內緣的小圖示鈕**（見 §15-4）：浮層會蓋住 26px 的小鈕，滑鼠移過去先開浮層、
再按下就點不到。這類保留原生 `title`，並**在原地寫下理由**，否則下一輪巡檢又會有人「順手統一」把它改回去。

### 官方主題會讓浮層自己截字，必須全域改掉

Nuxt UI 的 tooltip 主題把 `text` 設成 `truncate`，浮層只有單行、超過就是 `…`——
**提示本身被截斷等於這個提示沒有用**。全域 theme（`vite.config`）改成可換行＋限寬：

```ts
tooltip: {
  slots: {
    content: 'rounded-[2px] h-auto max-w-xs whitespace-normal',
    text: 'whitespace-normal',
  },
}
```

`h-auto` 不能漏——原主題有固定高度，只放開 `whitespace` 第二行會被切掉。

### 兩個踩過的坑

1. **`vite.config` 的 `ui` 設定裡同名 key 會整個蓋掉，而且是靜默的。**
   當時已經有一個 `tooltip` 區塊（設 `rounded-[2px]`），我又加了第二個 —— JS 物件取最後一個，
   原本的圓角就沒了。TypeScript 會報 `TS1117`（duplicate key），**看到這個錯不要只改名字閃過去，
   要去把兩個區塊合併**，否則等於默默丟掉先前的設定。

2. **把既有元件包進 `UTooltip` 會讓它整個失效，除非處理 fall-through attrs。**
   `ToolbarButton` 外面包一層 `UTooltip` 之後，全站的「新增」「匯出」按鈕**全部按不動**——
   因為 `@click` 這類沒宣告的 attr 預設落在元件根元素上，而根元素已經變成 `UTooltip` 了。
   修法：`defineOptions({ inheritAttrs: false })` ＋ 在真正的 `UButton` 上 `v-bind="$attrs"`。
   ⚠️ 這個症狀是「什麼都沒反應」，很容易誤判成 tooltip 擋住點擊，實際上事件根本沒綁到按鈕。

## 20. 狀態徽章的顏色語義（2026-07-20 拍板，收斂三份複製）

| 顏色 | 語義 | 什麼時候用 |
|---|---|---|
| `success` | 正常運作中 | 啟用、在職 |
| `warning` | **待處理——有人要去追** | 邀請中（還沒設密碼）、留職停薪 |
| `neutral` | 非啟用，但一切正常 | 停用、離職 |
| `error` | **真正的錯誤或破壞性結果** | 匯入失敗、驗證不通過 |

**關鍵判準：`error`（紅）不是「負面」的意思，是「出事了」的意思。**

事故案例：員工管理原本把「已停用」標成 `error` 紅。但員工離職導致帳號停用是系統
**照設計正確運作**的結果，那一列會長成「離職(灰) + 已停用(紅)」——紅色把視線拉到一件完全正常的事上。
紅色被這樣用久了，使用者會開始無視所有紅色，真的出事時就沒人看見。這與 §15-2
（警告色只留給真正需要行動的事）是同一條原則的兩個面向。

「邀請中」用 `warning` 而非 `info`：它代表「這個人還沒設定密碼、需要有人追」，列表甚至有「重寄邀請」
的動作可按。黃＝待處理，比藍＝純告知更貼近實際語意。

### 這組對照必須有單一來源

事故當時同一組 ACTIVE/INVITED/DISABLED 對照散在三個檔案、**而且顏色不一致**
（使用者管理 `warning`/`neutral`、員工管理 `info`/`error`、稽核翻譯層只有文字沒有顏色）。
平台已收斂到 `apps/portal/src/shared/lib/account-status.ts`，新頁面一律從那裡取。

⚠️ 順帶一提：**稽核詳情的「前後對照」也要吃同一份標籤**，否則同一件事在列表寫「停用」、
在稽核寫「已停用」，看起來像兩件不同的事。

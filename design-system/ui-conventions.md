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
- `overlay/`：`FormModal`/`ConfirmModal`　`layout/`：`LobbyShell`/`AccountShell`/`AppPageLayout`/`PageHeader`
- `nav/`：`UserMenu`/`MySystemsMenu`/`ListPanel`/`ListItemButton`　`card/`：`SystemAppCard`/`UserIdentityCard`　`feedback/`：`EmptyState`/`PasswordStrengthMeter`
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

## 11. 資訊架構：「個人/帳號」與「管理」兩世界（2026-07-12 拍板 B 案；07-16 升級帳號中心殼）

- 側欄（AppShell）＝管理員工作台、只放管理頁；大廳輕頂欄（`LobbyShell.vue`：logo=回我的系統、系統切換器、右上頭像 UserMenu）＝個人世界。**個人／帳號一律收右上頭像選單**，頂欄不擺個人按鈕（「找自己＝點頭像」全站一致）。
- **帳號中心＝單一殼左子導覽（2026-07-16 桌面重構定案）**：個人資料／帳號安全／個人化三分頁共用 `AccountShell.vue`（包在 LobbyShell 內、左欄身分區＋子導覽、右欄 RouterView、桌面 `max-w-7xl`、dark 毛玻璃同 my-apps glass），**推翻 07-12「兩頁分開」案**；**擴充走左子導覽非 tabs**。分頁名「個人化」（頭貼+背景），「外觀設定」保留給未來主題/縮放（`auth-user-preferences`）避免撞名。這條左導覽是帳號中心內部導覽、非管理側欄。手機版 RWD 未做（Steven 另有想法）。
- **SSO 特性**：個人資料多為公司資料唯讀（部門/職稱/角色由管理員/未來 HR 主檔管），本人能改的只有大頭貼＋背景（顯示名稱政策未定案；給的話稽核/官方列表仍用本名＋帳號）。

## 12. 角色體系顯示與代碼慣例（2026-07-15 拍板，決策本體在 [features/auth-role-management/](../features/auth-role-management/prd.md) example-mapping 決策表）

①**系統角色最少化**——只有 `SUPER_ADMIN`（全系統角色不掛 AUTH_ 前綴）是系統角色（不可刪/停用/改權限）；出廠角色（AUTH_ADMIN/AUTH_HR/CRM_*）＝起手範本、可改可刪，自建角色同等地位。②**授權/保護判斷看 permission、別看 role code**（超管 guard bypass 唯一例外）。③appCode=null 的 UI 用詞＝「**全系統**」。④前端超管代碼一律 import `SUPER_ADMIN_ROLE_CODE`（auth-api.ts），別散寫字串。⑤改角色代碼類 DB 遷移走 **seed 前置冪等收斂**，跑完要重新登入（舊 token 帶舊代碼）。

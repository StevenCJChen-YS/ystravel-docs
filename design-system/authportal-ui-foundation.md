# AuthPortal UI Foundation

> # ⛔ 本檔已停止維護（2026-07-22），且**多處內容已作廢**
>
> 有效內容已整併進 **[design.md](./design.md)＝設計系統唯一權威規範書**。
> **不可作為依據。** 已知作廢處（整併時發現的矛盾，詳見 design.md 附錄）：
>
> | 本檔說法 | 現行規則 |
> |---|---|
> | §3.1 `neutral = slate` | light gray／dark zinc 明暗混搭 |
> | §3.3 `bg-elevated` ＝ card、`bg-default` ＝ 整頁背景 | **卡片用 `bg-default`**；`bg-elevated` 是卡內次層帶（light 下與頁底同值會糊掉） |
> | §9 `AuthAdminShell` / `AuthAccountShell` 雙殼 | 已退役，全站單一 `AppShell` |
> | §10 `SidebarUserCard` | 已改名 `UserIdentityCard` |
> | §12 第一波落地範圍、§13 命名建議 | 已完成，屬歷史紀錄 |

更新日期：2026-07-06

> **落腳說明（2026-07-16）**：本檔原生於 `Ystravel-AuthPortal/docs/AUTHPORTAL_UI_FOUNDATION.md`。Phase 1 架構整併後 AuthPortal 併入 `ystravel-platform`（`apps/portal`）並將 GitHub Archive，這份「詳細實作規範」屬跨專案共通知識，故遷入 docs 共通層作為權威版；原 repo 的舊拷貝隨封存唯讀化，不再維護。文內 `Ystravel-AuthPortal` / `src/...` 等舊路徑對應平台的 `apps/portal/src/...`。與 [foundation.md](./foundation.md) 的分工：foundation＝設計系統本體；本檔＝詳細實作規範（shell、shared 元件、drawer、dropdown 等細節）。

## 1. 目的

這份文件定義 `Ystravel-AuthPortal` 第一版要落實的 UI foundation（介面基礎規範）。

目標：

- 讓 AuthPortal 內部頁面先有一致的 layout、color、typography、density 規則。
- 讓 light mode / dark mode 從一開始就一起開發，而不是晚點補。
- 讓未來 `Ystravel-CRM-Frontend` 可以沿用同一套 foundation，再依 CRM 任務密度做進一步延伸。

備註：

- `LoginPage` 是特殊品牌入口頁，可保留較強的品牌視覺，不直接當作 Auth Admin 與 CRM 的畫面模板。
- Auth Admin 與 CRM 仍應共享 semantic token（語意化 token）與主要元件規則。
- 已登入但非管理後台的 Auth 頁面，統一使用 `AuthAccountShell.vue`，不要直接沿用 `AuthAdminShell.vue`。

## 2. 設計方向

AuthPortal 內頁應呈現：

- 中性、安定、管理後台導向。
- 少量 teal accent（青綠主強調色），不要滿版色塊。
- 以 Nuxt UI 元件原生語言為主，必要時再小幅 override。

不採用的方向：

- 大量硬寫 `bg-white`、`text-slate-*`、`border-stone-*`。
- 每頁各自定義尺寸、字級、hover 狀態。
- light mode 與 dark mode 分開晚點才補。

## 3. 色彩基礎

### 3.1 Semantic palette

- `primary = teal`
- `secondary = sky`
- `success = emerald`
- `info = sky`
- `warning = amber`
- `error = rose`
- `neutral = slate`

### 3.2 為什麼選 `slate`

`slate` 是偏冷的中性灰，搭配 teal 主色時對比清楚，適合管理後台。

適合 AuthPortal 的原因：

- 管理介面穩定、資訊層次清楚。
- 與 Nuxt UI 預設中性色系相容，元件 override 較少。
- teal accent 不會與中性底搶戲。

### 3.3 中性底色分層

內頁不應只有「白底」一層，要有至少這四層語意：

- `bg-default`：整頁背景、sidebar、header、main content 容器
- `bg-muted`：toolbar、hover、表格表頭、次層背景、mobile drawer 外層切換軌道
- `bg-elevated`：card、panel、浮起來的表面
- `bg-accented`：選取態、active nav、當前項目

文字與邊框對應：

- `text-highlighted`：標題、重要資訊
- `text-default`：主要內文
- `text-toned`：中層文字、帳號選單標籤
- `text-muted`：輔助說明、次要資訊
- `border-muted`：大多數分隔線與 card border
- `border-toned`：需要在 `bg-muted` 上更明顯的分隔線（例如 dark mode segment 分隔線）
- `border-accented`：選取態或強調狀態

### 3.4 Button color 實務注意

- `color="neutral" + variant="solid"` 會接近深灰／近黑，不適合當一般灰色填滿按鈕。
- 需要灰色填滿感時，優先 `color="neutral" + variant="soft"`。
- 品牌主行為仍用 `color="primary" + variant="solid"`。

## 4. Light / Dark Mode

### 4.1 原則

- Light / dark mode 現在就一起開發。
- AuthPortal shell 直接提供切換按鈕。
- 顏色偏好存到同一個 key，之後 CRM 直接共用。

### 4.2 Storage key

- `ystravel.platform.color-mode`

### 4.3 開發規則

- 新增或重構頁面時，優先使用 semantic class 與 Nuxt UI color / variant。
- 禁止新增只支援 light mode 的硬寫色。
- `LoginPage` 可以保留頁面局部 CSS variable，但其餘 admin / shell / shared UI 一律優先走 semantic token。

### 4.4 `ColorModeToggle` 變體

元件路徑：`src/shared/ui/ColorModeToggle.vue`

- `variant="icon"`（預設）
  - 用於 desktop header。
  - 單顆圓形 ghost icon 按鈕。
  - 深色模式顯示 sun，淺色模式顯示 moon。
- `variant="segment"`
  - 用於 mobile 帳號 dropdown。
  - 左 sun / 右 moon 的圓角矩形 segmented control。
  - 選中側使用 `bg-default shadow-xs`；未選中為 `text-muted`。
  - 中間分隔線：`border-muted`，dark mode 補 `dark:border-toned/70`。
  - 外層與按鈕間保留 `gap-1`，避免分隔線貼住選中按鈕。

## 5. Typography

### 5.1 文字層級

- Page title：mobile `text-xl`，`sm` 以上 `text-2xl`
- Section title：mobile `text-base`，`lg` 以上 `text-lg`
- Body：固定 `text-sm`
- Help / description：固定 `text-sm`
- Meta / code / timestamp：固定 `text-xs`

### 5.2 原則

- 內部系統以穩定可掃讀為優先，不讓一般內文大小跟著斷點頻繁變化。
- 大標題可以隨斷點放大，但 body / table / form 文字應維持穩定。

### 5.3 Shell 品牌標題

- Header 固定顯示「永信生活旅遊事業」。
- 字級 `text-xl font-semibold tracking-wider`。
- LOGO：light `logo_small_original.png`、dark `logo_small_white.png`，寬度 `28px`。

## 6. Breakpoints

- `base`: `< 640px`
- `sm`: `>= 640px`
- `md`: `>= 768px`
- `lg`: `>= 1024px`
- `xl`: `>= 1280px`
- `2xl`: `>= 1536px`

## 7. Density 規則

### 7.1 Login density

- `UFormField size="xl"`
- `UInput size="xl"`
- `UButton size="xl"`

適用範圍：

- 只限 `LoginPage`

### 7.2 Auth Admin default density

- Mobile 與 tablet：`UInput` / `USelect` / `UButton` 優先 `lg`
- Desktop (`lg` 以上)：`UInput` / `USelect` / `UButton` 優先 `md`
- Row action button：固定 `sm`
- Badge：預設 `sm`
- Code / dense badge：`xs`

### 7.3 CRM compact density

先預留規則，不在 Auth 第一波全面套用：

- Desktop 表單、filter、table action 以 `sm` 或 `md` 為主
- 不複製 login 頁 `xl` 尺寸

## 8. Button Hierarchy

- Primary page action：`color="primary"` + `variant="solid"`
- Secondary action：`color="neutral"` + `variant="outline"`
- Toolbar icon / utility action：`color="neutral"` + `variant="ghost"`
- Filled neutral utility（例如 mobile 帳號選單登出）：`color="neutral"` + `variant="soft"`
- Risky but non-destructive：`color="warning"` + `variant="soft"`
- Destructive：`color="error"` + `variant="soft"` 或 `outline`

規則：

- 同一個 view 只保留一個最強的 primary action。
- 不在同一個 toolbar 混很多不同視覺重量的按鈕。
- Shell 登出：desktop 用 `outline` + `size="xl"`；mobile dropdown 用 `soft` + `size="md"` 或 `lg`。

## 9. Shell 規則

### 9.1 `AuthAccountShell.vue`

`AuthAccountShell.vue` 用於已登入的一般使用者頁面，例如：

- `/apps`
- `/account/profile`
- 未來 `/account/security`
- 未來其他 account center（帳號中心）頁面

規則：

- 它是 account center / app chooser shell，不是 admin console shell。
- 可共用品牌標題、theme toggle、avatar dropdown、header 高度與 RWD 規則。
- 不放 Auth admin 專屬導覽項目，不把 permission management 心智模型帶進一般使用者頁面。
- 若有左側導覽，僅限 account 層級項目，例如 apps、profile、security；不可混入 user/role/permission/audit 等管理頁目錄。
- 共用個人名片元件與資料來源，讓 Auth / CRM / EIP 三個系統維持一致身份展示。
- 一般使用者頁面數量即使暫時很少，也仍使用獨立 shell，避免未來 account center 擴充時再從 admin shell 拆出。

### 9.2 `AuthAdminShell.vue`

Auth Admin shell（`src/layouts/AuthAdminShell.vue`）要做到：

- Desktop 固定 sidebar，可收合。
- Mobile 左側漢堡按鈕開啟 drawer，右側 avatar dropdown 放主題切換與登出。
- Header 放品牌標題、theme toggle（desktop）、user info、logout。
- Navigation 要依 permission 顯示，不顯示無權限項目。
- 路由切換時自動關閉 mobile drawer。

### 9.3 Layout tokens

定義於 `src/assets/css/main.css`：

- `--app-sidebar-width: 240px`
- `--app-sidebar-collapsed-width: 4rem`
- `--app-header-height: 70px`

### 9.4 Desktop sidebar

- 背景 `bg-default`，右邊框 `border-muted`。
- 展開寬度 `240px`；收合 `64px`。
- 寬度與 main `padding-left` 使用 `transition` 200ms。
- 頂部個人卡：共用 `SidebarUserCard`。
- 導覽：`UNavigationMenu`，`color="neutral"`。
- Nav item 基礎樣式：`py-2.5`、`gap-3.5`、icon `size-4`、`before:rounded-sm`。
- 收合時 nav item `justify-center`，保留 tooltip。
- 預設 avatar：`src/assets/avatar_default.png`，`ring-1 ring-primary/12`。

### 9.5 Mobile drawer

- 元件：`UDrawer`，`direction="left"`，無 handle。
- 寬度對齊 sidebar：`w-[var(--app-sidebar-width)]`，`max-w-[86vw]`。
- 背景 `bg-default`；僅右下角圓角 `rounded-br-2xl`。
- 需覆寫 Nuxt UI 預設：left drawer 預設帶 `rounded-r-lg`，要加 `rounded-none` 再套 `rounded-br-2xl`。
- 內容結構與 desktop sidebar 對齊：頂部 `SidebarUserCard` + `UNavigationMenu`。
- 不另放登出按鈕；登出改由 avatar dropdown 處理。

### 9.6 Mobile avatar dropdown

- 元件：`UDropdownMenu`。
- Content：`w-48 p-3`，`viewport: 'hidden'`（自訂內容、不顯示空 item 區）。
- 區塊順序：
  1. `切換主題` 標籤 + `ColorModeToggle variant="segment"`
  2. `border-t border-muted` 分隔線
  3. 登出按鈕 `neutral soft`，`rounded-sm`，`block`

## 10. Shared UI 第一批

第一批 shared UI / shared rule：

- `PageHeader`
- `DataToolbar`
- `ColorModeToggle`
- `SidebarUserCard`
- `PermissionCode`

`SidebarUserCard`：

- 用於 desktop sidebar 與 mobile drawer 頂部。
- 背景圖 `profilecard_bg_japan.png` + 左到右黑色漸層薄紗。
- 顯示 avatar、displayName、role / username。
- 這個元件的產品角色是「跨系統共用身份名片」，不只是 AuthAdminShell 的 sidebar 卡片。
- 背景圖建議由 Auth 集中管理，使用固定素材庫 + `backgroundKey` 儲存，不提供使用者自訂上傳背景圖。
- avatar 可由使用者上傳；背景圖僅從預設素材集中選擇。
- 目前內容可先顯示 `avatar + name + role/username`；未來可演進為 `avatar + name + extension + job title`。
- 若 extension / job title 的主資料暫時仍在 EIP 或未來 HRM，可先由 Auth 顯示同步後的展示欄位，不必一開始就由 Auth 成為原始主檔來源。

下一批候選：

- `StatusBadge`
- `EmptyState`
- `UserRolePanel`
- `PermissionMatrix`

## 11. Nuxt UI 實作策略

- 優先使用 Nuxt UI 原生元件與 semantic class。
- 全域 default variants 先定義在 `vite.config.ts`。
- 單頁真的需要特殊效果時，再使用 `ui` prop 或局部 class override。

### 11.1 已知元件 override

- `UButton` icon-only 固定 `size-9` 時，需加 `:ui="{ base: 'justify-center p-0' }"` 才能置中。
- `UDrawer direction="left"` 預設右側圓角，需明確覆寫 content slot class。
- `UDropdownMenu` 若完全自訂內容，建議不傳 `items`，並將 `viewport` 設為 `hidden`。

### 11.2 全域 button scale

`vite.config.ts` 已客製 `UButton` size scale（`font-normal`、padding、icon 尺寸）。後續 Input / Select 應沿用同一套 scale 規則，避免表單列高度不一致。

## 12. 第一波落地範圍

這一波先落實：

- 全域 color mode 基礎
- App shell dark mode toggle
- `AuthAdminShell` desktop sidebar / mobile drawer / avatar dropdown
- `UserIdentityCard`（現況實作名稱可暫時維持 `SidebarUserCard`）共用個人卡
- `ColorModeToggle` icon + segment 兩種變體
- `PageHeader` 響應式標題規則
- `AdminRolesPage` / `AdminPermissionsPage` / `AdminAuditLogsPage` 的 page header、toolbar 與 table chrome
- Layout tokens（sidebar / header 尺寸）
- 文件化規範寫回 repo

第二波再落實：

- `AdminUsersPage` 深度表單化（create / reset password / role assign）
- `AppsPage`
- `AuthAccountShell.vue`
- 再把同樣 foundation 複製到 CRM shell

## 13. Account Center Naming

Auth 已登入但非 admin 的 route / page 命名，統一走 account center 語意。

推薦 route：

- `/apps`
- `/account/profile`
- `/account/security`
- `/account/notifications`

推薦 page component：

- `AccountAppsPage.vue`
- `AccountProfilePage.vue`
- `AccountSecurityPage.vue`
- `AccountNotificationsPage.vue`

說明：

- `AppsPage.vue` 在過渡期可以保留，但目標命名建議改為 `AccountAppsPage.vue`，因為它本質上屬於 account center，而不是未登入 auth flow。
- route 可保持 `/apps` 短網址，不必強制改成 `/account/apps`。
- 一般使用者的頁面應放在 `src/pages/account/`，不要繼續和 `LoginPage` 一起放在 `src/pages/auth/`。

共用身份名片元件命名：

- 目標命名：`UserIdentityCard.vue`
- 過渡命名：`SidebarUserCard.vue`

命名理由：

- 這個元件未來會出現在 Auth、CRM、EIP 三個系統，不只存在 sidebar。
- 它代表的是使用者跨系統身份展示，不只是版位用途。
- 若目前 repo 已大量使用 `SidebarUserCard`，可先保留現名，等 account shell 與 CRM shell 穩定後再統一實際 rename。

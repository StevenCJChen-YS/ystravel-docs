# CRM + AuthService 系統分析 SA

| 項目 | 內容 |
|---|---|
| 文件目的 | 規劃 CRM 新系統與 EIP 權限/角色改版方向 |
| 目標系統 | AuthService、CRM、既有 EIP |
| 建議優先順序 | 先建立 AuthService 邊界，再開發 CRM 核心功能 |
| 文件狀態 | 初版討論稿，2026-07-02 補實作狀態 |
| 進度交接 | `docs/operations/YSTRAVEL_AUTH_CRM_PROGRESS.md.docx` |

---

## 0. 命名策略

新系統與新 repo 統一使用 `Ystravel` 前綴，避免延續舊稱 `GInternational`。

建議命名：

- `Ystravel-AuthService`：登入、使用者、角色、權限與 token API。
- `Ystravel-AuthPortal`：登入入口、系統入口、Auth Admin 後台。
- `Ystravel-CRM-Frontend`：CRM 前端。
- `Ystravel-CRM-Backend`：CRM 後端。

既有 EIP 的 `GInternational-Frontend` 與 `GInternational-Backend` 暫時不改名，避免牽動既有部署路徑、環境變數、Git remote、CI/CD、API 設定與文件引用。等 AuthService 與 CRM 穩定後，再視需要規劃 EIP repo 改名。

---

## 1. 背景

目前 EIP 已經有使用者、角色、權限、個人額外權限與拒絕權限。CRM 未來也會給 EIP 使用者使用，且可能有數十位以上使用者。若 CRM 另做一套登入與權限，未來會產生重複帳號、重複權限、登入狀態不同步、離職停權不同步等問題。

因此建議將「登入、使用者、角色、權限」抽成 AuthService，由 EIP 與 CRM 共用。

---

## 2. 現況問題

### 2.1 使用者流程

目前 EIP 使用者新增方式大致有兩種：

1. 管理員在用戶管理手動新增。
2. HR 在員工管理新增員工後，再透過一鍵建立 EIP 使用者。

建立後還需要指派角色與頁面權限。

### 2.2 權限維護流程過於分散

目前一個新功能可能需要同時維護多個地方：

1. 後端 route 寫入 `checkPermission('XXX')`。
2. 前端 route 或 menu 寫入 `permission: 'XXX'`。
3. `PermissionSelector.vue` 內硬寫權限模組與權限 code。
4. 資料庫手動新增 permission。
5. 角色才能在 PermissionSelector 選到該權限。

這代表「權限定義」沒有單一來源。功能越多，遺漏風險越高。

### 2.3 CRM 導入後會放大問題

CRM 需要：

- 單一登入。
- 共用公司使用者。
- 共用停權/離職狀態。
- CRM 自己的角色與權限。
- 客戶/訂單資料的細粒度存取規則。

若 EIP 與 CRM 各自管理帳號與權限，長期維護成本會明顯增加。

---

## 3. 目標

### 3.1 系統目標

1. 建立 AuthService 作為公司內部登入與授權中心。
2. EIP 與 CRM 使用同一套身份與權限。
3. 權限定義由後端版本控管，自動同步到資料庫。
4. 角色與使用者指派由管理 UI 維護。
5. 保留個人額外權限與拒絕權限，但定位為例外管理。
6. CRM 使用 MySQL 建立可長期擴充的客戶與訂單資料模型。

### 3.2 非目標

初期不建議一次完成：

- 完整微服務拆分。
- 一次重寫 EIP。
- 導入複雜外部授權引擎。
- 讓 AI 自動決定正式客戶合併。

---

## 4. 建議系統邊界

### 4.1 AuthService 負責

- 登入。
- Google login。
- access token / refresh token。
- 使用者主檔。
- 系統清單，例如 EIP、CRM。
- 權限 catalog。
- 角色。
- 角色權限。
- 使用者角色。
- 使用者個人權限例外。
- 權限查詢 API。

### 4.2 Auth Portal / Auth Admin 負責

AuthService 是後端 API，仍需要一個前端讓使用者登入與讓管理者設定權限。建議第一版將登入入口與管理後台放在同一個前端專案中。

Auth Portal 負責：

- `/login`：所有使用者登入頁。
- `/apps`：登入後顯示可進入系統，例如 EIP、CRM。
- `/auth/callback`：處理登入導回流程。

Auth Admin 負責：

- `/admin/users`：管理使用者、啟用/停用、指派 EIP/CRM 角色。
- `/admin/roles`：管理角色與角色權限。
- `/admin/permissions`：檢視 permission catalog 與同步狀態。
- `/admin/audit-logs`：查看登入、角色、權限異動紀錄。

第一版可將 Auth Portal 與 Auth Admin 放在同一個 `Ystravel-AuthPortal` 前端專案，之後規模變大再拆開。

### 4.3 EIP 負責

- 員工資料。
- 公司/部門資料。
- 公告、考核、行銷、硬體、通知等既有 EIP 業務功能。
- 逐步改為使用 AuthService 權限。

### 4.4 CRM 負責

- 客戶資料。
- 客戶身份辨識。
- 聯絡方式。
- 地址。
- ERP 匯入原始資料。
- 訂單/消費紀錄。
- 客戶合併候選。
- 行銷與業務分析。

---

## 5. 權限管理原則

### 5.1 Permission Catalog

權限本身不建議由前端手動新增。權限應由工程端定義，透過 seed/sync 自動 upsert 到資料庫。

範例：

```ts
{
  code: 'CRM.CUSTOMER.READ',
  app: 'CRM',
  module: 'customer',
  resource: 'customer',
  action: 'read',
  name: '查看客戶',
  description: '可查看客戶列表與客戶基本資料'
}
```

### 5.2 Role

角色由管理者透過 UI 建立與調整。

範例：

- `CRM_SALES`
- `CRM_MANAGER`
- `CRM_MARKETING`
- `CRM_ADMIN`
- `EIP_HR`
- `EIP_IT`

### 5.3 User Role

使用者可有多個角色。角色可以支援到期日，以便處理臨時權限。

### 5.4 User Permission Override

個人額外權限與拒絕權限可保留，但要加強治理：

- 必填原因。
- 可設定到期日。
- 記錄授權者。
- 可稽核。
- 可列表檢查。

定位：例外處理，不是日常主要管理方式。

---

## 6. Auth 管理角色分層

AuthService 會管理 EIP 與 CRM 的帳號、角色、權限，因此 Auth 管理權限本身也要分層。你與主管可以同時是 Auth 管理者、EIP 管理者、CRM 管理者，但這些身份要用角色分開表示。

建議角色：

| 角色 | 說明 |
|---|---|
| `AUTH_SUPER_ADMIN` | 可管理所有系統、所有使用者、所有角色與最高權限 |
| `AUTH_ADMIN` | 可管理使用者與一般角色，但不可管理最高管理者 |
| `EIP_ADMIN` | 可管理 EIP 業務資料與 EIP 角色指派 |
| `CRM_ADMIN` | 可管理 CRM 業務資料與 CRM 角色指派 |
| `CRM_MANAGER` | CRM 主管權限 |
| `CRM_SALES` | CRM 業務權限 |

重要原則：

- `CRM_ADMIN` 不等於 `AUTH_ADMIN`。
- `EIP_ADMIN` 不等於 `AUTH_ADMIN`。
- Auth Admin 管「人可以進哪個系統、在系統裡是什麼角色」。
- EIP/CRM 各自管自己的業務資料。

---

## 7. 正式環境部署路徑

目前不規劃新增網域，正式環境可採用同一網域加子路徑部署。

建議路徑：

| 系統 | 正式路徑 |
|---|---|
| EIP | `https://eip.ystravel.com.tw/` |
| CRM Frontend | `https://eip.ystravel.com.tw/CRM/` |
| Auth Portal/Admin | `https://eip.ystravel.com.tw/Auth/` |
| AuthService API | `https://eip.ystravel.com.tw/api/auth/` |
| CRM API | `https://eip.ystravel.com.tw/api/crm/` |

登入流程：

```text
使用者開啟 /CRM/
  -> 未登入
  -> 導到 /Auth/login?app=CRM&redirectUri=/CRM/auth/callback
  -> Auth Portal 登入成功
  -> 回到 /CRM/auth/callback
  -> CRM 進入系統
```

注意事項：

- `redirectUri` 必須在 AuthService applications 白名單中。
- CRM 前端 build base 應設定為 `/CRM/`。
- Auth Portal 前端 build base 應設定為 `/Auth/`。
- API 建議統一放在 `/api/auth/`、`/api/crm/`。
- 同網域下 localStorage 是共用的，token key 與 app 狀態 key 必須明確命名。

---

## 8. CRM 資料來源分析

ERP Excel 匯出的資料較像「訂單明細 + 旅客/客戶明細」混合表，不是乾淨客戶主檔。

目前觀察到的特性：

- 約 108,535 筆資料列。
- 約 16,286 個訂單編號。
- 多數訂單會有多位旅客。
- 欄位包含客戶身份、證件、電話、Email、地址、團號、日期、國家、金額、通路、業務、參團狀態。
- 有重複欄位與資料品質問題。

因此不建議直接匯入正式 customers 表。

---

## 9. CRM 匯入原則

建議流程：

1. 匯入原始 Excel 到 raw import。
2. 保存每一列原始資料。
3. 清洗日期、金額、狀態、通路、國家。
4. 排除不應成為客戶主檔的資料，例如領隊/導遊。
5. 建立客戶候選身份。
6. 用規則與分數判斷可能同一人。
7. 高信心自動連結，低信心進人工審核。
8. 寫入正式 customers、orders、order_participants。

---

## 10. AI 使用定位

AI 可用，但建議放在第二階段。

適合 AI 的工作：

- 地址解析。
- 國家/路線分類。
- 髒資料辨識。
- 客戶偏好摘要。
- 疑似重複客戶合併建議。
- 行銷分群建議。

不建議一開始讓 AI 自動做：

- 客戶正式合併。
- 身分證/護照等敏感資料判斷唯一真相。
- 高風險權限決策。

---

## 11. 建議開發階段

### Phase 0：確認設計

- 確認 AuthService 是否使用 MySQL。
- 確認 CRM 使用 MySQL。
- 確認 EIP 過渡策略。
- 整理既有 EIP permission code。

### Phase 1：AuthService MVP

- 建 NestJS AuthService。
- 建 Prisma schema。
- 實作登入、token、使用者、角色、權限 catalog。
- 實作 permission sync。
- 建 `/me`、`/me/permissions`。
- 建 applications 與 redirect URI 白名單。

### Phase 2：Auth Portal / Auth Admin MVP

- 建 Vue 3 Auth Portal。
- 建 `/login` 登入頁。
- 建 `/apps` 系統入口頁。
- 建 `/admin/users` 管理使用者與角色指派。
- 建 `/admin/roles` 管理角色與權限勾選。
- 建 `/admin/permissions` 檢視 catalog 與同步狀態。

### Phase 3：CRM 基礎

- 建 CRM frontend。
- 建 CRM backend。
- CRM backend 驗證 AuthService token。
- CRM frontend 實作 `/auth/callback`。
- CRM frontend 使用 AuthService 權限。

### Phase 4：CRM 匯入

- 建 raw import tables。
- 匯入 Excel。
- 建清洗流程。
- 建客戶候選與訂單資料。

### Phase 5：EIP 過渡

- EIP 使用 AuthService token 或 permission API。
- EIP 權限管理 UI 改打 AuthService。
- EIP 後端 `checkPermission` 逐步改為 AuthService 驗證。

### Phase 6：AI 與分析

- AI 輔助資料清洗。
- 客戶分群。
- 行銷報表。
- 業務客戶管理。

---

## 12. 現在要做的事

建議下一步：

1. 先完成 CRM Backend 本機 DB migration，確認 `ystravel_crm` tables 可用。
2. CRM Backend 實作 customers / orders / imports 的第一版 API。
3. CRM Frontend 將 dashboard、customers、orders、imports、reports 的假資料替換成真 API。
4. CRM UI 做第二輪調整，優先改善表格、篩選、詳情、重複操作流程。
5. 補 AuthService global logout / single logout 設計與實作。
6. 規劃 EIP user migration dry-run。

---

## 13. 2026-07-02 實作狀態補充

已落地：

- `Ystravel-AuthService` 已完成 Auth MVP：
  - 帳密登入。
  - `/auth/me`。
  - refresh token 輪替。
  - logout 撤銷 refresh token。
  - 使用者、角色、權限、audit log 管理 API 初版。
- `Ystravel-AuthPortal` 已改接真 AuthService：
  - 登入。
  - session 還原。
  - 自動 refresh。
  - 登出。
  - 使用者/角色/權限/audit log 管理頁面改讀真資料。
- `Ystravel-CRM-Frontend` 已移除本機假登入，改走 AuthService SSO：
  - 未登入導到 `/Auth/login?app=CRM&redirectUri=/CRM/auth/callback`。
  - callback 接 token。
  - `/auth/me` 還原使用者。
  - sidebar 依 permission 顯示。

尚未完成：

- CRM 客戶/訂單/匯入頁面仍是骨架或假資料。
- CRM Backend 尚未開始接 customers / orders / imports 真 API。
- AuthPortal 登出後，CRM 前端 local token 不會立即消失；目前屬於 local logout 行為。

建議決策：

- 公司內部系統長期建議做 global logout / single logout。
- 客戶 Excel 與 ERP 訂單資料含大量個資，第一階段只做本機 raw import、規則清洗、去識別化統計；不要直接把原始個資上傳到外部 AI。

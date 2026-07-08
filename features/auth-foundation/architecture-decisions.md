# Auth Foundation — 架構決策文件（Phase 0）

| 項目 | 內容 |
|---|---|
| 文件性質 | 跨功能地基（Architecture Decision Record），不是單一 feature 的 SA/SD |
| 適用範圍 | Ystravel-AuthService、Ystravel-AuthPortal、Ystravel-CRM-Backend/Frontend，未來 EIP 整合 |
| 對應流程 | `docs/development-process/00-overview.md` Phase 0；資安欄位依 `docs/process/SECURITY_AND_ISO27001_BASELINE.md` §6.3 |
| 狀態 | **Phase 0 地基決策全部定案（2026-07-08）** |
| 建立日期 | 2026-07-08 |
| 最後更新 | 2026-07-08（**改為 native auth v2**：AuthService 主導登入與權限，Logto 實驗保留為歷史紀錄——見 §0.6） |
| 決策角色 | Steven = 拍板；AI = 草擬技術方案 |

---

## 0. 這份文件要解決什麼

SSO 的「身分模型、認證機制、權限結構、系統之間怎麼互相信任」是**所有系統共用的地基**。它不能一個 feature 一個 feature 長出來，必須先一次定清楚，否則之後每個功能都會各自對權限與 token 做假設，最後對不起來。

**這份文件不寫程式，只做決策。** 你只要看得懂第 1 節（決策摘要）和第 12 節（待決策），能點頭或提出不同意見即可。

---

## 0.5 重大修正：混合架構（2026-07-08）

> **歷史決策，已被 §0.6 覆蓋。** 這段保留用來說明為何曾經導入 Logto、Logto 解決了哪些問題，以及 native v2 必須補哪些安全債。

**背景**：本文件原本假設「從零自架 Logto」。但實際檢視 `Ystravel-AuthService` 程式碼後發現——**Steven 已經自建了一套相當完整、還在運作的登入/權限系統**（login/JWT/RBAC/多系統權限/audit/refresh 輪替都有），且大致符合本文件多數決策。

**因此改採 Option C（混合）**：登入交給 Logto，權限留在現有 AuthService。

### 保留 vs 汰換

| 現有 AuthService 的東西 | 決定 | 說明 |
|---|---|---|
| `Application` / `Role` / `Permission` / `UserRole` / `RolePermission` / `UserPermissionOverride` | ✅ **保留** | 這是核心價值，跨系統集中管權限的資料模型（D6）已內建 |
| `AuditLog` 表 + 稽核機制 | ✅ **保留** | 已符合 §8 audit 設計 |
| `PermissionsGuard` / `permissions.decorator` / 各 admin controller（users/roles/permissions/audit-logs） | ✅ **保留** | 授權與管理層 |
| `AuthUser` 表 | 🔧 **保留但改造**：拿掉 `passwordHash`，新增 `logtoSub` 連到 Logto 身分 | 本地仍存使用者的業務屬性（employeeNo/department/title/roles） |
| `auth.controller`（login/refresh/logout）+ `RefreshToken` 表 + 自簽 JWT | ⛔ **汰換給 Logto** | 這正是最不該自己維護的部分 |

### 為什麼這樣分
- 自簽 JWT 用**共用密鑰（HS256）**：多系統 SSO 下任何系統漏密鑰就能偽造全站 token → 改用 Logto 的 **OIDC 非對稱金鑰（JWKS）**，各系統只需公鑰驗證。
- 最高風險、最難獨力維護的 **Google 登入 / 2FA / 忘記密碼 / 登入鎖定 / 擋外洩密碼** → 由 Logto 內建提供，不自己寫。
- 現有 `findActiveRefreshToken` 全表掃描的效能坑，隨 refresh 交給 Logto 一併解決。

---

## 0.6 目前決策：native auth v2（2026-07-08）

Steven 實際看完 Logto 流程與家用機實裝結果後，決定回到原本方向：**AuthService 自己主導登入、token、refresh、權限與 audit；AuthPortal 使用原本的帳密登入頁體驗。**

這不是單純退回舊版，而是 **native auth v2**：

| 主題 | 決定 | 原因 |
|---|---|---|
| 登入入口 | AuthPortal 原本 Email + 密碼表單為主，保留 Google 登入按鈕位置 | UI/UX 已符合 Steven 想要的入口體驗，不需要跳到外部 IdP 頁 |
| 認證核心 | AuthService 管 `/auth/login`、`/auth/refresh`、`/auth/logout`、`/auth/me` | 與既有 `Application` / `Role` / `Permission` / `AuditLog` 模型一致 |
| 權限 | 繼續由 AuthService 集中定義與指派，各系統執行檢查 | 這是現有系統最有價值的部分，且貼合「SSO 一處管全部權限」需求 |
| 外部身分連結 | `logtoSub` 暫保留為 nullable 欄位，未來可改名/泛化成 external identity link | 避免破壞性 migration，也保留 Google/sub linking 的路 |

native v2 的必修補強：

- **Token 簽章**：已把 access token 從 HS256 共用密鑰升級為 RS256/JWKS（非對稱金鑰），避免多系統共用 secret 的風險。
- **Refresh token 查找**：第一階段先恢復可用；後續必須改成可索引的 token selector / jti / hash lookup，不再全表掃描後逐筆 bcrypt compare。
- **高風險登入功能**：Google 登入、2FA、忘記密碼、登入鎖定、外洩密碼檢查都要列入後續 feature，不當作已完成。
- **不要偏離原方向**：若 native auth 後續設計開始變成「每個系統各管一套登入/權限」，必須停下來調整；AuthService 仍是集中入口。

---

## 1. 決策摘要（一眼看完所有拍板）

| # | 決策 | 選擇 | 一句話理由 |
|---|---|---|---|
| D1 | 使用者範圍 | **只有內部員工** | 先做最小範圍，最快跑起來；外部客戶身分之後再擴 |
| D2 | 架構 | **native auth v2：AuthService 管登入與權限** | 見 §0.6；回到 Steven 原本方向，但補齊安全債 |
| D3 | IdP 選型 | **第一階段不導入外部 IdP**；Logto 作為已驗證過的備案 | 保留實驗結果，不讓登入體驗被外部頁面綁住 |
| D4 | SSO 協定 | **自有 JWT + RS256/JWKS** | AuthService private key 簽發；下游系統用 public JWKS 驗簽 |
| D5 | 登入方式 | **帳密登入先恢復；Google 公司帳號登入列下一刀** | 先回到原本 UI 與 native API，Google 另走安全設計 |
| D6 | 權限管理 | **一處集中管所有系統的角色權限（集中在現有 AuthService 的 Application/Role/Permission，執行檢查在各系統）** | 沿用已建好的資料模型；見 §4 |
| D7 | Token 驗證 | **RS256/JWKS** | 修掉 HS256 共用密鑰風險；`JwtAuthGuard` 只接受 RS256 |
| D8 | 登出 | **先恢復 AuthService logout；Single Logout 另列後續設計** | native SLO 要有各系統 session/token 撤銷策略，不能用口號帶過 |
| D9 | 帳號遷移 | **身分模型現在就要能承受帳號連結/合併**（為未來 EIP 使用者併入預留） | 見 §2、§6 |
| D10 | 異動紀錄 | **做成跨系統共用的 audit 機制**（interceptor/decorator + append-only 表） | 見 §8 |

> D2 已於 2026-07-08 從 Logto 混合架構改回 native auth v2。第 12 節列出拍板結果與後續安全事項。

---

## 2. 身分模型（AuthN 對象）

- **對象**：只有內部員工（D1）。暫不含外部客戶／廠商。
- **帳號識別（D5 的比對依據）**：
  - IT 在後台**預先建立**使用者，填公司 email、啟用。
  - 第一階段：員工用公司 email/username + 密碼登入，AuthService 檢查帳號是否存在且啟用。
  - 後續 Google 登入：仍採「預先佈建 + 社群登入 + 白名單」，Google `sub` 只作外部身分連結，不取代員工編號。
  - **穩定內部鍵**：以 `AuthUser.id` + `employeeNo` 做系統內對應；外部 provider 的 `sub` 另存連結欄位。
- **租戶**：**單租戶**（Ystravel 一家）。
- **身分唯一來源（Source of Truth）**：AuthService。各業務系統不自己存密碼、不自己建平行帳號。

---

## 3. 認證機制（AuthN）

### 3.1 流程
```
員工 ──▶ AuthPortal（登入畫面）
   │  email/username + password
   ▼
AuthService 驗帳密 / 帳號狀態 / audit
   │
   ▼
AuthService 發：access token(JWT, 短效) + refresh token(輪替)
   │
前端帶 access token 呼叫 ──▶ 後端（JwtAuthGuard 驗 token）
                                ▼ 通過才進 PermissionsGuard
```

### 3.2 IdP 選型：暫不導入

Logto 已經實測可行，但目前 UX 與維運感受不符合 Steven 想要的方向，因此第一階段不導入外部 IdP。Logto / Keycloak 保留為未來備案，不是當前主線。

### 3.3 Token 策略（D7）
- **第一階段 access token**：恢復現有 JWT，短效（15 分鐘）。
- **已完成**：access token 使用 RS256/JWKS，讓各系統只持有公鑰，不共享簽章 secret。
- **refresh token**：7 天、輪替（rotation）、一次性、可撤銷。後續需改成可索引 lookup，避免全表 bcrypt 掃描。

### 3.4 登入方式（D5）
- **第一階段**：公司 email/username + 密碼。
- **下一階段 Google 登入**：由 AuthService 對接 Google OAuth；只有 IT 預先建好的 email 能登入，外人有 Google 帳號也進不來；生產環境限制公司網域。

### 3.5 登出：Single Logout（D8）
- 第一階段：恢復 AuthService `/auth/logout`，撤銷 refresh token。
- 後續 SLO（Single Logout，一次登出全部系統）：需要設計各系統如何感知撤銷、access token 到期前如何處理、是否需要 session registry 或 token revocation cache。

---

## 4. 授權模型（AuthZ）—— 一處管所有系統的角色權限（D6）

### 4.1 你要的效果
> 在 AuthService 後台，打開「員工 A」，可以在**同一個地方**設定他跨所有系統的權限：
> - CRM：匯入頁使用權、查詢同業客戶資料權
> - EIP：直客詢問統計表使用權、講座活動統計表使用權

### 4.2 怎麼達成（native auth v2）
- **白話**：「你在後台勾了哪些權限」由**現有 AuthService 的管理後台**一處統一管；「員工實際點下去時各系統讓不讓他過」由各系統照權限放行。體驗上就是**一處管全部**。
- **技術**：權限的定義與指派沿用**現有 AuthService 的 `Application`/`Role`/`Permission`/`UserRole` 資料模型**（每個系統一筆 Application、權限綁到各 app）。使用者登入時由 AuthService 發 token；各系統後端驗 token 後，由 `PermissionsGuard` 依 AuthService 的權限資料放行。
- **好處**：登入、使用者、權限、audit 都集中在你已建好、最貼合需求的地方；新增一個 CRM 權限只在 AuthService 後台一個地方做。

### 4.3 權限命名
- 沿用既有慣例 `<系統>.<模組>.<動作>`，例如 `CRM.IMPORT.USE`、`CRM.CUSTOMER.READ`、`EIP.STATS_DIRECT.VIEW`。

---

## 5. 系統邊界與資料流

| 系統 | 角色 | 存什麼敏感資料 |
|---|---|---|
| AuthPortal | 登入 UI、保存登入狀態、導轉 | access token / refresh token（前端端儲存策略需持續收斂） |
| AuthService | 身分、密碼雜湊、token、角色權限、audit | 員工帳密雜湊、email、角色權限、refresh token hash |
| CRM-Backend | 業務邏輯、CRM 授權執行 | 客戶 PII（Restricted） |
| EIP（未來） | 各業務模組 | 依模組而定 |

- 員工密碼雜湊只在 AuthService；各業務系統不另存密碼。客戶 PII 只在 CRM，不放進 AuthService。

---

## 6. 帳號遷移／合併策略（D9，為未來 EIP 使用者預留）

- 現在 CRM 可自建使用者，未來 EIP 使用者要併過來——**現在就要把身分模型設計成能承受合併**：
  - 不用 email 當唯一鍵（email 會變）；用 `AuthUser.id` + 員工編號當穩定鍵。
  - 預留 **identity linking**：同一個人在不同來源（CRM 自建、EIP 舊帳號）的帳號，未來能綁成同一個身分。
  - 遷移時保留對照表（舊系統 user id ↔ `AuthUser.id` / external provider subject），可追溯、可回滾。

---

## 7. 安全與稽核（SD 模板 §6 / baseline §6.3 必填）

| 欄位 | 內容 |
|---|---|
| 認證與授權方式 | AuthService native login + RS256/JWKS；權限集中定義於 AuthService、各系統 `PermissionsGuard` 執行 |
| 欄位加密/雜湊/遮罩 | 密碼 hash 存 AuthService；secrets 走環境變數不寫死；客戶身分證件值採 `hash + masked`（baseline §7.2） |
| Audit log | 見 §8（跨系統共用機制） |
| 備份/還原 | AuthService DB 含帳號、密碼雜湊、refresh token hash、權限與 audit，備份亦屬敏感資料 |
| 錯誤處理與告警 | 登入失敗鎖定、token 驗簽失敗一律 401、refresh 重放偵測即撤銷並告警（後續補強） |
| 權限控管/角色矩陣 | 最小權限原則；角色矩陣在各 feature 的 SD 具體列出 |
| 測試資料策略 | 測試/開發環境不得用真實員工或客戶資料，用假資料 seed（baseline §4.3） |

---

## 8. 異動紀錄 / Audit（D10，跨系統共用地基）

業界的「異動紀錄」分兩種，兩種都要：

| 類型 | 記什麼 | 用途 |
|---|---|---|
| **操作稽核（audit trail）** | 誰(who)、何時(when)、動作、對象、來源 IP、成功/失敗 | 安全稽核、ISO 27001 |
| **欄位變更歷史（change history）** | 某筆資料某欄位「舊值→新值」、誰改的、何時 | 追查資料怎麼被改 |

**建議設計：**
- 一張 **append-only（只增不改不刪）`audit_logs` 表**：`actor / action / entity_type / entity_id / before(json) / after(json) / ip / created_at`。
- 敏感值（身分證、護照）在 before/after **遮罩，不存明文**。
- 高風險動作一定記（對齊 baseline §7.3）：**登入/登出、權限異動、匯入、匯出、客戶合併、人工覆核**。
- 做成**共用機制**：NestJS interceptor/decorator，每個功能標記一下就自動寫紀錄，不用每支各寫。
- ❌ MVP 不上 event sourcing（太重）。

---

## 9. 部署與環境
- **本地開發**：沿用 `Ystravel-LocalDocker` 的 MySQL/MariaDB，不需要 Logto/Postgres。
- **環境分離**：local / staging / production 的 DB 與 secrets 完全分離（baseline §7.5）。
- **secrets**：RS256 private key、DB 連線字串全走環境變數，不進 repo。

---

## 10. 落地順序（Phase 1 → 2 預告）
1. **Phase 1a**：native auth v2 第一刀：恢復 AuthService 登入/refresh/logout/me + AuthPortal 原登入頁，補 DB restore migration。
2. **Phase 1b**：✅ 已完成 token 安全：HS256 → RS256/JWKS。後續另補 key rotation 多 key 支援與下游系統驗簽整合文件。
3. **Phase 1c**：修 refresh token：可索引 lookup、reuse detection、裝置/工作階段列表。
4. **Phase 1d**：Google 登入（AuthService 對接 Google OAuth），採預先佈建 + 公司網域 + external subject linking。
5. **Phase 1e**：2FA / 忘記密碼 / 登入鎖定 / 外洩密碼檢查。
6. **Phase 2**：CRM-Frontend 走 AuthPortal 登入打 CRM API；把既有 `crm-import-cleaning` 疊到權限控管之下。

> 每個 Phase 各走一次 feature pipeline（PRD 可精簡，但因涉及 SSO，SA/SD 需補資安欄位）。

---

## 11. 測試策略（概要）
- AuthService 登入、Guard、權限查詢、refresh/logout 屬高風險，優先寫 Integration + BDD（`jest-cucumber`）。
- 必測：「正確帳密登入成功」「錯誤帳密/停用帳號 → 401」「無 token 打受保護 API → 401」「有 token 無權限 → 403」「權限異動要寫 audit log」「refresh token 撤銷後不可再用」。

---

## 12. 決策事項（2026-07-08 拍板結果）

| # | 問題 | 結果 |
|---|---|---|
| Q1 | IdP 最終選型 | ✅ **暫不導入外部 IdP，改回 AuthService native auth v2**；Logto 保留為備案與學習紀錄 |
| Q2 | access / refresh token 效期 | ✅ access **15min**、refresh **7 天**（7 天為「閒置多久要重登」，有在用會無感延續） |
| Q3 | 2FA 強制範圍 | ✅ **管理員/高權限強制、一般員工先自願**；native v2 需另做 TOTP，不再假設 Google/Logto 代管 |
| Q4 | 密碼政策 | ✅ 採最小現代政策（12 碼、擋弱密碼、不強制定期換、失敗鎖定），已寫成 `docs/process/PASSWORD_POLICY.md` |
| Q5 | 登入方式 | ✅ **帳密先恢復；Google 並存列下一刀**（帳密當備援，Google 中斷時仍可登入） |

---

*目前 Phase 1a 對應 `docs/features/auth-native-login-v2/`。*
*UI/UX 設計規範見 `docs/design-system/foundation.md` 與 `docs/development-process/10-uiux-guide.md`。*

# Auth Foundation — 架構決策文件（Phase 0）

| 項目 | 內容 |
|---|---|
| 文件性質 | 跨功能地基（Architecture Decision Record），不是單一 feature 的 SA/SD |
| 適用範圍 | Ystravel-AuthService、Ystravel-AuthPortal、Ystravel-CRM-Backend/Frontend，未來 EIP 整合 |
| 對應流程 | `docs/development-process/00-overview.md` Phase 0；資安欄位依 `docs/process/SECURITY_AND_ISO27001_BASELINE.md` §6.3 |
| 狀態 | **Phase 0 地基決策全部定案（2026-07-08）** |
| 建立日期 | 2026-07-08 |
| 最後更新 | 2026-07-08（**改為混合架構 C**：Logto 只管登入，保留現有 AuthService 管權限——見 §0.5） |
| 決策角色 | Steven = 拍板；AI = 草擬技術方案 |

---

## 0. 這份文件要解決什麼

SSO 的「身分模型、認證機制、權限結構、系統之間怎麼互相信任」是**所有系統共用的地基**。它不能一個 feature 一個 feature 長出來，必須先一次定清楚，否則之後每個功能都會各自對權限與 token 做假設，最後對不起來。

**這份文件不寫程式，只做決策。** 你只要看得懂第 1 節（決策摘要）和第 12 節（待決策），能點頭或提出不同意見即可。

---

## 0.5 重大修正：混合架構（2026-07-08）

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

## 1. 決策摘要（一眼看完所有拍板）

| # | 決策 | 選擇 | 一句話理由 |
|---|---|---|---|
| D1 | 使用者範圍 | **只有內部員工** | 先做最小範圍，最快跑起來；外部客戶身分之後再擴 |
| D2 | 架構 | **混合（C）：Logto 管登入，現有 AuthService 管權限** | 見 §0.5；最高風險的登入核心交給成熟專案，保留已建好的 RBAC/audit |
| D3 | IdP 選型 | **Logto（建議）**，Keycloak 為備案 | Logto 對 Vue3+NestJS 的 DX 好、後台現代、Docker 自架簡單；見 §3.2 |
| D4 | SSO 協定 | **OIDC（OpenID Connect）** | 業界標準，CRM／未來 EIP 都用同一套接法 |
| D5 | 登入方式 | **帳密登入 ＋ Google 公司帳號登入（社群登入）** | 由 Logto 提供；見 §3.4 |
| D6 | 權限管理 | **一處集中管所有系統的角色權限（集中在現有 AuthService 的 Application/Role/Permission，執行檢查在各系統）** | 沿用已建好的資料模型；見 §4 |
| D7 | Token 驗證 | **各系統用 Logto 的 JWKS（非對稱金鑰）驗 OIDC token**，配短效 access + refresh | 取代原本的 HS256 共用密鑰 |
| D8 | 登出 | **Single Logout（一次登出全部系統）** | 你的需求；OIDC back-channel logout，見 §3.5 |
| D9 | 帳號遷移 | **身分模型現在就要能承受帳號連結/合併**（為未來 EIP 使用者併入預留） | 見 §2、§6 |
| D10 | 異動紀錄 | **做成跨系統共用的 audit 機制**（interceptor/decorator + append-only 表） | 見 §8 |

> D1/D2/D5/D6 你已在對話中確認。第 12 節列出仍需你拍板的技術細項。

---

## 2. 身分模型（AuthN 對象）

- **對象**：只有內部員工（D1）。暫不含外部客戶／廠商。
- **帳號識別（D5 的比對依據）**：
  - IT 在後台**預先建立**使用者，填公司 email、啟用。
  - 員工在登入頁按「Google 登入」→ 選公司帳號 → 系統拿 Google 回傳的 email 比對「這個 email 是否已在系統且啟用」→ 是才授權。（這叫**預先佈建 + 社群登入 + 白名單**，比自助註冊更安全。）
  - **穩定內部鍵**：第一次 Google 登入成功後，把 Google 給的**永遠不變的使用者 ID（`sub`）**綁到這個帳號，之後認這個 ID，不認 email（email 可能會變）。另存**員工編號**作為跨系統對應鍵。
- **租戶**：**單租戶**（Ystravel 一家）。Logto 的 organization 功能保留但先不啟用。
- **身分唯一來源（Source of Truth）**：IdP。各系統不自己存密碼。

---

## 3. 認證機制（AuthN）

### 3.1 流程
```
員工 ──▶ AuthPortal（登入畫面）──▶ IdP（Logto，驗帳密/Google/2FA）
                                      │  發 Authorization Code
下游系統前端 ◀───────────────────────┘
   │  用 Code + PKCE 換 token
   ▼
IdP 發：access token(JWT, 短效) + refresh token(輪替) + id token
   │
前端帶 access token 呼叫 ──▶ 後端（用 IdP 的 JWKS 驗簽、驗 aud/exp）
                                ▼ 通過才進 PermissionsGuard
```

### 3.2 IdP 選型：建議 Logto
| | Logto（建議） | Keycloak（備案） |
|---|---|---|
| 定位 | 現代、開發者導向 | 企業級、老牌、功能最全 |
| 對 Vue3/NestJS | SDK 與文件新、上手快 | 可用但設定較重、學習曲線陡 |
| 自架 | Docker 一把梭、資源需求低 | 吃資源、JVM |
| 適合誰 | 單人／小團隊、要快 | 有專職維運、需要極多企業功能 |

以你「非本科、單打獨鬥、要快」的情況，**Logto 綜合最省心**。若未來確定要對接大量企業 SAML/LDAP 舊系統，Keycloak 才更划算。**此項列入 §12 待你最終拍板。**

### 3.3 Token 策略（D7）
- **access token**：JWT，短效（建議 15 分鐘），各系統後端用 IdP 公開的 JWKS 離線驗簽。
- **refresh token**：較長效（建議 7 天）、**啟用輪替（rotation）**、一次性、偵測重放即撤銷。
- **id token**：只取使用者基本資料，不當 API 憑證。

### 3.4 登入方式（D5）
- **帳密登入**：公司 email + 密碼。
- **Google 登入**：接 Logto 的 Google connector。
  - **關掉自助註冊**：只有 IT 預先建好的 email 能登入，外人有 Google 帳號也進不來。
  - **限制公司網域**：只接受公司網域帳號（Google `hd` 網域標記），多一道防線。

### 3.5 登出：Single Logout（D8）
- 目標：在任一系統按登出 → 所有系統的登入狀態一起清掉。
- 做法：OIDC **back-channel logout**——IdP 通知每個已登入的系統清 session。
- 提醒：SLO 比 SSO 複雜（要每個系統實作登出通知端點），但 Logto 支援，做得到。列為 Phase 1 要驗證的項目。

---

## 4. 授權模型（AuthZ）—— 一處管所有系統的角色權限（D6）

### 4.1 你要的效果
> 在 AuthService（Logto）後台，打開「員工 A」，可以在**同一個地方**設定他跨所有系統的權限：
> - CRM：匯入頁使用權、查詢同業客戶資料權
> - EIP：直客詢問統計表使用權、講座活動統計表使用權

### 4.2 怎麼達成（混合架構 C）
- **白話**：「你在後台勾了哪些權限」由**現有 AuthService 的管理後台**一處統一管；「員工實際點下去時各系統讓不讓他過」由各系統照權限放行。體驗上就是**一處管全部**。
- **技術**：權限的定義與指派沿用**現有 AuthService 的 `Application`/`Role`/`Permission`/`UserRole` 資料模型**（每個系統一筆 Application、權限綁到各 app）。使用者登入時由 Logto 發身分 token；各系統後端用 Logto JWKS 驗身分後，拿 Logto `sub` 對到本地 `AuthUser`，再由 `PermissionsGuard` 依 AuthService 的權限資料放行。
- **好處**：權限集中在你已建好、最貼合需求的地方；登入的身分驗證交給 Logto；新增一個 CRM 權限只在 AuthService 後台一個地方做。
- **對照**：Logto = 「這個人是誰」；AuthService = 「這個人在各系統能做什麼」。

### 4.3 權限命名
- 沿用既有慣例 `<系統>.<模組>.<動作>`，例如 `CRM.IMPORT.USE`、`CRM.CUSTOMER.READ`、`EIP.STATS_DIRECT.VIEW`。

---

## 5. 系統邊界與資料流

| 系統 | 角色 | 存什麼敏感資料 |
|---|---|---|
| AuthPortal | 登入 UI、導轉 | 不存，純前端 |
| IdP (Logto) | 身分、密碼、2FA、發 token、集中權限定義 | 員工帳密（雜湊）、email、角色權限對應 |
| CRM-Backend | 業務邏輯、CRM 授權執行 | 客戶 PII（Restricted） |
| EIP（未來） | 各業務模組 | 依模組而定 |

- 員工密碼只在 IdP，各系統拿不到密碼；客戶 PII 只在 CRM，IdP 不碰。職責清楚切開。

---

## 6. 帳號遷移／合併策略（D9，為未來 EIP 使用者預留）

- 現在 CRM 可自建使用者，未來 EIP 使用者要併過來——**現在就要把身分模型設計成能承受合併**：
  - 不用 email 當唯一鍵（email 會變）；用 IdP `sub` + 員工編號當穩定鍵。
  - 預留 **identity linking**：同一個人在不同來源（CRM 自建、EIP 舊帳號）的帳號，未來能綁成同一個身分。
  - 遷移時保留對照表（舊系統 user id ↔ 新 IdP sub），可追溯、可回滾。

---

## 7. 安全與稽核（SD 模板 §6 / baseline §6.3 必填）

| 欄位 | 內容 |
|---|---|
| 認證與授權方式 | OIDC（Auth Code + PKCE）＋ JWT/JWKS 驗簽；權限集中定義於 Logto、各系統 `PermissionsGuard` 執行 |
| 欄位加密/雜湊/遮罩 | 密碼由 IdP 標準雜湊；secrets 走環境變數不寫死；客戶身分證件值採 `hash + masked`（baseline §7.2） |
| Audit log | 見 §8（跨系統共用機制） |
| 備份/還原 | IdP 與各業務資料庫分開備份；備份亦屬敏感資料 |
| 錯誤處理與告警 | 登入失敗鎖定、token 驗簽失敗一律 401、refresh 重放偵測即撤銷並告警 |
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
- **本地開發**：沿用 `Ystravel-LocalDocker`，把 Logto 加為一個 service，與 MySQL/MariaDB 並列。
- **環境分離**：local / staging / production 的 IdP 與 secrets 完全分離（baseline §7.5）。
- **secrets**：client secret、JWKS、DB 連線字串全走環境變數，不進 repo。

---

## 10. 落地順序（Phase 1 → 2 預告）
1. **Phase 1a**：本地把 Logto 跑起來 + 建測試員工帳號 + AuthPortal 能導到 Logto 登入（含 Google）並拿到 token。
2. **Phase 1b**：後端加 `JwtAuthGuard`（驗 JWKS），做一支受保護 `GET /me` 證明 token 有效。
3. **Phase 1c**：Logto 集中定義 CRM 的角色/權限（API resource）＋ 後端 `PermissionsGuard` 照 scopes 放行。
4. **Phase 1d**：驗證 Single Logout。
5. **Phase 2**：CRM-Frontend 走 AuthPortal 登入打 CRM API；把既有 `crm-import-cleaning` 疊到權限控管之下。

> 每個 Phase 各走一次 feature pipeline（PRD 可精簡，但因涉及 SSO，SA/SD 需補資安欄位）。

---

## 11. 測試策略（概要）
- IdP 驗簽、Guard、權限查詢、SLO 屬高風險，優先寫 Integration + BDD（`jest-cucumber`）。
- 必測：「無 token 打受保護 API → 401」「有 token 無權限 → 403」「權限異動要寫 audit log」「登出後其他系統 session 失效」。

---

## 12. 決策事項（2026-07-08 拍板結果）

| # | 問題 | 結果 |
|---|---|---|
| Q1 | IdP 最終選型 | ✅ **Logto** |
| Q2 | access / refresh token 效期 | ✅ access **15min**、refresh **7 天**（7 天為「閒置多久要重登」，有在用會無感延續） |
| Q3 | 2FA 強制範圍 | ✅ **管理員/高權限強制、一般員工先自願**（系統一律支援 TOTP，僅「強制範圍」分階段；一般員工多走 Google 登入已受 Google 2FA 保護，之後再視情況全面要求） |
| Q4 | 密碼政策 | ✅ 採最小現代政策（12 碼、擋弱密碼、不強制定期換、失敗鎖定），已寫成 `docs/process/PASSWORD_POLICY.md` |
| Q5 | 登入方式 | ✅ **帳密 + Google 並存**（帳密當備援，Google 中斷時仍可登入） |

---

*確認第 12 節後，下一步：把 Phase 1a 當作第一個 feature，走一次 `feature-kickoff`。*
*UI/UX 設計規範見 `docs/design-system/foundation.md` 與 `docs/development-process/10-uiux-guide.md`。*

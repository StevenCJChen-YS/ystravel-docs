# SD: AuthService Native 登入 v2（Phase 1a）

| 項目 | 內容 |
|---|---|
| 對應 SA | ./sa.md |
| 對應地基 | ../auth-foundation/architecture-decisions.md §0.6 |
| 狀態 | 已核准（2026-07-08 第一刀＋第二刀完成） |
| 建立日期 | 2026-07-09 |

## 0. 白話摘要

AuthService 自己做完整登入：`POST /auth/login` 驗帳密、發 access token（RS256）+ refresh token；`POST /auth/refresh` 輪替；`POST /auth/logout` 撤銷；`GET /auth/me` 回身分權限。access token 改用**非對稱金鑰（RS256）**簽章，AuthService 持 private key，另開 `GET /.well-known/jwks.json` 讓下游只拿 public key 驗 token。DB 用回 `passwordHash` + `RefreshToken` 表。**本文件重點在第 10 節：把幾個尚未完成的安全債排出必修優先序，不可默默視為完成。**

## 1. 系統架構

異動範圍：

- **Ystravel-AuthService（後端）**：
  - `auth/auth.controller.ts`：`POST /auth/login`、`/auth/refresh`、`/auth/logout`、`GET /auth/me`（native flow，已復原）。
  - `auth/auth.config.ts` + `validateAuthConfig()`：啟動時驗 RS256 key pair 與 `JWT_ACCESS_KEY_ID`，缺就 fail fast；並簽/驗一段 probe 確認 private/public 是同一組。
  - `auth/jwt-auth.guard.ts`：用 public key 驗 access token，限定 `algorithms:['RS256']`（拒 HS256/none）。
  - `auth/jwks.controller.ts`：`GET /.well-known/jwks.json` 輸出 public JWKS。
  - 授權沿用 `permissions.guard.ts` + `permissions.decorator.ts`；權限組裝沿用 `buildAuthUser`（角色 isActive 過濾、override ALLOW/DENY、override 過期過濾）。
- **Ystravel-AuthPortal（前端）**：回復原本 `LoginPage.vue`（Email+密碼、記住我、忘記密碼、Google 按鈕位置）、native `auth-api.ts`、native auth store；移除 Logto 的 `CallbackPage.vue` 與 `services/logto.ts`。
- **Ystravel-LocalDocker**：回 main 版本（MySQL 單一 DB），移除 Logto/Postgres。

> 全域路由前綴為 `api/auth`，故實際對外路徑為 `/api/auth/auth/login`、`/api/auth/.well-known/jwks.json` 等。路徑略顯冗餘（prefix `auth` + controller `auth`），可用但之後可整理（記於待辦，不在本刀）。

## 2. API 設計

| Method | Path（含 global prefix `api/auth`） | 說明 | 權限 |
|---|---|---|---|
| POST | /api/auth/auth/login | 帳密登入，回 access+refresh+user | 公開 |
| POST | /api/auth/auth/refresh | refresh token 輪替 | 需有效 refresh token |
| POST | /api/auth/auth/logout | 撤銷 refresh token | 帶 refresh token |
| GET | /api/auth/auth/me | 回身分與角色/權限 | 需有效 access token |
| GET | /api/auth/.well-known/jwks.json | 輸出 public JWKS | 公開 |

`GET /auth/me` 回傳結構 `{ user: { sub, username, displayName, roles, permissions } }`，login/refresh 另附 `accessToken`、`refreshToken`。沿用現有結構，不另開 api-spec.md。

## 3. DB Schema（Prisma）

`AuthUser`：
- `passwordHash String?`（已加回；bcrypt 雜湊）。
- `logtoSub String? @unique`（nullable，暫留作未來 external identity link，本階段不用）。
- 既有：`Application`／`Role`／`Permission`／`UserRole`／`RolePermission`／`UserPermissionOverride`／`AuditLog` 全保留。

`RefreshToken`（已加回）：`id`、`userId`、`tokenHash`、`expiresAt`、`revokedAt`、`createdAt`。

已套用 migration：
- `20260708223000_restore_native_auth_fields`（加回 passwordHash + RefreshToken）。

**種子資料**：`scripts/create-first-admin.ts` 建 Super Admin `steven_chen@ystravel.com.tw`（`AUTH_SUPER_ADMIN`）；舊 `admin` 已停用未刪。跑 `npm run permissions:sync` + `npm run admin:create`。

## 4. Service / Module 分層

沿用既有 `src/auth/` 結構：
- controller：`auth.controller.ts`（login/refresh/logout/me）、`jwks.controller.ts`。
- config/驗證：`auth.config.ts`（RS256 key、效期、`validateAuthConfig`）。
- guard：`jwt-auth.guard.ts`（RS256 驗簽）、`permissions.guard.ts`。
- 權限組裝：`buildAuthUser`（controller 內）。
- **套件**：`@nestjs/jwt`（簽 RS256）、`bcrypt`（密碼/refresh 雜湊）、`crypto.randomBytes`（refresh token）。

## 5. 認證與授權方式

- **認證**：AuthService 自建帳密（bcrypt.compare）。
- **簽章**：access token `alg=RS256`、帶 `kid`（`JWT_ACCESS_KEY_ID`）、payload = `{ sub, username, displayName, roles, permissions }`、`expiresIn=15m`。
- **驗證**：`JwtAuthGuard` 用 public key 驗簽，限定 `RS256`。
- **授權**：沿用 `PermissionsGuard`；權限來源為本地 Role/Permission/Override。
- **refresh**：`randomBytes(48)` base64url 明文回前端一次；DB 存 bcrypt 雜湊；輪替時撤舊發新。

### Environment（RS256 / JWKS contract）

| 變數 | 說明 |
|---|---|
| `JWT_ACCESS_PRIVATE_KEY_BASE64` | PEM PKCS#8 private key 的 base64。只在 AuthService，不給下游。 |
| `JWT_ACCESS_PUBLIC_KEY_BASE64` | PEM SPKI public key 的 base64。驗自身 token + 輸出 JWKS。 |
| `JWT_ACCESS_KEY_ID` | token header 的 `kid`；rotation 換新值。 |
| `JWT_ACCESS_EXPIRES_IN` | access 效期，預設 `15m`。 |
| `JWT_REFRESH_EXPIRES_IN` | refresh 效期，預設 `7d`。 |

> 本機 `.env` 的 RS256 私鑰是 local-dev key，**不進 git**；`.env.example` 只放 placeholder + 產 key pair 指令。缺這三個 key 相關變數，AuthService 啟動即 fail。

## 6. 安全與稽核（必填 — 依 SECURITY_AND_ISO27001_BASELINE.md §6.3）

| 欄位 | 內容 |
|---|---|
| 敏感欄位保護 | 密碼 bcrypt 雜湊、refresh token 雜湊、明文皆不落地；access token 走 RS256 非對稱金鑰，下游只需 public key |
| Audit log | 集中在 `AuditLog`：LOGIN、TOKEN_REFRESH、LOGOUT。**登入失敗目前未寫，需補（§9 gap）** |
| 備份 / 還原 | 單一 MySQL，納入既有備份範圍（移除 Logto 後不再有第二個 DB） |
| 錯誤處理與告警 | 驗簽失敗一律 401；帳密錯統一模糊訊息「Invalid username or password.」 |
| 測試資料策略 | 測試用少量假帳號，不使用真實客戶資料 |

## 7. 效能與非功能需求

- access token 驗證無狀態（public key 可快取），低成本。
- ✅ **效能債已修（2026-07-09，§10 P1）**：改用 selector 唯一索引查找（`WHERE selector = ?`）+ verifier SHA-256 固定時間比對，取代全表逐筆 bcrypt；並加入 reuse detection（已撤銷 token 被重放時連坐撤銷整組）。

## 8. 錯誤碼

| 情境 | HTTP Status | 說明 |
|---|---|---|
| 帳號不存在／無 passwordHash／停用／密碼錯 | 401 | 統一「Invalid username or password.」 |
| 無 access token／token 無效或過期 | 401 | 未授權 |
| 偽造 HS256/none token | 401 | guard 限 RS256 |
| refresh token 缺少 | 400 | refreshToken is required. |
| refresh token 無效／過期／已撤銷／帳號停用 | 401 | Invalid refresh token. |

## 9. 測試策略 + 已知 gap

依 07-testing-strategy.md：
- **BDD（jest-cucumber）✅ 已接線（2026-07-09）**：`src/auth/login-access.feature` + `login-access.steps.ts` — 4 場景（登入成功含 RS256 token 簽發、不存在/密碼錯拒絕含統一訊息與 LOGIN_DENIED、停用拒絕、停用角色權限不計入）。steps 走**真的 `AuthController.login()`**（bcrypt 驗密、RS256 簽 token、buildAuthUser、audit），只 mock Prisma。feature 檔改為英文 Gherkin 關鍵字＋中文內文（對齊 CRM-Backend 前例，jest-cucumber 對 `# language:` 支援不可靠）；jest `testRegex` 改收 `(spec|steps)`。
- **Unit/整合（已綠）**：`jwt-rs256.spec.ts` — 2 suites / 6 tests：RS256 接受、HS256 拒絕、JWKS metadata、kid 必填、private/public key mismatch fail-fast。
- **Live smoke（已過）**：手簽 RS256 token 打 `/auth/me` 回 200；legacy HS256 回 401；JWKS 回 `kty=RSA/alg=RS256/kid=local-dev-key-1`。

### ✅ Gap 已修：登入失敗寫 audit（2026-07-09）
原本 `login()` 失敗路徑直接 throw `UnauthorizedException`、未寫 audit；`login-access.feature` 要求登入失敗要留痕。**已補**：三種失敗路徑（`USER_NOT_FOUND`／`USER_DISABLED`／`BAD_PASSWORD`，`NO_PASSWORD` 併入）統一寫 `action=LOGIN_DENIED`，reason 放 `diff.reason`，並記 `attemptedUsername`（**絕不記密碼**）；帳號不存在時 `actorUserId=null`。新增 `login-audit.spec.ts`（4 tests）鎖定此行為，含「密碼不得出現在 audit」的斷言。jest 全綠（3 suites / 10 tests）、lint 綠。

## 10. 後續必修：安全債優先序（不可默默視為完成）

> 來源：README §4 安全債表。native v2 = 回到 native flow **並修掉這些**，不是「復原就好」。

| 優先 | 項目 | 現況問題 | v2 方向 | 狀態 |
|---|---|---|---|---|
| ✅ 已完成 | Token 簽章 | HS256 共用密鑰，任一系統漏 secret 可偽造全站 token | RS256/JWKS，下游只拿 public key | 第二刀已完成、已驗證 |
| ✅ 已完成 | Config validation | 缺 JWT 設定時登入才 500 | 啟動 fail fast + probe 驗 key pair | 第一刀已完成 |
| ✅ 已完成 | 登入失敗 audit | 失敗路徑未留痕，BDD 場景要求要留 | login 失敗補 LOGIN_DENIED audit + login-audit.spec.ts | 2026-07-09 完成（§9） |
| ✅ 已完成 | BDD 接線 | AuthService 未裝 jest-cucumber | jest-cucumber@4.5.0 + login-access.steps.ts（4 場景全綠） | 2026-07-09 完成（§9） |
| ✅ 已完成 | Refresh token 查找 | `findActiveRefreshToken` 全表掃描逐筆 bcrypt | selector（明文唯一索引）+ verifier（SHA-256 固定時間比對）+ reuse detection（連坐撤銷整組 token + TOKEN_REUSE_DETECTED audit） | 2026-07-09 完成；本機 DB migrate 實套 + e2e smoke 12/12、jest 21 綠 |
| ✅ 已完成 | 登入防暴力 | 無失敗計數/鎖定/告警 | 依 **(帳號, IP) 複合**失敗計數、暫時鎖定（連錯 **10** 次鎖 **15** 分，可用 env 調）、`LOGIN_LOCKED` audit/alert；被鎖回 **429**「稍後再試」 | 2026-07-09 完成（§10.1）；code+jest 綠，migration 待本機實套 |
| ✅ 已完成 | 密碼政策落地 | 僅檢查 8 碼，政策文件其餘規則未落地 | ≥12 碼、黑名單+zxcvbn 擋弱/可預測、擋個資型密碼、重設不可與現用相同；套用於 create/reset | 2026-07-09 完成（§10.2）；**自助 email 重設流程另立一輪** |
| **P4** | Google 登入 | 前端有按鈕、後端無端點 | 對接 Google OAuth；預先佈建+公司網域+external subject linking（用 `logtoSub` 欄位） | 待做 |
| **P5** | 2FA | 管理員強制已拍板、native 未實作 | TOTP；高權限強制、一般員工自願 | 待做 |
| **P6** | Single Logout | `/auth/logout` 只撤 refresh，access 到期前仍短暫有效 | session registry / token version / revocation cache，再談 SLO | 待做 |

### 10.1 P2 登入防暴力（brute-force throttle，2026-07-09 完成）

設計選擇（Steven 2026-07-09 拍板，偏離草案處已標註）：

- **鎖定維度：(username, IP) 複合**（非草案的「帳號或 IP 擇一」）。只鎖帳號會被惡意連錯把別人鎖死；只鎖 IP 對同出口多人不公平。以正規化 username（`trim().toLowerCase()`）+ client IP 為 key。
- **門檻／時長：連錯 10 次鎖 15 分**（草案是 5 次；改 10 次降低誤鎖真人）。皆可 env 覆寫：`LOGIN_MAX_FAILURES` / `LOGIN_LOCK_MINUTES` / `LOGIN_FAILURE_WINDOW_MINUTES`（預設 10 / 15 / 15）。
- **計數儲存：專用表 `login_attempts`**（非直接掃 `audit_logs`）。欄位 `(username, ipAddress)` 唯一索引 + `failedCount` / `lockedUntil` / `lastFailedAt`；查詢快、鎖定狀態可原子維護。migration `20260709140000_login_attempt_throttle`。
- **被鎖回應：HTTP 429「Too many failed login attempts. Please try again later.」**（前端可據狀態碼區分「稍後再試」；訊息本身仍不透露帳號是否存在）。

流程：`login()` 進入先 `assertNotLocked()`——若該 (username, IP) 仍在鎖定窗內，直接 429 且**不查帳號、不比密碼**（鎖定期間無法被試探）。任何憑證失敗（`USER_NOT_FOUND`/`USER_DISABLED`/`NO_PASSWORD`/`BAD_PASSWORD`）走 `denyLogin()`：`registerFailedAttempt()` 累加計數（上次失敗超過 window 或前次鎖已過期則歸零重算），仍寫既有 `LOGIN_DENIED` audit；當這次失敗觸發鎖定，另寫一筆 `LOGIN_LOCKED`（reason `THRESHOLD_REACHED`）作告警並回 429。登入成功則 `clearFailedAttempts()` 清該 (username, IP) 的計數。成功登入的 `LOGIN` audit 亦補記 `ipAddress`。

告警落點：查 `audit_logs WHERE action='LOGIN_LOCKED'`——`THRESHOLD_REACHED`=剛觸發鎖定、`ACCOUNT_LOCKED`=鎖定期間又被嘗試。

測試：`login-throttle.spec.ts`（6 tests，真 `AuthController.login()` + 有狀態的 in-memory `loginAttempt` fake）：達門檻由 401 轉 429、鎖定期短路不查帳號、(username,IP) 各自獨立、逾窗計數歸零、成功清零、密碼不入 audit。全套 jest 6 suites / 27 tests 綠、lint 綠、build 綠。

**未套用**：本機 DB migration 尚未實跑（`migrate deploy` 被 auto 模式當 production 擋下）；套用指令見交接。競態備註：`registerFailedAttempt` 用 read-then-upsert，內部低併發可接受；極端並發下計數可能少算 1，不影響鎖定最終生效。

### 10.2 P3 密碼政策落地（password policy，2026-07-09 完成）

範圍（Steven 2026-07-09 拍板）：**密碼強度政策 + 更新政策文件**；弱/外洩密碼用**內建黑名單 + zxcvbn**（離線、無外部連線）。自助「忘記密碼」email 重設流程（需寄信基礎建設 + reset token 表）**另立一輪**，本輪不做。

實作 `src/auth/password-policy.ts`（`assertPasswordMeetsPolicy(password, owner)`），套用於 `UsersController.create()` 與 `resetPassword()`：

- **≥ 12 碼**（原本只檢查 8；`MIN_PASSWORD_LENGTH` 匯出供前端/測試對齊）。
- **擋弱/可預測**：小型常見密碼黑名單（明確拒絕訊息）+ `zxcvbn@4.4.2` 強度估計要求分數 **≥ 3**（0–4）。zxcvbn 內建字典即涵蓋常見/外洩密碼與鍵盤序列、leetspeak。
- **擋個資型密碼**：username / email（含 @ 前段）/ 顯示名稱 / 員工編號，精確內嵌硬擋（BadRequest），並把這些識別碼餵給 zxcvbn 的 `user_inputs`，讓「姓名+數字」這類近似也算可猜。
- **重設不可與現用相同**（政策 §2）：`resetPassword` 以 bcrypt 比對現有 `passwordHash`，相同則拒絕。
- 錯誤訊息全英文（沿慣例），前端 `auth-api.ts` 翻譯表補 5 筆對應中文（P2 教訓：訊息靠英文原字串當 key）。

前端（AuthPortal）：`AdminUsersPage.vue` 建立/重設的長度前檢 8→12；兩處 catch 改用 `getAuthApiErrorMessage` 讓後端政策拒絕原因浮現（原本吞成「新增/重設失敗」）。

政策文件 `docs/process/PASSWORD_POLICY.md` 同步更新為 native 現實（原文假設 Logto）：擋弱密碼機制改 AuthService 自建、§3 鎖定 5 次→10 次且 (帳號,IP) 複合、§7 儲存改 bcrypt cost 12、§5 標註自助 email 重設待後續。

測試：`password-policy.spec.ts`（8 tests：長度、常見、個資內嵌、近似、強通過、訊息不回顯密碼）。全套 jest **7 suites / 35 tests 綠**、lint 綠、build 綠；AuthPortal `vue-tsc` 綠。

**未做（後續一輪）**：自助「忘記密碼」email 重設（reset token 表 + 寄信）、管理員「不得得知明文／首登強制改密」目標態、密碼歷史多筆（目前只擋與「現用」相同）。

### Key rotation note
第一階段單一 active key。未來 rotation：JWKS 同時輸出 current + previous public keys；簽發只用 current private key；previous key 至少保留一個 access token 最長效期後再移除。

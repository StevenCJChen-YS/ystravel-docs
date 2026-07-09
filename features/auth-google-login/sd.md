# SD: Google 登入整合（Phase 1d）

| 項目 | 內容 |
|---|---|
| 對應 SA | ./sa.md |
| 狀態 | 已確認（2026-07-09） |
| 建立日期 | 2026-07-09 |
| 對應地基 | ../auth-foundation/architecture-decisions.md §0.6、§10 Phase 1d |

## 0. 白話摘要

這份 SD 的核心是把 Google 驗過的身分，安全地接進現有 native auth 架構，而不把整套 session 模型搞成兩半。做法是：**AuthService 後端自己處理 Google OAuth**、**用獨立 `AuthIdentity` 模型保存外部身分**、**callback 回前端只帶一次性 `authCode`**，再由前端換回現有 native access token / refresh token。這樣可以避免 token 出現在 URL，也避免為了 Google 登入臨時引進一整套 Cookie session，讓現有帳密登入和 Google 登入變成兩種不同架構。

## 1. 系統架構

異動範圍：

- **Ystravel-AuthService（後端）**
  - `auth.controller.ts`
    - 新增 `GET /auth/google`
    - 新增 `GET /auth/google/callback`
    - 新增 `POST /auth/google/exchange`
  - 新增 Google OAuth config / service
  - 新增外部身分綁定 service
  - 新增一次性 OAuth state / exchange code 驗證
  - `AuthUser` 移除 `logtoSub`
  - 新增 `AuthIdentity`、`OAuthState`、`AuthExchangeCode` Prisma model
- **Ystravel-AuthPortal（前端）**
  - `LoginPage.vue` 的 Google 按鈕維持，但導向既有 AuthService 路徑
  - 新增 `CallbackPage.vue`
  - `auth store` 增加用交換碼完成登入的方法
  - router 新增 `/callback`
- **docs**
  - 新增本 feature 的 PRD / Example Mapping / SA / SD

## 2. API 設計

| Method | Path（含 global prefix `api/auth`） | 說明 | 權限 |
|---|---|---|---|
| GET | /api/auth/auth/google | 啟動 Google OAuth 流程，導向 Google | 公開 |
| GET | /api/auth/auth/google/callback | Google callback，驗證後回前端 callback 頁並附一次性 `authCode` | 公開 |
| POST | /api/auth/auth/google/exchange | 用一次性 `authCode` 換 native access token / refresh token | 公開 |
| GET | /api/auth/auth/me | 不變；登入完成後取本地 user / roles / permissions | 需 bearer token |

### 2.1 Request / Response 摘要

`GET /auth/google`
- Query:
  - `app?: string`
  - `redirectUri?: string`（只接受站內相對路徑）
- Response:
  - `302` redirect 到 Google OAuth

`GET /auth/google/callback`
- Query:
  - `code`
  - `state`
- Response:
  - 成功：`302` redirect 到 `AuthPortal /Auth/callback?code=<authCode>`
  - 失敗：`302` redirect 到 `AuthPortal /Auth/login?error=<generic>`

`POST /auth/google/exchange`
- Request:
  - `{ code: string }`
- Response:
  - `{ accessToken, refreshToken, user, redirectUri }`

## 3. DB Schema 異動

### 3.1 Prisma model

1. `AuthUser`
   - **移除** `logtoSub`

2. `AuthIdentity`
   - `id`
   - `userId`
   - `provider`（enum，首個值 = `GOOGLE`）
   - `subject`
   - `email`
   - `createdAt`
   - `updatedAt`
   - unique:
     - `@@unique([provider, subject])`
     - `@@unique([userId, provider])`

3. `OAuthState`
   - `id`
   - `stateHash`
   - `nonceHash`
   - `appCode`
   - `redirectUri`
   - `expiresAt`
   - `usedAt`
   - `createdAt`

4. `AuthExchangeCode`
   - `id`
   - `codeHash`
   - `userId`
   - `provider`
   - `appCode`
   - `redirectUri`
   - `expiresAt`
   - `usedAt`
   - `createdAt`

### 3.2 Migration 規劃

Migration 名稱規劃：`20260709170000_google_auth_identity_exchange`

此 migration 需：
- 刪除 `auth_users.logto_sub`
- 新增 `auth_identities`
- 新增 `oauth_states`
- 新增 `auth_exchange_codes`

## 4. Service / Module 分層

沿用既有 `src/auth/` pattern，不引入新風格：

- `auth.controller.ts`：保留既有 native login / refresh / logout / me，再新增 Google 相關 endpoints
- `google-auth.config.ts`：讀取與驗證 Google OAuth env
- `google-oauth.service.ts`
  - 產生 Google auth URL
  - 處理 callback code → token
  - 驗證 Google ID token
- `external-identity.service.ts`
  - 查找 / 建立 `AuthIdentity`
  - 綁定本地 `AuthUser`
- `auth-exchange.service.ts`
  - 產生 / 驗證一次性 `authCode`
- Prisma 仍走既有 `PrismaService`

### 套件選擇

- 新增 `google-auth-library`（Google 官方 OAuth / ID token 驗證函式庫）

理由：
- 比手寫 Google token 驗證更穩定
- 比把整套流程塞進 `passport-google-oauth20` 更顯式，較容易控制 state / nonce / audit / authCode 交換
- 對資安文件說明也更直觀

## 5. 認證與授權方式

- **Google 認證**：由 AuthService 後端處理 Google OAuth Authorization Code flow
- **本地登入完成**：前端用一次性 `authCode` 換回**既有 native** access token / refresh token
- **授權**：完全沿用既有 `JwtAuthGuard`、`PermissionsGuard`、`buildAuthUser`
- **permission code**：本功能不新增 permission code

## 6. 安全與稽核（必填 — 依 SECURITY_AND_ISO27001_BASELINE.md §6.3）

| 欄位 | 內容 |
|---|---|
| 敏感欄位保護 | `state` / `nonce` / `authCode` 僅存 hash；Google subject 與 token 不寫入 URL；access token / refresh token 不放 callback query |
| Audit log | Google 登入成功（`LOGIN`）、白名單拒絕 / 帳號停用 / state 無效 / 網域不符 / 交換碼重放（`LOGIN_DENIED`），都記 `provider=GOOGLE` 與 reason |
| 備份 / 還原 | 新增三張表都在 AuthService DB，沿用既有備份策略；無第二個本地身分 DB |
| 錯誤處理與告警 | callback 驗證失敗一律拒絕並導回登入頁；exchange code 過期 / 重放需記 audit；production 僅允許公司網域 |
| 測試資料策略 | 測試使用假 email / 假 Google subject，不使用真實員工資料；Google callback 測試以 mock claims 代替真連線 |

### 6.1 ISO 27001 對齊重點

- **最小揭露**：前端只拿到完成登入所需的最小資料
- **可稽核**：成功 / 失敗 / 重放都有 audit
- **風險控制**：避免 token 出 URL、避免 open redirect、避免白名單放行錯誤
- **職責分離**：Google 只證明外部身分，最終授權仍由 AuthService 控制

### 6.2 Non-production 測試帳號規則

- production 若只設定單一公司網域，Google Authorization URL 會帶 `hd=<company-domain>`，把帳號選擇器導向公司帳號。
- non-production 若設定 `GOOGLE_ALLOWED_TEST_EMAILS`，則**不送 `hd` hint**，避免私人 Gmail 測試帳號在 Google 選號畫面被隱藏。
- 後端放行判斷不變：
  - production：只接受允許的公司網域帳號
  - non-production：公司網域帳號可登入；另可額外接受 `GOOGLE_ALLOWED_TEST_EMAILS` 白名單
- `GOOGLE_ALLOWED_TEST_EMAILS` **只影響 Google 登入流程，不影響一般帳密登入**。

## 7. 效能與非功能需求

- Google 登入流量遠低於一般 API，主要瓶頸不在效能，而在安全控制與可追查性
- `AuthIdentity` 與 `AuthExchangeCode` 查詢都走索引欄位，不應有明顯效能瓶頸
- `OAuthState` 與 `AuthExchangeCode` 屬短生命週期資料，後續可加清理 job；本輪先不做背景清理，靠逾時失效即可

## 8. 錯誤碼

| 情境 | HTTP Status | 說明 |
|---|---|---|
| `GET /auth/google` 的 `redirectUri` 非法 | 400 | 非法登入啟動參數 |
| callback `state` 無效 / 已過期 | 401 | 拒絕登入 |
| Google token 驗證失敗 | 401 | 拒絕登入 |
| 不在白名單 / 帳號停用 / 網域不符 | 401 | 拒絕登入，前端顯示模糊訊息 |
| `POST /auth/google/exchange` 的 `authCode` 無效 / 已過期 / 已使用 | 401 | 需重新登入 |

> 雖然 callback 最終會用 `302` 導回前端頁面，但內部判斷仍依上述錯誤語意區分，方便測試與 audit。

## 9. 測試策略

依 `07-testing-strategy.md`：

- **BDD（jest-cucumber）**
  - 新增 `src/auth/google-login.feature`
  - 覆蓋至少這些 scenario：
    - 已預建 ACTIVE 員工第一次 Google 登入成功
    - 已綁定 Google 身分再次登入成功
    - 不在白名單的 Google 帳號被拒絕
    - 停用帳號被拒絕
    - `authCode` 被重複使用時被拒絕

- **Unit / Integration**
  - `redirectUri` 驗證
  - `AuthIdentity` 綁定邏輯
  - `OAuthState` / `AuthExchangeCode` 的有效期與單次使用規則
  - Google callback 驗證失敗的負向情境

- **前端**
  - 仍以 Steven 手動 QA 為主
  - 至少手動驗：
    - Google 成功登入
    - callback 失敗回登入頁
    - 錯誤 toast 文案

## 10. 待決策事項

本輪架構取捨已在 PRD 階段定案，**無新增待決策事項**。  
實作前提（非設計決策）：

- [ ] `.env` 提供 Google client id / secret / callback URL
- [ ] `.env` 提供 allowed hosted domains
- [ ] 非 production 若需要私人 gmail 測試，提供 allowed test emails

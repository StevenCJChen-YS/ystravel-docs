# SD: Logto 登入整合（Phase 1a）

| 項目 | 內容 |
|---|---|
| 對應 SA | ./sa.md |
| 對應地基 | ../auth-foundation/architecture-decisions.md |
| 狀態 | 已核准（2026-07-08） |
| 建立日期 | 2026-07-08 |

## 0. 白話摘要

把 AuthService 現有的「帳密登入」關掉，改由 Logto 負責登入（帳密＋Google），Logto 發出標準 OIDC token；AuthService 改成「驗證 Logto 的 token（用公鑰）→ 把身分對應到本地員工帳號 → 回傳角色與權限」。資料庫只小改一處：員工資料表加一個「Logto 身分連結欄位（logtoSub）」，原本的密碼欄位停用（先保留、日後全面切換後再移除）。需要你拍板的有三個取捨（第 10 節），都附了建議。

## 1. 系統架構

異動範圍：

- **Logto（新增服務）**：跑在本地 `Ystravel-LocalDocker`，設定帳密登入、Google connector、公司網域限制。
- **Ystravel-AuthService（後端，改造）**：
  - `auth/jwt-auth.guard.ts`：從「驗自簽 JWT」改為「驗 Logto token（JWKS 公鑰）」。
  - 新增身分解析：token → 本地 `AuthUser`（先 logtoSub、否則 email 首次綁定）→ 檢查 ACTIVE。
  - `GET /auth/me`：保留，回傳身分 + 角色/權限（沿用 `buildAuthUser`）。
  - **退役**：`POST /auth/login`、`POST /auth/refresh`（改由 Logto）；`RefreshToken` 表停用。
  - **新增 Logto webhook 接收端點**：接收 Logto 的登入事件（含失敗），統一寫入既有 `AuditLog`（Q2=b）。
  - Audit：登入成功、我方拒絕、以及 Logto 推送的事件，全部集中在 `AuditLog`。
- **Ystravel-AuthPortal（前端，改造）**：
  - `LoginPage.vue`：帳密與 Google 按鈕改為導向 Logto（用 Logto 官方 SDK 的授權碼＋PKCE 流程）。
  - 登入回來後呼叫 `GET /auth/me` 取身分，成功導轉 `/my-apps`。

## 2. API 設計

| Method | Path | 說明 | 權限 |
|---|---|---|---|
| GET | /auth/me | 驗 Logto token、回傳身分與角色/權限 | 需有效 Logto token |
| POST | /webhooks/logto | 接收 Logto 登入事件、寫入 AuditLog | 以 Logto webhook 簽章驗證 |
| (退役) | POST /auth/login、/auth/refresh | 改由 Logto 負責 | — |

request/response 型別細節：本階段 `GET /auth/me` 沿用現有回傳結構（`{ user: { sub, username, displayName, roles, permissions } }`），故不另開 api-spec.md；如後續有異動再補。

## 3. DB Schema 異動（Prisma）

`AuthUser` model：
- 新增 `logtoSub String? @unique @db.VarChar(255)` — 綁定 Logto 身分。
- **移除 `passwordHash`**（現有皆測試資料，可清除重建，Q3=b）。

Migration 名稱規劃：`switch_auth_user_to_logto`（加 logtoSub、移除 passwordHash）。
`RefreshToken` 表本階段停用（refresh 交給 Logto）。

**種子資料（重置後）**：建立受保護的 Super Admin（owner，真實公司 email、`SUPER_ADMIN` 角色、`isSystem=true`）＋ 少量測試員工帳號；對應在 Logto 也建立同 email 帳號。

## 4. Service / Module 分層

沿用既有 `src/auth/` 結構，不引入新分層：
- guard 改造在 `jwt-auth.guard.ts`；
- 身分解析邏輯放 `auth` 模組內的 service（沿用 `PrismaService`）；
- 權限組裝沿用既有 `buildAuthUser`。

**新增套件**（需你知悉，理由充分）：Logto token 驗證需要 JWKS 驗簽套件（如 `jose` 或 `jwks-rsa`）；前端需 Logto 官方 SDK。皆為必要，非可有可無。

## 5. 認證與授權方式

- **認證**：Logto（OIDC Authorization Code + PKCE）。
- **驗證**：AuthService 以 Logto 的 JWKS 公鑰驗 token（檢查簽章、`iss`、`aud`、`exp`）。
- **授權**：沿用既有 `PermissionsGuard` 與 `permissions.decorator`；權限來源仍是本地 Role/Permission。
- 本功能不新增 permission code。

## 6. 安全與稽核（必填 — 依 SECURITY_AND_ISO27001_BASELINE.md §6.3）

| 欄位 | 內容 |
|---|---|
| 敏感欄位保護 | 密碼移出 AuthService（改由 Logto 保管雜湊）；token 走非對稱金鑰，各系統只需公鑰 |
| Audit log | 統一集中在 `AuditLog`：登入成功、我方拒絕（未授權/停用）、以及 Logto 經 webhook 推送的登入事件（含密碼錯等失敗）。稽核只查一處。 |
| 備份 / 還原 | 新增 Logto 資料庫，納入備份範圍（與 AuthService DB 分開） |
| 錯誤處理與告警 | token 驗簽失敗一律 401；email 比對到多筆視為異常並告警 |
| 測試資料策略 | 測試用少量假帳號（假 email/displayName），不使用真實員工或客戶資料 |

## 7. 效能與非功能需求

- token 驗證用 JWKS，公鑰可快取，屬低成本、無狀態驗證。
- 移除既有 `findActiveRefreshToken` 全表掃描（refresh 交給 Logto），效能疑慮一併解除。
- 其餘沿用既有標準。

## 8. 錯誤碼

| 情境 | HTTP Status | 說明 |
|---|---|---|
| 無 token / token 無效或過期 | 401 | 未授權 |
| Logto 驗證過但本地無對應帳號 / 帳號停用 | 401（對外統一訊息「帳號或密碼錯誤」） | 白名單 fail-closed |
| email 比對到多筆（異常） | 401 + 內部告警 | 不應發生 |

## 9. 測試策略

依 07-testing-strategy.md：
- **BDD（jest-cucumber）**：`src/auth/login-access.feature` 的 5 個情境（白名單、停用、角色停用、audit）。
  - 註：AuthService 目前尚未裝 jest-cucumber（CRM-Backend 已有），實作時需先導入。
- **Integration**：`GET /auth/me` 在有效/無效 token 下的行為。
- **手動 QA**：Logto 設定面（帳密正確性、Google 流程、公司網域限制）——屬 Logto 設定，非本專案程式碼。

## 10. 待決策事項（需要 Steven 拍板，皆附建議）

| # | 取捨 | 結果（2026-07-08 定案） |
|---|---|---|
| Q1 | 登入 token 放哪、由誰換 | ✅ **(a) 前端 Logto SDK（Authorization Code + PKCE）**。符合 OAuth 2.1 / OWASP 對 SPA 的建議做法、亦符合 ISO 27001（非強制 BFF）。條件：access token 存記憶體（非 localStorage）、短效、配 XSS 防護（CSP）。BFF 列為日後強化選項。 |
| Q2 | 登入失敗如何記 audit | ✅ **(b) 統一**：Logto 登入事件（含失敗）經 webhook 推送、寫入 AuthService 既有 `AuditLog`，稽核**只查一個地方**。 |
| Q3 | `passwordHash` 欄位 | ✅ **(b) 移除**（AuthService 現有資料皆測試資料、可清除重建）。 |

### 補充：最高管理者（Super Admin）
- 重置測試資料時，**種一個受保護的 Super Admin**（owner）：指派 `SUPER_ADMIN` 角色、`isSystem=true`。
- 保護規則（避免自我鎖死）：不可刪除/停用「最後一個 Super Admin」；不可移除最後持有者的 `SUPER_ADMIN` 角色。
- 權限：`SUPER_ADMIN` 視為擁有所有權限（bypass 檢查）。
- 帳號用**真實可控的公司 email**（讓 Logto 登入、之後 2FA/重設可運作），並文件化 break-glass 程序。
- 完整的「保護規則強制」屬日後帳號管理功能；本階段先做**種子 + isSystem 保護標記**。

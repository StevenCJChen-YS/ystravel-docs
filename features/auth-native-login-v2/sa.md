# SA: AuthService Native 登入 v2（Phase 1a）

| 項目 | 內容 |
|---|---|
| 對應 PRD | ./prd.md |
| 對應地基 | ../auth-foundation/architecture-decisions.md §0.6 |
| 狀態 | 草稿 |
| 建立日期 | 2026-07-09 |

## 0. 白話摘要

本功能讓員工「登入」這件事**完全由 AuthService 自己負責**：驗證帳密、簽發／輪替 token、回傳角色與權限、寫稽核，全在一個服務內。Logto 不再參與（降為歷史備案）。AuthService 同時回答「你是誰」與「你能做什麼」。**最關鍵的控制點是「白名單」：只有預先建立且 ACTIVE 的員工能進，帳號不存在／停用／密碼錯一律拒絕（fail-closed，預設拒絕），且對外訊息統一模糊化。** 相較舊 native 設計，本版本把 token 簽章從 HS256 共用密鑰升級為 **RS256／JWKS 非對稱金鑰**，並明確把幾個尚未完成的高風險項（refresh 查找效能、登入鎖定、外洩密碼、Google、2FA、SLO）列為後續必修，不當作已完成。

## 1. 系統邊界

| 系統 | 本功能是否異動 | 角色 |
|---|---|---|
| Ystravel-AuthPortal（前端） | 是（回復 native 登入頁體驗） | 帳密登入 UI、存 token、呼叫 `/auth/*`、導轉 `/my-apps` |
| Ystravel-AuthService（後端） | 是（native flow + RS256 升級） | 驗帳密、簽／驗／輪替 token、組角色權限、寫 audit、出 JWKS |
| Ystravel-LocalDocker | 是（移除 Logto/Postgres 依賴，回 MySQL 單一 DB） | 本地跑 MySQL |
| Logto | 否（退場） | 降為 `auth-logto-login` 歷史實驗，不在本主線 |
| Ystravel-CRM-*（前端／後端） | 否（本階段不接） | 之後 Phase 2 |

已有架構文件：`../auth-foundation/architecture-decisions.md`（§0.6 native auth v2、§4 授權模型），本文件不重述，僅聚焦登入這條路徑。

## 2. 角色與權限

| 角色 | 在本功能可以做什麼 |
|---|---|
| 內部員工 | 用自己的公司帳號密碼登入 |
| IT / 管理員 | 在 AuthService 預先建立員工帳號、指派角色（沿用既有 users/roles 管理） |
| Super Admin | `steven_chen@ystravel.com.tw`，`AUTH_SUPER_ADMIN`，視為擁有所有權限 |

本功能**不新增業務 permission code**（它是登入基礎建設）；它讓「登入後能正確取得既有的權限」這件事成立。

## 3. 操作流程（正常路徑）

```
1. 員工開 AuthPortal 登入頁
2. 輸入公司帳號（username 或 email）+ 密碼，送 POST /auth/login
3. AuthService 查 AuthUser（username OR email）
4. 檢查：帳號存在？有 passwordHash？status=ACTIVE？
5. bcrypt.compare 驗密碼
   - 通過 → 組角色/權限（buildAuthUser）
            → 簽 RS256 access token（15m）+ 產 refresh token（存雜湊，7d）
            → 記 lastLoginAt + LOGIN audit
            → 回 { accessToken, refreshToken, user }
   - 任一不符 → 拒絕、統一訊息「Invalid username or password.」
6. 前端拿 access token 呼叫受保護 API；JwtAuthGuard 用 public key 驗 RS256 簽章
7. access token 快到期 → POST /auth/refresh（輪替：撤舊發新 + TOKEN_REFRESH audit）
8. 登出 → POST /auth/logout（撤銷 refresh token + LOGOUT audit）
```

## 4. 狀態轉換

- **AuthUser 帳號狀態**：ACTIVE ↔ DISABLED（由既有管理功能控制）；DISABLED 一律拒絕登入。
- **RefreshToken 生命週期**：`created`（登入／refresh 時建）→ `active`（未撤銷且未過期）→ `revoked`（refresh 輪替、logout、或帳號停用時撤銷）／`expired`（超過 7 天）。
- **未來 external identity（`logtoSub` nullable）**：目前不使用；保留給日後 Google／外部身分綁定。

## 5. 例外流程

| 情況 | 系統行為 |
|---|---|
| 帳號不存在 | 拒絕、統一訊息「帳號或密碼錯誤」 |
| 帳號存在但無 `passwordHash` | 拒絕（視同不可用帳密） |
| 帳號 status=DISABLED | 拒絕 |
| 密碼錯誤 | 拒絕、統一訊息（不透露是帳號還密碼錯） |
| refresh token 無效／過期／已撤銷 | 401 Invalid refresh token |
| refresh 時帳號已停用 | 撤銷該 refresh token 並拒絕 |
| 被指派的角色 isActive=false | 該角色權限不計入（沿用既有邏輯） |
| override 過期（expiresAt ≤ now） | 該 override 不計入 |
| 帳號登入後才被停用 | 最多 15 分鐘（access token 效期）後自然失效；本階段不即時撤銷 access token |
| 偽造 HS256 / none 演算法的 token | guard 限定 `algorithms:['RS256']`，一律 401 |

## 6. 資料流與敏感資料

- **密碼**：以 bcrypt 雜湊存於 AuthService `AuthUser.passwordHash`，全程不落地明文，前端不接觸雜湊。
- **refresh token**：明文只回前端一次，DB 只存 bcrypt 雜湊（`RefreshToken.tokenHash`）。
- **access token**：**RS256 簽章**；AuthService 持 private key，下游／自身用 public key（JWKS）驗；解掉 HS256 共用密鑰弱點。
- **身分／權限模型**：Internal。
- **登入事件**（LOGIN／TOKEN_REFRESH／LOGOUT）→ 寫既有 `AuditLog`。（登入**失敗**目前未寫，列為需補，見 SD §9。）

## 7. 與外部系統的互動

- 本功能**不對接任何外部 IdP**。登入完全自足於 AuthService。
- 下游系統（未來 CRM 等）只需取得 AuthService 的 **public JWKS**（`GET /.well-known/jwks.json`）即可離線驗 access token，無需共用密鑰、無需連回 AuthService。

## 8. 風險點與人工審核點（Steven 重點看這節）

| 風險點 | 為什麼要注意 | 建議控制 |
|---|---|---|
| **白名單必須 fail-closed** | 若驗證失敗時「放行」而非「拒絕」，等於門戶大開 | 預設拒絕；只有明確查到 ACTIVE 且密碼正確才放行（現況已如此） |
| **refresh token 全表掃描 bcrypt** | `findActiveRefreshToken` 撈所有 active token 逐筆 compare，量大變慢＝「小樣本沒事、真實量出事」典型坑 | 改 selector + verifier hash、可索引；列為後續必修（SD §10 P1） |
| **登入失敗未寫 audit** | feature 場景要求登入失敗要留痕，現況只在成功時寫 | 在 login 失敗路徑補 LOGIN_DENIED／LOGIN_FAILED audit（SD §9） |
| **無登入鎖定** | 帳密登入沒有失敗計數／節流，易被暴力嘗試 | 依帳號／IP 失敗計數 + 暫時鎖定 + 告警（後續必修 SD §10 P2） |
| **密碼政策未完整落地** | 政策文件已定，程式端弱／外洩密碼檢查尚未做 | 密碼設定／重設流程納入檢查（後續必修 SD §10 P3） |
| **RS256 私鑰保管** | private key 外洩＝可偽造全站 token | private key 只在 AuthService、base64 進 env 不進 git、支援 kid rotation |
| **Super Admin 自我鎖死** | 停用／移除最後一個 super admin 會鎖死系統 | 種子 isSystem 保護標記；完整強制規則屬日後帳號管理功能 |

## 9. 待決策事項（交給 SD 細化）

- refresh token 查找改 selector/verifier 的 schema 與 migration 細節、是否加 reuse detection。
- 登入失敗 audit 的 action 命名與寫入點。
- 登入鎖定計數的儲存層（DB vs cache）與門檻（沿用密碼政策「連錯 5 次鎖 15 分」）。
- Google／external subject 綁定沿用 `logtoSub` 欄位或改名。

# Example Mapping: auth-native-login-v2

> Step 2 產物。AI 草擬規則與例子；Steven 確認／刪減規則、回答 ❓ 問題。確認後轉 Gherkin（Step 3）。
> BDD 場景已落在 `Ystravel-AuthService/src/auth/login-access.feature`（native 版）。

## Rule 1：只有「預先建立且啟用」的帳號能登入（白名單以 AuthService 為準）

- ✅ 已建立、狀態 ACTIVE 的員工，用正確帳號 + 密碼 → 登入成功
- ❌ 從沒建過的帳號 → 拒絕
- ❌ 已建立但狀態 DISABLED（停用）→ 拒絕
- ❌ 正確帳號、錯誤密碼 → 拒絕
- ❌ 帳號存在但 `passwordHash` 為空（例如尚未設密碼的外部身分帳號）→ 拒絕

## Rule 2：登入成功簽發 access + refresh token，且 access token 走 RS256

- ✅ 登入成功 → 回傳 access token（`alg=RS256`、帶 `kid`）+ refresh token
- ✅ access token 可用 `GET /.well-known/jwks.json` 的 public key 驗證
- ❌ 偽造的 `HS256` token → guard 拒絕（限定 algorithms 只接受 RS256）
- ✅ refresh token 只存**雜湊**，明文只回給前端一次

## Rule 3：登入後的身分與權限來自 AuthService

- ✅ 登入後拿到的角色／權限 = 該員工在 AuthService 的角色／權限（`buildAuthUser`）
- ✅ 畫面顯示正確的 displayName
- ✅ 邊界：被指派的某角色 `isActive=false` → 該角色權限不計入
- ✅ 邊界：`UserPermissionOverride` 的 ALLOW／DENY 生效，且過期（`expiresAt`）的 override 不計入

## Rule 4：token 輪替與登出

- ✅ 用有效 refresh token 換發 → 舊 refresh token 撤銷、發新的（rotation）+ 記 `TOKEN_REFRESH`
- ✅ refresh 時帳號已被停用 → 撤銷該 token 並拒絕
- ✅ logout 帶 refresh token → 撤銷該 token + 記 `LOGOUT`

## Rule 5：登入相關事件要寫 audit log

- ✅ 登入成功 → 寫 audit（action=LOGIN）
- ✅ refresh 成功 → 寫 audit（action=TOKEN_REFRESH）
- ✅ logout → 寫 audit（action=LOGOUT）
- ❓ 登入**失敗** → 也要寫 audit（Steven 於 logto 版已確認「要」；**native 現況尚未實作**，需補）

---

## ❓ 待 Steven 確認

1. **登入被擋的訊息**：沿用 logto 版共識 → 統一模糊講「帳號或密碼錯誤」，不讓人試出某帳號是否存在。（預設沿用，除非你要改）
2. **登入失敗寫 audit**：logto 版你答「要」。native 目前失敗路徑**沒寫** audit → 確認要補嗎？（見 SD §9）
3. **停用帳號的既有登入狀態**：沿用「最多 15 分鐘後 access token 過期自然失效，本階段不做即時撤銷」？

## 交給 SA/SD 處理的技術問題（先記著，你不用答）

- refresh token 全表掃描逐筆 bcrypt → 改 selector/verifier 的具體 schema 與 migration。
- 登入鎖定計數放哪一層、用什麼儲存（DB vs cache）。
- Google 首次登入的 external subject → AuthUser 綁定（沿用 logto 版比對思路，改走 native）。

## 刻意不在這次涵蓋（對齊 PRD Out of Scope）

- 登入鎖定、忘記密碼、外洩密碼檢查、2FA、Google 後端、Single Logout、CRM 前端接入。

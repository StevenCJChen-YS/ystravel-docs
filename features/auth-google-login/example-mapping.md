# Example Mapping: auth-google-login

> Step 2 產物。依 Steven 2026-07-09 已拍板的 1A / 2B / 3A 草擬。

## Rule 1：只有「預先建立且啟用」的本地帳號可以完成 Google 登入

- ✅ 已建立、狀態 ACTIVE 的員工，用公司 Google 帳號登入 → 成功
- ❌ 從沒建過的 Google email → 拒絕
- ❌ 已建立但狀態 DISABLED → 拒絕
- ❌ Google 驗證成功，但綁到另一個已存在的外部身分 subject → 拒絕

## Rule 2：Google 外部身分要用乾淨的新模型綁定，不再沿用 `logtoSub`

- ✅ 第一次成功登入時，若本地帳號尚未綁定 Google → 建立 `AuthIdentity(provider=GOOGLE)` 綁定
- ✅ 之後同一個 Google subject 再登入 → 直接對應同一個本地帳號
- ❌ 若同一個 Google subject 想綁到不同本地帳號 → 拒絕

## Rule 3：Google callback 不可以把 access token / refresh token 帶進 URL

- ✅ callback 回到前端時，只帶一次性 `authCode`
- ✅ 前端再用 `authCode` 跟 AuthService 換 native access token / refresh token
- ❌ 重複使用同一個 `authCode` → 拒絕
- ❌ 過期的 `authCode` → 拒絕

## Rule 4：網域限制與開發例外要分環境

- ✅ 生產環境：公司網域帳號 + 白名單帳號 → 成功
- ❌ 生產環境：私人 gmail，即使流程跑到 callback，也應拒絕
- ✅ 開發環境：若 email 在額外測試清單中，且本地帳號已預建 → 可成功

## Rule 5：登入結果與權限仍以 AuthService 為準

- ✅ Google 登入成功後，畫面顯示的是本地 `displayName`
- ✅ 角色 / 權限來自 AuthService 的 role / permission / override
- ✅ 已停用的角色不計入權限（沿用既有 `buildAuthUser` 邏輯）

## Rule 6：登入相關事件要寫 audit log

- ✅ Google 登入成功並完成 token 交換 → 寫 `LOGIN`
- ✅ 白名單不符 / 帳號停用 / 網域不符 / Google 驗證失敗 → 寫 `LOGIN_DENIED`
- ✅ 交換碼重放或過期 → 寫 `LOGIN_DENIED`

---

## Steven 已回答（2026-07-09）

1. ✅ Google OAuth 由 **AuthService 後端**處理（不是前端直連 Google）
2. ✅ **不要再沿用 `logtoSub`**，避免歷史實驗殘留讓後面越來越亂
3. ✅ callback 採 **一次性登入交換碼**；這一輪不混入 HttpOnly cookie 改造
4. ✅ 資安要求要對齊 ISO 27001，不能把 token 直接放 URL

## 仍要在實作中遵守的技術規則

- callback 參數中的 `redirectUri` 必須只接受站內相對路徑，避免 open redirect
- `authCode` 必須短效、單次使用、驗證失敗要記錄 audit
- 開發環境允許的私人 gmail 名單不可直接沿用到 production

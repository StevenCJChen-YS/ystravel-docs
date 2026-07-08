# Example Mapping: auth-logto-login

> Step 2 產物。AI 草擬規則與例子；Steven 確認/刪減規則、回答 ❓ 問題。確認後才轉 Gherkin（Step 3）。

## Rule 1：只有「預先建立且啟用」的帳號能登入（白名單以 AuthService 為準）

- ✅ 已建立、狀態 ACTIVE 的員工，用正確公司 email + 密碼 → 登入成功
- ❌ 從沒建過的 email → 拒絕
- ❌ 已建立但狀態 DISABLED（停用）→ 拒絕
- ❌ 正確 email、錯誤密碼 → 拒絕

## Rule 2：Google 登入須對應到已建立帳號，且（生產）限公司網域

- ✅ 已建立員工，用**公司 Google 帳號**登入 → 成功
- ❌ 沒建過的公司 Google 帳號（網域對，但白名單沒有）→ 拒絕
- ❌ 外部 gmail（生產環境）→ 拒絕
- ✅ 開發環境、在「測試 email 清單」內、且已建帳號的私人 gmail → 成功

## Rule 3：登入成功後，身分與權限來自 AuthService（不是 Logto）

- ✅ 登入後拿到的角色／權限 = 該員工在 AuthService 的角色／權限
- ✅ 畫面顯示正確的 displayName
- ✅ 邊界：員工被指派的某角色 isActive=false → 該角色的權限不計入（沿用現有 `buildAuthUser` 邏輯）

## Rule 4：登入相關事件要寫 audit log

- ✅ 登入成功 → 寫 audit（action=LOGIN）
- ✅ 登入**失敗** → 也寫 audit（Steven 2026-07-08 確認）

---

## Steven 已回答（2026-07-08）

1. **登入被擋的訊息**：✅ 統一模糊講「帳號或密碼錯誤」，不讓人試出某 email 是否存在。
2. **登入失敗寫 audit**：✅ 要。
3. **停用帳號的既有登入狀態**：✅ 接受最多 15 分鐘後（access token 過期）自然失效，本階段不做即時撤銷。

## 交給 SA/SD 處理的技術問題（先記著，你不用答）

- Logto 有身分、AuthService 有 AuthUser——首次 Google 登入如何用 email 把 Logto `sub` 綁到既有 AuthUser？比對到 0 筆或多筆怎麼辦？
- 「測試 email 清單」與「公司網域限制」放在哪一層（Logto 設定 vs AuthService 檢查）。
- 白名單判定放哪：確認以 AuthService 的 AuthUser 是否存在且 ACTIVE 為準。

## 刻意不在這次涵蓋（對齊 PRD Out of Scope）

- 密碼錯誤次數鎖定、忘記密碼、2FA、Single Logout。

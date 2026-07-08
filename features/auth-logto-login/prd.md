# PRD: Logto 登入整合（Phase 1a）

| 項目 | 內容 |
|---|---|
| 功能 slug | auth-logto-login |
| 狀態 | 已核准（2026-07-08） |
| 負責決策 | Steven |
| 建立日期 | 2026-07-08 |
| 最後更新 | 2026-07-08 |
| 對應地基 | `docs/features/auth-foundation/architecture-decisions.md`（混合架構 C） |

## 1. 一句話摘要

讓員工透過 Logto（公司帳密 ＋ Google 公司帳號）登入 AuthPortal，AuthService 信任 Logto 的身分並回傳員工原本的角色／權限，打通 SSO 登入的第一條完整路徑。

## 2. 背景與動機

- 目前 AuthService 是自簽 JWT（HS256 共用密鑰），多系統 SSO 下有「任一系統漏密鑰就能偽造全站 token」的弱點；且 Google 登入、2FA、忘記密碼等高風險功能尚未實作。
- 依地基決策改採**混合架構 C**：登入交給成熟的 Logto（用 OIDC 非對稱金鑰／JWKS），權限保留在現有 AuthService。
- 這是把 auth-foundation 真正落地的**第一步**：先證明「登入 → 拿到身分 → 權限還在」整條走得通。

## 3. 目標使用者

- **內部員工**：在 AuthPortal 登入（帳密或 Google）。
- **IT／管理員**：在 Logto 與 AuthService 預先建立員工帳號、指派角色。
- 本階段以**少量測試帳號**驗證，不做全員遷移。

## 4. User Story

作為**內部員工**，
我希望**用公司帳密或 Google 公司帳號登入一次**，
以便**進入系統、且系統知道我是誰、我原本的角色權限都還在**。

## 5. Acceptance Criteria（驗收標準，Steven 可自己操作驗證）

- [ ] 本地環境能啟動 Logto，並能打開 Logto 的管理後台
- [ ] 在 AuthPortal 登入頁，用測試員工的**公司 email + 密碼**登入 → 成功，進到 **`/my-apps`**（我的應用頁）
- [ ] 在 AuthPortal 登入頁按**「Google 登入」**→ 選公司帳號 → 成功登入
- [ ] 登入後，畫面能看出「我是誰」（顯示 displayName），且我原本在 AuthService 的**角色／權限仍然有效**
- [ ] 🔒 **白名單（主要防線）**：一個**沒有被預先建立**的帳號（不在名單內）用 Google 登入，會被**拒絕**、進不來
- [ ] 🔒 **公司網域（分環境）**：生產環境只接受公司網域的 Google 帳號；開發環境可透過「額外允許測試 email 清單」放行指定私人 gmail 測試帳號

## 6. UI / UX 重點

- **沿用現有 `LoginPage.vue`**（畫面已完成，含帳密欄位、Google 登入按鈕、記住我），這次主要是把行為接到 Logto，不重畫。
- 登入後落地頁：沿用現有 **`/my-apps`**（`AccountAppsPage.vue`）。（註：`pages/auth/AppsPage.vue` 為未使用的孤兒死檔，另案清理，不在本 feature。）
- 四種狀態：
  - Loading：登入送出中（現有已有 `loading`）
  - 錯誤：登入失敗顯示 toast（現有已有）；補「帳號未授權／非公司網域」的清楚訊息
  - 空／正常：登入成功導轉
- **回饋方式（訊息呈現規則，全站通用）**：
  - **前端就能判斷的錯**（email 格式、必填未填）→ 顯示在**該 input 欄位下方**（即時、不打斷）
  - **需送後端才知道的結果**（登入成功/失敗）→ 用 **toast**：登入成功 → 成功 toast 後導轉到 `/my-apps`；登入失敗 → 「帳號或密碼錯誤」toast
  - 沿用現有 `LoginPage.vue` 的 `useToast`
- 顏色沿用 design-system；AuthPortal 主色 = Blue（見 `docs/design-system/foundation.md`）。

## 7. 資料分級與風險（必填 — 依 SECURITY_AND_ISO27001_BASELINE.md §6.1）

| 欄位 | 內容 |
|---|---|
| 涉及的資料類型 | 員工 email、密碼、displayName、角色/權限、Google 帳號識別（sub） |
| 資料分級 | 密碼與身分識別＝**Restricted**（存於 Logto）；角色/權限模型＝Internal |
| 誰可以看 | 員工只能登入自己的帳號；管理員可在後台看帳號清單 |
| 誰可以匯出 | 無匯出功能 |
| 誰可以編輯 | IT／管理員建立帳號、指派角色；一般員工不可 |
| 是否涉及 AI | 否 |
| 風險摘要 | 登入／身分是 SSO 核心，做錯＝**未授權者可進入所有系統、客戶 PII 外洩**。緩解：關閉自助註冊（只有預建帳號能進）、限公司網域、token 走 Logto JWKS 非對稱金鑰、登入事件寫 audit。 |

## 8. 不做的事（Out of Scope）

- **2FA / TOTP**（下一刀；管理員強制、一般員工自願的規則已在地基定案）
- **忘記密碼 / 密碼重設流程**
- **Single Logout（一次登出全系統）**
- **把現有既有帳號批次遷移到 Logto**（本階段只用少量測試帳號手動建立）
- **CRM 前端接入登入**（屬 Phase 2）
- **登入失敗鎖定、擋外洩密碼**（由 Logto 政策提供，之後再開啟設定）

## 9. 待釐清問題（交給 SA/SD 處理，先記著）

- [ ] 現有 `AuthUser` 與 Logto 使用者怎麼對應？（用 email 比對既有帳號、並在 AuthUser 加 `logtoSub` 欄位）
- [ ] Logto 本地怎麼跑（`Ystravel-LocalDocker` 加一個 service）？正式環境之後再談。
- [ ] 首次 Google 登入成功後，如何把 Logto `sub` 綁到既有 AuthUser。

## 10. 決策紀錄

| 日期 | 決策 | 決定人 |
|---|---|---|
| 2026-07-08 | 採混合架構 C（Logto 管登入、AuthService 管權限） | Steven |
| 2026-07-08 | 第一版含 Google 登入（不延後） | Steven |
| 2026-07-08 | 第一版只用少量測試帳號，不做全員遷移 | Steven |
| 2026-07-08 | 登入落地頁＝`/my-apps`（AccountAppsPage） | Steven |
| 2026-07-08 | 白名單為主要防線；公司網域限制分環境＋測試 email 清單（兼顧安全與私人 gmail 測試） | Steven |

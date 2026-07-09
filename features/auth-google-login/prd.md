# PRD: Google 登入整合（Phase 1d）

| 項目 | 內容 |
|---|---|
| 功能 slug | auth-google-login |
| 狀態 | 已核准（2026-07-09） |
| 負責決策 | Steven |
| 建立日期 | 2026-07-09 |
| 最後更新 | 2026-07-09 |
| 對應地基 | `docs/features/auth-foundation/architecture-decisions.md` §0.6、§10 Phase 1d |

## 1. 一句話摘要

讓內部員工可以從 AuthPortal 使用 Google 公司帳號登入，由 AuthService 自己處理 Google OAuth、白名單驗證、外部身分綁定與 native token 簽發，並透過**一次性登入交換碼**把登入結果安全帶回前端。

## 2. 背景與動機

- `auth-native-login-v2` 的 P4 安全債明確寫著「前端已有 Google 按鈕、後端無端點」，目前這顆按鈕點下去不會完成登入。
- Steven 已拍板主線是 **native auth v2**，所以 Google 登入也要落在 AuthService，而不是再引回外部 IdP（Identity Provider，身分提供者；以下以 IdP 代稱）架構。
- 前一次 Logto 實驗留下 `logtoSub` 之類命名，但主線已回到 native；這次要把 Google 身分模型整理乾淨，避免後續文件與程式碼越疊越亂。
- 這次功能又直接碰到登入、白名單、權限與 token 流程，必須符合資安與 ISO 27001（International Organization for Standardization 27001，資訊安全管理標準；以下以 ISO 27001 代稱）要求，不能只求「能登入就好」。

## 3. 目標使用者

- **內部員工**：希望用已登入 Google 的公司帳號快速進入 AuthPortal。
- **IT / 管理員**：預先建立本地帳號、維護帳號啟用狀態、必要時追查 audit log。
- 本階段仍是**內部員工登入**，不是對外客戶登入。

## 4. User Story

作為內部員工，  
我希望從 AuthPortal 按「Google 登入」後可以直接用公司 Google 帳號進入系統，  
以便在不額外記住另一組密碼的情況下，安全地取得我原本在 AuthService 的角色與權限。

## 5. Acceptance Criteria（驗收標準）

- [ ] 在 AuthPortal 登入頁按「Google 登入」後，會被導向 Google 登入流程，成功後回到 AuthPortal 並進入 `/my-apps`
- [ ] 只有 **AuthService 中已預先建立且狀態為 ACTIVE** 的帳號可以完成 Google 登入
- [ ] 第一次 Google 登入成功後，系統會建立「Google 外部身分綁定」，之後再次登入會直接對應同一個本地帳號
- [ ] 登入成功後，回來的 `displayName`、角色與權限仍以 AuthService 為準，不由 Google 決定
- [ ] Google callback 回前端的 URL **不可以帶 access token 或 refresh token**
- [ ] 後端回前端只帶**一次性、短效、用過即失效**的登入交換碼；前端再用它向 AuthService 換 native access token / refresh token
- [ ] 不在白名單、帳號停用、或不符合網域規則的 Google 帳號，都會被拒絕，且前端只顯示不洩漏帳號存在性的模糊錯誤訊息
- [ ] 成功登入、拒絕登入、交換碼重放/失效等關鍵事件都會留下 audit log

## 6. UI / UX 重點

- **沿用既有 `LoginPage.vue` 設計**，不重做整個登入頁。
- Google 按鈕仍留在現有位置，但行為改為走 AuthService 的 Google OAuth 流程。
- 新增一個前端 `CallbackPage`（回跳頁）：
  - 顯示「登入處理中」
  - 拿 URL 上的一次性 `authCode` 去跟 AuthService 換 native token
  - 成功後導回原本 `redirectUri`
  - 失敗則導回 `/login` 並顯示 toast
- 錯誤訊息規則：
  - 帳號不存在 / 帳號停用 / 網域不符 / 白名單未建立：前端一律顯示模糊登入失敗訊息
  - 交換碼過期 / 被重放：前端顯示「登入已失效，請重新登入」

## 7. 資料分級與風險（必填 — 依 SECURITY_AND_ISO27001_BASELINE.md §6.1）

| 欄位 | 內容 |
|---|---|
| 涉及的資料類型 | 員工 email、Google subject、displayName、角色/權限、access token、refresh token |
| 資料分級 | Google subject / token / 員工身分識別＝**Restricted**；角色/權限模型＝Internal |
| 誰可以看 | 員工只能登入自己的帳號；管理員可透過既有後台看帳號與 audit |
| 誰可以匯出 | 本功能無匯出 |
| 誰可以編輯 | IT / 管理員可預先建立或停用本地帳號；Google 身分綁定由系統自動建立 |
| 是否涉及 AI | 否 |
| 若涉及 AI,AI 會接觸哪些欄位 | 不適用 |
| 風險摘要 | 若 Google 身分綁錯人、白名單判斷失效、或 token 出現在 URL/日誌中，可能造成未授權存取與帳號冒用。緩解：白名單 fail-closed、Google subject 唯一綁定、state/nonce 驗證、一次性短效 authCode、token 不進 URL、關鍵事件寫 audit。 |

## 8. 不做的事（Out of Scope）

- **不在這輪把整站 session 改成 HttpOnly cookie**
- **不在這輪把原本帳密登入一起改掉**（仍沿用現有 native token 模型）
- **不做自動建帳 / Just-In-Time Provisioning**（仍以預先建帳為主）
- **不做 2FA / TOTP**
- **不做忘記密碼 / 自助重設密碼**
- **不做 Single Logout**
- **不做多個外部 provider 一次上線**（本輪只做 Google）

## 9. 待釐清問題

- [ ] 生產環境的 Google OAuth client / callback URI 由 Steven 實際建立後補進 `.env`
- [ ] 開發環境若要允許私人 gmail 測試，需要列出額外允許的測試 email 清單
- [ ] 上線前 release checklist 要增加一條：再次確認 production 的 Google 網域限制已開啟

## 10. 決策紀錄

| 日期 | 決策 | 決定人 |
|---|---|---|
| 2026-07-09 | Google 登入由 **AuthService 後端**處理 OAuth 與 callback（1A） | Steven |
| 2026-07-09 | **不要再沿用 `logtoSub` 命名**；改做獨立外部身分模型（2B） | Steven |
| 2026-07-09 | Google callback 回前端只帶**一次性登入交換碼**，不帶 token（3A） | Steven |
| 2026-07-09 | 本輪不把全站 session 一次改成 HttpOnly cookie，避免混成兩套模型 | Steven |

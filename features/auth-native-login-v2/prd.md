# PRD: AuthService Native 登入 v2（Phase 1a）

| 項目 | 內容 |
|---|---|
| 功能 slug | auth-native-login-v2 |
| 狀態 | 已核准（2026-07-08） |
| 負責決策 | Steven |
| 建立日期 | 2026-07-09 |
| 最後更新 | 2026-07-09 |
| 對應地基 | `docs/features/auth-foundation/architecture-decisions.md` §0.6 |
| 前身實驗 | `docs/features/auth-logto-login/`（Logto 混合架構，已降為備案） |

## 1. 一句話摘要

讓員工用公司帳密登入 AuthPortal，由 **AuthService 自建**完成身分驗證、簽發／輪替 token、回傳角色與權限並記錄稽核，成為整個 SSO 體系「登入 → 拿到身分 → 權限生效」的第一條完整路徑。

## 2. 背景與動機

- 團隊曾評估「混合架構 C：登入交給自架 Logto」，並實裝跑通（見 `auth-logto-login`）。Steven 實際操作後，決定**改回 AuthService native 主導登入**：登入體驗與整體方向更貼近需求，且最危險的部分（Google／2FA／密碼流程）當時尚未寫進 Logto，換回來不算浪費成品。
- 改回 native **不是「復原舊碼就結束」**。原 native 設計有幾個具體缺點必須修：HS256 共用密鑰、多系統 secret 外洩風險、`findActiveRefreshToken` 全表掃描逐筆 bcrypt、Google／2FA／忘記密碼／登入鎖定／外洩密碼檢查尚未完成。本 PRD 把這些列為 native v2 的**後續必修**，而非可有可無。
- 這是把 auth-foundation 落地的第一步：先證明 native 這條路「登入 → 身分 → 權限還在」整條走得通，並把 token 簽章升級到非對稱金鑰。

## 3. 目標使用者

- **內部員工**：在 AuthPortal 用公司 email／username + 密碼登入。
- **IT／管理員**：在 AuthService 預先建立員工帳號、指派角色（沿用既有 users／roles 管理）。
- 本階段以**少量測試帳號**驗證，不做全員遷移。

## 4. User Story

作為**內部員工**，
我希望**用公司帳號密碼登入一次**，
以便**進入系統、且系統知道我是誰、我原本的角色權限都還在**。

## 5. Acceptance Criteria（驗收標準，Steven 可自己操作驗證）

- [x] AuthService 缺 RS256 private／public key 或 `JWT_ACCESS_KEY_ID` 時**啟動即失敗**（fail fast）；設定齊全才起得來
- [x] 在 AuthPortal 登入頁，用測試員工的**公司帳號 + 密碼**登入 → 成功，進到 **`/my-apps`**
- [x] 登入後，畫面能看出「我是誰」（顯示 displayName），且原本在 AuthService 的**角色／權限仍然有效**
- [x] 登入回傳 **access token（RS256）+ refresh token**；access token 可被 `GET /auth/me` 驗證通過
- [x] 🔒 access token 用 **RS256** 簽章；`GET /.well-known/jwks.json` 對外只提供 public key；guard 只接受 `RS256`、拒絕 `HS256`／`none`
- [ ] 🔒 **白名單（主要防線）**：不存在／已停用（DISABLED）的帳號 → 登入被**拒絕**、進不來
- [ ] 登入相關事件（成功／失敗／refresh／logout）寫入 `AuditLog`

> 註：已勾選項為第一刀 + 第二刀（RS256／JWKS）已完成並驗證（build/lint/jest 6 tests/live smoke 皆綠）。未勾選項屬本階段收尾與後續必修，見 §8。

## 6. UI / UX 重點

- **沿用原本 `LoginPage.vue`**（Email + 密碼、記住我、忘記密碼連結、Google 登入按鈕位置）。native v2 回復此頁原始體驗，Google 按鈕**位置保留但後端尚未接**（見 Out of Scope）。
- 登入後落地頁：沿用現有 **`/my-apps`**（`AccountAppsPage.vue`）。
- 四種狀態：Loading（登入送出中）／錯誤（登入失敗 toast）／空／正常（成功導轉）。
- **回饋方式（全站通用）**：
  - **前端就能判斷的錯**（email 格式、必填未填）→ 顯示在該 input 欄位下方（即時、不打斷）。
  - **需送後端才知道的結果**（登入成功／失敗）→ 用 **toast**：成功 → toast 後導轉 `/my-apps`；失敗 → 統一「帳號或密碼錯誤」。
- 顏色沿用 design-system；AuthPortal 主色 = Blue（見 `docs/design-system/foundation.md`）。

## 7. 資料分級與風險（必填 — 依 SECURITY_AND_ISO27001_BASELINE.md §6.1）

| 欄位 | 內容 |
|---|---|
| 涉及的資料類型 | 員工帳號、密碼（雜湊）、displayName、角色／權限、refresh token（雜湊） |
| 資料分級 | 密碼雜湊與 refresh token＝**Restricted**（存於 AuthService DB）；角色／權限模型＝Internal |
| 誰可以看 | 員工只能登入自己的帳號；管理員可在後台看帳號清單 |
| 誰可以匯出 | 無匯出功能 |
| 誰可以編輯 | IT／管理員建立帳號、指派角色；一般員工不可 |
| 是否涉及 AI | 否 |
| 風險摘要 | 登入／身分是 SSO 核心，做錯＝**未授權者可進入所有系統、客戶 PII 外洩**。緩解：白名單 fail-closed（只有 ACTIVE 帳號能進）、密碼 bcrypt 雜湊不落地明文、access token 走 **RS256／JWKS 非對稱金鑰**（下游只拿 public key，解掉 HS256 共用密鑰弱點）、登入事件寫 audit。**尚未完成的高風險項**（登入鎖定、外洩密碼、2FA、Google）列為後續必修，見 §8。 |

## 8. 不做的事（Out of Scope）＋ 後續必修

本階段（Phase 1a）**只做「native flow 可用 + token 簽章升級 RS256」**。以下**不在這一刀**，但已列為 native v2 的後續必修（優先序在 SD §10）：

- **Refresh token 查找優化**：現況 `findActiveRefreshToken` 全表掃描逐筆 bcrypt，資料量一大就慢。→ 改 selector + verifier hash、可索引、支援 reuse detection。
- **登入鎖定 / 防暴力**：依帳號／IP 失敗計數、暫時鎖定、告警。
- **密碼政策落地**：12 碼、弱密碼／外洩密碼檢查、重設流程（政策文件已定，程式未完整落地）。
- **Google OAuth 後端**：前端有按鈕，後端端點未實作。預先佈建 + 公司網域 + external subject linking（`logtoSub` 欄位暫留作未來 external identity link）。
- **2FA / TOTP**：管理員／高權限強制、一般員工自願（規則已在地基拍板）。
- **Single Logout（跨系統一次登出）**：`/auth/logout` 目前只撤銷 refresh token，access token 到期前仍短暫有效。需獨立設計 session／token version／revocation cache。
- **CRM 前端接入登入**（屬 Phase 2）。

## 9. 待釐清問題（交給 SA/SD 處理，先記著）

- [ ] 登入失敗是否即時寫 audit？（feature 場景已描述要寫，現況 `login()` 失敗路徑尚未寫 audit — 見 SD §9 gap）
- [ ] refresh token 輪替（rotation）reuse detection 的具體策略。
- [ ] Google 首次登入如何用 email／external subject 綁到既有 AuthUser（沿用 auth-logto-login 的比對思路）。

## 10. 決策紀錄

| 日期 | 決策 | 決定人 |
|---|---|---|
| 2026-07-08 | 主線改回 AuthService native auth v2（推翻 Logto 混合架構，Logto 降為備案） | Steven |
| 2026-07-08 | access token 從 HS256 升級 RS256／JWKS，下游只拿 public key | Steven |
| 2026-07-08 | Super Admin 種子帳號＝`steven_chen@ystravel.com.tw`（舊 `admin` 停用未刪） | Steven |
| 2026-07-08 | RS256 private key 屬本機測試 key，不進 git | Steven |
| 2026-07-09 | 若 native 設計有安全／效能／UX／架構問題，AI 須主動提出，不可只照做 | Steven |

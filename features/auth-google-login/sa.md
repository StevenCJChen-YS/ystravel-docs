# SA: Google 登入整合（Phase 1d）

| 項目 | 內容 |
|---|---|
| 對應 PRD | ./prd.md |
| 狀態 | 已確認（2026-07-09） |
| 建立日期 | 2026-07-09 |
| 對應地基 | ../auth-foundation/architecture-decisions.md §0.6、§10 Phase 1d |

## 0. 白話摘要

這份 SA 的核心是在分析：Google 已經驗過「你是誰」之後，AuthService 要怎麼**安全地**把這個外部身分對回本地員工帳號，再發出 native token。最大風險不是 Google 本身，而是「綁錯人、放錯人、或把 token 暴露在 URL」。因此本功能採三個關鍵控制：**白名單 fail-closed、獨立外部身分模型、一次性登入交換碼**。整體方向是讓 Google 只負責身分驗證，最終角色與權限仍由 AuthService 決定。

## 1. 系統邊界

| 系統 | 本功能是否異動 | 角色 |
|---|---|---|
| Ystravel-AuthPortal（前端） | 是 | 顯示登入頁、導向 Google、接 callback、拿一次性 `authCode` 換 native token |
| Ystravel-AuthService（後端） | 是 | 發起 Google OAuth、處理 callback、做白名單驗證、綁定外部身分、發 native token、寫 audit |
| Google OAuth / Google Identity | 是（外部互動） | 驗證 Google 帳號、回傳授權碼與 ID token |
| docs | 是 | 補 PRD / Example Mapping / SA / SD |
| Ystravel-LocalDocker | 否 | 本輪不需要新增本地身分服務 |

已有架構文件：`../auth-foundation/architecture-decisions.md` 已明訂 Phase 1d 是 Google 登入，且主線為 native auth v2；本文件不重述主線決策，只聚焦 Google 這條登入路徑。

## 2. 角色與權限

| 角色 | 可以做什麼 |
|---|---|
| 內部員工 | 在登入頁按 Google 登入，若帳號已預建且啟用，則完成登入 |
| IT / 管理員 | 在 AuthService 預先建立 / 啟用 / 停用員工帳號，必要時查 audit |

本功能本身**不新增 permission code**。Google 登入成功後，使用者可做的事仍由既有 role / permission 決定。

## 3. 操作流程（正常路徑）

1. 員工開啟 AuthPortal `LoginPage`，按下「Google 登入」。
2. 前端導向 `GET /api/auth/auth/google?app=<app>&redirectUri=<path>`。
3. AuthService 產生短效 `state` / `nonce`，保存本次登入上下文（app、redirectUri），再導向 Google OAuth 授權頁。
4. 員工在 Google 完成登入後，Google 把 browser 導回 AuthService 的 callback。
5. AuthService 驗證：
   - callback `state` 是否存在、未過期、未使用
   - Google token 是否有效
   - email 是否已驗證
   - 網域是否允許
   - 本地 `AuthUser` 是否存在且 `ACTIVE`
6. AuthService 用新的外部身分模型綁定 `provider=GOOGLE` 與 Google subject。
7. AuthService 產生**一次性登入交換碼**，只把這個碼帶回 AuthPortal callback 頁。
8. AuthPortal callback 頁呼叫 `POST /api/auth/auth/google/exchange` 交換 native access token / refresh token。
9. AuthService 驗證交換碼有效且未使用後，發 native token，前端存入既有 session store，導向原本 `redirectUri`（預設 `/my-apps`）。

## 4. 狀態轉換

### 4.1 外部身分綁定狀態

- **未綁定**：本地帳號存在，但尚未建立 Google 外部身分記錄
- **已綁定**：已有 `provider=GOOGLE` 的外部身分記錄

轉換規則：
- 第一次 Google 登入成功：未綁定 → 已綁定
- 後續再登入：已綁定 → 已綁定（不重建）

### 4.2 OAuth 啟動狀態

- **PENDING**：Google 流程已啟動，等待 callback
- **USED**：callback 已成功消耗該 `state`
- **EXPIRED**：超過有效時間，自動視為不可再用

### 4.3 一次性登入交換碼狀態

- **ISSUED**：已核發，等待前端換 token
- **USED**：已成功換過 token，不可再用
- **EXPIRED**：逾時失效

## 5. 例外流程

| 情況 | 系統行為 |
|---|---|
| Google 驗證成功，但本地查無對應 `AuthUser` | 拒絕、記 `LOGIN_DENIED`、回登入頁模糊錯誤 |
| 本地帳號存在，但狀態為 `DISABLED` | 拒絕、記 `LOGIN_DENIED` |
| callback `state` 不存在 / 已過期 / 已使用 | 拒絕、記 `LOGIN_DENIED` |
| Google email 未驗證 | 拒絕、記 `LOGIN_DENIED` |
| Google 網域不符合 production 規則 | 拒絕、記 `LOGIN_DENIED` |
| callback 帶入的 `redirectUri` 不是站內相對路徑 | 視為非法請求，拒絕 |
| 同一個 `authCode` 被重複使用 | 拒絕、記 `LOGIN_DENIED` |
| 本地帳號有多筆 email 相同（理論上不應發生） | 拒絕並告警，不允許猜測式綁定 |

## 6. 資料流與敏感資料

- **Google 端資料**：授權碼、ID token、Google subject、email、displayName
- **本地敏感資料**：`AuthUser`、角色/權限、access token、refresh token、audit log
- **Restricted 資料流**：
  - Google subject / token 只在 AuthService 處理
  - callback 回前端時不帶 token，只帶一次性 `authCode`
  - `authCode` 也屬敏感資料，需短效且一次性
- **資料離開受控環境風險**：
  - 若 token 放進 URL，會進瀏覽器歷史與代理伺服器日誌
  - 若 `redirectUri` 不收斂，會變成 open redirect

## 7. 與外部系統的互動

- **Google OAuth 2.0 / OpenID Connect**：
  - 授權端點：導向 Google 登入 / 同意畫面
  - Token 端點：後端用授權碼換 token
  - ID token 驗證：確認 audience、issuer、nonce、email_verified 等 claims

本功能不引入新的外部 IdP 平台，也不把 Google token 直接交給前端長期保存。

## 8. 風險點與人工審核點

| 風險點 | 為什麼要注意 | 建議控制 |
|---|---|---|
| **把 Google 身分綁錯本地帳號** | 綁錯人就等於越權登入 | 先查既有 `AuthIdentity`，首次綁定只允許對到唯一且 ACTIVE 的本地 email |
| **沿用 `logtoSub` 造成語意混亂** | 後面會讓設計、文件、migration 越看越亂 | 直接移除 `logtoSub`，改成獨立 `AuthIdentity` |
| **token 出現在 URL** | 會洩漏到歷史紀錄、代理日誌、分享連結 | callback 只帶一次性 `authCode`，不帶 access/refresh token |
| **state / authCode 被重放** | 可能讓攻擊者重複消耗同一次登入 | `state` 與 `authCode` 都要短效、單次使用、使用後失效 |
| **開發例外規則滲進 production** | 外部 gmail 可能意外被放行 | 生產環境只看允許網域；測試 email 清單只在非 production 啟用 |

## 9. 待決策事項

本輪核心設計已於 2026-07-09 拍板完成，**無新增架構待決策事項**。  
待準備的是環境與上線前資料：

- [ ] Steven 提供 production / local 的 Google OAuth client id、client secret、callback URI
- [ ] 補上 production 允許網域設定
- [ ] 若開發要允許私人 gmail 測試，補上測試 email 清單

# SA: Logto 登入整合（Phase 1a）

| 項目 | 內容 |
|---|---|
| 對應 PRD | ./prd.md |
| 對應地基 | ../auth-foundation/architecture-decisions.md（混合架構 C） |
| 狀態 | 草稿 |
| 建立日期 | 2026-07-08 |

## 0. 白話摘要

本功能將員工「登入」這件事，從目前 AuthService 自建的帳密驗證，改由成熟的身分服務 **Logto** 負責（帳密與 Google 登入皆是），而「使用者有哪些角色與權限」仍由既有 AuthService 掌管。Logto 負責回答「你是誰」，AuthService 負責回答「你能做什麼」。**最關鍵的控制點是「白名單」：只有 IT 預先在 AuthService 建立且啟用的員工才能進入，Logto 驗證通過但本地沒有對應帳號者一律拒絕（fail-closed，預設拒絕）。** 最大風險在「把 Logto 身分正確對應到本地既有員工帳號」的比對邏輯，若對錯人等同越權，故列為需重點審核項目。

## 1. 系統邊界

| 系統 | 本功能是否異動 | 角色 |
|---|---|---|
| Ystravel-AuthPortal（前端） | 是（登入改導向 Logto） | 登入 UI、接收登入結果、導轉 `/my-apps` |
| Logto（新增） | 是（新導入） | 認證：帳密、Google、發 OIDC token |
| Ystravel-AuthService（後端） | 是（改造） | 驗 Logto token、身分對應本地 AuthUser、回傳角色/權限、寫 audit |
| Ystravel-LocalDocker | 是（新增服務） | 本地跑 Logto |
| Ystravel-CRM-*（前端/後端） | 否（本階段不接） | 之後 Phase 2 |

已有架構文件：`../auth-foundation/architecture-decisions.md`（§0.5 保留 vs 汰換、§4 授權模型），本文件不重述，僅聚焦登入這條路徑。

## 2. 角色與權限

| 角色 | 在本功能可以做什麼 |
|---|---|
| 內部員工 | 用自己的公司帳密或 Google 帳號登入 |
| IT / 管理員 | 在 AuthService 預先建立員工帳號、指派角色（沿用既有 users/roles 管理）；在 Logto 端管理連接器設定 |

本功能**不新增業務 permission code**（它是登入基礎建設）；它讓「登入後能正確取得既有的權限」這件事成立。

## 3. 操作流程（正常路徑）

```
1. 員工開 AuthPortal 登入頁
2. 選擇：帳密 / Google 登入
3. AuthPortal 導向 Logto（OIDC Authorization Code + PKCE）
4. Logto 驗證身分（帳密或 Google）→ 回傳授權碼
5. 換發 token（access/refresh，OIDC 標準）
6. AuthService 用 Logto 的 JWKS(公鑰) 驗證 token
7. 取 token 內的身分（email / Logto sub）→ 對應本地 AuthUser
8. 檢查：AuthUser 存在且 status=ACTIVE？
   - 是 → 組出角色/權限（沿用既有 buildAuthUser 邏輯）→ 記 LOGIN → 導轉 /my-apps
   - 否 → 拒絕 → 記登入失敗 → 顯示「帳號或密碼錯誤」
```

## 4. 狀態轉換

- **AuthUser 與 Logto 身分的綁定狀態**：
  - 未綁定（本地有 AuthUser、但尚無 logtoSub）→ 首次 Logto 登入且 email 比對成功 → 寫入 logtoSub → 已綁定
  - 已綁定 → 之後以 logtoSub 直接對應（不再靠 email）
- **AuthUser 帳號狀態**：ACTIVE ↔ DISABLED（由既有管理功能控制）；DISABLED 一律拒絕登入。

## 5. 例外流程

| 情況 | 系統行為 |
|---|---|
| Logto 驗證過，但本地查無對應 AuthUser（不在白名單） | 拒絕、統一訊息「帳號或密碼錯誤」、記登入失敗 |
| AuthUser 存在但 status=DISABLED | 拒絕、記登入失敗 |
| email 比對到 0 筆 | 視為不在白名單，拒絕 |
| email 比對到多筆 | 不應發生（email 於 AuthUser 為唯一）；若發生則拒絕並記異常告警 |
| 被指派的角色 isActive=false | 該角色權限不計入（沿用既有邏輯） |
| 帳號登入後才被停用 | 接受最多 15 分鐘（access token 效期）後自然失效，本階段不即時撤銷 |
| 外部 gmail（生產環境） | 由 Logto 網域限制擋下；開發環境可用測試 email 清單放行 |

## 6. 資料流與敏感資料

- **密碼**：只存在 Logto，AuthService 與前端全程不接觸密碼（相較現況把 passwordHash 移出 AuthService，是安全性提升）。
- **身分識別**（email、Logto sub）：Restricted。
- **角色/權限模型**：Internal。
- **token**：走 Logto 非對稱金鑰（JWKS），各系統僅需公鑰驗證，無共用密鑰外洩風險。
- **登入事件**（含失敗）→ 寫既有 `AuditLog`。

## 7. 與外部系統的互動

- **Logto**：透過標準 OIDC 互動（授權碼流程、JWKS 公鑰端點、使用者資訊）。Google 登入由 Logto 的 Google connector 代為處理，AuthService 不直接對接 Google。

## 8. 風險點與人工審核點（Steven 重點看這節）

| 風險點 | 為什麼要注意 | 建議控制 |
|---|---|---|
| **身分對應（email→AuthUser）比對錯誤** | 對錯人＝該員工被當成另一個人，等同越權存取客戶 PII | 首次綁定以 email 精確比對＋唯一性保證；綁定後只認 logtoSub；比對到多筆一律拒絕並告警 |
| **白名單必須 fail-closed** | 若比對失敗時「放行」而非「拒絕」，等於門戶大開 | 預設拒絕；只有明確查到 ACTIVE 帳號才放行 |
| **公司網域限制的環境設定** | 開發放寬若不小心帶到生產＝外部帳號可進 | 設定分環境，生產強制公司網域；上線檢查清單納入此項 |
| **登入事件捕捉點**（混合架構下由誰記 audit） | 登入在 Logto 發生，AuthService 要能記到成功/失敗 | 於 SD 決定：Logto webhook 或 AuthService 首次換 token 時記錄 |

## 9. 待決策事項（交給 SD 細化）

- Logto 本地部署方式（docker compose 服務、埠號、資料庫）。
- 混合架構下「登入成功/失敗」事件如何確實寫入 audit（Logto webhook vs AuthService 端記錄）。
- token 交換由 AuthPortal（前端）或 AuthService（後端）執行（影響 refresh token 存放安全）。
- `AuthUser` schema 異動：拿掉/停用 `passwordHash`、新增 `logtoSub`（唯一）。

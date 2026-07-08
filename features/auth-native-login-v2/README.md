# Auth Native Login v2

| 項目 | 內容 |
|---|---|
| 狀態 | Phase 1a 實作中 |
| 決策日期 | 2026-07-08 |
| 對應地基 | `docs/features/auth-foundation/architecture-decisions.md` §0.6 |
| 相關 repo | `Ystravel-AuthService`、`Ystravel-AuthPortal`、`Ystravel-LocalDocker` |

## 1. 決策摘要

Steven 實際看完 Logto 流程與實裝後，決定改回原本方向：**AuthService 主導登入、token、refresh、權限與 audit；AuthPortal 使用原本登入頁體驗。**

這不是回到舊版就結束，而是 **native auth v2**：保留原本最貼近需求的集中權限模型，並把舊設計的安全債列成後續必修。

## 2. 第一階段範圍

本階段只做「回到 native flow 並可編譯」：

- AuthService 恢復 `/auth/login`、`/auth/refresh`、`/auth/logout`、`/auth/me`。
- AuthService schema 加回 `passwordHash` 與 `RefreshToken`。
- AuthService 啟動時驗證 `JWT_ACCESS_SECRET`，缺少時直接 fail fast，不等登入時才 500。
- AuthService 新增 `.env.example`，列出 native auth v2 必要環境變數。
- 初始 super admin seed 預設帳號改為 `steven_chen@ystravel.com.tw`，角色為 `AUTH_SUPER_ADMIN`。
- 保留 nullable `logtoSub`，暫作未來 external identity link，不在第一階段使用。
- AuthPortal 恢復原本登入頁：Email + 密碼、記住我、忘記密碼、Google 登入按鈕。
- LocalDocker 移除 Logto/Postgres 依賴。

## 3. 本階段不做

- 不在這一刀完成 Google OAuth。
- 不在這一刀完成 2FA / TOTP。
- 不在這一刀完成忘記密碼、登入鎖定、外洩密碼檢查。
- 不在這一刀完成 RS256/JWKS。
- 不在這一刀完成 Single Logout。

## 4. native 原設計需要修的地方

這些不是可有可無，是 native v2 後續必修：

| 風險 | 現況問題 | v2 方向 |
|---|---|---|
| Token 簽章 | access token 目前用 HS256 共用密鑰；多系統共用 secret 時，任一系統外洩 secret 就可能偽造全站 token | 升級 RS256/JWKS；AuthService 持 private key，各系統只拿 public key 驗簽 |
| Refresh token 查找 | `findActiveRefreshToken` 會掃描有效 token 後逐筆 bcrypt compare；資料量變大會慢 | token 拆 selector + verifier，selector 可索引，verifier hash 後比對；支援 reuse detection |
| 登入防暴力 | 舊版沒有完整登入鎖定、節流、告警 | 依帳號/IP 做失敗計數、暫時鎖定、audit + alert |
| 密碼政策 | 密碼政策文件已定，但程式未完整落地 | 12 碼、弱密碼/外洩密碼檢查、重設流程、不得明文記錄 |
| Google 登入 | 前端原本有 Google 按鈕，但後端端點尚未實作 | AuthService 對接 Google OAuth；預先佈建 + 公司網域 + external subject linking |
| 2FA | 管理員/高權限強制已拍板，但 native 尚未實作 | TOTP 優先；高權限角色強制，一般員工先自願 |
| SLO | `/auth/logout` 只能撤銷 refresh token；access token 到期前仍可能短暫有效 | 設計 session registry / token version / revocation cache，再談 Single Logout |
| Config validation | 舊版缺 `JWT_ACCESS_SECRET` 時登入才 500 | 啟動時 fail fast；後續集中整理 auth config schema |

## 5. 驗收條件

- AuthService build 通過。
- AuthPortal build 通過。
- Prisma schema 與 migration 能表示 native v2 目標狀態。
- 缺少 `JWT_ACCESS_SECRET` 時 AuthService 啟動即失敗；設定後 native login API 能成功回傳 access/refresh token。
- 文件清楚記錄：Logto 是歷史實驗，當前主線是 native auth v2。
- 若發現 native 設計偏離「AuthService 集中登入與權限」方向，必須提出，不默默實作。

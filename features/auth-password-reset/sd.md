# SD: 密碼重設與帳號啟用流程

| 項目 | 內容 |
|---|---|
| 對應 SA | ./sa.md |
| 狀態 | 草稿 |
| 建立日期 | 2026-07-09 |

## 0. 白話摘要

新增一張「密碼連結 token」表（重設與邀請共用，資料庫只存 hash）、一張寄信頻率表、一個寄信模組（nodemailer + SMTP 抽象，本機接 Mailpit 假信箱），與三支公開 API（申請重設、驗證連結、送出新密碼）＋兩支管理員 API（發重設連結、重發邀請）。帳號狀態多一個 `INVITED`（邀請中）。最大取捨：token 沿用 P1 refresh token 的 selector/verifier 模式，安全等級一致、也省一套新機制。**刻意不動 `auth.controller.ts`**（公司機有未收回的修補，避免衝突），全部放在新檔案。

## 1. 系統架構

**Ystravel-AuthService（後端）**
- 新增 `src/mail/`：`MailModule` + `MailService`（nodemailer SMTP transport）＋信件模板（TS 函式，繁中 HTML＋純文字）
- 新增 `src/auth/password-reset.controller.ts` + `src/auth/password-reset.service.ts`（公開三支 API 與核心邏輯）
- 修改 `src/users/users.controller.ts`：`create()` 去密碼化＋寄邀請信；移除 `PUT :id/password`；新增發送重設連結/重發邀請兩支
- Prisma：新增 `PasswordToken`、`PasswordMailThrottle`，`UserStatus` 加 `INVITED`
- **不動 `src/auth/auth.controller.ts`**（INVITED 帳號因 `passwordHash` 為空，登入本來就會走既有 BAD_PASSWORD 路徑被拒，無需改登入邏輯）

**Ystravel-AuthPortal（前端）**
- `LoginPage.vue`：加「忘記密碼？」連結
- 新增 `ForgotPasswordPage.vue`（路由 `/forgot-password`）
- 新增 `ResetPasswordPage.vue`（路由 `/reset-password?token=...`；重設與邀請**共用**，依 validate API 回的 `purpose` 切換文案）
- `AdminUsersPage.vue`：移除兩處密碼欄位；建帳號成功改「邀請信已寄出」；列表顯示「邀請中」＋「重發邀請」；「重設密碼」改「發送重設連結」
- `services/auth-api.ts`：新 API 方法＋**錯誤訊息翻譯表補新字串**（英文原字串當 key 的既有慣例）

**Ystravel-LocalDocker**
- 新增 Mailpit 容器：SMTP `:1025`、Web UI `:8025`（開瀏覽器看信）

## 2. API 設計

| Method | Path | 說明 | 權限 |
|---|---|---|---|
| POST | `/auth/password/forgot` | 提交 email 申請重設（一律回 202） | 公開 |
| POST | `/auth/password/validate` | 驗證連結 token 是否有效，回 `purpose`（RESET/INVITE） | 公開 |
| POST | `/auth/password/reset` | 用 token + 新密碼完成重設/邀請設定 | 公開 |
| POST | `/users/:id/send-reset-link` | 管理員對啟用帳號發重設信 | `AUTH.USER.MANAGE` |
| POST | `/users/:id/resend-invite` | 管理員對邀請中帳號重發邀請信 | `AUTH.USER.MANAGE` |
| POST | `/users`（修改） | 建帳號：移除 `password` 欄位，建立即寄邀請信 | `AUTH.USER.MANAGE` |
| ~~PUT~~ | ~~`/users/:id/password`~~ | **移除**（管理員不再能設密碼） | — |

Request/response 型別見 `./api-spec.md`。

## 3. DB Schema 異動

Migration 規劃：`20260709210000_password_reset_and_invite`

```prisma
enum UserStatus {
  ACTIVE
  DISABLED
  INVITED   // 新增：已建立、尚未設定密碼，不可登入
}

enum PasswordTokenPurpose {
  RESET    // 忘記密碼重設，30 分鐘
  INVITE   // 新帳號設定密碼，72 小時
}

model PasswordToken {
  id           String               @id @default(uuid()) @db.Char(36)
  userId       String               @db.Char(36)
  purpose      PasswordTokenPurpose
  selector     String               @unique @db.Char(32)
  verifierHash String               @db.VarChar(64)   // SHA-256 hex
  expiresAt    DateTime
  usedAt       DateTime?
  createdAt    DateTime             @default(now())
  user         AuthUser             @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@map("password_tokens")
}

model PasswordMailThrottle {
  id              String   @id @default(uuid()) @db.Char(36)
  email           String   @unique @db.VarChar(160)  // 正規化後的 email
  windowStartedAt DateTime @default(now())
  sentCount       Int      @default(0)
  lastSentAt      DateTime @default(now())
  updatedAt       DateTime @updatedAt

  @@map("password_mail_throttles")
}
```

- 既有資料不受影響（純新增，無資料搬移）
- 「一人同時只有一個有效連結」：發新 token 前先把該 user 同 purpose 的未用 token 全部標記 `usedAt`（作廢）

## 4. Service / Module 分層

沿用既有 pattern（`src/auth/` 內 controller + service + config + spec 平放；`MailModule` 比照 `PrismaModule` 做全域 provider）。token 產生/驗證邏輯收在 `password-reset.service.ts`，selector/verifier 做法直接對齊 `auth.controller.ts` 既有 refresh token 實作（`randomBytes` + SHA-256 固定時間比對）。

## 5. 認證與授權方式

- 公開三支不掛 guard（等同 `/auth/login`），靠 token 本身與頻率限制防護
- 管理員兩支沿用 `JwtAuthGuard` + `PermissionsGuard`（`AUTH.USER.MANAGE`），與既有 users API 相同
- 不新增 permission code

## 6. 安全與稽核（必填 — 依 SECURITY_AND_ISO27001_BASELINE.md §6.3）

| 欄位 | 內容 |
|---|---|
| 敏感欄位保護 | token：DB 只存 selector＋SHA-256(verifier)，明文只在信件連結；密碼：bcrypt cost 12（沿用）；驗證用固定時間比對 |
| Audit log | `PASSWORD_RESET_REQUESTED`（申請，含 email/IP）、`PASSWORD_RESET_DENIED`（停用帳號申請）、`PASSWORD_RESET_RATE_LIMITED`、`PASSWORD_RESET_COMPLETED`（含全裝置登出數）、`INVITE_SENT`（含操作者）、`INVITE_COMPLETED`、`ADMIN_RESET_LINK_SENT`（含操作者）。**絕不記錄密碼、token 明文、連結 URL** |
| 備份 / 還原 | 無影響（新表隨既有備份策略） |
| 錯誤處理與告警 | 寄信失敗：API 回 502＋audit `MAIL_SEND_FAILED`；建帳號不回滾（帳號已建，可重發邀請）。token 驗證失敗一律同一種 400，不透露過期/已用/不存在的差別 |
| 測試資料策略 | 測試全用假 email（`@ystravel.com.tw` 假帳號）；mailer 在測試中 mock，不真寄信；本機手動 QA 用 Mailpit，信不出本機 |

重設成功的全裝置登出：復用 P6 `logout` 的「撤銷該 user 全部 active refresh token」邏輯，`revokedReason` 用既有 `LOGOUT`（或新增 `PASSWORD_RESET`，實作時視 enum 寬度決定，見 §10 備註）。

## 7. 效能與非功能需求

- 低頻功能，無效能疑慮；token 表用 selector 唯一索引查詢，無全表掃描
- 過期 token 清理：發新 token 時順手刪該 user 的過期記錄（lazy cleanup），不另建排程
- 頻率限制 read-then-upsert 的極端併發誤差可接受（同 P2 `login_attempts` 的既有取捨）

## 8. 錯誤碼

| 情境 | HTTP Status | 訊息（英文原字串 = 前端翻譯 key） |
|---|---|---|
| 申請重設（無論結果） | 202 | `If the email is registered, a reset link has been sent.` |
| token 無效/過期/已用 | 400 | `Reset link is invalid or expired.` |
| 新密碼不合政策 | 400 | 沿用 P3 既有政策訊息（前端表已有） |
| 新密碼與現用相同 | 400 | `New password must be different from the current password.`（既有） |
| 管理員對停用帳號發連結 | 400 | `Cannot send a reset link to a disabled account.` |
| 管理員對非邀請中帳號重發邀請 | 400 | `This account has already completed setup.` |
| 無權限 | 403 | 既有 guard 行為 |
| 寄信失敗 | 502 | `Failed to send email. Please try again later.` |

⚠️ 實作時 AuthPortal `auth-api.ts` 翻譯表**必須同步新增**上述新字串（記憶明訓：後端訊息字串就是前端翻譯 key）。

## 9. 測試策略

- **Unit（jest）**：`password-reset.service` token 產生/驗證/過期/單次性/作廢舊 token；throttle 計數與視窗重置；email 正規化
- **BDD（jest-cucumber）**：`password-reset.feature` 22 個場景接 steps，走真 controller/service、mock Prisma 與 MailService（比照 `login-access.steps.ts` 風格）
- **前端**：`vue-tsc -b` typecheck＋`vite build`；AuthPortal 無測試框架，互動走手動 QA
- **手動 QA（Mailpit）**：全流程實測——忘記密碼收信、過期/重放、邀請信、重發、管理員發連結、全裝置登出（兩個瀏覽器驗）

## 10. 待決策事項（Steven 拍板）

1. **新套件 `nodemailer`**（寄信業界標準、維護活躍、零額外服務依賴）。**建議：採用。** 備選：自己用 raw SMTP socket（重造輪子）或第三方 SDK（綁服務商）。
2. **`UserStatus` 加 `INVITED`**。**建議：採用**——列表要顯示「邀請中」，明確的狀態比「用 passwordHash 是否為空來猜」乾淨（記憶明訓：不沿用隱晦判斷省工）。備選：不加 enum、前端自己推導（省 migration 但語意藏在程式碼裡）。
3. **Mailpit 進 LocalDocker**（`axllent/mailpit` 官方 image，SMTP :1025 / UI :8025）。**建議：採用。** LocalDocker 非 git repo，直接改 compose。

（備註，非決策：重設成功撤 token 的 `revokedReason` 若用新值 `PASSWORD_RESET` 需確認欄位 VarChar(20) 夠寬——`PASSWORD_RESET` 恰 14 字元，可用；實作時直接採新值，audit 語意較清楚。）

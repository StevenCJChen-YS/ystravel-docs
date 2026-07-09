# API Spec: auth-password-reset

> 對應 `./sd.md` §2。所有訊息字串為英文（前端 `auth-api.ts` 翻譯表轉繁中）。

## POST /auth/password/forgot（公開）

申請重設密碼連結。**無論 email 存不存在都回 202 同一內容**（防帳號枚舉）。

```jsonc
// Request
{ "email": "amy@ystravel.com.tw" }

// Response 202（一律）
{ "message": "If the email is registered, a reset link has been sent." }
```

- email 先 `trim().toLowerCase()` 正規化
- 啟用帳號 → 寄重設信（30 分）；邀請中帳號 → 重寄邀請信（72 小時）；停用/不存在 → 不寄
- 頻率限制內（60 秒 1 封、每小時 5 封）→ 靜默不寄，仍回 202

## POST /auth/password/validate（公開）

前端載入重設頁時先驗證連結，決定顯示重設或邀請文案。

```jsonc
// Request
{ "token": "<selector>.<verifier>" }

// Response 200
{ "purpose": "RESET" }   // 或 "INVITE"

// Response 400（無效/過期/已用，一律同一種）
{ "message": "Reset link is invalid or expired." }
```

## POST /auth/password/reset（公開）

用有效 token 設定新密碼（重設與邀請共用）。

```jsonc
// Request
{ "token": "<selector>.<verifier>", "newPassword": "X7#mQ2vLp9$wK4" }

// Response 200
{ "message": "Password has been updated." }

// Response 400：token 無效同 validate；密碼政策沿用 P3 訊息；
// 重設時新舊相同 → "New password must be different from the current password."
```

- 成功時：寫入新 bcrypt hash → token 標記已用 → 撤銷該 user 全部 active refresh token（`revokedReason=PASSWORD_RESET`）→ INVITE 時帳號 `INVITED → ACTIVE`
- audit：`PASSWORD_RESET_COMPLETED` 或 `INVITE_COMPLETED`

## POST /users/:id/send-reset-link（`AUTH.USER.MANAGE`）

```jsonc
// Request：無 body
// Response 200
{ "message": "Reset link sent." }

// Response 400（目標帳號非 ACTIVE）
{ "message": "Cannot send a reset link to a disabled account." }
```

- 對 INVITED 帳號回 400（前端此時只該出現「重發邀請」）
- audit：`ADMIN_RESET_LINK_SENT`（actor = 操作管理員）

## POST /users/:id/resend-invite（`AUTH.USER.MANAGE`）

```jsonc
// Request：無 body
// Response 200
{ "message": "Invitation sent." }

// Response 400（目標帳號非 INVITED）
{ "message": "This account has already completed setup." }
```

- 舊邀請 token 全部作廢，重新起算 72 小時
- audit：`INVITE_SENT`（actor = 操作管理員）

## POST /users（既有，修改）

- Request 移除 `password` 欄位（後端不再接受；帶了也忽略）
- 建立後 `status = INVITED`、`passwordHash = null`，自動寄邀請信
- 寄信失敗：帳號仍建立成功，回 502 `Failed to send email. Please try again later.`，管理員可重發

## PUT /users/:id/password（既有，移除）

整支刪除，AdminUsersPage 對應 UI 一併移除。

## 信件連結格式

```
{AUTH_PORTAL_BASE_URL}/reset-password?token=<selector>.<verifier>
```

- selector：16 bytes randomBytes → 32 字元 hex（DB 唯一索引查詢用）
- verifier：32 bytes randomBytes → base64url（DB 只存 SHA-256 hex，固定時間比對）

## 新增 env（`.env.example` 同步更新）

| 變數 | 預設 | 說明 |
|---|---|---|
| `SMTP_HOST` / `SMTP_PORT` | `127.0.0.1` / `1025` | 本機 = Mailpit |
| `SMTP_SECURE` | `false` | 正式環境視公司 SMTP 設定 |
| `SMTP_USER` / `SMTP_PASS` | 空 | Mailpit 不需要 |
| `MAIL_FROM` | `no-reply@ystravel.com.tw` | 寄件者 |
| `PASSWORD_RESET_TOKEN_TTL_MINUTES` | `30` | 重設連結效期 |
| `INVITE_TOKEN_TTL_HOURS` | `72` | 邀請連結效期 |
| `PASSWORD_MAIL_MIN_INTERVAL_SECONDS` | `60` | 同 email 最小寄信間隔 |
| `PASSWORD_MAIL_HOURLY_LIMIT` | `5` | 同 email 每小時上限 |

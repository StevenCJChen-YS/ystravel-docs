# SD：帳號中心 Phase 1

| 項目 | 內容 |
|---|---|
| 對應 SA | [./sa.md](./sa.md) |
| 對應 PRD | [./prd.md](./prd.md) |
| 狀態 | 已確認（2026-07-15，§10 D1~D3 拍板：vue-advanced-cropper／密碼政策沿用 10／改密頻率限制開新表） |
| 建立日期 | 2026-07-15 |

## 0. 白話摘要

後端在 AuthService 開一個新的 `account` 模組，放三支「只能改自己」的自助 API：頭貼上傳/移除、名片背景、改密碼；頭貼會在伺服器端重新編碼成 512×512 WebP（安全核心），檔案存磁碟具名 volume、DB 只存相對路徑。前端把現在內嵌在 UserMenu 的名片抽成共用 `UserIdentityCard`，個人資料頁改唯讀、新增設定頁集中所有編輯。三個真正要你拍板的：**①頭貼裁切要引入一個前端套件（新依賴）②PRD 寫密碼 ≥12 但現有政策是 10，改密要用哪個 ③改密頻率限制要不要開一張小表**。

## 1. 系統架構

**AuthService（後端）**
- 新增 `src/account/` 模組：`account.module.ts`、`account.controller.ts`、`account.service.ts`（沿用 `users`/`roles` 既有分層）。
- 新增 `src/account/avatar-storage.ts`：storage 抽象介面＋本機磁碟實作（未來可換 MinIO）。
- 修改 `AuthUser` 的 token/回應組裝（`auth.controller.ts` `buildAuthUser`＋`auth-user.ts` 型別）：帶出 `avatarUrl`、`backgroundKey`。
- 修改 `main.ts`：改用 `NestExpressApplication` 掛靜態檔服務頭貼。
- 新增依賴：`sharp`（SA-2 已拍板）。

**AuthPortal（前端）**
- 新增共用元件 `src/shared/ui/UserIdentityCard.vue`（從 UserMenu popover 抽出，支援 `popover` 小尺寸與 `hero` 大尺寸）。
- 新增 `src/shared/lib/profile-backgrounds.ts`：backgroundKey → 素材圖對照（日本為第一張＋預設）。
- 改寫 `AccountProfilePage.vue`（唯讀版）；新增 `AccountSettingsPage.vue`（`/account/settings`）。
- 改 `UserMenu.vue`（用 UserIdentityCard、加「設定」選項）、`stores/auth.ts`（型別＋動作）、`services/auth-api.ts`（API client）、`router/index.ts`（新路由）。
- 新增前端依賴：裁切套件（見 §10 D1）。

> UI 動工前依規範先讀 `/nuxt-ui` skill＋`docs/design-system/foundation.md`＋`AUTHPORTAL_UI_FOUNDATION.md`，優先沿用既有共用組件（FormModal、AppInput、LobbyShell、AppPageLayout…）與既定色階/dark 明度分層。

## 2. API 設計（含 request/response，取代獨立 api-spec.md）

全域前綴 `api/auth`（既有），下列 path 皆再加該前綴。除註明外都需 `Authorization: Bearer`，操作對象一律取 token 內 `sub`（本人），**不接受指定他人 id**。

| Method | Path | 說明 | 權限 |
|---|---|---|---|
| POST | `/account/avatar` | 上傳頭貼（multipart，欄位 `file`） | 已登入本人 |
| DELETE | `/account/avatar` | 移除頭貼回預設 | 已登入本人 |
| PUT | `/account/background` | 設定名片背景 | 已登入本人 |
| POST | `/account/password` | 自助改密碼 | 已登入本人 |
| GET | `/account/security` | 回帳號安全狀態（供設定頁判斷顯示） | 已登入本人 |
| GET | `/avatars/:file` | 靜態服務頭貼圖（無需 bearer，檔名為不可猜的 uuid） | 公開讀取 |

**POST `/account/avatar`**（`multipart/form-data`，`file`）
- 回 `200 { avatarUrl: string }`（相對路徑，如 `/api/auth/avatars/<uuid>.webp`）
- 流程：Multer memory storage（`limits.fileSize = 5MB`）→ `sharp(buffer).metadata()` 驗 `format ∈ {jpeg,png,webp}`（非副檔名）→ `sharp(buffer).rotate().resize(512,512,{fit:'cover'}).webp().toBuffer()` → 以新 `<uuid>.webp` 寫入 storage → 更新本人 `avatarUrl`（相對路徑）→ 刪除舊檔（若有）→ audit `AVATAR_UPDATED`。

**DELETE `/account/avatar`** → `200 { avatarUrl: null }`；`avatarUrl=null`、刪實體檔、audit `AVATAR_REMOVED`。

**PUT `/account/background`** `{ backgroundKey: string }`
- 驗 `backgroundKey ∈ ALLOWED_BACKGROUND_KEYS`（後端常數清單，與前端素材庫同步）→ 更新 → audit `BACKGROUND_UPDATED` → `200 { backgroundKey }`；未知 key → `422`。

**POST `/account/password`** `{ currentPassword, newPassword, currentRefreshToken }`
- 頻率限制檢查（見 §7）→ `bcrypt.compare(currentPassword, passwordHash)`；錯 → audit `PASSWORD_CHANGE_FAILED`＋計數＋`401`
- `assertPasswordMeetsPolicy(newPassword, ownerContext)`；不符 → `400`（沿用既有政策訊息）
- 擋「與現用相同」（`bcrypt.compare(newPassword, passwordHash)`）→ `400`
- 交易：寫新 `passwordHash` → 撤銷「本人除 `currentRefreshToken` 之 selector 外」全部 active refresh token（`revokedReason='PASSWORD_CHANGE'`）→ audit `PASSWORD_CHANGED` → `200 { ok: true }`
- `passwordHash` 為 null（無密碼帳號）→ `400 NO_PASSWORD`（防禦；正常不會走到，設定頁已擋）

**GET `/account/security`** → `200 { hasPassword: boolean }`（`hasPassword = passwordHash != null`）；設定頁據此顯示改密表單或替代狀態（Rule C4）。

**登入/refresh/me 回應**：`user` 物件新增 `avatarUrl`（展開為可載入路徑）、`backgroundKey`。

## 3. DB Schema 異動

`AuthUser` 新增一欄（`avatarUrl` 已存在，改為語意＝相對路徑）：

```prisma
model AuthUser {
  // ...既有欄位...
  avatarUrl     String?  @db.VarChar(255)  // 既有；語意改為相對路徑 avatars/<uuid>.webp
  backgroundKey String?  @db.VarChar(40)   // 新增；null = 預設（japan）
}
```

改密頻率限制表（§10 D3 若採「開新表」）：

```prisma
model PasswordChangeAttempt {
  id           String    @id @default(uuid()) @db.Char(36)
  userId       String    @db.Char(36)
  ipAddress    String    @db.VarChar(80)
  failedCount  Int       @default(0)
  lockedUntil  DateTime?
  lastFailedAt DateTime  @default(now())
  updatedAt    DateTime  @updatedAt
  @@unique([userId, ipAddress])
  @@map("password_change_attempts")
}
```

- `RefreshToken.revokedReason` 註解補 `PASSWORD_CHANGE`（不改型別，只擴充語意）。
- Migration：`add_background_key_and_password_change_attempt`（`migrate dev` 產生；本機 `migrate deploy`）。

## 4. Service / Module 分層

沿用既有模式（controller 收 HTTP＋權限、service 放邏輯、prisma 注入）：
- `AccountController`：路由、`@UseGuards(JwtAuthGuard)`、`FileInterceptor`、`@CurrentAuthUser()`。
- `AccountService`：頭貼驗證/re-encode/存取、背景 key 驗證、改密（政策＋token 撤銷＋頻率限制）、audit。
- `AvatarStorage`（介面）＋`LocalDiskAvatarStorage`（實作）：`save(buffer, key)`、`delete(key)`、`resolveUrl(key)`。未來換 MinIO 只換實作、不動 service。
- 測試用 fake-prisma 比照 `password-reset.fake-prisma.ts`。

## 5. 認證與授權方式

- 全部沿用 `JwtAuthGuard`；**不新增 permission code**（自助操作＝登入身分即可，對象恆為本人）。
- 靜態頭貼 `GET /avatars/:file` 不掛 guard：檔名是不可猜的 uuid、內容分級 Internal（名片本來就給同事看），比照公開頭像慣例；不接受目錄跳脫（只服務 storage 目錄內既有檔）。

## 6. 安全與稽核（必填）

| 欄位 | 內容 |
|---|---|
| 敏感欄位保護 | 密碼：只存 bcrypt(12) hash；明文僅記憶體短暫存在，**不落 log/audit/回應**。頭貼：伺服器端 re-encode 去除原始 metadata/夾帶內容，只出點陣 WebP |
| 上傳防護 | ①Multer memory storage `fileSize=5MB` 上限 ②型別白名單走 `sharp` metadata（magic bytes）非副檔名，只收 jpeg/png/webp、**拒 SVG/GIF** ③強制 re-encode＋resize，丟棄原始檔 ④檔名用系統 uuid，非使用者輸入（擋 path traversal）⑤靜態服務限定 storage 目錄 |
| Audit log | `AVATAR_UPDATED`／`AVATAR_REMOVED`／`BACKGROUND_UPDATED`／`PASSWORD_CHANGED`／`PASSWORD_CHANGE_FAILED`／`PASSWORD_CHANGE_RATE_LIMITED`（`resourceType='auth_user'`、`resourceId=本人`、actor=target=本人）；**密碼類 audit 僅記事件與時間，絕無明文** |
| 備份 / 還原 | 頭貼 volume 需與 `ystravel_mysql_data` 一起納入備份（SA-1）；DB 存相對路徑，還原後不因網域改變而失效 |
| 錯誤處理與告警 | 沿用既有 Nest exception filter＋前端 `getAuthApiErrorMessage` 翻譯表（新錯誤字串補譯）；頻率限制觸發留 audit 供查 |
| 測試資料策略 | 一律合成帳號（`amy@ystravel.com.tw` 等），不用真實員工個資；測試頭貼用程式產生的小圖，不放真人照片 |

## 7. 效能與非功能需求

- 頭貼：單檔 ≤5MB、`sharp` 同步處理單張約數十毫秒，低頻操作，無分頁/批次需求。
- 改密頻率限制：比照 `LoginAttempt`（`(userId, ip)` 計數＋鎖定視窗，門檻沿用登入設定或另設環境變數）。
- 其餘沿用既有標準。

## 8. 錯誤碼

| 情境 | HTTP | 說明 |
|---|---|---|
| 未登入/token 失效 | 401 | `JwtAuthGuard` |
| 頭貼格式不符（假圖/SVG/非圖片） | 400 | 明確訊息，不寫檔 |
| 頭貼超過大小上限 | 413 / 400 | Multer 擋下 |
| 背景 key 未知 | 422 | 拒絕，背景不變 |
| 改密：目前密碼錯誤 | 401 | 「目前密碼不正確」，計入頻率限制 |
| 改密：新密碼不合政策/與現用相同 | 400 | 沿用政策訊息 |
| 改密：觸發頻率限制 | 429 | 暫時擋下 |
| 無密碼帳號打改密 | 400 | `NO_PASSWORD`（防禦） |

## 9. 測試策略

- **後端 BDD（jest-cucumber）**：`src/account/account-self-service.feature`（已確認的 Gherkin）＋`account-self-service.steps.ts`，用 fake-prisma。涵蓋：頭貼本人限定/型別/大小/移除、背景 key 白名單、改密正/誤/政策/相同/頻率限制/保留本機、無密碼替代狀態、audit 無明文。
- **後端單元**：`sharp` re-encode 型別驗證（假圖被擋、真圖產出 512 WebP）、改密 token 撤銷「保留本機 selector」邏輯、頻率限制計數。
- **前端**：手動 QA（依 verify 流程）——唯讀個人資料頁、設定頁三功能、頭貼上傳裁切、背景選擇、改密後「其他裝置登出／本機保留」實測、Google-only 帳號替代狀態、light/dark 兩模式。
- **不做**：真人頭像、真實個資、跨系統名片顯示（本輪僅定資料模型）。

## 10. 待決策事項

| # | 問題 | 選項與建議 |
|---|---|---|
| **D1** | 頭貼裁切要引入哪個前端套件（PRD 要正方形裁切＋縮放拖曳，這是新依賴） | **建議 `vue-advanced-cropper`**（Vue3 原生、輕、API 清楚、可鎖正方形＋縮放拖曳）。備選：`cropperjs`（成熟但要自己包 Vue wrapper）。**這是本輪唯一新增的前端依賴，依規則明確提出請你點頭。** |
| **D2** | PRD §5 C 寫新密碼「≥12 碼」，但現有密碼政策 `MIN_PASSWORD_LENGTH` 已於 2026-07-09 你實測後調成 **10** | **建議改密直接重用既有政策（＝10）**，全站單一真相來源、與登入/重設一致；PRD 那句 12 視為撰寫前的舊值、我回頭把 PRD 註記修正。若你堅持改密要 12，就得為「改密」開特例（不建議，會不一致） |
| **D3** | 改密頻率限制存哪 | **建議開新小表 `PasswordChangeAttempt`**（跟登入鎖定 `LoginAttempt` 分開，語意乾淨）。備選：共用 `LoginAttempt`（省一張表，但會把「改密失敗」混進「登入鎖定」計數，較混淆） |
| D4（技術，將逕行） | `avatarUrl`/`backgroundKey` 放進 token 組裝 `buildAuthUser` | 直接加進去，登入/refresh/me 都帶出、名片各處直接吃；改頭貼後前端用 API 回應即時更新 store。token 略增數十 bytes，可接受 |
| D5（技術，將逕行） | 新模組 `account` vs 塞進 `auth` | 開獨立 `account` 模組（自助功能聚一處，好維護） |

**決議（2026-07-15 Steven 拍板）**：D1＝`vue-advanced-cropper`；D2＝改密沿用既有政策（`MIN_PASSWORD_LENGTH=10`），PRD §5 C「≥12」註記修正為沿用政策值；D3＝開新表 `PasswordChangeAttempt`；D4/D5＝照建議（加進 `buildAuthUser`、開獨立 `account` 模組）。

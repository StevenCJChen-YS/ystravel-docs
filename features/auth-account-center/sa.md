# SA：帳號中心 Phase 1

| 項目 | 內容 |
|---|---|
| 對應 PRD | [./prd.md](./prd.md) |
| 對應 Example Mapping | [./example-mapping.md](./example-mapping.md) |
| 狀態 | 已確認（2026-07-15，§9 決策全數拍板） |
| 建立日期 | 2026-07-15 |

## 0. 白話摘要

這份分析「員工自助改自己的頭貼、名片背景、密碼」這三件事，在系統裡實際會怎麼流動、有哪些風險。三件事都只有**本人**能改自己的，公司資料（部門/職稱/角色）維持唯讀。技術上兩塊是既有機制的延伸（改密碼＝重用密碼政策＋refresh token 撤銷；背景＝存一個代號欄位），**唯一全新且風險最高的是「頭貼上傳」**——AuthService 目前完全沒有檔案上傳能力，上傳檔案是新的攻擊面（偽裝圖片的惡意檔、超大檔），需要嚴格的型別/大小驗證＋伺服器端重新編碼。最大的兩個待決策：①頭貼檔案存哪裡（本機磁碟 vs 物件儲存）②改密碼要「只登出其他裝置、保留本機」需要一個機制辨識「本機」。

## 1. 系統邊界

| 系統 | 這次的角色 |
|---|---|
| **AuthService**（NestJS + Prisma + MySQL） | 新增自助 profile API：頭貼上傳/移除、背景設定、改密碼；新增頭貼檔案儲存與服務；擴充 audit |
| **AuthPortal**（Vue3 + Nuxt UI） | 個人資料頁改唯讀版、新增設定頁、抽 `UserIdentityCard` 共用元件、串接自助 API |
| **CRM / EIP** | 本輪不動；資料模型（avatarUrl/backgroundKey）定好供未來共用（PRD §8） |

既有架構邊界不重寫，引用：
- [CRM_AUTH_SERVICE_SA.md](../../architecture/CRM_AUTH_SERVICE_SA.md)、[CRM_AUTH_SERVICE_SD.md](../../architecture/CRM_AUTH_SERVICE_SD.md)（AuthService 為集中身分中樞、RS256/JWKS、refresh token selector 機制）
- [DATABASE.md](../../architecture/DATABASE.md)（DB＝MySQL 8，schema≠data 遷移原則）
- [`auth-native-login-v2/sd.md`](../auth-native-login-v2/sd.md) §10.2（密碼政策）、[`auth-password-reset/`](../auth-password-reset/)（去密碼化、寄信基建、`completeReset` 全裝置登出）

既有可直接重用的機制（都已在 code 裡驗證存在）：
- `passwordHash`（bcrypt cost 12）、`assertPasswordMeetsPolicy()`（含 zxcvbn 強度＋擋個資型＋≥12 碼）
- `avatarUrl String?` 欄位已存在（目前僅管理員填字串）
- `RefreshToken`（selector.verifier、`appCode`、`revokedAt`/`revokedReason`、`updateMany` 批次撤銷）
- `AuditLog`（actorUserId/targetUserId/action/appCode/ipAddress/diff）
- `LoginAttempt`（username+ip 的失敗計數＋鎖定，改密頻率限制可比照）

## 2. 角色與權限

自助功能的核心原則：**不需要任何管理權限，只需「已登入且操作對象是自己」**。後端一律以 bearer token 內的 `sub`（本人 id）為操作對象，**不接受請求指定 targetUserId**。

| 角色 | 可以做什麼 |
|---|---|
| 任何已登入員工（ACTIVE） | 改自己的頭貼、背景、密碼；讀自己的帳號安全狀態 |
| 管理員 | 本輪**不新增**任何管理功能；不能替員工改頭貼/背景/密碼 |
| 未登入 / INVITED / DISABLED | 無法使用（受既有 `JwtAuthGuard` 擋下；INVITED/DISABLED 拿不到有效 access token） |

- **不需要新的 permission code**：這些是「對自己的資源」的操作，用登入身分即可，不進 permission catalog（符合既有「授權判斷看 permission、但自助操作不掛管理權限」的分界）。
- 「移除不當頭像」的管理員權限＝PRD §9 待議、Phase 1 不做。

## 3. 操作流程（正常路徑）

**A 大頭貼**
1. 員工在設定頁 🪪 個人化區塊按「更換頭像」→ 選圖 → 前端正方形裁切框（縮放/拖曳）→ 確認。
2. 前端把裁切後的圖以 multipart 上傳到 `POST /account/avatar`（bearer）。
3. 後端：驗登入 → 驗檔案型別（magic bytes）＋大小 → **伺服器端重新編碼＋縮放**成固定規格（去除原始 metadata/夾帶）→ 存檔 → 更新本人 `avatarUrl` → 寫 `AVATAR_UPDATED` audit。
4. 回傳新 avatarUrl；前端更新 hero＋UserMenu；其他人下次載入名片時看到新圖。
- 移除：`DELETE /account/avatar` → `avatarUrl = null`、刪實體檔（或留孤兒待清）→ `AVATAR_REMOVED` audit。

**B 名片背景**
1. 員工按「更換名片背景」→ 縮圖清單（前端內建素材庫）→ 選一張 → 確認。
2. `PUT /account/background` 送 `backgroundKey`（bearer）。
3. 後端驗 key 在允許清單內 → 更新本人 `backgroundKey` → `BACKGROUND_UPDATED` audit → 回傳。
4. 前端更新 hero＋UserMenu popover 背景。

**C 改密碼**
1. 員工在 🔐 帳號安全區塊按「修改密碼」→ modal 三欄位（目前/新/確認）＋強度即時提示。
2. `POST /account/password` 送 `currentPassword` + `newPassword`（bearer）。
3. 後端：頻率限制檢查 → 驗 `currentPassword`（bcrypt.compare）→ 過密碼政策 → 擋「與現用相同」→ 寫新 hash → **撤銷其他裝置 refresh token、保留本機** → `PASSWORD_CHANGED` audit → 回傳成功。
4. 前端 toast「已更新，其他裝置已登出」。

## 4. 狀態轉換

- 頭貼：`無頭貼(預設) ⇄ 有自訂頭貼`（上傳/移除切換）。
- 背景：`預設(japan) ⇄ 指定 backgroundKey`（永遠有值，未設＝預設）。
- 帳號 `status`：本功能**不改變** `status`（INVITED→ACTIVE 仍只由邀請流程負責，見 Example Mapping C 前置事實）。改密碼不改 status。

## 5. 例外流程

| 情境 | 系統行為 |
|---|---|
| 未登入 / token 失效 | `JwtAuthGuard` 回 401，前端導回登入 |
| 頭貼格式不符（假圖/非圖片） | 400，明確錯誤訊息，不寫檔、不改 avatarUrl |
| 頭貼超過大小上限 | 413/400，明確錯誤，畫面已先顯示限制值 |
| 背景 key 不在允許清單 | 400/422 拒絕，背景不變 |
| 改密：目前密碼錯誤 | 400/401「目前密碼不正確」，寫 `PASSWORD_CHANGE_FAILED` audit，計入頻率限制 |
| 改密：短時間多次錯誤 | 429 頻率限制，寫 `PASSWORD_CHANGE_RATE_LIMITED` audit |
| 改密：新密碼不合政策/與現用相同 | 400，回具體原因（沿用重設頁體驗） |
| 無密碼帳號查改密狀態 | 回「未設定密碼」，前端顯示替代狀態而非表單（Rule C4） |
| 並發：同時兩次上傳/改密 | 以最後成功者為準；改密用交易確保 hash 更新與 token 撤銷原子性 |

## 6. 資料流與敏感資料

- **密碼**（PRD §7＝比照 Restricted）：明文只在「前端輸入 → HTTPS → 後端記憶體比對/雜湊」這條路徑短暫存在，**絕不落 log/audit/回傳**；儲存僅 bcrypt hash。audit 只記事件類型與時間。
- **頭貼影像**（Internal，可識別個人）：前端 → multipart 上傳 → 後端重新編碼 → 存檔（磁碟或物件儲存，見 §9）→ 以 URL 提供給已登入者。原始上傳檔不保留（重新編碼後丟棄），去除 EXIF 等 metadata。
- **backgroundKey**（Internal）：只是素材代號字串，無敏感性。
- 無資料匯出功能；無外部 AI 接觸；頭貼不外流到受控環境外。

## 7. 與外部系統的互動

- 本功能不呼叫外部系統。頭貼服務若採物件儲存（MinIO/S3）才會多一個內部依賴（見 §9）。
- 未來 CRM/EIP 讀名片＝讀 AuthService 的 `avatarUrl`/`backgroundKey`（本輪只定資料模型，不實作跨系統顯示）。

## 8. 風險點與人工審核點

| 風險 | 說明 | 對策 |
|---|---|---|
| **頭貼上傳是新攻擊面**（最高） | 偽裝成圖片的惡意檔、超大檔打爆儲存/記憶體、SVG/含 script 圖、path traversal 檔名 | ①型別白名單走 magic bytes 非副檔名 ②大小上限（memory storage 限制）③**伺服器端 re-encode＋resize**（用 `sharp`）產出乾淨圖、丟棄原始檔 ④檔名用系統產生的 id 非使用者輸入 ⑤只接受點陣格式（JPG/PNG/WebP），不接 SVG |
| 不當頭像（冒犯性圖片） | 無自動審核 | Phase 1 行政管理；管理員移除權限＝§9 待議 |
| 改密表單被拿來暴力試「目前密碼」 | 攻擊者用被盜 access token 猜舊密碼 | 頻率限制（比照 `LoginAttempt`）＋audit |
| 「保留本機」誤殺當前 session | 若撤銷邏輯把本機也撤掉，使用者改完密碼立刻被登出 | 需明確機制辨識「本機」refresh token（見 §9），並寫測試守住 |
| 密碼明文外洩 | log/audit/錯誤訊息不慎帶入 | 明文只進 bcrypt；audit 只記事件；code review 檢查 |

**需要 Steven 拍板的（非全自動）**：頭貼儲存位置、大小/尺寸上限、是否本輪就做管理員移除頭像——見 §9。

## 9. 待決策事項

| # | 問題 | 決議（2026-07-15 Steven 拍板） |
|---|---|---|
| SA-1 | 頭貼檔案存哪裡 | **本機磁碟＋具名 Docker volume**（比照現有 `ystravel_mysql_data` 持久化）；**DB `avatarUrl` 只存相對路徑，完整網址執行時用當前環境網域組出**（解決 EIP「存完整網址、換環境就壞」的坑）；storage 介面設計成可抽換，未來多實例再換 MinIO |
| SA-2 | 新依賴 `sharp`（影像重新編碼/縮放） | **引入**——re-encode 是頭貼安全核心防線；本輪唯一新增的後端依賴 |
| SA-3 | 「保留本機、登出其他裝置」怎麼辨識本機 | **改密請求帶上本機當前 refresh token**，後端撤銷「該使用者除此 selector 外」的所有 active token；本機保留 |
| SA-4 | 頭貼輸出規格 | **輸出 512×512 WebP、上傳原檔上限 5MB、只收 JPG/PNG/WebP**（實作細節 SD 定） |
| SA-5 | 頭貼如何對外服務 | **靜態路徑服務**（`/<base>/avatars/<檔名>`，已登入前端直接 `<img>`），非每次走 API；搭配 SA-1 相對路徑 |
| SA-6 | 管理員「移除不當頭像」是否本輪做 | **本輪不做**（先行政管理，PRD §9 傾向；避免擴大範圍） |

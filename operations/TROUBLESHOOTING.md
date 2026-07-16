# 專案踩坑與環境排障（Troubleshooting）

> **定位**：專案層的踩坑教訓與環境排障手冊——2026-07-16 知識分層整理時從 AI 分身記憶（my-agent `MEMORY.md`）遷移至此，之後新的專案坑直接記這裡。
> 跨專案的「機器環境坑」（Windows port 保留區段、PowerShell 5.1 編碼）仍在 my-agent MEMORY.md（跟機器走、不跟專案走）。
> ⚠️ 檔內 repo/路徑以 Auth/CRM 現行為準；Phase 1 整併入 ystravel-platform 後由平台 docs 接手。

---

## 啟動類（起不來先看這）

### pull 之後服務起不來／TS 報錯一堆（超常見，先做這兩步）
`Cannot find module 'sharp'`、Prisma client 缺新欄位（如 `backgroundKey`）＝ **pull 後沒 `npm install`＋`npx prisma generate`**。修：兩個都跑＋重啟 watch。DB 通常沒事：跑唯讀的 `npx prisma migrate status` 確認 schema up-to-date 即可（別急著 `migrate deploy`；它會被安全分類器判「正式部署」擋，且多半不需要）。（2026-07-16）

### AuthService 起不來：先查 RS256 env
native auth v2 必須有 `JWT_ACCESS_PRIVATE_KEY_BASE64`、`JWT_ACCESS_PUBLIC_KEY_BASE64`、`JWT_ACCESS_KEY_ID`。缺 signing config 會在 `/auth/login` 簽 token 時 500；已補啟動時 config validation（會驗 private/public 是否同組）。新環境起不來先檢查這三個 env，別誤判成登入程式壞掉。（2026-07-08）

### 頁面全噴 500／「無法載入」：先查 migration 有沒有套
本機 DB 長期沒 `migrate deploy` 過 → 資料表根本不存在。環境重建或換機器時先檢查 migration 狀態，不要一看到「無法載入」就以為程式碼壞掉。（2026-07-08 CRM 曾中招：migration 檔早寫好、從沒 deploy，`crm_customers` 等表全缺）

### schema/migration 改動後 watch 抱舊 client
改完 schema 要在 repo 目錄跑 `npx prisma generate`＋重啟 watch，否則 TS error 且跑舊 code。（2026-07-10）

## 開發類

### 頁面初始化別用 Promise.all 綁死「不同權限等級的 API」
2026-07-10 AuthPortal 使用者頁：載入同抓使用者/部門/角色三清單，HR 帳號（無 AUTH.ROLE.MANAGE）抓角色被正確 403，但一個請求失敗整頁「資料讀取失敗」。
**修法**：依權限條件式抓取（沒權限就不發）、非必要資料失敗不拖垮整頁、無權限的 UI 區塊藏掉（別留「0 個角色」空殼）。
**驗證**：用低權限測試帳號實走頁面（`admin:create` 腳本可帶 `FIRST_ADMIN_ROLES=AUTH_HR` 建測試帳號）。畫面上有依權限顯示/隱藏區塊的頁，初始抓取都要跟同一套權限條件。

### 匯入/批次功能：先估真實資料量，禁一次性讀入記憶體
2026-07-08 CRM 匯入第一版「一次讀進記憶體＋一次性 `createMany`」，真實 10 萬多列一上傳 backend 記憶體爆掉崩潰重啟、前端 loading 永遠轉圈（process 死了舊 HTTP 連線變孤兒）。
**修法**：ExcelJS streaming reader 逐列讀＋每 1000 列 `createMany` 分批寫，不一次持有全部資料，也避免單一 SQL 超過 packet 上限。
**驗證**：用 exceljs 寫一次性腳本（repo 內、`npx ts-node -T` 直呼 Service 繞過 HTTP/認證）產生真實規模合成資料實測，跑完清掉；別拿正式資料庫當測試場。
**套用**：涉及匯入/批次/大量寫入的功能，設計時就問「真實資料量多少」並寫進 SD 的效能需求節。**此經驗直接適用 platform 的 `integration/cowell` 科威匯入管線。**

### Google 登入測試規則（native login 與 Google login 是兩條路徑）
`GOOGLE_ALLOWED_TEST_EMAILS` 只影響 Google OAuth 流程，**不影響帳密登入**（「同 email 不在白名單仍可帳密登入」不是 bug）。非 production 若設了測試白名單，Google 帳號選擇器**不要送公司網域 `hd` 提示**，否則私人 Gmail 白名單帳號會在選號畫面被藏起來造成誤判；正式環境維持 `hd` 提示。（2026-07-09）

### `npm run lint --fix`（全專案 glob）會重排未修改檔案的 import 格式
prettier 規則的自然結果，不算 bug；寫 PR 時把這類格式化 diff 跟本體改動分開，別混進 commit。

## 建置與工具鏈類（`ystravel-platform` monorepo，2026-07-16 Phase 0 建置踩到）

### Prisma 7：datasource `url` 不能寫在 schema
Prisma 7 breaking change——`datasource` 的 `url` 不再寫在 `schema.prisma`。改法：CLI（`generate`/`migrate`）走 `prisma.config.ts`（`defineConfig({ datasource: { url } })`）；runtime 一律用 driver adapter（PostgreSQL＝`@prisma/adapter-pg`，`new PrismaPg(connectionString)` 傳給 `new PrismaClient({ adapter })`）。升級 Prisma 或新建專案先確認這兩處都改到位，否則 CLI 或 runtime 其一會連不到 DB。（2026-07-16）

### `vue-tsc` 撞 TypeScript 7（`typescript/lib/tsc` not exported）
monorepo 根被 hoist 上來的 TypeScript 7（來自 `@prisma/client` 的相依）會讓 `vue-tsc` 掛掉。解法＝monorepo **根** `devDependencies` 直接釘 `typescript@^5.9`——根的直接相依優先佔住 root `node_modules`，`vue-tsc` 就吃到 5.9。工具鏈報 `typescript/lib/tsc` 之類的 export 錯，先查 root 實際被 hoist 到哪個 TS 版本，別先懷疑 vue-tsc 本身。（2026-07-16）

### Docker Desktop 反覆「unexpected error」起不來：`dockerInference` socket 殘留
`%LOCALAPPDATA%\Docker\run\dockerInference` socket 檔殘留、無法移除 → Docker Desktop 開機反覆跳「unexpected error」。解法＝關掉所有 Docker 程序 → 刪掉殘留 socket 檔 → 重啟 Docker Desktop。Docker Desktop 起不來又找不到明顯原因時先查這個殘留檔。（2026-07-16）

## 本機驗證環境（家用機，2026-07-10 驗證有效；Phase 1 後改看 platform 文件）

- 前端 preview port **5299**（base `/Auth/`），Steven 自己的 dev 在 5174 勿佔；後端 `npm run start:dev --prefix <根>\Ystravel-AuthService`（3001），開工先確認 3001 跑的是 start:dev 不是舊 dist build。
- **自動化登入**（UInput v-model 吃不到自動 fill）：後端直接 POST `http://localhost:3001/api/auth/auth/login`（e2e-admin）拿 token，寫 localStorage（`ystravel.auth.accessToken`／`refreshToken`／`session`＝user JSON）再導頁；⚠️**token 不要印進 transcript**（會被憑證守則擋），用同源 fetch 或後端取值再灌。
- 色彩模式直接寫 `ystravel.platform.color-mode`=light/dark 後 reload，比點 toggle 可靠。

## Auth 技術現況速記（細節見 [ADR-2026-07-16](../architecture/ADR-2026-07-16-platform-modular-monolith.md)）

- Access token＝RS256＋JWKS（`/.well-known/jwks.json` 可離線驗章），`JwtAuthGuard` 只收 RS256——**別再把 HS256 共用密鑰當現行方案**（舊 CRM-Backend 還是 HS256，該 repo 已棄用）。
- SSO＝自建 OAuth2 式：登入後發一次性 exchange code（綁 appCode+redirectUri、sha256、單次、TTL）換各 app 自己的 token（`auth-exchange.service.ts`）。**沒有 SAML、還不是標準 OIDC provider**（缺 `/authorize`/`id_token`/discovery）；要對外互通才補 OIDC 層。架構轉向後 exchange 機制**保留閒置**（未來第二應用伏筆）。
- 待辦（backlog）：refresh token selector/jti/hash lookup、2FA、登入鎖定。

### NestJS `deleteOutDir`＋`tsBuildInfoFile` 在 dist 外＝殘缺 dist（MODULE_NOT_FOUND）
`nest-cli.json` 開 `deleteOutDir: true` 會在 build 前清空 `dist/`，但 tsconfig `incremental: true` 的 `tsconfig.tsbuildinfo` 若放在 dist **外**（預設 package 根），tsc 仍以為所有檔案都編譯過、只回填有變動的檔——結果 dist 只剩零星檔案，啟動就 `Cannot find module './xxx'`。症狀特徵：typecheck/tests 全綠、只有 `node dist/main` 掛。解法＝tsconfig 補 `"tsBuildInfoFile": "./dist/tsconfig.tsbuildinfo"`，讓 buildinfo 跟 dist 同生共死。（2026-07-17，hr-employee-master 輪踩到）

### Git Bash（Windows）curl -d 帶中文＝寫進 DB 的就是亂碼
Windows 的 Git Bash console codepage 非 UTF-8，`curl -d '{"name":"王小明"}'` 送出的 bytes 已經是壞的——**資料進 DB 就毒了**，不是顯示問題（用 psql 看 length 對不上即可確認）。測試 API 帶中文一律寫成 UTF-8 檔案再 `-d @file.json`。（2026-07-17）

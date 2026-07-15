# 資料庫策略與遷移指南（Database Strategy & Migration）

> **這是「資料庫用什麼、以後怎麼換」的權威文件。** 問到 DB 選型、MySQL/PostgreSQL、換 DB、資料搬遷，看這份就對了。
> 最後更新：2026-07-15

---

## 1. 現況決策（Decision）

| 項目 | 決策 | 備註 |
|---|---|---|
| 正式資料庫 | **MySQL 8** | AuthService、CRM-Backend 皆是（見各自 `prisma/schema.prisma` 的 `provider = "mysql"`） |
| 本地開發 | MySQL / MariaDB | 由 `Ystravel-LocalDocker` 提供 |
| 存取層 | **Prisma ORM** | 不手寫 SQL，用 `prisma.xxx` 查詢；schema 是唯一真相來源 |
| PostgreSQL | **暫不採用，待特定場景再議** | 見下方 §2 |

**權威技術棧清單**：[CRM_AUTH_SERVICE_SD.md](./CRM_AUTH_SERVICE_SD.md)（開頭「建議技術」表：Vue 3、Tailwind、Nuxt UI、NestJS、Prisma、MySQL 8）。

---

## 2. 為什麼是 MySQL，不是 PostgreSQL

- 目前需求（登入、角色權限、CRM 客戶/訂單）都是**標準 CRUD**，MySQL 8 完全夠用，看不到 PostgreSQL 強項（`JSONB`、複雜分析、地理資訊）的著力點。
- 已有實體資產在 MySQL：LocalDocker 環境、AuthService/CRM 兩套 schema + migration + seed 都已落地。
- PostgreSQL 早期曾因評估 **Logto**（第三方登入服務，底層綁 Postgres）而出現在文件裡；後來改自建原生登入（`auth-native-login-v2`），Postgres 一併排除。**它從沒被拿來跟 MySQL 正面比較過。**

### 什麼情況才值得回頭認真評估 PostgreSQL

出現以下**具體訊號**時，再開一份選型 ADR 正式評估，不要憑感覺換：

- CRM 匯入清洗要塞大量半結構化 JSON 欄位（PostgreSQL `JSONB` 明顯較強）
- 要跑複雜客戶分析報表（window function、CTE 遞迴）
- 要做地理/地圖功能（PostGIS）

### MySQL vs PostgreSQL 速查

| 面向 | MySQL | PostgreSQL |
|---|---|---|
| 定位 | 快、簡單、Web 主流 | 功能強、規格嚴謹 |
| 資料型別 | 基本型別為主 | 多：原生 `JSONB`、陣列、範圍、地理 |
| 複雜查詢/分析 | 夠用 | 更強 |
| 字串比對 | 預設**不分**大小寫 | 預設**分**大小寫 ⚠️（見 §3 雷區） |
| 適合 | CRUD 商業後台 | 分析型、JSON 彈性、金融級嚴謹 |

---

## 3. 未來若要換 DB：遷移 checklist

> ⚠️ **最重要的觀念：schema 和 data 是兩件獨立的事。** Prisma 換 `provider` 只解決 schema，**不會搬任何資料**。

**換 DB = 以下四步，缺一不可：**

### Step 1 — Schema：建新 DB 的空表（Prisma 負責）

1. 改 `prisma/schema.prisma` 最上面：`provider = "mysql"` → `"postgresql"`
2. 換 `.env` 的 `DATABASE_URL` 連線字串
3. 跑 `npx prisma migrate deploy`（或重新 `migrate dev`）
4. 結果：新 DB 裡有一堆**空表**，舊資料一筆都還沒過去 → 進 Step 2/3

### Step 2 — 基礎資料：重跑 seed（不用搬）

角色（`SUPER_ADMIN`…）、權限目錄、出廠範本角色本來就是 `seed.ts` 灌的，且冪等。

- 直接 `npx prisma db seed` 重灌一份即可，**不需要從舊 DB 搬**。
- 這是 seed 冪等的隱藏好處：可重新產生的資料不算「要遷移的資料」。

### Step 3 — 使用者資料：真正要搬的部分

真人帳號、CRM 客戶、訂單、群組成員關係等 runtime 累積的資料，seed 產不出來，**只有這部分要做資料遷移**。三種做法：

- **做法 A｜`pgloader`（MySQL→PostgreSQL 業界標準工具）**：一條指令整庫倒過去，自動處理型別對應。最省事，但照 MySQL 原樣搬，可能跟 Prisma 期望型別對不齊要微調。
- **做法 B｜Prisma 搬運腳本（本專案規模最推薦）**：開兩個 PrismaClient（一連舊 MySQL、一連新 PG），`findMany` 讀出 → `create` 寫入。型別安全、關聯順序自控、能順便清資料。**注意外鍵順序**：先公司 → 部門 → 使用者 → 關聯。
  ```ts
  const mysql = new PrismaClient({ datasources: { db: { url: OLD_MYSQL_URL } } });
  const pg    = new PrismaClient({ datasources: { db: { url: NEW_PG_URL } } });
  const users = await mysql.user.findMany({ include: { roles: true } });
  for (const u of users) { await pg.user.create({ data: /* 映射欄位 */ }); }
  ```
- **做法 C｜CSV/JSON 中繼**：舊 DB 匯出 → 新 DB 匯入。土法煉鋼，適合單表少量。

### Step 4 — 驗證：搬完必測（尤其大小寫敏感邏輯）

MySQL→PostgreSQL 會咬人的雷，搬完務必逐項測：

1. **大小寫敏感度（最陰）**：MySQL 字串比對預設不分大小寫，PostgreSQL 預設分。**登入、查 email 的邏輯可能因此壞掉** → 一定要測登入。
2. **`tinyint(1)` vs `boolean`**：MySQL 用 0/1 存布林，PG 是真 `boolean`，型別對應要盯。
3. **自增主鍵**：MySQL `AUTO_INCREMENT` vs PG `sequence`，搬完要把 sequence 下一個值重設到 `max(id)+1`，否則新增撞主鍵。（本專案多用 `cuid()` 字串 id 的表免疫。）
4. **`datetime` vs `timestamptz`**：時區處理不同，跨時區資料要留意。
5. **資料筆數對帳**：每張表 `count` 新舊比對，確認沒漏搬。

**執行原則**：先在**測試環境**完整演練四步、驗證筆數與登入都對，再對正式環境動手。**絕不線上直接改 provider。**

---

## 4. 一句話總結

換 DB = **「migrate 建空表」＋「seed 重灌基礎資料」＋「腳本/pgloader 搬使用者資料」＋「測登入等大小寫敏感邏輯」**。Prisma 讓第一步變一行，後三步是它管不到的工。目前維持 MySQL 是對的，Prisma 保有「以後想換隨時能換」的彈性，所以不必急著現在導入 PostgreSQL。

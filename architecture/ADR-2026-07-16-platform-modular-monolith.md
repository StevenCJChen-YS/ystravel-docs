# ADR：內部系統重建改走單一 monorepo 模組化單體（ystravel-platform）

> 狀態：**已定案（2026-07-16）**
> 本 ADR 取代先前「每個業務領域一對前後端 repo」的多系統路線，並**一併取代 [DATABASE.md](./DATABASE.md) 的 MySQL 決策**（DB 改為 PostgreSQL，見 §5）。
> 平台的最高指導原則＝未來 `ystravel-platform` repo 根目錄的 `CLAUDE.md`（Phase 0 放入）；本 ADR 記錄「為什麼這樣決定」，CLAUDE.md 記錄「怎麼做」。

---

## 1. 背景與問題

- 公司約 200 人，技術團隊 1–3 人（現況實際 1 人＋AI 協作）。
- 舊 EIP（`GInternational-*`）＝人事/IT/行銷大雜燴，功能高度耦合、難維護，已確定重建淘汰。
- 重建原路線＝多 repo 多系統：Auth（AuthService＋AuthPortal）獨立 SSO 中樞已建成，CRM 前後端獨立 repo，下一個 HRM 原本也要再開一對 repo——照此軌跡最終會有 8~10 個 repo。
- 2026-07-16 三份輸入收斂出轉向決策：
  1. claude.ai 架構討論（200 人公司單體 vs 多專案）；
  2. 《系統架構決策建議書》（2026-07）；
  3. 平台 `CLAUDE.md`（最終定案文件）。

### 多 repo 路線已浮現的成本（實證，非理論）

| 成本 | 實例 |
|---|---|
| 跨系統主檔同步 | HRM↔Auth「組織樹 master 在誰、怎麼同步、權限連動延遲」整份討論檔（2026-07-15）都在處理這個 |
| 共用 UI 重複 | AuthPortal `shared/ui` 24 個元件＋theme＋色階，CRM-Frontend 全部要再來一份（實查：CRM 前端當時全是手刻 table，功能重疊、實作各自為政） |
| 維運乘法 | migration 沒 deploy、env 缺 key、port 被吃……每一套系統各踩一次；系統數 × 環境 × 兩台開發機 |

## 2. 決策

**採用單一 monorepo 模組化單體 `ystravel-platform`**：

```
apps/backend/src/          # NestJS 11 模組化單體
├── core/auth/             # 登入、Google OAuth、JWT、Guards、密碼重設、跨應用交換碼
├── core/users/            # 帳號、roles、permissions、groups、audit log
├── hr/                    # 人事（擁有 Company/Department 等組織資料）
├── appraisal/             # 考核
├── crm/                   # 客戶、聯繫紀錄、標籤分眾
├── marketing/  it/
└── integration/cowell/     # 科威 Excel 匯入管線（防腐層）
apps/portal/src/           # Vue3 + NuxtUI 單一入口網站（shared/ui 包裝層 + modules/*）
packages/shared/           # 前後端共用型別、權限代碼常數
```

分離關注點靠**程式碼中的模組邊界**（NestJS module exports 介面、資料主權、lint/code review 擋件），不靠拆專案。模組邊界即未來拆分的 API 邊界（保留「拆分逃生門」）。

### 拆分觸發條件（明訂，避免反覆討論）

同時滿足才把特定模組獨立成服務：①技術團隊 ≥15 人可分組；②某模組流量/擴展需求顯著不均（例如對外服務）；③需截然不同的技術棧或發布節奏。未滿足前，**禁止**為「未來可能需要」引入微服務、message queue、額外資料庫。

## 3. SSO 的處理：併入單體成 core 模組

- **查證事實**：舊 EIP 前後端從未串接 AuthService（全域 grep exchange/JWKS/AuthService 零命中）→ 過渡期「獨立 SSO」實際只有一個 client（新平台），沒有服務對象。
- **決策**：AuthService/AuthPortal「**搬入整理、不重寫**」——後端成為 `core/auth`＋`core/users` 模組；AuthPortal 升格為全公司入口網站（設計系統、共用元件、帳號中心全數保留）。
- **保留**：RS256+JWKS、`AuthExchangeCode` 跨應用交換機制（**閒置但不刪**，未來 ERP／對外應用的 SSO 伏筆）、登入節流、密碼重設、稽核日誌、既有 BDD 測試。
- 未來若出現第二個應用（如對外會員系統），再把 core 抽回獨立 SSO 服務，邊界乾淨時成本可控。

## 4. 資料主權（HRM↔Auth 爭點結案）

- `hr` 模組擁有組織與人事資料（Company/Department/職稱等）；`core` 只擁有身分與權限（帳號憑證、roles/permissions），對組織資料**唯讀**。
- 同一單體、同一顆資料庫 → 原「兩系統主檔同步」問題不存在；原討論檔（my-agent `100_Todo/archive/2026-07-15_HRM-Auth關係討論.md`）已結案，殘餘小題（顯示名稱政策、組織管理頁移交過渡）記於該檔結案節。

## 5. 資料庫：PostgreSQL 取代 MySQL（supersedes DATABASE.md §1–2）

- 決策：**單一 PostgreSQL 資料庫，依模組劃分 schema（core/hr/crm/…）**。
- 時機理由：開發期**無正式資料**，是換 DB 成本最低的一刻；正式上線後再換要搬資料、停機、回歸測試，成本翻數倍。
- 遷移要點（MariaDB/MySQL → PG）：改 Prisma datasource、移除 `@prisma/adapter-mariadb`、migrations 砍掉重建、模糊搜尋補 `mode: "insensitive"`（PG 預設大小寫敏感）。
- [DATABASE.md](./DATABASE.md) 的「為什麼是 MySQL」一節作廢；其「schema≠data」遷移 checklist 思路仍可參考。

## 6. 科威系統整合（新記錄事實）

- 科威＝外包系統（會計/財務/訂單/官網後台），旅行社業界通用，**持續使用、不在重建範圍**。
- **科威沒有 API**：資料靠人工下載 Excel 再匯入。
- `integration/cowell` 防腐層：上傳 Excel → exceljs 解析 → staging 表（原始列＋批次 ID）→ 驗證正規化 → 自然鍵比對（正規化電話＋email＋證號）→ **upsert（禁裸 insert，冪等）** → 匯入報告；部分失敗不整批死、欄位對不上明確報錯、寫 AuditLog。科威資料格式只准出現在此層內部。
- 既有經驗直接適用：CRM 匯入 10 萬列曾把一次性讀入的實作撐爆記憶體，已驗證解法＝exceljs streaming 逐列＋每千列分批寫入（參考碼在舊 `Ystravel-CRM-Backend`）。

## 7. 部署與 CI/CD

Docker Compose 於單台 VPS（遠振）、nginx 同網域反代（`/` 前端、`/api` 後端）；GitHub Flow＋GitHub Actions＋GHCR；migration 用 `prisma migrate deploy`。

## 8. 分階段路線圖

| 階段 | 內容 |
|---|---|
| 0 | monorepo 骨架＋CI/CD＋Docker Compose＋PostgreSQL 轉換 |
| 1 | 整併 AuthService/AuthPortal 為 core 模組與入口網站（功能已完成，重點是結構整理） |
| 2 | HR＋考核模組（重寫舊 EIP 最痛處，驗證架構） |
| 3 | 科威 Excel 匯入管線＋CRM 模組 |
| 4 | 行銷、IT 等陸續以模組加入，舊 EIP 退役 |

## 9. 對既有 repo／文件的影響

| 對象 | 處置 |
|---|---|
| Ystravel-AuthService / AuthPortal | Phase 1 整併入 platform 後 GitHub **Archive**（唯讀封存，不刪） |
| Ystravel-CRM-Backend / Frontend | **已棄用（2026-07-16）**：不搬入；匯入 streaming 邏輯留作 cowell 參考，取完後 Archive |
| GInternational-*（舊 EIP） | 服役至 Phase 4 退役 |
| docs repo（本庫） | **保留獨立**：SDLC 流程、設計系統、功能文件的家；platform 的 `docs/` 只放平台專屬 ADR |
| [DATABASE.md](./DATABASE.md) | 標 superseded 指向本 ADR |
| [design-system/foundation.md](../design-system/foundation.md) | 範式全部沿用；「業務頁禁直用 `U*`、一律 `App*` 包裝層」新規於 Phase 1 整併時補章節（與 AuthPortal 現況衝突，屆時處理） |
| development-process/ | 流程不變，僅 repo 名稱引用隨整併更新 |

## 10. 已知風險與對策

- **模組邊界靠紀律會失守** → CLAUDE.md 鐵則列為 code review 擋件標準；後續在 CI 加 eslint import 邊界規則。
- **單體再變 EIP 2.0？** 舊 EIP 的病因是「無邊界＋自管帳號」，不是「部署在一起」；本方案模組有邊界、身分權限集中 core。
- **PG 轉換翻車** → 趁無正式資料期執行；migrations 重建＋seed 重灌，`mode:"insensitive"` 全面檢查模糊搜尋。
- **併掉獨立 SSO 後第二應用出現** → AuthExchangeCode 保留閒置，core 邊界乾淨，屆時抽回獨立服務。

# SA: 角色 CRUD ＋ 權限分配（Phase 1）

| 項目 | 內容 |
|---|---|
| 對應 PRD | ./prd.md |
| 狀態 | 已確認（2026-07-14） |
| 建立日期 | 2026-07-14 |

## 0. 白話摘要

把「角色」從唯讀變成可管理：AuthService 的 `/roles` 補齊建立/編輯/刪除＋「整組替換角色權限」端點；AuthPortal 角色頁從唯讀表格改成完整 CRUD，並新增「角色權限設定」獨立整頁（勾選矩陣）。**不改資料庫 schema**——Role/Permission/RolePermission 三張表本來就齊，只是沒有寫入端點。最大風險：①寫錯權限＝提權（全程 audit＋系統角色保護矩陣防）②`isSystem` 欄位預設值是 `true`，新角色建立時必須明寫 `false`，忘了寫＝管理員建的角色全變成不可刪的系統角色。

## 1. 系統邊界

- **會動**：Ystravel-AuthService（roles 端點、permissions guard、seed）、Ystravel-AuthPortal（角色頁改造、新權限設定頁、共用 PermissionMatrix 元件）
- **不動**：schema（無 migration）、`permission-catalog.ts`（權限碼不增不改）、token 結構與權限彙總邏輯（沿用「角色權限 ∪ ALLOW − DENY」寫入 access token）、CRM/EIP
- 既有架構：token 流見 [CRM_AUTH_SERVICE_SA.md](../../architecture/CRM_AUTH_SERVICE_SA.md)；角色指派給使用者（UserRole）與超管防呆屬 [auth-org-and-permissions](../auth-org-and-permissions/)，本輪沿用不動

## 2. 角色與權限

- 不新增權限代碼。所有角色 mutation 沿用 **`AUTH.ROLE.MANAGE`**（GET /roles 既有同權限）
- `GET /permissions` 放寬為 **any-of（`AUTH.PERMISSION.READ` 或 `AUTH.ROLE.MANAGE`）**：權限設定頁要抓完整目錄，角色管理員不需另掛一顆讀取權限
- 有效權限計算不動：改角色權限後，掛該角色的使用者在**下次 token refresh（≤15 分）**生效

## 3. 操作流程（正常路徑）

1. 管理員開角色管理頁 → 「新增角色」→ 選系統範圍、填代碼/名稱/說明 → 建立（`isSystem=false`）
2. 列表點「權限設定」→ 進獨立整頁 → 系統→模組分卡勾選（卡片全選/清除）→ 儲存 → 單筆 audit 記 added/removed
3. 編輯角色：改名稱/說明/啟停（code 與系統範圍鎖定）
4. 刪除角色：確認對話框 → 後端檢查（無人掛著＋非系統角色）→ 刪除＋快照 audit
5. 測試帳號掛上該角色 → 下次 token refresh 拿到新權限

## 4. 狀態轉換

- 角色：啟用 ⇄ 停用（超管角色不可停用）＋ 刪除（終態；僅一般角色且無人掛著）
- 權限指派：無狀態機，一份 RolePermission 集合整組替換

## 5. 例外流程（對應 Example Mapping 負向情境）

- code 重複 409／格式錯誤 400／未知系統或權限碼 400
- 編輯帶不同 code → 400（前端 disabled、後端仍擋）
- 刪除：有人掛著 409（含人數）；系統角色 400
- 超管角色：停用 400、改權限 400（唯讀）
- 無 `AUTH.ROLE.MANAGE` → 403（既有 guard）
- 併發：低頻內部操作，最後寫入為準（同 org 管理，不做樂觀鎖）

## 6. 資料流與敏感資料

- 全部資料留在 AuthService DB；角色/權限對應屬 Internal 級權限治理資料，不含個資/金流，無匯出、無 AI
- 風險放大器：權限勾錯＝提權。防線＝系統角色保護矩陣（後端強制）＋全程 audit＋刪除/停用防呆

## 7. 與外部系統的互動

無新增外部互動；無新套件。

## 8. 風險點與人工審核點（Steven 必看）

1. **`isSystem @default(true)` 陷阱**：POST /roles 必須明寫 `isSystem: false`，BDD 有專門斷言
2. **seed 修正的部署行為**：改成「不存在才建、權限僅首次寫入」後，既有環境的系統角色不再被 seed 覆蓋＝管理員的調整會保留；但也表示**之後 catalog 加新權限給既有系統角色要靠畫面補勾**（或一次性腳本），不再自動下發——已知取捨，接受
3. **guard 放寬**：`GET /permissions` 對 `AUTH.ROLE.MANAGE` 持有者開放讀取（Internal 級中繼資料，低風險）
4. **生效延遲 ≤15 分**：驗收時別把「還沒 refresh」誤判成儲存失敗（toast 有標註）

## 9. 待決策事項

- [ ] 無——8 題已於 Example Mapping 全部拍板

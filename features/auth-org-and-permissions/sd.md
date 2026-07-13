# SD: Auth 組織結構與權限模型完備化

| 項目 | 內容 |
|---|---|
| 對應 SA | ./sa.md |
| 狀態 | 已確認（2026-07-10） |
| 建立日期 | 2026-07-10 |

## 0. 白話摘要

Schema 加四張表（Company/Department/Group/GroupMember）、UserRole 補兩欄、AuthUser 增改欄位；AuthService 加 org 與 groups 兩個新模組；權限彙總補「過期角色過濾」，token 加 roles 欄位讓 guard 認得 SUPER_ADMIN 直接放行。遷移採**兩段式**：先加新結構與回填資料（對應表 Steven 人工確認），確認無誤後第二段才鎖 email 必填並刪舊字串欄位——任何一步出錯都可回頭。無新套件、無新外部依賴。

## 1. 系統架構

- **AuthService**（NestJS + Prisma）：
  - 新模組 `src/org/`（公司/部門）、`src/groups/`（群組/成員）
  - 修改：`src/users/`（建立/停用規則）、`src/roles/` 或使用者角色指派端點（有效期/超管規則）、`src/auth/`（權限彙總過濾過期角色、token roles claim、PermissionsGuard 超管放行）、`src/permissions/permission-catalog.ts`（新代碼與 AUTH_HR 角色）
- **AuthPortal**（Vue3 + NuxtUI）：新頁 `OrgManagementPage`、`GroupManagementPage`；修改 `AdminUsersPage`（部門下拉、note、停用按鈕權限、角色指派對話框加有效期）

## 2. API 設計（皆掛 `api/auth` 前綴，管理端點）

| Method | Path | 說明 | 權限 |
|---|---|---|---|
| GET | /admin/org/companies | 公司+部門樹（含停用，供管理頁） | AUTH.ORG.MANAGE 或 AUTH.USER.READ（下拉用唯讀輸出） |
| POST/PATCH | /admin/org/companies(/:id) | 建/改/停用/排序公司 | AUTH.ORG.MANAGE |
| POST/PATCH | /admin/org/departments(/:id) | 建/改/停用部門（停用前檢查無成員） | AUTH.ORG.MANAGE |
| PATCH | /admin/org/companies/reorder | 重排公司順序（body `ids[]`，交易重編 order 0,1,2…；供拖曳＋依名稱排序） | AUTH.ORG.MANAGE |
| PATCH | /admin/org/departments/reorder | 重排某公司內部門順序（body `companyId,ids[]`，同層＝同公司） | AUTH.ORG.MANAGE |
| PATCH | /admin/options/reorder | 重排某類別內選項順序（body `category,ids[]`，同層＝同類別） | AUTH.ORG.MANAGE |
| GET | /admin/groups | 群組清單 | AUTH.GROUP.MANAGE |
| POST/PATCH | /admin/groups(/:id) | 建/改/停用群組 | AUTH.GROUP.MANAGE |
| GET | /admin/groups/:id/members | 成員名單（含停用標示） | AUTH.GROUP.MANAGE |
| POST/DELETE | /admin/groups/:id/members(/:userId) | 加/移除成員 | AUTH.GROUP.MANAGE |
| POST | /users | 建帳號（email+departmentId 必填）＋寄邀請 | AUTH.USER.MANAGE **或** AUTH.USER.ONBOARD |
| POST | /users/:id/deactivate | 停用（ONBOARD 呼叫時服務層檢查對象非管理者） | MANAGE 或 ONBOARD |
| POST | /users/:id/resend-invite | 重寄邀請（既有，權限加 ONBOARD） | MANAGE 或 ONBOARD |
| PUT | /users/:id | 編輯使用者（含 departmentId/note/avatar） | AUTH.USER.MANAGE（ONBOARD 不可） |
| POST/DELETE | /users/:id/roles | 指派/移除角色（body 含 expiresAt?；涉 AUTH_SUPER_ADMIN 時服務層限超管操作＋最後一位防呆） | AUTH.ROLE.MANAGE |

Request/Response 型別實作時寫 `api-spec.md`（若前後端需要對齊）。

## 3. DB Schema 異動（兩段式 migration）

**第一段 `20260710_org_groups_role_expiry`**：
```prisma
model Company    { id, name @unique, order Int, isActive, timestamps }
model Department { id, companyId FK, name, order Int, isActive, timestamps, @@unique([companyId, name]) }
model Group      { id, code @unique, name, description?, isActive, timestamps }
model GroupMember{ groupId+userId @@id, addedById?, createdAt }
AuthUser  += departmentId? FK, avatarUrl?, note?          // 舊 department 字串先保留
UserRole  += assignedById?, expiresAt?
```
回填腳本（repo 內一次性 script，非 migration）：舊 department 字串 → 產出對應表 → **Steven 確認** → 建 Department 列並回填 departmentId。

**第二段 `20260710_user_email_required_drop_department`**（回填確認後）：email 改 NOT NULL、刪 `department` 字串欄。前置：email 空值盤點為零（或已逐筆處理）。

## 4. Service / Module 分層

沿用既有 AuthService 模組 pattern（controller + service + fake-prisma 測試檔，參照 `src/users/`、`src/auth/password-reset.*`）。不引入新分層、不加套件。

## 5. 認證與授權方式

- 沿用 `JwtAuthGuard` + `PermissionsGuard` + `@RequirePermissions()`
- 新權限代碼：`AUTH.ORG.MANAGE`、`AUTH.GROUP.MANAGE`、`AUTH.USER.ONBOARD`（語意：MANAGE ⊇ ONBOARD，端點檢查 any-of）
- **SUPER_ADMIN 放行**：access token payload 增加 `roles: string[]`（角色 code 清單）；`PermissionsGuard` 首行：`roles` 含 `AUTH_SUPER_ADMIN` → 直接通過。行為寫死在 guard，資料庫無萬用權限
- **過期角色**：權限彙總（login/refresh 的 aggregation）過濾 `expiresAt <= now` 的 UserRole；生效延遲上限＝access token 15 分鐘窗口（既有決策）

## 6. 安全與稽核（依 baseline §6.3）

| 欄位 | 內容 |
|---|---|
| 敏感欄位保護 | 無新增祕密類欄位；email 屬 Restricted，僅後台顯示（無匯出）。avatarUrl/note 為 Internal |
| Audit log | ORG_COMPANY_/ORG_DEPARTMENT_ CREATED/UPDATED/DISABLED/**REORDERED**；OPTION_CREATED/UPDATED/DISABLED/**REORDERED**；GROUP_CREATED/UPDATED/MEMBER_ADDED/MEMBER_REMOVED；ROLE_ASSIGNED/ROLE_REMOVED（含 expiresAt）；**SUPER_ADMIN_ASSIGNED/REMOVED（必記，含操作者與對象）**；USER_DEACTIVATED（含操作者，HR 停用可追溯） |
| 備份 / 還原 | 不影響既有策略；遷移前手動備份 DB 一次（兩段式之間可回復） |
| 錯誤處理與告警 | 沿用既有 error filter；「最後一位超管」檢查在交易內執行防併發歸零 |
| 測試資料策略 | 全部用合成姓名/email（@example.test），沿用 fake-prisma pattern，不碰真實資料 |

## 7. 效能與非功能需求

內部低頻管理操作；公司數個、部門數十、使用者數百量級——沿用既有標準，無需分頁以外的特殊處理（使用者列表分頁既有）。

## 8. 錯誤碼

| 情境 | HTTP Status | 說明 |
|---|---|---|
| 欄位驗證失敗（缺 email/部門） | 400 | class-validator 既有格式 |
| 無權限 / HR 停用管理者 / 非超管指派超管 | 403 | 統一權限不足語意 |
| 對象不存在 | 404 | |
| 部門重名 / email 重複 / 群組 code 重複 | 409 | 明確指出欄位 |
| 停用有成員的部門 / 移除最後一位超管 | 422 | 業務規則阻擋，訊息含解法提示 |

## 9. 測試策略（依 07-testing-strategy.md）

- **BDD（jest-cucumber）**：`src/org/org-and-permissions.feature` 全 23 情境，steps 走 fake-prisma
- **單元測試**：權限彙總的過期過濾（邊界：剛好到期/未設到期）；PermissionsGuard 超管放行；「最後一位超管」交易內防呆；ONBOARD 停用對象檢查
- **既有測試不退步**：全套 12 suites / 90+ tests 綠
- **遷移驗證**：回填腳本先 dry-run 輸出對應表（Steven 審核）；本機 DB 實跑後抽查
- **手動 smoke**：超管/AUTH_ADMIN/AUTH_HR 三種帳號各走一輪畫面（建部門、建帳號、群組、暫時角色、被拒場景）
- **前端**：typecheck + build 綠；UI 第一版完成後列入 UI 待美化清單

## 10. 待決策事項

無——需要 Steven 拍板的問題已在 PRD/Example Mapping 階段全部解決。兩個純技術選擇已定（知悉即可）：①token 加 `roles` claim（含超管判斷，順帶讓下游 app 未來可讀角色）②遷移兩段式（安全優先，多一次 migration 的代價換可回頭）。

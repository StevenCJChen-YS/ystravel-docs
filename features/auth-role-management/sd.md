# SD: 角色 CRUD ＋ 權限分配（Phase 1）

| 項目 | 內容 |
|---|---|
| 對應 SA | ./sa.md |
| 狀態 | 已確認（2026-07-14） |
| 建立日期 | 2026-07-14 |

## 0. 白話摘要

後端擴充既有 `src/roles/roles.controller.ts`（不開新模組）：加 5 支端點（查單筆/建/改/刪/整組換權限），沿用 repo 慣例（plain type request＋手動驗證＋直接注入 Prisma＋controller 內 audit helper）。前端照「使用者頁黃金範式」把角色頁改成 CRUD，權限設定是**新獨立頁**＋新共用元件 `PermissionMatrix`（Phase 2 使用者權限微調可重用）。無 migration、無新套件。

## 1. 系統架構

- **AuthService**：
  - 修改 `src/roles/roles.controller.ts`（5 支新端點＋`serializeRole()` 統一回傳形狀＋audit helper）
  - 修改 `src/permissions/permissions.controller.ts`（GET /permissions guard 放寬 any-of）
  - 修改 `prisma/seed.ts`（角色不存在才建、權限僅首次寫入）
  - 新增 `src/roles/role-management.feature` ＋ `.spec.ts` / `.steps.ts` / `.fake-prisma.ts`（BDD 四件套，仿 `src/org/`）
- **AuthPortal**：
  - 修改 `src/pages/admin/AdminRolesPage.vue`（唯讀 → CRUD；移除 detail modal）
  - 新增 `src/pages/admin/RolePermissionsPage.vue`（權限設定整頁）
  - 新增 `src/entities/permission/PermissionMatrix.vue`（共用勾選矩陣）
  - 修改 `src/services/auth-api.ts`（helpers＋錯誤翻譯）、`src/router/index.ts`（新 route）、`src/shared/audit-log-i18n.ts`（5 個新動作）

## 2. API 設計（詳細 request/response 見 ./api-spec.md）

| Method | Path | 說明 | 權限 |
|---|---|---|---|
| GET | /roles | 角色清單（既有，不動） | AUTH.ROLE.MANAGE |
| GET | /roles/:id | 單一角色（權限頁 deep-load 用） | AUTH.ROLE.MANAGE |
| POST | /roles | 建立角色（**明寫 `isSystem:false`**） | AUTH.ROLE.MANAGE |
| PATCH | /roles/:id | 改名稱/說明/啟停（code 不可改；超管不可停用） | AUTH.ROLE.MANAGE |
| DELETE | /roles/:id | 刪除（系統角色 400；有人掛著 409；cascade 清 RolePermission；audit 留快照） | AUTH.ROLE.MANAGE |
| PUT | /roles/:id/permissions | 整組替換角色權限（**請求全量、實作 diff、audit 單筆**；超管 400 唯讀） | AUTH.ROLE.MANAGE |
| GET | /permissions | 既有，guard 放寬 any-of（PERMISSION.READ 或 ROLE.MANAGE） | 見左 |

**PUT permissions 的三層設計**（仿 users `PUT :id/roles` 的 diff 套用）：
- 請求語意＝全量替換（前端整頁一次送完整集合）
- 實作＝diff（只 deleteMany 被移除、createMany 新增，不動未變的列），`$transaction([deleteMany, createMany, auditLog.create])`
- audit＝**單筆** `ROLE_PERMISSIONS_UPDATED`，diff `{ roleCode, roleName, added[], removed[], total }`（不學 group batch 逐筆——一次存 30 顆權限會灌爆稽核）；無變更＝不寫 audit
- 回傳更新後完整角色，前端以伺服器真相重設 dirty 基準

## 3. DB Schema 異動

**無。** Role / Permission / RolePermission（複合 PK＋cascade）三張表既有即足夠。角色無 order 欄（順序管理明確 out of scope）。

## 4. Service / Module 分層

沿用 repo 慣例：thin controller 直接注入 `PrismaService`、plain `type XxxRequest`＋手動驗證（`BadRequestException`/`ConflictException`/`NotFoundException`）、private `audit()` helper（`resourceType: 'role'`）。`SUPER_ADMIN_ROLE_CODE` 常數從 `src/auth/permissions.guard` import，不另複製字串。

## 5. 認證與授權方式

- 全部 mutation 沿用 `JwtAuthGuard + PermissionsGuard + @RequirePermissions('AUTH.ROLE.MANAGE')`（可提到 class 層，仿 groups）
- `GET /permissions` 改 `@RequireAnyPermission('AUTH.PERMISSION.READ', 'AUTH.ROLE.MANAGE')`（decorator 既有）
- 系統角色保護矩陣（後端強制、前端只是鏡射 disabled UI）：見 example-mapping Rule 4

## 6. 安全與稽核（依 baseline §6.3）

| 欄位 | 內容 |
|---|---|
| 敏感欄位保護 | 角色/權限對應＝Internal 權限治理資料；無個資/金流/祕密欄位；不提供匯出 |
| Audit log | `ROLE_CREATED`（code/name/appCode/appName）、`ROLE_UPDATED`/`ROLE_DISABLED`、`ROLE_DELETED`（**刪前快照** code/name/appCode/permissionCount——audit viewer 不回查）、`ROLE_PERMISSIONS_UPDATED`（added/removed/total）。diff 存人可讀標籤，沿用既有慣例 |
| 備份 / 還原 | 無 schema 變更，不影響既有策略 |
| 錯誤處理與告警 | 沿用既有 error filter；防呆全在交易前檢查、寫入走交易 |
| 測試資料策略 | fake-prisma 合成資料，不碰真實 DB |

## 7. 效能與非功能需求

內部低頻管理操作；權限目錄數十顆、角色數十個量級，整頁載入一次抓全目錄無壓力。無特殊需求。

## 8. 錯誤碼

| 情境 | HTTP | 訊息（英文原文，前端翻譯） |
|---|---|---|
| code 缺/格式錯 | 400 | `Role code must contain only uppercase letters, numbers, and underscores.` |
| name 缺 | 400 | `name is required.` |
| 未知系統代碼 | 400 | `Unknown application code.` |
| code 重複 | 409 | `Role code already exists.` |
| 編輯帶不同 code | 400 | `Role code cannot be changed.` |
| 停用超管角色 | 400 | `Super admin role cannot be disabled.` |
| 刪系統角色 | 400 | `System roles cannot be deleted.` |
| 刪除有人掛著 | 409 | `Role still has ${n} assigned user(s). Reassign them first.` |
| 改超管權限 | 400 | `Super admin role permissions are read-only.` |
| 未知權限碼 | 400 | `Unknown permission codes: X, Y.` |
| 角色不存在 | 404 | `Role not found.` |
| 無權限 | 403 | 既有 guard 語意 |

## 9. 前端設計

### AdminRolesPage（唯讀 → CRUD）
- **移除**唯讀 detail modal（權限頁取代，避免兩處真相）
- 工具列 `ToolbarButton`「新增角色」；操作欄三顆 ghost icon 鈕＋tooltip：權限設定（`i-lucide-shield-check` → `/admin/roles/:id/permissions`）、編輯、刪除（isSystem＝disabled＋「系統角色不可刪除」）
- 建立/編輯 modal：`useEditModal` ＋ `FormModal`（`sm:max-w-md`），欄位＝系統範圍 `AppSelect`（編輯 disabled；清單從 /permissions 導出）、代碼 `AppInput`（mono、自動大寫、編輯 disabled）、名稱、說明、啟用 `USwitch`（僅編輯；超管隱藏）；valibot schema
- 刪除：`useConfirm`；`userCount>0` 前端先擋提示改派，後端 409 為真防線

### RolePermissionsPage（新頁）
- Route `/admin/roles/:id/permissions`，meta permission `AUTH.ROLE.MANAGE`；`UBreadcrumb` 角色管理 → {角色名}
- `Promise.all([fetchRole, fetchPermissions])`；頁首卡＝角色名＋code＋badges＋「已選 n/共 m」＋儲存鈕（dirty 才亮）；`<sm` sticky 底部儲存列
- **搜尋＋批次列**（07-14 補）：搜尋只過濾「顯示」（已勾但被濾掉仍在 selected、儲存送完整集合）；「全部選取/全部清除」只作用在目前顯示的權限（比照 Imasphere）
- 範圍過濾：指定系統角色只給該系統權限；平台共用給全部
- 超管：info alert「系統寫死全權，僅供檢視」＋matrix readonly＋無儲存鈕
- **dirty＋離開防護**（portal 首見範式）：`onBeforeRouteLeave`＋`useConfirm`；`beforeunload` mount 掛/unmount 拆
- 儲存 toast 標明「下次 token 更新後生效（≤15 分）」

### PermissionMatrix（新共用元件，`src/entities/permission/`）
- Props：`permissions`、`selected`（`v-model:selected`）、`readonly?`、`resourceLabels?`
- **appCode→資源(resource) 分組**（07-14 拍板，原按 module——AUTH 全掛 admin 模組會擠一卡）；資源卡 `grid lg:grid-cols-2 2xl:grid-cols-3`＋子項 `min-w-0`（foundation §11 雷）；卡頭＝資源中文名＋已選計數＋全選/清除；卡身 `UCheckbox` 一權限一列（**中文動作短名**（檢視/管理/入職作業…）＋code mono 小字＋description）；`<sm` 單欄
- 資源/動作中文對照表放元件內（`DEFAULT_RESOURCE_LABELS`/`ACTION_LABELS`），catalog 加新資源時記得補中文（沒對到顯示原值）
- 遵守 foundation §10/§11（字重、toast/confirm、深表頭風格慣例）

### 權限模型定調（07-14 拍板）
- **維持粗粒度混合**：每資源預設「檢視(read)／管理(manage)」兩檔（USER 另有 onboard），真有「只能改不能刪」需求的資源才在 catalog 加 create/update/delete 顆粒（UI 的 ACTION_LABELS 已預留），**不做全面 CRUD 細分**（權限數×4、端點逐支重對、勾選負擔重）。與 Imasphere／GitHub 同模式。

## 10. Seed 修正（決策 8）

`prisma/seed.ts`：角色改「`findUnique` 不存在才 `create`」（不再 upsert 蓋 name/description）；`rolePermission.createMany` 移進「首次建立」分支。**取捨**：之後 catalog 加新權限給既有系統角色，不再由 seed 自動下發，要在畫面補勾（或一次性腳本）——已知並接受。

## 11. 測試策略（依 07-testing-strategy.md）

- **BDD（jest-cucumber）**：`src/roles/role-management.feature` 16 情境（清單見 ./test-plan.md），steps 走 fake-prisma
- **既有測試不退步**：全套既有 suites 綠
- **前端**：`npm run build`（含 vue-tsc）綠；UI 第一版列入 UI 待美化清單
- **手動 smoke**：見 ./test-plan.md §2

## 12. 待決策事項

無——Example Mapping 已全部拍板。

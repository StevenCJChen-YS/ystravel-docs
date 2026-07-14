# Test Plan: auth-role-management（Phase 1）

> 對應 `./sd.md` §11。無 CI，合併前手動跑：AuthService `npm run lint && npm run build && npm test`；AuthPortal `npm run build`（含 vue-tsc typecheck）。

## 1. BDD 情境（AuthService `src/roles/role-management.feature`，16 條）

| # | 情境 | 預期 |
|---|---|---|
| 1 | 管理員建立角色（全系統/指定系統） | 201、**`isSystem=false`**、audit `ROLE_CREATED` |
| 2 | 建立時 code 重複 | 409 |
| 3 | 建立時 code 格式不符（小寫/空白） | 400 |
| 4 | 建立時指定不存在的系統代碼 | 400 |
| 5 | 編輯名稱/說明/停用 | 200、audit `ROLE_UPDATED`/`ROLE_DISABLED` |
| 6 | 編輯時嘗試改 code | 400 |
| 7 | 停用 SUPER_ADMIN | 400 |
| 8 | 刪除無使用者的一般角色 | 200、audit diff 含刪前快照 code/name |
| 9 | 刪除仍有使用者的角色 | 409、訊息含人數 |
| 10 | 刪除系統角色 | 400 |
| 11 | 整組替換權限 | 只 deleteMany 被移除、只 createMany 新增（斷言參數）、**單筆** audit 含 added/removed/total |
| 12 | 權限清空（空陣列） | 200 允許 |
| 13 | 未知權限碼 | 400、訊息列出缺碼 |
| 14 | 修改 SUPER_ADMIN 權限 | 400 |
| 15 | 無 AUTH.ROLE.MANAGE 呼叫任一 mutation | 403 |
| 16 | 持 AUTH.ROLE.MANAGE（無 PERMISSION.READ）GET /permissions | 200（any-of 放寬） |

另：既有測試全套不退步。

## 2. 手動驗證（對應 PRD §5 AC，Steven 畫面操作）

1. **建角色**：新增 TEST_VIEWER（全系統）→ 列表出現、稽核頁顯示「建立角色」
2. **表單防呆**：重複 code／小寫 code → 表單顯示中文錯誤
3. **權限整頁**：進 TEST_VIEWER 權限設定 → 勾數項＋一張卡全選 → **不儲存**點側欄別頁 → 出現未儲存警告；儲存 → toast、重新整理後勾選保留；稽核頁顯示「調整角色權限：新增 X 項」
4. **有效權限生效**：測試帳號掛 TEST_VIEWER → 給角色加 `AUTH.PERMISSION.READ` → 測試帳號**重新登入（或等 token refresh，≤15 分）** → 權限目錄選單/頁面可進。⚠️ 生效前別誤判為儲存失敗（toast 有標註生效時點）
5. **刪除防呆**：TEST_VIEWER 指派給一人 → 刪除被 409 擋＋人數訊息；移除指派 → 刪除成功、稽核顯示快照
6. **系統角色保護（2026-07-15 收斂後）**：SUPER_ADMIN——權限頁唯讀 alert＋無儲存鈕；編輯 modal 無停用開關；刪除鈕 disabled。AUTH_ADMIN 等出廠角色——已降級一般角色，可改可刪（無人掛著時）
7. **RWD**：375px 手機寬——矩陣單欄、sticky 底部儲存列、篩選正常
8. **權限矩陣範圍**：指定 CRM 的角色只看到 CRM 權限；全系統角色看到全部系統分區

## 3. Seed 修正驗證

- 本機對既有 DB 重跑 `npx prisma db seed`：既有角色的名稱/權限**不被還原**（先在畫面改一筆再跑 seed 驗證）
- 清空 DB 重建：角色與權限照 catalog 正常種入（首次建立路徑不退步）
- **2026-07-15 收斂驗證**：對舊 DB 重跑 seed → `AUTH_SUPER_ADMIN` 改名為 `SUPER_ADMIN`（不多建新角色、使用者指派不掉）；AUTH_ADMIN/AUTH_HR/CRM_* 的 `isSystem` 變 `false`（角色頁「系統」標籤消失、可編輯可刪）。⚠️ 跑完要**重新登入**（舊 token 還帶舊代碼，超管判斷會失效到 refresh 為止）

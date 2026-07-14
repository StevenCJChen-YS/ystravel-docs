# API Spec: auth-role-management（Phase 1）

> 對應 `./sd.md` §2。所有訊息字串為英文（前端 `auth-api.ts` 翻譯表轉繁中）。全部端點掛 `api/auth` 前綴、`JwtAuthGuard + PermissionsGuard`，權限 `AUTH.ROLE.MANAGE`（除另註明）。

## 角色回傳形狀（所有端點共用 `serializeRole`）

```jsonc
{
  "id": "uuid",
  "code": "CRM_SUPPORT_LEAD",
  "name": "客服主管",
  "description": "客服團隊管理與報表",
  "appCode": "CRM",          // null = 平台共用
  "appName": "Ystravel CRM", // null = 平台共用
  "isSystem": false,
  "isActive": true,
  "userCount": 3,
  "permissionCount": 5,
  "permissions": [ { "id": "…", "code": "CRM.CUSTOMER.READ", "name": "…", "module": "customer", "resource": "customer", "action": "read", "appCode": "CRM" } ]
}
```

## GET /roles（既有，不動）

`?app=` 可選過濾。回 `{ "data": [Role...], "meta": { "total": n } }`。

## GET /roles/:id（新）

```jsonc
// Response 200
{ "data": Role }

// Response 404
{ "message": "Role not found." }
```

## POST /roles

```jsonc
// Request
{ "code": "CRM_SUPPORT_LEAD", "name": "客服主管", "description": "…", "appCode": "CRM" }
// appCode 省略或 null = 平台共用

// Response 201
{ "data": Role }   // isSystem 一律 false
```

| 錯誤 | HTTP | 訊息 |
|---|---|---|
| code 缺/格式錯（非 `^[A-Z0-9_]+$`） | 400 | `Role code must contain only uppercase letters, numbers, and underscores.` |
| name 缺 | 400 | `name is required.` |
| 未知系統代碼 | 400 | `Unknown application code.` |
| code 重複（全域唯一） | 409 | `Role code already exists.` |

- audit：`ROLE_CREATED`，diff `{ code, name, appCode, appName }`

## PATCH /roles/:id

```jsonc
// Request（皆可選；description 傳 null = 清空）
{ "name": "…", "description": "…", "isActive": false }

// Response 200
{ "data": Role }
```

| 錯誤 | HTTP | 訊息 |
|---|---|---|
| 帶不同 code | 400 | `Role code cannot be changed.` |
| 超管角色＋`isActive:false` | 400 | `Super admin role cannot be disabled.` |
| 不存在 | 404 | `Role not found.` |

- audit：`isActive === false` → `ROLE_DISABLED`，否則 `ROLE_UPDATED`

## DELETE /roles/:id（200）

```jsonc
// Response 200
{ "message": "Role deleted." }
```

| 錯誤 | HTTP | 訊息 |
|---|---|---|
| 系統角色 | 400 | `System roles cannot be deleted.` |
| 還有使用者掛著 | 409 | `Role still has ${n} assigned user(s). Reassign them first.` |
| 不存在 | 404 | `Role not found.` |

- RolePermission 隨角色 cascade 刪除
- audit：`ROLE_DELETED`，diff 為**刪前快照** `{ code, name, appCode, permissionCount }`

## PUT /roles/:id/permissions

整組替換角色權限。請求＝全量集合；實作＝diff 套用；audit＝單筆。

```jsonc
// Request（空陣列合法 = 清空全部權限）
{ "permissionCodes": ["CRM.CUSTOMER.READ", "CRM.ORDER.READ"] }

// Response 200
{ "data": Role }   // 更新後完整角色，前端以此重設 dirty 基準
```

| 錯誤 | HTTP | 訊息 |
|---|---|---|
| 非陣列 | 400 | `permissionCodes must be an array.` |
| 超管角色 | 400 | `Super admin role permissions are read-only.` |
| 未知權限碼 | 400 | `Unknown permission codes: X, Y.` |
| 不存在 | 404 | `Role not found.` |

- `$transaction([deleteMany(removed), createMany(added), auditLog.create])`；無變更＝不寫 audit、直接回現狀
- audit：`ROLE_PERMISSIONS_UPDATED`，diff `{ roleCode, roleName, added: string[], removed: string[], total: n }`

## GET /permissions（既有，guard 放寬）

`@RequirePermissions('AUTH.PERMISSION.READ')` → `@RequireAnyPermission('AUTH.PERMISSION.READ', 'AUTH.ROLE.MANAGE')`。

理由：權限設定頁需要完整權限目錄，角色管理員不需另掛讀取權限。資料屬 Internal 級權限中繼資料，風險低（安全審視已知悉）。

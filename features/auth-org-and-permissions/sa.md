# SA: Auth 組織結構與權限模型完備化

| 項目 | 內容 |
|---|---|
| 對應 PRD | ./prd.md |
| 狀態 | 已確認（2026-07-10） |
| 建立日期 | 2026-07-10 |

## 0. 白話摘要

這輪在 AuthService 加四樣東西：公司→部門組織表、群組名單表、角色有效期、SUPER_ADMIN 程式級放行；並改 AuthUser 欄位（email 必填、部門改結構化、加 avatar/note）。後台（AuthPortal）新增組織管理頁、群組管理頁，並改使用者管理頁。**最大風險是兩個**：①權限彙總邏輯是全系統授權的心臟，SUPER_ADMIN 放行與角色到期都動到它，測試必須全綠且涵蓋邊界；②既有使用者的部門字串要遷移成結構化資料，遷移對應表需 Steven 人工確認。

## 1. 系統邊界

- **會動**：Ystravel-AuthService（schema、API、權限彙總、seed catalog）、Ystravel-AuthPortal（後台三頁）
- **不動**：CRM 前後端（未來輪才消費組織資料）、EIP（只讀它的舊資料做遷移對照，不寫）
- 既有架構文件：系統邊界與 token 流見 [CRM_AUTH_SERVICE_SA.md](../../architecture/CRM_AUTH_SERVICE_SA.md)／[CRM_AUTH_SERVICE_SD.md](../../architecture/CRM_AUTH_SERVICE_SD.md)，本文不重複。權限彙總現況：登入/刷新時計算（角色權限 ∪ ALLOW覆寫 − DENY覆寫）寫入 access token，`PermissionsGuard` 對照 token 內清單。

## 2. 角色與權限

新增權限代碼（沿用既有 `AUTH.<資源>.<動作>` 命名，加進 `permission-catalog.ts`）：

| 代碼 | 涵蓋 |
|---|---|
| `AUTH.ORG.MANAGE` | 公司/部門的建立、編輯、停用、排序 |
| `AUTH.GROUP.MANAGE` | 群組建立/編輯/停用＋成員加入/移除 |
| `AUTH.USER.ONBOARD` | 建帳號＋寄/重寄邀請＋停用**非管理者**帳號（HR 的日常三件事；「非管理者」限制在服務層檢查對象，不靠權限代碼表達） |

角色（`AUTH_SUPER_ADMIN`／`AUTH_ADMIN` 已在 catalog，新增 `AUTH_HR`）：

| 角色 | 可以做什麼 |
|---|---|
| AUTH_SUPER_ADMIN | 程式明定通過所有檢查（不再依賴逐顆授權）；指派/移除本角色僅限持有者，且不可移除最後一位 |
| AUTH_ADMIN | AUTH.USER.MANAGE + ORG.MANAGE + GROUP.MANAGE + ROLE.MANAGE 等（完整後台） |
| AUTH_HR（新） | AUTH.USER.READ + AUTH.USER.ONBOARD，僅此 |
| 語意關係 | `AUTH.USER.MANAGE` 涵蓋 ONBOARD 的所有動作（持有 MANAGE 不需再拿 ONBOARD；端點檢查 any-of） |

## 3. 操作流程（正常路徑）

1. **組織維護**：管理員開組織管理頁 → 建公司 → 其下建部門（排序、停用）→ 使用者表單部門下拉即時反映
2. **HR 入職**：HR 開使用者管理頁 → 建帳號（姓名/email/部門必填，username 預設帶員編）→ 系統寄邀請信 → 員工自助設密碼轉 ACTIVE（沿用 auth-password-reset 機制）
3. **HR 離職**：HR 停用一般使用者 → refresh token 全撤（沿用既有停用行為）
4. **群組維護**：管理員建群組 → 用使用者搜尋選擇器加成員 → 名單即查即用
5. **暫時角色**：管理員指派角色＋到期日 → 到期後下一次 token 刷新即不含該角色權限

## 4. 狀態轉換

- 使用者狀態機不變（INVITED → ACTIVE ⇄ DISABLED）
- 公司/部門/群組：啟用 ⇄ 停用（無刪除，保關聯完整；停用部門前置條件＝無在職成員）

## 5. 例外流程（對應 Example Mapping 的負向情境）

- 無權限呼叫任何管理 API → 403（既有 guard 行為）
- 同公司重名部門、重複 email → 409/400 明確訊息
- 停用有成員的部門 → 422，提示先轉移
- HR 停用管理者、非超管指派超管、移除最後一位超管 → 403/422，逐條有 BDD 測試
- 併發：同時編輯組織以「最後寫入為準」（內部低頻操作，不做樂觀鎖）；「移除最後一位超管」檢查必須在交易內做，防兩人同時移除兩位造成歸零

## 6. 資料流與敏感資料

- 全部資料留在 AuthService DB；無匯出、無外部 AI、無新增跨系統資料流
- 員工 Email 屬 baseline §3 預設 Restricted：僅後台顯示與管理，本輪不新增任何外流面
- 遷移：讀既有 `auth_users.department` 字串 → 人工確認對應表 → 寫入新部門表與 `departmentId`

## 7. 與外部系統的互動

- 邀請信沿用既有 mail 基礎建設（nodemailer/Mailpit），無新外部依賴
- 無其他新增外部互動

## 8. 風險點與人工審核點（Steven 必看）

1. **權限彙總改動 = 心臟手術**：SUPER_ADMIN 放行與角色到期過濾都改在權限彙總/guard 這條路上，改壞＝所有 app 授權失效。要求：全 BDD 情境綠 + 既有 90 測試不退步 + 手動 smoke（超管/一般/HR 三帳號各走一輪）
2. **部門字串遷移的對應表要人工看過**：程式產出「舊字串 → 新部門」清單後，Steven 確認才執行寫入（字串可能有錯字、同義詞）
3. **email 空值帳號**：遷移前盤點，若有，逐筆決定補填或停用，不自動處理
4. **SUPER_ADMIN 上線第一刀**：seed 後把 SUPER_ADMIN 指給 Steven 本人帳號的動作要人工確認對象無誤（指錯人＝全權後門）
5. 「最後一位超管」防呆必須有測試證明在交易內生效

## 9. 待決策事項

- [ ] 無——PRD/Example Mapping 階段已全部拍板；權限代碼粒度依上表（若 Steven 對 `AUTH.USER.ONBOARD` 單顆設計有疑慮再談）

# Git 分支與發版流程

## 為什麼要改

目前 `Ystravel-CRM-Backend`、`Ystravel-CRM-Frontend` 等 repo 都是直接 commit 到 `main`,沒有 feature branch、沒有 PR。這樣做在功能單純時沒問題,但當:

- 多人同時改同一個 repo
- 改動涉及權限、SSO、客戶資料
- 需要有「這次改了什麼、為什麼這樣改」的紀錄可查

直接推 `main` 就會出問題(沒有 review 機會、難以回溯、難以回滾單一改動)。

## 採用:GitHub Flow

```text
main(永遠可部署)
  -> 從 main 開新 branch: feature/<feature-slug> 或 fix/<簡短描述>
  -> 開發、commit
  -> 開 Pull Request 回 main
  -> 走過 08-code-review-checklist.md
  -> CI 通過(lint + build + test,若尚未設定 CI,先手動跑過)
  -> 合併(merge 或 squash merge)
  -> 部署到 staging 驗證
  -> 部署到 production
```

沒有選擇 Git Flow 的長期 release/hotfix 分支結構,因為目前是內部系統、非合約交付型專案,分支越少越好維護。

## 分支命名

| 類型 | 命名 | 範例 |
|---|---|---|
| 新功能 | `feature/<feature-slug>` | `feature/crm-import-cleaning` |
| 修 bug | `fix/<簡短描述>` | `fix/order-amount-null` |
| 純文件/流程調整 | `docs/<簡短描述>` | `docs/dev-process-templates` |

`<feature-slug>` 與 `docs/features/<feature-slug>/` 的資料夾名稱保持一致,方便對照。

## Commit 訊息

不要求嚴格 Conventional Commits,但建議格式:

```text
<類型>: <一句話說明>

類型: feat / fix / docs / refactor / test / chore
```

範例:`feat: 新增客戶匯入清洗規則與候選比對`

## Pull Request 規則

即使目前只有一人開發,也要開 PR,不要直接 push 到 main。原因:

- PR 描述本身就是這次改動的紀錄,之後要查「這功能什麼時候、為什麼這樣做」會需要它。
- 用 [08-code-review-checklist.md](./08-code-review-checklist.md) 對自己的 PR 走一次審查,能抓到「邊做邊改、忘記檢查」的問題。
- 之後第 2、3 人加入,PR 是唯一需要改的協作習慣(其他文件流程不變)。

PR 描述至少包含:

- 對應的 `docs/features/<feature-slug>/`(或 issue/需求描述)
- 這次改了什麼
- 涵蓋了哪些 BDD scenario
- 手動測試過哪些情境
- 是否有資料庫 migration,如何執行

## 部署路徑(依 `docs/architecture/CRM_AUTH_SERVICE_SA.md` §7)

| 系統 | 正式路徑 |
|---|---|
| EIP | `https://eip.ystravel.com.tw/` |
| CRM Frontend | `https://eip.ystravel.com.tw/CRM/` |
| Auth Portal/Admin | `https://eip.ystravel.com.tw/Auth/` |
| AuthService API | `https://eip.ystravel.com.tw/api/auth/` |
| CRM API | `https://eip.ystravel.com.tw/api/crm/` |

合併到 `main` 後,依 [09-release-checklist.md](./09-release-checklist.md) 部署。

## CI(現況與下一步)

目前這幾個 repo 都還沒有 CI workflow。在 CI 補齊之前,PR 合併前**至少手動跑過**:

```bash
npm run lint
npm run build
npm test
```

補 CI(GitHub Actions,在 PR 上自動跑 lint/build/test)列為後續待辦,不阻擋這次流程上線。

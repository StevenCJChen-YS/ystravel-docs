# 測試策略

## 分工

| 測試類型 | 適合測什麼 | 工具 |
|---|---|---|
| Unit Test | 單一函式、規則、純邏輯(例如清洗規則、分數計算) | Jest |
| Integration Test | API + DB、Service 整合 | Jest + Prisma(測試用資料庫或 mock) |
| BDD Test | 核心商業行為、跨角色規則(例如「誰可以停用會員」) | `jest-cucumber` |
| E2E Test | 關鍵使用者路徑(登入 -> 進入 CRM -> 看到客戶列表) | 暫緩,待前端功能穩定後再評估(Playwright 為候選) |
| Manual QA | 探索性測試、視覺感受、複雜例外情境 | Steven 手動操作畫面 |

**重要原則:不要把所有測試都寫成 BDD。** BDD 用來保護「核心商業行為」,細節邏輯用 Unit Test 覆蓋就好,否則測試會變得又慢又難維護。

## 後端(NestJS)

- 既有:Jest,測試檔案 `*.spec.ts`,跟程式碼放在同一個模組資料夾。
- 新增:`jest-cucumber`,`.feature` 檔跟對應模組放一起,例如:

```text
src/imports/
  imports.controller.ts
  imports.service.ts
  imports-cleaning.feature
  imports-cleaning.steps.ts
```

- 執行方式不變:`npm test` 會同時跑 `*.spec.ts` 與 `.steps.ts`。

## 前端(Vue)

目前沒有測試框架。優先順序:

1. 先不引入測試框架,靠 Steven 的手動 QA(這是他最擅長的部分,畫面問題他自己看得出來)。
2. 當某個畫面的邏輯開始變複雜(不只是顯示,而是有分支判斷、計算),再針對那個邏輯補 Vitest unit test。
3. E2E 測試留到有明確痛點(例如常常在某個流程壞掉卻沒發現)時再導入。

## 測試資料

- 一律使用假資料,不使用真實旅客/客戶 Excel 或真實個資(依 `SECURITY_AND_ISO27001_BASELINE.md` §4.3)。
- 假資料要能反映真實資料的「形狀」(例如手機號碼格式、Email 格式),但內容是虛構的。

## 什麼時候必須要有測試

- 新增或修改 API:至少要有 Integration Test。
- 有明確業務規則(權限、狀態轉換、資料清洗規則):要有 BDD scenario。
- 純 UI 調整:不需要測試,手動看畫面確認即可。
- 修 bug:至少補一個能重現這個 bug 的測試,避免之後回歸。

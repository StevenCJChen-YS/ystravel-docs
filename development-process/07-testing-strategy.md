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

## BDD 與 TDD 的關係(2026-07-23 補)

BDD 是 TDD 的**重新表述**,不是取代——Dan North 2006 年發現大家學 TDD 都卡在「該測什麼」「測試怎麼命名」,
問題出在 `test` 這個字,於是換成 `should` / `behaviour`,才有了 Gherkin。**兩者不在同一層級**:

```text
外圈 BDD:使用者看得到的行為(.feature / Given-When-Then)
         「這個功能對不對?」 <- Steven 讀得懂、能確認
   └─ 內圈 TDD:一個函式/模組的行為
              「這段邏輯對不對?」 <- 只有工程師關心
```

實作時的迴圈(業界稱 Double Loop / Outside-In):

```text
外圈:寫一個會失敗的 .feature scenario
  └ 內圈:red -> green -> refactor,重複 N 次,直到 scenario 變綠
外圈:下一個 scenario
```

**判準——這條規則 Steven 需不需要看得懂?**

| 需要看得懂 | 不需要(純技術正確性) |
|---|---|
| 「誰可以停用會員」 | 電話 `0912-345-678` / `+886912345678` 正規化成同一個自然鍵 |
| 「沒權限的人看不到客戶列表」 | role × permission × 資料範圍的組合矩陣 |
| -> **BDD scenario** | -> **Unit Test**(用 `it.each` 表格,別寫成幾十個 scenario) |

## 測試的三個反模式(AI 最容易踩)

這三個是讓 AI 寫測試時**一定要檢查**的,尤其前兩個——AI 寫 step definition 時特別容易掉進去:

### 1. 同義反覆(Tautological)——最危險

斷言用「跟程式碼同一套邏輯」算出期望值,所以**建構上就必定通過,永遠不可能跟程式碼意見不同**。

```ts
// ❌ 這個測試永遠會過,測不出任何東西
expect(add(a, b)).toBe(a + b);

// ❌ step definition 裡最常見的版本:
//    呼叫 service 拿結果,然後用同樣的邏輯再算一次期望值
const expected = normalizePhone(input);      // 就是被測的那個函式
expect(result.phone).toBe(expected);

// ✅ 期望值必須來自獨立來源:手算的字面值、規格書上的數字
expect(normalizePhone('0912-345-678')).toBe('+886912345678');
```

### 2. 與實作綁死(Implementation-coupled)

mock 內部協作者、測 private method、走側門驗證(直接查資料庫而不是走介面)。
**判準:重構之後測試壞了,但行為根本沒變。**

### 3. 水平切片(Horizontal slicing)

先寫完所有測試、再寫所有實作。批量寫出來的測試驗證的是**想像中的行為**——
測到的是「形狀」而不是使用者行為,而且在理解實作之前就把測試結構鎖死了。
要**垂直切**:一個測試 -> 一段實作 -> 下一個,每一輪都根據上一輪學到的東西調整。

> 另外:**refactor 不屬於 red-green 迴圈**,它屬於 review 階段(見 [08](./08-code-review-checklist.md))。

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

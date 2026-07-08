# Customer Filter Rules V1 Analysis

- Source file: `C:\Users\HPC091\Desktop\客戶分析報表_綜合.xlsx`
- Total rows: `108,535`
- Total orders: `16,286`

## Current Agreed Exclusion Rules

- R1: `身份證字號` 為空者移除
- R2: `身分` 為 `導遊` 或 `領隊` 者移除
- R3: `警示訊息` 有值者移除
- R4: `來源帳號` 為 `永信旅行社`、`永信旅行社股份有限公司` 或 `永信福委會` 者移除
- R5: `護照號碼` 為 `未建旅客檔` 者移除

## Rule Impact By Row

- `R1_無身分證`: `9,507` rows (`8.76%`)
- `R2_身分為導遊或領隊`: `2,722` rows (`2.51%`)
- `R3_警示訊息有值`: `96` rows (`0.09%`)
- `R4_來源帳號為永信旅行社或永信福委會`: `2,818` rows (`2.6%`)
- `R5_護照號碼為未建旅客檔`: `8,968` rows (`8.26%`)

## Rule Impact By Order

- `R1_無身分證`: touches `1,147` orders (`7.04%`)
- `R2_身分為導遊或領隊`: touches `2,705` orders (`16.61%`)
- `R3_警示訊息有值`: touches `90` orders (`0.55%`)
- `R4_來源帳號為永信旅行社或永信福委會`: touches `2,287` orders (`14.04%`)
- `R5_護照號碼為未建旅客檔`: touches `974` orders (`5.98%`)

## Combined Result

### Interpretation A: Row-Level Exclusion

- Excluded rows: `12,508`
- Remaining rows: `96,027`
- Remaining orders: `13,175`
- Remaining traveler rows: `96,027`

### Interpretation B: Whole-Order Exclusion

- Excluded orders: `3,857`
- Remaining orders: `12,429`
- Remaining rows: `79,942`
- Remaining traveler rows: `79,942`

## Overlap

- Rows matching 0 rules: `96,027`
- Rows matching exactly 1 rule: `1,331`
- Rows matching exactly 2 rules: `10,755`
- Rows matching 3+ rules: `422`

## Notes

- 這批規則目前已明確是 `移除規則`。
- `員工身分證黑名單` 尚未提供，先列為後續待辦。
- `同業 / 直客` 分類尚未做，需等另一份名單資料（可能是 PDF）再建立對照規則。
- 目前尚未決定正式採 `row-level` 還是 `whole-order` 作為主邏輯，這會直接影響最後剩餘訂單數。

## Next Discussion

- 員工身分證黑名單格式
- 同業名單 PDF 轉為可比對清單的方式
- 正式要寫入資料庫的旅客欄位與訂單欄位
- 同業 / 直客 的最終判斷優先順序
